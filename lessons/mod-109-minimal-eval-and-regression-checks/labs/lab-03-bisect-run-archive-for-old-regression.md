# Lab 03 — Bisect a Run Archive to Attribute an Old Still-Failing Row

**Estimated duration:** 25–40 minutes
**Covers:** [chapter 4 — Tracking regressions over time](../04-tracking-regressions-over-time.md) ("Old regressions: bisect + attribute")

## Objective

Chapter 4 is blunt about why the "still failing" list matters: without attribution, it turns into "a graveyard where every row is someone's problem and therefore nobody's." The fix is the `when_did_row_start_failing` walk — look at each manifest in order, find the earliest run where a row flipped from passing to failing, and name that run's commit and author.

In this lab you will build and drive that walk against a **pre-written archive of five run manifests** that lives on disk. No live models. No golden set. No pytest. Just the attribution primitive — so when exercise 04 has a long-lived still-failing list to annotate, the only thing new is the data, not the logic.

By the end you will have:

- An `evals/<feature>/runs/` directory with five manifest + JSONL pairs spanning six weeks of chronological eval history, one row of which (`sum-007`) deliberately starts failing partway through.
- A `bisect.py` CLI that implements chapter 4's `when_did_row_start_failing` walk and prints `(run_id, git_sha, author, pr_number)` for the earliest fail-after-pass transition.
- A rendered still-failing attribution line matching chapter 4's example shape:
  `sum-007  judge  "adds unsupported claim" — introduced 2026-07-02 by @tliu (#458)`.
- A `NOTES.md` with the one-sentence policy from chapter 4 that the attribution enforces, and one sentence on the two failure modes the walk guards against (`row never failed` and `row always failed`).

> This lab complements [exercise-04](../exercises/exercise-04-regression-report-for-a-model-swap.md), whose stretch goal asks you to bisect on the still-failing list for *one* row. Here we isolate the bisect mechanism itself on a canned archive, so the "walk the manifests oldest → newest" shape is understood before the exercise bolts it onto a live model-swap diff.

## Prerequisites

- Finished chapter 4.
- Lab 02 helps context-wise but is not required — the run-record shapes used here are the same as in that lab.
- Python 3.10+. No API keys needed; no network calls.
- Standard library only (`json`, `pathlib`, `argparse`).

## Step 1 — Lay down a five-run archive

Create the directory:

```bash
mkdir -p evals/summariser/runs
```

The archive covers six weeks. `sum-007` passes in the first two runs, starts failing in the third (the PR that will be the "introduced by" target), and stays failing after. Row `sum-012` fails in the last run only (useful for the "first failure is the last run" edge in step 3). All five share the same `golden_set_sha`.

Save the five manifest + JSONL pairs. For brevity each JSONL has three rows: `sum-001` (always passing), `sum-007` (our target), `sum-012` (late failer).

**Run 1 — oldest.** `evals/summariser/runs/2026-06-14T09-00-00Z_scheduled_111aaaa.manifest.json`:

```json
{
  "run_id": "2026-06-14T09-00-00Z_scheduled_111aaaa",
  "started_at": "2026-06-14T09:00:00Z",
  "finished_at": "2026-06-14T09:02:11Z",
  "git_sha": "111aaaacafed0001000000000000000000000001",
  "git_branch": "main",
  "pr_number": null,
  "author": "ci@example.com",
  "trigger": "scheduled",
  "feature": "summariser",
  "golden_set_sha": "d2e0f11ab79c04b1",
  "code_versions": {"prompt_version": "summariser@v11", "model": "gpt-4o-mini-2024-07-18", "temperature": 0, "seed": 42, "sdk_versions": {"openai": "1.51.0"}},
  "totals": {"rows": 3, "passed": 3, "failed": 0, "cost_usd": 0.09}
}
```

`evals/summariser/runs/2026-06-14T09-00-00Z_scheduled_111aaaa.jsonl`:

```jsonl
{"row_id": "sum-001", "pass": true,  "checks": [{"field": "reference_summary", "kind": "similarity", "pass": true, "score": 0.83, "threshold": 0.75}]}
{"row_id": "sum-007", "pass": true,  "checks": [{"field": "rubric", "kind": "judge", "pass": true, "reason": "faithful summary"}]}
{"row_id": "sum-012", "pass": true,  "checks": [{"field": "must_mention", "kind": "keyword", "pass": true, "missing": []}]}
```

**Run 2.** `evals/summariser/runs/2026-06-28T09-00-00Z_scheduled_222bbbb.manifest.json`:

```json
{
  "run_id": "2026-06-28T09-00-00Z_scheduled_222bbbb",
  "started_at": "2026-06-28T09:00:00Z",
  "finished_at": "2026-06-28T09:02:04Z",
  "git_sha": "222bbbbcafed0002000000000000000000000002",
  "git_branch": "main",
  "pr_number": null,
  "author": "ci@example.com",
  "trigger": "scheduled",
  "feature": "summariser",
  "golden_set_sha": "d2e0f11ab79c04b1",
  "code_versions": {"prompt_version": "summariser@v12", "model": "gpt-4o-mini-2024-07-18", "temperature": 0, "seed": 42, "sdk_versions": {"openai": "1.51.0"}},
  "totals": {"rows": 3, "passed": 3, "failed": 0, "cost_usd": 0.09}
}
```

`evals/summariser/runs/2026-06-28T09-00-00Z_scheduled_222bbbb.jsonl`:

```jsonl
{"row_id": "sum-001", "pass": true, "checks": [{"field": "reference_summary", "kind": "similarity", "pass": true, "score": 0.84, "threshold": 0.75}]}
{"row_id": "sum-007", "pass": true, "checks": [{"field": "rubric", "kind": "judge", "pass": true, "reason": "faithful summary"}]}
{"row_id": "sum-012", "pass": true, "checks": [{"field": "must_mention", "kind": "keyword", "pass": true, "missing": []}]}
```

**Run 3 — the PR that broke `sum-007`.** `evals/summariser/runs/2026-07-02T14-10-00Z_pr-458_333cccc.manifest.json`:

```json
{
  "run_id": "2026-07-02T14-10-00Z_pr-458_333cccc",
  "started_at": "2026-07-02T14:10:00Z",
  "finished_at": "2026-07-02T14:12:51Z",
  "git_sha": "333cccccafed0003000000000000000000000003",
  "git_branch": "tighten-summariser-tone",
  "pr_number": 458,
  "author": "tliu@example.com",
  "trigger": "pull_request",
  "feature": "summariser",
  "golden_set_sha": "d2e0f11ab79c04b1",
  "code_versions": {"prompt_version": "summariser@v13", "model": "gpt-4o-mini-2024-07-18", "temperature": 0, "seed": 42, "sdk_versions": {"openai": "1.51.0"}},
  "totals": {"rows": 3, "passed": 2, "failed": 1, "cost_usd": 0.10}
}
```

`evals/summariser/runs/2026-07-02T14-10-00Z_pr-458_333cccc.jsonl`:

```jsonl
{"row_id": "sum-001", "pass": true,  "checks": [{"field": "reference_summary", "kind": "similarity", "pass": true, "score": 0.82, "threshold": 0.75}]}
{"row_id": "sum-007", "pass": false, "checks": [{"field": "rubric", "kind": "judge", "pass": false, "reason": "adds unsupported claim"}]}
{"row_id": "sum-012", "pass": true,  "checks": [{"field": "must_mention", "kind": "keyword", "pass": true, "missing": []}]}
```

**Run 4.** `evals/summariser/runs/2026-07-15T09-00-00Z_scheduled_444dddd.manifest.json`:

```json
{
  "run_id": "2026-07-15T09-00-00Z_scheduled_444dddd",
  "started_at": "2026-07-15T09:00:00Z",
  "finished_at": "2026-07-15T09:02:40Z",
  "git_sha": "444ddddcafed0004000000000000000000000004",
  "git_branch": "main",
  "pr_number": null,
  "author": "ci@example.com",
  "trigger": "scheduled",
  "feature": "summariser",
  "golden_set_sha": "d2e0f11ab79c04b1",
  "code_versions": {"prompt_version": "summariser@v13", "model": "gpt-4o-mini-2024-07-18", "temperature": 0, "seed": 42, "sdk_versions": {"openai": "1.51.0"}},
  "totals": {"rows": 3, "passed": 2, "failed": 1, "cost_usd": 0.10}
}
```

`evals/summariser/runs/2026-07-15T09-00-00Z_scheduled_444dddd.jsonl`:

```jsonl
{"row_id": "sum-001", "pass": true,  "checks": [{"field": "reference_summary", "kind": "similarity", "pass": true, "score": 0.81, "threshold": 0.75}]}
{"row_id": "sum-007", "pass": false, "checks": [{"field": "rubric", "kind": "judge", "pass": false, "reason": "adds unsupported claim"}]}
{"row_id": "sum-012", "pass": true,  "checks": [{"field": "must_mention", "kind": "keyword", "pass": true, "missing": []}]}
```

**Run 5 — newest.** `evals/summariser/runs/2026-07-29T09-00-00Z_scheduled_555eeee.manifest.json`:

```json
{
  "run_id": "2026-07-29T09-00-00Z_scheduled_555eeee",
  "started_at": "2026-07-29T09:00:00Z",
  "finished_at": "2026-07-29T09:02:33Z",
  "git_sha": "555eeeecafed0005000000000000000000000005",
  "git_branch": "main",
  "pr_number": null,
  "author": "ci@example.com",
  "trigger": "scheduled",
  "feature": "summariser",
  "golden_set_sha": "d2e0f11ab79c04b1",
  "code_versions": {"prompt_version": "summariser@v13", "model": "gpt-4o-mini-2024-07-18", "temperature": 0, "seed": 42, "sdk_versions": {"openai": "1.51.0"}},
  "totals": {"rows": 3, "passed": 1, "failed": 2, "cost_usd": 0.10}
}
```

`evals/summariser/runs/2026-07-29T09-00-00Z_scheduled_555eeee.jsonl`:

```jsonl
{"row_id": "sum-001", "pass": true,  "checks": [{"field": "reference_summary", "kind": "similarity", "pass": true, "score": 0.80, "threshold": 0.75}]}
{"row_id": "sum-007", "pass": false, "checks": [{"field": "rubric", "kind": "judge", "pass": false, "reason": "adds unsupported claim"}]}
{"row_id": "sum-012", "pass": false, "checks": [{"field": "must_mention", "kind": "keyword", "pass": false, "missing": ["Q3 revenue"]}]}
```

### Checkpoint

- Ten files are on disk (five manifests + five JSONLs). `ls evals/summariser/runs/` should sort them chronologically by `run_id` because of the ISO-8601 timestamp prefix. That sortability is the whole reason chapter 4's "Store runs in the repo" section recommends the `<iso-timestamp>_<trigger>_<git-sha-short>` naming shape — a `sorted(glob("*.manifest.json"))` walks the archive in order with no extra tooling.
- Walk the data once by eye: on which run does `sum-007` flip? (Run 3, PR #458, author `tliu`.) On which run does `sum-012` flip? (Run 5, no PR — it is a scheduled trunk run, which is exactly the case chapter 4's "Alerting outside of PRs" section calls out.)
- Why is `sum-001` here at all? To prove the walk does **not** misattribute an always-passing row to anything. The negative case matters just as much as the positive one.

## Step 2 — Write `bisect.py`

Save as `evals/bisect.py`. The core is chapter 4's `when_did_row_start_failing` function, wrapped in a CLI that takes one `--row-id` and prints the attribution line.

```python
"""
Walk the archive of run manifests (oldest -> newest) and find the earliest run
where <row-id> flipped from passing to failing.

Chapter 4: "Old regressions: bisect + attribute."

Usage:
    python evals/bisect.py --runs-dir evals/summariser/runs --row-id sum-007
"""

from __future__ import annotations

import argparse
import json
import sys
from pathlib import Path


def _read_jsonl(path: Path) -> list[dict]:
    with path.open() as f:
        return [json.loads(line) for line in f if line.strip()]


def when_did_row_start_failing(
    row_id: str, runs_dir: Path
) -> tuple[dict, dict] | None:
    """
    Return (manifest_dict, failed_row_dict) of the earliest run in which
    `row_id` is failing after a previous run passed it.

    Returns None if the row never appears, never flipped (always passed,
    or always failed), or has only ever been seen once.
    """
    prev_pass: bool | None = None
    for manifest_path in sorted(runs_dir.glob("*.manifest.json")):
        jsonl_path = manifest_path.parent / manifest_path.name.replace(
            ".manifest.json", ".jsonl"
        )
        rows = {r["row_id"]: r for r in _read_jsonl(jsonl_path)}
        if row_id not in rows:
            continue
        currently_passing = rows[row_id]["pass"]
        if prev_pass is True and currently_passing is False:
            manifest = json.loads(manifest_path.read_text())
            return manifest, rows[row_id]
        prev_pass = currently_passing
    return None


def _first_failed_reason(row: dict) -> str:
    for check in row.get("checks", []):
        if not check.get("pass", True):
            kind = check.get("kind", "?")
            field = check.get("field", "?")
            if kind == "judge":
                return f'{field}  judge  "{check.get("reason", "?")}"'
            if kind == "keyword":
                return f"{field}  keyword  missing {check.get('missing', [])}"
            if kind == "similarity":
                return (
                    f"{field}  similarity  "
                    f"{check.get('score', '?'):.2f} < {check.get('threshold', '?')}"
                )
            return f"{field}  {kind}  failed"
    return "(no failing sub-check recorded)"


def render_attribution(row_id: str, manifest: dict, failed_row: dict) -> str:
    """One line in chapter 4's still-failing format."""
    date = manifest["started_at"][:10]
    pr = manifest.get("pr_number")
    pr_part = f" (#{pr})" if pr else " (trunk, no PR)"
    handle = manifest["author"].split("@")[0]
    return (
        f"{row_id}  {_first_failed_reason(failed_row)}"
        f"   — introduced {date} by @{handle}{pr_part}"
    )


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--runs-dir", required=True, type=Path)
    parser.add_argument("--row-id", required=True)
    args = parser.parse_args(argv)

    result = when_did_row_start_failing(args.row_id, args.runs_dir)
    if result is None:
        print(f"{args.row_id}: no fail-after-pass transition found in the archive")
        return 0

    manifest, failed_row = result
    print(render_attribution(args.row_id, manifest, failed_row))
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## Step 3 — Run the walk against three rows

Three rows exercise three different cases chapter 4 names.

```bash
python evals/bisect.py --runs-dir evals/summariser/runs --row-id sum-007
python evals/bisect.py --runs-dir evals/summariser/runs --row-id sum-012
python evals/bisect.py --runs-dir evals/summariser/runs --row-id sum-001
```

### Expected output (shape)

```
sum-007  rubric  judge  "adds unsupported claim"   — introduced 2026-07-02 by @tliu (#458)
sum-012  must_mention  keyword  missing ['Q3 revenue']   — introduced 2026-07-29 by @ci (trunk, no PR)
sum-001: no fail-after-pass transition found in the archive
```

### Checkpoint

- `sum-007` names the exact run that flipped it — Run 3 (PR #458, author `tliu`). The attribution line is chapter 4's still-failing-row shape verbatim. A developer reading it on a PR comment knows exactly which past PR to open.
- `sum-012` is attributed to a trunk run with no PR. This is chapter 4's "Alerting outside of PRs" case — a drift that no PR touched (a shared prompt template edit, a provider silently rolling the pinned model, a retrieval-corpus refresh). Your walk correctly names it `(trunk, no PR)` rather than inventing a PR number.
- `sum-001` returns the "never flipped" message. The walk does *not* misattribute an always-passing row. This is the negative case chapter 4's function docstring explicitly names.

## Step 4 — Compose with a hypothetical diff's still-failing list

Chapter 4's PR-comment shape combines the diff with the bisect walk. The diff produces the still-failing *list*; the bisect walk annotates each line. The composition is three lines of code — add them to `bisect.py` as a `--row-ids` batch variant, or write the composition inline in a tiny script. Here is the composition in shell:

```bash
for row in sum-007 sum-012 sum-001; do
  python evals/bisect.py --runs-dir evals/summariser/runs --row-id "$row"
done
```

Which is what you would call from the diff-rendering code (lab 02's `diff.py`) once the diff produces the still-failing list. In `NOTES.md`, write the three annotated lines under a mock "Still failing" header to feel the full PR-comment shape:

```
Still failing (2):
  - sum-007  rubric  judge  "adds unsupported claim"   — introduced 2026-07-02 by @tliu (#458)
  - sum-012  must_mention  keyword  missing ['Q3 revenue']   — introduced 2026-07-29 by @ci (trunk, no PR)
```

### Checkpoint

- Compare that block to the "Still failing" block in chapter 4's example in the chapter. Same shape: row ID, failing sub-check with its specific value, dash, "introduced <date> by @<author> (#PR)". The `(trunk, no PR)` variant matters — it is the signal that routes to the pin maintainer, not to a code author.
- In `NOTES.md`, write the one-sentence policy chapter 4 names: a still-failing row that no longer has an owner because the PR that introduced it was long ago is fine; a still-failing row with a named owner on every PR comment is the thing that keeps old failures from being nobody's problem. The attribution walk is the mechanism that keeps the list short because it keeps it visible.

## Step 5 — Write a 5-sentence debrief in `NOTES.md`

Answer in 5–7 sentences total:

1. For `sum-007`, the archive has four consecutive runs where it is failing (runs 3, 4, 5, and later). Why does the walk stop at run 3 and name that run, not run 4 or 5?
2. For `sum-001`, the walk returns "no fail-after-pass transition." Chapter 4's function docstring says this covers two distinct cases ("never failed or always failed"). Why does conflating them into one return value matter less than you might think — and when would you want to tell them apart?
3. `sum-012` is attributed to a trunk run with no PR. Chapter 4 says the routing for that case is different from the PR-author routing for a new regression. In one sentence, name the chapter-4 recipient and why they are the right person.
4. The walk re-reads every manifest and every JSONL in the archive. For a long-lived feature (tens of runs per week, over a year) that gets slow. What is the cheapest small index you could maintain to make the walk fast without changing its shape?
5. One sentence on the sort order. If the archive were not timestamp-prefixed and `sorted(glob("*.manifest.json"))` walked in alphabetical order that did not match chronology, how would the walk's output change — and what does this tell you about the naming convention chapter 4 recommends?

That write-up is the deliverable. Commit the five run pairs, `bisect.py`, and `NOTES.md`.

## Debrief

Three habits pay forward from here:

- Attribution is a one-function problem on a well-shaped archive. Chapter 4's `<iso-timestamp>_<trigger>_<git-sha-short>` naming is not an aesthetic choice — it makes `sorted(glob(...))` the whole "walk the history chronologically" implementation. The naming is the index.
- The walk's job is to preserve **why** and **when** across time. The *why* comes from the per-check trail chapter 2 preserves (`judge: adds unsupported claim`); the *when* comes from the manifest fields (`started_at`, `author`, `pr_number`); the walk composes them. If any one of the three — per-check reason, sortable run IDs, or manifest metadata — is missing, the still-failing list is back to "someone's problem."
- The `(trunk, no PR)` case is the one that routes differently. A still-failing row whose attribution names a trunk run with no PR is almost certainly drift in an ambient dependency — a prompt template, a model snapshot, a retrieval corpus — and routing it to a code author is wrong. Chapter 4's "model changed under us" section is the counterpart to this walk: the version-pin assertion catches the drift at the point it enters, and the bisect walk names it at the point someone reads a PR comment two weeks later.

The exercise next door (exercise 04) focuses on *new* regressions caused by an explicit model-swap PR. The stretch section asks you to bisect one still-failing row on top of that. If this lab's three attribution lines made sense, that stretch will too — the archive is the data, the walk is yours.

## Stretch

- **Handle the "flake and recover" row.** Add a sixth run where `sum-007` is suddenly passing again (a one-off flake). Run the walk. It still names run 3 as the introduction — which is correct for attribution, but a human reading the still-failing list for `sum-007` on a diff against that sixth run would be surprised to see it. Extend `bisect.py` with a second mode `--last-fail-after-pass` that names the *most recent* fail-after-pass transition, and write one sentence on when you want each mode. Chapter 4 explicitly flags "a row that fails once every ten runs is telling you the feature is genuinely non-deterministic" — the two modes together let you see both the original introduction and the latest recurrence.
- **Build a per-row index.** Precompute, on every run write, a tiny `row_history.jsonl` keyed by `row_id` with `[(run_id, pass)]` lists. The bisect walk becomes O(number of runs for one row) instead of O(all rows × all runs). This is the "maintain a cheap small index" the step 5 debrief hinted at.
- **Attribute by changed variable, not only by commit.** When the walk finds the flip at run 3, inspect the `code_versions` delta between runs 2 and 3. If the only variable that moved was `prompt_version`, print `introduced 2026-07-02 by @tliu (#458) via prompt summariser@v12 -> summariser@v13`; if the model moved, say so. This is the composition chapter 4 names: "The diff above should also compare `code_versions` from baseline vs. current run and, when a regression coincides with a `model` change or `prompt_version` change, note it."
- **Age the still-failing list.** For each still-failing row, add `(open for N days)` to the attribution line computed from `manifest['started_at']` and today. A still-failing row that has been open for 60+ days is a signal in its own right — either the team is OK with that row's failure (loosen the tolerance or delete the row), or it should have been fixed by now. Chapter 1's "when a row is deleted, log why" habit is the recipe for the first option.
