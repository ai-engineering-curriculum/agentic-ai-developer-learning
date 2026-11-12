# Lab 02 — Dry-Run `is_confident` on Canned Cascade Outputs

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 4 — Routing by task shape: cascade, fast-first, batch](../04-routing-cascade-and-batch.md) ("Pattern 1 — cascade routing" and "Measuring the cascade — the escalation rate `p`")

## Objective

Chapter 4's one-line claim about cascade routing is "the `is_confident` function is the whole game." If it returns True too often, you ship low-quality small-model outputs. If it returns True too rarely, you escalate almost everything and pay for both calls — *worse* than always-frontier. In this lab you will build an `is_confident` function in isolation, drive it over **ten pre-written cascade decisions** that you can inspect by eye, and compute the two audit numbers chapter 4 names: the **miscalibration rate** (confident but wrong) and the **over-escalation rate** (escalated but frontier agreed).

No LLM calls. No pricing math. No latency. Ten canned examples, one routing function, two audit numbers — the smallest honest atom of a cascade's control plane.

By the end you will have:

- A `canned_cascade.py` script whose `is_confident` function you have written and inspected.
- Printed, per example: the small-model output, whether it was confident, whether it escalated, and (for escalated examples) the frontier result.
- Two aggregate numbers — miscalibration rate and over-escalation rate — matching the audits chapter 4 requires.
- A `NOTES.md` with the one threshold (or structural rule) you picked, and one sentence per audit on what you would change if the number were worse.

> This lab complements [exercise-04](../exercises/exercise-04-cascade-routing-cheap-then-expensive.md), which runs a real cascade on 50 live calls across two models and reports accuracy, mean cost, and p50/p95 latency vs. always-small and always-frontier baselines. Here we use **pre-written** small-model and frontier-model outputs so the question "did my routing logic do what I meant?" is answered before any real-API cost or latency is involved. Do this lab, then do the exercise — the control-plane will not surprise you on real data.

## Prerequisites

- Finished chapter 4.
- Python 3.10+. No API keys needed; no network calls.
- (Optional) `pip install pydantic` if you want to validate the structural variant with a real schema instead of a hand-rolled check.

## Step 1 — Copy the canned decision table

Chapter 4 lists four `is_confident` shapes: **structural**, **explicit confidence**, **self-consistency**, and **second-model classifier**. For this lab we focus on the first two, because they are the cheapest and the ones most teams reach for first.

Save the following as `canned_cascade.py`. The `EXAMPLES` list is the whole point of the lab — ten support-ticket classification cases with the small model's output, a frontier-model "second opinion," and the ground-truth label. Some small-model outputs are confident and right; some are confident and wrong (the miscalibration failure); some are unconfident but would have been right (the over-escalation failure); some are correctly unconfident.

```python
from dataclasses import dataclass
from typing import Optional

# Allowed label set for the task. Anything else = structurally invalid.
LABELS = {"billing", "shipping", "account", "technical", "other"}


@dataclass
class CannedExample:
    ticket: str
    small_output: dict           # what the cheap model returned
    frontier_output: dict        # what the frontier model would return if escalated
    truth: str                   # ground-truth label


# Hand-built so you can inspect every decision by eye.
# Mix: 6 confident-right, 2 confident-wrong (miscalibration),
#      1 unconfident-but-right (over-escalation), 1 correctly-unconfident.
EXAMPLES: list[CannedExample] = [
    # 1. confident + right
    CannedExample(
        ticket="My invoice from June is missing a line item.",
        small_output={"label": "billing", "confidence": 0.92},
        frontier_output={"label": "billing", "confidence": 0.98},
        truth="billing",
    ),
    # 2. confident + right
    CannedExample(
        ticket="Package says delivered but I never received it.",
        small_output={"label": "shipping", "confidence": 0.95},
        frontier_output={"label": "shipping", "confidence": 0.99},
        truth="shipping",
    ),
    # 3. confident + right
    CannedExample(
        ticket="I can't log in — password reset email never arrives.",
        small_output={"label": "account", "confidence": 0.88},
        frontier_output={"label": "account", "confidence": 0.97},
        truth="account",
    ),
    # 4. confident + WRONG  (miscalibration — small model guessed billing on an account issue)
    CannedExample(
        ticket="My subscription auto-renewed on the wrong card linked to my account.",
        small_output={"label": "billing", "confidence": 0.81},
        frontier_output={"label": "account", "confidence": 0.90},
        truth="account",
    ),
    # 5. confident + right
    CannedExample(
        ticket="App crashes on startup after the latest update.",
        small_output={"label": "technical", "confidence": 0.90},
        frontier_output={"label": "technical", "confidence": 0.97},
        truth="technical",
    ),
    # 6. STRUCTURALLY INVALID — label not in LABELS. Must escalate regardless of confidence.
    CannedExample(
        ticket="I want to cancel my membership and get a refund.",
        small_output={"label": "cancellation", "confidence": 0.87},
        frontier_output={"label": "account", "confidence": 0.92},
        truth="account",
    ),
    # 7. unconfident — small model hedged. Frontier agrees with the small model's top guess.
    #    If we escalate, this is an *over-escalation* (we pay twice for the same answer).
    CannedExample(
        ticket="Charged $9.99 but my plan is supposed to be $4.99.",
        small_output={"label": "billing", "confidence": 0.55},
        frontier_output={"label": "billing", "confidence": 0.96},
        truth="billing",
    ),
    # 8. unconfident — small model hedged and was wrong; frontier gets it right.
    #    Correct escalation.
    CannedExample(
        ticket="Your API is throwing 500s on every POST to /orders.",
        small_output={"label": "other", "confidence": 0.40},
        frontier_output={"label": "technical", "confidence": 0.94},
        truth="technical",
    ),
    # 9. confident + right
    CannedExample(
        ticket="Where is my order? Tracking hasn't updated in five days.",
        small_output={"label": "shipping", "confidence": 0.91},
        frontier_output={"label": "shipping", "confidence": 0.98},
        truth="shipping",
    ),
    # 10. confident + WRONG  (miscalibration — small model guessed shipping on a technical issue)
    CannedExample(
        ticket="I can't track my order because the tracking page itself is broken.",
        small_output={"label": "shipping", "confidence": 0.83},
        frontier_output={"label": "technical", "confidence": 0.89},
        truth="technical",
    ),
]
```

Walk the table once with your eye before you touch the routing function. Three questions to answer out loud:

- Which examples are confident+wrong? (Chapter 4 calls these the miscalibration failures.)
- Which examples are unconfident but the frontier ends up agreeing with the small model's guess anyway? (Chapter 4 calls these the over-escalation cases.)
- Which example is structurally invalid — the label is not in `LABELS` — and must escalate regardless of the confidence number?

### Checkpoint

You should be able to label each row as one of:
`confident+right` / `confident+wrong` / `unconfident+right-if-believed` / `unconfident+wrong-if-believed` / `structural fail`.

That hand-labelling is exactly what chapter 4's two audits — miscalibration and over-escalation — formalise.

## Step 2 — Write `is_confident` (structural + explicit-confidence)

Add the routing function. Chapter 4 says: "write down **why** you picked [the check] and what the escalation trigger is, at the top of the function." Do that in the docstring.

```python
CONFIDENCE_THRESHOLD = 0.70   # <--- the single knob. tune in Step 4.


def is_confident(output: dict) -> bool:
    """
    Combined structural + explicit-confidence check.

    Picked because:
      - Structural (label ∈ LABELS) is a free hard gate — a result that doesn't
        even match the output schema cannot be shipped. Escalate immediately.
      - Explicit confidence (output['confidence'] >= THRESHOLD) is the knob we
        tune. Below the threshold we distrust the small model's own estimate
        and escalate.

    Escalation trigger: label not in the enum, OR confidence below THRESHOLD.
    """
    if output.get("label") not in LABELS:
        return False                               # structural fail -> escalate
    if output.get("confidence", 0.0) < CONFIDENCE_THRESHOLD:
        return False                               # low confidence -> escalate
    return True


def classify_cascade(example: CannedExample) -> dict:
    """
    One step of cascade routing against canned outputs.

    Returns a record for the audit table — no real model calls here.
    """
    small = example.small_output
    if is_confident(small):
        return {
            "ticket": example.ticket,
            "final_label": small["label"],
            "escalated": False,
            "small_label": small["label"],
            "small_confidence": small.get("confidence"),
            "frontier_label": None,
            "truth": example.truth,
        }
    frontier = example.frontier_output
    return {
        "ticket": example.ticket,
        "final_label": frontier["label"],
        "escalated": True,
        "small_label": small["label"],
        "small_confidence": small.get("confidence"),
        "frontier_label": frontier["label"],
        "truth": example.truth,
    }
```

### Checkpoint

- Chapter 4 names three failure modes for cascade routing. Your `is_confident` already defends against the structural-fail mode (label not in the enum). The remaining two — confidence-too-permissive and confidence-too-strict — are the ones Step 4's audits will reveal on this dataset.
- The function has **one tunable number**: `CONFIDENCE_THRESHOLD`. Chapter 4's "Starter guidance" says to try 2–3 values and pick the one where cascade accuracy is close to always-frontier and cost is close to always-small — Step 4 of this lab is that drill at the 10-example scale.

## Step 3 — Run it and print the decision table

Add the driver.

```python
def main() -> None:
    rows = [classify_cascade(ex) for ex in EXAMPLES]

    # Per-row detail.
    header = (
        f"{'#':>2}  {'small→':<12}{'conf':>6}  {'esc':>3}  "
        f"{'frontier→':<12}{'final':<12}{'truth':<12}{'correct?':<8}"
    )
    print(header)
    print("-" * len(header))
    for i, r in enumerate(rows, start=1):
        ok = "yes" if r["final_label"] == r["truth"] else "NO"
        print(
            f"{i:>2}  "
            f"{str(r['small_label']):<12}"
            f"{r['small_confidence']:>6.2f}  "
            f"{('yes' if r['escalated'] else '.'):>3}  "
            f"{(str(r['frontier_label']) if r['escalated'] else '-'):<12}"
            f"{r['final_label']:<12}{r['truth']:<12}{ok:<8}"
        )

    # Aggregates chapter 4 asks for.
    n = len(rows)
    escalated      = [r for r in rows if r["escalated"]]
    not_escalated  = [r for r in rows if not r["escalated"]]
    p              = len(escalated) / n

    # Miscalibration: confident (did NOT escalate) but final answer was wrong.
    miscalibrated = [r for r in not_escalated if r["final_label"] != r["truth"]]
    miscal_rate   = len(miscalibrated) / max(len(not_escalated), 1)

    # Over-escalation: escalated, but frontier came back with the same answer the
    # small model already had. We paid twice for the same label.
    over_escalated = [r for r in escalated if r["frontier_label"] == r["small_label"]]
    over_rate      = len(over_escalated) / max(len(escalated), 1)

    cascade_accuracy = sum(1 for r in rows if r["final_label"] == r["truth"]) / n

    print()
    print(f"escalation rate p:        {p:.2f}  ({len(escalated)}/{n})")
    print(f"cascade accuracy:         {cascade_accuracy:.2f}  "
          f"({sum(1 for r in rows if r['final_label'] == r['truth'])}/{n})")
    print(f"miscalibration rate:      {miscal_rate:.2f}  "
          f"({len(miscalibrated)}/{len(not_escalated)} of 'confident' calls were wrong)")
    print(f"over-escalation rate:     {over_rate:.2f}  "
          f"({len(over_escalated)}/{len(escalated)} escalations returned the small model's answer)")


if __name__ == "__main__":
    main()
```

Run it:

```bash
python canned_cascade.py
```

### Expected output (shape)

A per-row table followed by four aggregate numbers. On the canned 10-row dataset with `CONFIDENCE_THRESHOLD = 0.70`, you should see roughly:

- `escalation rate p` around 0.3–0.4 — rows 6 (structural), 7, 8 escalate; the exact count depends on where you set the threshold.
- `cascade accuracy` well above the always-small baseline, because the correct escalations (rows 6 and 8) swap wrong small-model answers for right frontier ones.
- `miscalibration rate` **non-zero** — rows 4 and 10 are both above threshold and both wrong. This is chapter 4's "confidence check is too permissive" failure, visible by inspection.
- `over-escalation rate` **non-zero** — row 7 escalates but the frontier returns the same label the small model guessed. This is chapter 4's "confidence check is too strict" failure.

```
 #  small→        conf  esc  frontier→   final       truth       correct?
-------------------------------------------------------------------------
 1  billing       0.92    .  -           billing     billing     yes
 2  shipping      0.95    .  -           shipping    shipping    yes
 3  account       0.88    .  -           account     account     yes
 4  billing       0.81    .  -           billing     account     NO
 5  technical     0.90    .  -           technical   technical   yes
 6  cancellation  0.87  yes  account     account     account     yes
 7  billing       0.55  yes  billing     billing     billing     yes
 8  other         0.40  yes  technical   technical   technical   yes
 9  shipping      0.91    .  -           shipping    shipping    yes
10  shipping      0.83    .  -           technical   technical   NO
```

(Your numbers will match this shape for this canned data; the point is to read the three failure classes off the table.)

### Checkpoint

- Point at the row that **escalated because of the structural check**, not the confidence check. (Row 6 — `"cancellation"` is not in `LABELS`.)
- Point at the row that escalated *and* got the right answer that the small model would have missed. (Row 8.)
- Point at the row that escalated *redundantly* — frontier agreed with the small model. (Row 7, your over-escalation case.)
- Point at the rows that **didn't** escalate but should have — the small model was confident and wrong. (Rows 4 and 10, your miscalibration cases.)

The four pointable rows are the four cells in chapter 4's quality matrix for cascade routing. Seeing them on a 10-row grid before you run the full exercise is the whole point of the lab.

## Step 4 — Flip one knob and observe

Pick **one** of these and re-run `canned_cascade.py`. Record what moves in `NOTES.md`.

- **Tighten the threshold: `CONFIDENCE_THRESHOLD = 0.85`.** More rows fall below. Escalation rate `p` rises. Miscalibration rate may drop (row 4 is 0.81, so it now escalates — you catch one previously-shipped wrong answer). Over-escalation may rise. Cascade cost per call rises. This is the chapter 4 trade-off made concrete.
- **Loosen the threshold: `CONFIDENCE_THRESHOLD = 0.50`.** Fewer rows fall below. Row 7 now ships without escalation (its confidence 0.55 clears), dropping over-escalation. Miscalibration rate holds — the confident-wrong rows are still confident-wrong. Cost drops but at the cost of leaving one case where the small model *would* have been fine.
- **Delete the structural check.** Comment out the `if output.get("label") not in LABELS` branch. Row 6 now ships `"cancellation"` — a label your downstream code does not know what to do with. Cascade accuracy drops. This is why chapter 4 lists "cascade routing with no confidence signal" as the first common mistake — the structural check is the cheapest, most honest confidence signal available.

Record which variant you ran and one sentence on each of: `p`, cascade accuracy, miscalibration rate, over-escalation rate.

## Step 5 — Write a 5-sentence debrief

In `NOTES.md`, answer in 5–7 sentences total:

1. On the baseline run (threshold 0.70, structural check on), what were your four numbers — `p`, cascade accuracy, miscalibration rate, over-escalation rate? Chapter 4 says one of them is the metric you alarm on in production. Which?
2. The Step-4 knob you flipped: did miscalibration and over-escalation move in the **same direction** or in **opposite directions**? Chapter 4's "Failure mode to watch" predicts the shape — restate it in your own words.
3. For the two miscalibration rows (4 and 10), look at the ticket text. Is there a pattern in the kind of input that fools the small model? (This is the seed chapter 4 and exercise 02 call out — "examine the top 10 examples where frontier was right and small was wrong.")
4. If you were going to add **one more** confidence check (self-consistency, a second-model classifier, or something else), which miscalibration or over-escalation row would you be building it to catch, and why?

That write-up is the deliverable. Commit `canned_cascade.py` and `NOTES.md`.

## Debrief

Three habits pay forward from here:

- Before you wire a cascade against real LLM calls, dry-run it against canned outputs that include every failure class (confident+wrong, unconfident+right-anyway, structural fail). If the audit numbers look wrong on 10 rows you built by hand, they will not get better on 1000 rows you did not.
- The two audits — miscalibration and over-escalation — are what the chapter calls "the control plane" of a cascade. Log them per call in production (`escalated` flag plus "final answer matched small answer" plus "final answer matched ground truth if you have one") and alarm on either drifting.
- The structural check is free. Any time the small model's output has a schema (JSON with an enum field, a known label set, a required shape), a structural check belongs in `is_confident` before any confidence number is looked at. Rows like #6 are what it is for.

The exercise next door takes exactly this `is_confident` function, points it at a real small model and a real frontier model on 50 live examples, measures cost and latency instead of just the audit counts, and asks whether the cascade's cost per call is actually below always-frontier. If this lab's audit numbers made sense to you, that exercise's cost numbers will too.

## Stretch

- **Self-consistency variant.** Add a `small_output_second: dict` field to the canned examples and a second variant of `is_confident` that escalates when the two small-model outputs disagree, regardless of either one's confidence. Which rows change? Chapter 4's "Self-consistency" option costs you one extra small-model call per decision — on this 10-row dataset, when was the extra call worth it?
- **Second-model classifier variant.** Replace the `output['confidence']` check with a small "looks right" classifier — a regex, a rule-based scorer, or a tiny prompt against a fictitious tiny model represented by a hard-coded dict. Does it catch row 4 (confident + wrong) that the explicit-confidence check missed?
- **Measure cost on the canned data.** Paste the small and frontier `$/MTok` numbers from the pricing page (lab 01 built this habit). Assume 60 input tokens and 10 output tokens per call on both models. Compute, by hand: cost per call for always-small, always-frontier, and the cascade. Which of the two routing bugs — miscalibration or over-escalation — was dollar-expensive on the 10-row dataset, and which was accuracy-expensive?
- **Promote the structural check to a reusable gate.** Many features reuse the same "output matches a schema" check. Pull it into `structural_ok(output, schema)` and keep `is_confident` short. That is what a production cascade usually ends up with.
