# Quiz 01 — Your First Agent: The Reason-Act Loop

**Covers:** chapters 1–6 of `mod-105-first-agent`.
**Format:** 12 multiple-choice questions. Pick the single best answer for each. Answer key with per-answer explanations is at the bottom.

Treat this as a self-check: open the chapters when you want to look something up, but try to answer first.

---

## 1. Chapter 1 contrasts a mod-103 fixed tool loop, a mod-104 RAG query, and the new agent shape across several axes. Which of the following is the single property the chapter names as the pivotal shift when you move to an agent?

- A. The provider SDK gains a new agent-specific endpoint distinct from `messages.create`.
- B. The model, rather than your application, decides what the next step is — including whether to step at all.
- C. Tool definitions become multi-modal and can accept images directly.
- D. Retrieval is now performed on every turn by default, as in RAG.

## 2. Chapter 1 says step caps "stop being polite — they are load-bearing." The chapter's stated reason is:

- A. Modern providers enforce a 3-step maximum on multi-step tool use, so the application cap only mirrors the server's.
- B. Longer trajectories always drift, so the cap is required for answer quality.
- C. Costs and latency scale roughly linearly with steps, and a misbehaving prompt can turn a $0.01 request into a $0.50 one with nothing else braking it.
- D. Native tool use requires `max_steps` to be set in the API payload.

## 3. Chapter 2 maps the three ReAct "slots" onto the fields of a native tool-use API. Which mapping matches the chapter?

- A. Thought → `tool_use`; Action → assistant `text`; Observation → `tool_result`.
- B. Thought → assistant `text` content; Action → `tool_use` block in the assistant turn; Observation → `tool_result` block in the next user turn.
- C. Thought → system prompt; Action → user turn; Observation → assistant `text`.
- D. Thought, Action, and Observation are all emitted by the model as parsed text — exactly as in the 2022 paper — with the SDK only serving as transport.

## 4. Chapter 2 lists cases where ReAct is the wrong shape. Which of the following is the chapter's example of a task that should NOT be rebuilt as a ReAct agent?

- A. Looking up a customer's current order status in a support thread.
- B. Searching a knowledge base, reading the result, and re-searching with a sharper query.
- C. For each row in a CSV, call the model to compute a known transform — a fixed sequential plan over a list.
- D. Routing one message through two or three unrelated tools in a single turn.

## 5. Chapter 3 lists termination conditions for the ReAct loop in order. Which of these is NOT one of the four explicit termination conditions the chapter says you must handle?

- A. `stop_reason == "end_turn"`.
- B. `stop_reason == "max_tokens"`.
- C. `step >= max_steps`.
- D. The assistant turn contains no leading `text` content block.

## 6. Chapter 3 says a tool that raises should become a structured `tool_result` with `is_error: True` rather than propagating an exception. The chapter calls this "the single most important behavior for a usable agent." Why?

- A. Providers charge double for raised exceptions compared to structured tool results.
- B. The model never sees a raised exception, so it cannot adapt; converted to an observation, the error lets the model apologize, retry with corrected arguments, or give up cleanly.
- C. `is_error: True` makes the observation count against the step cap, which prevents runaway trajectories.
- D. The Anthropic API rejects any request whose content blocks are missing the `is_error` flag.

## 7. Chapter 4 explains that a well-written `search_kb` description includes a line like *"Do NOT use it for chit-chat, math, or actions like placing an order."* The chapter calls this line "load-bearing." Why?

- A. Without it the API refuses the tool definition on schema validation.
- B. Without the "when NOT to use" guidance the model dutifully calls `search_kb` for messages like *"hi"* — burning an embedding call and an index lookup on nothing.
- C. The regex in the description prevents prompt injection via malformed query strings.
- D. The clause changes the tool's `input_schema` at runtime so the parameter description is enforced.

## 8. Chapter 4 describes an "iterative retrieval" pattern — the agent sees a weak first retrieval and issues a sharper second query. The chapter names a specific, cheap change to the tool's *output* that enables this behavior. What is it?

- A. Return every chunk's full text with no per-chunk truncation.
- B. Include the similarity *score* in each returned chunk so the model can tell a strong hit (0.81) from a weak one (0.42) and decide to re-search.
- C. Always return exactly 10 chunks regardless of the `k` the model asked for.
- D. Add the embedding model name to each returned chunk.

## 9. Chapter 5 defines what "memory" actually is for an LLM. Which statement matches the chapter?

- A. Modern Claude and GPT models maintain persistent per-user state server-side; the client only needs to opt in via a header.
- B. The model itself has no state between API calls; "the agent's memory" is whatever you put back into the `messages` list on the next call, optionally supplemented by application state like a key:value scratchpad.
- C. Memory is implemented by increasing `max_tokens`, which widens the model's attention window to prior turns.
- D. Memory is a property of the system prompt alone; multi-turn transcripts are optional and only improve style.

## 10. Chapter 5 recommends a small key:value scratchpad with `remember`/`recall` tools **and** rendering the scratchpad into the system prompt on every turn. Why render it rather than relying on `recall` calls?

- A. `recall` cannot run in parallel with other tools in a single assistant turn.
- B. Rendering the scratchpad into the system prompt costs essentially nothing per turn, lets the model use known facts without burning a `recall` round trip, and keeps those facts alive across transcript trimming or summarization.
- C. Rendering is required by the Anthropic API whenever tool use is enabled.
- D. `recall` returns stale values unless the scratchpad is also written to a vector store on every change.

## 11. Chapter 6 lists five failure modes you will see in a single-agent system, in roughly the order you will encounter them. Which of the following is NOT on that list?

- A. Loops — the agent calls the same tool with the same arguments repeatedly.
- B. Drift — the agent wanders onto a tangentially interesting topic a retrieval surfaced.
- C. Planning depth — tasks whose right answer three steps ahead constrains the step you make now.
- D. Token leakage — the model emits training-set content verbatim from the transcript.

## 12. Chapter 6 offers a pragmatic checklist for *"is this still a single-agent problem?"* Which answer pattern indicates you should keep shipping as a single agent rather than graduate to a planner/executor or multi-agent architecture?

- A. Median trajectory is 15+ steps, you advertise 12+ tools, and failures routinely require human intervention.
- B. Tool count ≤ ~7, median trajectory ≤ ~10 steps, one role, recoverable failures, stakes low enough that "the model said so" is acceptable verification.
- C. The task has a verifiable success criterion like "the test passes" that benefits from a Reflexion-style critique loop.
- D. Every action is high-stakes and needs a human approval checkpoint before it fires.

---

## Answer key

1. **B.** Chapter 1's table ("Tool loop vs. agent — what actually changes") spells it out on the "Who picks the next step" row: the model picks, *including whether to step at all*. The text immediately after (*"The model is now the planner"*) names this as the pivotal shift; there is no new SDK endpoint (A), tools haven't changed shape (C), and the agent's whole point is that it does *not* always retrieve (D).

2. **C.** Chapter 1 ("Tool loop vs. agent — what actually changes"): *"Costs and latency add up linearly with steps, and a misbehaving prompt can turn a $0.01 request into a $0.50 one. Step caps stop being polite — they are load-bearing."* No server-side cap exists (A), the chapter does not claim longer trajectories always drift (B), and `max_steps` is not an API payload field (D).

3. **B.** Chapter 2's "The modern (native tool-use) form" table gives the mapping exactly: Thought → assistant `text`, Action → `tool_use` block, Observation → next user turn's `tool_result`. Option A swaps the first two; option C puts everything in the wrong roles; option D describes the *original prompted* form that chapter 2 says you will not use.

4. **C.** Chapter 2 ("When ReAct shines, and when it doesn't"): *"Tasks with a known sequential plan. 'For each row in this CSV, look up X and write Y.' A for-loop over rows with two model calls each is more reliable and much cheaper than asking the model to drive the iteration."* The other options are the chapter's examples of where ReAct *does* shine.

5. **D.** Chapter 3 ("Termination conditions, in order") lists exactly four: `end_turn`, `max_tokens`, step cap, and `stop_sequence`. "No leading text content" is a style observation chapter 2 made (the Thought is optional), not a termination condition.

6. **B.** Chapter 3 ("Error responses as observations"): *"The model never sees the failure; it doesn't learn from it; rerunning the same prompt fails the same way…"* Converting to a structured observation lets the model apologize, retry, or give up cleanly. Pricing (A), step-cap bookkeeping (C), and provider rejection (D) are not claims the chapter makes.

7. **B.** Chapter 4 ("The tool definition"): *"The 'do NOT use it for chit-chat' line is load-bearing — without it the model will dutifully search for 'hi.'"* Description text is not enforced by the API (A, D) and does not change input validation (C).

8. **B.** Chapter 4 ("Iterative retrieval — the model gets a second swing"): *"The model decides to re-search based on the observation it got. If you don't surface the score, it can't tell the difference between a strong and weak hit, and you lose the trigger. Including the score is the smallest change that enables this behavior."* Full text (A) blows the token budget (chapter 4's "Common mistakes" warns against it); `k` and `embed_model` (C, D) are not what unlocks re-search.

9. **B.** Chapter 5 ("What 'memory' actually is for an LLM"): *"The model itself has no memory between API calls. Every messages.create(...) is independent. What we call 'the agent's memory' is whatever you put back into the messages list on the next call."* Chapter 5 and its "Anti-patterns" section explicitly reject A, C, and D.

10. **B.** Chapter 5 ("Working memory: a key:value scratchpad"): *"You inject the current scratchpad into the system prompt at the start of every call so the model knows what's in there without having to call recall every turn… at near-zero cost."* The common-mistakes list calls out "a scratchpad that nobody reads" — the chapter's whole point for rendering is cost, latency, and durability across transcript pressure, not API requirements (A, C) or vector freshness (D).

11. **D.** Chapter 6 ("Where the single agent breaks") enumerates exactly five: loops, drift, planning depth, tool-surface bloat, and verification/trust. "Token leakage" is not on the list and is not a failure mode the chapter raises; the five are about *trajectory behavior*, not training-data memorization.

12. **B.** Chapter 6 ("When to graduate") gives the checklist verbatim: steps under ~10, tool count under ~7, one role, recoverable failures, and stakes low enough that "the model said so" is acceptable verification — a "yes" to all of these means single-agent. A, C, and D are the chapter's examples of signals that the next architecture (planner/worker, critique loops, human-in-the-loop) earns its complexity.
