# Quiz 01 — LLM Fundamentals for Application Developers

**Covers:** chapters 1–8 of `mod-101-llm-fundamentals`.
**Format:** 12 multiple-choice questions. Pick the single best answer for each. Answer key with per-answer explanations is at the bottom.

Treat this as a self-check: open the chapters when you want to look something up, but try to answer first.

---

## 1. The API is stateless. Which of the following is a direct consequence?

- A. The provider stores your messages so you can resume a conversation later from any client.
- B. The server remembers the last `temperature` you used so you can omit it on follow-up calls.
- C. If you want multi-turn behavior, your client must hold the conversation history and resend it on every call.
- D. The SDK caches the model's reply locally so repeated identical calls are free.

## 2. You are calling Anthropic's Messages API. Which placement of the system prompt is correct?

- A. Pass it as the first entry in `messages` with `role: "system"`.
- B. Pass it as a top-level `system` parameter on the request; `messages` starts with a `user` turn.
- C. Prepend it to the first `user` message's content, separated by a newline.
- D. It must be sent in a separate HTTP request before the first `messages.create` call.

## 3. You want to force a model to begin its reply with a `{` so the output parses as JSON. The chapter calls out which technique as "the cheapest, most reliable way to force structured output"?

- A. Set `response_format = {"type": "json_object"}` on every provider that supports it.
- B. Lower `temperature` to 0 and hope the model picks `{` as the top token.
- C. Append a final `assistant` message with partial content (e.g., `"{"`) so the model continues from there — a prefill.
- D. Add a `stop_sequence` of `}` so the model cannot run past the closing brace.

## 4. A 200K-token context window with `max_tokens=4096` and a 190K-token input will most likely result in which behavior?

- A. The server silently drops the oldest messages to make room for the reply.
- B. The call succeeds and the model has roughly 10K tokens of headroom to spend on the reply.
- C. The provider enlarges the context window automatically for a surcharge.
- D. The input is zero-padded on the right so prefill never spans more than 128K tokens.

## 5. You wrote `temperature=1.5` in a helper that targets both Anthropic and OpenAI. What does the chapter say will happen?

- A. Both providers accept it and interpret it identically.
- B. Both providers clamp it to their maximum before sampling.
- C. OpenAI accepts it ("high"); Anthropic rejects it because its legal range is `[0.0, 1.0]`.
- D. Anthropic treats it as high creativity; OpenAI rejects it because its max is `1.0`.

## 6. You set `temperature=0` and send the same prompt twice. The outputs differ in a few tokens. The chapter says this is because…

- A. The server has a hidden `top_p=0.9` default that overrides your temperature.
- B. GPU floating-point non-associativity, batching, hardware routing, and silent server-side updates can all shift the top token even at `temperature=0`.
- C. `temperature=0` is reserved for streaming only; non-streaming requests ignore it.
- D. The provider mixes multiple models under one ID and A/B tests responses randomly.

## 7. You are picking a model for a new chat feature. The chapter recommends which sequence?

- A. Start with the flagship tier for safety; only drop to a cheaper tier if the bill becomes a problem.
- B. Start with the workhorse tier on a fixed test set; if it passes, also try the cheap tier; only reach for flagship if workhorse fails.
- C. Always use the cheapest model and iterate on the prompt until quality is acceptable.
- D. Use a reasoning/thinking model by default — the quality gain is worth the latency and cost.

## 8. Streaming over SSE is on. Compared to a non-streaming call for the same prompt and model, you should expect…

- A. Lower time-to-first-token and similar total wall time.
- B. Lower total wall time; TTFT is unchanged.
- C. Lower cost per request because partial deltas are not billed.
- D. Higher total wall time but lower per-token latency.

## 9. Your 200,000-call/day feature uses a 4,000-token system prompt that is identical on every call. Which lever will cut the most cost with the least change to behavior?

- A. Drop to a smaller model tier even though your evals are tight on the current one.
- B. Lower `max_tokens` so replies get truncated.
- C. Enable prompt caching on the stable prefix so repeat input tokens are discounted (typically ~90%) after the first write.
- D. Switch every call to the batch API regardless of latency requirements.

## 10. Your call fails with HTTP `400 Bad Request`. According to the chapter, the correct behavior is…

- A. Retry with exponential backoff — `400`s are usually transient.
- B. Honor the `Retry-After` header; `400` is rate-limiting.
- C. Do not retry — a `400` means your request is malformed; fix the request.
- D. Retry once immediately; if it fails again, escalate to a different model.

## 11. The chapter prescribes "exponential backoff with full jitter" for retrying `429` and `5xx`. Which snippet matches that policy?

- A. `sleep = base * (2 ** attempt)` — deterministic doubling, no randomness.
- B. `sleep = random.uniform(0, min(cap, base * (2 ** attempt)))` — a uniform draw inside the growing window, overridden by `Retry-After` when the server sets one.
- C. `sleep = base + random.random()` — small random bump added to a constant.
- D. `sleep = random.uniform(base, cap)` — one fixed-range random wait regardless of attempt number.

## 12. You get a response back with no exception. The content is non-empty but ends mid-sentence. Which `stop_reason` is most consistent with what you observed, and what should you do?

- A. `end_turn` — the model finished cleanly; the sentence is intentionally short.
- B. `tool_use` — the model wants a tool to be invoked; parse the tool call.
- C. `max_tokens` — the output budget was exhausted; raise `max_tokens` or ask for a shorter answer.
- D. `stop_sequence` — your stop string matched inside a word; remove the stop sequence.

---

## Answer key

1. **C.** Chapter 1 ("What 'stateless' means and why you will get burned") says the server has no memory of your last call — if you want a conversation, your client resends the history every request.
2. **B.** Chapter 2 ("Vendor placement of the system prompt") contrasts the two patterns: Anthropic uses a top-level `system` parameter and `messages` starts with `user`; OpenAI puts a `system` message first inside `messages`.
3. **C.** Chapter 2's "Prefill" section calls prefill "the cheapest, most reliable way to force structured output" — append an `assistant` message with partial text (e.g., `"{"`) and the model continues from there.
4. **B.** Chapter 3 says the context window is shared between input and output. 200,000 − 190,000 = 10,000 tokens of headroom, which is more than the `max_tokens=4,096` you reserved, so the call fits. If the input had been larger, the reply would have been truncated with `stop_reason: max_tokens`.
5. **C.** Chapter 4 ("Temperature") pins Anthropic's legal range at `[0.0, 1.0]` and OpenAI's at `[0.0, 2.0]`, and warns that `temperature=1.5` is invalid for Anthropic and merely "high" for OpenAI.
6. **B.** Chapter 4 ("'Deterministic' sampling") lists GPU floating-point non-associativity, batching changes, heterogeneous hardware routing, and silent server-side updates as reasons `temperature=0` is best-effort, not guaranteed-reproducible.
7. **B.** Chapter 5 ("A practical sequence") recommends: start with workhorse on a fixed test set; if it passes, try the cheap tier; only reach for flagship when workhorse fails. "Most teams skip the 'try cheaper' step and overspend by 5–10×."
8. **A.** Chapter 6 ("Why this matters") makes the point explicitly: streaming does not lower total latency; it lowers *perceived* latency by delivering the first token in a few hundred milliseconds.
9. **C.** Chapter 7 ("Levers to lower cost") puts "cache stable prefixes" first in the payoff order; the worked example in the same chapter calls out system-prompt caching as the primary source of "I cut my bill in half" wins. The chapter notes typical cached-token discounts around 90%.
10. **C.** Chapter 8's taxonomy marks `400` as "your fault, not theirs" — do not retry; fix the request. Only transient classes (`429`, `5xx`, Anthropic's `529`, timeouts, connection errors) are retryable.
11. **B.** Chapter 8 shows the `retry_with_backoff` helper using `sleep_s = random.uniform(0, min(cap, base * (2 ** attempt)))` ("full jitter") and emphasizes that a server-provided `Retry-After` overrides your computed value.
12. **C.** Chapter 4 ("stop_sequences and max_tokens") says that a `max_tokens` stop reason means the reply is truncated and tells you to raise `max_tokens` or ask for a shorter answer. Chapter 8's "Things that look like errors but are not" reinforces that `max_tokens` is not an exception — you must inspect the field.
