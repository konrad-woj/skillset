# Maintainability Review Patterns

## Code Organization

### Function Length
- **Check**: Functions longer than 50 lines
- **Symptom**: Multiple responsibilities, hard to test
- **Good**: Extract logical sections into helper functions
- **Example**: Extract validation, data transformation, business logic

### Class Responsibilities
- **Check**: Classes doing multiple unrelated things
- **Principle**: Single Responsibility Principle
- **Good**: Split into focused classes

### Cyclomatic Complexity
- **Check**: Deeply nested conditionals (>3 levels)
- **Good**: Extract conditions into named functions or use early returns

## Naming & Clarity

### Unclear Names
- **Check**: Variable names like `data`, `tmp`, `x`, `result`
- **Bad**: `def process(x): return f(x)`
- **Good**: `def extract_features(document: Document) -> Features`

### Magic Numbers
- **Check**: Hardcoded numbers without explanation
- **Bad**: `if score > 0.85:`
- **Good**: `CONFIDENCE_THRESHOLD = 0.85; if score > CONFIDENCE_THRESHOLD:`

### Boolean Flags
- **Check**: Boolean parameters that change behavior
- **Bad**: `def fetch_data(use_cache: bool)`
- **Good**: Split into `fetch_data()` and `fetch_fresh_data()`

## Error Handling

### Bare Except
- **Check**: `except:` or `except Exception:` without specific handling
- **Bad**: Silently catching all exceptions
- **Good**: Catch specific exceptions, log unexpected ones

### Missing Validation
- **Check**: No validation before operations
- **Example**: Using dictionary keys without checking existence
- **Good**: Validate inputs early with Pydantic or explicit checks

### Error Context
- **Check**: Generic error messages without context
- **Bad**: `raise ValueError("Invalid input")`
- **Good**: `raise ValueError(f"Invalid input: expected positive number, got {value}")`

## Type Safety

### Missing Type Hints
- **Check**: Functions without type hints (especially public APIs)
- **Good**: Add type hints for parameters and return values
- **Note**: Focus on public interfaces and complex functions

### Type: ignore Overuse
- **Check**: Multiple `# type: ignore` without explanations
- **Good**: Fix the type issue or add specific ignore with explanation

### Any Type
- **Check**: Excessive use of `Any` type
- **Good**: Use specific types or generic types

## Testing & Testability

### Hard to Test
- **Check**: Functions with side effects, external dependencies in logic
- **Good**: Inject dependencies, separate pure logic from I/O

### Missing Test Coverage
- **Check**: Complex logic without tests
- **Priority**: Business logic, edge cases, error handling

### Test Quality
- **Check**: Tests without assertions or that test framework code
- **Good**: Clear arrange-act-assert structure, meaningful assertions

## Documentation

### Missing Docstrings
- **Check**: Public functions/classes without docstrings
- **Good**: Add Google-style docstrings with Args, Returns, Raises

### Outdated Comments
- **Check**: Comments contradicting code
- **Good**: Update or remove outdated comments

### Self-Documenting Code
- **Check**: Comments explaining what code does (not why)
- **Good**: Use clear names and structure; add comments only for "why"

## Dependency Management

### Tight Coupling
- **Check**: Direct imports of implementation details from other packages
- **Good**: Use defined interfaces or abstraction layers

### Circular Dependencies
- **Check**: Package A imports B, B imports A
- **Good**: Restructure to eliminate cycle (extract common, invert dependency)

## Configuration Management

### Hardcoded Config
- **Check**: Configuration values hardcoded in logic
- **Good**: Load from config files or environment variables

### Config Validation
- **Check**: No validation of configuration values
- **Good**: Use Pydantic models for config with validation

## Logging

### Missing Logs
- **Check**: Important operations without logging
- **Good**: Log key events, errors, performance metrics

### Log Levels
- **Check**: Everything logged at INFO or DEBUG
- **Good**: ERROR for errors, WARNING for unexpected situations, INFO for key events

### Structured Logging
- **Check**: String formatting in logs
- **Good**: Use structured logging with fields: `logger.info("user_login", user_id=123)`

## Code Duplication

### Identical Logic
- **Check**: Same logic duplicated across functions/files
- **Good**: Extract to shared utility function

### Similar Patterns
- **Check**: Similar code with slight variations
- **Good**: Parameterize differences, create generic version

### Existing Utilities
- **Check**: Reimplementing functionality that exists in:
  - `data-utils`: Generic utilities
  - `data-models`: Pydantic models
  - `llm-clients`: LLM wrappers
  - `logger`: Logging utilities
- **Action**: Suggest using existing implementation

## Monorepo-Specific Patterns

### Cross-Package Duplication
- **Check**: Similar logic exists in another package
- **Search**: Use grep to find similar patterns in:
  - `packages/data-utils/`
  - `packages/data-models/`
  - Other package-specific utilities
- **Suggest**: Consider moving to shared location if used by 2+ packages

### Package Dependencies
- **Check**: Depending on package that's not in `pyproject.toml`
- **Good**: Declare all package dependencies explicitly

## Review Approach

### What to Flag
1. **Major maintainability issues**: Hard to understand, test, or modify
2. **Missing best practices**: No tests for complex logic, no docstrings on public APIs
3. **Code duplication**: Especially if similar code exists in `data-utils` or other packages

### What to Skip
1. **Style issues**: Handled by formatters (black, isort)
2. **Minor naming**: Unless genuinely confusing
3. **Personal preferences**: If code is clear and functional

### How to Review
1. **Start with high-level structure**: Does organization make sense?
2. **Check for duplication**: Is this reinventing existing utilities?
3. **Focus on complexity**: Are there simpler ways to achieve the same goal?
4. **Consider testability**: Can this be easily tested?

### Feedback Format
Provide:
- Specific issue and why it matters
- Concrete suggestion with code example
- Location reference (file:line)
