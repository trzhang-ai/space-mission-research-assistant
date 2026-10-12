# Evaluation guide

How answer and retrieval quality are measured, and the saved baseline.

## Metric meanings

| Metric | What it asks |
|---|---|
| Response relevancy | Does the answer address the question? |
| Faithfulness | Are answer claims supported by the retrieved context? |
| Context precision | How much of the retrieved context is useful for the reference answer? |
| Context recall | How much of the reference answer is covered by retrieved context? |
| Factual correctness | How well do answer claims agree with the reference answer? |
| Retrieval F1 | Harmonic mean of context precision and context recall |

All scores range from 0 to 1, where higher is better, but there is no universal
pass threshold. Response relevancy compares the question with questions
reconstructed from the answer. Faithfulness checks whether answer claims are
supported by retrieved text. Context precision rewards useful chunks appearing
early in the ranking, while context recall checks whether retrieved text
contains the claims needed by the reference answer. Factual correctness is
claim-level F1 between the generated and reference answers. Retrieval F1 is
this project's harmonic mean of context precision and context recall.

## Worked example

Suppose an Apollo 13 question asks what happened to the oxygen and fuel-cell
systems and what decision followed:

1. The first retrieved chunk describes the tank and fuel-cell failures
   (`useful`).
2. The second contains only launch-time information (`not useful`).
3. The third says the landing was scrubbed and the lunar module became a
   lifeboat (`useful`).

The answer repeats those supported facts, but the reference also says the
mission continued as a lunar-flyby abort. Under these simplified judge
decisions:

- response relevancy would be high because the answer directly addresses the
  question;
- faithfulness would be `1.00` if every answer claim is supported;
- context precision would be `(1 + 2/3) / 2 = 0.833`, because useful chunks
  occur at ranks 1 and 3;
- context recall would be `4/5 = 0.80` if four of five reference claims are
  supported by the retrieved text;
- factual correctness would be about `0.89` for four matching claims, no false
  answer claim, and one missing reference claim;
- retrieval F1 would be approximately `0.816`, the harmonic mean of `0.833`
  and `0.80`.

These numbers are illustrative, not observed results. RAGAS uses an LLM to
generate questions, split claims, and make support judgments, so the actual
decomposition and scores can vary between runs.

RAGAS metrics are model-based and may vary slightly between runs. If one
metric fails, its value is marked as `NaN` internally and serialized as JSON
`null` with a metric-specific error, while the remaining metrics and questions
continue.

## Verified baseline

The local collection used for the baseline contains:

- 5,584 chunks from 12 source files
- Apollo 11: 2,707 chunks
- Apollo 13: 2,526 chunks
- Challenger: 351 chunks
- 500-token maximum chunk size and 100-token configured overlap
- 1,536-dimensional `text-embedding-3-small` vectors

The 2026-07-28 full run completed all 17 test questions without a
question-level failure. Its configuration and complete per-question evidence
are stored in
[`evaluation_results_2026-07-28.json`](../evaluation_results_2026-07-28.json).

| Metric | Mean |
|---|---:|
| Response relevancy | 0.7846 |
| Faithfulness | 0.8158 |
| Context precision | 0.5560 |
| Context recall | 0.5000 |
| Factual correctness | 0.2953 |
| Retrieval F1 | 0.4300 |

The metric means answer different questions:

- `0.7846` response relevancy: answers generally address the questions.
- `0.8158` faithfulness: most generated claims are supported by retrieved
  evidence.
- `0.5560` context precision: useful evidence is only moderately concentrated
  near the top of the retrieval ranking.
- `0.5000` context recall: retrieval covers only about half of the
  reference-answer claims under the evaluator's attribution judgments.
- `0.2953` factual correctness: generated and reference claims align weakly
  overall, even though answers are usually grounded in what was retrieved.
- `0.4300` retrieval F1: the combined precision/recall balance is
  moderate-to-low.

Together, these results suggest that answer grounding is stronger than
retrieval completeness. The main improvement target is therefore to retrieve
more complete, reference-relevant evidence and then reduce missing or
mismatched answer claims. These results are a baseline, not a claim that
retrieval is fully optimized.
