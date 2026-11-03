# Resources for mod-108 — Model Selection, Cost, and Prompt Caching

Prefer primary sources — provider docs, pricing pages, and the SDK / library maintainers you actually import. Prices, model IDs, and cache mechanisms change; the fix when a link goes stale is to find the current index page, not to trust a blog post from a search result.

## Provider pricing and model catalogues

Cite pricing pages by URL **and** by fetch date whenever you record a number.

### Anthropic

- **Pricing** — per-million-token input / output prices, cached-input prices, batch prices. <https://www.anthropic.com/pricing>
- **Model overview** — current model IDs, context windows, tool-use support. <https://docs.anthropic.com/en/docs/about-claude/models>
- **Extended thinking** — for reasoning-token accounting when you enable it. <https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking>

### OpenAI

- **API pricing** — per-million-token input / output / cached / batch prices. <https://openai.com/api/pricing/>
- **Model catalogue** — current model IDs and capabilities. <https://platform.openai.com/docs/models>

## Prompt caching

- **Anthropic prompt caching** — `cache_control` blocks, TTLs, cache read / write pricing model. <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>
- **OpenAI prompt caching** — automatic caching above a minimum prompt size; `usage.prompt_tokens_details.cached_tokens`. <https://platform.openai.com/docs/guides/prompt-caching>

## Token counting (input cost planning)

- **`tiktoken`** — OpenAI's client-side tokenizer. Free to run on your laptop. <https://github.com/openai/tiktoken>
- **Anthropic token-counting endpoint** — authoritative server-side count for a given `messages` / `system` / `tools` payload. <https://docs.anthropic.com/en/docs/build-with-claude/token-counting>

## Batch APIs (offline / non-user-facing workloads)

- **OpenAI Batch API** — turnaround SLA, discount, input/output JSONL format. <https://platform.openai.com/docs/guides/batch>
- **Anthropic Message Batches** — turnaround SLA, discount, request/result shape. <https://docs.anthropic.com/en/docs/build-with-claude/batch-processing>

## Errors, rate limits, and reliability

- **Anthropic errors** — the terminal-vs-transient taxonomy (400/401/403/404 terminal; 429/500/529 transient). <https://docs.anthropic.com/en/api/errors>
- **OpenAI error codes** — same taxonomy, different wording. <https://platform.openai.com/docs/guides/error-codes>
- **Anthropic rate limits** — headers, tier structure. <https://docs.anthropic.com/en/api/rate-limits>
- **OpenAI rate limits** — RPM/TPM, headers, tier progression. <https://platform.openai.com/docs/guides/rate-limits>
- **AWS Builders' Library — timeouts, retries, backoff with jitter** — the canonical write-up on why full jitter is the right backoff shape. <https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/>
- **`tenacity`** — the mature Python retry library; worth reading the source once. <https://tenacity.readthedocs.io/>
- **`pybreaker`** — small Python circuit-breaker implementation, useful when retries alone aren't enough. <https://github.com/danielfm/pybreaker>
- **CircuitBreaker (Martin Fowler)** — the language-agnostic pattern description. <https://martinfowler.com/bliki/CircuitBreaker.html>

## Provider status pages

Bookmark both. Wire the RSS feeds into your on-call channel.

- **Anthropic status** — <https://status.anthropic.com/>
- **OpenAI status** — <https://status.openai.com/>

## SDKs (for `base_url` overrides in exercise 05, retry configuration, batch clients)

- **`anthropic-sdk-python`** — repo and README. <https://github.com/anthropics/anthropic-sdk-python>
- **`openai-python`** — repo and README. <https://github.com/openai/openai-python>

## Standards worth knowing

- **RFC 9110 — HTTP semantics** — the normative reference for 429, 5xx, `Retry-After`, and every other status code chapters 2 and 5 rely on. <https://www.rfc-editor.org/rfc/rfc9110>
- **`Retry-After` header (RFC 9110 §10.2.3)** — the header your retry policy must respect when present. <https://www.rfc-editor.org/rfc/rfc9110#field.retry-after>

## Evaluation background (paired with mod-109)

- **JSON Schema** — the specification behind the structured outputs the A/B benchmark scores against. <https://json-schema.org/>
- **`statistics.quantiles` in the Python standard library** — the low-friction way to compute p50/p95 on the small samples this module's exercises produce. <https://docs.python.org/3/library/statistics.html#statistics.quantiles>
- **`numpy.percentile`** — the fast option once samples get large. <https://numpy.org/doc/stable/reference/generated/numpy.percentile.html>

## Related modules and tracks

- **`mod-101-llm-fundamentals` chapter 3** (this track, prerequisite) — tokens, `tiktoken`, and the token-counting endpoint. All of chapter 2 in this module leans on it.
- **`mod-107-streaming-async-and-orchestration` chapters 4 and 5** (this track, prerequisite) — retry-with-jitter and honest latency measurement. Both are reused in exercises 04 and 05.
- **`mod-104-retrieval-basics`** (this track) — retrieval makes prompts longer, so every lever in this module (small-model routing, caching, budget caps) becomes more valuable when RAG is turned on.
- **`mod-109-minimal-eval-and-regression-checks`** (this track) — the A/B benchmark shape from exercise 02 evolves into a real eval harness there.
- **`mod-106-llm-app-deployment`** (this track) — the operational contract (SLOs, budgets, fallback path) from chapter 5 becomes part of the ship checklist.
- **`mod-105-first-agent`** (this track) and **`agentic-ai-engineer-learning`** (next rung) — agent and multi-agent systems inherit every cost lever here and multiply the impact; cascade routing shows up there as router-of-routers.

## What is deliberately not on this list

- Vendor-neutral "LLM routing" services and gateways. Some are fine to use later; learning to route yourself first, which this module does, makes their failures debuggable.
- Third-party leaderboards and benchmark charts. They measure model quality *in general* and do not tell you whether the small model is good enough on *your* narrow task. Exercise 02 replaces them for your feature.
- Screenshotted token-per-second numbers or specific price examples. Prices and speeds change; a link to the current pricing / model page is more useful than yesterday's number.
- Blog posts titled "How we cut our LLM bill by 90%". They are almost always either specific to a stack you do not run or missing the enforcement / measurement discipline this module teaches. Read the provider docs above instead.
