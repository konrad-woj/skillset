# Queues and Workers (SQS, RabbitMQ)

## 1. Move long-running work out of the request/response cycle

Don't await work in a request handler that might exceed the load balancer / API gateway timeout (often 30-60s).

```python
async def handle_request():
    try:
        async with asyncio.timeout(25):  # stay under upstream timeout
            return await slow_task()
    except TimeoutError:
        job_id = await enqueue_job(...)
        return {"status": "processing", "job_id": job_id}
```

Better: for anything that predictably takes more than a few seconds, skip the inline attempt entirely — enqueue immediately and return `202 Accepted` with a job id. This decouples request latency from work duration, which is the actual fix (a longer timeout just delays the same problem).

## 2. The queue consumer is your concurrency boundary, not a semaphore inside FastAPI

```python
import aio_pika

async def consume():
    connection = await aio_pika.connect_robust(RABBITMQ_URL)
    channel = await connection.channel()
    await channel.set_qos(prefetch_count=10)  # ← concurrency limit lives here

    queue = await channel.declare_queue("jobs", durable=True)

    async def on_message(message: aio_pika.IncomingMessage):
        async with message.process():  # auto ack/nack on exception
            await handle_job(message.body)

    await queue.consume(on_message)
```

⚠️ Mistake: pulling a batch of messages and firing `asyncio.gather` over all of them with no bound — this reintroduces the overload problem the queue was meant to prevent.

```python
sem = asyncio.Semaphore(prefetch_count)

async def handle_job(body):
    async with sem:
        ...
```

## 3. SQS visibility timeout vs. task duration

`VisibilityTimeout` controls how long before an unacked message becomes visible to another consumer again. If a task can run longer than the visibility timeout, another consumer may pick up the same message while the first is still working — set the timeout comfortably above expected duration, or use heartbeat/`ChangeMessageVisibility` extension calls for long tasks so it doesn't get redelivered mid-processing.

## 4. Idempotency is not optional

Both SQS (at-least-once delivery) and RabbitMQ (redelivery on nack/crash/lock-TTL-expiry) can deliver the same message more than once. Every handler should check "have I already done this?" keyed by a stable job/message id before doing side-effecting work (charging a card, sending an email, writing a row) — don't rely on the queue promising exactly-once.

```python
async def handle_job(body):
    job_id = body["job_id"]
    if await already_processed(job_id):
        return
    await do_work(body)
    await mark_processed(job_id)
```

Your own dedup check can still race (two redelivered copies both pass the "already processed?" check before either finishes). For calls to external APIs that support it — payment providers especially — also pass a stable idempotency key on the call itself (e.g. Stripe's `Idempotency-Key` header), so the provider collapses duplicates even if your side-effecting write happens twice. Treat your own dedup table as the first line of defense, not the only one.

## 5. Dead-letter queues are your safety net, not an afterthought

Configure a DLQ (or RabbitMQ dead-letter exchange) with a max-retry count. Without one, a poison message (malformed payload, a bug that always throws) gets redelivered forever, burning consumer capacity and hiding the real failure. Alert on DLQ depth.

## 6. Ordering guarantees are easy to assume and rarely free

Standard SQS queues don't guarantee order; FIFO queues do but cap throughput per message group. RabbitMQ preserves order per-queue/per-consumer but not across multiple consumers processing concurrently. If your workflow needs strict per-entity ordering (e.g., events for the same user must apply in sequence), route by a partition/group key (SQS FIFO `MessageGroupId`, or a consistent-hash-based queue split) rather than assuming a single queue gives you order for free once you scale consumers.
