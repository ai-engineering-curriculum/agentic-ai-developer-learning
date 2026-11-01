# Job Requirements — Agentic AI Developer

**Role level:** 20 (entry rung, Agentic track)
**Track:** `agentic-ai-developer-learning`
**Research window:** 2026-08-03 → 2026-11-01 (last 90 days)
**Today:** 2026-11-01
**Previous cycle:** 2026-10 (see `git log` for the superseded file)

This file maps verbatim requirements from current Agentic AI Developer / AI Engineer / LLM Application Developer / Generative AI Developer / Applied AI Developer / AI Application Engineer job postings to the existing curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the strictly-additive proposal lives in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Summary

- Fresh postings sampled this cycle: **25** entry-to-mid, observed live on 2026-11-01, plus **5** senior/staff postings retained as reinforcing evidence only.
- **Proposed delta this cycle: 0 modules, 0 exercises, 0 projects.** One watch-list theme crossed the ≥30% threshold — **T18 Model Context Protocol (MCP), 15% → 32%, 8 distinct postings** — but it fails the third continuity-bias precondition: existing content *can* be incrementally extended to cover the level-20 share of the requirement without a new exercise, and MCP server implementation depth is explicitly owned by `agentic-ai-engineer mod-202-frameworks/exercise-04-mcp-tool-server` per the ownership rule.
- The 2026-10 cycle's proposed exercise (`mod-106-llm-app-deployment` / `exercise-04` Prompt-injection awareness) remains unmerged; adding further additions in Nov would compound review backlog.
- Sub-threshold / threshold-grazing signals tracked for next cycle: T18 MCP awareness (reconsider in Dec if the client-side share grows), T15 Observability (28%, oscillating), T19 Multi-agent orchestration depth (32%, structurally upstream), T21 AI-assisted coding workflow (24%, climbing — not a curriculum skill), T13 Docker (28%, Docker owned here / Kubernetes upstream).

## Methodology

- Sources: `job-boards.greenhouse.io` (direct WebFetch, primary), `jobs.lever.co` (direct WebFetch), `builtin.com` (direct WebFetch), WebSearch across title variants scoped to 2026-Q3/Q4. Ashby (`jobs.ashbyhq.com`) was excluded because it renders client-side and WebFetch gets a title-only shell.
- Per-posting capture: employer, verbatim title, URL, `date_observed`, `date_posted` (marked `est:2026-Q3/Q4` when the fetched body did not expose an exact date — all 25 were observed live on 2026-11-01), location, seniority, and 5–10 verbatim technical requirement quotes.
- Frequency = distinct in-scope postings citing the theme ÷ 25 in-scope postings.
- In-scope filter: entry-to-mid altitude. Postings with explicit **Senior / Staff / Principal / Lead / Architect** modifiers are excluded from the denominator (they inherit upward to `agentic-ai-engineer` / `senior-agentic-ai-engineer` / `applied-ai-engineer` / `staff-ml-engineer`). Five senior postings are retained as **reinforcing** evidence only; they are NOT counted in any frequency.
- Overlap with the 2026-10 denominator: zero. All 25 postings this cycle are distinct employers from the 20 captured in October. This gives maximally independent evidence that the structural shape of entry-to-mid Agentic-AI-Developer hiring has not materially shifted in 30 days.
- Source caveats: no captured posting exposed a visible `date_posted` on the fetched body (all flagged `est:2026-Q3/Q4`). All listings were observed live on 2026-11-01. Nine Greenhouse employer-landing pages and one Lever URL returned a listing index instead of the per-job body and were dropped (see `.aicg/job-requirements.json` → `sample.exclusion_summary`).

## Requirement themes → curriculum ownership

Every requirement observed, its in-scope frequency, the role that owns it per the level hierarchy, and the curriculum coverage path. Frequencies ≥30% are **bolded** — those are the themes the continuity-bias rule treats as load-bearing. The `Δ vs 2026-10` column shows the point-change from the previous cycle (same methodology, different denominator — Oct used N=20, Nov uses N=25).

| # | Theme | Freq | Δ vs 2026-10 | Owner role | Coverage |
|---|---|---|---|---|---|
| T01 | Python + software-engineering habits (Git, testing, packaging, HTTP/JSON) | **90%** | −10pp | `agentic-ai-developer` (prerequisite tier) | [`PREREQUISITES.md`](PREREQUISITES.md) — assumed at entry; not re-taught |
| T02 | LLM APIs (OpenAI, Anthropic, Google/Gemini, Azure OpenAI, Bedrock, Mistral) — tokens, context, streaming, cost, errors, retries | **80%** | ±0pp | `agentic-ai-developer` | [`mod-101-llm-fundamentals`](lessons/mod-101-llm-fundamentals) — 8 chapters + 3 exercises |
| T03 | Prompt engineering (system prompts, few-shot, chain-of-thought, context management, iterating with evals) | **56%** | −9pp | `agentic-ai-developer` | [`mod-102-prompt-engineering`](lessons/mod-102-prompt-engineering) — 7 chapters + 3 exercises |
| T04 | Tool / function calling (define tools, call/result loop, when-to-call, malformed args) | **56%** | +1pp | `agentic-ai-developer` | [`mod-103-tool-and-function-calling`](lessons/mod-103-tool-and-function-calling) — 6 chapters + 3 exercises |
| T05 | Structured output (JSON mode, schema-constrained, Pydantic / JSON Schema validation) | 16% | −19pp | `agentic-ai-developer` | [`mod-102-prompt-engineering/exercises/exercise-02-structured-json-output.md`](lessons/mod-102-prompt-engineering/exercises/exercise-02-structured-json-output.md), [`exercise-03-output-validation.md`](lessons/mod-102-prompt-engineering/exercises/exercise-03-output-validation.md); Pydantic-validated requests in [`mod-106-llm-app-deployment/02-fastapi-and-input-validation.md`](lessons/mod-106-llm-app-deployment/02-fastapi-and-input-validation.md) |
| T06 | Retrieval-Augmented Generation (RAG) — embeddings, similarity search, chunking, indexing, grounded generation | **60%** | −5pp | `agentic-ai-developer` | [`mod-104-retrieval-basics`](lessons/mod-104-retrieval-basics) — 7 chapters + 3 exercises |
| T07 | Vector databases (Pinecone, Weaviate, Chroma, pgvector, FAISS, Milvus, Qdrant, Bedrock KB) | **52%** | +7pp | `agentic-ai-developer` (basics) → [`rag-engineer`](https://github.com/ai-engineering-curriculum/rag-engineer-learning) (depth) | [`mod-104-retrieval-basics/03-similarity-search-and-indexes.md`](lessons/mod-104-retrieval-basics/03-similarity-search-and-indexes.md), [`exercise-01-embeddings-similarity.md`](lessons/mod-104-retrieval-basics/exercises/exercise-01-embeddings-similarity.md); rerankers / hybrid / GraphRAG deferred to `rag-engineer` |
| T08 | First agent (ReAct / reason-act loop, tool loop, retrieval-as-tool, conversation memory) | **76%** | +16pp | `agentic-ai-developer` | [`mod-105-first-agent`](lessons/mod-105-first-agent) — 6 chapters + 3 exercises |
| T09 | Agent frameworks / SDKs — LangChain, LangGraph, LlamaIndex, CrewAI, PydanticAI, Google ADK, Semantic Kernel, smolagents, strands-agents (basic use) | **56%** | +11pp | `agentic-ai-developer` (awareness) → [`agentic-ai-engineer`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) (bake-off depth) | [`mod-105-first-agent/06-limits-of-a-single-agent.md`](lessons/mod-105-first-agent/06-limits-of-a-single-agent.md) handoff; framework/SDK bake-off owned by `agentic-ai-engineer` `mod-202-frameworks` |
| T10 | Deploy an LLM app behind an API (FastAPI, REST, containerization, secrets, structured logging, cost/usage guardrail) | **40%** | −10pp | `agentic-ai-developer` | [`mod-106-llm-app-deployment`](lessons/mod-106-llm-app-deployment) — 7 chapters + 3 exercises |
| T11 | Basic evaluation of LLM outputs (rubrics, offline benchmarks, retrieval quality, LLM-as-judge, golden sets, regression) | **60%** | ±0pp | `agentic-ai-developer` (basics) → [`ai-eval-engineer`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) (program depth) | [`mod-102-prompt-engineering/07-iterating-with-evals.md`](lessons/mod-102-prompt-engineering/07-iterating-with-evals.md), [`mod-104-retrieval-basics/07-evaluating-retrieval-quality.md`](lessons/mod-104-retrieval-basics/07-evaluating-retrieval-quality.md); judge-tier eval programs deferred to `ai-eval-engineer` |
| T12 | Cloud platform basics (AWS / Azure / GCP) for LLM app hosting | **52%** | −3pp | `agentic-ai-developer` (basics) → ai-infra tracks (depth) | [`mod-106-llm-app-deployment`](lessons/mod-106-llm-app-deployment) covers Docker + local run; deep cloud is out-of-scope link-out |
| T13 | Docker / containerization (and Kubernetes mention) | 28% | +8pp | `agentic-ai-developer` (Docker) → ai-infra tracks (Kubernetes depth) | [`mod-106-llm-app-deployment/06-containerizing-with-docker.md`](lessons/mod-106-llm-app-deployment/06-containerizing-with-docker.md); Kubernetes deferred to ai-infra tracks |
| T14 | Guardrails / prompt-injection defense / input+output filtering / hallucination mitigation | 24% | −11pp | `agentic-ai-developer` (awareness) → [`agentic-ai-engineer`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) `mod-206-guardrails-implementation` → `agentic-safety-engineer` | Awareness: [`mod-103/06-tool-errors-and-malformed-arguments.md`](lessons/mod-103-tool-and-function-calling/06-tool-errors-and-malformed-arguments.md) + [`mod-106/05-cost-and-usage-guardrails.md`](lessons/mod-106-llm-app-deployment/05-cost-and-usage-guardrails.md). The 2026-10 cycle's proposed `mod-106/exercise-04-prompt-injection-and-input-output-filtering` remains the designated hands-on home when it merges. One standout posting this cycle (Turning Point Global Solutions) explicitly cites OWASP Top 10 for LLM Applications and prompt-injection testing — reinforcing the Oct proposal's framing. |
| T15 | Observability / tracing (Langfuse, OpenTelemetry Gen-AI conventions, Datadog, LangSmith / Langtrace / AgentOps) for LLM apps | 28% | −2pp | `ai-eval-engineer` / `agentic-ai-engineer` `mod-205-evaluation-observability` | Awareness only via [`mod-106-llm-app-deployment/04-logging-llm-requests.md`](lessons/mod-106-llm-app-deployment/04-logging-llm-requests.md); depth link-out |
| T16 | Cost / latency depth (prompt caching, token budgets, model routing, semantic caching, cost attribution) | 12% | +2pp | `agentic-ai-engineer` `mod-207-productionizing-agents` (depth) | Awareness owned here via [`mod-101/07-cost-and-latency.md`](lessons/mod-101-llm-fundamentals/07-cost-and-latency.md) and [`mod-106/05-cost-and-usage-guardrails.md`](lessons/mod-106-llm-app-deployment/05-cost-and-usage-guardrails.md); depth link-out |
| T17 | Streaming responses (SSE, WebSockets, chunked transfer, cancellation) | 0% | −5pp | `agentic-ai-developer` | [`mod-101-llm-fundamentals/06-streaming-responses.md`](lessons/mod-101-llm-fundamentals/06-streaming-responses.md) — already covered |
| T18 | **Model Context Protocol (MCP)** — server implementation and tool exposure | **32%** | **+17pp (CROSSED THRESHOLD)** | `agentic-ai-engineer` `mod-202-frameworks/exercise-04-mcp-tool-server` (depth) — level-20 share is "familiarity" only | Server-build depth stays upstream. Entry-tier awareness is appropriately covered by extending existing handoff chapters: [`mod-103/06-tool-errors-and-malformed-arguments.md`](lessons/mod-103-tool-and-function-calling/06-tool-errors-and-malformed-arguments.md) and [`mod-105/06-limits-of-a-single-agent.md`](lessons/mod-105-first-agent/06-limits-of-a-single-agent.md) via editorial pointers rather than a new exercise. See the "Delta recommendation" section below for the rationale against promoting a dedicated exercise this cycle. |
| T19 | Multi-agent orchestration / planners / handoffs / sub-agents / A2A protocols | **32%** | −3pp | `agentic-ai-engineer` `mod-204-multi-agent-implementation` | Explicit handoff at [`mod-105-first-agent/06-limits-of-a-single-agent.md`](lessons/mod-105-first-agent/06-limits-of-a-single-agent.md); depth owned upstream. One posting (SimplifyNext) explicitly uses the "A2A" (Agent-to-Agent) phrasing — handoff coverage already acknowledges this transition. |
| T20 | Fine-tuning depth (SFT, LoRA, RLHF, DPO) | 16% | +6pp | [`fine-tuning-engineer`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning) | Out of scope. Link-out only. |
| T21 | AI-assisted developer tools (Cursor, Claude Code, Codex, Gemini CLI, GitHub Copilot) as workflow | 24% | +4pp | workflow / not-a-curriculum-topic | Not a curriculum-owned skill — a workflow signal. Still ubiquitous in reinforcing senior/staff postings. Revisit if freq climbs above 30% in-scope. |
| T22 | Human-in-the-loop / approval checkpoints | 8% | −7pp | `agentic-ai-engineer` `mod-206-guardrails-implementation` (depth) | Below threshold. Awareness naturally sits inside the pending `mod-106 exercise-04` (approval loop as one filter pattern). |

## Posting evidence

Every posting below was observed live on **2026-11-01**. Denominator postings are entry-to-mid altitude; the five senior postings at the bottom are retained as **reinforcing** evidence only (not counted toward any frequency).

### In-scope postings (denominator = 25)

| # | Employer | Title | URL | Location | Seniority |
|---|---|---|---|---|---|
| p101 | Materiom | AI Engineer | https://job-boards.greenhouse.io/materiom/jobs/5150829007 | London (hybrid) | mid |
| p102 | Lantern Studios | Generative AI Developer — Microsoft Azure AI Stack | https://job-boards.greenhouse.io/lanternstudiosinc/jobs/4985296008 | Calgary / Edmonton / Toronto | mid |
| p103 | ShyftLabs | Associate AI Engineer | https://jobs.lever.co/shyftlabs/1a665cf5-1f3f-4610-be00-b46ffa13a675 | Toronto (hybrid) | entry |
| p104 | Zone 5 Technologies | Machine Learning Ops Engineer | https://job-boards.greenhouse.io/zone5technologies/jobs/5381203008 | US (onsite) | mid |
| p105 | Honeycomb | Software Engineer I — Agent Observability | https://job-boards.greenhouse.io/honeycomb/jobs/5452584008 | Remote US | entry-mid |
| p106 | Scale AI | Software Engineer — AI Enablement | https://job-boards.greenhouse.io/scaleai/jobs/4722512005 | San Francisco (hybrid) | mid |
| p107 | Cargomatic | AI Engineer — Agentic Automation | https://job-boards.greenhouse.io/cargomatic/jobs/5207803008 | Remote US | mid |
| p108 | Turning Point Global Solutions | AI-Assisted Full-Stack Engineer | https://jobs.lever.co/tpgsi/1cb2b7f2-506b-4cb2-8d59-bc6e4771363f | Washington DC Metro (hybrid) | mid |
| p109 | DeepLearning.AI | AI Engineer | https://jobs.lever.co/AIFund/273af06c-9114-4b9c-83c9-a3627f4b875f | Mountain View, CA | entry-mid |
| p110 | Artera | Associate AI Builder | https://jobs.lever.co/artera-2/eae88c70-fbf5-4525-890c-d3f9377418b0 | Seattle / Santa Barbara / Kansas City | entry |
| p111 | AHEAD | Associate AI Engineer (Associate Technical Consultant) | https://jobs.lever.co/thinkahead/1ebcedcb-82ba-4d19-ac88-aa73a812dd81 | Chicago (hybrid) | entry |
| p112 | Talentus Global | Generative AI Engineer (LATAM) | https://jobs.lever.co/TalentusGlobal/9760b72b-c022-4be9-9858-10ea64edef89 | LATAM remote | mid |
| p201 | Kpler | Engineer II (AI Enablement) | https://jobs.lever.co/kpler/e914f4a8-2a6a-41d1-9c0a-51fdd23a5e75 | Singapore / Paris / Athens | mid |
| p202 | Binance | Binance Accelerator Program — Applied AI Agent Developer | https://jobs.lever.co/binance/0b78f793-4313-4a2a-a4cb-8cbeb2bc7d7b | Asia (remote) | entry |
| p203 | Xsolla | Junior Developer (AI Engineering) | https://jobs.lever.co/xsolla/d370a7d6-0923-441a-8af3-f5526d3d061c | Vladivostok | entry |
| p204 | Engelhart | Applied AI Engineer | https://job-boards.greenhouse.io/engelhart/jobs/8305681 | Berlin, DE | entry-mid |
| p205 | CodeRoad | Agentic AI Engineer | https://job-boards.greenhouse.io/coderoad/jobs/4406476009 | LatAm (100% remote) | mid |
| p206 | Seso | Software Engineer, AI/Agents | https://job-boards.greenhouse.io/sesolabor/jobs/4720111005 | Remote US | entry-mid |
| p207 | Pallet | Software Engineer, Agent Delivery | https://job-boards.greenhouse.io/pallet/jobs/5242991007 | San Francisco | entry-mid |
| p208 | Emergence AI | AI Engineer | https://job-boards.greenhouse.io/emergenceai/jobs/4374497009 | Irvine, CA | mid |
| p209 | G-P (Globalization Partners) | AI Engineer I | https://job-boards.greenhouse.io/globalizationpartners/jobs/8034413003 | UK (remote-first) | entry-mid |
| p210 | Dark Wolf Solutions | Junior AI Software Engineer | https://job-boards.greenhouse.io/darkwolfsolutions/jobs/7886153003 | Chantilly/Herndon, VA | entry |
| p211 | SimplifyNext | AI Engineer | https://job-boards.greenhouse.io/simplifynext/jobs/5019281007 | Singapore | mid |
| p212 | AppZen | AI/ML Engineer | https://jobs.lever.co/appzen/cce93662-196a-4814-b80c-bfa0b37384af | Pune, IN | entry-mid |
| p213 | Loka | Machine Learning Engineer | https://job-boards.greenhouse.io/lokainc/jobs/4097610007 | LatAm / Portugal / Serbia / North Macedonia | entry-mid |

### Reinforcing postings (not in denominator — senior / staff tier)

| # | Employer | Title | URL | Reason excluded |
|---|---|---|---|---|
| p301 | Material Bank | Applied AI Engineer (Senior) | https://job-boards.greenhouse.io/materialbank/jobs/7732615003 | Senior — inherits upward. Reinforces multi-agent (LangGraph/LangChain/Mastra), RAG, HITL, cost/latency depth. |
| p302 | Figma | AI Engineer, Internal Tools (5+ yrs) | https://job-boards.greenhouse.io/figma/jobs/6233043004 | Mid-senior (5+ yrs). Reinforces MCP, agent orchestration, HITL approval, prompt/version management. |
| p303 | Weedmaps | Senior Software Engineer, AI Platform (Remote) | https://job-boards.greenhouse.io/weedmaps77/jobs/8861322002 | Senior — inherits upward. Reinforces prompt-injection hardening, cost routing + caching, agent tracing. |
| p304 | Cloudflare | Software Engineer, AI Agents (5–10 yrs) | https://job-boards.greenhouse.io/cloudflare/jobs/8259402 | Mid-to-senior. Reinforces Agent SDK + MCP + function calling, A2A/multi-agent, LLM eval frameworks. |
| p305 | ION Group (Lab49) | Senior Agentic Developer, LLM-Native / Enterprise, NY | https://jobs.lever.co/ion/f7bf6c5b-b70e-44d9-92b7-d9593f8220e6 | Senior title. Reinforces AI-assisted dev workflow + hardening AI-generated code (compliance/audit). |

### Representative quotes anchoring the shifts this cycle

**Theme T18 — MCP crossed threshold (15% → 32%):**
> *"Build tool interfaces, such as MCP servers, that give scientists and agents structured access to internal and external resources."* — Materiom
> *"AI agents and MCP servers for task automation and reasoning."* — Lantern Studios
> *"Familiarity with the Model Context Protocol (MCP) or similar tool-integration frameworks for LLMs."* — Zone 5 Technologies
> *"Build LLM integrations, RAG pipelines, vector search, and agentic workflows using MCP, with human-in-the-loop oversight."* — Turning Point Global Solutions
> *"Hands-on experience building production agentic systems, RAG pipelines, or MCP-based tool integrations."* — Artera
> *"Experience with agent frameworks, MCP-style tool interfaces, RAG, or knowledge-base systems."* — Kpler
> *"build core agentic infrastructure, integrate tooling via AWS Lambda and MCP servers."* — Engelhart
> *"Connect AI agents to enterprise tools, APIs, and external platforms via Model Context Protocol (MCP)."* — CodeRoad

Breakdown by level 20 share: 5 postings (Materiom, Lantern Studios, Engelhart, CodeRoad, Scale-AI-implicit) frame MCP as **server implementation** — explicitly owned by `agentic-ai-engineer mod-202`. 3 postings (Zone 5, Artera, Kpler) frame MCP as "familiarity" / "MCP-style tool interfaces" — the level-20 share, coverable by an editorial handoff pointer in existing chapters.

**Theme T14 — Guardrails / prompt injection (24%, below threshold this cycle but reinforcing the pending 2026-10 proposal):**
> *"Run functional, performance, and adversarial testing (prompt injection, jailbreaks) and evaluate AI output for accuracy, hallucination, bias, and safety."* — Turning Point Global Solutions
> *"Apply secure development practices aligned with customer requirements, the OWASP Top 10 for LLM Applications, and NIST AI RMF."* — Turning Point Global Solutions
> *"Design agents that act safely against real systems, with appropriate guardrails, human-in-the-loop where warranted."* — Zone 5 Technologies
> *"Apply the crew's security guardrails and responsible-AI practices: access control, data privacy."* — Kpler
> *"Execute automated evaluations, regression testing, task-success metrics, and safety guardrails."* — CodeRoad

**Theme T15 — Observability (28%, oscillating at threshold):**
> *"Deliver AI observability capabilities end to end, from ingestion to in-product visualizations ... AI-specific signals, such as cost, token usage, errors, and anomalies."* — Honeycomb (vendor builds AI observability products)
> *"Observability and monitoring (Datadog, OpenTelemetry)."* — Xsolla
> *"leverage observability platforms such as LangSmith, Langtrace, or AgentOps."* — CodeRoad
> *"Contribute to evaluation frameworks and production monitoring for GenAI solutions."* — Loka

**Theme T19 — Multi-agent orchestration (32%, structurally upstream):**
> *"Develop production-grade AI agents, state management mechanisms, and multi-agent workflows ... frameworks such as LangGraph, CrewAI, or Agent Development Kit (ADK)."* — CodeRoad
> *"Design Agent-to-Agent (A2A) communication protocols that allow multiple autonomous agents to collaborate, delegate tasks."* — SimplifyNext
> *"Develop the runtime that powers long-running AI workflows, managing execution state ... agent orchestration, workflow execution, context management, memory, governance."* — Emergence AI

**Theme T21 — AI-assisted developer tools (24%, climbing, not a curriculum skill):**
> *"The ability to use AI coding tools (e.g., Claude Code, Codex, Gemini CLI)."* — DeepLearning.AI
> *"AI-native engineer who uses Claude Code, Cursor, Codex, or similar AI development tools."* — Emergence AI
> *"Experience using Claude Code, Cursor or similar tools as a working environment."* — Xsolla

## Out-of-scope requirements — link-outs

Requirements that appear in postings but belong to a peer or upward role per the ownership rule:

- **MCP server implementation depth** (T18) → [`agentic-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) `mod-202-frameworks/exercise-04-mcp-tool-server`
- **Multi-agent orchestration depth** (T19), **agent-framework bake-off** (LangGraph / CrewAI / AutoGen / OpenAI Agents SDK / Anthropic Claude Agent SDK — depth), **deep guardrails** (T14 depth), **durable execution** (Temporal / Airflow), **cost/latency depth** (prompt caching, model routing) (T16) → [`agentic-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning)
- **AI-application evaluation program depth** (trajectory eval, LLM-as-judge programs, OTel Gen-AI conventions, trace warehouses like Langfuse / Phoenix / Weave / Braintrust / LangSmith / Langtrace / AgentOps) (T15, T11 depth) → [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning)
- **RAG systems engineering depth** (rerankers, hybrid search, GraphRAG, agentic RAG) (T07 depth) → [`rag-engineer-learning`](https://github.com/ai-engineering-curriculum/rag-engineer-learning)
- **Applied AI production integration** (feature flags, canary rollouts, incident response for LLM features) → [`applied-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/applied-ai-engineer-learning)
- **Fine-tuning depth** (SFT, PEFT, RLHF, DPO) (T20) → [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning)
- **Deep prompt-injection defence, OWASP LLM Top 10 program, model extraction, LLM security** → `agentic-ai-engineer` `mod-206` → `agentic-safety-engineer` / `ai-infra-security`
- **Kubernetes / cloud platform depth** → `ai-infra-junior-engineer` / `ai-infra-mlops`

## Delta recommendation

**Zero additions this cycle.** The rationale is deliberate and sits inside the continuity-bias precondition set:

- The only theme that crossed the ≥30% threshold is **T18 Model Context Protocol (15% → 32%, 8 distinct postings)** — but of the 8 citing postings, 5 require **server implementation** (depth owned by `agentic-ai-engineer mod-202-frameworks/exercise-04-mcp-tool-server`) and only 3 require **"familiarity"** or **"MCP-style tool interfaces"** — the level-20 share. The level-20-only frequency is 3/25 = 12%, below the 30% threshold.
- The continuity-bias rule requires all three preconditions for a new item: (a) ≥ 3 distinct postings, (b) ≥ 30% frequency, (c) no existing module / exercise / project can be incrementally extended. Precondition (c) fails for T18: `mod-103/06-tool-errors-and-malformed-arguments.md` and `mod-105/06-limits-of-a-single-agent.md` can both absorb an editorial handoff paragraph naming MCP as "the emerging industry protocol for exposing tools to agents; server-build depth is covered in `agentic-ai-engineer mod-202`." That editorial extension is not an item this packet proposes (the delta schema covers only new modules / exercises / projects) — it is noted here for the human reviewer as a lightweight follow-up suitable for a separate editorial change.
- The 2026-10 cycle's proposed `mod-106/exercise-04-prompt-injection-and-input-output-filtering` **remains unmerged**. Adding another unmerged proposal in Nov would compound review backlog with no corresponding merge upside.
- All other ≥30% themes (T01–T04, T06–T12, T19) are already owned by existing modules (`mod-101`..`mod-106`) or by the explicit upstream handoff at `mod-105/06-limits-of-a-single-agent.md` (T19) and the entry-tier basics with upstream depth link-outs (T07, T09, T11, T12).
- Secondary movements: T15 Observability edged down (30% → 28%) — awareness coverage in `mod-106/04` remains defensible; T14 Guardrails edged down (35% → 24%) but one standout posting (Turning Point) explicitly cites OWASP Top 10 for LLM Applications + NIST AI RMF, reinforcing the Oct proposal's relevance when it merges; T19 Multi-agent is structurally upstream and unchanged.

Watch list for the 2026-12 cycle:
- **T18 MCP** — if the level-20 ("familiarity" / client-side consumption of MCP tools) share grows to ≥30% independently of server-build mentions, promote to a `mod-103` exercise next cycle. First `requires_explicit_approval: false` candidate.
- **T15 Observability** — if ≥30% holds for a third consecutive cycle, the awareness-only coverage in `mod-106/04` is no longer defensible; propose a `mod-106 exercise-05` (basic LLM-call trace with Langfuse / OTel Gen-AI conventions).
- **T21 AI-assisted dev tools** — if ≥30% in-scope, write a short appendix (not an exercise) framing the workflow expectation.

See [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) for the (empty) delta.
