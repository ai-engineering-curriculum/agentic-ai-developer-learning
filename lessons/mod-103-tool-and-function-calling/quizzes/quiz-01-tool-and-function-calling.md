# Quiz 01 — Tool & Function Calling

**Covers:** chapters 1–6 of `mod-103-tool-and-function-calling`.
**Format:** 12 multiple-choice questions. Pick the single best answer for each. Answer key with per-answer explanations is at the bottom.

Treat this as a self-check: open the chapters when you want to look something up, but try to answer first.

---

## 1. A "tool call" in this module is defined as…

- A. The model directly executing a Python function it was given access to.
- B. A structured request the model emits instead of a final user-facing reply, which your application decides whether to honor by running the corresponding function.
- C. A background HTTP request the provider makes on the model's behalf before returning text.
- D. A hidden system message the provider injects when the user mentions a tool-like action.

## 2. Every tool definition has three required pieces. Which list matches the chapter?

- A. `name`, `description`, and a JSON Schema for the arguments.
- B. `name`, a Python callable, and a list of allowed stop sequences.
- C. `name`, `description`, and a natural-language example of a correct invocation.
- D. `name`, a return-value schema, and an authorization policy.

## 3. You have a `status` field that should only ever be `pending`, `shipped`, `delivered`, or `cancelled`. Which schema fragment follows chapter 2's design guidance?

- A. `{"type": "string"}` — let the model use any string so new statuses are not blocked.
- B. `{"type": "string", "description": "The order status (pending, shipped, delivered, or cancelled)."}` — put the allowed values in prose.
- C. `{"type": "string", "enum": ["pending", "shipped", "delivered", "cancelled"]}` — close the set with `enum`.
- D. `{"type": "string", "pattern": "^[a-z]+$"}` — constrain the shape only.

## 4. In Anthropic's protocol, after the model emits a `tool_use` block, your follow-up turn carrying the `tool_result` blocks must have which role?

- A. `assistant`, because the assistant is producing the result.
- B. `user`, because from the model's perspective your application is the "user" that supplies tool outputs.
- C. `tool`, matching OpenAI.
- D. `system`, so the result is treated as high-priority context.

## 5. You call the model once with `tools=[...]`, get a `tool_use` block, run the function, and call the model again. On the second call you drop the `tools=[...]` parameter to save tokens. What does the chapter say happens?

- A. Nothing — the provider remembers the tools you advertised on the first call for the duration of the conversation.
- B. The provider rejects the request because `tools` is now required for the whole conversation.
- C. Tools are part of the prompt, not a one-time registration; drop them and the model forgets they exist.
- D. The tools are retained but at a reduced priority, so the model will only reuse the previously called tool.

## 6. On OpenAI, a `tool_calls[*].function.arguments` value arrives as…

- A. A parsed Python object, just like Anthropic's `input` field.
- B. A JSON-encoded **string** — you must `json.loads(...)` it before dispatching.
- C. A base64-encoded JSON payload.
- D. A list of key-value pairs that the SDK merges into kwargs automatically.

## 7. Your `get_order_status` function is asked for an order ID that does not exist. Chapter 4 says the right thing to do is…

- A. Raise `OrderNotFound`; the dispatcher catches it, logs it, and sends a 404-style error back to the model.
- B. Return `None`; the model will treat that as "no answer available."
- C. Return a plain data payload such as `{"status": "not_found", "order_id": order_id}`. Not-found is an outcome, not a bug.
- D. Return a plausible placeholder record so the model can still produce a confident answer for the user.

## 8. Even with `additionalProperties: false` in the schema, you still want to validate `block.input` with a Pydantic model at the dispatcher. Why?

- A. Because Anthropic's `additionalProperties` flag is advisory and the provider does not actually enforce it.
- B. Because `fn(**args)` happily accepts any keyword arguments and skipping validation hides type bugs.
- C. Defense in depth: boundary validation catches extra keys, wrong types, and missing required fields, and converts them into structured feedback the model can act on instead of a raw `TypeError`.
- D. Pydantic rewrites the JSON Schema on the fly to match the Python signature.

## 9. The user asks "Do you do refunds in general?" and your app has a `create_refund` tool advertised. The chapter's decision rubric says the right move is…

- A. Call `create_refund` with placeholder arguments so the model can "see" the tool shape before answering.
- B. Call `get_order_status` first to establish context, then answer.
- C. Plain generation — the question is about policy/general knowledge, not fresh data or a side effect.
- D. Force a tool call with `tool_choice={"type": "any"}` so the model doesn't skip the capability.

## 10. The model is not calling a tool you expected it to call. You consider flipping `tool_choice` from `auto` to `any`. Chapter 5 flags this as a specific anti-pattern because…

- A. `tool_choice="any"` is deprecated; you must use `required` instead.
- B. Sometimes the right answer *is* plain generation; forcing a tool when the model wasn't going to call one produces nonsense arguments. Tighten the description first.
- C. `tool_choice="any"` disables schema validation on the model side.
- D. Providers bill `any` at a higher rate than `auto`.

## 11. Chapter 6 lists four failure buckets — malformed arguments, expected business outcomes, transient failures, internal bugs. Which claim about handling them matches the chapter?

- A. Treat them identically: return `{"error": "failed"}` from the dispatcher in every case so the model has a uniform signal.
- B. Only bugs should be surfaced to the model; business outcomes and transient failures should be silently retried from the dispatcher.
- C. Each gets a different response: validate args at the boundary, encode expected outcomes as data, convert exceptions into structured `{"error": ...}` payloads with an `is_error` flag, and cap retries with a budget.
- D. The protocol does not distinguish categories; the only safe policy is to raise out of the loop on any failure.

## 12. Which pair of error payloads matches chapter 6's "good vs. bad" error-feedback guidance?

- A. Good: `{"error": "failed"}`. Bad: `{"error": "invalid_arguments", "detail": [{"field": "amount_cents", "msg": "must be >= 1"}]}` — the second one leaks internals.
- B. Good: `{"error": "invalid_arguments", "detail": [{"field": "amount_cents", "msg": "must be greater than or equal to 1"}]}`. Bad: `{"error": "bad input"}` — opaque, gives the model nothing to fix.
- C. Good: a full Python traceback as a string. Bad: a structured JSON payload — too rigid for the model to read.
- D. Good: silently return a plausible default so the conversation keeps moving. Bad: any payload that names the failing field.

---

## Answer key

1. **B.** Chapter 1 ("What 'tool calling' actually is") defines a tool call as a structured request the model emits *instead of* a final reply, with the application deciding whether and how to honor it. The model "never executes anything itself."
2. **A.** Chapter 2 ("The three required pieces") names the trio: a `name`, a `description`, and an input JSON Schema. Everything else (examples, policies, callables) lives elsewhere.
3. **C.** Chapter 2 ("Designing the input schema") explicitly contrasts the two: `{"type": "string"}` will be called with `"PENDING"`, `"pending"`, `"in-transit"`, `"on its way"`; an `enum` closes the set.
4. **B.** Chapter 3 ("The protocol, in one diagram" / "Tool results go in a *user* turn") spells this out: "counterintuitive but how the protocol is defined: from the model's perspective, your application is the 'user' that supplies tool outputs."
5. **C.** Chapter 3 ("Walking through one round trip") lists "Pass `tools=` on every turn" as a must-do detail: "The tools are part of the prompt, not a one-time registration. Drop them and the model forgets they exist."
6. **B.** Chapter 3 ("The same loop on OpenAI") flags this exact difference: "OpenAI sends `arguments` as a **JSON-encoded string**; Anthropic sends `input` as a **parsed object**. Always parse before dispatching on OpenAI."
7. **C.** Chapter 4 ("What the function should return" and "Encode failure as data, not exceptions") makes the rule explicit: "a not-found order should return `{\"status\": \"not_found\"}`, not raise. Exceptions are for *your* bugs; outcomes are for the model."
8. **C.** Chapter 4 ("Argument handling") and chapter 6 ("Step 1: validate arguments at the boundary") both argue for validation at the dispatch layer as defense in depth: parse, validate, then dispatch — and convert failures into structured feedback rather than letting `TypeError` or `ValidationError` bubble out.
9. **C.** Chapter 5 ("When plain generation is the right move") names this exact kind of question — "the question is about the tools themselves" / policy explanations — as a case where the model should explain and optionally offer to call, not call with a placeholder.
10. **B.** Chapter 5 ("Common mistakes") and the `tool_choice` section call this out: "`tool_choice` is not a substitute for good descriptions." Forcing `any` when plain generation was correct produces nonsense arguments; the fix is a sharper description.
11. **C.** Chapter 6 ("The four failure modes" and the following steps) prescribe exactly this split: validate at the dispatcher, return business outcomes as data, convert transient and internal exceptions into structured payloads with `is_error`, and cap retries with a budget. Conflating the buckets (A) breaks recovery.
12. **B.** Chapter 6 ("What good error feedback looks like (and what doesn't)") uses exactly these two payloads as the good/bad pair: named field + specific message beats opaque `"bad input"`. A silent default (D) is the pattern the chapter explicitly warns against.
