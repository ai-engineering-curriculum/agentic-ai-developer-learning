# Exercise 03 — Rule vs. similarity vs. LLM-as-a-judge scoring

Paired with [chapter 2 — scoring outputs](../02-scoring-outputs.md).

**Estimated effort:** 2 hours.

## Objective

On the same golden set from exercise 01, score the *same* model outputs three different ways — rule-based, similarity-based, and one calibrated LLM-as-a-judge — and produce a comparison table that shows the cost, latency, agreement with your own hand-labels, and per-row disagreement between the three scoring families. By the end of the exercise you can defend which scorer to reach for on which kind of row, from data on your own feature — not from vibes or from a blog post.

You are not adding scorers to production CI in this exercise. You are running a scoring comparison as a one-off, and then adopting the outcome by editing the tolerance blocks in `golden.jsonl`.

## Problem statement

1. **Select ~10 rows** from your golden set whose `expected` is a free-text field (an answer, a summary, an explanation, a paraphrase) — not the JSON-shape rows where an `exact` check obviously wins. You are trying to compare scorers on the rows where the choice actually matters.
2. **Hand-label those 10 rows** twice:
   - Once, judge the *reference* answer as pass (`ref_label = "pass"` for every row — the reference is by construction correct).
   - Then, run the model to produce *actual* outputs for each row and hand-label each as `hand_label = "pass" | "fail"` based on your own read of the rubric. This is the ground truth you will compare the scorers against. Do this **before** looking at any of the three scorer outputs — otherwise you will unconsciously anchor to whichever one you look at first.
3. **Score the 10 outputs three ways:**
   - **Rule-based.** Whatever keyword / regex / length / schema check most cheaply captures your rubric. Even if it is coarse. Even if it obviously misses some rows.
   - **Similarity.** Cosine of the actual against your `reference_*` field under one pinned embedding model, with a threshold you have calibrated on the row set (see below).
   - **LLM-as-a-judge.** A single judge prompt with a written-out rubric that says what "correct" means, returning `{"pass": bool, "reason": str}`. Use a cheap-tier model (e.g. `gpt-4o-mini` / `haiku`) — do not default to frontier.
4. **Calibrate the judge, minimally.** On the 10 rows you have hand-labelled, run the judge and compute agreement with your hand-labels. If agreement is worse than ~85% (i.e. more than 1–2 mismatches), rewrite the rubric — tighter definitions, one worked "pass" example and one worked "fail" example in the prompt — and rerun until you clear 85% or run out of budget. Log every iteration.
5. **Calibrate the similarity threshold, minimally.** From the 10 rows you have hand-labelled `pass` or `fail`, compute the cosine similarity of each actual against its reference. Pick the threshold that best separates the two groups. If the two groups overlap heavily, similarity is not the right scorer for this rubric — note that explicitly.
6. **Produce a comparison table** with one row per golden-set row and columns for: `hand_label`, `rule_pass`, `similarity_score`, `similarity_pass`, `judge_pass`, `judge_reason`, `disagreement` (which pair of scorers disagreed with your hand-label). Then aggregate: total cost per scoring family, total wall-clock per family, agreement rate with hand-labels per family.
7. **Write a short conclusions file** (`SCORING.md`) with one paragraph per family: what it caught, what it missed, and which per-row shape it is the right default for. Then update `golden.jsonl` — for each of the 10 rows, adopt the scorer that best matches the rubric, and record the calibration numbers in the tolerance block so a future reader can see where the threshold came from.

## Requirements

### The hand-labels

- Do them **before** you look at any scorer output. If you look at the judge first you will grade rows to match it.
- Two people is better than one; even a colleague spending five minutes on the same rows exposes rubric ambiguity. If you do it alone, at minimum re-read your own labels an hour later and flip any you second-guess.
- Save the hand-labels in a `labels.jsonl` file — one line per row with `{"id": "...", "actual": "...", "hand_label": "pass|fail", "rationale": "one sentence"}`. This is the file every scorer's agreement is computed against.

### The rule-based scorer

- Reuse `evals/scoring.py` from exercise 02.
- For each of the 10 rows, add a `tolerance` shape that expresses the rule ("must mention X, Y, Z"; "must not mention W"; "must match regex"; "max_words: 40"). If the rule you can write is obviously coarse, use it anyway — you are measuring what coarse rules catch and miss.

### The similarity scorer

- One embedding model, pinned. `text-embedding-3-small` is a fine default.
- Cache the reference embedding once per row (do not re-embed the reference on every run). Cache the actual's embedding once per unique actual.
- Threshold calibration: hand-write two acceptable rewordings and two wrong-but-plausible answers per row on 3 of the 10 rows. Compute cosine of each against the reference. Pick the threshold at the midpoint of the two distributions. State the number, the model, and the calibration data in `SCORING.md`.

### The judge scorer

- Judge prompt is stored as a plain Python constant `JUDGE_SYSTEM` (or in `evals/judges/<name>.md`, if you prefer a file). Not inline in the test.
- Model call uses `temperature=0` and `response_format={"type": "json_object"}` (or equivalent for your provider). Judge returns `{"pass": bool, "reason": str}` and nothing else.
- **Run the judge three times per row** at `temperature=0` and log all three. This measures the judge's own non-determinism — if the same input produces different verdicts, that is a scorer stability signal you need to write down. If two of three agree, take the majority; report cases where the three disagree.
- Agreement floor: ≥ ~85% agreement with your hand-labels. If you cannot clear that after two rubric iterations, write in `SCORING.md` that "judge is not usable for this rubric at this cost tier" and move on — that is a valid finding.

### The comparison table

- One CSV or Markdown table in `SCORING.md`, one row per golden-set row you scored, columns as above.
- Aggregates below the table:
  - Total cost of the scoring run per family: rule (essentially $0), similarity (embedding calls × per-token price), judge (model calls × per-token price × 3 for the stability runs).
  - Total wall-clock per family.
  - Agreement rate with hand-labels: `judge_agreement = matches / 10`, similarly for `similarity` and `rule`.
- Highlight per-row disagreement — which pair of scorers said different things than your hand-label.

### `SCORING.md`

Three short sections:

1. **Rule.** What did the rule catch that other scorers missed? What did it miss that others caught? What is the per-row shape it is the right default for on this feature?
2. **Similarity.** Same three questions. Include the threshold calibration data — a small before/after table of `pass` cosines and `fail` cosines.
3. **Judge.** Same three questions. Include a summary of the judge's stability (of the 10 rows, how many produced the same verdict in all 3 stability runs). Include the final rubric and the calibration agreement number.

Then, in a final paragraph, name your **default scorer per field kind** for this feature, e.g. "IDs and enums: `exact`. Reference summaries: `similarity` at 0.75 under `text-embedding-3-small`. Tone-plus-factuality rubrics: judge, with the rubric in `evals/judges/summariser_v1.md`, calibrated 2026-08-02 at 9/10 hand-label agreement."

### Adoption

- For each of the 10 rows, edit the row's tolerance in `golden.jsonl` to use the scorer you concluded is best for that row's rubric. If the rule-based scorer was ~90% agreement and cost 0, use it. If the judge was the only scorer that agreed with your hand-labels *and* the row is important enough to justify the cost, use it.
- If you added judge dispatch to `evals/scoring.py`, add a `@pytest.mark.eval_slow` mark (or similar) so judge rows can run on a separate, slower CI lane. Judges are neither free nor deterministic; do not put them on the main PR gate without acknowledging the cost.

## Starter guidance

- Judge rubrics are the load-bearing artefact — the model does what you *specifically* wrote, not what you meant. A three-bullet rubric with one worked "pass" example and one worked "fail" example beats a two-paragraph one every time.
- Positional bias is real. If your judge prompt puts the actual before the reference and always uses "compare A to B," the judge tends to favour the one in position A. Randomise the order across rows if you find yourself worried about this — but do not over-engineer it in the first pass; note it as a concern in `SCORING.md` and move on.
- Do not use the same model family as the model under test for the judge. A `gpt-4o-mini` judging a `gpt-4o-mini` output has a subtle self-preference bias. `claude-3-5-haiku` judging `gpt-4o-mini`, or vice versa, is the cheap version of "use a different judge."
- If the rule you wrote scores 100% agreement, that is the right answer — do not manufacture judge complexity to feel more sophisticated. A row where a keyword check is the right answer is a row where the keyword check *is* the answer, and adopting the judge on that row is a strict downgrade.

## Acceptance criteria

- `labels.jsonl` exists with 10 hand-labels done before any scorer was run.
- `evals/scoring.py` now dispatches judge tolerances; `evals/judges/<name>.md` (or the constant equivalent) holds the judge prompt.
- The comparison table in `SCORING.md` lists per-row and aggregate results for the three families, including per-family cost and wall-clock.
- The similarity threshold and the judge rubric are calibrated on data and the calibration data is *shown* in `SCORING.md`, not just described.
- Judge stability is measured (three runs per row) and reported.
- `SCORING.md` concludes with a per-field-kind default scorer choice for this feature, and `golden.jsonl` has been updated to use the chosen scorer per row.
- You can, in one paragraph without looking at the artefact, explain to a colleague: "for our feature, `exact` handles X, `similarity` handles Y, and we reserve the judge for Z because …" — grounded in your numbers, not in a general "judges are best" instinct.

## Stretch goals

- **Rubric iterations, versioned.** Save each iteration of your judge rubric as `evals/judges/<name>_v1.md`, `_v2.md`, and log the agreement number for each. When a colleague asks "why is our judge rubric shaped this way?" you have the answer.
- **A two-judge quorum.** Run two judges from different model families and require agreement for a `pass`. Cost doubles; noise drops sharply. On the same 10 rows, does this improve agreement with hand-labels enough to justify the cost?
- **Break similarity on a negation row.** Add a row whose reference is `"Refunds are eligible for orders placed within 30 days."` and whose actual is `"Refunds are NOT eligible for orders placed within 30 days."`. Score both. Similarity will pass a factually wrong answer. Use this as the case study in `SCORING.md` for why similarity is not the right scorer for negation-sensitive rows — even if the numbers on your 10 general rows looked fine.
- **A one-cheap-check-before-judge policy.** Wrap the judge in a preflight: run the cheap rule first, only run the judge if the rule failed. On your 10 rows, this cuts judge calls by whatever fraction of your rows the rule handles cleanly. Report the savings.
- **Publish the judge as a small microservice.** If you have infra, host the judge behind an HTTP endpoint so multiple eval suites (and offline analyses) can call it without duplicating the rubric. This is a foreshadowing of the eval-track's shared-scoring infrastructure.
