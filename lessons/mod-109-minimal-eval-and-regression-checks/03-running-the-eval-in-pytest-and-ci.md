# Chapter 3 — Running the eval in pytest and CI

You now have a golden set (chapter 1) and per-row scorers (chapter 2). This chapter puts them where they earn their keep: as a pytest suite that runs the whole set on every prompt / model change, and as a CI job that turns "the score dropped" into a red build that blocks the PR. That is the mechanism the whole module is building towards. Without it, the golden set is a nice-looking JSONL file and the feature still regresses silently.

## Why pytest, and why not a bespoke runner

You could write a custom eval CLI. Every third team eventually does. The reason the mod-109 default is `pytest` is straightforward:

- **Every developer on the team already knows how to run one test.** `pytest tests/eval/test_summariser.py::test_row[sum-014]` is a skill they have; a bespoke `python -m evals run --feature summariser --row 14 --verbose` is a skill you have to teach.
- **You get the machinery for free.** Parametrisation, selection by expression, `-x` fail-fast, `-k` filter, `--last-failed`, `-n` xdist parallelism, JUnit XML output, HTML reports, coverage, marks — none of these are worth reinventing.
- **CI already knows what a pytest failure means.** Every CI provider on the market renders pytest output natively. You do not need to teach the CI what "eval failed" means; a red X on a test is the universal signal.

Two things pytest does *not* do out of the box that you have to bolt on:

- It does not know that some tests are expensive and non-deterministic. You mark those and skip them by default; see the *Marks* section below.
- It does not remember scores over time. That is [chapter 4's](04-tracking-regressions-over-time.md) job — the pytest job's job is to answer "does the current change pass the current threshold."

## Directory shape

Colocate the eval with the feature it evaluates. A typical layout:

```
src/
  summariser/
    __init__.py
    prompt.py
    run.py                 # def summarise(article: str) -> dict
evals/
  summariser/
    golden.jsonl           # from chapter 1
    calibration/           # 20 hand-scored rows for the judge (chapter 2)
      set.jsonl
      README.md            # who scored, when, and against which prompt version
    conftest.py
    test_golden.py
```

Keeping `evals/summariser/` next to `src/summariser/` (rather than under `tests/`) makes two boundaries clearer:

- **Evals are not unit tests.** They call live APIs, cost money, and have flakiness the ordinary test suite does not. Storing them under `tests/` invites CI configurations that run them on every push. Storing them under `evals/` invites CI configurations that run them on the changes that matter (see *Gating in CI* below).
- **Each feature owns its own eval.** When the summariser is deprecated, its eval is deleted with it. A monolithic `tests/eval/` folder outlives the features it once tested and rots.

## The minimal pytest file

Everything you need is one parametrised test. The `test_golden.py` for a summariser:

```python
# evals/summariser/test_golden.py
from __future__ import annotations

import json
from pathlib import Path
from typing import Any

import pytest

from src.summariser.run import summarise
from evals.scoring import score_row   # the score_row from chapter 2

GOLDEN_PATH = Path(__file__).parent / "golden.jsonl"


def _load_rows() -> list[dict[str, Any]]:
    with GOLDEN_PATH.open() as f:
        return [json.loads(line) for line in f if line.strip()]


ROWS = _load_rows()


@pytest.mark.eval
@pytest.mark.parametrize("row", ROWS, ids=[r["id"] for r in ROWS])
def test_golden_row(row: dict[str, Any]) -> None:
    actual = summarise(**row["input"])
    result = score_row(actual, row)

    assert result["pass"], _format_failure(row, actual, result)


def _format_failure(row: dict, actual: Any, result: dict) -> str:
    failed = [c for c in result["checks"] if not c["pass"]]
    return (
        f"row {row['id']} failed on: {', '.join(c['field'] + ':' + c['kind'] for c in failed)}\n"
        f"input: {row['input']}\n"
        f"expected: {row['expected']}\n"
        f"actual: {actual}"
    )
```

That is the whole load-bearing file. Notes:

- `ids=[r["id"] for r in ROWS]` — surfaces your row IDs from chapter 1 in pytest's output. `test_golden_row[sum-014]` is what appears in the JUnit XML, in the CI log, and in `pytest --last-failed`. This is why row IDs were made stable in chapter 1.
- `@pytest.mark.eval` — a mark you define in `pyproject.toml` (see next section) so `pytest` alone does *not* run these on every developer save. `pytest -m eval` runs them; `pytest -m "not eval"` runs everything else.
- The failure message includes the row ID, the failing sub-checks, and the actual / expected. A one-line assertion `assert result["pass"]` gives you `AssertionError: assert False` and forces the developer to hunt through logs — do not do that. The extra six lines pay for themselves every failure.

## `conftest.py` — where the fixtures live

Anything the tests need that is not per-row belongs in `conftest.py`:

```python
# evals/summariser/conftest.py
import os
import pytest


def pytest_collection_modifyitems(config, items):
    if not os.environ.get("RUN_EVALS"):
        skip_eval = pytest.mark.skip(reason="set RUN_EVALS=1 to run eval suite")
        for item in items:
            if "eval" in item.keywords:
                item.add_marker(skip_eval)


@pytest.fixture(scope="session", autouse=True)
def _require_api_keys():
    missing = [k for k in ("OPENAI_API_KEY",) if not os.environ.get(k)]
    if missing:
        pytest.skip(f"missing env vars for eval: {missing}", allow_module_level=True)
```

Two things this does:

- **Skip evals unless explicitly asked to run them.** Local developers running `pytest` should not get billed for a run they did not intend. `RUN_EVALS=1 pytest -m eval` is the explicit opt-in; CI sets that variable on the jobs where evals should run (see *Gating in CI*).
- **Fail fast when credentials are missing.** Nothing is more annoying than a 50-row eval that fails all 50 rows with the same "no API key" traceback. Skip cleanly at the module level with a clear message.

Register the `eval` mark in `pyproject.toml` (or `pytest.ini`) so `-m` does not emit a `PytestUnknownMarkWarning`:

```toml
[tool.pytest.ini_options]
markers = [
    "eval: golden-set regression suite; runs live API calls (opt in with RUN_EVALS=1)",
]
```

## Running a subset — the developer inner loop

The inner loop when a row fails, in order of speed:

```bash
# rerun only the last-failed rows
RUN_EVALS=1 pytest -m eval --last-failed

# rerun one specific row
RUN_EVALS=1 pytest -m eval "evals/summariser/test_golden.py::test_golden_row[sum-014]"

# rerun every row whose id contains "merger"
RUN_EVALS=1 pytest -m eval -k merger

# run the whole eval with a verbose failure format
RUN_EVALS=1 pytest -m eval -vv --tb=short
```

Two habits worth building:

- **`--last-failed` first.** After the initial run, this is the fastest loop — you edit the prompt, you rerun only the rows the last run failed on. Do not sit through the 45 passing rows to see if the 5 you were fixing pass now.
- **`-n auto` when the eval is I/O-bound**, which it always is (network calls to the model). `pip install pytest-xdist` and `pytest -m eval -n auto` cuts a 5-minute eval down to sub-30-seconds on any reasonable laptop, and CI runs get proportionally cheaper.

## Determinism, retries, and flakiness

LLM outputs are non-deterministic even at `temperature=0`. Your eval will occasionally flap on a row that "usually" passes. Two knobs to know:

- **Pin sampling.** `temperature=0`, `top_p=0` (or default), a fixed `seed` if your provider exposes one. This does not eliminate variance — providers are allowed to change model behaviour and hardware fanout — but it removes the variance you can remove.
- **Retry with intent, not blindly.** If a row is stochastic *and* important, run it `N` times and require `k of N` passes rather than `1 of 1`. This is a tolerance, not a workaround — write it into the row explicitly:

```json
{
  "id": "sum-014",
  "input": {...},
  "expected": {...},
  "tolerance": {"reference_summary": {"kind": "similarity", "min_cosine": 0.75, "model": "text-embedding-3-small"}},
  "flake_budget": {"passes_required": 2, "of_runs": 3}
}
```

The scoring layer of chapter 2 grows one line: if `flake_budget` is present, run the model call `of_runs` times, score each, pass if `passes_required` of them pass. Do this on rows where the underlying task is genuinely non-deterministic (a creative rewrite, a multi-choice with ties). Do *not* do it on rows that fail intermittently for other reasons — a rule check that fails 1 in 10 runs is telling you the prompt is wrong, not that the check needs retries.

**Do not use `pytest-rerunfailures` as a substitute for a real flake budget.** It re-runs failed tests silently and gives you a passing eval that hides a real signal. If you use it, log every rerun loudly so a chronically flaking row is visible.

## Cost and time budgets

An eval that costs $20 to run once will get skipped. Budget explicitly:

- **Cost budget per run.** Compute expected cost from token counts × per-token pricing × row count, plus judge-call overhead. If it exceeds the team's tolerance (a rough rule: under a dollar per full run for a first eval), cut the row count, downgrade the judge model, or move similarity references to be cached at ingest (chapter 2). See mod-108 for cost estimation shape.
- **Wall-clock budget.** Rows × latency, divided by xdist parallelism. Under 3 minutes for a full run is the pragmatic target for a first eval — long enough to cover a real set, short enough that developers will actually wait for it. Longer than 10 minutes and it belongs on a nightly schedule, not a per-PR one.
- **Log actual cost per run.** Sum the `usage` blocks across every model call in the run and print a summary line at the end. When someone asks "how much does our eval cost?" the answer should be a number from your logs, not a guess.

## Gating in CI

Wiring the eval into CI is the moment the module's core claim — "a prompt or model change cannot silently regress the feature" — becomes true. The gate has three layers.

### Layer 1: run the eval on the PRs that need it

Do not run the eval on every push to every branch. It costs money, adds latency to unrelated changes, and trains people to ignore the check. Run it on changes that plausibly affect the feature:

- Any change under `src/summariser/**` or `evals/summariser/**`.
- Any change to shared prompt / model / provider config (`src/prompts/**`, `src/providers/**`, `pyproject.toml` version pins for LLM SDKs).
- Manually, on any PR, via a `/run-evals summariser` comment or a workflow-dispatch button — for the "I edited a shared util that *might* affect this" case.

Modern CI providers all support path filters; the details differ by provider (GitHub Actions `on.pull_request.paths`, GitLab `rules:changes`, CircleCI `path-filtering` orb, Buildkite dynamic pipelines). See `resources.md` for links.

A GitHub Actions job that runs the summariser eval on relevant PRs and blocks merge on failure:

```yaml
# .github/workflows/eval-summariser.yml
name: eval-summariser
on:
  pull_request:
    paths:
      - "src/summariser/**"
      - "evals/summariser/**"
      - "src/prompts/**"
      - "src/providers/**"

jobs:
  eval:
    runs-on: ubuntu-latest
    concurrency:
      group: eval-summariser-${{ github.ref }}
      cancel-in-progress: true
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[dev]"
      - name: Run eval
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY_EVAL }}
          RUN_EVALS: "1"
        run: |
          pytest -m eval -n auto \
            --junitxml=eval-results.xml \
            evals/summariser
      - if: always()
        uses: actions/upload-artifact@v4
        with:
          name: eval-results
          path: eval-results.xml
```

Two operational rules:

- **Use a dedicated CI-only API key.** Rotate it independently, budget-cap it, and revoke it without disrupting anyone's local key. Never commit keys; use whatever secret store your provider offers.
- **`concurrency.cancel-in-progress: true`.** A developer pushing five commits to a PR in ten minutes should cause four cancellations and one run. Without it you pay for every push.

### Layer 2: post the score back to the PR

A red X is a signal; a red X with a diff table is a signal a human can act on. Post the eval outcome as a PR comment (or a check-run summary) so the developer sees which rows failed without having to open the CI logs:

- `Row sum-014 failed on reference_summary:similarity (0.63 < 0.75). Model wrote: "…". Expected: "…".`
- Baseline score: `47/50`. This PR: `44/50`. Regressions: `sum-014, sum-022, ext-041`.

You do not need a fancy tool for this — a small script that reads the JUnit XML output and comments via the CI's API is enough for a first module. Third-party tooling (`pytest-md-report`, `dorny/test-reporter`, and the like) will do it too; pick one and stay with it.

### Layer 3: make it a required check

In your branch-protection / merge-queue settings, mark the eval job as **required for merge into the protected branch**. This is the load-bearing step. Without it, the eval is advisory; developers will merge anyway when it is inconvenient, and the whole exercise is theatre.

The one legitimate override: an explicit label on the PR (`eval-override`) that lets a maintainer skip the check when the eval itself is broken. Every use of the override should be logged and reviewed — a repeated override is a signal the eval is not calibrated to the feature.

### What "the eval passed" should mean, exactly

A common early mistake is defining "pass" as "100% of rows passed." Once your set is ~30 rows, one flaky row will keep you from ever hitting 100%, and the team will get used to green-with-known-flakes — which trains them to ignore new failures.

The pragmatic pass criterion for a first eval:

- **Per-row:** the row's `flake_budget` (default: `1 of 1`) is met.
- **Suite-level:** at least `N` rows pass, where `N` is the *baseline* — the pass count on the branch's merge base — minus a small tolerance (0 or 1). Regressions below baseline block the PR; a new failure on a row that was passing on `main` is a regression, always.

Chapter 4 covers how to compute that baseline over time and how to attribute a regression to a specific commit / prompt version / model version.

## What to build in this chapter's exercise

Exercise 02 takes your golden set from exercise 01, wires it into pytest with the shape above, sets up the `RUN_EVALS` gating, and adds a CI job (or a local-dry-run script that stands in for CI if you do not have one available) that runs the suite on a PR-like change and posts a summary. You should end the exercise able to open a hypothetical PR that changes the prompt, watch the eval run, and see the specific row that regressed — without having to explain anything to a colleague reviewing the PR.

## Summary

- Run evals as parametrised pytest tests, with row IDs as parametrisation IDs so pytest output and JUnit XML surface your row names verbatim.
- Colocate evals with the feature (`evals/<feature>/`), not under `tests/`. Mark them with `@pytest.mark.eval` and skip them unless `RUN_EVALS=1` is set.
- Use path filters in CI so evals run on the changes that matter — not on every push. Use a dedicated CI-only API key and cancel superseded runs.
- Post the pass/fail summary back to the PR (comment or check-run) so failures are visible without opening logs. Require the eval job for merge, with a documented, logged override for the case where the eval itself is broken.
- "Pass" means "no row that was passing on the merge base is failing now." A hard 100% target is a trap; a "no new regressions" target is the honest one.

The next chapter takes the pass/fail signal you now emit on every PR and turns it into something you can look at over time: which change moved which row, and how to route that regression to the developer who caused it.
