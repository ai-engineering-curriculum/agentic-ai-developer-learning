# Lab 03 — Render the Scratchpad vs. Rely on Recall

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 5 — Conversation Memory](../05-conversation-memory.md) ("Working memory: a key:value scratchpad")

## Objective

Chapter 5 describes a small key:value scratchpad with two tools — `remember` and `recall` — and then adds one more sentence that is the whole point of the chapter's design: *"You inject the current scratchpad into the system prompt at the start of every call so the model knows what's in there without having to call `recall` every turn."* It then says exactly what breaks when you forget to render: *"a scratchpad that nobody reads. The model saves a fact and never references it because you forgot to render the scratchpad into the system prompt."*

In this lab you will prove both points on a four-turn conversation. You will stand up two `AgentSession` variants that share the same `remember`/`recall` tools and differ **only** in one thing: whether `_build_system()` renders the scratchpad into the system prompt each turn. Then you will run the same user script through each and watch what the model does differently.

The expected result is that the rendered variant answers a durable-fact question *without a tool call*, and the unrendered variant either calls `recall` (one wasted round trip) or, more often, asks the user to restate the fact.

By the end you will have:

- A `scratchpad_demo.py` with two `AgentSession` classes: `RenderedSession` (renders the scratchpad into the system prompt) and `BlindSession` (does not, so `recall` is the only access path).
- A `session.jsonl` with one row per turn per session, recording the user message, every tool call, and the reply.
- A short `NOTES.md` pinning (a) the step count at the question turn for each session, (b) whether `recall` fired, and (c) the final answer each session gave.

> This lab complements [exercise-03](../exercises/exercise-03-basic-conversation-memory.md), which exercises the full memory stack: running transcript, token-budget policy, and scratchpad durability across summarization. Exercise-03 already takes it for granted that the scratchpad is rendered into the system prompt. Here we isolate *that one rendering step* so you can see what it earns before bundling it into the broader session work there.

## Prerequisites

- Finished chapter 5.
- A funded Anthropic *or* OpenAI API key. The code shows the Anthropic shape; OpenAI uses `tool_calls` and a `role: "tool"` follow-up as described in chapter 3.
- Python 3.10+ with `anthropic` (or `openai`) installed.
- Budget: ~8 short tool-using calls at `max_tokens=512`. Cents.

## Step 1 — Define the scratchpad tools exactly once

Save as `scratchpad_tools.py`:

```python
REMEMBER_TOOL = {
    "name": "remember",
    "description": (
        "Save a single fact about the current conversation to a key:value "
        "scratchpad that survives across turns. Use for user-stated facts you "
        "expect to need later: order IDs, email addresses, preferences, "
        "confirmations. Keys are short slugs (e.g. 'preferred_carrier'). "
        "Overwrites existing values for the same key."
    ),
    "input_schema": {
        "type": "object",
        "required": ["key", "value"],
        "properties": {
            "key":   {"type": "string",
                      "description": "A short slug like 'preferred_carrier'."},
            "value": {"type": "string",
                      "description": "The fact to remember, as a short string."},
        },
    },
}

RECALL_TOOL = {
    "name": "recall",
    "description": (
        "Read back a fact previously saved with the `remember` tool. Returns "
        "the saved value or {\"error\": \"not_found\"} if no such key exists."
    ),
    "input_schema": {
        "type": "object",
        "required": ["key"],
        "properties": {"key": {"type": "string"}},
    },
}

TOOLS = [REMEMBER_TOOL, RECALL_TOOL]


def remember(session, key: str, value: str) -> dict:
    session.scratchpad[key] = value
    return {"ok": True, "key": key}


def recall(session, key: str) -> dict:
    if key in session.scratchpad:
        return {"key": key, "value": session.scratchpad[key]}
    return {"error": "not_found"}


def dispatch(session, name: str, args: dict) -> dict:
    if name == "remember": return remember(session, **args)
    if name == "recall":   return recall(session, **args)
    return {"error": f"unknown_tool: {name}"}
```

Both lab variants will import `TOOLS` and `dispatch` unchanged.

### Checkpoint

- `TOOLS` contains exactly two entries, both with required fields and a `description` that tells the model *when* to call.
- `dispatch` routes by name and takes a `session` argument so the tools can mutate `session.scratchpad`.

## Step 2 — Write the two session classes

Save as `scratchpad_demo.py`. The two classes differ in exactly one method: `_build_system()`.

```python
import json
import os
from typing import Any

from anthropic import Anthropic

from scratchpad_tools import TOOLS, dispatch

MODEL = "claude-sonnet-4-6"
client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

BASE_SYSTEM = (
    "You are a helpful customer-support assistant. "
    "When the user states a fact you expect to need later in the "
    "conversation (order IDs, preferences, confirmations), save it with the "
    "`remember` tool. Before calling a tool, briefly state what you are "
    "about to do."
)


class _SessionBase:
    def __init__(self):
        self.messages: list[dict[str, Any]] = []
        self.scratchpad: dict[str, str] = {}

    def _build_system(self) -> str:
        raise NotImplementedError

    def chat(self, user_message: str, *, max_steps: int = 6) -> dict:
        self.messages.append({"role": "user", "content": user_message})
        system = self._build_system()
        tool_calls_this_turn: list[dict] = []

        for step in range(max_steps):
            r = client.messages.create(
                model=MODEL, max_tokens=512, temperature=0,
                system=system, tools=TOOLS, messages=self.messages,
            )
            text = "".join(b.text for b in r.content if b.type == "text")
            tool_uses = [b for b in r.content if b.type == "tool_use"]
            for b in tool_uses:
                tool_calls_this_turn.append({"name": b.name, "input": b.input})

            if r.stop_reason in ("end_turn", "max_tokens"):
                self.messages.append({"role": "assistant", "content": r.content})
                return {"reply": text, "steps": step + 1,
                        "tool_calls": tool_calls_this_turn}

            self.messages.append({"role": "assistant", "content": r.content})
            tool_results = []
            for b in tool_uses:
                payload = dispatch(self, b.name, b.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": b.id,
                    "content": json.dumps(payload),
                })
            self.messages.append({"role": "user", "content": tool_results})

        raise RuntimeError(f"exceeded max_steps={max_steps}")


class RenderedSession(_SessionBase):
    """Renders the current scratchpad into the system prompt each turn."""

    def _build_system(self) -> str:
        if self.scratchpad:
            lines = [f"- {k}: {v}" for k, v in self.scratchpad.items()]
            memory_block = "Known facts about this session:\n" + "\n".join(lines)
        else:
            memory_block = "No saved session facts yet."
        return BASE_SYSTEM + "\n\n" + memory_block


class BlindSession(_SessionBase):
    """Does NOT render the scratchpad — model can only reach it via `recall`."""

    def _build_system(self) -> str:
        return BASE_SYSTEM


SCRIPT = [
    "Hi! I'm about to place a few orders this week.",
    "By the way, I prefer FedEx for any future shipments — just so you know.",
    "What's the easiest way to update my shipping address on file?",
    "Remind me — what carrier did I say I preferred?",
]


def run_session(cls) -> list[dict]:
    session = cls()
    rows = []
    for turn, user_message in enumerate(SCRIPT, 1):
        result = session.chat(user_message)
        rows.append({
            "variant": cls.__name__,
            "turn": turn,
            "user": user_message,
            "reply": result["reply"],
            "steps": result["steps"],
            "tool_calls": result["tool_calls"],
            "scratchpad_after": dict(session.scratchpad),
        })
    return rows


def main() -> None:
    with open("session.jsonl", "w") as f:
        for cls in (RenderedSession, BlindSession):
            rows = run_session(cls)
            for row in rows:
                f.write(json.dumps(row) + "\n")
                names = [t["name"] for t in row["tool_calls"]] or ["(none)"]
                reply_preview = (row["reply"][:80] + "…"
                                 if len(row["reply"]) > 80 else row["reply"])
                print(f"[{row['variant']}][turn {row['turn']}] "
                      f"steps={row['steps']} tools={names} "
                      f"reply={reply_preview!r}")
            print(f"[{cls.__name__}] final scratchpad: {rows[-1]['scratchpad_after']}\n")


if __name__ == "__main__":
    main()
```

### Checkpoint

- The only difference between `RenderedSession` and `BlindSession` is `_build_system`. Diff the two methods line by line to be sure — no hidden changes.
- `SCRIPT` has exactly four turns. Turn 2 states a durable fact; turn 3 is unrelated noise; turn 4 asks the model to recall the fact. Do not reword the turns — the exact phrasing is tuned to trigger `remember` on turn 2.

## Step 3 — Run and read

```bash
python scratchpad_demo.py
```

### Expected output (shape)

Eight stanzas, one per (variant, turn). The shape that matters:

```
[RenderedSession][turn 2] steps=2 tools=['remember'] reply="Noted — I'll keep FedEx on file for your future shipments."
[RenderedSession][turn 4] steps=1 tools=['(none)'] reply="You said you prefer FedEx."
[RenderedSession] final scratchpad: {'preferred_carrier': 'FedEx'}

[BlindSession][turn 2] steps=2 tools=['remember'] reply="Got it, I'll remember you prefer FedEx."
[BlindSession][turn 4] steps=2 tools=['recall'] reply="You said you prefer FedEx."
[BlindSession] final scratchpad: {'preferred_carrier': 'FedEx'}
```

Three things to notice:

- Both variants call `remember` on turn 2 — the tool description told them to. The scratchpads at the end of both runs contain the same key.
- On turn 4, `RenderedSession` answers in **1 step** with **no tool calls**: the fact is in the system prompt, so the model reads it off the prompt directly.
- `BlindSession` answers on turn 4 in **2 steps** with a `recall` call: it had to consult the scratchpad through the tool-use round trip, because the system prompt no longer carries the fact.

Exact wordings vary by model snapshot. The claim is qualitative: the rendered variant saves at least one tool round trip on any turn that references a durable fact.

<!-- needs-research: rate at which a modern Sonnet-tier model falls back on asking the user instead of calling `recall` under the blind variant. Chapter 5 flags "a scratchpad that nobody reads" as the failure mode but does not pin a rate. Record what you see — including whether the model asks the user to restate the fact rather than calling the tool. -->

### Checkpoint

- `session.jsonl` has 8 rows (4 turns × 2 variants).
- For both variants, turn 2's `tool_calls` contains exactly one entry with `name == "remember"`, and the resulting `scratchpad_after` includes `preferred_carrier: FedEx` (or a near-synonym — "FedEx Express" is fine).
- If `BlindSession` on turn 4 produces `tool_calls=[]` and a reply like *"I don't have that information — can you remind me?"*, the lab's trap has triggered harder than the "chapter-5 anti-pattern" wording — the model didn't even reach for `recall`. That is itself the finding; note it verbatim in `NOTES.md`.
- If `RenderedSession` on turn 4 *does* call `recall`, read the system prompt it would have seen (print `session._build_system()` after turn 3). The scratchpad line may be absent due to a typo in your branching logic. Fix and re-run.

## Step 4 — Count the savings

Open `session.jsonl` and fill in this table in `NOTES.md`:

| turn | variant          | steps | tool calls             | scratchpad after                 |
|------|------------------|-------|------------------------|----------------------------------|
| 1    | RenderedSession  | …     | …                      | …                                |
| 2    | RenderedSession  | …     | …                      | …                                |
| 3    | RenderedSession  | …     | …                      | …                                |
| 4    | RenderedSession  | …     | …                      | …                                |
| 1    | BlindSession     | …     | …                      | …                                |
| 2    | BlindSession     | …     | …                      | …                                |
| 3    | BlindSession     | …     | …                      | …                                |
| 4    | BlindSession     | …     | …                      | …                                |

Compute two quantities:

- **Step delta on turn 4** = `BlindSession.steps - RenderedSession.steps`. On a clean run this is at least 1 (the extra `recall` round trip).
- **Tool-call delta on turn 4** = number of tool calls the Blind variant made minus the number the Rendered variant made. Usually 1.

Report both numbers in `NOTES.md`. Chapter 5's "near-zero cost" claim lands when the Rendered answer is 0 tool calls and the Blind answer is 1 tool call on the same question.

### Checkpoint

- Both deltas are in `NOTES.md`.
- If either delta is negative (the Rendered variant used *more* steps than the Blind one), something else is driving the difference — most likely the model fabricating a `remember`/`recall` chain in the Rendered run. Re-read the turn 4 trace in `session.jsonl` and note what actually happened.

## Step 5 — Write the debrief

In `NOTES.md`, answer in 5–8 sentences:

1. Chapter 5's "Anti-patterns that look like memory but aren't" lists three things people reach for that don't help. Which of those three best describes what the `BlindSession` run looked like from the outside — the user saying "remember my preference" and the model agreeing, then forgetting? Quote the specific anti-pattern.
2. The `recall` tool was present under both variants. In the Rendered run, did the model *ever* call it on turn 4? If no, what signal told it the fact was already in the system prompt? Chapter 5 says the model "knows what's in there" because you injected it — argue from the trace whether that assumption held in your run.
3. Chapter 5 adds one more design note: "You can ship the scratchpad to a DB. Same pattern, different backing store." If `self.scratchpad` were backed by a Postgres row keyed by `(user_id, key)` instead of a `dict`, would the rendered-vs-blind trade-off change? (Hint: the rendered path would now be a DB read per turn; the blind path is still a tool round trip.) Argue briefly.

## Debrief

Three things to take with you:

- Giving the model a `recall` tool is *necessary but not sufficient*. Without the render step, the model has to spend a round trip to see what's in its own memory — and some runs it won't bother; it will ask the user instead. The chapter-5 phrase "nobody reads" is exactly this.
- Rendering the scratchpad is close to free: a few extra tokens on the system prompt that, if you're using prompt caching (chapter 4's cost section), are cached. The trade-off is strongly in favour of rendering until the scratchpad grows past ~1–2kB, at which point you'd start summarizing stale keys.
- The rendered variant is the *floor* of memory design. Everything production (per-user scratchpads, DB-backed persistence, vector memory for longer-term recall) sits on top of this one pattern. Exercise-03 extends it with transcript trimming and summarization — the render step there carries the scratchpad through summarization events, which is the whole reason it matters.

## Stretch

- **Shrink the base system prompt.** Chapter 5 renders the scratchpad in a specific block (`"Known facts about this session: …"`). Replace that block with just the raw `k: v` lines (no header). Re-run. Does the model still use them? This tests whether the *header* or the *content* is load-bearing.
- **Blind + pre-render once.** Modify `BlindSession` to render the scratchpad into the system prompt *once at startup* and never refresh. Run the same script. Does turn 4 still work? Does a hypothetical fifth turn that added a *second* fact ("my email is …") then asked about it also work? This probes the "every turn" part of chapter 5's wording.
- **Add a durable-fact confirmation prompt.** After turn 4, add a fifth turn: *"Also, my email is me@example.com."* Then a sixth turn: *"What email did I give you?"* Re-run both variants. Record whether the second durable fact survives the same way — and whether the model used the same key naming convention it did on turn 2. Chapter 5's "scope keys carefully" line lands here.
