# Chapter 3 — Async fan-out with `asyncio` and `httpx`

Chapters 1 and 2 handled one long-lived stream. This chapter handles *many* concurrent requests: fan out N LLM calls, gather them under a bounded concurrency limit, and cancel the whole fan-out cleanly when the caller goes away. This is the runtime foundation for batch jobs, page-rendering flows that make several LLM calls in parallel, and any server that needs to hand off work to the model without blocking a worker.

## Motivation

An LLM call is almost entirely wall-clock time on the network. Your process is idle for the several seconds it takes the model to produce a response. If you fan out ten calls serially, you wait ten times as long as any single one. If you fan them out concurrently — even from a single Python process — the wall-clock time drops to roughly the slowest single call.

`asyncio` is the right tool for this because LLM I/O is exactly what async runtimes are designed for: many independent, long-lived I/O operations that release the event loop while they wait. `httpx` is the async-capable HTTP client that makes it pleasant.

## Why async over threads for LLM I/O

Threads work too. But async has three concrete advantages for this workload:

- **No thread-per-request overhead.** A single asyncio event loop can supervise hundreds of in-flight requests. A thread pool of hundreds costs meaningful RAM per thread and adds GIL contention on the small CPU-bound moments (JSON parsing, tokenising).
- **Structured concurrency.** `asyncio.TaskGroup` (Python 3.11+) lets you scope a batch of tasks to a block; when the block exits, they are all guaranteed to be done or cancelled. Threads have no analogue you can rely on.
- **Clean cancellation.** `task.cancel()` propagates a `CancelledError` up the awaited chain. If the client disconnects, one call to `task.cancel()` on the whole batch stops every in-flight request. Cancelling a thread is not a well-defined operation.

Use threads when your tool implementations are synchronous and the SDK doesn't offer an async client. For LLM API calls themselves, async is the default.

Reference: `asyncio` overview — <https://docs.python.org/3/library/asyncio.html>. `httpx` async support — <https://www.python-httpx.org/async/>.

## The async provider clients

Both provider SDKs ship async clients that mirror the sync ones. Use them.

```python
import asyncio
from anthropic import AsyncAnthropic
from openai import AsyncOpenAI

anthropic = AsyncAnthropic()
openai = AsyncOpenAI()

async def one_haiku_anthropic(topic: str) -> str:
    resp = await anthropic.messages.create(
        model="claude-opus-4-7",
        max_tokens=128,
        messages=[{"role": "user", "content": f"Haiku about {topic}."}],
    )
    return resp.content[0].text

async def one_haiku_openai(topic: str) -> str:
    resp = await openai.chat.completions.create(
        model="gpt-4.1",
        messages=[{"role": "user", "content": f"Haiku about {topic}."}],
    )
    return resp.choices[0].message.content
```

The `AsyncAnthropic` and `AsyncOpenAI` clients use `httpx.AsyncClient` under the hood, so all of the connection-pool tuning below applies whether you use the SDKs directly or drop down to raw `httpx`.

## Fanning out with `asyncio.gather`

The simplest fan-out is `asyncio.gather`:

```python
async def haikus(topics: list[str]) -> list[str]:
    return await asyncio.gather(*(one_haiku_anthropic(t) for t in topics))

# Later:
results = asyncio.run(haikus(["SSE", "asyncio", "backoff", "percentiles"]))
```

Two things this loses on its own:

- **No concurrency cap.** If `topics` has 500 items, this fires 500 concurrent requests. The provider will rate-limit you or refuse. Your process may run out of sockets first.
- **No fault isolation.** If one coroutine raises, `gather` by default cancels the others. That is sometimes what you want and sometimes not; know which.

For the second: pass `return_exceptions=True` to receive exceptions as list items instead of raising. Use this when partial success is meaningful (batch summarisation) and not when a single failure invalidates the whole batch (multi-step tool orchestration).

## Bounded concurrency with `asyncio.Semaphore`

Providers publish per-minute request and token rate limits, and your own downstream systems (databases, tool APIs) have their own throughput ceilings. A `Semaphore` caps in-flight work to a fixed number.

```python
async def bounded_fanout(topics, concurrency=8):
    sem = asyncio.Semaphore(concurrency)

    async def one(topic):
        async with sem:
            return await one_haiku_anthropic(topic)

    return await asyncio.gather(*(one(t) for t in topics))
```

Guidelines for picking `concurrency`:

- **Start well below the provider's request-per-minute quota.** If your model tier allows 300 RPM and each request averages 3 seconds, sustained concurrency of 15 is roughly the ceiling. Concurrency of 30 will hit 429s.
- **Match it to your slowest downstream.** If each LLM call also hits a database with a 20-connection pool, concurrency > 20 stalls waiting for connections. That looks like a slow LLM but is not.
- **Log the current in-flight count.** If your log shows in-flight stays at the semaphore limit for the whole batch, you are back-pressured — either raise the limit if it is safe or add capacity downstream.

`asyncio.Semaphore` reference: <https://docs.python.org/3/library/asyncio-sync.html#asyncio.Semaphore>.

### `TaskGroup` — structured concurrency with cancellation

If you are on Python 3.11+, prefer `TaskGroup` over `gather` for anything larger than a one-shot batch:

```python
async def bounded_fanout_tg(topics, concurrency=8):
    sem = asyncio.Semaphore(concurrency)
    results: dict[int, str] = {}

    async def one(idx, topic):
        async with sem:
            results[idx] = await one_haiku_anthropic(topic)

    async with asyncio.TaskGroup() as tg:
        for i, topic in enumerate(topics):
            tg.create_task(one(i, topic))

    return [results[i] for i in range(len(topics))]
```

Two guarantees `TaskGroup` gives you that `gather` does not:

- If any task raises, the group cancels the rest and re-raises an `ExceptionGroup` with the original errors preserved.
- The `async with` block does not exit until every task has finished or been cancelled. No task leaks past the block boundary.

Reference: <https://docs.python.org/3/library/asyncio-task.html#task-groups>.

## `httpx.AsyncClient` — reuse the connection pool

If you make more than a handful of async requests, do not create a fresh `httpx.AsyncClient` per call. Create one, share it, and let its connection pool do its job.

```python
import httpx

async def main():
    limits = httpx.Limits(max_connections=32, max_keepalive_connections=16)
    async with httpx.AsyncClient(
        base_url="https://api.anthropic.com",
        headers={
            "x-api-key": os.environ["ANTHROPIC_API_KEY"],
            "anthropic-version": "2023-06-01",
            "content-type": "application/json",
        },
        timeout=httpx.Timeout(60.0, connect=5.0),
        limits=limits,
    ) as client:
        # ... reuse `client` across many concurrent tasks ...
```

Why this matters:

- **TLS + connection setup is expensive.** Every fresh client re-does the handshake. A shared client with keep-alive pays it once and reuses.
- **`Limits(max_connections=...)` is a second, transport-level bound on concurrency.** It backs up requests waiting for a socket instead of opening a new one. Combine with `Semaphore` — the semaphore is your intent, the limits are your safety net.
- **The `async with` block guarantees the pool is closed.** Otherwise you will leak sockets on process exit.

`httpx` limits reference: <https://www.python-httpx.org/advanced/resource-limits/>.

### Timeouts, always

Every HTTP call needs a wall-clock timeout — this is not optional. LLM APIs occasionally hang; a request without a timeout hangs your task forever, and a hung task holds a semaphore slot forever, and now your whole batch is wedged.

`httpx.Timeout(60.0, connect=5.0)` gives you a 5-second connect deadline and a 60-second overall deadline. Pick numbers to match your feature's budget — a chat request may be fine with 30 seconds; a long-form generation may want 120.

If you use the provider SDKs, both accept a `timeout=` argument at the client and per-call level. Set it.

## Cancelling on client disconnect

The point of streaming and fan-out on a server is that you can stop early when the client goes away. In FastAPI this is `request.is_disconnected()` on the incoming request; in raw asyncio it is `task.cancel()` on whatever you spawned.

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from anthropic import AsyncAnthropic

app = FastAPI()
anthropic = AsyncAnthropic()

@app.get("/haiku")
async def haiku(request: Request, topic: str):
    async def event_source():
        async with anthropic.messages.stream(
            model="claude-opus-4-7",
            max_tokens=256,
            messages=[{"role": "user", "content": f"Haiku about {topic}."}],
        ) as stream:
            async for text in stream.text_stream:
                if await request.is_disconnected():
                    # Exit the generator; the `async with` closes the model stream.
                    return
                yield f"data: {text}\n\n"
            yield "event: done\ndata: {}\n\n"

    return StreamingResponse(event_source(), media_type="text/event-stream")
```

<!-- needs-research: verify the exact name of the streaming context manager on the current anthropic-python async client (`anthropic.messages.stream` vs `anthropic.messages.create(stream=True)`) as of 2026-08 — check https://github.com/anthropics/anthropic-sdk-python. -->

Three practical points:

- **Check disconnect between yields.** If you only check once at the start you gain nothing. Do it on every chunk.
- **The `async with` on the model stream is what makes cancellation clean.** When the generator exits, the context manager closes the socket to the model API and stops billing you for the rest of the response.
- **For fan-out, wrap the whole batch in a `TaskGroup` and cancel the group on disconnect.** One `tg.cancel()` propagates to every in-flight request.

FastAPI request disconnection reference: <https://fastapi.tiangolo.com/advanced/using-request-directly/>.

## A worked pattern: bounded, cancellable fan-out with per-item timeouts

Combining the pieces into a small reusable helper:

```python
import asyncio
from contextlib import asynccontextmanager
from typing import Awaitable, Callable, TypeVar

T = TypeVar("T")
R = TypeVar("R")

async def fanout(
    items: list[T],
    fn: Callable[[T], Awaitable[R]],
    *,
    concurrency: int = 8,
    per_item_timeout: float = 30.0,
) -> list[R | BaseException]:
    """Run fn on every item with bounded concurrency and per-item timeout.

    Returns a list aligned with `items`. Exceptions (including TimeoutError)
    are returned in-place rather than raised.
    """
    sem = asyncio.Semaphore(concurrency)
    results: list = [None] * len(items)

    async def one(idx: int, item: T):
        async with sem:
            try:
                results[idx] = await asyncio.wait_for(fn(item), timeout=per_item_timeout)
            except BaseException as exc:
                results[idx] = exc

    async with asyncio.TaskGroup() as tg:
        for i, item in enumerate(items):
            tg.create_task(one(i, item))

    return results
```

Why this shape:

- **Aligned output.** The caller gets a list indexed the same as `items`. Losing that alignment is a very common bug when you `gather(*coros)` without threading indices through.
- **Exceptions in-line, not raised.** Fan-out callers usually want to know which items failed *and* keep the good results. Catch `BaseException` explicitly if you want to include `CancelledError` too; otherwise use `Exception`.
- **`wait_for` gives you the per-item timeout the semaphore alone does not.** A stuck coroutine cannot hold its semaphore slot forever.

The exercise for this chapter turns this helper into the fan-out that all subsequent modules assume you own.

## The `KeyboardInterrupt` / `CancelledError` distinction

Two exceptions look similar during cancellation but mean different things:

- `asyncio.CancelledError` is raised inside a task when someone called `task.cancel()`. It is part of normal shutdown; do not swallow it silently. If you `except BaseException:`, re-raise `CancelledError` after any cleanup.
- `KeyboardInterrupt` is raised on Ctrl-C. `asyncio.run` catches it, cancels the main task, and re-raises after the loop shuts down. Your fan-out should let this propagate.

The correct pattern in cleanup code:

```python
try:
    await something()
except asyncio.CancelledError:
    # do minimal cleanup; do not log-noise this
    raise
except Exception as exc:
    log.exception("something failed: %s", exc)
    raise
```

Swallowing `CancelledError` is the async equivalent of `except Exception: pass` — a bug amplifier that makes shutdown hang.

## Common bugs to prepare for

- **Unbounded fan-out.** `asyncio.gather(*[call() for _ in range(500)])` fires 500 concurrent calls. Cap it with a semaphore.
- **One shared `httpx.AsyncClient` per call.** Creates a new pool each time, defeats keep-alive, exhausts sockets under load. Create once, share across the whole batch.
- **No timeout.** A hung request stalls the semaphore, then the batch, then the process. Set `httpx.Timeout` at the client and `asyncio.wait_for` per item.
- **Blocking calls inside a coroutine.** `time.sleep(1)` inside an async function blocks the entire event loop, not just your task. Use `await asyncio.sleep(1)`. Same for sync HTTP libraries — use `httpx.AsyncClient`, not `requests`.
- **Ignoring `CancelledError`.** Swallowing it prevents structured cancellation from working. Always re-raise.
- **Mixing `asyncio.run` inside an event loop that is already running.** Common inside notebooks and test runners. Use `asyncio.get_event_loop().run_until_complete(...)` or (in a notebook) `await` directly.
- **Fan-out from a sync context by spinning up a new event loop per call.** Convert the whole call site to async, or use `anyio.from_thread.run(...)` — do not repeatedly create and tear down loops.

## Sync fallback: `ThreadPoolExecutor`

If a piece of your workload is genuinely synchronous — a legacy tool, a library with no async surface — do not force it into `async def`. Wrap it in a thread instead:

```python
from concurrent.futures import ThreadPoolExecutor
import asyncio

def sync_tool(x): ...

async def call_sync(x):
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(None, sync_tool, x)
```

`loop.run_in_executor(None, ...)` uses the default thread pool. If you have many such calls, create your own `ThreadPoolExecutor(max_workers=N)` and pass it explicitly so the pool size is legible.

## Summary

- Use the async provider clients (`AsyncAnthropic`, `AsyncOpenAI`) for LLM I/O. Fall back to threads only for genuinely-synchronous tools you cannot rewrite.
- Fan out with `asyncio.gather` for small one-shot batches; prefer `asyncio.TaskGroup` for anything larger — it gives structured concurrency and clean cancellation.
- Bound concurrency with `asyncio.Semaphore` at the intent layer *and* `httpx.Limits(max_connections=...)` at the transport layer. The two together protect you and your provider from unbounded fan-out.
- Reuse one `httpx.AsyncClient` (or one SDK client) across a batch. Its connection pool is the whole point.
- Every request needs a timeout. `httpx.Timeout` at the client, `asyncio.wait_for` per item. A hung request otherwise stalls the whole batch.
- Cancel cleanly on client disconnect: check `request.is_disconnected()` between yields; wrap fan-outs in a `TaskGroup` so one cancel propagates.
- Never swallow `CancelledError`. It is part of correct shutdown, not an error to log-away.

The next chapter is the other half of "concurrent LLM I/O is unreliable I/O": retry policies that turn transient failures into brief hiccups without amplifying real bugs into thundering herds.
