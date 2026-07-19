# Decision Guide: Which Tool/Pattern, and When

Most async mistakes aren't "wrong syntax" — they're the right primitive applied at the wrong layer (in-process where it needed to be cluster-wide, or vice versa). Use this file when you're choosing *between* approaches, not just implementing one you've already picked. Each row points back to the reference file with the full explanation.

## 1. Concurrency control — the single question that matters

Before picking a primitive, ask: **"If this runs on N replicas at once, does this variable still mean what I think it means?"**

| You need to... | Use | Not | Why |
|---|---|---|---|
| Cap concurrent DB/CPU-heavy work in one process | `asyncio.Semaphore(n)`, sized as `total_budget / replica_count` | A hardcoded guess | A per-process semaphore multiplies by replica count — see `distributed-scaling.md` §1 |
| Cap total concurrent work across the whole fleet | Redis-backed semaphore, or push the limit into the queue consumer's `prefetch_count` | An `asyncio.Semaphore` | In-process objects have no visibility into other pods — `queues-and-workers.md` §2 |
| Ensure only one instance runs a given job | Redis distributed lock **with a TTL** | `asyncio.Lock` | A local lock only blocks within one process; a lock without TTL deadlocks forever if the holder crashes — `distributed-scaling.md` §2 |
| Cap requests/sec to a rate-limited API from one process | `aiolimiter.AsyncLimiter` (token bucket) | A semaphore | Concurrency limit ≠ rate limit — a semaphore says "how many at once," not "how fast" — `resilience-patterns.md` §1 |
| Cap requests/sec to that API across the whole fleet | Redis-backed token bucket / `limits` library | A per-process `AsyncLimiter` | 15 pods × 10 req/s each is still 150 req/s to the vendor — `resilience-patterns.md` §2 |
| Stop unbounded memory growth when producers outpace consumers | `asyncio.Queue(maxsize=...)` + reject/`429` when full | An unbounded queue "for now" | Unbounded in-memory queues under sustained overload are a slow-motion OOM — `resilience-patterns.md` §5 |
| Bound concurrency when jobs vary wildly in RAM/CPU cost (20 light jobs fine, 1 heavy job maxes out the pod) | Separate pools/semaphores per cost class (bulkhead), or a weighted semaphore keyed by estimated cost | One flat `asyncio.Semaphore(n)` sized for the average job | A count-based limit doesn't know a "slot" can cost 20x more depending on what's in it — `variable-cost-workloads.md` |

## 2. Background work — BackgroundTasks vs. a real queue vs. create_task

| Situation | Use |
|---|---|
| Small, fire-and-forget, OK to lose if the process dies (confirmation email) | FastAPI `BackgroundTasks` |
| Must survive the pod/instance dying, needs retries, or takes more than a few seconds | A real task queue (SQS, RabbitMQ, Celery, arq) + worker |
| Tracked concurrent work within the same request/task lifecycle that you'll await before returning | `asyncio.TaskGroup` or a `create_task` you keep a reference to and join |
| Anything you're tempted to `asyncio.create_task(...)` and never await or track | Don't. If it must complete, await it (bounded by a timeout); if it must survive a restart, put it on a queue |

Full reasoning: `fastapi-patterns.md` §4, `asyncio-fundamentals.md` §4, `distributed-scaling.md` §7 (Lambda freeze makes this especially sharp).

## 3. `def` vs `async def` in FastAPI

| Your stack is... | Use |
|---|---|
| Fully sync (blocking drivers/libraries you can't replace) | Plain `def` routes — FastAPI threads them automatically |
| Fully async (async DB driver, `httpx.AsyncClient`, etc., end to end) | `async def` routes |
| A mix, in the same route or its dependencies | Never — pick one lane and stay in it (`fastapi-patterns.md` §1-2) |

## 4. Queue technology — ordering, throughput, and fan-out

| You need... | Use |
|---|---|
| Max throughput, order doesn't matter | SQS standard, or RabbitMQ with multiple consumers |
| Strict per-entity ordering (events for the same user must apply in sequence) | SQS FIFO with a `MessageGroupId` per entity, or a consistent-hash queue split |
| Pub/sub broadcast to many listeners (e.g., websocket fan-out across pods) | Redis pub/sub or NATS — a work queue is the wrong shape (each message should be consumed once, not fanned out) |

Full reasoning: `queues-and-workers.md` §6, `distributed-scaling.md` §6.

## 5. Resilience layering — retry, circuit breaker, backpressure, timeout

These aren't alternatives — they answer different questions, and a mature call site usually needs more than one:

| Question | Answer |
|---|---|
| "The dependency fails occasionally but is generally healthy" | Retry with exponential backoff + jitter |
| "The dependency is down or failing consistently" | Circuit breaker — stop hammering it, fail fast, retry occasionally to test recovery |
| "I'm the one accepting more work than I can handle" | Backpressure — bounded queues, reject early (`429`) rather than queue unboundedly |
| "Any external call, regardless of the above" | Explicit timeout, always — a missing timeout undermines every other pattern in this table by holding a slot open indefinitely |

Full reasoning and code: `resilience-patterns.md` §3-6.

## 6. Quick gut-check before choosing anything

1. Does this decision only need to make sense for one process, or for the whole fleet? If you're not sure, assume the fleet.
2. If the answer is "the whole fleet," does the state backing it (semaphore count, lock, rate limit, dedup check) live somewhere shared (Redis, the DB, the queue broker) — or only in this process's memory?
3. If it only lives in this process's memory, it is a single-instance-only guarantee. That's fine for things that are genuinely single-instance-scoped (e.g., "don't open more than 20 DB connections from *this* pod") — it is not fine for things framed as global ("don't call this API more than 100 times/sec, period").
