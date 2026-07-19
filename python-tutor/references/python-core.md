# Python Core Patterns

Modern Python best practices covering type hints, performance, and idiomatic patterns.

## Type Hints

### Modern Type Hints (3.9+)

```python
# Before: Using typing module
from typing import List, Dict, Optional, Union

def process(items: List[str]) -> Dict[str, int]:
    pass

# After: Built-in generics (3.9+)
def process(items: list[str]) -> dict[str, int]:
    pass
```

### Type Unions (3.10+)

```python
# Before: Union type
from typing import Union, Optional

def get_value(key: str) -> Union[str, int, None]:
    pass

def find_user(id: str) -> Optional[User]:
    pass

# After: Pipe syntax (3.10+)
def get_value(key: str) -> str | int | None:
    pass

def find_user(id: str) -> User | None:
    pass
```

### Type Aliases (3.10+)

```python
# Before: Simple assignment
UserId = str
Coordinates = tuple[float, float]

# After: TypeAlias for clarity (3.10+)
from typing import TypeAlias

UserId: TypeAlias = str
Coordinates: TypeAlias = tuple[float, float]
```

## Performance Patterns

### List Comprehensions vs Loops

```python
# Before: Loop with append
results = []
for item in items:
    if item.is_valid:
        results.append(item.transform())

# After: List comprehension (faster, more pythonic)
results = [item.transform() for item in items if item.is_valid]
```

### Generator Expressions for Large Data

```python
# Before: Loading everything into memory
data = [process_row(row) for row in huge_dataset]
total = sum(data)

# After: Generator (memory efficient)
data = (process_row(row) for row in huge_dataset)
total = sum(data)
```

### Dict Operations

```python
# Before: Checking and adding
if key not in my_dict:
    my_dict[key] = []
my_dict[key].append(value)

# After: defaultdict
from collections import defaultdict
my_dict = defaultdict(list)
my_dict[key].append(value)

# Or: setdefault
my_dict.setdefault(key, []).append(value)
```

### String Building

```python
# Before: String concatenation (slow for many items)
result = ""
for item in items:
    result += str(item) + "\n"

# After: Join (much faster)
result = "\n".join(str(item) for item in items)
```

## Idiomatic Patterns

### Context Managers

```python
# Before: Manual resource management
file = open("data.txt")
try:
    data = file.read()
finally:
    file.close()

# After: Context manager
with open("data.txt") as file:
    data = file.read()
```

### Custom Context Managers

```python
# Before: Manual setup/teardown
def process():
    setup_resource()
    try:
        do_work()
    finally:
        cleanup_resource()

# After: Context manager
from contextlib import contextmanager

@contextmanager
def managed_resource():
    resource = setup_resource()
    try:
        yield resource
    finally:
        cleanup_resource(resource)

with managed_resource() as res:
    do_work(res)
```

### Unpacking

```python
# Before: Index access
first = items[0]
rest = items[1:]

# After: Unpacking
first, *rest = items

# Multiple values
head, *middle, tail = items
```

### Walrus Operator (3.8+)

```python
# Before: Duplicate call
data = fetch_data()
if data:
    process(data)

# After: Walrus (assign and check)
if data := fetch_data():
    process(data)

# In comprehensions
results = [processed for item in items if (processed := process(item)) is not None]
```

### Match Statements (3.10+)

```python
# Before: If-elif chains
if isinstance(data, dict):
    handle_dict(data)
elif isinstance(data, list):
    handle_list(data)
elif isinstance(data, str):
    handle_string(data)
else:
    handle_other(data)

# After: Structural pattern matching (3.10+)
match data:
    case dict():
        handle_dict(data)
    case list():
        handle_list(data)
    case str():
        handle_string(data)
    case _:
        handle_other(data)

# With destructuring
match response:
    case {"status": "success", "data": data}:
        process(data)
    case {"status": "error", "message": msg}:
        log_error(msg)
```

## Async Patterns

### Async Comprehensions (3.6+)

```python
# Before: Loop with append
results = []
async for item in async_generator():
    results.append(await process(item))

# After: Async comprehension
results = [await process(item) async for item in async_generator()]
```

### Concurrency: gather and TaskGroup

Sequential `await` calls that could run concurrently, and manual `create_task` + `gather` that could be a 3.11+ `TaskGroup`, are core `python-async-scaling` territory (`asyncio-fundamentals.md` §3) — invoke that skill for the pattern, cancellation semantics, and `return_exceptions` nuance. This skill's own value-add here is purely the version gate: don't suggest `TaskGroup` on a codebase pinned below 3.11 (check `requires-python` first).

## Error Handling

### Specific Exceptions

```python
# Before: Bare except
try:
    risky_operation()
except:
    handle_error()

# After: Specific exceptions
try:
    risky_operation()
except (ValueError, KeyError) as e:
    handle_error(e)
except Exception as e:
    log_unexpected_error(e)
    raise
```

### Exception Groups (3.11+)

```python
# Before: Catching first exception only
try:
    await asyncio.gather(*tasks)
except Exception as e:
    handle_error(e)

# After: Exception groups (3.11+)
try:
    await asyncio.gather(*tasks)
except* ValueError as eg:
    for e in eg.exceptions:
        handle_value_error(e)
except* KeyError as eg:
    for e in eg.exceptions:
        handle_key_error(e)
```

## Dataclasses

### Basic Dataclass

```python
# Before: Manual __init__
class User:
    def __init__(self, name: str, email: str, age: int):
        self.name = name
        self.email = email
        self.age = age

    def __repr__(self):
        return f"User(name={self.name}, email={self.email}, age={self.age})"

# After: Dataclass
from dataclasses import dataclass

@dataclass
class User:
    name: str
    email: str
    age: int
```

### Frozen Dataclass

```python
# Immutable dataclass
@dataclass(frozen=True)
class Config:
    host: str
    port: int
    timeout: float = 30.0
```

### Dataclass with Defaults and Validation

```python
from dataclasses import dataclass, field

@dataclass
class Product:
    name: str
    price: float
    tags: list[str] = field(default_factory=list)

    def __post_init__(self):
        if self.price < 0:
            raise ValueError("Price cannot be negative")
```

## Itertools and Functools

### Common Itertools

```python
from itertools import chain, islice, groupby, batched

# Flatten nested lists
nested = [[1, 2], [3, 4], [5, 6]]
flat = list(chain.from_iterable(nested))

# Take first n items from iterator
first_10 = list(islice(large_iterator, 10))

# Group by key
from operator import itemgetter
data = [{"type": "A", "val": 1}, {"type": "A", "val": 2}, {"type": "B", "val": 3}]
for key, group in groupby(sorted(data, key=itemgetter("type")), key=itemgetter("type")):
    print(key, list(group))

# Batch items (3.12+)
items = range(10)
for batch in batched(items, 3):  # Groups of 3
    process_batch(batch)
```

### Functools

```python
from functools import lru_cache, cache, partial

# Memoization
@lru_cache(maxsize=128)
def expensive_computation(x: int) -> int:
    return x ** 2

# Unlimited cache (3.9+)
@cache
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# Partial application
from operator import mul
double = partial(mul, 2)
result = double(5)  # 10
```

## Logging Best Practices

```python
# Before: Print statements
print(f"Processing {item}")
print(f"Error: {error}")

# After: Structured logging
import structlog

logger = structlog.get_logger()
logger.info("processing_item", item_id=item.id, item_type=item.type)
logger.error("processing_failed", item_id=item.id, error=str(error))
```

## Path Handling

```python
# Before: String concatenation
import os
file_path = base_dir + "/" + subfolder + "/" + filename

# After: pathlib
from pathlib import Path
file_path = Path(base_dir) / subfolder / filename

# Path operations
if file_path.exists():
    content = file_path.read_text()
    file_path.rename(file_path.with_suffix(".bak"))
```
