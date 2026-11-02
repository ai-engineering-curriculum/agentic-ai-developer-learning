# Lab 02 — Side-by-Side Model-Tier Comparison

**Estimated duration:** 60–75 minutes
**Covers:** [chapter 5 — Choosing a Model](../05-choosing-a-model.md), [chapter 7 — Estimating Cost and Latency](../07-cost-and-latency.md), [chapter 4 — Sampling, Temperature, and Determinism](../04-sampling-temperature-and-determinism.md)

## Objective

Chapter 5's practical sequence is "start workhorse, try cheap, only pay for flagship when you have to." In this lab you will put that into practice as a mini-eval: run the same ten prompts through two tiers of the same provider, score the replies by hand, and look at the measured `input_tokens`, `output_tokens`, and wall time for each. The deliverable is a decision — which tier would you ship? — grounded in numbers, not vibes.

By the end you will have:

- A small, repeatable `compare.py` runner.
- A CSV (`compare.csv`) with per-call model ID, tier, token usage, and latency.
- A short "which tier wins" note backed by that CSV.

> This lab complements [exercise-02](../exercises/exercise-02-tokens-context-cost.md) (which focuses on counting tokens and projecting cost from a single conversation) by forcing a cross-tier head-to-head on the same inputs.

## Prerequisites

- Finished chapters 4, 5, and 7.
- A funded API key for one provider:
  - Anthropic: `ANTHROPIC_API_KEY` exported.
  - OpenAI: `OPENAI_API_KEY` exported.
- Python 3.10+ with the SDK installed:
  - `pip install anthropic` or `pip install openai`.
- Budget note: ten short prompts × two tiers = twenty calls. Keep `max_tokens` modest; this lab should cost cents.

## Step 1 — Pick two tiers and pin exact model IDs

Chapter 5 ("Pinning models in production") says to use exact model IDs, not aliases like `latest`. Pin one workhorse-tier ID and one cheap-tier ID for your provider. Example pinnings:

- **Anthropic:** workhorse `claude-sonnet-4-6`, cheap `claude-haiku-4-5`.
- **OpenAI:** workhorse `gpt-4.1-mini`, cheap `gpt-4.1-nano`. (If your account exposes different IDs, pin whatever your account can call.)

Record the two IDs at the top of a new file `compare.py`:

```python
WORKHORSE = "claude-sonnet-4-6"
CHEAP     = "claude-haiku-4-5"
```

### Checkpoint

- Do both IDs appear verbatim in your provider's current models page? If one is deprecated, swap it before continuing.
- Are the two tiers from the **same provider**? Cross-provider comparisons are useful but out of scope for this lab — adapters differ and will muddy the comparison.

## Step 2 — Build a fixed test set of ten prompts

Chapter 5 recommends "a fixed test set of 10–50 realistic examples." Ten is enough to feel real signal here. Mix easy and hard so you see where the cheap tier breaks down.

Create `prompts.txt`, one prompt per line. Suggested mix:

1. "In one sentence, define 'idempotency' in HTTP."
2. "Classify this review as positive, negative, or neutral: 'It arrived late but worked fine.'"
3. "Extract the email addresses from: 'contact ada@example.com or ops+paging@example.co.uk'. Return a JSON array."
4. "Write a Python function `is_prime(n: int) -> bool` with a short docstring. Code only."
5. "Translate to French: 'The context window is shared between input and output.'"
6. "Given temperature=0 and the same prompt, is identical output guaranteed? Answer in two sentences."
7. "Rewrite this run-on sentence into two: 'The server is stateless so the client must resend history on every call because the model has no memory of the previous turn.'"
8. "What is the time complexity of binary search on a sorted array? Answer in one sentence."
9. "Produce a 3-item bulleted list of common mistakes when assembling chat messages."
10. "Summarize in one sentence: a chat API call is stateless, token-metered, and samples one token at a time from a probability distribution."

Keep each prompt under ~40 words. We want quality signal, not a long-context test.

## Step 3 — Run both tiers and log usage

Save as `compare.py` (Anthropic shown; the OpenAI variant is in Step 3b):

```python
import csv
import os
import time

from anthropic import Anthropic

WORKHORSE = "claude-sonnet-4-6"
CHEAP     = "claude-haiku-4-5"

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def one_call(model: str, prompt: str) -> dict:
    t0 = time.monotonic()
    r = client.messages.create(
        model=model,
        max_tokens=256,
        temperature=0,
        messages=[{"role": "user", "content": prompt}],
    )
    t1 = time.monotonic()
    return {
        "model":        r.model,
        "input_tokens": r.usage.input_tokens,
        "output_tokens": r.usage.output_tokens,
        "stop_reason":  r.stop_reason,
        "wall_s":       round(t1 - t0, 3),
        "text":         r.content[0].text,
    }

with open("prompts.txt") as f, open("compare.csv", "w", newline="") as out:
    w = csv.writer(out)
    w.writerow(["prompt_idx", "tier", "model", "input_tokens",
                "output_tokens", "stop_reason", "wall_s", "reply"])
    for i, line in enumerate(f):
        prompt = line.strip()
        if not prompt:
            continue
        for tier, model in (("workhorse", WORKHORSE), ("cheap", CHEAP)):
            r = one_call(model, prompt)
            w.writerow([i, tier, r["model"], r["input_tokens"],
                        r["output_tokens"], r["stop_reason"],
                        r["wall_s"], r["text"].replace("\n", "\\n")])
            print(f"[{i}][{tier}] {r['wall_s']}s  in={r['input_tokens']} out={r['output_tokens']}  "
                  f"stop={r['stop_reason']}  → {r['text'][:60]!r}")
```

### Step 3b — OpenAI variant

Swap the imports/clients and the response access:

```python
from openai import OpenAI
client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

def one_call(model: str, prompt: str) -> dict:
    t0 = time.monotonic()
    r = client.chat.completions.create(
        model=model,
        max_tokens=256,
        temperature=0,
        messages=[{"role": "user", "content": prompt}],
    )
    t1 = time.monotonic()
    return {
        "model":         r.model,
        "input_tokens":  r.usage.prompt_tokens,
        "output_tokens": r.usage.completion_tokens,
        "stop_reason":   r.choices[0].finish_reason,
        "wall_s":        round(t1 - t0, 3),
        "text":          r.choices[0].message.content,
    }
```

Run it:

```bash
python compare.py
```

### Expected output

Twenty log lines on stdout, pairs of `[workhorse]` / `[cheap]` per prompt, plus `compare.csv` with 20 rows.

### Checkpoint

- Does `output_tokens` ever equal 256 (your `max_tokens`)? If so, that row was truncated — the comparison is unfair; raise `max_tokens` or ask for a shorter answer, as chapter 4 directs.
- Is `temperature=0` the same across both tiers? (Chapter 5: compare at your deterministic setting first so run-to-run noise does not confound tier differences.)

## Step 4 — Score the replies by hand

Chapter 5 is blunt: "Score them — even by hand, even just 'good/ok/bad.'" Open `compare.csv` in a spreadsheet or a plain editor. Add a `score` column with values in {`good`, `ok`, `bad`} for each row. Judge each reply by:

- Did it answer the question?
- Did it follow the format constraint (one sentence, JSON array, code only, etc.)?
- Did it include wrong facts?

Fill in all 20 rows. Resist tie-breaking with the model's "polish" — a wordier answer that is also correct is still correct.

### Checkpoint — score tally

Count by tier:

```
             good   ok   bad
workhorse      ?     ?    ?
cheap          ?     ?    ?
```

If `cheap` is `good + ok` on at least 8/10 prompts and the `bad` rows aren't on the ones you actually care about, chapter 5's recommendation is to ship the cheap tier. If the cheap tier fails on prompts you need to handle, note *which* kinds of prompt — classification, extraction, code, free-form — and move on.

## Step 5 — Translate numbers into a cost projection

Still in `compare.csv`, compute the token totals per tier:

```
workhorse_input  = sum(input_tokens  where tier=workhorse)
workhorse_output = sum(output_tokens where tier=workhorse)
cheap_input      = sum(input_tokens  where tier=cheap)
cheap_output     = sum(output_tokens where tier=cheap)
```

Then look up the **current** per-million-token input and output prices for each pinned model ID on your provider's pricing page. Chapter 7 is explicit: do not memorize prices — look them up at the time you need them. In a new file `projection.md`, record:

- The two model IDs and the pricing-page URL + today's date.
- The cost of your 10-prompt run for each tier using the formula from chapter 7:
  `$ ≈ (input_tokens × $in + output_tokens × $out) / 1_000_000`.
- The projected daily cost at 100,000 calls/day assuming your 10-prompt average `(input, output)` per call. (Chapter 7: multiply by traffic.)
- A one-line note on wall-time medians: which tier is faster on this traffic shape? Chapter 5 warns that cheap is usually faster, "but not always" — your numbers may surprise you.

### Checkpoint

- Is the cheap tier's dollar total actually lower on these ten calls? It should be — if not, re-check that you are hitting the pinned model IDs (the `model` column in `compare.csv` is what the server reports).
- Is the latency median under your target? If you have no target, "under 2 seconds for a 256-token reply" is a reasonable interactive bar.

## Step 6 — Write your "which tier wins" paragraph

In `projection.md`, add a 4–6 sentence decision:

- Which tier would you ship for this test set, and why?
- Which prompts (by index) would you route to the heavier tier if you added a classifier (chapter 5's routing pattern)?
- What would you change about the test set to make this decision more defensible — more prompts, more adversarial prompts, prompts drawn from real users?

That paragraph is the deliverable. The CSV and projection support it; without the paragraph you have run a benchmark but not made a choice.

## Debrief

You have now walked chapter 5's practical sequence end-to-end on real data:

1. Pinned exact model IDs for two tiers.
2. Evaluated on a fixed test set at `temperature=0`.
3. Measured tokens and wall time per call.
4. Projected cost from measured `usage`, with current prices cited.
5. Made a tier decision in writing.

Keep `compare.py`, `compare.csv`, and `projection.md` around — the same shape works for every "which model should we use?" question you will get in later modules.

## Stretch

- Add a third tier (flagship) and re-score. Do any `bad` rows from the cheap tier become `good` only at flagship? Those are the prompts a router should escalate.
- Add a per-prompt latency column and compute p50 / p95 wall time per tier. Chapter 7 lists this instrumentation as the baseline you want in production.
- Replace one of the ten prompts with a longer input (e.g., 2,000 tokens of pasted documentation). Which tier's wall time grows more, and by how much? Chapter 7's "prefill scales with input length" prediction should hold.
