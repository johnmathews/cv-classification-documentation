# Truncation cap bump + report drafted — 2026-04-27

The full classify run was executed on the 2,484-row dataset, the bracket distribution was interrogated, and the
truncation cap was raised from 6,000 to 10,000 characters off the back of the analysis. The deliverable report was
drafted at `report.md`.

## Findings from the first full run (6k cap)

End-to-end wall time **17m 20s** on a serverless cluster. Bracket distribution was heavily skewed:

| Bracket | Count | Share |
|---------|-------|-------|
| 10+     | 1,861 | 74.9% |
| 7-10    | 382   | 15.4% |
| 3-5     | 178   | 7.2%  |
| 5-7     | 36    | 1.4%  |
| 0-2     | 27    | 1.1%  |

Three observations seeded the next move:

1. **Confidence segregates by bracket.** Mean self-rated confidence on 10+ is 0.95; every other bracket sits at
   0.76–0.83. The 75% skew toward 10+ therefore looks like genuine signal rather than the model hedging — when the
   model commits to 10+ it does so confidently.
2. **Only two low-confidence outliers, both 0-2** — including one with confidence exactly 0 over 17.5 KB of text.
   That suggests **0-2 is acting as a fallback bucket** for resumes whose date signal the model cannot extract.
3. **Truncation is heavy.** 969 of 2,484 resumes (39%) hit the 6,000-char cap. Because the truncation keeps the
   _first_ N chars, the earliest job — the one establishing career start date — is the part most likely to be cut.
   That is exactly the wrong half of a resume for seniority estimation, and the bias goes one direction:
   _underestimating_ experience.

## The 6k → 10k decision

A separate query established how the truncated population thinned out as the cap grew:

| `text_length >` | rows | share |
|-----------------|------|-------|
| 6,000           | 969  | 39%   |
| 10,000          | 136  | 5.5%  |
| 15,000          | 26   | 1.0%  |

Going from 6k to 10k closes 86% of the truncation gap, leaving only 5.5% still affected. Head-plus-tail concatenation
would buy the remaining 5.5% but adds prompt-construction complexity and a second `expr()` call shape — not worth it
when a single-constant bump captures most of the value. The mega-resumes >15k chars (26 rows, ~1%) are the residual
outliers and would be the right place for a different strategy if it ever became necessary.

The change was a one-line edit to `classify.py`: `MAX_TEXT_CHARS = 6000 → 10000`.

## Findings from the re-run (10k cap)

End-to-end wall time **17m 6s** — within noise of the 6k run. The cap increase had **no measurable latency cost**;
`ai_query`'s Spark-native parallelism absorbed the extra input tokens without serialising.

Bracket distribution shifts:

| Bracket | 6k    | 10k   | Δ   |
|---------|-------|-------|-----|
| 10+     | 1,861 | 1,895 | +34 |
| 7-10    | 382   | 344   | -38 |
| 3-5     | 178   | 175   | -3  |
| 5-7     | 36    | 41    | +5  |
| 0-2     | 27    | 29    | +2  |

**38 rows moved from 7-10 to 10+**, exactly as the truncation-bias hypothesis predicted, but the magnitude is small
(1.4 percentage points). This is a clean answer: **the dominant 10+ skew is a corpus property, not a cap artefact.**
Confidence numbers barely moved (10+ still 0.95; others still 0.77–0.83), which is itself reassuring — the model
isn't suddenly less sure with more text.

The low-confidence-outlier count dropped from 2 to 1. The IT resume (0.5 confidence on 13 KB of text in the 6k run)
reclassified confidently with the extra context. The remaining outlier is a CONSTRUCTION resume with confidence
exactly 0 over 17.5 KB — a structurally awkward PDF rather than a thin career history.

The 5-7 dip survived: 41 rows, still the smallest bracket. Bracket-boundary effects, not truncation.

## Report deliverable

`report.md` was drafted in the outer repo at ~750 words / 1.5 pages. Sections: Methodology, Architecture decisions,
Key findings, Challenges, What would extend this with more time. The report cites the 10k-run numbers, narrates the
6k → 10k experiment as a methodological caveat in the truncation paragraph, and ends on concrete next steps
(manual-labelling sample, head-plus-tail for mega-resumes, embedding retrieval, cost projection).

## Outer repo doc updated

`docs/04-pipeline-structure.md` was updated to reference the 10,000-char cap. The earlier journal entry
(`260427-llm-classification.md`) still references the 6,000-char cap, which is correct as a point-in-time record —
journals are append-only.
