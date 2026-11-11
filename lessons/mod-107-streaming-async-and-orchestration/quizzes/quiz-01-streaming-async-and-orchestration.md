# Quiz 01 — Streaming, Async, and Parallel LLM Orchestration

**Covers:** chapters 1–5 of `mod-107-streaming-async-and-orchestration`.
**Format:** 12 multiple-choice questions. Pick the single best answer for each. Answer key with per-answer explanations is at the bottom.

Treat this as a self-check: open the chapters when you want to look something up, but try to answer first.

---

## 1. Chapter 1 says that in SSE, one "event" is terminated by…

- A. A single newline (`\n`).
- B. A blank line — the empty line between two `field: value` blocks is what tells the client "dispatch the event above."
- C. A literal `data: [DONE]` sentinel, the way OpenAI terminates its stream.
- D. A `Content-Length` header on the response, counted in bytes.

## 2. You wire the OpenAI Chat Completions `stream=True` loop, but your `usage` field at the end is always empty and you cannot report cost per request. The chapter's fix is…

- A. Call `client.usage.get(request_id)` after the stream closes.
- B. Pass `stream_options={"include_usage": True}`; the final chunk then carries a populated `usage` field.
- C. Sum `len(delta.content)` across chunks and divide by an average bytes-per-token ratio.
- D. Switch to a non-streaming call for cost accounting; streaming responses never include usage on OpenAI.

## 3. You are parsing an Anthropic SSE stream by hand with `httpx`. Mid-stream your reader starts raising on a line that reads `: ping`. Chapter 1 says the right handling is…

- A. Treat `:`-prefixed lines as comment / keep-alive frames and skip them; parsing them as data will kill the stream after ~15 seconds.
- B. `json.loads` them anyway — Anthropic ping events carry a timestamp you want.
- C. Close the connection and reconnect from the last event you saw.
- D. Switch `anthropic-version` to a newer header value that disables keep-alives.

## 4. A streamed tool-call argument arrives across several chunks. Chapter 2 names exactly one legal time to `json.loads` the accumulated buffer. Which?

- A. On every `content_block_delta` event — if it fails, retry on the next chunk.
- B. After the first `}` byte arrives in the stream (that's the end of the first object).
- C. Only once `content_block_stop` (Anthropic) or `finish_reason == "tool_calls"` (OpenAI) has fired; buffer, then parse, then act.
- D. Only on the final `message_stop` event at the very end of the whole response.

## 5. The model is streaming a report. The current buffer ends `..., "risk_level": "l`. Your UI wants to render `risk_level`. What does chapter 2 say?

- A. Render `"l"` — it's the beginning of a legal enum string.
- B. Render a lowercased best-guess (`"low"`); enum completions are almost always the shortest member.
- C. Render nothing for `risk_level`. The string is still open — wait for the closing `"` before showing any value. A later chunk may make it `"low"` or `"limited"`.
- D. Render it as `"unknown"` and overwrite when the field completes.

## 6. Chapter 3 lists two concurrency controls that work together. Which pair, and what role does each play?

- A. `asyncio.Semaphore` is the primary; `httpx.Limits(max_connections=...)` is redundant and should be removed.
- B. `asyncio.Semaphore(concurrency)` expresses your *intent* (don't exceed N in-flight requests); `httpx.Limits(max_connections=...)` is a transport-layer *safety net* that backs up extra requests on the connection pool. Use both.
- C. `asyncio.gather(..., limit=N)` caps concurrency; `httpx.Limits` configures retries.
- D. `asyncio.Semaphore` on the client side; `Retry-After` header on the server side. No other controls are required.

## 7. The chapter recommends `asyncio.TaskGroup` over `asyncio.gather` for anything larger than a one-shot batch. The reason it gives is…

- A. `TaskGroup` runs tasks on a dedicated thread pool and bypasses the GIL.
- B. `TaskGroup` is faster per-task because it skips the event-loop scheduler.
- C. Structured concurrency: if any task raises, `TaskGroup` cancels the rest and re-raises as an `ExceptionGroup`; the `async with` block does not exit until every task has finished or been cancelled, so no task leaks.
- D. `gather` was deprecated in Python 3.11 and will be removed in 3.14.

## 8. In an async FastAPI endpoint that streams a model response to a browser, chapter 3 says the client-disconnect check belongs where?

- A. Once, at the top of the handler, before the stream starts.
- B. Between every yielded chunk — if you only check once, you gain nothing, because the whole point is to stop generation early when the user goes away.
- C. In a background task that polls the connection every 5 seconds.
- D. Nowhere — FastAPI handles disconnection transparently; manual checks are redundant.

## 9. Which of the following is listed by chapter 4 as a **terminal** error — one you should *not* retry?

- A. HTTP 429 (rate limit) with a `retry-after` header.
- B. HTTP 503 Service Unavailable.
- C. `httpx.ReadTimeout` on an otherwise-healthy request.
- D. HTTP 400 `invalid_request_error` for a context-length violation (and your code has not just trimmed the transcript).

## 10. Ten clients all receive a 429 at the same instant and all retry with the formula `delay = base * 2 ** attempt`. Chapter 4 calls this a specific failure mode because…

- A. Exponential backoff with no jitter sends every retry at exactly the same second, so the provider gets a synchronised burst — a thundering herd. **Full jitter** (uniform between 0 and the current delay) breaks the synchronisation.
- B. `2 ** attempt` overflows the retry counter and wraps to zero after 30 attempts.
- C. Exponential backoff violates the SSE spec; retries should be linear.
- D. The clients should all call `time.sleep(0)` instead to let the provider recover on its own schedule.

## 11. Chapter 5 is emphatic that you should not report the **mean** latency for an LLM feature. The reason is…

- A. The mean cannot be computed from streaming responses — only from blocking ones.
- B. LLM latency distributions are heavy-tailed; a few outliers dominate the mean, so it reflects the worst recent burst, not what a typical user sees. Report `p50`, `p95`, `p99` — the median tells you "typical," the upper percentiles make the tail visible.
- C. Means require a sample size of at least 10,000 to be meaningful; LLM load tests never gather that many samples.
- D. Means include retry sleeps and percentiles do not, so the two cannot be compared.

## 12. A streaming feature has `TTFT p50 = 320 ms` and `TTC p50 = 4.8 s`. Chapter 5 says each number tells you something different. Which description matches the chapter?

- A. TTFT and TTC measure the same thing — total round-trip latency — but on different providers.
- B. TTFT is wall-clock from `t0` to the first content chunk and is what determines whether the user thinks the app "responded"; TTC is wall-clock from `t0` to the last chunk and is what constrains any downstream step that waits for the full response. A streaming feature needs both.
- C. TTFT is a client-side metric; TTC is a server-side metric; you cannot compare them.
- D. TTFT is for OpenAI, TTC is for Anthropic — the two providers measure latency at different points.

---

## Answer key

1. **B.** Chapter 1 ("Server-Sent Events (SSE) in one page") spells it out: "The empty line is a **record terminator** — it means 'the event above is complete, dispatch it.'" The chapter's "What can go wrong" section lists "parsing per line instead of per event" as a specific bug.
2. **B.** Chapter 1 ("OpenAI streaming" → notes and "What can go wrong") names the fix exactly: "To get final usage totals with Chat Completions streaming, pass `stream_options={"include_usage": True}`. The final chunk then carries a populated `usage` field."
3. **A.** Chapter 1 ("Server-Sent Events (SSE) in one page" and "What can go wrong" → "Ignoring keep-alive comments") says: "Comment lines start with `:` and are typically used as keep-alives (e.g. `: ping`). Ignore them; do not parse them as data." If your parser errors on `:`-prefixed lines the stream dies after ~15 seconds for no visible reason.
4. **C.** Chapter 2's "Streaming tool_use for tool execution" and chapter 1's "Streaming tool calls" both land on the same rule: "buffer, then parse" — wait for `content_block_stop` (Anthropic) or `finish_reason == "tool_calls"` (OpenAI) before `json.loads`. Parsing mid-stream is "the class of bug chapter 2 exists to prevent."
5. **C.** Chapter 2 ("When is a partial parse legal?" and the catalogue in "Recognising legal-vs-illegal partial parses") uses the truncated-enum case as a worked example. The rule: "render complete tokens (whole strings, whole numbers, whole arrays, whole objects) — never partial ones."
6. **B.** Chapter 3 ("Bounded concurrency with `asyncio.Semaphore`" and "`httpx.AsyncClient` — reuse the connection pool") describes exactly this split: the semaphore is your intent, `httpx.Limits(max_connections=...)` is the transport-level safety net. The chapter's summary repeats: "Bound concurrency with `asyncio.Semaphore` at the intent layer *and* `httpx.Limits(max_connections=...)` at the transport layer."
7. **C.** Chapter 3 ("`TaskGroup` — structured concurrency with cancellation") lists the two guarantees verbatim: on any task raising, the group cancels the rest and re-raises an `ExceptionGroup`; the `async with` block does not exit until every task has finished or been cancelled. `gather` is not deprecated (D is wrong); concurrency does not escape the GIL (A is wrong).
8. **B.** Chapter 3 ("Cancelling on client disconnect" → "Three practical points") says: "Check disconnect between yields. If you only check once at the start you gain nothing. Do it on every chunk."
9. **D.** Chapter 4 ("Terminal — do not retry") lists `400 Bad Request` with a schema or context-length error as terminal: "Your request is wrong. Retrying sends the same wrong request." 429 with `retry-after` (A), 503 (B), and `httpx.ReadTimeout` (C) are all named as *transient* in the preceding list.
10. **A.** Chapter 4 ("Motivation" and "Exponential backoff plus jitter") spells this out: synchronised retries produce a thundering herd; full jitter — `random.uniform(0, min(cap, base * 2 ** n))` — breaks the pattern. The chapter cites the AWS Builders' Library article on jittered backoff.
11. **B.** Chapter 5 ("Percentiles: p50, p95, p99, and never the mean") is explicit: LLM latency is heavy-tailed; "the mean of a heavy-tailed distribution is dominated by the worst outliers; the *median* (p50) tells you what a typical user sees; the *p95* and *p99* tell you what the worst-serviced 5% and 1% of users see."
12. **B.** Chapter 5 ("The four measurements that matter" and "What 'streaming feels faster' actually means") defines TTFT = `t2 - t0` as perceived latency ("whether the user thinks the app 'responded'") and TTC = `tN - t0` as "what determines whether the next step in your workflow can start." Options A, C, and D are not claims the chapter makes.
