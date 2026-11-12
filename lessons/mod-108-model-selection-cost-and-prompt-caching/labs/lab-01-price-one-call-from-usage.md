# Lab 01 — Price One Feature Call From the `usage` Block

**Estimated duration:** 25–40 minutes
**Covers:** [chapter 2 — Cost per feature call and enforcing token budgets](../02-cost-per-call-and-token-budgets.md) ("Cost per call, done properly" and "Logging cost, not just tokens")

## Objective

Chapter 2 is blunt: "Cost from `usage`, not from the pricing page." A spreadsheet of sticker prices is an estimate; the `usage` block on a real response is the authoritative count the provider will bill against. In this lab you will make **one real API call**, read the `usage` block, look up the current per-million-token input and output prices on the provider's pricing page, and compute the exact dollar cost of the call by hand. Then you will do it a second time with `max_tokens` raised 10× to feel how the sticker worst-case moves while the actual cost barely does.

No forecasting. No budget enforcement. No sweep over models. One call, one usage block, one dollar number — the smallest honest atom of LLM cost accounting.

By the end you will have:

- A single-call Python script (`price_one_call.py`) that prints the `usage` fields and the derived cost.
- The pricing-page URL and the exact `$/MTok` numbers you fetched, pasted into a comment at the top of the script.
- A `NOTES.md` with the four dollar numbers you computed and one paragraph on which of them would appear on an invoice.

> This lab complements [exercise-01](../exercises/exercise-01-cost-per-feature-call-estimate.md), which is the full forecasting + per-request / per-user budget-enforcement drill at 50,000 calls/day. Here we stay with a single call, so the mechanics of reading `usage` and computing cost from it are sharpened before the exercise bundles them with volume projection.

## Prerequisites

- Finished chapter 2.
- A funded API key for one provider:
  - **Anthropic path:** `ANTHROPIC_API_KEY` exported, `pip install anthropic`.
  - **OpenAI path:** `OPENAI_API_KEY` exported, `pip install openai`.
- Budget: two short calls. Fractions of a cent.
- Pick **one** provider path for this lab. The stretch covers the other one.

## Step 1 — Look up today's input and output prices

Open the pricing page for your chosen provider and find the row for a current small-tier model (e.g., a recent Haiku or a `-mini` tier). Copy the `$/MTok input` and `$/MTok output` numbers. Note the URL and today's date.

- Anthropic: <https://www.anthropic.com/pricing>
- OpenAI: <https://openai.com/api/pricing/>

<!-- needs-research: the exact per-million-token input and output prices for the current Anthropic Haiku and OpenAI mini tiers as of 2026-11-12 — paste the numbers from the live pricing page, do not guess. -->

### Checkpoint

- Can you point to the two numbers on the page — `$/MTok input` and `$/MTok output` — and read them without ambiguity? If the page shows a per-token price instead, multiply by 1,000,000 to normalise.
- Did you also note the **cached input** price? You will not use it in this lab (that is chapter 3), but chapter 2's `cost_per_call` formula breaks it out as a separate line item, and you will want the number on hand.

## Step 2 — Make one call and dump the `usage` block

Save as `price_one_call.py`. Paste the two prices at the top of the file, with the URL and today's date, so your cost math cites its source.

```python
# Pricing fetched 2026-11-12 from https://www.anthropic.com/pricing
# (substitute today's date and the row you copied).
P_IN_PER_MTOK  = 0.00  # <--- fill in from the pricing page
P_OUT_PER_MTOK = 0.00  # <--- fill in from the pricing page

MODEL = "claude-haiku-4-5-20251001"  # or your chosen small-tier model
```

Then the Anthropic path:

```python
import os

from anthropic import Anthropic

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

response = client.messages.create(
    model=MODEL,
    max_tokens=150,
    messages=[
        {
            "role": "user",
            "content": (
                "Classify the sentiment of the following product review as "
                "'positive', 'neutral', or 'negative'. Reply with a single JSON "
                "object of shape {\"sentiment\": \"...\"} and nothing else.\n\n"
                "Review: The battery life is fine, nothing special, nothing bad."
            ),
        }
    ],
)

u = response.usage
input_tokens  = u.input_tokens
output_tokens = u.output_tokens

cost_input  = input_tokens  * P_IN_PER_MTOK  / 1_000_000
cost_output = output_tokens * P_OUT_PER_MTOK / 1_000_000
cost_total  = cost_input + cost_output

print(f"model:         {MODEL}")
print(f"input_tokens:  {input_tokens}")
print(f"output_tokens: {output_tokens}")
print(f"cost_input:    ${cost_input:.8f}")
print(f"cost_output:   ${cost_output:.8f}")
print(f"cost_total:    ${cost_total:.8f}")
print(f"response:      {response.content[0].text!r}")
```

Or the OpenAI path:

```python
import os

from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

MODEL = "gpt-4.1-mini"  # or your chosen small-tier model

response = client.chat.completions.create(
    model=MODEL,
    max_tokens=150,
    messages=[
        {
            "role": "user",
            "content": (
                "Classify the sentiment of the following product review as "
                "'positive', 'neutral', or 'negative'. Reply with a single JSON "
                "object of shape {\"sentiment\": \"...\"} and nothing else.\n\n"
                "Review: The battery life is fine, nothing special, nothing bad."
            ),
        }
    ],
)

u = response.usage
input_tokens  = u.prompt_tokens
output_tokens = u.completion_tokens

cost_input  = input_tokens  * P_IN_PER_MTOK  / 1_000_000
cost_output = output_tokens * P_OUT_PER_MTOK / 1_000_000
cost_total  = cost_input + cost_output

print(f"model:         {MODEL}")
print(f"input_tokens:  {input_tokens}")
print(f"output_tokens: {output_tokens}")
print(f"cost_input:    ${cost_input:.8f}")
print(f"cost_output:   ${cost_output:.8f}")
print(f"cost_total:    ${cost_total:.8f}")
print(f"response:      {response.choices[0].message.content!r}")
```

Run it:

```bash
python price_one_call.py
```

### Expected output (shape)

Five numbers and a one-line response. The input token count will be larger than the output; the output cost per token is ~3–5× the input cost per token, so even with far fewer output tokens, `cost_output` is often a sizeable fraction of `cost_total`.

```
model:         claude-haiku-4-5-20251001
input_tokens:  58
output_tokens: 14
cost_input:    $0.00005800
cost_output:   $0.00005600
cost_total:    $0.00011400
response:      '{"sentiment": "neutral"}'
```

(Your numbers will differ with the pricing you pasted and the model you chose. What matters is the shape.)

### Checkpoint

- Did `cost_output` come out within an order of magnitude of `cost_input`, despite `output_tokens` being much smaller than `input_tokens`? If yes, you are feeling chapter 2's "output tokens are typically 3–5× more expensive than input tokens on the same model" in a single call.
- Does `cost_total` round down to zero when you look at it as cents? Welcome to LLM cost: an individual call is nearly free. The whole module exists because 50,000 of them a day is not.
- Did you compute cost from `response.usage`, or did you estimate input tokens with `len(prompt) / 4`? Chapter 2's "Common bugs" is explicit — only `usage` is authoritative.

## Step 3 — Raise `max_tokens` 10× and watch the sticker move

Change `max_tokens` from `150` to `1500` and re-run. Everything else stays the same.

```bash
python price_one_call.py
```

Record the new `output_tokens` and `cost_total`. Two observations the chapter predicts:

1. The *actual* `output_tokens` on the response probably barely moved — the model finished with the same ~14-token JSON. You raised the **cap**, not the output.
2. The *billed* `cost_total` is essentially unchanged. The provider charges for what it generated, not for the cap.

But the **worst-case** your per-request budget would have to reserve (input + `max_tokens`) went up 10×. Chapter 2's per-request-budget skeleton (and exercise 01's `enforce_request_budget`) reserves `input + max_output_tokens`, not actual output, precisely because output size is unknown at enforcement time.

In `NOTES.md`, record both runs side by side:

```
run 1: max_tokens=150,  input=58,  output=14,  cost=$0.00011400,  reserved_worst_case_tokens=208
run 2: max_tokens=1500, input=58,  output=14,  cost=$0.00011400,  reserved_worst_case_tokens=1558
```

### Checkpoint

- Can you articulate, in one sentence, the difference between the **billed** cost (fixed by actual `usage`) and the **reserved worst case** (fixed by `input + max_tokens`) — and which one your per-user budget counter has to reason about *before* the call is placed?
- Chapter 2 says this is the reason enforcement reserves worst-case and reconciles down to actual later. Does your `NOTES.md` have one sentence linking the two runs to that rule?

## Step 4 — Scale the single-call number to a volume projection

In `NOTES.md`, extend the table with the arithmetic from chapter 2's "A worked example":

```
calls/day        10,000
cost/call        $0.00011400   (from run 1 above)
daily  cost      $...
monthly cost     $...  (daily * 30)
```

Then compare against what you would have paid at the **frontier** tier for the same call. Open the pricing page again and read the input and output prices for the current frontier model (e.g., Opus or the `gpt-*` flagship). Compute `cost/call` as if you had sent the same `(input_tokens, output_tokens)` through that tier, and scale the same way:

```
calls/day              10,000
cost/call frontier     $...
daily  cost frontier   $...
monthly cost frontier  $...
ratio frontier / small ...x
```

This single multiplication is the "10–30× cheaper per token" claim chapter 1 (and chapter 2's motivation) rests on — made concrete with the prices you just read and the token counts you just measured.

### Checkpoint

- The "10–30×" range is what the chapters claim as typical. Is yours in that range? If it is well outside, double-check the two pricing numbers you pasted — small-tier vs. frontier-tier on the same provider's same generation.
- If the monthly-frontier number is one you would be nervous to put in front of a budget approver, that is exactly the motivation the module is built on. Save the number.

## Step 5 — Write a 5-sentence debrief

In `NOTES.md`, answer in 5–7 sentences total:

1. For your single call, what was the ratio `cost_output / cost_input`? Chapter 2 names a rough band — does yours fall in it?
2. When you raised `max_tokens` 10×, which of these moved: `input_tokens`, `output_tokens`, `cost_total`, `reserved_worst_case_tokens`? Which did **not**? Chapter 2's per-request-budget section hangs on this distinction — restate it in your own words.
3. If a teammate pointed at the pricing page and said "this call will cost $X," what would you ask them to compute instead? (Chapter 2 answers this in one sentence — "Cost from `usage`, not from the pricing page.")
4. For the 10,000-calls/day projection, which of the numbers (small-tier monthly, frontier-tier monthly, ratio) would you put at the top of a design-review doc, and why?

That write-up is the deliverable. Commit `price_one_call.py` and `NOTES.md`.

## Debrief

You have now closed the smallest honest loop in the module: real call → `usage` block → pasted pricing numbers → dollar amount. Three habits pay forward from here:

- When a feature's cost looks surprising, dump `usage` on a representative call before touching anything else. Guessing input tokens from `len(prompt) / 4` or output tokens from "it felt about a paragraph" will mislead you.
- When you change `max_tokens`, you are changing the worst case your budget layer has to reserve — not the typical bill. Chapter 2's reserve-then-reconcile pattern (exercise 01 drills it) exists because the two numbers diverge.
- When you write a design-review doc, cite the pricing page URL and the date you fetched it. Prices change; the citation is what lets the next person audit your math six months later.

## Stretch

- **Second provider on the same question.** Run the lab against the other provider with roughly comparable small-tier and frontier models. Does the input / output price asymmetry sit in the same 3–5× band? Does the small-vs-frontier ratio?
- **Cached input, line-item.** If your prompt is long enough and you make the same call twice in quick succession, the second call may show cached input tokens in `usage` (Anthropic needs an explicit `cache_control` block; OpenAI caches automatically above a prompt-size threshold). Extend `price_one_call.py` to read `cache_read_input_tokens` / `prompt_tokens_details.cached_tokens` and compute cost with the three-line-item formula from chapter 2:
  ```
  cost = fresh_input * P_in_fresh + cached_input * P_in_cached + output * P_out
  ```
  You are doing a mini-preview of chapter 3 and exercise 03.
- **Reasoning tokens, if your model has them.** Chapter 2's "Nuance 3" warns that enabling extended thinking adds internal deliberation tokens that are billed as output. On a model that supports it (e.g., an Anthropic extended-thinking variant), re-run with thinking enabled and watch `cost_output` and the response's reasoning-token field. Chapter 2: "A prompt that reasons for 2000 tokens before producing 100 tokens of answer costs the same as one that produces 2100 tokens of answer."
- **Log one row to JSONL.** Append a single line of `{"ts", "model", "input_tokens", "output_tokens", "cached_input_tokens", "cost_total", "feature"}` to a `calls.jsonl` file. You now have the seed of the per-request cost log chapter 2's "Logging cost, not just tokens" section recommends.
