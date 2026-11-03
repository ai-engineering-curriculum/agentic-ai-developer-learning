# Chapter 5 — Status pages, SLOs, and graceful degradation

Every LLM provider has, sooner or later, an incident. A region-scale outage, an overloaded model, a degradation that manifests as elevated latency and elevated error rates for hours. The retry logic from mod-107 handles the *short* transient — a 429 that clears in a second or two. This chapter is about the *longer* transient — the 30-minute-to-multi-hour provider incident where retrying is not going to help, and your feature has to do something more considered than either spinning forever or 500-ing at the user.

## Motivation

Three failure patterns:

1. **The infinite retry.** Provider is genuinely down for an hour. Your retry loop with exponential backoff sends a request every ~60 seconds. Your users see a slow spinner forever. Your logs fill up. Your bill fills up. When the provider comes back, your feature does too — but it never should have kept the user waiting.
2. **The plain 500.** Provider is down. Your code returns a 500 to the user with no explanation. The user tries again. Same 500. They file a ticket. Your on-call engineer, who does not have a status-page-check in their runbook, treats it as an internal bug for the first 20 minutes.
3. **The silent quality regression.** The provider isn't down, it's *degraded* — 20% of requests are 500-ing, or model quality dropped for some reason. Your dashboard shows normal traffic and normal latency. Your users see intermittently-bad answers. Nobody notices for hours because nobody was watching the right signal.

The techniques in this chapter — reading status pages honestly, defining explicit SLOs for cost / latency / availability, and designing graceful-degradation paths in advance — defend against all three.

## Provider status pages

Both major providers publish a status page. Bookmark them; put them in the on-call runbook.

- Anthropic: <https://status.anthropic.com/>
- OpenAI: <https://status.openai.com/>

Two things to know about status pages in general:

- **They are a lagging indicator.** A real incident is often visible in your own error rates and latency histograms 5–15 minutes before the status page updates. Your monitoring must be the primary signal; the status page is confirmation and public communication.
- **They have per-component granularity.** Both providers break out API vs. specific models vs. dashboard/console. A "degraded" indicator on the console is not the same as a degraded indicator on the API. Read the components.

The status page RSS/Atom feed or webhook is worth wiring up. When you see an incident open on the provider status page, that is a legitimate signal to stop retrying and start degrading.

## Provider SLAs and your SLOs — they are different

A **SLA** (Service Level Agreement) is what the provider commits to and what they will refund you for missing. Both major providers publish uptime SLAs — typically 99.5–99.9% depending on the tier. Look up the current terms:

- OpenAI SLAs are documented in the enterprise agreements; individual and startup tiers have limited or no SLA.
- Anthropic publishes SLA terms for eligible plans.

<!-- needs-research: cite the current SLA percentages and eligible tiers for both providers as of 2026-08 — link to the official documents. -->

A **SLO** (Service Level Objective) is what *your* feature commits to *your* users. Yours is typically stricter than the provider's — and if your provider is 99.5%, your feature cannot be more than 99.5% *from that single provider alone*. That is the arithmetic that motivates the fallback provider from chapter 1.

For an LLM feature, three SLOs are worth explicitly writing down before you launch:

- **Availability.** "This feature responds with a non-error answer on X% of requests over a 30-day window." (Where "non-error" includes the graceful-degraded response, see below.)
- **Latency.** "p95 time-to-completion under X seconds. p99 under Y seconds."
- **Cost.** "Cost per successful call does not exceed X cents at expected volume."

The last one is a real SLO, not a soft target. If cost per call moves outside its band, that is an incident just like a latency regression is.

## The trade-off triangle

Every incremental improvement on one SLO trades against the others.

- **Higher availability** costs money (redundant providers, warmer standby models) and often adds latency (multi-provider routing has more failure modes to try).
- **Lower latency** either costs money (bigger models are not always faster; sometimes a fast small model plus a slow frontier verify runs cheaper) or trades availability (aggressive timeouts abandon calls the provider would have completed).
- **Lower cost** trades availability (fewer retries, no fallback) or accuracy (smaller models on the tail).

You cannot maximise all three. You can pick two hard targets, treat the third as a bound, and write down the trade you made. This is what the one-page decision doc from chapter 1 evolves into — a small "operational contract" for the feature that names the numbers and the trade.

## What graceful degradation looks like

"Graceful degradation" is the umbrella term for what a feature does when the primary path fails. There is no single answer; the right answer depends on what the feature does and how tolerant the user is of a partial or delayed reply. Common shapes:

### Shape 1 — degrade to a smaller model (or a different provider)

If your primary is unavailable and you have a fallback model that is cheaper / smaller / worse-but-adequate, use it. This is where the two-provider habit from chapter 1 pays off directly.

```python
def call_with_fallback(prompt):
    try:
        return call(PRIMARY, prompt, timeout=5)
    except (ProviderError, TimeoutError):
        return call(FALLBACK, prompt, timeout=5)
```

Two subtleties:

- **The fallback must have been evaluated on the task.** If you have never measured accuracy on the fallback, you are shipping an unknown quality on incident day. This is why chapter 1's exercise 02 makes you evaluate both providers.
- **The fallback must have been *warmed*.** If you have never actually sent traffic to it, you don't know your effective rate limits, your auth is stale, or your keys are wrong. Send at least a small trickle of real traffic to the fallback continuously so you know it works.

### Shape 2 — degrade to a cached / template response

Some features can accept "we can't answer that right now" more gracefully than a 500. A support-bot feature can respond with "We're routing your question to a human — expect a reply within 24 hours" when the LLM is unavailable. A summarisation feature can show the raw text with a "Summary temporarily unavailable" note. These are not great user experiences, but they are dramatically better than a spinner or a 500.

The pattern requires you to have written the template ahead of time and have a code path that returns it when the LLM path fails. If it isn't in the code, it doesn't happen during the incident.

### Shape 3 — degrade to no LLM at all

Some features have a purely-deterministic fallback: return the top result from search directly instead of a summarised answer; skip the LLM re-ranker and use the retrieval score directly. Not applicable to every feature; when it is, it is often the cleanest degradation.

### Shape 4 — queue and return later

For non-urgent features, the graceful response to an outage is "we've queued your request; we'll email you when it's done." The batch API pattern from chapter 4 is the underlying mechanism.

### Shape 5 — fail loudly (yes, sometimes this is right)

If the feature is business-critical and no fallback is acceptable (e.g., a contract-generation flow that must be right or must not run), the correct graceful degradation is to **refuse the request** with a clear error message and page the on-call. Better a loud refusal than a wrong output. This is the case for feature paths where "silently wrong" would be worse than "temporarily unavailable."

## The health-check pattern

A common way to short-circuit an incident is a **circuit breaker on the primary provider**:

- If the last N calls to the primary have failed within the last M seconds, mark the primary "down" for a cooldown window.
- All requests during the cooldown skip straight to the fallback (or to the degraded path), no retry, no waiting.
- After the cooldown, probe with a single request. If it succeeds, close the breaker and resume routing to the primary. If it fails, extend the cooldown.

This composes with the retry policy from mod-107 chapter 4. The retry policy handles the sub-minute transient; the breaker handles the multi-minute outage. The two together mean:

- Retries do not amplify the outage into a 5×-bill for failure.
- Users don't wait through a slow retry chain when the provider is down for real.
- When the provider recovers, your feature notices within one probe cycle.

Reference for the pattern (language-agnostic): <https://martinfowler.com/bliki/CircuitBreaker.html>

## Reading a status page during an incident — the checklist

The five-minute drill when your dashboards show elevated error rates:

1. **Open the provider status page** for the affected provider. Read the components. Confirm whether the API is degraded, a specific model is degraded, or nothing is publicly acknowledged.
2. **Check the incident's blast radius.** Region-scoped? A specific model? A specific endpoint? This determines whether the fallback is another provider, another model on the same provider, or another region.
3. **Verify the circuit breaker tripped** (or trip it manually if you have a control). Traffic should be flowing to the fallback path.
4. **Verify the fallback is healthy.** A common failure mode is that both providers have correlated incidents; the fallback path is also failing. If so, degrade further (Shape 2/3/4/5 above).
5. **Communicate.** Update your own status page / product notification. "We're aware; we're on a fallback path; expected impact is <slower / less-detailed / limited> replies." Users tolerate the truth much better than they tolerate a silent regression.

Every incident that involves a provider outage is a chance to sharpen this drill and the runbook it lives in. Do a short post-incident write-up capturing what tripped, what fell over, and what you'd change.

## Alarming — the four alarms every LLM feature should have

- **Error rate.** Percent of requests returning an error (any 4xx you couldn't degrade, any 5xx). Alarm at a low threshold (e.g., 1% over 5 minutes).
- **Latency p95 / p99.** Alarm when either moves out of its SLO band. Streaming features: alarm on TTFT and time-to-completion separately.
- **Cost per successful call.** From chapter 2. Alarm on drift (e.g., +30% week-over-week).
- **Escalation / fallback rate.** From chapter 4 (cascade) and this chapter (fallback provider). A spike in either is a leading indicator — for a small-model regression or for a primary-provider incident, respectively.

These are all cheap to define once you're logging per-request `usage`, latency, model used, and outcome.

## Common mistakes

- **No fallback path in code.** If the primary is unavailable and there is no code path that does anything other than raise, the user gets a spinner or a 500. "Add a fallback" is not something you do during the incident.
- **Fallback path never exercised.** It works in unit tests, has never seen real traffic, and turns out to be broken (bad key, stale prompt, missing feature flag) exactly when you need it.
- **Retrying past the caller's timeout.** Repeating the mod-107 warning here because it matters more during long outages, not less.
- **Silent degradation.** Falling back to a smaller model / template response without telling the user or the operator. If the fallback is meaningfully worse, users deserve to know, and operators deserve a metric that reflects it.
- **Confusing SLA and SLO.** Your feature's SLO cannot exceed the provider's SLA on a single-provider architecture. If you promise 99.99% and your provider promises 99.5%, you have a broken promise the moment the provider misses their own bound.
- **Treating status pages as the primary signal.** They are lagging. Your dashboards are the primary signal.

## Summary

- Every provider will have outages. Design the feature to survive them without spinning forever, 500-ing silently, or shipping a silent quality regression.
- Read status pages, but treat them as confirmation, not detection. Your monitoring is the primary signal.
- Explicit SLOs for availability, latency, and cost. All three trade off. Pick two hard targets; bound the third.
- Graceful degradation shapes: fallback provider, smaller model, cached/template response, no LLM at all, queued-for-later, loud refusal. Pick the one that fits the feature.
- Circuit breaker plus retries: retries handle sub-minute transients; the breaker handles multi-minute outages.
- Four alarms every LLM feature should carry: error rate, latency, cost per successful call, escalation/fallback rate.

That closes the module. You now know how to pick a model with evidence, cost it before deploying it, cache the stable parts of its prompt, route requests by shape, and keep the feature alive when the provider is not. `mod-104-retrieval-basics` picks up from here with retrieval-augmented generation — where every one of these levers becomes even more valuable, and the cost curves get steeper.
