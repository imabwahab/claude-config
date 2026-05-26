---
name: review-implementation
description: Senior-reviewer code review of a completed implementation before a PR is opened or merged. Use after `implement` finishes, or whenever the user has a working branch and wants a critical pass over the diff. Trigger on phrases like "review my code", "review this branch", "review before I PR", "code review please", "look over the diff", "anything I missed", or when the user points at a branch/diff and asks for feedback. The skill delegates convention-discovery and diff-investigation to subagents (Task tool) so the main thread's review context stays clean. The subagents derive review focus and a common-miss checklist FROM the diff and the repo, not from a pre-baked template — so the skill works on any kind of repo (application, library, infrastructure, scripts, docs).
---

# Review Implementation

Review the diff on the current branch as a senior reviewer would.

The skill has four phases: **establish scope**, **gather conventions and diff findings** (via subagents in parallel), **synthesize** the review, then **deliver**. Fixes are surfaced for the user to apply, not auto-applied.

The skill makes no assumption about what kind of repo it's reviewing. Review focus areas, the common-miss checklist, and convention enforcement all derive from the diff and the repo itself, not from this skill.

---

## Phase 1 — Establish scope

### 1.1 No repo available?

If no local repository is accessible, stop immediately and ask the user to provide repo access or paste the diff directly. If the user pastes a diff without a repo:
- Skip the convention auditor subagent entirely.
- Mark all convention checks in the review as **unverifiable — no repo provided**.
- Proceed with diff investigation only, using the pasted diff as input.

### 1.2 Check for upstream warnings

Ask the user: "Did the implement session surface any warnings in its final report (plan not reviewed, dirty working tree, branch mismatch, etc.)? If so, please paste them here and I'll carry them into the review."
If the user provides warnings, carry them forward into the Warnings passthrough field in Phase 3. If not, omit the field entirely.

### 1.3 Get the diff

By default, review the diff between the current branch and the repo's default branch:

```
git diff <default-branch>...HEAD
```

Detect the default branch via `git symbolic-ref refs/remotes/origin/HEAD` or `git remote show origin`. If detection fails, ask the user. **Do not proceed with subagent spawning until the base branch is confirmed** — the base branch is required to produce a meaningful diff.

If the user wants to review something else (a specific commit range, the staged changes, an unmerged PR they've fetched), accept that instead.

Also collect:
- The list of changed files (`git diff --stat <base>...HEAD`).
- The current branch name (`git branch --show-current`).
- The plan markdown file if one exists for this work — check `.claude/plans/` first (the convention from `make-plan`), then fall back to `plans/` or `docs/plans/`. If none is found, ask the user once; if they don't have one, proceed without it.

### 1.4 Skim the diff to characterize it

Before delegating, look at `git diff --stat` and the file paths to characterize the diff in one or two sentences for the subagents. Examples:
- "Adds a Terraform module for a new VPC plus updates to the network outputs."
- "Adds a new endpoint and its tests; touches the data layer and the API client on the frontend."
- "Refactor of the parser internals; no public API change."
- "Documentation reorganization; no code changes."

This characterization helps the subagents target their work. It does not bake in any review template — the actual focus areas and checks are still derived by the subagents from the diff itself.

---

## Phase 2 — Gather conventions and diff findings (delegate to subagents in parallel)

> **Spawn both subagents in a single message so they run concurrently.** This is the most important performance step in the skill — do not spawn them sequentially.

Spawn two subagents via the Task tool:

1. **Convention auditor** — establishes what this repo's conventions actually are.
2. **Diff investigator** — reviews the specific diff and derives a common-miss checklist from what the diff contains.

Don't read repo files directly in the main thread.

### After both subagents return

If the two reports conflict on a convention (e.g., the convention auditor says the repo uses snake_case for DB columns but the diff investigator says the diff follows camelCase consistent with nearby files), do not silently pick one. Surface the conflict explicitly in the review under **Recommended**, note both findings and their sources, and let the user decide.

If either subagent flags major gaps (couldn't find expected files, something needs deeper investigation), spawn a follow-up subagent with a narrower prompt rather than reading files in the main thread.

### 2.1 Convention auditor prompt

```
You are documenting a codebase's conventions to support a code review.
Your job is to report what THIS repo's conventions actually are.
Do not review code.

DIFF CHARACTERIZATION (one-line, from the reviewer):
<paste the one-sentence characterization>

YOUR JOB:

1. Identify the kind of repo and stack
   - Application (web backend / web frontend / mobile / desktop / CLI), library,
     infrastructure-as-code, scripts, documentation, monorepo with several of these.
   - Languages, frameworks, build tools, test frameworks, package managers visible
     from config files.

2. Read whichever orientation files exist: README.md, CLAUDE.md, AGENTS.md,
   ARCHITECTURE.md, CONTRIBUTING.md, STYLE.md, .editorconfig, code-of-conduct.

3. Identify formatter / linter / type-checker by config file presence (prettier,
   eslint, black, ruff, rustfmt, gofmt, mypy, tsconfig, terraform fmt, hadolint,
   etc.). Note how each is run.

4. Identify the test framework(s) and where tests live.

5. Identify any tooling specific to the kinds of change in the diff:
   - If the diff touches schema/migrations: which migration tool, where do migrations
     live, what's the naming convention, are migrations expected to be reversible?
   - If the diff touches infrastructure: which IaC tool, how is state managed, what's
     the deploy/plan flow?
   - If the diff touches a public API: how is API stability/versioning handled?
   - If the diff adds dependencies: where are they declared and locked?
   - If the diff touches docs: where do docs live, what's the doc format, is there a
     build/preview command?
   Skip categories that don't apply.

6. Note naming conventions across the categories that apply: variables, files,
   functions/types, modules, public-API symbols, DB columns, config keys.

7. Note branch and commit conventions visible from `git log --oneline -30` and
   `git branch -a`.

8. Note where new config values / env vars / secrets / dependencies should be
   documented.

9. Note anything in CLAUDE.md, AGENTS.md, or CONTRIBUTING.md that the reviewer
   should enforce.

REPORT BACK as titled sections covering items 1-9.
Cite real paths. Keep under ~700 words.
DO NOT review the diff. Just report what the repo's conventions are.
```

### 2.2 Diff investigator prompt

```
You are checking a freshly-written diff against existing repo patterns and surfacing
issues for a senior reviewer to weigh. Your job is to surface findings, NOT to write
the review.

DIFF:
<paste the diff>

DIFF CHARACTERIZATION:
<paste the one-sentence characterization>

PLAN (if available):
<paste the plan; otherwise omit this block>

YOUR JOB:

A. Pattern conformance — for each non-trivial change in the diff, find the closest
   existing pattern in the repo (use grep/find/glob) and report whether the diff
   follows it. Cover only what's in the diff. Examples:
   - New endpoint/handler/command added → does it match how existing ones are
     structured, registered, and protected?
   - New module/package added → does it match how existing ones are organized,
     exported, and documented?
   - New external integration → does it go through any existing wrapper layer, or
     is it a new ad-hoc call?
   - New infrastructure resource → does it match the conventions of nearby resources
     (tagging, naming, IAM scoping)?
   - New test → does it match where and how existing tests are organized?
   - New dependency → is it declared and pinned consistent with other dependencies?

B. Derive a common-miss checklist FROM what THIS diff actually contains.
   Do NOT use a generic template. For each kind of change in the diff, think:
   "what's the most common way for this kind of change to ship incomplete or wrong?"
   Then check the diff for that.

   Some examples of how the checklist varies — adapt to the diff:
     - Diff adds a new endpoint → check: registered in the entry point?
       auth/permissions consistent with neighbors? input validation? error handling
       for realistic failures?
     - Diff adds a Terraform resource → check: outputs added if other modules need
       them? IAM scoped tightly? tags consistent with org standard? state backend OK?
     - Diff adds a library function → check: exported from the public surface?
       documented? tested? handles the realistic edge cases?
     - Diff adds a migration → check: code that uses the new schema also present?
       down/reverse path written if the repo expects it? not modifying an
       already-merged migration?
     - Diff adds a frontend component → check: shared primitives used where they
       exist? state/data fetching follows the repo's pattern? accessible attributes
       present where expected?
     - Diff adds a CLI command → check: registered with the parser? help text?
       exit codes consistent?
     - Diff adds a Dockerfile/CI change → check: cache layers sensible? secrets
       handled? matrix consistent?
     - Diff is docs-only → check: links not broken? cross-references consistent?
       formatting matches the rest?
   These are EXAMPLES, not a checklist to run. Build the actual checklist from
   the diff in front of you.

C. Internal consistency check — verify that the diff is consistent within itself:
   - Are new types, functions, or exports used consistently across all the changed
     files, or does the diff introduce something and use it incorrectly elsewhere
     in the same diff?
   - Are there any call sites in the diff that reference a new function/type with
     the wrong signature or wrong field names?
   - Are there any imports in the diff that reference something the diff itself
     hasn't added yet (i.e., the diff is incomplete)?

D. Universal hygiene checks (apply to any diff regardless of kind):
   1. Strings that look like secrets, API keys, tokens, passwords, private keys.
   2. Debug output left in: console.log, print, println!/dbg!/eprintln! used for
      debugging, fmt.Println debug, p() / pp(), `puts` debug, etc. — adapt to
      the language.
   3. TODO/FIXME/XXX/HACK markers added in this diff.
   4. Commented-out code added in this diff.
   5. Hardcoded values that look like they should be configurable (URLs, paths,
      magic numbers, timeouts, retry counts).
   6. Files that look accidentally committed (build artifacts, lock files for tools
      the repo doesn't use, IDE config that isn't gitignored elsewhere, OS junk
      like .DS_Store).
   7. New dependencies that aren't reflected in the manifest/lockfile or vice versa.

E. If a plan was provided: cross-check.
   - Plan items present in the diff.
   - Plan items missing from the diff.
   - Diff items not mentioned in the plan (could be fine, could be scope creep —
     flag, don't judge).

REPORT BACK with:
- **Pattern conformance**: one block per non-trivial change, with `path:line`.
- **Derived common-miss findings**: numbered list, only items that fired. For each:
  the issue, the file/line, and a severity hint (blocking / recommended / nit).
- **Internal consistency findings**: only items that fired, with `path:line`.
- **Universal hygiene findings**: only items that fired.
- **Plan vs. diff** (if a plan was provided): three lists.
- **Cannot determine**: anything needing runtime knowledge or out-of-scope checking.

Cite real paths. Keep under ~800 words.
Do NOT write the review — just findings for the reviewer to weigh.
```

---

## Phase 3 — Synthesize the review

The main thread does this. Combine the two subagent reports with your own reading of the diff to produce the review.

### Review output structure

1. **Warnings passthrough** — any warnings carried forward from `implement`'s Phase 5 report. Omit if none.
2. **Verdict** — one of: **looks good (ship it)** / **needs changes** / **needs major rework**.
3. **Blocking** — must fix before merge. Each item: what's wrong (with `path:line`), why it matters, the concrete fix.
4. **Recommended** — strong recommendations, not blocking. Each item: what + why + suggested fix. The user may justify skipping these. Convention conflicts between subagent reports go here, with both findings and their sources noted.
5. **Nits** — optional polish. Naming, ordering, wording. Brief.
6. **Looks good** — include only things that are genuinely well done and worth calling out: a non-obvious design choice that paid off, thorough tests for a tricky case, clean handling of an edge case the plan didn't specify. Skip this section entirely if there's nothing substantive — do not pad with faint praise.
7. **Could not verify** — anything the subagents flagged as needing runtime knowledge or out-of-scope checking. If verification was skipped due to no repo, say so here.

### Needs major rework verdict

When the verdict is **needs major rework**, include a **Correct understanding** note directly below the verdict — 3–5 sentences summarising what's fundamentally wrong and what the correct approach should be. This gives the user concrete inputs when they revisit `make-plan`. Then still list **Blocking** items so the full scope of problems is visible, but skip **Recommended** and **Nits** — they're not worth addressing on a diff that needs a rethink.

### What goes in which tier

These are universal across repo types:

- **Blocking**: security issues, data-loss risk, breaking changes to a stable surface, secrets in diff, irreversible destructive changes without explicit acknowledgement, modifications to append-only history, broken tests, code that won't run, broken builds.
- **Recommended**: layering or convention violations, missing input validation on non-critical paths, hardcoded values that should be configurable, missing error handling on realistic failure paths, missing tests for new behaviour, conventions diverged from but not catastrophically so.
- **Nits**: naming preferences, comment wording, import ordering, minor structural suggestions, anything where reasonable people could disagree.

### Style rules

- **Be direct.** A senior reviewer doesn't hedge. If the diff is good, a one-line "looks good — ship it" is a valid review.
- **Cite specifics.** Every finding should reference `path/to/file.ext:LINE` from the diff or the subagent reports. No vague "consider error handling" notes.
- **Don't invent categories.** If the diff doesn't touch a thing, no section about that thing.
- **Distinguish your inferences from subagent findings.** When you assert something, note whether it came from the convention auditor, the diff investigator, or your own reading.
- **Don't pad.** A finding belongs only if you can name the file, the issue, and the fix.

---

## Phase 4 — Deliver

Present the review. Do not auto-apply fixes — the user wrote the code and should apply changes themselves or explicitly ask for help on specific items. This is a deliberate choice: code review is a conversation, not a transformation.

After delivering, offer:

> Want me to walk through any of the **blocking** items in more detail, or help fix a specific one?

If the user picks a specific item to fix:
1. Read the relevant file(s) fresh from disk.
2. Propose the minimal targeted change — show it as a diff or inline edit.
3. Ask the user to confirm before applying.
4. Apply only that item. Do not sweep related issues at the same time.

If the verdict is **needs major rework**, direct the user to revisit `make-plan` / `review-plan` for the affected area, and give them the **Correct understanding** note from Phase 3 as their starting input.

---

## Rules

- **Don't write replacement implementations** in the review. Show small illustrative snippets if needed (1–3 lines) to clarify a fix, but the user does the actual fix.
- **Don't expand scope.** Review what's in the diff. If the diff exposes problems with code outside the diff, mention it under **Could not verify** or **Recommended** — don't drag the review into the rest of the codebase.
- **Cite real paths only.** Sourced from the subagent reports or the diff itself. No fabricated paths.
- **Respect the author.** The goal is to make the change better and ship it.