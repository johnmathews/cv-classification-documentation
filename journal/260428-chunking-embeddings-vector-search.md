# Chunking, embeddings, Vector Search — 2026-04-28

The case-study brief asked for preprocessing to "prepare documents for embedding /
retrieval / analysis". The first pass through the pipeline stopped at clean text in
`cv_silver` — embedding-ready only in the trivial sense that any string can be fed
to an embedder. No chunking, no embeddings, no vector index. This session closed that
gap.

## What was built

Two new pipeline stages and a hand-runnable demo, plus the supporting Vector Search
infrastructure.

```
ingest ─► preprocess ─┬─► chunk ─► index
                      └─► classify
```

- `chunk` (`cv_classification.chunk:main`) — reads `cv_silver`, filters out
  `__EXTRACTION_ERROR__` rows, applies `RecursiveCharacterTextSplitter` (1,600 chars
  / 200 overlap, separators `["\n\n", "\n", ". ", " ", ""]`), explodes one row per
  chunk, and embeds via `ai_query('databricks-gte-large-en', chunk_text)`. Writes
  `cv_silver_chunks` with `(sha256, category, chunk_id, chunk_text, char_start,
  char_end, chunk_uid, embedding)`. Sets `delta.enableChangeDataFeed = true` and a
  PK constraint on `chunk_uid` so Vector Search can sync.
- `index` (`cv_classification.vector_index:main`) — idempotently creates the
  `cv_search` Vector Search endpoint and the `cv_silver_chunks_index` Delta Sync
  Index over the embedding column, then triggers a sync and waits for ready state.
- `retrieve` (`cv_classification.retrieve:main`) — hand-runnable demo. Embeds four
  canned queries via the SDK's `serving_endpoints.query`, runs `similarity_search`,
  prints top-5 results with score, category, sha prefix, and a 160-char preview.
  Not wired into the DAG.

`chunk` and `classify` are independent — both read `cv_silver` — so the DAG fans out
after preprocess and they run in parallel. `index` is sequenced after `chunk`.

## Decisions worth recording

**Self-managed embeddings + Delta Sync.** Direct Vector Access indexes aren't
supported on Free Edition, so Delta Sync was the only legal index type. Within
Delta Sync, embeddings live in a Delta column rather than being computed inside the
index. Trade is: more inspectable (vectors are queryable in SQL), more reproducible
(embeddings tied to a model version, portable if the index is rebuilt), and chunking
parameters can be tuned without re-embedding inside the index. Loses the slight ops
simplicity of "managed embeddings just work."

**Chunk size 1,600 / overlap 200.** Resumes are short, dense documents.
Sub-256-token chunks fragment specific role claims; 1,000+-token chunks blur the
centroid. 1,600 chars (~400 tokens at GTE's tokenisation) is the sweet spot. The
12.5% overlap stops a single sentence ("5 years at Goldman as a quant") being split
across two chunks where neither carries the full claim.

**LangChain text splitters, not hand-rolled.** Pulled in `langchain-text-splitters`
rather than writing a recursive splitter ourselves. The package is small (split out
from monolithic `langchain` for exactly this reason), well-tested on edge cases like
CRLF and oversized atomic units, and is what a production team would reach for. Cost
is a few transitive deps (`pydantic`, `langchain-core`, `tenacity`); benefit is one
fewer thing to debug.

**Synthetic primary key.** Vector Search Delta Sync indexes need a single
primary-key column. The natural composite (sha256, chunk_id) doesn't fit, so
`chunk_uid = sha256 || '_' || chunk_id` is added in the chunk stage and pinned via
an `ALTER TABLE … ADD CONSTRAINT … PRIMARY KEY` statement.

## Free Edition findings

Worth pinning down before designing further:

| | Free Edition |
|---|---|
| Vector Search endpoints | 1 |
| Vector Search units | 1 per endpoint |
| Direct Vector Access | not supported |
| Delta Sync Index | supported |
| Foundation Model API | pay-per-token only (no provisioned throughput, no GPU) |
| Compute | serverless only |

The single-endpoint cap means dev and prod can't run side-by-side on the same
workspace; for a case study that's a non-issue. One unit easily handles 12k vectors
(units handle millions).

## Embedding model — `databricks-gte-large-en`

Based on Alibaba GTE-large-en-v1.5: 434M parameters, 1024-dim output, 8192-token
context, MTEB 65.39 (state-of-the-art for its size class on English retrieval).
Hosted by Databricks behind the same `ai_query()` mechanism used for classification,
so no second auth surface to manage. The 8192-token window comfortably accommodates
1,600-char (~400-token) chunks — no tokeniser overflow.

Cost for the full 2,484-CV dataset is well under $1 of pay-per-token DBUs at the
volumes embedding models are typically priced at; not worth a precise measurement.

## Files touched

- `cv_classification/pyproject.toml` — added `langchain-text-splitters` and
  `databricks-vectorsearch` deps; added `chunk`, `index`, `retrieve` entry points.
- `cv_classification/src/cv_classification/chunk.py` — new.
- `cv_classification/src/cv_classification/vector_index.py` — new.
- `cv_classification/src/cv_classification/retrieve.py` — new.
- `cv_classification/resources/cv_pipeline.job.yml` — added `chunk` and `index`
  tasks.
- `cv_classification/tests/test_chunk.py` — new; 5 tests covering short/long/empty
  inputs, char-offset correctness, parameter sanity. All pass alongside the existing
  8 tests (13 total).
- `report.md` — methodology rewritten for five stages; assumptions section
  (English-only, pypdf strips layout); architecture decisions for embedding model,
  self-managed embeddings, chunking parameters; "what would extend" updated.
- `docs/04-pipeline-structure.md` — silver-chunks schema, vector search section,
  updated DAG ASCII, updated current-state table.
- `interview-qa.md` — Q12 added covering Delta Sync vs Direct Access, self-managed
  vs managed embeddings, sync modes, Free Edition caps.

## Deploy outcome

`databricks bundle deploy && databricks bundle run` succeeded on first try. DAG
shape from the Workflows UI:

```
ingest (1m 28s) ─► preprocess (6m 55s) ─┬─► chunk (2m 4s) ─► index (still running 11m+)
                                        └─► classify (still running 13m+)
```

`chunk` finishing in 2m 4s is faster than expected; the `pypdf` UDF didn't
accidentally re-run on the chunk read (the lazy-DataFrame trap from preprocess
was avoided by always rebinding to `spark.table(...)` after a write), and the
embedding `ai_query` calls parallelised cleanly. `index` taking 10+ minutes is
normal-ish — endpoint provisioning on a cold workspace is 5–10 min on its own
before sync starts.

## Logging gaps caught after the run

The first deploy surfaced two classes of logging weakness:

1. **`chunk`** — between "now embedding…" and "done", a stuck `ai_query` would
   be invisible. Added an explicit "embedding write complete" line and a
   NULL-embedding count in the final `done` line. Also added a chunk-length
   distribution (min/avg/max) and chunks-per-CV ratio after the first write —
   cheap sanity checks that the splitter is doing what we think. And a skipped
   `__EXTRACTION_ERROR__` count at the start, symmetric with `preprocess`.
2. **`vector_index`** — the sync polling loop logged the same line every 15s
   with no elapsed time and no row-count progress. Added `elapsed`,
   `indexed_row_count`, and a 30-minute deadline that raises `TimeoutError`
   rather than spinning forever. Endpoint state is now logged on the "already
   exists" path so a half-built endpoint from a prior failed run is visible.

`retrieve` got a result-count log per query so the silent "0 results" case
fails loudly.

## Index task: 30+ minutes "Provisioning resources..." with no pipeline_id

When the bundle ran end-to-end, the `index` task hung in `Provisioning resources…`
state for over 30 minutes with `Pipeline id: -` on the index detail page. Our
30-minute timeout in the polling loop never fired because the SDK was still
blocked on `create_delta_sync_index_and_wait` — the polling loop is on the
*other* side of that call.

Three diagnostic dead-ends along the way before the actual cause surfaced:

1. **Free Edition pipeline-slot exhaustion theory.** Plausible from the docs
   ("one active pipeline per pipeline type"), but Jobs & Pipelines showed only
   the user's `cv_pipeline` job — no internal Vector Search pipelines visible.
   Inconclusive.
2. **Source table prerequisites theory.** Verified `cv_silver_chunks` had
   `delta.enableChangeDataFeed=true` (Details tab) and a PK constraint on
   `chunk_uid` (Overview tab, the orange PK chip). Both fine.
3. **`array<double>` vs `array<float>` theory.** The embedding column showed
   as `array<double>` in Catalog Explorer and Vector Search docs say
   `array<float>` is required. Looked like the smoking gun. Was wrong — the
   index eventually came online with the existing `array<double>` column.

**Actual cause**: just Free Edition's shared serverless infrastructure being
slow at the initial provision step. The user cancelled the task at ~30 min,
but the underlying Vector Search resource kept provisioning server-side
(resources are independent of the Python process that requested them) and
eventually transitioned to `Online` with all 12,013 rows synced. A subsequent
manual re-run of the `index` task hit the "already exists" branches in
`_ensure_endpoint`/`_ensure_index` and completed in under a minute.

The `array<float>` cast got kept in `chunk.py` anyway — not load-bearing for
Vector Search, but worth doing on its own merits: `databricks-gte-large-en` is
a 32-bit model so the extra precision in `double` is illusory, and halving the
column width saves ~50MB at this scale. Comment in the code reflects the
honest reasoning.

## Refactor: non-waiting create + unified polling

Direct lesson from the above: `create_delta_sync_index_and_wait` is the wrong
abstraction when you want a deadline you control. Switched to non-waiting
`create_delta_sync_index`, which returns immediately. The caller's polling
loop now owns the entire provisioning + sync window with a single 30-minute
timeout. `_ensure_index` returns `(idx, is_new)` so the caller knows whether
to fire an explicit `sync()` (existing index) or rely on creation's auto-sync
(new index). One observable timeout instead of two opaque ones.

## Retrieval validated end-to-end

Built a quick notebook at `/Workspace/Users/jonnosgone@gmail.com/retrieval-demo`
to exercise the index. Two gotchas worth knowing for future debugging:

1. **`databricks-vectorsearch` isn't pre-installed on serverless.** Needs
   `%pip install databricks-vectorsearch` + `dbutils.library.restartPython()`
   before the import.
2. **Score format hides variation.** Initial results showed every score as
   `0.002` and looked broken — turned out to be the `f"{score:.3f}"` format
   rounding away the actual variation. Switched to `:.6f` and saw the real
   values: top-5 for "python data engineer with spark and aws experience"
   landed in `[0.00225, 0.00246]`, while a contrastive query
   ("unicorns in a tree?") landed in `[0.00171, 0.00174]` with a totally
   different set of chunks. Real semantic discrimination, just on a tight
   distance-style scale rather than the [0, 1] cosine convention I expected.

The contrastive-query test is what definitively confirmed the embedding +
retrieval surface is doing real work. Without the unicorns query, the
narrow score range on the data-engineer query was ambiguous.

The "Additional Information" / personal-trivia sections of CVs surface for
the unicorns query — sensible, since those chunks are the parts of the
corpus least connected to any concrete professional concept.

## Test-gating via the bundle's build artifact

Wired pytest into the bundle's build step in `databricks.yml`:

```yaml
artifacts:
  python_artifact:
    type: whl
    build: uv run pytest && uv build --wheel
```

`databricks bundle deploy` runs that command verbatim, so a red test fails
the build, which means the wheel never uploads, which means the deploy
aborts. Verified by deliberately breaking a test, redeploying, observing the
failure at "Building python_artifact…". Zero new infrastructure (no GitHub
Actions, no pre-commit hook), one line of YAML, and every deploy is now
gated on a green test suite. Pattern generalises to ruff / mypy / any other
pre-build check.

The Databricks CLI suppresses subcommand stdout by default, so `--debug`
is the way to surface pytest output during a deploy. The definitive proof
that gating works is breaking a test on purpose, not reading deploy logs.

## Wrap-up housekeeping

- Removed unused `PRIMARY_KEY_COLS` constant in `vector_index.py` (leftover
  from a draft that considered a composite PK).
- `ruff check` clean across `src/` and `tests/`. Coverage at 59% — the
  pure-Python `_underscore` helpers are 100% covered; `main()` functions
  intentionally aren't, since they require a Databricks runtime to test
  meaningfully and are validated end-to-end by `bundle deploy`.
- Outer repo `.gitignore` extended with standard secret patterns
  (`.env`, `*.key`, `*.pem`, etc.) as defence-in-depth, even though it's a
  docs-only repo.
- New Q14 in `interview-qa.md` distils the test-gating pattern.
