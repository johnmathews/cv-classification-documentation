# Report & Presentation Notes

Talking points for walking through the deliverable, ordered roughly to match `report.md`. Not a duplicate of `interview-qa.md` — the Q&A doc has the deep-dive material, this file is the surface-level narrative.

## Opening pitch (~30 seconds)

- A five-stage Databricks pipeline — `ingest → preprocess`, then `chunk` and `classify` fan out in parallel with `index` after `chunk` — processes 2,484 unique CVs end-to-end on Free Edition serverless compute in **23m 34s**.
- **Headline challenge:** building a production-pattern pipeline (medallion tables, Vector Search, Foundation Model API, Asset Bundles) within Databricks Free Edition's constraint set — one Vector Search endpoint, Delta Sync only, pay-per-token Foundation Models, serverless-only compute.
- **Headline finding:** the corpus is heavily senior-skewed — 76% of resumes land in 10+, with model confidence on 10+ averaging 0.95 against 0.77–0.83 elsewhere; that gap supports a genuine corpus property, not a hedging-model artefact.

## What was built

- Five-stage Databricks pipeline. `ingest → preprocess` runs serially; `chunk` and `classify` then fan out from `preprocess` and run in parallel, with `index` sequenced after `chunk`.
- Wired as a DAG of dependent tasks in a single Databricks Job, deployed via Asset Bundles. One wheel artefact, five entry points (`ingest:main`, `preprocess:main`, `chunk:main`, `vector_index:main`, `classify:main`).
- Stage outputs follow medallion-architecture naming: `cv_bronze` → `cv_silver` → `cv_silver_chunks` → `cv_gold`, all in `cv_classification_catalog.dev.*`.
- A Mosaic AI Vector Search index (`cv_silver_chunks_index`) is layered on top of `cv_silver_chunks` to satisfy the brief's "prepared for embedding/retrieval/analysis" requirement.
- End-to-end wall time on serverless compute: **23m 34s** — ingest 1m 14s, preprocess 6m 46s, chunk 1m 58s, index 21s, classify 15m 32s. `chunk` and `classify` fan out after `preprocess` and run in parallel, so classify (15m 32s) drives the post-fork branch on its own; `pypdf` extraction in `preprocess` is the second dominant cost.

## Architecture & design decisions

- **Five tasks, not one.** Failure isolation, separate task logs, free Delta caching between tasks. Re-running classify after a prompt change costs nothing upstream; re-running chunk after a parameter change leaves the rest of the pipeline alone.
- **Bundles + Jobs, not DLT.** DLT's declarative model fits the LLM step poorly (paid, non-deterministic, externally dependent — not a clean materialisation primitive), and the dataset is too small for DLT's streaming/incremental story to earn its weight. Bundles also let pytest gate the build via the `artifacts.python_artifact.build` step.
- **`ai_query` over an external provider for both classification and embedding.** No API key, no secret scope, Spark-native parallelism, structured-output support via `responseFormat`. Same auth surface for both calls.
- **Llama 3.3 70B over 405B.** Equivalent quality on short-context classification at lower cost and latency.
- **Self-managed embeddings + Delta Sync Index.** Embeddings live in a Delta column rather than being computed inside the index, so vectors are inspectable in SQL, the embedding step is just another `ai_query()` call (uniform medallion story), and chunking parameters can be tuned without re-embedding inside the index.
- **Chunk size 1,600 chars / 200 overlap.** Resumes are short, dense documents — sub-256-token chunks fragment specific role claims; 1,000+-token chunks blur the centroid. ~400 tokens per chunk, ~12.5% overlap.

## Constraints (worth naming explicitly)

- **PySpark for ingestion + preprocessing** — brief-mandated for scalability. Single-node pandas would be faster at 2,484 rows; the architecture trades that overhead for "100k+ CVs scales without rewriting."
- **Databricks Free Edition** — caps Vector Search to one endpoint × one unit, forces Delta Sync (Direct Vector Access not supported), restricts Foundation Model API to pay-per-token, all compute serverless. Several architecture decisions are downstream of this.
- **External storage access via custom Access Connector.** The auto-created Access Connector lives in Free Edition's locked managed resource group and is unusable due to deny assignments — a second was created in an unlocked resource group and registered via the CLI rather than SQL. Worth flagging as the supporting blocker behind the headline challenge.
- **Classification as add-on analytics, not production.** Per the brief — no retry-with-backoff, no per-row cost monitoring, no shadow-mode comparison. These would all come with productionisation.

## PySpark / Databricks tradeoff

- Kaggle dataset is ~2,400 records and fits comfortably in memory on a single node — pandas would be simpler and faster for this exact scale.
- PySpark introduces overhead: cluster setup, serialisation costs, JVM↔Python crossings on UDFs, more complex debugging.
- The architecture is built so that the same code scales to 100k+ CVs without rewriting, which is the point of the brief.
- For a dataset this size, Spark overhead likely makes the pipeline *slower* than a single-node pandas equivalent.
- The right production decision would be to start simple (pandas) and migrate to Spark only when data volume demands it. Here the migration was forced by brief, not by data volume.

## LLM classification

- Classification is API-bound, not compute-bound — Spark parallelism helps because `ai_query` parallelises naturally across executors, but the floor is set by Foundation Model API rate limits, not cluster size.
- Llama 3.3 70B via `ai_query('databricks-meta-llama-3-3-70b-instruct', _prompt, responseFormat => ...)`. The `responseFormat` JSON schema constrains `bracket` to the five-value enum and adds a `confidence` field — guaranteeing the model can only return a legal label.
- Prompt pinned to count paid roles and internships; education and personal projects excluded.
- Bracket assignment is genuinely ambiguous: total years working? years in a specific field? years of relevant experience? Prompt makes the choice explicit but a recruiter use case would split this into multiple labels per role.
- Resume text truncated to 10,000 chars before the model sees it (raised from 6,000 after first run — see Challenges).

## Embedding & retrieval

- Chunking via LangChain's `RecursiveCharacterTextSplitter` (1,600 / 200, paragraph→newline→sentence→space fallback separators).
- Embedding via `ai_query('databricks-gte-large-en', chunk_text)` — 434M-parameter Alibaba GTE-large-en-v1.5, 1024-dim, 8192-token context, MTEB 65.39 (state-of-the-art for size class on English retrieval).
- Vector Search index is a Delta Sync Index over the embedding column. Source table requires `delta.enableChangeDataFeed=true` and a single-column primary key (`chunk_uid`) so the index can sync incrementally.
- TRIGGERED sync mode (job-driven, not continuous) — fits the batch DAG.
- **Retrieval validated end-to-end via a contrastive-query test.** In-domain query "python data engineer with spark and aws experience" returned a top-5 band of `[0.00225, 0.00246]` with relevant engineering chunks. Out-of-domain control "unicorns in a tree?" returned a *different* top-5 band of `[0.00171, 0.00174]` with hobbies/personal-info chunks. The delta between the two bands confirms the embedding + ANN surface is doing real semantic work, not returning generic top-K.

## Headline findings to mention

- **Heavy senior skew.** 76% of resumes land in 10+, 14% in 7-10, 7% in 3-5, ~1.5% in 5-7, ~1% in 0-2. Average model confidence on 10+ is 0.95 vs 0.77–0.83 for every other bracket — a corpus property, not a hedging-model artefact.
- **0-2 acts as a fallback bucket.** Only one resume scored confidence < 0.6 — a structurally awkward CONSTRUCTION PDF where the date signal couldn't be extracted.
- **Bracket-boundary effect at 5-7.** Smallest bracket despite a wider population window than 0-2; the model appears to hop to a neighbour rather than commit to "around 6."
- **24 categories captured at ingest.** Profession is parsed from the parent-directory name and propagated through silver and gold — a second slicing axis for downstream consumers without a second pass over the data.

## Challenges to mention

- **Truncation cap.** 6,000-char initial cap was raised to 10,000 after the first full run revealed 39% of resumes exceeded 6k. Truncation keeps the *first* N chars, so the earliest job — the one establishing career start date — is the part most likely to be cut, biasing classification toward underestimating experience. The before/after re-run moved 38 rows from 7-10 into 10+ (10+ share lifted from 74.9% to 76.3%) — direction confirmed but small magnitude, so the senior skew is genuinely a corpus property.
- **UDF observability bug.** First end-to-end run had `silver.write` taking 7 min and a follow-up `silver.filter(...).count()` taking another 6 min — Spark silently re-ran the entire `pypdf` UDF because the post-write count used the lazy DataFrame whose query plan still contained the UDF. Fix: rebind to `spark.table("cv_silver")` for any post-write reads. Cut preprocess wall time from 13m 21s to 7m 25s. The class of bug pure-Python unit tests can't surface — only a real cluster run with a task timeline reveals it.
- **Vector Search cold-provisioning latency.** First-time endpoint creation took ~10 min; the Delta Sync index then sat in `Provisioning resources…` for 30+ min before transitioning to Online. None of this is in the docs as typical-case latency. Subsequent runs hit the "already exists" branches and finish in seconds.
- **Bracket ambiguity.** "Years of experience" is undefined in the brief — pinned in the prompt but a production deployment would need it nailed down per role.

## Things that would strengthen the solution (if more time)

- Manual labelling of a 50-resume sample to measure LLM accuracy against ground truth — and to interrogate whether the 10+ skew survives review.
- Head-plus-tail prompt construction for the residual mega-resumes (>15,000 chars).
- Hybrid retrieval (vector + BM25) and reranking via Databricks-hosted `bge-reranker` for higher precision on top-k.
- Section-aware chunking once the corpus is more uniform — splitting on `EXPERIENCE` / `EDUCATION` / `SKILLS` boundaries to let recruiters filter retrieval to specific resume sections.
- Cost projection at production scale: per-token Foundation Model API pricing applied to expected monthly CV volume, plus a budget for retries on `__EXTRACTION_ERROR__` rows handled by an OCR fallback.
- A simple UI or CLI for recruiters to upload CVs and query the index by skills / experience rather than only filtering by bracket.
- Data-contract validation at every layer (Great Expectations, Soda, or `@dlt.expect` if migrating to a hybrid DLT + Jobs architecture at scale).
