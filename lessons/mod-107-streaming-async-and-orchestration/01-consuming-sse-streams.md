# Chapter 1 — Consuming SSE streams from OpenAI and Anthropic

Every request you have made so far in this track has been blocking: you sent a request, waited for the entire response to be generated on the server, and then processed it. Streaming inverts that. You start reading the response the moment the model starts writing it — one small chunk at a time, over a long-lived HTTP connection.

This chapter is about the transport (Server-Sent Events), the two provider-specific event shapes riding on top of it, and how to render partial output to a console or an HTTP client without buffering the whole answer first.

## Motivation

For a 400-token response, streaming can turn a 5-second wall-clock latency into a 300-ms *feel*. The user sees the answer forming word-by-word instead of watching a spinner. That perceived-latency win is why every serious chat product streams — and why chapter 5 spends a whole section warning you not to confuse "feels faster" with "is faster."

Two more concrete reasons streaming matters:

- **Long generations become interactive.** A 2000-token summary rendered on completion is a 20-second delay. Streamed, it is 400 ms to first character, then continuous.
- **You can cut generation short.** If the user closes the tab, your server can close the socket to the model API and stop paying for tokens that will never be seen. Chapter 3 wires that cancellation into an async server.

## Server-Sent Events (SSE) in one page

SSE is a one-way, text-based streaming protocol layered on top of HTTP/1.1. The server responds with `Content-Type: text/event-stream`, keeps the connection open, and pushes one **event** at a time. An event is a small block of lines terminated by a blank line:

```
event: message_delta
data: {"delta": {"text": "Hel"}}

event: message_delta
data: {"delta": {"text": "lo"}}

event: message_stop
data: {}

```

Rules to know:

- Each line is either `field: value` or an empty line. The empty line is a **record terminator** — it means "the event above is complete, dispatch it."
- The `data:` field carries the payload. If a payload spans multiple `data:` lines within one event, the client is supposed to join them with `\n`.
- The `event:` field is an optional event name. If absent, the default name is `"message"`.
- Comment lines start with `:` and are typically used as keep-alives (e.g. `: ping`). Ignore them; do not parse them as data.

Reference: the WHATWG HTML Living Standard defines SSE. See <https://html.spec.whatwg.org/multipage/server-sent-events.html>. The MDN page is friendlier: <https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events>.

Two things SSE is **not**: it is not WebSocket (SSE is one-way, server-to-client, and rides plain HTTP), and it is not a raw byte stream (the record structure matters — you cannot just decode chunks as they arrive without respecting the blank-line boundary).

## Anthropic streaming

Anthropic's Messages API streams a well-defined sequence of typed events. You either read them via the SDK's stream helper or parse the raw SSE yourself.

### The SDK helper — recommended default

```python
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-opus-4-7",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a haiku about SSE streams."}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
    final_message = stream.get_final_message()
```

Three things this snippet already gets right:

- `stream.text_stream` yields the *text deltas only* — the SDK has already unpacked `content_block_delta` events for you and thrown away the wrapping. That is what you want 90% of the time.
- `print(..., end="", flush=True)` renders each chunk immediately. Without `flush=True` the terminal buffers per-line and your streaming effect disappears.
- `stream.get_final_message()` reassembles the complete `Message` object after the stream closes — useful when you need the final token counts, stop reason, or usage.

Reference: <https://docs.anthropic.com/en/api/messages-streaming>.

### The event shape when you need it

Underneath, Anthropic sends a predictable sequence of typed events per response:

| Event | When | You typically want… |
|---|---|---|
| `message_start` | First event. Contains model, initial usage, empty content. | Log the model and start a per-request timer. |
| `content_block_start` | A new content block begins (text, tool_use, etc.). | Note the block type. |
| `content_block_delta` | Incremental content for the current block. Text deltas carry `{"type": "text_delta", "text": "..."}`. Tool arguments arrive as `{"type": "input_json_delta", "partial_json": "..."}`. | Append to a per-block buffer. |
| `content_block_stop` | Current content block is finished. | Finalise the block; on tool_use, parse the accumulated JSON. |
| `message_delta` | Incremental updates to the outer message (stop reason, usage). | Record the final stop reason and token counts. |
| `message_stop` | Terminal event. | Close the stream. |
| `ping` | Periodic keep-alive. | Ignore. |

If you build the parser yourself (chapter's exercise walks you through it) the loop is: read lines, group on blank line, on `event:` name switch on the type, on `data:` `json.loads` the payload, and dispatch.

## OpenAI streaming

OpenAI's Chat Completions endpoint streams by passing `stream=True`. The response is an iterator of **chunk objects** — small, partially-populated deltas of the final response shape.

### The SDK helper — recommended default

```python
from openai import OpenAI

client = OpenAI()

stream = client.chat.completions.create(
    model="gpt-4.1",
    stream=True,
    messages=[{"role": "user", "content": "Write a haiku about SSE streams."}],
)

for chunk in stream:
    delta = chunk.choices[0].delta
    if delta.content:
        print(delta.content, end="", flush=True)
```

<!-- needs-research: confirm the currently recommended default OpenAI streaming model and whether stream_options={"include_usage": True} is still the way to get final usage on Chat Completions as of 2026-08 — check https://platform.openai.com/docs/api-reference/chat/streaming. -->

Notes:

- `chunk.choices[0].delta.content` is a string containing the next batch of text, or `None` on chunks that carry no text (e.g. the first chunk carrying only role, or a chunk carrying `tool_calls`).
- `chunk.choices[0].finish_reason` becomes non-null on the *final* chunk. Watch for it if you need to know why generation stopped without waiting for the loop to exit.
- To get final usage totals with Chat Completions streaming, pass `stream_options={"include_usage": True}`. The final chunk then carries a populated `usage` field.

Reference: <https://platform.openai.com/docs/api-reference/chat/streaming>.

### The raw SSE frames

Under the hood, each chunk arrives as an SSE `data:` line whose payload is a JSON object of the same shape as a regular Chat Completion, but with `delta` in place of `message`. The stream terminates with a literal `data: [DONE]` line — that is OpenAI-specific and not part of the SSE standard. If you parse the SSE yourself, treat `[DONE]` as your stop condition and do not try to `json.loads` it.

## Streaming tool calls, not just text

Both providers stream tool-call arguments too, and the failure modes are worth knowing before chapter 2 goes deep.

- **Anthropic:** the `content_block_delta` events for a tool_use block carry `partial_json` deltas. The JSON is not valid until `content_block_stop` fires. Do not `json.loads` in the middle.
- **OpenAI:** each streamed `tool_calls[i].function.arguments` is a partial JSON string. Concatenate all deltas for the same tool-call index, then parse at `finish_reason == "tool_calls"`.

In both cases: **buffer, then parse**. Trying to parse mid-stream is the class of bug chapter 2 exists to prevent.

## Rendering partial output to a console

For a terminal, the pattern above (`print(..., end="", flush=True)`) is enough. Two extra habits pay off quickly:

- **Do not overwrite text you already printed.** Streaming is append-only; a partial word will be corrected as more tokens arrive, but the earlier characters should not move. If you find yourself using `\r` and clearing lines, you are trying to render structured state, not stream — go read chapter 2.
- **Print a per-response timing summary at the end.** Elapsed wall-clock, time-to-first-token, and total tokens. It is three lines of code and it will save you from ever having to guess whether "it feels slow" is actually true. Chapter 5 makes this rigorous.

## Rendering partial output to an HTTP client

If your product is a web frontend, you almost always want to forward the model stream to the browser as its own SSE stream. FastAPI makes this easy:

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import anthropic

app = FastAPI()
client = anthropic.Anthropic()

@app.get("/haiku")
def haiku():
    def event_source():
        with client.messages.stream(
            model="claude-opus-4-7",
            max_tokens=256,
            messages=[{"role": "user", "content": "Haiku about SSE."}],
        ) as stream:
            for text in stream.text_stream:
                # SSE frame: one data line, blank line as terminator.
                yield f"data: {text}\n\n"
            yield "event: done\ndata: {}\n\n"

    return StreamingResponse(event_source(), media_type="text/event-stream")
```

Two practical points:

- **`StreamingResponse` writes bytes as your generator yields them.** No buffering. Confirm this end-to-end — Nginx and some CDN configurations will buffer `text/event-stream` unless explicitly told not to. Set `X-Accel-Buffering: no` if you sit behind Nginx.
- **JSON payloads inside SSE `data:` lines must not contain literal newlines.** SSE uses `\n` as a field terminator; you will parse the client side to garbage if a `data:` payload spans lines. Either escape newlines (`json.dumps` does it for you) or emit multiple `data:` lines and rely on the spec's join-on-newline rule.

FastAPI's `StreamingResponse` reference: <https://fastapi.tiangolo.com/advanced/custom-response/#streamingresponse>. For the browser side, `EventSource` is the built-in: <https://developer.mozilla.org/en-US/docs/Web/API/EventSource>.

## Streaming without an SDK — `httpx`

Sometimes you want to read the raw stream yourself: a hosted-model proxy, a language the SDK does not target, or a debugging session where you need to see exactly what the wire is doing. `httpx` has first-class streaming:

```python
import httpx, json, os

with httpx.stream(
    "POST",
    "https://api.anthropic.com/v1/messages",
    headers={
        "x-api-key": os.environ["ANTHROPIC_API_KEY"],
        "anthropic-version": "2023-06-01",
        "content-type": "application/json",
    },
    json={
        "model": "claude-opus-4-7",
        "max_tokens": 256,
        "stream": True,
        "messages": [{"role": "user", "content": "Haiku about SSE."}],
    },
    timeout=60.0,
) as response:
    response.raise_for_status()
    event_name = None
    for line in response.iter_lines():
        if not line:                    # blank line → event boundary
            event_name = None
            continue
        if line.startswith(":"):        # comment / keep-alive
            continue
        if line.startswith("event:"):
            event_name = line[6:].strip()
        elif line.startswith("data:"):
            payload = json.loads(line[5:].strip())
            if event_name == "content_block_delta":
                delta = payload.get("delta", {})
                if delta.get("type") == "text_delta":
                    print(delta["text"], end="", flush=True)
```

<!-- needs-research: confirm the current Anthropic API version header value ("anthropic-version") required for the Messages API as of 2026-08 — check https://docs.anthropic.com/en/api/versioning. -->

This is exactly what the SDK's stream helper does under the hood, minus the SDK's typed content-block objects. Write it once as an exercise and you will forever stop being confused by "why does my stream sometimes freeze" — the answer is almost always "your reader is buffering on newline instead of on the blank-line terminator."

Reference: <https://www.python-httpx.org/async/#streaming-responses>.

## What can go wrong

Six failure modes you will meet:

- **Buffering somewhere in the middle.** A proxy, a load balancer, your framework's response wrapper, or your terminal all can decide to buffer bytes until they see a newline or until the connection closes. If your "streaming" output arrives all at once, walk the path from server to eye and disable buffering at each hop.
- **Parsing per line instead of per event.** SSE events are terminated by a **blank line**, not by every newline. If you `json.loads` each `data:` line without waiting for the blank line, multi-line `data:` payloads (rare but legal) will crash.
- **Trying to parse mid-stream JSON.** Tool arguments and structured outputs stream as chunks that are not valid JSON on their own until the last chunk arrives. Chapter 2 is the whole story.
- **Ignoring keep-alive comments.** Anthropic and many proxies send `: ping` or `event: ping` frames every few seconds. If your parser errors on `:`-prefixed lines or on unknown event names, the stream will die after ~15 seconds for no visible reason.
- **Not closing the stream on disconnect.** If the client closes the socket, your server should stop reading from the model and close its connection to the model API too. Otherwise you keep paying for tokens the user will never see. Chapter 3 wires this into the async loop.
- **`stream_options={"include_usage": True}` missing on OpenAI.** Without it, the final streamed chunk has no `usage` field and you cannot report cost per request. Add it once and forget.

## Summary

- Streaming turns a wait into a scroll. It is a perceived-latency win, not a wall-clock one — chapter 5 keeps that distinction honest.
- The transport is Server-Sent Events: text/event-stream, `data:` payload lines, blank line as event terminator, `:` comments as keep-alives. Parse to the spec, not to the "usually looks like" shape.
- Anthropic streams a typed sequence of events (`message_start`, `content_block_delta`, `message_stop`, …). The SDK's `stream.text_stream` gives you the text-only fast path.
- OpenAI streams `ChatCompletionChunk` objects with `delta.content` pieces and a final `[DONE]` marker. Pass `stream_options={"include_usage": True}` if you need usage totals.
- Tool-call arguments stream too — but the JSON is not valid until the block finishes. Buffer then parse; never mid-stream.
- Rendering to a browser is best done by forwarding the model stream as your own SSE stream (`StreamingResponse` in FastAPI, `EventSource` on the client). Disable proxy buffering.
- Prefer the SDK helpers by default; drop to `httpx.stream` when you need to see the raw wire or run outside the SDK's language.

The next chapter picks up the trickier case: streaming *structured* output — JSON that only becomes valid at the very end — and how to render partial state without misrendering it.
