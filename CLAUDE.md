# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Data engineering case study for Simmons & Simmons (disguised as "Lawyers and Partners"). The task is to build a CV/resume ingestion, preprocessing, and classification pipeline.

The full brief is in `case-study.md`.

## Pipeline Stages

1. **Ingestion** — Load the Kaggle resume dataset, deduplicate
2. **Preprocessing** — Clean and prepare documents for embedding/retrieval/analysis
3. **Classification** — Use an LLM to classify each resume into experience brackets: [0-2, 3-5, 5-7, 7-10, 10+] years

## Tech Requirements

- **PySpark (Databricks)** for ingestion and preprocessing (scalability requirement)
- **LLM** for classification (add-on analytics, not production)
- **Python 3.13**, **uv** for dependency management
- **pytest** for tests

## Data Source

Kaggle resume dataset: https://www.kaggle.com/datasets/snehaanbhawal/resume-dataset/data

## Deliverables

- Clean, modular code (75% of effort)
- Brief report covering methodology, findings, challenges (25% of effort)
- Private GitHub repo with Aleksandrs.Krivickis@simmons-simmons.com as collaborator

## Repo layout

Two repos. The outer `case-study/` (this directory) holds the brief, docs, journal, and `report.md`. The inner `cv_classification/` is a separate git repo containing the actual Databricks Asset Bundle project — `pyproject.toml`, `uv.lock`, `databricks.yml`, `src/`, `tests/`, `resources/`.

- `docs/` — architecture and design notes (Azure storage, bundles, Unity Catalog, pipeline structure). Start with `docs/04-pipeline-structure.md` for the medallion layout and current pipeline state.
- `journal/` — dated decision log (`yymmdd-name.md`).
- `report.md` — deliverable report.

## Useful commands

Run from inside `cv_classification/`:

- `uv sync --dev` — install deps
- `uv run pytest` — run tests
- `databricks bundle deploy --target dev` — deploy bundle
- `databricks bundle run` — run the pipeline job

## Conventions

- Medallion tables: `cv_bronze` → `cv_silver` → `cv_silver_chunks` → `cv_gold` in `cv_classification_catalog.dev.*`. Vector Search index `cv_silver_chunks_index` is layered on top of `cv_silver_chunks`.
- Pipeline entry points (`ingest`, `preprocess`, `chunk`, `index`, `classify`) are wired as console scripts in `cv_classification/pyproject.toml` and chained as tasks in `resources/cv_pipeline.job.yml`. `chunk` and `classify` both fan out from `preprocess` and run in parallel; `index` runs after `chunk`. A separate `retrieve` entry point is a hand-runnable retrieval demo and is not part of the DAG.
- Each stage parses `--catalog` / `--schema` via `common.parse_args`, then reads upstream / writes downstream.
