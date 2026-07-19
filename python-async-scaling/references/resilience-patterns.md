# Resilience Patterns (Throttling, Retries, Circuit Breakers, Backpressure)

## 1. Concurrency limit ≠ rate limit

A semaphore bounds *how many* calls run at once; it says nothing about *how fast* they fire. For APIs with a strict calls/sec cap, use a token/leaky bucket:

```python
from aiolimiter import AsyncLimiter

limiter = AsyncLimiter(10, 1)  # 10 requests per second, per instance

async def call_partner_api():
    async with limiter:
        return await client.get(url)
```

## 2. Per-instance limiting isn't enough once you scale out

`AsyncLimiter` above is in-process — 15 pods each limited to 10 req/s still sends 150 req/s to the vendor. For a true cluster-wide cap ("100 req/sec total, no matter how many pods"), the limiter state must live outside the process — Redis + a token-bucket Lua script, or a library like `limits` backed by Redis.

```python
# conceptual shape — actual implementation via redis + lua for atomicity
async def acquire_global_slot(key: str, rate: int, period: int) -> bool:
    allowed = await redis_token_bucket(key, rate, period)
    return allowed
```

## 3. Retries need backoff + jitter, not a fixed delay

```python
import random

async def call_with_retry(fn, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            return await fn()
        except TransientError:
            if attempt == max_attempts - 1:
                raise
            delay = min(2 ** attempt, 30) + random.uniform(0, 1)  # exp backoff + jitter
            await asyncio.sleep(delay)
```

⚠️ Mistake: fixed-delay retries across many instances create synchronized retry storms (thundering herd) — jitter spreads them out. Also: only retry on transient/idempotent-safe failures, never blindly retry a non-idempotent write without a dedup key.

## 4. Circuit breakers stop hammering a failing downstream

Retrying a dependency that's already down just adds load to it and slows your own service. A circuit breaker trips after N consecutive failures, fails fast for a cooldown period, then allows a trial request through (half-open) before fully closing again.

```python
# using a library like `aiobreaker` or `purgatory` conceptually:
async def call_downstream():
    async with breaker:  # raises immediately if circuit is open
        return await client.get(url)
```

Pair this with a sensible fallback (cached response, degraded feature, queued-for-later) rather than surfacing a raw 500 to the user.

## 5. Backpressure: know what happens when you're overwhelmed

Async concurrency makes it easy to *accept* far more in-flight work than downstream systems can handle. Decide deliberately what happens at the limit:
- Reject new work early (`429 Too Many Requests`) once a semaphore/queue is full, instead of queuing unboundedly in memory — an unbounded in-memory queue under sustained overload is a slow-motion OOM.
- Prefer bounded queues (`asyncio.Queue(maxsize=...)`) so producers block or fail explicitly rather than growing memory without limit.

```python
queue: asyncio.Queue = asyncio.Queue(maxsize=1000)

async def producer(item):
    try:
        queue.put_nowait(item)
    except asyncio.QueueFull:
        raise HTTPException(status_code=429, detail="overloaded, try again shortly")
```

## 6. Timeouts belong at every layer that talks to something external

A missing timeout on any single hop (DB call, third-party API, internal service call) can hold a connection/task open indefinitely, which then exhausts whatever pool or semaphore is upstream of it. Set an explicit timeout on every external call, not just the outermost request handler.
