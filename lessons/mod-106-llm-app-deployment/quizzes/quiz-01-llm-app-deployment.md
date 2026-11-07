# Quiz 01 — Shipping an LLM Application

**Covers:** chapters 1–7 of `mod-106-llm-app-deployment`.
**Format:** 12 multiple-choice questions. Pick the single best answer for each. Answer key with per-answer explanations is at the bottom.

Treat this as a self-check: open the chapters when you want to look something up, but try to answer first.

---

## 1. Chapter 1 names concurrency as "the single most surprising property of a service vs. a script." A module-level `THERMOSTAT_LOG = []` that worked fine in your `mod-105` driver suddenly behaves badly once the same code is hosted as a FastAPI service. What does chapter 1 say is the specific concern?

- A. Module-level objects cannot be serialized across the FastAPI/uvicorn boundary, so the list is empty on every request.
- B. The Python GIL serializes writes to the list, so a second request always waits behind the first and latency collapses.
- C. Module-level state is shared across all in-flight requests; what used to be a per-run list is now one process-wide list that leaks and mutates between callers unless you deliberately moved per-request state into the handler.
- D. FastAPI copies module globals into the request scope on each call, so your list resets between requests and you lose the state you expected to accumulate.

## 2. Chapter 2 puts `Field(min_length=1, max_length=4000)` on the `question` field of `ChatRequest`. The chapter is explicit about what `max_length=4000` is buying you at the HTTP boundary. Which best matches the chapter's reasoning?

- A. It keeps the Pydantic schema under the JSON Schema spec's maximum string length, which some clients enforce.
- B. It bounds the input size at the HTTP boundary so a 2 MB body cannot become a free path to a huge token bill, and rejects over-length requests as `422` before the model is ever called.
- C. The provider's tokenizer rejects strings longer than 4000 characters, so Pydantic mirrors the provider's hard limit.
- D. 4000 characters is the longest input FastAPI can validate without falling back to slower streaming parsing.

## 3. Chapter 2 calls out "the single biggest mistake" when writing `/healthz`. According to the chapter, what is it?

- A. Returning plain text instead of JSON, which breaks some orchestrator parsers.
- B. Making the health check do work — calling the model, hitting the database, loading a large file — so a downstream blip takes you down with it and the failure cascade has nothing to do with your code's actual health.
- C. Mounting the route at `/healthz` instead of the conventional `/health`, which some load balancers probe by default.
- D. Returning a `200` status even on unhealthy states; chapter 2 recommends `503` as the default so orchestrators retry.

## 4. Chapter 3 declares `anthropic_api_key: SecretStr = Field(alias="ANTHROPIC_API_KEY")` with no `default=`. Suppose the env var is missing at startup. What does the chapter say should happen — and why is that the correct behavior?

- A. The app should start normally and return `500` on the first `/chat` request so the client learns about the misconfiguration; startup-time failure hides the problem from monitoring.
- B. Pydantic should raise a `ValidationError` and the process should die *before* it serves a single request, because a service that boots fine and only discovers the missing key at request time returns `500` to a real caller and confuses three downstream observers.
- C. `SecretStr` quietly substitutes an empty string so local development isn't blocked; the operator sees the empty-key state in logs and fills it in later.
- D. `pydantic-settings` falls through to any OS environment variable with a similar prefix as a convenience default.

## 5. A teammate adds `ANTROPIC_API_KEY=sk-...` (note the typo) to `.env` and restarts. Chapter 3's `Settings` uses `extra="forbid"`. What does the chapter say happens, and why is that preferred over the default?

- A. The typo is silently ignored and the real `ANTHROPIC_API_KEY` field stays unset — the next `/chat` returns `500` and the developer spends an afternoon diagnosing it.
- B. `extra="forbid"` triggers a `ValidationError` at startup that names the unexpected field, so a typo is loud instead of silent — the alternative (silent acceptance under `extra="allow"`) is "the loudest possible foot-gun."
- C. `pydantic-settings` fuzzy-matches the typo to the nearest known field and loads the value anyway; `extra="forbid"` disables the fuzzy match for safety.
- D. `extra="forbid"` rejects the request the next time `/chat` is called; it has no effect at startup.

## 6. Chapter 4 describes the destination of your log lines. Which pair of claims matches the chapter?

- A. Write to a rotating file under `/var/log/app/` and configure your logger with a `RotatingFileHandler`; the orchestrator mounts the directory and ships from there.
- B. Call the log-aggregation vendor's SDK (Datadog, Honeycomb, Sentry) directly from inside the `/chat` handler so lines are shipped synchronously and never lost.
- C. Write JSON lines to stdout and let the orchestrator (Docker, Kubernetes, systemd) route them — a twelve-factor app "never concerns itself with routing or storage of its output stream."
- D. Log to a local SQLite database so queries like "yesterday's total spend" can run with SQL instead of `jq`.

## 7. Chapter 4 draws a boundary around prompt content. What is the chapter's *default* recommendation for handling the user's raw prompt in your per-request log line?

- A. Log the full prompt and the full model output on every request — you cannot debug bad answers without them.
- B. Log the prompt only when the request is a failure; success cases don't need the text.
- C. Do not log raw prompts or raw model output by default; log a SHA-256 hash (truncated), the character length, and *optionally* a 200-character preview gated on a settings flag like `LOG_PROMPT_PREVIEW=true`.
- D. Log the first and last 200 characters of the prompt by default and the full prompt only in `ENV=dev`.

## 8. Chapter 5 shows the handler calling `guardrail.check_and_charge(0.0)` *before* `run_agent(...)` and a second `guardrail.check_and_charge(cost)` *after*. Why does the chapter say both calls are needed?

- A. Two calls amortize the lock cost across the request, which single-locking cannot.
- B. The pre-check fails the very first request made after the cap is hit (otherwise you serve one extra over-budget request), and the post-charge records the actual estimated cost so the counter is honest for the next request. One without the other misses one of the two cases.
- C. The pre-check is for async requests and the post-charge is for sync requests; FastAPI dispatches to whichever path the handler is declared as.
- D. The second call is defensive — it re-raises `GuardrailTripped` on behalf of the exception handler so the `429` body is emitted twice, once per middleware layer.

## 9. Chapter 5 is explicit about what the in-memory `DailySpendGuardrail` does *not* protect against. Which of the following is NOT one of the limits the chapter names?

- A. Multiple processes or multiple uvicorn workers each hold their own counter, so each thinks it has the full budget.
- B. A crash mid-day resets the in-memory counter; the next request thinks it has the full daily budget again.
- C. A 5× burst in three minutes can drain the entire budget before the next request even arrives, because the hard cap is a total — not a velocity limit.
- D. Provider-side billing lags the counter by 24 hours, so the guardrail always under-counts and the budget tops out 20% over the configured cap.

## 10. Chapter 6 lists several deliberate properties of the multi-stage Dockerfile. Which pairing of property and reason matches the chapter?

- A. Setting `PYTHONDONTWRITEBYTECODE=1` so the container's filesystem can be made read-only; the `.pyc` write would otherwise crash uvicorn.
- B. Setting `PYTHONUNBUFFERED=1` so Python writes log lines to stdout immediately instead of buffering them — without this, `docker logs` shows nothing until the buffer flushes, which breaks the stdout-log contract from chapter 4.
- C. Running the container as `root` so `uvicorn` can bind to the low port 8000 inside the container.
- D. Omitting `HEALTHCHECK` because the orchestrator (Kubernetes, ECS) owns liveness probing and a container-level probe would double-fire.

## 11. Chapter 6 tells you to add `.env` (and `.env.*` except `!.env.example`) to `.dockerignore`. The chapter calls one specific line "the single most important." Which claim best captures the chapter's reasoning?

- A. Without the exclusion, the `COPY . .` step inside the Dockerfile copies `.env` into the image, baking the API key into an artifact that is meant to be shared — "an image is only one accidental `docker push` away from a public registry."
- B. Without the exclusion, Docker's build cache stores `.env` as a separate layer, causing every rebuild to invalidate the dependency layer and slow builds to a minute each.
- C. Without the exclusion, `docker build` refuses to continue because Docker Desktop scans build contexts for secrets and halts on matches.
- D. Without the exclusion, uvicorn reads `.env` twice at runtime (once from the baked file, once from `--env-file`) and the second wins, producing nondeterministic configuration.

## 12. Chapter 7 lists several smoke-test behaviors you should see with your own eyes before shipping. One of them is: *"Startup with a missing required secret."* According to the chapter's checklist, what is the correct observable outcome?

- A. `docker run` starts the container; the first `/chat` request returns `500` with a generic body; the container keeps running so the operator can hit `/healthz` and confirm the process is up.
- B. `docker run` starts the container; `/healthz` returns `200` but `/chat` returns `503` until the secret is injected via the running container's API.
- C. The process *refuses to start*: the Pydantic validation error at startup names the missing field (`ANTHROPIC_API_KEY`), the container exits non-zero, and no request is ever served. `docker run -e ANTHROPIC_API_KEY="" support-agent` is the exact invocation the pre-commit checklist uses to demonstrate this.
- D. The process starts, logs a `WARNING` that the key is missing, and silently routes `/chat` requests to a canned stub response until the key is injected.

---

## Answer key

1. **C.** Chapter 1 ("What changes when concurrency shows up"): *"Module-level state is shared. If you keep a dict at module scope — say, `THERMOSTAT_LOG = []` from mod-105 — every request sees and mutates the same list. That is sometimes what you want (a counter, a cache) and sometimes a bug (a 'current user' that leaks between requests). Be deliberate."* FastAPI does not copy globals per request (D), there is no serialization boundary (A), and the GIL claim (B) is unrelated.

2. **B.** Chapter 2 ("Pydantic request models, in practice"): *"`max_length=4000` enforces an upper bound on input size at the HTTP boundary. Without it a 2 MB body is a free path to a huge token bill."* The number is per-app, chosen to be large enough that no real user hits it and small enough that none floods your token bill. No JSON-Schema spec limit (A), no provider-enforced 4000 (C), and no streaming-parse threshold (D) exists in the chapter.

3. **B.** Chapter 2 ("Health checks"): *"The single biggest mistake here is making the health check do work. Do not call the model in `/healthz`. Do not hit the database. Do not load a large file. The health check is supposed to answer 'is this process up' in microseconds. If it depends on a downstream system you don't control, every blip downstream takes you down too — and the failure cascade has nothing to do with the actual health of your code."* The path name (C), content type (A), and status-code policy (D) are not the mistake the chapter names.

4. **B.** Chapter 3 ("Required vs. optional"): *"A field with no `default=` is required. If the env var is missing at startup, Pydantic raises `ValidationError` and the process dies before it serves a single request. That is the correct behavior — a service that boots fine and only finds out the API key is missing on the first request returns `500` to a real caller and confuses three downstream observers."* Chapter 1's "What changes when secrets show up" makes the same claim: a missing secret should be a startup failure, not a runtime crash.

5. **B.** Chapter 3 (`Settings` snippet and surrounding prose): *"`extra='forbid'`. If somebody types `ANTROPIC_API_KEY` (note the typo) in `.env`, the app refuses to start. The alternative — silent acceptance — is the loudest possible foot-gun."* `pydantic-settings` does no fuzzy matching (C); the behavior fires at startup, not at `/chat` time (D).

6. **C.** Chapter 4 ("Where logs go"): *"In this module, logs go to stdout. That is correct and also the entire production answer... A twelve-factor app treats logs as event streams... A twelve-factor app never concerns itself with routing or storage of its output stream."* Chapter 4 explicitly argues against vendor SDKs in the handler (B) and files (A, D); routing is the orchestrator's job.

7. **C.** Chapter 4 ("The prompt-and-PII boundary"): *"A useful default for this module: do not log raw prompts or raw model output by default. Log a hash, a length, and a truncated preview if the env flag says so."* The chapter's `prompt_metadata` helper produces `prompt_sha256_16`, `prompt_chars`, and optionally `prompt_preview` gated on a flag; the hash lets you recognize a specific prompt without storing the text.

8. **B.** Chapter 5 ("Where to call it"): *"Two checks because the order matters: the pre-check fails the very first request after the cap is hit; the post-charge keeps the counter honest. If you only post-charge, you serve one extra request after the cap; if you only pre-check, the counter never moves."* The async/sync claim (C), amortization (A), and double-emission (D) are not in the chapter.

9. **D.** Chapter 5 ("What this doesn't protect against") enumerates exactly four: multiple processes (A), process restarts (B), burstiness — "A 5× spike in three minutes can drain the entire budget before the next request asks for it" (C), and adversarial input. A systematic 24-hour billing lag is not one of the chapter's named gaps; the `estimated_cost_usd` field is explicitly called out as an *estimate*, not a reconciled invoice, which is a different concern.

10. **B.** Chapter 6 ("The multi-stage Dockerfile"): *"`PYTHONUNBUFFERED=1`. Python writes log lines to stdout immediately instead of buffering them. Required for any container that ships logs by tailing stdout. Without this, `docker logs` shows nothing until the buffer flushes."* The chapter is explicit that the runtime image runs as a *non-root* user (so C is wrong), that `HEALTHCHECK` *is* wired up in the Dockerfile (so D is wrong), and that `PYTHONDONTWRITEBYTECODE=1` is for image size, not read-only filesystems (A).

11. **A.** Chapter 6 ("The `.dockerignore`"): *"`.env` is excluded. This is the single most important line. A `COPY . .` that includes `.env` bakes your API key into the image — and an image is meant to be shared. Excluding `.env` is non-negotiable."* Chapter 1 reinforces this: *"A Docker image built once and shared is a public-ish artifact even when it's private. Bake your key into the image and it's only one accidental `docker push` away from a public registry."* No cache-layer (B), build-time scan (C), or uvicorn double-read (D) mechanism is claimed by the chapter.

12. **C.** Chapter 7 ("Things you should test by hand at least once"): *"Startup with a missing required secret. Comment out `ANTHROPIC_API_KEY` in `.env`, restart. The process refuses to start; the validation error names the missing field."* The pre-commit checklist at the end of chapter 7 states it even more mechanically: *"`docker run -e ANTHROPIC_API_KEY='' support-agent` fails to start (missing required setting)."* Chapter 3's required-vs-optional rule is what makes this outcome correct — serving `/healthz` while `/chat` would fail (A, B) or silently stubbing the handler (D) hides the misconfiguration until a real user hits it.
