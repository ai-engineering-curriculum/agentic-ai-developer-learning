# Lab 01 — Prefill Forces JSON Without a JSON Mode

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 2 — Anatomy of an Effective Prompt](../02-anatomy-of-an-effective-prompt.md) ("Where each block goes" / assistant prefill), [chapter 5 — Strategies for Reliable Structured Output](../05-structured-output-strategies.md) ("Strategy 1: prompt-only" and prefill)

## Objective

Chapter 5 claims that **prefilling the assistant turn with `{`** is a fast, nearly-free trick that eliminates preambles ("Sure! Here's the response:") and code fences for short, fixed-shape JSON outputs — without buying into a vendor JSON mode or a schema. In this lab you will prove it on a single provider by running the *same* prompt twice: once without prefill, once with prefill, over a tiny input set, and counting how often each path produces output you can `json.loads` on the first try.

By the end you will have:

- A `prefill_demo.py` that calls Anthropic (or any provider that supports a leading assistant turn) once with prefill and once without, for 8 inputs.
- A `results.csv` with a boolean per call for "parsed on first try," plus the raw reply so you can see the preamble/fence when they appear.
- A one-line conclusion about when prefill is the right escalation step.

> This lab complements [exercise-02](../exercises/exercise-02-structured-json-output.md), which compares *all four* structured-output strategies on a bigger input set. Here we isolate prefill alone, so you can see what that single move earns.

## Prerequisites

- Finished chapters 2 and 5.
- A funded Anthropic API key: `ANTHROPIC_API_KEY` exported.
- Python 3.10+ with the SDK installed: `pip install anthropic`.
- Budget note: 16 short calls at `max_tokens=256`. This should cost cents.

> If your provider does not accept a leading assistant turn, prefill is not available there — skip to [lab 02](lab-02-scope-constrained-prompt.md). Chapter 5's prefill example is Anthropic-specific.

## Step 1 — Pin the model and write the system prompt

Pick an exact dated model ID (chapter 1 — "Pin the surroundings"). The examples below pin `claude-sonnet-4-6`; substitute whatever ID your account can call.

The task is the triage shape from chapter 2, deliberately kept small so the output is short enough that preamble incidents are easy to spot:

```python
SYSTEM = """You are a triage assistant for an e-commerce support team.

Classify the incoming message. Reply with a single JSON object and nothing else.

Output format (JSON):
{
  "category": "billing" | "shipping" | "returns" | "product" | "other" | "unknown",
  "urgency":  "low" | "normal" | "high",
  "summary":  "<one sentence, <= 140 chars>"
}
"""
```

Do **not** add "Do not wrap in a code fence." or "Do not include prose." in this lab — the point is to observe how often the model does that unprompted, so prefill has something to fix.

### Checkpoint

- The system prompt names the enum values literally.
- The system prompt does not include negative constraints that would mask the behaviour prefill is supposed to fix.

## Step 2 — Build the 8-input set

Create `inputs.txt` with eight short messages. A reasonable mix:

```text
Please refund order #4421, the box arrived crushed.
Hi, when will my order ship? It's been a week.
I was charged twice for the same purchase yesterday — please fix this asap.
Do you carry the blue color in size medium?
The website keeps logging me out. I'm trying to check out.
The replacement headset still doesn't work after the firmware update.
Where is my order!!!!!!

```

The last line is a deliberately empty input — a known-weird case that often triggers chatty refusals.

### Checkpoint

- Eight non-blank lines plus the trailing empty line (eight *inputs* total; the empty one counts).
- No two inputs are near-duplicates (chapter 3's "example contamination" warning — not what this lab is testing, but keeping inputs varied avoids confounding).

## Step 3 — Call once without prefill, once with prefill

Save as `prefill_demo.py`:

```python
import csv
import json
import os
import re
import time

from anthropic import Anthropic

MODEL = "claude-sonnet-4-6"

SYSTEM = """You are a triage assistant for an e-commerce support team.

Classify the incoming message. Reply with a single JSON object and nothing else.

Output format (JSON):
{
  "category": "billing" | "shipping" | "returns" | "product" | "other" | "unknown",
  "urgency":  "low" | "normal" | "high",
  "summary":  "<one sentence, <= 140 chars>"
}
"""

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def call(message: str, *, prefill: bool) -> dict:
    messages = [{"role": "user", "content": message}]
    if prefill:
        messages.append({"role": "assistant", "content": "{"})

    t0 = time.monotonic()
    r = client.messages.create(
        model=MODEL,
        max_tokens=256,
        temperature=0,
        system=SYSTEM,
        messages=messages,
    )
    wall = round(time.monotonic() - t0, 3)

    raw = r.content[0].text
    body = ("{" + raw) if prefill else raw

    # First-try parse (no fallback helper on purpose — we want to see raw failures).
    try:
        parsed = json.loads(body)
        parsed_ok = True
    except json.JSONDecodeError:
        parsed = None
        parsed_ok = False

    has_fence = bool(re.search(r"```", raw))
    has_preamble = bool(re.match(r"\s*[A-Za-z]", raw)) and not prefill
    # (With prefill, the first visible token should be '"' or whitespace inside the object body, not a letter.)

    return {
        "parsed_ok": parsed_ok,
        "has_fence": has_fence,
        "has_preamble": has_preamble,
        "wall_s": wall,
        "stop_reason": r.stop_reason,
        "raw": raw,
    }

with open("inputs.txt") as f, open("results.csv", "w", newline="") as out:
    w = csv.writer(out)
    w.writerow(["idx", "variant", "parsed_ok", "has_fence", "has_preamble",
                "wall_s", "stop_reason", "raw"])
    for i, line in enumerate(f):
        msg = line.rstrip("\n")
        for variant, prefill in (("no_prefill", False), ("prefill", True)):
            r = call(msg, prefill=prefill)
            w.writerow([i, variant, r["parsed_ok"], r["has_fence"],
                        r["has_preamble"], r["wall_s"], r["stop_reason"],
                        r["raw"].replace("\n", "\\n")])
            flag = "OK " if r["parsed_ok"] else "FAIL"
            print(f"[{i}][{variant}] {flag}  fence={r['has_fence']} "
                  f"preamble={r['has_preamble']}  → {r['raw'][:60]!r}")
```

Run it:

```bash
python prefill_demo.py
```

### Expected output (shape)

Sixteen log lines, alternating `[i][no_prefill]` / `[i][prefill]` for `i` in `0..7`, plus `results.csv` with 16 rows. For the typical dated snapshot of Sonnet-tier Claude, you should see:

- The `no_prefill` rows occasionally include a leading preamble ("Sure, here's the triage:") or wrap the JSON in a ```` ```json ```` fence. First-try `json.loads` fails on those.
- The `prefill` rows start straight into the object body. `{` + raw parses on the first try.

<!-- needs-research: exact rate at which modern Sonnet-tier Claude emits preambles / code fences on a bare prompt-only JSON request; the chapters make a qualitative claim ("prevents the preamble and the code fence in one move") but do not pin a number. Students should report the number they observe, not a canned expectation. -->

### Checkpoint

Open `results.csv` and answer:

1. How many of the 8 `no_prefill` rows had `parsed_ok = True`? How many `prefill` rows?
2. On any `no_prefill` row that failed, what does `raw` show — code fence, preamble, trailing commentary, something else?
3. On the empty-input row, did the two variants behave differently? (Chapter 2 warned about "chatty refusals on ambiguous input" — this is one place you can see it.)

## Step 4 — Trip the one failure mode prefill still cannot fix

Chapter 5 is explicit: prefill does **not** guarantee schema correctness, only that the reply starts with `{`. Prove it to yourself. Modify the system prompt to add a field prefill cannot force:

```python
SYSTEM = """... as before ...

Also include a field "escalate" that is one of: "yes", "no", "maybe".
"""
```

Re-run only the `prefill` variant on the 8 inputs. Watch for:

- JSON that parses but omits `escalate`.
- JSON that parses but uses a value outside the enum (e.g., `"escalate": "unsure"`).
- JSON that parses but uses the wrong casing (`"Yes"`).

Record one example of each in a short `NOTES.md`. These are the exact failures that would motivate escalating to **schema-constrained generation** or **tool-calling for structured output** (chapter 5, strategies 3 and 4).

### Checkpoint

- Did at least one `prefill` call produce *valid JSON* that was nonetheless *wrong shape* under the new field? That is the "valid-but-wrong" case chapter 6 names and the reason validation at the boundary is still required even after prefill.

## Step 5 — Write a one-paragraph conclusion

In `NOTES.md`, answer in 4–6 sentences:

- How many parse-on-first-try wins did prefill give you across 8 inputs?
- What specific raw-output pathologies did prefill eliminate (preamble, fence, both)?
- Where did prefill *not* help (the "valid-but-wrong" case from Step 4)?
- Given that result, when would you reach for prefill in a real app, and when would you go straight to schema-constrained generation (chapter 5, strategy 3) or tool calling (strategy 4)?

That paragraph is the deliverable.

## Debrief

Prefill is the smallest, cheapest escalation step from "ask for JSON and hope" toward a real structured-output strategy. It lands on the input side of the call — a single `assistant` turn with `{` — and it costs essentially nothing. What it fixes is narrow (preambles and fences for short fixed-shape outputs); what it does not fix is anything about *shape* or *values*. That is why chapter 5 orders it inside Strategy 1 (prompt-only), not alongside Strategies 3 and 4.

Keep this pattern. Whenever a short-reply JSON prompt is "mostly fine" except that half the responses have a leading sentence, prefill is the move. For anything you ship under real traffic, you still want JSON mode, schema-constrained, or tool calling (exercise-02), and you still want a typed validator at the boundary (chapter 6, exercise-03).

## Stretch

- Add a `no_prefill_with_negative_constraints` variant: keep the plain prompt-only shape but add explicit lines like "Do not include prose outside the JSON. Do not wrap the output in a code fence." Does negative-constraint phrasing match prefill's parse rate, or lag behind? Chapter 2 names "negative constraints" as one of the three types that earn their tokens.
- Repeat Step 3 at `temperature=0.7` and run each input three times per variant. Count the parse-rate per variant across the 24 calls. Chapter 1's "'Deterministic' sampling" warning applies: at higher temperature, prompt-only JSON failures are *more* frequent, which should widen prefill's lead.
- Swap in your provider's actual JSON mode (OpenAI: `response_format={"type": "json_object"}`; see chapter 5 strategy 2) and compare its parse rate to the prefill variant on the same inputs. This is the natural next step on the escalation ladder from Strategy 1 → Strategy 2.
