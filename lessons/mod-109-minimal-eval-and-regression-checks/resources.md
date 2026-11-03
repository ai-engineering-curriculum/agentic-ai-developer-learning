# Resources for mod-109 — Minimal Evaluation and Regression Checks for LLM Features

Prefer primary sources — provider documentation, tool maintainers, and standards — over blog posts. The list below is deliberately short. If a link goes stale, the fix is to look up the maintainer's current documentation index, not to grab whatever a search engine returns first.

## pytest — the test runner this module standardises on

- **`pytest` documentation index.** <https://docs.pytest.org/en/stable/>
- **Parametrising tests** — the `@pytest.mark.parametrize` machinery every eval row goes through, including the `ids=` argument for surfacing row IDs in output. <https://docs.pytest.org/en/stable/how-to/parametrize.html>
- **Working with custom markers** — how to register `@pytest.mark.eval` in `pyproject.toml` so `-m eval` does not raise `PytestUnknownMarkWarning`. <https://docs.pytest.org/en/stable/how-to/mark.html>
- **Skipping and xfail** — the machinery `conftest.py` uses to skip eval tests unless `RUN_EVALS=1` is set. <https://docs.pytest.org/en/stable/how-to/skipping.html>
- **JUnit XML output** — `--junitxml` for CI-consumable results. <https://docs.pytest.org/en/stable/how-to/output.html#creating-junitxml-format-files>
- **`pytest-xdist`** — parallel test execution (`-n auto`) that makes a 30-row eval finish in under a minute. <https://pytest-xdist.readthedocs.io/en/stable/>

## Provider docs — structured output, pricing, model reference

Every eval is only as strict as the shape it can assert on. Structured-output APIs make schema conformance cheap; pricing pages are the source of truth for the cost line in your run manifest.

### OpenAI

- **Structured Outputs** — strict JSON Schema mode used in mod-102 chapter 5 and reused in mod-109 rule-based schema checks. <https://platform.openai.com/docs/guides/structured-outputs>
- **Model reference** — current model IDs and snapshot names. Pin these in your prompts and check the pin in your CI (exercise 04). <https://platform.openai.com/docs/models>
- **Pricing** — canonical per-million-token prices for the `cost_usd` field in your run manifest. <https://openai.com/api/pricing/>
- **Deprecations** — which snapshots are being retired and when. Bookmark this; a silent model roll-forward is the single most easily missed source of regression drift (chapter 4). <https://platform.openai.com/docs/deprecations>

### Anthropic

- **Tool use** — the tool-schema forcing pattern used as the Anthropic-side equivalent for structured output. <https://docs.anthropic.com/en/docs/build-with-claude/tool-use>
- **Models** — current Claude model IDs and snapshot names. <https://docs.anthropic.com/en/docs/about-claude/models>
- **Pricing** — canonical per-million-token prices. <https://docs.anthropic.com/en/docs/about-claude/pricing>

<!-- needs-research: confirm the current cheap-tier and frontier-tier Claude and GPT model IDs and their per-token prices as of 2026-08 — cite from the provider pages linked above. -->

## Embedding-based similarity scoring

- **OpenAI Embeddings guide** — request shape, model reference, batch limits. <https://platform.openai.com/docs/guides/embeddings>
- **Voyage AI Embeddings reference** — the Anthropic-recommended embeddings companion for Claude workloads. Includes the `input_type` (`"document"` vs `"query"`) distinction — for similarity scoring in evals you use `"document"` for both sides, since you are comparing two "documents" for meaning, not doing retrieval. <https://docs.voyageai.com/docs/embeddings>
- **Cohere `embed` API reference.** <https://docs.cohere.com/reference/embed>
- **mod-104 chapter 2** — the primer on cosine similarity, embedding-model determinism, and the exact failure modes (identifiers / negation / dates) that make similarity a bad default for negation-sensitive eval rows. `../mod-104-retrieval-basics/02-embeddings-and-vector-space.md`

## JSON Schema — for rule-based schema conformance checks

- **JSON Schema specification and reference** — the language your `schema` tolerance is written in. Prefer the current published draft. <https://json-schema.org/>
- **`jsonschema` for Python** — the standard validator used in chapter 2's `schema_conforms`. <https://python-jsonschema.readthedocs.io/en/stable/>

## LLM-as-a-judge — starter reading before calibration

These are the widely-cited papers that formalise the LLM-as-judge idea and its known biases (positional bias, verbosity bias, self-preference). Skim, do not memorise — full calibration methodology lives in `ai-eval-engineer-learning`.

- **"Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena"** — Zheng et al., 2023. The paper that introduced the term and the biases. <https://arxiv.org/abs/2306.05685>

<!-- needs-research: verify the arXiv link and canonical citation for the Zheng et al. MT-Bench / Chatbot Arena paper; confirm this is the primary source, not a superseded revision. -->

- **OpenAI Evals cookbook — "How to evaluate LLMs"** — pragmatic patterns for judge prompts and calibration on OpenAI's side. <https://cookbook.openai.com/examples/evaluation/getting_started_with_openai_evals>

<!-- needs-research: confirm the exact current URL for OpenAI's evaluation cookbook article and its title as of 2026-08 — this article has been renamed in the past. -->

- **Anthropic — "Reduce hallucinations" and prompt engineering guide.** Not judge-specific, but the discipline of writing a rubric with worked pass/fail examples comes directly from the same techniques. <https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview>

## CI providers — path filters and required checks

The exercise-02 gate needs two features from your CI provider: run-on-relevant-paths and required-for-merge. Every major provider has them under different names.

- **GitHub Actions — path filters on `pull_request` triggers.** <https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onpushpull_requestpull_request_targetpathspaths-ignore>
- **GitHub — required status checks and branch protection.** <https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches>
- **GitLab CI — `rules:changes`.** <https://docs.gitlab.com/ee/ci/yaml/#ruleschanges>
- **CircleCI — path filtering orb.** <https://circleci.com/developer/orbs/orb/circleci/path-filtering>
- **Buildkite — dynamic pipelines.** <https://buildkite.com/docs/pipelines/defining-steps#dynamic-pipelines>

<!-- needs-research: confirm the GitHub required-checks docs URL as of 2026-08 — this page has been reorganised more than once. -->

## Cost and version pinning — from mod-108, forward-referenced

The `cost_usd` field in the exercise-04 run manifest and the pinned `MODEL` constant both trace back to mod-108's discipline. If either feels shaky, re-read:

- **mod-108 chapter 2** — cost estimation shape. `../mod-108-model-selection-cost-and-prompt-caching/`
- **mod-108 chapter 3** — prompt caching, which is what makes the eval affordable to run on every relevant PR.

## Where the deep material lives

The topics mod-109 deliberately does *not* teach in depth all live in the specialist track `ai-eval-engineer-learning`. Chapter 5 explains the boundary in detail; the pointer:

- **`ai-eval-engineer-learning` (level 30)** — LLM-as-judge calibration methodology (inter-annotator agreement, bias mitigation, meta-evaluation), RAG-specific eval (RAGAS, TruLens, precision@k / recall@k / faithfulness / context precision), online eval on production traffic, observability / tracing platforms (Langfuse, LangSmith, Arize Phoenix, Braintrust, OpenTelemetry with LLM semantic conventions, Datadog LLM Observability), cost / latency / quality dashboards, agent-trajectory eval, and statistical rigour on eval numbers. <https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning>

The eval-adjacent frameworks named for the graduation are:

- **RAGAS** — RAG-specific eval framework. <https://docs.ragas.io/>
- **TruLens** — RAG + agent eval framework. <https://www.trulens.org/>
- **promptfoo** — general-purpose LLM eval and A/B framework. <https://www.promptfoo.dev/docs/intro/>
- **DeepEval** — pytest-flavoured LLM eval framework. <https://docs.confident-ai.com/>
- **Inspect (UK AISI)** — model / agent eval framework built for safety-eval workloads. <https://inspect.aisi.org.uk/>

These are named so you know what to compare against when you graduate. Do not adopt one before you have a raw pytest golden set — you will not know what any of them is opinionated about until you have built the primitives yourself.

## Observability platforms named in the boundary chapter

Not for use in this module — for use *after* the boundary chapter tells you the primitives here are not enough. Named here so the terminology is not foreign.

- **Langfuse** — open-source LLM observability + eval. <https://langfuse.com/docs>
- **LangSmith** — LangChain's hosted observability + eval. <https://docs.smith.langchain.com/>
- **Arize Phoenix** — open-source LLM tracing and eval. <https://docs.arize.com/phoenix>
- **Braintrust** — hosted LLM eval + observability. <https://www.braintrust.dev/docs>
- **OpenTelemetry semantic conventions for GenAI** — the emerging vendor-neutral standard for tracing LLM calls. <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **Datadog LLM Observability.** <https://docs.datadoghq.com/llm_observability/>

## What is deliberately not on this list

- **"How to build an LLM eval in 10 lines of code" tutorials.** Almost all of them use one framework (RAGAS, DeepEval, promptfoo) and hide the exact primitives — golden-set shape, per-row tolerance, scorer dispatch, run manifest — this module is trying to teach you to see. Learn the raw shape first (this module), pick your framework second.
- **Provider pricing screenshots.** Prices change; screenshots do not. Always link to the current pricing page and put the price you used in the run manifest, dated.
- **Blog posts arguing for or against LLM-as-a-judge.** The trade-offs — cost, coverage, calibration, non-determinism — are the ones chapter 2 lays out from the primitives. If you find yourself deep in a "judges are dangerous" or "judges are the answer" post, close it and go back to your own agreement numbers from exercise 03.
