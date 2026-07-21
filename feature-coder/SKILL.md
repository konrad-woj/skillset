---
name: feature-coder
description: Systematic feature development workflow for complex features and refactoring using planning-first TDD approach. Use when user asks to implement a feature, add functionality, refactor code, or build something complex that requires planning. Also trigger proactively when an implementation request has multiple steps, requires architectural decisions, or involves touching shared code. Triggers on "implement feature", "add functionality", "build this", "create this capability", "refactor", "refactoring", "clean up code", "unify code", "DRY".
---

# Feature Coder

A disciplined workflow for implementing complex features and refactoring: plan first, test first, implement incrementally, validate continuously.

## Core Principles

These principles exist because guessing wastes time — asking is always faster than fixing assumptions:

1. **Interactive first** — Ask before acting. When requirements are ambiguous or multiple approaches are valid, present the options and wait. The user is the domain expert.
2. **Tests before code** — Propose test cases, get approval, then implement. Tests define the contract and prevent guessing at intent.
3. **Plan before coding** — Write a plan in markdown first; iterate until the user gives explicit approval before touching code.
4. **Incremental** — Implement one piece at a time. Show diffs for each change; get approval before applying.
5. **Data model first** — Define types and models before using them. Read actual data structures; never assume.
6. **Context managers always** — Use `with`/`async with` for all resources. Never manual `.close()`.
7. **Plain comments only** — Use standard `#` comments with normal ASCII characters. Never use decorative box-drawing or divider characters (e.g. `═══════`, `───────`, `╔╗╚╝`) to create visual section banners — these are a giveaway of AI-generated code, not something a human engineer writes.

## Terminology

- **Stub test**: Test body with only `pass` or `# TODO` — not yet implemented, not counted as passing
- **Active test**: Test with real assertions — the only kind that counts toward coverage
- **Draft**: Function skeleton with signature, docstring, and `raise NotImplementedError()`

---

## New Feature Workflow (6 Phases)

### Phase 1: Planning (Documentation Only)

Interview the user about requirements, async/sync choice, caching opportunities, and constraints. Read existing code to understand patterns. Write `{FEATURE_NAME}_PLAN.md` and iterate until the user gives **explicit approval** ("looks good", "let's implement").

Key interview questions: async or sync? which asyncio patterns? what can be cached? what shared packages exist to reuse?

If the feature is async, FastAPI-based, a background worker/queue consumer, or will run across multiple instances/replicas, also invoke the `python-async-scaling` skill during planning and again during implementation (Phases 1 and 5) — it covers event-loop-blocking pitfalls, cross-instance concurrency limits, idempotency, and resilience patterns that this skill doesn't duplicate. Skip it for plainly synchronous, single-process work.

→ **[references/planning.md](references/planning.md)** — Interview steps, plan structure, iteration patterns

### Phase 2: Configuration & Data Models

Set up YAML configs in `conf/` and define all Pydantic models/TypedDicts/dataclasses before writing any business logic.

→ **[references/models-and-config.md](references/models-and-config.md)**

### Phase 3: Test Stubs

Write test stubs with `pass` bodies that define the expected API through tests — executable specifications. Show them to the user and get approval before proceeding.

→ **[references/test-stubs.md](references/test-stubs.md)**

### Phase 4: Draft Implementation

Create skeletons of all modules, classes, and functions with complete signatures, docstrings, type hints, and `raise NotImplementedError()` bodies. Let the user review the API design. Wait for **explicit approval** before Phase 5.

→ **[references/draft-implementation.md](references/draft-implementation.md)**

### Phase 5: TDD Implementation

Uncomment tests one at a time; implement logic to make each pass. Before claiming any tests pass, verify no stubs remain:

```bash
# Primary — slash command
/check-inactive-tests tests/unit/test_*.py

# Fallback
grep -E "(^#.*def test_|^\s+pass\s*$)" tests/unit/test_*.py
```

→ **[references/tdd-implementation.md](references/tdd-implementation.md)**

### Phase 6: Integration & Validation

Run full test suite, move all imports to module level, refactor globals, run typecheck and linting.

```bash
uv run pytest tests/unit/test_{module}.py -v
uv run task typecheck
uv run task lint --fix --unsafe-fixes
uv run task save-api  # after API changes
```

→ **[references/integration-validation.md](references/integration-validation.md)**

---

## Refactoring Workflow

Use this instead of the 6-phase flow when the task is "clean up", "unify", or "DRY".

### R1: Discovery — Scan and present

Scan for the patterns mentioned. Present findings as a numbered list with `file:line` references. Ask which items to address (not assumed "all").

### R2: Direction — Decide where code lives

For each duplication, present options: extract to shared package (`data-utils`, etc.), keep local, or consolidate to existing version A or B. Wait for explicit choice before proceeding.

### R3: Incremental changes with diffs

For each change:
1. Show **BEFORE** block with file:lines
2. Show **AFTER** block
3. Explain **IMPACT** (what other code is affected)
4. Ask: "Proceed? (yes/no/modify)"
5. Only after "yes": apply, run tests, report results
6. Move to the next change — never batch

### R4: Verification

Run full test suite and typecheck. Show a summary of all changes made. Ask if anything else needs refactoring.

---

## Test-First Interaction Pattern

When starting any implementation, first propose test cases:

```
Before I implement this, let's define what "working" looks like.

I suggest:
1. test_basic_case — input X → expected output Y
2. test_edge_case_empty — empty input → returns None
3. test_error_handling — invalid input → raises ValueError

Does this cover your expectations? Add/remove/modify?
```

Show each test's code and get approval before writing it. Run the tests expecting failure, then implement, then report: "✓ Test X passes."

---

## Progress Tracking

Use TodoWrite to track phase progress. Mark tasks completed immediately after finishing, not in batches.

```python
TodoWrite([
    {"content": "Write and validate plan", "status": "completed", ...},
    {"content": "Create data models", "status": "in_progress", ...},
    ...
])
```

---

## Reference Files

| File | When to read |
|---|---|
| [references/planning.md](references/planning.md) | Phase 1 — interview and plan structure |
| [references/models-and-config.md](references/models-and-config.md) | Phase 2 — YAML config, Pydantic models |
| [references/test-stubs.md](references/test-stubs.md) | Phase 3 — writing test stubs |
| [references/draft-implementation.md](references/draft-implementation.md) | Phase 4 — skeleton with NotImplementedError |
| [references/tdd-implementation.md](references/tdd-implementation.md) | Phase 5 — uncommenting and implementing |
| [references/integration-validation.md](references/integration-validation.md) | Phase 6 — final validation |
| [references/best-practices.md](references/best-practices.md) | Codebase conventions: shared packages, caching, imports (defers to `python-async-scaling` for async patterns) |
| [references/common-mistakes.md](references/common-mistakes.md) | 15 common mistakes with examples and fixes |
| [references/examples.md](references/examples.md) | Worked examples of all three workflow modes |

## Related Skills

| Skill | Invoke alongside this one when |
|---|---|
| `python-async-scaling` | The feature/refactor is async, FastAPI-based, a background worker/queue consumer, or scales across multiple instances/replicas. Not needed for plainly synchronous, single-process work. |
