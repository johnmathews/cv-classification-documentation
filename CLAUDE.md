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
