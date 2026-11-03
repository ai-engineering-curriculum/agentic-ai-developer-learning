# Chapter 3 — Prompt caching and cache-hit ratio

Most production LLM features send the same tokens over and over. A system prompt of 800 tokens, a set of few-shot examples of 1500 tokens, a chunk of documentation of 3000 tokens — all of it appears at the top of every single request, unchanged from one call to the next. Without caching you pay full input-token price for those 5300 tokens on every call. With caching enabled, the provider computes them once, keeps the intermediate state, and bills you a fraction of the price on subsequent requests for as long as the cache lives.

This chapter is about turning that mechanism on, structuring your prompt so it actually works, and measuring the cache-hit ratio in a real workload so you know it is working.

## Motivation

Prompt caching is unusual among cost-reduction techniques in that it does not require any change to what your feature does or how it behaves. The output is identical whether the cache hit or missed. The only difference is the invoice. That makes it one of the highest-return, lowest-risk changes you can make to an LLM application — and, correspondingly, one of the changes teams most often forget.

Two specific failure modes justify a full chapter:

1. **"We turned it on, we're fine."** Enabling caching is not the same as caching working. If your prompt structure changes on every call (a timestamp near the top, a random user ID inserted before the few-shot examples), your cache-hit ratio is zero. The bill doesn't move; nobody looks; the assumption "it's enabled" persists for months.
2. **"The cache is cold most of the time."** Provider caches expire on a short TTL — Anthropic and OpenAI both document minutes-scale TTLs by default. A feature that gets one request every 10 minutes may still have a zero cache-hit ratio simply because the cache always expired before the next call arrived. Different remedies apply depending on which failure mode you're in — measure to find out which.

The rest of the chapter is: how the two big providers expose caching, how to structure a prompt so it caches, how to read the `usage` block for hit-vs-miss data, and how to compute the cache-hit ratio as an operational metric.

## How the mechanism works, briefly

The core idea is the same across providers:

- You mark a section of your prompt as **cacheable**.
- On the first request, the provider processes those tokens and stores the resulting internal state, keyed by the exact token content up to and including the marked boundary. You are billed full input price for that section on this call.
- On subsequent requests, if the *prefix* of the prompt (from the start up to a cacheable boundary) matches an existing cache entry, the provider reuses the stored state instead of re-processing. You are billed a **reduced input price** for the cached tokens on those calls.

Two properties matter:

- **Caching is prefix-based.** The cache matches from the beginning of the message list forward. If the first token of your prompt differs from a cached entry, no hit. Put the volatile stuff (user input, timestamps, per-request IDs) at the **end** of the prompt; put the stable stuff (system prompt, few-shot examples, tool definitions, big context blocks) at the **beginning**.
- **Caching has a TTL.** Providers currently default to ~5 minutes; longer TTLs may be available at a different price point. Cold-start requests always miss.

## Anthropic prompt caching

Anthropic exposes caching via a `cache_control` field on any content block. Marking a block with `{"type": "ephemeral"}` tells the API "the prompt up to and including this block is cacheable."

Reference: <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>

Minimal example:

```python
import anthropic

client = anthropic.Anthropic()

STABLE_SYSTEM = "You are a support-ticket classifier..."  # long, stable
LARGE_FEW_SHOT = "..." * 500  # long, stable

response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=200,
    system=[
        {
            "type": "text",
            "text": STABLE_SYSTEM,
        },
        {
            "type": "text",
            "text": LARGE_FEW_SHOT,
            "cache_control": {"type": "ephemeral"},  # mark cacheable up to here
        },
    ],
    messages=[
        {"role": "user", "content": "The refund never arrived."},   # volatile
    ],
)

print(response.usage)
# Fields include (names may vary — verify against the current SDK):
#   cache_creation_input_tokens  — tokens processed and stored this call
#   cache_read_input_tokens      — tokens served from cache this call
#   input_tokens                 — total fresh input tokens this call
#   output_tokens                — reply tokens
```

<!-- needs-research: confirm the exact Anthropic usage field names (`cache_creation_input_tokens`, `cache_read_input_tokens`) and default cache TTL as of 2026-08 — cite from the docs page above. -->

Notes:

- You can mark up to a small number of cache breakpoints per request (check the current limit in the docs). Common pattern: one breakpoint at the end of the system prompt / few-shot / retrieved-context section.
- The `ephemeral` type is the default TTL (short — minutes-scale). Longer-TTL cache tiers may be available; look them up before assuming they are.
- The cache is scoped to your API key / organisation. Two different customers making the same call do not share cache — no data leakage risk from caching.

## OpenAI cached input

OpenAI's mechanism is slightly different in shape: **caching is automatic** for prompts above a minimum size (currently ~1024 tokens; verify in the docs). You don't mark cache boundaries yourself. Any request whose prompt shares a long-enough prefix with a recent request will be served with cached input tokens.

Reference: <https://platform.openai.com/docs/guides/prompt-caching>

Minimal example:

```python
from openai import OpenAI

client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4.1",  # or whichever current model — check the pricing page for caching support
    messages=[
        {"role": "system", "content": STABLE_SYSTEM + LARGE_FEW_SHOT},
        {"role": "user", "content": "The refund never arrived."},
    ],
    max_tokens=200,
)

print(response.usage)
# usage.prompt_tokens_details.cached_tokens — how many input tokens hit the cache
# usage.prompt_tokens                       — total input tokens
# usage.completion_tokens                   — reply tokens
```

<!-- needs-research: confirm OpenAI `prompt_tokens_details.cached_tokens` field name and the current minimum-prompt-size threshold for caching to be considered as of 2026-08. -->

Notes:

- Because caching is automatic, the trick on OpenAI is **prompt structure**: put stable content first, volatile content last, and keep the stable section long enough (currently ≥1024 tokens) to be eligible.
- OpenAI's cached input tokens are billed at a reduced rate; check the pricing page for the current ratio.

## Structuring the prompt for hits

The single most important structural rule, for both providers:

**Stable at the top. Volatile at the bottom.**

Concretely, in order from top to bottom of your message list:

1. **System prompt / instructions** — changes rarely; stable across requests.
2. **Tool definitions** — changes only when you add/remove/rename a tool.
3. **Few-shot examples** — stable while you're iterating.
4. **Large retrieved context** — stable within a session or a document.
5. **Conversation history** — grows over the session; the earlier turns are stable, the latest turn is not.
6. **Current user input** — always volatile.

A prompt in this order is a prompt that caches. A prompt that puts the timestamp at the top does not.

### Antipatterns that quietly break caching

- **`"You are Claude, and today is {current_date}"` at the top of the system prompt.** Every day's requests miss every other day's cache. Move the date to a per-turn note in the user message, or omit it if the model does not truly need it.
- **`"User {user_id} asks: ..."` at the very start of the message list.** Every user's requests miss every other user's cache. Move the user ID to a user-role message deeper in the list.
- **Randomising the order of few-shot examples per request.** Each order is a distinct prefix; hit ratio collapses.
- **Adding a UUID or request ID as the first line of the system prompt.** Every request is a cache miss.
- **A/B testing two prompt variants by randomly picking one.** Two disjoint cache lines; roughly half the hit rate you'd otherwise get. Cache each variant separately, but at least understand the trade.

### Antipatterns that quietly reduce the *value* of caching

- **The stable section is too short.** Below the provider's minimum (OpenAI) or too small to matter (Anthropic), caching either doesn't engage or the savings are trivial.
- **Feature traffic is too sparse.** If your feature gets one call every 10 minutes and the TTL is 5, every call is a cold miss. Solutions: pin the cache warm with a periodic no-op call, or accept that this feature is not one caching is going to help.
- **You cache a prompt that changes every deploy.** Every deploy invalidates your cache; the first requests after a deploy always miss. This is fine, just don't be surprised.

## Measuring the cache-hit ratio

You cannot improve what you do not measure. Every response `usage` block from Anthropic and OpenAI carries fields that let you compute a per-request hit ratio. Aggregate across a window and you have an operational metric.

### The formula

Per request:

```
cache_read_tokens   = tokens served from cache
cache_write_tokens  = tokens newly written to cache this call
fresh_input_tokens  = tokens processed as fresh input (never cache-eligible or cache miss)
total_input_tokens  = cache_read + cache_write + fresh_input

hit_ratio_this_call = cache_read_tokens / total_input_tokens
```

Aggregated over N calls in a window:

```
window_hit_ratio = sum(cache_read_tokens across calls)
                 / sum(total_input_tokens across calls)
```

**Report the ratio, not just the raw numbers.** "80% of input tokens served from cache" is a metric you can put a threshold on. "42,900 cache read tokens today" is not.

### What to log per request

- `cache_read_tokens` (Anthropic: `cache_read_input_tokens`; OpenAI: `prompt_tokens_details.cached_tokens`).
- `cache_write_tokens` (Anthropic: `cache_creation_input_tokens`; OpenAI: not always broken out explicitly — check the docs).
- `total_input_tokens`.
- Which prompt version / template ID produced the call. If you don't record the template ID, you cannot investigate a drop in hit ratio — was it a prompt change or a traffic-shape change?

### What "healthy" looks like

There is no universal target. A rough set of expectations:

- **> 70% hit ratio** on a feature with a large stable system prompt / few-shot / context section and steady traffic is achievable and normal.
- **< 30% hit ratio** on the same shape of feature usually means: (a) the volatile section is high in the message list, (b) traffic is too sparse for the TTL, or (c) something is randomising the prefix.
- **0% hit ratio** on a feature you enabled caching for means either (a) the caching mechanism isn't actually engaged (Anthropic: no `cache_control` block; OpenAI: prompt below the minimum), or (b) the prefix changes on every call. Check the prompt structure.

Exercise 03 is a real measurement — you'll enable caching, run a workload, and compute the window ratio.

## Cost savings from caching

The pricing pages spell this out per model — the same page you used in chapter 2. Two common shapes:

- **Anthropic**: cached input tokens are typically ~10% of the fresh input price (i.e. ~90% cheaper). Cache *writes* are typically ~125% of the fresh input price (i.e. ~25% more expensive) because the provider is doing extra work to store the state. Net effect: caching is a win the moment you have more than a small handful of reads per write.
- **OpenAI**: cached input tokens are typically ~50% of the fresh input price. There is no separate write charge on OpenAI — the discount applies whenever the cache hits.

<!-- needs-research: confirm the exact input / cache-read / cache-write pricing ratios for both providers as of 2026-08. -->

Two implications:

- **Anthropic caching becomes valuable when you have many reads per write.** A prompt cached once and read 100 times is a big win. A prompt cached once and read twice may barely pay off. The break-even is a function of the read/write price ratio and your traffic pattern.
- **OpenAI caching pays off on the very first hit.** You don't need to reason about break-even; if the prompt is long enough to be cache-eligible and stable enough to hit, you save money the first time it hits.

## Prompt caching and correctness

Two things that are true and worth stating explicitly, because teams get anxious about caching:

- **The response is not read from cache.** Only the intermediate state derived from the input is. The model still generates the response from scratch every call. Caching does not deduplicate two identical questions into one identical answer.
- **Cache is scoped to your account.** Users do not share cache lines. A user's inputs are not made visible to another user via caching.

Both providers document these guarantees in the caching docs above.

## Common mistakes

- **Enabling caching without measuring.** You cannot verify a hit ratio you never computed. Log the cache tokens; compute the ratio in a dashboard.
- **Putting `cache_control` in the wrong place** (Anthropic). The cache breakpoint marks the *end* of the cacheable prefix — content after it is not cached. Put the breakpoint at the last stable block, not the first.
- **Measuring hit ratio in requests instead of tokens.** "80% of calls hit cache" is not the same as "80% of input tokens were served from cache." The token-weighted number is the one that maps to cost.
- **Ignoring cold-start requests.** If your service restarts or scales up, the first N requests hit the cold cache. In a sparse workload these dominate the ratio.
- **Assuming caching helps every workload.** A feature whose prompt is 95% user-generated content and 5% system prompt does not benefit meaningfully from caching. Do the arithmetic before assuming a win.

## Summary

- Prompt caching lets the provider reuse the intermediate state from previously-seen prompt prefixes, billing you a reduced price for the cached section. Same output; different invoice.
- Structure prompts **stable-at-top, volatile-at-bottom**. Anthropic requires a `cache_control` breakpoint; OpenAI caches automatically above a minimum prompt size.
- Measure the cache-hit ratio from the `usage` block on every response. Token-weighted, not request-weighted.
- Healthy on a large-stable-prompt feature: 70%+. If you see 0%, the prompt structure is wrong, not the provider.
- The financial benefit depends on the provider's cache-read discount and (Anthropic) cache-write premium. Verify from the current pricing page.

The next chapter shifts from making the same model cheaper to picking a different model per request — cascade routing, fast-first-then-escalate, and using batch APIs when the latency is not user-facing.
