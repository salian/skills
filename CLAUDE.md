## Self-Improvement Loop

This project maintains a learning system that gets smarter over each session. The following behaviors are **automatic** — do them without being asked.

### When to capture knowledge (`/learn`)

Invoke `/learn` (or save directly to memory/decisions) when any of these happen:
- An approach fails and you pivot — record what didn't work and why
- A library or API behaves unexpectedly — record the gotcha
- The user corrects your approach — record the feedback
- You discover a non-obvious pattern that works well — record it
- A design decision is made during implementation — add an ADR to `docs/decisions/`

### When to create or update skills (`/manage-skill`)

- You perform the same multi-step workflow twice in a session — extract it into a skill
- An existing skill produces suboptimal results — update it
- The user describes a workflow they want repeatable — create a skill for it

### When to run a retrospective (`/retro`)

- After completing a phase checklist item
- After completing a significant milestone (even mid-phase)
- When the user says they're done, wrapping up, or stepping away
- At the end of any session where substantial work was done

### When to update this file (CLAUDE.md)

- A new code convention emerges from implementation
- An existing rule proves wrong or unhelpful
- The project structure changes in a way that affects how work is done

### Decision log (`docs/decisions/`)

Record architecture and design decisions as numbered ADRs (`001-title.md`, `002-title.md`). Capture:
- Library/tool choices and why
- Data format decisions
- Patterns adopted or rejected
- Anything where "why did we do it this way?" might come up later

## Automatic Behaviors

### Changelog

After completing any meaningful code change, update `CHANGELOG.md`:
- Use Keep a Changelog format (keepachangelog.com)
- Group under: Added, Changed, Fixed, Removed
- Use the `/changelog` skill to update it

### Commits

When asked to commit or after completing a task that warrants a commit:
- Use the `/commit` skill (built-in) or `/smart-commit` skill
- Commit message style: imperative mood, concise
- Always update CHANGELOG.md before committing
- Never commit `.DS_Store`, `node_modules/`, `output/`, or `.env` files

## Phase Workflow

1. Read the relevant spec in `specs/` before starting work
2. Break work into the checklist items from the spec
3. Each checklist item should produce a visible result
4. Update CHANGELOG.md after each meaningful change
5. Commit at natural milestones
6. Run `/retro` after completing each checklist item or milestone

## Skills Reference

| Skill | Trigger | Purpose |
|-------|---------|---------|
| `/changelog` | After code changes | Update CHANGELOG.md |
| `/smart-commit` | At milestones | Changelog + stage + commit |
| `/manage-skill` | When patterns repeat | Create/update/list/delete skills |
| `/retro` | End of session or milestone | Full retrospective: lessons, skills, rules |
| `/learn` | Mid-session discovery | Quick knowledge capture |


