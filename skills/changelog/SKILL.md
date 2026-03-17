---
name: changelog
description: Update CHANGELOG.md with recent changes. Use after completing meaningful code changes, before committing. Auto-invoked when changes are ready to be recorded.
argument-hint: "[description of changes (optional)]"
allowed-tools: Read, Edit, Grep, Glob, Bash(git diff*), Bash(git log*)
---

# Update Changelog

Update `CHANGELOG.md` to reflect recent changes made in this session.

## Process

1. **Identify changes**: If `$ARGUMENTS` is provided, use that description. Otherwise, review recent work by checking `git diff` (staged and unstaged) and the conversation context to understand what changed.

2. **Read current changelog**: Read `CHANGELOG.md` to see existing entries and avoid duplicates.

3. **Categorize changes** using Keep a Changelog sections:
   - **Added** — new features, files, capabilities
   - **Changed** — modifications to existing functionality
   - **Fixed** — bug fixes
   - **Removed** — removed features or files
   - **Deprecated** — features marked for future removal
   - **Security** — vulnerability fixes

4. **Write entries**: Add concise, meaningful entries under `## [Unreleased]`. Each entry should:
   - Start with a verb (Add, Implement, Fix, Remove, Update, Refactor)
   - Be one line, describing the *what* and *why* (not the *how*)
   - Reference the spec phase if relevant (e.g., "Phase 1:")
   - NOT duplicate entries already in the changelog

5. **Confirm**: Tell the user what was added to the changelog.

## Rules

- Never remove existing entries
- Never change entries under a released version heading
- Keep entries concise — one line each
- Group related small changes into a single entry when it makes sense
- If no meaningful changes to record, say so and skip the update
