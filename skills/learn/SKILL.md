---
name: learn
description: Capture a specific lesson, gotcha, or pattern mid-session without a full retrospective. Use when something surprising happens, an approach fails, a useful pattern emerges, or the user points out something important to remember.
argument-hint: "[what was learned]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(ls *)
---

# Capture Learning

Quickly record a lesson, gotcha, or pattern for future sessions.

## Process

1. **Classify the learning** from `$ARGUMENTS` and conversation context:

   | Type | Where it goes | Example |
   |------|--------------|---------|
   | User preference or feedback | Memory (`feedback` type) | "Don't mock the database in tests" |
   | Project insight or context | Memory (`project` type) | "Remotion's staticFile() requires files in public/" |
   | Architecture decision | Decision log (`docs/decisions/`) | "Chose Zod over Ajv for schema validation" |
   | Reusable workflow | New skill (`.claude/skills/`) | "Character validation follows a 5-step pattern" |
   | Code convention | Update `CLAUDE.md` | "All async functions must handle errors with Result type" |

2. **Check for duplicates**: Read `MEMORY.md` and scan existing memories/decisions. If this updates an existing entry, edit it rather than creating a new file.

3. **Save it**:

   - **Memory**: Create/update a file in the memory directory with proper frontmatter (`name`, `description`, `type`). Update `MEMORY.md` index.
   - **Decision**: Create a numbered ADR in `docs/decisions/` using the template from the retro skill.
   - **Skill**: Use the manage-skill pattern to create it.
   - **Convention**: Edit `CLAUDE.md` to add the rule in the appropriate section.

4. **Confirm**: Tell the user in one sentence what was saved and where.

## Rules

- Be concise — a learning entry should be 1-5 lines, not an essay
- Use specific, actionable language — "Remotion requires `staticFile()` for assets in `public/`" beats "asset loading is tricky"
- Include *why* when possible — the reason makes the learning applicable in future contexts
- Don't interrupt workflow — this should take 30 seconds, not 5 minutes
- If `$ARGUMENTS` is empty, ask the user what they'd like to capture
