# Chapter 4 — Routing by task shape: cascade, fast-first, batch

Chapters 1–3 assumed a single model per feature. This chapter drops that assumption. Real production LLM systems route different requests to different models — a cheap small model for the easy 80% of inputs, a frontier model for the hard 20%, a batch API for the workload that runs overnight and does not care about latency. The wins are not marginal: a well-cascaded feature can be 3–5× cheaper than always running the frontier model, at the same or better final accuracy.

This chapter is about the three shapes routing takes — **cascade routing**, **fast-first-then-escalate**, and **batch APIs** — and how to pick between them for a given feature.

## Motivation

There are three cost curves you can pick from for an LLM feature:

1. **"Always frontier."** Cost per call is the frontier price. Simple; expensive.
2. **"Always small."** Cost per call is the small-tier price. Cheap; broken on the hard tail.
3. **"Route by shape."** Cost per call is the small-tier price *most* of the time, with a fallback to frontier for cases the small model handles poorly. Cheap on average; correct on the tail.

If you're always running the frontier model on a feature whose modal input is "categorise this ticket" and whose worst input is "categorise this ticket after considering these 12 corner-case policies", you are overpaying by roughly the frontier-to-small price ratio on the 80% of easy inputs. That ratio is often 10–30×. On a feature with any non-trivial volume, the arithmetic is unignorable.

## The three routing patterns

Each pattern is a different answer to the question "which model handles this request?"

### Pattern 1 — cascade routing (cheap first, escalate on confidence)

**Idea.** Run the request against the small (cheap) model first. Inspect the result. If it looks like the small model was confident and correct, ship it. If not, re-run against the frontier model.

**When to use.** Any feature whose task admits a **confidence signal** — the model returns a `confidence` field, the output is a structured JSON you can validate, or a small classifier can label the result as "good enough" cheaply. Classification, extraction, retrieval reranking, and simple summarization all fit; open-ended generation often does not.

**Skeleton.**

```python
CHEAP_MODEL     = "claude-haiku-4-5-20251001"   # or the current small tier
FRONTIER_MODEL  = "claude-opus-4-7"             # or the current frontier tier

def classify_cascade(user_input: str) -> dict:
    cheap_result = call_model(CHEAP_MODEL, user_input)
    if is_confident(cheap_result):
        return cheap_result
    return call_model(FRONTIER_MODEL, user_input)

def is_confident(result: dict) -> bool:
    """
    Task-specific. Examples:
      - result parsed as valid JSON matching the schema
      - result['confidence'] > 0.7 (if the model returns one)
      - result['label'] appears in the known label set
    """
    ...
```

**Economics.** If p is the fraction of requests that escalate, and `C_cheap` and `C_frontier` are the per-call costs, average cost per call is:

```
avg_cost = C_cheap + p * C_frontier
```

For `p = 0.2` and `C_frontier = 20 * C_cheap`, average cost is `C_cheap + 4 * C_cheap = 5 * C_cheap` — versus `20 * C_cheap` for always-frontier. A 4× cost reduction from a five-line change.

**Failure mode to watch.** The `is_confident` function is the whole game. If it returns True too often, you ship low-quality small-model outputs. If it returns True too rarely, you escalate almost everything and pay both the small-model call and the frontier call, which is *worse* than always-frontier. Measure `p` in production and re-tune the confidence check when it drifts.

### Pattern 2 — fast-first-then-escalate (latency-driven cascade)

**Idea.** The user is waiting. Show the small model's result immediately — often good enough — and asynchronously invoke the frontier model in the background. If the frontier disagrees meaningfully, quietly update the UI.

**When to use.** User-facing features where **perceived latency matters more than absolute correctness** on the first paint. Chat interfaces are the canonical case: users read the small model's answer while the frontier one is still generating, and only see a change if the frontier substantively disagrees.

**Skeleton.** This is really the pattern from mod-107 chapter 3 (async fan-out) combined with cascade logic:

```python
async def answer_fast_first(user_input):
    cheap_task    = asyncio.create_task(call_model_async(CHEAP_MODEL, user_input))
    frontier_task = asyncio.create_task(call_model_async(FRONTIER_MODEL, user_input))

    cheap_result = await cheap_task
    yield {"type": "provisional", "answer": cheap_result}

    frontier_result = await frontier_task
    if disagrees(cheap_result, frontier_result):
        yield {"type": "final", "answer": frontier_result}
    # else: keep the cheap result, don't re-render
```

**Economics.** You pay for both calls on every request. Fast-first is not a cost optimisation — it's a **latency optimisation** that costs money. The trade is worth it when the small-model latency is 3–5× faster than the frontier model *and* the user actually benefits from seeing something sooner.

**Failure mode to watch.** UI churn. If the frontier reply meaningfully changes the answer more than ~5% of the time, users see the answer flip and lose trust. Either the small model is too weak (go pure frontier) or the disagreement check is too sensitive.

### Pattern 3 — batch APIs (no-latency routing)

**Idea.** Some LLM workloads don't need to respond to a user at all. Overnight enrichment jobs, corpus classification, backfill runs, LLM-as-judge evaluations — everything you'd run in a cron. Both major providers offer **batch APIs** that accept a large collection of requests, run them within a slower SLA (often up to 24 hours), and return the results at a **substantial discount** over the interactive per-token price.

**When to use.** Any workload whose latency budget is measured in hours, not seconds. Cron jobs. Eval runs. Data enrichment. Backfills. Most eval and retrieval-index builds live here.

**Reference implementations.**

- OpenAI Batch API: <https://platform.openai.com/docs/guides/batch>
- Anthropic Message Batches: <https://docs.anthropic.com/en/docs/build-with-claude/batch-processing>

<!-- needs-research: confirm current batch-API discount percentages and turnaround SLAs for both providers as of 2026-08. -->

**Skeleton (Anthropic Message Batches).**

```python
import anthropic

client = anthropic.Anthropic()

batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": f"ticket-{i}",
            "params": {
                "model": "claude-opus-4-7",
                "max_tokens": 200,
                "messages": [{"role": "user", "content": ticket_text}],
            },
        }
        for i, ticket_text in enumerate(tickets)
    ]
)

# Poll until done, then fetch results:
result = client.messages.batches.retrieve(batch.id)
if result.processing_status == "ended":
    for entry in client.messages.batches.results(batch.id):
        # entry.custom_id lets you rejoin against your input
        ...
```

**Economics.** Batch typically offers ~50% off the interactive per-token price on both providers. For a nightly enrichment job on 100,000 rows, that's a 2× cost reduction for a five-line change — as long as you can wait for the results.

**Failure mode to watch.** Reaching for the batch API when the workload is actually latency-sensitive. If a user is waiting on it, you cannot batch it. If you already have a working interactive endpoint and it's cheap enough, you also don't need to.

## When to pick which pattern

| Latency budget | Feature shape | Pattern |
|---|---|---|
| Interactive; user is watching | Confidence signal available; escalation acceptable | **Cascade** |
| Interactive; latency is the pain | User will notice provisional then final | **Fast-first-then-escalate** |
| Not user-facing (cron, eval, enrichment) | Waiting hours is fine | **Batch API** |
| Interactive; no confidence signal; latency fine | — | Just pick one model (chapter 1) |

Do not mix patterns for their own sake. Cascade routing behind a fast-first shell behind a batch pipeline is expensive to build and hard to reason about. Pick the simplest pattern that fits the latency and cost budget.

## Measuring the cascade — the escalation rate `p`

For cascade routing, the metric you have to watch is the **escalation rate**: what fraction of requests fell through to the frontier model?

```
p = frontier_calls / total_requests
```

Two things to alarm on:

- **`p` climbing.** Small model got worse (retrained backend, regression), or input mix changed. Check both.
- **`p` collapsing.** Confidence check is now returning True for outputs that used to fail it. Sometimes this is fine (small model got better); often it is a broken confidence check silently shipping low-quality outputs. Sample and human-audit.

The number to report alongside `p`: **average cost per successful call**, which is `C_cheap + p * C_frontier` at steady state. This is the number that maps directly to the invoice.

## Cascade routing and prompt caching interact

If you cascade, both the small model call and the (occasional) frontier model call include the same system prompt and few-shot examples. Both are eligible for prompt caching from chapter 3. Make sure you enable caching on both models' calls — most teams do the small-model side and forget the frontier side because it's the "unusual" path.

Caveat: caches are usually per-model. A cache warmed on the small model does not help the frontier call, and vice versa. Traffic to each model has to be dense enough on its own to keep its cache warm.

## Cascade routing and evaluations interact

You should have an eval on the small model's outputs (mod-109). You should also have an eval on the **combined cascade output** — what actually shipped to the user, small or frontier. A common bug: the small model's per-call accuracy is 88% and the frontier's is 96%, and the team assumes the cascade is somewhere in between — but if the confidence check is bad, the cascade can be *worse* than either. Measure the whole pipeline, not just its parts.

## Fast-first-then-escalate — the disagreement threshold

The disagreement check for fast-first is task-specific. Examples:

- Classifier: `cheap.label != frontier.label` or (softer) `cheap.top_1 != frontier.top_1 and frontier.confidence > 0.8`.
- Extraction: field-level diff on more than 1 field.
- Summary: semantic similarity below a threshold (a small embedding call).

Whatever you pick, make it **sticky**. If the check fires 50% of the time in practice, the pattern is not fast-first — it's "always show two answers." Tune the threshold so real disagreement is uncommon, and log the disagreement rate as a metric.

## Batch APIs — practical details

Both providers' batch APIs have similar shape:

- You upload a set of requests (JSONL file for OpenAI; array for Anthropic).
- The batch runs asynchronously.
- You poll for completion, then download the results.
- Requests can succeed and fail independently — a batch is not "all or nothing."

Things to know before shipping a batch job:

- **Idempotency.** Each request needs a `custom_id` you can use to rejoin the result to the input. Do not rely on ordering; some providers do not guarantee it.
- **Retry on individual failures.** A batch that came back with 200 out of 100,000 failures is normal. Have code that identifies and re-queues those, either in the next batch or via interactive calls.
- **The SLA is a ceiling, not a target.** Batches often complete in minutes. Design for the worst case (up to 24 hours) but don't be surprised when it's faster.
- **Batch does not compose with interactive rate limits.** A batch job does not consume your interactive per-minute quota. This matters if you were previously throttling to stay under limits.

## Common mistakes

- **Cascade routing with no confidence signal.** If you can't tell whether the small model succeeded, the cascade can't work. Design the confidence signal first, then wire the cascade.
- **Assuming a cheap call plus a frontier call is cheaper than one frontier call.** In the escalated branch you pay both. Cascade is only a win if the escalation rate `p` is well below 1.
- **Fast-first on non-idempotent operations.** Chapter 3 of mod-107 warned you about this. If the small-model reply triggers a side effect (a booked meeting, a sent email), you cannot silently revise it. Fast-first requires the reply to be safely revisable.
- **Batch API for something that has a user waiting.** The batch SLA can be 24 hours. A user staring at a spinner will not appreciate this.
- **Using batch for tiny jobs.** For 10 requests, batch overhead (upload, poll, download) exceeds the discount. Batch pays off in the hundreds-of-requests-and-up range.
- **No monitoring on `p`.** Cascade routing works until the input distribution shifts. If nobody is watching the escalation rate, the cost regression happens silently.

## Summary

- Cascade routing: run the cheap model first, escalate to frontier on low confidence. Big cost wins when the confidence check is honest and the escalation rate is low. Requires a real confidence signal.
- Fast-first-then-escalate: show the small model's answer, run the frontier in parallel, revise if it meaningfully disagrees. Latency win, not cost win — you pay for both calls.
- Batch APIs: for non-user-facing workloads (crons, evals, enrichment). Roughly 50% off the interactive price on both providers; SLA measured in hours.
- Metric that matters for cascade: escalation rate `p`. Report it and alarm on it.
- Cascade composes with prompt caching (chapter 3) — enable it on both models' calls, not just one.

The next chapter closes the loop: what happens when the model or the provider you routed to is down, how to read a status page, and what graceful degradation looks like in practice.
