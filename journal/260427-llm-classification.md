# LLM classification — 2026-04-27

The classify stage was wired up to a real LLM, replacing the constant-`"0-2"` stub.

## What was built

`classify.py` now:

1. Reads `cv_silver`.
2. Builds a prompt by concatenating a fixed instruction prefix with the first 6,000 chars of resume text.
3. Skips rows whose text starts with `__EXTRACTION_ERROR__:` (no point asking the LLM about a corrupt PDF).
4. Calls `ai_query('databricks-meta-llama-3-3-70b-instruct', _prompt, responseFormat => ...)` via `expr()`.
5. The `responseFormat` argument pins a JSON schema with an `enum` of the five legal brackets, guaranteeing the model returns one of them.
6. Extracts `bracket` and `confidence` via `get_json_object`.
7. Writes everything to `cv_gold` (silver columns + `experience_bracket` + `llm_confidence`).

## Why `ai_query` and not an external provider

- **No API key, no secret scope.** Foundation Model APIs are billed per-token through the workspace, and `ai_query` is a built-in SQL function — nothing to install, no `requests`, no retry loop to write.
- **Spark-native batching.** `ai_query` parallelizes the calls across executors automatically; calling OpenAI/Anthropic from a Python UDF would mean either single-threaded driver calls or DIY thread pools.
- **Structured output is supported.** The `responseFormat` parameter accepts a JSON-schema spec and constrains the model's reply, which removes the need for a fragile JSON-extraction regex.

Llama 3.3 70B was the obvious pick over the 405B variant — same quality bracket for short-context classification, much cheaper, much faster.

## Why structured output rather than `ai_classify`

`ai_classify` works on text labels, but the input would be the resume text and it would pick whichever of `["0-2", "3-5", ...]` the embedding nearest-neighbours to. That throws away the model's reasoning ability over years-of-experience phrasing. A free-form LLM call with an enum-constrained JSON output is more honest: the model thinks, then commits to one of the five labels.

The schema also includes `confidence` — useful as a soft signal for "the resume was thin / contradictory" without needing to look at log-probs.

## What is testable in pure Python

Only two things, both trivial: the prompt builder (truncation logic) and the JSON-schema spec (constants well-formed). The Spark / `ai_query` path can't be exercised without a workspace, so the test suite stays small (3 new tests, 7 total).

## A `--limit` smoke-test knob

`ai_query` is pay-per-token, so a first run that touches all 2,484 rows is risky for both cost and latency. A `--limit` CLI flag (default `0` = no cap) was added to `classify.py`, plumbed through a new `limit` job-level parameter in `cv_pipeline.job.yml`, and surfaced via the `{{job.parameters.limit}}` substitution syntax. That way the smoke-test is one CLI flag away without redeploying:

```sh
databricks bundle run cv_pipeline -t dev --params limit=50
```

Catalog/schema were switched to the same job-parameter indirection at the same time for consistency — previously they were substituted at deploy time only.

## Progress logging and a UDF observability lesson

The first end-to-end run on the cluster surfaced two related problems:

1. **No visible progress.** The original scripts called `print()` exactly once per stage, after every Spark action had completed. From outside, a 13-minute `preprocess` looked indistinguishable from a hung job. A small `log()` helper was added to `common.py` (timestamp + `flush=True`) and used at every meaningful checkpoint — start, before each Spark action, after each write — so the Workflows UI Output tab now updates in near-real-time.
2. **Hidden double work.** The cancelled run's task timeline showed `silver.write` taking ~7 min and a follow-up `silver.filter(...).count()` taking another **~6 min**. The cause: the original code re-used the lazy `silver` DataFrame (which still contained the `pypdf` UDF in its query plan) for the post-write counts, so Spark re-ran the entire UDF instead of reading from the materialized Delta table. Fixed by introducing `saved = spark.table("cv_silver")` and `saved = spark.table("cv_gold")` for any post-write counting in both `preprocess.py` and `classify.py`. The next run dropped from 13m 21s to **7m 25s** preprocess.

This drove a side discussion about why Python UDFs are expensive (JVM↔Python serialization, no Catalyst optimization, no codegen, opaque to the optimizer) and confirmed we want `ai_query` (a SQL function, JVM-native) rather than a Python-side LLM call wrapped in a UDF. The `pypdf` UDF in preprocess is the only unavoidable one in the pipeline and is now only run once per row.

## End-to-end smoke run timings

After the fixes, with `--params limit=50`:

| Stage | Wall time | Notes |
|---|---|---|
| ingest | 1m 9s | 2,484 PDFs scanned + sha256 dedup |
| preprocess | 7m 25s | pypdf UDF on 2,484 rows, no second pass |
| classify | 53.8s | 50 LLM calls via `ai_query`; `gold.write` itself is 17.5s |

Linear extrapolation suggests a full classify pass costs ~14 min; with `ai_query`'s internal batching, more like 10–15 min, so a full pipeline run is ~20–25 min wall time.

## What is not done

- The actual classify run on the full 2,484-row dataset has not been executed.
- The report (25% of the deliverable per the brief) has not been started.
