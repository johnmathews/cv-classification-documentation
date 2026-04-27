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

## What is not done

- The actual classify run on the full 2,484-row dataset has not been executed; ai_query rate limits or token budgets may force a `LIMIT` for first runs.
- The report (25% of the deliverable per the brief) has not been started.
