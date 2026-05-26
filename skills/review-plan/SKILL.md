---
name: review-plan
description: Critique an implementation plan before coding begins, then optionally revise it in place. Use when a plan has been drafted (by make-plan or by a human) and needs a second opinion before work starts. Trigger on phrases like "review this plan", "check my plan", "second opinion on this", "anything missing from this plan", "is this plan good", or when the user points at a plan markdown file (e.g., `plans/PROJ-123-foo.md`) and asks for feedback. The skill verifies the plan's claims against the actual repository via a subagent, surfaces gaps and risks, and — if the user approves — edits the plan file directly to incorporate the fixes. It does not assume any fixed architecture and infers review focus areas from the plan's content.
---

# Review Plan

Critique an implementation plan before any code is written. If the user agrees with the critique, revise the plan file in place.

The skill has four phases: **read** the plan, **verify** it against the repo (via a subagent), **critique** with a verdict, then optionally **revise** the plan file.

---

## Phase 1 — Read the plan

If the user has not pointed at a plan, ask for one. Accept:
- A path (e.g., `plans/PROJ-123-add-csv-export.md`) — read it from disk. If the path does not resolve, tell the user and ask them to confirm the path or paste the content directly.
- Pasted plan content.

Extract the plan's claims:
- **What it says will change** — the affected areas, file paths, schema changes, new routes, etc.
- **What it says about flow** — the end-to-end approach.
- **What it says about risk** — what's already flagged.
- **What it says is out of scope or open** — open questions, deferred items.

Identify the **review focus areas** — derive these from the plan's own content, not from a fixed checklist. For example:
- A plan that adds a new database table → schema correctness, migrations, indexes, constraints become focus areas.
- A plan that adds a webhook handler → auth, signature verification, idempotency, retry behaviour become focus areas.
- A plan that calls a paid third-party API → cost, rate limits, caching, failure handling become focus areas.
- A plan that's purely frontend → component reuse, state management, accessibility become focus areas.

If the plan does not touch a category, do not invent concerns about it. A pure UI tweak does not need a "migrations" section in the review.

---

## Phase 2 — Verify against the repo (delegate to a subagent)

### No repo available?

If the user provides a plan but no repository (no local path, no files, no context), skip subagent spawning entirely. Mark all verification in the Phase 3 output as **skipped — no repo provided**, and note that the user should manually confirm all cited paths and patterns before starting implementation. Proceed to Phase 3 using only your reading of the plan itself.

### Spawning the verifier

**Spawn a verifier subagent via the Task tool.** The subagent's job is to check whether the plan's claims about the repo are actually true: do the files it cites exist, do the patterns it claims to follow actually exist, are there obvious files or patterns the plan missed?

Do not read repo files directly in the main thread. Same reasoning as in `make-plan`: keep the review context clean for synthesis.

### Verifier prompt template

```
You are verifying an implementation plan against the actual repository.
Your job is to check the plan's claims, NOT to write a new plan.

PLAN:
<paste the plan content>

YOUR JOB:
1. For each file path the plan cites (`path:line` references): confirm the file exists
   and the cited code does what the plan claims.
2. For each pattern the plan says it will copy ("follows the pattern in X"): confirm
   that pattern actually exists at X.
3. For each "affected area" the plan lists: scan the relevant part of the repo and
   check whether the plan missed any obvious files. Use grep/find/glob.
4. Read repo orientation files if they exist (README.md, CLAUDE.md, AGENTS.md,
   ARCHITECTURE.md, CONTRIBUTING.md) and check whether the plan conforms to
   documented conventions.
5. Check git history briefly (`git log --oneline -20`, `git branch -a`) for
   branch/commit conventions if the plan proposes a branch name.
6. Check the plan for internal consistency:
   - Does the Approach match the Step-by-step order? (e.g., does anything appear
     out of sequence between the two sections?)
   - Do the Affected Areas cover everything the Approach references, and vice versa?
   - Are there any file paths mentioned in one section but absent from another where
     they'd be expected?

REPORT BACK with:
- **Verified claims**: list of plan claims confirmed against the repo.
- **Incorrect claims**: anything the plan got wrong — wrong path, nonexistent pattern,
  misread file. Include the actual truth.
- **Missing files/areas**: things the plan should have included but didn't, with `path:line`.
- **Convention conflicts**: places the plan diverges from documented conventions in
  CLAUDE.md / AGENTS.md / etc.
- **Internal inconsistencies**: contradictions found in step 6 above.
- **Cannot verify**: claims that require runtime knowledge or access you don't have.

DO NOT propose a new plan. DO NOT write code. Just report what's true, false, missing,
inconsistent, or unverifiable.

Keep the report under ~500 words. (The higher cap vs. make-plan's ~400 is intentional:
verification requires quoting specific incorrect vs. correct paths side by side.)
Cite real paths only.
```

### After the subagent returns

If the verifier flags major gaps in a specific area (e.g., "couldn't find any of the auth middleware the plan references"), spawn a follow-up verifier with a narrower prompt. Don't read files directly in the main thread.

---

## Phase 3 — Critique and verdict

The main thread does this. Synthesize the verifier's report with your own reading of the plan to produce the critique.

### Output structure

1. **Verdict** — one of: **approve** / **revise** / **reject**.
   - *approve* — plan is sound, ready to implement (nits are fine).
   - *revise* — plan has fixable gaps; list them and offer to apply.
   - *reject* — plan has fundamental problems (wrong approach, misread of the ticket, ignores major existing patterns) and should be rewritten before review. See the reject rules below.

2. **Must fix** — blocking issues. Things that, if not addressed, will cause the implementation to fail, ship a bug, or waste rework time. Each item: what's wrong, why it matters, and the concrete fix.

3. **Should fix** — strong recommendations. Things that won't break the implementation but will produce a worse result (worse performance, harder maintenance, missed edge cases). Each item: what + why + suggested fix.

4. **Nits** — optional polish. Naming, ordering of steps, wording. Brief.

5. **Verified by the subagent** — short list of plan claims that were confirmed, so the user can see what was checked. If verification was skipped due to no repo, say so here.

6. **Could not verify** — claims the subagent flagged as unverifiable. For each item, note whether it should be confirmed *before starting* (high-stakes: schema assumptions, external API contracts, auth behaviour) or whether it's low-stakes enough to proceed and verify during implementation.

### Reject verdict rules

When the verdict is **reject**, do not offer to revise the plan — it needs a rewrite, not a patch. Instead, include a **Correct understanding** note directly below the verdict:

> **Correct understanding**: summarise in 3–5 sentences what the plan got wrong and what the right framing is. This gives the user the corrected inputs to re-run `make-plan` with.

Then still list **Must fix** items so the user understands the full scope of what's wrong, but skip **Should fix** and **Nits** — they're not worth addressing on a plan that needs a rewrite.

### Style rules

- **Be direct.** Don't hedge. If the plan is good, say so in one line and move on.
- **Cite specifics.** Every "must fix" should reference either a line/section of the plan or a path in the repo (from the verifier's report). No vague "consider error handling" notes.
- **Don't invent categories.** If the plan doesn't touch the database, don't write a "Database" subsection. Review what's there, not what isn't relevant.
- **Distinguish your inferences from the verifier's findings.** When you say "the plan misses X", note whether that's from the verifier's report or your own reading of the plan.

---

## Phase 4 — Revise the plan (optional)

After presenting the critique, ask the user:

> Want me to apply the **must fix** items to the plan file? I can also include the **should fix** items if you'd like.

If the user agrees:

1. Read the plan file fresh (in case it changed since Phase 1).
2. Edit the plan file in place using **targeted `str_replace`-style edits** — find the exact section that needs changing and replace only that section. Do not write the whole file out from scratch; a full rewrite risks losing content and restructuring sections that don't need touching.
3. Preserve the plan's original structure and voice. Don't rewrite sections that don't need fixing.
4. After editing, summarize what changed in a short bulleted diff-style note. Cover every section touched — for example:
   - *Added*: new migration step before route changes in §5
   - *Corrected*: path in §3 from `src/api/foo.js` → `src/routes/foo.js`
   - *Added*: webhook signature verification under Risks
   - *Resolved*: Open Question #2 closed with confirmed pattern from `src/lib/auth.ts:44`
   - *Reworded*: Approach step 3 to match actual service method name `generateCSV` (was `exportCSV`)
5. Show the user the updated file path.

If the user only wants some fixes applied, ask which ones and apply only those.

If the verdict was **reject**, do not offer to revise — direct the user to re-run `make-plan` with the corrected understanding from Phase 3.

---

## Rules

- **Do not write implementation code** in the review or in the revised plan. Same rule as `make-plan`.
- **Do not pad the critique.** If there's nothing in a section, omit it. A two-line "approve" is a valid review.
- **Cite real paths only**, sourced from the verifier's report. No fabricated paths.
- **Respect the plan author.** The goal is to make the plan better, not to rewrite it in your own style.