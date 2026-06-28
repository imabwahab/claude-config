---
name: sprint-postmortem
description: Audit a completed sprint against the repo(s) it landed in, then produce QA verification instructions for everything that actually shipped. The user provides a CSV of the sprint's tickets (typically from Linear, with whatever columns Linear exports — at minimum ID, title, description, and status); the skill detects the columns, audits each ticket marked "Done" against the repo(s) to confirm or refute the team's done-claim, and writes QA walkthroughs for confirmed-done tickets. The result is one document combining (a) a retrospective summary — what the team said vs. what actually shipped — and (b) a runnable testing plan for the shipped work. Saved to `.claude/sprint-postmortems/<slug>.md`. Trigger on phrases like "post-mortem this sprint", "audit the completed sprint", "verify what shipped this sprint", "sprint retro analysis", "did we actually ship X?", or when the user provides a sprint CSV after the sprint has wrapped. Read-only across all repos and git history. Done-claims are treated as the team's reasonable statement of fact; mismatches are surfaced as noteworthy exceptions, not assumed.
---

# Sprint Postmortem

Take a CSV of a completed sprint's tickets and produce a single document that does two things:

1. **Audits** the team's "Done" claim against the actual state of the repo(s), confirming what truly shipped and surfacing any mismatches as exceptions.
2. **Generates QA verification instructions** for each confirmed-shipped ticket, so the team can walk through what actually landed.

The skill is **read-only across all repos and git history**. It reads files and runs read-only git inspection only — no commits, no edits, no DB queries, no app execution.

The skill has six phases: **prepare**, **parse the CSV**, **collect repo paths and route tickets**, **audit each Done ticket** (via batched subagents), **synthesize the postmortem**, then **save and report**.

The skill operates under one important assumption you confirmed: in your team's process, "Done" means shipped and QA'd. The audit treats Done as the team's reasonable claim of fact — it confirms most, surfaces the rare mismatch as noteworthy.

---

## Phase 1 — Prepare

1. **Get the CSV.** Accept either:
   - A path to a CSV file (typical case — Linear's Export feature produces this), or
   - Pasted CSV content directly in the message.

   Don't accept markdown here — this skill is specifically for CSV input. If the user pastes a Linear markdown export instead, point them at `plan-project-tests` (similar shape, markdown input).

2. **Note the postmortem context.**
   - A **slug** for the postmortem. Derive from a week marker, sprint name, or cycle reference: `sprint-23-postmortem`, `week-of-2025-11-17`, `q4-launch-cycle`. Ask the user once if ambiguous.
   - Today's date — useful for the report header.

3. **Set up the output path.** The postmortem saves to `.claude/sprint-postmortems/<slug>.md`. Create the directory if it doesn't exist.

4. **If `.claude/sprint-postmortems/<slug>.md` already exists**, ask whether to overwrite, save under a versioned name (`<slug>-2.md`), or stop. Don't silently overwrite — postmortems often get annotated after retro meetings.

---

## Phase 2 — Parse the CSV and detect columns (main thread)

Linear's CSV export format varies by workspace configuration. The skill detects columns intelligently rather than expecting a fixed schema.

### Column detection

Read the CSV header. Match each column to a known role using both header name and content shape:

- **ID column** — header containing `id`, `identifier`, `key`, or matching a pattern like `PROJ-123`. Required; if no ID column exists, generate sequential IDs (`T-1`, `T-2`, ...) and flag in the report.
- **Title column** — header containing `title`, `name`, `summary`. Required.
- **Description column** — header containing `description`, `body`, `details`. May be absent — many CSV exports omit descriptions; flag if absent.
- **Status column** — header containing `status`, `state`, `workflow`. Required for this skill (status is the audit's anchor). If absent, the skill can't run — ask the user to re-export with status included.
- **Assignee column** — header containing `assignee`, `assigned to`, `owner`. Optional.
- **Priority column** — header containing `priority`, `urgency`. Optional.
- **Estimate column** — header containing `estimate`, `points`, `size`. Optional.
- **Labels column** — header containing `labels`, `tags`. Optional. Useful for repo routing in Phase 3.
- **Cycle / sprint column** — header containing `cycle`, `sprint`, `iteration`. Optional. Useful for confirming you're looking at the right scope.
- **Completed date column** — header containing `completed`, `done at`, `closed`. Optional. Useful for the postmortem header.
- **Other columns** — preserve in the parsed structure but don't act on them unless the user calls them out.

### Status value detection

Linear's status values are configurable per workspace. Detect them from the data:

- Map values to four buckets:
  - **DONE** — values like `Done`, `Completed`, `Closed`, `Shipped`, `Resolved`. These are the team's claim of "this shipped" — the audit's primary scope.
  - **IN_FLIGHT** — values like `In Progress`, `In Review`, `In QA`, `Ready for QA`. Did not ship this sprint.
  - **NOT_STARTED** — values like `Backlog`, `Todo`, `Triage`, `Planned`. Did not ship.
  - **DROPPED** — values like `Cancelled`, `Wontfix`, `Duplicate`. Intentionally not shipped.
- If a value doesn't match these patterns, ask the user how to bucket it before proceeding.

### Surface the parse to the user before continuing

Show the detection:

```
Parsed N tickets from the CSV. Detected columns:
  ID: "Identifier"
  Title: "Title"
  Description: "Description"
  Status: "Status"
  Assignee: "Assignee"
  Priority: "Priority"
  Labels: "Labels"

Status distribution:
  DONE: 12 (the audit will check these)
  IN_FLIGHT: 3
  NOT_STARTED: 1
  DROPPED: 2

Other status values seen: none.

Confirm this looks right, or correct any column / status mappings.
```

Don't proceed until the user confirms. A wrong column mapping silently corrupts the audit.

---

## Phase 3 — Collect repo paths and route tickets (main thread)

### Collect repo paths

Don't default to the current working directory. Ask the user to name every repo path the audit should cover.

For each path:
- Confirm it exists and is a directory.
- If it's a git repo, record its current branch and commit SHA.
- Ask the user for a **short label** (e.g., `backend`, `frontend`, `infra`). The label is used throughout the postmortem.

If the user names only one path, single-repo mode is implicit — no routing needed.

### Route tickets to repos (multi-repo only)

For each DONE ticket, decide which repo(s) the audit should target. Routing strategy (same as `preview-sprint`):

1. **Explicit signals in the ticket** — labels like `backend` / `frontend` / `infra`, repo names mentioned in the description.
2. **Subject-matter inference** — UI/components → frontend, endpoints/migrations → backend, Terraform/CI → infra.
3. **Cross-cutting** — tickets that touch multiple repos get investigated in each.
4. **Ambiguous** — mark `ROUTING_UNCLEAR` and ask the user for a hint before investigating.

Show the routing as a table and ask the user to confirm before investigating. Misrouting in a postmortem produces wrong audit verdicts that get reported up as "things didn't ship" when they actually did.

### Non-DONE tickets

IN_FLIGHT, NOT_STARTED, and DROPPED tickets do NOT get repo investigation. They're listed in the postmortem for completeness but no audit work happens against them. Saves subagent budget for what matters.

---

## Phase 4 — Audit DONE tickets (delegate to batched subagents)

This is the main investigation. Each DONE ticket needs the repo checked for evidence that what the team claimed shipped actually shipped.

### Batching

- Group DONE tickets by repo (so same-repo tickets share subagent context).
- Batch 4–8 DONE tickets per subagent.
- Cap parallelism at ~6 subagents at once.

Cross-cutting tickets get one investigation per repo they route to; the main thread merges those findings in Phase 5.

### Subagent prompt template

```
You are auditing tickets from a completed sprint against a codebase. The
team has marked these tickets as DONE, meaning they believe each one
shipped and was QA'd. Your job is to (a) confirm the done-claim with
evidence, and (b) for confirmed-done tickets, gather the information a
QA-style verification walkthrough would need.

The default expectation is that DONE tickets are genuinely done. A
confirmed-done verdict is the common case. Mismatches (DONE-claimed but
not visibly shipped) are real findings — surface them clearly when you
see them, but don't manufacture mismatches by being overly strict.

REPO:
  Label: <repo label>
  Path: <repo path>
  Branch: <branch>
  Commit: <short SHA>

TICKETS TO AUDIT (in this batch):

<for each ticket in the batch, paste:
  ### TICKET <ID>: <title>
  Status (team's claim): DONE
  Assignee: <name or "unassigned">
  Priority: <or "not set">
  Labels: <or "none">
  Estimate: <or "not set">
  Completed: <date or "not recorded">
  Description:
  <verbatim, or "no description provided">
  --->

YOUR JOB:

For each ticket, in order:

1. Extract the concrete claims from the description — what specific
   behavior, files, endpoints, schema, UI, or capability does this ticket
   say should now exist?
   - If the description is missing or too vague to extract concrete
     claims, fall back to extracting claims from the title plus any
     labels. If even that yields nothing concrete, mark
     UNVERIFIABLE_BY_DESCRIPTION and explain.

2. Look for evidence in the repo. Use grep/find/glob to locate files
   matching the claims. Read the relevant files to confirm.

3. Assign ONE of four verdicts:

   - CONFIRMED_DONE — clear evidence the ticket's claims are implemented.
     Cite at least one path:line per major claim. This is the expected
     verdict for most DONE tickets.

   - PARTIAL — most of the ticket's claims appear implemented, but one
     or more are missing or notably incomplete. Cite both what was found
     AND what's missing. Be specific: "the API endpoint exists at
     <path:line>, but no admin authorization check is present despite
     the description requiring it."

   - MISMATCH — significant claims from the ticket have no visible
     implementation in the repo. The team claimed DONE; the repo
     disagrees. This is a real and noteworthy finding. Before locking
     in MISMATCH, search with multiple angles (synonyms, related file
     paths, adjacent modules) — false MISMATCH verdicts are expensive
     because they get raised in retro as "we didn't actually ship X"
     when we did. Confirm with multiple searches.

   - UNVERIFIABLE_BY_DESCRIPTION — the ticket's claims are not concrete
     enough to verify from code alone (a perf improvement with no
     measurable artifact, a UX tweak with no testable signature, a
     refactor with no single visible change). Explain what would be
     needed to verify, e.g., "would need benchmark data to confirm
     latency improvement claimed in the description."

4. FOR CONFIRMED_DONE AND PARTIAL TICKETS: gather the information a QA
   walkthrough would need. (Same shape as `plan-project-tests` —
   reusing that pattern intentionally.)

   a. User-visible surface — where does this change appear from the
      user's perspective? URL, button, CLI command, API endpoint,
      background job, etc. Cite path:line where the surface is defined.

   b. Entry point — path:line where execution begins for this feature.

   c. Preconditions — what state does the system need before the feature
      can be exercised? User account, permissions, feature flags, seeded
      data, env vars, third-party setup. Look at the code's guards,
      middleware, env checks, required inputs.

   d. Inputs that exercise different paths — happy-path input, edge-case
      or boundary input, and an invalid/error-path input. Drawn from
      what the code actually checks, not invented.

   e. Observable side effects — what should the user see, or what
      should change in the system, after exercising the feature? UI
      text, response shape, database rows, log lines, sent emails,
      queued jobs, file outputs. Cite where each side effect is
      produced.

   f. Known limitations or partial gaps — for PARTIAL tickets, what's
      explicitly missing that QA should NOT flag as a bug.

   g. Visible dependencies on other tickets — only if you can directly
      see them in the code (an import, a foreign-key reference, a shared
      module, an explicit comment). Don't speculate.

REPORT BACK with one block per ticket, in the order given:

  TICKET <ID>: <title>
  VERDICT: <CONFIRMED_DONE | PARTIAL | MISMATCH | UNVERIFIABLE_BY_DESCRIPTION>

  IMPLEMENTATION STATUS:
  <one-line summary of what's in the repo for this ticket>

  EVIDENCE:
  <bulleted path:line references that support the verdict; empty for
   MISMATCH (which means no evidence found) and UNVERIFIABLE_BY_DESCRIPTION>

  MISSING (for PARTIAL only):
  <bulleted list of claims not found in the repo>

  MISMATCH DETAIL (for MISMATCH only):
  <bulleted list of claims that were searched for and not found, with
   the search angles you tried — this helps the team distinguish "we
   actually didn't ship this" from "the ticket was about something
   subtle the audit missed">

  TEST INFO (for CONFIRMED_DONE and PARTIAL only):
    user-visible surface: <what + path:line>
    entry point: <path:line>
    preconditions: <list>
    inputs to exercise:
      - happy path: <description of input>
      - edge case: <description of input>
      - error path: <description of input>
    observable side effects:
      - <effect> at <path:line where produced>
    known limitations (PARTIAL only): <what QA should not flag as a bug>
    visible dependencies on other tickets: <list, or "none">

Cite real paths only. If a path you'd expect doesn't exist, say so
explicitly — don't invent.

Keep each ticket's block under ~300 words for CONFIRMED_DONE / PARTIAL,
under ~120 words for MISMATCH / UNVERIFIABLE_BY_DESCRIPTION.

DO NOT modify anything. DO NOT propose code or fixes. DO NOT write the
QA walkthrough — the main thread will do that from the TEST INFO you
gather.
```

### After subagents return

Collect all per-ticket blocks across all batches.

If any subagent locked in MISMATCH on what looks like a single weak signal, spawn a focused follow-up subagent for that specific ticket with a narrower prompt before treating the verdict as final. A false MISMATCH is one of the most painful errors in this skill — it surfaces in retro as "we didn't ship X" when we actually did, and the team chases a phantom problem.

For cross-cutting tickets investigated in multiple repos: merge their per-repo findings. The strictest verdict wins (any MISMATCH → overall MISMATCH; any PARTIAL → overall PARTIAL if no MISMATCH). Union the EVIDENCE, MISSING, and TEST INFO across repos.

Partition the results into four groups for the postmortem structure:
- **CONFIRMED_DONE** → audit summary entry + QA walkthrough in the testing plan.
- **PARTIAL** → audit summary entry (with partial note) + QA walkthrough with partial-gap caveat.
- **MISMATCH** → audit summary entry; NO QA walkthrough (nothing to walk through); highlighted in the retro section.
- **UNVERIFIABLE_BY_DESCRIPTION** → audit summary entry; appendix with "needs Linear-side clarification" framing; NO QA walkthrough.

---

## Phase 5 — Synthesize the postmortem (main thread)

Combine subagent reports and the Phase 2 parse into a single document with two main sections: the **audit summary** (retro-style) and the **testing plan** (QA-style).

### Audit-side synthesis

Compute summary metrics:

- Total tickets in the sprint
- Counts by team-claimed status (DONE / IN_FLIGHT / NOT_STARTED / DROPPED)
- For DONE tickets:
  - Count by audit verdict (CONFIRMED_DONE / PARTIAL / MISMATCH / UNVERIFIABLE_BY_DESCRIPTION)
  - The team's done-claim rate (claimed-DONE count)
  - The actual-shipped rate (CONFIRMED_DONE + PARTIAL count, as a fraction of claimed-DONE)
- By repo (multi-repo): how the audit broke down per repo
- By assignee (if metadata available): how each person's claimed-DONE work audited

Build the retro-focused parts:

- **What confirmed-shipped** — the bulk; quick list.
- **What landed partial** — surfaced as a list with what's missing per ticket.
- **What's a mismatch** — the headline retro finding; framed factually, not accusatorially. Each item: "Team marked DONE; audit found no visible implementation. Search angles tried: ..."
- **What couldn't be audited from code** — UNVERIFIABLE_BY_DESCRIPTION tickets. Framed as a ticket-quality issue, not an implementation issue.

### Testing-plan-side synthesis

For each CONFIRMED_DONE and PARTIAL ticket, write a QA walkthrough using the TEST INFO from the subagent. Same structure as `plan-project-tests`'s walkthroughs:

- Preconditions list
- 3+ test cases per ticket: happy path, edge case, error path
- Each test case: explicit steps, expected result, pass/fail criteria
- For PARTIAL tickets, a "known partial gaps" note so QA doesn't flag the gaps as new bugs

### Inter-ticket dependencies and test ordering

From the subagent's TEST INFO `visible dependencies on other tickets`, construct a suggested test ordering (same logic as `plan-project-tests`):

- Tickets without in-sprint dependencies first.
- Dependent tickets after the ones they depend on.
- Cycles surfaced, not resolved.
- Tickets in the same area grouped where possible for tester ergonomics.

### Don't editorialize

The postmortem reports facts. It does NOT:
- Recommend who should do what next.
- Suggest sprint-process changes.
- Speculate about *why* something is a mismatch (was it cancelled mid-sprint? scope-changed? marked done by mistake?).
- Propose fixes for the mismatches.

If something looks like a process problem, the analysis surfaces the symptom; the retrospective meeting decides what to do.

---

## Phase 6 — Save and report

### Postmortem file structure

Save to `.claude/sprint-postmortems/<slug>.md`. Use this structure:

```markdown
# Sprint postmortem: <slug>

> Audited against:
>   - `<repo-label-1>` at `<path>` (branch `<branch>` @ `<short-sha>`)
>   - `<repo-label-2>` at `<path>` (branch `<branch>` @ `<short-sha>`)
> Audit date: <ISO date>
> Source: <"pasted" or path to the CSV>
> Total tickets in sprint: <N>

## Headline

<2-4 sentences. Examples:
"12 of 12 tickets marked Done confirmed shipped. Testing plan covers all
12, organized in dependency order."
or
"10 of 12 tickets marked Done confirmed shipped. 2 mismatches need
retro attention (see below). Testing plan covers the 10 shipped.">

## Sprint composition (team-claimed)

| Status (team's claim) | Count |
|---|---|
| Done | <n> |
| In flight | <n> |
| Not started | <n> |
| Dropped | <n> |

## Audit of "Done" tickets

| Audit verdict | Count | % of claimed-Done |
|---|---|---|
| Confirmed Done | <n> | <%> |
| Partial | <n> | <%> |
| Mismatch | <n> | <%> |
| Unverifiable by description | <n> | <%> |

<By repo, if multi-repo:>
| Repo | Done claimed | Confirmed | Partial | Mismatch | Unverifiable |
|---|---|---|---|---|---|
| <label> | <n> | <n> | <n> | <n> | <n> |

<By assignee, if metadata available:>
| Assignee | Done claimed | Confirmed | Partial | Mismatch |
|---|---|---|---|---|
| <name> | <n> | <n> | <n> | <n> |

---

## Notable findings for retro

### Mismatches (team marked Done; audit found no visible implementation)

<For each MISMATCH ticket. Omit section if none.>

- **<TICKET-ID>: <Title>** — assigned to <assignee>, repo `<label>`
  - Audit claims searched for: <list>
  - Search angles tried: <list>
  - One-line factual framing of what's missing: <"the new admin permission check
    described in the ticket isn't present at any of the searched paths">

### Partials (Done-claimed but with notable missing pieces)

<For each PARTIAL ticket. Omit section if none.>

- **<TICKET-ID>: <Title>** — assigned to <assignee>, repo `<label>`
  - Found: <one-line summary>
  - Missing: <bulleted list of specific gaps>

### Tickets that can't be audited from code alone

<UNVERIFIABLE_BY_DESCRIPTION tickets. Omit section if none.>

- **<TICKET-ID>: <Title>** — <one-line reason it couldn't be verified;
  what would be needed to confirm shipped state (e.g., "needs benchmark
  output", "needs visual screenshot of refreshed UX")>

---

## Testing plan for what shipped

This section covers everything CONFIRMED_DONE and PARTIAL — i.e., tickets
the audit confirmed at least partially shipped. Tickets are ordered to
respect visible dependencies between them.

### Project-wide setup

<Common preconditions across many tickets, consolidated to reduce
repetition. Derived from the union of per-ticket preconditions. Omit
section if nothing project-wide.>

### Suggested test order

<Numbered list, dependency-respecting:>

  1. <TICKET-ID>: <Title> — <one-line reason for this position>
  2. ...

<Note any dependency cycles or long chains.>

### Test cases per ticket

#### <TICKET-ID>: <Title>

**Status:** Confirmed Done <or> Partial (see "Known partial gaps" below)
**Repo:** <label>
**Assignee:** <name>

**Developer reference:**
- User-visible surface: <what + `path:line`>
- Entry point: `<path:line>`
- Key side-effect locations: <path:line for each, one line each>
- Known partial gaps (PARTIAL only): <bullet list of what's explicitly
  not implemented; omit subsection for CONFIRMED_DONE>

**Preconditions:**
- <Specific actionable items>

**Test case 1 — Happy path: <short title>**

1. <Explicit step>
2. <Next step>

**Expected result:** <Specific, observable outcome>
**Pass / fail:** <One-line criteria>

**Test case 2 — Edge case: <short title>**

<Same structure>

**Test case 3 — Error path: <short title>**

<Same structure — at least one negative test per ticket>

**Notes:** <Any caveats; omit if none>

---

<repeat per ticket in test-order>

---

## Caveats

- This postmortem reflects each repo at the commit listed in the header.
  Work on other branches, unmerged PRs, or stashes is not visible.
- The audit treats team-marked "Done" as a reasonable claim and confirms
  the bulk. Mismatches are surfaced as findings, not accusations; they
  may have legitimate explanations (e.g., the ticket was reverted, scope
  changed but Linear wasn't updated, or the implementation lives in a
  repo not included in this audit).
- The skill is **read-only**. No code, repo, or external state was
  modified.
- Dependencies between tickets are derived only from code-visible signals
  (imports, foreign keys, shared modules). Implicit dependencies
  discussed in meetings are not captured.
- Linear status was the audit's anchor. If the export captured stale
  status values, the audit may treat tickets as Done that the team has
  since unmarked.
- The skill does NOT propose fixes for mismatches, recommend process
  changes, or assign next steps. Those are retro decisions.
```

### Report to the user

After saving, summarize in chat:

- File path: `.claude/sprint-postmortems/<slug>.md`.
- Headline result: e.g., "12 of 14 Done tickets confirmed shipped (1 partial, 1 mismatch). Testing plan covers 13 of 14."
- The repos and commits the audit reflects.
- The mismatch headlines, if any (this is what the team most wants to see at a glance).
- Total test cases generated (sum across all testing-plan entries).
- A reminder that no changes were made to any repo.

---

## Rules

- **Read-only across all repos and git.** No commits, branches, edits, pushes, stashes, rebases — in any repo. File reads and read-only git inspection (`status`, `diff`, `log`, `branch`) are fine.
- **Cite real paths only.** Every `path:line` in the postmortem must point to a file that actually exists at the audited commit. No fabricated paths.
- **Done is the default truth.** The audit confirms the team's done-claim; it doesn't assume the team is wrong. Mismatches are noteworthy exceptions, not the expected case.
- **Don't manufacture mismatches.** Before locking in MISMATCH, the subagent must confirm with multiple search angles. False MISMATCH verdicts cause retro damage.
- **A test case without a concrete expected result is not a test case.** "It should work" is a smell — replace with an observable, specific outcome.
- **Every CONFIRMED_DONE and PARTIAL ticket gets at least one negative test.** Happy-path-only walkthroughs are incomplete.
- **MISMATCH and UNVERIFIABLE_BY_DESCRIPTION tickets do NOT get walkthroughs.** Nothing testable.
- **Don't editorialize.** The postmortem reports state. It does NOT recommend process changes, assign blame, speculate about causes of mismatches, or propose fixes. Surface the findings; let the team decide.
- **Confirm CSV parse and routing before auditing.** Phase 2 and Phase 3 confirmations are not optional. Wrong column mapping or wrong routing silently corrupts every downstream verdict.
- **The postmortem is a snapshot.** Stamped with each repo's branch and commit so it's reproducible. Re-run produces a fresh document.
- **No fix proposals.** Mismatches that the team wants to act on go through `investigate-request` (if they need scoping) or `make-plan` (if the path forward is clear), not back into this document.
