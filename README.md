# Salian's Claude Toolkit

A collection of reusable [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) **skills** and **slash commands**, packaged as an installable plugin. Use it to back up your custom workflows, sync them across machines, and share them.

## What's inside

### Commands (you invoke explicitly)

A prompt-pack build pipeline — run them in order — plus a few utilities.

| Command | What it does |
|---|---|
| `/create-prompt-packs [spec]` | Decompose specs (PRD, roadmap, agents.md) into milestone-by-milestone build plans that `/build` can execute. **Run this first.** |
| `/verify-coverage [scope]` | Audit spec→pack ownership: every spec'd capability has an owning pack or an explicit deferred home. The per-pack commands can't see features that landed in no pack — this can. Run after generation, after spec changes, and at phase boundaries. |
| `/build [pack-id]` | Execute a prompt pack — build all features in a milestone with tests, committing incrementally. Resume-safe. |
| `/verify-build [pack-id]` | Audit build completeness: every deliverable, test, and acceptance criterion specified in the pack. Run before `/verify-wiring`. |
| `/verify-wiring [pack-id]` | Audit integration wiring: every feature is reachable end-to-end (navigation, API callers, schedulers, UI), not just present. |
| `/review-externally [scope]` | Run an external code review (via Codex) on prompt-pack commits or recent changes. |
| `/build-verify-review [flags]` | Orchestrator: run build → verify-build → verify-wiring → review-externally for every pack, fixing gaps autonomously. |
| `/retrofit-pipeline-gates [flags]` | Retrofit later pipeline upgrades (coverage registry, spec red-team, backlog sweep, ideas ledger) onto an in-flight pack project whose packs predate them. Upgrades unbuilt packs in place; never regenerates. |
| `/self-improvement-loop` | Add a self-improvement loop to a project's `CLAUDE.md` — learning behaviors, decision logging, backlog/ideas/process-notes ledgers, commit/changelog practices. Idempotent. |
| `/changelog [description]` | Update `CHANGELOG.md` (Keep a Changelog format) from your recent changes. |
| `/smart-commit [message]` | Stage changes, update the changelog, and create a well-formatted commit. |

### Skills (Claude invokes automatically from context)

| Skill | Triggers when |
|---|---|
| `learn` | Something surprising happens, an approach fails, a useful pattern emerges, or you point out something to remember — captures it to memory / decisions / `CLAUDE.md`. |
| `manage-skill` | A reusable workflow pattern emerges, or you ask to create/update/list/delete a skill. |

> **Skill vs. command:** a *command* lives in `commands/<name>.md`, is invoked by you as `/<name>`, and supports `$ARGUMENTS`. A *skill* lives in `skills/<name>/SKILL.md`, is auto-invoked by Claude based on its `description`, and does not take `$ARGUMENTS`.

## Install

### As a plugin (recommended — syncs across machines)

```
/plugin marketplace add salian/claude-toolkit
/plugin install salian-claude-toolkit@salian-claude-toolkit
```

Once installed, the commands and skills are available in every project on that machine. Re-run `/plugin marketplace update salian-claude-toolkit` to pull the latest.

### Manual (single machine, no plugin)

Copy or symlink the pieces into your Claude Code config:

```bash
git clone https://github.com/salian/claude-toolkit.git
cd claude-toolkit

# user-level (all projects)
ln -s "$PWD/commands"/*.md   ~/.claude/commands/
ln -s "$PWD/skills"/*        ~/.claude/skills/
```

Or drop them under a single project's `.claude/commands/` and `.claude/skills/` instead.

## Scope

These are **Claude Code–specific** in format (`SKILL.md`, slash commands with `$ARGUMENTS`, `allowed-tools` frontmatter) — they won't run as-is in other AI coding tools. Some commands also assume a few conventions: a `memory/` directory with `MEMORY.md`, `docs/decisions/` for ADRs, and a prompt-pack build flow. Where those are absent the skills adapt or ask; nothing breaks.

## Layout

```
.claude-plugin/
  plugin.json        # plugin manifest
  marketplace.json   # marketplace listing
commands/            # slash commands (build pipeline + utilities)
skills/              # auto-invoked skills (learn, manage-skill)
```

## License

[MIT](LICENSE)
