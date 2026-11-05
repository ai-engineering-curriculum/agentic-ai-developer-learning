# Lab 01 — Cross-Model Vector Poisoning

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 2 — Embeddings and Vector Space](../02-embeddings-and-vector-space.md) ("What goes wrong" / "Mixed model vectors"), [chapter 5 — The Indexing Pipeline](../05-indexing-pipeline.md) ("Embedding model versioning")

## Objective

Chapter 2 claims that a vector from `text-embedding-3-small` "cannot be compared to" a vector from a different model — "different geometries, different training, different scales" — and that comparing across models returns "nonsense" silently. Chapter 5 turns that into a rule: tag every chunk with `embed_model` and refuse to mix at query time. In this lab you will reproduce the failure on a 10-line corpus so you have seen it with your own eyes, and then you will add the one-line `WHERE embed_model = ?` guard that prevents it.

By the end you will have:

- A `cross_model.py` that embeds the same 10 docs with two different models (default: `text-embedding-3-small` and `text-embedding-3-large`), runs the same three queries against each, and prints the top-3 side by side.
- A deliberately broken variant where a query embedded with one model is scored against chunks embedded with the other — producing a visibly degraded ranking.
- A one-line guard added to the search function that filters by `embed_model`, and a demo that it refuses to run the mismatched query at all.
- A short `NOTES.md` with the top-1 score gap you observed on the mixed run.

> This lab complements [exercise-01](../exercises/exercise-01-embeddings-similarity.md), which builds the single-model embedding primitive, and [exercise-02](../exercises/exercise-02-simple-rag-pipeline.md), which stores vectors with their model tag. Here we isolate the one specific foot-gun those two assume away.

## Prerequisites

- Finished chapter 2 and chapter 5.
- A funded OpenAI API key: `OPENAI_API_KEY` exported. (OpenAI is used because `text-embedding-3-small` and `text-embedding-3-large` are two models you can reach with one key and they return *different dimensionalities* — 1536 vs 3072 — which also shows up visibly in the failure. If you prefer another provider, pick any two of its models with different dimensions and substitute names below.)
- Python 3.10+ with `openai` and `numpy` installed.
- Budget: two batches of 10 short strings plus six single-query embeddings. Cents.

## Step 1 — Pin two models and embed the corpus with each

Pick two exact model IDs from the same provider. The rest of the lab assumes OpenAI's small/large pair; substitute what your account can call.

Save as `cross_model.py`:

```python
import numpy as np
from openai import OpenAI

client = OpenAI()

MODEL_A = "text-embedding-3-small"   # dim 1536
MODEL_B = "text-embedding-3-large"   # dim 3072

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


def embed(texts, model):
    resp = client.embeddings.create(model=model, input=texts)
    vecs = np.array([d.embedding for d in resp.data], dtype=np.float32)
    norms = np.linalg.norm(vecs, axis=1, keepdims=True)
    return vecs / np.clip(norms, 1e-12, None)


vecs_a = embed(DOCS, MODEL_A)
vecs_b = embed(DOCS, MODEL_B)
print(f"{MODEL_A}: shape {vecs_a.shape}")
print(f"{MODEL_B}: shape {vecs_b.shape}")
```

Run it:

```bash
python cross_model.py
```

### Expected output (shape)

```
text-embedding-3-small: shape (10, 1536)
text-embedding-3-large: shape (10, 3072)
```

Two embedding matrices, same 10 rows, different widths. The width mismatch alone is the first thing that would make a mixed-model index fail — `vecs @ q` with mismatched dims is a shape error, which is actually the *best-case* failure mode here. The nastier case comes next, when both sides happen to share a dimension.

### Checkpoint

- Both shapes print.
- The first column of `vecs_a` and `vecs_b` are *not* equal. The vectors live in different coordinate systems; there is no coordinate-by-coordinate meaning to compare.

## Step 2 — Run three queries on each index separately (the control)

Still in `cross_model.py`, append:

```python
QUERIES = [
    "how long do I have to return something?",
    "do you ship to non-mainland states?",
    "when do I get my money back after a refund?",
]


def search(doc_vecs, docs, q_vec, k=3):
    scores = doc_vecs @ q_vec
    top = np.argsort(-scores)[:k]
    return [(float(scores[i]), docs[i]) for i in top]


for q in QUERIES:
    q_a = embed([q], MODEL_A)[0]
    q_b = embed([q], MODEL_B)[0]
    print(f"\n> {q}")
    print(f"  [{MODEL_A}]")
    for s, d in search(vecs_a, DOCS, q_a):
        print(f"    {s:.3f}  {d}")
    print(f"  [{MODEL_B}]")
    for s, d in search(vecs_b, DOCS, q_b):
        print(f"    {s:.3f}  {d}")
```

Re-run. Each query now prints two top-3 lists — one per model — both self-consistent (query and chunks embedded with the same model).

### Expected output (shape)

```
> how long do I have to return something?
  [text-embedding-3-small]
    0.6xx  We accept returns of unused items within 30 days of delivery.
    0.4xx  All electronics must be returned in their original packaging.
    0.4xx  Refunds appear on the original payment method within 5–10 business days.
  [text-embedding-3-large]
    0.5xx  We accept returns of unused items within 30 days of delivery.
    0.4xx  Refunds appear on the original payment method within 5–10 business days.
    0.3xx  All electronics must be returned in their original packaging.
```

Scores will differ (different models, different scales) but the *top-1 should be the same document* for every query. This is your baseline for "the system works."

### Checkpoint

- For each of the three queries, the top-1 document is the same under both models.
- The actual numeric scores differ between the two models. Note one score pair in `NOTES.md` — the fact that scores do not line up is already a hint that the two spaces are not the same space.

## Step 3 — Deliberately mismatch a query and the index

This is the broken path. We want a query embedded with model A, scored against chunks embedded with model B — but they have different dimensionalities, so step 1's exact pair will error out. To reproduce the nastier *silent* failure, pick two models that happen to share a dimension. The simplest way is to pass OpenAI's `dimensions=1536` parameter to `text-embedding-3-large`, forcing it down to the same width as `text-embedding-3-small`:

```python
def embed_large_1536(texts):
    resp = client.embeddings.create(model=MODEL_B, input=texts, dimensions=1536)
    vecs = np.array([d.embedding for d in resp.data], dtype=np.float32)
    norms = np.linalg.norm(vecs, axis=1, keepdims=True)
    return vecs / np.clip(norms, 1e-12, None)


vecs_b_1536 = embed_large_1536(DOCS)
print(f"\n{MODEL_B} @ dim=1536: shape {vecs_b_1536.shape}")

# Now embed each query with MODEL_A (1536) and score against vecs_b_1536.
# Shapes line up. Nothing errors. Rankings should still be wrong.
print(f"\n--- MISMATCH: query with {MODEL_A}, chunks with {MODEL_B}@1536 ---")
for q in QUERIES:
    q_a = embed([q], MODEL_A)[0]
    print(f"\n> {q}")
    for s, d in search(vecs_b_1536, DOCS, q_a):
        print(f"    {s:.3f}  {d}")
```

<!-- needs-research: confirm OpenAI's `dimensions` parameter is still available and that truncating `text-embedding-3-large` to 1536 does not implicitly re-normalize into `text-embedding-3-small`'s geometry; the chapter 2 claim is that even same-dim vectors from different models are not comparable, but the exact degree of degradation here is empirical. Students should report what they see. -->

Re-run.

### Expected output (shape)

```
--- MISMATCH: query with text-embedding-3-small, chunks with text-embedding-3-large@1536 ---

> how long do I have to return something?
    0.0xx  <some document, not necessarily the returns one>
    0.0xx  <another document>
    0.0xx  <a third document>
```

The code runs. There is no error. The scores are typically much lower than in step 2 (often near zero because the two spaces are effectively uncorrelated at the coordinate level), and the top-1 is often *not* the document that the correctly-matched queries returned. This is the "silent and corrosive" failure that chapter 2's "Mixed model vectors" bullet warns about.

### Checkpoint

Open your output and answer in `NOTES.md`:

1. For each of the three queries, did the mismatched top-1 match the step-2 top-1 for that query? Any disagreement is the lab's main finding.
2. What is the top-1 score for one mismatched query, and what is the top-1 score for the same query in step 2? Report both numbers. The gap (often large) is the signal that something is off — but note that nothing in the code made that gap visible on its own.
3. Imagine this mix had happened in production: half your chunks ingested with model A, half with model B after a "quick upgrade." A user complains that answers are wrong. What single piece of data would you want to see to diagnose it? (Chapter 5 names the field.)

## Step 4 — Add the one-line guard from chapter 5

The fix from chapter 5 ("Embedding model versioning"): every chunk row carries `embed_model`, and queries filter on it. To simulate that, store each doc with its model tag, and gate the search:

```python
# A tiny in-memory "index" that mirrors the WHERE embed_model = ? pattern
index = []  # list of (model, vec, text)
for v, d in zip(vecs_a,       DOCS): index.append((MODEL_A, v, d))
for v, d in zip(vecs_b_1536,  DOCS): index.append((MODEL_B, v, d))


def search_guarded(index, q_vec, q_model, k=3):
    rows = [(v, t) for (m, v, t) in index if m == q_model]
    if not rows:
        raise ValueError(f"No chunks for embed_model={q_model!r}")
    doc_vecs = np.stack([v for v, _ in rows])
    texts    = [t for _, t in rows]
    scores   = doc_vecs @ q_vec
    top = np.argsort(-scores)[:k]
    return [(float(scores[i]), texts[i]) for i in top]


print("\n--- GUARDED: query embed_model must match chunk embed_model ---")
for q in QUERIES:
    q_a = embed([q], MODEL_A)[0]
    print(f"\n> {q}  (query model: {MODEL_A})")
    for s, d in search_guarded(index, q_a, q_model=MODEL_A):
        print(f"    {s:.3f}  {d}")

# And prove the guard refuses the mismatch:
print("\n--- GUARDED with wrong model name — should raise ---")
try:
    search_guarded(index, embed([QUERIES[0]], MODEL_A)[0], q_model="nonexistent-model")
except ValueError as e:
    print(f"OK, raised: {e}")
```

Re-run. The guarded path returns the same (good) top-1 as step 2, and the mismatched path refuses to run at all.

### Checkpoint

- The guarded output for each query matches the step-2 top-1 for that query.
- The "wrong model name" case raises instead of returning a silent wrong answer.
- In your real ingest (exercise-02), the equivalent of this guard is `WHERE embed_model = ?` in the SQLite search, pinned from the same config value you pass to the embedding call. Add it there if you have not yet.

## Step 5 — Write the debrief

In `NOTES.md`, answer in 6–10 sentences total:

1. In step 3, what was the single most surprising thing about how the failure *looked*? (Shape of the scores, change in top-1, no exception, something else.)
2. Chapter 2 says mixed-model vectors are "silent and corrosive." In your run, what was silent about it — i.e., which observable (score, top-1 order, exception, log line) would have told you something was wrong *without* step 2 as a baseline to compare against?
3. Translate the lesson into a concrete code check: in your own codebase, where would the `embed_model` filter live, and what would it do when the filter rejected every row? (Return empty? Raise? Return a fallback refusal?) Pick one and argue for it.

That short write-up plus the four output blocks (steps 1, 2, 3, 4) are the deliverable.

## Debrief

Three things to take with you:

- A vector is only meaningful *relative to the model that produced it*. Dimension-matching is not geometry-matching.
- The failure mode of a mixed-model index is silent — the math still works, the scores still rank, the top-K still returns something. That "something" is just not what you asked for.
- The fix is cheap and permanent: tag every row with `embed_model` and filter on it at query time. One column, one `WHERE` clause.

Everything downstream in this module — the FAISS index in chapter 3, the ingest pipeline in chapter 5, the query loop in chapter 6, the eval harness in chapter 7 — assumes this invariant holds. If it does not, every metric lies.

## Stretch

- **Same model, different versions.** If your provider publishes a dated snapshot for its embedding model (e.g., `voyage-3@YYYY-MM-DD`), re-run step 3 with two snapshots of the *same* model instead of two models. The failure is usually milder but present. Chapter 2 warns that "providers occasionally retrain."
- **Use `input_type`.** If you are on Cohere or Voyage, embed the chunks with `input_type="search_document"` and the queries with `input_type="search_query"`. Re-run step 2 and note whether top-1 scores shift. Chapter 2 flags this as a "small but real win for retrieval." Note this is *not* the same foot-gun as cross-model — it is a free-money micro-optimization on the same model.
- **Rescue the broken index.** You discover in production that 40% of your rows are tagged `text-embedding-3-small` and 60% are the untagged legacy `voyage-2`. You cannot afford to re-embed the whole corpus today. Sketch a two-week migration plan using the two-index pattern from chapter 5 ("run the ingest from scratch (or in parallel into a new index), and cut over only when the new index passes the eval"). What does CI (chapter 7) need to know during the cutover?
