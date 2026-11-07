# Lab 01 — Pydantic Rejects Before the LLM: Proving the Boundary

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 2 — FastAPI and Input Validation](../02-fastapi-and-input-validation.md) ("Pydantic request models, in practice")

## Objective

Chapter 2 claims that a Pydantic-validated request body produces a `422 Unprocessable Entity` *before* your handler runs, so the model is never called on a malformed input. The chapter phrases it twice: *"Pydantic models reject malformed bodies with a 422 before the model is called. Bad input never reaches the LLM."* And later: *"The model declaration is the validator. Spend your validation effort on naming the constraints that matter."*

This lab proves the claim mechanically. You will stand up the smallest possible FastAPI app with a `ChatRequest` matching the chapter's example, replace the real LLM call with a **counter-incrementing stub** so you can tell whether the "model" ran, and fire a known mix of valid and invalid bodies through `curl`. The counter is the witness: if Pydantic rejects at the boundary, the counter stays at the number of *valid* requests, no matter how many 2 MB bodies, empty strings, or out-of-range `max_steps` you throw at it.

This lab complements [exercise-01](../exercises/exercise-01-fastapi-llm-endpoint.md), which writes a full service and captures eight smoke-test responses. Here we isolate *one* claim — "the LLM is never called on an invalid body" — with a minimal app and a counter, so you can see what Pydantic alone earns before bundling the handler, error handlers, and OpenAPI contract together in the exercise.

By the end you will have:

- A `boundary_app.py` with a FastAPI app, a `ChatRequest` matching chapter 2's shape, and a `fake_model()` stub that increments a module-level counter and returns a canned string.
- A `probes.sh` that fires 10 labeled requests covering the chapter's validation axes.
- A short `NOTES.md` recording, for each request, the HTTP status and the counter value *after* the request completed.

## Prerequisites

- Finished chapters 1 and 2. You should recognize `Field(min_length=1, max_length=4000)` and `response_model=ChatResponse` as chapter-2 vocabulary.
- Python 3.10+ with `fastapi[standard]` installed (`pip install "fastapi[standard]"`).
- `curl` and `jq` on your PATH. No API key needed — the model is a stub.
- Budget: $0. Nothing leaves your machine.

## Step 1 — Write the minimal app

Save as `boundary_app.py`:

```python
# boundary_app.py
from __future__ import annotations

from typing import Annotated

from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(title="boundary-lab", version="0.1.0")

# Module-level witness: how many times the "model" was called.
MODEL_CALL_COUNT: int = 0


class ChatRequest(BaseModel):
    question:   Annotated[str, Field(min_length=1, max_length=4000)]
    session_id: Annotated[str | None, Field(default=None, max_length=64)]
    max_steps:  Annotated[int, Field(default=8, ge=1, le=12)]


class ChatResponse(BaseModel):
    answer:      str
    calls_so_far: int


def fake_model(prompt: str) -> str:
    """Stand-in for `client.messages.create(...)`. Increments the witness."""
    global MODEL_CALL_COUNT
    MODEL_CALL_COUNT += 1
    return f"(fake) echo: {prompt[:40]}"


@app.get("/calls")
def get_calls() -> dict[str, int]:
    """Read the counter without touching the model."""
    return {"calls": MODEL_CALL_COUNT}


@app.post("/chat", response_model=ChatResponse)
def chat(req: ChatRequest) -> ChatResponse:
    answer = fake_model(req.question)
    return ChatResponse(answer=answer, calls_so_far=MODEL_CALL_COUNT)
```

Three things worth naming before you run it:

- `MODEL_CALL_COUNT` is module-level on purpose. Chapter 1 ("What changes when concurrency shows up") flagged this exact pattern as a potential bug for shared state; here you are using it deliberately as a witness, with a single-threaded `fastapi dev` server.
- `fake_model` is where the real `client.messages.create(...)` would live. If Pydantic rejects at the boundary, `fake_model` never runs and the counter does not move.
- `/calls` reads the counter *without* calling the model. You use it between probes to check the witness.

Run it:

```bash
fastapi dev boundary_app.py
# ... INFO: Uvicorn running on http://127.0.0.1:8000
```

### Checkpoint

- `curl -s localhost:8000/calls` returns `{"calls":0}`.
- `curl -s localhost:8000/docs` renders a schema page listing `ChatRequest` with all three fields and their constraints. If `question` doesn't show `minLength: 1, maxLength: 4000`, re-read `boundary_app.py` — the `Field(...)` arguments are the validator.

## Step 2 — Write the probes

Save as `probes.sh`:

```bash
#!/usr/bin/env bash
set -u
URL=http://localhost:8000/chat

probe() {
    local label="$1"; shift
    local body="$1"; shift
    local status
    status=$(curl -s -o /tmp/last_body.json -w '%{http_code}' \
        -X POST "$URL" \
        -H 'content-type: application/json' \
        --data "$body")
    local calls
    calls=$(curl -s http://localhost:8000/calls | jq -r '.calls')
    printf '%-40s  status=%s  calls_after=%s\n' "$label" "$status" "$calls"
}

# 1-2: happy paths (should run the model)
probe "01-happy-short"        '{"question":"hello"}'
probe "02-happy-with-session" '{"question":"hi","session_id":"abc"}'

# 3-9: malformed in various axes (model must NOT run)
probe "03-empty-question"     '{"question":""}'
probe "04-missing-question"   '{}'
probe "05-question-null"      '{"question":null}'
probe "06-question-int"       '{"question":42}'
probe "07-max-steps-too-high" '{"question":"x","max_steps":99}'
probe "08-max-steps-zero"     '{"question":"x","max_steps":0}'
probe "09-session-too-long"   "$(python3 -c 'import json; print(json.dumps({"question":"x","session_id":"s"*65}))')"

# 10: a big body — chapter 2 names the 2 MB case as the one max_length is for
probe "10-oversize-question"  "$(python3 -c 'import json; print(json.dumps({"question":"x"*4001}))')"

echo
echo "Final witness:"
curl -s http://localhost:8000/calls | jq .
```

Make it executable and run it:

```bash
chmod +x probes.sh
./probes.sh
```

### Expected output (shape)

The *exact* line formatting will vary; the shape that matters is this:

```
01-happy-short                            status=200  calls_after=1
02-happy-with-session                     status=200  calls_after=2
03-empty-question                         status=422  calls_after=2
04-missing-question                       status=422  calls_after=2
05-question-null                          status=422  calls_after=2
06-question-int                           status=422  calls_after=2
07-max-steps-too-high                     status=422  calls_after=2
08-max-steps-zero                         status=422  calls_after=2
09-session-too-long                       status=422  calls_after=2
10-oversize-question                      status=422  calls_after=2

Final witness:
{ "calls": 2 }
```

Two invariants the output must satisfy:

1. **Every `status=422` row has the same `calls_after` as the row above it.** The counter does not move on a rejected request.
2. **The final witness equals the number of happy-path probes.** Here: two happy paths → `calls == 2`. Any other number means the model ran when it shouldn't have (or didn't run when it should have).

<!-- needs-research: confirm FastAPI's default behavior for request bodies larger than uvicorn's request-size limit. Chapter 2 explicitly notes uvicorn doesn't enforce a default body size cap and that Pydantic's `max_length` is the meaningful guard in this module; a 4001-character question is well under any ambient limit and should produce `422` via Pydantic, not a lower-level `413`. If your probe 10 returns `413` instead of `422`, note it and move on — the point still lands (the model didn't run) but the mechanism is one layer lower than chapter 2 describes. -->

### Checkpoint

- Two statuses only across all 10 probes: `200` for rows 1–2, `422` for rows 3–10 (or occasionally `413` for row 10; see the research note above).
- The witness read via `/calls` matches `calls_after` on the last probe.
- If a `422` row shows `calls_after` larger than the previous row's, something routed around the validator — stop and check that your handler is annotated `def chat(req: ChatRequest) -> ChatResponse:` and not `def chat(req: dict) -> dict:`. A `dict` body accepts anything and would move the witness.

## Step 3 — Read one of the 422 bodies

The status code is only half the picture; chapter 2 says the body "points at the offending field." Pick one `422` and inspect its response:

```bash
curl -s -X POST localhost:8000/chat \
    -H 'content-type: application/json' \
    -d '{"question":""}' | jq
```

You should see a `detail` array with at least one entry naming `body.question` and a message like `String should have at least 1 character`. The `type` field carries Pydantic's machine-readable reason (`string_too_short`, `missing`, `int_type`, `greater_than_equal`, `less_than_equal`, `string_pattern_mismatch`, etc.).

Repeat for probe 07 (`max_steps=99`) and probe 09 (`session_id` too long). Record the `type` from each response in `NOTES.md`:

| probe                 | HTTP status | `detail[0].type`          | `detail[0].loc`             |
|-----------------------|-------------|---------------------------|-----------------------------|
| 03-empty-question     | 422         | `string_too_short`        | `["body","question"]`       |
| 07-max-steps-too-high | 422         | `less_than_equal`         | `["body","max_steps"]`      |
| 09-session-too-long   | 422         | `string_too_long`         | `["body","session_id"]`     |

Exact `type` strings follow Pydantic v2's error names. If your Pydantic version returns a different string (e.g. `value_error`), note what you saw and move on — the point is that the body names the failing field, not the exact enum.

### Checkpoint

- The table in `NOTES.md` is filled out from your own runs, not pasted from this lab.
- At least one row's `detail[0].loc` is `["body","question"]`. That field name is coming from `ChatRequest.question` in your Pydantic model — you did not write the string anywhere in `boundary_app.py`.

## Step 4 — Count the writes you didn't make

Chapter 2 says the point of the validator is that *you don't write validation code*. Open `boundary_app.py` and count the lines you wrote for each of the following checks. The answer should be zero for all of them:

- "Reject empty `question`."
- "Reject missing `question`."
- "Reject `max_steps` outside 1..12."
- "Reject `session_id` longer than 64 chars."
- "Reject `question` longer than 4000 chars."

Every one of those is enforced by the `Field(...)` arguments on `ChatRequest`. In `NOTES.md`, count and record:

- **Validator lines written by you:** ___ (expected: 0)
- **Constraints enforced:** 5 (`min_length=1`, `max_length=4000` on `question`; `max_length=64` on `session_id`; `ge=1, le=12` on `max_steps`)
- **`if`/`raise`/`try` statements in the `chat` handler:** ___ (expected: 0)

Then answer, in 2–3 sentences, in `NOTES.md`: if you had instead written these as `if`/`raise` blocks inside the handler, name two ways the bug could surface later (a refactor, a test, a production call) that Pydantic's declarative rejection avoids. Chapter 2's closing paragraph phrases the choice as *"Spend your validation effort on naming the constraints that matter — what the input is allowed to be — and let the framework enforce them."*

### Checkpoint

- You wrote zero lines of validation code.
- Your `chat` handler is two statements long: one call to `fake_model`, one `return ChatResponse(...)`. If it is longer, you likely re-implemented a validator by hand — delete those lines and rely on the Pydantic field.

## Step 5 — Turn one validator off, re-run, and read the counter

Prove the witness is working by *breaking* one validator and watching the counter move.

Edit `boundary_app.py` and change this:

```python
question: Annotated[str, Field(min_length=1, max_length=4000)]
```

to this:

```python
question: str   # no Field() — accepts any string including the empty one
```

Save (fastapi dev will reload) and re-run **only** probe 3:

```bash
curl -s -o /dev/null -w 'status=%{http_code}\n' \
    -X POST localhost:8000/chat \
    -H 'content-type: application/json' \
    -d '{"question":""}'
curl -s localhost:8000/calls | jq
```

Expected: `status=200`, and the counter increments by 1. The empty question is now reaching `fake_model`. In `NOTES.md`, record the before/after counter values and a one-sentence note: *"With the `Field(min_length=1)` removed, the empty-question probe reached the model."*

Then **put the `Field(...)` back** and confirm the counter stops moving on the empty probe again. This is the witness confirming the validator.

### Checkpoint

- You observed the counter increment when the validator was removed, and stop incrementing when it was restored.
- `NOTES.md` has both counter readings and the one-sentence note.

## Debrief

Three things to take with you:

- **The validator is a contract, not glue.** In chapter 2's words, "the model declaration *is* the validator." The counter you wrote is proof: the handler body never ran on 8 of 10 probes, and that happened without you writing a single line of defensive code.
- **The 422 body names the failing field.** The `detail[0].loc` field — `["body","question"]` — came from the field name in your Pydantic model and the request-body hint FastAPI adds. A client (or your `curl` loop) can act on it; a free-form `{"error": "bad input"}` cannot.
- **Removing the `Field(...)` is a one-line regression that ships behavior directly to the model.** Chapter 2 names field lengths as "the smallest useful guards" and `max_length` on `question` as the thing standing between your service and a 2 MB request body "as a free path to a huge token bill." That is one line of code and most of a weekend of invoices.

## Stretch

- **Enforce the shape of the response.** `response_model=ChatResponse` in the handler decorator does more than annotate. Try returning a dict that includes an extra field (`{"answer":"x","calls_so_far":3,"debug":"internal"}`) and read the wire response. The extra field is dropped on the way out. Chapter 2 ("Pydantic response models, in practice") says this is the mechanism that keeps you from accidentally leaking `trace` or `debug` into production.
- **Oversize bodies.** Probe 10 sends a 4001-character question. Try 40_000 and 400_000 next. Does the status stay `422` or switch to something else like `413 Payload Too Large`? Note which layer rejected it (Pydantic, uvicorn, or your OS). Chapter 2's "A word on request size and rate limits" warns that uvicorn has no default body-size cap, so `max_length` on the Pydantic field is the one guard in this module.
- **Turn on the docs page.** Open `localhost:8000/docs`, click "Try it out" on `POST /chat`, and send `{"question":""}` from the browser. The response shows the same 422 body with the same `loc` trail. That is chapter 2's "documentation that can't lie" claim: the docs page renders the same schema the validator enforces, so a teammate driving your service through `/docs` sees the real constraints and not a stale README.
