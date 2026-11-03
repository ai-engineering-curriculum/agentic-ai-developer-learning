# Exercise 04 — Retry with backoff and jitter

Paired with [chapter 4 — retrying transient failures without amplifying real bugs](../04-retrying-transient-failures.md).

**Estimated effort:** 60–90 minutes.

## Objective

Implement `retry_with_backoff` — a small, correct wrapper that retries transient failures with exponential backoff plus full jitter, respects `retry-after` headers when present, and refuses to retry terminal errors. Prove it with a fault-injecting fake server so you never have to reproduce a rate limit against a real provider.

## Problem statement

Build a retry helper with the following contract:

```python
async def retry_with_backoff(
    do_request,                     # zero-arg async callable that returns the response
    *,
    max_attempts: int = 5,
    base: float = 1.0,
    cap: float = 30.0,
    transient: set[int] = {408, 429, 500, 502, 503, 504, 529},
) -> httpx.Response: ...
```

Behaviour:

- On a transient failure (HTTP status in `transient`, or `httpx.ConnectError` / `httpx.ReadTimeout` / `httpx.RemoteProtocolError`), sleep and retry — up to `max_attempts - 1` retries.
- On a terminal failure (any other HTTP status, including 400 / 401 / 403 / 404 / 422), raise immediately. **Do not retry.**
- Sleep duration for retry `n` (zero-indexed): `random.uniform(0, min(cap, base * 2 ** n))`. If the response has a numeric `retry-after` header, use that instead (with a small additional random offset if you want extra safety).
- On the final attempt, do not sleep. Raise the last exception.
- Log every attempt: attempt number, status or exception type, delay applied for the *next* attempt (or `None` on the last).

Then, build a `fake_server` (any small local HTTP server — FastAPI, `aiohttp`, `starlette` — will do) that produces controllable failure sequences on demand. Use it to prove the helper's behaviour without touching a real provider.

## Requirements

1. Implement `retry_with_backoff` per the contract above. Use `asyncio.sleep`, not `time.sleep`.
2. The list of transient status codes is a parameter, not a hard-coded set. Callers must be able to add or remove codes for their own workloads.
3. On the `retry-after` header:
   - Parse it as seconds (integer or float).
   - If present, use it in place of the jittered exponential delay.
   - If it is an HTTP-date string, fall back to the jittered delay rather than parsing the date (note the limitation in a comment).
4. On the final attempt, re-raise the last exception verbatim. Do not wrap it in a generic `RuntimeError`.
5. Build a small `fake_server` with routes that let you configure the response for the next N requests via a control endpoint. Suggested routes:
   - `/queue?statuses=429,429,200` — the next three requests get 429, 429, 200 in order.
   - `/queue?statuses=429,429,429&retry_after=2` — the 429s carry `retry-after: 2`.
   - `/queue?statuses=400` — a single terminal 400.
   - `/reset` — clear the queue.
6. Run the helper against the fake server for the four test scenarios below and assert on the behaviour.

## Requirements — the four assertions

1. **All-transient-then-success.** Queue `[429, 429, 200]` with no `retry-after`. `retry_with_backoff(...)` returns the 200 response. It made 3 total attempts. The total elapsed time is > 0 (delays applied) and < `base * (2^0 + 2^1) + a small ceiling` (delays capped).
2. **Respects `retry-after`.** Queue `[429, 429, 200]` with `retry-after: 2` on each 429. Total elapsed is at least ~4 seconds (2 + 2) and not much more.
3. **Terminal error is not retried.** Queue `[400]`. `retry_with_backoff(...)` raises after exactly 1 attempt, with the original 400 exception. No sleep occurred.
4. **Exhaustion.** Queue `[500, 500, 500, 500, 500]` with `max_attempts=3`. The helper makes 3 attempts and then re-raises the 500 exception. It did not make 4 or 5 attempts.

## Requirements — the jitter check

Run `retry_with_backoff` 100 times against a queue that always returns `429`, `429`, `200` with `max_attempts=3` and no `retry-after`. Collect the total elapsed time from each run. Assert that:

- The distribution is *not* clustered around one value. Standard deviation of the elapsed times is at least ~30% of the mean.
- Some runs finish quickly (< 1 s total sleep), and some take close to the cap. If the histogram is a single tight spike, you are not jittering.

This is the "thundering herd defence" check. Without it, ten clients would all retry at exactly the same time.

## Starter guidance

- AWS jittered backoff article: <https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/>
- OpenAI error codes: <https://platform.openai.com/docs/guides/error-codes>
- Anthropic errors: <https://docs.anthropic.com/en/api/errors>
- `tenacity` (mature retry library — read its source once for reference, don't necessarily import it): <https://tenacity.readthedocs.io/>

Do not use the SDK's built-in retries for this exercise — the point is to write and test the policy yourself. If your SDK client is already retrying, set `max_retries=0` on the client so your loop is the only retrier.

## Acceptance criteria

- The four assertion scenarios all pass on your helper.
- The jitter distribution has visible spread (std dev ≥ 30% of mean, or eyeball a histogram — the spread should be obvious).
- Grepping your codebase for `time.sleep` in the retry helper returns no matches. It is `asyncio.sleep` throughout.
- Terminal errors (400, 401, 403, 404, 422) never trigger a retry, in any code path.
- On exhaustion, the exception raised is the *original* `httpx.HTTPStatusError` (or the transport exception), not a wrapper. Callers upstream can pattern-match on the exception type without special-casing your helper.
- Per-attempt logs are complete: attempt number, status/exception, delay for next attempt (or `None`). Enough to reconstruct a retry sequence from the log alone.

## Stretch goals

- Add a **circuit breaker**: after `M` consecutive failures across all calls (not per-call), `retry_with_backoff` refuses to attempt anything for `cooldown` seconds. Test it against a `fake_server` scripted to fail for 60 seconds straight.
- Add an **idempotency key** to every retried request. Generate one UUID per *logical* request; pass it as `Idempotency-Key` on every attempt. Confirm your fake server sees the same key on all attempts of the same request and different keys across requests.
- Compare your helper against `tenacity.AsyncRetrying(...)` on the same test scenarios. Note where the semantics differ (usually around what counts as "the last attempt" and how `retry-after` is respected).
- Wire the helper into the fan-out from exercise 03: `fanout(items, fn, ...)` where `fn` is itself wrapped in `retry_with_backoff`. Run a batch with a fake server that randomly returns 429s to a third of requests. Confirm the batch completes with no failures (retries absorbed all of them) and that total wall-clock is only modestly longer than the no-failure baseline.
