# Testing Async Code

Async tests fail in ways sync tests don't: silently-unawaited mocks, event loops leaking state between tests, and retry/backoff logic that makes your suite slow (or flaky under CI load) because it actually sleeps. Read this before writing tests for anything covered elsewhere in this skill.

## 1. Basic setup: `pytest-asyncio`

```python
# pytest.ini / pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"   # lets you write `async def test_...` without a decorator on every test
```
Without `asyncio_mode = "auto"` (or an explicit `@pytest.mark.asyncio` on every test), an `async def test_...` silently returns a coroutine object that pytest treats as "passed" without ever running its body — a false-positive test that never actually executed. If a test suite has async tests that all seem to "pass" suspiciously fast, this is the first thing to check.

## 2. Event loop scope: prefer function-scoped, know why

```python
# ❌ A session-scoped loop/fixture shares state across every test
@pytest.fixture(scope="session")
async def client():
    async with httpx.AsyncClient() as c:
        yield c

# ✅ Function-scoped by default — each test gets a clean slate
@pytest.fixture
async def client():
    async with httpx.AsyncClient() as c:
        yield c
```
A broader-scoped loop/fixture is occasionally worth it for expensive setup (a shared DB connection pool), but it means tasks, cached state, or half-finished background work from one test can leak into the next. If a test only fails when run after another specific test (not in isolation), suspect a shared-scope fixture before suspecting the test itself.

## 3. Mocking async functions needs `AsyncMock`, not `MagicMock`

```python
# ❌ MagicMock's return value is a MagicMock, not something awaitable
mock_fetch = MagicMock()
await mock_fetch()  # TypeError: object MagicMock can't be used in 'await' expression

# ✅
from unittest.mock import AsyncMock
mock_fetch = AsyncMock(return_value={"status": "ok"})
result = await mock_fetch()
```
`unittest.mock.patch` auto-detects an async target and uses `AsyncMock` for you in modern Python — but double check when patching an attribute directly, wrapping a sync fallback function, or using an older mocking helper, since a plain `Mock`/`MagicMock` standing in for an async function is a common source of "works in the test, breaks in prod" (the mock never actually exercised the `await`).

## 4. Don't let retry/backoff tests actually sleep

```python
# ❌ A test for 5-attempt exponential backoff really waits ~30+ seconds
async def test_retries():
    await call_with_retry(flaky_fn, max_attempts=5)

# ✅ Patch the sleep, assert the behavior, not the wall-clock time
async def test_retries(monkeypatch):
    sleeps = []
    async def fake_sleep(seconds):
        sleeps.append(seconds)
    monkeypatch.setattr(asyncio, "sleep", fake_sleep)

    await call_with_retry(flaky_fn, max_attempts=5)
    assert len(sleeps) == 4  # 4 backoffs between 5 attempts
```
If a test suite takes minutes to run and much of that is retry/backoff code under test, this is almost always why — and it also means CI is exercising real timing instead of the actual retry logic, which is a worse test, not just a slower one.

## 5. Testing cancellation and timeouts

```python
async def test_cleanup_runs_on_cancellation():
    cleaned_up = False

    async def work():
        nonlocal cleaned_up
        try:
            await asyncio.sleep(10)
        except asyncio.CancelledError:
            cleaned_up = True
            raise

    task = asyncio.create_task(work())
    await asyncio.sleep(0)  # let it start
    task.cancel()
    with pytest.raises(asyncio.CancelledError):
        await task
    assert cleaned_up
```
This directly tests the "re-raise `CancelledError` after cleanup" rule from `asyncio-fundamentals.md` §5 — don't just assert the happy path completes; assert cancellation actually triggers cleanup, since that's the code path most likely to silently regress.

## 6. Idempotency and duplicate-delivery are testable, not just reviewable

The idempotency requirement from `queues-and-workers.md` §4 is a concrete, automatable assertion:

```python
async def test_handler_is_idempotent():
    body = {"job_id": "abc-123", "amount": 10}
    await handle_job(body)
    await handle_job(body)  # simulate at-least-once redelivery
    assert await charge_count(body["job_id"]) == 1
```
If you can't write this test because the handler has no way to check "did I already do this," that's the bug the test just found — write the dedup check first.

## 7. Bugs that only appear at replica-count > 1 need more than mocks

Per `observability-debugging.md` §5: pool exhaustion, lock contention, and cluster-wide rate limits are properties of *shared state across processes*, not of any single function. A unit test with everything mocked can't catch "two instances raced for the same distributed lock." For these, either:
- write an integration test against a real (or containerized) Redis/DB and spin up two consumers/handlers concurrently against it, or
- accept that this class of bug is caught by review + the checklists in this skill, not by unit tests, and say so explicitly rather than implying test coverage that isn't there.

## 8. Common flakiness checklist

- [ ] Does every task created inside a test get awaited or cancelled before the test ends? A leaked running task can affect the *next* test, not just this one.
- [ ] Is a broad-scoped (session/module) fixture holding a client/pool/cache that tests assume is fresh?
- [ ] Does any test assert on the relative timing/order of two concurrent coroutines without deterministic control (an `Event`, a mocked clock)? Racing real timing is inherently flaky under CI load.
- [ ] Does the code under test call `asyncio.run(...)` itself? That raises inside a pytest-asyncio loop that's already running — the function under test should be a plain coroutine you await, with `asyncio.run` reserved for the real entrypoint (`asyncio-fundamentals.md` §8).
