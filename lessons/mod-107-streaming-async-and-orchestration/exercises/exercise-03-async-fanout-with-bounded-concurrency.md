# Exercise 03 — Async fan-out with bounded concurrency

Paired with [chapter 3 — async fan-out with `asyncio` and `httpx`](../03-async-fanout-with-asyncio-and-httpx.md).

**Estimated effort:** 90–120 minutes.

## Objective

Build the reusable async fan-out helper you will use everywhere else in the track. Fire N LLM requests concurrently under a bounded semaphore, keep the per-request results aligned with the input list, enforce a per-item timeout, and cancel the whole batch cleanly when the caller (or the client behind the caller) goes away.

## Problem statement

Given a list of `N` short prompts (say, `N = 40` news-headline summarisation asks), your helper must:

- Send all `N` requests to the provider using its async client.
- Never have more than `concurrency` requests in flight at once (start with `concurrency = 8`).
- Return a list aligned with the input — `results[i]` corresponds to `prompts[i]`.
- On any single request failure or timeout, still return a result for every other prompt (partial success). Failed items appear as exception objects at their index, not silently dropped.
- On a `CancelledError` from the caller, cancel every in-flight request within roughly one second — do not wait for them to complete.

## Requirements

1. Implement `async def fanout(items, fn, *, concurrency=8, per_item_timeout=30.0)` matching chapter 3's shape.
2. Use `asyncio.Semaphore(concurrency)` for the cap. Do not use `asyncio.gather(*[...])` without a semaphore — that is a bug even at 40 items.
3. Wrap each item's coroutine in `asyncio.wait_for(..., per_item_timeout)`. On timeout, put a `TimeoutError` at the corresponding index and continue.
4. Return the aligned result list. `fanout(["a","b","c"], fn)` must return a list of length 3, with `results[0]` corresponding to `"a"`. This rules out shuffling by completion order.
5. Reuse a single `httpx.AsyncClient` (or a single `AsyncAnthropic` / `AsyncOpenAI` client) for the whole batch. Do not create a fresh client per item.
6. Prefer `asyncio.TaskGroup` (Python 3.11+) over `asyncio.gather`. If you must use `gather`, use `return_exceptions=True` and post-process.
7. Instrument the helper. Log, for each item: index, start timestamp, end timestamp, elapsed, success/failure. Print a summary at the end: total wall-clock, sum of per-item elapsed, and the ratio of the two (the ratio tells you how effective the concurrency was).

## Requirements — the cancellation test

Once the happy path works, prove cancellation is clean:

1. Spawn `fanout(...)` as a top-level task with `task = asyncio.create_task(fanout(...))`.
2. After 500 ms, call `task.cancel()`.
3. Confirm the task raises `CancelledError` within another ~500 ms.
4. Confirm your active-request logger shows no in-flight requests once the CancelledError propagates. If you see requests still completing after cancellation, your `async with` scopes are missing somewhere — the SDK client's request context is what closes the socket.

## Starter guidance

- `asyncio.TaskGroup`: <https://docs.python.org/3/library/asyncio-task.html#task-groups>
- `asyncio.Semaphore`: <https://docs.python.org/3/library/asyncio-sync.html#asyncio.Semaphore>
- `httpx.AsyncClient` and `httpx.Limits`: <https://www.python-httpx.org/async/>
- Anthropic async client: <https://github.com/anthropics/anthropic-sdk-python> (search for `AsyncAnthropic`)
- OpenAI async client: <https://github.com/openai/openai-python> (search for `AsyncOpenAI`)

Recommended prompt set for the load: 40 short headlines from a public RSS feed you already have handy, summarised to one sentence each. Keep the token counts small so the per-request cost stays under a dollar total.

Do not put `time.sleep` anywhere inside your coroutine — it blocks the event loop for every task, not just your own. Use `await asyncio.sleep`.

## Acceptance criteria

- On a happy-path run with `concurrency=8` and 40 prompts, total wall-clock is close to `40 / 8 = 5×` the median per-request latency, not close to the sum of all 40. If it is close to the sum, your executor is serialising — usually because you awaited outside the `async with sem:` block.
- Setting `concurrency=1` produces total wall-clock close to the sum. This is your sanity check.
- With one deliberately-injected prompt that raises (e.g. a system prompt so long it 400s), the helper still returns 40 items — 39 successes and one exception at the failing index.
- With `per_item_timeout=0.001`, every item's result is a `TimeoutError` and the total wall-clock is under ~2 seconds — the timeouts are actually stopping in-flight requests, not waiting for them.
- The cancellation test from above passes: `task.cancel()` after 500 ms produces a `CancelledError` within another 500 ms, and no requests finish after cancellation.
- Only one `AsyncAnthropic` / `AsyncOpenAI` client is created per run. Grep your code for its construction — if it is inside the per-item coroutine, fix it.

## Stretch goals

- Add an `async` context manager version: `async with FanoutRunner(concurrency=8) as runner: results = await runner.run(items, fn)`. The context manager owns the client construction/teardown and can be reused across multiple runs.
- Wrap the fan-out in a FastAPI endpoint that streams *progress* (not results) as SSE: `data: {"done": 12, "total": 40, "failed": 1}`. Confirm the browser sees progress updates in real time.
- Replace `asyncio.wait_for` per item with a global deadline: no request can push the total run past `T` seconds. On deadline, cancel all remaining in-flight tasks and return whatever completed. This is what a "return everything you have by 9 AM" batch job actually needs.
- Measure and print the median in-flight count during the run. If it is always at the semaphore limit, you are back-pressured; if it drifts below the limit, some other layer (`httpx.Limits`, provider rate limit) is throttling you.
- Rewrite the helper using `anyio` primitives instead of `asyncio` directly. Confirm it works under both the asyncio and trio backends.
