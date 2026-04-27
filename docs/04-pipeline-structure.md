# Pipeline structure

The case study has three stages: ingest → preprocess → classify. Each is a separate Python entry point in the wheel, executed as a chained task in a single Databricks Job.

## Bronze / silver / gold layout

Conventional medallion-architecture naming:

| Layer | Table | Contents | Stage |
|---|---|---|---|
| Bronze | `cv_bronze` | Raw bytes + path + sha256 | ingest |
| Silver | `cv_silver` | Extracted text, normalized, deduped | preprocess |
| Gold | `cv_gold` | Silver + experience bracket + LLM confidence | classify |

All tables in `cv_classification_catalog.dev.*`.

## Task DAG

```
   ingest  ─►  preprocess  ─►  classify
   (bronze)    (silver)         (gold)
```

Defined in `resources/cv_pipeline.job.yml` with `depends_on` between tasks. Each task calls a `python_wheel_task` and passes `--catalog` / `--schema` parameters.

## Entry points

`pyproject.toml`:

```
[project.scripts]
ingest = "cv_classification.ingest:main"
preprocess = "cv_classification.preprocess:main"
classify = "cv_classification.classify:main"
```

Each module is a thin shell:
1. Parse `--catalog` and `--schema` (via `common.parse_args`)
2. `USE CATALOG ...; USE SCHEMA ...` (via `common.use_namespace`)
3. Read the upstream table, transform, write the downstream table

## Why three separate tasks (not one)

- **Failure isolation** — a flaky LLM in classify shouldn't force re-ingestion
- **Observability** — separate task logs, separate timing
- **Reprocessing** — easy to re-run just classify when the LLM prompt changes
- **Caching** — Databricks caches table reads between tasks for free

## Why not DLT?

Considered, rejected. See [`02-databricks-bundles.md`](./02-databricks-bundles.md). Short version: LLM step doesn't fit DLT's declarative model, and the dataset is small enough that streaming/incremental aren't valuable.

## Current state

| Stage | Implementation | Output count |
|---|---|---|
| Ingest | Real — recursive `binaryFile` read of PDFs from `abfss://kaggle-cv-dataset@kagglecvdataset.dfs.core.windows.net/`, sha256 dedup, write as Delta | 2,484 unique rows |
| Preprocess | Real — `pypdf`-backed UDF extracts text page by page, whitespace normalized, error-flagged on extraction failure | 2,484 rows, 0 errors |
| Classify | Real — `ai_query()` against `databricks-meta-llama-3-3-70b-instruct` with structured JSON output (bracket + confidence) | 2,484 rows |

### Bronze schema (`cv_bronze`)

| Column | Type | Source |
|---|---|---|
| `path` | string | `binaryFile` reader |
| `modificationTime` | timestamp | `binaryFile` reader |
| `length` | long | `binaryFile` reader |
| `content` | binary | `binaryFile` reader |
| `sha256` | string | computed via `sha2(content, 256)` |

### Silver schema (`cv_silver`)

| Column | Type | Source |
|---|---|---|
| `sha256` | string | from bronze |
| `path` | string | from bronze |
| `text` | string | extracted text, whitespace-normalized |
| `num_pages` | int | from `pypdf` |
| `text_length` | int | computed |

Failed extractions are written with `text` starting `__EXTRACTION_ERROR__:` so they can be filtered downstream.

### Gold schema (`cv_gold`)

Silver columns plus:

| Column | Type | Source |
|---|---|---|
| `experience_bracket` | string | LLM choice from `[0-2, 3-5, 5-7, 7-10, 10+]`; NULL for extraction-error rows |
| `llm_confidence` | double | model self-rated confidence in `[0, 1]`; NULL for extraction-error rows |

## Classification details

- **Model:** `databricks-meta-llama-3-3-70b-instruct` (Foundation Model API, pay-per-token, no API key)
- **Invocation:** `ai_query()` SQL function called from PySpark via `expr()`
- **Structured output:** the `responseFormat` argument pins a JSON schema with an `enum` of valid brackets, so the model cannot return a free-form label
- **Prompt:** asks for years of *paid* working experience (excluding education and projects unless no work history is present); the resume text is truncated to 6,000 chars to bound input cost
- **Error rows:** rows whose `text` starts with `__EXTRACTION_ERROR__` are not sent to the LLM; their `experience_bracket` and `llm_confidence` are NULL
- **JSON parsing:** `get_json_object` extracts the two fields, so no Python UDF is needed in the LLM path
