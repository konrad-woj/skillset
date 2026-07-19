# Phase 1: Planning (Documentation Only)

This guide covers the planning phase where you create a comprehensive implementation plan that the user validates BEFORE any code is written.

## Goal

Create a comprehensive implementation plan that the user validates BEFORE any code.

## Steps

### Step 1.0: Interview User About Implementation

**CRITICAL - Always start here**:

- Ask the user about implementation details, requirements, and constraints
- Clarify the feature's purpose, scope, and expected behavior
- Understand user preferences for architecture, libraries, or approaches
- Identify any existing patterns or conventions to follow
- **Async vs Sync**: Determine if the feature should be async or sync
  - Ask: "Should this be implemented as async or sync?"
  - If async, invoke the `python-async-scaling` skill and use its `decision-guide.md` to pick the right primitive (gather vs. TaskGroup, semaphore vs. distributed lock, queue vs. BackgroundTasks, etc.) — it accounts for whether the choice needs to hold up across multiple replicas, which a primitive list alone doesn't capture
  - Inform user about trade-offs and help choose the best approach
  - **Remember**: Async code requires async test fixtures and `pytest-asyncio`
- **Caching opportunities**: Identify what can be cached
  - Ask: "Are there repeated computations or file reads we should cache?"
  - Consider: config loading, model initialization, expensive computations, API responses
  - Suggest: `@lru_cache`, `@cache`, `@cached_property` as starting points
  - Start simple; optimize later if needed
- Ask follow-up questions rather than making assumptions about intent or approach
- **Do not proceed to Step 1.1 until you have clear answers**

### Step 1.1: Gather Context

- Read relevant existing code to understand patterns
- Check data models, config files, existing implementations
- Look for reusable code in shared packages (`../data-models/`, `../data-utils/`, `../logger/`, etc.)
- Check if similar features exist that can be used as reference
- Understand the architecture and conventions
- **NEVER guess** - always verify actual structure

### Step 1.2: Draft Initial Plan

Create a markdown document (e.g., `{FEATURE_NAME}_PLAN.md`) with:
- Feature overview and goals
- Architecture/design decisions
- Step-by-step implementation phases
- Required data models and types
- **Configuration changes** - All config exposed in YAML files in `conf/` dir, managed by Hydra
- Test strategy (unit tests in `tests/unit/`, integration tests in `tests/integration/`)
- Dependencies and integration points
- Example usage (show actual commands like `uv run python -m {module}`)
- Evaluation criteria (how to verify it works correctly)

**Key Questions to Answer**:
- What existing patterns should we follow?
- What data structures are involved?
- What config changes are needed?
- How will this integrate with existing code?
- What tests are needed?

### Step 1.3: Iterate on Plan with User

- Present plan to user
- User will spot issues, missing pieces, or improvements
- **Listen to feedback carefully** - user knows the domain
- Refine plan based on feedback
- Repeat until user approves
- **CRITICAL: Do not exit planning phase without explicit user approval** (e.g., "looks good", "let's implement", "approved")

**Common User Feedback Patterns**:
- "Are all mentioned X still valid?" - Verify no obsolete references
- "Do we have examples?" - Add usage examples
- "What about Y?" - Add missing considerations
- "That won't work because..." - User knows constraints you don't
