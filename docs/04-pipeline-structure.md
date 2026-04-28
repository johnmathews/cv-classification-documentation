# Pipeline structure

The case study has five stages: ingest → preprocess → chunk → index → classify. Each is a separate Python entry point in the wheel, executed as a task in a single Databricks Job.

## Bronze / silver / gold layout

Conventional medallion-architecture naming:

| Layer | Table | Contents | Stage |
|---|---|---|---|
| Bronze | `cv_bronze` | Raw bytes + path + sha256 | ingest |
| Silver | `cv_silver` | Extracted text, normalized, deduped | preprocess |
| Silver-chunks | `cv_silver_chunks` | One row per chunk, with `chunk_uid`, `chunk_text`, char offsets, and 1024-dim `embedding` | chunk |
| (no table) | Vector Search index `cv_silver_chunks_index` | Delta Sync index over `cv_silver_chunks.embedding` | index |
| Gold | `cv_gold` | Silver + experience bracket + LLM confidence | classify |

All tables in `cv_classification_catalog.dev.*`.

## Task DAG

```
   ingest ─► preprocess ─┬─► chunk ─► index
   (bronze)   (silver)   │    (silver_chunks + VS index)
                         └─► classify
                              (gold)
```

`chunk` and `classify` are independent — both read `cv_silver`. Defined in `resources/cv_pipeline.job.yml` with `depends_on` between tasks. Each task calls a `python_wheel_task` and passes `--catalog` / `--schema` parameters.

## Entry points

`pyproject.toml`:

```
[project.scripts]
ingest = "cv_classification.ingest:main"
preprocess = "cv_classification.preprocess:main"
chunk = "cv_classification.chunk:main"
index = "cv_classification.vector_index:main"
retrieve = "cv_classification.retrieve:main"   # demo, not part of DAG
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
| Chunk | Real — LangChain `RecursiveCharacterTextSplitter` (1,600/200), explode to one row per chunk, then `ai_query('databricks-gte-large-en')` populates the 1024-dim embedding column. CDF + PK constraint set so the index can sync. | ~12k chunks (≈5 per CV) |
| Index | Real — creates Vector Search endpoint `cv_search` (if missing) and Delta Sync Index `cv_silver_chunks_index` (self-managed embeddings, triggered sync), then triggers `index.sync()`. | n/a |
| Classify | Real — `ai_query()` against `databricks-meta-llama-3-3-70b-instruct` with structured JSON output (bracket + confidence) | 2,484 rows |

### Bronze schema (`cv_bronze`)

| Column | Type | Source |
|---|---|---|
| `path` | string | `binaryFile` reader |
| `modificationTime` | timestamp | `binaryFile` reader |
| `length` | long | `binaryFile` reader |
| `content` | binary | `binaryFile` reader |
| `sha256` | string | computed via `sha2(content, 256)` |
| `category` | string | parent-directory name parsed from `path` (e.g. `accountant`, `arts`); lowercase |

### Silver schema (`cv_silver`)

| Column | Type | Source |
|---|---|---|
| `category` | string | from bronze |
| `sha256` | string | from bronze |
| `path` | string | from bronze |
| `text` | string | extracted text, whitespace-normalized |
| `num_pages` | int | from `pypdf` |
| `text_length` | int | computed |

Failed extractions are written with `text` starting `__EXTRACTION_ERROR__:` so they can be filtered downstream.

### Silver-chunks schema (`cv_silver_chunks`)

| Column | Type | Source |
|---|---|---|
| `sha256` | string | from silver |
| `category` | string | from silver |
| `chunk_id` | int | sequential within a `sha256`, assigned by the splitter |
| `chunk_text` | string | output of `RecursiveCharacterTextSplitter` |
| `char_start` | int | offset of the chunk in the parent `text` |
| `char_end` | int | offset (exclusive) of the chunk in the parent `text` |
| `chunk_uid` | string | `sha256 || '_' || chunk_id` — primary-key column for Vector Search |
| `embedding` | array<float> | 1024-dim, `databricks-gte-large-en` via `ai_query` |

Table properties: `delta.enableChangeDataFeed = true` and a primary-key constraint on `chunk_uid`. Both are required for the Delta Sync Index to consume the table.

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
- **Prompt:** asks for years of *paid* working experience (excluding education and projects unless no work history is present); the resume text is truncated to 10,000 chars to bound input cost
- **Error rows:** rows whose `text` starts with `__EXTRACTION_ERROR__` are not sent to the LLM; their `experience_bracket` and `llm_confidence` are NULL
- **JSON parsing:** `get_json_object` extracts the two fields, so no Python UDF is needed in the LLM path

## Chunking + embedding details

- **Splitter:** LangChain `RecursiveCharacterTextSplitter`, `chunk_size=1600`, `chunk_overlap=200`, separators `["\n\n", "\n", ". ", " ", ""]`. Sized for short, structured documents — large enough to keep a job entry intact, small enough for embeddings to remain specific.
- **Char offsets:** preserved (`char_start`, `char_end`) so a chunk can be located back in the parent `cv_silver.text` for citation / highlight UX.
- **Embedding model:** `databricks-gte-large-en` (Alibaba GTE-large-en-v1.5; 434M params; 1024-dim; 8192-token context). English-only.
- **Error rows skipped:** the chunk stage filters out `__EXTRACTION_ERROR__` rows before splitting, so no nonsense embeddings end up in the index.

## Vector Search details

- **Endpoint:** one endpoint named `cv_search`, `STANDARD` type, single Vector Search unit (Free Edition cap).
- **Index:** Delta Sync Index `cv_silver_chunks_index` over `cv_silver_chunks`, `pipeline_type=TRIGGERED`. Self-managed embeddings: the `embedding` column is consumed directly, no index-side embedder is configured.
- **Why Delta Sync (forced):** Direct Vector Access indexes are not supported on Free Edition — Delta Sync is the only legal path.
- **Sync:** the `index` task calls `index.sync()` and polls `describe()` until `status.ready` is true.
- **Query:** `databricks.vector_search.client.VectorSearchClient` → `index.similarity_search(query_vector=…, columns=[…], num_results=…)`. The `retrieve` entry point demonstrates this with four canned queries.
