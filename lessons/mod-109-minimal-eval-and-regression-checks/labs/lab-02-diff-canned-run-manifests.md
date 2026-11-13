# Lab 02 — Diff a Baseline Run Against a Candidate Run on Canned Manifests

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 4 — Tracking regressions over time](../04-tracking-regressions-over-time.md) ("The diff: what regressed vs. what fixed vs. what stayed the same", "A baseline: what 'known-good' points at")

## Objective

Chapter 4's load-bearing output is the PR-comment-shaped diff: "newly failing, newly passing, still failing," one line per row with the specific sub-check that failed, plus attribution to the change that moved it. In this lab you will build that diff in isolation on **two canned run manifests** that live on disk already. No live models. No golden set. No pytest. Just the diff primitive and the Markdown rendering — so when exercise 04 drives it from real model runs, the only thing new is the data, not the logic.

By the end you will have:

- `evals/<feature>/runs/` populated with two manifest + JSONL pairs: a baseline run (49 of 50 passing) and a candidate run (45 of 50 passing) that share the same `golden_set_sha`.
- A `diff.py` CLI that accepts `--baseline <manifest>` and `--candidate <manifest>`, joins on `row_id`, and emits the three lists plus added / dropped rows.
- A Markdown `REPORT.md` matching the shape in chapter 4's example (header with run IDs, summary counts, three categorised lists with per-sub-check reasons, attribution line).
- A **deliberate mismatch demo** in `NOTES.md` — rerun the CLI against a third manifest whose `golden_set_sha` differs, confirm the refusal message fires, and write one sentence on why the refusal is the right behaviour.

> This lab complements [exercise-04](../exercises/exercise-04-regression-report-for-a-model-swap.md), which runs the eval twice on real models, persists the runs you need, and wraps the diff with cost math, version-pin assertions, and a one-paragraph shipping recommendation. Here we stay with pre-written run files so you can see what one defensible diff actually looks like before the exercise adds live model cost and attribution on top.

## Prerequisites

- Finished chapter 4.
- Python 3.10+. No API keys needed; nothing in this lab calls a model.
- No third-party dependencies beyond the standard library (`json`, `pathlib`, `argparse`).

## Step 1 — Write the two canned run records

Create the directory and drop the four files chapter 4's manifest + JSONL shape prescribes.

```bash
mkdir -p evals/summariser/runs
```

Save as `evals/summariser/runs/2026-10-01T09-00-00Z_scheduled_a19c0f2.manifest.json` (the baseline):

```json
{
  "run_id": "2026-10-01T09-00-00Z_scheduled_a19c0f2",
  "started_at": "2026-10-01T09:00:00Z",
  "finished_at": "2026-10-01T09:02:18Z",
  "git_sha": "a19c0f2c8f39ea4b5c01a2d78b1234cafedb0001",
  "git_branch": "main",
  "pr_number": null,
  "author": "ci@example.com",
  "trigger": "scheduled",
  "feature": "summariser",
  "golden_set_sha": "d2e0f11ab79c04b1",
  "code_versions": {
    "prompt_version": "summariser@v14",
    "model": "gpt-4o-mini-2024-07-18",
    "temperature": 0,
    "seed": 42,
    "sdk_versions": {"openai": "1.51.0"}
  },
  "totals": {"rows": 5, "passed": 4, "failed": 1, "cost_usd": 0.14}
}
```

And its sibling `evals/summariser/runs/2026-10-01T09-00-00Z_scheduled_a19c0f2.jsonl` (five rows, one failing):

```jsonl
{"row_id": "sum-001", "pass": true,  "checks": [{"field": "reference_summary", "kind": "similarity", "pass": true,  "score": 0.83, "threshold": 0.75}], "latency_ms": 812, "usage": {"input_tokens": 1204, "output_tokens": 87}}
{"row_id": "sum-002", "pass": true,  "checks": [{"field": "must_mention", "kind": "keyword", "pass": true,  "missing": []}, {"field": "reference_summary", "kind": "similarity", "pass": true,  "score": 0.81, "threshold": 0.75}], "latency_ms": 743, "usage": {"input_tokens": 1190, "output_tokens": 71}}
{"row_id": "sum-003", "pass": true,  "checks": [{"field": "must_mention", "kind": "keyword", "pass": true,  "missing": []}, {"field": "reference_summary", "kind": "similarity", "pass": true,  "score": 0.78, "threshold": 0.75}], "latency_ms": 910, "usage": {"input_tokens": 1388, "output_tokens": 95}}
{"row_id": "sum-004", "pass": true,  "checks": [{"field": "max_words", "kind": "length", "pass": true,  "words": 32, "max_words": 40}], "latency_ms": 680, "usage": {"input_tokens": 1004, "output_tokens": 47}}
{"row_id": "sum-005", "pass": false, "checks": [{"field": "reference_summary", "kind": "similarity", "pass": false, "score": 0.72, "threshold": 0.75}], "latency_ms": 790, "usage": {"input_tokens": 1120, "output_tokens": 82}}
```

Now the candidate (a plausible model swap — same golden set, new model pinned). Save as `evals/summariser/runs/2026-10-14T13-22-17Z_pr-482_bfa1c04.manifest.json`:

```json
{
  "run_id": "2026-10-14T13-22-17Z_pr-482_bfa1c04",
  "started_at": "2026-10-14T13:22:17Z",
  "finished_at": "2026-10-14T13:24:41Z",
  "git_sha": "bfa1c04c8f39ea4b5c01a2d78b1234cafedb0482",
  "git_branch": "model-swap-gpt-4.1-mini",
  "pr_number": 482,
  "author": "achen@example.com",
  "trigger": "pull_request",
  "feature": "summariser",
  "golden_set_sha": "d2e0f11ab79c04b1",
  "code_versions": {
    "prompt_version": "summariser@v14",
    "model": "gpt-4.1-mini-2025-04-14",
    "temperature": 0,
    "seed": 42,
    "sdk_versions": {"openai": "1.51.0"}
  },
  "totals": {"rows": 5, "passed": 3, "failed": 2, "cost_usd": 0.03}
}
```

And its sibling `evals/summariser/runs/2026-10-14T13-22-17Z_pr-482_bfa1c04.jsonl`:

```jsonl
{"row_id": "sum-001", "pass": true,  "checks": [{"field": "reference_summary", "kind": "similarity", "pass": true,  "score": 0.82, "threshold": 0.75}], "latency_ms": 540, "usage": {"input_tokens": 1204, "output_tokens": 81}}
{"row_id": "sum-002", "pass": false, "checks": [{"field": "must_mention", "kind": "keyword", "pass": false, "missing": ["Globex"]}, {"field": "reference_summary", "kind": "similarity", "pass": true,  "score": 0.79, "threshold": 0.75}], "latency_ms": 510, "usage": {"input_tokens": 1190, "output_tokens": 68}}
{"row_id": "sum-003", "pass": true,  "checks": [{"field": "must_mention", "kind": "keyword", "pass": true,  "missing": []}, {"field": "reference_summary", "kind": "similarity", "pass": true,  "score": 0.76, "threshold": 0.75}], "latency_ms": 615, "usage": {"input_tokens": 1388, "output_tokens": 90}}
{"row_id": "sum-004", "pass": false, "checks": [{"field": "max_words", "kind": "length", "pass": false, "words": 46, "max_words": 40}], "latency_ms": 470, "usage": {"input_tokens": 1004, "output_tokens": 61}}
{"row_id": "sum-005", "pass": true,  "checks": [{"field": "reference_summary", "kind": "similarity", "pass": true,  "score": 0.78, "threshold": 0.75}], "latency_ms": 560, "usage": {"input_tokens": 1120, "output_tokens": 79}}
```

### Checkpoint

- Four files are on disk (`baseline.manifest.json` + `baseline.jsonl`; `candidate.manifest.json` + `candidate.jsonl`). The two manifests share a `golden_set_sha` (`d2e0f11ab79c04b1`) — the gate chapter 4 requires before a diff is allowed. The two differ on `code_versions.model` — the one variable that moved.
- Walk the data once by eye: which rows flipped, and in which direction?
  - `sum-002`: baseline pass → candidate fail (newly failing; `must_mention` dropped `Globex`).
  - `sum-004`: baseline pass → candidate fail (newly failing; `length` went from 32 to 46 words).
  - `sum-005`: baseline fail → candidate pass (newly passing; similarity 0.72 → 0.78).
  - `sum-001`, `sum-003`: unchanged pass.
- That three-category walk is exactly what chapter 4's diff function computes. Doing it by eye first is how you know the CLI output in step 3 is right.

## Step 2 — Write `diff.py`

Save as `evals/diff.py`. The shape is chapter 4's `diff(baseline_rows, current_rows)` function, wrapped in a tiny CLI with argument parsing and the `golden_set_sha` guard.

```python
"""
Diff two run records (chapter 4's shape), render a PR-comment-shaped Markdown report.

Usage:
    python evals/diff.py --baseline <baseline.manifest.json> --candidate <candidate.manifest.json>
"""

from __future__ import annotations

import argparse
import json
import sys
from pathlib import Path


def _load_run(manifest_path: Path) -> tuple[dict, dict[str, dict]]:
    manifest = json.loads(manifest_path.read_text())
    jsonl_path = manifest_path.with_suffix("")  # strip ".json"
    jsonl_path = jsonl_path.with_suffix(".jsonl")  # swap the preceding ".manifest" with ".jsonl"
    # The above tolerates the "<run_id>.manifest.json" naming by requiring the
    # sibling "<run_id>.jsonl" to exist. If your layout differs, adjust here.
    jsonl_path = manifest_path.parent / (manifest_path.name.replace(".manifest.json", ".jsonl"))
    rows: dict[str, dict] = {}
    with jsonl_path.open() as f:
        for line in f:
            if not line.strip():
                continue
            row = json.loads(line)
            rows[row["row_id"]] = row
    return manifest, rows


def _check_reason(check: dict) -> str:
    """Compact reason string for a failing sub-check — never strip the 'why'."""
    kind = check.get("kind", "?")
    if kind == "similarity":
        return f"similarity {check.get('score', '?'):.2f} < {check.get('threshold', '?')}"
    if kind == "keyword":
        missing = check.get("missing", [])
        return f"keyword missing {missing}"
    if kind == "length":
        return f"length {check.get('words', '?')} > max_words {check.get('max_words', '?')}"
    if kind == "schema":
        return f"schema: {check.get('error', 'invalid')}"
    if kind == "regex":
        return f"regex /{check.get('pattern', '?')}/ did not match"
    return f"{kind} failed"


def _first_failed_reason(row: dict) -> str:
    for check in row.get("checks", []):
        if not check.get("pass", True):
            return f"{check.get('field', '?')}  {_check_reason(check)}"
    return "(no failing sub-check recorded)"


def diff(baseline_rows: dict[str, dict], current_rows: dict[str, dict]) -> dict:
    common = baseline_rows.keys() & current_rows.keys()
    newly_failing = sorted(
        r for r in common
        if baseline_rows[r]["pass"] and not current_rows[r]["pass"]
    )
    newly_passing = sorted(
        r for r in common
        if not baseline_rows[r]["pass"] and current_rows[r]["pass"]
    )
    still_failing = sorted(
        r for r in common
        if not baseline_rows[r]["pass"] and not current_rows[r]["pass"]
    )
    added = sorted(current_rows.keys() - baseline_rows.keys())
    dropped = sorted(baseline_rows.keys() - current_rows.keys())
    return {
        "newly_failing": newly_failing,
        "newly_passing": newly_passing,
        "still_failing": still_failing,
        "added": added,
        "dropped": dropped,
    }


def _attribution(baseline_manifest: dict, candidate_manifest: dict) -> list[str]:
    """Which variables moved between the two manifests. One line per change."""
    lines: list[str] = []
    b_versions = baseline_manifest["code_versions"]
    c_versions = candidate_manifest["code_versions"]
    for key in sorted(set(b_versions) | set(c_versions)):
        if b_versions.get(key) != c_versions.get(key):
            lines.append(f"{key}: {b_versions.get(key)} -> {c_versions.get(key)}")
    return lines


def render_report(
    baseline_manifest: dict,
    candidate_manifest: dict,
    baseline_rows: dict[str, dict],
    current_rows: dict[str, dict],
    d: dict,
) -> str:
    feature = candidate_manifest["feature"]
    attr = _attribution(baseline_manifest, candidate_manifest)
    attr_str = "; ".join(attr) if attr else "(no code-version changes recorded)"

    b_tot = baseline_manifest["totals"]
    c_tot = candidate_manifest["totals"]

    lines: list[str] = []
    lines.append(f"Eval: {feature} ({c_tot['rows']} rows)")
    lines.append(
        f"  Baseline: {b_tot['passed']}/{b_tot['rows']} passing @ "
        f"{baseline_manifest['git_sha'][:7]} on {baseline_manifest['git_branch']}"
    )
    lines.append(
        f"  This run: {c_tot['passed']}/{c_tot['rows']} passing @ "
        f"{candidate_manifest['git_sha'][:7]}"
    )
    lines.append("")
    lines.append(f"Change: {attr_str}")
    lines.append("")

    lines.append(f"Regressions ({len(d['newly_failing'])}):")
    for row_id in d["newly_failing"]:
        reason = _first_failed_reason(current_rows[row_id])
        lines.append(f"  - {row_id}  {reason}   — ({attr_str})")
    lines.append("")

    lines.append(f"Fixed ({len(d['newly_passing'])}):")
    for row_id in d["newly_passing"]:
        lines.append(f"  - {row_id}  now passing")
    lines.append("")

    lines.append(f"Still failing ({len(d['still_failing'])}):")
    for row_id in d["still_failing"]:
        reason = _first_failed_reason(current_rows[row_id])
        lines.append(f"  - {row_id}  {reason}")
    lines.append("")

    if d["added"] or d["dropped"]:
        lines.append(f"Set changes: added={d['added']} dropped={d['dropped']}")

    lines.append("")
    lines.append(f"Cost: baseline ${b_tot['cost_usd']:.2f}  candidate ${c_tot['cost_usd']:.2f}")
    return "\n".join(lines)


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--baseline", required=True, type=Path)
    parser.add_argument("--candidate", required=True, type=Path)
    args = parser.parse_args(argv)

    baseline_manifest, baseline_rows = _load_run(args.baseline)
    candidate_manifest, current_rows = _load_run(args.candidate)

    # Chapter 4: "The diff step should refuse to compare across different
    # golden_set_sha values ... That refusal is the correct behaviour."
    if baseline_manifest["golden_set_sha"] != candidate_manifest["golden_set_sha"]:
        print(
            f"baseline is stale; regenerate on trunk: "
            f"baseline golden_set_sha={baseline_manifest['golden_set_sha']} "
            f"candidate golden_set_sha={candidate_manifest['golden_set_sha']}",
            file=sys.stderr,
        )
        return 2

    d = diff(baseline_rows, current_rows)
    print(render_report(baseline_manifest, candidate_manifest, baseline_rows, current_rows, d))
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## Step 3 — Run the diff on the two canned manifests

```bash
python evals/diff.py \
  --baseline evals/summariser/runs/2026-10-01T09-00-00Z_scheduled_a19c0f2.manifest.json \
  --candidate evals/summariser/runs/2026-10-14T13-22-17Z_pr-482_bfa1c04.manifest.json \
  > REPORT.md

cat REPORT.md
```

### Expected output (shape)

```
Eval: summariser (5 rows)
  Baseline: 4/5 passing @ a19c0f2 on main
  This run: 3/5 passing @ bfa1c04

Change: model: gpt-4o-mini-2024-07-18 -> gpt-4.1-mini-2025-04-14

Regressions (2):
  - sum-002  must_mention  keyword missing ['Globex']   — (model: gpt-4o-mini-2024-07-18 -> gpt-4.1-mini-2025-04-14)
  - sum-004  max_words  length 46 > max_words 40   — (model: gpt-4o-mini-2024-07-18 -> gpt-4.1-mini-2025-04-14)

Fixed (1):
  - sum-005  now passing

Still failing (0):


Cost: baseline $0.14  candidate $0.03
```

### Checkpoint

- The three lists match what you walked by eye in step 1. If any row is in the wrong bucket, the bug is in `diff()` — fix it before step 4.
- Each regression line names the **specific sub-check** that failed (`must_mention keyword missing ['Globex']`, not just "sum-002 failed"). Chapter 4 is explicit: "The regression lines name the sub-check that failed — this is why chapter 2 preserves the per-check trail." A regression report that only says "row failed" is not actionable.
- The attribution string on each regression line names the one variable that moved — `model: gpt-4o-mini-2024-07-18 -> gpt-4.1-mini-2025-04-14`. If your two manifests ever disagree on more than one variable, this is where you would notice. Chapter 4's "one variable at a time" rule becomes visible in this line.

## Step 4 — Trigger the `golden_set_sha` refusal on purpose

Chapter 4's "refuses to diff across mismatched `golden_set_sha` values" guard is a *feature*, not a bug. Prove it fires.

Open the candidate manifest and change its `golden_set_sha` to any other value — a single-character edit is enough:

```bash
# Change "d2e0f11ab79c04b1" to "dEADbEEf79c04b1" in the candidate manifest.
# Then rerun:
python evals/diff.py \
  --baseline evals/summariser/runs/2026-10-01T09-00-00Z_scheduled_a19c0f2.manifest.json \
  --candidate evals/summariser/runs/2026-10-14T13-22-17Z_pr-482_bfa1c04.manifest.json
echo "exit_code=$?"
```

Expected (the CLI exits non-zero and prints the refusal message to stderr):

```
baseline is stale; regenerate on trunk: baseline golden_set_sha=d2e0f11ab79c04b1 candidate golden_set_sha=dEADbEEf79c04b1
exit_code=2
```

Restore the sha. In `NOTES.md`, write one sentence on **why this refusal is correct**: two runs against different golden sets are comparing different sets of rows, so the pass-count diff would silently misrepresent what changed. The CLI's refusal tells the developer to regenerate baseline on trunk rather than letting the comparison produce misleading numbers.

### Checkpoint

- The exit code is non-zero. If the CI job that runs this diff is wired as "fail the build on non-zero exit," the mismatch becomes a visible red X on the PR — not a quiet "the diff looked weird today."
- A colleague reading the stderr message cold knows exactly what to do: regenerate the baseline on trunk after the golden-set change merges. No archaeology.

## Step 5 — Write a 5-sentence debrief in `NOTES.md`

Answer in 5–7 sentences total:

1. Which regression line in `REPORT.md` is the most actionable, and what makes it actionable? (Compare the chapter-4-shaped line to a hypothetical "sum-002 failed." What is preserved here that the shorter form loses?)
2. The `Fixed (1): sum-005` row is a *win*. Chapter 4 says "newly passing rows ... are wins; surface them so improvements do not vanish." Why would it matter, long-term, to list wins alongside regressions rather than only listing regressions?
3. If both the model and the prompt had changed between baseline and candidate, `Change:` would list two lines. Why does chapter 4 insist on calling that out explicitly, and what is the "one variable at a time" rule it is defending?
4. On this canned data, cost dropped 4.7× from baseline to candidate ($0.14 → $0.03). That is not enough information to recommend shipping the candidate. What is the chapter-4-style question you would ask before making the call?
5. One sentence on the `golden_set_sha` refusal: why is a diff that refuses to compare better than a diff that silently compares anyway?

That write-up is the deliverable. Commit the two run pairs, `diff.py`, and `NOTES.md`.

## Debrief

Three habits pay forward from here:

- The diff primitive is small. `diff()` is nine lines; the rendering is the rest. Every hosted eval platform wraps a version of this; chapter 4's "manifest + baseline + diff shape is the primitive under every hosted eval platform" is the sentence this lab makes concrete. Understanding the raw shape makes the platforms make sense.
- Never ship a diff that papers over the reason. Every failing row carries the sub-check that failed and the numeric value involved (`similarity 0.72 < 0.75`, `missing ['Globex']`, `46 > 40`). The reason is the artefact; a developer on the PR will skim the diff in ten seconds and either know what to look at or move on without reading further. Chapter 4 is explicit: "`sum-014 failed` is not actionable; `sum-014 must_mention missing ['Globex']` is."
- The `golden_set_sha` refusal is a load-bearing guardrail, not an edge case. It is the only thing between you and a regression report that silently compares two different sets of rows. Trip it on purpose when you first wire the diff — if you cannot get it to fire, the guard is not there.

The exercise next door (exercise 04) takes this same diff CLI and feeds it real model outputs from two runs (baseline model vs. candidate model) on your actual golden set, wraps it with cost math, a one-paragraph shipping recommendation, and chapter 4's version-pin assertion. If this lab's diff shape made sense, that exercise's model-swap report will too.

## Stretch

- **Add an `--added-row-hint` note.** In chapter 4's recommended comment shape, when a golden-set row was added between baseline and candidate, it shows up in `added=[...]` without a baseline counterpart. Extend `render_report` to print an explicit `Added since baseline: [...]` line and, for each such row, its *current-run* pass/fail. The point is that an added-and-already-failing row is a signal (the author added a row that fails on the candidate) that a naïve diff would otherwise lose.
- **Per-tolerance-family regression breakdown.** Below the three lists, aggregate how many regressions were `similarity` vs. `keyword` vs. `length` vs. `schema`. Exercise 04's stretch calls this out — a cluster of regressions on one scorer family is a signal that the *scorer* needs recalibration (e.g. the candidate model phrases things differently and the similarity threshold is now too tight), not that the model is bad.
- **GitHub-Markdown table version.** Render the Regressions / Fixed / Still-failing sections as proper GitHub Markdown tables with columns `row | sub-check | reason | attribution`. The ugly plain-text block in this lab is fine for a learning drill, but a comment on a real PR usually benefits from the table rendering. Chapter 4's example in the chapter is in table-like prose; GitHub renders it cleanly either way, but the table scales better once you have 20 rows failing on a single PR.
- **Still-failing attribution stub.** In the current lab the still-failing list is empty. Construct a third baseline-era run that already had `sum-005` failing on an earlier trunk commit, point `BASELINE.json` at it, and rerun the diff — `sum-005` now shows up in `Still failing`. For each still-failing row, print `(introduced at <older_sha> by <older_author>)` by walking the `runs/` directory from oldest to newest. This is a preview of the `when_did_row_start_failing` bisect from chapter 4's "Old regressions: bisect + attribute" section.
