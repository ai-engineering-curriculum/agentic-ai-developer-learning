# Lab 01 — Dissect an LLM HTTP Call

**Estimated duration:** 45–60 minutes
**Covers:** [chapter 1 — What an LLM API Actually Is](../01-what-is-an-llm-api.md), [chapter 6 — Streaming Responses](../06-streaming-responses.md)

## Objective

The SDK hides the wire. In this lab you will issue a chat request **without** the SDK — once non-streaming, once streaming — so you can see the exact JSON that goes over the wire, the response headers (model, request ID, rate-limit counters), and the raw Server-Sent Events (SSE) stream. This is the mental model you need before the SDK is your only lens.

By the end you will have:

- An HTTP POST that produces a complete non-streaming response, inspected header-by-header.
- The same call in streaming mode, with each `event:` / `data:` line read from the socket.
- A reconstructed assistant message built by concatenating `content_block_delta` text pieces (or OpenAI delta chunks).

> This lab complements [exercise-01](../exercises/exercise-01-first-llm-api-calls.md), which uses the SDK. Here we go one level lower.

## Prerequisites

- Finished chapters 1 and 6.
- A funded API key for **one** provider:
  - Anthropic: `ANTHROPIC_API_KEY` exported in your shell.
  - OpenAI: `OPENAI_API_KEY` exported in your shell.
- `curl` 7.70+ on your path. (Any recent macOS/Linux has it; on Windows use WSL or Git Bash.)
- Python 3.10+ for the streaming walk-through, with `httpx` installed: `pip install httpx`.

The lab shows both providers side-by-side. Pick the one whose key you have and run those commands; the other block is for reference.

## Step 1 — A single non-streaming POST with curl

Pick the provider you have a key for and run the matching command. We ask for a short reply so the response is easy to read.

**Anthropic (Messages API):**

```bash
curl -sS https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -D headers.txt \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 64,
    "system": "You are a precise technical assistant. Reply in one sentence.",
    "messages": [
      { "role": "user", "content": "What is an LLM API, in one sentence?" }
    ]
  }' | tee response.json | python -m json.tool
```

**OpenAI (Chat Completions):**

```bash
curl -sS https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "content-type: application/json" \
  -D headers.txt \
  -d '{
    "model": "gpt-4.1-mini",
    "max_tokens": 64,
    "messages": [
      { "role": "system", "content": "You are a precise technical assistant. Reply in one sentence." },
      { "role": "user",   "content": "What is an LLM API, in one sentence?" }
    ]
  }' | tee response.json | python -m json.tool
```

### Expected output (shape, not exact text)

A single JSON object. The fields you want to find:

- A one-sentence assistant reply inside the response.
- A `usage` block with `input_tokens` and `output_tokens` (Anthropic) or `prompt_tokens` / `completion_tokens` (OpenAI).
- A `stop_reason` / `finish_reason` of `end_turn` or `stop` (clean finish).
- A `model` field echoing the model ID the server actually used.

### Checkpoint

Answer these from `response.json` and `headers.txt` **without re-running the call**:

1. Which header carries the provider's **request ID**? (Hint: `request-id` or `x-request-id` on the two providers.)
2. What is the exact `model` string the server reports — does it match what you sent, or did an alias resolve to a dated snapshot?
3. What are `input_tokens` and `output_tokens` for your call? Does `output_tokens` match or exceed `max_tokens=64`? (If it equals 64, the reply was truncated and `stop_reason` will say so.)

Chapter 1 described the request shape as four parameter clusters (which model, what context, how to sample, optional shaping). Point to one field in your request body for each cluster.

## Step 2 — Switch on streaming and read the SSE lines

Still without the SDK, add `"stream": true` and watch the raw SSE lines appear on stdout. `curl -N` disables buffering so you see them in real time.

**Anthropic:**

```bash
curl -N -sS https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -H "accept: text/event-stream" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 120,
    "stream": true,
    "messages": [
      { "role": "user", "content": "Explain SSE in two short sentences." }
    ]
  }'
```

**OpenAI:**

```bash
curl -N -sS https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4.1-mini",
    "max_tokens": 120,
    "stream": true,
    "messages": [
      { "role": "user", "content": "Explain SSE in two short sentences." }
    ]
  }'
```

### Expected output (shape)

Blocks of two or three lines separated by blank lines, exactly as chapter 6 described.

Anthropic:

```
event: message_start
data: {"type":"message_start", ...}

event: content_block_start
data: {"type":"content_block_start", "index":0, ...}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"SSE"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":" is"}}

...

event: message_stop
data: {"type":"message_stop"}
```

OpenAI chunks end with the literal sentinel line:

```
data: [DONE]
```

### Checkpoint

- Which event in your transcript carries the token-usage totals? (Anthropic: look at `message_delta`'s `usage`; OpenAI: look at the final chunk when `stream_options.include_usage` is set, otherwise usage is omitted on streaming.)
- The deltas are "pure additions" — reconstruct the full assistant text by concatenating every `text_delta` (Anthropic) or `delta.content` (OpenAI) in order. Does your concatenation match a non-streaming call to the same prompt in meaning?
- Chapter 6 lists `ping` events as "keepalive; ignore." Did you see any?

## Step 3 — A Python streaming client without the SDK

Now replay the stream with `httpx` so you can process events programmatically. This is the shape of your own backend relay from chapter 6.

Save as `raw_stream.py`:

```python
import json
import os
import sys
import time

import httpx

PROVIDER = os.environ.get("PROVIDER", "anthropic")  # or "openai"

def anthropic_request():
    return {
        "url": "https://api.anthropic.com/v1/messages",
        "headers": {
            "x-api-key": os.environ["ANTHROPIC_API_KEY"],
            "anthropic-version": "2023-06-01",
            "accept": "text/event-stream",
            "content-type": "application/json",
        },
        "json": {
            "model": "claude-sonnet-4-6",
            "max_tokens": 200,
            "stream": True,
            "messages": [{"role": "user", "content": "List three uses of SSE in two sentences each."}],
        },
    }

def openai_request():
    return {
        "url": "https://api.openai.com/v1/chat/completions",
        "headers": {
            "Authorization": f"Bearer {os.environ['OPENAI_API_KEY']}",
            "content-type": "application/json",
        },
        "json": {
            "model": "gpt-4.1-mini",
            "max_tokens": 200,
            "stream": True,
            "messages": [{"role": "user", "content": "List three uses of SSE in two sentences each."}],
        },
    }

req = anthropic_request() if PROVIDER == "anthropic" else openai_request()

t0 = time.monotonic()
t_first = None
pieces = []

with httpx.stream("POST", req["url"], headers=req["headers"], json=req["json"], timeout=30.0) as r:
    r.raise_for_status()
    for line in r.iter_lines():
        if not line or not line.startswith("data:"):
            continue
        payload = line[len("data:"):].strip()
        if payload == "[DONE]":
            break
        event = json.loads(payload)

        if PROVIDER == "anthropic":
            if event.get("type") == "content_block_delta":
                text = event["delta"].get("text", "")
            else:
                text = ""
        else:
            text = (event.get("choices", [{}])[0].get("delta", {}) or {}).get("content", "") or ""

        if text:
            if t_first is None:
                t_first = time.monotonic()
            pieces.append(text)
            sys.stdout.write(text)
            sys.stdout.flush()

t_end = time.monotonic()
print()
print(f"[ttft={t_first - t0:.3f}s total={t_end - t0:.3f}s chars={sum(len(p) for p in pieces)}]")
```

Run it:

```bash
PROVIDER=anthropic python raw_stream.py   # or PROVIDER=openai
```

### Expected output

Tokens stream to stdout in real time, followed by a `[ttft=... total=... chars=...]` line. `ttft` is time-to-first-token (chapter 6/7); `total` is total wall time.

### Checkpoint

- Does your `ttft` land in the hundreds of milliseconds? If it is suspiciously high or arrives in one big chunk, chapter 6 warns the proxy chain may be buffering — check that `accept: text/event-stream` is honored by any intermediary.
- Kill the process mid-stream with Ctrl-C. The partial tokens you already saw are **already billed** (chapter 6, "What streaming costs you"). Does your program handle that cleanly, or did it crash?

## Debrief

You just saw everything the SDK normally hides:

- A JSON body with four parameter clusters (model, context, sampling, shaping).
- Response headers carrying the request ID, model ID, and (if you look) rate-limit counters.
- A `usage` block as the authoritative source of billed tokens.
- A stream of SSE events where deltas are pure additions and the final event closes the stream.

Whenever an SDK behaves surprisingly in a later module, drop back to this level. The bytes do not lie.

## Stretch

- Pipe `curl -N` output through `grep ^event:` to count how many of each event type you receive on a longer prompt.
- Add a 1-second `time.sleep(1)` inside the Python loop between reads. Does the server stall, or does it buffer? Chapter 6 flags "backpressure" — this is how to see it.
- Repeat Step 1 with a deliberately bad `model` string ("gpt-made-up-mini"). Record the HTTP status, response body, and `request-id`. Which error class in chapter 8's taxonomy does this fall under?
