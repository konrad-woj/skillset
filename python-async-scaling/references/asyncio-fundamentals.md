# Asyncio Fundamentals

Single-process correctness. Read this before writing any `async def` code.

## 1. Never block the event loop

```python
# ❌ Blocks everything sharing this event loop
async def bad():
    time.sleep(5)
    requests.get(url)

# ✅ Use async equivalents, or offload sync work
async def good():
    await asyncio.sleep(5)
    async with httpx.AsyncClient() as client:
        await client.get(url)

# CPU-bound or unavoidable sync code → thread pool
result = await asyncio.to_thread(cpu_heavy_func)
```

## 2. Always await your coroutines

```python
# ❌ Creates a coroutine object, never runs it, no error raised
fetch_data()

# ✅
await fetch_data()
```
Treat "coroutine was never awaited" warnings as bugs, not noise.

## 3. Run concurrent work with `gather` or `TaskGroup`

```python
# ❌ Sequential — defeats the purpose of async
result1 = await fetch(url1)
result2 = await fetch(url2)

# ✅ Concurrent
results = await asyncio.gather(fetch(url1), fetch(url2))

# ✅ Python 3.11+: structured concurrency, cancels siblings on failure
async with asyncio.TaskGroup() as tg:
    tg.create_task(fetch(url1))
    tg.create_task(fetch(url2))
```

`gather(..., return_exceptions=True)` if one failure shouldn't cancel the rest — otherwise a single exception cancels all sibling awaitables.

## 4. Don't fire-and-forget tasks without keeping a reference

```python
# ❌ Task can be garbage-collected mid-run
asyncio.create_task(background_job())

# ✅ Keep a reference (or use TaskGroup)
tasks: set[asyncio.Task] = set()
t = asyncio.create_task(background_job())
tasks.add(t)
t.add_done_callback(tasks.discard)
```

## 5. Handle cancellation and timeouts explicitly

```python
try:
    async with asyncio.timeout(5):
        await slow_operation()
except TimeoutError:
    ...
```

`CancelledError` is a `BaseException`, not `Exception` — don't swallow it with a bare `except Exception`. If you catch it to clean up, re-raise it.

```python
async def handler():
    try:
        await work()
    except asyncio.CancelledError:
        await cleanup()
        raise  # always re-raise
```

## 6. Understand what "concurrent" actually buys you

`asyncio` gives concurrency, not parallelism — one thread, one core. It helps I/O-bound work (network, disk, DB) massively. It does nothing for CPU-bound work; a tight CPU loop inside a coroutine still blocks the whole loop just like `time.sleep` would. Use `asyncio.to_thread` (I/O-adjacent blocking calls) or a `ProcessPoolExecutor` (genuinely CPU-bound) instead.

## 7. Exceptions in tasks are silent unless retrieved

```python
# ❌ Exception raised inside the task is swallowed until GC, then just logged
asyncio.create_task(might_fail())

# ✅ Either await it, or attach a done callback that checks .exception()
def _log_if_failed(task: asyncio.Task):
    if not task.cancelled() and task.exception():
        logger.exception("task failed", exc_info=task.exception())

t = asyncio.create_task(might_fail())
t.add_done_callback(_log_if_failed)
```

## 8. Don't create a new event loop per call

```python
# ❌ In a running async app, this raises or creates a nested/duplicate loop
asyncio.run(something())

# ✅ Just await it — you're already inside a loop
await something()
```
`asyncio.run()` is for the single top-level entry point of a script, never called from inside already-running async code.

## 9. `contextvars` are copied into a task at creation time, not shared live

`contextvars.ContextVar` (used for request IDs, logging context, etc.) looks like it "just works" across coroutines, but the actual rule is: a task captures a **snapshot** of the current `contextvars.Context` the moment it's created. After that:
- The task sees whatever was set *before* it was created.
- Changes the parent makes *after* creating the task are invisible to it (and vice versa) — each has its own copy from that point on.

```python
request_id_var: contextvars.ContextVar[str] = contextvars.ContextVar("request_id", default="-")

async def log_something():
    logging.info("request_id=%s", request_id_var.get())

async def handler_right_order():
    request_id_var.set("abc-123")
    asyncio.create_task(log_something())  # ✅ sees "abc-123" — set before the task existed

async def handler_wrong_order():
    task = asyncio.create_task(log_something())  # snapshot taken here, still default "-"
    request_id_var.set("abc-123")                 # ❌ too late for `task`
    await task
```

⚠️ Mistake: creating one long-lived background task (e.g. at app startup, or a connection kept alive across requests) and expecting each new request's context vars to show up inside it. They won't — its context was frozen once, at creation. For per-request background work, create a new task per request, after setting the relevant vars, rather than reusing one. (`asyncio.to_thread` propagates context into the thread the same way a task does.)

## 10. Async generators need explicit cleanup on early exit

```python
async def stream_rows(conn):
    try:
        async for row in conn.cursor():
            yield row
    finally:
        await conn.close()

# ❌ Breaking out early doesn't run the generator's `finally` promptly —
# cleanup is left to garbage collection, which isn't synchronous under asyncio
async for row in stream_rows(conn):
    if row.id == target:
        break
```
For any async generator wrapping a resource (DB cursor, file handle, HTTP stream), close it explicitly instead of relying on GC:

```python
# ✅ contextlib.aclosing (3.10+) guarantees aclose() runs
from contextlib import aclosing

async with aclosing(stream_rows(conn)) as gen:
    async for row in gen:
        if row.id == target:
            break
```
This matters most for FastAPI `StreamingResponse` bodies and DB-driver async cursors — a client disconnecting mid-stream *is* the early-break case in production, and it's exactly when the leak happens.
