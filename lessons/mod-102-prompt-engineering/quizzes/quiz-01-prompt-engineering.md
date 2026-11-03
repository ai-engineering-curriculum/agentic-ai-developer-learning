# Quiz 01 — Prompt Engineering & Structured Output

**Covers:** chapters 1–7 of `mod-102-prompt-engineering`.
**Format:** 12 multiple-choice questions. Pick the single best answer for each. Answer key with per-answer explanations is at the bottom.

Treat this as a self-check: open the chapters when you want to look something up, but try to answer first.

---

## 1. In this module, a "prompt" is treated as every input that conditions a model call. Which of the following is *not* on that list?

- A. The system instructions and the user message.
- B. Any assistant prefill forcing the opening of the reply.
- C. The sampling parameters (`temperature`, `top_p`, `max_tokens`, `stop_sequences`).
- D. The user's local operating system version and installed SDK version.

## 2. You tweaked the system prompt, swapped a few-shot example, and lowered `temperature` all in one commit. The chapter calls this out as a specific anti-pattern because…

- A. The provider will reject bundled changes until they are split.
- B. Prompt caching stops working whenever more than one block changes between calls.
- C. If quality moves, you cannot attribute the result to any single change — change one thing at a time.
- D. Few-shot examples are only valid at `temperature=0`, so the three changes contradict each other.

## 3. According to chapter 2's "where each block goes," which placement maps to the recommended skeleton?

- A. Role, output format, and hard rules in the **system** block; per-call input (and any retrieval results) in the **user** block.
- B. Everything in the user block so the system block can stay empty and cheap.
- C. Role in the system block; constraints and output format in the user block, below the per-call input.
- D. Constraints and format in a separate `assistant` message that precedes the user message.

## 4. The chapter says vague specificity ("be concise", "be professional") is "barely better than nothing." Which rewrite replaces it with specificity that is actually checkable?

- A. "Be clear and helpful, in a professional tone."
- B. "Summarize this in five bullet points of fewer than 15 words each, written for a non-technical reader."
- C. "Please be as accurate as possible, and avoid being verbose."
- D. "Do your best to produce a high-quality summary suitable for the audience."

## 5. For a narrow classification task, which situation earns the tokens that few-shot examples cost?

- A. The model already handles the task perfectly zero-shot.
- B. The task is open-ended idea generation and you want more variety.
- C. The output shape is fiddly, the model keeps drifting from a specific format, or borderline cases need demonstration rather than description.
- D. The input space is wide and unstructured and your handful of examples would not be representative.

## 6. Few-shot ordering: where should you place the trickiest, edge-case example?

- A. First, so the model weights the hardest case most.
- B. Last, because it is the freshest "instruction" the model sees before the real input.
- C. In the middle, so neither position effect dominates.
- D. Randomize ordering per call so no position effect can take hold.

## 7. Chain-of-thought (CoT) and decomposition are related but not the same. Which distinction matches the chapter?

- A. CoT splits work across multiple calls; decomposition keeps everything in one call with a scratchpad.
- B. CoT keeps the work in a single call and asks the model to reason before answering; decomposition splits the task across multiple calls (or LLM-and-code hybrids).
- C. They are synonyms; the chapter uses the two terms interchangeably.
- D. CoT applies only to math problems; decomposition applies only to classification.

## 8. The chapter warns against one combination in particular when you need strict structured output. Which combination, and why?

- A. Asking for reasoning *and* strict JSON in the same call — reasoning tends to leak out as prose around the JSON and the result fails to parse.
- B. Using `temperature=0` with few-shot examples — determinism cancels the demonstrations.
- C. Decomposing a task into two calls — the second call always re-pays for context, which is forbidden.
- D. Using JSON mode and a system prompt in the same request — providers reject the combination.

## 9. OpenAI's `response_format={"type": "json_object"}` (vendor JSON mode) gives you which guarantee?

- A. The response is valid JSON *and* conforms to any schema you describe in the system prompt.
- B. The response is valid JSON, but the shape can still be wrong (e.g., `{"answer": "billing"}` instead of `{"category": "billing"}`).
- C. The response is constrained to a strict JSON Schema you supply as part of the flag.
- D. The response is streamed as newline-delimited JSON objects, one per field.

## 10. You want a *strict* JSON object out of Anthropic's API, and the chapter calls one approach "currently the most reliable way." Which is it?

- A. Ask for JSON in the prompt and lower `temperature` to 0 until the output stops drifting.
- B. Pass `response_format={"type": "json_schema", ...}` with `strict: True`.
- C. Define a tool whose `input_schema` is your target shape and set `tool_choice={"type": "tool", "name": "..."}` to force the model to call it; the tool's `input` is your structured object.
- D. Use `stop_sequences=["}"]` so the model cannot continue past the closing brace.

## 11. Even after schema-constrained generation or tool-calling, chapter 6 says you still need to validate at the boundary. Which failure mode does validation at the boundary catch that structured-output alone does not?

- A. Network timeouts between your client and the provider.
- B. Non-UTF-8 bytes in the raw HTTP body before parsing.
- C. "Valid-but-wrong" values inside a correct shape — e.g., a `Classification` object whose `category` field is a legal enum value but the semantically wrong one, or a `max_tokens` truncation producing a partial object.
- D. Billing anomalies in the `usage` block.

## 12. Chapter 7 describes a "golden set" that gates prompt changes. Which property is required?

- A. The golden set is regenerated from scratch for each prompt change so stale cases do not drag the score down.
- B. Once a case is in the golden set, its expected output is **frozen** — you do not edit it to make a failing run pass; you only revise expectations when the *requirements* change.
- C. The golden set must be at least 1,000 cases and sampled uniformly at random from production.
- D. Expected outputs should be filled in by an LLM-as-judge so the golden set reflects current model behavior.

---

## Answer key

1. **D.** Chapter 1 ("What a prompt actually controls") lists system instructions, the user message, assistant prefill, few-shot turns, tool/function schemas, and sampling parameters. The OS and SDK version are not conditioning inputs to the model.
2. **C.** Chapter 1 ("The engineering loop") says "Change one thing at a time. If you change the system prompt, the few-shot examples, and `temperature` in the same commit, you cannot attribute the result."
3. **A.** Chapter 2 ("Where each block goes") puts role, persona, output format, and hard rules in the system block (stable, cache-friendly) and per-call input, including retrieval results, in the user block.
4. **B.** Chapter 2 ("Be specific; do not assume tone") contrasts vague adjectives with concrete, numeric, named, checkable constraints — a number of bullets, a word count, a named audience.
5. **C.** Chapter 3 ("What examples are good at") names three triggers for few-shot: fiddly output shape, borderline edge cases hard to describe in prose, and persistent format drift. Open-ended generation (B) and unrepresentative example sets (D) are listed as weak cases; perfect zero-shot (A) is spent tokens.
6. **B.** Chapter 3 ("Ordering matters more than people expect") recommends putting the strongest, most representative example first and a tricky/edge-case example last — the one closest to the input is the freshest "instruction."
7. **B.** Chapter 4 ("Two related ideas") pins the distinction exactly: CoT is a prompting move inside one call; decomposition is an application-architecture move across multiple calls.
8. **A.** Chapter 4 ("Chain-of-thought, briefly") and its "Common mistakes" both call this out — "think then JSON" prompts routinely produce prose-then-JSON that fails to parse. Fix by separating steps, using a structured-output mode, or putting reasoning in a discardable field.
9. **B.** Chapter 5 ("Strategy 2: vendor JSON mode") says JSON mode buys you valid JSON but not shape correctness: the model can return `{"answer": "billing"}` when you wanted `{"category": "billing"}`. Shape correctness requires schema-constrained generation or tool calling.
10. **C.** Chapter 5 ("Strategy 4: tool / function calling for structured output") describes forcing a tool call via `tool_choice` as "currently the most reliable way to get a strict JSON object out" on Anthropic — the tool's `input` is your structured object.
11. **C.** Chapter 6 ("What to do about the model lying inside a valid shape" and "Step 1: parse defensively") makes both points: schema validation tells you the shape is right, not that the values are right, and `max_tokens` truncation can still produce a partial object you must detect. Options A/B/D are either lower-level transport issues (A/B) or orthogonal to validation (D).
12. **B.** Chapter 7 ("Build a golden set, deliberately") lists "frozen" as a required property: once a case is in the golden set, you do not edit its expected output unless requirements changed. Editing expectations to match a regression is called a "placebo" in the same chapter.
