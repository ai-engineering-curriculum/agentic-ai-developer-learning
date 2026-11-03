# Exercise 02 — Pytest eval in CI

Paired with [chapter 3 — running the eval in pytest and CI](../03-running-the-eval-in-pytest-and-ci.md).

**Estimated effort:** 2 hours.

## Objective

Take the golden set you built in exercise 01 and turn it into a pytest-based eval suite that: (a) runs each row as a parametrised test with your row IDs visible in the output; (b) skips by default unless an explicit `RUN_EVALS=1` env var is set; (c) fails cleanly with a message a developer can act on; and (d) is wired into CI on a hypothetical PR so that a prompt change is gated by the eval outcome — with the eval marked as required for merge and the diff summary posted back to the PR.

You are not building history-tracking here (that is exercise 04). You are building the pass/fail gate the rest of the module rests on.

## Problem statement

Starting from your `evals/<feature>/golden.jsonl` and the feature's runtime entry point:

1. Implement `evals/scoring.py` — the `score_row(actual, row)` function from chapter 2. It dispatches by `tolerance.kind`, returns `{"id", "pass", "checks": [...]}` with the per-check trail, and handles at least: `exact`, `case_insensitive`, `keyword`, `regex`, `schema`, `length`, `similarity`. Judge dispatch can be a stubbed `NotImplementedError` for now — exercise 03 fills it in.
2. Implement `evals/<feature>/test_golden.py` — the parametrised pytest file from chapter 3. Use `ids=[r["id"] for r in ROWS]` so failures surface as `test_golden_row[sum-014]`.
3. Implement `evals/<feature>/conftest.py` — skip the `@pytest.mark.eval` tests unless `RUN_EVALS=1` is set, and skip the whole file cleanly if required API keys are missing.
4. Register the `eval` mark in `pyproject.toml` (or `pytest.ini`).
5. Add a CI workflow (`.github/workflows/eval-<feature>.yml` for GitHub Actions, or the equivalent in your CI provider) that:
   - Triggers on pull_request with a `paths:` filter matching the feature's source and eval directories.
   - Runs the suite with `RUN_EVALS=1 pytest -m eval -n auto --junitxml=eval-results.xml` on a dedicated CI API key stored in secrets.
   - Uploads the JUnit XML as an artifact.
   - Posts a summary comment back to the PR: pass/fail counts, and the specific `row_id + failing sub-check` lines for regressions.
6. **Demonstrate the gate works** by opening (or simulating) a PR that deliberately breaks the feature — flip a keyword in the system prompt, comment out a schema field, downgrade the model to a weaker one — and showing that the eval fails on the expected rows and that the CI job is red.

If you do not have a CI provider available at the moment, meet the same criteria with a local `Makefile` / `justfile` / shell script whose `make eval-ci` target does the same thing (run eval, generate JUnit XML, produce a PR-comment-shaped Markdown summary) and paste the output into a hypothetical PR description. Do not skip the CI section entirely — the point of the exercise is that the gate is real, not that it runs on someone else's server.

## Requirements

### The scoring layer

- `score_row(actual, row)` returns the shape from chapter 2: `{"id", "pass", "checks": [{"field", "kind", "pass", ...extras...}]}`.
- Each dispatched check preserves *why* it failed in the check's dict — the missing keyword list, the failed regex, the schema-validation error message, the similarity score and the threshold, the length that exceeded the max. Do not lose the reason on the floor.
- **Schema check short-circuits.** If a row has a `schema` tolerance and the response is not valid JSON, the row fails with the schema check first and no other checks are attempted. Malformed JSON is a single, actionable failure — do not report five downstream sub-check failures that are all really "no JSON."
- If a row has `flake_budget`, run the model call `of_runs` times and pass the row if `passes_required` of the individual scored results pass. Log every attempt to the pytest output; do not silently retry.

### The pytest layer

- `evals/<feature>/test_golden.py` is a single parametrised test. One row = one test. No hidden fixtures that call the model multiple times per parameter.
- The `ids=[r["id"] for r in ROWS]` argument to `parametrize` is present. Failed test IDs in `pytest --last-failed` correspond exactly to row IDs.
- Failure message includes at minimum: `row_id`, the failing sub-check `field:kind`, the row's `input`, the row's `expected`, and the model's `actual`. A colleague reading the CI log can start debugging without opening any other file.
- Tests are marked `@pytest.mark.eval`. The mark is registered in `pyproject.toml`.
- `RUN_EVALS=1` is required to run any eval test locally; without it, all eval tests skip with a clear reason.

### The CI job

- The workflow's `paths:` (or equivalent) filter runs the job only on PRs that touch the feature's source, the eval directory, or shared prompt / provider / SDK-version configuration.
- API keys are pulled from CI secrets. **No API keys anywhere in the workflow YAML or the repo.**
- The job uses `pytest-xdist` (`-n auto`) so a 30-row eval runs in under a minute on the CI runner.
- The job cancels superseded runs (concurrency group per branch, `cancel-in-progress: true`).
- The job publishes the JUnit XML as an artifact and either fails the job on any failing eval or emits a check with the failure count so branch protection can require it.
- The PR summary comment (or check-run summary) lists at least: total passed, total failed, and the `row_id + failing field:kind` line per regression. The template from chapter 4's example is fine — you do not need the baseline diff yet, just the current-run summary.
- The job is marked **required for merge** on the protected branch. If your test repo has no protected branch, document how you *would* mark it in a short `CI.md`.

### The proof-it-works demo

- Open a "break the feature" PR (branch + commit + push, even if you never merge). The break must be one line so a reviewer can eyeball it:
  - **A prompt-only break:** flip a required instruction ("respond in JSON" → "respond however you like").
  - **A model-only break:** swap the pinned model to a much weaker one.
  - **A schema-only break:** rename a required field in the response schema so downstream checks fail.
- Screenshot (or paste) the CI output. The failing rows should be exactly the ones you would expect from the change (a prompt break might fail 8 rows, a model break might fail 3, a schema break might fail every row via short-circuit).
- Add a short note to a `DEMO.md` explaining what the break was and what failed. This is the artefact you show a colleague when you say "our eval works."

## Starter guidance

- If you have not used pytest parametrisation before: <https://docs.pytest.org/en/stable/how-to/parametrize.html>. The `indirect` and `pytest_generate_tests` machinery is not needed here — plain `@pytest.mark.parametrize("row", ROWS, ids=...)` is enough.
- For CI-only API keys, provider dashboards let you create a scoped key with a monthly spending cap. Set the cap to a small dollar figure — an eval that runs away because of a bug should fail loudly, not silently drain the budget.
- For posting PR comments from GitHub Actions, the simplest path is `gh pr comment $NUMBER --body-file summary.md` from a step that runs after the eval; if you use GitLab / CircleCI / Buildkite, look up the equivalent (see `resources.md`).
- Do not roll your own JUnit XML writer — `pytest --junitxml=` produces the file for you. A tiny script can parse it into a Markdown summary; even simpler, use `pytest-md-report` for a ready-made Markdown output.
- If your CI runner is on shared infra with a corporate egress proxy or Anthropic / OpenAI endpoints blocked, that is a *real* discovery. Fix it before the eval matters, not after.

## Acceptance criteria

- Running `pytest` locally does **not** run eval tests. Running `RUN_EVALS=1 pytest -m eval` runs the whole golden set.
- Every row of the golden set corresponds to one parametrised test with the row ID as the test ID. `pytest --last-failed` after a failure reruns *only* the failing rows.
- A failing row's pytest output tells a reader (a) which row failed, (b) which sub-check failed, (c) the row's input, (d) the expected, (e) the actual. All five are present without editing any files.
- The CI workflow exists, runs on relevant PRs, uses a secret API key, uploads JUnit XML, and posts a comment or check-run summary listing failing `row_id + field:kind`.
- The eval job is marked as a required merge check (or the `CI.md` documents how it would be).
- The "break the feature" demo PR (or its saved artefact) shows the CI job going red on exactly the rows a reader would expect from the break.
- The full run cost is under a dollar and the wall-clock (with `-n auto`) is under 3 minutes. If it is not, either your set is too big, your row inputs are too long, or your dispatch is calling the model more times per row than it should.

## Stretch goals

- **Add a slow-path lane.** Split your eval into `@pytest.mark.eval` (fast, all rule + similarity) and `@pytest.mark.eval_slow` (any judge-scored rows you have or will have). Run the fast lane on every PR and the slow lane on a nightly cron. This is the shape you will want by the end of exercise 03.
- **Cost log.** Sum the `usage` blocks across every model call in the run and print `TOTAL COST: $0.14` at the end of the pytest output. This becomes chapter 4's cost field in the manifest.
- **Coloured diff comment.** Post the PR comment as a proper Markdown table with checkmark / cross emoji per row (only if the user explicitly permits emoji in comments; check with the reviewer for your team's conventions). Do not emoji-in-code — just in the CI comment.
- **Fail on unexpected pass.** If a row is marked as a known failure (`{"expected_status": "fail"}`) and it unexpectedly passes, fail the eval and force the developer to remove the marker. Nothing regresses trust in an eval faster than a known-fail row that quietly starts passing and the marker never gets removed.
- **A retry-on-transient-error wrapper.** Provider APIs occasionally throw 5xx. Wrap the model call in a bounded retry (3 attempts, exponential backoff) that logs the retries visibly. Do *not* use `pytest-rerunfailures` for this — it hides the retries and pollutes your signal.
