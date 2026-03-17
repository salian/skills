---
name: manage-skill
description: Create, update, or list custom Claude Code skills for this project. Use when a reusable workflow pattern emerges, or when asked to add/edit/remove a skill.
argument-hint: "[create|update|list|delete] [skill-name] [description (optional)]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(ls *), Bash(mkdir *), Bash(rm *)
---

# Manage Project Skills

Create, update, list, or delete custom skills in `.claude/skills/`.

## Commands

### `create <skill-name> <description>`
or just `/manage-skill create <skill-name>`

1. Create directory `.claude/skills/<skill-name>/`
2. Create `SKILL.md` with appropriate frontmatter:
   - `name`: the skill name (lowercase, hyphens only)
   - `description`: clear description for auto-invocation — be specific about WHEN to trigger
   - `argument-hint`: what arguments it accepts (if any)
   - `allowed-tools`: minimum set of tools needed
3. Write the skill body with:
   - A clear heading
   - Step-by-step process
   - Rules/constraints
4. Update `CLAUDE.md` if the skill relates to a project workflow

### `update <skill-name>`

1. Read the existing skill at `.claude/skills/<skill-name>/SKILL.md`
2. Discuss what needs to change with the user
3. Edit the skill file
4. If the description changed significantly, confirm auto-invocation still makes sense

### `list`

1. Glob for `.claude/skills/*/SKILL.md`
2. Read each skill's frontmatter
3. Display a table: name, description, invocation

### `delete <skill-name>`

1. Confirm with the user before deleting
2. Remove the skill directory

## Skill Writing Guidelines

- **Description is critical** — Claude uses it to decide when to auto-invoke. Be specific about trigger conditions. Include "Use when..." phrasing.
- **Keep skills focused** — one workflow per skill, not a Swiss Army knife
- **Minimal allowed-tools** — only grant what the skill needs
- **Use `$ARGUMENTS`** — make skills accept parameters for flexibility
- **Include rules** — explicit constraints prevent mistakes
- **Test after creating** — try invoking the skill to verify it works
