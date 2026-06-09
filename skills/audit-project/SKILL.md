---
name: audit-project
description: Audit a whole Linear project against a single repository to determine which tickets are actually implemented, partially implemented, not done, or unverifiable from code alone. The user pastes a Linear project export (ticket titles + descriptions) into a markdown file; the skill checks each ticket against the repo on the user's current branch and produces an audit report at `.claude/audits/<project>.md`. Trigger on phrases like "audit this project", "check what's actually done", "compare these tickets to the repo", "post-mortem on this project", "which tickets shipped", or when the user points at a Linear export and asks what state the project is really in. Delegates the per-ticket evidence-gathering to batched subagents (Task tool) so dozens of tickets can be audited in one run without blowing the main thread's context. Does not modify the repo or git history in any way.
---

# Audit Project

Take a Linear project export (titles + descriptions for many tickets) and produce a post-mortem audit telling the user, for each ticket: is this actually done in the repo? Partially? Not at all? Or impossible to tell from code alone?

The skill is **read-only** against the repo and git. It inspects working-tree and history; it does not commit, push, branch, edit code, or alter anything.

The skill has five phases: **prepare**, **parse the ticket list**, **investigate in batches** (via parallel subagents), **synthesize the audit**, then **save and report**.

---

## Phase 1 — Prepare

1. **Confirm repo access.** If no local repository is accessible, stop and ask for repo access before proceeding. Don't fabricate evidence against a hypothetical codebase.

2. **Get the Linear export.** If the user hasn't pointed at one, ask for either:
   - A path to a markdown file containing the project export, or
   - Pasted content directly in the message.

   The expected format is loose — Linear exports vary — but typically: each ticket has a title (often with an identifier like `PROJ-123`) and a description. Common shapes the skill should handle:
   - Markdown headers per ticket (`## PROJ-123: Title` followed by the description)
   - A flat list with separators between tickets
   - A copy-paste from Linear's "Copy as Markdown" feature

3. **Note the audit context.**
   - Current branch (`git branch --show-current`). Use this branch's state as the source of truth for the audit. Note it in the final report header.
   - Current commit (`git rev-parse HEAD`). Include the short SHA in the audit so the report is reproducible against a known repo state.
   - Project name. Derive from the input file name if it has one, or ask the user for a short slug to use in the output filename. Example: input `linear-q4-search-revamp.md` → slug `q4-search-revamp`.

4. **Set up the output path.** The audit will save to `.claude/audits/<slug>.md`. Create `.claude/audits/` if it doesn't exist.

5. **If `.claude/audits/<slug>.md` already exists**, ask whether to:
   - Overwrite (the previous audit was stale)
   - Save under a versioned name like `<slug>-2.md`
   - Stop and let the user decide what they want

   Don't silently overwrite — the previous audit may have notes the user wants to keep.

---

## Phase 2 — Parse the ticket list

The main thread does this — it's a small structural pass, not investigation.

Read the Linear export and produce a structured list of tickets. For each ticket extract:

- **ID** (e.g., `PROJ-123`) — if present. If absent, generate a sequential `T-1`, `T-2`, etc. and note in the audit that IDs were synthesized.
- **Title** — short, one line.
- **Description** — everything else for that ticket. Keep this verbatim; it's what the subagent will reason against.

If the parser can't cleanly separate tickets (the format is ambiguous), surface the issue to the user with an example of the confusion and ask how to interpret it. Don't guess — a wrong parse propagates into every downstream check.

Count the tickets. If there are fewer than 5, skip the batching in Phase 3 and use a single subagent. If there are 5 or more, batch them.

---

## Phase 3 — Investigate in batches (delegate to parallel subagents)

This is where the skill spends most of its time. Each ticket needs the repo checked for evidence; the main thread cannot do all of it without saturating its context. Batched subagents are the right tool.

### Batching

Group tickets into batches of **5–10 tickets each**. Aim for **3–6 batches total** running in parallel. If the project has 60 tickets, ~6 batches of 10 each is reasonable. If it has 12 tickets, 2–3 batches of 4–6 each.

Spawn all batches in a single message so they run concurrently. Beyond ~6 parallel subagents the synthesis cost starts to outweigh the parallelism benefit; if you'd exceed that, increase the batch size instead of the batch count.

Each subagent handles its tickets sequentially within its own context. This is fine — subagent context is fresh per batch, so 10 tickets per subagent is well within budget.

### Subagent prompt template

```
You are auditing a set of Linear tickets against a real codebase to determine,
for each ticket, whether it is implemented. You are NOT writing code, opening
PRs, or modifying anything. You are inspecting only.

TICKETS TO AUDIT (in this batch):

<for each ticket in the batch, paste:
  ### TICKET <ID>: <title>
  <description>
  ---
>

YOUR JOB:

For each ticket above, in order, do the following:

1. Extract the concrete claims from the description — what specific behavior,
   files, endpoints, schema, UI, or capabilities does this ticket call for?
   If the ticket is too vague to extract concrete claims (e.g., "improve
   onboarding" with no detail), say so and assign the verdict
   `CANNOT_DETERMINE` — don't guess.

2. Look for evidence in the repo. Use grep/find/glob to locate files, modules,
   functions, schema entries, config keys, UI strings, route handlers, or
   tests that correspond to the ticket's claims. Read the files to confirm.

3. Assign ONE of four verdicts:

   - DONE — clear evidence that all of the ticket's concrete claims are
     implemented. Cite at least one `path:line` per major claim.
   - PARTIAL — evidence that some of the claims are implemented but not all.
     Cite what was found AND list what's missing.
   - NOT_DONE — no evidence of implementation in the repo for any of the
     ticket's claims. Confirm with multiple search angles (don't conclude
     NOT_DONE from a single failed grep — try synonyms, related file paths,
     and adjacent modules first).
   - CANNOT_DETERMINE — the ticket's claims are not verifiable from code
     alone (e.g., performance improvements without measurable artifacts in
     the repo, UX polish, refactors that don't have a single visible
     signature, or any ticket so vague no concrete claim can be extracted).
     Explain what would be needed to verify it (e.g., "would need
     benchmark output", "would need to see the UX in a running app").

REPORT BACK with one block per ticket, in the same order they were given:

  TICKET <ID>: <title>
  VERDICT: <DONE | PARTIAL | NOT_DONE | CANNOT_DETERMINE>
  EVIDENCE: <bulleted list of path:line references that support the verdict;
             empty for NOT_DONE>
  MISSING: <bulleted list of concrete claims not found in the repo; empty
            for DONE>
  NOTES: <any caveats — alternate implementations, suspected partial work
          in other branches, ambiguous evidence>

Cite real paths only. If you cannot find a file you'd expect, say so explicitly
rather than inventing a path. Keep each ticket's block under ~150 words.

DO NOT modify anything. DO NOT propose code. DO NOT recommend next steps —
the main thread will synthesize those.
```

### After the subagents return

Collect all per-ticket blocks across all batches into a single list, preserving original ticket order from the Linear export.

If any subagent flagged a ticket as `CANNOT_DETERMINE` because the description was too vague, that's a real finding — keep it as-is. Don't try to "rescue" it by guessing.

If a subagent's report looks thin (e.g., NOT_DONE based on a single grep), spawn a focused follow-up subagent for that specific ticket with a narrower prompt before locking in the verdict. False NOT_DONE verdicts are the most expensive kind of error in this audit — they tell the user to redo work that's already shipped.

---

## Phase 4 — Synthesize the audit

The main thread does this. Combine the subagent reports into a coherent audit. Do not just concatenate — produce a coherent narrative with summary counts up top and per-ticket detail below.

### Compute summary counts

- Total tickets audited
- Count and percentage per verdict (Done / Partial / Not done / Cannot determine)
- Of the Partial tickets, how many are >50% done vs <50% done (judgment call based on the MISSING list length vs EVIDENCE list length — note this is heuristic)

### Build the "recommended next tickets" section

Look at all PARTIAL and NOT_DONE tickets and propose which ones to tackle next, with brief reasoning. Criteria for ranking:

- **PARTIAL tickets often beat NOT_DONE tickets** for next-up: less remaining work, momentum, less context to rebuild.
- **Dependency hints in the descriptions matter.** If ticket B's description references "the X feature from ticket A", and A is NOT_DONE, A should come before B.
- **CANNOT_DETERMINE tickets are not in the recommendations list.** Mention them separately as "needs clarification before it can be audited or worked on."

Cap the recommendations at the top 5–7 tickets. A long ranked list of 40 isn't useful.

---

## Phase 5 — Save and report

### Audit file structure

Save to `.claude/audits/<slug>.md` (or the versioned path from Phase 1). Use this structure:

```markdown
# Project audit: <project name or slug>

> Audited against: branch `<branch-name>` at commit `<short-sha>`
> Audited on: <ISO date>
> Source: <path to the Linear export, or "pasted">
> Total tickets: <N>

## Summary

| Verdict | Count | % |
|---|---|---|
| Done | <n> | <%> |
| Partial | <n> | <%> |
| Not done | <n> | <%> |
| Cannot determine | <n> | <%> |

<One short paragraph (2-3 sentences) summarizing the overall state of the
project. E.g.: "Roughly two-thirds of the tickets in this project are shipped.
A cluster of UI tickets remains incomplete, and several performance tickets
cannot be verified from the codebase alone.">

## Recommended next tickets

<Top 5-7 PARTIAL and NOT_DONE tickets, ranked, each with a one-line reason.
Format:
  1. **<TICKET-ID>: <title>** — <reason this is a good next pick>
Where reasons reference: how much work remains (for Partial), whether
dependencies are met, whether other recommended tickets depend on this one.>

### Tickets needing clarification before they can be picked up

<Any CANNOT_DETERMINE tickets that the user should resolve in Linear before
working on or re-auditing. One line each. Omit if none.>

---

## Per-ticket findings

### Done (<n> tickets)

#### <TICKET-ID>: <title>

**Evidence:**
- `path:line` — <one-line description of what this file/line shows>
- `path:line` — ...

**Notes:** <any caveats; omit if none>

---

#### <next done ticket>
...

### Partial (<n> tickets)

#### <TICKET-ID>: <title>

**Evidence found:**
- `path:line` — ...

**Missing:**
- <concrete claim from the ticket that isn't in the repo>
- ...

**Notes:** <any caveats; omit if none>

---

#### <next partial ticket>
...

### Not done (<n> tickets)

#### <TICKET-ID>: <title>

**Missing:**
- <concrete claim from the ticket that isn't in the repo>
- ...

**Notes:** <any caveats — e.g., "searched for X, Y, and Z; nothing found";
omit if none>

---

#### <next not-done ticket>
...

### Cannot determine (<n> tickets)

#### <TICKET-ID>: <title>

**Why unverifiable:** <one or two sentences>

**To verify, you would need:** <what artifact, measurement, or running-app
observation would settle the question>

---

#### <next cannot-determine ticket>
...

---

## Caveats about this audit

- This audit reflects the state of branch `<branch-name>` at commit
  `<short-sha>`. Work on other branches, in stash, or unmerged is not visible.
- The audit relies on Linear ticket descriptions being concrete enough to map
  to code. Tickets that describe outcomes without describing implementation
  (performance, UX, refactors) often land in CANNOT_DETERMINE — that is a
  property of the ticket, not the code.
- Linear status was not used as an input. The audit reflects what the repo
  shows, regardless of what Linear says.
- <Any other repo-specific caveats surfaced by the subagents, e.g.,
  "search was limited to top-level src/; if any tickets refer to vendored
  code under third_party/, those were not deeply checked".>
```

### Report to the user

After saving, summarize in chat:

- File path: `.claude/audits/<slug>.md`
- One-line headline: e.g., "Audited 47 tickets — 28 done, 9 partial, 7 not done, 3 cannot determine."
- The branch and commit the audit reflects.
- Anything surprising the user should look at first (e.g., "Two tickets marked Partial appear to overlap with three tickets marked Not done — likely scope drift; first three recommendations address this.").
- A reminder that the skill made no changes to the repo or git history.

---

## Rules

- **Read-only against the repo.** No commits, branches, edits, pushes, stashes, rebases. `git status`, `git diff`, `git log`, and file reads are fine.
- **Cite real paths only.** Every `path:line` in the audit must point to a file that actually exists at the audited commit. No fabricated paths anywhere.
- **CANNOT_DETERMINE is a respectable verdict.** Don't downgrade vague tickets into false DONE / NOT_DONE verdicts to make the audit look more decisive. Loud uncertainty beats fake certainty.
- **Don't tell the user a ticket is NOT_DONE based on a single grep.** Confirm with multiple angles before assigning that verdict.
- **The audit is a snapshot.** It is stamped with the branch and commit so the user can re-run and diff later. Don't pretend it's a living document.
- **No recommendations beyond the audit.** Don't propose how to implement the missing tickets — that's `make-plan`'s job. This skill audits; it does not plan.