# Chapter 4 — Retrying transient failures without amplifying real bugs

Every LLM API you call is, occasionally, going to fail for reasons that have nothing to do with your code. The remote model is overloaded, a network hop drops the connection, a rate limiter fires. These are transient failures — retry them and they usually succeed on the second or third attempt. Every LLM API is also, occasionally, going to fail for reasons that are entirely your fault: your prompt is longer than the model's context window, your API key expired, your schema is malformed. Retrying these does nothing but pay for the failure four times instead of once.

The whole content of this chapter is telling those two classes apart and building a retry policy that helps the first without amplifying the second.

## Motivation

A naive retry loop is one of the cheapest ways to turn a small provider incident into a self-inflicted DoS. Every client, on the same failure, retries at the same second. The provider recovers, gets hit with a synchronised burst of retries, falls over again. This is called a thundering herd, and jittered backoff is the standard defence.

At the other extreme, a retry loop that retries everything — including the 400 that says "your input is invalid" — burns time and money and often masks the real bug until it shows up in production as "we sometimes charge users four times for one failed booking."

## The taxonomy: transient vs terminal

Every failure your LLM call can produce belongs to one of two categories.

### Transient — safe to retry

- **429 Too Many Requests.** Provider rate limit. Retry after the delay in `retry-after` (if present) or a jittered backoff.
- **500 Internal Server Error / 502 Bad Gateway / 503 Service Unavailable / 504 Gateway Timeout.** Something on the provider's side is unhealthy. Retry.
- **Connection errors** — `httpx.ConnectError`, `httpx.ReadTimeout`, `httpx.RemoteProtocolError`. Network hop failed. Retry.
- **Provider-specific overload signals.** Anthropic returns `overloaded_error` (typically as an HTTP 529 or a `type: "overloaded_error"` payload). Treat as transient. Reference: <https://docs.anthropic.com/en/api/errors>.
- **`context_length_exceeded` on OpenAI** *only if* your code has just trimmed the transcript. Otherwise treat as terminal — see below.

### Terminal — do not retry

- **400 Bad Request** with a schema or context-length error. Your request is wrong. Retrying sends the same wrong request.
- **401 Unauthorized / 403 Forbidden.** Wrong or missing credentials. Retrying with the same key does not fix authentication.
- **404 Not Found.** The endpoint or model does not exist for you.
- **413 Payload Too Large** with the same payload. Trim the payload; do not retry as-is.
- **422 Unprocessable Entity.** Body was valid JSON but not valid for the endpoint. Fix the body.
- **`invalid_api_key`, `model_not_found`, `permission_denied`.** Configuration problems. Human-fix, not machine-retry.

The single sentence that captures it: **retry when the state that caused the failure might have changed on the server; do not retry when the state that caused the failure is in your request.**

### The one class that is neither: idempotency-sensitive failures

A retry only makes sense if the operation is safe to re-do. For LLM chat completions, this is nearly always true — you are generating a new response either way. But if you have chained a tool call that had a *side effect* (booking a room, charging a card), a client-side retry can double-book. Rules to follow:

- **Idempotent operations** (read-only tools, chat completions, embeddings) are safe to retry freely.
- **Non-idempotent operations** need an idempotency key so the server (or your tool) can de-duplicate. OpenAI supports an `Idempotency-Key` header on many operations. Where the server does not offer one, deduplicate on your side by request hash.

Reference: OpenAI idempotency notes appear in the API reference for request headers. See <https://platform.openai.com/docs/api-reference> and search for "Idempotency-Key."

## Exponential backoff plus jitter

The correct backoff formula is short:

```
delay_n = min(cap, base * 2 ** n)          # exponential
delay_n = random.uniform(0, delay_n)       # full jitter
```

Two decisions to make:

- **`base`** — the first-retry delay. 0.5 s to 2 s is typical. Too small and you hit the provider again before it recovered; too large and you waste time on failures that would have cleared in a second.
- **`cap`** — the maximum single delay. 30 s to 60 s is typical. Without a cap, a 6-attempt backoff can wait several minutes on the last try.

**Full jitter** — picking uniformly between 0 and the current exponential delay — is what breaks the synchronised-retry pattern. If ten clients all hit the same 429 at the same instant, exponential-only sends them all again at exactly `base * 2`. Full jitter spreads them across the window.

Reference: AWS's canonical article on the shapes and their trade-offs is worth reading once. <https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/>

An `asyncio` implementation:

```python
import asyncio, random, httpx

TRANSIENT_STATUSES = {408, 409, 425, 429, 500, 502, 503, 504, 529}

async def call_with_retry(
    do_request,
    *,
    max_attempts: int = 5,
    base: float = 1.0,
    cap: float = 30.0,
):
    for attempt in range(max_attempts):
        try:
            return await do_request()
        except httpx.HTTPStatusError as exc:
            status = exc.response.status_code
            if status not in TRANSIENT_STATUSES or attempt == max_attempts - 1:
                raise
            retry_after = _parse_retry_after(exc.response)
            delay = retry_after if retry_after is not None else random.uniform(0, min(cap, base * 2 ** attempt))
        except (httpx.ConnectError, httpx.ReadTimeout, httpx.RemoteProtocolError):
            if attempt == max_attempts - 1:
                raise
            delay = random.uniform(0, min(cap, base * 2 ** attempt))
        await asyncio.sleep(delay)

def _parse_retry_after(response: httpx.Response) -> float | None:
    header = response.headers.get("retry-after")
    if not header:
        return None
    try:
        return float(header)   # seconds
    except ValueError:
        # RFC 7231 allows an HTTP-date form too. Parse if you need to.
        return None
```

Two design decisions embedded above:

- **`retry-after` wins over jitter when present.** The provider is telling you how long to wait; respect it. Add a small random offset if you want extra safety against synchronised retries.
- **The last attempt does not sleep and does not retry — it raises.** After `max_attempts` you have earned the failure.

## Which errors mean what — a per-provider quick reference

### Anthropic

- `400 invalid_request_error` — terminal. Look at the message.
- `401 authentication_error` — terminal.
- `403 permission_error` — terminal.
- `404 not_found_error` — terminal.
- `413 request_too_large` — terminal.
- `429 rate_limit_error` — transient.
- `500 api_error` — transient.
- `529 overloaded_error` — transient. Back off longer than the average; provider is under pressure.

Reference: <https://docs.anthropic.com/en/api/errors>.

### OpenAI

- `400 invalid_request_error` — terminal. `context_length_exceeded` is the most common; see the trimming discussion below.
- `401 authentication_error` — terminal.
- `403 permission_error` — terminal.
- `404 model_not_found` — terminal.
- `429 rate_limit_error` — transient. **Check the message** — a `429` for hitting the request-per-minute limit is transient, but a `429` for hitting the *daily quota* is terminal for the next 24 hours.
- `500` / `502` / `503` / `504` — transient.

Reference: <https://platform.openai.com/docs/guides/error-codes>.

Do not hand-craft this list. Both SDKs raise typed exceptions — `anthropic.APIStatusError` subclasses, `openai.APIStatusError` subclasses — that already carry the category. Prefer catching the typed exceptions where you can and fall back to status-code matching only when you have to.

## Handle rate limits deliberately, not accidentally

Rate limits are the most common transient. Two habits that keep them from becoming outages:

- **Read `x-ratelimit-*` headers on every response and log them.** Both providers publish remaining quota and reset time; if your median request has `x-ratelimit-remaining-requests` in single digits, you are one traffic spike away from 429s regardless of your retry policy.
- **Bound your concurrency to well below the limit.** Chapter 3's `Semaphore` is the *primary* rate-limit control. Retries are a fallback for the tail, not a substitute for capacity planning.

If your workload is genuinely rate-limited (a batch job that must run through a large input set), consider provider batch APIs — OpenAI Batch and Anthropic Message Batches accept a large set of requests at once and return them within a longer SLA, at lower cost, and without touching your live rate limits. References: <https://platform.openai.com/docs/guides/batch> and <https://docs.anthropic.com/en/docs/build-with-claude/batch-processing>.

## When the SDK is already retrying for you

Both the OpenAI and Anthropic Python SDKs perform some retries automatically. Two things to know:

- **The default retry counts are small.** Two retries on transient failures is a common default. Fine for most cases; too few for a long-running batch.
- **The SDK's retry counts *toward* your budget.** If you also wrap the call in your own retry loop, one logical request becomes two-times-your-loop-count physical requests. Pick one layer to own retries and disable the other.

If you keep the SDK's retries: `max_retries=` at client construction on both SDKs; check the current SDK reference for the exact parameter name and defaults.

<!-- needs-research: confirm the exact `max_retries` parameter name and default on the current openai-python and anthropic-python clients as of 2026-08 — check the SDK READMEs at https://github.com/openai/openai-python and https://github.com/anthropics/anthropic-sdk-python. -->

## Circuit breakers — when to stop retrying entirely

If a provider is broadly down for the count, retries are wasted budget. A circuit breaker is a small piece of state that trips after N consecutive failures, refuses further calls for a cooldown window, then allows a probe request to check whether the provider has recovered.

For a single-process app, you can hand-roll this in ~40 lines. For a multi-process service, use a shared store (Redis) or a library:

- `pybreaker` — Python circuit breaker. <https://github.com/danielfm/pybreaker>
- `tenacity` — general-purpose retry library that composes with breakers. <https://tenacity.readthedocs.io/>

Breakers are worth adding when you are running enough LLM traffic that a 5-minute provider outage would produce a five-figure retry bill. Otherwise a bounded retry policy is enough.

## Retries interact with idempotency and observability

Two often-missed corollaries of retry policies:

### Idempotency keys

If a retry could cause a side effect twice, generate a per-logical-request idempotency key and send it on every attempt. The server (or your tool) uses the key to deduplicate.

```python
import uuid
key = str(uuid.uuid4())
async def request():
    return await client.post(..., headers={"Idempotency-Key": key})
```

Reference again: <https://platform.openai.com/docs/api-reference> (search "Idempotency-Key").

### Log every attempt, not just the last

If a request eventually succeeds after 3 retries, log all 3 attempts (with the delay chosen for each) as well as the final success. Two reasons:

- **Alerting on retries.** If your retry rate spikes, that is a leading indicator of a provider incident. You cannot alert on it if you do not record it.
- **Debugging.** "It worked in the end" is not enough when you are trying to figure out why a request took 15 seconds. The 3 retries and their 2-second, 4-second, 8-second delays add up.

A minimal per-attempt log record: request ID, attempt number, status, latency, delay applied. That is enough to plot retry-rate over time and to reconstruct any specific request's path.

## Common bugs to prepare for

- **Retrying 400s.** The single most common bug in this space. Do not do it. If your policy retries on "any exception", you are.
- **Retrying without jitter.** Ten clients hit the same 429 at the same instant; without jitter they all retry at exactly `base * 2` seconds and hit the provider in a synchronised burst.
- **Retrying without a cap.** Six exponential backoffs on `base=1` and no cap sums to over a minute. If your feature's user-facing latency budget is 8 seconds, this is a bug pretending to be a retry.
- **Retrying past the caller's timeout.** If the caller has a 10-second deadline and your retry policy takes 20 seconds to give up, you are burning tokens for a response no one will read. Chapter 3's `asyncio.wait_for` at the caller layer bounds this correctly.
- **Both the SDK and your loop retrying.** Multiplies the retry count silently. Pick one owner.
- **Retrying non-idempotent tool calls without an idempotency key.** Two bookings, one API failure. Insert the key.
- **Swallowing the final failure.** After `max_attempts`, raise. Do not return `None` or an empty dict; that is a lie the model will happily narrate around (see mod-103 chapter 6).

## Summary

- Every failure is transient or terminal. Retry transient (429, 5xx, connection errors, provider overload); do not retry terminal (400 schema/context length, 401, 403, 404).
- Use exponential backoff with **full jitter** and a **cap**. Respect `retry-after` when present.
- Provider SDKs already retry a little. Pick one layer (SDK *or* your loop) to own retries; disable the other or explicitly compose the two counts.
- Rate-limit control lives in your concurrency cap, not in your retry policy. Retries are for the tail, not for capacity.
- Non-idempotent operations need an idempotency key. Otherwise a retry can double-book.
- Log every attempt. Retry-rate spikes are a leading incident indicator; without per-attempt logs you have no signal.
- Consider a circuit breaker at scale; use the provider batch APIs when your workload is really "many requests, longer SLA is fine."

The next chapter closes the module: measuring the honest end-to-end latency of a streaming feature — p50, p95, p99, and the difference between time-to-first-token and time-to-completion.
