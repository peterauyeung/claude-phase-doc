# phase-doc

A [Claude Code](https://claude.com/claude-code) skill that documents a completed software
development phase as an auditable record:

```text
Phase Prompt → Implementation → Repository Changes → Testing / Validation
→ Phase Result → Blockers / Deviations → Next Steps
```

The guiding principle is **evidence over assumption**: nothing is marked complete or passing
without a commit, file, or command output to back it up.

## What it produces

Running the skill creates or updates documentation in the repository you are working in:

```text
docs/phases/
├── README.md              # index of all phases
└── phase-<id>/
    ├── PROMPT.md          # original phase prompt (verbatim) and requirements
    ├── RESULT.md          # what actually happened, requirement traceability, blockers, ADRs
    ├── CHANGES.md         # repository changes, derived from Git
    └── VALIDATION.md      # tests and validation that were actually executed
```

Key behaviors:

- Preserves the original prompt word for word; any reconstruction is clearly labeled.
- Separates *implemented* from *verified*. Allowed statuses: `COMPLETE`, `PARTIAL`, `BLOCKED`,
  `FAILED`, `NOT DONE`, `NOT TESTED`, `NOT APPLICABLE`.
- Tracks blockers with stable global IDs (`B001`, `B002`, …) carried across phases, with history.
- Records significant architecture decisions as `ADR-<PHASE>-NNN`, or follows an existing ADR
  convention in the repository.
- Re-running for the same phase updates that phase in place without discarding history.
- Never modifies application code and never stages, commits, or pushes. You review and commit
  the documentation yourself.

## Installation

Claude Code loads skills from `~/.claude/skills/` (all projects) or `.claude/skills/` inside a
repository (that project only). Skills are discovered when a session starts, so **restart
Claude Code after installing**.

### Option 1: user-level (available in every project)

```bash
git clone git@github.com:peterauyeung/claude-phase-doc.git
mkdir -p ~/.claude/skills
cp -r claude-phase-doc/.claude/skills/phase-doc ~/.claude/skills/
```

### Option 2: project-level (shared with a repository's collaborators)

From the root of the target repository:

```bash
mkdir -p .claude/skills
cp -r /path/to/claude-phase-doc/.claude/skills/phase-doc .claude/skills/
git add .claude/skills/phase-doc
git commit -m "chore: add phase documentation skill"
```

### Verify

Start a new Claude Code session and type `/phase-doc`. The skill should appear in the
slash-command list.

### Updating

Pull the latest version of this repository and copy the skill directory again, overwriting the
installed copy:

```bash
cd claude-phase-doc && git pull
cp -r .claude/skills/phase-doc ~/.claude/skills/
```

### Uninstalling

```bash
rm -r ~/.claude/skills/phase-doc        # user-level
rm -r .claude/skills/phase-doc          # project-level
```

## Usage

At the end of a development phase, from the repository being documented:

```text
/phase-doc Phase 3B
```

With no argument, the skill infers the phase from the conversation, branch, and existing
documentation, and asks if the phase name is ambiguous:

```text
/phase-doc
```

The skill is user-invoked only (`disable-model-invocation: true`), so Claude will not run it
on its own.

For best results, run it in the same session in which the phase was implemented, so the
original prompt and test output are still available in the conversation.

Optionally, add a short note to the target repository's `CLAUDE.md`:

```markdown
## Development Phase Documentation

Use `/phase-doc` when completing a development phase. Phase documentation is maintained under
`docs/phases/<phase>/` (`PROMPT.md`, `RESULT.md`, `CHANGES.md`, `VALIDATION.md`) and must
distinguish requested work from verified implementation.
```

## Repository layout

```text
.claude/skills/phase-doc/SKILL.md   # the skill definition
README.md
```
