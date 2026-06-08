---
name: investigate-bug
description: Investigate a bug report against a repository and produce a written findings document — without fixing anything. Use when the user has a bug to diagnose (pasted report, stack trace, failing test output, Linear/Jira ticket, Sentry output, screenshot of an error, or a free-form description) and wants to understand where the bug lives, what's actually going wrong, and why. The skill reads code, forms a hypothesis grounded in `path:line` evidence, and saves a markdown findings document to `.claude/investigations/<slug>.md`. Trigger on phrases like "investigate this bug", "what's going wrong here", "find the cause of this", "diagnose this error", "where does this bug live", "look into this Sentry error", or when the user pastes a bug-shaped artifact and asks for a diagnosis. Delegates code-reading to subagents (Task tool) so the main thread's diagnostic context stays clean. STRICT SEPARATION: this skill investigates only — it does not fix, modify, or commit anything. Fixes go through make-plan → review-plan → implement.
---

# Investigate Bug

Take a bug report — in whatever shape the user pastes — and produce a written investigation that explains, with evidence, where the bug lives and what is most likely going wrong. **The skill does not fix anything.** It reads code, forms a hypothesis, and writes a findings document. Fixing is a separate downstream skill.

The skill is **read-only** against the repo and git. It inspects working-tree and history; it does not commit, push, branch, edit code, or alter anything.

The skill has five phases: **prepare**, **characterize the bug**, **investigate** (via subagents), **synthesize the findings**, then **save and report**.

---

## Phase 1 — Prepare

1. **Confirm repo access.** If no local repository is accessible, stop and ask for repo access before proceeding. An investigation against a hypothetical codebase is worse than no investigation — fabricated evidence reads like real evidence.

2. **Get the bug report.** Accept any of:
   - A pasted description
   - A stack trace or error output
   - A failing test name and its output
   - Sentry / error-tracker output
   - A Linear / Jira / GitHub issue ID *and* its description (don't try to fetch; ask the user to paste the content)
   - A screenshot of an error (if the user provides one — describe what you see and proceed)

   If the report is too vague to act on (e.g., "something is broken on the homepage"), ask the user for at least one concrete observation before continuing: an exact error message, a reproduction step, a URL, a user ID, anything that anchors the investigation in a specific signal.

3. **Note the investigation context.**
   - Current branch (`git branch --show-current`). The investigation reflects this branch's state.
   - Current commit (`git rev-parse HEAD`). Include the short SHA in the findings so the document is reproducible.
   - A short **slug** for the bug. Derive from the bug title or error message, kebab-cased, e.g. `csv-export-empty-response`, `login-redirect-loop`. Ask the user once if ambiguous.

4. **Set up the output path.** Findings will save to `.claude/investigations/<slug>.md`. Create `.claude/investigations/` if it doesn't exist.

5. **If `.claude/investigations/<slug>.md` already exists**, ask whether to overwrite, save under a versioned name like `<slug>-2.md`, or stop. Don't silently overwrite — prior investigations may contain notes the user wants to keep.

---

## Phase 2 — Characterize the bug (main thread)

Before delegating to subagents, the main thread does a quick structural pass on the bug report. This is fast and unblocks the subagents' work.

Extract:

- **Observable symptom** — what the user / system reports seeing. The thing that's wrong from the outside.
- **Trigger** — what action or condition causes the symptom. May be empty if the report doesn't say.
- **Surface area** — what part of the system the symptom appears in. UI page, API endpoint, background job, CLI command, build step, deploy pipeline, etc. May be inferable from a stack trace or error path.
- **Anchors** — concrete strings the investigation can grep for: function names from a stack trace, error messages, file paths in the trace, request paths, log lines, exception class names, ticket IDs in error metadata. These are the entry points into the repo. **If there are no anchors, the investigation is going to be hard** — note that and ask the user for any additional detail they can produce.
- **Known-good and known-bad observations**, if any. "It worked yesterday" is a known-good. "Only fails for users in EU" is a partition. "Started after the v2 deploy" is a temporal anchor.

If the bug report includes a stack trace, list the frames the investigation should start from (typically the deepest application-code frame, not the framework frames). If the report includes a failing test, note the test file and the assertion that failed.

If the report is too thin to characterize meaningfully — only one or two of the above slots are filled — stop and tell the user what's missing. Don't proceed with weak inputs; the subagents will just hallucinate evidence to fill the gap.

---

## Phase 3 — Investigate (delegate to subagents)

Spawn one or two subagents in parallel, depending on the bug's shape.

### Single subagent (most bugs)

A single bug with a clear surface area gets one investigation subagent. This covers the typical case: "the X endpoint returns 500", "the Y page renders blank", "the Z test fails".

### Two subagents in parallel (cross-cutting bugs)

A bug that visibly spans two distinct areas of the system gets two subagents — one per area. Examples:
- Frontend symptom that may be a backend issue → one subagent on the frontend code path, one on the backend code path.
- Bug that happens in production but not locally → one on the application code, one on environment/config/infrastructure differences.
- Bug introduced by a recent dependency upgrade → one on the application code, one on the dependency's changes (via package manifest and lockfile).

Don't spawn more than two. The investigation is convergent work (forming one hypothesis), not divergent (covering many areas). Beyond two, synthesis cost dominates.

### Investigation subagent prompt template

```
You are investigating a bug in a codebase. Your job is to gather evidence and
form a hypothesis. You are NOT fixing, modifying, or proposing code changes.
You are reading and reporting.

BUG REPORT (raw, as provided by the user):

<paste the full bug report>

CHARACTERIZATION (from the main thread):

- Observable symptom: <symptom>
- Trigger: <trigger or "unknown">
- Surface area: <area>
- Anchors: <list of grep-able strings>
- Known-good / known-bad: <observations or "none">

YOUR FOCUS:

<For single-subagent runs: "the bug as a whole".
For two-subagent runs: the specific area this subagent owns,
e.g. "the backend code path from the failing endpoint inward",
or "environment and config differences between prod and local".>

YOUR JOB:

1. Locate the relevant code. Use the anchors to grep/find the entry points,
   then read from there. Read the call sites, the called functions, the data
   they touch, the error paths. Cite `path:line` for every file you actually
   open.

2. Reason about WHAT is going wrong. State explicitly:
   - The expected behavior at each relevant point (what the code clearly
     intends).
   - The observed behavior implied by the bug report.
   - Where the divergence appears to be introduced — i.e., which specific
     `path:line` is the most likely site of the bug, or the most likely
     site where the wrong value first enters the system.

3. Reason about WHY it's going wrong. Pick the strongest hypothesis you can
   support with evidence from the code. Common categories — adapt to the bug:
     - Null / undefined value not handled
     - Race condition / ordering assumption
     - State that's stale (cache, memoized value, untracked dependency)
     - Boundary condition (empty input, off-by-one, timezone, very large value)
     - Wrong assumption about external behavior (API contract, library
       semantics, OS behavior)
     - Configuration mismatch between environments
     - Recent change that broke a prior assumption (look at git log / blame
       on the suspect lines)
     - Dead code path being reached because some upstream guard was removed
     - Type confusion (string vs number, string vs enum, etc.)

4. If you cannot form a single strongest hypothesis, list the top 2-3
   competing hypotheses with the evidence for and against each. Do NOT
   collapse them into a vague "it could be one of several things" — make
   each one concrete and falsifiable.

5. Identify what evidence the user could gather to confirm or refute the
   hypothesis. Examples:
     - "Run the function with input X and check whether the returned Y is
       null — if yes, the hypothesis is confirmed."
     - "Check the production logs for the line at `path:line` around the
       failure time."
     - "Check whether the env var FOO is set in the failing environment."
   This is NOT a fix proposal. It's a list of cheap checks that would tell
   the user whether you're right.

6. Note anything you investigated but ruled out, briefly — so the user
   knows what's already been checked and doesn't redo it.

7. Note anything you could NOT investigate from code alone (e.g., needs
   runtime logs, needs to see the failing request, needs production database
   access). Be explicit.

REPORT BACK with:

- **Code path traced**: ordered list of `path:line` references from the entry
  point inward, with one-line notes on what each does.
- **Most likely site of the bug**: a single `path:line` if you have one
  strongest hypothesis; otherwise the 2-3 candidate sites.
- **Hypothesis**: the strongest explanation, in 2-4 sentences, grounded in
  the code path you traced.
- **Competing hypotheses** (if any): each in 1-2 sentences, with evidence
  for and against.
- **Confirmation steps**: cheap checks the user can run to confirm or refute.
- **Ruled out**: brief list of things investigated but dismissed, with why.
- **Could not investigate**: what would require runtime / production access.

DO NOT propose a fix. DO NOT write or modify code. Cite real paths only.
If a path you'd expect to find doesn't exist, say so explicitly rather
than inventing one.

Keep the report under ~800 words.
```

### After the subagents return

If a subagent's hypothesis looks thin (single weak signal, lots of "this might be it" language) — spawn a focused follow-up subagent with a narrower prompt targeting the weakest spot. A weak hypothesis presented confidently is worse than no hypothesis.

If two parallel subagents disagree (one says the bug is in the backend, the other says it's in the frontend), don't paper over the disagreement. Surface it explicitly in the synthesis as competing hypotheses, with the evidence each subagent provided.

---

## Phase 4 — Synthesize the findings

The main thread does this. Combine the subagent report(s) with the characterization from Phase 2 into a coherent findings document.

### What synthesis means here

- **One strongest hypothesis up front, if the evidence supports it.** Bury competing hypotheses below.
- **If the evidence does NOT support a single strongest hypothesis**, lead with the competing hypotheses, ranked by likelihood, each with concrete evidence. Do not invent confidence the evidence doesn't support.
- **Confirmation steps come from the subagent and are reproduced verbatim** (or near-verbatim). These are the user's next move; they're the most actionable part of the document.
- **Caveats are explicit.** Anything the investigation couldn't determine from code alone is named clearly, not hidden.

---

## Phase 5 — Save and report

### Findings file structure

Save to `.claude/investigations/<slug>.md`. Use this structure:

```markdown
# Investigation: <short bug title>

> Investigated against: branch `<branch-name>` at commit `<short-sha>`
> Investigated on: <ISO date>
> Source: <"pasted bug report" | path to bug source file | ticket ID + path>

## Bug report (as received)

<The bug report as the user pasted it, lightly reformatted for readability.
Keep this section verbatim where possible — the investigation is reasoning
ABOUT this content, so it should be present.>

## Characterization

- **Observable symptom:** <symptom>
- **Trigger:** <trigger, or "unknown from the report">
- **Surface area:** <area>
- **Anchors used:** <list of grep-able strings>
- **Known-good / known-bad observations:** <or "none provided">

## Findings

### Most likely cause

<2-4 sentences stating the strongest hypothesis, grounded in the code path
traced. Reference the most likely site with a `path:line`.

If the evidence does not support a single strongest hypothesis, write
instead: "The investigation did not converge on a single cause. See
**Competing hypotheses** below.">

### Code path traced

<Ordered list of `path:line` references from the entry point inward, each
with a one-line note on what it does. This is the map of the investigation
— it shows what was actually read.>

### Competing hypotheses

<If there's a single dominant hypothesis, omit this section.
Otherwise list 2-3 alternatives, ranked, each in 2-3 sentences with
evidence for and against. Each must be concrete and falsifiable —
no "could be one of several things".>

## Confirmation steps

<Numbered list of cheap, concrete things the user can do to confirm or
refute the hypothesis. Each one is one or two lines. Examples:
  1. Check whether `<env var>` is set in the failing environment.
  2. Run the failing input through `<function at path:line>` in isolation
     and check whether the return value is what the next line expects.
  3. Look at the logs around the failure time for the line at `<path:line>`.

These are NOT fix proposals. They are tests that would tell the user
whether the hypothesis is right.>

## Ruled out during this investigation

<Brief list of things considered but dismissed, with one-line reasons.
Saves the user from re-checking these. Omit if nothing was ruled out.>

## Could not investigate from code alone

<What would require runtime access, production logs, a real failing request,
database state, etc. Be explicit so the user knows the limits of this
investigation. Omit if nothing.>

## Caveats

- This investigation reflects the state of branch `<branch-name>` at commit
  `<short-sha>`. Work on other branches is not visible.
- The investigation is **read-only**. No code was modified.
- A fix has NOT been proposed. To fix the bug, take this findings document
  to `make-plan` and start the plan → review → implement pipeline.
- <Any other repo-specific caveats the subagents surfaced.>
```

### Report to the user

After saving, summarize in chat:

- File path: `.claude/investigations/<slug>.md`
- One-line headline: e.g., "Most likely cause: the `formatRow` function at `src/csv/export.ts:42` is called with `null` when no rows match the filter."
- Whether the investigation converged on one hypothesis or has competing ones.
- The number of confirmation steps, so the user knows how much follow-up is involved.
- A reminder that no code was modified and that fixing goes through `make-plan`.

---

## Rules

- **Read-only against the repo.** No commits, branches, edits, pushes, stashes, rebases. `git status`, `git diff`, `git log`, `git blame`, and file reads are fine.
- **Cite real paths only.** Every `path:line` in the findings must point to a file that actually exists at the investigated commit. No fabricated paths.
- **No fix proposals.** This skill investigates. Confirmation steps are tests, not patches. If the user asks the skill to fix the bug mid-investigation, point them at `make-plan` — using this findings document as the ticket content.
- **Loud uncertainty over false confidence.** If the evidence supports a single hypothesis, state it clearly. If it doesn't, list competing hypotheses with evidence — don't manufacture a verdict.
- **Don't conclude from a single signal.** A grep hit alone is not a bug location. Read the file, trace the call path, confirm the line is reachable in the bug's scenario before naming it as the most likely site.
- **The investigation is a snapshot.** Stamped with branch and commit so it's reproducible and diffable against later runs.