# Lab 03 — Calibrate a Refusal Score Floor

**Estimated duration:** 45–60 minutes
**Covers:** [chapter 6 — The RAG Query Loop](../06-the-rag-query-loop.md) ("Handling 'I don't know'"), [chapter 7 — Evaluating Retrieval Quality](../07-evaluating-retrieval-quality.md) ("Refusal correctness")

## Objective

Chapter 6 calls a score floor "the single highest-leverage upgrade you can make over a naive RAG loop." The idea is simple: if the top-K retrieval scores are all low, short-circuit and return "I do not see that in the documents" *without calling the model*. You save the model call, and — more importantly — you avoid handing the model irrelevant context that it will be tempted to use. But the right threshold is corpus-dependent, so in this lab you will build the one-shot calibration from chapter 7: plot the top-1 score distribution for a handful of *answerable* questions and a handful of *unanswerable* ones, pick the cut that separates them, and lock it in.

By the end you will have:

- A `floor_calibrate.py` that embeds a small corpus, runs 8 answerable + 6 unanswerable queries, and prints each query's top-1 score alongside its label.
- A sorted two-column table where the right threshold is visually obvious (if the two distributions separate cleanly).
- A chosen floor value and a short justification in `NOTES.md`, including the one metric from chapter 7 that would tell you the choice was wrong: `over_refuse_rate` on answerable questions and `over_answer_rate` on unanswerable ones.

> This lab complements [exercise-02](../exercises/exercise-02-simple-rag-pipeline.md) (stretch goal: "Add a score-floor short-circuit"). The exercise's stretch goal asks you to try three arbitrary thresholds; here we calibrate it empirically on a labeled set, the chapter 7 way.

## Prerequisites

- Finished chapters 6 and 7.
- Finished [exercise-01](../exercises/exercise-01-embeddings-similarity.md) (you have normalized-embedding + dot-product search working).
- A funded embedding API key (`OPENAI_API_KEY` for the defaults; any provider works).
- Python 3.10+ with `openai` and `numpy`.
- Budget: one batch of ~10 corpus embeddings plus 14 single-query embeddings. Cents.

## Step 1 — Fix a small corpus you know well

Save as `floor_calibrate.py`:

```python
import numpy as np
from openai import OpenAI

client = OpenAI()
MODEL = "text-embedding-3-small"

DOCS = [
    "We accept returns of unused items within 30 days of delivery.",
    "All electronics must be returned in their original packaging.",
    "Sale items marked 'final' cannot be returned or exchanged.",
    "Standard shipping to the contiguous US takes 2–4 business days.",
    "Orders to Hawaii and Alaska usually arrive in 5–7 business days.",
    "International orders are sent via UPS Worldwide; tracking is included.",
    "Gift wrap is available at checkout for an additional $4.99 per item.",
    "Our customer service team is available Monday–Friday, 9am–6pm Pacific.",
    "We do not currently offer phone support; please use the contact form.",
    "Refunds appear on the original payment method within 5–10 business days.",
]


def embed(texts):
    resp = client.embeddings.create(model=MODEL, input=texts)
    vecs = np.array([d.embedding for d in resp.data], dtype=np.float32)
    norms = np.linalg.norm(vecs, axis=1, keepdims=True)
    return vecs / np.clip(norms, 1e-12, None)


doc_vecs = embed(DOCS)
print(f"Corpus: {len(DOCS)} docs, embedded with {MODEL}")
```

Everything in the lab builds on this fixed corpus. Do not change `DOCS` between runs — the floor you calibrate is only valid for the corpus you calibrate it on.

### Checkpoint

- The print line confirms `len(DOCS)` chunks were embedded.
- The corpus covers clear topic clusters: returns (3 docs), shipping (3 docs), billing (1 doc), support (2 docs), extras (1 doc). This spread matters for step 3 — you want unanswerable questions that are *near-adjacent topics*, not wildly off-corpus.

## Step 2 — Write a labeled query set: answerable + unanswerable

Append to `floor_calibrate.py`:

```python
# Label ANSWERABLE = should pass the floor (top-1 score HIGH)
# Label REFUSAL    = should trip the floor (top-1 score LOW)
QUERIES = [
    # answerable, paraphrase or direct
    ("ANSWERABLE", "how long do I have to return an item?"),
    ("ANSWERABLE", "do you ship to Hawaii?"),
    ("ANSWERABLE", "when do refunds appear on my card?"),
    ("ANSWERABLE", "can I return a sale item?"),
    ("ANSWERABLE", "do you offer international shipping?"),
    ("ANSWERABLE", "what are your customer service hours?"),
    ("ANSWERABLE", "is there a phone number I can call?"),
    ("ANSWERABLE", "how much does gift wrap cost?"),
    # refusal — not in corpus, varying degrees of topical adjacency
    ("REFUSAL",    "do you offer price matching?"),
    ("REFUSAL",    "can I pay with cryptocurrency?"),
    ("REFUSAL",    "what is the CEO's name?"),
    ("REFUSAL",    "do you have a retail store in Chicago?"),
    ("REFUSAL",    "what is the airspeed velocity of an unladen swallow?"),
    ("REFUSAL",    "is there a warranty on third-party items?"),
]
```

Two things to notice about the refusal set:

- Some refusal queries are *topically adjacent* (price matching, cryptocurrency, warranty) — they share vocabulary with the corpus but the specific answer is not there. These are the hard refusal cases.
- Some are *obviously off-corpus* (airspeed velocity, CEO's name). These should score low; use them as sanity checks.

Chapter 7 recommends 10–20% refusal cases in a real eval set. This small lab tilts the ratio toward refusals (about 40%) because the whole point is to see where the two distributions separate.

### Checkpoint

- You have exactly 8 ANSWERABLE and 6 REFUSAL queries.
- At least 3 of the REFUSAL queries are topically adjacent to a corpus doc (share one or two content words).
- At least 1 REFUSAL query is deliberately off-topic as a sanity check.

## Step 3 — Score every query, sort by top-1, see the gap

Append:

```python
rows = []
for label, q in QUERIES:
    q_vec = embed([q])[0]
    scores = doc_vecs @ q_vec
    top1_idx = int(np.argmax(scores))
    rows.append({
        "label":    label,
        "query":    q,
        "top1":     float(scores[top1_idx]),
        "top1_doc": DOCS[top1_idx],
    })

rows.sort(key=lambda r: r["top1"])

print(f"\n{'top1':>6}  {'label':<10}  {'query':<55}  top-1 doc")
print("-" * 120)
for r in rows:
    doc = r["top1_doc"][:55]
    print(f"{r['top1']:>6.3f}  {r['label']:<10}  {r['query']:<55}  {doc}")
```

Run it:

```bash
python floor_calibrate.py
```

### Expected output (shape)

A 14-row table, sorted ascending by `top1`. If the corpus and queries behave the way the chapter predicts, you should see something like:

```
  top1  label       query                                                    top-1 doc
--------------------------------------------------------------------------------------
 0.0xx  REFUSAL     what is the airspeed velocity of an unladen swallow?     Gift wrap is available ...
 0.1xx  REFUSAL     what is the CEO's name?                                  Our customer service te...
 0.2xx  REFUSAL     do you have a retail store in Chicago?                   Standard shipping to th...
 0.2xx  REFUSAL     can I pay with cryptocurrency?                           Refunds appear on the o...
 0.3xx  REFUSAL     is there a warranty on third-party items?                All electronics must be...
 0.3xx  REFUSAL     do you offer price matching?                             Sale items marked 'fina...
 0.4xx  ANSWERABLE  how much does gift wrap cost?                            Gift wrap is available ...
 0.4xx  ANSWERABLE  do you offer international shipping?                     International orders ar...
 0.5xx  ANSWERABLE  what are your customer service hours?                    Our customer service te...
 0.5xx  ANSWERABLE  is there a phone number I can call?                      We do not currently off...
 0.5xx  ANSWERABLE  do you ship to Hawaii?                                   Orders to Hawaii and Al...
 0.6xx  ANSWERABLE  how long do I have to return an item?                    We accept returns of un...
 0.6xx  ANSWERABLE  can I return a sale item?                                Sale items marked 'fina...
 0.6xx  ANSWERABLE  when do refunds appear on my card?                       Refunds appear on the o...
```

<!-- needs-research: the exact score levels here are empirical and depend on the embedding model; what matters is the *shape* — REFUSAL at the bottom, ANSWERABLE at the top, with a visible gap in between. If your table does not show that shape, the fix is usually to tighten the refusal queries (make them more topically adjacent) or widen the corpus (one-sentence docs produce noisier scores than paragraph-scale docs). -->

### Checkpoint

Open the sorted table and answer in `NOTES.md`:

1. What is the highest `top1` score for a REFUSAL row? What is the lowest `top1` score for an ANSWERABLE row? If REFUSAL-max < ANSWERABLE-min, you have a clean gap and the threshold is any value in that gap. If they overlap, record the overlap and continue to step 4.
2. Which REFUSAL query had the highest top-1 score? Is it one of the "topically adjacent" cases you deliberately included? (If so, that is the lab working — adjacent refusals are the hard ones.)
3. Which ANSWERABLE query had the lowest top-1 score? Read its `top1_doc`. Is the retrieval actually correct (right doc, low score) or actually wrong (wrong doc, low score)? These are two different problems.

## Step 4 — Pick a floor and compute `over_refuse` / `over_answer`

In `NOTES.md`, pick one threshold `FLOOR` based on the sorted table. Options:

- **Clean-gap pick:** midpoint between the highest REFUSAL score and the lowest ANSWERABLE score.
- **Overlap pick:** any threshold minimizes one error but not the other. Pick the point that gives you the best trade for your app — for a support bot where over-answering is dangerous, bias toward a higher floor; for a search UI where over-refusing is annoying, bias lower.

Then append to `floor_calibrate.py` (filling in your chosen value):

```python
FLOOR = 0.35  # <-- replace with your chosen value

def would_short_circuit(top1: float) -> bool:
    return top1 < FLOOR

over_refuse = sum(1 for r in rows
                  if r["label"] == "ANSWERABLE" and would_short_circuit(r["top1"]))
over_answer = sum(1 for r in rows
                  if r["label"] == "REFUSAL"    and not would_short_circuit(r["top1"]))
n_ans = sum(1 for r in rows if r["label"] == "ANSWERABLE")
n_ref = sum(1 for r in rows if r["label"] == "REFUSAL")

print(f"\nFLOOR = {FLOOR}")
print(f"over_refuse_rate (answerable mis-rejected): "
      f"{over_refuse}/{n_ans} = {over_refuse/n_ans:.2f}")
print(f"over_answer_rate (refusal mis-accepted):    "
      f"{over_answer}/{n_ref} = {over_answer/n_ref:.2f}")
```

Re-run. You now have the two chapter-7 metrics on your calibration set *for the floor you chose*. If both are zero on this tiny set, the floor is well-chosen for this corpus — treat it as a first draft that should be re-checked on a bigger labeled set (exercise-03's `eval/dataset.jsonl`).

### Checkpoint

- Your `FLOOR` is a number drawn from the gap in step 3's sorted table (or picked deliberately when the gap is not clean).
- Both counts and rates are printed. Ideally both are low; perfectly zero only means the calibration set was easy.
- The two cases are not symmetric: `over_answer` is almost always worse than `over_refuse` in a support/factual system, because an over-answer is a confident hallucination while an over-refuse is merely a "please rephrase."

## Step 5 — Validate by flipping one thing

Pick **one** of the following and re-run. Record in `NOTES.md` which variant you tried and what moved.

- **Raise the FLOOR by 0.05.** Which ANSWERABLE queries now get wrongly refused? Which REFUSAL queries get correctly refused that were not before? Is the trade worth it?
- **Use top-3 average instead of top-1 as the floor signal.** (`np.sort(scores)[-3:].mean()` instead of `float(np.max(scores))`.) Does the separation between the two labels widen or narrow? Chapter 6 speaks only of "the top hit"; this variant explores an alternative that some teams prefer.
- **Add two more topically adjacent REFUSAL queries** ("do you offer same-day delivery?", "can I schedule a return pickup?") and re-score. If they land *above* your chosen floor, the floor is too low. Chapter 7 says "add cases the system struggles with" — this is a tiny version of that.

### Checkpoint

- One variant was run and one or two numbers moved.
- In `NOTES.md`, you have stated whether the baseline floor still looks correct or needs to shift, and by how much.

## Step 6 — Write the debrief

In `NOTES.md`, answer in 6–10 sentences total:

1. State the FLOOR value you picked and the rule you used to pick it ("midpoint of the clean gap," "max REFUSAL + 0.02," "min ANSWERABLE − 0.02", etc.).
2. On your calibration set, what were `over_refuse_rate` and `over_answer_rate`? Which one would you tolerate higher in production for *this* use case, and why?
3. Chapter 6 says the floor "saves the model call" and "avoids tempting the model with irrelevant context." Pick one REFUSAL query whose top-1 doc was suspiciously plausible. What would the model have done with that context if the floor had not short-circuited? Argue in one or two sentences.
4. This calibration used 14 queries. Chapter 7 recommends 30–80 on a real eval set. Where in the pipeline should the floor live, and how often should it be re-calibrated — every ingest, every model change, every corpus refresh, or something else?

That write-up plus the sorted table from step 3 and the floor numbers from step 4 are the deliverable.

## Debrief

Three things to keep:

- A score floor is corpus-specific and model-specific. A floor calibrated on one corpus does not transfer to another; a floor calibrated on `text-embedding-3-small` is not the right floor for `voyage-3`.
- The two error rates — `over_refuse_rate` and `over_answer_rate` — trade against each other and against each other only. There is no universal "right" floor; there is a right floor *for your app's tolerance*.
- Floor calibration is a one-shot operation per `(corpus, embedding_model)` pair. Re-calibrate when either changes. When your corpus ages and new answerable topics drift in, your eval set — and therefore your floor — must track.

Everything else in the RAG query loop is more expensive than this move. Spend 30 minutes calibrating a floor before you spend a week re-prompting the answer model.

## Stretch

- **Use the eval set from exercise-03.** Replace the hand-rolled 14 queries with the `eval/dataset.jsonl` you built in exercise-03. Compute the same two rates against your chosen floor. The numbers get noisier (more cases) but more trustworthy (more cases).
- **Plot the distributions.** Install `matplotlib` and draw two overlaid histograms — top-1 scores for ANSWERABLE vs REFUSAL. The right floor is now visible rather than tabulated. Save the figure; it is the kind of artifact that belongs in a design doc.
- **Short-circuit in the real loop.** Wire the floor into your exercise-02 `answer(conn, question)`: if the top-1 is below `FLOOR`, return the refusal phrase and skip the `messages.create` call entirely. Compare per-query latency and token cost vs. the no-floor baseline over the 14 queries. Chapter 6 claims the floor saves both; confirm the numbers.
- **Per-topic floors.** Some corpora have one topic cluster that scores systematically lower than others (short FAQ entries vs long policy pages). Try computing a separate floor per `document_id` and routing based on which doc was top-1. Report whether the two-metric rates improve or whether you have overfit to the small set.
