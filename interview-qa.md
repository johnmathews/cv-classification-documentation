# Interview Q&A

A topic-grouped reference of questions about the codebase and their answers, kept for interview prep. Each entry leads with a one-line **TL;DR**; drill into the body if pressed.

## Contents

**Pipeline architecture & orchestration**

- [Q2 — Why are some imports inside `main()` rather than at the top of the module?](#q2)
- [Q6 — How do I run a single stage of the pipeline on Databricks?](#q6)
- [Q7 — Why doesn't `uv build` need to be run manually before `databricks bundle deploy`?](#q7)
- [Q8 — How is `--limit 50` wired all the way from CLI to the LLM call site?](#q8)
- [Q11 — What does `parse_args(...)` actually do?](#q11)
- [Q14 — How are tests gated against deploys without setting up CI?](#q14)
- [Q17 — Why Databricks Asset Bundles + Jobs, not DLT?](#q17)

**Ingestion & preprocessing**

- [Q3 — How is the resume's profession/category captured from the dataset?](#q3)
- [Q4 — What does the `binaryFile` reader chain do in `ingest.py`?](#q4)
- [Q5 — Does `raw["content"]` hold extracted text? And does sha256 dedup catch metadata-only differences?](#q5)
- [Q9 — Why is a UDF needed in `preprocess.py`, and why are PySpark UDFs slow?](#q9)

**Classification & LLM**

- [Q1 — Why is `ai_query(...)` passed as a SQL string in `classify.py`?](#q1)
- [Q18 — Why `ai_query` over OpenAI / Anthropic / Bedrock directly?](#q18)
- [Q21 — How would you measure LLM classification accuracy without ground truth?](#q21)

**Embedding & Vector Search**

- [Q12 — Why Delta Sync Index with self-managed embeddings, and why no choice in the matter on Free Edition?](#q12)
- [Q13 — What surprised you running the chunk + index + retrieval surface end-to-end?](#q13)
- [Q15 — What actually is `cv_silver_chunks_index`, and where does Vector Search sit in Mosaic AI?](#q15)
- [Q16 — What is Change Data Feed, and where does this project use it?](#q16)

**Data quality, cost, security, productionisation**

- [Q10 — Does this project need data contract validation?](#q10)
- [Q19 — What does this cost to run, and at production scale?](#q19)
- [Q20 — How would you handle PII / GDPR for CV data?](#q20)
- [Q22 — What does the v2 production deployment look like?](#q22)

---

# Pipeline architecture & orchestration

<a id="q2"></a>

## Q2 — Why are some imports inside `main()` rather than at the top of `ingest.py`?

**TL;DR:** Module-level imports cover anything that works anywhere; runtime-only imports (PySpark, Databricks SDK) live inside `main()` so the module stays importable in pytest, IDEs, and the bundle build with no SparkSession.

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

<a id="q6"></a>

## Q6 — How do I run a single stage of the pipeline (e.g. just `ingest`) on Databricks?

**TL;DR:** `databricks bundle run -t dev cv_pipeline --only <task_key>` runs a single stage, ignoring `depends_on`. Comma-separated for multiple. Notebook-direct invocation of the stage's `main()` is the fastest dev loop.

The bundle defines five tasks — `ingest`, `preprocess`, `chunk`, `index`, `classify` — wired via `depends_on`. `ingest → preprocess` is serial; `chunk` and `classify` then fan out from `preprocess` in parallel; `index` is sequenced after `chunk`. A default `databricks bundle run cv_pipeline` runs the full DAG. To run one task in isolation:

**1. CLI single task:**

```bash
cd cv_classification
databricks bundle deploy --target dev
databricks bundle run -t dev cv_pipeline --only ingest
```

`--only <task_key>` runs that task only, ignoring `depends_on`. Comma-separated to run more than one (e.g. `--only chunk,index`). Assumes any upstream tables it reads already exist — running `--only classify` against an empty `cv_silver` will fail.

**2. Databricks Jobs UI:** open the job, click the task in the DAG, "Run task". Same effect, with a live log view.

**3. Notebook on the cluster (fastest dev loop):**

```python
import sys
sys.argv = ["classify", "--catalog", "cv_classification_catalog", "--schema", "dev", "--limit", "50"]
from cv_classification.classify import main
main()
```

Skips the job machinery entirely. Still requires `bundle deploy` first so the cluster has the latest wheel.

All stages use `mode("overwrite")` on their writes, so re-running is idempotent — but if upstream code has changed without redeploy, you'll be running stale logic on the cluster.

---

<a id="q7"></a>

## Q7 — Why doesn't `uv build` need to be run manually before `databricks bundle deploy`?

**TL;DR:** The bundle's `artifacts.python_artifact.build` step declares the build command itself, so `databricks bundle deploy` runs it before uploading the wheel. Manual `uv build` is only useful for inspecting the wheel locally.

In `databricks.yml`:

```yaml
artifacts:
 python_artifact:
  type: whl
  build: uv run pytest && uv build --wheel
```

The `artifacts` block tells the Databricks CLI that there is a wheel artifact and gives it the build command. When `databricks bundle deploy` runs, the CLI executes that `build` command first, then uploads the resulting wheel from `dist/` to the workspace. The job's `environments` section (`dependencies: - ../dist/*.whl`) then references it for the cluster install.

So the minimal dev loop is just:

```bash
databricks bundle deploy --target dev
databricks bundle run -t dev cv_pipeline --only classify --params limit=50
```

Manual `uv build` is only useful for inspecting the wheel locally or in CI pipelines that need the artifact independently of a Databricks deploy.

---

<a id="q8"></a>

## Q8 — How is `--limit 50` wired all the way from CLI to the LLM call site?

**TL;DR:** Four hand-offs — argparse declaration in `common.py`, stage applies it before the LLM column is built, bundle declares a job-level parameter, classify task substitutes `{{job.parameters.limit}}` into its CLI args. Override per-run with `--params limit=50`, no redeploy needed.

`--limit` is a smoke-test knob on the `classify` stage: it short-circuits the row count before any pay-per-token `ai_query` calls fire. Default `0` means no cap. The plumbing:

**1. Argparse declaration** — `common.py`:

```python
parser.add_argument("--limit", type=int, default=0, help="0 = no limit")
```

**2. Stage applies it** — `classify.py`:

```python
if args.limit > 0:
    silver = silver.limit(args.limit)
```

Applied _before_ the prompt column or `ai_query` column is built, so capped-out rows are never sent to the model.

**3. Bundle declares a job-level parameter** — `resources/cv_pipeline.job.yml`:

```yaml
parameters:
 - name: limit
   default: "0"
```

Stringly-typed because Databricks job parameters are strings; argparse coerces back to int via `type=int`.

**4. The classify task substitutes the parameter into its CLI args:**

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

Only `classify` consumes `--limit` because it's the only stage with variable per-row cost. The other four stages always do all the rows.

---

<a id="q11"></a>

## Q11 — In `ingest.py`, what does `parse_args("Ingest CVs from blob storage into cv_bronze")` actually do?

**TL;DR:** Shared CLI helper in `common.py` — declares `--catalog`, `--schema`, `--limit`, parses `sys.argv`, returns an `argparse.Namespace`. Centralised so all five stages share one CLI surface and the flag names stay in sync.

```python
def parse_args(description: str) -> argparse.Namespace:
    parser = argparse.ArgumentParser(description=description)
    parser.add_argument("--catalog", required=True)
    parser.add_argument("--schema", required=True)
    parser.add_argument("--limit", type=int, default=0, help="0 = no limit")
    return parser.parse_args()
```

Step by step on the call `parse_args("Ingest CVs from blob storage into cv_bronze")`:

1. **`description` is just human-readable help text.** It's passed to `ArgumentParser(description=...)`, where it appears at the top of `--help` output. The reason it's a parameter — rather than hard-coded — is that all five stages share this helper, and each one wants its own description in `--help`.
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

**Why centralise the parsing.** All five stages need exactly the same `--catalog` / `--schema` interface; only `classify` additionally cares about `--limit` (which defaults to 0 and is a no-op everywhere else). Putting the parser in `common.py` keeps the stage modules free of argparse boilerplate and guarantees the flag names stay in sync.

---

<a id="q14"></a>

## Q14 — How are tests gated against deploys without setting up CI?

**TL;DR:** The bundle's build step is `uv run pytest && uv build --wheel`. A red test fails the build, the wheel never uploads, the deploy aborts. Zero new infrastructure — last-mile guard, not a substitute for branch-protection CI.

The Databricks Asset Bundle declares its build step in `databricks.yml`:

```yaml
artifacts:
  python_artifact:
    type: whl
    build: uv run pytest && uv build --wheel
```

The `&&` chains pytest before the wheel build. `databricks bundle deploy` invokes that command end-to-end, so a red test fails the build, which means the wheel never uploads, which means the deploy aborts.

**Why this works specifically here.** The whole pipeline is built on one wheel artefact, declared as a single `artifacts` entry. The Databricks CLI runs the `build:` shell command verbatim, so anything you can express on a shell line becomes a pre-build check. The pattern generalises beyond pytest:

- `uv run ruff check src/ && uv build --wheel` — lint gate
- `uv run mypy src/ && uv build --wheel` — type-check gate
- `uv run pytest && uv run ruff check src/ && uv build --wheel` — chain multiple

**How to verify it's actually running.** The Databricks CLI suppresses subcommand stdout by default, so a successful `bundle deploy` doesn't visibly show pytest output. Two ways to confirm:

1. `databricks bundle deploy --debug` — surfaces subprocess output including pytest's pass count
2. Deliberately break a test, redeploy, observe the deploy fail at "Building python_artifact…" (the definitive proof — done once during this case study)

**Why it isn't a complete CI replacement.** It only fires on `bundle deploy`, not on every commit. A teammate could merge code with broken tests if they don't deploy. For a single-author case study this is fine; for a team project, a GitHub Actions workflow on push to `main` is the next step up. The bundle-stage gate complements that — it's a last-mile guard, not a substitute for CI.

---

<a id="q17"></a>

## Q17 — Why Databricks Asset Bundles + Jobs, not DLT (Lakeflow Declarative Pipelines)?

**TL;DR:** DLT is the cleaner shape for pure Delta-to-Delta workloads with quality checks and incremental updates as first-class concerns. Here, the LLM step doesn't fit the declarative materialisation primitive, the dataset is too small for streaming to matter, and iteration ergonomics + local pytest gating are materially better with imperative Jobs.

Both are first-class Databricks resources for orchestrating data work, both can be deployed via Asset Bundles, and they are routinely confused. The split:

|                        | **Jobs** (chosen here)                                                       | **DLT / Lakeflow Declarative Pipelines**                |
| ---------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------- |
| Programming model      | Imperative DAG of arbitrary tasks (Python wheel, notebook, SQL, dbt, etc.)   | Declarative — `@dlt.table` functions; Databricks infers the DAG |
| Dependency wiring      | Explicit `depends_on` in YAML                                                | Inferred from `dlt.read("upstream_table")` references   |
| Incremental ingest     | Hand-rolled (the developer chooses overwrite vs merge vs streaming)          | Built-in; STREAMING tables track CDF / Auto Loader      |
| Data quality           | Hand-rolled (asserts, GE, dbt tests)                                         | `@dlt.expect_or_drop` / `_or_fail` baked into runtime   |
| Schema evolution       | Hand-rolled                                                                  | Managed                                                 |
| Scheduling             | Cron, file arrival, continuous                                               | Triggered or continuous pipeline run                    |
| Compute model          | Any cluster, including serverless                                            | DLT-managed pipeline cluster (separate billing line)    |
| Local testability      | Pure Python; pytest works                                                    | Can't fully run `@dlt.table` outside a DLT pipeline     |

DLT is the cleaner shape **when the workload is pure Delta-to-Delta with quality checks and incremental updates as first-class concerns** — bronze → silver → gold using Auto Loader, evolving CSV schemas, expectation-driven dropping of bad rows, near-real-time streaming. For that use case, the declarative model genuinely earns its weight.

### Why it was rejected here

1. **The LLM step doesn't fit the declarative model.** `classify` calls `ai_query()` against a Foundation Model API endpoint — pay-per-token, non-deterministic, externally dependent. DLT's `@dlt.table` is a *materialisation* primitive; it expects "given upstream, produce downstream" to be a deterministic function with idempotent re-runs. Wrapping a paid LLM call in `@dlt.table` works mechanically, but every prompt tweak triggers a full re-materialisation of the dependent tables, and the runtime's incremental smarts (recompute only what changed) lose meaning when "what changed" is the prompt, not the input data.

2. **Dataset size makes the streaming / incremental story irrelevant.** 2,484 PDFs ingested once; no Auto Loader, no daily increments, no schema evolution. DLT's headline features — STREAMING tables, materialised views, incremental refresh — would be carrying no weight. At 100M rows arriving daily this calculus flips entirely.

3. **Iteration ergonomics.** Each pipeline stage is an entry-point function (`ingest:main`, `chunk:main`, etc.) that can be called directly from a notebook, a pytest test, or `python -m`. Hand-running one stage on the cluster while tweaking the prompt is a 10-second loop. DLT pipelines run as a unit; partial reruns are possible but coarser. For a four-hour case study with pay-per-token cost on every full pass, the imperative shape is materially cheaper to iterate.

4. **Test gating.** The bundle's `artifacts.python_artifact.build` step runs `uv run pytest && uv build --wheel` (see [Q14](#q14)). DLT pipelines don't have an equivalent local-pytest gate — `@dlt.table` decorated functions can't be unit-tested in isolation without mocking the DLT runtime, which is brittle.

5. **Free Edition compute.** DLT pipelines run on a managed pipeline cluster type that has separate Free Edition limits and slower cold-start than the serverless job clusters used here. Already running into endpoint-provisioning latency for Vector Search; adding a second class of cold-start to the workflow wasn't worth it.

### What was given up

- **`@dlt.expect` quality gates.** Built-in row-level expectations with drop / fail / quarantine semantics. Replaced here by lightweight log-line invariants (input/output row counts per stage, `__EXTRACTION_ERROR__` count) and the `responseFormat` JSON schema constraining `bracket` to a literal enum at the LLM step. See [Q10](#q10) on data contracts.
- **Auto-managed dependency inference.** `cv_pipeline.job.yml` declares `depends_on` explicitly between tasks. With five stages this is trivial; at twenty-plus tables the inference would start to earn its keep.
- **Lineage UI polish.** DLT's pipeline view renders the DAG with row counts and quality metrics per node. Jobs' Workflows UI shows tasks-in-a-DAG but doesn't graph table-level lineage in the same way (UC lineage covers it separately).

### Where the decision flips

If the brief had been "ingest 100k CVs/day from an SFTP drop, dedupe against the existing corpus, enforce schema contracts at every layer, expose a near-real-time gold view to recruiters" — DLT becomes the correct tool. The LLM classification would be split off into a downstream Job that consumes a clean `cv_silver` materialised view, keeping DLT's declarative rigour where it pays off and the imperative LLM step isolated from it. That hybrid (DLT for the deterministic stack, Jobs for the LLM tier) is the production shape this project would evolve into at scale (see [Q22](#q22)).

### Reference

`docs/02-databricks-bundles.md` covers the full Bundles + Jobs surface — wheel artefact, job parameters, environment targets, deploy/run/destroy lifecycle, and why notebooks were rejected as the deliverable shape.

---

# Ingestion & preprocessing

<a id="q3"></a>

## Q3 — How do we capture the resume's profession/category from the dataset?

**TL;DR:** Parse the parent directory out of `path` at ingest time using `element_at(split(path, "/"), -2)`, lower-case it, and propagate `category` through silver and gold. Already implemented — `ingest.py:29` writes it to bronze, `preprocess.py:49` carries it into silver.

The Kaggle dataset is laid out as `<root>/<CATEGORY>/<id>.pdf` (e.g. `.../ACCOUNTANT/10001.pdf`). The `binaryFile` reader already exposes `path`, so the category is the parent directory.

**Approach used:**

```python
from pyspark.sql.functions import element_at, lower, sha2, split

deduped = (
    raw
    .withColumn("sha256", sha2(raw["content"], 256))
    .withColumn("category", lower(element_at(split(raw["path"], "/"), -2)))
    .dropDuplicates(["sha256"])
)
```

`element_at(..., -2)` grabs the second-to-last path segment. `lower(...)` normalises the uppercase folder names so downstream comparisons are case-stable.

**Defensive alternative (`regexp_extract`):**

```python
.withColumn("category", lower(regexp_extract(raw["path"], r"/([^/]+)/[^/]+\.pdf$", 1)))
```

This returns NULL if the path shape changes, which is louder than a silent wrong answer. The `element_at` form was preferred for readability; at scale the regex form would be safer.

**Caveat worth flagging.** Dedup is on `sha256` alone, so if the same PDF appears under two categories one is dropped. Almost certainly not the case in this dataset, but worth a one-time check:

```python
deduped.groupBy("sha256").agg(countDistinct("category").alias("n_cats")).filter("n_cats > 1").count()
```

---

<a id="q4"></a>

## Q4 — What does the `binaryFile` reader chain do in `ingest.py`?

**TL;DR:** Builds a Spark DataFrame where each row is one PDF read as raw bytes. `recursiveFileLookup=true` descends subdirectories *and* prevents Spark from auto-promoting folder names to Hive partitions, so `category` info stays inside `path`.

```python
raw = (
    spark.read.format("binaryFile")
    .option("pathGlobFilter", "*.pdf")
    .option("recursiveFileLookup", "true")
    .load(SOURCE_URL)
)
```

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

<a id="q5"></a>

## Q5 — Does `raw["content"]` hold extracted text? And does sha256 dedup catch metadata-only differences?

**TL;DR:** `content` is raw bytes, not text. sha256 catches identical files but misses re-exports of the same CV by different software (different embedded PDF metadata = different bytes = different hash).

**`content` is raw bytes, not text.** Spark's `binaryFile` reader does no decoding — it `read()`s the file and dumps its on-disk bytes into a column of type `binary`. For a PDF, that means the literal `%PDF-1.x...` byte stream. Text extraction happens later, in `preprocess.py`, where `pypdf.PdfReader` parses those bytes.

**Whether two "same-content" files hash equal depends on what kind of metadata differs:**

- **File-system metadata** (path, modificationTime, filename) — NOT part of `content`, NOT hashed. Two files with identical bytes but different paths or upload times produce **identical sha256 hashes** and dedup correctly.
- **Embedded PDF metadata** (`/Author`, `/Title`, `/CreationDate`, `/ModDate`, `/Producer`, document ID in the trailer) — IS part of the file bytes, IS hashed. Two PDFs that render identical content but were exported by different software, or even just opened and re-saved, will have **different sha256 hashes** and will NOT be deduped.

**Practical consequence.** A candidate who exports the same CV twice (e.g. once from Word and once from LibreOffice) creates two byte-distinct PDFs that both end up in `cv_bronze`. Byte-level dedup misses them.

**If true content dedup is required**, shift it downstream — dedup on normalised extracted text in `cv_silver` (`silver.dropDuplicates(["text"])`). Heavier, but catches near-duplicates byte hashing can't see. For this dataset (curated Kaggle), byte-level dedup is sufficient; worth a sentence in the report acknowledging the limit.

---

<a id="q9"></a>

## Q9 — Why is a UDF needed in `preprocess.py`, and why are PySpark UDFs slow?

**TL;DR:** PySpark has no built-in PDF parser, so `pypdf` has to be wrapped in a UDF. UDFs are slow because of JVM↔Python serialisation per row, opacity to Catalyst, no Tungsten codegen, and per-worker GIL. They're also invisible to the optimiser — accidental re-execution post-write is easy and expensive (this project hit it).

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

1. **JVM ↔ Python serialisation on every row.** Spark runs on the JVM. Python UDFs execute in a separate Python worker process per task slot. Each row's input is serialised in the JVM, sent through a pipe, deserialised in Python, processed, re-serialised, sent back, deserialised in the JVM. That round-trip is fixed cost; for cheap functions it dwarfs the actual work.
2. **No Catalyst optimisation.** The query optimiser can't reason about Python code, so it can't push filters through the UDF, can't reorder, can't constant-fold, can't fuse it with other operations. The UDF is an opaque black box.
3. **No code generation (Tungsten).** Built-in Spark expressions compile to Java bytecode at query plan time. UDFs run as interpreted Python, with no codegen.
4. **GIL-bound per worker.** Each Python worker is a single-threaded process, so executor parallelism caps at "one row at a time per Python worker" — coarse compared to JVM expressions running on the executor's native thread pool.

**Mitigations** (not used here, but worth knowing):

- **Pandas UDFs** (`@pandas_udf`) batch rows into Arrow record batches before the JVM↔Python crossing, often 10–100× faster than row-at-a-time UDFs for cheap functions. For `pypdf` extraction the batch wouldn't help much because the work per row is heavy already.
- **Push the work into a SQL function.** Whenever the operation is expressible as built-in Spark functions or a SQL function, use those instead. This is exactly why classify uses `ai_query` (a SQL function, JVM-native) rather than a Python UDF wrapping the OpenAI/Anthropic SDK — the UDF version would single-thread on the driver or require a hand-rolled thread pool inside Python workers.

**Lesson learned the hard way in this project.** The first end-to-end run had `silver.write` taking ~7 min and a follow-up `silver.filter(...).count()` taking another ~6 min — Spark re-ran the entire `pypdf` UDF because the post-write count used the lazy `silver` DataFrame whose query plan still contained the UDF. The fix: rebind to `spark.table("cv_silver")` after writing, so post-write reads come from the materialised Delta table. That cut preprocess from 13m 21s → 7m 25s. UDFs are slow _and_ invisible to the optimiser, so accidental re-execution is easy and expensive.

---

# Classification & LLM

<a id="q1"></a>

## Q1 — Why is `ai_query(...)` passed as a SQL string in `classify.py`?

**TL;DR:** `ai_query` is a Databricks SQL-only function with no PySpark wrapper, so calling it from the DataFrame API requires `expr()`, `selectExpr()`, or `spark.sql()`. The string form is unavoidable; current shape is idiomatic Databricks with a single-quote import-time guard.

```python
response_col = when(
    col("_prompt").isNotNull(),
    expr(f"ai_query('{MODEL}', _prompt, responseFormat => '{RESPONSE_FORMAT}')"),
).otherwise(lit(None))
```

**What it does.** `response_col` defines the contents of a new `_response` column. For rows where `_prompt` is non-null (extraction-error rows have already been nulled out one column earlier in the same stage, by `prompt_col`), it calls Databricks' `ai_query` SQL function, which invokes a Foundation Model endpoint server-side and returns the JSON response. The `responseFormat` named arg pins a JSON schema so the model can only emit `{bracket, confidence}` with `bracket` constrained to the allowed enum. Error rows fall through to `lit(None)`, so no tokens are spent on them.

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

<a id="q18"></a>

## Q18 — Why `ai_query` over OpenAI / Anthropic / Bedrock directly?

**TL;DR:** Spark-native parallelism, no API key or secret scope, native structured-output support, and a single auth/billing surface across both classification and embedding. External providers would either single-thread on the driver or require a hand-rolled threadpool wrapped in a Python UDF.

**The mechanical advantage.** `ai_query()` is a Spark SQL function — when applied as a column expression, it parallelises across executors as part of the normal Catalyst plan. 2,484 LLM calls fan out across the cluster's task slots automatically. Calling OpenAI / Anthropic from PySpark means one of:

- **Driver-side `for` loop** — single-threaded, no benefit from cluster.
- **Python UDF wrapping `openai.chat.completions.create()`** — works, but each Python worker is GIL-bound and serial inside its task slot, plus you pay JVM↔Python serialisation on every row (see [Q9](#q9)).
- **Hand-rolled `concurrent.futures` threadpool inside a Python UDF** — works, but you're now writing infrastructure that `ai_query` gives you for free, and the threadpool size has to be tuned by hand against the provider's rate limits.

**Auth & ops surface.** External providers need:

- API key in a Databricks secret scope (provisioning, rotation, access control).
- Network egress configured if the workspace is locked down.
- Per-call cost tracking has to be built (no UC visibility into external API spend — see [Q19](#q19)).

`ai_query` skips all of that: it inherits Unity Catalog auth, cost shows up on the workspace bill, and Foundation Model API usage is visible in `system.serving.endpoint_usage`.

**Structured output.** Both classification *and* embedding use `ai_query` here, with the same parameter shape and the same `responseFormat` mechanism for JSON-schema constraint. Mixing in OpenAI for one and Databricks for the other would mean two SDKs, two auth surfaces, two failure modes.

**When you'd flip the decision.** If a specific external model is materially better than what's hosted on Foundation Model APIs (e.g. GPT-4o for a multimodal task that needs image understanding, or Claude for a long-context task above Llama's context window), the operational tax is worth paying. For experience-bracket classification on short text, Llama 3.3 70B is plenty.

---

<a id="q21"></a>

## Q21 — How would you measure LLM classification accuracy without ground truth?

**TL;DR:** A stratified hand-labelled sample is unavoidable — pick 50–100 CVs across the predicted brackets, compute confusion matrix, off-by-one rate, macro-F1. Layer in inter-annotator agreement on a subset to validate the labelling task itself. Self-rated `confidence` is a triage signal, not a calibrated metric.

**Step 1 — Hand-label a stratified sample.** Pick 50–100 CVs **stratified across the predicted brackets** (not a random sample, which would be 76% in the 10+ bracket and miss the small ones entirely). Hand-label each by reading the CV. Compute:

- **Confusion matrix** of predicted vs true bracket.
- **Off-by-one rate** — most likely error mode for ordinal labels (5-7 vs 7-10 bracket-boundary errors).
- **Per-bracket precision and recall**.
- **Macro-F1** — averages across brackets, less sensitive to the corpus skew than micro-F1.

**Step 2 — Inter-annotator agreement.** Have two people label the same 30 CVs independently and compute Cohen's κ. If κ < 0.6, the labelling task itself isn't well-defined — bracket boundaries need refinement before model accuracy can be measured. This is the check that the question "how many years of experience does this person have?" has a stable human answer.

**Step 3 — Use self-rated confidence as a triage signal, not a metric.** The model's `confidence` field correlates with accuracy *on average* but isn't calibrated — a 0.95 confidence doesn't mean 95% accuracy. What it's good for: flagging the lowest-confidence rows for human review (e.g. all rows with confidence < 0.7 get manual sign-off). One row in this run scored exactly 0 — that's a "must review" flag, not noise.

**Step 4 — Held-out gold set for ongoing evaluation.** Once calibrated:

- Set aside a fixed 100-CV gold set with hand-labelled brackets. Don't touch it for tuning.
- Re-run the gold set after every prompt or model change. Track the macro-F1 over time.
- When macro-F1 drifts, that's the signal to re-evaluate the prompt or check whether the model behind `databricks-meta-llama-3-3-70b-instruct` has been silently updated.

**Step 5 — Production-grade accuracy.** A real recruiter use case would also need:

- **Disaggregated error analysis** by demographic subgroup (e.g. gender or ethnicity inferred from name) to surface bias. GDPR-tricky but legally required in many jurisdictions for automated hiring decisions — see [Q20](#q20).
- **Drift detection** on the input distribution — if CV style changes (e.g. a wave of fresher graduates), accuracy may degrade silently.
- **Human-in-the-loop on borderline cases.** Confidence < threshold triggers manual review before the bracket is treated as authoritative.

**For the case study.** Listed in the report under "What would extend this with more time." The 50-resume sample is the unblocking step — without it, the headline 76% senior skew is plausible but unverified.

---

# Embedding & Vector Search

<a id="q12"></a>

## Q12 — Why Delta Sync Index with self-managed embeddings, and why no choice in the matter on Free Edition?

**TL;DR:** Free Edition forces Delta Sync (Direct Vector Access not supported). Within Delta Sync, self-managed embeddings keep vectors inspectable in SQL, lineage in the medallion story, and let chunking be tuned without re-embedding inside the index.

**Mosaic AI Vector Search has two index types:**

- **Delta Sync Index** — index points at a Unity Catalog Delta table. Databricks reads from the table and keeps the index synchronised with it. Sync mode is either *triggered* (manual / job-driven) or *continuous* (driven by the Delta change feed).
- **Direct Vector Access Index** — index is populated by REST/SDK calls; you push vectors into it. No Delta table involved.

**On Databricks Free Edition, Direct Vector Access is explicitly not supported.** Free Edition allows exactly one Vector Search endpoint, capped at one Vector Search unit, and the only available index type is Delta Sync. So the design choice is made for us — the source of truth has to be a Delta table, and the index syncs from it.

This happens to align with the medallion architecture already in place: `cv_silver_chunks` becomes the Delta source, the index references it, and there is no out-of-band data path to manage.

**Within Delta Sync, the second axis is who computes the embeddings:**

- **Managed embeddings** — point the index at a *text* column. Databricks calls a Foundation Model API embedding endpoint internally and stores the resulting vectors inside the index (you never see them).
- **Self-managed embeddings** — you compute the embeddings yourself into an `array<float>` column on the Delta table; the index just consumes that column.

**Self-managed was picked for four reasons:**

1. **Inspectable.** Vectors live in `cv_silver_chunks.embedding`, so similarity, dimensionality, and per-row cost are queryable in SQL. Managed embeddings are opaque inside the index.
2. **Lineage stays in the medallion story.** The embedding step is just another `ai_query()` call into a Delta column — same mechanism as classification, no hidden index-side embedder.
3. **Reproducibility.** Embeddings are tied to a specific model version. Migrating off Vector Search (or rebuilding the index) doesn't require re-embedding 12k rows.
4. **Experiment ergonomics.** Tweaking chunking parameters means recomputing one column, then resyncing the index. Managed embeddings would re-embed inside the index on every change.

**Sync mode: triggered, not continuous.** The pipeline is batch — CVs are processed in a single job run, not streamed in over time. Triggered sync is cheaper and fits the DAG: the chunk task writes the table, then explicitly calls `index.sync()`. Continuous sync is for use cases where the source table changes constantly (e.g. user-generated content) and the index needs to track those changes in near-real-time.

**Free Edition limits worth flagging:**

|                                       | Free Edition                                                       |
| ------------------------------------- | ------------------------------------------------------------------ |
| Vector Search endpoints per workspace | 1                                                                  |
| Vector Search units per endpoint      | 1                                                                  |
| Direct Vector Access indexes          | not supported                                                      |
| Foundation Model API mode             | pay-per-token only (no provisioned throughput, no GPU endpoints)   |
| Compute                               | serverless only                                                    |

For 2,484 CVs × ~5 chunks ≈ 12,400 vectors, one unit is well within capacity (a unit handles millions of vectors). The single-endpoint cap means dev and prod can't run side-by-side on the same workspace; for a case study that's a non-issue.

---

<a id="q13"></a>

## Q13 — What surprised you when running the chunk + index + retrieval surface end-to-end?

**TL;DR:** Three things — Vector Search cold provisioning is much slower than the docs suggest (~10 min endpoint, 30+ min index); `*_and_wait` SDK calls take the timeout out of your hands (use the non-waiting variant); always validate similarity surfaces with a contrastive query — score *deltas*, not absolutes, are the signal.

**Free Edition Vector Search provisioning is much slower than the docs suggest.** First-time endpoint creation took ~10 minutes; the Delta Sync index then sat in `Provisioning resources…` with `Pipeline id: -` for 30+ minutes before transitioning to `Online`. None of this is documented anywhere as a typical-case latency — Databricks' guidance reads as if endpoints come up in single-digit minutes. The first-time hit isn't repeated on subsequent runs (idempotent re-creation hits the "already exists" branch and finishes in seconds), but the first run on a cold workspace needs to budget for it.

**`create_delta_sync_index_and_wait` is the wrong abstraction when you want a deadline.** It's a synchronous SDK call that blocks until the index is fully provisioned. We added a 30-minute timeout to our polling loop expecting to bound the total wait — but the polling loop is on the *other* side of the `_and_wait` call, so the timeout never had a chance to fire. The fix is to switch to the non-waiting `create_delta_sync_index` and let the polling loop own the whole provisioning + sync window with one observable deadline. Generalisable principle: when an SDK offers `*_and_wait` and `*` variants, prefer the non-waiting one whenever you want explicit control over the wait.

**Score-format precision can hide whether retrieval is actually working.** Mosaic AI Vector Search returns scores from a distance-style metric, not the [0, 1] cosine convention. For our corpus the meaningful score range was `[0.0017, 0.0025]` — the `f"{score:.3f}"` print format I'd written rounded everything to `0.002`, making it look like every result had identical similarity. Two follow-up checks resolved the ambiguity: (a) bumping the formatter to `:.6f` revealed real variation in the 4th–6th decimal place, and (b) a contrastive nonsense query ("unicorns in a tree?") returned a *different* tight-but-lower band (`0.0017` vs `0.0024`), with totally different chunks. Tight score ranges on a homogeneous corpus aren't a bug; the formatter was. Generalisable principle: when validating a similarity-search surface, run a contrastive query early — the *delta* between an in-domain and out-of-domain query is much more informative than absolute score values.

---

<a id="q15"></a>

## Q15 — What actually is `cv_silver_chunks_index`, and where does Vector Search sit in the broader Mosaic AI product?

**TL;DR:** It's not a Delta table — it's a Mosaic AI Vector Search index, hosted on a Vector Search endpoint, with data in an off-cluster HNSW store synced from `cv_silver_chunks` via a managed Lakeflow pipeline. Mosaic AI is Databricks' umbrella brand (named after the MosaicML acquisition) for inference, retrieval, agent build, training, and governance.

`cv_silver_chunks_index` looks like a table in Catalog Explorer — it appears in Unity Catalog under `cv_classification_catalog.dev.*` alongside the Delta tables — but it isn't one. It's a **Mosaic AI Vector Search index**, a separate managed resource hosted on a Vector Search endpoint (`cv_search`). The data lives in an off-cluster HNSW (Hierarchical Navigable Small World) approximate-nearest-neighbour store managed by Databricks, not as Parquet files plus a `_delta_log/` in the project's storage account. A small managed Lakeflow pipeline runs behind the scenes to keep the index synchronised with its source Delta table — the `pipeline_id` field on the index detail page is that pipeline.

### Concrete differences from a normal Delta table

|                  | `cv_silver_chunks` (Delta)                  | `cv_silver_chunks_index` (Vector Search)                   |
| ---------------- | ------------------------------------------- | ---------------------------------------------------------- |
| Storage          | Parquet + Delta log in ADLS                 | Opaque HNSW store on a VS endpoint                         |
| Query path       | SQL, Spark, `spark.table(...)`              | `client.get_index(...).similarity_search(...)` over HTTP   |
| Index structure  | Delta data skipping, optional Z-order       | HNSW for ANN + scalar metadata filters                     |
| Updates          | Direct writes                               | Sync from a *source* Delta table only                      |
| Lifecycle        | Independent                                 | Tied to endpoint + source table                            |
| Compute to query | Any cluster / SQL warehouse                 | Only the endpoint serves queries                           |

### What is Mosaic AI

"Mosaic AI" is Databricks' umbrella brand for everything AI/ML on the platform. The name comes from **MosaicML**, a startup Databricks acquired in mid-2023 for ~$1.3B; that team's pretraining / fine-tuning stack was folded into the lakehouse and the wider AI surface was rebranded. The umbrella covers the full lifecycle:

| Layer            | Product                              | What it does                                                                          |
| ---------------- | ------------------------------------ | ------------------------------------------------------------------------------------- |
| Inference        | Foundation Model APIs                | Pay-per-token access to hosted Llama, DBRX, GTE-large, etc. — what `ai_query()` calls. |
|                  | Model Serving                        | Custom MLflow-registered models or external APIs proxied as Databricks endpoints.     |
|                  | AI Gateway                           | Routing, rate-limiting, PII redaction, unified billing in front of multiple providers. |
| Retrieval        | **Vector Search**                    | Managed vector DB integrated with Unity Catalog.                                      |
|                  | Feature Serving                      | Low-latency lookup of feature-store columns at inference time.                        |
| Build            | Agent Framework + Agent Evaluation   | Build and offline-score RAG / tool-using agents.                                      |
|                  | Playground                           | Interactive prompt/model testing in the workspace.                                    |
| Train            | Foundation Model Training            | Continued pretraining and fine-tuning on top of the MosaicML stack.                   |
| Govern / observe | MLflow 3 + Lakehouse Monitoring      | Lineage, eval logs, drift, cost tracking — all written to UC tables.                  |

The selling point isn't any one of these in isolation (Pinecone, Bedrock, OpenAI all overlap on individual capabilities) — it's that everything shares one auth surface (UC), one billing surface (DBUs), one storage surface (Delta in the customer's own object store), and one governance surface (UC permissions, lineage, audit).

### Vector Search architecture in two resources

- **Endpoint** — the compute layer. A serving cluster that hosts indexes and answers similarity queries. Billed in **Vector Search Units (VSUs)**; one unit handles tens of millions of vectors. Multiple indexes can share an endpoint. On Free Edition the cap is one endpoint × one unit.
- **Index** — the data layer. An HNSW graph plus metadata, attached to one endpoint. Lives in the UC namespace as `catalog.schema.index_name`.

The two index types (Delta Sync vs Direct Vector Access), the two embedding modes (managed vs self-managed), and the two sync modes (TRIGGERED vs CONTINUOUS) are covered in [Q12](#q12).

### Why it's tied to Delta + UC, not standalone

- **Permissions** — the index inherits read ACLs from its source table. Whoever can `SELECT` on `cv_silver_chunks` can query `cv_silver_chunks_index`.
- **Lineage** — appears in UC's lineage graph alongside the source.
- **Updates** — the source table must have `delta.enableChangeDataFeed=true` plus a single-column primary key, exactly so sync can be incremental rather than full-rescan (see [Q16](#q16)).
- **Canonical copy** — vectors physically live in the endpoint's HNSW store, but in self-managed mode the embeddings are also in the source Delta column. One canonical copy that can be rebuilt from.

### Where it sits competitively

Pinecone, Weaviate, Qdrant, pgvector, Azure AI Search all do similar work. Vector Search's distinguishing pitch is "the lakehouse never has to be left." Same auth, governance, billing, and SDK as the rest of the data platform. The trade is platform lock-in plus the Free Edition capacity caps surfaced in this case study (single endpoint, slow cold provisioning).

---

<a id="q16"></a>

## Q16 — What is Change Data Feed, and where does this project use it?

**TL;DR:** CDF records every row-level change between Delta versions. Mosaic AI Vector Search Delta Sync indexes use CDF as their sync mechanism — without it enabled on the source table, sync calls fail. In this project it's set on `cv_silver_chunks` once, on the write in `chunk.py`.

**Change Data Feed (CDF)** is a Delta Lake feature that records every row-level insert, update, and delete between table versions, on top of the table's normal history. When CDF is enabled, the change log can be read as a stream of rows with three extra columns:

| Column              | Meaning                                                            |
| ------------------- | ------------------------------------------------------------------ |
| `_change_type`      | `insert`, `update_preimage`, `update_postimage`, or `delete`       |
| `_commit_version`   | The Delta version that introduced the change                       |
| `_commit_timestamp` | When that commit happened                                          |

CDF is opt-in. It's enabled per-table by setting the table property `delta.enableChangeDataFeed=true` either at create time (`CREATE TABLE … TBLPROPERTIES (...)`), on a writer (`.option("delta.enableChangeDataFeed", "true")`), or via `ALTER TABLE … SET TBLPROPERTIES`. Once on, every subsequent commit has its row-level changes materialised; readers consume them via:

```python
spark.read.format("delta") \
     .option("readChangeFeed", "true") \
     .option("startingVersion", N) \
     .table("cv_silver_chunks")
```

### Where it appears in this pipeline

`chunk.py` enables CDF on `cv_silver_chunks` at write time:

```python
# chunk.py:98 — on the overwrite write
(
    exploded.write.mode("overwrite")
    .option("delta.enableChangeDataFeed", "true")
    .saveAsTable("cv_silver_chunks")
)
```

A separate first-run-only block sets a NOT NULL constraint and a primary key on `chunk_uid`, both of which Vector Search Delta Sync requires:

```python
# chunk.py:108-109 — first creation only
spark.sql("ALTER TABLE cv_silver_chunks ALTER COLUMN chunk_uid SET NOT NULL")
spark.sql("ALTER TABLE cv_silver_chunks ADD CONSTRAINT cv_silver_chunks_pk PRIMARY KEY (chunk_uid)")
```

These ALTER statements are gated on `not table_existed` because they're schema-mutating commits — running them every overwrite would either no-op (and clutter Delta history) or, if Delta semantics changed, risk breaking CDF compatibility. Constraints persist across `mode("overwrite")` data replacements in Delta.

The reason CDF is required is that **Mosaic AI Vector Search Delta Sync indexes use CDF as their sync mechanism.** When `idx.sync()` fires, the index doesn't rescan the entire source table — it reads the change feed since the last successful sync version, then upserts (or deletes) only the affected rows in the HNSW store, keyed by the primary key (`chunk_uid`). Without CDF on, the sync call fails with an explicit "source table must have Change Data Feed enabled" error.

### CDF vs sync mode — orthogonal concerns

A common confusion: `pipeline_type="TRIGGERED"` vs `"CONTINUOUS"` is about *when* the sync pipeline runs, while CDF is about *how* the sync pipeline computes the delta to apply. Both modes use CDF; the difference is whether sync is fired by an explicit `idx.sync()` call (TRIGGERED — what this project uses, fitting the batch DAG) or by a managed pipeline tailing the change feed in near-real-time (CONTINUOUS — for streaming workloads).

### Other uses of CDF outside this project

CDF generalises beyond Vector Search. Anywhere a downstream consumer needs incremental updates from a Delta table, CDF is the lakehouse-native mechanism: feeding a Kafka topic from a Delta table, materialised views in DBSQL, dbt incremental models targeting Delta, mirroring rows into an external OLTP system, or audit logs of every row change at production scale. It carries the same trade as any change-log feature: extra storage and write overhead in exchange for cheap, exact-incremental reads downstream. For a 12k-row Vector Search index the overhead is invisible; at production scale it should be enabled deliberately on the tables that need it, not blanket across the catalogue.

---

# Data quality, cost, security, productionisation

<a id="q10"></a>

## Q10 — Does this project need data contract validation?

**TL;DR:** No, but flag it in the report. Schemas are tiny and stable; `responseFormat`'s JSON-schema enum already enforces the most important invariant; lightweight `assert` statements would catch the genuine failure modes for this scale. At production scale, expectations / dbt tests at every layer become essential.

**What "data contract validation" means in this context.** Explicit, enforced assertions about table schemas and invariants at stage boundaries — typically with a tool like Great Expectations, Soda, dbt tests, or Databricks DLT expectations. E.g.:

- `cv_bronze.sha256` is non-null and unique
- `cv_silver.text_length > 0` for non-error rows
- `cv_gold.experience_bracket` ∈ {0-2, 3-5, 5-7, 7-10, 10+} or NULL
- `cv_silver.category` is non-null

**Why it isn't worth the ceremony here.**

- **Schemas are tiny and stable** — six columns in bronze, six in silver, eight in gold. The schema _is_ the contract; reading the code is faster than reading a tests file.
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

<a id="q19"></a>

## Q19 — What does this cost to run, and at production scale?

**TL;DR:** Free Edition is free; at production scale per-token Foundation Model API pricing dominates. Back-of-envelope at 100k CVs/month: low-hundreds of dollars on classification tokens + tens on embeddings + ~$150 for a single always-on Vector Search Unit + minor serverless DBUs. Verify against current published rates before quoting.

**This case study.** Zero direct cost — Databricks Free Edition includes pay-per-token Foundation Model API access without billing, capped at the Free Edition rate limits. The exercise was deliberately scoped to fit those caps.

**At 100k CVs/month (back-of-envelope; verify before quoting).** The dominant variable cost is the LLM:

- **Classification:** ~5,000 input tokens average per CV (10k-char prompt at ~3 chars/token), ~30 output tokens (`{"bracket": "...", "confidence": 0.xx}`). At Llama 3.3 70B's published rate, 100k CVs ≈ 500M input + 3M output tokens. The dominant line on the bill.
- **Embedding:** ~5 chunks × 400 tokens × 100k CVs ≈ 200M tokens. GTE-large-en is meaningfully cheaper per token than the chat model. Secondary line.
- **Vector Search Unit:** one VSU at a flat hourly rate × 730h/month for one always-on unit. One unit handles tens of millions of vectors, so this scales sub-linearly with corpus size.
- **Serverless DBUs:** the `pypdf` extraction is the dominant compute cost (~7 min preprocess at the case-study volume), but at 100k CVs it's a minor line item compared to LLM tokens.

**Production-grade cost monitoring.** Three layers I'd add:

1. **`system.billing.usage`** — Databricks-managed UC table that records per-workspace DBU consumption. Daily summary query in a dashboard.
2. **`system.serving.endpoint_usage`** — per-model-call breakdown for Foundation Model APIs. Lets you split classification vs embedding spend.
3. **Per-row tagging at ingest** — record `cost_category` and `tenant_id` if billing is per-customer; join against the model-usage table for per-tenant spend.

**Cheapest cost optimisations.**

- **Truncation cap** — already applied. Bounded prompt size = bounded token spend. Doubling the cap from 6k to 10k chars increased per-row classification cost by ~67%; raising further has diminishing accuracy returns.
- **CDF-aware re-runs** — only re-classify rows that changed. CDF makes this trivial (see [Q16](#q16)).
- **Aggressive smoke-test caps** — `--limit 50` before full runs catches prompt regressions cheaply (see [Q8](#q8)).
- **Smaller model for the easy cases** — a hybrid classifier where Llama 3.3 8B handles high-confidence cases and the 70B model is reserved for low-confidence retries would cut spend significantly. Worth a feasibility test against the gold set.

---

<a id="q20"></a>

## Q20 — How would you handle PII / GDPR for CV data?

**TL;DR:** CVs are personal data under GDPR — production deployment needs lawful basis, data minimisation (PII redaction before the LLM call), retention TTL, access control via Unity Catalog, and a DPA covering Foundation Model API usage. Article 22 means LLM-driven hiring screening needs human-in-the-loop sign-off.

**Lawful basis.** For recruitment use, typical bases are:

- **Consent** from the candidate at upload time — most defensible, but revocable.
- **Legitimate interest** for hiring purposes — lower friction, but needs a balancing test.
- **Contract** if the candidate has applied for a specific role.

This isn't a technical decision — it's a legal/policy one. The pipeline needs to support whichever basis the data controller picks, particularly **deletion on request** (consent withdrawal or GDPR Article 17).

**Data minimisation.** GDPR Article 5(1)(c) requires only the data needed for the stated purpose. For experience-bracket classification, names, addresses, phone numbers aren't needed. A production pipeline would:

- Run a **PII redaction pass** on the extracted text *before* it reaches the LLM — Microsoft Presidio is the standard open-source detector, integrable as another preprocess UDF.
- Store the redacted text in a separate column, route the LLM call to that, keep the unredacted text behind stricter access control.

**Access control.** Unity Catalog gives you table-level and (with row-level filters) row-level access. Production setup:

- Raw `cv_bronze` (containing personal data) — access restricted to a small data engineering group.
- Redacted `cv_silver_redacted` and downstream `cv_gold` — broader access for analytics consumers.
- Audit logging on table access via UC system tables.

**Retention policy.** GDPR Article 5(1)(e) — kept no longer than necessary. Concretely:

- TTL column at ingest (e.g. `expires_at = ingestion_date + 18 months`).
- Scheduled job that hard-deletes rows past TTL (Delta `DELETE FROM ... WHERE ...`, then `VACUUM` to remove the underlying files).
- The Vector Search index inherits — when source rows delete, CDF propagates the delete to the index on next sync.

**Sub-processor coverage.** Calling Foundation Model APIs sends candidate data to Databricks-hosted models. Need:

- A Data Processing Agreement (DPA) with Databricks covering Foundation Model API usage.
- Confirmation that data sent to `ai_query` doesn't leave the customer's tenant or get used for model training. Databricks' published terms cover this for Foundation Model APIs (worth re-confirming for the specific deployment).

**The model-bias angle (Article 22).** Automated decisions with legal/significant effects need human review. An "experience bracket" assigned by an LLM that feeds into hiring screening would qualify. Production deployment needs human-in-the-loop sign-off before any candidate is filtered out on LLM output alone — see [Q21](#q21) on disaggregated error analysis by demographic subgroup.

**For this case study specifically.** The Kaggle resume dataset is publicly available and consent-collected by the dataset publisher; using it for a technical exercise sits outside the production GDPR question. Worth flagging in the report that production use would need the above.

---

<a id="q22"></a>

## Q22 — What does the v2 production deployment look like?

**TL;DR:** Hybrid DLT (deterministic stack) + Jobs (LLM tier). Add retry-with-backoff and dead-letter on `ai_query`, per-row cost monitoring via UC system tables, shadow-mode comparison against a held-out gold set, schema contracts at every layer, OCR fallback on extraction errors, and PII redaction before the LLM call.

**Architecture shift: hybrid DLT + Jobs.** At production scale (100k+ CVs/month, daily incremental ingest):

- **DLT for ingest → preprocess → chunk.** Pure Delta-to-Delta with quality checks. `@dlt.expect_or_drop` on `text_length > 0`, on `category IS NOT NULL`, on `sha256` uniqueness. Auto Loader for streaming ingest from a landing zone. STREAMING tables for incremental appends.
- **Jobs for the LLM tier (classify, index sync, retrieval).** Imperative shape, easy iteration on prompts and model versions, separate from the deterministic stack. See [Q17](#q17) for why this split matches the strengths of each tool.
- **Vector Search index sync** stays triggered, fired by a downstream Job after each chunk batch lands.

**Reliability — retry & dead-letter.** `ai_query` failures (429, 500, transient network) are not handled in the case study. v2:

- Wrap classify in a retry-with-backoff loop at the row level — `_response` initially null, retry pipeline catches nulls and re-fires.
- After N retries, route to a `cv_classify_dlq` table with the failure reason.
- DLQ is monitored daily; persistent failures trigger an alert.

**Cost monitoring** (see [Q19](#q19) for the full breakdown). Three additions:

- **Per-row cost tagging.** Record approximate input/output tokens per row. Multiply by published per-token rates.
- **Daily cost dashboard** built on `system.billing.usage` and `system.serving.endpoint_usage`.
- **Budget alerts** — Databricks Workspace-level budget config + Slack webhook on threshold breach.

**Shadow-mode comparison.** When swapping the classifier model (e.g. Llama 3.3 → Llama 4.x):

- Run both models on the same input batch, write to `cv_gold_shadow`.
- Compare on the gold set (see [Q21](#q21)) for accuracy regression.
- Promote new model only if macro-F1 ≥ current model on the gold set, AND manual review of disagreements doesn't surface new failure modes.

**Schema contracts** (see [Q10](#q10)). Replace lightweight asserts with `@dlt.expect` on the DLT tables and dbt tests on the Jobs-managed tables:

- `cv_bronze.sha256` non-null and unique
- `cv_silver.text_length BETWEEN 50 AND 100000` (catches extraction-error and mega-resume edge cases)
- `cv_gold.experience_bracket IN ('0-2', '3-5', '5-7', '7-10', '10+')`
- `cv_silver_chunks.embedding` array length == 1024 (catches embedding model drift)

**Extraction reliability.** ~1% of resumes hit `__EXTRACTION_ERROR__` from `pypdf` failures. v2:

- OCR fallback via Tesseract or Azure Document Intelligence when `pypdf` returns empty or garbage.
- Track extraction quality per CV: `extraction_method` ∈ {`pypdf`, `tesseract`, `azure_di`} for downstream slicing.

**Compliance** (see [Q20](#q20)). Becomes a hard requirement — Presidio redaction before LLM calls, retention TTLs, UC access control on raw vs redacted tables, human-in-the-loop sign-off for any candidate filtered on LLM output.

**Recruiter-facing surface.** A simple UI or API:

- Filter by `experience_bracket` and `category`.
- Free-text search via Vector Search retrieval.
- "Show similar CVs to this one" via the embedding column.

**Observability.** Add Lakehouse Monitoring on `cv_gold` for drift on `experience_bracket` distribution, and on `cv_silver` for `text_length` distribution. Either signals upstream changes that warrant investigation.
