# Variable-Cost Workloads (Heterogeneous Payload Sizes)

Everything elsewhere in this skill about sizing a semaphore (`total_budget / replica_count`) quietly assumes every unit of concurrency costs about the same amount of RAM/CPU. That assumption breaks the moment payload cost varies widely: a pod that comfortably serves 20 small requests at once can be brought to its knees by a single large one. Read this file whenever "light vs. heavy" or "some jobs are 10-100x the cost of others" describes your workload.

## 1. Why count-based concurrency is the wrong tool here

```python
# ❌ Semaphore(20) treats a 50KB request and a 500MB request identically
sem = asyncio.Semaphore(20)

async def handle(payload):
    async with sem:
        return await process(payload)
```
If 20 "light" slots happen to fill with 20 "heavy" jobs instead, you haven't violated the semaphore's contract — you've just discovered its contract never accounted for cost. The pod OOMs or CPU-throttles, and every other in-flight request on that pod (including light, unrelated ones) pays for it. This is a **correctness/availability leak across unrelated requests**, not just a slowdown for the heavy one.

## 2. Bulkhead isolation: separate pools for separate cost classes

The cleanest fix is usually architectural, not a smarter semaphore: don't let heavy and light work compete for the same capacity at all.

- **At the queue layer**: route heavy and light jobs to separate queues (`jobs-light` / `jobs-heavy`) consumed by separate worker deployments. Size each independently — light workers: many replicas, high per-pod concurrency, small memory footprint; heavy workers: few replicas, low per-pod concurrency (often 1-2), larger memory/CPU requests, possibly a separate node pool. See `distributed-scaling.md` §1 and §3 for the per-pool sizing math, applied separately to each pool now instead of once globally.
- **Within one process**, if you can't split deployments yet, use two separate semaphores rather than one shared one:

```python
# ✅ Heavy traffic can't starve the capacity light traffic needs
light_sem = asyncio.Semaphore(18)
heavy_sem = asyncio.Semaphore(2)

async def handle(payload, is_heavy: bool):
    sem = heavy_sem if is_heavy else light_sem
    async with sem:
        return await process(payload)
```
This is the same idea as a ship's bulkhead: a hole in one compartment doesn't sink the whole vessel. A burst of heavy jobs can only ever consume the capacity reserved for heavy jobs.

- **If heavy work is genuinely CPU-bound** (image/video processing, ML inference, heavy parsing), route it through a `ProcessPoolExecutor` instead of the shared event loop entirely — this isn't just a resource-sizing choice, it's the same "don't block the loop" rule from `asyncio-fundamentals.md` §1 and §6, applied to the case where the blocking work is unavoidable and expensive. A dedicated process pool is a bulkhead too: light requests on the main event loop are physically unaffected by heavy CPU work happening elsewhere.

## 3. Weighted/cost-based concurrency, when you can't cleanly split pools

Sometimes light and heavy work must share one pool (e.g., a single endpoint whose cost varies per-request based on input size). Here, model capacity as *units of cost*, not a count of requests, and have each job acquire the number of units its estimated cost requires:

```python
class WeightedSemaphore:
    """Like asyncio.Semaphore, but callers acquire a variable number of units."""
    def __init__(self, capacity: int):
        self._capacity = capacity
        self._in_use = 0
        self._condition = asyncio.Condition()

    async def acquire(self, weight: int):
        async with self._condition:
            await self._condition.wait_for(lambda: self._in_use + weight <= self._capacity)
            self._in_use += weight

    async def release(self, weight: int):
        async with self._condition:
            self._in_use -= weight
            self._condition.notify_all()

capacity = WeightedSemaphore(100)  # 100 "cost units" per pod, calibrated to memory/CPU budget

async def handle(payload):
    weight = estimate_cost(payload)  # e.g. 1 for light, 20 for heavy
    await capacity.acquire(weight)
    try:
        return await process(payload)
    finally:
        await capacity.release(weight)
```

⚠️ Mistake: `wait_for` above grants requests in the order their condition first becomes true, not strictly FIFO — under sustained load, a stream of light requests can keep out a heavy one indefinitely. If starvation of heavy jobs is unacceptable, add a queue/ticket order on top, or fall back to bulkhead isolation (§2) instead, which doesn't have this problem.

`estimate_cost` doesn't need to be exact — a coarse tier (`small=1, medium=5, large=20`) based on an observable proxy (payload size in bytes, item count in the request, or historical p95 memory/duration for that job type) is enough to prevent the worst case. Perfect cost prediction isn't the goal; keeping the sum of concurrent cost under the pod's real budget is.

## 4. Size container resources for the worst case the concurrency limit allows, not the average

If your pod's memory limit is sized for "20 requests at the average cost," and your concurrency control (weighted or bulkheaded) still permits, say, 4 heavy jobs at once, size the memory limit for 4 heavy jobs concurrently — not for 20 average-cost ones. Whatever your semaphore/weight cap *allows* to run simultaneously is the number that must fit in the resource limit, because it will eventually happen simultaneously under load.

## 5. Autoscaling and metrics need to reflect cost, not request count

A custom autoscaling metric of "in-flight request count" (per `distributed-scaling.md` §4) can be misleading here too: 5 heavy in-flight requests can matter more than 50 light ones. If pools are split (§2), scale each pool on its own metric (light: request count or queue depth; heavy: its own queue depth or in-flight count, which will naturally be a much smaller number at saturation). If pools are merged with weighted concurrency (§3), track and alert on `_in_use` as a fraction of `capacity` — that number already represents the thing that actually matters.

## 6. Quick checklist

- [ ] Does a single count-based semaphore currently gate both cheap and expensive work? → likely needs splitting (§2) or weighting (§3).
- [ ] Is the container memory/CPU limit sized for the worst case the concurrency control actually permits, not the average request? (§4)
- [ ] Is heavy CPU-bound work still running inline on the same event loop as latency-sensitive light requests? → move it to a `ProcessPoolExecutor` or a dedicated worker pool (§2, `asyncio-fundamentals.md` §6)
- [ ] Does the autoscaling metric distinguish heavy from light load, or does it average them into something that under-reacts to a heavy burst? (§5)
