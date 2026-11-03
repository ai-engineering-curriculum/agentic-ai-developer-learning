# Exercise 03 — Prompt caching hit ratio on a real workload

Paired with [chapter 3 — prompt caching and cache-hit ratio](../03-prompt-caching-and-cache-hit-ratio.md).

**Estimated effort:** 90–120 minutes.

## Objective

Enable prompt caching for a real workload on both Anthropic and OpenAI (pick one to start; do both if time allows). Structure the prompt for cache reuse. Measure the token-weighted cache-hit ratio over a run. Compute the observed cost savings vs. the no-caching baseline. Prove — with numbers, not vibes — that caching is engaged and paying off.

## Problem statement

Build a runnable workload script that:

- Takes ~50 varied user inputs (short questions on a shared topic — programming questions, product questions about a fictional company, whatever).
- For each input, calls the LLM with a **large stable prefix** followed by the user's input.
  - "Large stable prefix" = a system prompt of at least 2,000 tokens (a fictional product manual, a coding style guide, a long few-shot example list — anything real, so the model has to actually process it).
- Runs the same 50 inputs twice: once with caching **disabled**, once with caching **enabled** and correctly structured.

## Requirements — the caching structure

**Anthropic** (if you pick Anthropic):

- Put the stable prefix in the `system` array as one or more text blocks.
- Mark the last stable block with `cache_control: {"type": "ephemeral"}`.
- Put the user input in the `messages` list as a `user` message. Nothing volatile in the system prompt.
- Reference: <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>.

**OpenAI** (if you pick OpenAI):

- Caching is automatic above a minimum prompt size (currently ~1024 tokens; verify from the docs).
- Put the stable prefix in a single `system` message (or as the leading `system` content). Volatile user text goes in the `user` message.
- Reference: <https://platform.openai.com/docs/guides/prompt-caching>.

For both, the rule is the same: **stable at the top, volatile at the bottom.**

## Requirements — the measurement

Log per response:

- `input_tokens_total` (total input tokens processed for the call).
- `cache_read_tokens` (Anthropic: `cache_read_input_tokens`; OpenAI: `usage.prompt_tokens_details.cached_tokens`).
- `cache_write_tokens` if the provider reports it separately (Anthropic: `cache_creation_input_tokens`).
- `output_tokens`.
- Wall-clock latency.
- Whether caching was enabled for this call.

Then compute and print:

1. **Per-call hit ratio** = `cache_read_tokens / input_tokens_total` on the caching-enabled run.
2. **Window (aggregate) hit ratio** = `sum(cache_read_tokens) / sum(input_tokens_total)` across the whole run.
3. **Cost with caching** (from the `usage` block, using the current per-token and per-cached-token prices from the provider's pricing page).
4. **Cost without caching** (the same 50 calls, computed with fresh-input price on all input tokens).
5. **% cost saved** by caching.
6. **Latency comparison**: mean and p95 for cache-off vs. cache-on runs. (Caching should also make the second-and-subsequent calls **noticeably faster** on the input side, in addition to cheaper.)

## Requirements — the failure exploration

After the happy-path measurement, deliberately **break caching** two ways and re-measure. Explain in a short `NOTES.md` what happened to the hit ratio:

1. **Put a timestamp at the top of the stable prefix.** Prepend `datetime.now().isoformat()` to the system prompt on every call. Re-run.
2. **Randomise the order of two few-shot examples per call.** Re-run.

For each, print the new hit ratio and note the delta in `NOTES.md`. The expected result: caching collapses in both cases.

## Requirements — the "cold cache" observation

Wait 15+ minutes (or whatever exceeds the current provider TTL — the docs above will tell you) with no requests. Send a single request. Confirm the response reports `cache_read_tokens = 0` or a much smaller number than usual — the cache expired. Note this in `NOTES.md`.

The point is not that TTLs are a bug; it's that **sparse-traffic features cannot rely on caching** without pinning the cache warm somehow.

## Starter guidance

- Chapter 3 above.
- Anthropic prompt caching: <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>.
- OpenAI prompt caching: <https://platform.openai.com/docs/guides/prompt-caching>.
- Both providers publish per-cached-token pricing on their pricing pages — cite the pricing page and the date fetched at the top of your script.
- For the stable prefix, don't just copy-paste the same sentence 200 times. The model does have to *process* the prefix on the first call; using real-looking text (a fake product manual, a real style guide) makes the exercise honest.

## Acceptance criteria

- Two runs of the same 50 inputs completed: caching off and caching on.
- Per-call and aggregate token-weighted hit ratios printed.
- Aggregate hit ratio on the "caching on" run is > 50% (well-structured stable prefix + steady traffic should easily clear this).
- Aggregate cost with caching is **at least 30% lower** than without, based on the `usage` block. (The exact ratio depends on the provider's cache-read discount and cache-write premium; note the numbers from the pricing page in your output.)
- Latency comparison shows the cached run is faster on average — quantify by how much.
- Deliberate break #1 (timestamp at top) drops the hit ratio close to zero. Deliberate break #2 (randomised few-shot order) meaningfully reduces it too. Both are noted in `NOTES.md`.
- Cold-cache observation is noted, with the actual `cache_read_tokens` value after the wait.

## Stretch goals

- Do the same exercise on the **other** provider, so you have hit-ratio and savings numbers on both. Note how the mechanism differs (Anthropic explicit breakpoint vs. OpenAI automatic).
- Simulate **sparse traffic**: schedule 20 requests spaced 6 minutes apart via `time.sleep` or a scheduler. Measure the hit ratio; it should be low. Then add a "warm-up pinger" — a background loop that calls the endpoint every 2 minutes to keep the cache warm — and re-run. Compare hit ratios.
- Add a **cache-hit-ratio dashboard entry**: parse your log JSONL and print daily aggregates (`hit_ratio_by_day`, `hit_ratio_by_prompt_version`). Adding a `prompt_version` tag on every call is the trick that lets you attribute a cache regression to a prompt edit.
- Extend the cost-estimate script from exercise 01 to account for the measured hit ratio here. Compare the estimated monthly bill with and without caching. This is the number you'd cite in a design review.
