# Chapter 2 — Cost per feature call and enforcing token budgets

Chapter 3 of mod-102 taught you to count tokens. This chapter takes those counts and turns them into two things: a dollar-number estimate of what a feature call will cost you before you ship it, and a hard enforcement boundary inside your application that refuses requests that would exceed a per-request or per-user budget. Both are cheap to build. Both are hard to add later, after the first surprising invoice.

## Motivation

The way LLM features overspend is almost never "we shipped and each call cost 10× what we thought." It is one of these three:

1. **Volume drift.** The per-call cost was fine. Then usage grew 100× and nobody added a budget cap. The invoice moved with usage; there was no ceiling.
2. **Per-user runaway.** One user (or one buggy client, or one abusive script) sent 10,000 calls in a day. Cost per call was still fine. There was no per-user quota.
3. **Prompt bloat.** Someone added a "helpful context" block to the system prompt — three paragraphs, a few-shot example, a chunk of documentation. The per-call cost quietly doubled. Nobody noticed until the monthly report.

The techniques in this chapter — pre-flight cost estimation, per-request token budgets, and per-user quotas at the application boundary — defend against all three.

## Cost per call, done properly

The formula from mod-101 chapter 3, restated for a specific feature call:

```
cost_per_call = (input_tokens  * P_in  / 1_000_000)
              + (output_tokens * P_out / 1_000_000)
```

Where `P_in` and `P_out` are per-million-token prices from the provider's pricing page. Two pieces of nuance the mod-102 version elides:

### Nuance 1: input and output prices are different

Output tokens are typically **3–5× more expensive than input tokens** on the same model. Anthropic and OpenAI both publish separate `$/MTok` numbers for input and output. Do not average them; the two are different levers.

- Anthropic pricing: <https://www.anthropic.com/pricing>
- OpenAI pricing: <https://openai.com/api/pricing/>

<!-- needs-research: exact per-million input and output token prices for the current default Claude and OpenAI models as of 2026-08 — cite from provider pricing pages once confirmed. -->

This matters for prompt design. A prompt that adds 500 input tokens to trim 100 output tokens is often a good trade; the reverse (500 more output tokens to save 100 input tokens) usually is not.

### Nuance 2: cached input tokens are their own line item

If you use prompt caching (chapter 3), the provider charges a **reduced price** for input tokens that hit the cache. Anthropic and OpenAI both report `cached_input_tokens` (or an equivalent field) in the `usage` block on every response — count them separately when you compute cost:

```
cost_per_call = (fresh_input_tokens  * P_in_fresh  / 1_000_000)
              + (cached_input_tokens * P_in_cached / 1_000_000)
              + (output_tokens       * P_out       / 1_000_000)
```

Chapter 3 covers the mechanism; note here that your cost calculation must break out cached vs. fresh input tokens or you will over- or under-report.

### Nuance 3: reasoning / thinking tokens

Some Claude and OpenAI models can produce "reasoning" or "thinking" tokens — internal deliberation that is billed as output, whether or not it is shown to the user. If you enable reasoning, count those tokens; they are usually returned as a separate field in the response `usage`. A prompt that reasons for 2000 tokens before producing 100 tokens of answer costs the same as one that produces 2100 tokens of answer.

- Anthropic extended thinking: <https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking>

<!-- needs-research: confirm the exact response field for output/reasoning tokens on the current Anthropic and OpenAI SDKs as of 2026-08. -->

## Pre-flight estimation vs. post-hoc measurement

There are two related questions:

- **Before deploy** — "if this feature launches, what will 10,000 calls per day cost me at p50 and worst-case?"
- **After deploy** — "what did today's actual usage cost?"

The first is answered from a **sample** of expected inputs, using the `tiktoken` client-side counter (OpenAI) or the token-counting endpoint (Anthropic) to size the input, and a chosen `max_tokens` cap as the worst-case output. Reference implementations in mod-101 chapter 3.

The second is answered from **`usage` on every response** — the provider's own count is authoritative, and it already includes framing overhead you would have to estimate.

The two do not agree perfectly. Server-side count includes small framing overhead the client-side count does not. Budget with a 5–10% safety margin on the client-side estimate.

### A worked example

Feature: a support-ticket classifier. Inputs are typically 200–500 tokens; the output is a `{"category": "...", "priority": "..."}` JSON, typically 30 tokens. You cap output at 100 tokens to be safe.

Expected traffic: 50,000 calls/day. You've picked a small-tier Claude or OpenAI model whose input price is `P_in` and output price is `P_out` per million tokens.

Numbers you fill in from the pricing page:

```
per_call_p50  = 350 * P_in / 1_000_000 + 30  * P_out / 1_000_000
per_call_max  = 500 * P_in / 1_000_000 + 100 * P_out / 1_000_000
daily_p50     = 50_000 * per_call_p50
daily_max     = 50_000 * per_call_max
monthly_p50   = daily_p50 * 30
monthly_max   = daily_max * 30
```

`monthly_max` is the number you show to the person approving the launch. If `monthly_max` is a number you would not want on an invoice, either:

- Lower the `max_tokens` cap (reduces worst-case output cost).
- Trim the input template (reduces both p50 and worst-case input cost).
- Enable prompt caching for the stable parts of the input (chapter 3).
- Route via a smaller model for the easy 80% of inputs (chapter 4).
- Add per-user quotas so a single user cannot walk off with the budget.

That is the workflow. Every LLM feature you ship should have this table filled out somewhere in the repo before it goes live. Exercise 01 makes you do exactly this for a specified feature.

## Enforcement at the application boundary

An estimate is a plan. A budget cap is a promise. If you shipped only the estimate you would still lose money to the drift patterns from the motivation section. This section is about enforcement — a chunk of code that refuses to make the LLM call at all if it would exceed a threshold.

There are two kinds of enforcement worth building.

### Per-request budget

A single request that is *bigger* than expected is often the sign of an upstream bug or a hostile input. A user pasting a 200,000-token document into a support ticket, a template that stopped truncating, a fanout that concatenated ten users' history into one prompt. Refuse it at the boundary.

Skeleton:

```python
class RequestBudgetExceeded(Exception):
    pass

def enforce_request_budget(
    *,
    input_tokens: int,
    max_output_tokens: int,
    max_input_tokens: int = 8_000,
    max_total_tokens: int = 16_000,
) -> None:
    if input_tokens > max_input_tokens:
        raise RequestBudgetExceeded(
            f"input {input_tokens} exceeds cap {max_input_tokens}"
        )
    if input_tokens + max_output_tokens > max_total_tokens:
        raise RequestBudgetExceeded(
            f"input+max_output {input_tokens + max_output_tokens} exceeds cap {max_total_tokens}"
        )
```

Call it *before* the SDK call. `input_tokens` is the pre-flight count from `tiktoken` (OpenAI) or the token-counting endpoint (Anthropic). `max_output_tokens` is whatever cap you're going to pass on the request itself.

Two rules for setting the caps:

- **`max_input_tokens` is a business decision, not a model decision.** The model's context window is 200K; that does not mean your feature should accept a 200K-token prompt. Pick the smallest cap that fits your real workload — 4K, 8K, 16K.
- **The cap should be visible in error messages.** When a caller gets a `RequestBudgetExceeded`, they should know exactly which cap they hit and why. Silent truncation is worse than a loud refusal.

### Per-user budget

A per-request cap prevents one *pathological request*. A per-user cap prevents one user (or one client) from making 100,000 normal-sized requests. This is where the cheapest way to lose money — an abusive user or a stuck client — gets contained.

The mechanism is a **rolling window counter** keyed by user (or API key, or IP, or whatever your notion of a caller is). Every request adds to the counter; the counter refuses requests once it exceeds the quota within the window.

For a small deployment, an in-memory dict with expiring keys is enough. For a real service, use Redis or a comparable store — you want the counter to be shared across processes.

A pattern:

```python
import time

class UserBudgetExceeded(Exception):
    pass

class RollingTokenBudget:
    """
    Per-key rolling-window token budget.
    For a real service, back this with Redis; for local work, dict is fine.
    """
    def __init__(self, *, tokens_per_window: int, window_seconds: int):
        self.tokens_per_window = tokens_per_window
        self.window_seconds = window_seconds
        self._events: dict[str, list[tuple[float, int]]] = {}

    def _prune(self, key: str, now: float) -> None:
        cutoff = now - self.window_seconds
        events = [e for e in self._events.get(key, []) if e[0] >= cutoff]
        self._events[key] = events

    def check_and_reserve(self, key: str, tokens: int) -> None:
        now = time.monotonic()
        self._prune(key, now)
        used = sum(t for _, t in self._events[key])
        if used + tokens > self.tokens_per_window:
            raise UserBudgetExceeded(
                f"user {key} would exceed {self.tokens_per_window} tokens/{self.window_seconds}s "
                f"(used {used}, requested {tokens})"
            )
        self._events[key].append((now, tokens))
```

Two subtleties:

- **Reserve *before* the call, reconcile *after*.** You do not know the true output size until the response comes back. Reserve the worst case (input + `max_tokens`); after the response, decrement by the unused portion. Otherwise a heavy user hits their cap earlier than they should because the code is pessimistic about output size.
- **The window matters.** A per-minute cap prevents spikes; a per-day cap prevents drain. Most services want both — a small per-minute cap layered on top of a larger per-day cap.

### Where to enforce

The rule: **the enforcement layer is between "the request is authenticated and its size is known" and "the SDK is called."** Not in the SDK. Not on the LLM. In your code, at the boundary where user identity and payload size are both available.

If your app has a service layer, this is a middleware-like wrapper around the LLM call. If it's a single script, it's a function you call before the SDK call. Either way, the SDK call itself must never be reachable without going through the enforcement layer.

## Logging cost, not just tokens

Every response has a `usage` block. Log the derived cost from it, not just the raw counts, so your dashboards do not require a downstream join with the pricing table.

```python
def cost_from_usage(usage, *, p_in_fresh, p_in_cached, p_out) -> float:
    return (
        usage.input_tokens        * p_in_fresh  / 1_000_000
      + usage.cached_input_tokens * p_in_cached / 1_000_000
      + usage.output_tokens       * p_out       / 1_000_000
    )
```

Log the cost per request. Aggregate per feature, per user, per day. When a bill goes up, you want to be able to answer "who spent it?" in a single query, not "let me reconstruct it from token counts and last month's price sheet."

## Alerting on drift

The metric you want to alarm on is not raw daily spend — that moves with legitimate growth. It's **cost per successful call**, and **max cost per user in the window**, both broken down per feature.

- If cost per successful call drifts up 30% week-over-week, someone bloated the prompt.
- If max cost per user in the window is 100× the median, one caller is either buggy or abusive.

Both are cheap alarms to define once you're logging per-request cost.

## Common bugs

- **Enforcing on token count of the raw string, not the tokenized version.** `len(prompt) / 4` is a rough estimate — do not use it for enforcement. Count tokens with the provider's tokenizer / token endpoint.
- **Enforcing only on input, not on `input + max_tokens`.** A request with 100 input tokens and `max_tokens=10_000` is still a 10,100-token bill in the worst case.
- **Applying the same budget to every user regardless of role.** A background job runs on service credentials and needs a much larger budget than a signed-in user; a free-tier user needs a much smaller one than a paying customer. Budgets should be a function of caller role.
- **Refusing without a machine-parseable error.** Callers may want to retry with a smaller payload; make the error carry the cap and the exceeded amount, not just a message string.
- **No safe-mode fallback.** The right response to a budget refusal is often to fall back to a cheaper path (a smaller model, a template response, a cached answer). If enforcement returns a 500, the feature is worse than not having enforcement at all — chapter 5 goes deeper on this pattern.
- **Forgetting reasoning tokens in the budget.** Reasoning tokens are billed as output and can dwarf the visible reply. Include them in the reserved amount.

## Summary

- Cost per call is `input_tokens * P_in + output_tokens * P_out`, plus a separate line item for cached input tokens (chapter 3). Compute from the `usage` block; do not memorise prices.
- Before you ship, fill in a small table: p50 and worst-case per-call cost, daily cost at expected volume. If worst-case scares you, fix it before launch, not after.
- Enforce a **per-request** budget: refuse requests whose input, or input plus `max_tokens`, exceeds your cap. Cap is a business decision, not a model decision.
- Enforce a **per-user** budget: rolling-window token counter keyed by user, checked before the SDK call. Reserve worst-case; reconcile after the response.
- Log cost per request, not just tokens. Alarm on cost-per-successful-call drift and on per-user max, not on raw spend.

The next chapter turns to the single largest cost-reduction lever available inside a prompt: prompt caching. Enable it and measure your hit ratio, and you'll often cut input-token cost on stable prompts by 80–90%.
