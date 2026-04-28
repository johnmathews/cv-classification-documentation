# Vector Search vs Delta tables, CDF, and Bundles vs DLT — interview-qa additions — 2026-04-28

A walk-through session focused on three concepts that the existing interview-qa
document didn't cover at depth: what `cv_silver_chunks_index` actually is at
runtime (versus how it appears in Catalog Explorer), what Change Data Feed is
and where the project depends on it, and why the project uses Asset Bundles +
Jobs rather than DLT / Lakeflow Declarative Pipelines. Three new entries
added to `interview-qa.md` (Q15, Q16, Q17) and one stale memory pointer
corrected.

## What was added

### Q15 — Mosaic AI umbrella + how the index differs from a Delta table

The starting question was "there seems to be some magic going on in the
`_index` table — why is it different to a normal table?" The Q15 entry is the
written-up version of that walkthrough:

- The index is not a Delta table. It surfaces in Unity Catalog alongside
  Delta tables but is a Mosaic AI Vector Search resource: an HNSW
  approximate-nearest-neighbour store hosted on a Vector Search endpoint
  (`cv_search`), with a managed Lakeflow pipeline behind the scenes
  (the `pipeline_id` field on the index detail page is that pipeline).
- A six-row comparison table contrasts storage, query path, index structure,
  update mechanism, lifecycle, and queryable compute between
  `cv_silver_chunks` (Delta) and `cv_silver_chunks_index` (Vector Search).
- "What is Mosaic AI" — origin in the MosaicML acquisition (~$1.3B, mid-2023)
  and a layered table of the umbrella products: Foundation Model APIs,
  Model Serving, AI Gateway, Vector Search, Feature Serving, Agent Framework,
  Agent Evaluation, Playground, Foundation Model Training, MLflow 3,
  Lakehouse Monitoring.
- Vector Search architecture in two resources (endpoint = compute billed in
  VSUs; index = data layer). The Free Edition cap (one endpoint × one unit)
  is cross-referenced rather than re-stated since Q12 already covers it.
- Why the index is tied to Delta + UC (permissions inheritance, lineage,
  CDF-driven sync, canonical-copy embeddings in the Delta column).
- Where Vector Search sits competitively (Pinecone, Weaviate, Qdrant,
  pgvector, Azure AI Search) — the distinguishing pitch is "the lakehouse
  never has to be left," with the trade being platform lock-in.

### Q16 — Change Data Feed

The CDF question came up naturally from Q15 because Vector Search Delta Sync
indexes require it. Written up as a self-contained explainer:

- What CDF is — a Delta Lake feature recording row-level inserts/updates/deletes
  between commits, with a three-column extension (`_change_type`,
  `_commit_version`, `_commit_timestamp`).
- How it's enabled — the `delta.enableChangeDataFeed=true` table property,
  set at create time, on a writer, or via `ALTER TABLE`.
- Where this project enables it — three explicit sites in `chunk.py`
  (initial write, overwrite path, and a defensive `ALTER TABLE`) — a
  belt-and-braces pattern grounded in the actual `chunk.py:89/115/121`
  references.
- Why it's required — Vector Search Delta Sync uses CDF as the *mechanism*
  for incremental sync. Without it, `idx.sync()` fails with an explicit
  "source table must have Change Data Feed enabled" error.
- The orthogonality of CDF and `pipeline_type`: TRIGGERED vs CONTINUOUS is
  about *when* sync runs; CDF is about *how* it computes the delta. Both
  modes use CDF.
- Where CDF generalises beyond Vector Search (Kafka mirroring, materialised
  views in DBSQL, dbt incremental models, OLTP mirroring, audit trails).

### Q17 — Why Bundles + Jobs, not DLT

The starting question was "does the interview-qa doc explain why we use
bundles and not DLT?" — and it didn't, although the report has a one-line
bullet and `docs/02-databricks-bundles.md` has a "Jobs vs Pipelines"
subsection. Pulled the scattered material into one Q&A entry:

- Eight-row comparison table (programming model, dependency wiring,
  incremental ingest, data quality, schema evolution, scheduling, compute,
  local testability) between Jobs (chosen) and DLT.
- Where DLT is the right tool — pure Delta-to-Delta workloads with quality
  checks and incremental updates as first-class concerns.
- Five reasons DLT was rejected for this project: (1) the LLM step
  doesn't fit a deterministic-materialisation primitive, (2) dataset is
  too small for streaming/incremental to earn weight, (3) iteration
  ergonomics (per-stage hand-runs from notebooks/pytest), (4) pytest
  gating is wired into the bundle's build step (cross-references Q14),
  (5) Free Edition's separate DLT pipeline cluster type with its own
  cold-start latency was not worth a second class of provisioning lag.
- What was given up by skipping DLT — `@dlt.expect` quality gates,
  auto-inferred dependency wiring, the DLT lineage UI.
- Where the decision flips — a 100k CVs/day SFTP-drop production scenario
  would invert the calculus, with DLT for the deterministic stack and
  Jobs for the LLM tier as a hybrid.
- Cross-reference to `docs/02-databricks-bundles.md` for the broader
  Bundles surface (wheel artefact, job parameters, environment targets,
  deploy/run/destroy lifecycle).

## Memory housekeeping

The persistent memory at
`~/.claude/projects/-Users-john-projects-jobs-simmons-and-simmons-case-study/memory/`
had a stale `MEMORY.md` index entry claiming "ingest + preprocess real,
classify still stubbed; next session = LLM classification" — a snapshot
from before the LLM and Vector Search work landed. The underlying
`project_pipeline_state.md` file itself was already current (covers all five
stages real, end-to-end validated incl. retrieval contrastive-query test),
only the one-line index entry needed updating. Fixed in place.

## Why a separate journal file from the earlier 260428 entry

The earlier `260428-chunking-embeddings-vector-search.md` entry captures
implementation work: the chunk and index stages being built, the LangChain
splitter parameter choice, the 30-minute `Provisioning resources…` debugging
saga, the score-format precision gotcha, the test-gating writeup. That work
shipped with code changes in the inner `cv_classification/` repo.

This session is purely conceptual / interview-prep work in the outer repo.
The artefacts are three new Q&A entries in `interview-qa.md` and one
single-line memory index fix. Folding it into the earlier file would
muddle the implementation narrative; keeping it separate keeps each
journal file scoped to one coherent unit of work.

## Report gap-closure pass

After the Q&A additions landed, a deliberate read-through of `report.md`
against the brief surfaced four gaps worth closing before submission. Three
were addressed in this session:

1. **No executive summary.** A reviewer skimming the deliverable hit
   methodology detail before any orientation. Added a four-sentence
   summary at the top covering the pipeline shape, the brief-mandated
   PySpark + LLM choices, the headline finding (76% in 10+, confidence
   0.95 there vs 0.77–0.83 elsewhere), and the headline challenge
   (truncation-cap bias diagnosed and corrected).
2. **Free Edition / PySpark / "add-on analytics" not framed as
   constraints.** Free Edition's caps shape several architecture choices
   (Delta Sync forced, pay-per-token only, serverless-only compute), but
   the report read as if those decisions were free. PySpark was
   brief-mandated for scalability; the report acknowledged the
   single-node over-engineering trade in Challenges but never named the
   mandate. Classification being add-on rather than production was
   implicit. Added a `### Constraints` subsection under Methodology
   alongside Assumptions, naming all three explicitly so a reviewer
   ticking boxes can see them.
3. **Retrieval validation missing from Key findings.** The contrastive-
   query test from the chunking session (in-domain query in
   `[0.00225, 0.00246]` with relevant engineering chunks; "unicorns in
   a tree?" control in `[0.00171, 0.00174]` with hobbies/personal-info
   chunks) is the concrete proof that the embedding + ANN surface does
   real semantic work. Lived in the journal but not the report. Added
   a paragraph to Key findings, noting the distance-style score scale
   (so the absolute values look tight) and that the in-domain vs
   out-of-domain *delta* is the meaningful signal — not absolute scores.

The fourth potential gap — "the report doesn't echo the PySpark mandate
as a deliberate compliance bullet" — is folded into the new Constraints
subsection rather than being called out separately.

## report-notes.md follow-up

After the report itself was reconciled against the brief, a final pass on
`report-notes.md` (the presentation-talking-points companion) caught that
the file had drifted badly from reality. It read like pre-build planning
notes that were never updated as the work landed:

- Line 26 referenced "HTML/formatting cleanup" — but the dataset is PDFs,
  not HTML.
- Line 28 framed "what 'prepared for embedding/retrieval/analysis' means"
  as an open question — but chunking + embeddings + Vector Search
  index got built and validated.
- Line 42 listed embedding-based retrieval as a future enhancement —
  but it exists now.
- No mention of Mosaic AI Vector Search, the chunk/index stages, the
  truncation-cap diagnosis, the UDF re-execution bug, the 76% senior
  skew, the contrastive-query retrieval validation, Bundles vs DLT,
  or CDF.

Rewrote the file as eight sections mirroring the now-current report
(What was built / Architecture & design decisions / Constraints / PySpark
tradeoff / LLM classification / Embedding & retrieval / Headline findings
/ Challenges / Things that would strengthen). Kept it tight — the
talking-points doc explicitly should not duplicate the depth of
`interview-qa.md`. The presentation narrative leans on the report's
structure and the Q&A doc handles the deep-dive material.

## Files touched

- `interview-qa.md` — three new entries (Q15, Q16, Q17). Total document is
  now 17 questions covering the project's design rationale, Databricks-
  specific patterns, and operational lessons.
- `report.md` — added executive summary; added Constraints subsection
  under Methodology; added retrieval-validation paragraph to Key findings.
- `report-notes.md` — full rewrite to mirror the current report and reflect
  what actually got built. The original file had stale references (HTML
  cleanup, "embedding-based retrieval as a future enhancement") and was
  missing all five-stage / Vector Search / contrastive-validation material.
- `~/.claude/projects/-Users-john-projects-jobs-simmons-and-simmons-case-study/memory/MEMORY.md` —
  one-line index correction so the pipeline-state pointer reflects the
  current "all five stages real, end-to-end validated" reality rather than
  the long-stale "ingest + preprocess real, classify stubbed" line.
