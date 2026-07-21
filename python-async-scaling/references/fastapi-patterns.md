# FastAPI Patterns

## 1. `def` vs `async def` matters

- `async def` endpoint → runs directly on the event loop. Blocking work inside it stalls **every other request** on that worker.
- Plain `def` endpoint → FastAPI runs it in a thread pool automatically. Safe for blocking libraries, at the cost of thread-pool overhead.

```python
# ❌ async def + blocking DB driver = blocks the whole server
@app.get("/users")
async def get_users():
    return psycopg2_blocking_query()

# ✅ Option A: plain def, let FastAPI thread it
@app.get("/users")
def get_users():
    return psycopg2_blocking_query()

# ✅ Option B: use an async driver
@app.get("/users")
async def get_users():
    return await asyncpg_query()
```

## 2. Pick one lane for DB access and stay in it

Mixing sync SQLAlchemy sessions inside `async def` routes is one of the most common silent-blocking bugs in FastAPI apps. Use either a fully sync stack (`def` routes + sync driver) or a fully async stack (`async def` + `asyncpg`/SQLAlchemy async engine) — not both in the same route.

## 3. Reuse HTTP clients and connection pools; don't create per-request

```python
# ❌ Wasteful, no connection pooling, slow under load
async def call_api():
    async with httpx.AsyncClient() as client:
        return await client.get(url)

# ✅ Reuse a client via lifespan/app state
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.client = httpx.AsyncClient()
    app.state.db_pool = await asyncpg.create_pool(dsn, min_size=5, max_size=20)
    yield
    await app.state.client.aclose()
    await app.state.db_pool.close()

app = FastAPI(lifespan=lifespan)
```
Same applies to DB connection pools — created once at startup, sized deliberately (see `distributed-scaling.md` for sizing across replicas).

## 4. `BackgroundTasks` is not a job queue

`BackgroundTasks` runs after the response is sent, in the same process. Fine for small fire-and-forget work (send a confirmation email). Not fine for:
- work that must survive the pod/instance dying,
- work that needs retries,
- work heavier than a few seconds.

For that, use a real task queue (Celery, arq, SQS + worker, RabbitMQ + worker) — see `queues-and-workers.md`.

If middleware sets request-scoped context (request id, auth info) via `contextvars` and a background task or `create_task` reads it, the ordering matters — see `asyncio-fundamentals.md` §9.

## 5. Blocking dependencies/middleware block every route that uses them

A slow sync call inside a dependency injected into many routes silently blocks all of them, not just one endpoint — easy to miss in review because the dependency looks innocuous.

```python
# ❌ sync, blocking, but injected everywhere
def get_current_user(token: str = Depends(oauth2_scheme)):
    return sync_blocking_db_call(token)  # blocks every route using this dependency

# ✅ async dependency, non-blocking
async def get_current_user(token: str = Depends(oauth2_scheme)):
    return await async_db_call(token)
```

## 6. Streaming responses and long-lived connections

For SSE/streaming/websocket endpoints, remember each open connection holds a slot for the connection's lifetime — this interacts directly with your concurrency limits and autoscaling (see `distributed-scaling.md` §6 on websockets). Always set a max duration or heartbeat/ping so dead connections don't accumulate.

## 7. Validate the timeout stack, not just app code

FastAPI's own request handling has no default timeout — a hung coroutine can hold a request open indefinitely unless you enforce one explicitly (`asyncio.timeout`) or rely on upstream layers (ingress/load balancer, ASGI server like `uvicorn --timeout-keep-alive`). Don't assume the framework protects you here.
