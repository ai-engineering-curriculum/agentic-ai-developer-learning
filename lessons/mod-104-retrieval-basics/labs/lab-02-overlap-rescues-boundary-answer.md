# Lab 02 — Overlap Rescues a Boundary-Straddling Answer

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 4 — Chunking Documents](../04-chunking-documents.md) ("Overlap")

## Objective

Chapter 4 says overlap is "the chunker's hedge against splitting an answer in half." The canonical example: a user asks "what is the SLA for refunds?" and the answer sentence happens to span a chunk boundary; with no overlap, neither chunk is a great hit, but with 100–200 characters of overlap one chunk contains the full sentence. In this lab you will build the minimum reproduction — one document, one question, one sentence that straddles the boundary — and watch the retrieval flip from miss to hit as you move overlap from `0` to `150`.

By the end you will have:

- A `boundary_doc.md` where the answer sentence is positioned so the recursive chunker from chapter 4 splits it.
- A `overlap_demo.py` that runs the same chunker twice — once with `overlap=0`, once with `overlap=150` — embeds both chunk sets, and shows the retrieval top-3 for the same query against each.
- A visible score gap: the correct chunk's top-1 score should be meaningfully higher in the overlap=150 run than in the overlap=0 run, with the same query.
- A short `NOTES.md` noting the two top-1 scores and which chunk index contained the straddling sentence in each run.

> This lab complements [exercise-03](../exercises/exercise-03-chunking-and-indexing.md), which sweeps chunkers and K against a labeled eval set. That exercise answers "which chunker wins overall." This lab isolates *one* chunker knob (overlap) against *one* adversarial document, so you can see the specific failure mode overlap fixes.

## Prerequisites

- Finished chapter 4.
- Finished [exercise-01](../exercises/exercise-01-embeddings-similarity.md) (you have a working normalized-embedding + dot-product search).
- A funded embedding API key (`OPENAI_API_KEY` for the defaults below; any provider works — see chapter 2).
- Python 3.10+ with `openai` and `numpy`. No vector library needed.
- Budget: two batches of a handful of chunks plus one query embedding. Cents.

## Step 1 — Craft a document whose answer sentence straddles the boundary

The recursive splitter from chapter 4 walks separators from biggest to smallest (`\n\n`, `\n`, `. `, ` `). We want a document where, at the max-chars limit we pick, the splitter *has* to cut inside the answer sentence — meaning there is no paragraph or sentence break to fall on first.

Save as `boundary_doc.md`:

```markdown
# Returns, Refunds, and Timing

Our returns process is designed to be straightforward. We accept returns on most items within 30 days of delivery, provided the item is unused and in its original packaging. Electronics have additional requirements due to compatibility testing that we must perform on each unit returned, especially for items that were opened, used briefly, and then returned for reasons unrelated to defects or damage during shipping; please see clause three of the extended warranty terms attached to each order confirmation email for the full policy details on electronics returns. Once the returned item arrives at our fulfillment center and passes inspection, the refund is issued to the original payment method and the service level agreement for refunds is that funds appear within five to ten business days of inspection completion for standard payment methods and up to fourteen business days for store credit, gift cards, or third-party installment plans that must be refunded through the original provider; this timeline is a commitment we hold ourselves to and publish on every order confirmation.
```

A couple of things to notice about the shape:

- There is exactly one paragraph. The only paragraph separator `\n\n` after the heading is at the top. That forces the recursive splitter to drop down to sentence (`. `) and then to space (` `) when it needs smaller pieces.
- The answer to "what is the SLA for refunds?" is embedded inside the final long sentence: "the service level agreement for refunds is that funds appear within five to ten business days of inspection completion…". That phrasing straddles the mid-point of the paragraph on purpose.

### Checkpoint

- Open the file. It should be one heading and one long paragraph of roughly 1,000–1,200 characters.
- The phrase "service level agreement for refunds is that funds appear within five to ten business days" should appear roughly 55–80% of the way through the paragraph, not near either end.

## Step 2 — Write the recursive splitter and the demo harness

Save as `overlap_demo.py`:

```python
import numpy as np
from openai import OpenAI

client = OpenAI()
MODEL = "text-embedding-3-small"


def recursive_split(text, max_chars=400, overlap=0,
                    separators=("\n\n", "\n", ". ", " ")):
    """From chapter 4, lightly simplified: split greedily on the biggest
    separator that produces small-enough chunks, then carry `overlap`
    characters of tail into the next chunk."""
    if len(text) <= max_chars:
        return [text]
    for sep in separators:
        if sep in text:
            parts = text.split(sep)
            chunks, buf = [], ""
            for p in parts:
                join = (buf + sep + p) if buf else p
                if len(join) <= max_chars:
                    buf = join
                else:
                    if buf:
                        chunks.append(buf)
                    if len(p) > max_chars:
                        chunks.extend(
                            recursive_split(p, max_chars, overlap, separators[1:])
                        )
                        buf = ""
                    else:
                        buf = p
            if buf:
                chunks.append(buf)
            if overlap <= 0 or len(chunks) <= 1:
                return chunks
            out = [chunks[0]]
            for i in range(1, len(chunks)):
                out.append(chunks[i - 1][-overlap:] + chunks[i])
            return out
    return [text]


def embed(texts):
    resp = client.embeddings.create(model=MODEL, input=texts)
    vecs = np.array([d.embedding for d in resp.data], dtype=np.float32)
    norms = np.linalg.norm(vecs, axis=1, keepdims=True)
    return vecs / np.clip(norms, 1e-12, None)


def search(doc_vecs, docs, q_vec, k=3):
    scores = doc_vecs @ q_vec
    top = np.argsort(-scores)[:k]
    return [(i, float(scores[i]), docs[i]) for i in top]


with open("boundary_doc.md") as f:
    DOC = f.read()

QUERY = "what is the SLA for refunds?"
MAX_CHARS = 400   # tuned to force a split inside the answer sentence

for overlap in (0, 150):
    chunks = recursive_split(DOC, max_chars=MAX_CHARS, overlap=overlap)
    print(f"\n=== overlap={overlap}  ->  {len(chunks)} chunks "
          f"(sizes: {[len(c) for c in chunks]}) ===")
    for i, c in enumerate(chunks):
        preview = c.replace("\n", " ")[:90]
        print(f"  [{i}] {preview}...")
    doc_vecs = embed(chunks)
    q_vec = embed([QUERY])[0]
    print(f"\n  query: {QUERY!r}")
    for idx, score, text in search(doc_vecs, chunks, q_vec):
        head = text.replace("\n", " ")[:80]
        print(f"    chunk[{idx}] score={score:.3f}  {head}...")
```

Run it:

```bash
python overlap_demo.py
```

### Expected output (shape)

```
=== overlap=0  ->  3 chunks (sizes: [...]) ===
  [0] # Returns, Refunds, and Timing  Our returns process is designed...
  [1] Electronics have additional requirements due to compatibility...
  [2] Once the returned item arrives at our fulfillment center and passes...

  query: 'what is the SLA for refunds?'
    chunk[2] score=0.4xx  Once the returned item arrives at our fulfillment center...
    chunk[1] score=0.3xx  Electronics have additional requirements due to...
    chunk[0] score=0.2xx  # Returns, Refunds, and Timing  Our returns process...

=== overlap=150  ->  3 chunks (sizes: [...]) ===
  [0] # Returns, Refunds, and Timing  Our returns process is designed...
  [1] ...inspection that we must perform on each unit returned...
  [2] ...service level agreement for refunds is that funds appear within...

  query: 'what is the SLA for refunds?'
    chunk[2] score=0.5xx  ...service level agreement for refunds is that funds appear...
    chunk[1] score=0.4xx  ...
    chunk[0] score=0.2xx  ...
```

The exact scores will vary run to run — what you are looking for is:

- The number and sizes of chunks are similar between the two runs (overlap adds ~150 chars to all but the first).
- The top-1 chunk in `overlap=150` contains the straddling phrase "service level agreement for refunds is that funds appear" *intact*.
- The top-1 chunk in `overlap=0` either does not contain that phrase at all (the answer sentence was split) or contains only half of it — and the top-1 score is noticeably lower than in the overlap run.

<!-- needs-research: the exact top-1 score delta between overlap=0 and overlap=150 on this document is empirical and depends on the embedding model and the current snapshot. Students should report what they observe rather than target a canned number. -->

### Checkpoint

Answer in `NOTES.md`:

1. How many chunks did each run produce? Record the two chunk-count-and-size pairs.
2. For the `overlap=0` run, which chunk index was top-1? Does it contain the phrase "service level agreement for refunds is that funds appear"? If not, which chunk *does* contain it, and what rank did it come back at?
3. For the `overlap=150` run, which chunk is top-1, and does it now contain that phrase in a single contiguous span?
4. What are the two top-1 cosine scores? Report both. The gap is the main finding.

## Step 3 — Prove the fix is specific, not general

Overlap is not magic — chapter 4 says "zero overlap when chunks are already short (under ~150 tokens) and self-contained" and "higher overlap only when you have measured a recall problem on boundary-straddling answers." To see this, run a query that does *not* straddle the boundary and confirm overlap barely moves the score.

Add at the bottom of `overlap_demo.py`:

```python
NON_STRADDLING_QUERY = "do electronics have special return requirements?"
for overlap in (0, 150):
    chunks = recursive_split(DOC, max_chars=MAX_CHARS, overlap=overlap)
    doc_vecs = embed(chunks)
    q_vec = embed([NON_STRADDLING_QUERY])[0]
    print(f"\n[overlap={overlap}]  non-straddling query: {NON_STRADDLING_QUERY!r}")
    for idx, score, text in search(doc_vecs, chunks, q_vec, k=1):
        head = text.replace("\n", " ")[:80]
        print(f"    chunk[{idx}] score={score:.3f}  {head}...")
```

Re-run.

### Expected output (shape)

```
[overlap=0]  non-straddling query: 'do electronics have special return requirements?'
    chunk[1] score=0.5xx  Electronics have additional requirements due to compatibility...

[overlap=150]  non-straddling query: 'do electronics have special return requirements?'
    chunk[1] score=0.5xx  ...due to compatibility testing that we must perform on each unit...
```

The top-1 chunk index should be the same in both runs, and the two top-1 scores should be very close (same sentence, same vector, modulo the extra 150 characters of preamble carried in). The lesson: overlap is a boundary-answer fix, not a free score boost.

### Checkpoint

- For the non-straddling query, the two top-1 scores differ by a small amount (typically well under 0.05). Record both.
- If the non-straddling scores differ more than the straddling-query scores did, the lab did not reproduce. Re-check that the straddling phrase in `boundary_doc.md` really does land near the middle of the paragraph.

## Step 4 — Write the debrief

In `NOTES.md`, answer in 6–10 sentences total:

1. State the top-1 score gap between `overlap=0` and `overlap=150` on the straddling query, and the top-1 score gap on the non-straddling query. Which gap is bigger?
2. In the `overlap=0` run, did the correct chunk make the top-3 at all, or did it fall out entirely? Translate your answer into the chapter-7 language: was the failure in *recall@3* or *rank within top-3*?
3. Chapter 4 says overlap costs storage — 20% overlap means you embed and store ~1.2× the corpus. Given what you saw, is that storage cost worth paying unconditionally on a prose corpus, worth paying only when you have measured a recall problem, or never? Argue briefly.
4. Chapter 4 also warns against the opposite extreme: "overlap=30%+ only when you have measured a recall problem." Based on your single-document experiment, could you argue for 300-char overlap on this document? What would it cost, and what would it buy?

That write-up plus the two output blocks (steps 2 and 3) are the deliverable.

## Debrief

The one thing to take with you: overlap is not a tuning parameter you set by taste. It is a specific fix for a specific failure (answer sentence straddles a chunk boundary), with a cost (storage and embedding spend) and a diminishing return (adding more does not help queries whose answer already sits inside a chunk). The chapter-7 workflow is the right way to decide: label a few boundary-adjacent questions, sweep `overlap ∈ {0, 100, 200}`, pick the row that wins on recall@K. This lab was a hand-crafted single case of exactly that sweep — it existed so you have seen the mechanism before you measure it at scale.

## Stretch

- **Shift `MAX_CHARS` instead of `overlap`.** Keep `overlap=0` and sweep `max_chars ∈ {300, 500, 800}`. For some sizes, the straddling phrase is *inside* a single chunk by accident and the problem disappears. That is the uncomfortable truth chapter 4 implies: the "right" chunk size is corpus-dependent, which is why chapter 7 sweeps.
- **Add a second straddling question.** Insert a second long sentence into `boundary_doc.md` positioned to straddle a *different* boundary at `MAX_CHARS=400`. Re-run step 2. Can one `overlap` value rescue both answers? What is the smallest value that does?
- **Compare to a structural splitter.** Add a second markdown heading mid-document (`## Refund Timeline`). Re-chunk by heading. Do you still need overlap? Chapter 4 ranks structural splitting ahead of the recursive splitter for exactly this reason; this stretch is the empirical check.
