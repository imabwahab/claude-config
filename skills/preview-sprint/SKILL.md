---
name: preview-sprint
description: Produce a sprint preview document for a set of upcoming-week tickets — covering readiness, effort/complexity, inter-ticket dependencies, and risk per ticket — by investigating one or more repositories the tickets will land in. The user pastes a Linear export (titles, descriptions, and any available metadata like assignee and priority); the skill investigates each ticket across the named repos and produces a markdown report at `.claude/sprint-previews/<slug>.md`. Trigger on phrases like "analyze these tickets for next week", "sprint preview", "what's in next week's sprint", "prep these tickets", "check upcoming tickets", "what's the state of these tickets before we start", or when the user pastes an upcoming-sprint ticket list and asks for analysis. Delegates per-ticket investigation to batched subagents (Task tool) running in parallel. Read-only across all repos and git history. Requires explicit repo paths from the user. Produces an analysis only — does not propose implementations, draft plans, or assign work.
---

# Preview Sprint

Take a list of upcoming-week tickets and produce a sprint preview — a single document that, for every ticket, tells the team: is it ready to start, how much work is it, what does it depend on, and what's risky about it. The analysis is grounded in the actual state of one or more repositories the tickets will land in.

The skill is **read-only across all repos and git history**. It runs no DB queries, no tests, no app code. It reads files and runs read-only git inspection only.

The skill has seven phases: **prepare**, **collect repo paths**, **parse the ticket list**, **route tickets to repos**, **investigate in parallel** (via batched subagents), **synthesize the sprint preview**, then **save and report**.

---

## Phase 1 — Prepare

1. **Get the ticket list.** Accept either:
   - A path to a markdown file containing the Linear export, or
   - A path to csv file containing the linear export, or
   - Pasted content directly in the message.

   The Linear export should include titles and descriptions at minimum. Assignee, priority, labels, estimate points, and any other Linear fields are useful when present — the analysis will incorporate them when available, and note when they're absent.

2. **Note the sprint context.**
   - A **slug** for the sprint preview. Derive from a week marker, a sprint name, or the dominant project: `week-of-2025-11-17`, `sprint-23`, `q4-launch-week-2`. Ask the user once if ambiguous.
   - Today's date — useful for the report header and for resolving relative phrases in ticket descriptions ("the rollout next Monday").

3. **Set up the output path.** The preview saves to `.claude/sprint-previews/<slug>.md`. Create `.claude/sprint-previews/` if it doesn't exist.

4. **If `.claude/sprint-previews/<slug>.md` already exists**, ask whether to overwrite, save under a versioned name like `<slug>-2.md`, or stop. Don't silently overwrite — sprint previews often get annotated during sprint planning meetings.

---

## Phase 2 — Collect repo paths (main thread)

Do NOT default to the current working directory. Ask the user to name every repo path the analysis should cover, even if it's just one.

Accept any number of paths. They can be:
- The current repo (`.`)
- Sibling repo paths (`../backend`, `../infra`)
- Absolute paths
- A mix

For each path:
- Confirm it exists and is a directory.
- If it's a git repo, record its current branch and current commit (`git rev-parse HEAD`).
- If it's not a git repo, record that.
- Ask the user for a **short label** for each repo (e.g., `backend`, `frontend`, `infra`, `data-pipeline`). Labels are used throughout the preview to route tickets and reference findings — they need to be short and stable. If the user doesn't suggest labels, derive them from the directory name.

Stamp all of this in the preview's header so the document is reproducible.

---

## Phase 3 — Parse the ticket list (main thread)

Read the export and produce a structured ticket list. For each ticket extract:

- **ID** (e.g., `PROJ-123`) — if present. If absent, generate sequential IDs (`T-1`, `T-2`, ...) and note in the preview that IDs were synthesized.
- **Title** — short, one line.
- **Description** — everything else for that ticket, verbatim.
- **Metadata** — whatever Linear fields the export includes:
  - Assignee
  - Priority
  - Labels / tags
  - Estimate / story points
  - Status (if present — but treat the analysis as forward-looking; status is context, not a gate)
  - Linked tickets (parent, blocks, blocked-by, related — if Linear exported them)
  - Cycle / sprint reference (if present)

If the parser can't cleanly separate tickets, surface the issue with an example and ask. A wrong parse propagates into every downstream analysis.

Count the tickets. If under 5, the skill will use a single subagent in Phase 5. If 5+, batch.

---

## Phase 4 — Route tickets to repos (main thread)

For each ticket, decide which repo(s) the investigation should target. Routing strategies, in order of confidence:

1. **Explicit signals in the ticket.** Labels like `backend`, `frontend`, `infra`, repo names mentioned in the description, file paths cited, ticket prefixes that map to a repo. If the ticket says "in the `infra/` repo we need to..." — that's the repo.

2. **Subject-matter inference.** Tickets about UI/components/routes/pages → frontend repo. Tickets about endpoints/migrations/jobs → backend repo. Tickets about Terraform/K8s/CI/deploy → infra repo. Only use this when the labels above are absent.

3. **Cross-cutting.** Some tickets genuinely touch multiple repos (e.g., "add a new field to the user model, expose it via the API, and surface it in the settings page"). Route these to all relevant repos; the investigation will report per-repo findings under one ticket.

4. **Ambiguous.** If routing is genuinely unclear, mark the ticket `ROUTING_UNCLEAR` and ask the user for a hint before investigation. Don't guess — investigating in the wrong repo wastes a subagent and produces wrong findings.

### Present the routing to the user before investigating

Show the routing as a table:

```
Routed N tickets across <repo labels>:

| Ticket | Title | Routed to | Routing reason |
|---|---|---|---|
| PROJ-101 | Add user preferences UI | frontend | label: frontend |
| PROJ-104 | User preferences API | backend | inference: API endpoint |
| PROJ-110 | Add Postgres replica | infra | label: infrastructure |
| PROJ-115 | Add preferences column + UI | backend, frontend | cross-cutting |
| PROJ-118 | Reduce checkout latency | ROUTING_UNCLEAR | which repo? |

Confirm or adjust routing before I investigate.
```

The user can correct routings, merge tickets, or split cross-cutting tickets into per-repo investigations.

---

## Phase 5 — Investigate tickets in parallel (delegate to batched subagents)

This is the heavy phase. Each ticket needs four kinds of analysis (readiness, effort, dependencies, risk) against one or more repos. Spawn subagents in parallel.

### Batching

- **Per repo, batch tickets that route to that repo.** A subagent's context is one repo, so co-locating same-repo tickets in one subagent is efficient.
- **Cross-cutting tickets** get one investigation per repo they touch — the main thread synthesizes the findings across repos.
- **Batch size:** 4–8 tickets per subagent.
- **Parallelism cap:** ~6 subagents at once. Beyond that, synthesis cost dominates. If the sprint has 60+ tickets across 3 repos, run more batches sequentially.

Spawn the first parallel set in a single message.

### Subagent prompt template (per repo, per batch)

```
You are analyzing upcoming-sprint tickets against a real codebase. For each
ticket you'll produce four findings: readiness, effort, dependencies, and risk.
You are NOT writing code, drafting plans, or proposing implementations.
You are inspecting and reporting.

REPO:
  Label: <repo label>
  Path: <repo path>
  Branch: <branch>
  Commit: <short SHA>

TICKETS TO ANALYZE (in this batch):

<for each ticket in the batch, paste:
  ### TICKET <ID>: <title>
  Assignee: <assignee or "unassigned">
  Priority: <priority or "not set">
  Labels: <labels or "none">
  Estimate: <story points / hours or "not set">
  Linked tickets: <parent / blocks / blocked-by / related, if present>
  Description:
  <full description verbatim>
  --->

OTHER TICKETS IN THIS SPRINT (for dependency analysis only — do not
analyze these, just be aware of them when checking dependencies):

<list of all OTHER ticket IDs + titles in the sprint, so this subagent
can spot dependencies between tickets in its batch and other tickets>

YOUR JOB:

For each ticket in your batch, produce all four findings.

A. READINESS — is this ticket ready to start work on?
   Inspect the description and any linked tickets. Assign one of:
     - READY — description is concrete enough to plan against, acceptance
       criteria are clear (explicit or implied), no obvious blockers in
       the description.
     - NEEDS_CLARIFICATION — description is too vague or has open
       questions that would block planning. List the specific
       clarifications needed.
     - BLOCKED — depends on something not done (another ticket in this
       sprint that's prerequisite, a design decision pending, external
       dependency not in place). Name the specific block.
   Cite path:line if the readiness verdict depends on code state (e.g.,
   "the model this ticket references is at src/models/user.ts:14, so
   the data layer is ready").

B. EFFORT — what's the implementation surface?
   Read the repo to understand what would have to change. Identify:
     - The files / modules / areas likely to be touched, with path:line
       references where the closest existing pattern lives.
     - Whether this ticket is mostly:
         TRIVIAL — one-file, mechanical change (config, copy, single
                   function)
         SMALL — a few files, follows an existing pattern straightforwardly
         MEDIUM — multiple files, introduces a new pattern or significant
                  new behavior
         LARGE — multiple modules, schema/migration changes, or
                 cross-system coordination
     - Any hidden complexity — places where the work LOOKS small but
       isn't (touches many call sites, requires data migration,
       integration tests need new fixtures, etc.).
   This is a sizing aid for sprint planning. It is NOT a formal estimate
   — be honest about what you can and can't tell from the code alone.

C. DEPENDENCIES — what does this ticket depend on?
   Look at the description and the code to identify:
     - Other tickets in this sprint that need to happen first. Cite the
       OTHER ticket's ID from the "other tickets" list above. Reasons can
       be: shared schema, shared module, sequenced rollout, one ticket's
       output is another ticket's input.
     - External dependencies — third-party services not yet integrated,
       data not yet available, decisions not yet made.
     - Code dependencies that already exist (good) vs need to be built
       (a sprint-internal dependency).
   Be conservative: only report dependencies you can directly see in the
   code or the description. Don't invent dependencies from ticket
   numbering alone.

D. RISK — what's risky about this ticket?
   Scan for any of:
     - Cost-bearing third-party API calls (payment, SMS, email at scale,
       LLM, etc.).
     - Auth/permissions changes — anywhere this ticket would alter what
       users can do.
     - Migrations or schema changes, especially destructive ones (drop
       column, drop table, irreversible data transforms).
     - Touches code with known fragility (a TODO from a prior incident,
       a test file marked flaky, a module with cyclic dependencies).
     - Public API changes (response shape, status codes, endpoint paths).
     - Performance-critical paths.
     - Background jobs / queues / cron — anything async with retry
       semantics that's easy to get subtly wrong.
   For each risk: name it, cite path:line, and explain WHY it's risky.
   If a ticket has no notable risks, say so explicitly.

REPORT BACK with one block per ticket, in the order given:

  TICKET <ID>: <title>
  ROUTING: <repo label>

  READINESS: <READY | NEEDS_CLARIFICATION | BLOCKED>
    Justification: <one line>
    <if NEEDS_CLARIFICATION: bulleted list of specific clarifications needed>
    <if BLOCKED: what's blocking>

  EFFORT: <TRIVIAL | SMALL | MEDIUM | LARGE>
    Surface: <bulleted list of files/areas with path:line>
    Hidden complexity: <list, or "none">

  DEPENDENCIES:
    On other sprint tickets: <list of TICKET IDs + one-line reasons, or
                              "none">
    External: <list, or "none">
    Already-built infrastructure: <bulleted list, or "none">

  RISK: <bulleted list of risks, each with name + path:line + why; or
         explicit "no notable risks">

Cite real paths only. If a path you'd expect doesn't exist, say so
explicitly — don't invent paths.

Keep each ticket's block under ~400 words.

DO NOT propose implementations. DO NOT draft plans. DO NOT recommend
sequencing decisions — the main thread will synthesize sequencing.
```

### After the subagents return

Collect all per-ticket blocks across all batches.

For cross-cutting tickets (investigated in multiple repos), merge their per-repo findings: combine the readiness verdicts (the strictest applies — if any repo says BLOCKED, the overall is BLOCKED), sum the effort sizing (LARGE in one repo + SMALL in another = LARGE overall), union the dependencies and risks.

If any subagent's report looks thin (e.g., a ticket investigated against a 3000-file repo that only cites 2 paths), spawn a focused follow-up subagent for that specific ticket with a narrower prompt before locking in the findings.

---

## Phase 6 — Synthesize the sprint preview (main thread)

Combine all per-ticket findings into a coherent document. Synthesis means:

### Compute summary metrics

- Total tickets in the sprint
- Counts by readiness verdict (READY / NEEDS_CLARIFICATION / BLOCKED)
- Counts by effort sizing (TRIVIAL / SMALL / MEDIUM / LARGE)
- Counts by repo
- Counts by assignee, if metadata available
- Tickets with one or more risks flagged

### Build the dependency graph

Collect all "depends on other sprint tickets" entries. Construct a simple topological-ish ordering: tickets with no dependencies in the sprint go first; tickets with dependencies come after the things they depend on.

If there's a cycle (A depends on B, B depends on A), surface the cycle prominently — these often indicate tickets that should have been merged or split.

If a chain of dependencies is long (A → B → C → D), call it out — these tend to compress sprint capacity.

### Identify the "start here" candidates

Tickets that are READY, low/medium effort, and have no in-sprint dependencies. These are the tickets that can be picked up first without waiting on anything.

### Identify the "needs attention" tickets

Any combination of:
- NEEDS_CLARIFICATION tickets — surface the specific clarifications needed; these need to be resolved before sprint kickoff.
- BLOCKED tickets — surface what's blocking; sprint planning should reconcile or descope.
- High-risk tickets — surface for explicit acknowledgement.
- Cross-cutting tickets — surface because they need coordination.

### Don't editorialize beyond the analysis

The preview reports what was found. It does NOT recommend sprint scope changes, suggest who should work on what, or propose how to break down complex tickets. If the analysis suggests issues, surface the analysis; let the team decide.

---

## Phase 7 — Save and report

### Sprint preview file structure

Save to `.claude/sprint-previews/<slug>.md`. Use this structure:

```markdown
# Sprint preview: <slug>

> Generated on: <ISO date>
> Source: <"pasted" or path to Linear export>
> Total tickets: <N>
> Repos analyzed:
>   - `<label-1>` at `<path>` (branch `<branch>` @ `<short-sha>`)
>   - `<label-2>` at `<path>` (branch `<branch>` @ `<short-sha>`)

## Summary

| Metric | Count |
|---|---|
| Total tickets | <N> |
| Ready to start | <n> |
| Needs clarification | <n> |
| Blocked | <n> |
| Trivial / Small | <n> |
| Medium | <n> |
| Large | <n> |
| Tickets with flagged risks | <n> |
| Cross-cutting (multi-repo) | <n> |

<By repo:>
| Repo | Tickets |
|---|---|
| <label-1> | <n> |
| <label-2> | <n> |
| cross-cutting | <n> |

<By assignee (if metadata available):>
| Assignee | Tickets | Total effort |
|---|---|---|
| <name> | <n> | <rough sum, e.g. "1 L + 2 M + 1 S"> |
| unassigned | <n> | ... |

## Start-here candidates

<Tickets that are READY, TRIVIAL/SMALL, and have no in-sprint dependencies.
Each one in one line:
  - <TICKET-ID>: <Title> — <repo> — <one-line note on why it's a good
    first pick>>

## Needs attention before sprint kickoff

### Clarifications needed

<NEEDS_CLARIFICATION tickets. For each:
  - <TICKET-ID>: <Title> — <repo>
    Open questions:
      - <specific clarification 1>
      - <specific clarification 2>
>

### Blocked tickets

<BLOCKED tickets. For each:
  - <TICKET-ID>: <Title> — <repo>
    Blocked by: <what's blocking>
    Resolution: <what needs to happen for it to unblock>
>

### High-risk tickets

<Tickets with notable risks. For each:
  - <TICKET-ID>: <Title> — <repo>
    Risks:
      - <risk 1 with path:line>
      - <risk 2 with path:line>
>

### Cross-cutting tickets

<Tickets that touch multiple repos. For each:
  - <TICKET-ID>: <Title>
    Repos: <list>
    Coordination notes: <what needs to be sequenced or aligned across
    repos>
>

## Dependency graph

<Suggested ordering based on in-sprint dependencies:

  1. <TICKET-ID>: <Title> — independent, can start immediately
  2. <TICKET-ID>: <Title> — independent, can start immediately
  3. <TICKET-ID>: <Title> — depends on (1)
  4. <TICKET-ID>: <Title> — depends on (3)
  ...

If cycles exist:
  Dependency cycles detected:
    - <TICKET-A> depends on <TICKET-B> which depends on <TICKET-A>
      (likely a sign these should be merged or split)

If long chains exist:
  Long dependency chains:
    - <TICKET-A> → <TICKET-B> → <TICKET-C> → <TICKET-D>
      (compresses sprint capacity; consider parallelizing or splitting)
>

---

## Per-ticket detail

### <TICKET-ID>: <Title>

**Routing:** <repo label(s)>
**Assignee:** <name or "unassigned">
**Priority:** <priority or "not set">
**Labels:** <labels or "none">
**Estimate:** <story points / hours or "not set">

#### Readiness: <READY | NEEDS_CLARIFICATION | BLOCKED>

<Justification in 1-2 sentences. For NEEDS_CLARIFICATION, the bulleted
list of clarifications. For BLOCKED, the specific blocker.>

#### Effort: <TRIVIAL | SMALL | MEDIUM | LARGE>

**Surface:**
- `<repo-label>:<path:line>` — <one-line note>
- ...

**Hidden complexity:** <list, or "none">

#### Dependencies

- **In-sprint:** <list of TICKET IDs with one-line reasons, or "none">
- **External:** <list, or "none">
- **Already-built infrastructure used:** <list, or "none">

#### Risks

<Bulleted list of risks with path:line, or explicit "no notable risks">

---

<repeat per ticket, in dependency-graph order from the section above>

## Caveats

- This preview reflects the state of the repos at the commits listed in
  the header. Work on other branches, in stash, or in unmerged PRs is
  not visible.
- The analysis is **read-only**. No code, repo, or external state was
  modified.
- Effort sizing (TRIVIAL/SMALL/MEDIUM/LARGE) is a rough sizing aid
  derived from code inspection, not a formal estimate. Use it as a
  conversation-starter for sprint planning, not as a commitment.
- Dependencies are derived from visible code relationships and explicit
  ticket links. Implicit dependencies (discussed in Slack, agreed in
  meetings) are not captured — surface those in sprint planning.
- The preview answers "what's the state of these tickets?" It does NOT
  propose how to do them, who should do them, or what to descope. Those
  are team decisions.
- <Any sprint-specific caveats the subagents surfaced.>
```

### Report to the user

After saving, summarize in chat:

- File path: `.claude/sprint-previews/<slug>.md`.
- One-line headline: e.g., "Analyzed 14 tickets across backend, frontend, and infra. 9 ready, 3 need clarification, 2 blocked. 4 tickets have flagged risks. 1 long dependency chain to be aware of."
- The repos and commits the analysis reflects.
- Top 2-3 things to look at first (e.g., "Tickets PROJ-118 and PROJ-122 need clarification before they're plannable; the PROJ-104 → PROJ-110 → PROJ-115 chain is the longest dependency").
- A reminder that no changes were made to any repo.

---

## Rules

- **Read-only across all repos.** No commits, branches, edits, pushes, stashes, rebases — in any repo. File reads and read-only git inspection are fine.
- **Cite real paths only.** Every `path:line` in the preview must point to a file that actually exists at the analyzed commit. No fabricated paths anywhere.
- **One analysis per ticket, four findings each.** Readiness, effort, dependencies, risk. Don't fold them together; don't skip any.
- **Confirm routing before investigating.** A wrongly-routed ticket wastes a subagent and produces wrong findings. Phase 4 confirmation is not optional.
- **No recommendations beyond the analysis.** The preview reports state. It does NOT propose sprint scope changes, suggest who should work on what, or recommend how to break down complex tickets. Surface the analysis; let the team decide.
- **Effort sizing is a sizing aid, not an estimate.** Don't dress it up with hour counts or story-point conversions the code can't actually tell you.
- **The preview is a snapshot.** Stamped with each repo's branch and commit so it's reproducible and re-runnable later in the sprint.
- **Loud about uncertainty.** Anything the investigation couldn't determine from code alone is named explicitly per ticket — don't paper over gaps.
- **No implementation proposals.** This skill analyzes. To act on a ticket, use `make-plan` next. To investigate a blocker, use `investigate-request` or `investigate-bug`.
