# Resources for mod-107 — Streaming, Async, and Parallel LLM Orchestration

Prefer primary sources — provider docs, RFC-adjacent specifications, and the maintainers of the libraries you actually import. The links below are intentionally short; if one goes stale, the fix is to find the current index page on the provider or standards site, not to grab a blog post from a search result.

## Provider streaming references

### Anthropic

- **Streaming messages** — canonical reference for the SSE event sequence (`message_start`, `content_block_delta`, `message_stop`, etc.) and the SDK stream helper. <https://docs.anthropic.com/en/api/messages-streaming>
- **Streaming tool use** — how tool arguments arrive as `input_json_delta` events and how to parse them at the end of the block. <https://docs.anthropic.com/en/docs/build-with-claude/tool-use/streaming-tool-use>
- **Errors and error codes** — the taxonomy chapter 4 uses (429 rate limit, 500 API error, 529 overloaded). <https://docs.anthropic.com/en/api/errors>
- **Rate limits** — headers, tier structure, and where the 429s come from. <https://docs.anthropic.com/en/api/rate-limits>
- **Message Batches** — the async, cheaper path for large offline workloads. <https://docs.anthropic.com/en/docs/build-with-claude/batch-processing>
- **Client SDKs** — official Python (async support via `AsyncAnthropic`) and TypeScript. <https://docs.anthropic.com/en/api/client-sdks>

### OpenAI

- **Chat Completions streaming** — request shape (`stream=True`), the `ChatCompletionChunk` shape, and the `data: [DONE]` sentinel. <https://platform.openai.com/docs/api-reference/chat/streaming>
- **Structured Outputs** — the schema-constrained output surface used in chapter 2 for streamed JSON. <https://platform.openai.com/docs/guides/structured-outputs>
- **Error codes** — the transient vs terminal taxonomy chapter 4 relies on. <https://platform.openai.com/docs/guides/error-codes>
- **Rate limits** — RPM, TPM, headers, tier progression. <https://platform.openai.com/docs/guides/rate-limits>
- **Batch API** — offline batch path with a 24-hour SLA and reduced pricing. <https://platform.openai.com/docs/guides/batch>
- **Client SDKs** — official Python (`AsyncOpenAI`) and TypeScript. <https://platform.openai.com/docs/libraries>

## Transport standards

- **Server-Sent Events** — the WHATWG HTML Living Standard section defining the wire format. Read at least the "Event stream format" subsection. <https://html.spec.whatwg.org/multipage/server-sent-events.html>
- **Using SSE (MDN)** — the friendlier read-first page. Covers `EventSource` on the browser side. <https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events>
- **`EventSource` API** — the browser-side reader you will use when you forward a model stream to a page. <https://developer.mozilla.org/en-US/docs/Web/API/EventSource>
- **RFC 9110 — HTTP semantics** — normative reference for the status codes chapter 4 categorises (429, 5xx, 413, 422, 401, 403). <https://www.rfc-editor.org/rfc/rfc9110>

## Async Python and HTTP clients

- **`asyncio` overview** — the standard library reference. Skim "Coroutines and Tasks" and "Synchronization primitives" once. <https://docs.python.org/3/library/asyncio.html>
- **`asyncio.TaskGroup`** — Python 3.11+ structured concurrency. Prefer over `gather` for anything that fans out. <https://docs.python.org/3/library/asyncio-task.html#task-groups>
- **`asyncio.Semaphore`** — the concurrency cap primitive from chapter 3. <https://docs.python.org/3/library/asyncio-sync.html#asyncio.Semaphore>
- **`asyncio.wait_for` and timeouts** — the per-item timeout primitive. <https://docs.python.org/3/library/asyncio-task.html#timeouts>
- **`httpx` async support** — sending, streaming, and cancelling async HTTP with a proper connection pool. <https://www.python-httpx.org/async/>
- **`httpx` limits** — `Limits(max_connections=..., max_keepalive_connections=...)`. The transport-side companion to your `Semaphore`. <https://www.python-httpx.org/advanced/resource-limits/>
- **`anyio`** — the cross-runtime async library (works over asyncio and trio). Useful when you need `from_thread` bridging between sync and async. <https://anyio.readthedocs.io/>

## FastAPI and streaming server-side

- **`StreamingResponse`** — how FastAPI (Starlette) forwards a streaming generator to the client without buffering. <https://fastapi.tiangolo.com/advanced/custom-response/#streamingresponse>
- **Using `Request` directly (disconnection detection)** — where `request.is_disconnected()` lives, for the client-cancellation pattern in chapter 3. <https://fastapi.tiangolo.com/advanced/using-request-directly/>
- **Nginx and SSE buffering** — `X-Accel-Buffering: no` and the `proxy_buffering off` directive. If your streaming looks blocked, this is usually why. <https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffering>

## Retry and reliability

- **AWS Builders' Library — timeouts, retries, and backoff with jitter** — the canonical write-up on why full jitter beats exponential-only. Read once. <https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/>
- **`tenacity`** — the Python retry library worth reading the source of even if you write your own helper. <https://tenacity.readthedocs.io/>
- **`pybreaker`** — a small circuit breaker implementation, useful when retries alone are not enough. <https://github.com/danielfm/pybreaker>

## Latency measurement

- **"How NOT to Measure Latency" — Gil Tene (InfoQ)** — the talk that reframes how the industry reports tail latency. If you only watch one talk from this module, watch this. <https://www.infoq.com/presentations/latency-response-time/>
- **HdrHistogram** — the constant-space, high-fidelity histogram library the talk describes. Python port on PyPI. <https://www.hdrhistogram.org/>
- **Prometheus histograms and quantile queries** — how to aggregate percentiles correctly across pods in a production service. <https://prometheus.io/docs/practices/histograms/>
- **OpenTelemetry metrics data model** — the standards-track view of the same aggregation problem. <https://opentelemetry.io/docs/specs/otel/metrics/data-model/>
- **`statistics.quantiles` in the Python standard library** — the low-friction option for exercise-scale sample sizes. <https://docs.python.org/3/library/statistics.html#statistics.quantiles>
- **`numpy.percentile`** — the fast option once samples get large. <https://numpy.org/doc/stable/reference/generated/numpy.percentile.html>

## Standards worth knowing

- **JSON Schema** — the specification behind the structured-output shapes streamed in chapter 2. Same one you learned for mod-102 chapter 5 and mod-103. <https://json-schema.org/>
- **HTTP `Retry-After` header (RFC 9110 §10.2.3)** — the header chapter 4's retry policy respects. <https://www.rfc-editor.org/rfc/rfc9110#field.retry-after>

## Load-testing tools

- **`hey`** — small focused HTTP load generator. Good for a quick p95 sanity check at a given concurrency. <https://github.com/rakyll/hey>
- **`k6`** — scriptable load test runner. Right when your load pattern includes auth flows, follow-ups, or scenario logic. <https://k6.io/docs/>

## Related modules and tracks

- **mod-103-tool-and-function-calling** (this track, prerequisite) — chapter 3 (parallel tool calls) is the sync analogue of this module's async fan-out; chapter 6 (failure handling) is the tool-side counterpart to this module's transport-side retry policy.
- **mod-108-model-selection-cost-and-prompt-caching** (this track) — every technique in this module makes more calls. Prompt caching is what keeps the bill sane.
- **mod-109-minimal-eval-and-regression-checks** (this track) — the latency numbers you produce in exercise 05 are one of the regression signals you will watch there.
- **`mod-105-first-agent`** (this track) and **`agentic-ai-engineer-learning`** (next rung, level 30) — agent loops and multi-agent coordination inherits every failure and latency lesson from this module and adds a planning layer on top.

## What is deliberately not on this list

- Wrapper frameworks that bury the streaming or retry surface behind a decorator. Those are fine to use later; learning what they abstract first — which this module does — makes their failures debuggable.
- Blog posts titled "How we cut our LLM latency by 90%". They are almost always either specific to a stack you do not run or measurements taken without the discipline chapter 5 spells out. Read the primary references above instead.
- Provider pricing screenshots or specific token-per-second numbers. Both change often; a link to the current model page is more useful than yesterday's benchmark.
