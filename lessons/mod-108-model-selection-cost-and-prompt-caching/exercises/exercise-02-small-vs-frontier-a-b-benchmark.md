# Exercise 02 — Small vs. frontier A/B benchmark

Paired with [chapter 1 — small vs. frontier: a decision, not a default](../01-small-vs-frontier-model-choice.md).

**Estimated effort:** 2–2.5 hours.

## Objective

Run a real, honest A/B on a small model and a frontier model — across **two providers** — on a fixed test set for a specific task. Produce three numbers per candidate (accuracy, cost per call, latency). Write a one-page decision doc that names the model you would ship and defends the choice with the numbers.

The point is not to pick the "right" model in some absolute sense; it is to practice the workflow — freeze the task, freeze the prompt, freeze the metric, measure, decide, write it down.

## Problem statement

Pick a task from the following list (or another well-scoped classification / extraction task if you already have one):

- **Sentiment classification.** Input: a product review. Output: `{"sentiment": "positive"|"neutral"|"negative"}`.
- **Support-ticket categorisation.** Input: a ticket body. Output: `{"category": one_of_5_labels, "priority": "low"|"medium"|"high"}`.
- **PII redaction.** Input: a paragraph of text. Output: same text with names, emails, and phone numbers replaced by `[REDACTED_NAME]`, `[REDACTED_EMAIL]`, `[REDACTED_PHONE]`.
- **Simple structured extraction.** Input: an email. Output: `{"sender_name": ..., "requested_action": ..., "deadline_iso": ...|null}`.

The choice of task does not matter — what matters is that it has a **clear ground-truth answer** for each input.

## Requirements — the test set

- **50 examples minimum.** Hand-labelled. Sampled to include easy, medium, and hard cases (roughly a third each).
- Store in a JSONL file: one input+expected-output per line.
- Do **not** iterate on your prompt against this test set (that's overfitting). Iterate on a separate dev set of ~10 examples if you need to.

## Requirements — the candidates

Compare four models — two per provider:

- **Anthropic** small tier (e.g., current Haiku) and frontier tier (e.g., current Opus).
- **OpenAI** small/mini tier and frontier tier.

Pick the specific model IDs from each provider's current model list. Look them up:

- Anthropic models: <https://docs.anthropic.com/en/docs/about-claude/models>
- OpenAI models: <https://platform.openai.com/docs/models>

## Requirements — the run

Write a `run_ab.py` that:

1. Loads the test set.
2. For each candidate model, runs every test-set input through the same prompt. Captures:
   - The raw model output.
   - `usage.input_tokens`, `usage.output_tokens` (and `cached_input_tokens` if the response provides it).
   - Wall-clock latency (measured client-side, from send to full response received).
3. Scores each output against the expected output using a **fixed metric**. Pick one appropriate to the task (exact-match on the label, field-level F1, exact string match on redaction, etc.).
4. Aggregates and prints:

   ```
   model                    accuracy   mean cost/call   p50 latency   p95 latency
   anthropic:haiku-...      88%        $0.00012         420ms         1.1s
   anthropic:opus-...       96%        $0.00280         890ms         2.4s
   openai:mini-...          ...        ...              ...           ...
   openai:gpt-4.1-...       ...        ...              ...           ...
   ```

Latency should be computed from a real distribution — the p50 and p95 across the 50 calls per model, not a single sample.

Cost should be computed from the `usage` block per call, then averaged. Include the pricing page URL and the date fetched as a comment at the top of the file.

## Requirements — the decision doc

Write `DECISION.md` (or similar) in the same directory as the run. Use the template from chapter 1:

- Candidates.
- Test set (size, source, held-out from prompt-tuning: yes / no).
- Metric used.
- Results table (the same table you printed).
- **Decision.** One sentence: "We are shipping X because Y" — where Y is grounded in the numbers.
- **When to re-evaluate.** Concrete triggers.

The decision must be defensible. "The frontier model was smarter" is not defensible without a number attached. "Frontier was 8 percentage points more accurate but 20× more expensive; the small model at 88% clears our threshold of 85% for this feature, so we ship the small model" is defensible.

## Requirements — one debrief

At the bottom of `DECISION.md`, add a short section:

- **What surprised you.** One paragraph. Maybe the small model was closer than you expected; maybe cross-provider quality differed more than cross-tier within a provider; maybe frontier's latency variance was worse than the small model's. Whatever it was, note it — this is what you will remember six months from now when this decision comes up again.

## Starter guidance

- Chapter 1 above; the template for the decision doc is copied there.
- Mod-107 chapter 5 for how to measure latency honestly (client-side wall clock, distribution not sample).
- For scoring: if the task is classification, exact-match is fine. For extraction, use field-level exact-match or F1 per field. For anything more subjective (a summarizer, an open-ended answer), an LLM-as-judge with a fixed rubric works — but validate that the judge agrees with you on 10 hand-scored examples first, or you're benchmarking the judge.
- Keep the prompt short and identical across models. If you want to compare "the best each model can do", write that decision down explicitly, and use a per-model prompt — but be aware you are now benchmarking your prompt-tuning ability as much as the models.

## Acceptance criteria

- Test set has at least 50 labelled examples in a JSONL file.
- All four candidate models ran through the same test set with the same prompt (or with per-model prompts, explicitly noted).
- The results table shows accuracy, mean cost per call, and p50 + p95 latency for each candidate.
- `DECISION.md` exists, names the model you'd ship, and cites the numbers in defence of the choice.
- `DECISION.md` includes a "when to re-evaluate" section with at least two concrete triggers.
- The "what surprised you" paragraph is honest and specific — one concrete observation, not a truism.

## Stretch goals

- Add a **fifth candidate**: a third provider (e.g., an open-weights model served via a router like OpenRouter, or an Azure-hosted OpenAI deployment). Note the extra provider dimension in the decision.
- Add a **per-example diagnostic**: for the top 10 examples where the frontier model was right and the small model was wrong, examine the inputs. Is there a pattern? Length? Ambiguity? Note it in the decision doc — this is the seed for chapter 4's cascade routing rules.
- Add a **per-example cost breakdown**: a scatter plot of accuracy vs. cost per call. The Pareto frontier is your decision surface.
- **Confidence check.** For your chosen ship model, add a small prompt-level tweak that asks the model for a confidence score, and measure whether the confidence correlates with actual correctness on the held-out test set. This is the seed for the cascade routing exercise.
