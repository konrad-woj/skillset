# Phase 6: Integration and Validation

This guide covers verifying everything works together and finalizing the implementation.

## Goal

Verify everything works together, ensure all tests are active, and ensure code quality meets standards.

## Steps

### Step 6.1: Verify Active Tests First

**MANDATORY: Before running pytest, verify no stub/commented tests remain**

```bash
# Use slash command to detect inactive tests
/check-inactive-tests tests/unit/test_{module}.py
```

**Only proceed to Step 6.2 if all tests are active.** If inactive tests are found:
- Implement remaining stub tests
- Remove low-value stub tests
- Update TODO list with remaining work

### Step 6.2: Run All Tests

**After verifying all tests are active**, run the test suite:

```bash
uv run pytest tests/unit/test_{module}.py -v
uv run pytest tests/integration/test_{module}.py -v  # if applicable
```

### Step 6.3: Refactor Code Quality Issues

**CRITICAL - Clean up development patterns before finalizing**:

1. **Scan for local imports**:
   - Search files for `import` statements inside functions
   - Move all imports to module level (except justified lazy loading)
   - Organize: stdlib → third-party → local

2. **Scan for global variables**:
   - Search for `global` keyword usage
   - Refactor to:
     - Function parameters
     - Class instance attributes
     - Config values (in YAML)
     - Passed state objects

3. **Verify patterns**:
   - All resources use context managers
   - Caching applied where appropriate
   - Proper async patterns if async code

### Step 6.4: Type Check and Lint

```bash
uv run task typecheck
uv run task lint --fix --unsafe-fixes  # Fix any linting issues
```

### Step 6.5: Manual Testing (if applicable)

- Test actual usage scenarios
- Verify config loads correctly with Hydra
- Check integration with existing code

### Step 6.6: Pre-commit Validation (Optional)

Use the `precommit` skill to run comprehensive checks:
- Detects changed packages via git
- Runs format, lint, typecheck, tests, API validation
- Ensures library changes don't break dependent packages

## Refactoring Checklist

Before marking the feature as complete, ensure:

### Import Organization
- [ ] All imports moved to module level (except justified lazy loading)
- [ ] Imports organized: stdlib → third-party → local
- [ ] No function-level imports remaining (unless explicitly justified)

### Variable Scoping
- [ ] No `global` keyword usage
- [ ] State managed through parameters, class attributes, or config
- [ ] No module-level mutable state

### Resource Management
- [ ] All file operations use `with open(...) as f:`
- [ ] Thread/process pools use `with ThreadPoolExecutor() as executor:`
- [ ] Async resources use `async with`
- [ ] Database connections use context managers
- [ ] Locks/semaphores use `with lock:` or `async with lock:`
- [ ] No manual `.close()` calls

### Performance
- [ ] Caching applied where appropriate (`@lru_cache`, `@cache`, `@cached_property`)
- [ ] Expensive operations cached (file reads, configs, computations)
- [ ] Cache invalidation strategy considered

### Code Quality
- [ ] DRY principle applied - no repetitive patterns
- [ ] Helper functions extracted for common operations
- [ ] Similar operations consolidated into reusable functions
- [ ] Google-style docstrings on all public functions/classes
- [ ] Type hints on all function signatures

### Testing
- [ ] **MANDATORY: Run `/check-inactive-tests tests/unit/test_{module}.py` to verify no stub/commented tests**
- [ ] All tests uncommented and have real assertions
- [ ] Mocks properly configured (AsyncMock for async code)
- [ ] No real resources loaded in unit tests
- [ ] Parametrized tests used for multiple similar cases
- [ ] Test data isolates features being tested

### Async (if applicable)
- [ ] Proper asyncio patterns used (Semaphore, gather, etc.)
- [ ] Async context managers used for resources
- [ ] AsyncMock used in tests
- [ ] Tests marked with `@pytest.mark.asyncio`
