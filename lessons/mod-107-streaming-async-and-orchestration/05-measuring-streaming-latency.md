# Chapter 5 — Measuring streaming latency honestly

You have built a streaming, async, retrying LLM feature. The product manager asks: "How fast is it?" You will give a number. The rest of this chapter is about making sure that number is honest.

## Motivation

Streaming distorts every intuitive latency measure. The user *sees* the first character in 300 ms; the full answer arrives in 6 seconds. Neither number, on its own, describes the experience. Report the wrong one and you will build the next feature against a false baseline: capacity planning based on a mean that hides a bad tail, alerting thresholds that page you on nothing, product decisions ("streaming is fast enough — let's not bother with prompt caching") that fall apart the first time real traffic hits.

The two skills this chapter builds:

- Split "latency" into the specific measurements a streaming feature actually has: time-to-first-token (TTFT), inter-token latency, time-to-completion (TTC), and total wall-clock as the user perceives it.
- Read percentiles (p50, p95, p99) correctly, and understand why the mean lies for this workload.

## The four measurements that matter

Every streaming LLM request produces at least these timestamps in your process:

```
t0: request built, about to send
t1: server acknowledged the request (first byte of the response headers)
t2: first content chunk received  ← time-to-first-token (TTFT) = t2 - t0
t_i: each subsequent content chunk received
tN: last content chunk / stream closed  ← time-to-completion (TTC) = tN - t0
```

Derived numbers:

- **Time to first token (TTFT):** `t2 - t0`. What determines whether the user thinks the app "responded." Ignore this at your peril.
- **Time to completion (TTC):** `tN - t0`. Total wall-clock from the moment you sent the request to the moment the last byte arrived. What determines whether the next step in your workflow can start.
- **Inter-token latency (ITL):** median of `t_i - t_{i-1}` across all content chunks. Determines the *feel* of the stream after the first token. Choppy vs smooth.
- **Output tokens per second (throughput):** `output_tokens / (tN - t2)`. Useful as a sanity check against your provider's published numbers; a value dramatically below normal means either the model tier is degraded or you have introduced a consumer-side bottleneck.

You want all four. Every one of them tells you a different thing.

## What "streaming feels faster" actually means

The user's perceived latency is roughly TTFT — the wait before "something started happening." The user's perceived *duration* is TTC. A feature with TTFT of 300 ms and TTC of 5 s feels fast for the first paragraph and reads at reading speed after that; a feature with TTFT of 2 s and TTC of 5 s feels slow the whole way through even though the total is the same.

Streaming does not make total wall-clock any shorter. It changes the *distribution* of what the user waits for. Confusing the two is the specific dishonesty this chapter exists to prevent — an internal dashboard that reports only TTFT tells your team the product is snappy; a downstream job that reads the whole response before starting does not care about TTFT and will hit the TTC.

## Percentiles: p50, p95, p99, and never the mean

Latency distributions for LLM calls are heavy-tailed. A single provider hiccup can add several seconds to a random request. The mean of a heavy-tailed distribution is dominated by the worst outliers; the *median* (p50) tells you what a typical user sees; the *p95* and *p99* tell you what the worst-serviced 5% and 1% of users see.

- **p50 (median):** half of requests are faster; half are slower. This is what "usually" means.
- **p95:** 95% of requests are at most this fast. 1 in 20 users is slower.
- **p99:** 99% of requests are at most this fast. 1 in 100 users is slower — and on a busy service, that is a lot of unhappy users.

The mean, for this workload, mostly reflects how bad the tail was during the sample window. Report percentiles, not means. If someone insists on a mean, quote it *with* p50, p95, p99 to make the tail visible.

Two properties of percentiles that people underestimate:

- **They do not add.** The p99 of a two-stage pipeline is *not* the sum of each stage's p99. If a request takes stage A then stage B, its total sits somewhere on the joint distribution — usually below the sum of the two p99s. Do not stitch p99s across stages by addition.
- **They shift with volume.** The p99 of 100 samples is roughly the max; the p99 of 100,000 samples is a stable number. Beware quoting p99 from a sample of 20 requests — it means almost nothing.

Reference on why the mean is misleading for latency: <https://www.hdrhistogram.org/> (Gil Tene's HdrHistogram is the reference tool). The talk "How NOT to Measure Latency" is worth watching once — <https://www.infoq.com/presentations/latency-response-time/>.

## Instrumenting a single streaming call

The simplest possible instrumentation:

```python
import time, anthropic

client = anthropic.Anthropic()

def measured_stream(topic: str) -> dict:
    t0 = time.perf_counter()
    ttft = None
    n_chunks = 0
    with client.messages.stream(
        model="claude-opus-4-7",
        max_tokens=1024,
        messages=[{"role": "user", "content": f"Write a paragraph on {topic}."}],
    ) as stream:
        for _ in stream.text_stream:
            if ttft is None:
                ttft = time.perf_counter() - t0
            n_chunks += 1
        final = stream.get_final_message()
    ttc = time.perf_counter() - t0
    output_tokens = final.usage.output_tokens
    return {
        "ttft_s": ttft,
        "ttc_s": ttc,
        "output_tokens": output_tokens,
        "output_tokens_per_s": output_tokens / max(ttc - ttft, 1e-6),
        "chunks": n_chunks,
    }
```

Three habits this snippet already gets right:

- **`time.perf_counter()`, not `time.time()`.** `perf_counter` is monotonic and not subject to NTP adjustments. Wall-clock jumps mid-stream will otherwise show up as negative latencies.
- **Timers around the whole streaming context**, not just the initial `.create()` call. The whole point of streaming is that generation continues after the initial response object exists.
- **Compute throughput from `ttc - ttft`, not `ttc`.** Dividing by `ttc` charges throughput for the pre-first-token setup. Throughput is a property of the generation phase.

## Aggregating across many requests — p50 / p95 / p99

Once you have per-request measurements, aggregate:

```python
import statistics

def percentiles(samples: list[float], ps=(50, 95, 99)) -> dict[int, float]:
    if not samples:
        return {p: float("nan") for p in ps}
    return dict(zip(ps, statistics.quantiles(samples, n=100, method="inclusive")[i] for i in [p - 1 for p in ps]))
```

Two things worth knowing about `statistics.quantiles`:

- It uses linear interpolation between adjacent samples. For small `n`, the p99 it returns is not the same as the empirical max — it is a smoothed estimate.
- For serious percentile reporting at high volume, use `numpy.percentile` (fast) or `hdrhistogram` (constant-space, very accurate at the tail).

An honest report for a streaming feature looks like this:

```
Sample: 200 requests, model claude-opus-4-7
TTFT   p50: 340 ms   p95: 620 ms   p99: 1350 ms
TTC    p50: 3.1 s    p95: 5.8 s    p99: 9.2 s
Tokens per s   p50: 62   p95: 48   p99: 31   (output-only)
Failures: 3 / 200 (1.5%)  — 2× 429, 1× 502
```

Notice what is *not* there: no mean. No "average speed." Just percentiles, the sample size, and the failure count. That is the report a product manager can act on.

## Backend vs perceived latency

If your feature is a browser talking to your API which talks to the model, there are two vantage points and you probably need both:

- **Server-side latency** — measured in your service, from request-received to response-fully-sent. This is what you can control and alert on.
- **Client-side latency** — measured in the browser, from click to first-visible-character. This includes network to your server, your server's queueing, everything.

Client-side is what the product manager actually cares about. Server-side is what wakes you up at 3 AM. Instrument both; expect the client-side numbers to be worse by the round-trip cost.

## Cold starts, warm-ups, and outliers you should exclude

Two categories of measurement to handle deliberately, not by accident:

- **First-request-after-idle** on serverless deployments can include cold-start latency (loading dependencies, warming the connection pool). Either measure only after a warm-up phase or report cold-start separately. Do not average them into your steady-state p95.
- **Rate-limit backoffs and retries** dominate the tail of any bounded-concurrency load test. If you are measuring the *model's* latency, exclude retry sleeps. If you are measuring the *feature's* latency, include them but flag the retry-attributable share.

An acceptable practice: split your histogram into "successful first attempt" and "eventual success after retry." Do not silently roll the two into one number.

## Sampling and load testing

Your production traffic gives you a percentile distribution over real requests. For pre-launch decisions you often want to *predict* what that distribution will look like under some hypothetical load. Two tools worth knowing:

- **`hey`** — a small, focused HTTP load generator. Good for a first look at p95 at N concurrent requests. <https://github.com/rakyll/hey>
- **`k6`** — script-driven load tests, good for scenarios with logic (auth, follow-up requests). <https://k6.io/docs/>

Both are for benchmarking the *transport*. For LLM-specific throughput and tail behaviour, run a small load in your own async harness (the exercise for chapter 3 is a good starting point) with realistic prompts. The published throughput numbers from providers are best-case; measure yours.

## Alerting on percentiles

If your service is in production, alert on p95 or p99, not on the mean. Two habits:

- **Alert on a rising p95 over a rolling window.** A 30-minute p95 above threshold X is a much better trigger than "one request over N seconds."
- **Alert on TTFT and TTC separately.** A regression in TTFT is usually a model-tier or provider issue; a regression in TTC without a change in TTFT is often a token-length or throughput problem. Different diagnoses; different runbooks.

Most APM tools (Datadog, Prometheus with histograms, OpenTelemetry) support percentile aggregation. Use their histogram types (`Histogram` in Prometheus, `distribution` in Datadog) rather than shipping every latency sample — histograms aggregate correctly across pods; raw samples do not.

Reference: Prometheus histograms and quantile queries — <https://prometheus.io/docs/practices/histograms/>. OpenTelemetry metrics data model — <https://opentelemetry.io/docs/specs/otel/metrics/data-model/>.

## Reporting to a product manager

The one-page latency report for a streaming LLM feature has four items:

1. **Sample size, model, and time window.** "200 requests, `claude-opus-4-7`, last hour." Without these the numbers below are unreadable.
2. **TTFT p50 / p95 / p99.** How fast the feature feels.
3. **TTC p50 / p95 / p99.** How long the full answer takes. If a downstream step waits for the full response, this is the number that constrains it.
4. **Failure rate and its composition.** "1.5% failures — 2× 429, 1× 502, 0 terminal." Distinguishing transient (retriable, provider-side) from terminal (bug or budget issue) is what turns "we had failures" into an actionable line.

Two anti-patterns to refuse:

- **"Latency: 2.4 seconds."** Which latency? Mean? Median? Server-side? Client-side? TTFT or TTC? Ask.
- **"99.9% of requests are fast."** Fast under what threshold? On what sample size? Fast on the median or fast on the p99? Ask.

The role of the developer in this conversation is not to hide the tail. The tail is real; the honest report makes the tail visible so the decision — invest in caching, upgrade the tier, tolerate the p99 — is informed.

## Common bugs to prepare for

- **Reporting means.** Nearly always misleading for latency. Switch to percentiles the moment you have more than a handful of samples.
- **Measuring TTFT with `time.time()`.** NTP corrections mid-stream will occasionally give you negative or comically large values. Use `time.perf_counter()`.
- **Averaging p99s across services.** The p99 of A → B → C is not the sum of the three p99s. Compute the p99 of the joint distribution or, if you cannot, at least name what you are computing.
- **Including retry backoff in "model latency" but excluding it from "user latency" (or vice versa).** Pick one convention per metric and document it.
- **Warm-up requests counted in the report.** Discard the first request or two on serverless / cold-start deployments before you compute the percentiles.
- **Silent sample drops.** A load-test harness that drops timeouts before percentile computation reports a rosy p99 that does not include the failures. Include timeouts as "very slow" samples, or report failure rate separately with the same visibility.
- **Alerting on the mean.** A single 90-second retry can spike the 5-minute mean by 3 seconds. Alert on the p95 or p99.

## Summary

- Streaming has four latency measurements you actually own: **TTFT** (time to first token), **TTC** (time to completion), **ITL** (inter-token latency), and **throughput** (output tokens/second). Track them separately.
- The mean lies for heavy-tailed workloads. Report **p50, p95, p99**, plus sample size and time window.
- Percentiles do not add across stages. Do not glue them together by addition; compute joint percentiles or report per-stage separately.
- `time.perf_counter()` for timers, not `time.time()`.
- Distinguish server-side from client-side latency. Instrument both; the product manager cares about client-side.
- Exclude cold starts from steady-state percentiles; include retry backoff *if* your report is about the user experience and *exclude* it *if* your report is about model behaviour. Never do both silently.
- Alert on percentiles over rolling windows, not on single-request means. TTFT and TTC regressions have different diagnoses.

That closes the module. You can now stream a response, parse partial JSON safely, fan out concurrent calls under a bounded semaphore, retry transient failures with jitter, and give a product manager the honest numbers to make a shipping decision on. The next module (`mod-108-model-selection-cost-and-prompt-caching`) picks up cost — because every technique in this module makes more calls, and prompt caching is what keeps that bill sane at scale.
