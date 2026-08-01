# Job Requirements — Agentic AI Developer

**Role level:** 20 (entry rung, Agentic track)
**Track:** `agentic-ai-developer-learning`
**Research window:** 2026-05-03 → 2026-08-01 (last 90 days)
**Today:** 2026-08-01

This file maps verbatim requirements from current Agentic AI Developer / AI Engineer / LLM Application Developer / Generative AI Developer / Applied AI Developer / AI Application Engineer job postings to the existing curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the strictly-additive proposal lives in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Summary

- Postings sampled: **22** (20 in-denominator + 2 senior-tier retained as reinforcing evidence).
- Distinct in-scope postings (entry-to-mid, actively observed on 2026-08-01): **20**.
- **Proposed delta this cycle: 0 modules, 0 exercises, 0 projects.** Every requirement above the continuity-bias threshold (≥3 distinct postings AND ≥30% frequency) is already owned by an existing module in `mod-101..mod-106`.
- Sub-threshold signals tracked for next cycle: guardrails / prompt-injection defense, observability / tracing (Langfuse / OpenTelemetry), MCP servers, multi-agent orchestration, cost / latency depth (prompt caching, model routing), AI-assisted coding tools (Cursor / Claude Code / Copilot).

## Methodology

- Sources: `job-boards.greenhouse.io` (direct WebFetch), `builtin.com` (direct WebFetch), `jobs.ashbyhq.com` (limited — Ashby renders client-side; only titles reliably captured on some), and Indeed snippet-level via WebSearch.
- Per-posting capture: employer, verbatim title, URL, `date_observed`, `date_posted` (marked `est:2026-Q2/Q3` when only an active-listing observation was possible), location, and 5-10 verbatim technical requirement quotes.
- Frequency = distinct in-scope postings citing the theme ÷ 20 in-scope postings.
- In-scope filter: entry-to-mid altitude. Postings with explicit **Senior / Staff / Principal / Lead / Architect** modifiers were excluded from the denominator (they inherit upward to `agentic-ai-engineer` / `senior-agentic-ai-engineer` / `applied-ai-engineer` / `staff-ml-engineer`). Two senior-only postings (SecurityScorecard, Mercury) are retained as reinforcing evidence but NOT counted in any frequency.
- Source caveats:
  - Several postings' exact `date_posted` was not visible in fetched HTML; those are marked `est:2026-Q2/Q3` — the active listing was observed on 2026-08-01. The theme-frequency conclusion is robust because every high-frequency theme is corroborated by ≥8 distinct postings.
  - Three Ashby postings (Zapier, Paraform, Arlo) required client-side render for the body; excluded from the denominator to avoid inflating frequencies with thin evidence.

## Requirement themes → curriculum ownership

Every requirement observed, its in-scope frequency, the role that owns it per the level hierarchy, and the curriculum coverage path. Frequencies ≥30% are **bolded** — those are the themes the continuity-bias rule treats as load-bearing.

| # | Theme | Freq | Owner role | Coverage |
|---|---|---|---|---|
| T01 | Python + software-engineering habits (Git, testing, packaging, HTTP/JSON) | **90%** | `agentic-ai-developer` (prerequisite tier) | [`PREREQUISITES.md`](PREREQUISITES.md) — assumed at entry; not re-taught |
| T02 | LLM APIs (OpenAI, Anthropic, Google/Gemini, Azure OpenAI, Bedrock, Mistral) — tokens, context, streaming, cost, errors, retries | **75%** | `agentic-ai-developer` (this) | [`mod-101-llm-fundamentals`](lessons/mod-101-llm-fundamentals) — 8 chapters + 3 exercises |
| T03 | Prompt engineering (system prompts, few-shot, chain-of-thought, decomposition, iterating with evals) | **70%** | `agentic-ai-developer` | [`mod-102-prompt-engineering`](lessons/mod-102-prompt-engineering) — 7 chapters + 3 exercises |
| T04 | Tool / function calling (define tools, call/result loop, when-to-call, malformed args) | **45%** | `agentic-ai-developer` | [`mod-103-tool-and-function-calling`](lessons/mod-103-tool-and-function-calling) — 6 chapters + 3 exercises |
| T05 | Structured output (JSON mode, schema-constrained, Pydantic / JSON Schema validation) | 20% | `agentic-ai-developer` | [`mod-102-prompt-engineering/exercises/exercise-02-structured-json-output.md`](lessons/mod-102-prompt-engineering/exercises/exercise-02-structured-json-output.md), [`exercise-03-output-validation.md`](lessons/mod-102-prompt-engineering/exercises/exercise-03-output-validation.md); Pydantic-validated requests in [`mod-106-llm-app-deployment/02-fastapi-and-input-validation.md`](lessons/mod-106-llm-app-deployment/02-fastapi-and-input-validation.md) |
| T06 | Retrieval-Augmented Generation (RAG) — embeddings, similarity search, chunking, indexing, grounded generation | **75%** | `agentic-ai-developer` | [`mod-104-retrieval-basics`](lessons/mod-104-retrieval-basics) — 7 chapters + 3 exercises |
| T07 | Vector databases (Pinecone, Weaviate, Chroma, pgvector, FAISS) | **50%** | `agentic-ai-developer` (basics) → [`rag-engineer`](https://github.com/ai-engineering-curriculum/rag-engineer-learning) (depth) | [`mod-104-retrieval-basics/03-similarity-search-and-indexes.md`](lessons/mod-104-retrieval-basics/03-similarity-search-and-indexes.md), [`exercise-01-embeddings-similarity.md`](lessons/mod-104-retrieval-basics/exercises/exercise-01-embeddings-similarity.md); rerankers / hybrid / GraphRAG deferred to `rag-engineer` |
| T08 | First agent (ReAct / reason-act loop, tool loop, retrieval-as-tool, conversation memory) | **60%** | `agentic-ai-developer` | [`mod-105-first-agent`](lessons/mod-105-first-agent) — 6 chapters + 3 exercises |
| T09 | Agent frameworks — LangChain, LangGraph, LlamaIndex (basic use) | **35%** | `agentic-ai-developer` (awareness) → [`agentic-ai-engineer`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) (bake-off depth) | [`mod-105-first-agent/06-limits-of-a-single-agent.md`](lessons/mod-105-first-agent/06-limits-of-a-single-agent.md) handoff; framework bake-off owned by `agentic-ai-engineer` `mod-202-frameworks` |
| T10 | Deploy an LLM app behind an API (FastAPI, REST, containerization, secrets, structured logging, cost/usage guardrail) | **55%** | `agentic-ai-developer` | [`mod-106-llm-app-deployment`](lessons/mod-106-llm-app-deployment) — 7 chapters + 3 exercises |
| T11 | Basic evaluation of LLM outputs (rubrics, offline benchmarks, retrieval quality / recall@K) | **50%** | `agentic-ai-developer` (basics) → [`ai-eval-engineer`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) (program depth) | [`mod-102-prompt-engineering/07-iterating-with-evals.md`](lessons/mod-102-prompt-engineering/07-iterating-with-evals.md), [`mod-104-retrieval-basics/07-evaluating-retrieval-quality.md`](lessons/mod-104-retrieval-basics/07-evaluating-retrieval-quality.md); judge-tier eval programs deferred to `ai-eval-engineer` |
| T12 | Cloud platform basics (AWS / Azure / GCP) for LLM app hosting | **60%** | `agentic-ai-developer` (basics) → ai-infra tracks (depth) | [`mod-106-llm-app-deployment`](lessons/mod-106-llm-app-deployment) covers Docker + local run; deep cloud is out-of-scope link-out |
| T13 | Docker / containerization | 20% | `agentic-ai-developer` | [`mod-106-llm-app-deployment/06-containerizing-with-docker.md`](lessons/mod-106-llm-app-deployment/06-containerizing-with-docker.md); Kubernetes deferred to ai-infra tracks |
| T14 | Guardrails / prompt-injection defense / input+output filtering | 15% | [`agentic-ai-engineer`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) `mod-206-guardrails-implementation` → deeper in [`agentic-safety-engineer`](https://github.com/ai-engineering-curriculum/agentic-safety-engineer-learning) | Awareness only via [`mod-103/06-tool-errors-and-malformed-arguments.md`](lessons/mod-103-tool-and-function-calling/06-tool-errors-and-malformed-arguments.md) and [`mod-106/05-cost-and-usage-guardrails.md`](lessons/mod-106-llm-app-deployment/05-cost-and-usage-guardrails.md); depth link-out |
| T15 | Observability / tracing (Langfuse, OpenTelemetry, Datadog) for LLM apps | 10% | [`ai-eval-engineer`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) / `agentic-ai-engineer` `mod-205-evaluation-observability` | Awareness only via [`mod-106-llm-app-deployment/04-logging-llm-requests.md`](lessons/mod-106-llm-app-deployment/04-logging-llm-requests.md); depth link-out |
| T16 | Cost / latency optimization (prompt caching, token budgets, model routing, cost attribution) | 15% | `agentic-ai-engineer` `mod-207-productionizing-agents` (depth) | Awareness owned here via [`mod-101/07-cost-and-latency.md`](lessons/mod-101-llm-fundamentals/07-cost-and-latency.md) and [`mod-106/05-cost-and-usage-guardrails.md`](lessons/mod-106-llm-app-deployment/05-cost-and-usage-guardrails.md); depth link-out |
| T17 | Streaming responses (SSE, chunked transfer, cancellation) | 5% | `agentic-ai-developer` | [`mod-101-llm-fundamentals/06-streaming-responses.md`](lessons/mod-101-llm-fundamentals/06-streaming-responses.md) — already covered |
| T18 | Model Context Protocol (MCP) — server implementation and tool exposure | 5% | `agentic-ai-engineer` `mod-202-frameworks/exercise-04-mcp-tool-server` | Out of scope here (below threshold; 1 in-scope + 1 reinforcing). Owned upstream. |
| T19 | Multi-agent orchestration / planners / handoffs / sub-agents | 15% | `agentic-ai-engineer` `mod-204-multi-agent-implementation` | Explicit handoff at [`mod-105-first-agent/06-limits-of-a-single-agent.md`](lessons/mod-105-first-agent/06-limits-of-a-single-agent.md); depth owned upstream |
| T20 | Fine-tuning depth (SFT, LoRA, RLHF) | 10% | [`fine-tuning-engineer`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning) | Out of scope. Link-out only. |
| T21 | AI-assisted developer tools (Cursor, Claude Code, GitHub Copilot) as workflow | 15% | workflow / not-a-curriculum-topic | Not a curriculum-owned skill — a workflow signal. Revisit if freq climbs above 30%. |
| T22 | Human-in-the-loop / approval checkpoints | 10% | `agentic-ai-engineer` `mod-206-guardrails-implementation` (depth) | Out of scope here (below threshold). |

## Posting evidence

Every posting below was observed live on **2026-08-01**. Denominator postings are entry-to-mid altitude; the two senior postings at the bottom are retained as **reinforcing** evidence only (not counted toward any frequency).

### In-scope postings (denominator = 20)

| # | Employer | Title | URL | Location | Seniority |
|---|---|---|---|---|---|
| p001 | Materiom | AI Engineer | https://job-boards.greenhouse.io/materiom/jobs/5150829007 | London (hybrid) | mid |
| p002 | Rackner | AI/ML Engineer — Generative AI Mission Systems | https://job-boards.greenhouse.io/rackner/jobs/4717776005 | Remote US / Laurel MD | mid |
| p003 | Cadence Solutions | AI Engineer | https://job-boards.greenhouse.io/solutions/jobs/4680769006 | Remote | mid |
| p004 | Splitero | Applied AI Engineer | https://job-boards.greenhouse.io/splitero/jobs/5162723008 | Remote | mid |
| p005 | CoreStory | AI Engineer | https://job-boards.greenhouse.io/corestory/jobs/4984207007 | Remote US | mid |
| p006 | Quadbridge | AI Application Developer | https://job-boards.greenhouse.io/quadbridge/jobs/4326329009 | Montreal QC | entry-mid |
| p007 | Future | Applied AI Engineer | https://job-boards.greenhouse.io/future/jobs/4683133005 | Remote | mid |
| p008 | Hasbro (WotC) | Applied AI Engineer | https://job-boards.greenhouse.io/hasbro/jobs/4250645009 | Boston MA | mid |
| p009 | GitLab | AI Engineer | https://job-boards.greenhouse.io/gitlab/jobs/8565469002 | Remote US | mid |
| p010 | Auctor | Software Engineer, Applied AI | https://jobs.ashbyhq.com/auctor/ca5b0c44-cafb-48ad-99fa-84aa3cfc5179 | NYC (on-site) | mid |
| p011 | Capco | Junior AI Engineer | https://www.indeed.com/q-capco-junior-ai-engineer-jobs.html | Orlando FL (hybrid) | entry |
| p012 | Q-Centrix | Junior AI Engineer | https://www.indeed.com/q-q-centrix-junior-ai-engineer-jobs.html | Remote/Hybrid US | entry |
| p013 | Optum | Associate AI/ML Engineer (RAG/LLM) | https://www.indeed.com/q-optum-associate-ai-ml-engineer-jobs.html | Basking Ridge NJ (hybrid) | entry |
| p014 | ICF | 2026 Summer Intern, AI Engineering | https://www.indeed.com/q-icf-summer-intern-ai-engineering-jobs.html | Remote US | intern |
| p015 | Roadie (UPS) | AI Engineer Intern - Summer 2026 | https://builtin.com/job/ai-engineer-intern-summer-2026/8880520 | Remote US | intern |
| p016 | Intetics Inc. | Python AI Engineer | https://builtin.com/job/python-ai-engineer/4803316 | Remote | mid |
| p017 | Nordstrom | Engineer 1: AI Agentic Solutions | https://www.indeed.com/q-nordstrom-engineer-1-ai-agentic-solutions-jobs.html | Seattle WA (hybrid) | entry |
| p018 | Fortinet | Junior Applied AI Engineer - HRIS | https://www.indeed.com/q-fortinet-junior-applied-ai-engineer-jobs.html | Sunnyvale CA | entry |
| p019 | Tata Consultancy Services (TCS) | Junior AI/ML Engineer | https://www.indeed.com/q-tcs-junior-ai-ml-engineer-atlanta-jobs.html | Atlanta GA | entry |
| p020 | Remote People | AI Engineer | https://job-boards.eu.greenhouse.io/remotepeople/jobs/4843110101 | Lisbon / Remote EU | mid |

### Reinforcing postings (not in denominator — senior tier)

| # | Employer | Title | URL | Reason excluded |
|---|---|---|---|---|
| p021 | SecurityScorecard | Senior AI Engineer | https://job-boards.greenhouse.io/securityscorecard/jobs/7776439 | Senior — inherits upward to `agentic-ai-engineer` / `applied-ai-engineer` |
| p022 | Mercury | Senior Software Engineer - AI Engineering | https://job-boards.greenhouse.io/mercury/jobs/5850044004 | Senior — inherits upward. Retained because MCP + LLM-gateway signal is worth tracking. |

### Representative quotes anchoring the load-bearing themes

**Theme T02 — LLM APIs (75%):**
> *"Solid understanding of LLM APIs (OpenAI, Anthropic, Mistral, Hugging Face)."* — CoreStory
> *"Proficiency with major LLM APIs (OpenAI, Google Gemini, Anthropic, or equivalent)."* — Hasbro
> *"Practical fluency across the LLM ecosystem: hands-on experience with models from Anthropic, OpenAI, open-source alternatives."* — GitLab

**Theme T03 — Prompt engineering (70%):**
> *"Deep expertise in prompt engineering — system prompt design, few-shot techniques, chain-of-thought, multi-turn conversation management."* — Hasbro
> *"Design prompt flows, orchestrations, and grounding strategies."* — Quadbridge
> *"Prompt engineering as a core discipline: designing effective system prompts, managing context windows."* — GitLab

**Theme T04 — Tool / function calling (45%):**
> *"Hands-on with LLM APIs (OpenAI, Anthropic, open-source) including prompt engineering, tool/function calling."* — Cadence Solutions
> *"Build and ship AI agents that serve real users: tool-calling LLM systems with structured output, parallel API orchestration."* — Future
> *"Implement individual components of agentic solutions — tool integrations, prompt templates, retrieval steps."* — Nordstrom

**Theme T06 — RAG (75%):**
> *"Building RAG systems: embeddings, vector stores, retrieval optimization, and grounding."* — Cadence Solutions
> *"Hands-on experience with RAG architectures: embedding models, vector stores, chunking strategies, and retrieval evaluation."* — Hasbro
> *"Develop and deploy LLM-based solutions and RAG architectures that solve real business problems."* — Remote People

**Theme T07 — Vector databases (50%):**
> *"Implement and maintain vector databases (Pinecone, Neo4j, Weaviate, Chroma, Azure)."* — CoreStory
> *"Vector databases (pgvector, FAISS)."* — Roadie
> *"Vector database experience (ChromaDB, Weaviate)."* — Intetics

**Theme T08 — First agent (60%):**
> *"Design, build, deploy clinical AI agents that reason over patient context, invoke tools, generate recommendations."* — Cadence Solutions
> *"Build and improve AI chatbot agents powered by LLMs."* — Roadie
> *"Experience with LLM-powered products, agent systems, or similar applied AI work."* — Auctor

**Theme T09 — Agent frameworks basics (35%):**
> *"Experience with LangChain/LangGraph or similar agent frameworks."* — Future
> *"Experience with multi-agent frameworks (LangChain, LlamaIndex, or similar)."* — Splitero
> *"LangChain, AutoGen, or OpenAI APIs."* — Capco

**Theme T10 — Deploy LLM app (55%):**
> *"Proficiency in Python, with experience in FastAPI, Pandas, NumPy."* — CoreStory
> *"Strong programming skills in Python with hands-on async patterns and FastAPI experience."* — Remote People
> *"Experience with OpenShift, Kubernetes, CI/CD, or containerized application delivery."* — Rackner

**Theme T11 — Basic evals (50%):**
> *"Design evaluation harnesses and quality scoring — we use Langfuse, rubrics to measure safety, effectiveness, personalization."* — Future
> *"Familiarity with vector databases, embedding pipelines, evaluation frameworks for LLM outputs."* — Splitero
> *"Evaluate AI outputs for grounding, reliability, accuracy, relevance."* — Rackner

**Theme T12 — Cloud basics (60%):**
> *"Familiarity with AWS (Bedrock, ECR, CloudFront, S3, Cognito) or other cloud agent hosting."* — Future
> *"Leverage Azure services (Azure AI, Azure Functions, Azure OpenAI) for scalable, secure architectures."* — Quadbridge
> *"Cloud platforms (AWS, Azure, Google Cloud)."* — ICF

## Out-of-scope requirements — link-outs

Requirements that appear in postings but belong to a peer or upward role per the ownership rule:

- **Multi-agent orchestration** (T19), **agent-framework bake-off** (LangGraph / CrewAI / AutoGen / OpenAI Agents SDK / Anthropic Agent SDK — depth), **MCP servers** (T18), **deep guardrails** (T14), **durable execution** (Temporal / Airflow), **cost/latency depth** (prompt caching, model routing) (T16) → [`agentic-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning)
- **AI-application evaluation program depth** (trajectory eval, LLM-as-judge, OTel Gen-AI conventions, trace warehouses) (T15, T11 depth) → [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning)
- **RAG systems engineering depth** (rerankers, hybrid search, GraphRAG, agentic RAG) (T07 depth) → [`rag-engineer-learning`](https://github.com/ai-engineering-curriculum/rag-engineer-learning)
- **Applied AI production integration** (feature flags, canary rollouts, incident response for LLM features) → [`applied-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/applied-ai-engineer-learning)
- **Fine-tuning depth** (SFT, PEFT, RLHF, DPO) (T20) → [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning)
- **Deep prompt-injection defence, model extraction, LLM security** → `agentic-safety-engineer` / `ai-infra-security`
- **Kubernetes / cloud platform depth** → `ai-infra-junior-engineer` / `ai-infra-mlops`

## Delta recommendation

**Zero additions this cycle.** Every ≥30% theme is already covered by existing modules (`mod-101` LLM APIs, `mod-102` prompts + evals + structured output, `mod-103` tool/function calling, `mod-104` RAG + embeddings + vector DBs, `mod-105` first agent, `mod-106` deployment + FastAPI + Docker + secrets + logging + cost guardrail). Sub-threshold signals either already have awareness-level coverage inside an existing module or belong to a peer/upward role per the ownership rule.

See [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) for the (empty) delta.
