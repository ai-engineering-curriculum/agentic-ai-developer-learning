# Quiz 01 — Model Selection, Cost, and Prompt Caching

**Covers:** chapters 1–5 of `mod-108-model-selection-cost-and-prompt-caching`.
**Format:** 12 multiple-choice questions. Pick the single best answer for each. Answer key with per-answer explanations is at the bottom.

Treat this as a self-check: open the chapters when you want to look something up, but try to answer first.

---

## 1. Chapter 1 says model choice is "a decision, not a default." Before you can defend the choice, you need three numbers per candidate on a fixed test set. Which triple does the chapter name?

- A. Benchmark leaderboard score, parameter count, and context-window size.
- B. Accuracy on your task-specific test set, cost per call computed from the `usage` block, and p50/p95 latency measured from a distribution.
- C. Peak tokens-per-second, number of trainable parameters, and provider uptime.
- D. Marketing-page quality rating, sticker-price per million tokens, and a single-sample latency reading.

## 2. The module's objective says "compare at least two providers." Chapter 1 gives two reasons for that constraint. Which pair?

- A. Pricing arbitrage (one provider is always cheaper) and so you can run A/B tests on real production traffic.
- B. Providers do not agree on which tasks their small-tier models handle well, and you need a known-good fallback for the day the primary provider has an incident (chapter 5).
- C. Regulatory compliance in different regions, and to avoid vendor-lock-in lawsuits.
- D. One provider for input, the other for output — their tokenizers are complementary.

## 3. Chapter 2 points out that output tokens are typically **3–5× more expensive** than input tokens on the same model. Which prompt-design implication does the chapter draw from that asymmetry?

- A. Always ask the model to produce the shortest possible output — input length does not matter for cost.
- B. Average `P_in` and `P_out` together to get a single per-token rate and reason about it once.
- C. A prompt that adds 500 input tokens to trim 100 output tokens is often a good trade; the reverse (500 more output tokens to save 100 input tokens) usually is not.
- D. Use `max_tokens=4096` by default, because output token cost is bounded anyway.

## 4. Chapter 2 tells you to **reserve the worst-case** on a per-user rolling-window budget *before* the call, then reconcile after the response. Why not just charge the counter with the actual output after the fact?

- A. Because the `usage` block is only populated on streaming responses, so you cannot read it before the response closes.
- B. Because two concurrent requests from the same user could both pass a stale under-the-cap check and together blow the cap — you must deduct the worst case at enforcement time so the counter reflects outstanding reservations, not just completed usage.
- C. Because the SDK rounds output tokens up; reserving worst-case is how you compensate.
- D. The chapter recommends the opposite — charge the actual output and never reserve worst-case.

## 5. One of chapter 2's "common bugs" is enforcing a per-request budget on `len(prompt) / 4` rather than a true token count. What exactly goes wrong?

- A. `len(prompt) / 4` is a rough character-to-token heuristic; it under- or over-counts depending on the language and the content (code, emoji, non-ASCII), so a cap set against it can either let a true oversize through or refuse a legitimate request. Count with the provider's tokenizer or token-counting endpoint.
- B. Python's `len` is O(n) on strings; for long prompts, enforcement adds unacceptable latency.
- C. The provider returns an error if the token-count estimate is more than 1% off, so `len / 4` triggers it.
- D. Nothing goes wrong — `len(prompt) / 4` is explicitly endorsed as the enforcement method for small deployments.

## 6. Chapter 3's single most important structural rule for prompt caching is "stable at the top, volatile at the bottom." Why does the order matter?

- A. Longer prompts are more token-efficient when the stable part is at the end.
- B. Providers process prompts bottom-up, so volatile content at the top wastes the model's attention.
- C. Caching is **prefix-based** — the cache matches from the start of the message list forward. If the first token differs from a cached entry there is no hit, so volatile content (user input, timestamps, per-request IDs) must sit at the end of the prompt and stable content (system prompt, few-shot, retrieved context) must sit at the beginning.
- D. Anthropic requires it; OpenAI does not care about order.

## 7. Chapter 3 lists an antipattern: starting the system prompt with `"You are Claude, and today is {current_date}"`. What specifically goes wrong?

- A. Nothing — it is a recommended way to ground the model in the current date.
- B. The model refuses to answer questions about dates when prompted this way.
- C. Every day's requests produce a different prefix, so no request cache-hits a cache entry written on another day — the token-weighted hit ratio collapses.
- D. The provider bills you an extra "dynamic content" surcharge for interpolated system prompts.

## 8. Chapter 3 is emphatic that you measure cache-hit ratio **token-weighted**, not request-weighted. Why?

- A. Request-weighted is undefined — Anthropic reports cache data only in tokens.
- B. Token-weighted directly maps to the invoice: "80% of input tokens served from cache" is the fraction of input-token cost moved to the cheaper line item. "80% of calls hit cache" could still mean most of the token volume was uncached if the hits were on short prompts and the misses on long ones.
- C. Request-weighted hit ratios are prohibited by the Anthropic terms of service.
- D. The two are numerically identical, so the distinction is a style preference.

## 9. Chapter 4's cascade routing formula is `avg_cost = C_cheap + p * C_frontier`, where `p` is the escalation rate. Which statement about the economics is true?

- A. A cascade is always cheaper than always-frontier, regardless of `p` or the price ratio.
- B. If `p` approaches 1, cascade cost per call approaches `C_cheap + C_frontier` — strictly *worse* than always-frontier, because you pay for both calls on every escalated request. Cascade is only a win when the escalation rate stays well below 1 and the confidence check is honest.
- C. If `p = 0.5` and `C_frontier = 10 * C_cheap`, the cascade is cheaper than always-small.
- D. The small-model call is free during the escalation branch, so cascade is never worse than always-frontier.

## 10. Chapter 4 recommends batch APIs for a specific class of workload. Which description matches the chapter?

- A. Any workload where you want streaming TTFT under 500 ms.
- B. Real-time chat replies where a user is watching the cursor blink.
- C. Non-user-facing workloads — overnight enrichment jobs, corpus classification, LLM-as-judge eval runs — where the latency budget is measured in hours. Both major providers offer roughly ~50% off the interactive per-token price in exchange for an up-to-24-hour turnaround.
- D. Any workload under 10 requests — batch overhead is lowest for small jobs.

## 11. Chapter 5 says provider status pages are a **lagging** indicator. The implication the chapter draws is…

- A. You should disable your own monitoring and rely on the status page as the single source of truth.
- B. The status page is wrong often enough that it should be ignored entirely.
- C. Your own monitoring (error rate, latency, cost per successful call) is the primary signal, and the status page is confirmation and public communication — treating the status page as the detector means you are 5–15 minutes behind the real incident.
- D. Status pages are not public; only enterprise customers can see them.

## 12. Chapter 5 pairs **retries** (from mod-107 chapter 4) with a **circuit breaker** on the primary provider. Which statement captures the chapter's reason for using both?

- A. Retries and circuit breakers are redundant; pick one and delete the other.
- B. Retries handle the sub-minute transient (a single 429 or a flaky 503 that clears in a second); the breaker handles the multi-minute outage (provider genuinely down for 30+ minutes) — together, retries don't amplify an outage into a 5×-bill-for-failure, and users don't sit through a slow retry chain when the provider is down for real.
- C. The breaker exists only to open during planned maintenance; retries handle everything unplanned.
- D. The breaker replaces exponential backoff because exponential backoff violates the SSE spec.

---

## Answer key

1. **B.** Chapter 1 ("The three axes" and "The evaluation loop, in one page") spells it out: accuracy on a task-specific test set, cost per call from the `usage` block, and p50/p95 latency measured across a distribution. Benchmark scores (A) are explicitly rejected as "measure model quality *in general*," sticker-price averages (D) are called out under "Cost from `usage`, not from the pricing page."
2. **B.** Chapter 1 ("Two providers, not one") lists exactly these two reasons: "Providers do not agree on which tasks their small models handle well" and "You need a fallback for chapter 5." A and D are not claims the chapter makes.
3. **C.** Chapter 2 ("Nuance 1: input and output prices are different") uses this exact trade as a worked example: "A prompt that adds 500 input tokens to trim 100 output tokens is often a good trade; the reverse (500 more output tokens to save 100 input tokens) usually is not." Averaging the two (B) is explicitly warned against.
4. **B.** Chapter 2 ("Per-user budget" → the "Reserve *before* the call, reconcile *after*" bullet) names this: "You do not know the true output size until the response comes back. Reserve the worst case (input + `max_tokens`); after the response, decrement by the unused portion." The reservation is what prevents two concurrent requests from both passing a stale check.
5. **A.** Chapter 2 ("Common bugs" → "Enforcing on token count of the raw string, not the tokenized version") says exactly this: "`len(prompt) / 4` is a rough estimate — do not use it for enforcement. Count tokens with the provider's tokenizer / token endpoint."
6. **C.** Chapter 3 ("How the mechanism works, briefly" and "Structuring the prompt for hits") spells out prefix-based matching: "Caching is prefix-based. The cache matches from the beginning of the message list forward. If the first token of your prompt differs from a cached entry, no hit."
7. **C.** Chapter 3 ("Antipatterns that quietly break caching") uses this exact example: "`\"You are Claude, and today is {current_date}\"` at the top of the system prompt. Every day's requests miss every other day's cache."
8. **B.** Chapter 3 ("Measuring the cache-hit ratio" → "Common mistakes" → "Measuring hit ratio in requests instead of tokens") says: "'80% of calls hit cache' is not the same as '80% of input tokens were served from cache.' The token-weighted number is the one that maps to cost."
9. **B.** Chapter 4 ("Pattern 1 — cascade routing" → "Economics" and "Failure mode to watch") makes this explicit: cost = `C_cheap + p * C_frontier`, and when `p` is near 1 you pay for both calls on every request, which is strictly worse than always-frontier. The whole value of a cascade depends on `p` staying well below 1.
10. **C.** Chapter 4 ("Pattern 3 — batch APIs") describes the use case verbatim: "Any workload whose latency budget is measured in hours, not seconds. Cron jobs. Eval runs. Data enrichment. Backfills." The ~50% discount and up-to-24-hour turnaround are both named. Small jobs (D) are explicitly flagged as a mistake — "batch overhead exceeds the discount."
11. **C.** Chapter 5 ("Provider status pages" → "Two things to know") says: "They are a lagging indicator. A real incident is often visible in your own error rates and latency histograms 5–15 minutes before the status page updates. Your monitoring must be the primary signal; the status page is confirmation and public communication." The "Common mistakes" section repeats this as "Treating status pages as the primary signal."
12. **B.** Chapter 5 ("The health-check pattern") names this composition exactly: "This composes with the retry policy from mod-107 chapter 4. The retry policy handles the sub-minute transient; the breaker handles the multi-minute outage." The two together produce the properties enumerated in the same section — retries do not amplify the outage, users do not wait through a slow retry chain during a real outage, and the feature notices recovery within one probe cycle.
