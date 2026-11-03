# Chapter 2 — Scoring outputs: rule-based, similarity, and judge

Chapter 1 gave you a file of rows. Each row has an `input`, an `expected`, and a `tolerance`. This chapter turns each row into a boolean — did the model's output pass this row's tolerance or not — using the three families of scorers you will actually reach for on a first eval: **rule-based checks**, **similarity checks**, and one **LLM-as-a-judge** check. The important thing is not that all three exist. The important thing is knowing the cost and calibration trade-off of each so you use the right one per row, and do not accidentally build an eval that costs more than the feature it is checking.

## The mental model: three families, three trade-off shapes

Every scorer you can write for an LLM output lives on the same three-way trade-off between **cost**, **coverage** (how much of "correct" the check can express), and **calibration** (how well the check agrees with a human's judgement of "correct").

| Family | Cost per row | Coverage | Calibration | Non-determinism |
|---|---|---|---|---|
| Rule-based (exact / keyword / regex / schema / length) | ~microseconds, zero API calls | Narrow — the check only asserts what the rule says | Perfect on the fields it covers | None; deterministic |
| Similarity (embedding cosine vs. reference) | ~one embedding API call per side, cached at ingest | Free-text semantic match | Middling — a good similarity threshold matches human "same meaning" about 70–90% of the time on well-scoped tasks | None once vectors are cached |
| LLM-as-a-judge (a second model reads the output + rubric) | One full model call — often the most expensive line item in the eval | Anything you can describe in a rubric | Depends entirely on the rubric and the judge model; needs to be calibrated against human labels before you trust it | Noisy — same input can score differently across runs |

Two consequences follow, and shape the rest of this chapter:

1. **Prefer the cheapest family that can express what "correct" means for the row.** If exact-match works, use exact-match. Do not reach for a judge because judges are impressive; reach for a judge because nothing cheaper covers the shape.
2. **Score the different dimensions of a row with different families.** A structured-output feature might use `schema` (rule) for shape, `exact` (rule) for the ID field, `keyword` (rule) for required nouns, and `similarity` for the free-text `explanation`. One row, three scorer calls, one aggregate pass/fail. Chapter 3 wires these up as pytest cases.

## Rule-based scorers: your default

Rule-based scorers are boring on purpose. They are near-free, deterministic, and — where they apply at all — they are exactly right. Reach for them first for every row, and only fall through to similarity or judge when the row's `expected` cannot be expressed as a rule.

The five you will use constantly:

### Exact match

```python
def exact(actual: str, expected: str) -> bool:
    return actual == expected

def exact_case_insensitive(actual: str, expected: str) -> bool:
    return actual.lower() == expected.lower()
```

Use for enum values, IDs, numeric fields, JSON keys. If your feature is an intent classifier, every row's `intent` field is checked with `exact`. Do not overthink it.

### Keyword presence / absence

```python
def contains_all(actual: str, keywords: list[str]) -> bool:
    lowered = actual.lower()
    return all(k.lower() in lowered for k in keywords)

def contains_none(actual: str, keywords: list[str]) -> bool:
    lowered = actual.lower()
    return not any(k.lower() in lowered for k in keywords)
```

Use for summaries and free-text answers where certain nouns / entities must (or must not) appear. This is the single most under-appreciated scorer — a summary that mentions the right names is *usually* an acceptable summary, and a summary that mentions a forbidden phrase ("rumoured," "allegedly," a competitor's name your policy forbids) is definitely a bad one. It catches ~80% of "the summary is off" bugs at zero cost.

Two operational rules:

- **Lowercase both sides once,** then substring-match. Do not try to be clever with token boundaries; you will fight tokenizer quirks that do not matter.
- **Keep the list short — 3 to 6 items per row.** A 20-item keyword list is not a scorer; it is a hidden model of "correct" and you should just write out a `reference_summary` and use similarity.

### Regex match

```python
import re

def regex_match(actual: str, pattern: str) -> bool:
    return re.search(pattern, actual) is not None
```

Use for fields with a *shape* rather than a value — dates (`^\d{4}-\d{2}-\d{2}$`), phone numbers, order IDs (`^A\d{2}-\d{3}$`), URLs. Also useful as a "must not match" negative check — reject responses that leak PII shapes (`\b\d{3}-\d{2}-\d{4}\b` for US SSN) even when the field is free text.

### Schema conformance

For any feature that returns JSON (which, after mod-102 chapter 5, will be many of your features), the *first* check on every response is that it is valid JSON matching the declared schema. If it is not, no other scorer matters — the response is malformed and the pipeline downstream will fail regardless of semantic quality.

```python
import json
import jsonschema

def schema_conforms(actual_json_str: str, schema: dict) -> bool:
    try:
        actual = json.loads(actual_json_str)
    except json.JSONDecodeError:
        return False
    try:
        jsonschema.validate(actual, schema)
    except jsonschema.ValidationError:
        return False
    return True
```

Two operational rules:

- **Run schema conformance first.** If it fails, short-circuit the row to `fail` and skip the semantic checks. Nothing else is meaningful on malformed JSON.
- **The schema in the eval should be the same schema the runtime code enforces.** Reuse the schema object across the code and the eval; do not maintain two copies.

If you are on OpenAI's Structured Outputs or Anthropic's tool-forcing path, schema conformance is theoretically guaranteed by the provider — but "theoretically" is not "empirically," and the failure mode is quiet (a stray refusal, a partial response, a rare provider bug). Checking it on every eval row is cheap insurance. See the OpenAI Structured Outputs and Anthropic tool-use documentation linked in `resources.md` for the runtime side.

### Length bounds

```python
def within_length(actual: str, max_chars: int | None = None, max_words: int | None = None) -> bool:
    if max_chars is not None and len(actual) > max_chars:
        return False
    if max_words is not None and len(actual.split()) > max_words:
        return False
    return True
```

Use for summaries with a length budget (`max_words: 40`), tweets, alt text, and anywhere your UI truncates. Cheap, catches "the model wrote you a novel again" instantly.

## Similarity scorers: free-text where wording can vary

Rule-based scorers cover fields where "correct" is a shape or a set of substrings. They cannot cover fields where "correct" is a *meaning* that can be phrased many ways — the summary field of a summariser, the answer field of a Q&A feature, the explanation field of a classifier.

For those, the workhorse is **embedding cosine similarity** against a reference answer you stored in the row.

```python
from openai import OpenAI
import numpy as np

client = OpenAI()

def cosine(a: np.ndarray, b: np.ndarray) -> float:
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

def similarity_score(actual: str, reference: str, model: str = "text-embedding-3-small") -> float:
    resp = client.embeddings.create(model=model, input=[actual, reference])
    a, b = (np.array(item.embedding) for item in resp.data)
    return cosine(a, b)

def passes_similarity(actual: str, reference: str, threshold: float, model: str) -> bool:
    return similarity_score(actual, reference, model) >= threshold
```

You have already met embeddings and cosine similarity in mod-104 chapter 2. The mechanics are the same; the *use* is different. In retrieval, embeddings decide which passages to fetch. In eval, they decide whether the model's answer means what the reference answer means.

### Choosing the threshold

The single question you have to answer for every similarity-scored row is: what cosine value counts as "close enough"? Guessing produces a scorer that either lets everything through (threshold too low) or fails on rewordings (threshold too high). The honest way to pick it:

1. Take 10–20 rows where you already know the correct answer.
2. For each, generate 3 acceptable rewordings and 3 wrong-but-plausible answers (do this by hand — an afternoon of manual work you only do once).
3. Compute the cosine of each rewording and each wrong-answer against the reference.
4. Pick the threshold that sits between the two distributions. Usually somewhere in `0.70–0.85` for `text-embedding-3-small`; higher for `text-embedding-3-large`. Write the number and the model name into the row's tolerance so it is reproducible.

Never pick a threshold on one row. A threshold that works for terse answers ("Yes, refunds are eligible") will not work for long ones ("Refunds are eligible for orders placed within the last 30 days, subject to the terms of the return policy..."). The threshold is a per-scorer, per-model calibration — not a per-row invention.

### Two failure modes to know

- **Similarity is blind to negation and dates.** From mod-104 chapter 2: "The bug is fixed" and "The bug is not fixed" have near-identical embeddings. A similarity check will pass a factually wrong answer whose wording matches the reference. If your row's correctness depends on a negation, a date, or an exact identifier, put those in a *rule-based* sub-check on the same row — do not rely on similarity to catch them.
- **Threshold drift.** If you change the embedding model — from `text-embedding-3-small` to `text-embedding-3-large`, or from OpenAI to Voyage — every threshold in your eval is now wrong, and the eval will silently loosen or tighten. Pin the embedding model in the tolerance field, and treat swapping it as an eval-versioning event, not a maintenance edit.

### Cheap operational habits

- **Cache the reference embedding.** The `reference` in each row does not change; its embedding does not need to be recomputed on every eval run. Compute once at ingest time; store the vector next to the row (or in a small SQLite / pgvector table keyed by row `id` + model name).
- **Batch the actuals.** If you have 50 rows to score by similarity, one embeddings request of 50 strings is dramatically faster than 50 requests of one, and roughly the same cost.

## LLM-as-a-judge: use sparingly, know what it costs

The third family is: prompt a *second* model to read the actual output, read a rubric, and return a pass/fail (or a score with reasoning). It is the most flexible tool — you can express any "correct" criterion you can write in English — and it is by far the easiest to get wrong.

A **minimal** judge for a summariser:

```python
JUDGE_SYSTEM = """You are an evaluator. You will be given an ARTICLE, a SUMMARY,
and RULES the summary must obey. Decide whether the summary passes the rules.

Return ONLY a JSON object of the form:
{"pass": true|false, "reason": "<one sentence>"}

Be strict. If in doubt, return pass: false with a specific reason.
"""

def judge(article: str, summary: str, rules: list[str]) -> dict:
    resp = client.chat.completions.create(
        model="gpt-4o-mini",   # judges do not need frontier; see below
        messages=[
            {"role": "system", "content": JUDGE_SYSTEM},
            {"role": "user", "content": f"ARTICLE:\n{article}\n\nSUMMARY:\n{summary}\n\nRULES:\n- " + "\n- ".join(rules)},
        ],
        response_format={"type": "json_object"},
        temperature=0,
    )
    return json.loads(resp.choices[0].message.content)
```

Judges do not come out of the box calibrated. Before you trust a judge in CI, you have to prove that its pass/fail agrees with a *human* pass/fail on the rubric you wrote — otherwise you have replaced one uncalibrated judgement (yours) with another (the judge model's).

**The minimum calibration procedure for this module** — full methodology lives in `ai-eval-engineer-learning` (chapter 5):

1. Take 20 rows where you have already scored the model's output as pass or fail *by hand*, on the same rubric.
2. Run the judge on the same 20 rows.
3. Compute the agreement rate. If the judge and you disagree on more than 2–3 rows out of 20 (~85% agreement floor), the judge is not usable as written. Rewrite the rubric — tighter definitions, examples of pass and fail — and try again.
4. Save the 20-row calibration set with the judge prompt. Every time you touch the judge prompt or swap the judge model, rerun the calibration. A judge that is not periodically recalibrated is a scorer whose meaning drifts.

### What judges are actually good for

- Rubrics that combine multiple properties in a way that rules cannot cleanly express — "the summary is factually consistent with the article AND does not add invented details AND uses a neutral tone." Three properties, one call.
- Free-text quality on tasks where similarity is too coarse — "does the reply politely refuse without lecturing the user" — where "polite" and "not lecturing" are things you can describe in prose but not in code.
- Ranking-style comparisons ("is answer A better than answer B") when you are choosing between two prompts or two models and a scalar quality signal matters more than a pass/fail.

### What judges are bad for

- **Anything a rule can express.** A judge that decides whether a field equals `"cancel_order"` is a $0.001 replacement for an `==` operator. Multiplied by 50 rows and every CI run, that is real money for no signal.
- **Correctness of factual claims about the world.** Judges have the same blind spots as the model under test. If your feature answers "what is the current dividend yield of TSLA," a judge will hallucinate a plausible number and mark a wrong answer correct.
- **Anything you have not calibrated against humans.** An uncalibrated judge is a coin flip with a confident tone.

### The cost of a judge, spelled out

For a 50-row eval where each row goes through one judge call:

- **Cost.** One frontier-model call per row, per run. Even at cheap-tier prices (~$0.15 per M input tokens), a 2k-token judge prompt × 50 rows × several runs per day per developer × several developers is the difference between an eval that costs a few dollars a month and one that costs hundreds.
- **Latency.** A judge round-trip adds seconds per row. A 50-row eval with one judge per row can take 3–5 minutes; developers will start skipping it locally.
- **Noise.** Even at `temperature=0`, judges are less deterministic than rule-based scorers. The same run on the same output will occasionally flip. This is the *"non-determinism"* column in the table above and it directly limits how tight you can set the CI pass threshold (chapter 4).

For a first eval: reach for a judge on **at most one dimension** of at most a **handful of rows** where nothing cheaper covers what "correct" means. If you find yourself writing a judge for every row, take a step back — you are building the eval methodology that `ai-eval-engineer-learning` covers, and doing it without the calibration methodology.

## Composing scorers per row

Every row in a real golden set will use several scorers, one per field or one per property. The scoring layer's job is:

1. Look at the row's `tolerance` block.
2. For each `(field, tolerance_kind)` pair, dispatch to the right scorer.
3. Aggregate — every sub-check must pass for the row to pass. Report the *specific sub-check that failed* so a regression report can point at "row `sum-003` failed on `must_mention`", not just "row `sum-003` failed."

A compact dispatcher:

```python
def score_row(actual: dict, row: dict) -> dict:
    checks = []
    for field, tol in row["tolerance"].items():
        expected = row["expected"].get(field)
        actual_val = actual.get(field) if isinstance(actual, dict) else actual

        if tol == "exact":
            ok = actual_val == expected
        elif tol == "case_insensitive":
            ok = str(actual_val).lower() == str(expected).lower()
        elif isinstance(tol, dict) and tol["kind"] == "keyword":
            required = expected if tol.get("mode", "all") == "all" else expected
            ok = (contains_all if tol.get("mode", "all") == "all" else contains_none)(str(actual_val), required)
        elif isinstance(tol, dict) and tol["kind"] == "regex":
            ok = regex_match(str(actual_val), tol["pattern"])
        elif isinstance(tol, dict) and tol["kind"] == "schema":
            ok = schema_conforms(actual_val, tol["schema"])
        elif isinstance(tol, dict) and tol["kind"] == "length":
            ok = within_length(str(actual_val), max_words=tol.get("max_words"), max_chars=tol.get("max_chars"))
        elif isinstance(tol, dict) and tol["kind"] == "similarity":
            ok = passes_similarity(str(actual_val), expected, tol["min_cosine"], tol["model"])
        elif isinstance(tol, dict) and tol["kind"] == "judge":
            ok = judge(row["input"], actual_val, tol["rules"])["pass"]
        else:
            raise ValueError(f"Unknown tolerance: {tol!r}")

        checks.append({"field": field, "kind": tol if isinstance(tol, str) else tol["kind"], "pass": ok})

    return {
        "id": row["id"],
        "pass": all(c["pass"] for c in checks),
        "checks": checks,
    }
```

This is deliberately verbose — one dispatch per tolerance kind, one aggregated boolean, and the full per-check trail so the regression report of chapter 5 can render it. Do not try to make it clever. The failure mode of a scorer library is silent miscategorisation of failures; every added abstraction increases the surface for that.

## A rough decision tree

For each field in each row, ask in order:

1. **Is the correct value a fixed value, a shape, or a set of substrings?** Rule-based (exact / regex / keyword / schema / length). Stop.
2. **Is the correct value free text where meaning matters but wording does not?** Similarity, with a calibrated threshold and cached reference embedding.
3. **Is the correct value a multi-property quality judgement that neither of the above expresses, on a row where the extra cost and noise are worth it?** LLM-as-a-judge, calibrated against 20 hand-scored rows.
4. **Otherwise:** the field is not scorable *yet*. Either rewrite the row so it is (usually possible with a bit of thought) or move the field out of the eval and handle it with human review or online monitoring — both topics of `ai-eval-engineer-learning`.

## Summary

- Rule-based scorers (exact, case-insensitive, keyword presence/absence, regex, schema conformance, length) are near-free, deterministic, and exactly right where they apply. They are your default.
- Similarity scorers (embedding cosine vs. a reference) cover free-text meaning-preserving fields. Calibrate the threshold on 10–20 hand-scored examples; pin the embedding model; do not use them to catch negation, dates, or IDs.
- LLM-as-a-judge scorers cover anything you can describe in English. They are the most expensive and noisiest option. Do not use one without calibrating agreement against a hand-scored set of at least 20 rows; do not use one where a rule works.
- Compose scorers per row. Every row's `tolerance` block dispatches to one scorer per field, all must pass, and the per-check trail is preserved so failures point at the specific sub-check.
- If you find yourself reaching for a judge on every row, you have moved out of mod-109 scope into eval methodology — [chapter 5](05-boundary-to-eval-engineer-track.md) tells you where to graduate.

The next chapter takes these scorers and wires them into pytest so each row is a real test, each failure a real red X, and running the eval a `pytest -m eval` away.
