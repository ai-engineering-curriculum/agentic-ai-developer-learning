# Exercise 01 — Author a golden set

Paired with [chapter 1 — authoring a golden set](../01-authoring-a-golden-set.md).

**Estimated effort:** 90 minutes.

## Objective

Build a real, defensible golden set of 20–50 rows for one LLM-backed feature you already own (or can bring up in a few minutes from an earlier module). By the end of the exercise you have a JSONL file that: (a) covers the feature with a mix of production-sampled, coverage-oriented, and adversarial rows; (b) has a `tolerance` block per row that is strict on the fields where strict is right and loose only where it must be; and (c) opens with a one-paragraph sampling defence that a colleague could read and immediately spot the coverage gaps in.

Every later exercise in this module builds directly on this file. Skimp on this and everything downstream is measuring the wrong thing.

## Problem statement

Pick one LLM-backed feature. Concrete options if you do not have one at hand:

- The **JSON-extraction feature** from mod-102 exercise 02 (schema-constrained output).
- The **tool-loop agent** from mod-103 exercise 02 (multi-step tool use).
- The **grounded Q&A feature** from mod-104 exercise 02 (retrieval + citations).
- A feature from your own work that you can call in code and that returns structured or free-text output.

Then, in `evals/<feature_name>/golden.jsonl` and a sibling `SAMPLING.md`:

1. Populate the golden set with **20 to 50 rows**. Each row has `id`, `input`, `expected`, `tolerance`, `notes`, and optionally `flake_budget`. Format is JSONL — one row per line, no wrapping array.
2. Sample the rows with a **defended mix**: roughly 60% production-representative (or dogfooding traffic if the feature is pre-launch), 30% behaviour-coverage (one row per documented intent / field / language / refusal class), 10% adversarial / regression cases.
3. Write per-row `tolerance` at the **strictest shape that would pass the correct answer**. Use a mix of `exact`, `case_insensitive`, `regex`, `keyword`, `schema`, `length`, and `similarity` — you do not need to use all of them, but no row should use `similarity` where `exact` would work.
4. Write `SAMPLING.md` (or the top-of-file comment block in `golden.jsonl`) as one paragraph answering: what source and window did the production sample come from, what coverage did you deliberately add, and what are the known coverage gaps.
5. **Do not include an LLM-as-a-judge tolerance yet.** Exercise 03 introduces the judge with calibration; if you find yourself wanting one here, write it as a `# TODO: consider judge` note in the row's `notes` and move on with a rule- or similarity-based tolerance.

## Requirements

### The rows

- **20 ≤ rows ≤ 50.** If you cannot get to 20, the feature is too narrow to eval usefully at this stage; pick a wider one. If you sail past 50, you are almost certainly duplicating existing rows — stop and prune.
- **Every row has a stable `id`** in the shape `<feature-abbrev>-<zero-padded-number>` — `ext-001`, `sum-014`, `qa-032`. IDs never change or get reused after a row is deleted.
- **Every row has a one-line `notes`** field naming where the example came from and why it is in the set. "Real support inbox 2026-07-14, PII-scrubbed. Canonical cancel-order case." is enough; "test row" is not.
- **`input` is the runtime input to the feature**, not the assembled prompt. If your feature is called `summarise(article)`, the input is `{"article": "…"}`, not the full system + user template.
- **`expected` is the shape needed for the row's tolerance.** For rule-based checks, that is often the correct value directly. For `similarity`, it is a `reference_summary` (or the analogous field) that describes the correct answer in prose. For `keyword`, it is the list.

### The sampling mix

- **Production sample (~60%).** If your feature has real traffic (yours or a dogfooding channel), take a *uniform random* window of ~2× the sample size you need, deduplicate near-duplicates, PII-scrub, and pick the sample. If it does not, generate a set of realistic inputs by imagining or logging what a real user would send — but treat this backbone as the source of the coverage gap you name in `SAMPLING.md`, since it is *your* imagination, not your users'.
- **Coverage rows (~30%).** Walk your prompt and your feature's documented behaviours. Every documented intent gets a row. Every JSON field the extractor is supposed to fill has at least one row where that field must be non-null and at least one where it must be null. Every language / locale you claim gets one row. Every explicit refusal or "I don't know" behaviour gets one row.
- **Adversarial rows (~10%).** Empty input, one-word input, near-maximum-length input, ambiguous input, an input in a language you do not claim to support, an input with a prompt-injection attempt if your feature accepts external text. These are the rows that break at 3 a.m. in production.

### Tolerance

- **Strict where strict is right.** Enums / IDs / numeric fields / language tags → `exact`. Categorical / name fields where casing is cosmetic → `case_insensitive`. Shaped fields (dates, IDs with a format) → `regex`.
- **Loose where free text is inevitable.** Summaries, answers, explanations → `similarity` against a `reference_*` field, with a `min_cosine` you have picked deliberately (see below) and a pinned `model` field naming the embedding model.
- **`similarity` threshold is picked from data, not vibes.** Hand-write two acceptable rewordings and two wrong-but-plausible answers for three of your similarity-scored rows. Compute the cosine of each against the reference (a scratch script; you can copy from mod-104 exercise 01). Pick a threshold that sits between the two distributions. Note the exact number and the model in the tolerance block.
- **Every JSON-returning feature has at least one row with `schema` tolerance** that validates the whole response, so a malformed JSON regresses the eval immediately.

### The sampling defence (`SAMPLING.md`)

A single paragraph — ~5–8 sentences — that reads well to a colleague. It must answer, in order:

1. What is the feature and where does its input come from at runtime?
2. What time window and source did the production sample come from? Volume? Deduplication approach?
3. What coverage did you deliberately add on top? Enumerate the coverage rows briefly ("one row per documented intent (5), one row per language (3), one `I don't know.` row (1)").
4. What are the known coverage gaps? (If you cannot name at least one gap, you are not looking hard enough.)

## Starter guidance

- If you are picking the JSON-extraction feature, `ext-001` should be the *canonical, easy* case — the "if this fails, the feature is broken" row. Do not skip it; it is the first thing you look at when the eval falls over.
- If you are picking the grounded Q&A feature from mod-104, remember the `I don't know.` row per corpus source is a coverage row you *must* include — the whole point of the grounded-prompt discipline is that instruction, and an eval that does not test it is not covering the feature.
- If your feature uses retrieval (mod-104), do **not** put the retrieved passages in the `input` field. The retrieval step is part of the feature; the input is the user's raw question. This is what will let exercise 04's model-swap actually test the whole pipeline.
- Store the JSONL under `evals/<feature_name>/golden.jsonl` — this is the layout chapter 3 assumes.
- A trivial validator script — `python -c "import json; [json.loads(l) for l in open('golden.jsonl') if l.strip()]"` — is enough to catch JSONL errors before you start the next exercise. Do not skip it; a broken JSONL is a wasted afternoon later.

## Acceptance criteria

- The golden JSONL file exists at `evals/<feature_name>/golden.jsonl` with 20 to 50 rows, each valid JSON, each with `id`, `input`, `expected`, `tolerance`, `notes`.
- Row IDs are unique and stable in the `<feature>-<NNN>` shape.
- At least three tolerance kinds are used across the file. `similarity` rows name the embedding model explicitly. No row uses `similarity` where a rule-based check would work.
- Every similarity threshold is defended by a short comment or a note pointing at the threshold-calibration scratch data — a colleague can tell how the number was picked.
- The mix roughly matches ~60% production / ~30% coverage / ~10% adversarial. It is fine if your ratio is 55/35/10 — it is not fine if it is 100% coverage.
- `SAMPLING.md` (or a top-of-file block in `golden.jsonl`) reads as a defensible one-paragraph explanation of the sampling. It names at least one coverage gap you are aware of.
- A colleague, given the file cold, can point at any row and answer "why is this row here?" from the row's own `notes` — without asking you.

## Stretch goals

- **A `deleted: true` row.** Deliberately leave one row you decided against including as `{"id": "…", "deleted": true, "reason": "duplicate of ext-014 after PII scrub"}` — this is the shape chapter 1 recommended for keeping deletion history. Confirm your scoring loop (in exercise 03) skips these cleanly.
- **Add a `flake_budget` to one row** where you already know the model's output varies. `{"passes_required": 2, "of_runs": 3}` is the shape. This is where your future self will be glad the schema supports it.
- **Cache the reference embeddings.** For similarity rows, compute the reference embedding at ingest, store it alongside the row (or in a small `references.parquet` / `references.jsonl` keyed by `id` + model name), and note the cache path in the row's tolerance. This is a preview of the "batch and cache" habit chapter 2 highlighted.
- **Cost-estimate the eval.** From your row count, average `input` length, and the model + embedding-model per-token prices (see `resources.md`), estimate the dollar cost of one full run. Compare to your gut estimate before you calculated it. This number matters in exercise 02.
