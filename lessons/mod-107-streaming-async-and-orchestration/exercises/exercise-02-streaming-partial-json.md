# Exercise 02 — Streaming partial JSON

Paired with [chapter 2 — streaming partial structured output safely](../02-streaming-partial-json.md).

**Estimated effort:** 90–120 minutes.

## Objective

Stream a schema-constrained structured response and render its fields to the terminal as each one *completes* — not on every chunk. Build (or use) an incremental parser that knows the difference between a legally-renderable value and an in-flight one, and prove your renderer never displays a truncated string, number, or enum.

## Problem statement

You are streaming a status report with the following shape:

```json
{
  "title": "string, 3-8 words",
  "risk_level": "low | med | high",
  "summary": "string, 2-4 sentences",
  "action_items": ["array of 3-5 short strings"]
}
```

Ask the model to produce a report and stream the response. Render to the terminal in the **skeleton-then-fill** style from chapter 2:

```
title:         [waiting …]
risk_level:    [waiting …]
summary:       [waiting …]
action_items:  [waiting …]
```

As each field completes, replace `[waiting …]` with the real value. Never render a value that is not final. Never render a `risk_level` string until its closing quote has arrived.

## Requirements

1. Force schema-constrained output. Choose one:
   - Anthropic: define the shape as a tool with `input_schema`, and stream with `tool_choice={"type": "tool", "name": ...}` (chapter 2 has the pattern).
   - OpenAI: use `response_format` with a `json_schema` (or the `.beta.chat.completions.stream` helper with a pydantic model).
2. Consume the stream. Build up the partial JSON in a buffer as `input_json_delta` (Anthropic) or content deltas (OpenAI) arrive.
3. Detect when each field is complete and only render at those moments. Two acceptable approaches:
   - Use the SDK's typed partial-parse helper if it exposes one. On OpenAI, `event.snapshot` between events gives you the current best-effort partial parse.
   - Roll your own incremental parser (state machine tracking brace depth, bracket depth, and whether the cursor is inside a string). Chapter 2 has pseudocode.
4. Render with skeleton-then-fill: print the four labelled placeholders once, then use terminal cursor moves (`\r`, `\x1b[F`, or the `rich` library's live-update) to replace each `[waiting …]` when its field completes.
5. Guard against the four mis-render bugs from chapter 2:
   - Never render a truncated `risk_level` string as if its prefix were the final value.
   - Never render a truncated number.
   - Never render an array whose closing bracket has not arrived (individual completed elements are fine).
   - Never render a mid-escape-sequence chunk (`"\` followed later by `"u00b0"`).
6. Confirm the final rendered output equals the model's final `parsed` object. Print `"OK: final render matches parsed"` at exit if it does; otherwise print the diff and exit non-zero.

## Requirements — the adversarial test

After the happy path works, prove your parser cannot be tricked. Add a `--fake-stream FILE` mode that replays a hand-crafted sequence of `partial_json` chunks from a file — bypassing the network — and use it to run these three test cases:

1. **Chunk split mid-string.** The chunks are `{"title": "Deploy `, then `pipeline update"}, ...`. Your renderer must not print `Deploy` as the title. It must wait for the closing `"` before rendering `Deploy pipeline update`.
2. **Chunk split mid-enum.** The chunks are `..., "risk_level": "l`, then `ow", ...`. Your renderer must not display `l` as the risk level. It must wait until the closing `"` shows `low`.
3. **Chunk split mid-escape.** The chunks contain `..., "summary": "Bake at 350\`, then `u00b0F ...`. Your renderer must not display a bare backslash. Either render up to the last known-safe boundary (before the backslash) or wait for the full escape.

If any of the three tests renders early or renders garbage, that is a bug to fix.

## Starter guidance

- Anthropic streamed tool use: <https://docs.anthropic.com/en/docs/build-with-claude/tool-use/streaming-tool-use>
- OpenAI Structured Outputs (streaming section): <https://platform.openai.com/docs/guides/structured-outputs>
- `ijson` (streaming JSON parser): <https://pypi.org/project/ijson/>
- `rich` (nice terminal live updates): <https://rich.readthedocs.io/en/stable/live.html>

For the adversarial test, do not shell out to the provider — split the fake JSON payload into chunks of varying size (1 byte, 3 bytes, mid-escape) and feed them through the same parser. That is the fastest way to prove your parser is correct without spending API credit.

## Acceptance criteria

- On a real streaming request, your terminal shows four `[waiting …]` placeholders first, and each is replaced with its final value only after that field is complete.
- At no point during a run does a partial string, partial number, or partial enum appear on the screen.
- The `--fake-stream` adversarial tests all pass on your parser: the renderer waits for the closing quote in every case.
- The exit line `OK: final render matches parsed` prints on every successful run. If your incremental parser and the SDK's final parse ever disagree, that is a bug in the parser.
- The whole render completes visibly *earlier* than the whole response — some fields (usually `title` and `risk_level`) appear well before the last chunk arrives. If every field only appears at the very end, your parser is being too conservative and you are giving up the streaming benefit; loosen it.

## Stretch goals

- Add a fifth field `evidence: list[{"source": str, "quote": str}]` — an array of objects. Render each element as it completes, not the whole array at once. This is the shape that catches parsers that treat "array" as an atomic unit.
- Replace your incremental parser with `ijson` (an event-driven streaming JSON parser). Compare wall-clock overhead and correctness.
- Report per-field TTFT — the wall-clock time from request start to the moment each field finished. Which field usually completes first? Which last? What does that tell you about how the model orders schema output?
- Run the same test against both providers with the same schema. Note any differences in how eagerly each provider streams individual fields.
