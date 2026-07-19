# Distributed Scaling (Kubernetes, Lambda, Azure Container Apps)

Single-instance concurrency primitives (`asyncio.Semaphore`, `asyncio.Lock`, in-memory rate limiters) only protect one process. Once you run N replicas, every one of those needs a cluster-aware equivalent, or a rethink of where the limit belongs.

## 1. Per-instance concurrency control (Semaphores)

```python
# Limit concurrent DB connections or CPU-heavy work per pod
db_semaphore = asyncio.Semaphore(20)

async def query_db():
    async with db_semaphore:
        return await db.fetch(...)
```

⚠️ Mistake: setting this to 20 thinking it caps total load, while running 15 pods → real limit is 300. Size it as `total_budget / replica_count`, and make it configurable via env var so it moves with your HPA (Horizontal Pod Autoscaler) target.

## 2. Distributed locks for CPU/RAM-heavy or "only one instance" work

```python
import redis.asyncio as redis

r = redis.Redis()

async def process_heavy_job(job_id):
    lock = r.lock(f"job:{job_id}", timeout=300)  # TTL = safety net
    if await lock.acquire(blocking=False):
        try:
            await do_heavy_work(job_id)
        finally:
            await lock.release()
    else:
        return  # another pod already has it
```

⚠️ Mistake: a lock without a TTL. If the pod is evicted or crashes mid-task, the lock never releases and the job is stuck forever. Always set an expiry longer than the expected job duration, and consider a watchdog/heartbeat pattern for jobs that might exceed it.

## 3. Connection pool exhaustion across replicas

A DB (esp. Postgres) has a hard max-connections limit. `pool_size=20` per pod × 30 pods can exceed it instantly during a scale-up event, taking down the database for everyone.

- Size `pool_size` as a function of `max_connections / expected_max_replicas`, with headroom for migrations/admin connections.
- Prefer a connection pooler (PgBouncer, RDS Proxy) between the app and DB so pod count and DB connection count aren't 1:1.
- Set `pool_timeout` so a pod fails fast with a clear error instead of hanging when the pool is exhausted.

## 4. Autoscaling and concurrency settings must agree

If your HPA scales on CPU/memory but each pod can hold 500 concurrent requests via async concurrency, you can get a burst of traffic that never triggers scale-up (CPU stays low because it's I/O-bound) while downstream systems (DB, third-party API) get overwhelmed by aggregate concurrency. Consider scaling on a custom metric (in-flight requests, queue depth) rather than CPU alone for I/O-heavy async workloads.

## 5. Container CPU throttling silently slows the event loop

Kubernetes CPU *limits* (not requests) throttle the container at the cgroup level once it exceeds its quota — this can make an otherwise-fine event loop appear to "hang" under load, because every coroutine on that loop slows down together. Symptoms: latency spikes that don't correlate with visible CPU% in generic dashboards (throttling shows up in `container_cpu_cfs_throttled_seconds_total`, not raw CPU usage). Prefer no CPU limit (only a request) for latency-sensitive async services, or size the limit generously and monitor throttling explicitly.

## 6. WebSockets / SSE and horizontal scaling

Long-lived connections don't fit typical stateless HTTP load balancing well:
- A client's websocket is pinned to one pod for its lifetime — that pod must handle reconnect/backpressure itself.
- Broadcasting to "all connected clients" across pods requires a shared pub/sub layer (Redis pub/sub, NATS), since pod B has no visibility into pod A's connections.
- Rolling deploys will disconnect every websocket on the terminated pod — make sure clients reconnect with backoff, and don't treat a disconnect as data loss if your protocol supports resume.

## 7. Lambda-specific considerations

- **Cold starts**: initializing an `httpx.AsyncClient` or DB pool at import/module time (not inside the handler) reduces work done during cold start, but connections may go stale across freezes — check/reconnect defensively.
- **Frozen execution environment**: after the handler returns, Lambda can freeze the process before background tasks finish. `asyncio.create_task(...)` without awaiting it before returning is unreliable on Lambda — await everything you need to complete before the handler returns, or use a real queue for anything that must survive.
- **Concurrency = instance count**: unlike a long-running FastAPI pod handling many concurrent requests, each Lambda invocation is roughly one execution context; reserved/provisioned concurrency settings are your replica-count-equivalent lever for downstream rate limiting math (see `resilience-patterns.md`).
- **15-minute hard limit**: anything that might approach it belongs in a queue + worker pattern instead (see `queues-and-workers.md`), not a longer Lambda timeout.

## 8. Graceful shutdown (SIGTERM handling)

Pods get SIGTERM before SIGKILL; Lambda freezes/kills after the handler returns. Undrained in-flight async work is lost either way unless handled explicitly.

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    # stop accepting new work, let in-flight finish, bounded by a timeout
    await app.state.queue_consumer.stop()
    await asyncio.wait_for(app.state.active_tasks_done.wait(), timeout=30)
```

Match `terminationGracePeriodSeconds` in your K8s pod spec to this timeout, or work gets killed mid-flight anyway.
