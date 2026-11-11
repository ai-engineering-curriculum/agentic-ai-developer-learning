# Lab 03 — TTFT vs. TTC: Blocking vs. Streaming on One Prompt

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 5 — Measuring streaming latency honestly](../05-measuring-streaming-latency.md) ("The four measurements that matter" and "What 'streaming feels faster' actually means")

## Objective

Chapter 5 is blunt: "Streaming does not make total wall-clock any shorter. It changes the *distribution* of what the user waits for." In this lab you will send **the same prompt twice** against the same model — once blocking, once streaming — and record TTFT and TTC on each call. The streaming run's TTC should be roughly the same as the blocking run's TTC. The streaming run's TTFT should be a small fraction of its TTC. That is the honest number behind "streaming feels faster," and this lab's whole purpose is to put it in front of your own eyes in under 45 minutes.

No fan-out. No percentiles. No retries. One prompt, two calls, four numbers.

By the end you will have:

- A `ttft_vs_ttc.py` that times both variants against one provider.
- A console table comparing the four numbers.
- A one-paragraph interpretation tying each number to the claim in chapter 5 that it supports.

> This lab complements [exercise-05](../exercises/exercise-05-measure-time-to-first-token-and-tail-latency.md), which builds a full load-testing harness with percentile aggregation across many requests. Here we do the opposite: a single pair of calls, so the distinction between TTFT and TTC is sharpened before the next exercise averages it across a distribution. Keep both habits — single-call storytelling and many-call percentiles — in your toolkit.

## Prerequisites

- Finished chapter 5.
- A funded API key for one provider:
  - **Anthropic path:** `ANTHROPIC_API_KEY` exported, `pip install anthropic`.
  - **OpenAI path:** `OPENAI_API_KEY` exported, `pip install openai`.
- Budget: two short calls, one asking for a longer response (~1500 output tokens). Cents.
- Pick **one** provider for this lab. Running both at once hides the point; the stretch runs the second provider.

## Step 1 — Pick a prompt whose output is long enough for TTFT to matter

A 10-token response has essentially no gap between TTFT and TTC — the model emits one burst and stops. The contrast you want to see comes from a response of at least a few paragraphs. Use something like:

> `"Write a detailed, three-paragraph explanation of how Server-Sent Events compare to WebSockets, aimed at a backend engineer."`

Set `max_tokens` high enough that the model does not truncate (`max_tokens=1024` is plenty for three paragraphs). Keep `temperature=0` so you can re-run and get comparable numbers.

### Checkpoint

- The prompt asks for enough output that the streamed version will visibly stream across several seconds. If your prompt elicits a two-sentence answer, pick a longer prompt now.
- Everything else about the two calls — the model ID, `temperature`, `max_tokens`, the prompt text itself — is **identical**. The only variable is blocking vs. streaming.

## Step 2 — Time a blocking call

Save as `ttft_vs_ttc.py`:

```python
import os, time
from anthropic import Anthropic

MODEL = "claude-sonnet-4-6"   # substitute your dated model ID
PROMPT = ("Write a detailed, three-paragraph explanation of how "
          "Server-Sent Events compare to WebSockets, aimed at a "
          "backend engineer.")

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def blocking():
    t0 = time.perf_counter()
    r = client.messages.create(
        model=MODEL,
        max_tokens=1024,
        temperature=0,
        messages=[{"role": "user", "content": PROMPT}],
    )
    t_done = time.perf_counter()
    return {
        "variant": "blocking",
        "ttft_s": t_done - t0,            # blocking: first visible char == last char
        "ttc_s":  t_done - t0,
        "output_tokens": r.usage.output_tokens,
    }

def streaming():
    t0 = time.perf_counter()
    t_first = None
    with client.messages.stream(
        model=MODEL,
        max_tokens=1024,
        temperature=0,
        messages=[{"role": "user", "content": PROMPT}],
    ) as stream:
        for _text in stream.text_stream:
            if t_first is None:
                t_first = time.perf_counter()
        final = stream.get_final_message()
    t_done = time.perf_counter()
    return {
        "variant": "streaming",
        "ttft_s": (t_first - t0) if t_first else float("nan"),
        "ttc_s":  t_done - t0,
        "output_tokens": final.usage.output_tokens,
    }

if __name__ == "__main__":
    # A warm-up call first (chapter 5's "cold starts, warm-ups" habit).
    # We discard this one; the point is to compare steady-state numbers.
    _ = blocking()

    rows = [blocking(), streaming()]

    print(f"{'variant':>10}  {'TTFT ms':>8}  {'TTC s':>7}  {'tokens':>6}  "
          f"{'tok/s (gen)':>12}")
    for r in rows:
        ttft_ms = r["ttft_s"] * 1000
        gen_s = max(r["ttc_s"] - r["ttft_s"], 1e-6)
        tok_s = r["output_tokens"] / gen_s
        print(f"{r['variant']:>10}  {ttft_ms:>8.0f}  "
              f"{r['ttc_s']:>7.2f}  {r['output_tokens']:>6}  {tok_s:>12.1f}")
```

For the OpenAI variant, swap the two functions for:

```python
# OpenAI blocking
def blocking():
    t0 = time.perf_counter()
    r = client.chat.completions.create(
        model="gpt-4.1",
        messages=[{"role": "user", "content": PROMPT}],
        temperature=0,
        max_tokens=1024,
    )
    t_done = time.perf_counter()
    return {"variant": "blocking",
            "ttft_s": t_done - t0, "ttc_s": t_done - t0,
            "output_tokens": r.usage.completion_tokens}

# OpenAI streaming
def streaming():
    t0 = time.perf_counter()
    t_first = None
    stream = client.chat.completions.create(
        model="gpt-4.1",
        messages=[{"role": "user", "content": PROMPT}],
        temperature=0, max_tokens=1024,
        stream=True,
        stream_options={"include_usage": True},   # chapter 1: needed for usage
    )
    output_tokens = 0
    for chunk in stream:
        if chunk.choices and chunk.choices[0].delta.content:
            if t_first is None:
                t_first = time.perf_counter()
        if chunk.usage:
            output_tokens = chunk.usage.completion_tokens
    t_done = time.perf_counter()
    return {"variant": "streaming",
            "ttft_s": (t_first - t0) if t_first else float("nan"),
            "ttc_s": t_done - t0,
            "output_tokens": output_tokens}
```

Three things both snippets get right that chapter 5 insists on:

- **`time.perf_counter()`, not `time.time()`.** `perf_counter` is monotonic and not subject to NTP corrections mid-request.
- **Timer around the whole streaming context manager**, not just the initial `.create()` call. For a blocking call, the first visible character *is* the last character — TTFT equals TTC by construction.
- **Tokens per second computed over the generation phase**, not the whole TTC. `output_tokens / (ttc - ttft)` is what the provider's throughput advertisement is comparable to.

## Step 3 — Run it and read the numbers

```bash
python ttft_vs_ttc.py
```

### Expected output (shape)

Two rows. The streaming row should have a `TTFT ms` that is a small fraction of the blocking row's — somewhere in the low hundreds of milliseconds on a normal connection — and a `TTC s` that is roughly the same as the blocking row's.

```
   variant   TTFT ms    TTC s  tokens   tok/s (gen)
  blocking      5240     5.24     437          83.4
 streaming       310     5.19     437          89.4
```

<!-- needs-research: the specific TTFT / TTC numbers above are illustrative. The expected *shape* — streaming TTFT a small fraction of TTC, streaming TTC comparable to blocking TTC — is the only claim the chapter makes. Report the numbers you actually observe, not these ones. -->

Four observations tied directly to chapter 5:

1. **Blocking TTFT ≈ Blocking TTC.** For a blocking call, the first visible character is the last character. There is no "feels faster" to extract.
2. **Streaming TTFT ≪ Streaming TTC.** The whole point of the chapter. This is the number a user feels.
3. **Streaming TTC ≈ Blocking TTC.** Streaming does not make total wall-clock shorter; it changes what the user waits for.
4. **Tokens/s (generation) is similar across both variants.** The model is generating at roughly the same speed either way. Streaming exposes the token stream to you; blocking hides it. The throughput number is the same.

### Checkpoint

Answer from your output:

1. Compute the ratio `Streaming TTFT / Streaming TTC`. Is it less than 0.2? If it is close to 1, your "streaming" is being buffered somewhere (chapter 1's failure mode) — go back to lab 01's Step 3 for the diagnostic.
2. Compute `Streaming TTC - Blocking TTC`. If the difference is more than a few percent in either direction, re-run once more — a single-sample comparison is noisy. Chapter 5's whole "percentiles over means" habit says one pair of calls is not a statistically meaningful comparison; this lab is for *intuition*, not statistics.
3. For a downstream workflow step that must consume the **entire** response before starting (say, summarising the output and feeding it to the next call), which of the four numbers constrains the pipeline? Chapter 5 names it by acronym.

## Step 4 — Flip one knob and observe

Pick **one** of these modifications and re-run `ttft_vs_ttc.py`:

- **Drop `max_tokens` to 50.** With a very short response, streaming TTFT and streaming TTC collapse toward each other. Streaming buys you less when the response is short, and chapter 5 makes this explicit: streaming's win is proportional to how much of the response is available to be "felt" after the first chunk.
- **Raise `max_tokens` to 4096 and change the prompt** to request a long, detailed response (e.g., "Write six paragraphs…"). The TTFT should stay roughly the same; the TTC should grow linearly with output length. This is the shape chapter 5 names when it says "the long generations become interactive."
- **Switch models to a faster-tier model (e.g., `claude-haiku-4-5` or `gpt-4.1-mini`).** TTFT and TTC both drop — but TTFT drops less in absolute terms, because it is dominated by prompt-prefill time, not generation speed. The *ratio* TTFT/TTC can grow even as both improve.

Record which variant you ran and one sentence on what the four numbers did.

## Step 5 — Write a 5-sentence debrief

In `NOTES.md`, answer in 5–7 sentences total:

1. State chapter 5's claim in your own words, grounded in the numbers you just recorded. ("Streaming does not make total wall-clock shorter; it…")
2. For your Step-4 variant, which of the four numbers moved the most? Which moved the least? Does the chapter's model predict what you saw?
3. The product manager asks: "How fast is the feature?" Which *one* number would you quote, and which other number would you insist on reporting alongside it? Chapter 5's "reporting to a product manager" section names the exact pair.
4. Your single-call measurement here is not a reliable estimate of `p50` or `p99`. What would you do next — before quoting a number on a design review — to turn this intuition into a defensible metric? (Hint: exercise 05.)

That note is the deliverable. Commit `ttft_vs_ttc.py` and `NOTES.md`.

## Debrief

Two numbers; one lesson. When someone argues that "streaming makes the service faster," you can now disagree precisely: the service is not faster; the *user's wait* is redistributed. That redistribution is worth a lot — a 300-ms TTFT and a 5-second TTC feels snappier than a 5-second blocking call every time — but it is not a throughput win. Chapter 5's whole point is to keep the two kinds of win separate so the next feature decision (invest in caching? smaller model? streaming on a new endpoint?) is made against the right baseline.

Keep this script. When you rebuild it into the full percentile harness in exercise 05, these four numbers will become the per-request measurements that feed `p50 / p95 / p99`, but the mental model — TTFT, TTC, tokens-per-second over generation, failures-as-composition — is the one you just built here on a single call.

## Stretch

- **Second provider on the same prompt.** If you have both API keys, run the lab against the second provider too and compare. Chapter 5's "backend vs perceived latency" discussion applies: two providers' TTFT distributions can differ by more than a factor of two for identical prompts.
- **Simulate a buffering proxy.** Wrap the streaming call in a thin FastAPI `StreamingResponse` as in chapter 1, deploy it behind a reverse proxy without `X-Accel-Buffering: no`, and drive it from the lab's script. Does the server-measured TTFT match the client-measured TTFT? Chapter 5's "backend vs perceived latency" is the exact gap you will see.
- **Three-pair sample.** Run the whole script five times back-to-back (discarding the warm-up each time) and compute the min, median, and max of each number across the five pairs. Chapter 5's "the p99 of 100 samples is roughly the max" cautions directly against doing this rigorously with so few samples — but the spread itself will tell you how noisy a single-sample measurement is.
- **Blocking with `include_usage` vs streaming with it.** OpenAI charges the same per-token rate for streaming and blocking. Measure `usage.completion_tokens` on both variants — they should match for `temperature=0` on an identical prompt. If they do not, pin why; it is almost always a tokenisation difference you did not expect.
