# Lab 01 — Calibrate a Similarity Threshold on a Hand-Written Rewording Set

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 2 — Scoring outputs: rule-based, similarity, and judge](../02-scoring-outputs.md) ("Similarity scorers" → "Choosing the threshold")

## Objective

Chapter 2 is blunt about the one question every similarity-scored row forces you to answer: "what cosine value counts as *close enough*?" Guessing produces a scorer that either lets everything through or fails on rewordings. The chapter prescribes an exact drill: take a few rows where you already know the correct answer, hand-write 3 acceptable rewordings and 3 wrong-but-plausible answers for each, embed them, compute cosines against the reference, and **pick the threshold that sits between the two distributions**.

In this lab you will do that drill — in isolation — on three hand-written references. No golden set. No pytest. No judge. Just the one mechanical step that separates a defensible similarity threshold from a vibes-chosen one.

By the end you will have:

- A `calibrate_threshold.py` script that embeds 3 reference answers plus 3 rewordings and 3 wrong-answers for each (21 strings total), computes cosines against each reference, and prints a two-column distribution (`good_cosines` vs. `bad_cosines`).
- A proposed threshold — the midpoint between the two distributions — pasted into `NOTES.md` with the embedding model name next to it, exactly as chapter 2 requires ("Write the number and the model name into the row's tolerance so it is reproducible").
- One sentence per reference in `NOTES.md` naming whether the two distributions separated cleanly or overlapped — chapter 2's test for whether similarity is the right scorer at all.

> This lab complements [exercise-03](../exercises/exercise-03-rule-vs-similarity-vs-judge-scoring.md), which drills the full "score 10 rows three ways and compare against hand-labels" comparison. Here we stay with the single sub-step — threshold calibration on hand-written rewordings — so you can see what one defensible number actually comes from before the exercise bundles it with the rule and judge families.

## Prerequisites

- Finished chapter 2.
- Python 3.10+.
- An `OPENAI_API_KEY` exported with a small funded budget. Twenty-one short strings through `text-embedding-3-small` costs well under a cent; if you want to pick another embedding model, chapter 2's rule applies — pin it and record which one.
- `pip install openai numpy`.

## Step 1 — Hand-write the three reference rows

Pick three short reference answers that are plausible things a free-text LLM feature might produce. Chapter 2's examples (summaries, Q&A answers, explanations) are good targets. The references used below are deliberately generic — substitute your own if your feature has a specific voice.

Save as `calibrate_threshold.py`:

```python
# Pin the embedding model exactly as chapter 2 instructs:
# "A threshold that works for terse answers will not work for long ones.
#  The threshold is a per-scorer, per-model calibration — not a per-row
#  invention."
EMBEDDING_MODEL = "text-embedding-3-small"


REFERENCES = [
    {
        "id": "ref-001",
        "reference": "Refunds are eligible for orders placed within the last 30 days.",
        "rewordings": [
            "You can request a refund on any order from the past 30 days.",
            "Orders placed in the last month qualify for a refund.",
            "Refund requests are honored for purchases up to 30 days old.",
        ],
        "wrong_but_plausible": [
            "Refunds are issued for orders placed within the last 7 days.",
            "Refunds apply only to subscription cancellations, not one-off orders.",
            "All sales are final; refunds are not available.",
        ],
    },
    {
        "id": "ref-002",
        "reference": "The merger between Acme and Globex closed on Tuesday for $12B in cash.",
        "rewordings": [
            "Acme and Globex completed their merger Tuesday in a $12 billion cash deal.",
            "On Tuesday the Acme-Globex merger closed at $12B, paid in cash.",
            "Acme bought Globex for $12 billion in cash; the deal closed Tuesday.",
        ],
        "wrong_but_plausible": [
            "The rumoured merger between Acme and Globex is expected to close soon.",
            "Acme and Globex announced a $1.2B stock-swap merger that closed on Friday.",
            "Acme called off its $12B bid for Globex after regulator pushback.",
        ],
    },
    {
        "id": "ref-003",
        "reference": "To reset your password, click 'Forgot password?' on the login screen and follow the email link.",
        "rewordings": [
            "Use the 'Forgot password?' link on the sign-in page; a reset email will be sent.",
            "Password resets start by clicking 'Forgot password?' on the login page and opening the email that arrives.",
            "Go to the login screen, pick 'Forgot password?', and finish the reset from the email we send.",
        ],
        "wrong_but_plausible": [
            "To reset your password, contact support and request a manual reset.",
            "Open the mobile app and go to Settings > Security > Reset PIN.",
            "Password resets are not supported on free accounts.",
        ],
    },
]
```

### Checkpoint

- Every reference has **exactly three rewordings** and **three wrong-but-plausible** answers. Chapter 2 names this count; sticking to it keeps the two distributions readable at a glance.
- The wrong-but-plausible answers are *plausibly on topic* — not nonsense. "The sky is blue" is not useful here. "Refunds are not available" is — it is the kind of wrong answer the model could plausibly produce, so it is the kind the threshold has to separate from a correct rewording.

## Step 2 — Embed every string and compute cosines

Add the embedding and cosine logic. Chapter 2's `similarity_score` function is the shape; here we batch the whole lab's strings in one API call for cost and speed.

```python
import os
from itertools import chain

import numpy as np
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])


def cosine(a: np.ndarray, b: np.ndarray) -> float:
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))


def embed_batch(strings: list[str]) -> list[np.ndarray]:
    resp = client.embeddings.create(model=EMBEDDING_MODEL, input=strings)
    return [np.array(item.embedding) for item in resp.data]


def main() -> None:
    # Collect every string to embed, in a stable order, then embed all at once.
    layout: list[tuple[str, str, str]] = []  # (ref_id, bucket, text)
    for row in REFERENCES:
        layout.append((row["id"], "reference", row["reference"]))
        for text in row["rewordings"]:
            layout.append((row["id"], "rewording", text))
        for text in row["wrong_but_plausible"]:
            layout.append((row["id"], "wrong", text))

    vectors = embed_batch([text for _, _, text in layout])
    by_key: dict[tuple[str, str, str], np.ndarray] = {
        (ref_id, bucket, text): vec
        for (ref_id, bucket, text), vec in zip(layout, vectors)
    }

    # Per-reference: cosines of rewordings vs. reference, and wrong-answers vs. reference.
    all_good: list[float] = []
    all_bad: list[float] = []
    for row in REFERENCES:
        ref_vec = by_key[(row["id"], "reference", row["reference"])]
        good = [cosine(ref_vec, by_key[(row["id"], "rewording", t)])
                for t in row["rewordings"]]
        bad  = [cosine(ref_vec, by_key[(row["id"], "wrong", t)])
                for t in row["wrong_but_plausible"]]
        all_good.extend(good)
        all_bad.extend(bad)

        print(f"\n{row['id']}  reference: {row['reference']!r}")
        for t, c in zip(row["rewordings"], good):
            print(f"  good  cos={c:.3f}  {t}")
        for t, c in zip(row["wrong_but_plausible"], bad):
            print(f"  bad   cos={c:.3f}  {t}")

    print("\nAggregate (across all three references):")
    print(f"  good cosines: min={min(all_good):.3f}  max={max(all_good):.3f}  "
          f"mean={np.mean(all_good):.3f}")
    print(f"  bad  cosines: min={min(all_bad):.3f}   max={max(all_bad):.3f}   "
          f"mean={np.mean(all_bad):.3f}")

    # Chapter 2: "Pick the threshold that sits between the two distributions."
    # The midpoint between the worst rewording and the best wrong-answer is the
    # simplest defensible choice.
    proposed = (min(all_good) + max(all_bad)) / 2
    print(f"\nProposed threshold (midpoint between worst good and best bad): "
          f"{proposed:.3f} under {EMBEDDING_MODEL}")


if __name__ == "__main__":
    main()
```

Run it:

```bash
python calibrate_threshold.py
```

### Expected output (shape)

A per-reference block showing six cosines each, then an aggregate. Shape (your numbers will differ by a few hundredths):

```
ref-001  reference: 'Refunds are eligible for orders placed within the last 30 days.'
  good  cos=0.871  You can request a refund on any order from the past 30 days.
  good  cos=0.849  Orders placed in the last month qualify for a refund.
  good  cos=0.832  Refund requests are honored for purchases up to 30 days old.
  bad   cos=0.812  Refunds are issued for orders placed within the last 7 days.
  bad   cos=0.598  Refunds apply only to subscription cancellations, not one-off orders.
  bad   cos=0.531  All sales are final; refunds are not available.
...
Aggregate (across all three references):
  good cosines: min=0.788  max=0.918  mean=0.854
  bad  cosines: min=0.412  max=0.812  mean=0.619

Proposed threshold (midpoint between worst good and best bad): 0.800 under text-embedding-3-small
```

### Checkpoint

- The `good` cosines should sit higher than the `bad` cosines on average. If they do not on any reference, that reference is a signal — chapter 2 explicitly calls this out: "If the two groups overlap heavily, similarity is not the right scorer for this rubric." Note which reference overlapped in `NOTES.md`; on your real feature, you would pick a rule- or judge-based tolerance for that row.
- Look specifically at `ref-001`'s `bad` row: *"Refunds are issued for orders placed within the last 7 days."* The reference's meaning is factually wrong in one small number (`30` → `7`). Chapter 2's "Two failure modes to know" warns that similarity is blind to small factual changes like dates and numbers. On this canned dataset you can often **see** that failure as a `bad` cosine that is suspiciously close to the `good` cluster — the nearest-neighbour failure that would let a factually wrong answer sneak through a loose threshold. This is exactly why the chapter says: "If your row's correctness depends on a negation, a date, or an exact identifier, put those in a *rule-based* sub-check on the same row — do not rely on similarity to catch them."
- The proposed midpoint threshold is a *starting point*, not a final number. Chapter 2's calibration says to pick "somewhere in 0.70–0.85 for `text-embedding-3-small`"; your midpoint probably falls in that band.

## Step 3 — Tighten and loosen the threshold, observe the trade

In `NOTES.md`, extend the output with a small table: at the proposed threshold, at the proposed threshold minus 0.05, and at the proposed threshold plus 0.05, count how many of the 9 `good` cosines pass and how many of the 9 `bad` cosines pass. Something like:

```
threshold      good_pass / 9     bad_pass / 9      shape
0.75           9                 2                 2 false passes (bad answers sneaking through)
0.80           8                 1                 1 false pass, 1 false fail
0.85           5                 0                 4 false fails (good rewordings rejected)
```

(Your numbers will depend on your rewordings and your embedding model.)

### Checkpoint

- Can you see the chapter's warning in action? "Threshold too low lets everything through" and "threshold too high fails on rewordings" are not abstract — they are two columns of this table.
- Are any `bad` cosines sitting *above* the midpoint? If one is, it is almost certainly the near-miss factual-change case (date / number / negation). Chapter 2 says to layer a rule-based sub-check on that field in the row's tolerance rather than trying to tighten the similarity threshold until it catches the near-miss — tightening would reject real rewordings.

## Step 4 — Write the three deliverables into `NOTES.md`

The chapter's test for whether calibration was done right is that a future reader can see **where the threshold came from**. Record that:

1. The embedding model (one line).
2. The proposed threshold (one line).
3. For each of the three references: did the two distributions separate cleanly, overlap at a near-miss, or overlap heavily? One sentence per reference.
4. The row's `tolerance` block in JSONL shape (the shape chapter 2 says must go into `golden.jsonl`) using the threshold you picked, for whichever reference you would actually ship a similarity-tolerance row for. Example:

```json
{
  "id": "qa-001",
  "expected": {"reference_answer": "Refunds are eligible for orders placed within the last 30 days."},
  "tolerance": {
    "reference_answer": {"kind": "similarity", "min_cosine": 0.80, "model": "text-embedding-3-small"}
  }
}
```

### Checkpoint

- A colleague reading `NOTES.md` cold can tell: what model you used, what number you picked, what hand-written data you picked it from, and which reference would **not** be a good similarity row on this feature. Chapter 2's calibration is designed around that — the artefact is the defence.

## Debrief

Three habits pay forward from here:

- Threshold calibration is a per-scorer + per-embedding-model fact. If you later swap `text-embedding-3-small` for `text-embedding-3-large`, every threshold in your eval is stale — chapter 2's "threshold drift" warning. Rerun this lab's drill on the new model before shipping the swap.
- The two distributions are the test for whether similarity is the right scorer at all. A reference whose `good` and `bad` cosines overlap is a reference whose tolerance should be a rule or a judge, not similarity. Writing that one sentence per reference (step 4) protects you from quietly adopting a scorer that cannot separate right from wrong on the row.
- The drill is small enough — 21 strings, one embeddings call, cents in cost — that you can repeat it on *every* reference row the first time you add similarity tolerance to it, not just the three you saw in this lab. Chapter 2's recommendation, "never pick a threshold on one row," becomes mechanical when the drill costs this little.

The exercise next door (exercise 03) takes this same calibration and pairs it with hand-labelled `pass` / `fail` ground truth on 10 rows, plus parallel runs of a rule-based scorer and an LLM-as-a-judge, so you can compare all three families. If this lab's two-distribution picture made sense, that exercise's three-column comparison table will too.

## Stretch

- **Repeat on `text-embedding-3-large`.** Change `EMBEDDING_MODEL`, rerun. Chapter 2 warns that the threshold band moves upward on larger embedding models — does yours? Record both numbers side by side; this is the artefact you will produce when a real model-swap PR touches the embedding layer.
- **Add a near-miss row deliberately.** Append a fourth reference whose single `bad` entry differs from the reference by **one number** (e.g. reference says `30 days`, wrong says `90 days`, otherwise identical wording). Compute the cosine. It will almost certainly sit above your threshold. In `NOTES.md`, write the two-sentence policy this forces onto the row's tolerance: a `regex` or `exact` sub-check on the numeric field plus the similarity check on the free-text wrapper. This is chapter 2's "do not rely on similarity to catch negation, dates, or identifiers" made concrete.
- **Cache the reference embeddings to a tiny JSONL.** Write the 3 reference vectors to `references.jsonl` keyed by `(id, model)`. A rerun of the lab should load the cache instead of re-embedding the references. Chapter 2's "cheap operational habits" names exactly this — compute once at ingest, cache next to the row.
- **Visualise the two distributions.** Plot `good_cosines` vs. `bad_cosines` on a 1D strip plot (even in the terminal with `*` and `.` markers). Seeing the gap (or the overlap) with your eyes is often faster than reading the numeric table. Chapter 2's "pick the threshold at the midpoint" becomes obvious when the picture is in front of you.
