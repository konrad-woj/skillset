---
name: python-async-scaling
description: Best practices, common mistakes, decision guidance, and everyday checklists for async Python, asyncio, and FastAPI — covering both single-instance code and distributed/scaled-out deployments (Kubernetes, Lambda, Azure Container Apps, SQS, RabbitMQ). Use this skill any time you write, review, or debug async Python code, FastAPI endpoints, background workers, or queue consumers — especially when concurrency control, timeouts, locks, rate limiting, throttling, or horizontal scaling are involved. Also use it whenever someone is deciding *between* approaches ("should this be a semaphore or a distributed lock", "BackgroundTasks or a real queue", "retry or circuit breaker") or wants a checklist for a routine task like reviewing a PR, adding a queue consumer, calling a third-party API, or scaling out a service. Trigger even if the user doesn't say "async" explicitly but describes symptoms like: server hangs/stalls, event loop blocked, connection pool exhausted, duplicate job processing, tasks lost on deploy/restart, rate limit errors from a third-party API, "works on one pod but breaks at scale," or "instances handle lots of small requests fine but fall over on one big/heavy one."
---

# Python Async & FastAPI Scaling

Guidance for writing correct, non-blocking async Python and for adapting that code to distributed environments where multiple instances run concurrently.

## Core principle

Single-instance async and distributed async solve different problems:
- **Single instance**: don't block the event loop; maximize concurrency within one process.
- **Distributed (K8s/Lambda/Container Apps + queues)**: don't overload *shared* resources (databases, third-party APIs) across N instances; survive instance death mid-task; make work resumable/idempotent.

A semaphore, lock, or rate limiter that only exists in local process memory does nothing for a fleet of pods. That's the #1 mistake this skill exists to prevent.

## How to use this skill

1. Identify which situation applies and read the matching reference file before writing code:

| Situation | Read |
|---|---|
| Writing/reviewing plain asyncio code (tasks, gather, cancellation, blocking calls) | `references/asyncio-fundamentals.md` |
| Writing/reviewing FastAPI endpoints, dependencies, DB access, HTTP clients | `references/fastapi-patterns.md` |
| Deploying to K8s/Lambda/Container Apps, sizing semaphores/pools across replicas | `references/distributed-scaling.md` |
| Consuming from SQS/RabbitMQ, background workers, long-running jobs | `references/queues-and-workers.md` |
| Rate limiting/throttling external APIs, retries, circuit breakers, backpressure | `references/resilience-patterns.md` |
| Debugging "it hangs" / "it's slow" / blocked event loop / silent failures | `references/observability-debugging.md` |
| Choosing *between* two approaches (which concurrency primitive, which queue tech, retry vs. circuit breaker, `def` vs `async def`) | `references/decision-guide.md` |
| Running through a checklist for a specific everyday task (new endpoint, new consumer, external API call, PR review, scaling change) | `references/checklists.md` |
| Payloads/jobs vary wildly in cost (pod is fine at 20 light calls, struggles with 1 heavy one) — sizing concurrency by weight, not just count | `references/variable-cost-workloads.md` |
| Writing/reviewing tests for async code (pytest-asyncio, mocking coroutines, flaky tests, testing cancellation/idempotency) | `references/testing-async-code.md` |

2. Default to checking **all** relevant files for a task that touches FastAPI + scaling — most real tasks span 2-3 of these files (e.g., a new endpoint that calls a rate-limited third-party API needs `fastapi-patterns.md` + `resilience-patterns.md`).

3. When generating code, always state explicitly which concurrency boundary is being enforced and at what layer (in-process vs. cluster-wide), since that's the detail most often glossed over.

4. If the task maps cleanly onto one of the everyday scenarios below, go straight to its checklist in `references/checklists.md` instead of re-deriving it from the pattern files — it's the same guidance, pre-assembled for that scenario:
   - Writing a new async FastAPI endpoint
   - Adding a queue consumer / background worker
   - Calling a third-party / external API
   - Reviewing a PR that touches async code
   - Preparing to scale out (1 instance → N, or raising N)
   - Debugging "it hangs" / "it's slow" / "works on one pod but not at scale"
   - Writing tests for async code

## Quick mistake checklist (apply before finalizing any async/FastAPI code)

- [ ] Any blocking call (`requests`, `time.sleep`, sync DB driver, `open()`) inside an `async def`? → fix or move to a thread/`def` route.
- [ ] Any `asyncio.Semaphore`/`Lock`/in-memory rate limiter meant to control *cross-instance* behavior? → must move to Redis or the queue layer instead.
- [ ] Any `asyncio.create_task(...)` without keeping a reference or joining it? → risk of silent drop, especially on Lambda freeze or pod SIGTERM.
- [ ] Any distributed lock without a TTL/expiry? → risk of permanent deadlock if the holder crashes.
- [ ] Any queue consumer without a bounded concurrency (`prefetch_count` / semaphore)? → reintroduces the overload problem the queue was meant to solve.
- [ ] Any job handler that isn't idempotent (including a provider-side idempotency key for side-effecting external calls)? → at-least-once delivery + retries will double-process it eventually.
- [ ] Any request handler doing work that could exceed the load balancer/gateway timeout? → should enqueue + return a job id instead of awaiting inline.
- [ ] Graceful shutdown handled (SIGTERM, Lambda freeze) so in-flight async work isn't silently dropped?
- [ ] Any background task created *before* the request-scoped `contextvars` it depends on are set? → it captured a snapshot at creation time, not a live view.
- [ ] Any async generator wrapping a resource (DB cursor, file, stream) relying on GC to run its `finally` block on early exit? → close it explicitly (`aclose()` / `contextlib.aclosing`).

Full explanations and code examples for each item are in the reference files above. This list is the fast, universal pass — for a deeper, scenario-specific version (e.g. "I'm specifically reviewing a PR" or "I'm specifically adding a queue consumer"), use `references/checklists.md`.
