# Lab 03 — Calibrate an LLM-as-Judge Against Human Scores

**Estimated duration:** 60–75 minutes
**Covers:** [chapter 7 — Iterating on Prompts with Simple Evals](../07-iterating-with-evals.md) ("LLM-as-judge: when and how to use it")

## Objective

Chapter 7 says the LLM-as-judge is a tool of last resort for things you cannot deterministically check — and it only earns its tokens if you *calibrate* it. "Calibrate" is a verb with teeth: score 10–20 cases by hand, run the judge against the same cases, and look at where the two scores disagree. Without that step, an LLM judge is a confident number with no idea what it means.

In this lab you will build a tiny calibration harness around one freeform task — summarizing short customer complaints — and measure the agreement between your hand scores and the judge's scores. You will then tighten the rubric once and re-measure, so you can see the rubric change the agreement number.

By the end you will have:

- A `human_scores.csv` with 10 (input, summary, human score) triples you graded yourself.
- A `judge.py` that scores the same 10 pairs with an LLM-as-judge and a versioned rubric.
- A `calibration.md` showing agreement for `rubric_v1` and `rubric_v2`, plus one-line notes on where they diverge.

> This lab complements [exercise-03](../exercises/exercise-03-output-validation.md), which uses *deterministic* checks (Pydantic + a frozen golden set) to gate structured output. Here we address the orthogonal problem: *freeform* output whose quality is a judgment call. The two patterns coexist in a real eval suite.

## Prerequisites

- Finished chapter 7.
- A funded API key for your provider. The lab uses Anthropic for the judge, but any provider works; chapter 7 explicitly notes you should use a *different* model (or at least a different prompt) for the judge than for the generator, to avoid same-model bias.
  - `pip install anthropic` (or `openai`) and `pip install pyyaml`.
- Python 3.10+.
- Budget: 10 "generate summary" calls + 20 "judge" calls = 30 calls. Cents.

## Step 1 — Produce 10 summaries to score

Create `summaries.yaml` with 10 short cases. Half come from short, factual customer messages; half come from longer, more opinionated ones. You can reuse `inputs.txt` from exercise-01 as a source for the messages.

Shape:

```yaml
# summaries.yaml
- id: refund-damaged-001
  message: "Please refund order #4421, the box arrived crushed."
  summary: "Customer requests refund for order 4421 after damaged shipment."
- id: double-charge-001
  message: "I was charged twice for the same purchase yesterday — please fix this asap."
  summary: "Customer reports double charge yesterday and wants it fixed."
- id: angry-002
  message: "Where is my order!!!!!!"
  summary: "Short, high-urgency inquiry about order status."
- id: polite-praise-003
  message: "Hi! Just wanted to say I love the new packaging."
  summary: "Customer compliments the new packaging."
# ... six more
```

Either hand-write the `summary` field for each case, or generate it with a cheap model call at `temperature=0.2`:

```python
from anthropic import Anthropic
client = Anthropic()
def summarize(msg: str) -> str:
    r = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=80,
        temperature=0.2,
        system="Summarize the customer message in one sentence suitable for a ticket title (<= 140 chars).",
        messages=[{"role": "user", "content": msg}],
    )
    return r.content[0].text.strip()
```

Important: for calibration you want some summaries to be *bad*. Deliberately hand-write 2 bad ones — one that fabricates a detail ("The customer confirmed they received a replacement"), one that is empty or two words long. Chapter 7's "cover the failure modes" guidance applies here: a judge calibrated only on good outputs will tell you nothing on the day one is bad.

### Checkpoint

- `summaries.yaml` has exactly 10 entries, each with `id`, `message`, `summary`.
- At least 2 of the 10 summaries are deliberately bad in different ways (fabrication, truncation, wrong focus).

## Step 2 — Score the 10 summaries by hand

Create `human_scores.csv` with columns `id,helpful,accurate,tone,notes`. Score each on three 1–5 axes drawn from chapter 7's judge example:

- **helpful** (1–5) — does it capture the actionable gist of the message?
- **accurate** (1–5) — does every claim come from the message (no fabrications)?
- **tone** (1–5) — appropriate for a ticket title in a support tool?

Fill in all 10 rows. Add a one-line `notes` field for anything that was close to a boundary — those are the rows the judge is most likely to disagree with you on.

### Checkpoint

- All 10 rows have numeric scores. No blanks.
- The two deliberately-bad summaries scored low on at least one axis (otherwise your calibration will be meaningless; the point of the bad ones is to put a floor under the judge's discrimination).
- You wrote the scores before looking at any model output.

## Step 3 — Write `rubric_v1` and run the judge

Create `judge.py` with a versioned rubric. `rubric_v1` is the "bad on purpose" starting point — vague and non-anchored, the kind of rubric chapter 7 warns against.

```python
import csv
import os
import yaml

from anthropic import Anthropic

JUDGE_MODEL = "claude-sonnet-4-6"   # Different from the generator (chapter 7: avoid same-model bias).

RUBRIC_V1 = """You are grading summaries of customer support messages.

Score each summary on three dimensions:
- helpful (1-5): is the summary helpful?
- accurate (1-5): is the summary accurate?
- tone (1-5): is the tone appropriate?

Reply with JSON: {"helpful": int, "accurate": int, "tone": int, "notes": str}
"""

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def judge(rubric: str, message: str, summary: str) -> dict:
    user = f"Message: {message}\nSummary: {summary}"
    r = client.messages.create(
        model=JUDGE_MODEL,
        max_tokens=256,
        temperature=0,
        system=rubric,
        messages=[
            {"role": "user", "content": user},
            {"role": "assistant", "content": "{"},
        ],
    )
    import json
    return json.loads("{" + r.content[0].text)

cases = yaml.safe_load(open("summaries.yaml").read())

with open("judge_v1.csv", "w", newline="") as out:
    w = csv.writer(out)
    w.writerow(["id", "helpful", "accurate", "tone", "notes"])
    for c in cases:
        s = judge(RUBRIC_V1, c["message"], c["summary"])
        w.writerow([c["id"], s["helpful"], s["accurate"], s["tone"], s.get("notes", "")])
        print(f"[{c['id']}] helpful={s['helpful']} accurate={s['accurate']} tone={s['tone']}")
```

Run it:

```bash
python judge.py
```

### Expected output

Ten log lines and `judge_v1.csv` with 10 rows. The scores will be numeric; whether they *agree* with your scores is the thing you are about to measure.

### Checkpoint

- All 10 rows parsed as JSON. If any row failed to parse, re-check the prefill (chapter 5 strategy 1) — this is exactly the preamble/fence case lab 01 covered.

## Step 4 — Measure agreement

In `calibration.md`, compute two numbers per axis:

- **Exact-match rate** — fraction of cases where the judge's score equals your score on that axis.
- **Within-one rate** — fraction of cases where the judge's score is within ±1 of yours.

A tiny helper:

```python
import csv

def load(path):
    return {row["id"]: {k: int(row[k]) for k in ("helpful", "accurate", "tone")}
            for row in csv.DictReader(open(path))}

h = load("human_scores.csv")
j = load("judge_v1.csv")

for axis in ("helpful", "accurate", "tone"):
    diffs = [j[k][axis] - h[k][axis] for k in h]
    exact = sum(1 for d in diffs if d == 0) / len(diffs)
    within1 = sum(1 for d in diffs if abs(d) <= 1) / len(diffs)
    print(f"{axis:9s}  exact={exact:.2f}  within1={within1:.2f}  diffs={diffs}")
```

Add the three lines to `calibration.md` under a heading `rubric_v1`. Also note the per-case `id`s where the diff was ≥ 2 — these are the cases the rubric is misreading and the targets for your next edit.

### Checkpoint

- Three numbers per axis land in `calibration.md`.
- You can name at least one case whose human and judge scores disagreed by ≥ 2. That is the specific failure the next step is going to fix.

## Step 5 — Tighten the rubric and re-run

Rewrite the rubric with **anchored** scoring (chapter 7: "Make rubrics specific. 'Is this good?' is useless. 'Is the answer supported verbatim by a sentence in the source?' is testable.").

```python
RUBRIC_V2 = """You are grading short summaries of customer support messages.

For each (message, summary) pair, score three axes. Anchor each number to a concrete criterion. Err low; a 5 is rare.

- helpful (1-5): does the summary surface the actionable intent of the message?
  1 = says nothing about intent (empty, or purely descriptive like "a customer wrote in").
  3 = names the topic (billing / shipping / returns / product) but omits the specific action requested.
  5 = names the specific action requested AND the key referent (order id, product, date) when present in the message.

- accurate (1-5): does every claim in the summary come from the message?
  1 = contains a fabricated fact (claim not in the message).
  3 = all claims are in the message but phrasing exaggerates urgency or scope.
  5 = every claim is supported verbatim or by direct paraphrase; nothing added.

- tone (1-5): is the tone appropriate for a ticket title in an internal support tool?
  1 = emotive, judgmental, or quotes the customer verbatim including profanity / excess punctuation.
  3 = neutral but wordy; reads like a sentence not a title.
  5 = concise, neutral, declarative — fits as the title of a ticket.

Reply with JSON only: {"helpful": int, "accurate": int, "tone": int, "notes": str}
Notes should be <= 20 words and point to the single thing that most affected the lowest axis.
"""
```

Re-run the judge against the same 10 cases, writing to `judge_v2.csv`. Compute the same exact-match and within-one rates against `human_scores.csv` and record them in `calibration.md` under `rubric_v2`.

### Checkpoint

- The within-one rate on at least one axis improved from v1 to v2.
- If a within-one rate got *worse*, that is informative too — chapter 7 warns that an LLM judge can over-fit to a verbose rubric. Keep the result and note it; do not quietly discard v2.

## Step 6 — Write a one-paragraph conclusion

In `calibration.md`, close with 4–6 sentences:

- Which axis is the judge's agreement weakest on, and why — did the rubric lack an anchor, or do you suspect same-task bias?
- Did `rubric_v2`'s anchors move the number? Was the move big enough to justify the extra tokens on every judge call?
- Where would you use this judge in production — gating a prompt change in CI, scoring a sample of daily traffic, both, neither?
- What is one check you would run *in addition to* the judge (chapter 7: "Pair with cheaper checks"), and why?

That paragraph is the deliverable.

## Debrief

An LLM-as-judge is a confident number generator. It is useful *only* for the problems that genuinely cannot be checked deterministically — freeform summaries, tone, "did the reply address the question" — and even then only after calibration. You just produced a tiny but real artifact of that calibration: an agreement rate per rubric version, grounded in your own hand scores. In a real codebase, that calibration lives next to the golden set from exercise-03 and gets re-run whenever you change the judge rubric, the judge model, or the generator you are judging.

Two habits to carry forward:

1. **Different model for the judge.** Chapter 7: same-model bias is real. If your generator and judge are the same model ID, your judge will systematically rate the generator too kindly.
2. **Anchor every number.** A rubric that scores on 1–5 with no anchors is a rubric that scores a 3 for anything mediocre. Anchors are the whole technique.

## Stretch

- Add a fourth axis: **length_appropriate** (1–5), anchored to "fits as a ticket title: between 20 and 140 characters, no code or markup." Score it both ways. Does the judge get this one right at a much higher rate than the subjective axes? That is chapter 7's implicit point about *deterministic* checks beating LLM judges when they are available.
- Replay Step 3 with a **second** judge model (switch providers if you have the key, or use a smaller model from the same provider). Compute the inter-judge agreement. If the two judges disagree more with each other than either does with you, your rubric is the bottleneck.
- Build the smallest-possible golden set companion: use `summary` length ≤ 140 chars and non-empty as a deterministic eval, and the judge's `accurate` axis as the subjective eval. Report both numbers in `calibration.md`. This previews the "cheap checks and the judge together" pattern chapter 7 recommends and that exercise-03 wires into CI.
