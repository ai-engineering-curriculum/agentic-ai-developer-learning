# Quiz 01 — Minimal Evaluation and Regression Checks for LLM Features

**Covers:** chapters 1–5 of `mod-109-minimal-eval-and-regression-checks`.
**Format:** 12 multiple-choice questions. Pick the single best answer for each. Answer key with per-answer explanations is at the bottom.

Treat this as a self-check: open the chapters when you want to look something up, but try to answer first.

---

## 1. Chapter 1 says the `input` field of a golden-set row is "the exact input your feature receives at runtime," not the assembled prompt. Why does the distinction matter?

- A. Storing the assembled prompt guarantees reproducibility across prompt edits.
- B. Storing the assembled prompt would make the golden set stop being valid the moment you edited the prompt template — exactly what prompt / model swap PRs are about. The input is what the feature receives; the prompt is what the code under test builds from it.
- C. Providers reject golden-set rows whose assembled prompt exceeds the context window.
- D. Assembled prompts take more disk space than the runtime input.

## 2. Chapter 1 recommends roughly a 60 / 30 / 10 sampling mix across three slices, and then says "what is not negotiable is that the mix is written down." Which triple does the chapter name?

- A. 60 % adversarial edge cases, 30 % production traffic, 10 % behaviour coverage.
- B. 60 % production-representative (uniform random from logs, or dogfooding traffic for a pre-launch feature), 30 % behaviour coverage (one row per documented intent / field / language / refusal class), 10 % adversarial / regression rows (every production bug that was fixed with a prompt change, plus a handful of intentionally-hard edges).
- C. 60 % success cases, 30 % borderline cases, 10 % failure cases, chosen by the author's intuition.
- D. 60 % hand-written imagined inputs, 30 % synthetic LLM-generated inputs, 10 % real traffic.

## 3. Chapter 1's operational rule is "default to the strictest tolerance that would pass the correct answer." Why that direction?

- A. Loosening a tolerance later on a real failure is cheap; tightening later is a scavenger hunt through past passes that have been silently letting bad outputs through.
- B. Strict tolerance makes the eval cheaper in API cost.
- C. Strict tolerance is required by JSONL format.
- D. Strict vs. loose is a style choice; the direction does not matter.

## 4. Chapter 2's trade-off table compares rule-based, similarity, and LLM-as-a-judge scorers on cost, coverage, and calibration. Which row of the table does the chapter actually give?

- A. Rule-based: high cost, broad coverage, low calibration. Similarity: free, broad coverage, perfectly calibrated. Judge: cheap, narrow coverage, deterministic.
- B. Rule-based (exact / keyword / regex / schema / length): ~microseconds and zero API calls, narrow coverage (the check only asserts what the rule says), perfect calibration on the fields it covers, deterministic. Similarity: ~one embedding API call per side (cacheable), free-text semantic coverage, roughly 70–90 % human agreement on well-scoped tasks, no non-determinism once vectors are cached. LLM-as-a-judge: one full model call per row (often the most expensive line in the eval), covers anything you can describe in a rubric, calibration depends entirely on the rubric and the judge model, noisy at `temperature=0`.
- C. All three families have identical cost; what differs is the API endpoint they call.
- D. Judge is the cheapest because judge models are small; rule-based is the most expensive because it runs on every field.

## 5. Chapter 2 names two specific blind spots of similarity scorers that come up on a first eval. Which pair?

- A. UTF-8 encoding issues and emoji tokenisation.
- B. Negation and dates — `"The bug is fixed"` and `"The bug is not fixed"` have near-identical embeddings, so a similarity check can pass a factually wrong answer whose wording matches the reference; and threshold drift, where swapping the embedding model (e.g. `text-embedding-3-small` → `text-embedding-3-large`, or OpenAI → Voyage) silently invalidates every threshold in the eval. Pin the model and treat the swap as an eval-versioning event.
- C. Latency jitter and provider rate-limiting.
- D. Similarity scorers have no blind spots the chapter identifies.

## 6. Chapter 2 is explicit about the minimum calibration procedure before shipping an LLM-as-a-judge check. Which description matches the chapter?

- A. Run the judge once on a new example and accept whatever it says.
- B. Score 20 rows by hand on the same rubric, run the judge on those same 20 rows, and compute the agreement rate. If the judge disagrees with you on more than 2–3 rows out of 20 (~85 % agreement floor), the judge is not usable as written — rewrite the rubric (tighter definitions, one worked `pass` example, one worked `fail` example) and try again. Save the 20-row calibration set with the judge prompt and rerun calibration every time you touch the rubric or swap the judge model.
- C. Trust the judge if its self-reported confidence is above 0.9.
- D. Hand-label 10,000 rows and train a bespoke classifier.

## 7. Chapter 2's "rough decision tree" says to ask, per field, in order: value / shape / substring set → free-text meaning that can be phrased many ways → multi-property quality judgement. Which per-family default does the chapter draw from that order?

- A. Default to judge. Judges are the most flexible and the most impressive.
- B. Default to similarity, because embeddings "understand meaning" and covering more with one scorer is cheaper than mixing families.
- C. Prefer the cheapest family that can express what "correct" means for the field. If exact-match works, use exact-match; only fall through to similarity when the field is free-text semantic; only reach for a judge when nothing cheaper covers what "correct" means and the extra cost and noise are worth it. Score different dimensions of one row with different families — a structured-output feature might use `schema` for shape, `exact` for the ID field, `keyword` for required nouns, and `similarity` for the free-text `explanation`.
- D. Default to whichever family your team already has code for. Consistency across rows beats per-field fit.

## 8. Chapter 3's `@pytest.mark.parametrize("row", ROWS, ids=[r["id"] for r in ROWS])` is specifically load-bearing. Why?

- A. It makes pytest run the rows in parallel automatically.
- B. It surfaces your row IDs verbatim in the pytest output, the JUnit XML, `pytest --last-failed`, and CI logs — so a failure reads as `test_golden_row[sum-014]` rather than `test_golden_row[7]`. This is exactly why chapter 1 insisted the per-row `id` be stable: the ID is the primary debugging key everywhere downstream.
- C. It is required by `pytest-xdist` to discover the tests.
- D. It is cosmetic and only affects console colour output.

## 9. Chapter 3 says "a common early mistake is defining pass as '100 % of rows passed.'" What is the chapter's pragmatic pass criterion for a first eval, and why?

- A. 100 % of rows pass. Anything less is unacceptable.
- B. Per-row, the row's `flake_budget` (default `1 of 1`) is met; suite-level, the number of passing rows is at least the pass count on the branch's merge base (the baseline) minus a small tolerance of 0 or 1. Regressions below baseline block the PR; a new failure on a row that was passing on `main` is a regression, always. The reasoning: once the set is ~30 rows, one flaky row will keep you from ever hitting 100 %, the team gets used to "green with known flakes," and that trains them to ignore new failures. "No new regressions" is the honest target.
- C. A rolling three-day average score above 0.9 on a hosted dashboard.
- D. 50 % of rows pass; the other 50 % are always-failing adversarial controls.

## 10. Chapter 4 warns about naïvely updating `BASELINE.json` after every trunk run. What specifically can go wrong, and what is the protection the chapter recommends?

- A. The JSON file can become corrupted by concurrent writes; use a database instead.
- B. If a bad change lands on trunk and updates the baseline, the new baseline is now a *lower* score — PRs that would have caught the regression now pass against the degraded baseline. The protection: only update `BASELINE.json` when the trunk run clears a hard-coded floor (e.g. "≥ 45/50" for a 50-row set), a hard bottom that the baseline can never sink below without a human-approved threshold change.
- C. Baselines must be stored outside the repo entirely; committing them is forbidden.
- D. Nothing goes wrong — any trunk run can safely update the baseline.

## 11. Chapter 4 insists the diff step must refuse to compare two runs whose `golden_set_sha` differs. Why is refusing the right behaviour, not a bug?

- A. Because the eval harness cannot load two different golden sets into memory at once.
- B. Because two runs against *different* golden sets are not comparable — a changed row (added, deleted, re-toleranced) means the pass/fail counts refer to different sets of rows. Printing `baseline is stale; regenerate on trunk` tells the developer the diff is not meaningful; silently diffing anyway would produce numbers that mislead.
- C. Because hashing JSONL files is slow and the refusal is a cost-optimisation.
- D. Because golden sets are meant to be immutable and the hash is a tamper-signature.

## 12. Chapter 4 calls the model-silently-rolls-forward case the most easily missed regression source. Which protection does it recommend, and what is the implied routing when it fires?

- A. Pin nothing and let the provider manage versions — the most-recent snapshot is always the best.
- B. Read the exact `model` string used at request time (either from the response's `model` echo or from a captured `feature.CURRENT_MODEL`) and compare it against the pinned string in your config. If they differ, halt the eval with a message like `model changed from gpt-4o-mini-2024-07-18 to gpt-4o-mini-2024-11-20 — this is a version-pin issue, not a regression; owner: @<maintainer of version-pin config>`. Route the failure to the pin-maintainer, not the PR author. When the pin has to change, ship the swap as its own PR whose diff is the regression report for that change.
- C. Disable the eval during the days a model deprecation is scheduled.
- D. Only trust frontier models; cheap-tier models should not be pinned.

---

## Answer key

1. **B.** Chapter 1 (*"What a golden-set example is"*) spells out that `input` is the runtime input to the feature — `{ticket_id, question}` if the feature takes that shape, the raw user message if it takes that shape — and names the failure mode of storing the assembled prompt verbatim: "Storing the assembled prompt would make your golden set stop being valid the moment you edited the prompt template." A, C, D are not claims the chapter makes.
2. **B.** Chapter 1 (*"Sampling: defend the choice, in one paragraph"*) enumerates the three slices exactly this way — production backbone, behaviour coverage, adversarial / regression — and says "Aim for roughly a 60/30/10 mix … Adjust to your feature. What is not negotiable is that the mix is written down." A inverts the ratio, C and D are explicit anti-patterns from the chapter ("the examples the author remembered, which turn out to be the easy ones").
3. **A.** Chapter 1 (*"Tolerance: how much wiggle room each row gets"*, "Two operational rules") puts this verbatim: "Default to the strictest tolerance that would pass the correct answer. Loosening later on a real failure is cheap; tightening later is a scavenger hunt through past passes."
4. **B.** Chapter 2 opens with this exact table (*"The mental model: three families, three trade-off shapes"*). Rule-based is near-free, deterministic, narrow coverage, perfect calibration where it applies. Similarity is one cacheable embedding call per side with ~70–90 % agreement on well-scoped tasks. Judge is one full model call per row, broadest coverage, calibration depends on the rubric, and is noisy — the "non-determinism" column the chapter adds to the table.
5. **B.** Chapter 2 (*"Two failure modes to know"* under similarity scorers) names both of these exactly — the negation example is the chapter's and traces back to mod-104 chapter 2; the threshold-drift warning says pin the embedding model and treat swapping it as an eval-versioning event. A, C, D are not claims the chapter makes.
6. **B.** Chapter 2 (*"The minimum calibration procedure for this module"*) numbers these four steps exactly: 20 hand-scored rows, run the judge on the same 20, compute agreement, rewrite the rubric if agreement drops below ~85 %, save the calibration set and rerun it on every rubric or judge-model change. The chapter is explicit that a judge you have not calibrated at all is "an uncalibrated judgement replacing an uncalibrated judgement."
7. **C.** Chapter 2 (*"A rough decision tree"* and the two consequences at the top of the chapter) says this in two places: "Prefer the cheapest family that can express what 'correct' means for the row" and "Score the different dimensions of a row with different families." A, B, D are all failure modes the chapter warns against — defaulting to judge, defaulting to similarity, or defaulting to whichever family you already have.
8. **B.** Chapter 3 (*"The minimal pytest file"*, note on `ids=[r["id"] for r in ROWS]`) says exactly this: "surfaces your row IDs from chapter 1 in pytest's output. `test_golden_row[sum-014]` is what appears in the JUnit XML, in the CI log, and in `pytest --last-failed`. This is why row IDs were made stable in chapter 1."
9. **B.** Chapter 3 (*"What 'the eval passed' should mean, exactly"*) names the per-row flake-budget check and the suite-level "baseline minus small tolerance" criterion, then explains the reasoning: "Once your set is ~30 rows, one flaky row will keep you from ever hitting 100 %, and the team will get used to green-with-known-flakes — which trains them to ignore new failures. … 'No new regressions' is the honest target."
10. **B.** Chapter 4 (*"A baseline: what 'known-good' points at"* → *"Two failure modes to design around"*) describes this exactly: "If a bad change lands on main and updates the baseline, the new baseline is now the lower score, and PRs that would have caught the regression now pass. Guard against this by only updating `BASELINE.json` when the trunk run passes its own hard-coded floor."
11. **B.** Chapter 4 (*"Two failure modes to design around"*, golden-set-drift bullet) says: "When you legitimately edit the golden set (add rows, fix tolerances), old baselines are no longer comparable. The diff step should refuse to compare across different `golden_set_sha` values … That refusal is the correct behaviour, not a bug — the eval is telling you the comparison is not meaningful."
12. **B.** Chapter 4 (*"The 'the model changed under us' case"*) describes both pieces: the pre-eval assertion that compares the runtime model string against the pinned string with the exact error-message shape above, and the "ship the pin change as its own PR whose diff is the regression report for that change" habit. The routing goes to the pin maintainer, not the PR author — because no PR touched the model, the drift is in the ambient dependency.
