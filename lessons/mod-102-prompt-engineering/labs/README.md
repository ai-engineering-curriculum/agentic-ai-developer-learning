# Labs for mod-102-prompt-engineering (Prompt Engineering & Structured Output)

Guided, step-by-step hands-on sessions. Each lab lists duration, prerequisites, numbered steps with commands or code, expected output, and checkpoints. Unlike the open-ended [exercises](../exercises/), labs are tightly scripted — you read a line, run a command, confirm the result, and move on.

Each lab isolates a *single* prompt-engineering move so you can see what that move alone earns before the exercises bundle several together. They can be worked in any order, though the technique each one isolates is referenced forward from the matching chapter.

- [lab-01 — Prefill Forces JSON Without a JSON Mode](lab-01-prefill-forces-json.md) *(30–45 min)* — send the same prompt-only JSON request twice to Anthropic, once with and once without an assistant prefill of `{`, and count parse-on-first-try wins. Isolates chapter 5's cheapest escalation step from Strategy 1 (prompt-only).
- [lab-02 — Scope Constraint as a Hallucination Guard](lab-02-scope-constrained-prompt.md) *(45–60 min)* — ask five questions about a self-contained invented document under a vague prompt and a scope-constrained prompt, and tabulate correct / hallucinated / unknown per cell. Then apply chapter 4's "extract, then answer" decomposition to one failure. Grounds chapter 2's claim that scope is "the single most reliable hallucination guard for retrieval prompts."
- [lab-03 — Calibrate an LLM-as-Judge Against Human Scores](lab-03-calibrate-llm-judge.md) *(60–75 min)* — hand-score ten summaries on three axes, run an LLM judge with a vague rubric (`rubric_v1`), tighten the rubric with anchored criteria (`rubric_v2`), and measure agreement with your scores each time. Grounds chapter 7's "LLM-as-judge: when and how to use it."
