---
name: implement
description: Execute an approved implementation plan, following whatever conventions the target repo encodes. Use after a plan has been drafted (by make-plan) and reviewed/approved (by review-plan or by the user). Trigger on phrases like "implement this plan", "start coding this", "build this", "execute the plan", "let's ship it", or when the user points at an approved plan markdown file and asks to begin work. The skill writes code and produces a commit-strategy markdown file proposing how to slice the diff into commits — it does NOT make commits, push, or alter git history. The user controls all git operations. Delegates pattern-discovery and verification to subagents (Task tool) so the main thread's context stays focused on writing code. Works on any kind of repo — application code, libraries, infrastructure, scripts, docs — because the build order and conventions come from the repo itself, not from a pre-baked web-app template.
---

# Implement

Turn an approved plan into code, conforming to the target repo's existing conventions. Produce a commit-strategy file proposing how to slice the diff. **Do not commit, push, or alter git history.** The user controls all git operations.

The skill has six phases: **prepare**, **discover patterns and order** (via subagent), **build** (in the order the subagent returned), **verify** (via subagent), **propose a commit strategy**, then **report**.

The skill does not assume what kind of repo it's working on. The build order, the conventions, and the common-miss checks all come from the plan and the repo, not from this skill.

---

## Phase 1 — Prepare

If the user has not pointed at a plan, ask for one. Accept a path or pasted content.

**If no local repository is accessible, stop immediately** and ask the user to provide repo access before proceeding. Do not begin the build phase against a hypothetical file structure — fabricated paths produce fabricated implementations.

Before writing any code:

1. **Confirm the plan was reviewed.** If the user hasn't run `review-plan` (or hasn't otherwise approved it), ask whether they want to review first. A bad plan executed precisely is still a bad outcome. If the user wants to proceed without review, fine — but flag it in the **Warnings** field of the final report.

2. **Confirm the branch.** Run `git branch --show-current`.
   - If the current branch already looks like a valid feature branch (not `main`, `master`, `trunk`, `staging`, or `develop`), confirm with the user whether to use it before doing anything else — don't silently switch.
   - If the user is on a protected branch and the plan didn't specify a branch, ask the user which branch they want to work on and have them check it out themselves. **Do not create or switch branches automatically.**
   - If the plan specifies a branch and the user is on a different one, surface the mismatch and let the user resolve it.

3. **Identify and note the base branch.** This is the branch the feature branch was cut from — typically `main`, `master`, `trunk`, `staging` or `develop`. Confirm it with the user if ambiguous. You'll need it for `git diff <base-branch>...HEAD` in Phase 4 and for the commit strategy in Phase 5.

4. **Check for pre-existing commits on the branch.** Run `git log <base-branch>...HEAD --oneline`. If commits already exist, note them — the commit strategy in Phase 5 covers only uncommitted working-tree changes, not work already committed.

5. **Confirm a clean working tree.** Run `git status`. If there are uncommitted changes that aren't part of this work, surface them — don't silently mix them into this implementation. Flag any unresolved dirty-tree situation in the final report's **Warnings** field.

6. **Read the plan into the main thread's context.** You'll need it throughout the build phase.

---

## Phase 2 — Discover patterns and dependency order (delegate to a subagent)

Spawn a **pattern-and-order subagent** via the Task tool. Its job is twofold: find existing patterns to mirror, AND tell the main thread what order to build things in.

For plans that touch very distinct areas (e.g., infra + application code), spawn one subagent per area in parallel. In practice, beyond ~4 agents the synthesis overhead tends to outweigh the parallelism benefit — use judgment rather than always hitting that number.

Don't read repo files in the main thread. Same reasoning as the other skills.

### Subagent prompt template

```
You are scouting a codebase for an upcoming implementation. Your job is to find
patterns to mirror AND determine the order things should be built in.
You are NOT writing the implementation.

PLAN:
<paste the plan>

YOUR FOCUS:
<the area, e.g., "everything in this plan", or "the infra/Terraform portion",
or "the backend portion">

YOUR JOB:

A. Identify the kind of repo and stack
   - Application (web backend, web frontend, mobile, desktop, CLI), library,
     infrastructure-as-code, scripts, documentation, monorepo with several of
     these — say which.
   - Languages, frameworks, build tools, test frameworks, package managers
     visible from config files.

B. For each kind of change the plan calls for, find the closest existing example
   in the repo and report the pattern. Use grep/find/glob to locate candidates,
   then read them. Cover only what the plan actually touches. Examples of "kinds
   of change" — adapt to what's actually in the plan:

     - Adding a new module/package/file of type X
     - Adding a new function/endpoint/handler/command
     - Adding a new test
     - Adding a new config value or environment variable
     - Adding a new schema change (if a database/schema exists)
     - Adding a new external integration
     - Adding new infrastructure (Terraform module, K8s manifest, GitHub Action,
       Dockerfile change)
     - Adding new documentation
     - Anything else the plan specifically calls for

   For each, report:
     PATTERN: <short name>
     Example: <path:line>
     Convention: <how it's done in this repo, in 1-3 sentences>
     Gotchas: <anything non-obvious — exceptions, dual conventions in the
               codebase, deprecated paths>

C. Determine the dependency order for THIS plan.
   Look at what the plan calls for and decide the order in which the work should
   be done so that each step's dependencies already exist when it begins.
   Do NOT use a fixed template. Derive the order from the actual work.

   Examples of how dependency order varies by repo type:
     - Application repo with DB: schema migrations → shared types → data layer
       → business logic → API/routes → frontend client → frontend UI → tests
       → config docs
     - Pure library: public API/types → internal modules → tests → docs/examples
     - Terraform module: variables → resources in dependency order (providers →
       IAM → networks → compute → DNS) → outputs → README
     - CLI tool: argument parsing → command handlers → output formatting → tests
     - Documentation change: outline/structure → sections → cross-links → examples
     - Static site: content → templates that reference content → styles →
       build config

   Pick the order that fits THIS plan and THIS repo. Skip steps the plan doesn't
   call for. Add steps the plan does call for that aren't on any template above.

D. Note any non-obvious operational facts the implementer needs:
   - Formatter/linter command, if one exists.
   - How to run the test suite locally.
   - How to run/build/exercise the thing being changed (start command, deploy
     preview, repl, etc.).
   - Where new env vars or config values get documented.
   - Branch and commit conventions visible from `git log --oneline -30` and
     `git branch -a`. (The commit convention will be used to propose a commit
     strategy in Phase 5 — the implementer does not commit.)

REPORT BACK with sections A, B, C, D in that order.
Cite real paths only. Keep under ~700 words.
DO NOT write the implementation. DO NOT propose code. Just report.
```

The main thread keeps this report and refers back to it during the build phase. The order in section C is the order to follow.

---

## Phase 3 — Build (main thread)

Implementation is interleaved work — each step depends on the previous one being in your context — so this phase stays in the main thread.

### Order of operations

**Follow the dependency order returned by the subagent in section C.** Do not impose any other order. If the subagent missed a step the plan calls for, add it in the right place; if it included a step the plan doesn't call for, skip it.

For each step in the order:

1. Look up the relevant pattern from section B of the subagent report.
2. Mirror that pattern. If the plan and the pattern conflict, surface the specific conflict to the user: state what the plan says, what the repo pattern shows, which you'd recommend following and why, and ask the user to decide. Don't silently pick one.
3. After completing the step, do whatever local check makes sense (run the migration in dry-run mode, run the test, type-check, build) before moving to the next step.
4. **Maintain an explicit running list** — after completing each step, record which files were added or modified and which logical step they belong to. Example: `[step: add migration] → db/migrations/0042_add_user_preferences.sql`. This list feeds directly into Phase 5's commit strategy. Do not commit; just track.

### Secrets and config values

When a step introduces a new secret, credential, or config value:
- **Never write the actual secret value into a tracked file.** Use placeholder values in `.env.example` (or the repo's equivalent) and ensure real values go into ignored files.
- Document the variable per the convention surfaced in section D.
- Remind the user to update their local config and any production secret store in the Phase 6 report.

### Universal non-negotiables

These apply regardless of repo type:

- **Do not commit, push, stash, rebase, merge, cherry-pick, reset, or otherwise alter git history.** The user owns all git operations. Read-only git inspection (`git status`, `git diff`, `git log`, `git branch --show-current`) is fine and expected.
- **Never write secrets, credentials, or real `.env` values into tracked files.**
- **Never modify a file that's already been merged in a way that's normally append-only** (committed migrations, changelogs that follow keep-a-changelog, signed/published artifacts). If unsure, check the repo's conventions and ask.
- **Never bypass auth, capability, or permission checks** that exist on similar paths. If unsure whether a new path should be protected, ask.
- **Comment WHY, not WHAT.** Code should be self-explanatory about what it does. Comments are for non-obvious reasons, tradeoffs, or links to context.
- **No defensive error handling for impossible conditions.** Handle errors that can actually occur. Don't add try/catch for paths the type system or business logic already rules out.
- **Match the repo's style.** Naming, file layout, import order, formatting — follow what's already there. If a formatter exists (per section D), run it before considering work done.

### When stuck

- **Pattern unclear** → spawn a follow-up pattern-discovery subagent with a narrower prompt. Don't read a pile of files in the main thread.
- **Plan ambiguous** → stop and ask the user. Don't guess on ambiguity that would change the design.
- **Repo convention conflicts with the plan** → surface the conflict (what the plan says, what the repo pattern shows, your recommendation), then ask the user to decide. The convention usually wins.
- **Subagent's order doesn't quite fit** → stop and ask, or spawn a follow-up subagent to refine the order. Don't silently reorder.

---

## Phase 4 — Verify (delegate to a subagent)

After the build phase, before producing the commit strategy, capture the full diff and spawn a **verification subagent** with fresh eyes on it. The main thread has been deep in the implementation; a fresh subagent will catch things you stopped seeing.

**Before spawning the verifier**, run the following in the main thread and include both outputs in the verifier prompt:
```
git diff <base-branch>...HEAD
git status
```
Both are read-only. `git status` captures untracked files the diff won't show. Use the base branch confirmed in Phase 1 step 3.

### Verification prompt template

```
You are reviewing a freshly-written implementation against the plan it was
supposed to follow. Your job is to verify, NOT to refactor or rewrite.

PLAN:
<paste the plan>

PATTERNS REPORT (from the earlier pattern-discovery subagent):
<paste section B from the earlier report>

DIFF:
<paste output of `git diff <base-branch>...HEAD`>

UNTRACKED FILES:
<paste output of `git status` — new files not yet in git diff>

YOUR JOB:

1. For each item the plan said would change: confirm it actually changed in
   the diff.
2. For each item the diff changed: confirm the plan said it would (or that the
   change is a reasonable side-effect).
3. For each pattern in the patterns report: spot-check that the diff follows it
   where applicable.
4. Derive a common-miss checklist FROM what this diff actually does.
   Do NOT use a generic checklist. For each kind of change present in the diff,
   ask: what's the most common way for this kind of change to be incomplete?
   Then check the diff for that. Examples:
     - If the diff adds a new module/file → is it imported/registered/exported
       anywhere it needs to be?
     - If the diff adds a new config value → is it documented? Is it actually
       read by the code that needs it?
     - If the diff adds a new external dependency → is it in the manifest
       (package.json, Cargo.toml, etc.) and lockfile?
     - If the diff adds a new schema/migration → is the migration reflected in
       the code that uses the new schema, and vice versa?
     - If the diff adds a new endpoint/handler/command → is it wired into the
       entry point that exposes it?
     - If the diff adds a new test → does it actually run with the rest of the
       suite?
     - If the diff adds new infrastructure → are outputs/variables wired to
       whatever consumes them?
   Skip kinds of change the diff doesn't contain.

5. General hygiene checks (apply to any diff):
   - Debug output left in (console.log, print, dbg!, fmt.Println for debug, etc.)
   - TODO/FIXME/XXX markers added in this diff.
   - Commented-out code added in this diff.
   - Strings that look like secrets/keys/tokens/passwords.
   - Hardcoded values that look like they should be configurable.

REPORT BACK with:
- **Plan items completed**: list with diff line references.
- **Plan items missing**: anything the plan called for that's not in the diff.
- **Diff items not in plan**: anything in the diff the plan didn't mention
  (could be fine, could be scope creep).
- **Pattern conformance**: for each pattern checked, whether the diff follows it.
- **Derived common-miss findings**: results of step 4, item by item, with
  `path:line`.
- **General hygiene findings**: results of step 5, only items that fired.

DO NOT propose refactors. DO NOT critique style unless it violates a convention
surfaced earlier. Just verify.

Cite real paths only. Keep under ~600 words.
```

### After the verifier returns

- **Trivial fixes** (a missing import, a single-line config addition, a typo) — fix in the main thread and note the fix in the Phase 6 report. No need to re-spawn.
- **Non-trivial fixes** (touches more than one file or adds new logic) — fix in the main thread, then re-spawn the verifier with an updated diff so the fix itself gets checked.

Do not proceed to Phase 5 until any blocking issues from Phase 4 are resolved.

---

## Phase 5 — Propose a commit strategy (save as markdown)

The main thread does this, using the running file list maintained during Phase 3. **Do not run any git commands that alter state** — only produce a markdown file that the user will execute manually.

### What to produce

A markdown file named using the plan's filename as a base. Save locations:
- If the plan was loaded from a file: save alongside it, appending `.commit-strategy.md` — e.g., `.claude/plans/PROJ-123-add-csv-export.md` → `.claude/plans/PROJ-123-add-csv-export.commit-strategy.md`.
- If the plan was pasted (no file path): save to `.claude/plans/<short-slug>.commit-strategy.md`. If `.claude/plans/` doesn't exist, ask the user for their preferred location once.

### File structure

```markdown
# Commit strategy: <one-line description of the work>

> Plan: <path to the plan file, or "pasted">
> Branch: <current branch name>
> Base: <base branch confirmed in Phase 1>
> Uncommitted files: <count of files in the working tree changes>

> **Note:** If you have files already staged (`git status` shows entries in
> green), stash or commit those separately before executing this strategy
> to avoid mixing unrelated changes into these commits.

## Pre-existing commits on this branch

<List any commits found in Phase 1 step 4 — these are already in history and
are NOT part of this strategy. Omit section if the branch had no prior commits.>

## Suggested commit slicing

Each section below is one proposed commit. Apply them in order — later commits
depend on earlier ones.

### Commit 1 — <short imperative subject following the repo's commit convention>

**Files:**
- `path/one.ext`
- `path/two.ext`

**Rationale:** <one or two sentences on why these files belong together>

**Suggested git commands:**
```
git add path/one.ext path/two.ext
git commit -m "<subject following the repo's convention>"
```

### Commit 2 — ...

...

## Notes

<Ordering and dependency caveats only — e.g., "run the migration after commit 1
before applying commit 2 if you want to verify the data layer compiles against
the new schema". Omit if there are no ordering dependencies to flag.>

## Files not assigned to any commit

<Files in the diff that don't fit cleanly into the slicing above — e.g.,
formatter-driven changes to unrelated files, generated lockfile updates, or
untracked build artifacts the user may or may not want to commit. For each,
note why it wasn't assigned and let the user decide. Omit section if empty.>
```

### Rules for the commit strategy

- **Follow the repo's commit convention** from section D of the Phase 2 report (conventional commits, ticket-prefixed, free-form, etc.). Don't impose a convention the repo doesn't use.
- **Aim for each suggested commit to be independently buildable** where feasible — i.e., the repo should compile and tests should pass after each one. Where a temporarily-broken intermediate state is unavoidable (e.g., a schema change before the code that uses it), call that out explicitly in the Notes section so the user knows not to push mid-strategy.
- **Group by logical unit, not by file type.** A schema migration and the code that uses it belong in adjacent commits, in dependency order. Don't bundle them by extension.
- **Aim for ~8 commits or fewer** for a single plan — beyond that the strategy document itself becomes hard to follow. Use judgment; a complex multi-area change may genuinely warrant more. For a small diff (<5 files, single logical change), propose a single commit.
- **Cite real paths only.** No fabricated paths.

### After saving

Tell the user:
- The path of the commit-strategy file.
- A one-line summary: "Proposed N commits across M files. Review the strategy and execute the git commands yourself when ready."
- Remind them that the skill made no commits and altered no git state.

---

## Phase 6 — Report

Produce a concise end-of-task summary:

- **Warnings** — flags raised during Phase 1: plan was not reviewed before implementation, dirty working tree was present at start, branch mismatch was surfaced, pre-existing commits were found on the branch, or any other pre-flight issue. Omit if none.
- **Files changed** — list of paths, grouped (added / modified / deleted / untracked). Note: these are working-tree changes; **none have been committed**.
- **Schema / migrations / infra changes** — whatever the plan called for that touches persistent or shared state, with one-line descriptions. Use whatever terms apply ("migrations", "Terraform resources", "schema", "manifests", "config").
- **New config / env vars / secrets** — names, where they're documented, and a reminder to update local config and any production secret store before running or deploying.
- **Manually verified** — what you actually ran, exercised, or tested.
- **Not verified** — be explicit. If you didn't run something on real data, say so. If a test in the plan was deferred, say so.
- **Verifier findings addressed** — list of issues the verification subagent surfaced and how each was resolved.
- **Commit strategy** — path to the `commit-strategy.md` file and a one-line summary (e.g., "5 proposed commits across 12 files"). Remind the user that the skill made no commits.
- **Next steps for the user** — typically: review the commit strategy, check for staged files before executing it, run the suggested git commands (or your own slicing), run the full test suite, push the branch, open a PR, update production config, deploy.

---

## Rules

- **The skill writes code; the user controls git.** No commits, no pushes, no history rewrites, no stash, no rebase. Read-only git inspection is fine.
- **Stop and ask** instead of guessing on anything that would change the design.
- **Cite real paths.** No fabricated paths in any report or in the commit strategy.
- **Respect the plan.** Don't quietly expand scope. If something outside the plan needs to change, surface it and ask.
- **No planning in this skill.** This skill executes; it does not redesign.