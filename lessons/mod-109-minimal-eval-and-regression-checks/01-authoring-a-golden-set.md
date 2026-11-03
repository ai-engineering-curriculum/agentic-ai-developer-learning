# Chapter 1 — Authoring a golden set

Every module before this one taught you to *build* an LLM feature — prompts, tools, streaming, model choice, retrieval. This module teaches you the one habit that keeps the feature working after you ship it: a small labelled set of examples the feature must keep getting right, that you can rerun on every prompt or model change and see the answer *before* a user does. That set is called a **golden set** (also "eval set," "regression set," or just "the eval"). This chapter is about building one honestly for the *first* feature you own.

## Motivation

You already do this in every non-LLM codebase you have worked in. You call it a test suite. The reason a normal test suite is not enough for LLM features is that the interesting behaviour of the feature — "does it answer the customer's question in the tone we asked for, using only the passages we retrieved, in the JSON shape we need" — is not a pure function of the inputs. It is a *distribution* over outputs, and the acceptance criterion is not "byte-equal to the expected string" but "close enough on the dimensions I care about."

Without a golden set, every change you make is done by vibes:

- You rewrite a system prompt to fix one bad answer a colleague showed you in Slack. You have no idea whether the rewrite silently broke five other cases.
- You swap `gpt-4o` for `gpt-4o-mini` to cut costs. Latency drops; one week later customer-facing quality drops with it, and you cannot tell whether it was the model, a prompt drift, or the retrieval layer.
- A provider retires the model you were pinned to. You cannot say "the replacement is fine" without a number.

A golden set is the smallest artefact that turns "the feature seemed better" into "the feature scored *N/50* before the change and *M/50* after." That is what the rest of this module compounds on. Chapter 2 turns your set into a pytest suite. Chapter 3 shows you how to score each row. Chapter 4 turns the pass/fail into a gated CI check. Chapter 5 tracks the number over time. None of them are usable without the artefact this chapter teaches you to build.

## What a golden-set example is

Every row in a golden set is a triple:

- **`input`** — the exact input your feature receives at runtime. If your feature takes `{ticket_id, question}` and looks up the ticket, the input is `{ticket_id, question}`. If it takes a raw user message, the input is the raw user message. It is *not* the full assembled prompt — the prompt is what the code under test builds from the input. Storing the assembled prompt would make your golden set stop being valid the moment you edited the prompt template.
- **`expected`** — what a correct output looks like. The shape depends on what the feature returns and how you plan to score it (chapter 3). For a JSON-extractor it might be the exact expected JSON. For a summariser it might be a set of required keywords, a maximum length, and a forbidden-phrase list. For a Q&A feature it might be a short expected answer plus the source passage that should have been cited.
- **`tolerance`** — how much slack the correct output has. LLM outputs are stochastic even at `temperature=0` — you cannot demand byte-equality on free text. Tolerance is where you write down, in code-readable form, what "close enough" means for this row. Concretely: which fields must be exact, which must match a regex, which must contain a keyword list, which can vary in wording as long as an embedding is within `X` cosine of a reference answer.

Store these as JSONL, one row per line, in a file that lives next to the feature — `evals/summariser/golden.jsonl`, `evals/support_q_and_a/golden.jsonl`. JSONL is boring on purpose: it diffs cleanly in code review, streams into pytest without a schema library, and does not tempt anyone to load it into a spreadsheet.

A minimal row for a JSON-extraction feature:

```json
{
  "id": "ext-001",
  "input": {"email_body": "Hi, please cancel order A47-991. My name is Alex Chen. Thanks."},
  "expected": {"intent": "cancel_order", "order_id": "A47-991", "customer_name": "Alex Chen"},
  "tolerance": {
    "intent": "exact",
    "order_id": "exact",
    "customer_name": {"kind": "case_insensitive"}
  },
  "notes": "Canonical cancel-order case. Sourced from real support inbox 2026-07-14, PII-scrubbed."
}
```

Two fields will save you time later:

- **`id`** — a stable identifier per row, not the array index. When a row is renamed or reordered, your regression report should not read as "example 17 changed" but as "`ext-001` changed."
- **`notes`** — one line saying *where the example came from* and *why it is in the set*. Six months from now you will be asked "why is this row here?" — writing the answer once when you added it beats reconstructing it from a git blame.

## Size: 20 to 50 examples for a first feature

The learning objective is explicit — 20 to 50 examples — and both bounds matter.

- **Fewer than ~20** and one bad row moves the score by more than 5%. A single flaky example dominates the signal; you spend your regression debugging on noise. You also cannot cover more than one or two behaviour dimensions before you run out of rows.
- **More than ~50** and, for a first eval, you are almost certainly guessing rather than sampling. The rows past ~50 tend to be paraphrases of the first ten and do not add new coverage. They also make the eval slow enough that people start skipping it in local runs, which is exactly the failure mode you are here to prevent.

The range is not a compromise; it is the shape that makes a first-pass eval useful:

- Big enough that a real regression moves the number visibly and a lucky one-row flip does not.
- Small enough that a person can read every row in one sitting when the score changes.
- Cheap enough that one full run costs cents, not dollars, so nobody has an excuse to skip it.

Add a real ~50th example when the cost of adding it is lower than the cost of missing the case in production. Do *not* add rows because the number looks impressive. `27/27 passing` is a stronger artefact than `41/50 passing` where the 9 failures are examples nobody looked at closely.

Grow the set later, guided by production incidents (see the "add a row every time a bug ships" habit at the end of this chapter). The first version's job is to exist and gate the next PR.

## Sampling: defend the choice, in one paragraph

The single most common way a first golden set fails is that it was built from whatever examples the author remembered, which turn out to be the easy ones. The feature scores 100% on the eval and 60% in production, and the author cannot say why. The fix is to sample deliberately and *write down where the examples came from*, so a colleague reading the eval can spot the coverage gap without your help.

There are three sampling strategies you compose. Every real golden set uses some mix of all three; the ratio is what you defend.

### 1. Production sampling

The most defensible strategy: pull real requests from your feature's own logs.

- Take a **uniform random sample** — not the failures, not the interesting ones, a random slice. This becomes the "representative what our users actually ask" backbone of the set.
- Pull from a window that covers the variability you care about. One week is usually short enough to be recent and long enough to smear across weekdays / weekends / time zones.
- Deduplicate near-duplicates *before* scoring. Two paraphrases of the same query at rank 1 and rank 2 pretend to be two examples and are really one.
- PII-scrub aggressively at ingest time. A golden set that contains real customer names or emails is a data-handling incident waiting to happen — even if the file is committed to a private repo.

If your feature is pre-launch and has no production traffic, use a **dogfooding sample** — a week of requests from the team using the feature internally — with the same rules. Do not skip to "we will imagine some inputs"; you will imagine the easy ones.

### 2. Behaviour coverage

Once you have a representative backbone, pass through it and check that every *behaviour you promised* has at least one row that exercises it. If the feature is documented as handling five intent types, you need at least one row per intent even if the traffic is 90% two intents. This is where you cover:

- **Every documented format / field / intent / language.** One row per JSON field the extractor is supposed to fill. One row per intent the classifier is supposed to distinguish. One row per language the summariser claims to handle.
- **The "unhappy" paths you have written prompt instructions for.** If the prompt says "respond `I don't know.` when the answer is not in the sources," you need rows where the correct answer *is* `I don't know.`, otherwise you are not testing the instruction.
- **The refusal boundary, if any.** If the feature refuses certain classes of request, one row per refusal class.

The rule is: if the prompt contains a sentence that says the model "should" do something, there is a row on the eval that would fail if the model stopped doing it.

### 3. Adversarial / regression rows

The last, smaller slice is examples that *bit you*:

- Every production bug that was fixed with a prompt change becomes a row. If you fixed a bug and did not add a row, the fix is one prompt edit away from silently regressing.
- Every "the model kept getting this wrong before we hardened the prompt" case, saved from your first week of building the feature.
- A handful of intentionally-hard cases — ambiguous inputs, minority-class inputs, edge lengths (empty string, one word, near max input length) — that stress the parts of the prompt least tested by production traffic.

Aim for roughly a **60/30/10** mix — 60% production-representative, 30% coverage, 10% adversarial. Adjust to your feature. What is not negotiable is that the mix is *written down*.

### Writing down the defence

At the top of the golden-set file (or in a `SAMPLING.md` next to it), add one paragraph that answers three questions:

1. **What time window / source did the production sample come from?** ("Uniform random sample of 30 requests to `POST /support/answer` between 2026-07-01 and 2026-07-08, PII-scrubbed.")
2. **What behaviour coverage did you deliberately add on top?** ("One row per documented intent (5 rows). One `I don't know.` row per corpus source. One row per supported language (en, es, pt).")
3. **What are the known coverage gaps?** ("No multi-turn conversations. No image / attachment inputs. Traffic before 2026-07-01 not represented — the June UX change added filters we do not yet sample.")

If you cannot write that paragraph, the golden set is not ready — you have a pile of examples, not a defensible sample.

## Tolerance: how much wiggle room each row gets

The single biggest first-time mistake is demanding byte-equality on free-text outputs. The eval then fails 90% of the time on cosmetic differences (`"OK"` vs `"Okay"`, trailing period, "Alex Chen" vs "alex chen"), everyone learns to ignore the red X, and the eval is dead.

Tolerance is what makes the eval strict on what matters and forgiving on what does not. Store it *per row* so a strict row (an ID must be exact) and a loose row (a summary can vary in wording) can live in the same set. Concretely, tolerance is a set of hints to the scoring layer of chapter 3:

- **`exact`** — after both sides are stringified, they must be byte-equal. Use for IDs, enum values, numeric fields, JSON keys.
- **`case_insensitive`** — as above but ignoring case. Use for names, categories.
- **`regex`** — the output must match a stored regex. Use for structured-ish fields where you care about a shape but not the content (`^\+?[0-9\- ]{7,}$` for a phone number).
- **`keyword`** — the output must contain (or must not contain) a list of substrings. Use for summaries that must mention certain nouns.
- **`schema`** — the output must validate against a stored JSON Schema (chapter 3). Use when the acceptance criterion is really "is it well-formed JSON with the right shape," and the semantic check lives elsewhere.
- **`similarity`** — the output must be within `X` cosine distance of a reference answer under a specified embedding model. Use for free-text answers where wording can vary but meaning must not (chapter 3 covers the cost and calibration trade-off).
- **`judge`** — an LLM-as-a-judge with a specific rubric decides. Use *sparingly* and only where the other tolerances genuinely cannot express what "correct" means. Chapter 3 covers when to reach for this and what it costs you in noise, latency, and dollars.

Two operational rules for tolerance:

- **Default to the strictest tolerance that would pass the correct answer.** Loosening later on a real failure is cheap; tightening later is a scavenger hunt through past passes.
- **Do not use the same tolerance shape on every row.** If every row is `similarity > 0.85`, you have a similarity eval, not a golden set — and you have lost the ability to be strict on the fields where strict is the right answer.

## What good and bad rows look like

**Good, well-scoped row (a JSON-extraction feature):**

```json
{
  "id": "ext-014",
  "input": {"email_body": "Nao quero mais este pedido, obrigado."},
  "expected": {"intent": "cancel_order", "order_id": null, "customer_name": null, "language": "pt"},
  "tolerance": {
    "intent": "exact",
    "order_id": "exact",
    "customer_name": "exact",
    "language": "exact"
  },
  "notes": "Portuguese cancellation with no order ID and no signature. Coverage row for pt-BR + null fields."
}
```

Everything strict, because the tolerated wiggle room for a classifier is essentially zero. If the model produces `"cancel"` instead of `"cancel_order"`, the eval must catch that — this row exists exactly for that.

**Bad, brittle row (a summariser feature) — do not copy:**

```json
{
  "id": "sum-003",
  "input": {"article": "…900 words about the merger…"},
  "expected": "The merger between Acme and Globex closed on Tuesday for $12B in cash.",
  "tolerance": {"kind": "exact"}
}
```

`exact` on a free-text summary means every prompt tweak fails the row for wording reasons. The row is now noise and someone will delete it.

**Better version of the same row:**

```json
{
  "id": "sum-003",
  "input": {"article": "…900 words about the merger…"},
  "expected": {
    "must_mention": ["Acme", "Globex", "$12B", "Tuesday"],
    "must_not_mention": ["rumoured", "unconfirmed"],
    "max_words": 40,
    "reference_summary": "Acme and Globex closed their merger on Tuesday for $12 billion in cash."
  },
  "tolerance": {
    "must_mention": {"kind": "keyword", "mode": "all"},
    "must_not_mention": {"kind": "keyword", "mode": "none"},
    "max_words": {"kind": "length"},
    "reference_summary": {"kind": "similarity", "min_cosine": 0.75, "model": "text-embedding-3-small"}
  }
}
```

Now the row asserts what the summary must contain, what it must not contain, that it is short enough, and that it is semantically close to a reference. Any of those four failing is a real signal, not a wording change.

## The "add a row every time a bug ships" habit

The golden set you finish this chapter with is version 1. It grows over time under one rule: **every user-visible bug in the feature that gets fixed adds exactly one row to the eval before the fix ships**. The row must be *the specific input that caused the bug*, with an `expected` shape that would have failed before the fix.

This is the LLM-eval analogue of the "regression test for every bug" habit from regular software. It has the same load-bearing property: the eval only stays representative of what actually breaks in production if you feed the real breaks back into it. Skip this and, within six months, the eval scores 100% and the feature ships bugs anyway, because the eval is a snapshot of your day-1 imagination, not of what your users do.

Two secondary habits reinforce it:

- **When a row starts flaking, do not delete it — investigate.** A row that fails once every ten runs is telling you the feature is genuinely non-deterministic on that input. That is a real product signal. If it turns out to be your scoring being too tight, loosen the tolerance and comment *why*. If it turns out to be the model, that is an incident on the feature — treat it as such.
- **When a row is deleted, log why.** A `DELETED_ROWS.md` (or a `deleted: true` field) with one line per deletion — "removed 2026-08-01: intent renamed in v3 of the schema; row `ext-005` no longer valid" — protects you from someone quietly deleting the rows that would have caught a bug next quarter.

## What to build in this chapter's exercise

Exercise 01 is where you build the artefact this chapter describes for one real LLM feature — 20 to 50 rows, mixed sampling, per-row tolerance, and a one-paragraph sampling defence at the top of the file. Everything else in this module rests on that file existing; do not skip ahead to pytest and CI without it.

## Summary

- A golden set is a small (~20–50 example) file of `{input, expected, tolerance, id, notes}` rows that gates future prompt / model / retrieval changes against silent regressions.
- Sample deliberately and defend the mix in one paragraph: a production-representative backbone (~60%), behaviour-coverage rows (~30%), adversarial / regression rows (~10%).
- Store tolerance *per row*, at the strictest shape that would pass the correct answer. Mix exact, regex, keyword, schema, similarity, and — sparingly — LLM-as-judge; do not force one shape onto every row.
- Give every row a stable `id` and a one-line `notes` field. Store as JSONL next to the feature. PII-scrub at ingest.
- Grow the set from production incidents: every user-visible bug adds a row before the fix ships.

The next chapter takes the file you built here and turns each row into a scored pass / fail — the three scoring families (rule-based, similarity, judge), what each costs, and when to reach for which.
