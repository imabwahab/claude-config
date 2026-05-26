---
name: make-plan
description: Produce a concrete implementation plan for a ticket before any code is written, then save it as a markdown file. Use this skill whenever the user provides ticket content (any format: pasted text, plain-text spec, or a ticket ID with description) and asks to plan, scope, break down, or "figure out how to implement" the work. Trigger on phrases like "plan this ticket", "how should I implement", "scope this out", "break this down", or when the user pastes a ticket/issue description and asks what to do next. The skill delegates repo investigation to subagents (Task tool) to keep the main thread's context clean, then synthesizes findings into a plan grounded in real files. It does not assume any fixed architecture and infers affected areas from the ticket.
---

# Make Plan

Produce a concrete implementation plan for a ticket, grounded in the actual repository, and save it as a markdown file.

The skill has four phases: **understand** the ticket, **investigate** the repo (via subagents), **synthesize** the plan, then **save** it.

---

## Phase 1 — Understand the ticket

If the user has not provided ticket content, ask for it. Accept any of:
- Pasted ticket text (Jira, Linear, GitHub issue, plain-text spec, etc.)
- A ticket ID/number — ask the user for the title and description if only an ID is given
- A link — **do not fetch the URL** (fetching external URLs is unreliable and may expose sensitive links); ask the user to paste the relevant content instead

Extract from the ticket:
- **Goal** — what the user/system should be able to do after this ships
- **Acceptance criteria** — explicit or implied
- **Constraints** — deadlines, must-not-break clauses, compatibility requirements
- **Open questions** — anything ambiguous

If the ticket is too vague to plan against (e.g., "improve onboarding" with no detail), stop and ask the user for clarification before proceeding.

Also identify the **investigation areas** — the distinct parts of the codebase the ticket likely touches. Examples:
- "Add CSV export to reports page" → two areas: reports backend, reports frontend.
- "Fix race condition in payment webhook" → one area: payment/webhook code.
- "Rename column X to Y across the system" → many areas: schema, every query referencing X, every API response, every frontend usage.

This list determines how many investigation subagents to spawn in Phase 2.

---

## Phase 2 — Investigate the repo (delegate to subagents)

**Delegate repo investigation to subagents via the Task tool.** Do not read repo files directly in the main thread — that pollutes the planning context with file contents you don't need to keep around. The subagent reads, distills, and returns a summary; the main thread keeps only the summary.

### No repo available?

If the user provides a ticket but no repository (no local path, no files, no context), skip subagent spawning entirely. Note in the plan that investigation was skipped and flag every path listed under Affected Areas as **assumed, not confirmed**. List the most likely file locations based on common conventions for the inferred stack, but mark them clearly so the implementer knows to verify before starting.

### When to spawn one vs. many

- **One investigator** — small or single-area tickets (one feature, one bug, one focused change). Most tickets are this.
- **Multiple investigators in parallel** — tickets that touch distinct, separable areas (frontend + backend + schema, or several unrelated modules). Spawn them in a single message with multiple Task tool calls so they run concurrently. In practice, beyond ~4 agents the synthesis overhead tends to outweigh the parallelism benefit — use judgment rather than always hitting the cap.

### Conflicting reports

If two investigators return contradictory findings (e.g., different assumed entry points, conflicting conventions about file layout or naming), do not silently pick one. Note the conflict explicitly in the relevant plan section and add it to Open Questions so the implementer can resolve it before starting.

### Investigator prompt template

When spawning an investigator subagent, give it a prompt with this shape:

```
You are investigating a codebase to support implementation planning for a ticket.

TICKET:
<paste the relevant ticket content, or just the parts relevant to this investigator's area>

YOUR AREA:
<the specific area this investigator owns, e.g., "the payment webhook handler and surrounding service code">

YOUR JOB:
1. Read repo orientation files if they exist: README.md, CLAUDE.md, AGENTS.md, ARCHITECTURE.md, CONTRIBUTING.md.
2. Identify the stack from package.json / pyproject.toml / Cargo.toml / go.mod / equivalent.
3. Find the files most relevant to the ticket area. Use grep/find/glob to locate them — do not guess paths.
4. Read those files (or the relevant sections) to understand existing patterns.
5. Note the repo's conventions: naming (snake_case vs camelCase), file layout, branch naming, commit style.
   To surface branch/commit conventions, run:
     git log --oneline -10
   And check: CONTRIBUTING.md, .github/pull_request_template.md, .github/ISSUE_TEMPLATE/

REPORT BACK with:
- **Stack & conventions**: language(s), framework(s), notable conventions to follow.
- **Relevant files**: list of `path:line` references the planner should know about, each with a one-line note on what it does or what pattern it demonstrates.
- **Existing patterns to copy**: 1–3 specific examples of how similar work has been done before in this repo, with paths.
- **Gaps**: anything the ticket implies the repo does not currently have (e.g., no feature flag system, no migration tooling).
- **Branch convention**: the pattern found (e.g., `feat/<initials>/<ticket>`), or "not visible" if nothing surfaced.

DO NOT propose an implementation. DO NOT write code. Just report what's there.

Keep the report under ~400 words. Cite real paths only.
```

Tailor the area description and the ticket excerpt to what each investigator needs. If you spawn multiple, give each a distinct, non-overlapping area so they don't duplicate work.

### After subagents return

Collect their reports. If something critical is missing or ambiguous (e.g., the investigator couldn't find a file you'd expect), spawn a follow-up investigator with a more targeted prompt rather than reading the file yourself.

---

## Phase 3 — Synthesize the plan

The main thread does this step — synthesis is where everything comes together and shouldn't be delegated.

Produce the plan with the sections below. Adapt: omit a section if it genuinely does not apply, but do not invent content to fill space.

### Section structure

Group sections into two logical clusters:

**Context** (orientates the reader)
1. **Summary** — one or two sentences restating the goal in plain language.
2. **Branch name** — use the convention surfaced by the investigator(s); if none was visible, propose `feat/<short-slug>` and note it's a guess.
3. **Affected areas** — grouped by category (backend, data layer, frontend, auth, integrations, infra, tests, docs — only the ones that apply). Each item names concrete paths from the investigator reports. If the ticket touches an area the repo does not currently have, say so explicitly. If investigation was skipped, mark all paths as *assumed*.

**Execution** (guides the implementer)

4. **Approach** — a numbered walkthrough of how the change flows end-to-end. Write it as a chain of responsibility, not pseudocode. Each step names the actor and the handoff:

   > 1. User clicks **Export** on `ReportPage`
   > 2. `ReportPage` dispatches `exportCSV(reportId)` action
   > 3. Action calls `POST /api/reports/:id/export`
   > 4. Route handler delegates to `ReportService.generateCSV()`
   > 5. Service queries DB, streams CSV response back to client

   Adapt the depth and actors to the ticket. The goal is a map of the change, not a spec.

5. **Step-by-step implementation order** — an ordered checklist of work items, sized so each one is a sensible commit. Schema/migration work and shared types come first so dependents can build on them.
6. **Risks and watch-outs** — destructive migrations, irreversible operations, cost-bearing API calls, auth/permission changes, breaking API changes, anything touching production data.
7. **Test approach** — what to verify before opening a PR. Be specific: which test files to add or extend, what to click through manually, what edge cases matter.
8. **Open questions** — anything the ticket did not specify that the implementer needs answered, including any conflicts surfaced between investigator reports. Omit if empty.

### Rules

- **Do not write implementation code.** Code snippets are acceptable only as one- or two-line examples to illustrate a pattern reference.
- **Cite real paths** from the investigator reports. Use `path/to/file.ext:line`. No fabricated paths — if the investigator didn't surface it, don't invent it. If investigation was skipped, mark assumed paths clearly.
- **Follow existing conventions** as reported by the investigator (naming, file layout, branch/commit style).
- **Be honest about uncertainty.** If something requires a decision the implementer or reviewer must make, list it under Open Questions rather than guessing.

---

## Phase 4 — Save the plan

Save the plan as a markdown file under a single consistent root: `.claude/plans/`

- If the ticket has an ID: `.claude/plans/<ticket-id>-<short-slug>.md`
  (e.g., `.claude/plans/PROJ-123-add-csv-export.md`)
- If no ID: `.claude/plans/<short-slug>.md`
  (e.g., `.claude/plans/add-csv-export.md`)

If the `.claude/plans/` directory does not already exist, confirm the path with the user before creating it — they may prefer a different location (e.g., `docs/plans/`). If the investigator surfaced a different plan/doc convention in the repo, defer to that and note the deviation.

Show the user the full path where the file was saved.