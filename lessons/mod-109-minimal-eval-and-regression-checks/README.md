# mod-109 — Minimal Evaluation and Regression Checks for LLM Features

A module on the Agentic AI Developer track. Mod-102 taught you how to shape a prompt, mod-103 how to drive tool calls in a loop, mod-107 how to stream and orchestrate concurrent requests, mod-108 how to defend a model choice with numbers, mod-104 how to ground an answer in your own documents. This module is the one that keeps all of that working after you ship it: build a small golden set for the feature, wire it into pytest, gate it in CI, track the score across model and prompt changes, and route every regression to the developer who caused it. It is small, it is unfashionable, and it is what separates an LLM feature that stays good from one that quietly degrades until a user files a bug.

By the end of the module you can build a 20–50 example golden set with a defended sampling strategy and per-row tolerance, score each row with the right mix of rule-based / similarity / LLM-as-a-judge scorers (and know what each costs you), run the set as a required CI check on every prompt or model PR, and produce a regression report that names the specific commit / model / prompt version behind each new failure.

**Estimated effort:** ~8 hours (chapters ~2 hours; four exercises 5–6 hours; the rest is time reading provider docs and eyeballing your own eval output).

## Prerequisites

- You have finished [`mod-102-prompt-engineering`](../mod-102-prompt-engineering/README.md) — chapter 4 (schema-constrained JSON output) is what makes rule-based schema conformance a cheap default, and chapter 7 (iterating with evals) is where the "change one thing at a time" habit came from.
- You have finished [`mod-108-model-selection-cost-and-prompt-caching`](../mod-108-model-selection-cost-and-prompt-caching/README.md) — the cost estimation shape carries into exercise 02 and exercise 04.
- Comfortable running `pytest` at the terminal and reading its output. If parametrisation is new, the pytest parametrize how-to is on the reading list — <https://docs.pytest.org/en/stable/how-to/parametrize.html>.
- A CI provider you can trigger a workflow on (GitHub Actions, GitLab CI, CircleCI, Buildkite — any of them). If you cannot get one for this module, exercise 02 explains how to meet the same criteria with a local shell script and treat the CI wiring as documented-not-yet-run.
- Roughly USD $2–5 of API credit across the whole module. Rule-based scoring is free; similarity scoring is cheap; the exercise-03 judge-calibration pass is where most of the cost sits.
- One LLM-backed feature to eval. If you do not have one at hand, the JSON-extractor from mod-102 exercise 02, the tool-loop agent from mod-103 exercise 02, or the grounded Q&A feature from mod-104 exercise 02 all work.

## Learning objectives

After finishing this module you will be able to:

1. **Author a golden set** of 20–50 examples for one LLM-backed feature — inputs, expected outputs, per-row tolerance — with the sampling strategy defended in one paragraph.
2. **Run the golden set as a CI check using pytest** so a prompt or model change cannot silently regress the feature, with the eval marked as a required merge check.
3. **Score outputs with the three families** — rule-based (exact / regex / keyword / schema / length), embedding similarity against a reference answer, and one calibrated LLM-as-a-judge check — knowing the cost, coverage, and calibration trade-off of each.
4. **Track quality regressions over time** across model / prompt changes, with a persisted run history, a trunk baseline, and a per-PR diff that routes each regression to the specific commit / model / prompt version that caused it.
5. **Draw the boundary to `ai-eval-engineer-learning`** (level 30): LLM-as-a-judge calibration methodology, RAG-specific eval, online eval, observability / tracing (Langfuse, OpenTelemetry, Datadog), and cost / latency / quality dashboards all belong there. Do not try to reproduce that depth here; graduate to the eval track when the feature count grows or a product decision hinges on the eval number.

## Chapters

Read in order. Each chapter maps to one learning objective.

| # | Chapter | Objective |
|---|---|---|
| 1 | [Authoring a golden set](01-authoring-a-golden-set.md) | Objective 1 |
| 2 | [Scoring outputs: rule-based, similarity, and judge](02-scoring-outputs.md) | Objective 3 |
| 3 | [Running the eval in pytest and CI](03-running-the-eval-in-pytest-and-ci.md) | Objective 2 |
| 4 | [Tracking regressions over time and routing them to the right developer](04-tracking-regressions-over-time.md) | Objective 4 |
| 5 | [The boundary to `ai-eval-engineer-learning`](05-boundary-to-eval-engineer-track.md) | Objective 5 |

## Exercises

Hands-on drills paced to the chapters. Do each exercise **after** its chapter and **before** starting the next.

| # | Exercise | Chapter |
|---|---|---|
| 01 | [Author a golden set](exercises/exercise-01-author-a-golden-set.md) | 1 |
| 02 | [Pytest eval in CI](exercises/exercise-02-pytest-eval-in-ci.md) | 3 |
| 03 | [Rule vs. similarity vs. judge scoring](exercises/exercise-03-rule-vs-similarity-vs-judge-scoring.md) | 2 |
| 04 | [Regression report for a model swap](exercises/exercise-04-regression-report-for-a-model-swap.md) | 4 |

Exercise 02 pairs with chapter 3 rather than chapter 2 on purpose — you want the pytest suite running first, using your rule-based scorers only, so exercise 03 can extend it with similarity and judge without building the scaffolding at the same time.

## Labs and quizzes

`labs/` and `quizzes/` are reserved for long-form hands-on work and knowledge checks authored in later cycles. If they are still empty when you get here, the exercises above are enough to cement the objectives.

## Resources

Real, citable references for the topics in this module — pytest parametrisation and marks, provider structured-output and pricing pages, JSON Schema, CI provider path filters, and the eval-track handoff pointers. See [`resources.md`](resources.md).

## How to work through this module

1. Read chapter 1, then do exercise 01. The 20–50 rows of `input`, `expected`, `tolerance` are the artefact everything else in the module compounds on. Skimping here breaks the rest.
2. Read chapter 2. Do not do a scoring-comparison exercise yet — you need the pytest scaffolding first.
3. Read chapter 3 and do exercise 02. This is the load-bearing habit: your golden set is now a real gate in CI, running on every relevant PR, with your row IDs in the pytest output and the failure surfaced back to the PR.
4. Do exercise 03. On the same feature, score ~10 rows three ways, calibrate the judge minimally, and adopt the outcome by editing the tolerance blocks. This is where "know the trade-offs of each scoring family" moves from a table in a chapter to a defence you can give from data on your own feature.
5. Read chapter 4 and do exercise 04. Swap the model (or the prompt), run the eval twice, and produce a real regression report that a PM could read. This is the shape every future model / prompt change ships in.
6. Read chapter 5. It is short. It tells you where the deep material lives (`ai-eval-engineer-learning`) and what signals mean you are ready to graduate to it — save yourself three months of accidentally reinventing the eval track's material.

## What comes next

`mod-106-llm-app-deployment` is next. It is where mod-102 through mod-109 stop being separate skills and start being the checklist for shipping a first end-to-end LLM feature: prompt discipline, tool use where needed, model choice defended by numbers, retrieval only when the model needs facts it does not know, and — this module's contribution — a golden-set-plus-CI gate that keeps the feature working after it ships.

The specialist track [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) picks up every topic this module deliberately does not go deep on — LLM-as-judge calibration at depth, RAG eval, online eval on live traffic, observability / tracing, statistical rigour on eval numbers, agent-trajectory eval. When you have more than one feature to eval, when judges start multiplying, or when a product decision hinges on the eval number, that is where you go.
