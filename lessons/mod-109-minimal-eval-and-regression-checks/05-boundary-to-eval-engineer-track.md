# Chapter 5 — The boundary to `ai-eval-engineer-learning`

This is a short chapter about a deliberately narrow module. Its job is to say what mod-109 is not trying to teach, so you do not spend three months building a bad copy of a track that already exists for the depth you need. The specialist track for this is [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) (level 30). Everything mod-109 stops short of, that track covers as its main material.

## Motivation

Evaluation of LLM systems is a discipline of its own. It has its own reading list — from psychometrics-inspired inter-annotator agreement, to LLM-as-judge calibration methodology, to RAG-specific eval frameworks like RAGAS, to production observability shapes borrowed from web operations. Trying to compress that whole practice into one module of the "Agentic AI Developer" track produces a shallow version of both. mod-109 is deliberately narrow: get you to a working golden set, wired into pytest, gated in CI, with over-time regression tracking that lets a first LLM feature ship without silently regressing. Everything past that is where the specialist track earns its keep.

Two failure modes this chapter is written to prevent:

1. **Building a serious evaluation practice out of just the tools in this module.** Everything in mod-109 works for one feature, a golden set of a few dozen rows, one team, one product surface. Push it into a multi-feature, multi-tenant, high-traffic surface with per-segment quality SLOs and continuous online evaluation — without the specialist track's material — and you will end up reinventing the missing pieces the expensive way.
2. **Reproducing the specialist track's material inside this module.** LLM-as-judge calibration methodology, RAG-specific eval, online eval on production traffic, observability and tracing platforms, cost / latency / quality dashboards — every one of those has a full chapter (or several) in `ai-eval-engineer-learning`. If you find yourself building a rolling per-segment latency dashboard on top of your pytest suite, the sign is on the wall.

## What is in scope for this module

The material is exactly what chapters 1–4 have covered:

- **Build a golden set** of 20–50 examples for one LLM feature, with an explicitly defended sampling strategy and per-row tolerance.
- **Score outputs** with rule-based checks (exact / regex / keyword / schema / length), embedding-similarity checks against a reference answer, and *one* calibrated LLM-as-a-judge check where nothing cheaper covers what "correct" means.
- **Run the golden set as a pytest suite** with a real `RUN_EVALS` gate, marks, per-row IDs, and a `--last-failed` inner loop.
- **Gate the suite in CI** on the changes that plausibly affect the feature. Post the diff back to the PR. Require the eval as a merge check with a documented override.
- **Persist run history** in the repo, maintain a trunk baseline, diff each run against it, and route each regression to a specific PR / commit / prompt version / model version and its author.
- **Draw the boundary,** the topic of this chapter.

Everything outside this list, and adjacent to it — see the next section.

## What is *out* of scope and lives in `ai-eval-engineer-learning`

Not exhaustive, but the biggest categories:

- **LLM-as-a-judge calibration methodology at depth.** Chapter 2 introduced the 20-row hand-labelled agreement check because you cannot ship a judge you have not calibrated at all. Real calibration is a much larger topic: inter-annotator agreement metrics (Cohen's / Fleiss' kappa, Krippendorff's alpha), pairwise vs. absolute rubrics, bias mitigation (positional bias, verbosity bias, self-preference when the judge is the same family as the model under test), meta-evaluation (evaluating the eval), rubric refinement loops, and using multiple judges to reduce noise.
- **RAG-specific evaluation.** The retrieval / generation split has its own methodology and its own metrics. Precision@k, recall@k, MRR, nDCG on the retrieval side. Faithfulness / answer relevance / context precision / context recall on the generation side. Frameworks like RAGAS and TruLens exist specifically because a general eval loop is not enough for RAG. Everything mod-104 chapter 7 called "the retrieval-vs-prompt triage" becomes a first-class metric here.
- **Online evaluation.** Running evals against live production traffic — not against a static golden set. A/B testing with real users, statistical significance, guardrail metrics, canary rollouts, user-implicit-feedback signals (thumbs up/down, follow-up-question rates, task-completion rates). None of this is on the pytest-in-CI path; it needs a different runtime and a different mental model.
- **Observability and tracing.** Instrumenting LLM apps with proper distributed tracing — Langfuse, LangSmith, Arize Phoenix, Braintrust, OpenTelemetry with LLM-specific semantic conventions, Datadog LLM Observability. Trace-level views of prompt + retrieval + tool calls + response + score. This is where the "log every call" habit chapter 3 mentioned gets scaled up.
- **Cost / latency / quality dashboards.** Time-series metrics that split by model, prompt version, customer segment, request class. Rolling averages, alerting on drift, blameless post-mortems on quality drops.
- **Human-in-the-loop labelling infrastructure.** How to source, brief, and QA a labelling workforce. When to use experts vs. crowd. Building and versioning labelling guidelines. Sample-active-learning to prioritise which live requests get labelled next.
- **Safety / red-team evaluation.** Adversarial suites for jailbreaks, prompt injection, harmful content, PII leakage, prompt-guard integrations. The mod-102 chapter on jailbreaks is where you first met this; the depth lives in the specialist track.
- **Multi-turn / agent-loop eval.** When the feature is a multi-step agent (multiple tool calls, multiple model turns, memory), a golden set is not a set of `(input, expected)` pairs — it is a set of *trajectories*. Trajectory-level metrics (step correctness, tool-use correctness, plan quality) belong here.
- **Statistical scoring.** Confidence intervals on the pass rate, minimum detectable effect on regressions given the set size, power analysis for "how many more rows would I need to detect a 2-point drop with 95% confidence." Once evals gate a product decision this depth pays.
- **Cross-feature and organisation-level eval infrastructure.** A shared eval platform that many features and teams use, with common storage, common scoring primitives, common dashboards. Building *that* is a specialist role — this module is about the first feature owning its own eval well.

Every one of the above is a *good* problem. Every one of the above is a problem the eval track will teach you to think about carefully. Every one of the above is a bad problem to invent from scratch inside a mod-109-shaped setup.

## When to graduate

Rough signals — none are strict; together they are unambiguous:

- **You have more than one LLM feature with its own eval.** The moment you have three, you start wanting shared primitives (a common scoring library, a shared run-storage format, a shared dashboard). "Shared eval infrastructure" is the specialist track's material.
- **You are calibrating more than one judge.** If your judges start multiplying, you need the depth of chapter 2 of the eval track — inter-annotator agreement, bias diagnostics, meta-evaluation — not the "hand-label 20 rows once" starter.
- **A product decision hinges on the eval number.** "We will ship a model swap iff the eval does not drop by more than 2 points" is a decision that lives or dies on how confident the number is. Once a PM is quoting it, you need statistical rigour on it.
- **You need production signal, not just PR signal.** The pytest-in-CI shape covers "did this PR regress the feature." It does not cover "the feature has been quietly degrading for a week on Spanish-language traffic." That is online eval and observability, both eval-track material.
- **A colleague is spending most of their week on evals.** Congratulations — that colleague is now an AI eval engineer. Give them the eval track and grow them into that role.
- **RAG-specific eval or agent-trajectory eval is the bottleneck.** The rubrics get specialised enough that a general golden-set-plus-pytest shape stops fitting the shape of the problem.

## What mod-109 does prepare you for

Even after graduating to the eval track, everything you built here still holds:

- **The 20–50 golden-set shape** from chapter 1 is the same shape you use for the first golden set of every new feature. The specialist track scales it up — segment-stratified sampling, active-learning-driven refresh, multi-annotator labelling — but the shape of "small, defensible, per-row tolerance, versioned" does not change.
- **The three-family scoring model** from chapter 2 is the same three families you use at any scale. What changes is the calibration methodology on the judge family and the addition of RAG-specific metrics; the "rule → similarity → judge" progression stays.
- **The pytest-in-CI shape** from chapter 3 does not go away when you add hosted eval platforms. Fast, gated, per-PR regression checks stay in CI. The hosted platforms cover the *other* runtime — the continuous online one — that CI does not touch.
- **The manifest + baseline + diff shape** from chapter 4 is the primitive shape under every hosted eval platform. Understanding it in raw form makes the platforms make sense; skipping it makes them opaque.
- **Every earlier-module discipline (mod-101 through mod-108) compounds.** Structured JSON output (mod-102) makes rule-based scoring cheap. Prompt caching (mod-108) is what makes the eval affordable. The retrieval-vs-prompt triage (mod-104 chapter 7) becomes a first-class dimension of RAG eval in the specialist track. Nothing you learned in the earlier modules gets un-learned.

If you have done this module honestly, the eval track's job is to add depth — not to un-teach anything.

## Two tools you will meet on the far side of the boundary

You do not need to install any of these for mod-109. They are named here because they are the tools mod-109's exercises deliberately do *not* introduce, so you know what to look up when you graduate:

- **Langfuse, LangSmith, Braintrust, Arize Phoenix, Weights & Biases** — hosted eval + observability platforms. Different feature sets, overlapping models. `ai-eval-engineer-learning` compares them; do not adopt one before you have a golden set that would fit any of them.
- **OpenTelemetry with LLM semantic conventions, Datadog LLM Observability** — provider-neutral tracing. Belongs on the observability side of the eval track.
- **RAGAS, TruLens, promptfoo, DeepEval, Inspect** — evaluation frameworks. Each has opinions about scoring, judge shape, and run storage. Meeting them *after* building a raw pytest eval is much easier than trying to learn them cold — you have the vocabulary to see what each is opinionated about.

Every one of them wraps the primitives from chapters 1–4. Understand the raw shape first (this module); adopt the framework when you know why you need what it is opinionated about.

## Summary

- mod-109 is the *intro* to LLM evaluation. It teaches enough to ship a first golden-set-plus-pytest regression check that gates one LLM feature in CI, tracks history, and routes regressions.
- Everything past that — LLM-as-judge calibration methodology, RAG eval, online eval, observability / tracing (Langfuse, OpenTelemetry, Datadog), cost / latency / quality dashboards, safety / red-team eval, agent-trajectory eval, statistical rigour on eval numbers — lives in [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning). Do not build that depth from scratch here.
- When the feature count grows, when judges multiply, when a product decision hinges on the eval number, or when production signal (not PR signal) is the bottleneck, that is your signal to graduate.
- The patterns you learned here — golden-set shape, tolerance-per-row, three-family scoring, pytest-in-CI, manifest + baseline + diff — carry forward. The eval track adds depth to them; it does not replace them.

You have finished mod-109. The next module (`mod-106-llm-app-deployment`) is where mod-102 through mod-109 all get used together to ship a first end-to-end LLM feature. The eval you built in this module is one of the load-bearing pieces of what mod-106 calls "ready to ship."
