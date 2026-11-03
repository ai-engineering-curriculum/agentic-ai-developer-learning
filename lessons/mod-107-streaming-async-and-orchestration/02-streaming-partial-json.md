# Chapter 2 — Streaming partial structured output safely

The last chapter streamed text — a medium that concatenates trivially. This chapter streams JSON, which does not. A half-received JSON string is not a smaller JSON object; it is invalid syntax. This chapter is about the specific ways that failure hits real products, and the small set of parsers, buffers, and rendering rules that let you show partial state without shipping garbage.

## Motivation

You want a form that fills in as the model generates: a title appearing first, then a summary, then a list of bullet points. If you wait for the whole JSON blob to complete before you show anything, streaming buys you nothing — the user still watches a spinner for two seconds. If you try to `json.loads` the buffer on every chunk, you crash on the first chunk that ends mid-string.

Between "wait for the end" and "parse on every chunk" there is a narrow, useful path: **incremental parsing**. Both providers' SDKs help. Where they don't, a small state machine of your own does the job.

## Two mental models: text-shaped vs tree-shaped

Not all partial output is the same shape, and the correct rendering strategy differs.

- **Text-shaped output** is a JSON string, a list of strings, or a JSON blob whose useful content is a long text field. You want to show characters as they arrive. Example: a `{"summary": "..."}` where the summary is 200 words.
- **Tree-shaped output** is a JSON object with distinct fields the user perceives independently — a title, a status, a set of tags, a nested breakdown. You want to render each field the moment it is *complete*, not on every keystroke.

Text-shaped is almost always safe to render incrementally. Tree-shaped is where the "legal to render" question actually bites.

## When is a partial parse legal?

Formally, a partial JSON string only becomes a valid JSON value at exactly one moment: when the last closing bracket arrives. Before that, the byte stream is not valid JSON. Any sensible partial parse has to be lenient — completing dangling strings, closing open arrays and objects, treating the last comma as if it never came.

Two safe operations on a partial buffer:

1. **Complete text-shaped tokens (whole strings) as soon as the string terminator arrives.** If the buffer ends `..."summary": "Hello, wo`, no complete string has arrived for `summary`; show nothing. If it ends `..."summary": "Hello, world"`, the string is complete; you can show `"Hello, world"`.
2. **Complete whole scalar values (`true`, `false`, `null`, numbers) as soon as they are followed by a delimiter (`,`, `}`, `]`, or whitespace).** A number is not complete just because you saw the first digit — `3` might become `3.14`.

Two dangerous operations:

1. **Rendering a string mid-content.** The model may emit a control-character escape (`\n`, `\uXXXX`) that spans several bytes. Slicing the buffer mid-escape gives you a broken glyph or, worse, HTML you didn't sanitise. Only render up to the last known-safe character.
2. **Treating a value as final because it "looks done."** `"status": "in_progr` is not `"status": "in_progress"`. The model may still finish the string in the next chunk. If your UI switches on `status`, do not switch until the string is closed.

The single sentence to memorise: **you can safely render any complete JSON node from a partial buffer, but you cannot safely render an incomplete one.** The rest of this chapter is about knowing which nodes are complete.

## The two provider mechanisms

Both providers give you tools that already do the buffering-and-partial-parsing work for the strict-schema case. Learn them before you write your own.

### Anthropic — streamed tool_use with `input_json_delta`

When you stream a request that produces a `tool_use` block, Anthropic sends `content_block_delta` events with `{"type": "input_json_delta", "partial_json": "..."}`. The SDK's stream helper concatenates these for you and offers a helper to parse the partial JSON incrementally.

```python
import anthropic

client = anthropic.Anthropic()

TOOL = {
    "name": "record_report",
    "description": "Record a structured status report.",
    "input_schema": {
        "type": "object",
        "required": ["title", "summary", "risk_level"],
        "properties": {
            "title": {"type": "string"},
            "summary": {"type": "string"},
            "risk_level": {"type": "string", "enum": ["low", "med", "high"]},
        },
    },
}

with client.messages.stream(
    model="claude-opus-4-7",
    max_tokens=1024,
    tools=[TOOL],
    tool_choice={"type": "tool", "name": "record_report"},
    messages=[{"role": "user", "content": "Report on today's ingest pipeline health."}],
) as stream:
    for event in stream:
        # The SDK also exposes `stream.input_json_stream` for partial-JSON events;
        # here we walk raw events to make the shape visible.
        if event.type == "content_block_delta" and event.delta.type == "input_json_delta":
            print(event.delta.partial_json, end="", flush=True)
    final = stream.get_final_message()
    tool_use = next(b for b in final.content if b.type == "tool_use")
    print("\nFinal parsed:", tool_use.input)
```

<!-- needs-research: confirm the exact SDK helper name Anthropic exposes for a partially-parsed input JSON stream (e.g. `stream.input_json_stream` vs `stream.current_input_json`) in the current Python SDK — check https://docs.anthropic.com/en/api/messages-streaming and the anthropic-sdk-python README. -->

Two things this snippet gets right and one to note:

- The final, complete tool_use `input` is always available on the reassembled `Message` after the stream closes — `stream.get_final_message()`.
- The raw `partial_json` chunks concatenate to a valid JSON object at the end. Do **not** try to parse them one at a time.
- The `tool_choice` forces the tool call. That is the same "schema-constrained output" pattern from mod-102 chapter 5, now streaming.

Reference: <https://docs.anthropic.com/en/docs/build-with-claude/tool-use/streaming-tool-use>.

### OpenAI — streamed Structured Outputs with `.stream()`

The OpenAI Python SDK's `.beta.chat.completions.stream(...)` helper (used with `response_format={"type": "json_schema", ...}` or with `parse=`-based typed parsing) exposes typed events that fire when a *field* becomes complete, not when an arbitrary byte arrives.

```python
from openai import OpenAI
from pydantic import BaseModel
from typing import Literal

client = OpenAI()

class Report(BaseModel):
    title: str
    summary: str
    risk_level: Literal["low", "med", "high"]

with client.beta.chat.completions.stream(
    model="gpt-4.1",
    messages=[{"role": "user", "content": "Report on today's ingest pipeline health."}],
    response_format=Report,
) as stream:
    for event in stream:
        if event.type == "content.delta":
            # incremental text of the JSON serialization — usually not what you want to render
            pass
        elif event.type == "content.done":
            report: Report = event.parsed
            print("Final parsed:", report)
```

<!-- needs-research: confirm the exact event types and helper name for streamed structured outputs in the current openai-python SDK (`.beta.chat.completions.stream` vs a stabilized `chat.completions.stream`, and the event names `content.delta` / `content.done`) as of 2026-08 — check https://github.com/openai/openai-python and https://platform.openai.com/docs/guides/structured-outputs. -->

Two useful patterns on top of this helper:

- **Snapshot the partially-parsed object between events.** The SDK maintains a partial parse (`event.snapshot`) that you can render field-by-field as they complete. Do not read fields whose completion event has not yet fired.
- **Prefer typed models over raw JSON schema.** Using a pydantic model as `response_format=` gives you a typed `parsed` object at the end and lets the SDK do the validation.

Reference: <https://platform.openai.com/docs/guides/structured-outputs>.

## Writing a small incremental parser (when you have to)

If you are streaming JSON that is neither a strict tool call nor a Structured Outputs response — say, the model is free-form generating a JSON blob inside a text field — you will need a small partial parser. The safest strategy is a **state machine over the raw byte stream** that tracks brace and bracket depth, whether it is currently inside a string, and whether the last brace/bracket has been closed.

Pseudocode:

```
depth = 0
in_string = False
escape_next = False
last_complete_at = 0
for i, ch in enumerate(buffer):
    if escape_next:
        escape_next = False; continue
    if in_string:
        if ch == "\\": escape_next = True
        elif ch == '"': in_string = False
        continue
    if ch == '"': in_string = True
    elif ch in "{[": depth += 1
    elif ch in "}]":
        depth -= 1
        if depth == 0:
            last_complete_at = i + 1   # buffer[:last_complete_at] is a valid JSON value
# Only call json.loads on buffer[:last_complete_at]. Everything after is in flight.
```

Two rules that keep this parser safe:

- **Never parse until `depth == 0`.** Objects and arrays are only complete when their brace count returns to zero at the top level.
- **Never render a string that is still open.** Track `in_string`; if the current byte position sits inside a string, the string's value is not final.

For most product code, reach for a maintained library instead of hand-rolling: `ijson` (pull-parser for streaming JSON) or one of the several partial-JSON parsers on PyPI. Only write the state machine yourself when you need to understand or debug what the library is doing.

- `ijson` — event-driven / streaming JSON parser. <https://pypi.org/project/ijson/>
- <!-- needs-research: cite one specific well-maintained partial-JSON parser on PyPI (e.g. `partial-json-parser`, `json-stream`) with its current homepage before merge. -->

## Recognising legal-vs-illegal partial parses in practice

A short catalogue of what typically goes wrong when developers try to render partial JSON eagerly:

- **Rendering a truncated enum.** The model writes `"status": "in_pr` and the UI switches to "in progress" because the string starts with `"in_"`. Then the model writes `"in_review"`, and the UI has already committed. **Fix:** only trust an enum after its string closes.
- **Rendering a truncated number.** `"score": 8` becomes `8.7` two chunks later. If your UI has already committed a rounded 8, it now shows the wrong value. **Fix:** wait for a delimiter (`,`, `}`, `]`, whitespace) before treating a number as complete.
- **Rendering an unescaped chunk of a string that contained a Unicode escape.** `"summary": "Bake at 350°"` split into `"Bake at 350\` and `u00b0"` — the first chunk ends mid-escape. Rendering it verbatim shows a backslash. **Fix:** only render up to the last completed escape sequence, or use the SDK helper that already handles this.
- **Rendering an array whose closing bracket has not arrived.** `"tags": ["a", "b"` — showing `["a", "b"]` is a guess. The model may continue with `, "c"]`. **Fix:** render individual completed elements (`"a"`, `"b"`) rather than "the array".
- **Treating markdown code fences as JSON boundaries.** The model wrote ` ```json {...} ``` ` inside a text response. Do not fish the `{...}` out mid-stream and try to parse it. Wait for the closing fence, then parse.

The single rule under all of these: **complete tokens, not partial strings.** If a token has finished (a whole string, a whole number, a whole array, a whole object), it is safe to render. Everything else is in flight.

## Rendering strategies that survive partial data

Three rendering patterns work reliably:

1. **Skeleton-then-fill.** Render the whole shape immediately with placeholders (spinner glyphs, greyed labels). As each field completes, swap in the real value. This is the strategy behind most well-behaved LLM UIs — the shape is stable; only the content updates.
2. **Append-only text stream.** For text-shaped output, forward characters as they arrive. Do not re-render or retract; do not colour parts of the text differently based on partial classification.
3. **Snapshot polling.** Every N ms (say, 100 ms), take the latest partial parse and diff it against the previous render. Only re-render fields that changed and whose completion event has fired. This costs a tiny amount of CPU and saves you from thrashing the DOM on every 5-byte chunk.

Which of the three fits depends on the shape of your data — but pick one deliberately. Mixing "render fields eagerly as strings partial" with "wait until the object is closed" is where the mis-render bugs come from.

## Streaming tool_use for tool execution

Streaming a tool call is a special case where the temptation to parse early is highest and the payoff is lowest. Two reasons:

- **The tool cannot execute until it has the whole arguments.** Calling `book_room(check_in="2026-07", check_out=` with half the arguments is not "starting early"; it is calling with wrong data. Wait for `content_block_stop` (Anthropic) or `finish_reason == "tool_calls"` (OpenAI), then parse, then execute.
- **You still want to stream *the display* of the tool call to the user.** "Looking up bookings for Paris…" is a useful UX signal. Just make sure the string you show is your own rendered summary, not a mid-stream JSON blob you have not confirmed is well-formed.

The pattern: display a friendly tool-call message when `content_block_start` fires for the tool_use block; execute the tool when the corresponding `content_block_stop` (or OpenAI-side finish reason) fires.

## Buffering budgets

Streaming lets you *start* early. It does not let you buffer *forever*. Two limits worth keeping in mind:

- **Memory.** A very long response streamed into an unbounded list eats memory linearly. If you are streaming into a queue for later processing, cap the queue and drop or block on full.
- **Latency.** If your consumer is slower than the model's token rate, back-pressure builds up in the network and the SDK's own buffers. On busy days this shows up as bursty rendering — long pauses followed by big chunks. Watch the delta between "chunk arrived at the SDK" and "chunk consumed by my code."

Chapter 3 shows the async version of these patterns; the same limits apply there, just with `asyncio.Queue(maxsize=...)` instead of a Python list.

## Common bugs to prepare for

- **`json.loads` on every chunk.** Guaranteed exceptions. Only parse at safe boundaries.
- **Partial parsers that "just work" until the string contains a `}`.** Braces inside strings are legal and common. Track `in_string`.
- **Assuming keys arrive in schema order.** With Structured Outputs (`strict: true` on OpenAI), field order is roughly deterministic — but do not rely on it. If your UI needs a specific field first, name it in the prompt or design the schema so that field is the sole content of the first content block.
- **Forgetting to wait for the string closer before switching UI state.** The classic enum-completion bug above.
- **Mixing display buffering with execution buffering.** The tool executes on the whole parsed input; the UI can show the friendly summary as soon as the tool call starts. Use two variables, not one.

## Summary

- Partial JSON only becomes valid at the last byte. Rendering "in-flight" values requires a stricter definition of "complete."
- The safe rule: render complete tokens (whole strings, whole numbers, whole arrays, whole objects) — never partial ones.
- Prefer the SDK helpers: Anthropic's stream events for `input_json_delta` and OpenAI's streamed Structured Outputs both do the buffering-and-partial-parsing work correctly.
- When you have to write your own parser, use a state machine that tracks brace/bracket depth and string state. Only call `json.loads` when depth returns to zero.
- Pick a rendering strategy (skeleton-then-fill, append-only, snapshot polling) deliberately. Mixing them is where mis-render bugs come from.
- Never execute a tool call on partial arguments. Stream the *display* of the tool call; wait for `content_block_stop` / `finish_reason == "tool_calls"` before executing.

The next chapter moves from one stream to many: `asyncio` and `httpx` for fanning out concurrent LLM calls, capping concurrency so you do not hammer the provider, and cancelling cleanly when the client disconnects.
