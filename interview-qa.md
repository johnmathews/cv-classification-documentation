# Interview Q&A

A running list of questions about the codebase and their answers, kept for interview prep.

---

## Q1 — Why is `ai_query(...)` passed as a SQL string in `classify.py`?

```python
response_col = when(
    col("_prompt").isNotNull(),
    expr(f"ai_query('{MODEL}', _prompt, responseFormat => '{RESPONSE_FORMAT}')"),
).otherwise(lit(None))
```

**What it does.** `response_col` defines the contents of a new `_response` column. For rows where `_prompt` is non-null (extraction-error rows are nulled out upstream), it calls Databricks' `ai_query` SQL function, which invokes a Foundation Model endpoint server-side and returns the JSON response. The `responseFormat` named arg pins a JSON schema so the model can only emit `{bracket, confidence}` with `bracket` constrained to the allowed enum. Error rows fall through to `lit(None)`, so no tokens are spent on them.

**Why it's a SQL string.** `ai_query` is a Databricks **SQL** function — there is no PySpark Python wrapper for it. To call SQL-only functions from the DataFrame API, the options are `expr()`, `selectExpr()`, or `spark.sql()`. The string form is unavoidable; this matches the pattern shown in Databricks' own `ai_query` documentation.

**Tradeoffs noted.**

- F-string interpolation into SQL is normally a smell (injection risk). Here `MODEL` and `RESPONSE_FORMAT` are module constants, so it is safe — but reads as if it could be unsafe.
- The `RESPONSE_FORMAT` JSON happens to contain no single quotes, so wrapping it in `'...'` parses cleanly. If anyone ever added a description with an apostrophe, the SQL would break silently.
- A module-level `assert "'" not in RESPONSE_FORMAT` was added as a guard.

**Cleaner alternatives considered.**

- `call_function("ai_query", lit(MODEL), col("_prompt"), lit(RESPONSE_FORMAT))` — Spark 3.5+, but named args (`responseFormat =>`) aren't well-supported, so it falls short for this specific function.
- Parameterized `spark.sql(...)` with `:model` / `:format` placeholders — cleaner but breaks out of the DataFrame chain.

**Verdict.** Current form is idiomatic Databricks; keep it, with the single-quote guard.

---

## Q2 — Why are some imports inside `main()` rather than at the top of `ingest.py`?

```python
from cv_classification.common import log, parse_args, use_namespace  # top-level

def main() -> None:
    ...
    from databricks.sdk.runtime import spark            # function-level
    from pyspark.sql.functions import sha2              # function-level
```

It's a deliberate Databricks-bundle pattern. The split is along one axis: **does this import only work on a Databricks cluster?**

- **Top of module** — anything that works anywhere: pure Python, project helpers (`common.parse_args`, etc.).
- **Inside `main()`** — Databricks-runtime-specific or Spark-specific:
  - `from databricks.sdk.runtime import spark` only resolves inside a live Databricks cluster; it grabs the active SparkSession.
  - `from pyspark.sql.functions import sha2` pulls in PySpark, which is heavy and is provided by the cluster runtime, not by the wheel's `pyproject.toml`.

**Why it matters.**

1. **The module stays importable in any environment.** `pytest` on a laptop, IDE static analysis, the `databricks bundle` packaging step — none of them have a SparkSession, and some don't even have `databricks.sdk.runtime` installed. Module-level imports there would fail or be slow.
2. **Honest dependency surface.** PySpark only appearing inside `main()` advertises the contract: "you need a Spark runtime to _execute_ this stage, not to import it."
3. **Faster cold starts for tests.** The wheel's `[project.scripts]` wiring imports the module to find `main`; deferring runtime-only imports keeps that cheap.

`pyspark.sql.functions` _could_ arguably live at the top (PySpark is pip-installable), but keeping it next to the runtime import makes the convention uniform: "everything that needs a cluster lives in `main()`."

---

## Q3 — How do we capture the resume's profession/category from the dataset?

The Kaggle dataset is laid out as `<root>/<CATEGORY>/<id>.pdf` (e.g. `.../ACCOUNTANT/10001.pdf`). The `binaryFile` reader already exposes `path`, so the category is the parent directory.

**Recommended approach (readable):**

```python
from pyspark.sql.functions import element_at, lower, sha2, split

deduped = (
    raw
    .withColumn("sha256", sha2(raw["content"], 256))
    .withColumn("category", lower(element_at(split(raw["path"], "/"), -2)))
    .dropDuplicates(["sha256"])
)
```

`element_at(..., -2)` grabs the second-to-last path segment. `lower(...)` normalizes the uppercase folder names so downstream comparisons are case-stable.

**Defensive alternative (`regexp_extract`):**

```python
from pyspark.sql.functions import lower, regexp_extract, sha2

.withColumn("category", lower(regexp_extract(raw["path"], r"/([^/]+)/[^/]+\.pdf$", 1)))
```

This returns NULL if the path shape changes, which is louder than a silent wrong answer.

**Caveat worth flagging.** Dedup is on `sha256` alone, so if the same PDF appears under two categories one is dropped. Almost certainly not the case in this dataset, but worth a one-time check:

```python
deduped.groupBy("sha256").agg(countDistinct("category").alias("n_cats")).filter("n_cats > 1").count()
```

Also: `category` needs to be added to the bronze schema in `docs/04-pipeline-structure.md` and propagated through `preprocess.py` so it survives into silver/gold.

---

## Q4 — What does the `binaryFile` reader chain do in `ingest.py`?

```python
raw = (
    spark.read.format("binaryFile")
    .option("pathGlobFilter", "*.pdf")
    .option("recursiveFileLookup", "true")
    .load(SOURCE_URL)
)
```

It builds a Spark DataFrame where each row is one file read from cloud storage as raw bytes.

- **`spark.read`** — entry point to the DataFrameReader builder.
- **`.format("binaryFile")`** — selects Spark's built-in binary-file source. Each matching file becomes one row with a fixed schema:

  | Column             | Type      | Meaning                                       |
  | ------------------ | --------- | --------------------------------------------- |
  | `path`             | string    | Full URI (`abfss://.../ACCOUNTANT/10001.pdf`) |
  | `modificationTime` | timestamp | Last modified time from storage               |
  | `length`           | long      | File size in bytes                            |
  | `content`          | binary    | Raw bytes of the file                         |

  Standard pattern for ingesting non-tabular files (PDFs, images, audio): one row per file, downstream stages decode the bytes via UDF.

- **`.option("pathGlobFilter", "*.pdf")`** — leaf-name glob; only PDFs are ingested.
- **`.option("recursiveFileLookup", "true")`** — descend into subdirectories AND tell Spark not to interpret directory names as Hive-style partitions. Without this, `<root>/ACCOUNTANT/...` would be auto-promoted to a `category=ACCOUNTANT` partition column. With it set to `true`, the category info stays inside `path` — which is why Q3 parses it out.
- **`.load(SOURCE_URL)`** — runs the metadata scan (file listing) and returns the DataFrame. Reading bytes is still lazy; it happens when an action like `.count()` or `.write` fires.

---

## Q5 — Does `raw["content"]` hold extracted text? And does sha256 dedup catch metadata-only differences?

**`content` is raw bytes, not text.** Spark's `binaryFile` reader does no decoding — it `read()`s the file and dumps its on-disk bytes into a column of type `binary`. For a PDF, that means the literal `%PDF-1.x...` byte stream. Text extraction happens later, in `preprocess.py`, where `pypdf.PdfReader` parses those bytes.

**Whether two "same-content" files hash equal depends on what kind of metadata differs:**

- **File-system metadata** (path, modificationTime, filename) — NOT part of `content`, NOT hashed. Two files with identical bytes but different paths or upload times produce **identical sha256 hashes** and dedup correctly.
- **Embedded PDF metadata** (`/Author`, `/Title`, `/CreationDate`, `/ModDate`, `/Producer`, document ID in the trailer) — IS part of the file bytes, IS hashed. Two PDFs that render identical content but were exported by different software, or even just opened and re-saved, will have **different sha256 hashes** and will NOT be deduped.

**Practical consequence.** A candidate who exports the same CV twice (e.g. once from Word and once from LibreOffice) creates two byte-distinct PDFs that both end up in `cv_bronze`. Byte-level dedup misses them.

**If true content dedup is required**, shift it downstream — dedup on normalized extracted text in `cv_silver` (`silver.dropDuplicates(["text"])`). Heavier, but catches near-duplicates byte hashing can't see. For this dataset (curated Kaggle), byte-level dedup is sufficient; worth a sentence in the report acknowledging the limit.

---

## Q6 — How do I run a single stage of the pipeline (e.g. just `ingest`) on Databricks?

The bundle defines `ingest → preprocess → classify` as chained tasks via `depends_on`, so a default `databricks bundle run cv_pipeline` runs all three. To run one in isolation:

**1. CLI single task:**

```bash
cd cv_classification
databricks bundle deploy --target dev
databricks bundle run -t dev cv_pipeline --only ingest
```

`--only <task_key>` runs that task only, ignoring `depends_on`. Comma-separated to run more than one (e.g. `--only ingest,preprocess`). Assumes any upstream tables it reads already exist.

**2. Databricks Jobs UI:** open the job, click the task in the DAG, "Run task". Same effect, with a live log view.

**3. Notebook on the cluster (fastest dev loop):**

```python
import sys
sys.argv = ["ingest", "--catalog", "cv_classification_catalog", "--schema", "dev"]
from cv_classification.ingest import main
main()
```

Skips the job machinery entirely. Still requires `bundle deploy` first so the cluster has the latest wheel.

All stages use `mode("overwrite")` on their writes, so re-running is idempotent — but if upstream code has changed without redeploy, you'll be running stale logic on the cluster.

---

## Q7 — Why doesn't `uv build` need to be run manually before `databricks bundle deploy`?

Because the bundle declares the build step itself. In `databricks.yml`:

```yaml
artifacts:
 python_artifact:
  type: whl
  build: uv build --wheel
```

The `artifacts` block tells the Databricks CLI that there is a wheel artifact and that the build command is `uv build --wheel`. When `databricks bundle deploy` runs, the CLI executes that `build` command first, then uploads the resulting wheel from `dist/` to the workspace. The job's `environments` section (`dependencies: - ../dist/*.whl`) then references it for the cluster install.

So the minimal dev loop is just:

```bash
databricks bundle deploy --target dev
databricks bundle run -t dev cv_pipeline --only ingest
```

Manual `uv build` is only useful for inspecting the wheel locally or in CI pipelines that need the artifact independently of a Databricks deploy.

---

## Q8 — How is `--limit 50` wired all the way from CLI to the LLM call site?

`--limit` is a smoke-test knob on the `classify` stage: it short-circuits the row count before any pay-per-token `ai_query` calls fire. Default `0` means no cap. The plumbing is four hand-offs:

**1. Argparse declaration** — `common.py:13` adds it to the shared parser used by all three stages:

```python
parser.add_argument("--limit", type=int, default=0, help="0 = no limit")
```

**2. Stage applies it** — `classify.py:41-43`:

```python
if args.limit > 0:
    silver = silver.limit(args.limit)
```

Applied _before_ the prompt column or `ai_query` column is built, so capped-out rows are never sent to the model.

**3. Bundle declares a job-level parameter** — `resources/cv_pipeline.job.yml:13-14`:

```yaml
parameters:
 - name: limit
   default: "0"
```

Stringly-typed because Databricks job parameters are strings; argparse coerces back to int via `type=int`.

**4. The classify task substitutes the parameter into its CLI args** — `cv_pipeline.job.yml:52-53`:

```yaml
- "--limit"
- "{{job.parameters.limit}}"
```

Databricks expands `{{job.parameters.limit}}` at task launch.

**Invocation override:**

```bash
databricks bundle run cv_pipeline -t dev --params limit=50
```

`--params limit=50` overrides the bundle's default just for that one run — no redeploy needed to flip between smoke and full runs. Catalog and schema are wired through the same indirection for consistency, even though they don't change between runs.

Only `classify` consumes `--limit` because it's the only stage with variable per-row cost. Ingest and preprocess always do all the rows.

---

## Q9 — Why is a UDF needed in `preprocess.py`, and why are PySpark UDFs slow?

**Why we need one.** The `cv_bronze` row carries the raw PDF bytes in `content`, and Spark has no built-in function for parsing PDFs. `pypdf` is a Python-only library, so the only way to call it on every row is to wrap it in a UDF:

```python
def _extract(content: bytes) -> tuple[str, int]:
    reader = PdfReader(io.BytesIO(content))
    text = "\n".join((page.extract_text() or "") for page in reader.pages)
    return text, len(reader.pages)

extract_udf = udf(_extract, extract_schema)
silver = bronze.withColumn("extracted", extract_udf(col("content")))
```

`udf(...)` registers the Python function with Spark; the schema arg pins the return type so Catalyst knows `extracted` is a `StructType` with `text` and `num_pages` fields. When the action fires, Spark applies the function row by row.

**Why Python UDFs are slow.**

1. **JVM ↔ Python serialization on every row.** Spark runs on the JVM. Python UDFs execute in a separate Python worker process per task slot. Each row's input is serialized in the JVM, sent through a pipe, deserialized in Python, processed, re-serialized, sent back, deserialized in the JVM. That round-trip is fixed cost; for cheap functions it dwarfs the actual work.
2. **No Catalyst optimization.** The query optimizer can't reason about Python code, so it can't push filters through the UDF, can't reorder, can't constant-fold, can't fuse it with other operations. The UDF is an opaque black box.
3. **No code generation (Tungsten).** Built-in Spark expressions compile to Java bytecode at query plan time. UDFs run as interpreted Python, with no codegen.
4. **GIL-bound per worker.** Each Python worker is a single-threaded process, so executor parallelism caps at "one row at a time per Python worker" — coarse compared to JVM expressions running on the executor's native thread pool.

**Mitigations** (not used here, but worth knowing):

- **Pandas UDFs** (`@pandas_udf`) batch rows into Arrow record batches before the JVM↔Python crossing, often 10–100× faster than row-at-a-time UDFs for cheap functions. For `pypdf` extraction the batch wouldn't help much because the work per row is heavy already.
- **Push the work into a SQL function.** Whenever the operation is expressible as built-in Spark functions or a SQL function, use those instead. This is exactly why classify uses `ai_query` (a SQL function, JVM-native) rather than a Python UDF wrapping the OpenAI/Anthropic SDK — the UDF version would single-thread on the driver or require a hand-rolled thread pool inside Python workers.

**Lesson learned the hard way in this project.** The first end-to-end run had `silver.write` taking ~7 min and a follow-up `silver.filter(...).count()` taking another ~6 min — Spark re-ran the entire `pypdf` UDF because the post-write count used the lazy `silver` DataFrame whose query plan still contained the UDF. The fix: rebind to `spark.table("cv_silver")` after writing, so post-write reads come from the materialized Delta table. That cut preprocess from 13m 21s → 7m 25s. UDFs are slow _and_ invisible to the optimizer, so accidental re-execution is easy and expensive.

---

## Q10 — Does this project need data contract validation?

No, but it's a useful thing to acknowledge in the report.

**What "data contract validation" means in this context.** Explicit, enforced assertions about table schemas and invariants at stage boundaries — typically with a tool like Great Expectations, Soda, dbt tests, or Databricks DLT expectations. E.g.:

- `cv_bronze.sha256` is non-null and unique
- `cv_silver.text_length > 0` for non-error rows
- `cv_gold.experience_bracket` ∈ {0-2, 3-5, 5-7, 7-10, 10+} or NULL
- `cv_silver.category` is non-null

**Why it isn't worth the ceremony here.**

- **Schemas are tiny and stable** — five columns in bronze, six in silver, eight in gold. The schema _is_ the contract; reading the code is faster than reading a tests file.
- **`ai_query`'s structured output already enforces the most important invariant.** The JSON-schema `responseFormat` constrains `bracket` to a literal enum, so an out-of-range bracket can't reach gold. That's a stronger guarantee than a post-hoc check.
- **Existing log lines act as smoke checks.** Every stage prints input row counts, output row counts, and error counts. A drift between stages would show up immediately in the Workflows UI.
- **The brief explicitly says "complexity does not equal sophistication"** and caps the task at four hours. Adding GE/Soda would be a measurable chunk of that.

**The lightweight version that _would_ fit.** A handful of `assert` statements after each write would catch the genuine failure modes for this project's scale:

```python
saved = spark.table("cv_silver")
assert saved.count() > 0, "cv_silver is empty"
assert saved.filter(col("category").isNull()).count() == 0, "category leaked NULL"
```

Worth one line in the report under "What would extend this with more time": at production scale, expectations / dbt tests at every layer become essential — particularly between bronze and silver, where upstream schema changes (Kaggle reorganising folder names, for instance) would silently re-shape downstream tables.

---

## Q11 — In `ingest.py`, what does `parse_args("Ingest CVs from blob storage into cv_bronze")` actually do?

`parse_args` is the shared CLI helper in `common.py:9-14`:

```python
def parse_args(description: str) -> argparse.Namespace:
    parser = argparse.ArgumentParser(description=description)
    parser.add_argument("--catalog", required=True)
    parser.add_argument("--schema", required=True)
    parser.add_argument("--limit", type=int, default=0, help="0 = no limit")
    return parser.parse_args()
```

Step by step on the call `parse_args("Ingest CVs from blob storage into cv_bronze")`:

1. **`description` is just human-readable help text.** It's passed to `ArgumentParser(description=...)`, where it appears at the top of `--help` output. The reason it's a parameter — rather than hard-coded — is that all three stages share this helper, and each one wants its own description in `--help`.
2. **Three flags are declared on the parser**: `--catalog` (required), `--schema` (required), `--limit` (int, default 0).
3. **`parser.parse_args()` is called with no arguments**, so argparse reads from `sys.argv[1:]` automatically.
4. **Returns an `argparse.Namespace`** — a lightweight object whose attributes are the parsed values. After the call, `args.catalog`, `args.schema`, and `args.limit` are accessible in `main()`.

**Where `sys.argv` actually comes from in this project:**

- **Locally (e.g. dev / pytest):** invoked as `python -m cv_classification.ingest --catalog ... --schema ...`; standard CLI invocation.
- **On Databricks:** the `python_wheel_task` block in `cv_pipeline.job.yml` declares:
  ```yaml
  parameters:
   - "--catalog"
   - "{{job.parameters.catalog}}"
   - "--schema"
   - "{{job.parameters.schema}}"
  ```
  Databricks substitutes the job parameters and passes that array as `sys.argv[1:]` when the wheel's entry-point function is invoked.

**Why centralise the parsing.** All three stages need exactly the same `--catalog` / `--schema` interface; only `classify` additionally cares about `--limit` (which defaults to 0 and is a no-op everywhere else). Putting the parser in `common.py` keeps the three stage modules free of argparse boilerplate and guarantees the flag names stay in sync.
