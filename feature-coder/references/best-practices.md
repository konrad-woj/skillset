# Codebase-Specific Best Practices

This guide covers all the best practices, patterns, and conventions specific to this codebase.

## 1. Configuration Management

- **All configuration in YAML**: Expose settings in `conf/` directory using Hydra
- **Main config**: Primary config is typically `default.yaml` which imports other configs
- **Shared configs**: Use underscore-prefixed files (e.g., `_paths.yaml`, `_validation.yaml`) for reusable defaults
- **Config imports**: Import shared configs using Hydra defaults list (e.g., `- _paths@paths`)
- **Test config loading**: Always verify Hydra loads config correctly before proceeding

## 2. Code Organization and Reuse

**Check shared packages first** - Before implementing anything, check:
- `../data-models/` - Cross-package Pydantic models (use `BaseSchema` for API contracts)
- `../data-utils/` - Generic utilities and helpers
- `../logger/` - Logging with structlog and timing decorators (`@timer`, `@async_timer`)
- `../llm-clients/` - LLM client wrappers

**Reuse over reinvent**: If similar code exists, adapt it rather than rewrite from scratch

## 3. Python Code Style

- **Docstrings**: Google-style for all public functions and classes
- **Type hints**: On all function signatures
- **Imports**: Should be at module level (top of file) in final code
  - OK during development: Local imports inside functions while iterating
  - MUST refactor: Move all imports to module level before completion
  - Exception: Lazy loading for expensive imports (must be explicitly justified)
  - Reason: Clarity, performance (imports cached), easier to see dependencies
  - Final check: Scan for function-level imports and move to top
- **Global variables**: Acceptable during development, refactor before completion
  - OK during development: Quick global state for testing/iteration
  - MUST refactor: Convert to proper parameters, class attributes, or config
  - Final check: Scan for global variables and refactor to better patterns
- **Static methods**: Use `@staticmethod` for class-specific methods that don't need `self`
- **Functions over methods**: For generic/reusable logic, use module-level functions
- **Private members**: Prefix with `_` for internal implementation details
- **Logging**: Use `structlog.get_logger()`, never `print()` statements
- **Timing**: Use `@timer` or `@async_timer` decorators for performance tracking
- **Environment variables**: Use `.env` file if present
- **Resource management**: ALWAYS use context managers (`with` statements) for:
  - File operations
  - Database connections
  - Thread pools / Process pools
  - Async resources (use `async with`)
  - Network connections
  - Locks and semaphores
  - Any resource that needs cleanup
- **Caching**: ALWAYS consider what can be cached to improve performance:
  - Use `@functools.lru_cache` for pure functions with repeated calls
  - Use `@functools.cache` (Python 3.9+) for unlimited cache size
  - Cache expensive computations, file reads, API responses
  - Start with simple caching; optimize later if needed
  - Consider cache invalidation strategy

### Import Organization

Final imports at module level should be organized in three groups:

```python
# GOOD - Organized imports at top of file
# Standard library
import json
import logging
from pathlib import Path
from typing import Optional, Dict, List

# Third-party
import pandas as pd
import numpy as np
from pydantic import BaseModel

# Local
from ..data_models import MySchema
from ..data_utils import process_file
from ..logger import timer

# Then your code...
```

## 4. Async/Sync Patterns

**When to use async**:
- I/O-bound operations (API calls, database queries, file operations)
- Need for concurrent processing with rate limiting
- User explicitly requests async

**Default async pattern** - Use `asyncio.Semaphore` for rate-limited concurrency:
```python
async def process_many(items, max_concurrent=10):
    semaphore = asyncio.Semaphore(max_concurrent)

    async def process_with_limit(item):
        async with semaphore:  # Context manager ensures proper release
            return await process_item(item)

    return await asyncio.gather(*[process_with_limit(i) for i in items])
```

**Other asyncio patterns** (always use context managers):
- `asyncio.gather()` - Collect all results
- `asyncio.as_completed()` - Process as available
- `asyncio.Lock` - Mutual exclusion (use `async with lock:`)
- `asyncio.wait_for()` - Timeouts
- `asyncio.Queue` - Producer-consumer

**Resource cleanup in async code**:
```python
# GOOD - Async context manager ensures cleanup
async with aiohttp.ClientSession() as session:
    async with session.get(url) as response:
        data = await response.json()

# GOOD - Async file operations
async with aiofiles.open("file.txt", "r") as f:
    content = await f.read()
```

## 5. Testing Strategy

**Test organization**:
- Unit tests: `tests/unit/` - Run with `uv run pytest tests/unit`
- Integration tests: `tests/integration/` - Run with `uv run pytest tests/integration`

**Test requirements**:
- Mock external dependencies (models, APIs, databases, files)
- Each test should be independent and repeatable
- Use pytest fixtures for mocks
- For async code: Use `AsyncMock` and `@pytest.mark.asyncio`
- **ALWAYS use `@pytest.mark.parametrize`** for testing multiple cases of the same behavior

**Test prioritization** (most to least important):
1. Critical integration tests (3-5 tests verifying main API works)
2. High-value functionality variations
3. Error paths and edge cases
4. Low-value edge cases (can wait)

**Test quality checks**:
- Do they test high-value scenarios?
- Does test data isolate the feature being tested?
- Are mocks correct (`AsyncMock` for async, no real resources)?
- Are there too many low-value edge case stubs?

## 6. Caching Strategy

**Always consider caching** for performance optimization:

**Simple caching with functools**:
```python
from functools import lru_cache, cache

# LRU cache with size limit (good for functions with many possible inputs)
@lru_cache(maxsize=128)
def expensive_computation(x: int, y: int) -> float:
    # Heavy computation here
    return complex_calculation(x, y)

# Unlimited cache (Python 3.9+, good for limited input space)
@cache
def load_config(config_path: str) -> dict:
    with open(config_path) as f:
        return yaml.safe_load(f)

# Cache method results (use on methods)
from functools import cached_property

class DataProcessor:
    @cached_property
    def model(self):
        # Loaded once, then cached
        return load_expensive_model()
```

**What to cache**:
- ✅ File reads that don't change during execution
- ✅ Configuration loading
- ✅ Expensive computations with repeated inputs
- ✅ API responses (when appropriate)
- ✅ Database query results (for read-heavy operations)
- ✅ Model loading / initialization
- ✅ Parsed data structures

**When NOT to cache**:
- ❌ Functions with side effects
- ❌ Data that changes frequently
- ❌ Large objects that consume too much memory
- ❌ User-specific data without proper cache keys

**Async caching**:
```python
# For async functions, consider using aiocache or custom solutions
from aiocache import cached

@cached(ttl=300)  # Cache for 5 minutes
async def fetch_data(url: str) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()
```

**Cache invalidation**:
```python
# Clear specific cache entry
expensive_computation.cache_clear()

# Check cache info
info = expensive_computation.cache_info()
print(f"Hits: {info.hits}, Misses: {info.misses}")
```

## 7. Commands and Workflow

Standard workflow commands (run from package directory):
```bash
# Testing
uv run pytest tests/unit/test_{module}.py -v
uv run pytest tests/integration/test_{module}.py -v

# Code quality
uv run task typecheck
uv run task lint --fix --unsafe-fixes
uv run task format

# Documentation
uv run task save-api  # Update API docs after interface changes
```

**Pre-commit validation**: Use `precommit` skill for comprehensive checks across affected packages

## 8. DRY Principle (Don't Repeat Yourself)

**Always look for repetitive patterns and extract them into reusable functions**:

- If you copy-paste code and change only variable names or values, extract a function
- Look for sequences like "check A, check B, check C" where each check has similar structure
- Multiple `if` blocks with nearly identical bodies should be refactored
- Similar loops with common iteration and processing logic should be consolidated

**Benefits of DRY code**:
- ✅ Easier to maintain (fix bugs in one place)
- ✅ Easier to test (test the pattern once)
- ✅ Easier to read (less code, clearer intent)
- ✅ Easier to extend (add new cases by updating data, not code)
- ✅ Fewer bugs (no risk of fixing one copy but not others)

See [common-mistakes.md](common-mistakes.md#mistake-15-not-keeping-code-dry-dont-repeat-yourself) for detailed examples.
