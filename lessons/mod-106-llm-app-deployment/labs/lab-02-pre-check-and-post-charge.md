# Lab 02 — Pre-Check and Post-Charge: Watching the Guardrail Cross the Cap

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 5 — Cost and Usage Guardrails](../05-cost-and-usage-guardrails.md) ("A process-level daily ceiling" → "Where to call it")

## Objective

Chapter 5 lays out a specific pattern for the daily-spend guardrail: call `check_and_charge(0.0)` *before* the LLM call (so a tripped cap blocks the call) and call it again with the real estimate *after* (so the counter stays honest). The chapter is blunt about why both calls matter: *"Two checks because the order matters: the pre-check fails the very first request after the cap is hit; the post-charge keeps the counter honest. If you only post-charge, you serve one extra request after the cap; if you only pre-check, the counter never moves."*

Three sentences, three failure modes. This lab makes each one visible on your screen.

You will stand up a tiny FastAPI app with the chapter's `DailySpendGuardrail`, wire three handler variants that differ **only** in which of the two calls they make (pre-only, post-only, both), and drive each variant through a short sequence of requests at a deliberately small cap. The variant with the full pattern stops *at* the cap. The post-only variant serves exactly one extra request *past* it. The pre-only variant never trips, because the counter never moves.

This lab complements [exercise-03](../exercises/exercise-03-logging-and-usage-guardrail.md), which drops the full pattern into a production-shaped service with Docker, logging, and a `429` response. Here we isolate the two-call contract itself so you can see what each call earns before bundling it into the broader exercise.

By the end you will have:

- A `guardrail_demo.py` with three variants of `/chat` on separate routes (`/chat/both`, `/chat/post_only`, `/chat/pre_only`), all sharing one `DailySpendGuardrail` and a stub LLM call.
- A `trace.jsonl` with one row per request recording the variant, the response status, the guardrail's `spend_usd` *after* the request, and whether the stub LLM ran.
- A short `NOTES.md` recording the step at which each variant first returned `429` (or that it never did) and a one-paragraph answer to the question *"which variant is the chapter's claim in B's first sentence, and which is the second?"*

## Prerequisites

- Finished chapter 5. You should recognize `DailySpendGuardrail`, `GuardrailTripped`, and the "pre-check-then-post-charge" phrasing on sight.
- Python 3.10+ with `fastapi[standard]` installed (`pip install "fastapi[standard]"`). No pydantic-settings or Docker needed for this lab; the guardrail is the whole surface.
- `curl` and `jq` on your PATH. No API key needed — the "LLM" is a stub that returns a fixed cost estimate.
- Budget: $0. Nothing leaves your machine.

## Step 1 — Write the shared guardrail and stub

Save as `guardrail.py` (ported straight from chapter 5 with one extra method for the lab):

```python
# guardrail.py
from __future__ import annotations

import datetime as dt
from dataclasses import dataclass, field
from threading import Lock


class GuardrailTripped(RuntimeError):
    pass


@dataclass
class DailySpendGuardrail:
    cap_usd:    float
    spend_usd:  float   = 0.0
    day:        dt.date = field(default_factory=lambda: dt.date.today())
    _lock:      Lock    = field(default_factory=Lock)

    def check_and_charge(self, estimated_cost_usd: float) -> None:
        with self._lock:
            today = dt.date.today()
            if today != self.day:
                self.day = today
                self.spend_usd = 0.0
            if self.spend_usd >= self.cap_usd:
                raise GuardrailTripped(
                    f"daily cap of ${self.cap_usd:.2f} reached"
                )
            self.spend_usd += estimated_cost_usd

    def snapshot(self) -> dict:
        """Lab helper: non-mutating read for the trace."""
        with self._lock:
            return {"spend_usd": round(self.spend_usd, 4),
                    "cap_usd":   self.cap_usd,
                    "day":       self.day.isoformat()}
```

Then save `fake_llm.py`:

```python
# fake_llm.py
FAKE_COST_PER_CALL = 0.04   # dollars; one "LLM call" costs 4 cents

CALL_COUNT = 0


def fake_llm(prompt: str) -> dict:
    """Returns a dict matching run_agent's shape: final_text + a cost estimate."""
    global CALL_COUNT
    CALL_COUNT += 1
    return {"final_text": f"(fake) echo: {prompt[:40]}",
            "estimated_cost_usd": FAKE_COST_PER_CALL,
            "call_number": CALL_COUNT}
```

Two deliberate choices here:

- The cost per call is a fixed constant, not a function of tokens. Chapter 5 is about the *counter*, not the pricing; a constant cost makes the math easy to follow.
- `CALL_COUNT` is a module-level witness, the same shape you used in lab 01. You will read it between requests to see which variants actually ran the "LLM" and which short-circuited.

### Checkpoint

- `guardrail.py` contains `check_and_charge` with both the pre-check (`if self.spend_usd >= self.cap_usd: raise`) and the post-increment (`self.spend_usd += ...`). If you only copy the `raise` or only copy the `+=`, you've already broken the pattern.
- `FAKE_COST_PER_CALL = 0.04` and the cap you'll use in step 3 is `$0.10`. So three calls (`0.04 × 3 = 0.12`) cross the cap on the third charge. This is deliberate — the trap triggers on a predictable request number.

## Step 2 — Three handler variants

Save as `guardrail_demo.py`:

```python
# guardrail_demo.py
from __future__ import annotations

from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel, Field
from typing import Annotated

from guardrail import DailySpendGuardrail, GuardrailTripped
from fake_llm import fake_llm, FAKE_COST_PER_CALL

app = FastAPI(title="guardrail-lab")

# ONE guardrail shared across all three variants; reset endpoint zeroes it.
GUARDRAIL = DailySpendGuardrail(cap_usd=0.10)


class ChatRequest(BaseModel):
    question: Annotated[str, Field(min_length=1, max_length=1000)]


class ChatResponse(BaseModel):
    answer:            str
    call_number:       int
    spend_after:       float


@app.exception_handler(GuardrailTripped)
def on_guardrail(req: Request, exc: GuardrailTripped) -> JSONResponse:
    return JSONResponse(status_code=429, content={
        "detail":      str(exc),
        "error_class": "guardrail_tripped",
    })


# ----- variant A: the chapter's full pattern (pre-check + post-charge) -----
@app.post("/chat/both", response_model=ChatResponse)
def chat_both(req: ChatRequest) -> ChatResponse:
    GUARDRAIL.check_and_charge(0.0)                      # pre-check
    result = fake_llm(req.question)
    GUARDRAIL.check_and_charge(result["estimated_cost_usd"])  # post-charge
    return ChatResponse(answer=result["final_text"],
                        call_number=result["call_number"],
                        spend_after=round(GUARDRAIL.spend_usd, 4))


# ----- variant B: post-only (no pre-check) -----
@app.post("/chat/post_only", response_model=ChatResponse)
def chat_post_only(req: ChatRequest) -> ChatResponse:
    result = fake_llm(req.question)
    GUARDRAIL.check_and_charge(result["estimated_cost_usd"])  # only charge
    return ChatResponse(answer=result["final_text"],
                        call_number=result["call_number"],
                        spend_after=round(GUARDRAIL.spend_usd, 4))


# ----- variant C: pre-only (no post-charge) -----
@app.post("/chat/pre_only", response_model=ChatResponse)
def chat_pre_only(req: ChatRequest) -> ChatResponse:
    GUARDRAIL.check_and_charge(0.0)                      # only pre-check
    result = fake_llm(req.question)
    # NB: deliberately no post-charge
    return ChatResponse(answer=result["final_text"],
                        call_number=result["call_number"],
                        spend_after=round(GUARDRAIL.spend_usd, 4))


# ----- helpers -----
@app.get("/state")
def state() -> dict:
    return GUARDRAIL.snapshot()


@app.post("/reset")
def reset() -> dict:
    GUARDRAIL.spend_usd = 0.0
    return GUARDRAIL.snapshot()
```

Run it:

```bash
fastapi dev guardrail_demo.py
```

### Checkpoint

- `curl -s localhost:8000/state | jq` returns `{"spend_usd": 0.0, "cap_usd": 0.1, ...}`.
- The three `/chat/...` routes exist (confirm at `localhost:8000/docs`).
- The exception handler registers `GuardrailTripped` → `429`. If a tripped guardrail returns a `500` instead, re-read the `@app.exception_handler(GuardrailTripped)` line; the class name must match exactly.

## Step 3 — Drive the three variants

Save as `probes.sh`:

```bash
#!/usr/bin/env bash
set -u
BASE=http://localhost:8000

probe() {
    local path="$1"
    local label="$2"
    local status
    status=$(curl -s -o /tmp/last.json -w '%{http_code}' \
        -X POST "$BASE$path" \
        -H 'content-type: application/json' \
        -d '{"question":"hi"}')
    local spend
    spend=$(curl -s "$BASE/state" | jq -r '.spend_usd')
    printf '%-25s req=%s  status=%s  spend_after=%s\n' "$path" "$label" "$status" "$spend"
}

for variant in both post_only pre_only; do
    curl -s -X POST "$BASE/reset" > /dev/null
    echo "=== /chat/$variant (cap=\$0.10, cost=\$0.04/call) ==="
    for i in 1 2 3 4 5; do
        probe "/chat/$variant" "$i"
    done
    echo
done
```

Make it executable and run:

```bash
chmod +x probes.sh
./probes.sh
```

### Expected output (shape)

```
=== /chat/both (cap=$0.10, cost=$0.04/call) ===
/chat/both                req=1  status=200  spend_after=0.04
/chat/both                req=2  status=200  spend_after=0.08
/chat/both                req=3  status=429  spend_after=0.08
/chat/both                req=4  status=429  spend_after=0.08
/chat/both                req=5  status=429  spend_after=0.08

=== /chat/post_only (cap=$0.10, cost=$0.04/call) ===
/chat/post_only           req=1  status=200  spend_after=0.04
/chat/post_only           req=2  status=200  spend_after=0.08
/chat/post_only           req=3  status=200  spend_after=0.12
/chat/post_only           req=4  status=429  spend_after=0.12
/chat/post_only           req=5  status=429  spend_after=0.12

=== /chat/pre_only (cap=$0.10, cost=$0.04/call) ===
/chat/pre_only            req=1  status=200  spend_after=0.0
/chat/pre_only            req=2  status=200  spend_after=0.0
/chat/pre_only            req=3  status=200  spend_after=0.0
/chat/pre_only            req=4  status=200  spend_after=0.0
/chat/pre_only            req=5  status=200  spend_after=0.0
```

Three distinct shapes, three distinct failure modes — each matching one half of a sentence in chapter 5:

- **`both` trips on req 3 at spend=$0.08.** The post-charge of req 2 set `spend_usd` to `0.08`. Req 3's pre-check sees `0.08 < 0.10`, charges `0.0` (no-op), runs the LLM, and then the post-charge of `0.04` would push us to `0.12`. But chapter 5's `check_and_charge` raises *before* the increment when `spend >= cap`. Trace: req 3's pre-check passes (0.08 < 0.10), the LLM runs, the post-charge pre-check *still* passes (0.08 < 0.10), and the spend ends at `0.12`. The trip actually fires on **req 4's pre-check** (where `0.12 >= 0.10`).

  Hmm — look again at your own output. If `both` stopped at req 3 (spend=0.08) or at req 4 (spend=0.12) depends on exactly where the `>=` check sits relative to the `+=`. Chapter 5 puts the `>=` *before* the `+=`, meaning a request that *crosses* the cap is still served in full and the *next* request is the one that's refused. Record what you observed either way — it is the chapter's pattern as written.

- **`post_only` trips exactly one request later than `both`.** This is chapter 5's sentence 2 verbatim: *"If you only post-charge, you serve one extra request after the cap."* The pre-check that would have refused req 3 (or 4) never runs; only the post-charge fires, so you serve the whole request, charge, and *then* refuse the next one. Compare `both` and `post_only` side by side — there is exactly one `200` row of difference.

- **`pre_only` never trips.** `spend_usd` stays at `0.0` across all five requests. Chapter 5 sentence 3: *"If you only pre-check, the counter never moves."* Without a post-charge, the LLM calls happen but their cost is never recorded, so the pre-check's `self.spend_usd >= self.cap_usd` test never becomes true.

### Checkpoint

- You see exactly three distinct shapes above: full-pattern (first `429` early), post-only (one extra `200` before the `429`), pre-only (all `200`s, spend stays zero).
- In `NOTES.md`, write out the step number at which each variant first returned `429` (or wrote "never" for `pre_only`). The delta between `both` and `post_only` is the headline: *one extra served request*. Chapter 5 sentence 2, in your own witness.

<!-- needs-research: confirm the exact request index where `both` first returns 429 given chapter 5's `check_and_charge` as written. The `>=` check runs before the `+=`, so a request that *crosses* the cap is still served and the *next* request's pre-check is the one that refuses. The expected-output block above is written for that interpretation; if a reader's behavior differs by one request, the lab's claim still holds — the delta between `both` and `post_only` is the witness, not the absolute index. -->

## Step 4 — Read the chapter sentence against your output

Open `NOTES.md` and paste chapter 5's three-sentence passage:

> Two checks because the order matters: the pre-check fails the very first request after the cap is hit; the post-charge keeps the counter honest. If you only post-charge, you serve one extra request after the cap; if you only pre-check, the counter never moves.

For each sentence, cite the *exact request* in your trace that demonstrates it:

| chapter 5 claim                                        | evidence from your trace                                        |
|--------------------------------------------------------|-----------------------------------------------------------------|
| "pre-check fails the very first request after the cap" | `/chat/both` req __ returned 429 (spend had already crossed cap) |
| "post-charge keeps the counter honest"                 | `/chat/both` spend_after grew from 0.04 → 0.08 → ...            |
| "post-only: serve one extra request after the cap"     | `/chat/post_only` req 3 returned 200 while `/chat/both` req 3 returned 429 (delta = 1) |
| "pre-only: the counter never moves"                    | `/chat/pre_only` all 5 requests returned 200 with spend 0.0     |

If any row is empty, you are reading a different failure mode than the chapter — re-read your trace against the expected shape above and find the row that fits.

### Checkpoint

- Four rows in the table, each with a concrete request number or observation from your own trace.
- If one claim doesn't match your trace (e.g. `pre_only` didn't stay at zero), you most likely have a stray `+=` in `chat_pre_only` — diff the three handlers; the only intended difference is which of the two `check_and_charge` lines each one calls.

## Step 5 — The `GuardrailTripped` shape on the wire

The exception handler at the top of `guardrail_demo.py` converts `GuardrailTripped` into a `429` with a specific body. Fire one more tripped request against `/chat/both` and read the full body:

```bash
curl -i -X POST localhost:8000/chat/both \
    -H 'content-type: application/json' \
    -d '{"question":"hi"}'
```

You should see:

```
HTTP/1.1 429 Too Many Requests
...
{"detail":"daily cap of $0.10 reached","error_class":"guardrail_tripped"}
```

Record the status line and body in `NOTES.md`. Chapter 5 ("The 429 contract") says the `error_class: guardrail_tripped` field is what lets a future log query distinguish "we hit the budget cap" from "the upstream provider rate-limited us" — both are `429`, but they are not the same event. In 2–3 sentences, answer: if your log aggregator only had `status_code=429` to go on (no `error_class`), which operational question could you *not* answer that you can with the extra field?

### Checkpoint

- The 429 body contains both `detail` and `error_class` keys.
- `NOTES.md` has an answer to the "what question couldn't you answer" prompt.

## Debrief

Three things to take with you:

- **The two-call pattern is a contract, not boilerplate.** Each of the three variants is one call short of correct, and each failure mode is distinct. The "one extra request after the cap" from `post_only` is not a rounding error — it is a specific, observable extra LLM call.
- **The guardrail does not need to know what the LLM did.** Your `fake_llm` returned a fixed cost and the pattern worked identically. In exercise-03, the real `estimate_cost(result["usage"], settings.model)` plugs into the same two lines. The guardrail's shape is independent of the pricing table.
- **The lock matters when you add concurrency.** You ran this lab against a single-threaded `fastapi dev`. Chapter 5 includes the `Lock` because *"even if you run one uvicorn worker, async cooperative scheduling means simultaneous requests can interleave"* — the stretch goal tests that.

## Stretch

- **Remove the lock and run concurrent probes.** Delete the `with self._lock:` block in `check_and_charge` (or replace it with a no-op context manager). Run `/chat/both` with 20 concurrent `curl` processes (`seq 20 | xargs -P 20 -I{} curl -s -X POST localhost:8000/chat/both -H 'content-type: application/json' -d '{"question":"hi"}'`). Reset first (`curl -X POST localhost:8000/reset`). Does the total `spend_usd` after the dust settles exactly equal `0.04 × N_200s`? Chapter 5 says "The lock is cheap and correct." Reproduce the race without the lock: a pair of pre-checks that both see `0.08 < 0.10` and both charge, overshooting. Record the overshoot and put the lock back.

- **Enforce the trip on the request that would *cross* the cap, not the next one.** Rewrite `check_and_charge` so that a request whose charge *would* push `spend_usd` over `cap_usd` is refused *before* its LLM call. (Hint: pass the estimated charge into the pre-check; raise if `self.spend_usd + estimated > self.cap_usd`.) Note: you now need a cost estimate *before* the LLM runs — which chapter 5 says is "a cost-aware pre-flight" and warns is not what the daily guardrail is for. Which failure mode does this variant fix, and which does it introduce? (Hint: it over-refuses on the boundary and requires a pre-call token count.)

- **A soft warning at 80%.** Extend `DailySpendGuardrail` with a `WARNING` log once when `spend_usd` first crosses 80% of `cap_usd`, using a flag so it only fires once per day. Chapter 5's "Approaching-the-cap signal" section sketches this exactly. Re-run `/chat/both` with the probe loop; note the request number at which the warning fires (should be the first one past `$0.08`). This is a stretch goal in exercise-03 too; isolating it here lets you see the shape before the broader wiring.
