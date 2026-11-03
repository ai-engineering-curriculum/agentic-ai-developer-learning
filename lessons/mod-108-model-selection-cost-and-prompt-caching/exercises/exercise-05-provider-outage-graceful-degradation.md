# Exercise 05 — Provider outage: graceful degradation

Paired with [chapter 5 — status pages, SLOs, and graceful degradation](../05-status-pages-slos-and-graceful-degradation.md).

**Estimated effort:** 60 minutes.

## Objective

Simulate a provider outage against your own feature. Build a small fault-injecting proxy that can make the primary provider fail on demand. Wire a **circuit breaker + fallback path** into your feature so that when the primary is unhealthy, requests degrade cleanly — to a fallback provider, a smaller model, a cached response, or an honest "temporarily unavailable" message. Prove that the feature stays alive with numbers, and that it recovers automatically when the primary comes back.

## Problem statement

Take a working single-provider feature you've built (the cost-enforced classifier from exercise 01, or the cascade router from exercise 04 — either is fine). Add graceful degradation to it. Then break the primary provider and observe the behaviour.

You will build three small pieces:

1. **A fault-injecting proxy** that sits between your app and the primary provider and can be told to return specific HTTP status codes (or hang) instead of forwarding to the real API.
2. **A circuit breaker** on the primary path. When the last N calls have failed within M seconds, the breaker "opens" and all requests for a cooldown window skip straight to the fallback path without touching the primary.
3. **A fallback path.** Pick one and implement it end-to-end:
   - Fallback to the other provider (e.g., Anthropic primary → OpenAI fallback, or the reverse).
   - Fallback to a smaller / cheaper model on the same provider.
   - Fallback to a template response (`{"status": "temporarily_unavailable", "message": "..."}`).

## Requirements — the fault-injecting proxy

Any small local HTTP proxy works (FastAPI, `aiohttp`, `starlette`). It should support control endpoints:

- `POST /control/fail_next?count=10&status=503` — the next 10 upstream calls return 503.
- `POST /control/fail_next?count=10&hang_seconds=30` — the next 10 upstream calls hang for 30 seconds each (simulating latency degradation).
- `POST /control/reset` — clear the fault queue; requests pass through normally.

Point your primary SDK client at the proxy's URL (both providers' Python SDKs let you set a custom `base_url`; check the SDK docs). The fallback path must **not** go through the proxy.

## Requirements — the circuit breaker

A minimal breaker suffices:

```python
class CircuitBreaker:
    def __init__(self, *, failure_threshold: int, cooldown_seconds: float):
        ...

    def record_success(self) -> None: ...

    def record_failure(self) -> None: ...

    def is_open(self) -> bool: ...       # True → skip primary, use fallback
```

Semantics:

- Consecutive failures ≥ `failure_threshold` opens the breaker.
- Breaker stays open for `cooldown_seconds`.
- After cooldown, next call is a probe. Success → close. Failure → re-open for another cooldown.
- Reset failure count on any success.

## Requirements — the wiring

Wrap your feature call with:

```python
def call_with_degradation(request):
    if breaker.is_open():
        return fallback_path(request)
    try:
        result = call_primary(request, timeout=5)
        breaker.record_success()
        return result
    except (ProviderError, TimeoutError):
        breaker.record_failure()
        return fallback_path(request)
```

Two things must be true:

- **The user always gets a non-error response** — either the primary result, the fallback result, or the graceful "unavailable" template. Not a 500. Not a spinner forever.
- **The response indicates whether it was degraded.** Add a field (`{"degraded": true, ...}` or a similar marker) so callers upstream can distinguish and so you can log it as a metric.

## Requirements — the four test scenarios

Run these against the proxy and assert on the observed behaviour. A short test script (`pytest` or a driver `demo.py`) is fine:

1. **All-primary-healthy.** Proxy returns 200 for every call. Send 20 requests. All served by primary. Breaker is closed the whole time. `degraded=false` on all responses.
2. **Provider unhealthy → breaker opens → fallback serves.** Proxy fails the next 10 calls (503). Send 20 requests. Some initial calls hit the primary and fail (up to `failure_threshold`), then the breaker opens and remaining calls skip straight to the fallback. Every request returns a non-error response.
3. **Provider recovers → breaker probes → closes.** After scenario 2, `reset` the proxy. Wait past the cooldown. Send another 20 requests. The probe call succeeds; the breaker closes; subsequent calls return to primary.
4. **Provider slow, not failing.** Proxy hangs each call for 30 seconds. Your primary call has a 5-second timeout. The timeouts count as failures; the breaker opens as in scenario 2. Every request still returns something to the user within the timeout budget.

Assert that:

- Scenario 2: no exception escapes to the caller; all responses were served (some from primary, some from fallback).
- Scenario 3: the recovery is automatic — no manual intervention.
- Scenario 4: no request takes meaningfully longer than the primary timeout + fallback time; the user does not wait 30 seconds even once.

## Requirements — the metrics

Log per request:

- `served_by`: `"primary"`, `"fallback:other_provider"`, `"fallback:template"`, etc.
- `degraded`: bool.
- `breaker_state`: `"closed"`, `"open"`, `"probe"` at time of call.
- Latency.

Print, at end of the driver script:

- Fraction of requests served by primary vs. fallback.
- Fraction of requests served with `degraded=true`.
- p50, p95 latency across all requests.
- Breaker state transitions with timestamps.

## Starter guidance

- Chapter 5 above, especially "the health-check pattern."
- Circuit breaker overview (language-agnostic): <https://martinfowler.com/bliki/CircuitBreaker.html>.
- If you'd rather use a library than hand-roll, `pybreaker` (<https://github.com/danielfm/pybreaker>) is a fine reference — but this exercise is about understanding the shape, so hand-roll first, then swap in a library as a stretch goal.
- OpenAI status page: <https://status.openai.com/>. Anthropic status page: <https://status.anthropic.com/>. Bookmark both; check the RSS feeds.
- For SDK `base_url` overrides, see the SDK READMEs: <https://github.com/openai/openai-python> and <https://github.com/anthropics/anthropic-sdk-python>.

## Acceptance criteria

- Fault-injecting proxy supports the three control endpoints above.
- Circuit breaker opens after `failure_threshold` consecutive failures and closes automatically after a successful probe past `cooldown_seconds`.
- All four scenarios pass. In every scenario, no request escapes as a raw error to the caller.
- Responses carry a `degraded` marker; the metric summary shows the fraction of degraded responses.
- Scenario 4 demonstrates that a hung primary does not cause a hung feature — timeouts, breaker, and fallback compose correctly.
- One short `RUNBOOK.md` in the same directory describes: how to tell when the breaker is open (which metric / log line), what the fallback returns, and what an operator should do during a real provider incident. Two paragraphs is enough.

## Stretch goals

- Add a **per-model breaker**, not a per-provider breaker: if Anthropic Opus is degraded but Anthropic Haiku is fine, cascade requests to Haiku only rather than falling over to the other provider entirely. This matches how real incidents partial-fail.
- Wire a **status-page check** into the runbook: subscribe to the provider's status page RSS/webhook and add a note to `RUNBOOK.md` on how to correlate an internal breaker-open event with the public status.
- Add a **synthetic warm-up trickle** to the fallback: send one real request through the fallback path every N minutes to make sure the fallback stays exercised even when the primary is healthy. Confirm this catches an auth failure on the fallback within one cycle.
- Layer this on top of the **retry policy from mod-107 chapter 4**: short retries handle sub-minute transients, breaker handles multi-minute outages. Verify that the two don't fight — e.g., that retries don't tip the breaker into "open" during a normal transient burst.
- Replace the template fallback with a **batch-queue fallback**: when the primary is down, enqueue the request to a batch API (chapter 4) and return the user a "we'll email you when it's done" acknowledgement. Wire the email/notification stub end-to-end.
