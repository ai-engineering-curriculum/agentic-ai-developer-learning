# Lab 02 — Watch the Thundering Herd (and Then Break It with Jitter)

**Estimated duration:** 45–60 minutes
**Covers:** [chapter 4 — Retrying transient failures without amplifying real bugs](../04-retrying-transient-failures.md) ("Motivation" → thundering herd, and "Exponential backoff plus jitter")

## Objective

Chapter 4 claims that **full jitter** — picking the next retry delay uniformly between 0 and the current exponential cap — is what prevents a crowd of clients from re-hitting the provider in a synchronised burst after a transient failure. In this lab you will see the synchronisation happen on your own machine: run many asyncio workers against a local always-429 fake server, once with exponential-only delays and once with full jitter, and plot the retry arrivals per second. The two histograms tell the whole story.

No real provider. No tokens spent. One contrast; one lesson.

By the end you will have:

- A local `fake_429.py` server that returns HTTP 429 to every request.
- A `herd.py` client that fans out 50 workers, each attempting the same request with up to 4 retries, and logs every attempt's timestamp.
- Two CSV outputs — `retries_no_jitter.csv` and `retries_full_jitter.csv` — and a short script that bins retry timestamps into 100-ms buckets so you can see the arrival pattern.
- A one-paragraph note comparing the two distributions.

> This lab complements [exercise-04](../exercises/exercise-04-retry-with-backoff-and-jitter.md), which builds the full `retry_with_backoff` helper and tests it against a scriptable fake server per-request. Here we isolate the one question "does jitter actually break synchronisation across concurrent clients?" and answer it with a single contrast — nothing else. This lab deliberately uses a stripped-down retry loop, not a production-grade helper; the exercise is where you build the real one.

## Prerequisites

- Finished chapter 4.
- Python 3.11+ on your path (for `asyncio.TaskGroup`).
- `pip install fastapi uvicorn httpx` in an activated venv.
- No API keys required — everything is local.

## Step 1 — Stand up an always-429 fake server

Save as `fake_429.py`:

```python
from fastapi import FastAPI, Response

app = FastAPI()

@app.post("/v1/echo")
def echo():
    # Every single request returns 429. No Retry-After header — the client
    # must fall back to its own backoff formula.
    return Response(
        content='{"type":"rate_limit_error","message":"too many requests"}',
        status_code=429,
        media_type="application/json",
    )
```

Start it in one terminal, bound to a fixed port:

```bash
uvicorn fake_429:app --port 8765 --log-level warning
```

Confirm it is up from a second terminal:

```bash
curl -sS -i -X POST http://127.0.0.1:8765/v1/echo
```

### Expected output (shape)

The curl should respond with `HTTP/1.1 429 Too Many Requests`. No `Retry-After` header — the client has to pick a delay on its own.

### Checkpoint

- Server responds within a few milliseconds. (If it does not, you are hitting the real internet somehow; fix the port/URL before continuing.)
- The server never returns 200. The lab's whole point is that *every* retry fails, so you can see the pure retry-timing pattern with no confounder.

## Step 2 — Write the two retry variants

Save as `herd.py`. The two coroutines differ by **one line**: the delay formula. That is the single variable you want to isolate.

```python
import asyncio, csv, random, sys, time
import httpx

URL = "http://127.0.0.1:8765/v1/echo"
N_WORKERS = 50
MAX_ATTEMPTS = 4
BASE = 1.0   # seconds
CAP  = 30.0  # seconds

async def attempt_and_log(client, writer, worker_id, variant):
    """Attempt the request MAX_ATTEMPTS times; log every attempt's timestamp.

    All workers start at wall time ~0 so they synchronise their first 429.
    """
    for attempt in range(MAX_ATTEMPTS):
        t_sent = time.perf_counter()
        try:
            r = await client.post(URL, json={})
            status = r.status_code
        except httpx.HTTPError as exc:
            status = f"err:{type(exc).__name__}"
        writer.writerow([worker_id, variant, attempt, f"{t_sent:.4f}", status])

        if attempt == MAX_ATTEMPTS - 1:
            break

        exp_cap = min(CAP, BASE * 2 ** attempt)
        if variant == "no_jitter":
            delay = exp_cap                   # synchronised: every client sleeps the same
        elif variant == "full_jitter":
            delay = random.uniform(0, exp_cap)
        else:
            raise ValueError(variant)
        await asyncio.sleep(delay)

async def run(variant: str, out_path: str):
    async with httpx.AsyncClient(timeout=10.0) as client, \
               open(out_path, "w", newline="") as fh:
        w = csv.writer(fh)
        w.writerow(["worker", "variant", "attempt", "t_sent_s", "status"])

        # Start the whole herd at roughly the same instant.
        start_barrier = asyncio.Event()

        async def worker(worker_id):
            await start_barrier.wait()
            await attempt_and_log(client, w, worker_id, variant)

        async with asyncio.TaskGroup() as tg:
            for i in range(N_WORKERS):
                tg.create_task(worker(i))
            # Give workers a beat to reach the barrier, then release them together.
            await asyncio.sleep(0.05)
            start_barrier.set()

    print(f"wrote {out_path}")

if __name__ == "__main__":
    variant = sys.argv[1]
    out = {"no_jitter": "retries_no_jitter.csv",
           "full_jitter": "retries_full_jitter.csv"}[variant]
    asyncio.run(run(variant, out))
```

Three things this snippet gets right on purpose:

- **One `AsyncClient` shared across workers.** Chapter 3's rule. A per-worker client would skew the comparison with connection-setup time.
- **A barrier to synchronise the first attempt.** The whole point is to start the herd together; `asyncio.Event` is the cheapest way to do that.
- **No real retry policy.** This is a demo loop, not a production helper. Exercise 04 is where you build the production helper.

## Step 3 — Run both variants

With the fake server still running in the first terminal:

```bash
python herd.py no_jitter
python herd.py full_jitter
```

Each run should take at most ~`BASE * (2^0 + 2^1 + 2^2) = 7` seconds in the worst case (no-jitter path waits the full exponential every time) and typically less (jitter path is on average half of each cap).

### Checkpoint

- Both commands exit without errors.
- `retries_no_jitter.csv` and `retries_full_jitter.csv` each contain `50 workers × 4 attempts = 200` rows.
- Every row's `status` is `429`. If you see 200s or 500s, something is wrong on the server side; stop and fix before Step 4.

## Step 4 — Bin the retry arrivals into 100-ms buckets

The whole lab lives or dies on this one view. Save as `bin_arrivals.py`:

```python
import csv, sys
from collections import Counter

path = sys.argv[1]
BUCKET_MS = 100

buckets: Counter[int] = Counter()
rows_by_attempt: dict[int, Counter[int]] = {}

with open(path) as fh:
    reader = csv.DictReader(fh)
    for row in reader:
        attempt = int(row["attempt"])
        t_ms = int(float(row["t_sent_s"]) * 1000)
        bucket = (t_ms // BUCKET_MS) * BUCKET_MS
        buckets[bucket] += 1
        rows_by_attempt.setdefault(attempt, Counter())[bucket] += 1

print(f"{path}: buckets of {BUCKET_MS} ms")
print(f"{'t_ms':>8}  {'total':>6}   histogram")
for bucket in sorted(buckets):
    bar = "#" * buckets[bucket]
    print(f"{bucket:>8}  {buckets[bucket]:>6}   {bar}")

print()
print("By attempt (how synchronised was each round?):")
for attempt in sorted(rows_by_attempt):
    sorted_counts = sorted(rows_by_attempt[attempt].items())
    peak_bucket, peak_count = max(rows_by_attempt[attempt].items(), key=lambda kv: kv[1])
    n_buckets = len(rows_by_attempt[attempt])
    print(f"  attempt {attempt}: spread across {n_buckets:>2} buckets, "
          f"peak = {peak_count} at t={peak_bucket} ms")
```

Run it on both files:

```bash
python bin_arrivals.py retries_no_jitter.csv
python bin_arrivals.py retries_full_jitter.csv
```

### Expected output (shape)

For `retries_no_jitter.csv`, the histogram has four tight spikes — one per attempt — because every worker slept the identical delay:

```
    t_ms   total   histogram
       0      50   ##################################################
    1000      50   ##################################################
    3000      50   ##################################################
    7000      50   ##################################################

By attempt:
  attempt 0: spread across  1 buckets, peak = 50 at t=0 ms
  attempt 1: spread across  1 buckets, peak = 50 at t=1000 ms
  attempt 2: spread across  1 buckets, peak = 50 at t=3000 ms
  attempt 3: spread across  1 buckets, peak = 50 at t=7000 ms
```

For `retries_full_jitter.csv`, the histogram is wider and flatter — every attempt after the first is smeared across the window `[0, exp_cap)`:

```
    t_ms   total   histogram
       0      50   ##################################################
     100       3   ###
     200       6   ######
     ... (attempt 1 smeared across ~0–1000 ms) ...
    1800       4   ####
     ... (attempt 2 smeared across ~1000–3000 ms, attempt 3 across ~3000–7000 ms) ...

By attempt:
  attempt 0: spread across  1 buckets, peak = 50 at t=0 ms
  attempt 1: spread across 10 buckets, peak = 8 at t=400 ms
  attempt 2: spread across 19 buckets, peak = 5 at t=1800 ms
  attempt 3: spread across 36 buckets, peak = 4 at t=4300 ms
```

(Your exact numbers will differ; the *shape* — four spikes vs. a smear — is what matters.)

### Checkpoint

Record in `NOTES.md`:

1. For the no-jitter run, how many distinct buckets did attempt 1 occupy? (Answer should be close to 1 — every worker hit the same second.)
2. For the full-jitter run, how many distinct buckets did attempt 1 occupy? (Answer should be much larger — the retries are spread.)
3. Which run would hurt the provider more on a real outage, and why?

## Step 5 — Write a short interpretation

In `NOTES.md`, answer in 4–6 sentences:

- Looking at your two histograms: what does "thundering herd" mean concretely, in terms of retry arrivals per 100-ms bucket?
- The no-jitter variant waits an average of `~3.5s` (sum of `1 + 2 + 4` for three retries) before giving up. The full-jitter variant waits on average half of each cap, so `~1.75s`. Does that mean full jitter is *cheaper* for the single client? (Think about what "cost" means — total wall-clock, or probability of success.)
- Chapter 4 cites the AWS Builders' Library article on backoff with jitter. What is the single sentence you would quote back to a teammate who argues "just use exponential, jitter is overkill"?

That note is the deliverable. Commit `fake_429.py`, `herd.py`, `bin_arrivals.py`, both CSVs, and `NOTES.md`.

## Debrief

The thundering-herd failure mode is invisible from any single client: your retries look fine; the provider looks overloaded; you shrug and blame the provider. The histogram in Step 4 is the view that makes the invisible visible. Three habits pay forward:

- Any retry formula you deploy should have a jitter term — `random.uniform(0, cap)` is the simplest and the one chapter 4 recommends.
- If you ever need to diagnose a suspected herd, log the retry attempt timestamps and bucket them. The "one tight spike per retry attempt" pattern is the smoking gun.
- Rate-limit control does not live in your retry loop. Chapter 4 is explicit: concurrency caps (chapter 3's semaphore) are the primary defence; retries are for the tail, not for capacity planning.

## Stretch

- **Add a `Retry-After` header.** Modify `fake_429.py` to return `Retry-After: 2` on every response. Modify `herd.py` to honour it on the no-jitter path (replace the fixed sleep with the header value). What does the arrival histogram look like now? Chapter 4's recommendation to "respect `retry-after` when present" interacts with your jitter in a non-obvious way — add a small random offset on top of the header to see the tension.
- **Vary `N_WORKERS`.** Rerun both variants with `N_WORKERS=5`, `50`, `500`. The herd gets visibly worse as N grows; at 500 the no-jitter run will probably saturate your CPU before the provider's rate limiter. (In a real production incident, this is the shape of the retry storm that takes down the service.)
- **Compose with exercise-03's fan-out.** Wire this loop into the `fanout` helper from exercise 03 so each item gets the retry policy applied under a semaphore. Confirm the semaphore bounds in-flight work *across* the retries, not just the first attempt — a common bug is that a retrying worker holds its semaphore slot across all four attempts, serialising the batch.
- **Plot it with matplotlib.** Replace the ASCII histogram with a matplotlib `bar` chart per variant. Chapter 5's "report percentiles, not means" habit applies here too — compute the p99 of per-worker total latency for each variant and note what it tells you.
