# 2026-04-27 — real pipeline + external storage access

Continuation of yesterday's session. Got past Free Edition's access-connector quirks and replaced two of three pipeline stubs with real Spark logic.

## What was built

### External storage access
- New Access Connector `cv-access-connector` created in unlocked `simmons-demo` resource group (the auto-created one in the locked `databricks-rg-*` is unusable for custom credentials)
- Storage Blob Data Contributor on `kagglecvdataset` granted to the new connector's managed identity
- Contributor role on the connector granted to `jonnosgone@gmail.com` (required for `databricks storage-credentials create`)
- Storage Credential `cv_credential` and External Location `cv_data` created via CLI/SQL
- `LIST` and `binaryFile` reads now succeed against `abfss://kaggle-cv-dataset@kagglecvdataset.dfs.core.windows.net/`

### Pipeline stages

- **Ingest (real):** recursive `spark.read.format("binaryFile").option("recursiveFileLookup", "true").option("pathGlobFilter", "*.pdf")` against the External Location. Computes `sha256(content)` and dedupes on it. End-to-end run: 2,484 PDFs, 0 duplicates.
- **Preprocess (real):** Python UDF wrapping `pypdf.PdfReader` extracts text per page, writes `(sha256, path, text, num_pages, text_length)` to `cv_silver`. Whitespace normalized via `regexp_replace(text, r"\s+", " ")`. Errors are flagged with a sentinel `__EXTRACTION_ERROR__:` prefix rather than silently dropped. End-to-end: 2,484 rows, 0 errors.
- **Classify (still stub):** every row gets `experience_bracket = "0-2"`. LLM integration is the next session.

### Schema migration
- Added `.option("overwriteSchema", "true")` to all three writers because the original stub schemas (`(path, size_bytes)` etc.) clashed with the real schemas. Necessary because Table ACLs prevent automatic schema merging on serverless.

### Tests + tooling
- `tests/test_preprocess.py` covers the `_extract` error path with corrupt and empty bytes.
- `coverage` added to dev deps; HTML report generated. Current coverage 67% (only pure-Python helpers reachable without Databricks Connect).
- `ruff format` applied to one file (line-length on the `print` statement in `ingest.py`).

### Hygiene
- Inner `.gitignore` extended with `.coverage`, `htmlcov/`, `.env`, `*.key`, `*.pem`, `credentials.json`, `*.secret`, `.DS_Store`.

## Decisions

- **Two-repo layout (outer + inner) kept.** The bundle directory is its own git repo from `bundle init`. Merging it into the outer repo would require destroying the inner history. For now: outer (`case-study/`) tracks docs/journal/brief; inner (`cv_classification/`) tracks the bundle.
- **Error sentinel over null on extraction failure.** Easier to filter and audit than `NULL` text values.
- **UDF over `pandas_udf`.** `pypdf` is fast enough on 25 KB files that batched UDFs aren't worth the complexity.

## Things that broke and why

- **`CREATE STORAGE CREDENTIAL` SQL parse error.** Serverless SQL warehouses don't support that DDL. Used `databricks storage-credentials create` CLI command instead.
- **`PERMISSION_DENIED: workspace default credential`.** Free Edition's auto-created `simmons_demo` credential is path-locked to Databricks-managed storage. Custom Access Connector required.
- **`Registering a storage credential requires the contributor role`.** The "right" connector wasn't being used (the new one in `simmons-demo` RG). The `unity-catalog-access-connector` in the locked RG is unusable due to deny assignment.
- **`DELTA_METADATA_MISMATCH`.** Real ingest tried to write a 5-column dataframe over a 2-column stub table. Fixed with `overwriteSchema=true`. Same fix needed in preprocess + classify.

## What's next

- Replace `classify.py` stub with a Databricks Foundation Model call (`ai_query()` SQL function or python wrapper). Map output to one of `[0-2, 3-5, 5-7, 7-10, 10+]`. Capture confidence.
- Sanity-spot-check `cv_silver` text quality (some PDFs may have unparseable text even if pypdf doesn't error).
- Write the report deliverable (25% of effort).
