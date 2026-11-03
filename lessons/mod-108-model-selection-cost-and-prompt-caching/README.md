# mod-108 — Model Selection, Cost, and Prompt Caching

A module on the Agentic AI Developer track. So far you have shaped prompts (mod-102), called tools in a loop (mod-103), and streamed / fanned out / retried (mod-107). Everything you have built runs against whatever model you picked on day one. This module is where you defend that pick — with numbers — and where you learn the two levers that decide whether your LLM feature is affordable at 10× the traffic: **the model you route each request to** and **how much of the prompt the provider gets to keep in cache instead of billing you for every time**.

By the end of this module you can compare small vs. frontier models across at least two providers on a real task, estimate the dollar cost of a feature call before you deploy it, enforce a per-request and per-user token budget at the application boundary, turn on and measure a provider's prompt-caching mechanism, route requests between models by task shape (cascade / fast-first-then-escalate / batch), and describe what graceful degradation looks like when a provider is down.

**Estimated effort:** ~10 hours (chapters ~2 hours; five exercises ~8 hours; the rest is reading provider pricing pages and staring at your own dashboards).

## Prerequisites

- You have finished [`mod-102-prompt-engineering`](../mod-102-prompt-engineering/README.md), [`mod-103-tool-and-function-calling`](../mod-103-tool-and-function-calling/README.md), and [`mod-107-streaming-async-and-orchestration`](../mod-107-streaming-async-and-orchestration/README.md). The token-counting content from mod-101 chapter 3 and the async fan-out from mod-107 chapter 3 are load-bearing here.
- Comfort in Python at the level of writing a script, reading provider SDK response objects, and computing a small histogram.
- A working install of **both** the OpenAI Python SDK and the Anthropic Python SDK (one provider is not enough for the small-vs-frontier and provider-outage exercises).
- API keys for both providers, and roughly USD $5 of credit across them. Prompt-caching and cascade-routing exercises make more calls than mod-102 did — still cheap, but budget accordingly.

## Learning objectives

After finishing this module you will be able to:

1. Compare small vs. frontier models across at least two providers on a real feature — accuracy delta, cost delta, latency delta — and defend the choice in one page.
2. Estimate the cost per feature call before you deploy, and enforce a per-request and per-user token budget at the application boundary.
3. Use provider prompt-caching mechanisms (Anthropic prompt caching, OpenAI cached input, comparable) and measure the cache-hit ratio in a real workload.
4. Route requests between models by task shape — cascade routing, fast-model-first-then-escalate, and batch APIs where latency is not user-facing.
5. Read a provider status page and reason about cost / latency / availability SLO trade-offs; describe what graceful degradation looks like when a provider is down.

## Chapters

Read in order. Each chapter maps to one learning objective.

| # | Chapter | Objective |
|---|---|---|
| 1 | [Small vs. frontier: a decision, not a default](01-small-vs-frontier-model-choice.md) | Objective 1 |
| 2 | [Cost per feature call and enforcing token budgets](02-cost-per-call-and-token-budgets.md) | Objective 2 |
| 3 | [Prompt caching and cache-hit ratio](03-prompt-caching-and-cache-hit-ratio.md) | Objective 3 |
| 4 | [Routing by task shape: cascade, fast-first, batch](04-routing-cascade-and-batch.md) | Objective 4 |
| 5 | [Status pages, SLOs, and graceful degradation](05-status-pages-slos-and-graceful-degradation.md) | Objective 5 |

## Exercises

Short hands-on drills paced to the chapters. Do the matching exercise **after** its chapter and **before** starting the next.

| # | Exercise | Chapter |
|---|---|---|
| 01 | [Cost per feature call — estimate before you ship](exercises/exercise-01-cost-per-feature-call-estimate.md) | 2 |
| 02 | [Small vs. frontier A/B benchmark](exercises/exercise-02-small-vs-frontier-a-b-benchmark.md) | 1 |
| 03 | [Prompt caching hit ratio on a real workload](exercises/exercise-03-prompt-caching-hit-ratio.md) | 3 |
| 04 | [Cascade routing: cheap then expensive](exercises/exercise-04-cascade-routing-cheap-then-expensive.md) | 4 |
| 05 | [Provider outage — graceful degradation](exercises/exercise-05-provider-outage-graceful-degradation.md) | 5 |

## Labs and quizzes

`labs/` and `quizzes/` are reserved for long-form hands-on work and knowledge checks authored in later cycles. If they are still empty when you get here, the exercises above are enough to cement the objectives.

## Resources

Real, citable references for the topics in this module — provider pricing pages, prompt-caching docs, batch API docs, status pages, and the arithmetic behind cost planning. See [`resources.md`](resources.md).

## How to work through this module

1. Read chapter 1, then do exercise 02. The A/B benchmark is the assignment that turns "the frontier model is smarter" from a slogan into a table with numbers on it. Everything else in the module leans on that habit.
2. Read chapter 2 and do exercise 01. The cost-estimate exercise is short but load-bearing — you should never ship an LLM feature without doing it once, and doing it once teaches you the reflex.
3. Read chapter 3 and do exercise 03. Prompt caching pays for itself the day you turn it on if your feature has a stable system prompt; measuring the hit ratio is what turns "we enabled it" into "we know it's working."
4. Read chapter 4 and do exercise 04. Cascade routing is where cost curves stop being linear in traffic — a five-line change that halves the bill for a class of features.
5. Read chapter 5 and do exercise 05. You will not learn what graceful degradation feels like until you have deliberately taken your primary provider away from your own app.

## What comes next

`mod-104-retrieval-basics` is next. Retrieval is where prompts get much longer (chunks of documents inline), so every lever you learn in this module — small-model routing, prompt caching, budget enforcement — becomes more valuable, not less, the moment you turn RAG on. If you skip mod-108 and go straight to retrieval, expect your first bill to be a surprise.
