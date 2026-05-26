---
name: onboard-repo
description: One-time repo orientation that produces a persistent `.claude/context.md` file capturing the repo's stack, conventions, directory layout, build/test/run commands, and workflow patterns. Run this once at the start of working on a new repo. All subsequent skills (make-plan, review-plan, implement, review-implementation) read this file at session start instead of re-discovering the same facts from scratch via their own subagents. Trigger on phrases like "onboard this repo", "learn this codebase", "set up context for this repo", "orient yourself on this project", or when the user starts a new repo and wants the pipeline skills to work without cold-start overhead. Re-run when the stack, structure, or conventions change significantly (major framework upgrade, repo restructure, new tooling).
---

# Onboard Repo

Investigate a repository once and produce a persistent `.claude/context.md` file that all pipeline skills can read at the start of their sessions — eliminating the cold-start re-discovery that would otherwise happen in every `make-plan`, `review-plan`, `implement`, and `review-implementation` session.

The skill has four phases: **confirm access**, **investigate** (three parallel subagents), **synthesize** into the context file, then **save and verify**.

---

## Phase 1 — Confirm access

If no local repository is accessible, stop immediately and ask the user to provide repo access before proceeding. Do not fabricate a context file for a hypothetical repo — it will silently mislead every skill that reads it.

Check whether `.claude/context.md` already exists:
- If it exists, read it and show the user the `Last updated` and `Re-run triggers` fields from the file header.
- Ask: *"A context file already exists from [date]. Do you want to refresh it (re-run the full investigation) or keep the existing one?"*
- If the user wants to keep it, stop here.
- If the user wants to refresh, proceed — the existing file will be overwritten at the end.

---

## Phase 2 — Investigate (delegate to three parallel subagents)

> **Spawn all three subagents in a single message so they run concurrently.** This is the most important performance step — do not spawn them sequentially.

Each subagent owns a distinct, non-overlapping area. Don't read repo files in the main thread.

### Subagent 1 — Stack & structure investigator

```
You are investigating a codebase to produce an orientation summary for a
persistent context file. Your job is to report facts, not make recommendations.

YOUR AREA: Stack, repo type, and directory structure.

YOUR JOB:

1. Identify the repo type:
   Application (web backend, web frontend, mobile, desktop, CLI), library,
   infrastructure-as-code, scripts, documentation, monorepo containing several
   of these. If monorepo, list each sub-package and its type.

2. Identify the stack from config files (package.json, pyproject.toml,
   Cargo.toml, go.mod, build.gradle, pom.xml, *.csproj, mix.exs, or equivalent):
   - Primary language(s) and version constraints
   - Runtime or VM (Node, JVM, BEAM, .NET, etc.)
   - Primary framework(s) (Rails, Django, Express, Next.js, Axum, etc.)
   - Key libraries that define the architecture (ORM, HTTP client, DI container,
     state management, etc.)

3. Map the directory structure — two levels deep, skipping build artifacts,
   node_modules, .git, and generated files. For each top-level directory,
   write a one-line note on what it contains.

4. Identify key entry points:
   - Application: the main file, server startup, CLI entry, worker entry
   - Library: the public API surface (index file, lib/ exports, crate root)
   - Infrastructure: the root module, workspace, or environment entry
   For each, give `path:line` references.

5. Identify the data layer if one exists:
   - Database type (relational, document, key-value, graph — and the specific engine)
   - ORM or query builder in use
   - Where schema definitions live
   - Where migrations live and what tool manages them

6. Identify any external service integrations visible in the codebase
   (payment processors, email services, cloud APIs, message queues, etc.)
   — just names and where they're configured.

REPORT BACK with titled sections for items 1-6.
Cite real paths only. Keep under ~600 words.
DO NOT recommend anything. Just report what's there.
```

### Subagent 2 — Conventions & patterns investigator

```
You are investigating a codebase to produce an orientation summary for a
persistent context file. Your job is to report facts, not make recommendations.

YOUR AREA: Coding conventions, patterns, and workflow.

YOUR JOB:

1. Naming conventions — examine 10-15 representative files and note:
   - File naming: kebab-case, snake_case, PascalCase, camelCase?
   - Variable/function naming style per language
   - Type/class/interface naming
   - DB column naming (if a schema exists)
   - Config key naming
   Note any inconsistencies (e.g. "mostly snake_case but older modules use camelCase").

2. Module/package structure — how is the code organised?
   - By layer (controllers/, services/, models/)
   - By feature/domain (users/, payments/, reports/)
   - Mixed? If mixed, describe the pattern.
   - Where do shared utilities, helpers, and types live?

3. Error handling pattern — look at 3-5 representative functions or handlers:
   - How are errors propagated? (exceptions, Result types, error callbacks,
     error-first callbacks, etc.)
   - Is there a central error handler? Where?
   - Any custom error classes or error codes? Where are they defined?

4. Testing patterns:
   - Where do tests live? (co-located, __tests__/, tests/, spec/)
   - File naming convention for tests
   - What's a representative unit test look like? (cite one real path)
   - What's a representative integration test look like? (cite one real path)
   - Any test fixtures, factories, or helpers? Where?

5. Auth/permissions patterns (if the repo has them):
   - How is authentication implemented?
   - How are permissions or capabilities checked on routes/handlers?
   - Where is the auth middleware or guard? (cite path)

6. Branch and commit conventions — run `git log --oneline -30` and
   `git branch -a`:
   - Branch naming pattern (feature/*, fix/*, PROJ-NNN/*, etc.)
   - Commit message format (conventional commits, free-form, ticket prefix, etc.)
   - Squash vs merge commit pattern if visible

7. PR and code review conventions — check CONTRIBUTING.md,
   .github/pull_request_template.md, and .github/ISSUE_TEMPLATE/:
   - PR description template if one exists
   - Required reviewers or CODEOWNERS if present
   - Any automated checks mentioned (CI required, coverage threshold, etc.)

REPORT BACK with titled sections for items 1-7.
Cite real paths only. Keep under ~600 words.
DO NOT recommend anything. Just report what's there.
```

### Subagent 3 — Operations & tooling investigator

```
You are investigating a codebase to produce an orientation summary for a
persistent context file. Your job is to report facts, not make recommendations.

YOUR AREA: Build, test, run, and deploy tooling — everything an engineer
needs to operate the repo day-to-day.

YOUR JOB:

1. Read orientation files in full: README.md, CLAUDE.md, AGENTS.md,
   CONTRIBUTING.md, DEVELOPMENT.md, docs/setup.md (or equivalent).
   Extract and list every command mentioned for: install, build, run/start,
   test, lint, format, type-check, migrate, seed, clean.

2. Formatter and linter:
   - Which tools are configured? (prettier, eslint, black, ruff, rustfmt,
     gofmt, terraform fmt, etc.)
   - Config file paths
   - Exact command to run them (check vs fix/write mode)

3. Test runner:
   - Exact command to run all tests
   - Exact command to run a single test file
   - Exact command to run tests matching a pattern
   - How to run tests with coverage

4. Build and start:
   - Exact command to build for production
   - Exact command to start in development (with hot reload if applicable)
   - Exact command to start in production
   - Any required environment setup before first run (e.g. `cp .env.example .env`)

5. Environment and configuration:
   - Is there a `.env.example` or equivalent? List every variable defined in it
     with a one-line description of what it controls.
   - Where are non-secret config values defined? (config files, constants, etc.)
   - Where should new env vars be documented when added?
   - Is there a secrets manager in use? (Vault, AWS Secrets Manager, Doppler, etc.)

6. CI/CD:
   - What CI platform is used? (GitHub Actions, CircleCI, GitLab CI, etc.)
   - Where are the pipeline definitions? (cite paths)
   - What does the CI pipeline do? (test, lint, build, deploy, etc.)
   - Is there a staging environment? How is it deployed?

7. Database / migration operations (if applicable):
   - Exact command to run migrations
   - Exact command to roll back a migration
   - Exact command to create a new migration file
   - Exact command to seed the database
   - Exact command to reset the database (drop + migrate + seed)

8. Any re-run triggers visible in the repo that would make this context
   file go stale:
   - Pending major version upgrades noted in TODOs or issues
   - Active framework migration in progress (e.g. partially migrated files)
   - Known upcoming structural changes documented in ADRs or READMEs

REPORT BACK with titled sections for items 1-8.
Cite real paths only. Keep under ~700 words.
DO NOT recommend anything. Just report what's there.
```

### After all three subagents return

If any subagent reports it couldn't find expected files or surfaced a significant gap (e.g., no README, no test examples found, CI config missing), spawn a follow-up subagent with a narrower prompt targeting that gap rather than reading files in the main thread.

If two subagents return contradictory findings on the same fact (e.g., different claims about the test runner), note the conflict in the context file explicitly under the relevant section with both findings cited, so users can resolve it manually.

---

## Phase 3 — Synthesize the context file

The main thread does this. Combine all three subagent reports into a single well-structured markdown file. Do not simply concatenate the reports — synthesize them: merge overlapping information, resolve redundancy, and organise into the standard sections below.

The context file must follow this exact structure so all pipeline skills can reliably read it:

```markdown
# Repo context

> Last updated: <ISO date>
> Re-run triggers: <one-line summary of conditions that would make this stale,
>   from subagent 3's item 8 — e.g. "major Next.js upgrade in progress,
>   Prisma → Drizzle migration planned">

---

## Repo type and stack

<repo type, primary language(s), runtime, framework(s), key libraries>

## Directory map

<two-level directory map with one-line notes — use a code block for alignment>

## Entry points

<key entry points with path:line references>

## Data layer

<database, ORM, schema location, migrations location and tool — or "none" if
not applicable>

## External integrations

<list of external services and where they're configured — or "none">

---

## Naming conventions

<file, variable, type, DB column, config key conventions — note any
inconsistencies>

## Module structure

<how code is organised — by layer, by feature, or mixed>

## Error handling

<how errors are propagated and handled — cite one representative path>

## Testing patterns

<where tests live, naming convention, one representative unit test path,
one representative integration test path, fixture/factory location>

## Auth and permissions

<how auth is implemented, where middleware/guards live — or "not present">

## Branch and commit conventions

<branch naming pattern, commit message format>

## PR conventions

<PR template summary, CODEOWNERS if present, required CI checks>

---

## Commands

### Install
```
<exact command>
```

### Build
```
<exact command>
```

### Start (dev)
```
<exact command>
```

### Start (prod)
```
<exact command>
```

### Test (all)
```
<exact command>
```

### Test (single file)
```
<exact command>
```

### Test (pattern)
```
<exact command>
```

### Test (coverage)
```
<exact command>
```

### Lint
```
<exact command>
```

### Format
```
<exact command>
```

### Type-check
```
<exact command — or "n/a">
```

### Migrate
```
<exact command — or "n/a">
```

### Rollback migration
```
<exact command — or "n/a">
```

### New migration
```
<exact command — or "n/a">
```

### Seed
```
<exact command — or "n/a">
```

### Reset database
```
<exact command — or "n/a">
```

---

## Environment variables

<table or list of every variable from .env.example with one-line descriptions>

### Where to document new variables
<path and format to follow when adding new env vars>

### Secrets management
<tool in use, or "none — use .env locally">

---

## CI/CD

<platform, pipeline file paths, what the pipeline does, staging deploy process>

---

## Conflicts and unresolved gaps

<any contradictions between subagent reports, or areas the subagents couldn't
investigate fully — so users know what to manually verify. Omit section if empty.>
```

**Rules for synthesis:**
- Every command must be the exact runnable string — no placeholders like `<your-command>` unless the command genuinely varies.
- If a command was not found, write `not found in repo` rather than guessing.
- If a section has no content (e.g., no data layer), write `not applicable` rather than omitting the section — omitting sections breaks the predictable structure other skills rely on.
- Cite at least one real `path:line` reference per major section so the context is grounded and verifiable.

---

## Phase 4 — Save and verify

1. Create the `.claude/` directory if it doesn't exist.
2. Write the synthesized content to `.claude/context.md`.
3. Run a quick self-check: re-read the file and confirm every Commands section has either an exact command or `not found in repo` — no blanks.
4. Report to the user:
   - The file path saved
   - A one-line summary of what was found (repo type, stack, number of commands documented)
   - The `Re-run triggers` line so they know when to refresh it
   - Any items under **Conflicts and unresolved gaps** that need manual attention

---

## How other skills use this file

Each pipeline skill should check for `.claude/context.md` at the start of its session and, if present, read it into the main thread's context before doing anything else. This replaces the stack/convention discovery that each skill's subagents would otherwise do from scratch.

Concretely, each skill's Phase 1 should include:

> *"Check for `.claude/context.md`. If it exists, read it and use it as the source of truth for stack, conventions, commands, and directory structure throughout this session. Note the `Re-run triggers` field — if it indicates the context may be stale for this work, flag it to the user before proceeding."*

Skills should still spawn their own subagents for work that `onboard-repo` doesn't cover — specifically, finding the precise files relevant to a particular ticket or diff. The context file eliminates cold-start re-discovery of stable facts; it does not replace targeted investigation.

---

## When to re-run

Re-run `onboard-repo` when:
- The primary framework has been upgraded to a new major version
- The repo has been significantly restructured (new monorepo layout, directory reorganisation)
- New tooling has replaced existing tooling (new test runner, new migration tool, new CI platform)
- The `Re-run triggers` field in the existing context file describes a change that has since landed
- More than ~3 months have passed and the repo is actively maintained

---

## Rules

- **Never fabricate commands.** If a command wasn't found in the repo, write `not found in repo`. A wrong command silently breaks every session that reads it.
- **Cite real paths only.** No invented paths in any section.
- **Keep the structure stable.** The section headers in Phase 3 are the contract between this skill and the skills that read the file. Don't rename or reorder them.
- **Conflicts go in the file, not just the report.** Any contradiction between subagent findings must appear in the `Conflicts and unresolved gaps` section of the context file itself — not just in the Phase 4 summary — so future sessions see it.