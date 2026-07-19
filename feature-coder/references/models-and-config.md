# Phase 2: Configuration & Data Models (Code Starts)

This guide covers setting up configuration and defining all data structures before using them.

## Goal

Set up configuration and define all data structures before using them.

## Steps

### Step 2.1: Configuration Setup

- Create YAML config files in `conf/` directory
- Main config typically named `default.yaml`
- Shared/reusable configs use underscore prefix (e.g., `_paths.yaml`, `_validation.yaml`)
- Import shared configs using Hydra defaults list in `default.yaml`
- Test config loads correctly with a simple script

### Step 2.2: Create/Update Data Models

- Check if models exist in `../data-models/` that you can reuse
- Create Pydantic models (use BaseSchema for API contracts), TypedDicts, or dataclasses
- Include all required fields based on plan
- Add proper types and Google-style docstrings
- **Verify** the model matches actual usage in plan

### Step 2.3: Validate Models

- Check they compile (run `uv run task typecheck`)
- Verify they match the planned interfaces
- User reviews model definitions

**Critical**: If you later find a model is missing fields, you made a mistake. The model should match actual usage from the start.
