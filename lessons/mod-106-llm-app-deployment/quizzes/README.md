# Quizzes for mod-106-llm-app-deployment (Shipping an LLM Application)

Self-check quizzes covering the module chapters. Pick the single best answer for each question; the answer key at the bottom of each file explains the reasoning and points back to the chapter it comes from.

- [quiz-01 — Shipping an LLM Application](quiz-01-llm-app-deployment.md) — 12 multiple-choice questions covering chapters 1–7: the script-to-service shift and shared module state, Pydantic request-body validation and the `/healthz` anti-pattern, `pydantic-settings` with `SecretStr` and `extra="forbid"`, structured JSON logging to stdout and the prompt/PII boundary, the per-request and process-level guardrails (and what the daily cap doesn't protect against), the multi-stage Dockerfile with `.dockerignore` keeping secrets out of the image, and the end-to-end smoke-test behavior on a missing required secret.
