# Lab 03 — Error-as-Feedback: Watch the Model Fix Its Own Call

**Estimated duration:** 45–60 minutes
**Covers:** [chapter 6 — Handling Tool Errors and Malformed Arguments](../06-tool-errors-and-malformed-arguments.md) ("Step 1: validate arguments at the boundary", "Step 3: convert tool exceptions into tool results", "What good error feedback looks like")

## Objective

Chapter 6 claims that a structured error payload fed back as a `tool_result` turns most malformed tool calls into **self-corrections** — the model sees the field-level complaint and reissues the call with corrected arguments. In this lab you will isolate that behavior on a single tool, three error-payload wordings, and one adversarial prompt. You will see three things in sequence:

1. A good error payload (names the field, names the problem) → the model self-corrects on the next turn.
2. An opaque payload (`"error": "failed"`) → the model either loops on the same bad call or gives up vaguely.
3. A silent default payload (pretend the call succeeded) → the model confidently produces a wrong user-facing reply.

By the end you will have:

- A single-tool, hardened loop that plays back three error-payload variants against the same adversarial prompt.
- A `results.jsonl` with one row per call recording the tool arguments, the payload wording, and whether the next turn self-corrected.
- A short `NOTES.md` pinning the specific sentence in the "good" payload the model used to repair its call.

> This lab complements [exercise-03](../exercises/exercise-03-tool-error-handling.md), which hardens the full four-tool surface with Pydantic, budgets, and an adversarial prompt suite. Here we isolate *one* behavior — error payload wording → self-correction — on a single tool, so you can see what the wording alone earns before bundling validation, budgeting, and categories together in exercise-03.

## Prerequisites

- Finished chapter 6. Having also run exercise-01 helps — you already have a loop shape.
- A funded Anthropic *or* OpenAI API key. The code shows the Anthropic shape; the OpenAI swap is one line and uses the `error` key on the payload (OpenAI has no `is_error` flag).
- Python 3.10+ with `anthropic` (or `openai`) installed. `pip install pydantic` for the validation step.
- Budget: ~12 short calls at `max_tokens=512`. Cents.

## Step 1 — Define one tool and one strict args model

Save as `error_feedback.py`. The tool is a slimmed `create_refund` so the field-level constraints are easy to trip:

```python
import json
import os
import time
from typing import Literal

from anthropic import Anthropic
from pydantic import BaseModel, Field, ValidationError

MODEL = "claude-sonnet-4-6"

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

CREATE_REFUND_TOOL = {
    "name": "create_refund",
    "description": (
        "Issue a refund against an order. Only call this after the user has confirmed "
        "they want a refund and the amount."
    ),
    "input_schema": {
        "type": "object",
        "required": ["order_id", "amount_cents", "reason", "idempotency_key"],
        "additionalProperties": False,
        "properties": {
            "order_id":        {"type": "string", "pattern": "^ORD-[0-9]{4,8}$"},
            "amount_cents":    {"type": "integer", "minimum": 1, "maximum": 100_000,
                                "description": "Refund amount in cents (USD). 1 to 100000."},
            "reason":          {"type": "string",
                                "enum": ["damaged", "wrong_item", "not_received", "other"]},
            "idempotency_key": {"type": "string", "minLength": 8},
        },
    },
}

class CreateRefundArgs(BaseModel):
    model_config = {"extra": "forbid"}
    order_id:        str = Field(pattern=r"^ORD-[0-9]{4,8}$")
    amount_cents:    int = Field(ge=1, le=100_000)
    reason:          Literal["damaged", "wrong_item", "not_received", "other"]
    idempotency_key: str = Field(min_length=8)

def create_refund(order_id: str, amount_cents: int, reason: str, idempotency_key: str) -> dict:
    # Pretend the write succeeded. The point of this lab is the *error* side.
    return {
        "refund_id": "rf_FAKE000000001",
        "order_id": order_id,
        "amount_cents": amount_cents,
        "reason": reason,
    }
```

### Checkpoint

- The schema and the Pydantic model agree on types and constraints. Chapter 4's "two-sided contract" — drift here is one of the bugs the lab would otherwise reveal as "the model keeps looping."
- `additionalProperties: false` + `model_config = {"extra": "forbid"}` ensure that invented keys fail validation on *both* sides of the contract.

## Step 2 — Build three dispatcher variants

The whole lab is one small switch on how validation failures are reported. Add this below the model:

```python
def validate_args(raw_args: dict) -> tuple[bool, dict | CreateRefundArgs]:
    """Return (ok, payload_or_model). payload is an error payload on failure."""
    try:
        validated = CreateRefundArgs.model_validate(raw_args)
        return True, validated
    except ValidationError as err:
        # Returned to the caller; shaped later by the variant.
        return False, {"raw_errors": err.errors()}

def dispatch_good(raw_args: dict) -> tuple[bool, dict]:
    ok, result = validate_args(raw_args)
    if not ok:
        # Chapter 6's "good" shape: named fields, actionable messages.
        detail = [
            {"field": ".".join(str(p) for p in e["loc"]), "msg": e["msg"]}
            for e in result["raw_errors"]
        ]
        return True, {"error": "invalid_arguments", "detail": detail}  # is_error = True
    payload = create_refund(**result.model_dump())
    return False, payload

def dispatch_opaque(raw_args: dict) -> tuple[bool, dict]:
    ok, result = validate_args(raw_args)
    if not ok:
        # Chapter 6's "bad" shape: opaque, no field, no hint.
        return True, {"error": "failed"}
    payload = create_refund(**result.model_dump())
    return False, payload

def dispatch_silent(raw_args: dict) -> tuple[bool, dict]:
    ok, result = validate_args(raw_args)
    if not ok:
        # Chapter 6's anti-pattern: pretend the call succeeded with a plausible default.
        return False, {"refund_id": "rf_FAKE000000001", "order_id": raw_args.get("order_id", "ORD-UNKNOWN"),
                       "amount_cents": raw_args.get("amount_cents", 0), "reason": raw_args.get("reason", "other")}
    payload = create_refund(**result.model_dump())
    return False, payload

VARIANTS = {
    "good":    dispatch_good,
    "opaque":  dispatch_opaque,
    "silent":  dispatch_silent,
}
```

Three design notes to eyeball before running:

1. All three variants share the exact same `validate_args`. The only thing changing is the **payload shape** returned when validation fails. That is the lab's lever.
2. The `good` variant marks `is_error=True` on the Anthropic tool result (next step); `silent` marks it `False` because by construction it is pretending the call succeeded.
3. `silent` is the anti-pattern chapter 6 names: "if the tool fails, return a plausible-looking default. Do not."

### Checkpoint

- Three dispatchers. Only the failure branches differ. The happy path is identical in all three.
- You could unit-test each variant without the model: pass `{"order_id": "4421", "amount_cents": 5000, "reason": "damaged", "idempotency_key": "k"}` (missing prefix, short key) and confirm which variant returns an error payload vs. a fake success. Write the test if you have 5 minutes — it is faster than re-running the loop for every typo.

## Step 3 — The loop that records what the model did next

The loop is exactly chapter 3's shape with a per-prompt cap of 4 steps and a per-call logging line:

```python
def run_variant(variant: str, prompt: str, max_steps: int = 4) -> list[dict]:
    """Run the loop and return one log row per tool_use block the model emitted."""
    dispatch = VARIANTS[variant]
    rows: list[dict] = []
    messages = [{"role": "user", "content": prompt}]

    for step in range(max_steps):
        r = client.messages.create(
            model=MODEL,
            max_tokens=512,
            temperature=0,
            tools=[CREATE_REFUND_TOOL],
            messages=messages,
        )
        if r.stop_reason == "max_tokens":
            rows.append({"variant": variant, "step": step, "event": "max_tokens"})
            return rows
        if r.stop_reason != "tool_use":
            rows.append({"variant": variant, "step": step, "event": "end_turn",
                         "final_text": next((b.text for b in r.content if b.type == "text"), "")})
            return rows

        messages.append({"role": "assistant", "content": r.content})

        tool_results = []
        for block in r.content:
            if block.type != "tool_use":
                continue
            is_error, payload = dispatch(block.input)
            rows.append({
                "variant": variant,
                "step": step,
                "event": "tool_use",
                "name": block.name,
                "input": block.input,
                "is_error": is_error,
                "payload": payload,
            })
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": json.dumps(payload),
                "is_error": is_error,
            })
        messages.append({"role": "user", "content": tool_results})

    rows.append({"variant": variant, "step": max_steps, "event": "budget_exhausted"})
    return rows
```

And a tiny driver:

```python
PROMPT = (
    "Please refund order 4421 for fifty bucks — the box arrived crushed. "
    "I don't have an idempotency key handy, just use something."
)

if __name__ == "__main__":
    out = open("results.jsonl", "w")
    for variant in ("good", "opaque", "silent"):
        print(f"\n--- variant={variant} ---")
        rows = run_variant(variant, PROMPT)
        for row in rows:
            out.write(json.dumps(row) + "\n")
            summary = row.get("event")
            if summary == "tool_use":
                summary += f" is_error={row['is_error']} input={row['input']}"
            elif summary == "end_turn":
                summary += f" text={row['final_text'][:80]!r}"
            print(f"[{variant}] step={row['step']} {summary}")
    out.close()
```

The adversarial prompt is deliberate. It encodes three *schema violations* a model often makes:

- `"order 4421"` → no `ORD-` prefix (fails `pattern`).
- `"fifty bucks"` → not an integer; the model must convert to cents and produce `5000`. Models often do that correctly; sometimes they emit `"50"` or `50.0`.
- `"just use something"` → an invitation to invent a short/empty `idempotency_key` (fails `minLength: 8`).

Run it:

```bash
export ANTHROPIC_API_KEY=...
python error_feedback.py
```

### Expected output (shape)

For `variant=good` you should see:

- Step 0: a `tool_use` block with at least one schema violation (short `idempotency_key`, no `ORD-` prefix, or similar). `is_error=True`, payload is the structured `invalid_arguments` shape.
- Step 1: a *second* `tool_use` block with the offending field corrected. `is_error=False`, payload is the fake success.
- Step 2: `end_turn` with a user-facing confirmation ("I've issued a $50 refund for order ORD-4421 …").

For `variant=opaque`:

- Step 0: same (or similar) malformed call. `is_error=True`, payload `{"error": "failed"}`.
- Step 1: either a retry with the same mistake (chapter 6's "Retrying the same call without changing inputs" smell) *or* a text apology that gives up without naming what went wrong. Record which happened.

For `variant=silent`:

- Step 0: same malformed call. `is_error=False`, payload is a plausible-looking fake success.
- Step 1: `end_turn` with a confident wrong answer — the model tells the user the refund succeeded when nothing of the sort happened. This is the hallucinated-tracking-number failure chapter 6 warns about.

<!-- needs-research: exact behavior of a modern Sonnet-tier model on the opaque variant varies by snapshot — in some runs it re-tries the same bad args, in others it gives a vague apology. Record which you observed rather than relying on a canned expectation. -->

### Checkpoint

- Each variant produced exactly one `results.jsonl` row per tool call plus one row for the terminal event. If you see `event=max_tokens`, raise `max_tokens` to 1024 and rerun.
- Grep for `is_error`:
  ```bash
  python -c "import json; [print(json.loads(l).get('event'), json.loads(l).get('is_error'), json.loads(l).get('payload')) for l in open('results.jsonl')]"
  ```
  Confirm the `silent` variant never has `is_error=True`, by construction.

## Step 4 — Score the three variants

In `NOTES.md`, fill in a 3×3 table:

| variant | corrected on turn 2? | final reply truthful? | notes                                          |
|---------|----------------------|------------------------|------------------------------------------------|
| good    | …                    | …                      | which field did the model fix first?           |
| opaque  | …                    | …                      | did it loop, give up, or something else?       |
| silent  | …                    | …                      | did the model repeat your fake `refund_id`?    |

The three rows correspond to chapter 6's guidance (`good`), the "Bad — opaque" counter-example (`opaque`), and the explicit anti-pattern ("Returning a plausible-looking default") (`silent`).

### Checkpoint

- The `good` row has `corrected on turn 2? = yes` and `final reply truthful? = yes`.
- The `silent` row has `corrected on turn 2? = n/a` and `final reply truthful? = NO`. The model will report the fake `refund_id` to the user. That is the hallucinated-outcome bug chapter 6 names, and it is the single most dangerous row in your table.
- The `opaque` row usually sits in between — the model does not confidently lie, but it also does not recover cleanly.

## Step 5 — Write the debrief

In `NOTES.md`, answer in 6–10 sentences:

1. Copy the exact field + message string from the `good` payload the model used to fix its call on turn 2. (You can grep the previous assistant turn's input against step 0's `detail` to see which field changed.) That string is the sentence earning its tokens — write it down so you can reproduce the shape.
2. Describe what the `opaque` variant did on turn 2 in one sentence. Was it a retry on the same error, a different (also wrong) argument, or a vague apology? What would you add to the payload to move it toward the `good` variant's behavior?
3. For the `silent` variant: the model produced a confident wrong answer. Chapter 6 calls this "the hallucinated tracking number" case. Which log row — in `audit.jsonl`-style terms — would have told you about this failure in production, given that the tool call *looks* successful from the outside?

## Debrief

Three habits are worth leaving this lab with:

- **Error payloads are prompts.** The field name + the one-line problem statement is what the model reads to repair the call. "Bad input" is not enough; chapter 6 names "name the field, name the problem" as the rule.
- **`is_error` is a protocol-level flag on Anthropic** and the `error` key in the payload is the convention on OpenAI. Using them correctly is what lets the model know it should recover rather than continuing as if the call succeeded.
- **The silent-fallback pattern is the most dangerous failure mode.** The model confidently reports an outcome that never happened, your logs look clean, and the user finds out on their next statement. Return an error; let the model react; surface the failure to the user. Chapter 6: "'I couldn't reach our order system — can you try again in a minute?' is a better experience than a hallucinated tracking number."

Exercise-03 bundles this move with a retry budget, a category breakdown, and a wider adversarial prompt suite. Carry this single observation — "a good error payload is a one-sentence instruction the model can act on" — into that exercise.

## Stretch

- **Add a fourth variant that distinguishes retriable from permanent failures.** Return `{"error": "not_eligible", "detail": "refunds are not available for orders older than 90 days"}` as **data** (`is_error=False`) and compare the model's continuation to the `good` variant's. Chapter 6: "distinguishes 'you should not retry' from 'you can retry'."
- **Cap retries with a `ToolBudget`.** Set `max_per_tool=2` and rerun the `opaque` variant. Does the budget prevent the loop case you saw in Step 3, or does the model naturally give up before you hit the cap? Chapter 6's step-5 `safe_dispatch` + budget combo is what exercise-03 asks you to build; this is the one-variable version.
- **Repeat with OpenAI.** OpenAI's `role: "tool"` message does not carry an `is_error` flag; the `error` key in the payload is your only signal. Does the model's recovery behavior on the `good` variant change in any observable way? Chapter 3 names the difference; this is how you see whether it matters for recovery.
