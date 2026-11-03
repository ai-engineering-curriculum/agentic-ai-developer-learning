# Exercise 01 — First SSE stream consumer

Paired with [chapter 1 — consuming SSE streams from OpenAI and Anthropic](../01-consuming-sse-streams.md).

**Estimated effort:** 60–90 minutes.

## Objective

Get one streaming LLM response rendering into your terminal, one character at a time, with the wire format visible. Then reproduce the same behaviour without the SDK's stream helper — parsing raw SSE frames yourself — so the abstraction is no longer magic.

## Problem statement

Write a small CLI that takes a prompt on the command line, streams the model's response to standard output as it is generated, and prints a per-request timing summary at the end. Run it against at least one provider (Anthropic *or* OpenAI). If you have credit for both, do both.

The exercise has two variants; do them in order:

- **Variant A — SDK path.** Use the provider SDK's streaming helper. This is the version you would actually ship.
- **Variant B — raw SSE path.** Use `httpx.stream` directly. Parse the SSE frames yourself, dispatching on event name and printing text deltas as they arrive.

Variant B is the one that will teach you the most. Do not skip it.

## Requirements

1. The script reads a prompt from `argv` (or stdin) and streams the answer to stdout with `flush=True` on every write, so the streaming effect is visible.
2. On completion, print a three-line summary:
   - Wall-clock time to first visible character (TTFT), in milliseconds.
   - Wall-clock time from request start to last chunk (TTC), in seconds.
   - Total output tokens (from the SDK's final usage on variant A; from the final message event's usage on variant B).
3. **Variant A** uses the provider's streaming helper (`client.messages.stream` on Anthropic; `stream=True` on OpenAI's Chat Completions with `stream_options={"include_usage": True}`).
4. **Variant B** uses `httpx.stream` directly and parses SSE by hand:
   - Split the stream on blank lines to identify event boundaries.
   - Ignore comment lines (`:` prefix).
   - On Anthropic: dispatch on `event:` name; only print `text_delta` deltas inside `content_block_delta` events.
   - On OpenAI: recognise the `data: [DONE]` sentinel as end-of-stream (do not `json.loads` it); everything else is a JSON chunk whose `choices[0].delta.content` is your next batch of text.
5. Confirm both variants produce the same output text on the same prompt (modulo model non-determinism if temperature > 0 — use `temperature=0` for this comparison).
6. Handle stream errors: if the request is rejected before streaming starts (auth failure, bad model name), print a clear error and exit non-zero. If the connection drops mid-stream, print what you got, note the exception, and exit non-zero.

## Starter guidance

Skim these before you start:

- Anthropic streaming reference: <https://docs.anthropic.com/en/api/messages-streaming>
- OpenAI streaming reference: <https://platform.openai.com/docs/api-reference/chat/streaming>
- SSE specification (skim): <https://html.spec.whatwg.org/multipage/server-sent-events.html>
- `httpx` streaming: <https://www.python-httpx.org/async/#streaming-responses> (the sync counterpart is `httpx.stream`)

For variant B, prefer `response.iter_lines()` over `response.iter_bytes()` — it handles the line-splitting for you but *not* the blank-line boundary logic. You still have to accumulate lines into events yourself.

Do not use `sseclient-py` or similar SSE-parsing libraries for variant B. The whole point is that you read the frames.

## Acceptance criteria

- Running your script with a prompt like `"Write a paragraph about the history of Server-Sent Events."` streams the answer to your terminal character-by-character (or chunk-by-chunk in visible bursts, not all at once at the end).
- Your TTFT number is reasonable — for the small prompts above, in the low hundreds of milliseconds on a normal connection. If your TTFT equals your TTC, your output is not actually streaming: some layer is buffering.
- On variant B against Anthropic, your script prints text as `content_block_delta` events arrive, ignores `ping` / `message_start` / `message_stop` events except for the summary, and correctly handles multi-line `data:` payloads (join with `\n`).
- On variant B against OpenAI, your script exits cleanly when it sees `data: [DONE]` — no `JSONDecodeError` on the sentinel.
- Deliberately breaking the model name (e.g. `claude-doesnt-exist`) produces a controlled error print and non-zero exit, not a stack trace.
- Deliberately terminating the process with Ctrl-C during a long generation stops the stream cleanly. Compare the timing summary — you should have TTC less than the full completion time, matching when you hit Ctrl-C.

## Stretch goals

- Replace `print(..., flush=True)` with a small function that prints characters at a fixed rate (e.g. clamp to 60 tokens/s). Streaming feels *worse* when it is faster than reading speed and stutters; a light rate limit smooths the render.
- Add a `--server` flag that spins up a FastAPI endpoint using `StreamingResponse` and forwards the model stream to the browser as its own SSE. Open the endpoint with an `EventSource` snippet in a browser tab and confirm the browser sees the same character-by-character streaming.
- Save the raw SSE frames to a file with `--record path.sse`. Later, replay them with `--replay path.sse` (skipping the network entirely) so you can iterate on your parser without spending API credit.
- On the SDK-path variant, force a mid-stream error by wrapping the SDK client with a mock that raises `httpx.RemoteProtocolError` after N chunks. Confirm your error path fires and the timing summary reflects the partial stream.
