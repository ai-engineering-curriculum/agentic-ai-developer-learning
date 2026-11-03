# mod-107 — Streaming, Async, and Parallel LLM Orchestration

A module on the Agentic AI Developer track, building on mod-101 through mod-103. So far you have driven the model in single, blocking request/response turns. Real products stream tokens as they are generated, fan out many requests in parallel, retry on transient failures, and measure their own tail latency honestly. This module is about the wire and the runtime — everything that sits between "the model produced an answer" and "the user actually saw it, fast."

By the end of this module you can consume a Server-Sent Events (SSE) stream from OpenAI or Anthropic and render partial output as it arrives, safely parse structured output that is still being generated, orchestrate concurrent LLM calls with `asyncio` and `httpx` under a bounded concurrency limit, distinguish retriable failures from terminal ones, and report the honest p50 / p95 / p99 latency of a streaming feature to a product manager who deserves the honest number.

**Estimated effort:** ~12 hours (chapters ~2 hours; exercises 8 hours; the rest is time reading provider docs and staring at your own latency histograms).

## Prerequisites

- You have finished [`mod-102-prompt-engineering`](../mod-102-prompt-engineering/README.md) and [`mod-103-tool-and-function-calling`](../mod-103-tool-and-function-calling/README.md). Streaming a tool-call turn is the point where both prior modules meet — trying to learn it without the loop mechanics from mod-103 is painful.
- Comfort in Python at the level of `async def`, `await`, `asyncio.gather`, and `try / except` around I/O. If `asyncio` is genuinely new, work through the standard library's coroutines-and-tasks page before starting chapter 3.
- A working install of the OpenAI Python SDK **or** the Anthropic Python SDK, plus `httpx` for the async exercises. One provider is enough; every chapter shows the same pattern on both.
- Roughly USD $1–2 of API credit. Streaming and fan-out don't cost more per token than blocking calls — you will just make more of them.

## Learning objectives

After finishing this module you will be able to:

1. Consume Server-Sent Events (SSE) streams from OpenAI and Anthropic and render partial output to a console or HTTP client without buffering to completion.
2. Stream partial structured output (JSON) safely — recognise when a partial parse is legal versus when it will mis-render.
3. Orchestrate parallel LLM calls with async Python (`asyncio` + `httpx`) — fan out N requests, gather with a bounded concurrency limit, cancel cleanly on client disconnect.
4. Retry with exponential backoff plus jitter on transient failures; distinguish transient (rate-limit, 5xx, timeout) from terminal (400 context length, 401 auth) errors so retries do not amplify a real bug.
5. Measure p50 vs. p95 vs. p99 latency for a streaming feature and report the honest number — streaming feels faster than it is, and the difference between time-to-first-token and time-to-completion matters.

## Chapters

Read in order. Each chapter maps to one learning objective.

| # | Chapter | Objective |
|---|---|---|
| 1 | [Consuming SSE streams from OpenAI and Anthropic](01-consuming-sse-streams.md) | Objective 1 |
| 2 | [Streaming partial structured output safely](02-streaming-partial-json.md) | Objective 2 |
| 3 | [Async fan-out with `asyncio` and `httpx`](03-async-fanout-with-asyncio-and-httpx.md) | Objective 3 |
| 4 | [Retrying transient failures without amplifying real bugs](04-retrying-transient-failures.md) | Objective 4 |
| 5 | [Measuring streaming latency honestly](05-measuring-streaming-latency.md) | Objective 5 |

## Exercises

Short hands-on drills paced to the chapters. Do the matching exercise **after** its chapter but **before** starting the next.

| # | Exercise | Chapter |
|---|---|---|
| 01 | [First SSE stream consumer](exercises/exercise-01-first-sse-stream-consumer.md) | 1 |
| 02 | [Streaming partial JSON](exercises/exercise-02-streaming-partial-json.md) | 2 |
| 03 | [Async fan-out with bounded concurrency](exercises/exercise-03-async-fanout-with-bounded-concurrency.md) | 3 |
| 04 | [Retry with backoff and jitter](exercises/exercise-04-retry-with-backoff-and-jitter.md) | 4 |
| 05 | [Measure time-to-first-token and tail latency](exercises/exercise-05-measure-time-to-first-token-and-tail-latency.md) | 5 |

## Labs and quizzes

`labs/` and `quizzes/` are reserved for long-form hands-on work and knowledge checks authored in later cycles. If they are still empty when you get here, the exercises above are enough to cement the objectives.

## Resources

Real, citable references for the topics in this module — provider streaming docs, the SSE specification, `asyncio` / `httpx` documentation, retry practices. See [`resources.md`](resources.md).

## How to work through this module

1. Read chapter 1 and do exercise 01. Streaming is easier to feel than to describe — get one working stream in front of your eyes before reading further.
2. Read chapter 2 and do exercise 02. Partial JSON is where "seems to work" and "actually works" diverge; the exercise is designed to expose the gap.
3. Read chapter 3 and do exercise 03. Bounded concurrency is a two-line change that decides whether your service melts under load or not.
4. Read chapter 4 and do exercise 04. This is the shortest chapter and the one every production incident review in this domain eventually references.
5. Read chapter 5 and do exercise 05. Ship this exercise's script; the p50/p95/p99 numbers it produces are the ones you will quote in every design review for the rest of the track.

## What comes next

`mod-108-model-selection-cost-and-prompt-caching` picks up cost — how to keep the round-trip bill from the loops and fan-outs you built here from ballooning as usage grows. Mod-104 (retrieval) and mod-109 (evals) build on the streaming and async foundations from this module. If you want to build systems where a single request coordinates many models over many turns, [`mod-105-first-agent`](../mod-105-first-agent/README.md) and the next rung, [`agentic-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning), are where that lives — but everything there assumes you can already stream, fan out, and measure like this module teaches.
