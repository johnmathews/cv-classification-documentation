# Category column + interview Q&A doc — 2026-04-27

A documentation-heavy session capping the case study with two pieces of work: capturing the
profession dimension end-to-end through the medallion pipeline, and starting an interview-prep
Q&A document covering every non-obvious decision in the codebase.

## Category column

The Kaggle dataset groups PDFs into one folder per profession (`accountant/`, `arts/`,
`information-technology/`, …). That dimension was being thrown away at ingest because
`recursiveFileLookup=true` collapses the directory tree into the `path` column rather than
auto-promoting each level to a Hive-style partition column.

`ingest.py` now parses the parent-directory name out of `path` and stores it as a lowercase
`category` string column on `cv_bronze`:

```python
.withColumn("category", lower(element_at(split(raw["path"], "/"), -2)))
```

`preprocess.py` carries it through to `cv_silver` (added as the first column in the silver
`select`); from silver onwards `classify.py` propagates everything automatically because it
just adds two columns to whatever silver gives it. End result: 24 lowercase categories on
`cv_gold`, distribution mostly flat at ~100–120 rows each with three smaller folders
(`bpo` 22, `automobile` 36, `agriculture` 63), totalling 2,484 — exactly matching the bronze
row count, so no rows were lost in transit.

A `regexp_extract` alternative was considered (would fail loud with NULL on path-shape
change rather than silently picking the wrong segment) but `element_at(split(...), -2)` was
chosen for readability, given the dataset is curated and stable.

The schema tables in `docs/04-pipeline-structure.md` were updated to include `category` in
both bronze and silver, and the report's Methodology and Key Findings sections were
extended to mention the new dimension as a structural fact (not a data-analytical claim —
this is an engineering interview, not a business analytics one).

## Single-quote guard in classify.py

A second small hardening: `RESPONSE_FORMAT` is a Python string literal containing the JSON
schema spec, and it is interpolated directly into a SQL string via
`expr(f"ai_query('{MODEL}', _prompt, responseFormat => '{RESPONSE_FORMAT}')")`. The current
JSON contains no single quotes so wrapping it in `'...'` parses cleanly in SQL. If a future
edit adds a description with an apostrophe — say in a JSON-schema `description` field — the
SQL would silently break. A module-level `assert "'" not in RESPONSE_FORMAT` was added to
fail fast at import time rather than at LLM-call time.

## Interview Q&A document

A new `interview-qa.md` at the case-study root captures questions and answers about
the codebase for interview prep. Eleven entries so far:

1. Why `ai_query(...)` is passed as a SQL string (it's a SQL function with no Python wrapper)
2. Why some imports live inside `main()` rather than module-top (Databricks-runtime-only deps)
3. How `category` is extracted from the path
4. What the `binaryFile` reader chain does
5. Whether `raw["content"]` holds text or bytes, and how sha256 dedup behaves with PDF-embedded metadata
6. How to run a single stage — the user's CLI uses `--only`, not `--task`
7. Why `uv build` does not need to be run manually before `databricks bundle deploy` (the `artifacts` block in `databricks.yml` declares the build command)
8. How `--limit 50` is wired CLI → argparse → bundle parameter → task substitution → `silver.limit(...)`
9. Why a UDF is needed in preprocess (no Spark-built-in PDF reader; `pypdf` is Python-only) and why Python UDFs are slow (JVM↔Python serialization, no Catalyst optimization, no codegen, GIL-bound workers)
10. Whether data-contract validation is needed for this project (no — schemas are tiny and stable, `ai_query`'s structured output already enforces the most important invariant; light `assert` statements would be the right scale at production)
11. What `parse_args(...)` does and how `sys.argv` arrives differently locally vs on Databricks (job-parameter substitution into the wheel-task `parameters:` array)

The doc is intended as a running list — future sessions can append.

## Memory updates

A new feedback memory was saved: `feedback_report_focus.md` — this case study is judged on
data-engineering proficiency, not business analytical skill, so report findings should stay
structural (numbers and shapes) rather than interpretive (per-category × per-bracket
breakdowns and the like). Came up explicitly when proposing a per-profession senior-skew
analysis, which the user declined.

`CLAUDE.md` was also extended with a "Repo layout" section, useful commands for the
Databricks bundle, and the medallion convention — pieces that were not in the existing
project memory and would otherwise have to be re-discovered by future sessions.

## Verification commands

```sql
SELECT category, COUNT(*) AS n
FROM cv_classification_catalog.dev.cv_gold
GROUP BY category ORDER BY n DESC;
-- 24 categories, 2,484 rows total, lowercase, no NULLs
```
