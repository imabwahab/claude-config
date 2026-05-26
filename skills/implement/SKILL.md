---
name: implement
description: Execute an approved implementation plan, following whatever conventions the target repo encodes. Use after a plan has been drafted (by make-plan) and reviewed/approved (by review-plan or by the user). Trigger on phrases like "implement this plan", "start coding this", "build this", "execute the plan", "let's ship it", or when the user points at an approved plan markdown file and asks to begin work. The skill delegates pattern-discovery and dependency-order derivation to subagents (Task tool) so the main thread's context stays focused on writing code, then runs a fresh-eyes verification subagent at the end. Works on any kind of repo — application code, libraries, infrastructure, scripts, docs — because the build order and conventions come from the repo itself, not from a pre-baked web-app template.
---

# Implement

Turn an approved plan into code, conforming to the target repo's existing conventions.

The skill has five phases: **prepare**, **discover patterns and order** (via subagent), **build** (in the order the subagent returned), **verify** (via subagent), then **report**.

The skill does not assume what kind of repo it's working on. The build order, the conventions, and the common-miss checks all come from the plan and the repo, not from this skill.

## Phase 1 — Prepare

If the user has not pointed at a plan, ask for one. Accept a path or pasted content.

Before writing any code:

1. **Confirm the plan was reviewed.** If the user hasn't run `review-plan` (or hasn't otherwise approved it), ask whether they want to review first. A bad plan executed precisely is still a bad outcome. If the user wants to proceed without review, fine — but note it in the final report.
2. **Confirm the branch.** Check `git branch --show-current`. If it doesn't match the plan's proposed branch (or any other approved branch name), create and check out the right one. If the plan didn't specify a branch convention, ask the user once and proceed.
3. **Confirm a clean working tree.** Run `git status`. If there are uncommitted changes that aren't part of this work, surface them — don't silently mix them into this implementation.
4. **Read the plan into the main thread's context.** You'll need it throughout the build phase.

## Phase 2 — Discover patterns and dependency order (delegate to a subagent)

Spawn a **pattern-and-order subagent** via the Task tool. Its job is twofold: find existing patterns to mirror, AND tell the main thread what order to build things in.

For plans that touch very distinct areas (e.g., infra + application code), spawn one subagent per area in parallel — cap ~4.

Don't read repo files in the main thread. Same reasoning as the other skills.

### Subagent prompt template

```
You are scouting a codebase for an upcoming implementation. Your job is to find patterns to mirror AND determine the order things should be built in. You are NOT writing the implementation.

PLAN:
<paste the plan>

YOUR FOCUS:
<the area, e.g., "everything in this plan", or "the infra/Terraform portion", or "the backend portion">

YOUR JOB:

A. Identify the kind of repo and stack
   - Application (web backend, web frontend, mobile, desktop, CLI), library, infrastructure-as-code, scripts, documentation, monorepo with several of these — say which.
   - Languages, frameworks, build tools, test frameworks, package managers visible from config files.

B. For each kind of change the plan calls for, find the closest existing example in the repo and report the pattern.
   Use grep/find/glob to locate candidates, then read them. Cover only what the plan actually touches. Examples of "kinds of change" — adapt to what's actually in the plan:

     - Adding a new module/package/file of type X
     - Adding a new function/endpoint/handler/command
     - Adding a new test
     - Adding a new config value or environment variable
     - Adding a new schema change (if a database/schema exists)
     - Adding a new external integration
     - Adding new infrastructure (Terraform module, K8s manifest, GitHub Action, Dockerfile change)
     - Adding new documentation
     - Anything else the plan specifically calls for

   For each, report:
     PATTERN: <short name>
     Example: <path:line>
     Convention: <how it's done in this repo, in 1-3 sentences>
     Gotchas: <anything non-obvious — exceptions, dual conventions in the codebase, deprecated paths>

C. Determine the dependency order for THIS plan.
   Look at what the plan calls for and decide the order in which the work should be done so that each step's dependencies already exist when it begins. Do NOT use a fixed template. Derive the order from the actual work.

   Examples of how dependency order varies by repo type:
     - Application repo with DB: schema migrations → shared types → data layer → business logic → API/routes → frontend client → frontend UI → tests → config docs
     - Pure library: public API/types → internal modules → tests → docs/examples
     - Terraform module: variables → resources in dependency order (providers → IAM → networks → compute → DNS) → outputs → README
     - CLI tool: argument parsing → command handlers → output formatting → tests
     - Documentation change: outline/structure → sections → cross-links → examples
     - Static site: content → templates that reference content → styles → build config

   Pick the order that fits THIS plan and THIS repo. Skip steps the plan doesn't call for. Add steps the plan does call for that aren't on any of the templates above.

D. Note any non-obvious operational facts the implementer needs:
   - Formatter/linter command, if one exists.
   - How to run the test suite locally.
   - How to run/build/exercise the thing being changed (start command, deploy preview, repl, etc.).
   - Where new env vars or config values get documented.
   - Branch and commit conventions visible from `git log --oneline -30` and `git branch -a`.

REPORT BACK with sections A, B, C, D in that order. Cite real paths only. Keep under ~700 words.

DO NOT write the implementation. DO NOT propose code. Just report.
```

The main thread keeps this report and refers back to it during the build phase. The order in section C is the order to follow.

## Phase 3 — Build (main thread)

Implementation is interleaved work — each step depends on the previous one being in your context — so this phase stays in the main thread.

### Order of operations

**Follow the dependency order returned by the subagent in section C.** Do not impose any other order. If the subagent missed a step the plan calls for, add it in the right place; if it included a step the plan doesn't call for, skip it.

For each step in the order:

1. Look up the relevant pattern from section B of the subagent report.
2. Mirror that pattern. If the plan and the pattern conflict, stop and ask — the plan may have been written without seeing the pattern.
3. After completing the step, do whatever local check makes sense (run the migration, run the test, type-check, build) before moving to the next step.

### Universal non-negotiables

These apply regardless of repo type:

- **Never commit secrets, credentials, or `.env` files.** New secrets/config values get documented per the convention from section D, and the user is reminded to update local config and any production secret store.
- **Never push directly to the default branch** (`main`, `master`, `trunk`, `develop`, whatever the repo uses). PR only.
- **Never modify a file that's already been merged in a way that's normally append-only** (committed migrations, changelogs that follow keep-a-changelog, signed/published artifacts). If unsure, check the repo's conventions and ask.
- **Never bypass auth, capability, or permission checks** that exist on similar paths. If unsure whether a new path should be protected, ask.
- **Comment WHY, not WHAT.** Code should be self-explanatory about what it does. Comments are for non-obvious reasons, tradeoffs, or links to context.
- **No defensive error handling for impossible conditions.** Handle errors that can actually occur. Don't add try/catch for paths the type system or business logic already rules out.
- **Match the repo's style.** Naming, file layout, import order, formatting — follow what's already there. If a formatter exists (per section D), run it before considering work done.

### When stuck

- **Pattern unclear** → spawn a follow-up pattern-discovery subagent with a narrower prompt. Don't read a pile of files in the main thread.
- **Plan ambiguous** → stop and ask the user. Don't guess on ambiguity that would change the design.
- **Repo convention conflicts with the plan** → stop and surface the conflict. The plan was probably written without seeing the relevant pattern, and the convention usually wins.
- **Subagent's order doesn't quite fit** → stop and ask, or spawn a follow-up subagent to refine the order. Don't silently reorder.

## Phase 4 — Verify (delegate to a subagent)

After the build phase, before declaring done, spawn a **verification subagent** with fresh eyes on the diff. The main thread has been deep in the implementation; a fresh subagent will catch things you stopped seeing.

### Verification prompt template

```
You are reviewing a freshly-written implementation against the plan it was supposed to follow.
Your job is to verify, NOT to refactor or rewrite.

PLAN:
<paste the plan>

PATTERNS REPORT (from the earlier pattern-discovery subagent):
<paste section B from the earlier report>

DIFF:
<paste output of `git diff <base-branch>...HEAD` plus untracked files>

YOUR JOB:

1. For each item the plan said would change: confirm it actually changed in the diff.
2. For each item the diff changed: confirm the plan said it would (or that the change is a reasonable side-effect).
3. For each pattern in the patterns report: spot-check that the diff follows it where applicable.
4. Derive a common-miss checklist FROM what this diff actually does. Do NOT use a generic checklist.
   For each kind of change present in the diff, ask: what's the most common way for this kind of change to be incomplete? Then check the diff for that. Examples:
     - If the diff adds a new module/file → is it imported/registered/exported anywhere it needs to be?
     - If the diff adds a new config value → is it documented? Is it actually read by the code that needs it?
     - If the diff adds a new external dependency → is it in the manifest (package.json, Cargo.toml, etc.) and lockfile?
     - If the diff adds a new schema/migration → is the migration reflected in the code that uses the new schema, and vice versa?
     - If the diff adds a new endpoint/handler/command → is it wired into the entry point that exposes it?
     - If the diff adds a new test → does it actually run with the rest of the suite?
     - If the diff adds new infrastructure → are outputs/variables wired to whatever consumes them?
   Skip kinds of change the diff doesn't contain.

5. General hygiene checks (apply to any diff):
   - Debug output left in (console.log, print, dbg!, fmt.Println for debug, eprintln! for debug, etc.).
   - TODO/FIXME/XXX markers added in this diff.
   - Commented-out code added in this diff.
   - Strings that look like secrets/keys/tokens/passwords.
   - Hardcoded values that look like they should be configurable.

REPORT BACK with:
- **Plan items completed**: list with diff line references.
- **Plan items missing**: anything the plan called for that's not in the diff.
- **Diff items not in plan**: anything in the diff the plan didn't mention (could be fine, could be scope creep).
- **Pattern conformance**: for each pattern checked, whether the diff follows it.
- **Derived common-miss findings**: results of step 4, item by item, with `path:line`.
- **General hygiene findings**: results of step 5, only items that fired.

DO NOT propose refactors. DO NOT critique style unless it violates a convention surfaced earlier. Just verify.

Cite real paths only. Keep under ~600 words.
```

If the verifier flags missing items, fix them in the main thread, then either re-spawn the verifier or note the fix in the final report.

## Phase 5 — Report

Produce a concise end-of-task summary:

- **Files changed** — list of paths, grouped (added / modified / deleted).
- **Schema/migrations/infra changes** — whatever the plan called for that touches persistent or shared state, with one-line descriptions. Use whatever terms apply ("migrations", "Terraform resources", "schema", "manifests", "config").
- **New config / env vars / secrets** — names, where they're documented, and a reminder to update local and production stores.
- **Manually verified** — what you actually ran, exercised, or tested.
- **Not verified** — be explicit. If you didn't run something on real data, say so. If a test in the plan was deferred, say so.
- **Verifier findings addressed** — list of issues the verification subagent surfaced and how each was resolved.
- **Next steps for the user** — usually some combination of: run formatter, run full test suite, push branch, open PR, update production config, deploy.

Do not declare the implementation done until Phase 4 has run and any blocking issues from it are resolved.

## Rules

- **Stop and ask** instead of guessing on anything that would change the design.
- **Cite real paths.** No fabricated paths in any report.
- **Respect the plan.** Don't quietly expand scope. If something outside the plan needs to change, surface it and ask.
- **No planning in this skill.** This skill executes; it does not redesign.