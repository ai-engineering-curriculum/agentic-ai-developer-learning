# Lab 02 — Score in the Observation Triggers a Second Swing

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 4 — Retrieval as a Tool](../04-retrieval-as-a-tool.md) ("Iterative retrieval — the model gets a second swing", "The tool definition" point 4)

## Objective

Chapter 4 makes a specific, cheap claim: if your `search_kb` observations include the similarity score, the agent will re-search on weak hits; if they don't, it can't tell the difference between a strong and weak result and will answer with the first thing it saw. *"Including the score is the smallest change that enables this behavior."*

In this lab you will prove the claim on a single prompt and a tiny corpus. You will stand up two near-identical `search_kb` tools — one that returns `(chunk_id, section_title, excerpt)` and one that also returns `score` — then drive both against a borderline query whose top hit is deliberately weak. The expected result is that the no-score version answers after one retrieval with a mediocre citation, and the with-score version issues a second, sharper query that returns a better chunk.

By the end you will have:

- A `score_demo.py` that stands up a 10-chunk in-memory index and two wrapper tools, `search_kb_no_score` and `search_kb_with_score`.
- A `trace.jsonl` with 2 rows (same prompt × two tool variants) recording every `search_kb*` call, its query text, and the retrieved chunks.
- A short `NOTES.md` pinning the top-1 score on the first retrieval and the model's two queries (if it issued a second one under the with-score variant).

> This lab complements [exercise-02](../exercises/exercise-02-tool-using-agent.md), which wires the full `mod-104` retriever in alongside action tools and drives a routing suite. Exercise-02 expects the score to be present and asks you to iterate on retrieval quality across many prompts. Here we isolate the single output-shape change that unlocks iterative retrieval, so you can see what the score *alone* earns before bundling it into the broader build.

## Prerequisites

- Finished chapters 1–4.
- A funded Anthropic *or* OpenAI API key. The code shows the Anthropic shape; OpenAI is a one-line swap.
- Python 3.10+ with `anthropic` (or `openai`), `openai` (for a cheap embedding call), and `numpy` installed.
- Budget: 1 batch of ~10 embeddings, 2 query embeddings, and up to 4 short tool-using calls at `max_tokens=512`. Cents.

## Step 1 — Stand up a 10-chunk in-memory index

The corpus is deliberately small and contains one *weak* match for the lab's borderline query plus one *strong* match. Save as `corpus.py`:

```python
import numpy as np
from openai import OpenAI

_oa = OpenAI()
EMBED_MODEL = "text-embedding-3-small"


CHUNKS = [
    # (chunk_id, section_title, text)
    ("kb#01", "returns-general",
     "We accept returns of unused items within 30 days of delivery."),
    ("kb#02", "returns-electronics",
     "Opened electronics: 14-day return window with a 15% restocking fee "
     "(clause 3). Sealed items follow the general 30-day policy."),
    ("kb#03", "returns-final-sale",
     "Final-sale items are not returnable and no refund will be issued."),
    ("kb#04", "shipping-contiguous",
     "Standard shipping to the contiguous US: 2–4 business days."),
    ("kb#05", "shipping-noncontiguous",
     "Orders to Hawaii and Alaska (non-contiguous US states): 5–7 business days."),
    ("kb#06", "shipping-international",
     "International orders ship via UPS Worldwide with tracking included."),
    ("kb#07", "refunds-timing",
     "Refunds appear on the original payment method within 5–10 business days."),
    ("kb#08", "refunds-damaged",
     "For damaged goods we refund the full purchase price and return shipping."),
    ("kb#09", "support-hours",
     "Our customer service team is available Monday–Friday, 9am–6pm Pacific."),
    ("kb#10", "support-contact",
     "We do not currently offer phone support; please use the contact form."),
]


def _embed(texts: list[str]) -> np.ndarray:
    resp = _oa.embeddings.create(model=EMBED_MODEL, input=texts)
    v = np.array([d.embedding for d in resp.data], dtype=np.float32)
    n = np.linalg.norm(v, axis=1, keepdims=True)
    return v / np.clip(n, 1e-12, None)


_DOC_VECS = _embed([c[2] for c in CHUNKS])


def search(query: str, k: int = 5) -> list[dict]:
    q = _embed([query])[0]
    scores = _DOC_VECS @ q
    top = np.argsort(-scores)[:k]
    return [
        {
            "chunk_id":      CHUNKS[i][0],
            "section_title": CHUNKS[i][1],
            "excerpt":       CHUNKS[i][2],
            "score":         round(float(scores[i]), 3),
        }
        for i in top
    ]
```

Run a quick smoke test to confirm the index is behaving:

```bash
python -c 'from corpus import search; import json; print(json.dumps(search("return policy"), indent=2))'
```

You should see `kb#01` (the generic 30-day line) or `kb#02` (electronics clause) near the top. The scores will be in roughly the 0.3–0.7 band for `text-embedding-3-small` on this corpus.

### Checkpoint

- `search("return policy")` returns 5 rows and includes a `score` field on each.
- The top-1 for the query `"return policy"` is a *generic* returns chunk (likely `kb#01`), not the electronics clause (`kb#02`). This is the corpus shape the lab's borderline prompt depends on — the generic match is the agent's "weak first swing."

## Step 2 — Define the two tool variants

Save `tools.py`:

```python
from corpus import search as _raw_search


SEARCH_KB_WITH_SCORE_TOOL = {
    "name": "search_kb",
    "description": (
        "Search the customer support knowledge base for passages relevant to a "
        "natural-language query. Use this when the user asks a question whose "
        "answer is likely in internal documentation (policies, FAQs). Do NOT "
        "use for chit-chat. If the top hit has a weak score (< 0.6) and the "
        "answer is unclear, consider refining the query and searching again. "
        "Returns up to 5 passages with chunk_id, score, section_title, excerpt."
    ),
    "input_schema": {
        "type": "object",
        "required": ["query"],
        "properties": {
            "query": {
                "type": "string",
                "description": (
                    "A focused, keyword-rich rephrasing of the user's question. "
                    "Prefer domain phrasings over the user's literal words."
                ),
            },
            "k": {"type": "integer", "minimum": 1, "maximum": 10},
        },
    },
}


SEARCH_KB_NO_SCORE_TOOL = {
    "name": "search_kb",
    "description": (
        "Search the customer support knowledge base for passages relevant to a "
        "natural-language query. Use this when the user asks a question whose "
        "answer is likely in internal documentation (policies, FAQs). Do NOT "
        "use for chit-chat. Returns up to 5 passages with chunk_id, "
        "section_title, excerpt."
    ),
    "input_schema": {
        "type": "object",
        "required": ["query"],
        "properties": {
            "query": {"type": "string"},
            "k":     {"type": "integer", "minimum": 1, "maximum": 10},
        },
    },
}


def search_kb_with_score(query: str, k: int = 5) -> list[dict]:
    return _raw_search(query, k)


def search_kb_no_score(query: str, k: int = 5) -> list[dict]:
    rows = _raw_search(query, k)
    return [{kk: v for kk, v in r.items() if kk != "score"} for r in rows]
```

Three differences worth naming before you continue:

- The **with-score** description tells the model the score convention and mentions the 0.6 heuristic. Chapter 4's `search_kb` definition uses this exact shape.
- The **no-score** description omits that clause because there's nothing to say about a field you don't return. Keeping the description otherwise identical isolates the *output* change.
- Both tools are named `search_kb` so the model can't tell the variants apart from their names; only the tool_definition visible to the model changes between runs.

### Checkpoint

- `search_kb_no_score("return policy")[0]` has keys `{chunk_id, section_title, excerpt}` — no `score`.
- `search_kb_with_score("return policy")[0]` has keys `{chunk_id, section_title, excerpt, score}`.
- Diff the two tool descriptions. The only material difference is the weak-score clause and the trailing "score, …" in the field list.

## Step 3 — Write the runner

Save `score_demo.py`:

```python
import json
import os
import time

from anthropic import Anthropic

from tools import (
    SEARCH_KB_NO_SCORE_TOOL,
    SEARCH_KB_WITH_SCORE_TOOL,
    search_kb_no_score,
    search_kb_with_score,
)

MODEL = "claude-sonnet-4-6"

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

SYSTEM = (
    "You are a helpful customer-support assistant. Answer using the knowledge "
    "base when the question is about our policies. Cite chunks you used by "
    "writing [chunk_id: <id>] inline. Before calling a tool, briefly state in "
    "one sentence what you are looking for and which tool you intend to use. "
    "After receiving a tool result, briefly state what you learned before "
    "deciding the next step."
)

PROMPT = "What's your return policy on opened headphones?"


def run_one(tool_def: dict, impl, *, max_steps: int = 6) -> dict:
    tools = [tool_def]
    dispatch = {"search_kb": impl}

    messages = [{"role": "user", "content": PROMPT}]
    steps = []
    for step in range(max_steps):
        t0 = time.monotonic()
        r = client.messages.create(
            model=MODEL,
            max_tokens=512,
            temperature=0,
            system=SYSTEM,
            tools=tools,
            messages=messages,
        )
        wall = round(time.monotonic() - t0, 3)

        text = "".join(b.text for b in r.content if b.type == "text")
        tool_uses = []
        for b in r.content:
            if b.type != "tool_use":
                continue
            tool_uses.append({"id": b.id, "name": b.name, "input": b.input})

        step_rec = {
            "step": step, "stop_reason": r.stop_reason, "wall_s": wall,
            "text": text, "tool_uses": tool_uses, "observations": [],
        }

        if r.stop_reason in ("end_turn", "max_tokens"):
            steps.append(step_rec)
            return {"steps": steps, "final_text": text}

        messages.append({"role": "assistant", "content": r.content})
        tool_results = []
        for b in r.content:
            if b.type != "tool_use":
                continue
            try:
                payload = dispatch[b.name](**b.input)
            except Exception as e:
                payload = {"error": str(e)}
            step_rec["observations"].append({"tool_use_id": b.id, "payload": payload})
            tool_results.append({
                "type": "tool_result", "tool_use_id": b.id,
                "content": json.dumps(payload),
            })
        steps.append(step_rec)
        messages.append({"role": "user", "content": tool_results})

    return {"steps": steps, "final_text": steps[-1]["text"]}


def main() -> None:
    with open("trace.jsonl", "w") as f:
        for variant, tool_def, impl in [
            ("no_score",   SEARCH_KB_NO_SCORE_TOOL,   search_kb_no_score),
            ("with_score", SEARCH_KB_WITH_SCORE_TOOL, search_kb_with_score),
        ]:
            result = run_one(tool_def, impl)
            row = {"variant": variant, "prompt": PROMPT, **result}
            f.write(json.dumps(row) + "\n")

            n_search = sum(
                1 for s in result["steps"] for t in s["tool_uses"] if t["name"] == "search_kb"
            )
            queries = [
                t["input"].get("query")
                for s in result["steps"] for t in s["tool_uses"] if t["name"] == "search_kb"
            ]
            print(f"\n[{variant}] prompt={PROMPT!r}")
            print(f"  search_kb calls: {n_search}")
            for i, q in enumerate(queries, 1):
                print(f"    query {i}: {q!r}")
            print(f"  final: {result['final_text'][:120]!r}")


if __name__ == "__main__":
    main()
```

Run it:

```bash
python score_demo.py
```

### Expected output (shape)

```
[no_score] prompt='What's your return policy on opened headphones?'
  search_kb calls: 1
    query 1: 'return policy opened headphones'
  final: "Our return policy... [chunk_id: kb#01] ..."

[with_score] prompt='What's your return policy on opened headphones?'
  search_kb calls: 2
    query 1: 'return policy opened headphones'
    query 2: 'opened electronics return window restocking fee'
  final: "For opened electronics... [chunk_id: kb#02] ..."
```

Exact wordings and chunk picks vary by model and by the embedding geometry on this corpus. The *qualitative* claim this lab tests is:

- `no_score` makes **1** `search_kb` call and cites whatever came back on the first swing — often `kb#01` (generic 30 days), which does not actually answer "opened headphones."
- `with_score` makes **2** `search_kb` calls. The first query is similar; after seeing the score on the first result, the second query is sharper and pulls `kb#02` (the electronics-specific clause).

<!-- needs-research: actual rate at which a modern Sonnet-tier model issues a second search_kb call under the with-score description on this corpus. Chapter 4 gives the behavior as a qualitative pattern ("the model decides to re-search based on the observation it got") but does not pin a number. Record what you observe, not a canned expectation. -->

### Checkpoint

- `trace.jsonl` has exactly 2 rows.
- In the `no_score` row, exactly one `tool_uses` entry with `name == "search_kb"` appears across all steps.
- In the `with_score` row, two `tool_uses` entries with `name == "search_kb"` appear — or, if only one appears, the top score on the first observation is already **≥ 0.6**. In that case, the model correctly decided the first hit was strong enough. Edit the prompt to use a rarer phrase ("return policy on opened Bluetooth earbuds") and re-run; this lab's teaching only works when the first hit is weak enough to trigger the pivot.

## Step 4 — Read the pivot

Open `trace.jsonl` and in `NOTES.md` fill in this table from the `with_score` row:

| field                                       | value |
|---------------------------------------------|-------|
| step 0 Thought (assistant `text` block)     | …     |
| step 0 query (`search_kb` input)            | …     |
| step 0 top-1 (chunk_id, score)              | …     |
| step 1 Thought (after first observation)    | …     |
| step 1 query (`search_kb` input)            | …     |
| step 1 top-1 (chunk_id, score)              | …     |
| final answer's cited chunk_id(s)            | …     |

Then quote step 1's Thought verbatim. Chapter 4 describes this exact pivot:

> *"Top hit is generic; the user asked specifically about opened electronics. Searching with that phrase."*

Your model's Thought may read differently. The question in your notes is: does it *reference the score* or *reference the content* as its reason to re-search? Both are legitimate. The one the chapter highlights is score-driven ("top score 0.42") because that's the signal that doesn't exist in the `no_score` variant.

### Checkpoint

- The step-1 Thought exists and references either the score or the mismatch between query and top hit. If step 1 is empty or does not justify the re-search, the model may be re-searching for an unrelated reason — note that in `NOTES.md`.
- In the `no_score` row, the final answer cites a chunk whose content doesn't fully answer "opened headphones" (usually `kb#01`). If it cites `kb#02` on the first swing, your corpus's embedding geometry puts the electronics chunk in the top-1 for the user's literal query — in which case the lab's trap didn't trigger. Try a prompt the first-query embedding will not find directly ("what's the policy on returning speakers I already took out of the box?").

## Step 5 — Write the debrief

In `NOTES.md`, answer in 5–8 sentences:

1. Compare the two `final_text` strings. In your run, did `with_score` produce a *more accurate* answer (the 14-day electronics window, 15% restocking fee), and did `no_score` under-answer (just the generic 30-day line) or over-answer (the generic line asserted *as if* it were the electronics policy)? Chapter 4's "Common mistakes" calls out "letting the agent search with the literal user text" and "not surfacing the score" as the two mistakes that compound here.
2. What was the top-1 score on step 0? Chapter 4 does not pin an exact threshold; it uses "weak" and "strong" qualitatively. From this single run, where would you set a numeric threshold for *this corpus* that the model's "re-search on weak hits" instruction could key off? Argue briefly.
3. If you added two more chunks that paraphrased `kb#02` with slightly different wording, would the model's step-0 top score go up — and would that make the with-score variant *stop* re-searching? Chapter 4 says the score is a signal, not a rule — walk through what you'd expect.

## Debrief

Three things to take with you:

- A tool's *output shape* is as much a prompt-engineering surface as its description. Adding one field (`score`) gave the model a new signal to think with, and it immediately used it.
- The score heuristic is not magic. The model's willingness to re-search on a weak hit is a product of (a) the score being in the observation, (b) the description telling it the convention, and (c) the first observation genuinely looking weak. If any of the three is missing, the behavior doesn't appear.
- You can diagnose a one-shot "RAG hallucination" by reading the trace. If the first observation had a score below the heuristic and the model still answered from it, the problem is a vague description or a corpus where the score never clears the threshold — not the model.

Everything downstream in exercise-02 (iterative retrieval, citation validation) assumes this behavior is available. If it isn't, exercise-02's "second swing" probe will keep failing until you add the score and the description clause.

## Stretch

- **Drop the "< 0.6" heuristic from the description.** Keep the score in the output but remove the sentence that names a threshold. Re-run. Does the model still pivot on weak hits? Chapter 4 says both the *field* and the *convention* are required for the behavior — this stretch tests whether the convention alone or the field alone is load-bearing.
- **Return an aggregate score summary.** Change `search_kb_with_score` to return `{"chunks": [...], "top_score": float, "n_above_threshold": int}`. Does the model use the aggregate differently from per-chunk scores? Note any change in wording in the step-1 Thought.
- **A harder corpus.** Add 20 more chunks that paraphrase `kb#01` and `kb#02` in different voices. Re-run both variants. On a denser corpus the first-swing score often clears 0.6, which removes the trigger — the "with score" path converges to "no score" behavior. Note the finding and relate it to chapter 7 of `mod-104` on recall@K.
