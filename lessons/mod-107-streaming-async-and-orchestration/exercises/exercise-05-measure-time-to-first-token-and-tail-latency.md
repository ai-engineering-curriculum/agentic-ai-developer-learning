# Exercise 05 — Measure time-to-first-token and tail latency

Paired with [chapter 5 — measuring streaming latency honestly](../05-measuring-streaming-latency.md).

**Estimated effort:** 90–120 minutes.

## Objective

Run a small streaming workload against a real provider, capture per-request time-to-first-token (TTFT), time-to-completion (TTC), and output-token throughput, and produce the one-page latency report chapter 5 describes. The output of this exercise is a report and a script — the script is the one you will re-run for every subsequent module when someone asks "how fast is it?"

## Problem statement

Build `measure_stream.py` — a load-testing harness that:

- Fans out `N` streaming requests to the provider under a bounded semaphore (reuse your fan-out from exercise 03).
- For each request, records per-request measurements: `t0` (request sent), `t_first_chunk`, `t_last_chunk`, `output_tokens`, and success/failure.
- Aggregates the successful requests into a percentile report (`p50`, `p95`, `p99`) for TTFT, TTC, and output-tokens-per-second.
- Prints the one-page report from chapter 5 and writes raw samples to a CSV for later analysis.

Use it to answer three concrete questions about your provider, model, and network:

1. What is the TTFT p50 / p95 / p99 for a small (~200-token) response?
2. What is the TTC p50 / p95 / p99 for a large (~1500-token) response?
3. What is the ratio of the mean to the p99 for TTC? (If it is close to 1, the tail is well-behaved. If it is close to 0.5, the tail is doing all the damage.)

## Requirements

1. Use `time.perf_counter()` for every timestamp. Do not use `time.time()`.
2. Instrument per-request:
   - `t0`: immediately before the call is issued.
   - `t_first_chunk`: the moment the first text delta arrives (Anthropic `content_block_delta` with `text_delta`; OpenAI `delta.content` non-null).
   - `t_last_chunk`: the moment the stream closes.
   - `output_tokens`: from the SDK's final usage (Anthropic `final_message.usage.output_tokens`; OpenAI `stream_options={"include_usage": True}` on the last chunk).
3. Derived per-request metrics:
   - `ttft_s = t_first_chunk - t0`
   - `ttc_s = t_last_chunk - t0`
   - `tokens_per_s = output_tokens / max(t_last_chunk - t_first_chunk, 1e-6)` — **not** `output_tokens / ttc_s` (chapter 5 explains why).
4. Aggregate:
   - Compute p50, p95, p99 for TTFT, TTC, and tokens-per-second across all successful requests.
   - Report the failure rate and its composition (e.g. "2× 429, 1× 502, 0 terminal").
5. Warm-up: discard the first 2 requests from the aggregation. Print how many were discarded. This is the cold-start guard from chapter 5.
6. Print the one-page report in the exact format from chapter 5:

   ```
   Sample: <N> requests, model <name>, concurrency <C>
   TTFT   p50: XXX ms   p95: XXX ms   p99: XXX ms
   TTC    p50: X.X s    p95: X.X s    p99: X.X s
   Tokens per s   p50: XX   p95: XX   p99: XX   (output-only)
   Failures: X / N (X.X%)  — <breakdown>
   ```

7. Write raw samples to `samples.csv`: one row per request with all timestamps, deltas, and outcome. Include failed requests with the failure reason instead of the measurements.
8. Provide two prompt sets and a `--profile short|long` flag:
   - `short`: 30 prompts asking for a 1-sentence answer. Target ~200 output tokens.
   - `long`: 20 prompts asking for a 3-paragraph answer. Target ~1500 output tokens.

## Requirements — three deliverables

At the end of this exercise you have:

1. `measure_stream.py`, the harness. Reusable in every following module.
2. `samples.csv`, the raw data from a real run against your provider of choice. Both `short` and `long` profiles run and committed.
3. `LATENCY_REPORT.md`, a short document (half a page) containing:
   - The one-page report from both profiles.
   - Your answers to the three questions above.
   - One sentence naming what surprised you.

Commit all three. The report is the artifact your future self and your team will reference.

## Starter guidance

- Prometheus histograms and quantiles: <https://prometheus.io/docs/practices/histograms/>
- OpenTelemetry metrics data model: <https://opentelemetry.io/docs/specs/otel/metrics/data-model/>
- HDR Histogram: <https://www.hdrhistogram.org/>
- "How NOT to Measure Latency" (Gil Tene, worth watching once): <https://www.infoq.com/presentations/latency-response-time/>
- `numpy.percentile` for large samples: <https://numpy.org/doc/stable/reference/generated/numpy.percentile.html>

Recommended concurrency: 4 to 8. High enough that percentiles are meaningful (30–50 successful samples); low enough that you do not tickle rate limits and get retry-dominated tails. If you run into 429s, drop the concurrency rather than adding retries — this exercise is about measuring the model, not measuring your retry policy.

Do not measure with the SDK's built-in retries on. They add hidden latency to a small share of requests and pollute the tail. Set `max_retries=0` on the client for this exercise.

## Acceptance criteria

- The one-page report prints in the format above and matches the shape from chapter 5. No mean-latency line. Percentiles only.
- `samples.csv` opens in any spreadsheet and every row has all fields populated (blanks are for failure-attributable fields on failed rows only).
- Running the harness with `--profile short` and again with `--profile long` produces two obviously-different reports — the `long` profile's TTC p99 is at least 3× the `short` profile's.
- Running the harness twice back-to-back with the same profile produces reports whose p50 values are within ~20% of each other. If they are very different, either your warm-up is not sufficient or your sample size is too small.
- The three questions in `LATENCY_REPORT.md` are answered with numbers, not adjectives. "TTC p99 was 8.4 seconds" — not "TTC p99 was slow."

## Stretch goals

- Add `--server` mode: your harness spins up a FastAPI endpoint that forwards a streaming response, and measures *client-side* latency by driving the endpoint with `httpx.AsyncClient` from a separate process. Compare client-side p95 to server-side p95 — the delta is your framework + network cost.
- Add a `--degrade` flag that injects synthetic per-request delays (`await asyncio.sleep(random.expovariate(1/0.5))` before each call) to simulate a bad-tail day. Confirm the p99 shifts and the p50 barely moves — that is the whole point of percentile reporting.
- Alert-simulator: add a rolling-window p95 tracker that fires "ALERT" to stderr when the last 30-request rolling p95 exceeds a threshold. Test it with the `--degrade` flag.
- Compare two models side-by-side on the same prompt set. Which has the better TTFT? The better TTC? The better tokens-per-second? Rank them; note the trade-off.
- Read Gil Tene's HdrHistogram talk and rebuild the percentile aggregation using HdrHistogram's Python port. Note the difference in reported p99 between `numpy.percentile` and HdrHistogram at N > 1000.
