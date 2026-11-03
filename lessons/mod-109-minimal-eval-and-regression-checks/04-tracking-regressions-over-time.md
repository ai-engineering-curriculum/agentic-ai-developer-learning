# Chapter 4 — Tracking regressions over time and routing them to the right developer

Chapter 3 got you to "the eval fails on this PR." That is enough to block a merge. It is not enough to answer the two questions that hit you a week later:

> *When did this row start failing?* *Which change caused it?*

Without answers, the eval degrades into a static gate that only catches the very next regression. Old regressions accumulate silently: rows get skipped, thresholds get relaxed, and by month three the pass rate is 94% of a set that has been quietly bent to fit the current model. This chapter is about the small amount of scaffolding that keeps that from happening — persisted history, per-row baselines, a diff between runs, and enough attribution to route each regression to the specific developer whose change caused it.

## What "over time" actually needs

You do not need a hosted dashboard, a database, or an observability stack for a first-track eval. You need four things:

1. **A run record**, per full eval invocation, that captures every scored row's outcome plus the metadata a colleague would need to reproduce the run.
2. **A baseline pointer** — some way of saying "the last known-good run on the trunk branch is this one, use it for comparison."
3. **A diff** — for the current PR's run vs. that baseline, which rows are newly failing, which were failing and are now passing, and which are still failing.
4. **Attribution** — who caused each regression. Not literally "which human," but "which commit / prompt version / model version / retrieval snapshot," because the commit author reduces cleanly from there.

The rest of the chapter is how to build each one at mod-109 scale. Everything past this — a real time-series database of scores, per-row trend charts, alerting on rolling averages, dashboards that split cost / latency / quality by customer segment — belongs to `ai-eval-engineer-learning` and to whatever observability stack (Langfuse, OpenTelemetry, Datadog) your org has standardised on. See [chapter 5](05-boundary-to-eval-engineer-track.md).

## A run record: what to persist per invocation

The unit of history is a **run**. A run is one invocation of the eval — one PR's CI job, one nightly cron, one local `RUN_EVALS=1 pytest -m eval` a developer typed. Every run writes exactly one JSONL file. Every line of that file is one row's outcome.

```jsonc
// evals/summariser/runs/2026-08-01T14-22-17Z_pr-482_bfa1c04.jsonl
{"row_id": "sum-001", "pass": true,  "checks": [{"field": "reference_summary", "kind": "similarity", "pass": true, "score": 0.83}], "latency_ms": 812, "usage": {"input_tokens": 1204, "output_tokens": 87}}
{"row_id": "sum-002", "pass": false, "checks": [{"field": "must_mention", "kind": "keyword", "pass": false, "missing": ["Globex"]}, {"field": "reference_summary", "kind": "similarity", "pass": true, "score": 0.79}], "latency_ms": 743, "usage": {"input_tokens": 1190, "output_tokens": 71}}
...
```

Alongside the per-row lines, write **one manifest line** at the top or in a sibling file with the run's metadata:

```jsonc
// evals/summariser/runs/2026-08-01T14-22-17Z_pr-482_bfa1c04.manifest.json
{
  "run_id": "2026-08-01T14-22-17Z_pr-482_bfa1c04",
  "started_at": "2026-08-01T14:22:17Z",
  "finished_at": "2026-08-01T14:24:41Z",
  "git_sha": "bfa1c04c8f39...",
  "git_branch": "prompt-fix-globex-mention",
  "pr_number": 482,
  "author": "achen@example.com",
  "trigger": "pull_request",
  "feature": "summariser",
  "golden_set_sha": "d2e0f11ab7...",         // hash of golden.jsonl
  "code_versions": {
    "prompt_version": "summariser@v14",       // from src/summariser/prompt.py
    "model": "gpt-4o-mini-2024-07-18",
    "temperature": 0,
    "seed": 42,
    "sdk_versions": {"openai": "1.51.0"}
  },
  "totals": {"rows": 50, "passed": 47, "failed": 3, "cost_usd": 0.14}
}
```

Every field above is load-bearing:

- **`run_id`** — a sortable, unique string. `<iso-timestamp>_<trigger>_<git-sha-short>` is the shape that reads well in a directory listing and sorts chronologically without extra tooling.
- **`git_sha`** and **`author`** — you cannot attribute a regression without them. Read them from the CI environment (`GITHUB_SHA`, `GITHUB_ACTOR`, `CI_COMMIT_SHA`, `CI_COMMIT_AUTHOR`) — do not hand-compute.
- **`golden_set_sha`** — a hash of the golden JSONL file. Two runs against *different* golden sets are not comparable; the hash lets your diff refuse to compare across incompatible sets rather than silently produce misleading numbers.
- **`prompt_version`** and **`model`** — the two variables that most often move the score. Read them from wherever the runtime code reads them, so what the eval records is what the code actually used. If your prompt lives in a Python module, a versioned constant (`PROMPT_VERSION = "summariser@v14"`) that you bump when you edit the prompt is enough for this module. Chapter 5 discusses when to graduate to a real prompt-registry / management layer.
- **`totals.cost_usd`** — sum the `usage` block × per-token pricing across every model call in the run. When someone asks "how much has our eval cost this quarter?" the answer should sum from these fields, not from a guess.

Store runs *in the repo*, under `evals/summariser/runs/`, alongside the golden set. Two consequences:

- Every run is versioned and reviewable. `git log evals/summariser/runs/` reads as the score history of the feature.
- History is not lost when a hosted CI instance is torn down or a dashboard tool is decommissioned.

For a feature that runs tens of evals per week, this file directory reaches a few thousand files a year — well within Git's comfort zone, especially if you compress or archive the manifests older than, say, six months into a single `runs/archive-2026H1.jsonl`.

If you are already using an eval-tracking or observability tool (Langfuse, Braintrust, Arize Phoenix, MLflow) and it fits your infra, use it — the mod-109 goal is not the specific storage backend, it is that history is queryable at all. Just persist the same manifest fields the tool needs, so you are not locked into that tool's schema.

## A baseline: what "known-good" points at

The baseline is the run you compare a new run *against* to decide whether it regressed. Two shapes:

- **Trunk-baseline (recommended for mod-109).** The baseline is the most recent run whose `git_sha` is on the trunk branch (`main`) and whose eval passed on that trunk commit. Every PR's run diffs against that baseline. When a PR merges, its post-merge run on trunk becomes the new baseline.
- **PR-base-baseline.** The baseline is the last run against the PR's *merge base* commit. Slightly more accurate for long-lived branches; more moving parts.

Trunk-baseline is the default for a first module because it is trivially defined and there is exactly one baseline at any moment. A single tiny file records it:

```json
// evals/summariser/BASELINE.json
{"run_id": "2026-07-31T09-11-04Z_scheduled_a19c0f2", "git_sha": "a19c0f2..."}
```

The trunk workflow — a `push` to `main`, not a PR — runs the eval and, on success, rewrites `BASELINE.json` and commits it back to the repo (or writes it to an S3 bucket / a CI cache — the storage does not matter, the pointer does). Then every PR-triggered eval reads `BASELINE.json` and compares.

Two failure modes to design around:

- **A regressed trunk baseline.** If a bad change lands on main and updates the baseline, the new baseline is now the *lower* score, and PRs that would have caught the regression now pass. Guard against this by *only* updating `BASELINE.json` when the trunk run passes its own hard-coded floor (e.g. "≥ 45/50" for a 50-row set) — a hard bottom that the baseline can never sink below without a human-approved threshold change.
- **Golden set drift.** When you legitimately edit the golden set (add rows, fix tolerances), old baselines are no longer comparable. The diff step should refuse to compare across different `golden_set_sha` values and instead print `baseline is stale; regenerate on trunk`. That refusal is the correct behaviour, not a bug — the eval is telling you the comparison is not meaningful.

## The diff: what regressed vs. what fixed vs. what stayed the same

The diff is the artefact developers actually read on their PR. It answers three questions:

- **Newly failing rows** — rows that were `pass: true` in the baseline and are `pass: false` in this run. These are regressions and block the PR.
- **Newly passing rows** — rows that were `pass: false` in the baseline and are `pass: true` in this run. These are wins; surface them so improvements do not vanish.
- **Still-failing rows** — rows that failed in both. Not caused by this PR; do not block on them, but list them so the ambient failure count is visible.

A minimal diff computation is one dict lookup:

```python
def diff(baseline_rows: dict[str, dict], current_rows: dict[str, dict]) -> dict:
    common = baseline_rows.keys() & current_rows.keys()
    newly_failing = [row_id for row_id in sorted(common)
                     if baseline_rows[row_id]["pass"] and not current_rows[row_id]["pass"]]
    newly_passing = [row_id for row_id in sorted(common)
                     if not baseline_rows[row_id]["pass"] and current_rows[row_id]["pass"]]
    still_failing = [row_id for row_id in sorted(common)
                     if not baseline_rows[row_id]["pass"] and not current_rows[row_id]["pass"]]
    added   = sorted(current_rows.keys() - baseline_rows.keys())   # rows added since baseline
    dropped = sorted(baseline_rows.keys() - current_rows.keys())   # rows removed since baseline
    return {
        "newly_failing": newly_failing,
        "newly_passing": newly_passing,
        "still_failing": still_failing,
        "added": added,
        "dropped": dropped,
    }
```

The **PR comment** the CI posts should render this compactly. Something like:

```
Eval: summariser (50 rows)
  Baseline: 47/50 passing @ a19c0f2 on main (2026-07-31)
  This run: 44/50 passing @ bfa1c04

Regressions (3):
  - sum-014  must_mention  missing ['Globex']       — @achen (this PR)
  - sum-022  similarity     0.61 < 0.75              — @achen (this PR)
  - sum-031  schema         `date` is not a string   — @achen (this PR)

Fixed (0):

Still failing (3):
  - sum-007  judge          "adds unsupported claim" — introduced 2026-06-14 by @mvale (#417)
  - sum-019  keyword        missing ['Q3 revenue']   — introduced 2026-07-02 by @tliu (#458)
  - sum-041  similarity     0.72 < 0.75              — introduced 2026-07-25 by @achen (#476)
```

That format compresses three separate signals into one thing a developer can scan in ten seconds. Notes on the fields:

- The regression lines name the *sub-check that failed* — this is why chapter 2 preserves the per-check trail. `sum-014 failed` is not actionable; `sum-014 must_mention missing ['Globex']` is.
- The still-failing lines are attributed to *when the row started failing* and *whom* — because the still-failing list is where old failures hide, and naming the change that introduced them is the only reliable way to keep them from being ignored forever (see the *Routing* section below).

## Routing regressions to the developer who caused it

Two flavours of routing, one for new regressions and one for old.

### New regressions: the PR is the routing

For rows that are failing on this PR but were passing on the baseline, there is exactly one candidate change: the PR itself. The routing is free — the failure is on the PR, the PR has an author, the author is the person who has to fix it. The one thing worth automating: **make the developer's own PR comment tag them explicitly**. `@achen — regressions on 3 rows.` A reviewer waiting on the author to look at the eval failure should not have to say "did you see the eval comment?"

Two commonly missed practices:

- **A regression on a row someone else added is still on the PR author.** The row was passing at baseline. The PR broke it. "But I did not add that row" is not a valid rebuttal to a regression on a passing row.
- **A refactor is not a licence to skip.** If the PR is "a refactor with no behaviour change," and the eval regresses, the refactor changed behaviour. That is exactly the value the eval provides — a behavioural safety net a refactor might not have — and the fix is not to bypass, it is to figure out what changed.

### Old regressions: bisect + attribute

For rows on the still-failing list — a row that started failing sometime before this PR and is still failing — the routing is more work. You need to answer "which past commit made this row start failing?" so the still-failing list does not turn into a graveyard where every row is "someone's problem" and therefore nobody's.

The mechanic is a **`git bisect`** across the archive of run manifests: find the earliest run in which this row was failing, look at that run's `git_sha`, look at that commit's author. That is your attribution.

You do not have to bisect by hand. If your run records live in the repo (recommended above), a small script is enough:

```python
def when_did_row_start_failing(row_id: str, runs_dir: Path) -> tuple[str, str] | None:
    """
    Walk manifests oldest -> newest. Return (run_id, git_sha) of the first run
    in which row_id failed after previously passing. None if never failed or
    always failed.
    """
    prev_pass = None
    for manifest in sorted(runs_dir.glob("*.manifest.json")):
        rows = {r["row_id"]: r for r in _read_jsonl(manifest.with_suffix(".jsonl"))}
        if row_id not in rows:
            continue
        currently_passing = rows[row_id]["pass"]
        if prev_pass is True and currently_passing is False:
            data = json.loads(manifest.read_text())
            return data["run_id"], data["git_sha"]
        prev_pass = currently_passing
    return None
```

Once you have the SHA, `git log --format='%an %ae' -1 <sha>` gives you the author. The rendered PR comment above uses exactly this: `introduced 2026-07-25 by @achen (#476)` reads as "here is when it started, here is who caused it, here is the PR to look at."

For a running feature this closes the loop — every failure eventually points at a specific PR, and the still-failing list is a set of open problems with owners, not a scoreboard of anonymous failures.

### Route by prompt / model change, not just by commit author

Not every regression is caused by a code change. A row can start failing because the provider updated a pinned model (this is why the manifest captures `model` with the *dated version string*, not just `gpt-4o-mini`), because the prompt template changed (versioned constant), because a retrieval snapshot changed (upstream data ingest). The diff above should also compare `code_versions` from baseline vs. current run and, when a regression coincides with a `model` change or `prompt_version` change, note it:

```
Regressions (3):
  - sum-014  must_mention  missing ['Globex']       — @achen (this PR)
    (prompt: summariser@v13 -> summariser@v14 on this PR)
```

For a regression that is *not* caused by the PR — the classic case being a provider silently rolling their pinned model forward — the "route to the PR author" reflex is wrong. Route to the maintainer of the version pin instead. See the next section.

## The "the model changed under us" case

The most easily-missed regression source is a model rolling forward. You pinned `gpt-4o-mini-2024-07-18` in your `prompt.py` and, six months later, the provider retires that snapshot. Your CI-only API key auto-falls-back to `gpt-4o-mini-2024-11-20`. Your next eval run drops the score by two points and nobody knows why, because no PR touched the model.

Two protections:

- **Fail fast on model change.** In the CI job (or in a small pre-eval assertion), read the exact `model` string used and compare against the pinned string in your config. If they differ, fail the run with `model changed from X to Y — this is not a regression, this is a version-pin issue; owner: @<maintainer of version-pin config>`.
- **When the pin has to change** — because the pinned snapshot is being retired, or because you are intentionally upgrading — treat the swap as its own PR whose entire job is the model change. Its eval diff *is* the regression report for that change. This is exactly what exercise 04 asks you to build.

The same shape applies to provider deprecations, SDK major-version bumps, and retrieval-corpus refreshes. Every ambient dependency the eval depends on gets pinned; every change to the pin ships as its own PR whose job is the diff.

## Alerting outside of PRs

For features that run continuously, add a **scheduled trunk eval** — a nightly (or per-commit-to-trunk) run of the same suite. The point is not to run more evals; it is to detect drift that never shows up on a PR:

- A prompt engineer edits a shared prompt template that this feature uses transitively.
- A provider silently changes model behaviour on the pinned snapshot (rare, but happens).
- A retrieval snapshot / ingested corpus changes upstream.

The nightly's failure mode is different from a PR eval's. Route it as an alert to a team channel (Slack, Discord, PagerDuty for higher-severity setups) with the same diff format, plus the attribution walk. A nightly failure with `introduced 2026-07-25 by @achen (#476)` is a targeted follow-up; a nightly failure with just "the number dropped" is a noisy alarm nobody will act on.

The exact routing tool is not the point. The point is: the same diff you post on a PR is the diff you post to the channel, and the same attribution walk names an owner.

## What good looks like

After chapter 4 has been running on a feature for a month:

- Every run of the eval is on disk, hashed, and reproducible. Nothing is "just in the CI logs."
- Every PR that changes the feature gets a comment showing regressions, wins, and still-failing rows, tagged to authors.
- The still-failing list points at specific PRs and their authors. It is short because it is visible.
- A model-pin change is a PR of its own whose diff is the regression report for that change.
- Cost is a summed number from the manifests, not an estimate.

That is what "tracking regressions over time" means at mod-109 altitude. Everything past it — real time-series dashboards, per-segment quality metrics, online eval against live traffic — is `ai-eval-engineer-learning`. But the mechanics above will carry a first feature for a long time before you graduate.

## Summary

- Persist one run record per eval invocation: a JSONL of per-row outcomes plus a manifest with git SHA, author, PR number, golden-set hash, prompt version, model version, and cost. Store it in the repo.
- Maintain a `BASELINE.json` that points at the last known-good trunk run. Update it only when a trunk run clears a hard-coded floor, so a bad trunk change cannot silently drag the baseline down.
- Diff every PR run against the baseline. Report newly failing, newly passing, and still failing separately, with the specific failed sub-check per row.
- Route new regressions to the PR author automatically. Route old regressions by bisecting the run history to the commit / prompt version / model version that introduced them, and name that author.
- Treat every ambient dependency change (model pin, prompt template, retrieval snapshot) as its own PR whose diff is its regression report. A silent model roll-forward is the most easily missed source of drift; guard against it with a version-pin assertion.
- Schedule a trunk-nightly eval that catches drift no PR touched, and route its failures with the same diff shape.

The last chapter draws the line between what this module deliberately covers and where the deep methodology — LLM-as-a-judge calibration, RAG eval, online eval, observability stacks, cost / latency / quality dashboards — lives.
