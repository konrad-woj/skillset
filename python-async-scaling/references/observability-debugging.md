# Observability & Debugging Async Systems

## 1. Detecting a blocked event loop

A blocked loop looks like generalized slowness, not a crash — hard to spot without explicit instrumentation.

```python
# Lightweight loop-lag monitor: schedule a callback and measure delay
def monitor_loop_lag(loop, interval=1.0, threshold=0.1):
    last = loop.time()

    def _check():
        nonlocal last
        now = loop.time()
        lag = now - last - interval
        if lag > threshold:
            logger.warning(f"event loop lag: {lag:.3f}s")
        last = now
        loop.call_later(interval, _check)

    loop.call_later(interval, _check)
```
In development, `asyncio.run(main(), debug=True)` (or `PYTHONASYNCIODEBUG=1`) logs coroutines that take too long and warns about non-awaited coroutines.

## 2. "It hangs" checklist

In rough order of likelihood:
1. A blocking call inside `async def` (sync DB driver, `requests`, CPU-bound loop) — see `asyncio-fundamentals.md` §1.
2. A distributed lock held by a dead process with no TTL — see `distributed-scaling.md` §2.
3. Connection pool exhausted (too many in-flight requests for `pool_size`) — check pool wait-time metrics, not just pool size.
4. A missing timeout on a downstream call, holding a semaphore slot indefinitely — see `resilience-patterns.md` §6.
5. CPU throttling at the container level (K8s cgroup limits) making everything on the loop slow, not stuck — see `distributed-scaling.md` §5.

## 3. Silent failures to watch for

- Un-awaited coroutines — no error, just nothing happens. Enable asyncio debug mode to catch these.
- Exceptions inside `asyncio.create_task(...)` that's never awaited or given a done-callback — logged only at garbage collection, easy to miss in production logs. Always attach a done-callback that checks `.exception()`.
- `gather()` without `return_exceptions=True` — one failing coroutine cancels its siblings, and callers sometimes don't realize partial work was silently discarded.

## 4. Metrics worth exposing for async services at scale

- In-flight request/task count (per pod) — catches silent backlog growth before latency does.
- Queue depth and DLQ depth (for queue-based workers) — leading indicator of a stuck or failing consumer.
- Connection pool wait time and utilization (DB, HTTP client) — distinguishes "pool too small" from "downstream too slow."
- `container_cpu_cfs_throttled_seconds_total` (K8s) — surfaces throttling invisible in raw CPU% dashboards.
- Event loop lag (see §1) — the single best proxy for "is this process actually responsive."

## 5. Reproducing distributed bugs locally

Most of the failure modes here (pool exhaustion, lock contention, rate-limit breaches) only appear at N > 1 instances. When debugging locally with a single process, deliberately simulate multiple instances (e.g., run several local workers/processes against the same Redis/DB) rather than trusting single-instance testing to catch them — a fix that "works" against one process is often the exact bug that breaks at replica count > 1.
