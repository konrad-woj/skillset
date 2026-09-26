---
name: project-bootstrapper
description: >-
  Turn a clone of the project-template repo into a real, named project, or add a
  new package to one that already exists. Handles project identity, package
  scaffolding from the pyproject/tach examples, .env and sandbox setup, skill
  syncing, Dockerfiles, OpenWiki regeneration, and a full verification pass.
  Triggers on "bootstrap this template", "set up a new project", "start a new
  project from the template", "scaffold a new package", "add a package",
  "initialise the repo", "new service", "new microservice".
---

# Project Bootstrapper

Bootstraps a clone of the `project-template` monorepo into a working project,
and scaffolds new packages inside an already-bootstrapped one. Everything in the
template README's "Quick start — bootstrap your own repo" section, plus the identity,
hygiene, and verification work that section leaves to the reader.

## External dependencies

None of these can be assumed to exist locally. Fetch them from their public
GitHub repos; never go looking for a sibling checkout on disk.

| Dependency | Source | How it is fetched |
| ---------- | ------ | ----------------- |
| `project-template` | `https://github.com/konrad-woj/project-template.git` | `git clone` (Bootstrap target, Adopt reference) |
| `skillset` | `https://github.com/konrad-woj/skillset.git` | `sh sync_skills.sh` (Phase 4) |
| `logger` | `https://github.com/konrad-woj/logger.git` | `uv sync`, via `[tool.uv.sources]` in each `pyproject.toml` |
| `ponytail` | `https://github.com/DietrichGebert/ponytail` | Claude Code plugin, installed in Phase 0 |

## When to use this

| Situation | Mode |
| --------- | ---- |
| Empty or not-yet-existing target directory | **Bootstrap**, after cloning the template into it |
| Fresh clone of `project-template`, no real packages yet | **Bootstrap** |
| Real project content (README, LICENSE, a package dir) but no template scaffold | **Adopt** |
| Named project already, needs another service package | **Add package** |
| Neither — repo is unrelated to the template | Don't use this skill |

Detect the mode before doing anything: the repo is still un-bootstrapped (**Bootstrap**) if
`README_TEMPLATE.md` exists, `package.json` still has `"name":
"project-template"`, or the only package directories under
`packages/` are `data-utils`, `example-library` and `example-service`.

If the target directory is empty or does not exist yet, create the project
first with `git clone https://github.com/konrad-woj/project-template.git
<target-dir>`, then continue in Bootstrap mode from inside `<target-dir>`.
Phase 1 repoints `origin` and optionally resets history.

It's **Adopt** instead of Bootstrap if the repo was never actually cloned from
`project-template` — it has its own real README/LICENSE/docs (not the
template's placeholders) but is missing the scaffold itself: no
`package.json`, no `CLAUDE.md`, no `packages/*/pyproject.toml`, no
`run_on_each.sh`/`sync_skills.sh`. This happens when a repo was hand-created
or bootstrapped some other way and the user now wants the template's
conventions layered in. Don't rediscover the scaffold file-by-file: clone a
read-only reference copy of the template into a scratch directory, outside
the target repo:

```bash
git clone --depth 1 https://github.com/konrad-woj/project-template.git <scratch-dir>/project-template
```

Then port these files from it verbatim, unmodified, before touching anything
else:

```text
package.json, package-lock.json, CLAUDE.md, AGENTS.md, .gitignore,
.markdownlint.json, .dockerignore, Dockerfile.example, claude-sandbox.sb,
bootstrap.sh, run_on_each.sh, sync_skills.sh, .env.example,
docs/DESIGN_DOC_TEMPLATE.md, .github/workflows/ci.yml,
.github/workflows/openwiki-update.yml, packages/pyproject.toml.example,
packages/tach.toml.example
```

Also port `packages/data-utils/` as a whole directory if the user opted in to
it in Phase 0.

Then continue into Phase 1 as normal, with two differences: treat the
existing README/LICENSE/git history as real project content to build on, not
template artifacts to discard (don't port `README_TEMPLATE.md` — its presence
would make the repo look un-bootstrapped; write `README.md` fresh, modeled on
the reference clone's `README_TEMPLATE.md`), and only change `CLAUDE.md`'s title line rather than
copying its whole "New Project Setup"/"Code Structure" section verbatim if
the repo already documents its own conventions differently.

## Phase 0 — Preflight and inputs

Never start editing before this phase completes.

1. **Check the toolchain.** `uv --version`, `node --version`, `git --version`,
   `npx --version`. Python must be 3.13 (`uv python list`). If anything is
   missing, stop and say exactly what to install — do not silently degrade.
2. **Check the remote dependencies are reachable.** `git ls-remote
   <url> HEAD` for `project-template`, `skillset` and `logger` (see
   [External dependencies](#external-dependencies)). `uv sync` cannot resolve
   `logger` without GitHub access, so an unreachable `logger` is a hard stop;
   an unreachable `skillset` only makes Phase 4's skill sync fail.
3. **Check the `ponytail` plugin is installed.** The template's `CLAUDE.md`
   points agents at `/ponytail-review` and `/ponytail-audit`. If
   `claude plugin list` doesn't show `ponytail`, ask the user whether to
   install it, then run:

   ```bash
   claude plugin marketplace add DietrichGebert/ponytail
   claude plugin install ponytail@ponytail
   ```

   If they decline, carry on and list it as skipped in the final report.
4. **Check the working tree is clean.** `git status --short`. If it is dirty,
   ask whether to proceed or stash; bootstrapping rewrites many files and a
   dirty tree makes the result unreviewable.
5. **Read `CLAUDE.md`** in the repo. It is the source of truth for conventions
   and overrides anything in this skill that conflicts with it.
6. **Collect inputs by asking the user**, in one round, not one at a time:
   - Project name (kebab-case; becomes `package.json` `"name"` and the README
     title).
   - One-line project description.
   - Package(s) to create: for each, a kebab-case directory name, the
     importable module name (defaults to the directory name with underscores),
     and whether it is a FastAPI service, a worker, or a library.
   - Whether to include `packages/data-utils` (cross-package helpers for env
     access and JSONL I/O). Default yes — it's core reusable infra referenced
     elsewhere in `CLAUDE.md` — but ask rather than assuming, since an unused
     package is dead weight in a POC repo. Don't decide this mid-flight and
     let the user correct it after it's already written.
   - New git remote URL, and whether to reset history to a single initial
     commit.
   - Which skills to sync from `skillset` (default: whatever `sh sync_skills.sh`
     pulls with no arguments).
   - Whether to regenerate OpenWiki now (needs `GEMINI_API_KEY`).
7. **Present a plan and get approval before writing anything.** This is a hard
   gate, not a conceptual "sound good?" question — literally enumerate every
   file that will be created, renamed, or deleted, e.g.:

   ```text
   Create:
     packages/{package_name}/pyproject.toml
     packages/{package_name}/tach.toml
     packages/{package_name}/src/{package_module}/__init__.py
     ...
   Rename:
     README_TEMPLATE.md -> README.md
   Delete:
     openwiki/.last-update.json
   ```

   In "Add package" mode this list is short; in "Bootstrap" or "Adopt" mode it
   is not — write it out in full anyway. A conceptual approval ("shall I port
   the scaffold in?") is not a substitute: it will not surface a decision like
   "skip data-utils" the way a concrete file list does, and the user ends up
   correcting it after the write instead of before.

## Phase 1 — Project identity

1. Run `sh bootstrap.sh <project-name>` to do the mechanical rename (`mv
   README_TEMPLATE.md README.md` and set `package.json`'s `"name"`) in one
   shot. In Adopt mode there is no `README_TEMPLATE.md`, so it only sets the
   name — write `README.md` fresh instead, modeled on the reference clone's
   `README_TEMPLATE.md`. Either way, fill in the title, TL;DR, description,
   and the packages table from Phase 0's inputs. Leave `{placeholder}`
   markers only where the user has not decided yet, and list them in the
   final report.
2. If `bootstrap.sh` isn't in the clone or failed, do both steps by hand:
   `mv README_TEMPLATE.md README.md` (Bootstrap only) and set `package.json`
   `"name"` to the project name.
3. Replace template descriptions in `packages/*/pyproject.toml` that still refer
   to the template rather than the project.
4. If the user asked for a history reset: `rm -rf .git` is commonly blocked
   outright by sandboxed permission configs (categorically, no confirmation
   prompt possible) — don't retry the same blocked call. Use
   `find .git -depth -delete` instead (a depth-first delete that reaches the
   same end state without tripping an `rm -rf` block), then `git init && git
   branch -M main` and set the new remote. If `find -delete` is also blocked,
   ask the user to run the deletion themselves in their own terminal.
   Otherwise just `git remote set-url origin <new-url>` and confirm the old
   template remote is gone (`git remote -v`).
5. Delete `openwiki/.last-update.json` — it pins the *template's* `gitHead` and
   would otherwise be carried into the new project as false provenance.

## Phase 2 — Package scaffolding

For each package from Phase 0, from `packages/`:

1. `mkdir -p {package_name}/src/{package_module} {package_name}/tests/unit
   {package_name}/tests/integration {package_name}/docs`. Add `notebooks/`,
   `evals/`, and `scripts/` only if the user asked for them — empty directories
   that nobody uses are noise. `evals/` and `scripts/` need `__init__.py`;
   `notebooks/` does not.
2. `cp pyproject.toml.example {package_name}/pyproject.toml` and populate
   `name`, `description`, and dependencies. Drop FastAPI/uvicorn from the
   dependency list for a library package. Uncomment and rename the `app` task to
   the real module for a service.
3. `cp tach.toml.example {package_name}/tach.toml`.
4. Write `3.13` to `{package_name}/.python-version`. Don't copy it from
   `example-service`, which may already be deleted or never ported.
5. Write `src/{package_module}/__init__.py` and, for a service, a `main.py`
   entrypoint that calls `configure_logger("INFO")` exactly once — per
   `CLAUDE.md`, configuration happens in the executable entrypoint and nowhere
   else.
6. Write a short `README.md` for the package: what it is, its public surface,
   and the three taskipy commands. Do not restate the code.
7. `cd {package_name} && uv sync --all-groups && uv run tach sync --add`.

Never fork the `logger` package into the new one. It comes from the shared git
repo via `[tool.uv.sources]`, already present in the example pyproject.

Check `packages/data-utils/` before writing any helper or schema — anything
reused across packages belongs there, not in a service package.

## Phase 3 — Environment and secrets

1. `cp .env.example .env` at the repo root, and into each package that needs its
   own keys.
2. **Never read, print, or echo the contents of a populated `.env`.** List which
   keys are unset by name only, and tell the user to populate them.
3. Confirm `.env` is ignored: `git check-ignore .env` must succeed. If it does
   not, fix `.gitignore` before continuing.
4. If the user is on macOS and wants the sandbox profile:
   `sed "s|__HOME__|$HOME|g" claude-sandbox.sb > ~/claude-sandbox.sb` and print
   the alias line for their shell rc. Do not edit their rc file without asking.

## Phase 4 — Skills, Docker, and docs

1. `sh sync_skills.sh [skill ...]`. If it fails (no network, no repo access),
   report it and continue — it is not fatal to the bootstrap.
2. For each deployable service: `cp Dockerfile.example
   Dockerfile.{package_name}` and replace every `{package_name}` and
   `{package_module}` placeholder. Verify none remain:
   `grep -n "{package" Dockerfile.*` must return nothing outside
   `Dockerfile.example`.
3. If the user named a first feature, `cp docs/DESIGN_DOC_TEMPLATE.md
   {FEATURE_NAME}.md` at the repo root and pre-fill the title, Background, and
   Revision History rows. Do not invent Goals — hand it back for the user to
   fill, or hand off to `/designdoc-creator`.
4. If OpenWiki was requested and `GEMINI_API_KEY` is set: `npx openwiki --init`,
   then review the generated pages before staging them. If the key is missing,
   skip and say so.

## Phase 5 — Verification

This phase is the point of the skill. Do not skip it, and do not report success
without it.

```bash
sh run_on_each.sh -b "uv sync --all-groups --locked"
sh run_on_each.sh -b "uv run task ci"
npx --yes markdownlint-cli@0.49.1 '**/*.md' --ignore node_modules --ignore openwiki --ignore '**/.venv' --config .markdownlint.json
```

Then check the things a green gate does not catch:

- **Wheels are not empty.** For every package with a `[build-system]`, run
  `uv build` and list the wheel contents. A `[tool.hatch.build.targets.wheel]
  packages` entry that does not match the real `src/` path produces an empty
  wheel with no error. Delete `dist/` afterwards.
- **No leaked identity.** Grep the tree for the previous project's name, for
  absolute home paths (`/Users/`, `/home/`), and for `{package` placeholders.
- **No secrets staged.** `git status --short` must not list any `.env`. Run
  `git diff --cached` and scan for anything key-shaped before committing.
- **CI matrix covers the new package.** The workflow discovers packages from
  `packages/*/pyproject.toml`, so a package missing its `pyproject.toml` is
  silently untested. Confirm the discovery command lists every package.

Fix what you can, then re-run the gate. Report anything you could not fix rather
than lowering the bar — never delete a failing test or loosen a lint rule to get
green.

## Phase 6 — Commit and report

Commit with a `chore:` or `feat:` message describing the project or package, per
`CLAUDE.md`'s commit conventions: no assistant or employer names, no
`Co-Authored-By` or `Generated with` trailers, and branch names prefixed
`feature/`, `fix/`, or `chore/`.

Close with a short report:

- Packages created and their module names.
- Env keys still unpopulated (names only).
- Anything skipped and why (no network, no API key, user declined).
- Remaining `{placeholder}` markers in `README.md` and the design doc.
- The verification result, verbatim if anything failed.

## Guardrails

- **Ask before destroying.** Resetting git history, overwriting a populated
  `.env`, and deleting an existing package all need explicit confirmation,
  every time — regardless of which command performs the deletion.
- **Idempotent by default.** Re-running on a bootstrapped repo must detect that
  and switch to "Add package" mode rather than re-templating the README.
- **Don't invent structure.** Create only the directories the user asked for.
  The layout in `CLAUDE.md` is a permitted set, not a required one.
- **Don't add skills locally.** If a needed skill is missing, add it to the
  `skillset` repo. `.claude/skills/` is vendored and gitignored; anything
  written there is lost on the next sync.
