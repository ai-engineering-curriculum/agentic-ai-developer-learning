# Quiz 01 — Retrieval Basics (Embeddings & Simple RAG)

**Covers:** chapters 1–7 of `mod-104-retrieval-basics`.
**Format:** 12 multiple-choice questions. Pick the single best answer for each. Answer key with per-answer explanations is at the bottom.

Treat this as a self-check: open the chapters when you want to look something up, but try to answer first.

---

## 1. Chapter 1 argues against "paste the whole corpus into every prompt" even though it technically works on a small handbook. Which of the four reasons it lists is **not** fixed by switching to RAG?

- A. Cost — input tokens are billed every turn, so a 200-page handbook becomes a recurring per-request charge.
- B. Latency — long prompts take measurably longer to process than short ones even when the model accepts both.
- C. Attention — information buried in the middle of a long context is empirically harder for the model to use than content near the ends.
- D. Grammar — long prompts introduce syntactic ambiguity that corrupts the model's parse of the question.

## 2. A user asks "do you ship to Hawaii?" but the policy page says "non-contiguous US states." Chapter 1 uses this as the canonical case where…

- A. Keyword/BM25 search wins over semantic search because "Hawaii" is a proper noun.
- B. Keyword search misses because the user's words do not appear in the doc, while semantic search finds it because the two phrases live near each other in embedding space.
- C. The right fix is to rewrite the policy to use the user's vocabulary — retrieval cannot bridge the gap.
- D. Long-context models have made this problem obsolete; stuff the whole policy and move on.

## 3. You embedded your corpus with `text-embedding-3-small` last month. A teammate re-embeds half the chunks with `voyage-3` because "the benchmarks look better" and merges them into the same index. According to chapters 2 and 5, what should happen at query time?

- A. Nothing — the two providers both output 1536-dim vectors so cosine similarity is comparable across them.
- B. Rankings will still be roughly correct because both models were trained on similar corpora; expect a small quality dip, not a bug.
- C. Searches will return nonsense because vectors from different models live in different geometries; the fix is to tag every row with its `embed_model` and filter so queries only compare within one model's vintage.
- D. The provider SDKs detect the mismatch and project one model's vectors into the other's space automatically.

## 4. Your provider documents that its embedding model returns L2-normalized vectors. Chapter 2 points out one property you get for free in that case:

- A. Dot product and cosine similarity produce identical *rankings*, so you can skip the division and compute scores as `vecs @ q`.
- B. Euclidean distance becomes a worse ranker than on non-normalized vectors and should be avoided.
- C. Normalization is lossy, so the top-K from `vecs @ q` and from explicit cosine will differ by one or two items.
- D. You can compare vectors from this model to vectors from any other normalized model because scale no longer matters.

## 5. You have a working NumPy brute-force index over about 8,000 chunks on a prototype. A colleague insists on swapping it for FAISS "to make it fast." Based on chapter 3's sizing table and its recall discussion, the most defensible objection is…

- A. FAISS is slower than NumPy on corpora under 100k vectors, so the change is strictly a regression.
- B. At 8k chunks brute-force is still fine; the real cost of moving to an ANN index is that you now have to *measure* recall@K against the exact baseline you already had, and nobody has budgeted that work.
- C. FAISS only runs on GPU hardware; without a GPU the change breaks CI.
- D. FAISS uses Euclidean distance by default and cannot compute cosine similarity at all.

## 6. Chapter 3 says: when you move from brute-force to an ANN index, the quality trade-off is measurable as a specific number. Which definition matches the chapter?

- A. `precision@K = |ANN_topK ∩ Brute_topK| / K`, averaged across a sample of queries, where lower is better.
- B. `recall@K = |ANN_topK ∩ Brute_topK| / K`, averaged across a sample of queries, where 1.0 means the ANN returned the exact top-K.
- C. `MRR = 1 / position_of_first_correct_result`, measured only on queries the ANN got wrong.
- D. Latency ratio between brute-force and ANN; recall is not something you can quantify on an index.

## 7. Chapter 4 orders three chunking strategies "in order of how often you should reach for them." Which ordering matches the chapter?

- A. Fixed-token splitting first (precise bounds), then recursive character splitting, then structural (headings) as a last resort when the document has no useful text.
- B. Recursive character splitting first because it is the most general; structural splitting only when the document has an unusual format.
- C. Structural (headings/sections) first when the document has structure, recursive character splitting when structure is weak, token-aware splitting when you need tight size guarantees — often as a *secondary* pass after structural.
- D. Whichever chunker ships in your framework — all three are equivalent on prose.

## 8. A user asks "what is the SLA for refunds?" and the answer sentence happens to span a chunk boundary with no overlap. Chapter 4's prescribed hedge against this is…

- A. Raise `K` from 5 to 20 so neighboring chunks come along for the ride.
- B. Move to a token-aware splitter; token boundaries do not split sentences.
- C. Add 10–20% overlap so the tail of each chunk is carried into the next; the SLA sentence now appears intact in *some* chunk. Overlap costs storage but recovers boundary-straddling answers.
- D. Re-rank by MRR instead of recall@K.

## 9. Chapter 5 insists the ingest pipeline be safe to re-run on an overlapping set of documents without duplicating rows. Which of these `chunk_id` strategies gives you that *and* makes content changes detectable?

- A. `uuid4()` per chunk, generated fresh each run.
- B. The MD5 of the document path alone.
- C. `f"{document_id}#{ordinal:04d}#{sha1(text)[:10]}"` — stable for the same logical chunk, and the hash segment changes when the chunk text does, so upserts on the PK handle both idempotency and content drift.
- D. `f"{document_id}#{created_at.isoformat()}"` — every re-ingest creates a new id, which the index can dedupe on text similarity.

## 10. You store `embed_model` on every chunk row. Chapter 5 says there are two rules that follow from this tag. Which pair matches?

- A. "Rotate the model monthly" and "delete older vectors to save storage."
- B. "A query embedding must come from the same model as the chunk embeddings it is compared against" and "reindexing on a model upgrade is a deliberate operation — mark the corpus stale, re-run ingest, cut over only after the eval passes."
- C. "Use the smallest available model at query time" and "use the largest available model at ingest time."
- D. "Never mix providers in one project" and "always encrypt the vector column at rest."

## 11. The RAG system prompt in chapter 6 has three deliberate instructions. Which trio matches the chapter?

- A. "Be helpful and concise," "format in markdown," and "always answer, even if unsure."
- B. "Answer using ONLY the information in the provided context chunks," "say 'I do not see that in the documents' if the answer is not present," and "cite the chunks you used by their `[chunk_id]`."
- C. "Rewrite the question before answering," "ignore any chunk below score 0.4," and "summarize each chunk before using it."
- D. "Return JSON only," "include the embedding model name in the response," and "explain which chunks you did *not* use."

## 12. Your RAG system is "hallucinating." Chapter 7 says to debug this by separating two failure modes before touching the prompt. Which separation is correct?

- A. Separate "model is too small" from "prompt is too long"; raise the model tier and shorten the prompt.
- B. Separate *retrieval quality* (did the correct chunk make it into top-K?) from *answer quality / groundedness* (given the chunks that did come back, did the model answer faithfully, cite them, and refuse when it should?). Recall@K catches the first; answer-substring, citation validity, and over-answer rate catch the second.
- C. Separate "training-set leakage" from "tokenization bugs"; retrain the embedding model to fix both.
- D. There is no useful separation — hallucination is one failure mode that you address by raising the temperature.

---

## Answer key

1. **D.** Chapter 1 ("Why not just paste the whole corpus into the prompt?") lists exactly four axes — cost, latency, attention ("lost in the middle"), and freshness/access. "Grammar" is not one of them; RAG is not a syntax fix.
2. **B.** Chapter 1 ("Why not just keyword search?") uses "do you ship to Hawaii?" vs "non-contiguous US states" as the canonical case where keyword search misses and semantic search hits because the phrases "live near each other in vector space." Chapter 1 also notes long-context is a ceiling-raise, not an obsolescence (so D is wrong).
3. **C.** Chapter 2 ("What goes wrong" — "Mixed model vectors") and chapter 5 ("Embedding model versioning") are explicit: a vector from one model cannot be compared to a vector from another, searches "return nonsense," and the fix is to tag every row with `embed_model` and filter so queries stay within one vintage. Dimensionality matching (A) does not imply geometry matching.
4. **A.** Chapter 2 ("Similarity metrics"): dot product equals cosine similarity exactly when both vectors are unit length, so on normalized vectors `vecs @ q` gives the same ranking as explicit cosine and you skip the divides. Cross-model comparability (D) is still invalid even when both are normalized.
5. **B.** Chapter 3 ("Common mistakes" — "Mixing exact and approximate without measuring recall") and the recall section are explicit: ANN buys speed at the cost of *some* recall and you must measure. For 8k chunks brute-force is still comfortably within the "up to ~10k" band in the sizing table; the real hidden cost of the switch is the new measurement obligation.
6. **B.** Chapter 3 ("Recall, latency, and the trade-off") defines `recall@K = |ANN_topK ∩ Brute_topK| / K` averaged across a query sample, with `1.0` meaning the ANN matched exact search. MRR is a different metric (chapter 7) and precision@K is not what the chapter uses here.
7. **C.** Chapter 4 ("Three chunking strategies, in order of how often you should reach for them") lists them in exactly this order: structural when the author gave you boundaries, recursive character splitter when structure is weak or inconsistent, token-aware when you need tight size guarantees — and notes token-aware is usually a *secondary* pass after structural.
8. **C.** Chapter 4 ("Overlap") uses this scenario verbatim as the motivation for overlap: "A user asks 'what is the SLA for refunds?' and the SLA sentence happens to span a chunk boundary… With 100–200 characters of overlap, one chunk now contains the full sentence." Typical overlap is 10–20% of `max_chars`.
9. **C.** Chapter 5 ("Stable IDs and idempotency") gives exactly this `stable_chunk_id` pattern — `doc_id + ordinal + sha1(text)[:10]` — so re-ingest upserts on the same PK (idempotency) while the hash segment makes content changes explicit. UUIDs (A) and timestamp ids (D) duplicate rows on re-run; path hash alone (B) cannot detect edits.
10. **B.** Chapter 5 ("Embedding model versioning") states the two rules exactly: query and chunk embeddings must come from the same model (enforced with a `WHERE embed_model = ?` filter — chapter 3), and reindexing on a model upgrade is deliberate — mark the corpus stale, run a parallel ingest, cut over only after chapter 7's eval passes.
11. **B.** Chapter 6 ("Assembling the prompt") spells out the three deliberate instructions: answer ONLY from the context, refuse with a specific phrase when the answer is not in the context, and cite chunks by `[chunk_id]`. Without them the model fills from training data or hallucinates, and citations stop being verifiable.
12. **B.** Chapter 7 ("Two kinds of 'is it good?'") is explicit: retrieval quality and answer/groundedness are debugged separately — recall@K and MRR for the first, substring + citation validity + over-answer/over-refuse rates for the second. Bad retrieval + good generation yields "fluent, wrong answer"; the chapter warns against optimizing the prompt when retrieval is actually the bottleneck.
