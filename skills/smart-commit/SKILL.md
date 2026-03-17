---
name: smart-commit
description: Stage changes, update the changelog, and create a well-formatted git commit. Use when the user asks to commit, save progress, or checkpoint work.
argument-hint: "[commit message (optional)]"
allowed-tools: Read, Edit, Grep, Glob, Bash(git *)
---

# Smart Commit

Create a git commit with automatic changelog update and well-crafted commit message.

## Process

1. **Check status**: Run `git status` and `git diff` to understand what changed. If there are no changes, tell the user and stop.

2. **Update changelog**: Read `CHANGELOG.md` and add entries for any changes not yet recorded. Follow Keep a Changelog format under `## [Unreleased]`. Do not duplicate existing entries.

3. **Stage files**: Stage all relevant changed files. Never stage:
   - `.DS_Store`
   - `node_modules/`
   - `output/`
   - `.env` or credential files
   - `.claude/settings.local.json`

4. **Craft commit message**: If `$ARGUMENTS` is provided, use it as the commit message. Otherwise, write one that:
   - Uses imperative mood ("Add feature" not "Added feature")
   - Is concise (under 72 chars for the first line)
   - Includes a body with bullet points if multiple things changed
   - References the phase if relevant (e.g., "Phase 1: Add CharacterRig compositor")

5. **Commit**: Create the commit. Use a HEREDOC for multi-line messages.

6. **Verify**: Run `git status` and `git log --oneline -3` to confirm success. Report the commit hash to the user.

## Rules

- Always update CHANGELOG.md before committing (include it in the commit)
- Never amend previous commits unless explicitly asked
- Never push unless explicitly asked
- Never use `--no-verify`
- If pre-commit hooks fail, fix the issue and create a NEW commit
