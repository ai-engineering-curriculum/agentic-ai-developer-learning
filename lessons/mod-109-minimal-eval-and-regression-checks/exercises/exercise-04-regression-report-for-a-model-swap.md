# Exercise 04 — Regression report for a model swap

Paired with [chapter 4 — tracking regressions over time and routing them to the right developer](../04-tracking-regressions-over-time.md).

**Estimated effort:** 90 minutes.

## Objective

Run your golden set on two different models (or two different prompt versions) — one your current baseline, one a plausible replacement — and produce the regression report chapter 4 described: a persisted run record per model, a diff between them, and per-row attribution to the specific change that caused each regression. By the end of the exercise you can defend a model-swap decision from a real diff, not a "seemed fine locally," and you have the mechanical shape that will let every future model swap ship as its own reviewable PR.

## Problem statement

Two variants of the same shape — pick whichever fits your feature:

- **A model swap.** Pin your current model in `src/<feature>/prompt.py` (e.g. `MODEL = "gpt-4o-mini-2024-07-18"`) as `MODEL_BASELINE`. Add a `MODEL_CANDIDATE` — a cheaper, older, newer, or different-family model that is a plausible swap (e.g. `gpt-4o-mini` → `gpt-4.1-mini`, or `claude-3-5-haiku` → `claude-3-haiku`, or `gpt-4o-mini` → `claude-3-5-haiku`). See `resources.md` for the current model reference pages; pick real, currently-available snapshot IDs.
- **A prompt swap.** If you cannot spare the budget for a model swap, keep the same model and vary the prompt. `PROMPT_BASELINE = "…v14"` and `PROMPT_CANDIDATE = "…v15"` where v15 is a deliberate one-line change to the system prompt.

Then:

1. **Run the eval twice** against the same golden set — once with the baseline pinned, once with the candidate. Save each run as its own run record: JSONL of per-row outcomes plus a manifest JSON with git SHA, author, timestamp, feature, golden-set hash, `code_versions.model` (or `.prompt_version`), and totals (rows, passed, failed, `cost_usd`).
2. **Diff the two runs** using the shape from chapter 4: newly-failing rows (rows that passed on baseline and fail on candidate), newly-passing rows (fail on baseline, pass on candidate), and still-failing rows (fail on both). For each row, report the specific failing sub-check and the score / values that caused the fail.
3. **Attribute each newly-failing row** to the change that caused it — for this exercise, that is the model swap (or the prompt swap) itself. Every regression line names *what changed*: `sum-014 must_mention missing ['Globex'] — model gpt-4o-mini-2024-07-18 -> gpt-4.1-mini-2025-04-14`.
4. **Render a PR-comment-shaped Markdown report** (`REPORT.md`) with: header (baseline vs. candidate identifiers), summary (X/N vs. Y/N), the three lists (newly failing / newly passing / still failing), a cost + wall-clock table (baseline vs. candidate), and a one-paragraph recommendation (ship / do not ship / needs prompt work first).
5. **Add a version-pin assertion** to your CI job: on eval start, assert that the `model` string used at request time equals the pinned `MODEL` constant, and fail the run with an actionable error if they diverge. This is the "the model changed under us" guard from chapter 4. Confirm it fires by intentionally swapping the pin without swapping the runtime and rerunning the eval — the assertion must halt the run.

You are not adding baseline-history bookkeeping (a `BASELINE.json` that auto-updates on merges) in this exercise. Assume the baseline is whichever model the current `MODEL_BASELINE` constant points at; the exercise is about the diff shape and attribution, not the trunk-baseline update workflow.

## Requirements

### The run records

- One `evals/<feature>/runs/<timestamp>_<label>.jsonl` per run, with one line per scored row: `{"row_id": "...", "pass": bool, "checks": [...], "latency_ms": int, "usage": {...}}`.
- One sibling `<timestamp>_<label>.manifest.json` per run, with the shape from chapter 4: `run_id`, `started_at`, `finished_at`, `git_sha`, `git_branch`, `author`, `trigger`, `feature`, `golden_set_sha`, `code_versions.{prompt_version, model, temperature, seed, sdk_versions}`, `totals.{rows, passed, failed, cost_usd}`.
- `golden_set_sha` is a hash (e.g. SHA-256, first 16 hex chars) of the golden JSONL. Compute it in the run script; do not maintain it by hand.
- `cost_usd` is a real number summed from the `usage` fields × the model's per-token pricing at the time of the run. Store the per-token prices you used at the top of the script (or in `pricing.py`) with a `# priced_at: 2026-08-02` comment. When prices change, update the constant.

### The diff

- A `python evals/diff.py --baseline <manifest> --candidate <manifest>` (or equivalent) CLI that:
  - Refuses to diff across mismatched `golden_set_sha` values with a clear message ("baseline was on golden set X; candidate is on Y; regenerate baseline first").
  - Reads both JSONL files, joins on `row_id`, and produces the three lists.
  - Emits the Markdown report to stdout (or `REPORT.md`) with the shape from chapter 4.
- Diff lines for regressions name the *specific* failing sub-check and, where possible, the numeric value: `similarity 0.61 < 0.75`, `missing ['Globex']`, `date is not a string`. Do not paper over the reason.

### The attribution

- Every newly-failing row line in the report ends with the specific change identifier from the manifest diff: `model gpt-4o-mini-2024-07-18 -> gpt-4.1-mini-2025-04-14` or `prompt summariser@v14 -> summariser@v15`.
- If only one variable moved (which is the case for this exercise), attribution is one line. If more than one moved, the report should list every changed variable — this is a general failure mode to guard against and a good place to sanity-check that only what you intended changed.

### The version-pin assertion

- In `evals/<feature>/conftest.py` (or as a pytest fixture), read the model string the runtime actually used (either by intercepting a client call or by exposing a `feature.CURRENT_MODEL` constant that is set on module import) and assert it equals `MODEL_BASELINE` (or the appropriate pinned constant for the run).
- On divergence, fail with a message like: `model changed from gpt-4o-mini-2024-07-18 to gpt-4o-mini-2024-11-20 — this is a version-pin change, not a regression; owner: @<person or team responsible for the pin>`.
- Demonstrate it fires. Change the pinned constant to a string the runtime does not use, rerun, and confirm the eval halts before running any row.

### The recommendation

The one-paragraph recommendation at the end of `REPORT.md` should:

- State the decision ("ship the candidate," "do not ship," "not decidable without more work").
- Ground the decision in the numbers you produced ("baseline 47/50, candidate 44/50, cost per run drops from $0.14 to $0.03, three newly failing rows are all on the Portuguese-language coverage rows").
- Name the follow-up ("open a prompt PR to add pt-BR examples to the few-shot block, rerun the eval, expect to recover sum-014, sum-022, sum-031"). If the recommendation is "ship anyway," name the trade-off explicitly ("we accept sum-041's regression because it is on the adversarial slice and the cost savings pay for a bigger set next quarter") — do not paper over it.

## Starter guidance

- Model IDs and pricing move fast. Get the current snapshot IDs and per-token prices from the provider's official reference (OpenAI: <https://platform.openai.com/docs/models>, Anthropic: <https://docs.anthropic.com/en/docs/about-claude/models>) at run time — do not copy from an older exercise or a blog post.
- The `golden_set_sha` refusal is where the "one bad refactor of the golden set silently invalidates every past run" bug gets caught. It is worth writing before you need it.
- For the version-pin assertion, one clean shape is a small `feature.get_model()` accessor whose returned string is captured at request time and asserted on. Do not intercept at the SDK level for this exercise; it is over-engineering.
- If you swapped models and the eval score *increased*, do not celebrate yet. Confirm the candidate did not accidentally get a shorter prompt, a different sampling parameter, or a cached response — the candidate must run with exactly the same inputs as the baseline, only the model differing. Chapter 4's "one variable at a time" rule applies here too.

## Acceptance criteria

- Two run records exist for the same golden set, with different `code_versions.model` (or `.prompt_version`). Both records are valid JSON and have the manifest fields above.
- The diff CLI produces a Markdown report listing newly-failing, newly-passing, and still-failing rows, each with the specific failing sub-check named.
- Every newly-failing row line includes the attribution string (the changed variable and its before → after).
- The diff refuses to compare across mismatched `golden_set_sha` values with a clear error, and this refusal has been triggered at least once (deliberately) to confirm it works.
- The version-pin assertion in CI halts the run if the runtime model does not match the pinned constant, and you have demonstrated this by an intentional mismatch.
- `REPORT.md` closes with a one-paragraph recommendation that a reader could act on without asking you a follow-up.
- Total cost of both runs is under two dollars combined and wall-clock is under five minutes per run.

## Stretch goals

- **A three-way diff.** Add a `MODEL_CHALLENGER` (a third model, e.g. the frontier tier). Run all three and produce a three-column report. Frame the decision as ordered: "cheapest that clears the threshold." This is the mod-108 model-selection discipline crossed with the mod-109 eval discipline.
- **Bisect on the still-failing list.** For one still-failing row, walk the eval's run history (from exercise 02 CI runs or from `evals/<feature>/runs/`) with the `when_did_row_start_failing` script from chapter 4. Add the "introduced by …" attribution to the row's still-failing line. This is the shape that keeps old failures from becoming nobody's problem.
- **Wire the diff into CI.** Have the CI job automatically diff the current run against the previous *passing* trunk run and post the diff back to the PR as a comment. This is one small step from the exercise 02 pipeline; do it once and you have chapter 4's core loop wired up end to end.
- **Regression report as a slash-command trigger.** On a PR, add a comment `/rerun-eval` that re-triggers the eval on demand. Useful for the "the eval flaked; rerun before merge" case without forcing an empty commit.
- **Per-tolerance-family regression breakdown.** Alongside the per-row diff, aggregate by tolerance kind: "of the 3 regressions, 2 are `similarity` and 1 is `must_mention`." This is a much earlier signal that a specific *scorer* is drifting (e.g. the similarity threshold is now too tight for the candidate model's slightly different phrasing) — the fix might be to recalibrate the threshold rather than reject the model.
