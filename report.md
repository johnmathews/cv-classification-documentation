# Document Data Engineering Case Study — Report

## Methodology

The pipeline is structured into three stages — ingest, preprocess, classify — chained as dependent tasks in a single
Databricks Job. Each stage is a separate Python entry point in one wheel artefact, deployed via Databricks Asset Bundles.
Stage outputs follow medallion-architecture naming: `cv_bronze` (raw bytes plus a content hash), `cv_silver` (extracted
text), `cv_gold` (silver plus experience bracket and LLM confidence).

The Kaggle resume dataset was uploaded to Azure Blob storage and surfaced to Databricks through a Unity Catalog external
location. Ingestion uses Spark's `binaryFile` reader for recursive PDF discovery, then `sha2(content, 256)` for
content-based deduplication — 2,484 unique PDFs remained, with no duplicates dropped on this run. Preprocessing uses a
`pypdf`-backed Python UDF for text extraction, with whitespace normalisation applied via `regexp_replace`. Classification
calls the Databricks Foundation Model API via `ai_query('databricks-meta-llama-3-3-70b-instruct', …)` with a JSON-schema
`responseFormat` whose `bracket` field is enum-constrained to the five required values, guaranteeing the model can only
return a legal label. A `confidence` field in the same schema is used as a soft signal for thin or contradictory resumes.

A `--limit` flag was added to the classify stage and surfaced as a job-level parameter, enabling cheap smoke tests (50
rows) before committing to the full pay-per-token run.

## Architecture decisions

- **Three tasks, not one.** Failure isolation, separate task logs, and free Delta caching between tasks outweigh the
  modest orchestration overhead. Re-running classify after a prompt change costs nothing upstream.
- **Bundles, not DLT.** DLT's declarative model fits the LLM step poorly, and the dataset is small enough that streaming
  or incremental ingest add no value. Bundles also keep local pytest-driven development against `databricks-connect`
  straightforward.
- **`ai_query` over an external provider.** No API key or secret scope, Spark-native parallelism, and native
  structured-output support; the equivalent OpenAI/Anthropic call would require either single-threaded driver calls or a
  hand-rolled threadpool wrapped in a Python UDF.
- **Llama 3.3 70B over 405B.** Equivalent quality on short-context classification at lower cost and latency.

## Key findings

The 2,484 resumes distribute heavily toward seniority: **76% land in 10+**, 14% in 7-10, 7% in 3-5, and roughly 1.5% in
5-7 and 1% in 0-2. Average self-rated model confidence on the 10+ bracket is **0.95**, against 0.77–0.83 for every other
bracket. The skew therefore appears to reflect a genuinely senior-leaning corpus rather than a hedging model — when the
model commits to 10+ it does so confidently, and when it picks elsewhere it is markedly less sure. Self-rated confidence
is a soft signal, not a calibrated probability, so this should be read as suggestive rather than definitive.

Only one row fell below confidence 0.6 — a CONSTRUCTION resume of 17.5 KB with confidence exactly 0. (An earlier 6,000-
char run had two such outliers; the second moved to a confident classification once additional resume text became
visible.) That residual case suggests **0-2 acts as a fallback bucket** for resumes whose date signal the model cannot
extract — likely a structurally awkward PDF rather than a thin career history.

The non-monotonic dip at 5-7 (41 rows, the smallest bracket despite a wider population window than 0-2) survived the
cap extension, supporting a **bracket-boundary effect**: when torn between "around 5" and "around 7", the model appears
to hop to a neighbouring bracket rather than commit to the middle.

End-to-end wall time for the full pipeline on a serverless cluster was **17m 6s**, with the `pypdf` UDF in preprocess
and the LLM calls in classify being the two dominant costs. Raising the truncation cap from 6,000 to 10,000 characters
made no measurable difference to wall time.

## Challenges

- **Truncation cap.** Resume text is truncated to 10,000 characters before being sent to the model, to bound input cost.
  An initial cap of 6,000 chars was raised to 10,000 after the first full run revealed that **969 of 2,484 resumes (39%)
  exceeded 6k**; 10k closes 86% of that gap, leaving 136 (5.5%) still truncated. Because truncation keeps the _first_ N
  chars, the earliest job — the one establishing career start date — is the part most likely to be cut, biasing the model
  toward underestimating experience. The before/after comparison confirmed the direction: re-running with the 10k cap
  moved **38 rows from 7-10 into 10+** and lifted the 10+ share from 74.9% to 76.3% — exactly the expected shape, but
  small enough in magnitude to confirm that the dominant skew is a corpus property, not a cap artefact. The 26
  mega-resumes over 15,000 chars are the residual concern; head-plus-tail concatenation would close that gap without
  materially expanding the context window.
- **UDF observability.** A first end-to-end run showed `silver.write` taking ~7 min and a follow-up
  `silver.filter(...).count()` taking another ~6 min. The lazy DataFrame still carried the `pypdf` UDF in its query plan,
  so post-write counts silently re-ran the entire UDF. Re-binding to `spark.table("cv_silver")` for any post-write reads
  cut preprocess wall time from 13m 21s to 7m 25s. This is exactly the class of bug pure-Python unit tests cannot surface
  — only a real cluster run with a task timeline reveals it.
- **Bracket ambiguity.** "Years of experience" is undefined in the brief: total working years, years in a specific field,
  or years of relevant experience? The prompt was pinned to count paid roles and internships and to exclude education and
  personal projects, but a production recruiter use case would need this nailed down per role and probably split into
  multiple labels.
- **Single-node Spark at this scale.** PySpark introduces real overhead for 2,484 rows; pandas would be faster on a
  single machine. The architecture is built so that the same code scales to 100k+ documents without rewriting, which is
  the point of the brief, but for the dataset as given it is over-engineered relative to a non-distributed equivalent.

## What would extend this with more time

- Manual labelling of a 50-resume sample to measure LLM accuracy against ground truth — and to interrogate whether the
  10+ skew survives review.
- Head-plus-tail prompt construction for the residual mega-resumes (>15,000 chars), where even the 10,000-char head can
  miss the earliest start date.
- Embedding-based retrieval, so recruiters can query by skills or seniority directly rather than only filter on the
  bracket label.
- Cost projection at production scale: Foundation Model API per-token pricing applied to expected monthly CV volume,
  alongside a budget for retries on `__EXTRACTION_ERROR__` rows handled by an OCR fallback.
