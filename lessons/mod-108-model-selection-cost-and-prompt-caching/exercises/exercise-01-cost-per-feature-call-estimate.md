# Exercise 01 — Cost per feature call: estimate before you ship

Paired with [chapter 2 — cost per feature call and enforcing token budgets](../02-cost-per-call-and-token-budgets.md).

**Estimated effort:** 60–90 minutes.

## Objective

Turn a feature spec into a defensible dollar-cost estimate — per call at p50 and worst case, per day at expected volume, per month at expected volume. Then wire a **per-request token budget** and a small **per-user token budget** into the call site so the estimate is bounded above by enforcement, not by hope.

## Problem statement

You are shipping a support-ticket classifier. Spec:

- **Input.** A support ticket body. Typical size 200–500 tokens. Worst-case (99th percentile of real inputs) around 2,000 tokens.
- **Prompt template.** A system prompt of ~700 tokens plus three few-shot examples of ~250 tokens each — a fixed ~1,450 tokens of stable content prepended to every request. (Cache this in exercise 03; for now, treat every input token as fresh.)
- **Output.** A JSON object of shape `{"category": "...", "priority": "...", "sentiment": "..."}` — typically ~40 tokens. `max_tokens` cap is 150.
- **Expected volume.** 50,000 requests / day.
- **Model.** You pick one from your preferred provider (Anthropic or OpenAI). Use a small-tier model.
- **Users.** ~1,000 daily active users; the p99 user makes ~200 requests / day.

## Requirements — the estimate

Produce a small script (`estimate.py` or similar) that answers:

1. **Per-call cost at p50.** Input = template + 350 tokens; output = 40 tokens.
2. **Per-call cost at worst case.** Input = template + 2,000 tokens; output = `max_tokens` = 150 tokens.
3. **Daily cost at expected volume.** 50,000 calls at p50, and 50,000 calls at worst case.
4. **Monthly cost.** Multiply daily by 30.

Requirements:

- Fetch the current per-million-token input and output prices for your chosen model from the provider's pricing page. Encode them at the top of the script with a comment naming the page and the date you fetched them — prices change.
- Count the template tokens using the provider's own tokenizer / token-counting endpoint. Do not eyeball or use `len(str) / 4`.
- Print a small table:

  ```
  scenario         input tokens   output tokens   cost / call
  p50              1,800          40              $0.0000...
  worst case       3,450          150             $0.0000...

  daily p50 (50k)                                 $...
  daily worst (50k)                               $...
  monthly p50                                     $...
  monthly worst                                   $...
  ```

## Requirements — the enforcement

Add a callable at the boundary of your service that enforces two caps before making the SDK call:

1. `enforce_request_budget(input_tokens, max_output_tokens, max_input_tokens=?, max_total_tokens=?)` — raises `RequestBudgetExceeded` if input alone or input plus `max_output_tokens` exceeds the caps. Pick caps consistent with your worst-case estimate above.
2. `RollingTokenBudget(tokens_per_window=?, window_seconds=?)` — per-user rolling-window budget. Pick a per-minute cap and a per-day cap consistent with the "p99 user makes ~200 requests / day" number. Enforce both.

Wire the two into a single function:

```python
def classify_ticket(user_id: str, ticket: str) -> dict:
    # 1. Count input tokens.
    # 2. Enforce per-request budget.
    # 3. Enforce per-user budget (reserve worst case: input + max_tokens).
    # 4. Call the SDK.
    # 5. Reconcile per-user budget with actual output tokens from response.usage.
    # 6. Return the parsed JSON.
```

## Requirements — the tests

Write a test file (`pytest` or unittest) that exercises the enforcement without hitting the provider (mock the SDK client):

- **Normal request** passes both caps, calls the SDK, updates the per-user counter with actual usage.
- **Oversized input** (> `max_input_tokens`) raises `RequestBudgetExceeded` and does not call the SDK.
- **A single user making N requests where N * worst-case exceeds the per-day cap** starts raising `UserBudgetExceeded` at the right request. Assert the exact request at which the cap trips.
- **Different users** have independent budgets — one user's cap does not affect another's.

## Starter guidance

- Provider pricing pages: <https://www.anthropic.com/pricing>, <https://openai.com/api/pricing/>.
- `tiktoken` for OpenAI token counts: <https://github.com/openai/tiktoken>.
- Anthropic token-counting endpoint: <https://docs.anthropic.com/en/docs/build-with-claude/token-counting>.
- Mod-101 chapter 3 for the mechanics of counting tokens and reading the `usage` block.
- Do **not** import a rate-limit library for the per-user cap (`slowapi`, `limits`, etc.) — build the rolling window yourself. This exercise is about understanding the shape, not about production plumbing.

## Acceptance criteria

- `estimate.py` prints the four dollar numbers (p50 per call, worst per call, daily worst, monthly worst) and cites the pricing page and date at the top of the file.
- Per-request enforcement refuses any request whose input alone or input + `max_tokens` exceeds the cap. Error message includes the cap and the exceeded amount.
- Per-user enforcement refuses at the correct request when a user exceeds their per-minute or per-day cap.
- Per-user counter reserves worst-case output at enforcement time and reconciles down to actual output after the response — verified by a test that makes one request with a cap tight enough to matter.
- All four tests pass without hitting the real provider (SDK is mocked).
- No `time.sleep` in the enforcement code. Use `time.monotonic()` for the rolling window.

## Stretch goals

- Add a **role-aware budget**: signed-in paying users get a larger cap than free-tier; background jobs on service credentials get a much larger cap. Encode roles as a small enum; pick caps for each.
- Replace the in-memory `RollingTokenBudget` with a **Redis-backed version** using `ZADD` / `ZRANGEBYSCORE` on a sorted set. Verify it works with two processes running concurrently against the same Redis.
- Add **cost logging**: on every request, log `{user_id, feature, model, input_tokens, output_tokens, cached_input_tokens, cost}` to a JSONL file. Write a small `summarise.py` that reads the file and prints cost per user and cost per feature over the last N minutes.
- Extend `estimate.py` to accept an assumed cache-hit ratio (0–1). Recompute the numbers assuming that fraction of input tokens is billed at the cached-input price. Compare to the no-caching baseline. (This anticipates exercise 03.)
