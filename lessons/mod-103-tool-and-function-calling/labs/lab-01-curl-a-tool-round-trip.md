# Lab 01 — Curl a Tool Round-Trip

**Estimated duration:** 45–60 minutes
**Covers:** [chapter 1 — From Generation to Action](../01-from-generation-to-action.md), [chapter 3 — The Call/Result Loop](../03-call-and-result-loop.md)

## Objective

The SDK packages a tool round-trip into a few method calls. In this lab you will run the same round-trip **without the SDK**, using `curl`, so you can see the exact bytes that cross the wire on both legs of the protocol: the request that advertises the tool, the `tool_use` block the model emits, the `tool_result` message you hand back, and the model's final continuation. This is the mental model you want before the SDK is your only lens.

By the end you will have:

- A single-turn `tool_use` response in `turn1.json`, inspected for `name`, `id`, and `input`.
- A hand-authored second request in `turn2.json` that echoes the assistant turn verbatim and attaches the `tool_result`.
- A final `turn2_response.json` where `stop_reason` is `end_turn` and the model has produced the user-facing reply.
- A short write-up pinning three pieces of the wire format the SDK normally hides.

> This lab complements [exercise-01](../exercises/exercise-01-first-function-calling.md), which uses the SDK to drive the loop against a fake order DB. Here we do one round-trip by hand so you have seen the JSON yourself.

## Prerequisites

- Finished chapters 1 and 3.
- A funded Anthropic API key: `ANTHROPIC_API_KEY` exported in your shell. (The lab is written for Anthropic because the `tool_use` block arrives as a *parsed object* in the response JSON, which makes the hand-authored follow-up turn easiest to read. OpenAI shape is in the stretch section.)
- `curl` 7.70+ and `jq` on your path.
- Budget: three short calls at `max_tokens=512`. Cents.

## Step 1 — Advertise one tool and send a user prompt

Pin an exact dated model ID (`claude-sonnet-4-6` here — substitute what your account can call) and keep `temperature=0` so the run is reproducible.

Save the request body as `turn1.body.json`:

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 512,
  "temperature": 0,
  "tools": [
    {
      "name": "get_order_status",
      "description": "Look up the current status of a customer order by ID. Use when the user asks where their order is, when it will arrive, or whether it shipped.",
      "input_schema": {
        "type": "object",
        "required": ["order_id"],
        "additionalProperties": false,
        "properties": {
          "order_id": {
            "type": "string",
            "pattern": "^ORD-[0-9]{4,8}$",
            "description": "Order identifier in the form 'ORD-1234'."
          }
        }
      }
    }
  ],
  "messages": [
    { "role": "user", "content": "Where is my order ORD-4421?" }
  ]
}
```

Send it:

```bash
curl -sS https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d @turn1.body.json \
  | tee turn1.json | jq .
```

### Expected output (shape)

A JSON object whose `content` list contains a `tool_use` block, and whose top-level `stop_reason` is `"tool_use"`. The shape you want to find:

```json
{
  "id": "msg_...",
  "model": "claude-sonnet-4-6-...",
  "stop_reason": "tool_use",
  "content": [
    { "type": "text", "text": "Let me check on that for you." },
    { "type": "tool_use", "id": "toolu_...", "name": "get_order_status", "input": { "order_id": "ORD-4421" } }
  ],
  "usage": { "input_tokens": ..., "output_tokens": ... }
}
```

The leading `text` block may or may not be there — chapter 3 mentions that the model "often narrates 'Let me check that for you' before the call." That is one of the things worth noticing in Step 5.

### Checkpoint

Answer from `turn1.json` without re-running:

1. What is the exact `stop_reason`?
2. Pull the `tool_use` block's `id`, `name`, and `input` with `jq`:
   ```bash
   jq '.content[] | select(.type=="tool_use") | {id, name, input}' turn1.json
   ```
   The `id` is what you must echo back on the next turn — chapter 3 ("`tool_use_id` must match exactly") is the rule you are about to exercise.
3. Did the content list contain a leading `text` block? If so, copy its text into your notes — you will be asked whether that string should be shown to the end user.

## Step 2 — Build the follow-up turn by hand

Chapter 3's three rules for the follow-up are:

- Append the assistant's `tool_use` turn to `messages` **verbatim**.
- Append a **user** turn whose content is a list of `tool_result` blocks.
- Echo the `tool_use_id` exactly; serialize the function payload to a string.

Pretend your (fake) `get_order_status("ORD-4421")` returned `{"order_id": "ORD-4421", "status": "shipped", "carrier": "UPS", "tracking": "1Z123ABC"}`.

Extract what you need from the first response:

```bash
TOOL_USE_ID=$(jq -r '.content[] | select(.type=="tool_use") | .id' turn1.json)
echo "tool_use_id=$TOOL_USE_ID"

# Keep the entire assistant content list, verbatim.
ASSISTANT_CONTENT=$(jq -c '.content' turn1.json)
```

Now author `turn2.body.json`. Every field inside `tool_result` is required:

```bash
cat > turn2.body.json <<EOF
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 512,
  "temperature": 0,
  "tools": $(jq -c '.tools' turn1.body.json),
  "messages": [
    { "role": "user", "content": "Where is my order ORD-4421?" },
    { "role": "assistant", "content": $ASSISTANT_CONTENT },
    { "role": "user", "content": [
        {
          "type": "tool_result",
          "tool_use_id": "$TOOL_USE_ID",
          "content": "{\"order_id\": \"ORD-4421\", \"status\": \"shipped\", \"carrier\": \"UPS\", \"tracking\": \"1Z123ABC\"}"
        }
    ]}
  ]
}
EOF

jq . turn2.body.json | head -40
```

Three invariants to eyeball before sending:

1. The second message is `role: "assistant"` and its `content` field is the **list of blocks** from `turn1.json`, not a string.
2. The third message is `role: "user"` with a list containing one `tool_result` block. The `tool_use_id` matches the `id` on the assistant's `tool_use` block exactly.
3. `tool_result.content` is a **string** (JSON-encoded). Chapter 3: "Even when your function returns a dict, serialize it (`json.dumps(...)`) before sending. The model treats it as text."

### Checkpoint

Deliberately break one invariant and look at the server's reply (then fix it before Step 3):

- Remove the assistant turn entirely. Resend. What does the provider say? This exercises chapter 3's "Sending tool results without first sending the assistant turn that requested them. The provider will reject the request."
- Change one character in `tool_use_id` so it no longer matches. Resend. Does the provider reject the call, or does the model behave oddly? (Behavior here is provider-dependent; record what you observed.)

Restore the correct file before moving on.

## Step 3 — Send the follow-up turn and collect the final reply

```bash
curl -sS https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d @turn2.body.json \
  | tee turn2_response.json | jq .
```

### Expected output (shape)

```json
{
  "id": "msg_...",
  "stop_reason": "end_turn",
  "content": [
    { "type": "text", "text": "Your order ORD-4421 has shipped via UPS. The tracking number is 1Z123ABC." }
  ],
  "usage": { "input_tokens": ..., "output_tokens": ... }
}
```

The model used the `tool_result` text from your follow-up turn to produce the user-facing answer. Point at each piece of the loop one more time: the `tool_use` block in `turn1.json` was the model asking; the `tool_result` block in `turn2.body.json` was your application replying; the final `text` block in `turn2_response.json` is the continuation.

### Checkpoint

Answer from `turn2_response.json`:

1. What is the exact `stop_reason`? (If it is `tool_use`, the model wants *another* tool call — you would loop. For this prompt it should be `end_turn`.)
2. Pull the final text:
   ```bash
   jq -r '.content[] | select(.type=="text") | .text' turn2_response.json
   ```
3. Did the model repeat any field from your `tool_result` payload in the reply (carrier, tracking number)? That is the model "reading" your serialized JSON as text — chapter 4's "Return JSON-serializable data" lands here.

## Step 4 — Flip one thing and observe

Pick **one** of the following modifications and re-run Step 1 + Step 3 (writing the fake tool payload the same way). Note what changed in `NOTES.md`.

- **Ask about an ID that fails the schema's `pattern`.** Change the user message to `"Where is order 4421?"` (no `ORD-` prefix). Does the model normalize to `ORD-4421` on its own? Does it emit a `tool_use` with an argument that violates the pattern? Does the provider reject the call at the schema layer, or is enforcement advisory? Chapter 2's "pattern" guidance meets reality here.
- **Drop the tool description.** Keep the schema and name; set `description` to the empty string. Does the model still choose to call `get_order_status`? Chapter 2's claim is that the description is the part the model actually reads to choose — this is how you check it on one prompt.
- **Set `max_tokens` to 8** on the first call. What `stop_reason` do you get? If the `tool_use` block came back truncated or absent, is that a `max_tokens` finish? Chapter 3's "Stop reasons you must handle" says you should treat `max_tokens` mid-call as a failure — this is what that failure mode looks like.

Record which variant you ran and one sentence on what the model did.

## Step 5 — Write the debrief

In `NOTES.md`, answer in 6–10 sentences total:

1. Name three pieces of the wire format the SDK normally hides that you can now point to (`stop_reason`, `tool_use.id`, `tool_use.input` as a parsed object, `tool_result` inside a `user`-role turn, `tool_result.content` being a string, the `usage` block, etc.).
2. For the leading `text` block from Step 1's checkpoint: would you surface it to the end user, suppress it, or make it configurable? Chapter 3's "What the user sees" has a rule worth citing by name.
3. Which of the three Step-4 variants *should* break the loop in a production app, and what is the smallest change (description, schema, prompt, or loop code) that would fix it?

That write-up is the deliverable.

## Debrief

You now have a concrete picture of the loop that every chapter after this one takes as given. Three details are easy to forget and expensive to debug:

- The assistant's `tool_use` turn is part of the conversation and must be appended back **verbatim** — including any leading `text` block.
- The `tool_result` lives inside a **user**-role turn on Anthropic, not an `assistant` turn and not a dedicated `tool` role.
- `tool_result.content` is a **string**. If your function returns a dict, you serialize it before handing it to the API.

When the SDK does something surprising in exercise-01 or exercise-02 — a missing field, a mismatched id, a dropped `tool_use` block — drop back to this level. `curl` and `jq` do not lie.

## Stretch

- **OpenAI shape, same prompt.** Run the first call against `https://api.openai.com/v1/chat/completions` with a `tools` array whose entries use `{"type": "function", "function": {...}}`. Note two differences chapter 3 flags: the response puts `tool_calls` on the assistant message with `function.arguments` as a **JSON string**, and the follow-up turn is a `role: "tool"` message, not a `user` turn with a `tool_result` block. You will do a `json.loads` on `function.arguments` before using it.
- **Two parallel tool calls.** Advertise a second tool (say, `get_customer_shipping_address`) and ask "Where is ORD-4421, and what address is it shipping to?" Does the first response contain one `tool_use` block or two? Chapter 3 ("Parallel tool calls") says you must execute every call in the assistant turn and send all results in a single follow-up user turn — eyeball whether your single hand-authored turn covers both `tool_use_id`s.
- **Replay without `tools=`.** Repeat Step 3 but strip the `tools` array out of `turn2.body.json`. The chapter claim ("drop them and the model forgets they exist") predicts that the model will not request further tool calls — but for a prompt that only needs one round-trip that is invisible. Try a prompt that needs *two* tool calls instead and see whether the model gives up on the second.
