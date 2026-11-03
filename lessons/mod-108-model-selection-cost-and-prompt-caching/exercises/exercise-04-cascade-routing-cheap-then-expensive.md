# Exercise 04 — Cascade routing: cheap then expensive

Paired with [chapter 4 — routing by task shape](../04-routing-cascade-and-batch.md).

**Estimated effort:** 90 minutes.

## Objective

Implement a cascade router: run the request against a small model first; if the small model's answer looks confident and correct-shaped, ship it; otherwise, re-run against the frontier model. Measure the escalation rate `p`, the accuracy of the cascade, and the average cost per call. Compare against the always-small and always-frontier baselines from exercise 02.

The goal is to see, with numbers, the cost curve of cascade routing on a real task — and to feel where the confidence check is the whole game.

## Problem statement

Reuse the task and 50-example test set from exercise 02 (or pick a new one that admits a confidence signal). Implement `classify_cascade(input) -> dict` with the following behaviour:

1. Call the **small** model with the same prompt used in exercise 02.
2. Determine whether the small-model output looks confident.
3. If confident, return it.
4. If not confident, call the **frontier** model with the same prompt. Return that result.

Then, measure:

- **Escalation rate `p`**: fraction of the 50 requests that escalated.
- **Cascade accuracy**: fraction of the 50 requests whose final answer matches the ground truth.
- **Average cost per call**: from the `usage` blocks — sum(cost for all calls, including both models on escalated requests) / 50.
- **Latency**: p50 and p95 wall-clock across the 50 requests. (Escalated requests are the slow tail — think about how much of the p95 they own.)

Compare against exercise 02's numbers:

| approach | accuracy | mean cost / call | p50 latency | p95 latency |
|---|---|---|---|---|
| always small | ... | ... | ... | ... |
| always frontier | ... | ... | ... | ... |
| cascade | ... | ... | ... | ... |

## Requirements — the confidence check

The `is_confident(cheap_result)` function is task-specific. Pick **one** of the following approaches and justify the choice in a comment:

- **Structural.** The small model's output is valid JSON matching the schema, and every field is a permitted enum value. If not, escalate.
- **Explicit confidence.** Ask the small model to also return a `"confidence": 0.0–1.0` field. Escalate if confidence is below a threshold you pick (0.6–0.8 is typical).
- **Self-consistency.** Call the small model twice with `temperature > 0` (or in a way that produces variation) and check whether the two outputs agree. If they disagree, escalate.
- **Second-model classifier.** Send the small model's output to a lightweight classifier (regex, another tiny prompt, a scoring rule) that returns "looks right" vs. "escalate."

Whichever you pick, write down **why** you picked it and what the escalation trigger is, at the top of the function.

## Requirements — the failure exploration

The cascade only works if the confidence check is honest. Run two sanity checks:

1. **Miscalibration audit.** Look at the requests the small model **did not escalate**. How many of those were actually wrong per the ground truth? If more than 20% of "confident" calls were wrong, your confidence check is too permissive.
2. **Over-escalation audit.** Look at the requests the small model **did escalate**. How many did the frontier model actually get differently? If most escalations end with the frontier producing the same answer the small model gave, you are paying twice for the same answer — the confidence check is too strict.

Print both numbers alongside the main table.

## Requirements — the observability

Log per request:

- Whether it escalated (`escalated: bool`).
- Both `usage` blocks (small model always; frontier only if escalated).
- The final model whose output was returned.
- Wall-clock latency for each model call.

Print, in addition to the summary table:

- Escalation rate `p`.
- Cost breakdown: `cost from small-model calls` vs. `cost from frontier-model calls`. This is how you'd communicate the win to the team ("we spent 40% on frontier, 60% on small, versus 100% on frontier before").

## Starter guidance

- Chapter 4 above, especially the "escalation rate" section.
- Cost arithmetic from chapter 2 and exercise 01. On an escalated request you pay for **both** the small call and the frontier call — do not forget the small one.
- Do not hard-code confidence thresholds without measuring. Try 2–3 values and pick the one where the cascade's accuracy is close to always-frontier and the cost is close to always-small.
- For latency, remember that the cascade adds the small-model latency to every request, even the escalated ones. This is fine as long as the small model is fast; but on a slow small model, the cascade can be a latency regression.

## Acceptance criteria

- Cascade router works on all 50 examples end-to-end.
- The summary table shows accuracy, mean cost / call, and p50 + p95 latency for always-small, always-frontier, and cascade — all three from real runs, not extrapolations.
- Escalation rate `p` is reported. Cost breakdown between small and frontier is reported.
- Cascade cost per call is meaningfully lower than always-frontier — for a good confidence check on a task where the small model handles the majority of inputs, expect something in the 2–5× reduction range.
- Cascade accuracy is within a small margin (e.g., ≤ 2 percentage points) of always-frontier. If it isn't, the confidence check is broken — fix it and re-run.
- Miscalibration audit and over-escalation audit numbers are printed alongside the summary.

## Stretch goals

- Add a **second confidence-check implementation** (e.g., structural + self-consistency together) and compare against the single-check version. Which trades better between escalation rate and cascade accuracy?
- Add a **fast-first-then-escalate** variant (chapter 4, pattern 2). Instead of blocking on the small model's confidence, fire both calls in parallel; yield the small model's answer immediately; then yield the frontier's if it meaningfully disagrees. Measure the "disagreement rate" and the perceived time-to-first-token.
- Wire the cascade into the **budget enforcement** from exercise 01: escalated requests should be counted correctly (input + output tokens for both models) against the per-user budget.
- Add a **provider-mixed cascade**: small model from one provider, frontier model from another. Measure whether the per-provider difference in the small-tier's escalation rate is worth the operational cost of two providers.
- Enable **prompt caching** (from exercise 03) on the small model's call. Measure the combined effect: cache savings + cascade savings. Is the small model now so cheap that the cascade is dominated entirely by the escalated calls?
