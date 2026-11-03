# Chapter 1 — Small vs. frontier: a decision, not a default

The single most-consequential line in most LLM applications is the one that names the model. `claude-opus-4-7` versus `claude-haiku-4-5-20251001`. `gpt-*` frontier tier versus a smaller sibling. That one string decides your cost curve, your latency budget, and — for the classes of task where the smaller model is good enough — whether the feature is affordable at all when traffic grows. This chapter is about how to make that choice on evidence instead of vibes, and how to write down the answer in one page so the next person on the team does not re-litigate it.

## Motivation

Two failure patterns show up in almost every young LLM team:

1. **The "always use the frontier model" default.** The first prototype was built on the smartest model on the day, it worked, and the model name was never revisited. Six months and 100× the traffic later, the invoice is 60% of the feature's revenue and half the calls did not need a frontier model to succeed. Nobody had the numbers to argue for downgrading, so nobody did.
2. **The "we tried the small model and it was worse" wave-off.** Someone spent an afternoon, ran ten prompts on both, said the small model "felt worse," and put the ticket to bed. There was no fixed evaluation set, no threshold, no honest measurement of *how much* worse and *on what shape of input*.

Both are the same bug. You do not have a decision until you have three numbers per candidate — an accuracy measure on a fixed test set, a per-call cost, and a per-call latency — and a written argument for why the numbers point at the model you chose. This chapter teaches the shape of that document. Exercise 02 is you writing one.

## The three axes

For any feature you are shipping, there are three axes to compare candidate models on. They trade off against each other; you cannot maximise all three.

### Accuracy (or quality, or task-fit)

The number that matters is **agreement with the ground truth on a task-specific test set you built yourself.** Not benchmark leaderboard scores — those measure model quality *in general* and do not tell you whether the small model is good enough for *your* narrow classification, summarization, extraction, or reranking task.

Two things about accuracy in this module:

- **Define the metric before you run the models.** For a classifier: exact-match on the label. For structured extraction: field-level F1 or exact-match per field. For a summarizer: a rubric an LLM-as-judge scores against, or human review of a random sample. Do not decide the metric after seeing the outputs — that is how you end up choosing the model that flatters the metric you cooked up.
- **Fix the test set.** Same 50–200 inputs run against every candidate. Same prompt (or the prompt each model performs best on — but decide which and be consistent). If the test set is not fixed, small variations in wording will drown out the real signal.

Mod-109 is a deeper dive into evals. For this chapter, a hand-labelled test set of ~50 examples is enough for the "small vs. frontier" question — you are not trying to publish a benchmark, you are trying to decide which model to ship.

### Cost per call

Every provider publishes per-million-input-token and per-million-output-token prices for every model in their lineup. Small models are typically **10–30× cheaper** per token than the same generation's frontier model on both input and output — the exact ratio varies by provider and by month, look it up:

- Anthropic: <https://www.anthropic.com/pricing>
- OpenAI: <https://openai.com/api/pricing/>

<!-- needs-research: exact per-token prices for claude-opus-4-7 vs claude-haiku-4-5-20251001 and for the current frontier vs mini OpenAI tier as of 2026-08 — cite from the provider pricing pages. -->

Cost per call is not the sticker price of a token; it is `input_price * input_tokens + output_price * output_tokens` for a real, representative input. Chapter 2 does the arithmetic in full; for the comparison here, run your fixed test set through both models and read the `usage` block on every response — that is the ground-truth token count and therefore the ground-truth cost.

### Latency

The user does not pay per token; the user pays per second of waiting. Two numbers to measure:

- **Time-to-first-token (TTFT).** How long from send to the first byte of the response. For a streaming UI, this is the "did anything happen?" latency the user actually feels.
- **Time-to-completion.** How long until the last token. For a non-streaming, block-and-wait feature, this is what latency means.

Small models are usually meaningfully faster on both — they have to do less work per token. On a short prompt / short response, the wall-clock difference can be 3–10×. On a very long prompt where most of the time is spent processing input, the gap narrows.

Mod-107 chapter 5 taught you to measure these honestly (p50/p95/p99, not a single sample). Bring that same discipline to the comparison table — a single sample from each model tells you nothing.

## The evaluation loop, in one page

The workflow to compare two candidates:

1. **Freeze a test set.** ~50 inputs from a representative sample of what production will actually send. Not the ten examples that were in your test file when you started.
2. **Freeze the prompt.** Same system prompt, same few-shot examples, same output shape, on both models. If you want to compare "the best each model can do", write down that decision explicitly — the numbers mean something different.
3. **Run the test set through each candidate.** Log the raw output, the `usage.input_tokens`, `usage.output_tokens`, and wall-clock latency for every call.
4. **Score with a fixed metric.** Exact-match, field-level F1, LLM-as-judge with a fixed rubric — whatever fits the task. The point is *fixed*: same scoring rule on every candidate.
5. **Compute three numbers per candidate**: accuracy on the metric, mean cost per call, p50 and p95 latency.
6. **Decide, in writing.** One page: which model you're shipping, what the three numbers were, and what you traded off. Store this in the repo next to the code, not on a slide deck that will vanish.

The one-page write-up is the artefact exercise 02 asks you to produce. The reason it is a real requirement, not a nice-to-have: six months from now, someone on your team will want to swap models, and they should be able to read your page and know what to re-measure to decide whether to override you.

## Two providers, not one

The objective says "at least two providers." That is a deliberate constraint. Two reasons:

- **Providers do not agree on which tasks their small models handle well.** Anthropic's small tier and OpenAI's small tier are optimised for different mixes of task shape. A small model that is bad on your task from one provider does not mean the equivalent from the other provider will be. You will not know until you measure.
- **You need a fallback for chapter 5.** If your entire feature depends on one provider's family, the day that provider has an incident you have no graceful degradation option. Building the A/B habit now means you already have a working prompt and known accuracy numbers for the fallback provider on the day you need them.

For this module, the two providers we assume are Anthropic and OpenAI, but the pattern generalises to any pair (add a third open-weight model served via a router, an Azure-hosted OpenAI deployment, a Bedrock deployment of the same model — the mechanics of the comparison are the same).

## What "defending the choice in one page" looks like

A minimal template. Copy this into your repo, fill it out, keep it next to the feature's code.

```
# Model choice: <feature name>
Date: <YYYY-MM-DD>
Decided by: <name>

## Candidates
- <provider A> / <small model>
- <provider A> / <frontier model>
- <provider B> / <small model>
- <provider B> / <frontier model>

## Test set
- N = <count> examples from <source>
- Metric: <exact match | field F1 | rubric score | ...>
- Held-out from any prompt-tuning: yes / no

## Results

| model | accuracy | mean cost / call | p50 latency | p95 latency |
| --- | --- | --- | --- | --- |
| ... | ... | ... | ... | ... |

## Decision
We are shipping <model> because <one-sentence reason grounded in the numbers>.
The other candidates were ruled out because:
- <candidate>: <accuracy delta was X percentage points at Y× cost> etc.

## When to re-evaluate
- If the p50 accuracy on <specific evaluation> drops below <threshold>.
- If per-call cost exceeds <threshold> in a month.
- If the provider releases a new model in either tier.
- Otherwise: revisit in 6 months.
```

The template is boring on purpose. Boring documents survive; interesting ones get argued into paralysis.

## Common mistakes

- **Comparing on the same prompt without asking whether the prompt is fair.** A prompt tuned for the frontier model may over-fit to its quirks and unfairly disadvantage the smaller candidate. If your write-up says "same prompt on both," note that decision explicitly.
- **Cost from the pricing page, not from `usage`.** Two models with the same per-token price can differ meaningfully in *tokens produced* for the same request — different tokenizers, different verbosity habits, different tool-call framing overhead. Always compute cost from the `usage` block, not from a spreadsheet with the sticker price.
- **A test set of 5–10 examples.** With N that small, one flaky sample flips the decision. Aim for 50 minimum; 100–200 is better if labelling cost allows.
- **Timing on a single sample.** Latency has a wide distribution. Measure the p50 and p95 on 20+ calls per candidate.
- **Comparing a frontier model with reasoning enabled against a small model without.** Reasoning tokens cost real money and real latency; make sure the comparison is like-for-like or is explicitly disclosed as "with reasoning on."
- **Ignoring the failure mode, not just the failure rate.** A model with 90% accuracy that fails silently is worse than a model with 85% accuracy that fails loudly. Look at the mistakes, not just the counts.

## Summary

- Model choice is a decision — three numbers per candidate on a fixed test set, written down.
- Compare on accuracy, cost per call, and latency. All three matter; they trade off.
- Compare at least two providers. Small-tier strengths vary; and you need a fallback for the day one provider goes down.
- Cost from `usage`, not from the pricing page. Latency from a distribution, not a sample.
- The output is a one-page write-up in the repo next to the code. It is what makes the decision resurvivable.

The next chapter takes the cost axis and turns it into arithmetic and enforcement: how to estimate the dollar cost of a feature call before you deploy it, and how to enforce a per-request and per-user token budget at the application boundary so a single bad prompt cannot walk off with your monthly budget.
