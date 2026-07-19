# Phase 3: Test Stubs

This guide covers writing test stubs that define the expected API through tests.

## Goal

Write comprehensive test stubs with `pass` statements that define what the implementation should do. These tests serve as executable specifications.

## Steps

### Step 3.1: Write Test Stubs

Create test file with comprehensive test cases:

- **Create test file** in `tests/unit/test_{module}.py`
- **Write test method signatures** with clear, descriptive names
- **Add docstrings** explaining what each test verifies
- **Add TODO comments** with example test code showing the expected flow
- **Include fixtures** for mocks and test data
- **All tests should use `pass`** initially (no actual assertions yet)

**If feature is async**:
- Mark test functions with `@pytest.mark.asyncio`
- Use `async def` for test functions
- Use `await` for async calls in TODO comments
- Mock async functions with `AsyncMock` (not regular `MagicMock`)
- Create async fixtures with `@pytest.fixture(scope="...")`
- Example async patterns: mocking `asyncio.gather()`, `asyncio.Semaphore`, etc.

**CRITICAL: Plan mock setup carefully**:
- Ensure mocked objects will remain accessible when asserted (not closed by context managers)
- Confirm tests won't load actual models, data, or clients when they should be mocked
- Check that mocks will be properly scoped to the test lifecycle
- For async code: ensure async mocks (`AsyncMock`) are used, not sync mocks

**Example test stubs**:
```python
import pytest
from unittest.mock import MagicMock, AsyncMock, patch
from my_package.scorer import TranslationScorer
from my_package.types import ScoringResult, ScoringIssueType


class TestTranslationScorer:
    """Test suite for TranslationScorer."""

    @pytest.fixture
    def scorer(self):
        """Create a scorer instance with default config."""
        # TODO: Will instantiate scorer in Phase 5
        pass

    @pytest.fixture
    def mock_similarity_model(self, mocker):
        """Mock the similarity calculation model."""
        # TODO: Will mock external model in Phase 5
        pass

    def test_score_field_returns_valid_result(self, scorer):
        """Test that score_field returns a valid ScoringResult.

        This is a CRITICAL integration test - verify the main API works.
        """
        # TODO: Implement in Phase 5
        # result = scorer.score_field(
        #     source="Gate Valve",
        #     target="Válvula de Compuerta",
        #     entity_type="material",
        #     language="es"
        # )
        # assert isinstance(result, ScoringResult)
        # assert 0.0 <= result.confidence_score <= 1.0
        # assert isinstance(result.issues, list)
        pass

    def test_high_similarity_yields_high_confidence(self, scorer, mock_similarity_model):
        """Test that high semantic similarity produces high confidence score."""
        # TODO: Implement in Phase 5
        # mock_similarity_model.return_value = 0.95
        # result = scorer.score_field(
        #     source="Valve",
        #     target="Válvula",
        #     entity_type="material",
        #     language="es"
        # )
        # assert result.confidence_score > 0.85
        # assert len([i for i in result.issues if i.issue_type == ScoringIssueType.LOW_SIMILARITY]) == 0
        pass

    def test_length_ratio_exceeds_threshold_triggers_warning(self, scorer):
        """Test that excessive length ratio triggers a warning issue."""
        # TODO: Implement in Phase 5
        # result = scorer.score_field(
        #     source="Valve",  # 5 chars
        #     target="Válvula de compuerta muy grande",  # 35 chars, ratio 7.0
        #     entity_type="material",
        #     language="es"
        # )
        # length_issues = [i for i in result.issues if i.issue_type == ScoringIssueType.LENGTH_MISMATCH]
        # assert len(length_issues) > 0
        # assert length_issues[0].severity == Severity.WARNING
        pass

    @pytest.mark.parametrize("source,target,expected_ratio", [
        ("Valve", "Válvula", 1.2),
        ("Gate Valve", "Válvula de Compuerta", 2.1),
        ("Pump", "Bomba", 1.0),
    ])
    def test_length_ratio_calculation(self, source, target, expected_ratio):
        """Test length ratio calculation for various inputs."""
        # TODO: Implement in Phase 5 with parametrized test
        # ratio = calculate_length_ratio(source, target)
        # assert abs(ratio - expected_ratio) < 0.2
        pass

    # Add more test stubs as needed...
```

**Async test stub example**:
```python
@pytest.fixture
async def async_scorer(self):
    """Create async scorer instance."""
    # TODO: Will instantiate async scorer in Phase 5
    pass

@pytest.fixture
def mock_async_client(self, mocker):
    """Mock async API client."""
    mock = mocker.patch("my_package.client.AsyncClient", new_callable=AsyncMock)
    # TODO: Configure mock return values in Phase 5
    return mock

@pytest.mark.asyncio
async def test_async_score_field(self, async_scorer, mock_async_client):
    """Test async scoring with mocked API client."""
    # TODO: Implement in Phase 5
    # mock_async_client.get_similarity.return_value = 0.92
    # result = await async_scorer.score_field(
    #     source="Valve",
    #     target="Válvula",
    #     entity_type="material",
    #     language="es"
    # )
    # assert result.confidence_score > 0.85
    # mock_async_client.get_similarity.assert_called_once()
    pass
```

### Step 3.2: Review Test Quality Before Proceeding

**CRITICAL**: Before proceeding to Phase 4, review test suite for quality.

**Questions to ask**:
1. Do we have 3-5 critical integration tests for the main API?
2. Are there too many (>10) stub tests for edge cases?
3. Does the test data isolate features or trigger multiple penalties?
4. Do tests guide implementation toward controlled, testable code?

**Common issues to fix**:
- Remove low-value edge case stubs (empty text, special chars) until core works
- Consolidate similar tests into parametrized tests (use `@pytest.mark.parametrize`)
- Fix test data to avoid unrelated check failures
- Focus on high-value scenarios first

**Prioritize tests by value**:
1. **Critical (must have)**: 3-5 core API integration tests
2. **High value**: Key functionality variations and error paths
3. **Medium value**: Optional features and boundary conditions
4. **Low value**: Edge cases (can implement later)

### Step 3.3: User Approval Required

**Show cleaned-up test stubs to user for approval**

**CRITICAL: Do not exit Phase 3 without explicit user confirmation**

Wait for responses like:
- ✅ "Tests look good"
- ✅ "Proceed with implementation"
- ✅ "Approved"
- ✅ "Go ahead"

Do not proceed if user says:
- ❌ "Wait"
- ❌ "Let me check"
- ❌ Asks clarifying questions

**User may request changes**:
- Add missing test cases
- Remove low-value tests
- Consolidate similar tests
- Fix test data issues
- Adjust mock setup

After approval, proceed to **Phase 4: Draft Implementation** to create the function skeletons.

## What NOT to Include Yet

❌ **No real assertions** - Tests have `pass` only
❌ **No actual test logic** - Only TODO comments showing intent
❌ **No real mocks configured** - Just fixture structure
❌ **Don't uncomment tests** - They stay as stubs until Phase 5

## Benefits

✅ **Early test design review** - User validates test approach before implementation
✅ **Clear expectations** - Tests define what success looks like
✅ **Guided implementation** - Tests show what needs to be built
✅ **Parametrize planning** - Identify similar tests to combine early
✅ **Mock planning** - Think through dependencies before coding

## Next Phase

After user approval, proceed to **Phase 4: Draft Implementation** where you'll:
- Create module structure
- Write function/class signatures
- Add docstrings
- Use `raise NotImplementedError()` for bodies
- Get user approval on API design
