# Lab 03 — Walk a Circuit Breaker Through a Scripted Event Timeline

**Estimated duration:** 25–40 minutes
**Covers:** [chapter 5 — Status pages, SLOs, and graceful degradation](../05-status-pages-slos-and-graceful-degradation.md) ("The health-check pattern")

## Objective

Chapter 5 names the state machine every LLM feature needs to survive a multi-minute provider outage: **closed → open → probe → closed** (or back to open). It describes the transitions in prose. In this lab you will build the state machine itself — nothing else. No HTTP, no provider SDK, no fallback path, no fault-injecting proxy. Just a `CircuitBreaker` class and a driver that walks it through a **scripted event timeline** (success / failure / wait / probe) and prints the state at every tick. You feel the state transitions directly, the way chapter 5 describes them, before the exercise bundles them into a full fallback-and-metrics system.

By the end you will have:

- A `circuit_breaker.py` with a hand-written state machine.
- A `walk_breaker.py` driver that applies a fixed sequence of events to the breaker and prints the state each tick.
- An output that visually matches chapter 5's four rules: `failure_threshold` opens it, `cooldown_seconds` elapses before probing, a successful probe closes it, a failed probe re-opens it.
- A `NOTES.md` recording which event line in your timeline moved the breaker, and one sentence on why.

> This lab complements [exercise-05](../exercises/exercise-05-provider-outage-graceful-degradation.md), which builds the full outage story: fault-injecting proxy, circuit-breaker wired to a real provider SDK, fallback path, four end-to-end scenarios, and `served_by` / `degraded` / `breaker_state` metrics. Here we strip the whole thing down to just the state machine, so the transitions are the only thing you have to reason about. Do this lab, then do the exercise — the breaker will not surprise you in the integration tests.

## Prerequisites

- Finished chapter 5.
- Python 3.10+. No API keys, no network, no dependencies.

## Step 1 — Write the state machine

Chapter 5's "The health-check pattern" lists four rules verbatim. Translate them one-to-one into methods.

Save as `circuit_breaker.py`:

```python
from dataclasses import dataclass, field


@dataclass
class CircuitBreaker:
    """
    Chapter 5's "health-check pattern" as a tiny state machine. Three states:

      - closed: calls pass through to primary.
      - open:   calls skip primary and go straight to fallback.
      - probe:  the cooldown has elapsed and the *next* call is a probe;
                success closes, failure re-opens.

    Time is injected as a monotonic float rather than read from the clock.
    This keeps the lab's event timeline reproducible and lets the breaker's
    unit tests (exercise 05) run deterministically.
    """

    failure_threshold: int
    cooldown_seconds: float

    state: str = "closed"             # "closed" | "open" | "probe"
    consecutive_failures: int = 0
    opened_at: float = 0.0

    def is_open(self, now: float) -> bool:
        """
        Called before every request. Returns True ⇒ skip primary, use fallback.

        If we were "open" but the cooldown has elapsed, transition to "probe"
        so the *next* record_* call treats this request as the probe.
        """
        if self.state == "open" and (now - self.opened_at) >= self.cooldown_seconds:
            self.state = "probe"
        return self.state == "open"   # "probe" returns False → caller will try primary

    def record_success(self, now: float) -> None:
        """
        A primary call succeeded. In every non-closed state this closes the
        breaker and resets the failure counter.
        """
        self.state = "closed"
        self.consecutive_failures = 0

    def record_failure(self, now: float) -> None:
        """
        A primary call failed (or timed out → chapter 5 says timeouts count
        as failures for breaker purposes).

        - In "closed": increment counter; open if we hit the threshold.
        - In "probe":  the probe failed; re-open and restart the cooldown.
        """
        if self.state == "probe":
            self.state = "open"
            self.opened_at = now
            # consecutive_failures stays as-is; the breaker is already open.
            return

        self.consecutive_failures += 1
        if self.consecutive_failures >= self.failure_threshold:
            self.state = "open"
            self.opened_at = now
```

Three choices worth pointing at:

- **`now` is an argument, not `time.monotonic()`.** Chapter 5 does not say this, but it is a lab-and-test habit mod-107 chapter 4 already pushed: never make a state machine read the wall clock, or you cannot reproduce it. The caller injects monotonic time.
- **`is_open` has a side effect — it transitions `open → probe` when the cooldown has elapsed.** This matches chapter 5's rule: "After the cooldown, probe with a single request." The next primary call *is* the probe.
- **A failed probe re-opens the breaker and resets `opened_at`.** Chapter 5: "If it fails, extend the cooldown." We do not bump the failure counter — the breaker is already open; the counter is only consulted in the `closed` state.

### Checkpoint

- Can you point to the lines in `CircuitBreaker` that implement each of chapter 5's four transitions (consecutive failures → open; cooldown elapses → probe; probe success → closed; probe failure → open)? If any transition requires reading two fields at once to understand, simplify.
- The class has no mention of "primary call" or "fallback." It is a *state* machine, not a *routing* machine. The routing (`if breaker.is_open(): use_fallback()`) happens in the exercise's wrapper function — chapter 5's "Requirements — the wiring" snippet.

## Step 2 — Write the scripted timeline driver

The driver is a loop over a list of `(timestamp, event)` tuples. The event is one of:

- `"call_ok"` — a primary call succeeded.
- `"call_fail"` — a primary call failed (error or timeout — breaker does not care which).
- `"check"` — no call is made; just read `is_open` so we can observe state transitions over time (e.g., to catch the `open → probe` transition when the cooldown elapses).

Save as `walk_breaker.py`:

```python
from circuit_breaker import CircuitBreaker

# A scripted timeline. Times are seconds on a monotonic clock, relative to t=0.
# Chapter 5 says failure_threshold ~ a handful and cooldown ~ tens of seconds.
# We use small numbers so the timeline fits in a page of output.
TIMELINE: list[tuple[float, str]] = [
    # --- primary is healthy ---
    ( 0.0, "call_ok"),
    ( 1.0, "call_ok"),
    ( 2.0, "call_ok"),

    # --- primary starts failing; breaker should open on the 3rd consecutive failure ---
    ( 3.0, "call_fail"),      # 1st failure (counter 1, still closed)
    ( 4.0, "call_fail"),      # 2nd failure (counter 2, still closed)
    ( 5.0, "call_fail"),      # 3rd failure → opens the breaker
    ( 6.0, "call_fail"),      # breaker is open; this is a fallback-served call (skip primary)

    # --- a "check" lets us watch time pass without a call ---
    (15.0, "check"),          # still inside cooldown (cooldown = 20)
    (25.0, "check"),          # cooldown has elapsed → breaker should be in "probe"

    # --- probe fires; the next primary call IS the probe ---
    (26.0, "call_fail"),      # probe fails → back to open, cooldown restarts at t=26

    # --- another cooldown wait, another probe, this time the probe succeeds ---
    (47.0, "check"),          # cooldown elapsed (26 + 20 = 46)
    (48.0, "call_ok"),        # probe succeeds → closed, counter reset

    # --- back to normal traffic ---
    (49.0, "call_ok"),
    (50.0, "call_ok"),
]


def main() -> None:
    breaker = CircuitBreaker(failure_threshold=3, cooldown_seconds=20.0)

    print(f"{'t':>5}  {'event':<10}  {'state before':<14}  "
          f"{'action':<16}  {'state after':<12}  {'conseq fails':>12}")
    print("-" * 80)

    for t, event in TIMELINE:
        state_before = breaker.state
        # Every tick, first ask the breaker whether primary is reachable.
        # Doing this on a "check" tick is what captures the open→probe transition.
        primary_blocked = breaker.is_open(now=t)
        state_mid = breaker.state

        if event == "check":
            action = "check (no call)"
        elif primary_blocked:
            # Breaker is open → we would route to fallback. Breaker state does
            # not change (fallback successes/failures do not drive the breaker).
            action = "fallback (skipped)"
        elif event == "call_ok":
            breaker.record_success(now=t)
            action = "primary ok"
        elif event == "call_fail":
            breaker.record_failure(now=t)
            action = "primary FAIL"
        else:
            raise ValueError(f"unknown event: {event}")

        print(
            f"{t:>5.1f}  {event:<10}  {state_before:<14}  "
            f"{action:<16}  {breaker.state:<12}  {breaker.consecutive_failures:>12}"
        )


if __name__ == "__main__":
    main()
```

### Checkpoint

- The timeline is seeded with exactly four state transitions: `closed → open` at t=5, `open → probe` at t=25, `probe → open` at t=26, and `probe → closed` at t=48. Trace each by hand *before* you run the script and write the four `(t, from, to)` tuples in `NOTES.md`.
- `state_before` and `state after` in the table can differ because `is_open(now=t)` has the side-effect of promoting `open → probe` when the cooldown elapses. That is intentional — chapter 5's "After the cooldown, probe with a single request" puts the promotion on the same request that would have gone to the primary. Make sure you can see where in the driver that happens.

## Step 3 — Run it and read the state transitions

```bash
python walk_breaker.py
```

### Expected output (shape)

A 14-row table that walks the breaker through its full state cycle. The important rows:

- Row at `t=5.0`: `state after = open`. This is the third consecutive failure — the `failure_threshold` trip.
- Row at `t=6.0`: `state before = open`, `action = fallback (skipped)`. The scripted "call_fail" never reached the primary; the breaker short-circuited to the fallback.
- Row at `t=25.0`: `event = check`, `state after = probe`. The cooldown (20 s since `opened_at = 5.0`) elapsed; `is_open(now=25.0)` promoted the breaker to `probe`.
- Row at `t=26.0`: `state before = probe`, `event = call_fail`, `state after = open`. The probe failed; back to open, with `opened_at` reset to 26.0.
- Row at `t=48.0`: `event = call_ok`, `state before = probe` (promoted on this tick's `is_open` call), `state after = closed`. The second probe succeeded; the breaker closes and traffic flows to primary again.

```
    t  event       state before    action            state after   conseq fails
--------------------------------------------------------------------------------
  0.0  call_ok     closed          primary ok        closed                   0
  1.0  call_ok     closed          primary ok        closed                   0
  2.0  call_ok     closed          primary ok        closed                   0
  3.0  call_fail   closed          primary FAIL      closed                   1
  4.0  call_fail   closed          primary FAIL      closed                   2
  5.0  call_fail   closed          primary FAIL      open                     3
  6.0  call_fail   open            fallback (skipped)open                     3
 15.0  check       open            check (no call)   open                     3
 25.0  check       open            check (no call)   probe                    3
 26.0  call_fail   probe           primary FAIL      open                     3
 47.0  check       open            check (no call)   probe                    3
 48.0  call_ok     probe           primary ok        closed                   0
 49.0  call_ok     closed          primary ok        closed                   0
 50.0  call_ok     closed          primary ok        closed                   0
```

### Checkpoint

Five pointable rows match chapter 5's four transitions plus the fallback short-circuit. In `NOTES.md`, write the row number and the chapter rule for each:

1. "After `failure_threshold` consecutive failures → open." → row at t=5.0.
2. "While open, requests skip primary → go to fallback without touching it." → row at t=6.0.
3. "After `cooldown_seconds` → probe." → row at t=25.0 (check tick), confirmed by t=47.0.
4. "Probe failure → re-open; restart cooldown." → row at t=26.0.
5. "Probe success → close; reset failure counter." → row at t=48.0.

If any of your five rows do not match, pause and re-read the method they exercise. The point of the lab is to make the five rows obvious in your output.

## Step 4 — Flip one knob and observe

Pick **one** of these and re-run `walk_breaker.py`. Record what changes in `NOTES.md`.

- **Tighter threshold: `failure_threshold=2`.** The breaker opens one tick earlier (t=4.0). Row at t=4.0 flips from `closed` to `open`; row at t=5.0 becomes a fallback-served call. One unit test to run in your head: does the counter resetting on any success (`record_success`) still work?
- **Longer cooldown: `cooldown_seconds=50`.** The first probe attempt at t=25.0 is still inside the cooldown — the row stays `open`, no `probe` state. Confirm: when does the first probe fire? (Answer: no earlier than t=55.0; your timeline has no event there, so the breaker stays open to the end.)
- **Insert a success mid-outage.** Add a `(4.5, "call_ok")` between the second and third failures. The counter resets to 0 at t=4.5; the third `call_fail` at t=5.0 only brings it back to 1, not 3; the breaker never opens. This is chapter 5's "Reset failure count on any success" rule — easy to lose if you refactor.
- **Fallback event that fails.** Add a parallel "fallback_fail" event between t=6.0 and t=15.0. It should have **no effect on the breaker's state**. The breaker tracks the *primary's* health, not the fallback's — chapter 5's "The response indicates whether it was degraded" distinction, pulled apart from the breaker.

## Step 5 — Write a 5-sentence debrief

In `NOTES.md`, answer in 5–7 sentences total:

1. State chapter 5's four transition rules in your own words. Which rule did Step 4's knob exercise, and what row in the output proved it?
2. Chapter 5 says the breaker is the companion to mod-107 chapter 4's retry policy — one handles sub-minute transients, the other handles multi-minute outages. In your timeline, `failure_threshold=3` and `cooldown=20s`. Where in a real stack would the retry policy sit — before or after the breaker? (Hint: think about what happens to a request that succeeds on retry attempt 2.)
3. A teammate proposes recording **fallback** successes and failures on the breaker too, so that "if the fallback is also failing we don't keep routing to it." Reread chapter 5's shape list — is that what the breaker is for, or is it a different concept entirely? Name the chapter-5 shape (1–5) that proposal is really reaching for.
4. If you were to add a fifth state, `half_open_rate_limited` (admit *some* traffic to probe, not just one request), which existing method would you have to change? This is one of the common refinements real-world breakers add; writing down what breaks is how you decide whether it is worth the complexity.

That write-up is the deliverable. Commit `circuit_breaker.py`, `walk_breaker.py`, and `NOTES.md`.

## Debrief

Three habits pay forward from here:

- A circuit breaker is a state machine. It has three states and five transitions. If you cannot draw them on a napkin, you cannot operate the feature during an incident — and you certainly cannot justify a refinement like "half-open with a rate limit." Start with the napkin; write the class second.
- Time is an input, not a global. Any method that calls `time.monotonic()` for you is one you cannot unit-test at the scale chapter 5 demands. Inject `now`.
- The breaker is **only** about the primary's health. Fallbacks, templates, "we'll email you later," batch-queue degradation — those are *routing* decisions that happen around the breaker's output. Chapter 5's five degradation shapes live in the wrapper function; the breaker just answers "is the primary healthy enough to try?"

The exercise next door takes exactly this `CircuitBreaker`, wires it to a fault-injecting proxy and a real provider SDK, adds a fallback path, and runs the four integration scenarios chapter 5 specifies. The state-machine logic you just stepped through will pass through that exercise untouched — the lab's point is that it *should*.

## Stretch

- **Unit tests for every transition.** For each of the five pointable rows, write a `pytest` test that drives the breaker to that transition with a sequence of `record_success` / `record_failure` / `is_open` calls (not with the full timeline). Five tests, each under 10 lines. This is the harness exercise 05 will ask for.
- **Monotonic-clock integration.** Replace `now=t` with `now=time.monotonic()` in the driver, keep the breaker's method signature, and sleep between events. Confirm the behaviour is identical. This is the one line of adapter code that separates the lab from a real deployment.
- **Per-model breaker, not per-provider.** Chapter 5's stretch for exercise 05 calls this out — if Opus is degraded but Haiku is fine, the breaker should be scoped to the model, not the provider. Extend `CircuitBreaker` to carry a `scope: str` field and build a `BreakerRegistry` that looks up (or creates) a breaker by scope. Walk the timeline twice — once for Opus, once for Haiku — and confirm the two states are independent.
- **Export the breaker state to a metric.** Chapter 5's "Alarming" section lists "escalation / fallback rate" as one of the four alarms. Add a `transitions: list[tuple[float, str, str]]` field that records every state change — `(t, from_state, to_state)` — and print it at the end of the driver. In production this is what you would export to a time-series database and alarm on.
