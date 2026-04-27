# Report & Presentation Notes

## Architecture & Design Decisions

- Pipeline is modular with clear separation: ingestion -> preprocessing -> classification
- Each stage can be run independently, tested independently, and swapped out
- Designed for production volumes even though the sample dataset is small

## PySpark / Databricks Tradeoff

- The Kaggle dataset is ~2,400 records and fits comfortably in memory on a single node — pandas would be simpler and faster for this scale
- PySpark introduces overhead: cluster setup, serialisation costs, JVM startup, and more complex debugging
- However, the architecture is built so that if the firm receives 100k+ CVs, the same code scales horizontally without rewriting
- Worth noting: for a dataset this size, the Spark overhead likely makes the pipeline _slower_ than a single-node pandas equivalent
- The right production decision would be to start simple (pandas) and migrate to Spark only when data volume demands it

## LLM Classification

- Classification is API-bound, not compute-bound — Spark parallelism doesn't help when you're rate-limited by an external API
- This is why the brief correctly separates classification as "add-on analytics that will not go into production"
- Prompt design matters more than infrastructure here: clear instructions, consistent output format, handling edge cases (career gaps, freelance work, overlapping roles)
- Experience bracket assignment is ambiguous — need to define: total years working? years in a specific field? years of relevant experience? The prompt should make this explicit

## Preprocessing Considerations

- HTML/formatting cleanup from the raw resume data
- Deduplication strategy: exact match vs fuzzy matching (same person, updated CV)
- What "prepared for embedding/retrieval/analysis" means in practice: chunking strategy, metadata extraction, text normalisation

## Challenges to Mention

- Ambiguity in experience calculation (total vs relevant vs domain-specific)
- Resume formats are inconsistent — dates, role descriptions, and structure vary wildly
- LLM classification consistency — same resume should get the same bracket on repeated runs (temperature=0, structured output)
- Balancing the brief's requirement for PySpark with pragmatic engineering for the actual data size

## Things That Would Strengthen the Solution (If More Time)

- Evaluation metrics: sample a subset, manually label, measure LLM accuracy
- Cost estimation for LLM classification at scale (price per CV, batch API pricing)
- A simple UI or CLI for recruiters to upload and query results
- Embedding-based retrieval so recruiters can search by skills/experience rather than just filter by bracket
