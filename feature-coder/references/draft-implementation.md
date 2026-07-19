# Phase 4: Draft Implementation (Skeleton)

This guide covers creating the skeleton of all modules, classes, and functions with signatures and docstrings but NO implementation logic.

## Goal

Create the complete API skeleton: all function signatures, class structures, docstrings, and type hints - but with `raise NotImplementedError()` instead of actual logic. This allows the user to review the API design before you invest time in implementation.

## Why This Phase Exists

**The Problem**: Jumping straight from test stubs to implementation often results in:
- Discovering API design issues mid-implementation
- Wasting time implementing the wrong interface
- Costly refactoring when the approach needs to change

**The Solution**: Draft the entire API first:
- User can review the design before implementation starts
- Type checking catches interface issues early
- Clear separation between design and implementation
- Easy to pivot if the API needs changes

## Steps

### Step 4.1: Create Module Structure

Create the necessary Python files in the correct locations:

```bash
# Example structure
src/package_name/
├── __init__.py
├── scorer.py          # Main module
├── checks.py          # Helper module
└── types.py           # Type definitions (if not in data-models)
```

**Guidelines**:
- Follow existing package structure conventions
- Place files in `src/{package_name}/` directory
- Create `__init__.py` only if the module needs public exports
- Check if similar modules exist to follow naming conventions

### Step 4.2: Write Class Definitions

For each class in the plan, write:
- Class name with proper inheritance
- Class docstring (Google style)
- `__init__` method with full signature and docstring
- Body with `pass` (for `__init__`) or `raise NotImplementedError()`

**Example**:
```python
from typing import Optional
from ..data_models import ScoringResult, ScoringIssue


class TranslationScorer:
    """Score translation quality with confidence calibration.

    This class evaluates translated text against source text using multiple
    quality checks including semantic similarity, length ratios, UNSPSC
    consistency, and historical patterns. It produces a confidence score
    with detailed issue reporting.

    Attributes:
        similarity_threshold: Minimum semantic similarity score (0.0-1.0)
        length_ratio_threshold: Maximum allowed target/source length ratio
        unspsc_enabled: Whether to perform UNSPSC consistency checks
    """

    def __init__(
        self,
        similarity_threshold: float = 0.8,
        length_ratio_threshold: float = 2.0,
        unspsc_enabled: bool = True,
    ):
        """Initialize the scorer with configuration thresholds.

        Args:
            similarity_threshold: Minimum semantic similarity score (0.0-1.0).
                Values below this trigger a warning.
            length_ratio_threshold: Maximum allowed target/source length ratio.
                Ratios above this indicate potential translation issues.
            unspsc_enabled: Whether to check UNSPSC code consistency.

        Raises:
            ValueError: If thresholds are out of valid range
        """
        # Basic validation - this is acceptable in __init__
        if not 0.0 <= similarity_threshold <= 1.0:
            raise ValueError(f"similarity_threshold must be 0.0-1.0, got {similarity_threshold}")
        if length_ratio_threshold <= 0:
            raise ValueError(f"length_ratio_threshold must be positive, got {length_ratio_threshold}")

        self.similarity_threshold = similarity_threshold
        self.length_ratio_threshold = length_ratio_threshold
        self.unspsc_enabled = unspsc_enabled
```

**What to include in `__init__`**:
- ✅ Parameter validation (simple checks)
- ✅ Attribute assignment
- ✅ Basic initialization
- ❌ NO complex logic
- ❌ NO external calls (APIs, databases, files)
- ❌ NO heavy computation

### Step 4.3: Write All Method/Function Signatures

For each method/function, write:
- Complete function signature (name, parameters, return type)
- Google-style docstring (Summary, Args, Returns, Raises)
- Type hints on ALL parameters and return values
- Body: `raise NotImplementedError("Method will be implemented in Phase 5")`

**Example**:
```python
def score_field(
    self,
    source: str,
    target: str,
    entity_type: str,
    language: str,
    context: Optional[dict] = None,
) -> ScoringResult:
    """Score a single translated field with confidence calibration.

    Evaluates translation quality by checking semantic similarity,
    length ratios, special character preservation, and UNSPSC consistency.
    Returns a confidence score (0.0-1.0) with detailed issue reporting.

    Args:
        source: Original source text to be translated
        target: Translated target text to evaluate
        entity_type: Type of entity (e.g., "material", "equipment", "document")
        language: Target language code (ISO 639-1, e.g., "es", "de")
        context: Optional context dict with additional metadata like UNSPSC codes

    Returns:
        ScoringResult containing:
            - confidence_score: Overall confidence (0.0-1.0)
            - issues: List of detected issues with severity levels
            - semantic_similarity: Computed similarity score if available

    Raises:
        ValueError: If source or target is empty, or language code is invalid

    Example:
        >>> scorer = TranslationScorer()
        >>> result = scorer.score_field(
        ...     source="Gate Valve",
        ...     target="Válvula de Compuerta",
        ...     entity_type="material",
        ...     language="es"
        ... )
        >>> result.confidence_score
        0.95
    """
    raise NotImplementedError("score_field will be implemented in Phase 5")

def _check_length_ratio(
    self,
    source: str,
    target: str,
) -> Optional[ScoringIssue]:
    """Check if target/source length ratio exceeds threshold.

    Compares the character length of target vs source text. Ratios
    significantly above 1.0 may indicate translation expansion issues,
    while very low ratios may indicate missing content.

    Args:
        source: Original source text
        target: Translated target text

    Returns:
        ScoringIssue if ratio exceeds threshold, None otherwise.
        Issue includes:
            - issue_type: LENGTH_MISMATCH
            - severity: WARNING
            - description: Actual ratio and threshold

    Example:
        >>> issue = scorer._check_length_ratio("VALVE", "VÁLVULA DE COMPUERTA GRANDE")
        >>> issue.severity
        Severity.WARNING
    """
    raise NotImplementedError("_check_length_ratio will be implemented in Phase 5")
```

**Docstring Requirements**:
- ✅ One-line summary (what it does)
- ✅ Detailed explanation (how it works, why it exists)
- ✅ Args: Each parameter with description
- ✅ Returns: What it returns, including structure for complex types
- ✅ Raises: All exceptions that can be raised
- ✅ Example: Optional but helpful for complex functions
- ✅ Type hints in signature, not in docstring (use modern style)

**Private vs Public**:
- Private methods: Prefix with `_` (e.g., `_check_length_ratio`)
- Public methods: No prefix (e.g., `score_field`)
- Helper functions: Module-level functions for generic utilities

### Step 4.4: Add Module-Level Functions (if needed)

For generic, reusable functions not tied to a class:

```python
def calculate_semantic_similarity(
    text1: str,
    text2: str,
    model: str = "sentence-transformers",
) -> float:
    """Calculate semantic similarity between two texts.

    Uses sentence embeddings to compute cosine similarity between
    the semantic representations of two text strings.

    Args:
        text1: First text to compare
        text2: Second text to compare
        model: Embedding model name to use

    Returns:
        Similarity score between 0.0 (no similarity) and 1.0 (identical)

    Raises:
        ValueError: If texts are empty
        RuntimeError: If model fails to load
    """
    raise NotImplementedError("calculate_semantic_similarity will be implemented in Phase 5")
```

### Step 4.5: Add Imports and Exports

**At the top of each module**:
```python
"""Module for translation quality scoring with confidence calibration.

This module provides the TranslationScorer class which evaluates translated
text against source text using multiple quality metrics.
"""

# Standard library
from typing import Optional, List, Dict

# Third-party
import numpy as np
from pydantic import BaseModel

# Local imports
from ..data_models import ScoringResult, ScoringIssue, ScoringIssueType
from ..data_utils import normalize_text, extract_numbers
from ..logger import get_logger

logger = get_logger()
```

**In `__init__.py` (if needed)**:
```python
"""Package for translation quality scoring."""

from .scorer import TranslationScorer
from .checks import calculate_semantic_similarity

__all__ = [
    "TranslationScorer",
    "calculate_semantic_similarity",
]
```

**Guidelines**:
- Only import what's actually used
- Organize: stdlib → third-party → local
- Only create `__init__.py` exports if the module is meant to be imported by other packages
- Check existing package conventions for export patterns

### Step 4.6: Verify Skeleton Compiles

**MANDATORY: Run type checking to verify the skeleton is valid**

```bash
# From package directory
uv run task typecheck
```

**Fix any type errors**:
- Missing imports
- Wrong return types
- Incorrect type annotations
- Missing type hints

**Expected result**: Typecheck passes with 0 errors (warnings are okay for now)

### Step 4.7: User Reviews Skeleton Before Implementation

**CRITICAL: Do not proceed to Phase 5 without explicit user approval**

**Show the user**:
- All class definitions with `__init__` signatures
- All method/function signatures with docstrings
- Module structure and organization
- Import structure

**Ask the user**:
1. "Does the API look correct?"
2. "Are the method signatures what you expected?"
3. "Should any methods be added, removed, or renamed?"
4. "Do the docstrings accurately describe what each function should do?"
5. "Ready to proceed to implementation, or should we adjust the design?"

**Wait for explicit approval**:
- ✅ "Looks good, proceed"
- ✅ "API approved, implement it"
- ✅ "Go ahead"
- ❌ Don't proceed if user says "wait", "let me check", or asks questions

**Common feedback patterns**:
- "Add a method for X" → Add to skeleton, re-review
- "This should return Y not Z" → Fix return type, re-review
- "Combine these two methods" → Refactor skeleton, re-review
- "This is too complex" → Simplify, re-review

## What NOT to Include

❌ **No implementation logic** - Only signatures and NotImplementedError
❌ **No business logic** - That comes in Phase 5
❌ **No external calls** - No API calls, database queries, file reads (except in simple `__init__`)
❌ **No complex computations** - No actual algorithms yet
❌ **No real data processing** - Only structure

## Example Complete Module Skeleton

```python
"""Scoring module for translation quality evaluation.

This module provides confidence calibration for machine-translated text
by evaluating multiple quality dimensions.
"""

from typing import Optional, List, Dict
from ..data_models import ScoringResult, ScoringIssue, ScoringIssueType, Severity
from ..logger import get_logger

logger = get_logger()


class TranslationScorer:
    """Score translation quality with confidence calibration.

    Evaluates translated text using semantic similarity, length ratios,
    UNSPSC consistency, and historical pattern analysis.
    """

    def __init__(
        self,
        similarity_threshold: float = 0.8,
        length_ratio_threshold: float = 2.0,
        unspsc_enabled: bool = True,
    ):
        """Initialize scorer with thresholds.

        Args:
            similarity_threshold: Min semantic similarity (0.0-1.0)
            length_ratio_threshold: Max length ratio threshold
            unspsc_enabled: Whether to check UNSPSC consistency

        Raises:
            ValueError: If thresholds out of valid range
        """
        if not 0.0 <= similarity_threshold <= 1.0:
            raise ValueError(f"similarity_threshold must be 0.0-1.0")
        if length_ratio_threshold <= 0:
            raise ValueError(f"length_ratio_threshold must be positive")

        self.similarity_threshold = similarity_threshold
        self.length_ratio_threshold = length_ratio_threshold
        self.unspsc_enabled = unspsc_enabled

    def score_field(
        self,
        source: str,
        target: str,
        entity_type: str,
        language: str,
        context: Optional[Dict] = None,
    ) -> ScoringResult:
        """Score a single translated field.

        Args:
            source: Original source text
            target: Translated target text
            entity_type: Entity type being translated
            language: Target language code
            context: Optional context metadata

        Returns:
            ScoringResult with confidence and issues

        Raises:
            ValueError: If inputs invalid
        """
        raise NotImplementedError("score_field will be implemented in Phase 5")

    def _check_length_ratio(self, source: str, target: str) -> Optional[ScoringIssue]:
        """Check length ratio between source and target.

        Args:
            source: Source text
            target: Target text

        Returns:
            ScoringIssue if ratio exceeds threshold, None otherwise
        """
        raise NotImplementedError("_check_length_ratio will be implemented in Phase 5")

    def _check_semantic_similarity(
        self,
        source: str,
        target: str,
    ) -> tuple[float, Optional[ScoringIssue]]:
        """Check semantic similarity between source and target.

        Args:
            source: Source text
            target: Target text

        Returns:
            Tuple of (similarity_score, optional_issue)
        """
        raise NotImplementedError("_check_semantic_similarity will be implemented in Phase 5")


def normalize_for_comparison(text: str) -> str:
    """Normalize text for comparison by removing extra whitespace.

    Args:
        text: Text to normalize

    Returns:
        Normalized text
    """
    raise NotImplementedError("normalize_for_comparison will be implemented in Phase 5")
```

## Benefits

✅ **Early API validation** - User reviews design before implementation effort
✅ **Type safety** - Typecheck catches interface issues immediately
✅ **Clear contracts** - Docstrings define exact behavior expected
✅ **Less rework** - Fix design issues before writing logic
✅ **Better planning** - See full scope clearly
✅ **Easier to pivot** - Changing signatures is cheap before implementation
✅ **Tests alignment** - Verify test expectations match drafted API

## Next Phase

After user approval, proceed to **Phase 5: Test-Driven Implementation** where you'll:
- Uncomment tests one by one
- Replace `NotImplementedError()` with actual logic
- Make each test pass incrementally
