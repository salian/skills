# Project Instructions

This repository is a **Claude Code plugin** that distributes reusable skills and slash
commands. It is meant to be installed on other machines and shared publicly, so keep
everything here generic and self-contained — do not assume a specific consuming project.

## Repository layout

- `.claude-plugin/plugin.json` — plugin manifest (name, version, metadata)
- `.claude-plugin/marketplace.json` — marketplace listing so others can `/plugin marketplace add salian/claude-toolkit`
- `commands/<name>.md` — slash commands, invoked explicitly by the user as `/<name>`
- `skills/<name>/SKILL.md` — skills, auto-invoked by Claude based on their `description`

## Authoring rules

- **Commands vs. skills.** If the user should trigger it explicitly and it takes input,
  make it a **command** (`commands/<name>.md`) with `description`, `argument-hint`, and
  `allowed-tools` frontmatter, and use `$ARGUMENTS` in the body. If Claude should trigger
  it automatically from context, make it a **skill** (`skills/<name>/SKILL.md`) with
  `name`, `description`, and `allowed-tools` — skills do **not** get `$ARGUMENTS`
  substitution, so write them to read input from the conversation instead.
- **Descriptions are the trigger.** A skill's `description` is how Claude decides when to
  invoke it. Be specific about WHEN, using "Use when..." phrasing.
- **Minimal `allowed-tools`.** Grant only what the workflow needs.
- **One workflow per file.** Keep each skill/command focused.
- **Stay portable.** Don't hardcode paths or conventions from a private project. If a
  skill relies on optional conventions (e.g. a `memory/` dir or `docs/decisions/`), make
  it degrade gracefully when they're absent.

## When you change anything here

- Update `README.md` if you add, remove, or rename a command or skill (it has a table of each).
- Update `CHANGELOG.md` for every meaningful change, in the same commit. Add one-line
  entries under `## [Unreleased]` using Keep a Changelog categories (Added / Changed /
  Fixed / Removed). Never rewrite entries under a released version heading.
- Bump `version` in both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`
  when publishing a meaningful change. When you bump, rename `## [Unreleased]` to
  `## [x.y.z] - YYYY-MM-DD` in `CHANGELOG.md` and start a fresh empty `## [Unreleased]`.
- Keep `name` fields lowercase-with-hyphens and matching the file/directory name.
