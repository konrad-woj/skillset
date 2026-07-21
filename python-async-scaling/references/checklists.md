# Checklists for Everyday Tasks

Pick the checklist matching what you're actually doing right now. Each item links back to the reference file section with the full explanation and code — use these to move fast, not as a substitute for reading the reasoning when something's unfamiliar.

## Writing a new async FastAPI endpoint

- [ ] `def` or `async def` decided deliberately, not by copy-paste — see `decision-guide.md` §3
- [ ] No blocking calls anywhere in the endpoint or in any dependency it uses (`fastapi-patterns.md` §1, §5)
- [ ] HTTP client and DB pool are reused via `lifespan`/app state, not created per-request (`fastapi-patterns.md` §3)
- [ ] Explicit timeout on the handler as a whole, and on every external call inside it (`resilience-patterns.md` §6)
- [ ] If the work could exceed the load balancer/gateway timeout, it enqueues + returns a job id instead of awaiting inline (`queues-and-workers.md` §1)
- [ ] If it calls a rate-limited third-party API, the limiter is cluster-aware once you run more than one replica (`decision-guide.md` §1, `resilience-patterns.md` §1-2)
- [ ] If it's a streaming/websocket/SSE endpoint, it has a max duration or heartbeat so dead connections don't accumulate (`fastapi-patterns.md` §6)

## Adding a queue consumer or background worker

- [ ] Concurrency is bounded by `prefetch_count` or a semaphore sized deliberately — not an unbounded `gather` over a batch (`queues-and-workers.md` §2)
- [ ] Visibility timeout (SQS) or lock TTL is comfortably longer than the expected task duration, or extended via heartbeat for long tasks (`queues-and-workers.md` §3, `distributed-scaling.md` §2)
- [ ] Handler checks "have I already done this?" by a stable job/message id before any side-effecting work, and uses a provider-side idempotency key too where the downstream API supports one (`queues-and-workers.md` §4)
- [ ] A dead-letter queue (or dead-letter exchange) with a max-retry count is configured, and its depth is alerted on (`queues-and-workers.md` §5)
- [ ] If the workflow needs per-entity ordering, messages are routed by a partition/group key (`queues-and-workers.md` §6, `decision-guide.md` §4)
- [ ] Graceful shutdown drains or correctly re-queues in-flight jobs on SIGTERM instead of losing or double-processing them (`distributed-scaling.md` §8)

## Calling a third-party / external API

- [ ] Explicit timeout set on the call (`resilience-patterns.md` §6)
- [ ] Rate limit respected, with a clear answer to "per-instance or cluster-wide?" (`decision-guide.md` §1, `resilience-patterns.md` §1-2)
- [ ] Retries use exponential backoff **and jitter**, and only fire on transient, idempotent-safe failures (`resilience-patterns.md` §3)
- [ ] A circuit breaker or equivalent fallback exists for when the dependency is down for an extended period, rather than retrying into a known outage (`resilience-patterns.md` §4)
- [ ] The client is a reused, pooled instance — not constructed fresh per call (`fastapi-patterns.md` §3)
- [ ] If the call is a write with real-world side effects (charges, emails, external state changes), it carries an idempotency key so a client-side retry can't duplicate it downstream (`queues-and-workers.md` §4)

## Reviewing a PR that touches async code

- [ ] Any blocking call (`requests`, `time.sleep`, a sync DB driver, `open()`) inside an `async def`? (`asyncio-fundamentals.md` §1)
- [ ] Every coroutine actually awaited — no accidental "created but never run" coroutine objects? (`asyncio-fundamentals.md` §2)
- [ ] Concurrent work uses `gather`/`TaskGroup` where it should, with `return_exceptions=True` considered where a partial failure shouldn't cancel siblings? (`asyncio-fundamentals.md` §3)
- [ ] `CancelledError` is re-raised after cleanup, never swallowed by a bare `except Exception`? (`asyncio-fundamentals.md` §5)
- [ ] Any `create_task(...)` keeps a reference and either gets awaited or has a done-callback that checks `.exception()`? (`asyncio-fundamentals.md` §4, §7)
- [ ] Any background task created before the `contextvars` it depends on are set (request id, logging context)? Context is snapshotted at task creation, not live (`asyncio-fundamentals.md` §9)
- [ ] Any async generator wrapping a resource (cursor, file, stream) that relies on GC to clean up on early break instead of explicit `aclose()`? (`asyncio-fundamentals.md` §10)
- [ ] Does any new semaphore, lock, or rate limiter assume single-instance semantics that silently breaks once this runs on more than one replica? This is the single highest-value thing to check in review — it's rarely caught by tests (`decision-guide.md` §1, `distributed-scaling.md` §1-2)
- [ ] Any new distributed lock has a TTL? (`distributed-scaling.md` §2)
- [ ] Any new job/queue handler is idempotent? (`queues-and-workers.md` §4)
- [ ] If job/payload cost varies a lot (some requests are far heavier on RAM/CPU than others), is a single flat count-based semaphore hiding a case where a few heavy items can starve or OOM the pod? (`variable-cost-workloads.md`)
- [ ] Would a test at replica-count > 1 (or two local processes against the same Redis/DB) catch a bug that single-instance testing wouldn't? If the PR only tests against one instance, say so explicitly rather than assuming it's covered (`observability-debugging.md` §5)

## Preparing to scale out (going from one instance to N, or raising N)

- [ ] Every in-process semaphore/lock/limiter re-derived as `total_budget / replica_count`, and made configurable so it moves with the autoscaler (`distributed-scaling.md` §1)
- [ ] DB pool sizing re-checked against `max_connections / max_replicas`, with a connection pooler in front if pod count and DB connections shouldn't be 1:1 (`distributed-scaling.md` §3)
- [ ] Autoscaling metric matches the actual bottleneck — for I/O-heavy async workloads, CPU alone often under-triggers; consider in-flight requests or queue depth (`distributed-scaling.md` §4)
- [ ] Container CPU *limits* reviewed for throttling risk on latency-sensitive services — check `container_cpu_cfs_throttled_seconds_total`, not just raw CPU% (`distributed-scaling.md` §5)
- [ ] Websocket/SSE fan-out has a shared pub/sub layer if broadcasting needs to reach clients connected to other pods (`distributed-scaling.md` §6)
- [ ] Graceful shutdown timeout matches `terminationGracePeriodSeconds` (or the Lambda freeze model) so scale-down doesn't silently drop in-flight work (`distributed-scaling.md` §8)
- [ ] If job cost varies widely, container resource limits are sized for the worst case the concurrency control actually permits to run at once — not the average job (`variable-cost-workloads.md` §4)

## Debugging "it hangs" / "it's slow" / "works on one pod but not at scale"

This one has its own fully worked checklist already — go straight to `observability-debugging.md` §2, and pull in `distributed-scaling.md` §5 if throttling is suspected and `observability-debugging.md` §5 for reproducing it locally.

## Writing tests for async code

- [ ] `asyncio_mode = "auto"` set (or every async test explicitly marked) — an unmarked `async def test_...` can silently "pass" without ever running (`testing-async-code.md` §1)
- [ ] Event loop/fixture scope is function-scoped unless there's a deliberate reason to share it — broader scope leaks state between tests (`testing-async-code.md` §2)
- [ ] Mocked async functions use `AsyncMock`, not a plain `Mock`/`MagicMock` (`testing-async-code.md` §3)
- [ ] Retry/backoff tests patch the sleep instead of actually waiting out real backoff delays (`testing-async-code.md` §4)
- [ ] At least one test asserts cleanup/`finally` code actually runs on cancellation, not just the happy path (`testing-async-code.md` §5)
- [ ] Idempotency is asserted directly — call the handler twice with the same job/message id, assert the side effect happened once (`testing-async-code.md` §6)
- [ ] Every task created inside a test is awaited or cancelled before the test ends, so it can't affect the next test (`testing-async-code.md` §8)
