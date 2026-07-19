# Common Mistakes to Avoid

This guide catalogs common mistakes encountered during feature development and how to avoid them.

## Mistake #1: Implementing Before Planning

**Wrong**: Start writing code immediately

**Right**: Write plan, get user approval, THEN code

## Mistake #2: Not Checking Data Structures

**Wrong**: Assume what fields a model has

**Right**: Read the actual model definition first

Example - You wrote:
```python
return {
    "issue_type": IssueType.VALIDATION_ERROR,
    "description": "...",
}
```

But the actual model requires 7 fields. **Check the model first!**

## Mistake #3: Counting Stub Tests as Passing

**Wrong**: "All 10 tests pass!" when 5 are just `pass` statements

**Right**: Count only tests with real assertions, and verify with detection tools

**The Problem**:
This is one of the MOST COMMON mistakes. During the stub phase, tests are written with `pass` or commented out. When implementing features, it's easy to forget which tests are active vs inactive, leading to false claims of completion.

**How to verify**:
```python
# This is a STUB (doesn't test anything):
def test_feature(self):
    pass

# This is a STUB (commented out):
# def test_deprecated_feature(self):
#     result = my_function(input)
#     assert result == expected

# This is REAL (actually tests):
def test_feature(self, fixture):
    result = my_function(input)
    assert result == expected
```

**MANDATORY: Use detection tools before claiming completion**:

```bash
# Recommended: Claude Code slash command
/check-inactive-tests tests/unit/test_scorer.py

# Alternative: Quick grep check
grep -E "(^#.*def test_|^\s+pass\s*$)" tests/unit/test_scorer.py
```

**Example output showing the problem**:
```bash
$ /check-inactive-tests tests/unit/test_scorer.py

📊 Test Status for test_scorer.py
============================================================
Total test functions: 15
✅ Active tests (with assertions): 8
⚠️  Stub tests (only 'pass'): 5
❌ Commented tests: 2

⚠️  Stub tests detected:
   - test_language_mismatch_warning
   - test_special_chars_preserved
   - test_numeric_consistency_error
   - test_multiple_penalties_accumulate
   - test_edge_case_empty_strings

❌ Found 7 inactive tests!
```

**Correct reporting**:
- ❌ WRONG: "All 15 tests pass!"
- ✅ RIGHT: "8 active tests pass. 7 stub/commented tests remain to be implemented."

**When to check**:
1. ✅ Before claiming "tests pass"
2. ✅ After implementing each feature
3. ✅ Before marking TODOs as complete
4. ✅ Before saying implementation is done

**Action when inactive tests found**:
1. Implement the remaining stub tests
2. Remove low-value stub tests that aren't needed
3. Update TODO list to track remaining test work
4. NEVER claim completion with inactive tests present

See [tdd-implementation.md Step 5.8](tdd-implementation.md) for full detection script and detailed guidance.

## Mistake #4: Not Using Helper Methods

**Wrong**: Duplicate complex object creation in every check method

**Right**: Create helper like `_create_issue()` to build objects consistently

## Mistake #5: Returning Wrong Types

**Wrong**: Return dict when code expects Pydantic model

**Right**: Check what the calling code expects, return proper type

**Pattern**: If integration fails with type errors, you assumed wrong. Read the actual interfaces.

## Mistake #6: Not Running Tests After Implementation

**Wrong**: Implement multiple features, then run tests

**Right**: Implement one feature, uncomment test, run test, verify pass, repeat

## Mistake #7: Ignoring User Feedback

**Wrong**: User says "X won't work" but you proceed anyway

**Right**: Listen - user knows constraints and requirements you don't

## Mistake #8: Writing Too Many Low-Value Tests Before Core Tests

**Wrong**: Write 40+ stub tests for edge cases before testing the main API works

**Right**: Write 3-5 critical integration tests first, then add edge cases

**Pattern**: Prioritize tests by value:
1. **Critical (write first)**: Core API integration tests that verify the feature works end-to-end
2. **High value**: Tests for key functionality variations and error paths
3. **Medium value**: Tests for optional features and boundary conditions
4. **Low value**: Detailed edge cases that can wait until core is working

**Example prioritization**:
```python
# CRITICAL - Write first
def test_score_field_returns_valid_result(scorer):
    """Test that main API returns correct structure."""
    result = scorer.score_field(source, target, entity_type, lang, context, store)
    assert isinstance(result, ScoringResult)
    assert 0.0 <= result.confidence_score <= 1.0

# HIGH VALUE - Write early
def test_high_similarity_no_penalty(scorer):
    """Test that good translations get high confidence."""
    # Mock high similarity, verify no issues triggered

def test_low_similarity_triggers_error(scorer):
    """Test that bad translations get flagged."""
    # Mock low similarity, verify ERROR issue created

# MEDIUM VALUE - Write after core works
def test_multiple_penalties_accumulate(scorer):
    """Test that multiple issues combine properly."""

# LOW VALUE - Can wait or be removed
def test_empty_source_text(scorer):
    """Test edge case with empty input."""
    # Nice to have but not critical for initial implementation
```

## Mistake #9: Test Data That Triggers Unrelated Penalties

**Wrong**: Test semantic similarity with text that also fails length ratio and language detection

**Right**: Use test data that isolates the feature being tested

**Example**:
```python
# BAD - Multiple checks fail, obscures what you're testing
def test_high_similarity():
    result = scorer.score_field(
        source="GATE VALVE",  # 10 chars
        target="VÁLVULA DE COMPUERTA",  # 20 chars - ratio 2.0, triggers penalty!
        # Also: lingua detector sees English chars, triggers language mismatch!
    )
    assert result.confidence_score > 0.9  # Fails due to OTHER penalties

# GOOD - Isolated test
def test_high_similarity():
    result = scorer.score_field(
        source="VALVE",  # 5 chars
        target="VALVULA",  # 7 chars - good ratio, similar strings
    )
    assert result.semantic_similarity > 0.9  # Tests ONLY similarity
    sim_issues = [i for i in result.issues if i.issue_type == ScoringIssueType.LOW_SEMANTIC_SIMILARITY]
    assert len(sim_issues) == 0
```

## Mistake #10: Not Handling Optional Fields in Assertions

**Wrong**: Directly compare optional fields that might be None

**Right**: Assert field is not None before comparing

**Example**:
```python
# BAD - Typecheck error if semantic_similarity can be None
assert result.semantic_similarity > 0.85

# GOOD - Narrow the type first
assert result.semantic_similarity is not None
assert result.semantic_similarity > 0.85
```

## Mistake #11: Incorrect Mock Setup in Unit Tests

**Wrong**: Mocks that get closed by context managers or tests that load real resources

**Right**: Ensure mocks remain accessible throughout test and verify no real resources are loaded

**Common mocking issues**:

### 1. Context manager closes mocked object

```python
# BAD - Mock gets closed before assertion
@pytest.fixture
def mock_client(mocker):
    with mocker.patch("module.Client") as mock:
        return mock  # Context exits here, mock is closed!

def test_something(mock_client):
    result = use_client()
    mock_client.method.assert_called_once()  # Fails - mock is closed!

# GOOD - Mock persists through test
@pytest.fixture
def mock_client(mocker):
    mock = mocker.patch("module.Client")
    return mock  # No context manager, mock stays alive
```

### 2. Test loads actual model/data/client

```python
# BAD - Accidentally loads real 500MB model
def test_prediction(mocker):
    mocker.patch("module.config_loader")  # Forgot to mock model loader!
    result = predict(input_data)  # Loads actual model from disk
    assert result is not None

# GOOD - Mock all external resources
def test_prediction(mocker):
    mocker.patch("module.config_loader")
    mock_model = mocker.patch("module.load_model")  # Mock model loading
    mock_model.return_value = MagicMock()
    result = predict(input_data)  # Uses mock, no disk I/O
    assert result is not None
```

### 3. Mock scope doesn't match test lifecycle

```python
# BAD - Mock applied in wrong scope
def test_something():
    with patch("module.external_api"):
        client = Client()  # Mock applies here
    result = client.call()  # Mock no longer applies, real API called!
    assert result is not None

# GOOD - Mock covers entire test
@patch("module.external_api")
def test_something(mock_api):
    client = Client()
    result = client.call()  # Mock still applies
    assert result is not None
```

**Validation checklist before running unit tests**:
- [ ] Are all external resources (models, APIs, databases, files) mocked?
- [ ] Do mocks persist through the entire test execution?
- [ ] Are context managers used correctly (not closing mocks prematurely)?
- [ ] Would this test run quickly without network/disk access?
- [ ] For async code: Are async mocks (`AsyncMock`) used instead of `MagicMock`?

## Mistake #12: Mixing Sync and Async or Wrong Async Patterns

**Wrong**: Using sync code for async functions, wrong async patterns, or sync mocks for async code

**Right**: Properly implement async/await patterns and use appropriate asyncio utilities

This mistake category — sync mocks (`MagicMock`) standing in for async functions, missing `pytest.mark.asyncio`, unbounded `gather()`, forgotten `await`, picking the wrong concurrency primitive — is exactly what `python-async-scaling` exists to catch. Invoke it (its `asyncio-fundamentals.md`, `testing-async-code.md`, and `decision-guide.md`) rather than re-deriving these examples here; it also covers the cross-instance failure modes (a `Semaphore` that only limits one pod, not the fleet) that a single-process fix like the ones above would miss.

**Ask user which pattern fits their use case before implementing!**

## Mistake #13: Not Properly Closing Resources

**Wrong**: Opening files, connections, threads without proper cleanup

**Right**: ALWAYS use context managers to ensure resources are closed

**Common resource leaks**:

### 1. Files without context managers

```python
# BAD - File might not close if exception occurs
def read_data(path):
    f = open(path, "r")
    data = f.read()
    f.close()  # Won't execute if exception above
    return data

# GOOD - Context manager ensures cleanup
def read_data(path):
    with open(path, "r") as f:
        data = f.read()
    return data  # File closed automatically
```

### 2. Thread/Process pools not cleaned up

```python
# BAD - Executor might not be cleaned up
from concurrent.futures import ThreadPoolExecutor

def process_items(items):
    executor = ThreadPoolExecutor(max_workers=10)
    results = list(executor.map(process_item, items))
    executor.shutdown()  # Might not execute if exception
    return results

# GOOD - Context manager ensures cleanup
from concurrent.futures import ThreadPoolExecutor

def process_items(items):
    with ThreadPoolExecutor(max_workers=10) as executor:
        results = list(executor.map(process_item, items))
    return results  # Executor cleaned up automatically
```

### 3. Async resources without async context managers

```python
# BAD - Session might not close properly
async def fetch_data(url):
    session = aiohttp.ClientSession()
    response = await session.get(url)
    data = await response.json()
    await session.close()  # Might not execute if exception
    return data

# GOOD - Async context manager ensures cleanup
async def fetch_data(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()
    # Session and response closed automatically
```

### 4. Database connections

```python
# BAD - Connection might not close
def query_db(query):
    conn = psycopg2.connect(dsn)
    cursor = conn.cursor()
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    conn.close()  # Might not execute if exception
    return results

# GOOD - Context managers ensure cleanup
def query_db(query):
    with psycopg2.connect(dsn) as conn:
        with conn.cursor() as cursor:
            cursor.execute(query)
            return cursor.fetchall()
    # Connection and cursor closed automatically
```

### 5. Locks and semaphores

```python
# BAD - Lock might not release
def update_shared_data(lock, data):
    lock.acquire()
    shared_state.update(data)
    lock.release()  # Might not execute if exception

# GOOD - Context manager ensures release
def update_shared_data(lock, data):
    with lock:
        shared_state.update(data)
    # Lock released automatically

# GOOD - Async lock
async def update_shared_data_async(lock, data):
    async with lock:
        await shared_state.update(data)
    # Lock released automatically
```

**Resource cleanup checklist**:
- [ ] All file operations use `with open(...) as f:`
- [ ] Thread/process pools use `with ThreadPoolExecutor() as executor:`
- [ ] Async resources use `async with` (sessions, connections, files)
- [ ] Database connections use context managers
- [ ] Locks/semaphores use `with lock:` or `async with lock:`
- [ ] Network connections use context managers
- [ ] No manual `.close()` calls (context manager handles it)

**Rule of thumb**: If a resource has a `.close()` method or needs cleanup, use a context manager!

## Mistake #14: Not Refactoring Local Imports and Global Variables

**Wrong**: Leaving local imports and global variables in final code

**Right**: Refactor to module-level imports and proper variable scoping before completion

**Development vs Final Code**:
- ✅ **OK during development**: Local imports and global variables while iterating
- ❌ **NOT OK in final code**: Must refactor before marking feature complete
- 🔍 **Final check required**: Scan for these patterns and clean them up

**Why this matters**:
- **Clarity**: All dependencies visible at top of file
- **Performance**: Python caches imports, but function-level imports add overhead
- **Debugging**: Easier to see what the module depends on
- **Testing**: Easier to mock imports when they're at module level
- **Maintainability**: Global state makes code harder to reason about
- **Standards**: PEP 8 recommends module-level imports

**Common patterns to refactor**:

### 1. Local imports (refactor to module level)

```python
# DEVELOPMENT - OK while iterating
def process_data(df):
    import pandas as pd  # OK during dev
    import numpy as np   # OK during dev
    return df.fillna(0)

# FINAL CODE - Move to top before completion
import pandas as pd
import numpy as np

def process_data(df):
    return df.fillna(0)
```

### 2. Global variables (refactor to parameters/config)

```python
# DEVELOPMENT - OK while iterating
SHARED_STATE = {}  # OK during dev for quick testing

def process_item(item):
    global SHARED_STATE
    SHARED_STATE[item.id] = item.value

# FINAL CODE - Refactor to proper pattern
class DataProcessor:
    def __init__(self):
        self.state: dict = {}  # Better: instance attribute

    def process_item(self, item):
        self.state[item.id] = item.value

# OR pass as parameter
def process_item(item, state: dict):
    state[item.id] = item.value
```

### 3. Multiple scattered imports (consolidate at top)

```python
# DEVELOPMENT - OK while iterating
def load_config():
    import yaml  # OK during dev
    with open("config.yaml") as f:
        return yaml.safe_load(f)

def save_results(data):
    import json  # OK during dev
    with open("results.json", "w") as f:
        json.dump(data, f)

# FINAL CODE - Consolidate all imports at module level
import yaml
import json

def load_config():
    with open("config.yaml") as f:
        return yaml.safe_load(f)

def save_results(data):
    with open("results.json", "w") as f:
        json.dump(data, f)
```

**Valid exception - Lazy loading expensive imports**:
```python
# Module-level imports for common dependencies
import json
from typing import Optional

# ACCEPTABLE - Lazy loading for expensive/optional dependency
def load_model(model_path: str):
    """Load ML model only when needed (expensive import)."""
    import torch  # OK: Lazy loading expensive ML library
    import transformers  # OK: Only load when actually using models

    model = transformers.AutoModel.from_pretrained(model_path)
    return model

# This is acceptable because:
# 1. torch/transformers are very heavy imports (slow startup)
# 2. Function might not be called in every execution path
# 3. Justification is clear (performance optimization)
```

**When lazy loading is acceptable**:
- Importing very large/slow libraries (e.g., torch, tensorflow, heavy ML libs)
- Import is only needed in rarely-used code paths
- Startup time is critical and import is expensive
- **MUST explicitly justify why in comments**

**Final check before completion**:
1. ✅ Scan all files for function-level imports → Move to module top
2. ✅ Scan for `global` keyword usage → Refactor to parameters/classes/config
3. ✅ Exception: Lazy loading with explicit justification is OK
4. ✅ Organize imports: stdlib → third-party → local

**Rule**: Local imports and globals are fine during development, but:
- Before marking feature complete, do a final refactoring pass
- Move imports to module level (except justified lazy loading)
- Refactor globals to proper parameters, class attributes, or config

## Mistake #15: Not Keeping Code DRY (Don't Repeat Yourself)

**Wrong**: Writing long functions with repetitive conditional logic that performs similar operations

**Right**: Extract common patterns into reusable functions and iterate over cases

**The Problem**:
AI-generated code often creates long, readable flows with condition-after-condition that look easy to follow but actually perform the same tasks repeatedly. This makes code harder to maintain and violates DRY principles.

**Common anti-patterns**:

### 1. Multiple conditions doing the same thing

```python
# BAD - Repetitive checks, each doing essentially the same thing
def validate_fields(data):
    issues = []

    if data.name is None or data.name == "":
        issues.append(create_issue(
            issue_type=IssueType.MISSING_FIELD,
            field="name",
            description="Name is missing or empty",
            severity=Severity.ERROR
        ))

    if data.email is None or data.email == "":
        issues.append(create_issue(
            issue_type=IssueType.MISSING_FIELD,
            field="email",
            description="Email is missing or empty",
            severity=Severity.ERROR
        ))

    if data.phone is None or data.phone == "":
        issues.append(create_issue(
            issue_type=IssueType.MISSING_FIELD,
            field="phone",
            description="Phone is missing or empty",
            severity=Severity.ERROR
        ))

    if data.address is None or data.address == "":
        issues.append(create_issue(
            issue_type=IssueType.MISSING_FIELD,
            field="address",
            description="Address is missing or empty",
            severity=Severity.ERROR
        ))

    return issues

# GOOD - Extract pattern into reusable function
def validate_fields(data):
    required_fields = ["name", "email", "phone", "address"]
    return [
        _check_required_field(data, field)
        for field in required_fields
        if not _is_field_present(data, field)
    ]

def _is_field_present(data, field: str) -> bool:
    """Check if field exists and is non-empty."""
    value = getattr(data, field, None)
    return value is not None and value != ""

def _check_required_field(data, field: str) -> Issue:
    """Create issue for missing required field."""
    return create_issue(
        issue_type=IssueType.MISSING_FIELD,
        field=field,
        description=f"{field.capitalize()} is missing or empty",
        severity=Severity.ERROR
    )
```

### 2. Similar operations on different data

```python
# BAD - Repetitive processing for each entity type
def process_entities(data):
    # Process users
    user_results = []
    for user in data.users:
        validated = validate_user(user)
        enriched = enrich_user(validated)
        transformed = transform_user(enriched)
        user_results.append(transformed)

    # Process products
    product_results = []
    for product in data.products:
        validated = validate_product(product)
        enriched = enrich_product(validated)
        transformed = transform_product(enriched)
        product_results.append(transformed)

    # Process orders
    order_results = []
    for order in data.orders:
        validated = validate_order(order)
        enriched = enrich_order(validated)
        transformed = transform_order(enriched)
        order_results.append(transformed)

    return user_results, product_results, order_results

# GOOD - Single function handles pattern
def process_entities(data):
    entity_configs = [
        ("users", data.users, validate_user, enrich_user, transform_user),
        ("products", data.products, validate_product, enrich_product, transform_product),
        ("orders", data.orders, validate_order, enrich_order, transform_order),
    ]

    results = {}
    for name, items, validate_fn, enrich_fn, transform_fn in entity_configs:
        results[name] = _process_entity_list(items, validate_fn, enrich_fn, transform_fn)

    return results["users"], results["products"], results["orders"]

def _process_entity_list(items, validate_fn, enrich_fn, transform_fn):
    """Apply validation, enrichment, and transformation pipeline to items."""
    return [transform_fn(enrich_fn(validate_fn(item))) for item in items]
```

### 3. Multiple similar scoring/checking functions

```python
# BAD - Each check is its own long function doing similar work
def check_length_ratio(source: str, target: str, threshold: float) -> Optional[Issue]:
    ratio = len(target) / len(source)
    if ratio > threshold:
        return create_issue(
            issue_type=IssueType.LENGTH_MISMATCH,
            description=f"Length ratio {ratio:.2f} exceeds threshold {threshold}",
            severity=Severity.WARNING
        )
    return None

def check_numeric_consistency(source: str, target: str, threshold: float) -> Optional[Issue]:
    source_nums = extract_numbers(source)
    target_nums = extract_numbers(target)
    consistency = calculate_numeric_match(source_nums, target_nums)
    if consistency < threshold:
        return create_issue(
            issue_type=IssueType.NUMERIC_MISMATCH,
            description=f"Numeric consistency {consistency:.2f} below threshold {threshold}",
            severity=Severity.WARNING
        )
    return None

def check_special_chars(source: str, target: str, threshold: float) -> Optional[Issue]:
    source_chars = extract_special_chars(source)
    target_chars = extract_special_chars(target)
    consistency = calculate_char_match(source_chars, target_chars)
    if consistency < threshold:
        return create_issue(
            issue_type=IssueType.SPECIAL_CHAR_MISMATCH,
            description=f"Special char consistency {consistency:.2f} below threshold {threshold}",
            severity=Severity.WARNING
        )
    return None

# GOOD - Single configurable function
def check_metric_threshold(
    source: str,
    target: str,
    metric_name: str,
    metric_fn: Callable[[str, str], float],
    threshold: float,
    comparison: Literal["above", "below"],
    issue_type: IssueType,
    severity: Severity = Severity.WARNING
) -> Optional[Issue]:
    """Generic threshold check for any metric."""
    value = metric_fn(source, target)
    exceeds = (comparison == "above" and value > threshold) or \
              (comparison == "below" and value < threshold)

    if exceeds:
        direction = "exceeds" if comparison == "above" else "below"
        return create_issue(
            issue_type=issue_type,
            description=f"{metric_name} {value:.2f} {direction} threshold {threshold}",
            severity=severity
        )
    return None

# Now all checks use the same pattern
def check_length_ratio(source: str, target: str, threshold: float) -> Optional[Issue]:
    return check_metric_threshold(
        source, target, "Length ratio",
        lambda s, t: len(t) / len(s),
        threshold, "above",
        IssueType.LENGTH_MISMATCH
    )

def check_numeric_consistency(source: str, target: str, threshold: float) -> Optional[Issue]:
    return check_metric_threshold(
        source, target, "Numeric consistency",
        lambda s, t: calculate_numeric_match(extract_numbers(s), extract_numbers(t)),
        threshold, "below",
        IssueType.NUMERIC_MISMATCH
    )
```

### 4. Data structure building with repeated patterns

```python
# BAD - Manually building similar dictionaries
def create_report(data):
    report = {}

    # User statistics
    report["user_stats"] = {
        "total": len(data.users),
        "active": sum(1 for u in data.users if u.is_active),
        "inactive": sum(1 for u in data.users if not u.is_active),
        "percentage_active": sum(1 for u in data.users if u.is_active) / len(data.users) * 100
    }

    # Product statistics
    report["product_stats"] = {
        "total": len(data.products),
        "active": sum(1 for p in data.products if p.is_active),
        "inactive": sum(1 for p in data.products if not p.is_active),
        "percentage_active": sum(1 for p in data.products if p.is_active) / len(data.products) * 100
    }

    # Order statistics
    report["order_stats"] = {
        "total": len(data.orders),
        "active": sum(1 for o in data.orders if o.is_active),
        "inactive": sum(1 for o in data.orders if not o.is_active),
        "percentage_active": sum(1 for o in data.orders if o.is_active) / len(data.orders) * 100
    }

    return report

# GOOD - Single function builds any stats dict
def create_report(data):
    return {
        "user_stats": _calculate_stats(data.users),
        "product_stats": _calculate_stats(data.products),
        "order_stats": _calculate_stats(data.orders),
    }

def _calculate_stats(items: list) -> dict:
    """Calculate standard statistics for any list of items with is_active."""
    total = len(items)
    active = sum(1 for item in items if item.is_active)
    return {
        "total": total,
        "active": active,
        "inactive": total - active,
        "percentage_active": (active / total * 100) if total > 0 else 0
    }
```

**How to identify DRY violations**:
1. **Copy-paste detector**: If you copy-paste code and change only variable names or values, extract a function
2. **Pattern recognition**: Look for sequences like "check A, check B, check C" where each check has similar structure
3. **Similar conditionals**: Multiple `if` blocks with nearly identical bodies
4. **Repeated computations**: Same calculation done in multiple places
5. **Loop similarity**: Multiple loops with similar iteration and processing logic

**Refactoring checklist**:
- [ ] Can these conditions be replaced by iteration over a list/dict?
- [ ] Do these functions have the same structure? Can I extract the pattern?
- [ ] Am I computing the same thing multiple times? Can I cache it?
- [ ] Would a helper function make this clearer and more maintainable?
- [ ] Can configuration data replace hardcoded repeated patterns?

**Benefits of DRY code**:
- ✅ Easier to maintain (fix bugs in one place)
- ✅ Easier to test (test the pattern once)
- ✅ Easier to read (less code, clearer intent)
- ✅ Easier to extend (add new cases by updating data, not code)
- ✅ Fewer bugs (no risk of fixing one copy but not others)

**When to apply DRY**:
- ✅ **During implementation**: Actively look for patterns as you write
- ✅ **During code review**: Ask "can this be simplified?"
- ✅ **Before completion**: Scan for repetitive patterns and refactor
- ⚠️ **Balance**: Don't over-abstract! If the pattern only appears twice and differs significantly, duplication might be clearer
