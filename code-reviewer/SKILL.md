---
name: code-reviewer
description: Comprehensive code review for Python AI/ML microservices focusing on security, performance, maintainability, and code duplication. Use when explicitly asked to review code, review merge requests (MRs), or provide code feedback. Triggers on "review this code", "review my changes", "review MR", "code review", "check this code for issues", "add comments to MR", "post review comments". Reviews local code changes and GitLab merge requests with existing comments and discussions. Supports interactive comment workflow where each comment is reviewed and approved by user before posting to GitLab. Focuses on medium-severity issues (security vulnerabilities, bugs, performance problems, major maintainability issues) while avoiding minor style nitpicks. Intelligently detects functional code duplication across the monorepo by understanding what code does, not just text matching.
---

# Code Reviewer

Provides thorough code reviews for Python AI/ML microservices, focusing on security, performance, maintainability, and avoiding code duplication in a monorepo structure.

## Review Philosophy

**Focus on what matters:**
- Security vulnerabilities, bugs, performance bottlenecks, major maintainability issues
- Code duplication (functional similarity, not just text matching)
- Missing tests for complex logic
- Poor error handling or type safety
- Not: style issues handled by formatters (black, isort, ruff), minor naming nitpicks, personal preferences

**Be helpful, not annoying:**
- Provide specific, actionable feedback with code examples
- Explain why something matters (security risk, performance impact, etc.)
- Acknowledge good code and improvements
- Suggest alternatives when multiple valid approaches exist

## Review Workflow

### Step 1: Understand the Context

Determine what's being reviewed:

**For local code changes:**
```bash
git status
git diff
```

**For GitLab MR review:**
```bash
glab mr view <mr-number>
glab mr diff <mr-number>
glab api projects/<project-path-encoded>/merge_requests/<mr-number>/notes
```

**For GitHub PR review:**
```bash
gh pr view <pr-number>
gh pr diff <pr-number>
gh api repos/<owner>/<repo>/pulls/<pr-number>/comments
```

**For specific files:** Read them directly.

### Step 2: Load Documentation and Relevant References

**Always check for and read project documentation first:**

```bash
ls README.md PLAN.md DESIGN_DOC.md 2>/dev/null
```

Read whichever of these files exist. Use them as:
- **Context** — understand intent, architecture decisions, and scope before reviewing code
- **Review targets** — check whether the docs are accurate, up to date, and consistent with the code changes being reviewed. Flag discrepancies (e.g., PLAN.md describes a feature that was implemented differently, README.md has stale setup steps).

**Also check `pyproject.toml`:**
```bash
cat pyproject.toml
```
- Verify dev dependencies (`[tool.uv.dev-dependencies]` or `[project.optional-dependencies]` with a `dev` group) are separate from production dependencies (`[project.dependencies]`). Flag test/lint/type-check tools in production deps.
- Mind package versions and make sure that your suggestions are valid for the declared versions.
- Look for task runner entries (e.g., `[tool.taskipy.tasks]` for taskipy, or scripts defined under `[project.scripts]`). If they exist, factor them into the review — the reviewer should use them and flag if they fail or if relevant tasks are missing (e.g., no `precommits` check or missing `openapi` schema generation).

**Load reference files selectively — read only 1-2 per review based on code type:**

- **Always load**: `references/code-duplication-detection.md`
- **Security-sensitive code** (APIs, auth, data handling): `references/security-checklist.md`
- **Performance-critical code** (ML pipelines, data processing, async FastAPI services): `references/performance-patterns.md`
- **Code organization**: `references/maintainability-patterns.md`
- **GitLab MRs / GitHub PRs**: `references/vcs-integration.md`

### Step 3: Analyze the Code

**Review scope:**
- Small changes (<100 lines): thorough review of all aspects
- Medium (100–500 lines): focus on security, performance, and duplication
- Large (>500 lines): focus on critical issues only
- Stop at 3–5 significant issues — don't overwhelm

**Priority order:**

#### 3.1 Security
Scan for input validation (SQL/command injection, path traversal), auth problems, secrets in code, sensitive data logging, LLM prompt injection. See `references/security-checklist.md`.

#### 3.2 Functionality & Correctness
Does the code do what it should? Obvious bugs, logic errors, unhandled edge cases?

**Wrapper/facade API contract** — when new code calls a method on an existing object, wrapper, or session-state value, grep for other call sites of that object in the codebase and confirm the method actually exists on it. Wrapper classes often expose a narrower API than the underlying type (e.g. a `Logger` wrapper that only exposes `get_logger()`, not `.warning()` directly). A call that looks correct in isolation can raise `AttributeError` at runtime if it skips a required indirection step that every other caller uses.

**Side effects before flow-interrupting calls** — if the diff places a side effect (notification, toast, metric write, external API call, cache update) immediately before a call that interrupts or restarts execution flow (a framework rerun, redirect, early return, exception raise, or process exit), verify the side effect will actually be committed before the interruption. Many frameworks discard pending renders or buffered output when execution is redirected. If a utility already exists in the codebase for deferring or queuing such side effects, flag that it should be used here.

#### 3.3 Code Duplication
Use reasoning, not text matching. Follow `references/code-duplication-detection.md`:

1. Understand what the code *does* — identify semantic keywords based on purpose
2. Search intelligently in `packages/data-utils/`, `packages/data-models/`, `packages/llm-clients/`, `packages/logger/`, `packages/api-services-utils/`
3. Read matches — is it functionally the same?
4. Flag only true duplicates (same purpose, same approach, same logic)

#### 3.4 Performance
Check for N+1 queries, missing batching in ML operations, sync I/O that could be async, missing caching, memory inefficiencies. See `references/performance-patterns.md`.

**Always check for event loop blocking in async code**: any `async def` function that calls a sync CPU-bound or I/O operation (ML inference, file reads, subprocess) without `asyncio.to_thread()` or `run_in_executor()` freezes the entire event loop. On probed deployments (ACA, Kubernetes) this causes container restarts mid-request — flag as 🔴 Critical.

#### 3.5 Maintainability
Check for functions doing too much, poor naming, missing type hints on public APIs, missing docstrings on complex functions, hard-to-test code. See `references/maintainability-patterns.md`.

#### 3.6 Testing Coverage

Are there tests for complex logic? Are edge cases covered? Can the code be tested easily?

Before reading test files, locate all conftest files:

```bash
find tests/ -name conftest.py
```

Read every file found. Understand what fixtures, helpers, and shared setup already exist — then flag any test that reimplements something already available there.

For every test file in the diff, critically evaluate each test case:

- **Mutation resistance** — mentally break the implementation in a trivial way (flip a condition, remove a branch, return a hardcoded value) and ask whether this test would catch it. If the test still passes, it adds no value — flag it.
- **Tautologies** — tests that only assert that a mock returns what you told it to return, or that call the function and assert the result equals itself, add no value. Flag and recommend deletion.
- **Improbable scenarios** — edge cases so contrived they'll never occur in production waste maintenance time. Flag unless they guard against a known historical bug.
- **Flakiness** — tests that depend on wall-clock time, ordering of unordered collections, external services, or random seeds without being pinned. Flag the source and suggest a fix.
- **Conciseness** — if a test is longer than ~30 lines without being a complex integration test, ask whether it can be parameterized or split.

#### 3.7 Naming — Explicit Variable Names

Flag any variable, parameter, or loop binding that uses a single letter or a vague abbreviation: `i`, `j`, `k`, `it`, `el`, `a`, `b`, `tmp`, `res`, `obj`, `val`, `cfg`, `fn`, `cb`, `ctx` (unless `ctx` is a widely-accepted convention in the framework being used, e.g. FastAPI `Request`). Suggest an explicit name that conveys the domain meaning (e.g., `index` → `batch_index`, `el` → `document`, `res` → `search_result`). This is a hard style requirement — do not skip it even for short lambdas or comprehensions.

### Step 4: MR/PR Context (if applicable)

Read existing comments, check responses to previous feedback, focus on gaps. See `references/vcs-integration.md`.

### Step 4.5: Internal Critic Pass

This is a separate verification step with concrete re-reads — not reflection on what you already read. Write your draft findings first, then do the following for each one before presenting anything to the user.

**For each proposed issue:**

1. **Re-read the flagged lines and their surrounding context** (10–20 lines around the issue). Confirm the problem is actually there, not a misread.
   - Is the input actually user-controlled, or is it an internal value?
   - Is the code path actually reachable in production?

2. **Re-read the proposed fix in the context of the file** — paste it mentally into the location and trace through:
   - Does it handle the same edge cases as the original (None, empty, exception paths)?
   - Does it preserve the original behavior in the non-error path?
   - Does it introduce a new problem nearby (e.g., a performance fix that creates a new N+1 elsewhere, a security fix that validates before URL-decoding)?

3. **Check if the flagged line is a symptom, not the root cause** — re-read the caller or the function that produces the flagged value. If the real problem is upstream, flag that instead.

4. **Verify impact claims** — "this is a security vulnerability" or "this causes a performance problem" must be grounded in the actual code path, not a pattern match on the surface shape of the code.

**Actions to take based on this pass:**
- Drop findings where the problem disappears on re-read
- Revise the suggested fix if it has a bug or a non-obvious trade-off — and if the trade-off is worth the user knowing, say so explicitly
- Escalate or redirect findings where the root cause is elsewhere

### Step 5: Present Findings

**For local code review:**

```markdown
## Code Review

### 🔴 Critical Issues
[Security vulnerabilities, bugs that will cause failures]

**Location**: file.py:42
**Issue**: [Specific problem]
**Why it matters**: [Impact]
**Fix**:
[Code example]

### 🟡 Important Issues
[Performance problems, maintainability concerns, code duplication]

### 💡 Suggestions
[Optional improvements, alternative approaches]

### ✅ Good Practices
[Acknowledge good code]
```

**For GitLab MR review:**

```markdown
## MR Review: <title> (!<number>)

### Summary
- **Status**: [opened/merged/etc]
- **Your previous comments**: [count]
- **Unresolved discussions**: [count]

### Review Findings
#### 🔴 Critical Issues
#### 🟡 Important Issues
#### 💡 Suggestions
#### ✅ Responses to Your Comments

### Recommendation
[Approve / Request Changes / Comment only]
```

See `references/examples.md` for complete worked examples of each issue type.

### Step 6: Independent Subagent Critic Pass

After generating findings and before presenting them to the user, spawn a subagent with the following strict brief:

> "You are an adversarial reviewer. Below are proposed code review findings. Your job is NOT to confirm them — your job is to challenge each one. For every finding, answer: (1) Is this actually a problem, or is the reviewer pattern-matching on surface shape? (2) Is the proposed fix correct and complete, or does it introduce a new issue? (3) Is there a simpler or better alternative the reviewer missed? (4) Is this finding actually worth the author's time, given its real-world impact? If a finding survives all four questions, say so. If it doesn't, explain, change or remove it."

Pass the full draft findings to the subagent. Incorporate the subagent's output:
- Drop findings the subagent identifies as false positives or low-value
- Revise suggested fixes the subagent finds incomplete or incorrect
- Add any better alternatives the subagent surfaces
- Present only the reconciled, validated findings to the user

Do not show the subagent's raw critique to the user — only the improved final findings.

---

## Interactive Comment Workflow (GitLab MRs)

When user asks to "add comments" or "post comments" to an MR:

1. **Review and categorize** — perform the standard review above
2. **Present summary and wait for approval** — show all findings grouped by severity, ask "Would you like me to prepare comments?"
3. **Prepare drafts** — write comment files to `.review-comments/mr-<number>/`
4. **Review each comment interactively** — show preview, ask `[A]dd / [E]dit / [S]kip / [Q]uit`
5. **Final confirmation** — list all approved comments, ask `[P]ost / [R]eview again / [C]ancel`
6. **Post only after explicit confirmation** — never post without user saying "P"
7. **Confirm success** — report which comments posted, provide MR URL

See **`references/vcs-comments-workflow.md`** for full bash commands, API calls, file formats, and error handling for both GitLab and GitHub.

---

## Project-Specific Context

**Key directories:**
- `packages/data-utils/` — Generic cross-package utilities
- `packages/data-models/` — Shared Pydantic models
- `packages/llm-clients/` — LLM client wrappers
- `packages/logger/` — Logging utilities
- `packages/api-services-utils/` — API utilities

**Code style:**
- Google-style docstrings, PEP8
- `@staticmethod` for class-specific static methods; top-level functions for generic reusable logic
- Logging via `logger` package (never print statements)
- `BaseSchema` from `data-models` for API contracts

**Common patterns:** `uv` for package management, FastAPI for APIs, pytest for testing, structlog for logging

---

## Reference Files

| File | When to load |
|---|---|
| `references/code-duplication-detection.md` | **Always** — intelligent duplication detection |
| `references/security-checklist.md` | APIs, auth, data handling, external integrations |
| `references/performance-patterns.md` | ML pipelines, data processing, API endpoints |
| `references/maintainability-patterns.md` | Code structure, refactoring, organization |
| `references/vcs-integration.md` | GitLab MR / GitHub PR reviews |
| `references/vcs-comments-workflow.md` | Posting comments to GitLab MRs or GitHub PRs |
| `references/examples.md` | Worked examples: security, duplication, performance |
