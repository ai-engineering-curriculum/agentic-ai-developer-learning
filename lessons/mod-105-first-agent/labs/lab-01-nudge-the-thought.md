# Lab 01 — Nudge the Thought: Make Your Agent Self-Narrate

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 2 — The ReAct Pattern](../02-the-react-pattern.md) ("Why the Thought matters even if the model already writes it"), [chapter 3 — Implementing the ReAct Loop](../03-implementing-the-react-loop.md) ("Surfacing intermediate text")

## Objective

Chapter 2 claims the Thought slot is "optional" in the native protocol — sometimes the assistant turn is *just* a `tool_use` block with no leading text — but a one-sentence system-prompt nudge "reliably produces Thought text," turning a silent agent into a self-narrating one at the cost of "a few extra output tokens per step." Chapter 3's "Surfacing intermediate text" section then says that readable Thoughts are the single most useful thing in a trace when something goes wrong.

In this lab you will prove both claims on a single prompt and a two-tool surface. You will run the same user message through two system prompts — one bare, one with the chapter-2 nudge — and look at what each run put in the `text` content block before the `tool_use` block. The expected result is that the bare prompt produces mostly empty `text` content (or a terse one-word lead-in), and the nudged prompt produces a one-sentence Thought that makes the trace self-explanatory.

By the end you will have:

- A `thought_demo.py` that loads two system prompts (`system_a_bare.txt`, `system_b_nudged.txt`) and runs one tool round-trip under each.
- A `trace.jsonl` with 4 rows (2 prompts × 2 system versions) capturing the `text` and `tool_use` blocks per step.
- A short `NOTES.md` quoting the exact Thought the model wrote under `system_b` for each run — and a count of how many characters the Thought added compared to `system_a`.

> This lab complements [exercise-01](../exercises/exercise-01-react-loop-intro.md), which already uses the chapter-2 nudge and prints every Thought to stdout. Exercise-01 takes the nudge for granted; here we isolate it so you can see what the nudge *alone* earns before bundling it into the broader loop you build there.

## Prerequisites

- Finished chapters 1, 2, and 3.
- A funded Anthropic *or* OpenAI API key. The lab shows the Anthropic shape; the OpenAI swap uses `tool_calls` / `role: "tool"` as described in chapter 3.
- Python 3.10+ with `anthropic` (or `openai`) installed.
- Budget: 4 short tool-advertising calls at `max_tokens=512` plus up to 4 follow-up turns. Cents.

## Step 1 — Define the two tools and the two prompts

Save `tools.py`:

```python
GET_WEATHER_TOOL = {
    "name": "get_weather",
    "description": (
        "Return current weather (temperature in F, conditions) for a known "
        "city. Use for any weather-related question. Raises if the city is "
        "unknown."
    ),
    "input_schema": {
        "type": "object",
        "required": ["city"],
        "properties": {"city": {"type": "string"}},
    },
}

SET_THERMOSTAT_TOOL = {
    "name": "set_thermostat",
    "description": (
        "Set the thermostat to a target temperature in Fahrenheit. Use when "
        "the user explicitly asks to change the temperature. Acceptable "
        "range is 60–80°F."
    ),
    "input_schema": {
        "type": "object",
        "required": ["target_f"],
        "properties": {
            "target_f": {"type": "integer", "minimum": 60, "maximum": 80},
            "note":     {"type": "string"},
        },
    },
}

WEATHER = {
    "Seattle":   {"tempF": 54, "conditions": "rain"},
    "Phoenix":   {"tempF": 102, "conditions": "sun"},
    "Boston":    {"tempF": 41, "conditions": "snow"},
    "Tokyo":     {"tempF": 68, "conditions": "cloudy"},
}

THERMOSTAT_LOG: list = []


def get_weather(city: str) -> dict:
    if city not in WEATHER:
        raise ValueError(f"unknown city: {city}")
    return {"city": city, **WEATHER[city]}


def set_thermostat(target_f: int, note: str | None = None) -> dict:
    if not (60 <= target_f <= 80):
        raise ValueError(f"target_f {target_f} outside 60–80")
    THERMOSTAT_LOG.append({"target_f": target_f, "note": note})
    return {"ok": True, "set_to_f": target_f}
```

Save `system_a_bare.txt` — deliberately silent, no Thought instruction:

```text
You are a helpful assistant. You may call tools to look up information or
perform actions.
```

Save `system_b_nudged.txt` — chapter 2's exact nudge, pasted verbatim:

```text
You are a helpful assistant. You may call tools to look up information or
perform actions. Before calling a tool, briefly state in one sentence what
you are looking for and which tool you intend to use. After receiving a tool
result, briefly state what you learned before deciding the next step. Keep
these statements short — one sentence each.
```

### Checkpoint

- The two system prompts differ *only* in the Thought-nudge paragraph. Diff them to be sure:
  ```bash
  diff system_a_bare.txt system_b_nudged.txt
  ```
- Both tools have strict descriptions. The experiment is about the system prompt, not the tools.

## Step 2 — Write the runner

Save `thought_demo.py`:

```python
import json
import os
import sys
import time

from anthropic import Anthropic

from tools import (
    GET_WEATHER_TOOL,
    SET_THERMOSTAT_TOOL,
    get_weather,
    set_thermostat,
)

MODEL = "claude-sonnet-4-6"

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

TOOLS = [GET_WEATHER_TOOL, SET_THERMOSTAT_TOOL]


def dispatch(name: str, args: dict):
    if name == "get_weather":    return get_weather(**args)
    if name == "set_thermostat": return set_thermostat(**args)
    raise RuntimeError(f"unknown tool: {name}")


def run_one(system: str, user_message: str, *, max_steps: int = 4) -> dict:
    messages = [{"role": "user", "content": user_message}]
    steps = []
    for step in range(max_steps):
        t0 = time.monotonic()
        r = client.messages.create(
            model=MODEL,
            max_tokens=512,
            temperature=0,
            system=system,
            tools=TOOLS,
            messages=messages,
        )
        wall = round(time.monotonic() - t0, 3)

        text = "".join(b.text for b in r.content if b.type == "text")
        tool_uses = [
            {"id": b.id, "name": b.name, "input": b.input}
            for b in r.content if b.type == "tool_use"
        ]
        steps.append({
            "step": step,
            "stop_reason": r.stop_reason,
            "text": text,
            "text_chars": len(text),
            "tool_uses": tool_uses,
            "wall_s": wall,
        })

        if r.stop_reason in ("end_turn", "max_tokens"):
            return {"steps": steps, "final_text": text}

        messages.append({"role": "assistant", "content": r.content})
        tool_results = []
        for b in r.content:
            if b.type != "tool_use":
                continue
            try:
                payload = dispatch(b.name, b.input)
            except Exception as e:
                payload = {"error": str(e)}
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": b.id,
                "content": json.dumps(payload),
                **({"is_error": True} if "error" in payload else {}),
            })
        messages.append({"role": "user", "content": tool_results})

    return {"steps": steps, "final_text": steps[-1]["text"]}


def main() -> None:
    system_a = open("system_a_bare.txt").read()
    system_b = open("system_b_nudged.txt").read()

    prompts = [
        "What's the weather in Seattle?",
        "It's snowing in Boston — turn the heat up a little for me.",
    ]

    with open("trace.jsonl", "w") as f:
        for prompt in prompts:
            for version, system in [("a_bare", system_a), ("b_nudged", system_b)]:
                result = run_one(system, prompt)
                row = {"version": version, "prompt": prompt, **result}
                f.write(json.dumps(row) + "\n")
                lead = result["steps"][0]
                preview = (lead["text"][:80] + "…") if len(lead["text"]) > 80 else lead["text"]
                print(f"[{version}] {prompt!r}")
                print(f"  step 0 text ({lead['text_chars']} chars): {preview!r}")
                print(f"  step 0 tool_uses: {[t['name'] for t in lead['tool_uses']]}")


if __name__ == "__main__":
    main()
```

Run it:

```bash
python thought_demo.py
```

### Expected output (shape)

Four stanzas, one per (prompt, version). The *shape* that matters:

```
[a_bare] "What's the weather in Seattle?"
  step 0 text (0 chars): ''
  step 0 tool_uses: ['get_weather']
[b_nudged] "What's the weather in Seattle?"
  step 0 text (62 chars): "I'll look up the current weather in Seattle with get_weather."
  step 0 tool_uses: ['get_weather']
[a_bare] "It's snowing in Boston — turn the heat up a little for me."
  step 0 text (0 chars): ''
  step 0 tool_uses: ['get_weather']
[b_nudged] "It's snowing in Boston — turn the heat up a little for me."
  step 0 text (54 chars): "First I'll check Boston's weather with get_weather."
  step 0 tool_uses: ['get_weather']
```

The exact character counts and wordings vary by model snapshot. The claim being tested is *qualitative*: under `system_a_bare` the `text` block before the tool call is empty (or a single word), and under `system_b_nudged` it is a one-sentence Thought that names the tool.

<!-- needs-research: exact rate at which a modern Sonnet-tier model emits an empty text block before a tool_use when given no Thought nudge. Chapter 2 says "sometimes the assistant turn is just a tool_use block with no leading text" but does not pin a number. Record what you observe. -->

### Checkpoint

- `trace.jsonl` has exactly 4 rows.
- For every row, `steps[0]["tool_uses"]` contains the correct tool. The nudge should *not* change which tool the model picks — only whether it writes a Thought beforehand.
- In at least one `a_bare` row, `steps[0]["text_chars"] == 0`. In every `b_nudged` row, `steps[0]["text_chars"]` is noticeably larger than the paired `a_bare` row. If both rows for the same prompt have non-empty `text`, you may need a prompt the model would otherwise answer tersely — the second (thermostat) prompt is included because it tends to elicit lead-in text even on the bare system.

## Step 3 — Read the Thoughts

For each of the two `b_nudged` rows, copy the exact `text` block from `trace.jsonl` into `NOTES.md` under this table:

| prompt                                                     | step 0 text (a_bare) | step 0 text (b_nudged) | character-count delta |
|------------------------------------------------------------|----------------------|------------------------|-----------------------|
| What's the weather in Seattle?                             | …                    | …                      | +N chars              |
| It's snowing in Boston — turn the heat up a little for me. | …                    | …                      | +N chars              |

Then, in `NOTES.md`, answer in 3–6 sentences:

1. For the second prompt (Boston → thermostat), the model needs two tool calls: first `get_weather`, then `set_thermostat`. Under `system_b_nudged`, what did the Thought on *step 1* (after seeing the weather observation) say? Copy it. Does it name the next tool?
2. Chapter 3's "Surfacing intermediate text" section lists three rendering choices: hide everything until the end; stream the final reply only; show every Thought plus the tool that's about to run. If you were building a chat UI on top of this loop, which choice would you pick for this two-tool agent, and why — latency, transparency, or something else?

### Checkpoint

- The two Thought quotes are in your notes verbatim. If either is empty, the nudge didn't take on that run — re-read your `system_b_nudged.txt` for typos, then re-run.
- You have an opinion about the three rendering choices and have written it down. A one-sentence answer is enough.

## Step 4 — Confirm the nudge doesn't cost you quality

The nudge adds output tokens. The trade-off is only worth it if the agent still routes correctly. Open `trace.jsonl` and verify three invariants in `NOTES.md`:

1. The tool called on step 0 is the same under `a_bare` and `b_nudged` for both prompts. (The nudge changed *what the model wrote*, not *what the model did*.)
2. The step count to `end_turn` is within ±1 across the two versions for the same prompt. (If `b_nudged` takes two more steps than `a_bare`, you have asked the model to narrate so much that it has become indecisive — rewrite the nudge to be terser.)
3. Pick either version's second prompt and sum `text_chars` across all steps. That's the output-token cost of the Thought column. Report it — this is the "few extra output tokens per step" chapter 2 said you'd pay.

If any of these fail, re-read `system_b_nudged.txt`. The chapter-2 nudge is deliberately short — "Keep these statements short — one sentence each." Longer nudges explode output length and sometimes route-drift.

### Checkpoint

- All three invariants hold, or you have a note in `NOTES.md` saying which one failed and why.

## Step 5 — Write the debrief

In `NOTES.md`, answer in 4–8 sentences:

1. In your `b_nudged` runs, pick the single clearest Thought. Would that line be understandable to a *user* if your UI rendered it directly ("Checking Boston's weather…")? Or is it written for the *developer* reading the trace? Chapter 2's "User-facing transparency" bullet treats these as the same slot; your run is a chance to test that.
2. The Thought nudge is a few words in the system prompt. If you also wanted the model to *post-observation* narrate ("Boston is below freezing, so I'll raise the heat"), which line of `system_b_nudged.txt` is already doing that work — and did the model actually produce such a line on step 1 of the second prompt? If yes, quote it; if no, suggest the smallest edit to the nudge that would reliably produce it.
3. Chapter 2 warns that ReAct as a *prompting technique* and ReAct as a *protocol* are not the same thing. In this lab, which one were you demonstrating — and would stripping the nudge back to zero break the protocol, the technique, both, or neither?

That short write-up plus `trace.jsonl` are the deliverable.

## Debrief

Three things to take with you:

- The Thought slot is "optional" in the sense that the native protocol does not require it to be populated. The Thought *as you read it* lives in the `text` content block of the assistant turn — the model writes it if the system prompt steers it to.
- A one-sentence nudge is enough. Longer "always explain your reasoning" prompts produce longer Thoughts, more output tokens, and sometimes worse routing. Chapter 2's phrasing is deliberately short.
- A trace without Thoughts is a sequence of tool calls for inscrutable reasons. A trace with Thoughts is a reviewable record. The cost is a few tokens per step. In every production agent you run later, the Thought column is what you look at first when something goes wrong.

## Stretch

- **Measure latency.** Chapter 3 says Thoughts cost "a few extra output tokens per step." Compare the `wall_s` field across `a_bare` and `b_nudged` for each prompt. Is the delta what you'd expect from the extra output tokens, or is there noise swamping the signal? Note the two numbers and your read.
- **A longer nudge.** Rewrite `system_b_nudged.txt` to ask for *two* sentences per step instead of one, or add "explain your reasoning fully." Re-run and compare step count, `text_chars` total, and whether tool-choice correctness holds. Chapter 2's "keep it short" warning should show up as a concrete regression here.
- **The other provider.** Reproduce step 2 with OpenAI's `tool_calls` shape (chapter 3 mentions the swap: `arguments` is a JSON string, follow-ups go in a `role: "tool"` message). Note any differences in how willingly the model emits leading `text` blocks without the nudge — the two providers have slightly different defaults, and you'll want to know which for exercise-01.
