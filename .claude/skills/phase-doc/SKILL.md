---
name: phase-doc
description: Document a completed software development phase by recording the original prompt, implementation results, repository changes, validation results, blockers, deviations, decisions, and next steps.
disable-model-invocation: true
---

# phase-doc — Development Phase Documentation

Produce an auditable record of one development phase:

```text
Phase Prompt → Claude Implementation → Repository Changes → Testing / Validation
→ Phase Result → Blockers / Deviations → Next Steps
```

**Core principle: evidence over assumption.** Every statement must be traceable to the
conversation, the repository, Git history, or command output. If something is unknown,
say it is unknown.

## Invocation

```text
/phase-doc              # infer the phase from conversation/branch/existing docs; ask if ambiguous
/phase-doc Phase 3B     # document the named phase
```

The argument is available as `$ARGUMENTS`. Normalize it to a directory slug:
lowercase, strip a leading `phase`, remove spaces/punctuation, prefix `phase-`.
Examples: `Phase 3B` → `phase-3b`, `phase 2` → `phase-2`, `3a` → `phase-3a`.

If no argument is given and the phase cannot be determined unambiguously, ask the user.
Never guess a phase name.

## Output layout

```text
docs/phases/
├── README.md              # phase index (all phases)
└── phase-<id>/
    ├── PROMPT.md          # original requirements
    ├── RESULT.md          # what actually happened
    ├── CHANGES.md         # repository changes (from Git)
    └── VALIDATION.md      # tests and validation actually performed
```

One directory per phase. Never create variants such as `phase-3b-1/`, `phase-3b-2/`,
or `phase-3b-final/` unless the user explicitly asks.

## Procedure

Follow these steps in order: **Inspect → Document → Validate → Report.**

### 1. Inspect

Gather evidence before writing anything.

1. Repository state:
   ```bash
   git rev-parse --show-toplevel
   git branch --show-current
   git status
   git log --oneline -30
   git remote -v
   ```
2. Determine the phase's **starting commit** and **ending commit**:
   - Prefer the starting commit recorded in an existing `PROMPT.md`/`RESULT.md`.
   - Otherwise use evidence from the conversation (e.g. a commit noted when the phase began),
     a merge base, or a tag. If it cannot be determined, write `UNKNOWN` and explain why.
   - The ending commit is `HEAD` unless the phase ended earlier. If uncommitted changes are
     part of the phase, say so explicitly ("HEAD + uncommitted working-tree changes").
3. Actual changes for the phase range:
   ```bash
   git diff --stat <start>..<end>
   git diff --name-status <start>..<end>
   git diff <start>..<end>
   git log --oneline <start>..<end>
   git status
   git diff               # uncommitted, if relevant
   git diff --cached      # staged, if relevant
   ```
4. Existing documentation:
   - `docs/phases/README.md` and every existing `docs/phases/*/` directory.
   - The target phase directory, if it already exists (this is an update, not a rewrite).
   - The previous phase(s): objectives, results, blockers (open and resolved), ADRs,
     dependencies, deviations, and recommended next steps.
   - Any existing ADR convention (e.g. `docs/adr/`, `docs/decisions/`, `ADR-*.md`).
   - `CLAUDE.md` and other repository conventions.
5. The conversation: the original phase prompt, commands actually executed and their output,
   problems hit, decisions made, and anything the user said about scope.

### 2. Document

Create or update the four phase files and the index using the templates below.

**Re-runs update in place.** If a file already exists:
- Read it fully first.
- Update sections with new evidence; keep prior content that is still accurate.
- Never silently delete historical information (blockers, decisions, deviations, earlier
  validation runs). If something is superseded, mark it as such and keep it.
- When appending a later validation run or result revision, label it with the date and commit.

### 3. Validate the documentation

Before reporting, check your own output:
- Every `COMPLETE` / `PASS` has concrete evidence (command + output, file path, commit).
- Every file listed in `CHANGES.md` appears in the Git output; no Git change is missing.
- Every blocker from previous phases is carried forward or explicitly resolved.
- No reconstructed prompt is presented as the original.
- Commit hashes, dates, and statuses in `README.md` match the phase files.
- Markdown tables are well-formed.

### 4. Report

Summarize to the user: files created/updated, overall phase status, open blockers,
validation gaps, and anything you could not determine. Do **not** commit or push.

## Status vocabulary

Requirement / result statuses (use only these):

```text
COMPLETE  PARTIAL  BLOCKED  FAILED  NOT DONE  NOT TESTED  NOT APPLICABLE
```

- **Never mark `COMPLETE` without evidence.**
- Distinguish **IMPLEMENTED** (code exists) from **VERIFIED** (executed and observed to work).
  Modified source files alone do not prove success. Example:

  ```text
  Code implemented:      YES
  Unit test executed:    NO
  Production validation: NO
  Overall status:        NOT TESTED
  ```

Validation statuses in `VALIDATION.md`: `PASS`, `FAIL`, `PARTIAL`, `NOT RUN`,
`NOT APPLICABLE`. Use `NOT RUN` for anything not actually executed in this phase.

Blocker statuses: `OPEN`, `IN PROGRESS`, `RESOLVED`, `WONT FIX`, `SUPERSEDED`.

## Template: PROMPT.md

Preserve the original prompt **verbatim** in "Original Prompt" (use a fenced block or
blockquote). Do not edit, summarize, or "clean up" it. The other sections may summarize it,
but must not change its meaning.

If the exact original prompt is not available, write this notice immediately under
`## Original Prompt`, then the reconstruction:

> The original prompt could not be recovered exactly. The prompt below is reconstructed from available repository/conversation history.

Never present reconstructed content as the original. If a previously recorded original prompt
exists, never replace it with a reconstruction.

```markdown
# Phase <PHASE> — Prompt

## Metadata

- Phase:
- Date:
- Repository:
- Branch:
- Starting commit:

## Objective

## Original Prompt

## Requirements

## Constraints

## Expected Deliverables

## Known Blockers Before Phase

## Success Criteria
```

"Known Blockers Before Phase" lists blockers carried in from earlier phases by ID
(e.g. `B005 — OPEN (from Phase 2)`).

## Template: RESULT.md

```markdown
# Phase <PHASE> — Result

## Status

- Overall:
- Date:
- Branch:
- Starting commit:
- Ending commit:

## Executive Summary

## Work Completed

## Technical Implementation

## Requirement Traceability

| Requirement | Status | Evidence | Notes |
|---|---|---|---|
| | COMPLETE/PARTIAL/BLOCKED/FAILED/NOT DONE/NOT TESTED | | |

## Tests Executed

## Test Results

## Problems Encountered

## Blockers

## Decisions Made

## Deviations From Original Prompt

## Remaining Work

## Recommendation for Next Phase
```

- Requirement Traceability: one row per requirement from `PROMPT.md`. "Evidence" must point at
  something checkable (commit, file path, command output in `VALIDATION.md`).
- Overall status is the most conservative honest summary, e.g. `PARTIAL` if any required item
  is not `COMPLETE`.
- Tests Executed / Test Results: only what was actually run; details live in `VALIDATION.md`.
- Blockers: use the blocker format below. Decisions Made: use the ADR format below for
  significant decisions; list minor decisions as bullets.
- Deviations: every place the implementation differs from the prompt, with the reason.

## Template: CHANGES.md

Derive everything from Git output. **Do not invent changes.** Write `None` for empty
categories rather than omitting them.

```markdown
# Phase <PHASE> — Repository Changes

## Summary

| Change Type | Count |
|---|---:|
| Added | |
| Modified | |
| Deleted | |
| Renamed | |

## Files Added

## Files Modified

## Files Deleted

## Database Changes

## API Changes

## Configuration Changes

## Infrastructure / Deployment Changes

## Dependency Changes

## Documentation Changes

## Git Information

- Starting commit:
- Ending commit:
- Branch:
```

- Counts come from `git diff --name-status` (`A`, `M`, `D`, `R*`).
- Note uncommitted changes separately if they are part of the phase.
- Category sections (Database, API, …) describe the substance of relevant diffs
  (migrations, endpoints, config keys, Dockerfiles/manifests, dependency manifests/lockfiles).
- The phase documentation files themselves belong under "Documentation Changes".

## Template: VALIDATION.md

Document **only validation that was actually performed.** Never claim a test passed unless it
was executed and its output observed. Include the exact command and a relevant excerpt of the
output (trim long logs; keep the summary lines and any failures).

```markdown
# Phase <PHASE> — Validation

## Environment

## Commands Executed

## Validation Results

| Validation | Status | Evidence / Notes |
|---|---|---|
| Build | | |
| Unit Tests | | |
| Integration Tests | | |
| Static Analysis | | |
| Database Migration | | |
| Container Build | | |
| Deployment | | |
| Smoke Test | | |

## Test Output / Evidence

## Known Validation Gaps
```

- Environment: OS, runtime/tool versions, and target (local / CI / cluster) — only what is known.
- Rows not exercised in this phase are `NOT RUN` (or `NOT APPLICABLE` with a reason).
- Known Validation Gaps: everything that would be needed to move `NOT TESTED` items to verified.

## Blocker tracking

Blocker IDs are **global and stable** across all phases: `B001`, `B002`, `B003`, …

- Before assigning a new ID, scan all `docs/phases/*/RESULT.md` and `PROMPT.md` for the
  highest existing `B###` and use the next number. Never reuse or renumber an ID.
- Carry every unresolved blocker from earlier phases forward into the current phase's
  `RESULT.md` with its current status.
- When a blocker's status changes, update it in the current phase and add a history line;
  do not edit away its history in earlier phases.

```markdown
### B001 — <Short Description>

**Status:** OPEN

**Description:**

**Root Cause:**

**Impact:**

**Evidence:**

**Required Resolution:**

**Target Phase:**

**History:**
- Phase 2 — OPEN — identified
- Phase 3B — RESOLVED — <evidence>
```

Statuses: `OPEN`, `IN PROGRESS`, `RESOLVED`, `WONT FIX`, `SUPERSEDED`
(for `SUPERSEDED`, name the replacing blocker).

Continuity example:

```text
Phase 2   B005 OPEN
Phase 3A  B005 OPEN
Phase 3B  B005 RESOLVED
```

## Architecture decisions

Record significant decisions involving architecture, database, API design, security,
infrastructure, deployment, major dependencies, runtime/framework, or migration strategy.
Do not create ADRs for trivial implementation details.

If the repository already has an ADR convention, follow it and link to the ADR from
`RESULT.md`. Otherwise record it in `RESULT.md` under "Decisions Made":

```markdown
### ADR-<PHASE>-001 — <Decision>

**Decision:**

**Reason:**

**Alternatives Considered:**

**Impact:**

**Phase:**
```

Number ADRs sequentially within a phase (`ADR-3B-001`, `ADR-3B-002`, …). If a later phase
reverses a decision, add a new ADR that references the old one; do not delete the old one.

## Phase index: docs/phases/README.md

Create it if missing; otherwise update the row for this phase (add it if absent) and leave
other rows intact unless they are demonstrably wrong.

```markdown
# Development Phases

| Phase | Status | Date | Starting Commit | Ending Commit | Blockers | Summary |
|---|---|---|---|---|---:|---|
```

- Blockers column: count of blockers **open** at the end of that phase.
- Use short commit hashes. Use `UNKNOWN` rather than fabricating any value.
- Keep rows in phase order.

## Rules

1. **Evidence over assumption.** Do not fabricate dates, commits, statuses, test results,
   blockers, or changes. Use `UNKNOWN` / `NOT RUN` and explain.
2. **Preserve the original prompt** verbatim; label any reconstruction.
3. **Preserve history.** Never silently remove blockers, decisions, deviations, or earlier
   results — mark them resolved/superseded instead.
4. **Idempotent.** Re-running for the same phase updates `docs/phases/phase-<id>/` in place.
5. **Documentation only.** Do not modify application code, tests, schemas, or configuration
   while running this skill. Do not run destructive commands.
6. **No Git writes.** Do not stage, commit, or push. The user decides when to commit the
   documentation.
7. **Do not create fake phases.** Only document the phase requested; do not create
   placeholder directories for other phases.
