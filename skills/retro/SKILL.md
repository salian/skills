---
name: retro
description: End-of-session retrospective. Reviews work done, captures lessons in memory, identifies skill opportunities, updates CLAUDE.md and decision log. Use at the end of a work session, after completing a phase checklist item, or when the user says they're done for now.
argument-hint: "[focus area (optional)]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git diff*), Bash(git log*), Bash(git status*), Bash(ls *), Bash(mkdir *)
---

# Session Retrospective

Review the session's work, extract durable knowledge, and strengthen the project's self-awareness.

## Process

### 1. Gather context

- Run `git log --oneline -20` and `git diff --stat HEAD~5..HEAD` (adjust range to cover this session's commits)
- Review the conversation context for decisions made, problems encountered, patterns discovered
- If `$ARGUMENTS` specifies a focus area, concentrate there

### 2. Capture lessons learned

For each significant insight from this session, decide if it belongs in:

- **Memory** (persistent across conversations) — save to `~/.claude/projects/-Users-pranab-Dropbox-Projects-zalapo-code-zalapo-stage-engine/memory/`
  - New understanding about the project, user preferences, or effective approaches
  - Gotchas or failure modes discovered
  - Update `MEMORY.md` index after adding/updating memory files
- **Decision log** (in-repo, version-controlled) — save to `docs/decisions/`
  - Architecture or design choices made during this session
  - Use the ADR template (see below)
- **Neither** — if the insight is already captured in code, comments, or specs, skip it

### 3. Evaluate skills

Review the session for repeated workflows or multi-step patterns that could become skills:

- Did I do something 2+ times that could be automated?
- Did the user ask me to do something that would benefit from a structured skill?
- Are existing skills missing steps or producing suboptimal results?

For each opportunity:
- If it's a new skill: create it using the manage-skill pattern (`.claude/skills/<name>/SKILL.md`)
- If it's an improvement to an existing skill: edit the skill file directly
- Report what was created or updated

### 4. Review project rules

Read `CLAUDE.md` and check:
- Are any rules outdated given what we built this session?
- Are there new conventions that emerged and should be codified?
- Are the automatic behavior triggers still appropriate?

If changes are needed, edit `CLAUDE.md` and explain what changed and why.

### 5. Update changelog

If changelog wasn't already updated during the session, update it now.

### 6. Report

Summarize to the user:
- **Work completed** — what was built/changed this session
- **Knowledge captured** — what memories, decisions, or skill updates were made
- **Open threads** — anything unfinished or worth picking up next session
- **Project health** — any concerns about direction, tech debt, or risks noticed

Keep the report concise — bullet points, not paragraphs.

## ADR Template

When creating a decision record in `docs/decisions/`, use this format:

```markdown
# ADR-NNN: Title

**Date:** YYYY-MM-DD
**Status:** accepted | superseded | deprecated
**Phase:** which build phase this relates to

## Context
What prompted this decision?

## Decision
What was decided and why?

## Alternatives considered
What else was considered and why it was rejected?

## Consequences
What are the implications — positive, negative, and risks?
```

Number ADRs sequentially: `001-title.md`, `002-title.md`, etc.

## Rules

- Don't fabricate lessons — only capture what actually happened
- Don't create skills speculatively — only for patterns that actually occurred
- Keep memory files focused — one topic per file, update existing files before creating new ones
- Be honest about what went wrong, not just what went right
- If nothing notable happened, say so — don't force a retro for trivial sessions
