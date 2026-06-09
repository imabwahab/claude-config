---
name: verify-qa
description: Execute the QA handoff document produced by the implement skill (or any QA plan in the same format) and report which test cases pass, fail, or remain manual. Classifies each test case into AUTOMATABLE-BROWSER, AUTOMATABLE-TERMINAL, MANUAL, or AMBIGUOUS at runtime based on what the case requires and what tooling is available. Drives a real browser via Chrome MCP / Playwright / Puppeteer when a harness is detected; otherwise produces a scoped manual checklist for the remaining cases. Also runs the specific tests the QA doc references (not the full suite) as part of verification. Trigger on phrases like "verify the QA plan", "run the QA handoff", "execute the QA doc", "verify this branch against QA", "QA check this", or when the user points at a `.claude/qa/<stem>.md` file and asks to verify the implementation. Saves a results report to `.claude/qa-results/<stem>.md`. Read-only on the repo and on git history. If tests fail, offers to spawn `investigate-bug` on each failure as a separate downstream investigation.
---

# Verify QA

Take a QA handoff document — typically the `.claude/qa/<stem>.md` file produced by the `implement` skill — and execute it against the current branch. Classify each test case by what kind of execution it needs, run the ones the skill can run, hand off the rest as a scoped manual checklist, and produce a results report.

The skill is **read-only on the repo and on git history**. It does NOT commit, push, edit code, modify migrations, or mutate database state. It DOES run code (test commands, type-checkers, linters, HTTP requests, `SELECT` queries against a local DB, browser automation against running services) — but only in ways that produce observations, not state changes.

The skill has seven phases: **prepare**, **detect tooling**, **classify test cases**, **execute** (in parallel where possible), **run referenced tests**, **synthesize**, then **save and offer follow-ups**.

---

## Phase 1 — Prepare

1. **Confirm repo access.** If no local repository is accessible, stop and ask the user to provide repo access. Verifying QA against a hypothetical codebase is meaningless.

2. **Get the QA doc.** Require a path — the input format is `path to a QA handoff document`. Default location is `.claude/qa/<stem>.md`. If the user hasn't pointed at one, ask.

   Read the doc. Confirm it has the structure produced by `implement`:
   - Header with branch, base, ticket reference
   - Summary of the change
   - Prerequisites (environment setup, test data, env vars)
   - Test cases (numbered, each with preconditions, steps, expected result, pass/fail criteria)
   - Regression checks
   - Known limitations

   If the structure looks different (e.g., a free-form QA note rather than the structured handoff), proceed best-effort but flag in the report that the input deviated from the expected format.

3. **Note the verification context.**
   - Current branch (`git branch --show-current`). Stamp in the report.
   - Current commit (`git rev-parse HEAD`). Stamp in the report.
   - Working-tree state via `git status --porcelain`. Don't refuse on dirty tree — just record what's dirty. Flag prominently in the report.
   - The stem from the QA doc filename, for the output filename.

4. **Set up the output path.** Results save to `.claude/qa-results/<stem>.md`. Create `.claude/qa-results/` if it doesn't exist.

5. **If `.claude/qa-results/<stem>.md` already exists**, ask whether to overwrite, save under a versioned name like `<stem>-2.md`, or stop. Don't silently overwrite — previous runs may have notes.

---

## Phase 2 — Detect tooling (main thread)

The skill's execution capability depends on what's available in this environment. Detect in order:

### Browser harness

Check in order; use the first available:

1. **Chrome MCP / Claude in Chrome** — check the tool list for tools tagged as browser-control. If present, prefer for ad-hoc browser steps.
2. **Playwright** — check for `@playwright/test` or `playwright` in `package.json` dependencies, and for a `playwright.config.{js,ts,mjs}` at the repo root.
3. **Puppeteer** — check for `puppeteer` or `puppeteer-core` in `package.json` dependencies.
4. **None** — record that no browser harness is available.

### Other tooling

- **Test runner** — derive the test command from the repo (the existing `make-plan` / `implement` skills already do this; reuse the logic: look at `package.json` scripts, `Makefile`, `pyproject.toml`, etc.). Note the exact command for `test`, `test:single-file`, and `test:pattern`.
- **Type-checker** — `tsc --noEmit`, `mypy`, etc., as documented in the repo.
- **Linter** — eslint, ruff, etc. Note the check-only command.
- **Local DB** — same detection as `investigate-request`. `DATABASE_URL`, `.env`, repo config, trial probe. Note connection summary if available.
- **HTTP client** — curl is universally available; note its presence. Note any repo-local HTTP-test setup (e.g., REST Client files, Bruno collections).

### Report tooling state

Surface the detected tooling to the user before running anything:

> Tooling detected:
> - Browser harness: Playwright (config at `playwright.config.ts`)
> - Test runner: `pnpm test` (single-file: `pnpm test <path>`)
> - Type-checker: `pnpm typecheck`
> - Local DB: connected to `dialer_3` on localhost:5432
> - Working tree: clean
>
> I'll proceed with this setup. Test cases that need other tooling will be handed off as manual checks.

If the working tree is dirty or any expected tool is missing, surface that here, not later. The user might want to fix the environment before continuing.

---

## Phase 3 — Classify each test case (main thread)

Read each test case in the QA doc and assign it one of four execution modes. The main thread does this classification; subagents do the execution.

### The four modes

- **AUTOMATABLE-TERMINAL** — the case can be verified via terminal commands, HTTP requests, file inspection, or `SELECT` queries. Examples:
  - "GET /api/reports/export returns 200 with a CSV body"
  - "The new column `companies.archived_at` is nullable and indexed"
  - "Running `pnpm test src/csv/export.test.ts` produces N passing tests"
  - "The env var `EXPORT_FEATURE_FLAG` is present in `.env.example`"

- **AUTOMATABLE-BROWSER** — the case requires a browser session: clicking, typing, observing rendered UI, asserting on DOM state. Only possible if a browser harness was detected in Phase 2. Examples:
  - "Click the Export button on /reports and verify a CSV downloads"
  - "Log in as a non-admin user and confirm the Admin tab is hidden"

- **MANUAL** — the case requires human judgment that no automatable check can replace. Examples:
  - "Verify the email template renders correctly in Outlook"
  - "Confirm the animation feels smooth on a slow connection"
  - "Check that the icon contrast is acceptable against the brand background"

- **AMBIGUOUS** — the skill isn't confident which mode applies. Default to MANUAL and flag explicitly in the report, with the reason for ambiguity.

### Classification rules

For each test case, look at the steps AND the expected result together:

- If the steps describe interactions a browser harness could drive AND a harness is available → **AUTOMATABLE-BROWSER**.
- If the steps describe interactions a browser would drive but NO harness is available → **MANUAL** (a step can't be browser-automated without a harness).
- If the steps describe commands, API calls, or file/DB observations → **AUTOMATABLE-TERMINAL**.
- If the expected result hinges on visual judgment (looks right / feels right / renders correctly) → **MANUAL**, even if the steps are otherwise automatable.
- If the test mixes automatable and non-automatable parts → split or note as **MANUAL** with a partial-verification note.

### Look for terminal-equivalents

A test case that says "click the Export button and verify a CSV downloads" is browser-shaped on its face. But often the same intent can be partially verified by terminal: "is the GET /api/reports/export endpoint live and returning a CSV?". When you spot a terminal-equivalent for a browser test, **classify the test as AUTOMATABLE-BROWSER if a harness is available, OR run the terminal-equivalent as a partial verification if it isn't.** Note partial verification clearly in the report — passing the terminal check doesn't prove the button works, but failing it proves the button can't work.

### Report the classification to the user before executing

Show the classification:

```
Classified 14 test cases:
  AUTOMATABLE-TERMINAL: 6
  AUTOMATABLE-BROWSER: 5
  MANUAL: 2
  AMBIGUOUS (defaulted to MANUAL): 1

I'll execute the 11 automatable cases (and partial-verify the AMBIGUOUS one
via terminal-equivalent where possible), then produce a manual checklist
for the remaining 3.
```

Ask the user to confirm or override classifications before executing. The user may want to bump a MANUAL into AUTOMATABLE-BROWSER (or vice versa) based on context the skill doesn't have.

---

## Phase 4 — Execute test cases (delegate to parallel subagents)

Run the automatable cases. Scale subagents by case count:

- 1–3 automatable cases: one subagent.
- 4–8 cases: split by mode — one subagent for terminal cases, one for browser cases (if any).
- 9+ cases: batch into 3–4 cases per subagent, cap at ~4 parallel.

**Browser subagents and terminal subagents do different work** — keep them separated when possible, since browser subagents need a live session per batch while terminal subagents are stateless.

### Subagent prompt template (AUTOMATABLE-TERMINAL)

```
You are executing terminal-side QA test cases against the current working
tree. Your job is to run the cases and report whether each passes or fails
with concrete evidence. You are NOT fixing failures, modifying code, or
diagnosing root causes — only verifying.

REPO STATE:
  Branch: <branch>
  Commit: <short-sha>
  Working tree: <clean | dirty (N files)>

TOOLING:
  Test runner: <command>
  Local DB: <connection summary, or "none">
  <other tooling>

TEST CASES TO EXECUTE (in this batch):

<for each case in the batch, paste:
  ### TEST <number>: <title>
  Preconditions: <from QA doc>
  Steps: <from QA doc, exactly as written>
  Expected result: <from QA doc>
  Pass/fail criteria: <from QA doc>
  --->

YOUR JOB:

For each test case, in order:

1. Walk the steps exactly as written. For each step:
   - If it's a command, run it. Capture stdout, stderr, and exit code.
   - If it's an HTTP request, run it (curl or equivalent). Capture status,
     headers if relevant, and body.
   - If it's a file/path check, read the file. Capture relevant excerpts.
   - If it's a `SELECT` query, run it. Capture the result rows.
   - If it's something else terminal-shaped that you can run, run it.
2. Compare the actual observed result to the expected result in the QA doc.
3. Assign a verdict:
   - PASS — observed matches expected
   - FAIL — observed contradicts expected (with concrete evidence)
   - INDETERMINATE — couldn't actually run the step due to missing setup,
     environment issues, or test data that wasn't seeded. NOT the same as
     FAIL — INDETERMINATE means the test didn't run.
4. Capture concrete evidence for the verdict — exact command output, exact
   HTTP response, exact query result, exact file content. Truncate to the
   relevant parts but don't summarize away the substance.

CRITICAL SAFETY:
  - SELECT-only queries against the DB. No INSERT/UPDATE/DELETE/CREATE/
    DROP/ALTER/GRANT/REVOKE, no state-changing transactions.
  - No git mutations. No file edits. No `npm install` / `pip install` /
    similar that mutate the environment unless the test case explicitly
    calls for it (it usually shouldn't — env setup should already be done).
  - No long-running daemons that don't terminate. If a step requires a
    running server, check whether one is already running first; don't
    start a new one as part of verification.

REPORT BACK with one block per test case:

  TEST <number>: <title>
  VERDICT: <PASS | FAIL | INDETERMINATE>
  ACTUAL OBSERVED:
    <concrete evidence — command output, HTTP response, query result, etc.>
  EXPECTED:
    <restate from the QA doc>
  NOTES:
    <anything that helps interpret the verdict — e.g., "test seeds didn't
     include the edge case the doc references", "got a 200 but the body
     was empty, which the doc didn't specify is acceptable">

Cite real paths only. Truncate output where appropriate but preserve
substance. Keep each test case's block under ~300 words.

DO NOT propose fixes. DO NOT diagnose root causes. DO NOT modify the repo,
DB, or git state.
```

### Subagent prompt template (AUTOMATABLE-BROWSER)

```
You are executing browser-side QA test cases via <Chrome MCP | Playwright |
Puppeteer>. Your job is to drive the browser through the steps and report
whether each passes or fails with concrete evidence. You are NOT fixing
failures, modifying code, or diagnosing root causes — only verifying.

REPO STATE:
  Branch: <branch>
  Commit: <short-sha>
  App running at: <URL detected or stated>

HARNESS: <Chrome MCP | Playwright | Puppeteer>
<harness-specific configuration if relevant>

TEST CASES TO EXECUTE (in this batch):

<as in the terminal template>

YOUR JOB:

For each test case:

1. Ensure preconditions are met before starting (logged-in user, seeded
   data, feature flag state, etc.). If a precondition isn't met and you
   can't establish it from the current state, mark the test INDETERMINATE
   and explain.
2. Walk the steps exactly as written. Capture a screenshot or DOM snapshot
   after the key actions (e.g., after clicking a button, after submitting
   a form).
3. Compare observed UI / DOM / network state to the expected result.
4. Assign a verdict — PASS, FAIL, or INDETERMINATE — same definitions as
   the terminal template.
5. Capture evidence: screenshots (path or inline reference), DOM excerpts
   for assertions, network response excerpts where the case checks them,
   any console errors observed.

CRITICAL SAFETY:
  - No destructive actions in the running app. If a test step would
    require irreversible state change (sending a real email, charging a
    real card, hitting a third-party API in production mode), mark
    INDETERMINATE and explain — don't proceed.
  - No actions outside the scope of the test case. If a step says "click
    Export", click Export — don't also click adjacent UI to "see what
    happens."

REPORT BACK with one block per test case:

  TEST <number>: <title>
  VERDICT: <PASS | FAIL | INDETERMINATE>
  ACTUAL OBSERVED:
    <description of what happened, with screenshot/DOM/network references>
  EXPECTED:
    <restate from the QA doc>
  NOTES:
    <anything that helps interpret the verdict>

Keep each test case's block under ~350 words.

DO NOT propose fixes. DO NOT diagnose root causes. DO NOT modify code,
git state, or app data beyond what the test case requires.
```

### After subagents return

Collect all per-case blocks. Preserve original test ordering from the QA doc.

If a subagent returned INDETERMINATE for a case that should have been executable (e.g., "the server wasn't responding" but tooling detection said it was), spawn a focused follow-up with the same case to confirm — don't lock in INDETERMINATE on a one-shot environment hiccup.

---

## Phase 5 — Run referenced tests (main thread)

The QA doc usually references specific test files or test patterns the developer wrote alongside the change (e.g., "test/csv/export.test.ts covers the happy path"). The user asked for these to be run as part of verification — but not the full suite.

For each test file or test pattern explicitly referenced in the QA doc:

1. Run it using the test runner command from Phase 2 detection.
2. Capture pass/fail counts, durations, and any failure output.
3. Also run the repo's type-checker and linter, but only on the files changed in the current diff (`git diff --name-only <base>...HEAD`). This catches incidental breakage in changed files without the noise of repo-wide checks.

If the QA doc doesn't reference specific test files, the skill skips this phase — and notes that in the report.

If a referenced test file doesn't exist, that's a finding — mark it explicitly. The QA doc claimed a test was written that isn't there.

---

## Phase 6 — Synthesize the results (main thread)

Combine subagent results and Phase 5 outputs into a results report. Synthesis:

- One verdict per test case from the QA doc, in original order.
- Aggregate counts: total / passed / failed / indeterminate / manual / ambiguous.
- The Phase 5 results as a separate section (referenced-test results), since they're tests-of-the-implementation rather than walk-throughs of behavior.
- A clearly-scoped **manual checklist** containing every MANUAL and AMBIGUOUS case, formatted so a human can walk through them — preconditions, steps, expected result, pass/fail boxes.

---

## Phase 7 — Save and offer follow-ups

### Results file structure

Save to `.claude/qa-results/<stem>.md`. Use this structure:

```markdown
# QA verification results: <stem>

> Verified against: branch `<branch-name>` at commit `<short-sha>`
> Verified on: <ISO date>
> QA doc source: <path to the QA handoff>
> Working tree: <clean | dirty — see "Dirty working tree" section below>
> Tooling used: <browser harness, test runner, DB connection, etc.>

## Summary

| Verdict | Count |
|---|---|
| PASS | <n> |
| FAIL | <n> |
| INDETERMINATE | <n> |
| MANUAL (handed off) | <n> |
| AMBIGUOUS (handed off as MANUAL) | <n> |

<One-paragraph headline. Examples:
"11 of 14 test cases verified automatically; 8 passed, 3 failed. The
remaining 3 cases require manual walkthrough (see Manual checklist below)."
"All 14 automated cases passed. Manual checklist contains 3 cases for QA.">

## Dirty working tree

<Only if the tree was dirty at verification time. List the dirty files
from `git status --porcelain` and warn that verification reflects the
working tree, not the committed state. Omit this section if clean.>

## Automated case results

### Test <number>: <title>

**Mode:** <AUTOMATABLE-TERMINAL | AUTOMATABLE-BROWSER>

**Verdict:** <PASS | FAIL | INDETERMINATE>

**Expected:**
<from the QA doc>

**Observed:**
<concrete evidence from the subagent — output, response, screenshot
reference, query result>

**Notes:** <anything from the subagent that interprets the verdict;
omit if none>

---

<repeat per case in original order>

## Referenced tests run

<From Phase 5. If no referenced tests existed in the QA doc, write
"The QA doc did not reference specific test files." instead of this section.>

| File / pattern | Tests | Pass | Fail | Duration |
|---|---|---|---|---|
| `test/csv/export.test.ts` | 12 | 12 | 0 | 1.2s |
| `pnpm typecheck` (diff only) | — | OK | — | 8.4s |
| `pnpm lint` (diff only) | — | OK | — | 2.1s |

<If any failures, paste the failure output below the table.>

## Manual checklist

<Every MANUAL and AMBIGUOUS case, formatted for a human tester. Use the
QA doc's original numbering. Each case copied with preconditions, steps,
expected result. Add a pass/fail box at the end.>

### Test <number>: <title>

**Why this is manual:** <one-line — e.g., "requires visual judgment of
animation timing", "no browser harness available">

**Preconditions:**
<copy from QA doc>

**Steps:**
<copy from QA doc>

**Expected result:**
<copy from QA doc>

**Pass / fail:** ☐ Pass  ☐ Fail  ☐ Blocked
**Notes:**
<empty space for the tester>

---

<repeat per manual case>

## Caveats

- This verification reflects branch `<branch-name>` at commit `<short-sha>`
  at the time of execution. Results may change if the working tree, DB
  state, or running services change.
- The skill made NO modifications to the repo, git history, or persistent
  database state. All checks were read-only or non-mutating.
- <If dirty working tree:> The working tree was dirty at verification
  time. Verification reflects the uncommitted state, not the committed
  base.
- <If browser harness was unavailable:> No browser harness was available;
  browser-shaped test cases were handed off as manual.
- <Any case-specific caveats from the subagents.>
```

### Report to the user

After saving, summarize in chat:

- File path: `.claude/qa-results/<stem>.md`.
- One-line headline: e.g., "11/14 automated cases passed; 3 manual cases for QA; 0 failures." Or: "8/11 passed, 3 failed (Tests 4, 7, 9). Manual checklist has 3 cases."
- The branch and commit verified.
- A reminder that no state was modified.

### Offer to investigate failures

If any test case verdict is FAIL, ask:

> 3 tests failed (Tests 4, 7, 9). Want me to run `investigate-bug` on any
> of them? I can do one, several, or all — each runs as a separate
> investigation and produces its own findings doc in
> `.claude/investigations/`.

If the user says yes:
- For each requested failure, construct a bug report from the test case + actual observed evidence.
- Run `investigate-bug` with that report (don't try to chain the skills automatically; treat this as a handoff — the user invokes the next skill, this skill stays out of it).

If the user says no, end cleanly.

---

## Rules

- **Read-only on repo and git.** No commits, branches, edits, pushes, stashes, rebases. `git status`, `git diff`, `git log`, file reads are fine.
- **Non-mutating on the database.** Only `SELECT` queries. No INSERT/UPDATE/DELETE/DDL/state-changing transactions.
- **Non-destructive in the running app.** Browser subagents do not perform irreversible actions (real emails, real charges, real third-party calls in production mode). Mark INDETERMINATE instead.
- **Concrete evidence for every verdict.** A PASS or FAIL without observed output is not a verdict. Every result block must include what was actually seen.
- **INDETERMINATE is a respectable verdict.** A test that couldn't run for environmental reasons isn't a FAIL. Don't collapse INDETERMINATE into FAIL to make the summary look more decisive.
- **No fix proposals in the results report.** This skill verifies. To act on a failure, invoke `investigate-bug` (offered at the end).
- **The results are a snapshot.** Stamped with branch, commit, working-tree state, and timestamp. Re-running may produce different results if anything has changed.
- **Cite real paths only.** No fabricated paths in the report.
- **Don't run the full test suite.** Only the specific tests the QA doc references. Repo-wide test runs are out of scope here.
- **Don't silently overwrite previous results.** Phase 1 asks before overwriting; treat each verification as a discrete snapshot.