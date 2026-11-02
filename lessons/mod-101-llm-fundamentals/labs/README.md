# Labs for mod-101-llm-fundamentals (LLM Fundamentals for Application Developers)

Guided, step-by-step hands-on sessions. Each lab lists duration, prerequisites, numbered steps with commands or code, expected output, and checkpoints. Unlike the open-ended [exercises](../exercises/), labs are tightly scripted — you read a line, run a command, confirm the result, and move on.

Work through them in order; later labs assume you have run the earlier ones.

- [lab-01 — Dissect an LLM HTTP Call](lab-01-dissect-llm-http-call.md) *(45–60 min)* — issue a chat request without the SDK, inspect headers and `usage`, then read raw SSE events on a streaming call. Grounds chapters 1 and 6 at the wire level.
- [lab-02 — Side-by-Side Model-Tier Comparison](lab-02-compare-model-tiers.md) *(60–75 min)* — pin a workhorse and a cheap model ID, run ten fixed prompts through both, score by hand, and project cost from measured `usage`. Walks chapter 5's practical sequence end-to-end.
- [lab-03 — See Prompt Caching in the `usage` Field](lab-03-prompt-caching-in-action.md) *(30–45 min)* — send a long, byte-identical prefix three times and watch cache-write and cache-read tokens appear in the `usage` field. Grounds chapter 7's highest-payoff cost lever.
