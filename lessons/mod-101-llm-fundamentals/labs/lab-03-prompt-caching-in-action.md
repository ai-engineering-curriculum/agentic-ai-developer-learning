# Lab 03 — See Prompt Caching in the `usage` Field

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 7 — Estimating Cost and Latency](../07-cost-and-latency.md), [chapter 3 — Tokens and Context Windows](../03-tokens-and-context-windows.md)

## Objective

Chapter 7 lists "cache stable prefixes" as the single highest-payoff cost lever, and names the exact `usage` fields that tell you whether it is working: Anthropic's `cache_creation_input_tokens` and `cache_read_input_tokens`, OpenAI's `prompt_tokens_details.cached_tokens`. In this lab you will make three identical calls with a large, stable system prompt, read those fields on each response, and watch cached-token counts replace un-cached ones. If prompt caching ever looks "like it should be working" in production, this is the pattern you will reach for to prove it.

By the end you will have:

- A long, byte-identical system prefix sent three times in a row.
- Three `usage` dicts showing cache-write on call 1 and cache-reads on calls 2 and 3.
- A one-paragraph note on what the `usage` field tells you and what it does **not** tell you.

> This lab complements [exercise-02](../exercises/exercise-02-tokens-context-cost.md) task 5 ("Try one optimization — enable prompt caching"). The exercise lets you pick any optimization; this lab walks the caching path step by step and focuses on reading the `usage` fields correctly.

## Prerequisites

- Finished chapters 3 and 7.
- An API key for the provider you want to demonstrate caching on:
  - **Anthropic path:** `ANTHROPIC_API_KEY` exported, `pip install anthropic`. Anthropic requires an explicit `cache_control` marker on the block you want cached.
  - **OpenAI path:** `OPENAI_API_KEY` exported, `pip install openai`. OpenAI caches automatically on long enough stable prefixes; you only need to read the right field. <!-- needs-research: current provider-specific minimum prefix size required to trigger cache; follow the pricing / prompt-caching docs linked from resources.md -->
- Pick **one** provider path for this lab. The patterns are different enough that running both at once hides the point.

## Step 1 — Prepare a long, stable prefix

Caching rewards stable prefixes. In a file `big_prefix.txt`, paste something long and realistic: a style guide, API reference, legal boilerplate, or a chapter from any technical document you have on disk. Aim for several thousand tokens — chapter 7's example used a 1,200-token system prompt and still saved real money on heavy traffic, but you want enough length that the cache clearly fires.

Verify the length with the official counter (chapter 3):

```python
# Anthropic
from anthropic import Anthropic
client = Anthropic()
with open("big_prefix.txt") as f:
    prefix = f.read()
count = client.messages.count_tokens(
    model="claude-sonnet-4-6",
    system=prefix,
    messages=[{"role": "user", "content": "hi"}],
)
print(f"prefix tokens: {count.input_tokens:,}")
```

```python
# OpenAI
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4.1-mini")
with open("big_prefix.txt") as f:
    prefix = f.read()
print(f"prefix tokens (approx, no chat overhead): {len(enc.encode(prefix)):,}")
```

### Checkpoint

- Is the prefix at least a few thousand tokens? If not, the provider may silently decide the prefix is not worth caching and you will see no cache hit. <!-- needs-research: exact minimum sizes vary by provider; confirm with the current prompt-caching doc listed in resources.md -->
- Is the prefix **byte-identical** across calls? Chapter 7 warns that re-rendering a system prompt with a changing timestamp or session ID invalidates the cache. If you want to test that negative case, do it in the stretch.

## Step 2A — Anthropic: three calls with `cache_control`

Anthropic requires an explicit `cache_control` block to mark where the cacheable prefix ends. Pass the system prompt as a *list of content blocks* and set `cache_control` on the last cacheable block.

Save as `cache_anthropic.py`:

```python
import os
import time

from anthropic import Anthropic

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

with open("big_prefix.txt") as f:
    prefix = f.read()

system_blocks = [
    {
        "type": "text",
        "text": prefix,
        "cache_control": {"type": "ephemeral"},
    }
]

questions = [
    "Summarize the attached material in two sentences.",
    "List three risks implied by the attached material.",
    "Pick one claim in the attached material and rephrase it precisely.",
]

for i, q in enumerate(questions, start=1):
    t0 = time.monotonic()
    r = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=256,
        system=system_blocks,                      # byte-identical every call
        messages=[{"role": "user", "content": q}],
    )
    dt = time.monotonic() - t0
    u = r.usage
    print(
        f"[call {i}] {dt:.2f}s  "
        f"input={u.input_tokens}  output={u.output_tokens}  "
        f"cache_write={getattr(u, 'cache_creation_input_tokens', 0)}  "
        f"cache_read={getattr(u, 'cache_read_input_tokens', 0)}  "
        f"stop={r.stop_reason}"
    )
```

Run it:

```bash
python cache_anthropic.py
```

### Expected output (shape)

Three lines. On call 1, `cache_write` is large (the whole prefix) and `cache_read` is 0 — this is the cache *write*. On calls 2 and 3, `cache_write` is 0 (or just the small delta for your short user message) and `cache_read` carries the prefix tokens. The visible `input_tokens` on calls 2 and 3 is only the non-cached delta — chapter 7 described this as "the first write to the cache" costing a surcharge and subsequent reads being charged at a fraction.

```
[call 1] 2.14s  input=24   output=89   cache_write=5218  cache_read=0     stop=end_turn
[call 2] 0.63s  input=27   output=74   cache_write=0     cache_read=5218  stop=end_turn
[call 3] 0.58s  input=31   output=102  cache_write=0     cache_read=5218  stop=end_turn
```

(Your numbers will differ; the *shape* — a one-time write, then reads — is what matters.)

### Checkpoint

- Does call 1 show `cache_creation_input_tokens` > 0 and call 2 show `cache_read_input_tokens` > 0? If not: did your prefix actually meet the minimum size, and is the `system` parameter *byte-identical* on every call?
- Did call 2 arrive faster than call 1? Chapter 7 notes that the first call after a cache miss is slower; subsequent reads skip prefill on the cached prefix.

## Step 2B — OpenAI: three calls, read `cached_tokens`

OpenAI caches long stable prefixes automatically on supported models — you do not pass a cache marker, you just send the same prefix and read `prompt_tokens_details.cached_tokens` on the response.

Save as `cache_openai.py`:

```python
import os
import time

from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

with open("big_prefix.txt") as f:
    prefix = f.read()

questions = [
    "Summarize the attached material in two sentences.",
    "List three risks implied by the attached material.",
    "Pick one claim in the attached material and rephrase it precisely.",
]

for i, q in enumerate(questions, start=1):
    t0 = time.monotonic()
    r = client.chat.completions.create(
        model="gpt-4.1-mini",
        max_tokens=256,
        messages=[
            {"role": "system", "content": prefix},   # byte-identical every call
            {"role": "user",   "content": q},
        ],
    )
    dt = time.monotonic() - t0
    u = r.usage
    details = getattr(u, "prompt_tokens_details", None)
    cached = getattr(details, "cached_tokens", 0) if details else 0
    print(
        f"[call {i}] {dt:.2f}s  "
        f"prompt={u.prompt_tokens}  completion={u.completion_tokens}  "
        f"cached={cached}  "
        f"finish={r.choices[0].finish_reason}"
    )
```

Run it:

```bash
python cache_openai.py
```

### Expected output (shape)

Three lines. On call 1, `cached` is 0 (no cache yet exists). On calls 2 and 3, `cached` is a large chunk of the prompt — chapter 7 described this as the signal that caching applied.

```
[call 1] 1.73s  prompt=5309   completion=94   cached=0      finish=stop
[call 2] 1.11s  prompt=5312   completion=71   cached=5120   finish=stop
[call 3] 1.08s  prompt=5316   completion=98   cached=5120   finish=stop
```

### Checkpoint

- On OpenAI, `prompt_tokens` stays roughly the same across calls — caching changes what you are **billed** for, not what the chapter calls the *count* of input tokens. Make sure your mental model separates "tokens the model saw" from "tokens the provider charged full rate for."
- If `cached_tokens` is 0 on calls 2 and 3, your prefix is probably below the automatic-cache threshold. Try a longer prefix.

## Step 3 — Write a short interpretation

In a `caching_notes.md` file, write 4–6 sentences answering:

1. On your three calls, what is the ratio of cache-write to cache-read tokens? (Anthropic: `cache_creation_input_tokens` / `cache_read_input_tokens`. OpenAI: `prompt_tokens - cached_tokens` / `cached_tokens` on calls 2 and 3.)
2. Which call's wall time dropped the most, and why does chapter 7 predict that? ("Pre-warm caches. First call after a cache miss is slower.")
3. What does caching *not* fix? (Chapter 7 is clear: output tokens are never cached; model behavior is not cached; a changing timestamp in the "stable" prefix breaks the cache.)

That note is the deliverable. The scripts prove the mechanism; the note proves you can read the signal.

## Debrief

You have now:

- Verified prompt caching changes the `usage` fields in the way chapter 7 described.
- Felt the latency win on cache-reads.
- Written down the limits of the technique in your own words.

In production, this same reading of `usage` tells you whether a deploy accidentally broke a cache (e.g., someone templated a timestamp into the system prompt and knocked everyone off the cache cliff). Watch the field; don't trust that caching is working.

## Stretch

- Add a changing suffix inside the "stable" prefix (e.g., `now = time.time()`). Does caching still apply? Why not? (Chapter 7: cache is on *stable prefixes* — any change in the first N bytes invalidates it.)
- On Anthropic, mark two different `cache_control` points and watch which cache fires with which prompt. <!-- needs-research: confirm current semantics for multiple cache_control markers in the prompt-caching docs linked from resources.md -->
- Build a tiny before/after cost table using the formula from chapter 7 and the current pricing-page numbers for your pinned model. How many cache-reads does it take to amortize the write surcharge? That is the number you should know before you turn caching on in a low-volume feature.
