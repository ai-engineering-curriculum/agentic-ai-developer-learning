# Lab 02 — Tighten a Tool Description to Fix a Routing Bug

**Estimated duration:** 45–60 minutes
**Covers:** [chapter 2 — Anatomy of a Tool Definition](../02-anatomy-of-a-tool-definition.md) ("Writing the description"), [chapter 5 — When to Call a Tool vs. Generate](../05-when-to-call-a-tool.md) ("Steering the model with `tool_choice`" and "Common mistakes")

## Objective

Chapter 5 is blunt about one habit: when the model picks the wrong tool, reaching for `tool_choice` is the wrong fix. "`tool_choice` is not a substitute for good descriptions." The right fix is to tighten the description.

In this lab you will prove that claim on a single prompt and a small two-tool surface. You will advertise two tools with deliberately overlapping descriptions, run four borderline prompts, watch the model misroute, then edit **only the descriptions** and re-run. The expected result is that correct routing rises without changing the schema, the prompt, or `tool_choice`.

By the end you will have:

- A `tools_v1.json` and a `tools_v2.json` that differ only in the two `description` strings.
- A `results.csv` with 8 rows (4 prompts × 2 versions) recording which tool the model called and with what arguments.
- A short `NOTES.md` explaining which phrases in `v2` carried the fix.

> This lab complements [exercise-02](../exercises/exercise-02-multi-tool-app.md), which drives a four-tool surface through a seven-prompt suite. Here we isolate the smallest unit of iteration in tool-using apps — editing one description — so you can see the move on its own before bundling it into exercise-02's broader audit.

## Prerequisites

- Finished chapters 2 and 5.
- A funded Anthropic *or* OpenAI API key; the lab shows the Anthropic shape and the OpenAI swap is one line.
- Python 3.10+ with `anthropic` (or `openai`) installed.
- Budget: 8 short tool-advertising calls at `max_tokens=512`. Cents.

## Step 1 — Advertise two tools whose descriptions overlap

The surface is a small customer-support assistant. The two tools you will advertise are *similar enough that a vague description routes wrong*:

- `search_products(query, limit=10)` — free-text search over a product catalog.
- `get_order_status(order_id)` — look up the current state of a specific order.

Save as `tools_v1.json` — the deliberately-vague version:

```json
[
  {
    "name": "search_products",
    "description": "Search for products.",
    "input_schema": {
      "type": "object",
      "required": ["query"],
      "additionalProperties": false,
      "properties": {
        "query":  { "type": "string", "minLength": 1 },
        "limit":  { "type": "integer", "minimum": 1, "maximum": 25 }
      }
    }
  },
  {
    "name": "get_order_status",
    "description": "Look up an order.",
    "input_schema": {
      "type": "object",
      "required": ["order_id"],
      "additionalProperties": false,
      "properties": {
        "order_id": { "type": "string", "pattern": "^ORD-[0-9]{4,8}$" }
      }
    }
  }
]
```

Both descriptions are the thing chapter 2 calls "the function name in English." Neither says *when* to use the tool, and neither names a common user phrasing.

### Checkpoint

- Both tools have strict `additionalProperties: false` schemas and tight types. The routing decision depends only on the descriptions.
- Neither description mentions "status," "shipped," "catalog," or "in stock." Those are the hooks `v2` will add.

## Step 2 — Write four borderline prompts

Create `prompts.txt`. These are the user messages where the overlap matters:

```text
1. Can you check whether ORD-4421 has shipped?
2. Do you have any blue t-shirts in medium?
3. I ordered a blue t-shirt last week — can you tell me if it's on its way?
4. What t-shirts do you sell?
```

Your mental key, for grading later (do not send to the model):

- Prompt 1 → `get_order_status(order_id="ORD-4421")`. The user named an order ID.
- Prompt 2 → `search_products(query="blue t-shirt medium")`. The user is browsing the catalog.
- Prompt 3 → `get_order_status(...)`. The user mentions a product but is asking about **their** order.
- Prompt 4 → `search_products(query="t-shirts")`. Catalog browsing.

Prompt 3 is the trap: a vague description for `search_products` can steal the call because "blue t-shirt" matches catalog language.

### Checkpoint

- Four distinct prompts. Two unambiguously belong to each tool; prompts 2 and 3 share noun phrases ("blue t-shirt") so the choice depends on the surrounding verbs.
- Your mental key is explicit. Do not change it after you see the model's output.

## Step 3 — Run the v1 (vague) version

Save as `route_demo.py`:

```python
import csv
import json
import os
import sys
import time

from anthropic import Anthropic

MODEL = "claude-sonnet-4-6"

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def load_tools(path: str) -> list[dict]:
    with open(path) as f:
        return json.load(f)

def route(message: str, tools: list[dict]) -> dict:
    t0 = time.monotonic()
    r = client.messages.create(
        model=MODEL,
        max_tokens=512,
        temperature=0,
        tools=tools,
        messages=[{"role": "user", "content": message}],
    )
    wall = round(time.monotonic() - t0, 3)

    tool_calls = []
    text_blocks = []
    for block in r.content:
        if block.type == "tool_use":
            tool_calls.append({"name": block.name, "input": block.input})
        elif block.type == "text":
            text_blocks.append(block.text)

    return {
        "stop_reason": r.stop_reason,
        "tool_calls": tool_calls,
        "text": " ".join(text_blocks),
        "wall_s": wall,
    }

def main(version: str, tools_path: str, prompts_path: str) -> None:
    tools = load_tools(tools_path)
    with open(prompts_path) as f:
        prompts = [line.rstrip("\n") for line in f if line.strip() and not line.startswith("#")]

    out_path = f"results_{version}.csv"
    with open(out_path, "w", newline="") as out:
        w = csv.writer(out)
        w.writerow(["idx", "version", "prompt", "stop_reason", "tool_calls_json", "text", "wall_s"])
        for i, prompt in enumerate(prompts, start=1):
            r = route(prompt, tools)
            w.writerow([i, version, prompt, r["stop_reason"], json.dumps(r["tool_calls"]), r["text"], r["wall_s"]])
            calls = ", ".join(c["name"] for c in r["tool_calls"]) or "(no tool)"
            print(f"[{version}][{i}] stop={r['stop_reason']:<10} calls={calls:<35} text={r['text'][:50]!r}")

if __name__ == "__main__":
    version = sys.argv[1]           # "v1" or "v2"
    tools_path = sys.argv[2]        # "tools_v1.json" or "tools_v2.json"
    main(version, tools_path, "prompts.txt")
```

Run v1:

```bash
python route_demo.py v1 tools_v1.json
```

### Expected output (shape)

Four lines of output, one per prompt, plus `results_v1.csv` with four rows. The lines will look like:

```
[v1][1] stop=tool_use   calls=get_order_status         text=''
[v1][2] stop=tool_use   calls=search_products          text=''
[v1][3] stop=tool_use   calls=search_products          text=''   ← the misroute
[v1][4] stop=tool_use   calls=search_products          text=''
```

The *specific* prompt the model misroutes under `v1` will vary by model snapshot. The claim this lab is testing is qualitative: at least one borderline prompt (usually prompt 3) gets routed to `search_products` under a vague description, when the user clearly meant `get_order_status`.

<!-- needs-research: exact rate at which a modern Sonnet-tier model misroutes prompt 3 under the vague v1 descriptions. The chapters make a qualitative claim ("the model picks between tool names more reliably than between enum values," "description is where the model decides whether to call this tool at all") but do not pin a number. Record what you observe, not a canned expectation. -->

### Checkpoint

- You ran all four prompts and have a `results_v1.csv` on disk.
- At least one prompt was routed to a tool you did not expect **under your mental key**. If every prompt routed correctly under `v1`, your schemas are doing more work than the descriptions — add a prompt to the suite where only description text could distinguish (e.g., "Is the blue shirt arriving soon?"). This is the data you need for Step 4.

## Step 4 — Write v2: tighten only the descriptions

Copy `tools_v1.json` to `tools_v2.json`. Edit **only** the two `description` strings. Do not touch names, schemas, or anything else.

Apply chapter 2's three rules:

1. **What the tool does**, in one sentence.
2. **When to use it**, in one short clause — ideally with a likely user phrasing.
3. **When *not* to use it**, naming the other tool by name (chapter 5's "anti-overlap descriptions").

A workable `v2`:

```json
[
  {
    "name": "search_products",
    "description": "Search the product catalog by free-text query. Use when the user is asking what the store sells, whether a specific product is in stock, or what sizes/colors are available. Do NOT use this to look up the status of an order the user has already placed — use get_order_status for that.",
    "input_schema": { "...": "..." }
  },
  {
    "name": "get_order_status",
    "description": "Look up the current status of a specific customer order by order ID. Use when the user mentions an order (by ID or by phrasing like 'my order', 'where is it', 'has it shipped', 'when will it arrive'). Do NOT use this to browse or search the catalog — use search_products for that.",
    "input_schema": { "...": "..." }
  }
]
```

Three things to notice before running:

- Both descriptions name common user phrasings. Chapter 2: "a description that is the function name in English … tells the model nothing about *when* to call it."
- Both descriptions include an anti-overlap clause that names the other tool by name.
- Schema fields and names are unchanged. If your fix for prompt 3 required changing the schema (e.g., tightening `search_products`'s `query` with a `minLength`), you have left the scope of this lab — put that change in exercise-02 instead.

### Checkpoint

- Diff v1 vs v2. The only lines that changed should be the two `description` values.
  ```bash
  diff tools_v1.json tools_v2.json
  ```
- Each new description includes an explicit "when NOT to use" clause that names the sibling tool.

## Step 5 — Run v2 and compare

```bash
python route_demo.py v2 tools_v2.json
```

Now consolidate both runs into `results.csv`:

```bash
cat results_v1.csv > results.csv
tail -n +2 results_v2.csv >> results.csv
```

Open it and fill in a 4-row scoring table in `NOTES.md`:

| prompt | expected tool      | v1 chose           | v2 chose           | notes                              |
|--------|--------------------|--------------------|--------------------|------------------------------------|
| 1      | get_order_status   | …                  | …                  |                                    |
| 2      | search_products    | …                  | …                  |                                    |
| 3      | get_order_status   | …                  | …                  | the trap — did v2 fix it?          |
| 4      | search_products    | …                  | …                  |                                    |

### Expected shape of the result

- The prompts that were already correct under `v1` (usually 1, 2, 4) stay correct under `v2`. Tightening a description should never cost you a correct route.
- At least one prompt that was wrong under `v1` becomes correct under `v2` — most often prompt 3.
- If `v2` breaks a prompt that `v1` had right, the fix over-shot: you added an anti-overlap clause that now blocks the legitimate call. Re-read the `v2` descriptions and remove the clause that fired incorrectly.

### Checkpoint

- Count correct routes per version in `NOTES.md`. If `v2 >= v1` *and* `v2` fixed the trap case, the "description is prompt design" claim lands for this run.
- If you see no difference, chapter 5 says the next thing to try is **not** `tool_choice` — it is tightening further (a sharper phrasing of "my order" vs. "products you sell") or removing one tool from the surface entirely. Flip `tool_choice` to `{"type": "tool", "name": "get_order_status"}` *only* to confirm that the model can produce correct arguments for prompt 3 when forced; that is a diagnostic, not a fix.

## Step 6 — Write the debrief

In `NOTES.md`, answer in 6–10 sentences:

1. Which prompt(s) did `v2` fix? Copy the *specific phrase or clause* you added to each description that you believe carried the fix. Deleting that phrase is the test; you do not need to run it.
2. Did any prompt that worked under `v1` break under `v2`? If so, which clause over-shot, and what is the minimal rewording that keeps the fix without the regression?
3. Chapter 5 says `tool_choice` is "not a substitute for good descriptions." Based on this run, in what situation *would* you reach for `tool_choice` — i.e., when is the description *not* the right lever? (Hint: the chapter names a few: triage steps, forced structured output, final synthesis turns.)

## Debrief

Editing one description is the smallest, cheapest iteration in a tool-using app. It costs 20 tokens on every turn and routinely turns a misrouted call into a correct one. When you reach for `tool_choice` or add a fifth tool to "cover" a confusing case, ask whether the description could carry the fix instead — chapter 2 says that is almost always the right first move, and chapter 5 says it beats `tool_choice` as a routing lever in the vast majority of cases.

The habit worth carrying out of this lab: when you see a misroute in a log, add the line to a running list. Once a week, open the tool with the worst count and spend ten minutes tightening its description. That loop is what keeps an app with 6+ tools behaving well over time.

## Stretch

- **Run the suite with `tool_choice={"type": "tool", "name": "get_order_status"}` on `v1`** for prompts 2 and 4 (the ones that correctly routed to `search_products`). Did the model produce sensible arguments, or did it invent an `order_id` out of thin air to satisfy the forced call? Chapter 5's "Treating `tool_choice='any'` as a fix" lands here — forcing a tool when the prompt did not warrant it produces nonsense arguments.
- **Add a third, confusable tool** (e.g., `lookup_order_for_customer(customer_email)`) with a vague description. Re-run the suite under `v2`'s tightened descriptions for the first two tools. Does the new tool steal calls that `v2` had correctly routed? That is the model showing you which anti-overlap clauses you now need to add.
- **Repeat with the other provider.** The claim ("description is the primary routing lever") is model-agnostic in chapter 5, but the exact phrasings that move the needle differ. Note which phrases carried the fix on each provider in `NOTES.md`.
