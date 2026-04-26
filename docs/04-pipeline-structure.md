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

## Current state: stubs

The first deploy proved the loop works end-to-end (deploy → wheel → cluster → tables). Real logic for each stage is the next pass:

- **Ingest** — read PDFs from `abfss://kaggle-cv-dataset@kagglecvdataset.dfs.core.windows.net/`, dedupe by sha256, store as Delta with binary content
- **Preprocess** — extract text per-PDF (PyMuPDF or pypdf), strip headers/footers, normalize whitespace
- **Classify** — call an LLM per row, parse to one of `[0-2, 3-5, 5-7, 7-10, 10+]`, capture confidence
