# Lab 01 — Curl an SSE Stream Byte-by-Byte

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 1 — Consuming SSE streams from OpenAI and Anthropic](../01-consuming-sse-streams.md) ("Server-Sent Events (SSE) in one page", "Anthropic streaming", and "What can go wrong" → "Buffering somewhere in the middle")

## Objective

Chapter 1 teaches the SSE wire format as a spec: `event:` / `data:` lines, blank-line event terminators, `:` comment lines as keep-alives, and the specific failure mode of a buffer sitting between you and the stream. In this lab you will hit the Anthropic Messages API directly with `curl`, watch the raw event frames scroll across the terminal at the pace the model is generating them, and then deliberately turn on buffering so you can feel the failure mode the chapter names.

No SDK. No Python. Just `curl`, so you can point at the exact bytes that cross the wire before any abstraction hides them.

By the end you will have:

- A terminal recording (or screen memory) of an SSE stream rendering live: `event: message_start`, several `event: content_block_delta`, `event: message_stop`.
- A saved `stream.sse` file with the raw frames for re-reading offline.
- A second run that produces the same bytes all at once at the end — a felt demonstration of the "buffering somewhere in the middle" failure mode.

> This lab complements [exercise-01](../exercises/exercise-01-first-sse-stream-consumer.md), which uses `httpx` to parse SSE from Python. Here we stay in the shell so you see the raw frames with no library in the way.

## Prerequisites

- Finished chapter 1.
- A funded Anthropic API key: `ANTHROPIC_API_KEY` exported in your shell. (The lab uses Anthropic because its typed-event names — `message_start`, `content_block_delta`, `message_stop` — are visible verbatim in `event:` lines; the OpenAI stream format is a variant you will try in the stretch.)
- `curl` 7.70+ on your path. `jq` is handy but not required.
- Budget: two short streaming calls. Cents.

## Step 1 — Start a stream and watch the frames scroll

Pin a dated model ID (`claude-sonnet-4-6` is fine; substitute whatever your account can call) and send a prompt long enough that you can see chunks arriving one at a time. `curl -N` disables output buffering so the terminal renders each line as it lands.

```bash
curl -N -sS https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 512,
    "stream": true,
    "messages": [
      {"role": "user", "content": "Write one paragraph about Server-Sent Events."}
    ]
  }'
```

<!-- needs-research: confirm the current `anthropic-version` header value for the Messages API is still `2023-06-01` on 2026-11-11 — check https://docs.anthropic.com/en/api/versioning. -->

### Expected output (shape)

A sequence of SSE frames scrolling down your terminal, one at a time, over several seconds. The first and last few frames should look roughly like:

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_...","model":"claude-sonnet-4-6-...","content":[],"usage":{"input_tokens":...,"output_tokens":0}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Server"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"-Sent"}}

...

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":...}}

event: message_stop
data: {"type":"message_stop"}
```

Three features of the wire that chapter 1 called out — now look at them in your own terminal output:

1. Each event is an `event:` line followed by one or more `data:` lines, **terminated by a blank line**. The blank line is the record terminator.
2. The `event:` names are the typed sequence from chapter 1's table: `message_start`, `content_block_start`, `content_block_delta`, `content_block_stop`, `message_delta`, `message_stop`.
3. You may also see a `: ping` line (a comment, used as keep-alive) or an `event: ping` frame. Chapter 1: "Ignore them; do not parse them as data."

### Checkpoint

- Did the output scroll gradually, or did it arrive in one big block at the end? If it scrolled, `curl -N` is doing its job and streaming is working end-to-end. If it arrived all at once, something on your path is buffering — jump to Step 3 early and come back.
- Can you point to the exact bytes that make up **one** `content_block_delta` event? Count the lines between two blank-line terminators.

## Step 2 — Save the raw frames and inspect them offline

Re-run the same request and `tee` the output to a file so you can re-read the SSE bytes without spending another request:

```bash
curl -N -sS https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 512,
    "stream": true,
    "messages": [
      {"role": "user", "content": "Write one paragraph about Server-Sent Events."}
    ]
  }' | tee stream.sse
```

Now poke at the frames with plain shell tools. Three questions to answer from `stream.sse`:

```bash
# How many distinct event names appeared?
grep '^event:' stream.sse | sort -u

# How many content_block_delta events carried text?
grep '"type":"text_delta"' stream.sse | wc -l

# Extract just the streamed text (concatenating text_delta payloads).
grep '"type":"text_delta"' stream.sse \
  | sed -E 's/.*"text":"([^"]*)".*/\1/' \
  | tr -d '\n'; echo
```

The last command reconstructs the full paragraph from only the `text_delta` deltas — the same operation the SDK's `stream.text_stream` performs for you in Python.

### Checkpoint

- The event-name list should contain every typed event from chapter 1's table except possibly `ping`. If you are missing `content_block_stop` or `message_stop`, your request was likely truncated (`max_tokens` too low, network drop) — note which frame was last and why.
- Compare the reconstructed paragraph to what you watched scroll in Step 1. They should match character-for-character.
- Count the `data:` lines. Did any event carry more than one `data:` line? (The SSE spec allows it; Anthropic's Messages API typically emits one `data:` per event, but your parser must still handle the multi-line case — chapter 1's "What can go wrong" lists this as a bug source.)

## Step 3 — Turn buffering on and watch streaming disappear

Chapter 1's single most common complaint about streaming is "it works in the SDK, but when I curl it, everything arrives at once." The usual culprit is a buffer somewhere between the server and your eye. Reproduce the failure mode on purpose.

Run the same curl **without** `-N`, and pipe through `cat` (which in many shells is line-buffered on its stdout):

```bash
curl -sS https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 512,
    "stream": true,
    "messages": [
      {"role": "user", "content": "Write one paragraph about Server-Sent Events."}
    ]
  }' | cat
```

### Expected output (shape)

Depending on your shell, you will see one of two patterns:

- **Blocking look:** nothing for several seconds, then a wall of SSE frames all at once at the very end. This is the chapter's "buffering somewhere in the middle" — `curl` without `-N` plus a block-buffered downstream consumer erases the streaming effect even though the server is streaming correctly.
- **Chunky look:** output arrives in large bursts separated by long pauses, instead of smoothly character-by-character. Some intermediate buffer is flushing on its own schedule, not on the model's.

Either way, note the contrast with Step 1. The server did not change its behaviour; your pipeline did.

### Checkpoint

Record in `NOTES.md`:

1. Which pattern did you see — all-at-end, or chunky bursts?
2. Where exactly is the buffer in your pipeline? (`curl`'s default stdout buffering, `cat`'s line buffering when stdout is not a terminal, your terminal emulator, the shell itself.) The chapter's "What can go wrong" lists four hops where a buffer can hide: "A proxy, a load balancer, your framework's response wrapper, or your terminal."
3. What would you do, in a FastAPI deployment behind Nginx, to prevent this failure mode? Chapter 1 names a header verbatim — write it in your notes.

## Step 4 — Write a 5-sentence debrief

In `NOTES.md`, answer in 5–7 sentences total:

1. Which `event:` names did you see in your `stream.sse`, and which one is the terminal event you must watch for?
2. What does an SSE **blank line** mean, and why does chapter 1 call "parsing per line instead of per event" a specific bug class?
3. What was the single change between Step 1 and Step 3 that made streaming look broken, and what does that tell you to look for the next time someone reports "my stream isn't streaming"?
4. If the model had been emitting a tool-call block instead of plain text, which `delta.type` would you have seen instead of `text_delta`, and what is the one rule chapter 1 gives for handling it?

That write-up is the deliverable.

## Debrief

You have now watched the wire format at the lowest level the module ever cares about: raw SSE frames, scrolling in real time, with no SDK, no `httpx`, no Python. Three habits pay forward from here:

- When a "streaming" feature is not actually streaming, curl the upstream directly with `-N` first. If `curl -N` shows frames scrolling, the problem is in your consumer. If `curl -N` shows one big blob at the end, the problem is upstream.
- When your parser crashes on a frame, look for `:`-prefixed comment lines and multi-`data:` events. Chapter 1's "Ignoring keep-alive comments" and "Parsing per line instead of per event" are the two bugs you will hit before any others.
- When you deploy behind a proxy, test the live SSE path end-to-end — do not trust that your framework's `StreamingResponse` survives the proxy. Chapter 1 flags `X-Accel-Buffering: no` for Nginx; your CDN may have its own switch.

## Stretch

- **Same request against OpenAI.** Repeat Step 1 against `https://api.openai.com/v1/chat/completions` with `"stream": true`. The response is a different SSE flavour: no `event:` names, just `data:` lines containing JSON chunks, and a terminal `data: [DONE]` sentinel. Confirm chapter 1's note that `[DONE]` is OpenAI-specific and not part of the SSE standard — if you `json.loads` it, you crash. <!-- needs-research: confirm `data: [DONE]` is still the Chat Completions streaming terminator on 2026-11-11; check https://platform.openai.com/docs/api-reference/chat/streaming. -->
- **Record and replay.** Use the saved `stream.sse` from Step 2 as a fixture: write a two-line Python snippet that reads it and prints only the `text_delta` text. You now have a zero-cost replay of a real streaming response for use in later exercises (exercise 02's `--fake-stream` adversarial tests build on exactly this idea).
- **Kill mid-stream.** Start the Step 1 curl and press Ctrl-C after two or three `content_block_delta` frames. Confirm the model side closes cleanly (no hanging socket, no leaked tokens). Chapter 3 wires the server-side equivalent (`request.is_disconnected()`) that makes this cancellation automatic in a real deployment.
- **Watch the ping.** Send a long-generating prompt (`max_tokens=4096`, "List 200 reasons…") so the response takes more than ~15 seconds. Look for `: ping` or `event: ping` frames in `stream.sse`. These are the keep-alives chapter 1 warns you must ignore.
