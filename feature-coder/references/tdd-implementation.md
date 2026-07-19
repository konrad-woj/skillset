# Phase 5: Test-Driven Implementation

This guide covers implementing the actual logic by uncommenting tests one at a time and making them pass.

## Goal

Replace `raise NotImplementedError()` with actual implementation logic, guided by tests. Implement incrementally - one test at a time.

## Steps

### Step 5.1: Implement Incrementally

For each feature/function, follow this cycle:

1. **Uncomment ONE test** (or group of closely related tests)
   - Remove `pass` statement
   - Remove `# TODO` comments
   - Add real assertions and test logic
   - Configure mocks with return values

2. **Implement the minimal code** to make that test pass
   - Go to the function with `raise NotImplementedError()`
   - Replace with actual implementation
   - Write only enough code to pass THIS test
   - Don't over-engineer - keep it simple

3. **Validate mock setup before running test**:
   - Double-check that mocked objects are still accessible when assertions run
   - Verify the test doesn't accidentally load actual models, data, or external clients
   - Ensure context managers don't close mocked resources prematurely
   - Confirm mock scope matches test execution lifecycle

4. **Run the test** - verify it passes
   ```bash
   uv run pytest tests/unit/test_{module}.py::test_specific_test -v
   ```

5. **Fix any issues** until test passes
   - Debug failures
   - Adjust implementation
   - Fix mock setup if needed
   - Re-run test

6. **Refactor if needed** (but keep it simple)
   - Extract helper methods if code is repetitive (DRY principle)
   - Improve readability
   - Don't over-abstract

7. **Move to next test** and repeat

### Step 5.2: Verify Active Tests Periodically

**Run periodically (every 3-5 tests implemented)**:
```bash
/check-inactive-tests tests/unit/test_{module}.py
```

This ensures you're not accidentally skipping tests or leaving stubs behind.

### Step 5.3: Critical Pattern - One Test at a Time

**DO** (Correct approach):
```python
# Cycle 1: Uncomment first test
def test_score_field_returns_valid_result(self, scorer):
    """Test that score_field returns a valid ScoringResult."""
    result = scorer.score_field(
        source="Gate Valve",
        target="Válvula de Compuerta",
        entity_type="material",
        language="es"
    )
    assert isinstance(result, ScoringResult)
    assert 0.0 <= result.confidence_score <= 1.0

# Other tests still have pass
def test_length_ratio_check(self):
    pass  # Not implemented yet

def test_similarity_check(self):
    pass  # Not implemented yet
```

Implement `score_field` to make first test pass, then move to next test.

**DON'T** (Wrong approach):
```python
# ❌ Don't uncomment all tests at once
def test_score_field_returns_valid_result(self, scorer):
    result = scorer.score_field(...)
    assert isinstance(result, ScoringResult)

def test_length_ratio_check(self):
    result = scorer.score_field(...)
    assert len(result.issues) > 0

def test_similarity_check(self):
    result = scorer.score_field(...)
    assert result.confidence_score < 0.5

# ❌ Then trying to implement everything at once
```

### Step 5.4: Implementation Example

**Before (from Phase 4)**:
```python
def score_field(
    self,
    source: str,
    target: str,
    entity_type: str,
    language: str,
) -> ScoringResult:
    """Score a single translated field.

    Args:
        source: Original source text
        target: Translated target text
        entity_type: Type of entity
        language: Target language code

    Returns:
        ScoringResult with confidence score and issues
    """
    raise NotImplementedError("score_field will be implemented in Phase 5")
```

**After (Phase 5 - first iteration for basic test)**:
```python
def score_field(
    self,
    source: str,
    target: str,
    entity_type: str,
    language: str,
) -> ScoringResult:
    """Score a single translated field.

    Args:
        source: Original source text
        target: Translated target text
        entity_type: Type of entity
        language: Target language code

    Returns:
        ScoringResult with confidence score and issues
    """
    # Basic validation
    if not source or not target:
        raise ValueError("source and target cannot be empty")

    # Initialize issues list
    issues: List[ScoringIssue] = []

    # Run all checks
    length_issue = self._check_length_ratio(source, target)
    if length_issue:
        issues.append(length_issue)

    similarity, similarity_issue = self._check_semantic_similarity(source, target)
    if similarity_issue:
        issues.append(similarity_issue)

    # Calculate confidence score
    base_confidence = 1.0
    for issue in issues:
        if issue.severity == Severity.ERROR:
            base_confidence -= 0.3
        elif issue.severity == Severity.WARNING:
            base_confidence -= 0.15

    confidence_score = max(0.0, base_confidence)

    return ScoringResult(
        confidence_score=confidence_score,
        issues=issues,
        semantic_similarity=similarity,
    )
```

Then implement each helper method as its tests are uncommented.

### Step 5.5: Async Implementation Example

**Before**:
```python
async def process_batch(
    self,
    items: List[str],
    max_concurrent: int = 10,
) -> List[Result]:
    """Process items concurrently with rate limiting."""
    raise NotImplementedError("process_batch will be implemented in Phase 5")
```

**After** (implementing with asyncio.Semaphore as planned):
```python
async def process_batch(
    self,
    items: List[str],
    max_concurrent: int = 10,
) -> List[Result]:
    """Process items concurrently with rate limiting."""
    semaphore = asyncio.Semaphore(max_concurrent)

    async def process_with_limit(item: str) -> Result:
        async with semaphore:
            return await self._process_single(item)

    tasks = [process_with_limit(item) for item in items]
    return await asyncio.gather(*tasks)
```

Before finalizing an implementation like this, invoke `python-async-scaling` to check whether `max_concurrent` needs to account for running across multiple replicas (an in-process `Semaphore` doesn't coordinate across pods) — see its `decision-guide.md`.

### Step 5.6: Keep Code DRY

As you implement, watch for repetitive patterns and extract them:

**BAD** (repetitive):
```python
def _check_length_ratio(self, source: str, target: str) -> Optional[ScoringIssue]:
    ratio = len(target) / len(source)
    if ratio > self.length_ratio_threshold:
        return ScoringIssue(
            issue_type=ScoringIssueType.LENGTH_MISMATCH,
            description=f"Length ratio {ratio:.2f} exceeds threshold {self.length_ratio_threshold}",
            severity=Severity.WARNING
        )
    return None

def _check_numeric_consistency(self, source: str, target: str) -> Optional[ScoringIssue]:
    source_nums = extract_numbers(source)
    target_nums = extract_numbers(target)
    consistency = calculate_match(source_nums, target_nums)
    if consistency < self.numeric_threshold:
        return ScoringIssue(
            issue_type=ScoringIssueType.NUMERIC_MISMATCH,
            description=f"Numeric consistency {consistency:.2f} below threshold {self.numeric_threshold}",
            severity=Severity.WARNING
        )
    return None
```

**GOOD** (DRY with helper):
```python
def _check_threshold(
    self,
    value: float,
    threshold: float,
    comparison: Literal["above", "below"],
    issue_type: ScoringIssueType,
    metric_name: str,
) -> Optional[ScoringIssue]:
    """Generic threshold checker."""
    exceeds = (comparison == "above" and value > threshold) or \
              (comparison == "below" and value < threshold)

    if exceeds:
        direction = "exceeds" if comparison == "above" else "below"
        return ScoringIssue(
            issue_type=issue_type,
            description=f"{metric_name} {value:.2f} {direction} threshold {threshold}",
            severity=Severity.WARNING
        )
    return None

def _check_length_ratio(self, source: str, target: str) -> Optional[ScoringIssue]:
    ratio = len(target) / len(source)
    return self._check_threshold(
        ratio, self.length_ratio_threshold, "above",
        ScoringIssueType.LENGTH_MISMATCH, "Length ratio"
    )

def _check_numeric_consistency(self, source: str, target: str) -> Optional[ScoringIssue]:
    consistency = calculate_match(
        extract_numbers(source),
        extract_numbers(target)
    )
    return self._check_threshold(
        consistency, self.numeric_threshold, "below",
        ScoringIssueType.NUMERIC_MISMATCH, "Numeric consistency"
    )
```

### Step 5.7: Use Parametrized Tests

When implementing tests with similar patterns, use `@pytest.mark.parametrize`:

**Example**:
```python
@pytest.mark.parametrize("source,target,expected_has_issue", [
    ("Valve", "Válvula", False),  # Good ratio
    ("Valve", "Válvula de compuerta muy grande con detalles", True),  # Bad ratio
    ("Pump", "Bomba", False),  # Good ratio
    ("A", "This is a very long translation of a single character", True),  # Bad ratio
])
def test_length_ratio_detection(self, scorer, source, target, expected_has_issue):
    """Test length ratio issue detection for various inputs."""
    result = scorer.score_field(source, target, "material", "es")
    length_issues = [i for i in result.issues if i.issue_type == ScoringIssueType.LENGTH_MISMATCH]

    if expected_has_issue:
        assert len(length_issues) > 0, f"Expected length issue for {source} -> {target}"
    else:
        assert len(length_issues) == 0, f"Unexpected length issue for {source} -> {target}"
```

### Step 5.8: MANDATORY - Verify All Tests Active Before Claiming Completion

**Before claiming "tests pass" or "implementation complete"**:

```bash
/check-inactive-tests tests/unit/test_{module}.py
```

**Expected output if all tests are active**:
```
📊 Test Status for test_scorer.py
============================================================
Total test functions: 28
✅ Active tests (with assertions): 28
⚠️  Stub tests (only 'pass'): 0
❌ Commented tests: 0

✅ All 28 tests are active with real assertions!
```

**If inactive tests found**:
- Implement remaining stub tests
- Remove low-value stub tests that aren't needed
- Update TODO list with remaining work
- **NEVER claim completion with inactive tests present**

### Step 5.9: Run All Tests

After all tests are active, run the complete test suite:

```bash
# Run all unit tests for the module
uv run pytest tests/unit/test_{module}.py -v

# Run with coverage if desired
uv run pytest tests/unit/test_{module}.py --cov=src/{package}/{module} -v
```

**Expected**: All tests pass ✅

## Critical Patterns

✅ **DO**:
- Uncomment tests one at a time
- Implement minimal code to pass each test
- Run `/check-inactive-tests` periodically
- Use `@pytest.mark.parametrize` for similar test cases
- Extract common patterns (DRY principle)
- Validate mock setup before running tests
- Use `AsyncMock` for async functions

❌ **DON'T**:
- Uncomment all tests at once
- Implement everything before testing
- Claim "tests pass" without running `/check-inactive-tests`
- Count stub tests (only `pass`) as passing
- Skip test verification
- Load real resources (models, APIs, files) in unit tests
- Use regular `MagicMock` for async functions

## Common Issues and Solutions

### Issue 1: Test Fails with Mock Not Called

**Problem**: `mock.assert_called_once()` fails

**Solution**: Check that the code path actually calls the mocked function
```python
# Make sure your implementation actually calls the mocked function
def score_field(self, source, target, ...):
    similarity = self.similarity_model.calculate(source, target)  # Must call mocked method
```

### Issue 2: Async Test Hangs or Mocks Misbehave

**Problem**: Test never completes, or a mocked async function raises `TypeError: object MagicMock can't be used in 'await' expression`

**Solution**: Missing `await` and `MagicMock` vs. `AsyncMock` mixups are the two most common causes. See `python-async-scaling`'s `references/testing-async-code.md` for the full set of async-testing pitfalls (mock types, leaked tasks, idempotency under retries) rather than debugging from scratch.

### Issue 3: Real Resources Loaded in Unit Test

**Problem**: Test loads actual 500MB model

**Solution**: Mock the loading function
```python
@pytest.fixture
def mock_model_loader(self, mocker):
    mock = mocker.patch("my_package.model.load_model")
    mock.return_value = MagicMock()  # Fake model
    return mock
```

### Issue 4: Context Manager Closes Mock

**Problem**: Mock is closed before assertion

**Solution**: Don't use context manager in fixture
```python
# ❌ BAD
@pytest.fixture
def mock_client(self, mocker):
    with mocker.patch("module.Client") as mock:
        return mock  # Context exits, mock closed!

# ✅ GOOD
@pytest.fixture
def mock_client(self, mocker):
    mock = mocker.patch("module.Client")
    return mock  # No context manager, mock stays alive
```

## Next Phase

After all tests pass and `/check-inactive-tests` confirms all tests are active, proceed to **Phase 6: Integration and Validation** where you'll:
- Run `/check-inactive-tests` again (mandatory first step)
- Run full test suite
- Refactor code quality issues (imports, globals, DRY)
- Run type checking and linting
- Finalize the implementation
