---
name: explore-code
description: Read-only exploration of a codebase across one or more directories, producing a written exploration document with cited evidence. Use when the user wants to understand how something works, compare how something is done across codebases, or enumerate every place a pattern appears — without diagnosing a bug, verifying a ticket, or planning a change. The skill operates in one of three explicit modes — EXPLAIN ("how does X work in this codebase?"), COMPARE ("how does X work in repo A vs repo B?"), or LOOKUP ("find every place that does Y") — and produces a markdown file at `.claude/explorations/<slug>.md`. Trigger on phrases like "explore how X works", "explain this codebase", "how does X work here", "compare X between these repos", "find every place that does Y", "where is Z used", or when the user wants to learn a piece of code rather than change or fix it. Requires explicit paths from the user (it won't default to cwd). Delegates code-reading to batched subagents (Task tool). Read-only — does not modify any repo or git history.
---

# Explore Code

Take a question about code — "how does X work?", "how does X differ between A and B?", or "where is Y used?" — and produce a written exploration grounded in real file references. The skill is purely diagnostic. It reads code, builds an answer, and writes a markdown document. It does not fix, plan, or change anything.

The skill is **read-only** across all directories it inspects. It runs file reads and read-only git inspection only; no commits, no edits, no pushes, no modifications anywhere.

The skill has six phases: **prepare**, **pick a mode**, **scope and clarify**, **investigate** (via batched subagents), **synthesize**, then **save and report**.

---

## Phase 1 — Prepare

1. **Get the question.** If the user hasn't asked one explicitly, ask what they want to explore. Don't proceed on vibes — exploration without a question produces a tour, not an answer.

2. **Require explicit paths.** Do NOT default to the current working directory. Ask the user to name every directory the exploration should cover, even if it's just `.`. This prevents the skill from silently exploring only part of what the user cared about, or from reading directories the user didn't intend.

   Accept any number of paths. They can be:
   - Sub-directories of the cwd
   - Sibling repo paths (`../other-repo`)
   - Absolute paths
   - A mix

   For each path the user names, confirm it exists and is a directory before proceeding. If a path doesn't exist, stop and ask.

3. **Note the exploration context.**
   - For each directory: if it's a git repo, record its current branch (`git branch --show-current`) and current commit (`git rev-parse HEAD`). If it's not a git repo, record that too. Stamp all of this in the exploration's header so the document is reproducible.
   - A short **slug** for the exploration. Derive from the user's question, kebab-cased: `auth-flow`, `redis-vs-keydb`, `where-is-feature-flag-used`. Ask the user once if ambiguous.

4. **Set up the output path.** The exploration will save to `.claude/explorations/<slug>.md`. Create `.claude/explorations/` in the cwd (not in any of the explored directories) if it doesn't exist.

5. **If `.claude/explorations/<slug>.md` already exists**, ask whether to overwrite, save under a versioned name like `<slug>-2.md`, or stop. Don't silently overwrite — explorations often contain notes the user wants to keep.

---

## Phase 2 — Pick a mode (main thread)

Three modes. Each has a distinct shape, distinct subagent prompts, and distinct output structure. Pick exactly one before starting investigation. If the user's question is ambiguous between modes, ask.

### EXPLAIN

"How does X work in this codebase?" The user wants to understand a system, feature, flow, or module. Output is a narrative explanation grounded in cited code.

Triggers: "how does X work", "explain this", "walk me through", "what does X do", "how is X implemented", "explore the X system".

Typically one directory, sometimes more. Even with multiple directories, the question is about understanding *one thing* that may span them.

### COMPARE

"How does X differ between A and B?" The user wants to see the same thing in two (or more) places side-by-side. Output is a structured comparison.

Triggers: "compare X between A and B", "how does X work in A vs B", "what's the difference between A and B in terms of X", "is X the same in both repos".

Always two or more directories. The subject (X) is the same; the directories are the axis of comparison.

### LOOKUP

"Find every place that does Y." The user wants exhaustive enumeration of a pattern, call site, dependency, or usage. Output is a list with snippets and `path:line` references.

Triggers: "find every place that does Y", "where is Y used", "every call to Y", "list all files that import Y", "show me where the Z feature flag is referenced".

Can be one or multiple directories.

### If the user's question doesn't fit any mode

Ask. Don't try to force-fit. "I want to learn this codebase" is too vague — narrow it to "explain the X system" or "compare how X works between two areas" or "find every place that does Y".

If the user has multiple distinct questions, do them as separate explorations. Don't try to weave them into one document.

---

## Phase 3 — Scope and clarify (main thread)

Once the mode is picked, narrow scope before delegating. This is fast and prevents wasted subagent work.

For all three modes, establish:

- **The subject.** What specific thing is being explored? Be concrete — "the auth system" is fine, "the codebase" is not. If the user's question is too broad, ask them to narrow it.
- **Initial anchors.** Concrete strings the investigation can grep for as entry points: function names, file paths the user mentioned, import strings, route paths, env var names, ticket IDs, feature flag names. If the user has no anchors and no specific terms to start from, ask for at least one — exploration with no anchor at all is much weaker.
- **Depth.** Quick overview (a few hours of reading equivalent), or thorough (every related file). Default to overview unless the user asks for thorough.
- **Out of scope.** What the user explicitly doesn't want covered, if anything. Tests? Generated code? Vendored dependencies? Documentation? If they don't say, the skill includes tests and excludes generated/vendored code by default.

Mode-specific additions:

- **EXPLAIN:** identify the likely entry point(s) — the function or file where execution begins for this subject. If unclear, the subagent will figure it out, but a user-provided hint cuts the search.
- **COMPARE:** confirm the axis of comparison is the same thing in each directory. "Compare authentication" assumes both directories have authentication. If one doesn't, the comparison degenerates; surface that early.
- **LOOKUP:** confirm what counts as a "match." "Every place that uses Redis" — does that include comments mentioning Redis? Tests that mock Redis? Config files? Pin the definition with the user.

---

## Phase 4 — Investigate (delegate to batched subagents)

Scale the subagent count to the scope.

### Batching rules

- **Small scope** (one directory, narrow subject, overview depth): one subagent.
- **Medium scope** (one directory + thorough, or two directories + overview): two to three subagents in parallel.
- **Large scope** (multiple directories, thorough depth, or LOOKUP across a big codebase): three to six subagents in parallel, batched by directory or by sub-topic.

Beyond ~6 subagents the synthesis cost dominates. If the scope feels bigger than that, either narrow the question or run multiple explorations.

Spawn all subagents in a single message so they run concurrently.

### Mode-specific subagent prompts

#### EXPLAIN mode subagent

```
You are exploring a codebase to explain how something works. Your job is to
read code and report what you found. You are NOT proposing changes, fixing
bugs, or writing tests.

SUBJECT: <what's being explored, e.g. "the authentication system">
DIRECTORY: <single directory path>
ANCHORS: <grep-able starting strings from the user>
DEPTH: <overview | thorough>
OUT OF SCOPE: <list, or "nothing">

YOUR JOB:

1. Locate the subject in the codebase. Start from the anchors. Use
   grep/find/glob to widen the search if needed. Find the entry point
   — the function, file, or surface where execution begins for this
   subject.

2. Trace the code outward from the entry point. Read the relevant files.
   For each file you read, cite path:line.

3. Build a mental model of how the subject works. Specifically:
   - What's the entry point?
   - What's the flow of control or data through the system?
   - What external dependencies does it touch (DB, third-party APIs,
     file system, env vars)?
   - What configuration or environment does it depend on?
   - Where are the boundaries — what's IN this system vs adjacent?
   - Where are the surprising or non-obvious parts? (Anything where
     someone new to this code would say "huh, why is it done that way?")

4. Identify open questions — things you saw but couldn't fully understand
   from the code alone (e.g., "the X variable comes from somewhere
   outside this directory; would need to inspect that to fully explain").

REPORT BACK with:

- **Entry point**: <path:line> with one-line description.
- **High-level flow**: 4-8 sentences describing how the subject works,
  end to end. No jargon the user didn't introduce.
- **Key files and what they do**: bulleted list of path:line references,
  each with a one-line note.
- **External dependencies**: bulleted list (DB tables, third-party APIs,
  env vars, etc.) with where each is touched (path:line).
- **Non-obvious aspects**: anything surprising or worth flagging to
  someone learning this code.
- **Open questions / limits**: what you couldn't determine from this
  directory alone.

Cite real paths only. If a path you'd expect to find doesn't exist, say
so explicitly. Do NOT invent paths.

Keep under ~700 words.
```

#### COMPARE mode subagent

For COMPARE mode, spawn one subagent per directory. Each gets the EXPLAIN-mode prompt above, with the same subject. The main thread will diff their reports in Phase 5.

If the directories are very different sizes or stacks, give each subagent a tailored DEPTH and ANCHORS so the reports come back balanced — a thorough exploration of a 50-file repo and an overview of a 5000-file repo will produce mismatched documents that are hard to compare.

#### LOOKUP mode subagent

```
You are finding every occurrence of a pattern in a codebase. Your job is
exhaustive enumeration with evidence. You are NOT proposing changes or
explaining the system.

WHAT COUNTS AS A MATCH: <exact definition from the user — e.g. "any
import of the redis package", "any call to the function getUserFlag",
"any file that references the env var FOO_API_KEY">

DIRECTORY: <single directory path>
OUT OF SCOPE: <list, or "nothing">

YOUR JOB:

1. Use grep/find/glob to locate every match. Use multiple search angles
   if appropriate — for example, looking for both `import redis` and
   `from redis import`, or both `getUserFlag(` and `getUserFlag,` (a
   variable reference). State which search angles you used.

2. For each match, read enough surrounding context to confirm it really
   is a match (not a comment about something with the same name, not a
   variable that happens to share a name, etc.).

3. Group matches by category if natural groupings emerge. Examples:
   - "Direct call sites" vs "Test mocks" vs "Documentation references"
   - "Active usage" vs "Commented-out usage"
   - "Production code" vs "Scripts" vs "Tests"

4. Note any matches you decided NOT to include and why (false positives
   you ruled out, ambiguous cases, etc.).

REPORT BACK with:

- **Search angles used**: bulleted list of the patterns you actually
  searched for.
- **Total matches**: count.
- **Matches by category** (or flat list if no natural groupings):
  for each match:
    `<path:line>` — <one-line context, what's happening there>
    ```<one-line code snippet from the file>```
- **Excluded**: anything that looked like a match but you decided wasn't,
  with one-line reason.
- **Coverage caveats**: anything that might cause you to miss matches
  (e.g., "code that uses Redis via a wrapper layer wouldn't show up in
  direct imports").

Cite real paths only. Snippets must be the actual file content at the
cited line. Keep under ~800 words.
```

### After the subagents return

If a subagent's report looks thin (one-paragraph EXPLAIN, three matches found in a LOOKUP that should have dozens), spawn a focused follow-up with a wider net or a different angle. Exploration is one of the places where a thin report is most likely to mislead — the user will read it as comprehensive.

If two subagents in COMPARE mode investigated the same subject but disagree about something (one says "auth uses sessions", the other says "auth uses JWT"), that's a real finding — keep both, don't paper over.

---

## Phase 5 — Synthesize (main thread)

Combine subagent reports into the final exploration document. Synthesis differs by mode.

### EXPLAIN synthesis

Combine into a single coherent narrative. Structure:

1. Lead with a 2-4 sentence summary that answers the question at a high level.
2. Walk through the flow with the entry point first, then the major steps in order.
3. Call out external dependencies and configuration.
4. Surface non-obvious parts in a dedicated section so they're not lost in prose.
5. End with open questions.

Don't just concatenate subagent reports — merge them. If two subagents covered overlapping ground, present each fact once.

### COMPARE synthesis

Build a side-by-side. Structure:

1. Lead with a 2-4 sentence summary of the comparison — what's the same, what's different at a high level.
2. A table or parallel-section layout for each axis of comparison: entry point in A vs entry point in B; data layer in A vs data layer in B; config in A vs config in B; etc.
3. **Differences** section: explicit list of where A and B diverge, each with citations to both sides.
4. **Common elements** section: where A and B converge.
5. Open questions and caveats.

If one directory simply doesn't have the subject (e.g., "compare auth between A and B" but B has no auth code), say so loudly and explain what B does instead.

### LOOKUP synthesis

Combine the per-directory lists. Structure:

1. Lead with the total match count across all directories.
2. Group by category if categories emerged, else flat list.
3. Each entry includes path, line, one-line context, and a snippet.
4. **Excluded** section: matches considered and ruled out, with reasons.
5. **Coverage caveats**: what the lookup might have missed (indirect usage, dynamic dispatch, string-templated calls, etc.).

LOOKUP is the mode where exhaustiveness matters most — be explicit about the limits of what was searched.

---

## Phase 6 — Save and report

### Exploration file structure

Save to `.claude/explorations/<slug>.md` in the cwd. Use this header for all modes:

```markdown
# Exploration: <short title>

> Mode: <EXPLAIN | COMPARE | LOOKUP>
> Subject: <one-line description of what was explored>
> Explored on: <ISO date>
> Directories:
>   - `<path-1>` (branch `<branch>` @ `<short-sha>`, or "not a git repo")
>   - `<path-2>` ...
> Depth: <overview | thorough>
> Anchors used: <list>
```

Then mode-specific body:

#### EXPLAIN body

```markdown
## Summary

<2-4 sentence high-level answer to the user's question>

## Entry point

`<path:line>` — <one-line description>

## How it works

<4-8 paragraphs walking through the flow end-to-end, with inline path:line
citations. This is the heart of the document.>

## Key files

- `<path:line>` — <one-line note>
- ...

## External dependencies

- <DB tables / third-party APIs / env vars / file system / etc.> at `<path:line>`
- ...

## Non-obvious aspects

<Things worth flagging to someone new to this code. Omit if nothing.>

## Open questions

<Things the exploration couldn't fully resolve from this scope. Omit if none.>
```

#### COMPARE body

```markdown
## Summary

<2-4 sentences: what's the same, what's different at a high level>

## Side-by-side

For each axis of comparison (entry point, data layer, config, etc.):

### <axis name, e.g. "Entry point">

- **<directory A>**: <description> at `<path:line>`
- **<directory B>**: <description> at `<path:line>`

(Repeat for each axis.)

## Differences

<Explicit list of divergences, each with both-side citations>

## Common elements

<Where A and B converge>

## Caveats

<E.g., "directory B has no equivalent for X — what it does instead is...">
```

#### LOOKUP body

```markdown
## Summary

<Total match count, plus a one-line characterization of what was found>

## Search angles used

- <pattern 1>
- <pattern 2>
- ...

## Matches

<Either flat list or grouped by category. Each entry:>

### <category, if grouped>

- `<path:line>` — <one-line context>
  ```<language>
  <one-line snippet>
  ```

- ...

## Excluded

<Matches considered and ruled out, with reasons. Omit if none.>

## Coverage caveats

<What the lookup might have missed — indirect usage, dynamic patterns,
out-of-scope directories, etc.>
```

### Report to the user

After saving, summarize in chat:

- File path: `.claude/explorations/<slug>.md`
- One-line headline appropriate to mode:
  - EXPLAIN: the high-level answer in one sentence.
  - COMPARE: the headline difference in one sentence.
  - LOOKUP: total match count and where the bulk lives.
- The directories the exploration covered and the branch/commit each was at.
- A reminder that the skill made no changes to any repo.
- Anything surprising the user should look at first.

---

## Rules

- **Read-only across all directories.** No commits, branches, edits, pushes, stashes, rebases — in any of the explored directories. File reads and read-only git inspection (`status`, `diff`, `log`, `branch`) are fine.
- **Cite real paths only.** Every `path:line` in the exploration must point to a file that actually exists at the recorded commit. No fabricated paths anywhere. Snippets in LOOKUP mode must be actual file content.
- **One mode per exploration.** EXPLAIN, COMPARE, or LOOKUP — not a mix. If the user has questions in multiple modes, run multiple explorations.
- **Loud about coverage limits.** Especially for LOOKUP: be explicit about what might have been missed. False sense of exhaustiveness is the most common exploration failure mode.
- **No proposals or recommendations.** This skill explores. It does not propose changes, suggest refactors, recommend approaches, or critique the code. If the exploration surfaces something that looks like a bug, mention it neutrally; don't pivot into bug-fixing.
- **The exploration is a snapshot.** Stamped with branch and commit for each git repo so it's reproducible. Non-git directories are stamped with the date only.
- **No silent path expansion.** The skill only reads paths the user explicitly named. No following symlinks out, no auto-including parent directories, no cwd-by-default.
