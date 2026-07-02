# Changelog

All notable changes to this plugin are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); version headings track
`version` in `.claude-plugin/plugin.json`.

## [Unreleased]

## [0.3.0] - 2026-07-03

### Added

- create-prompt-packs: STEP 1.5 adversarial spec red-team pass before decomposition, with blocking / fix-in-place / minor triage
- create-prompt-packs: mandatory final `NN_HARDENING_BACKLOG_SWEEP` pack so `docs/dev/backlog.md` gains a guaranteed reader, enforced via the generator self-audit

### Changed

- build, verify-build, verify-wiring, review-externally: define `M12`/`M7a` → `12_`/`07a_` file-prefix normalization at every pack lookup; ambiguous matches ask instead of guessing
- verify-build, verify-wiring: report without asking "Fix all N gaps?" when invoked by /build-verify-review (autonomy-contract exception)

### Fixed

- verify-build: stale "8a" section reference (Failure Modes is Section 11)
- verify-wiring: remove stray skill-style `name:` field from command frontmatter

## [0.2.0] - 2026-07-02

### Added

- create-prompt-packs: Schema Object Registry (`_SCHEMA_REGISTRY.md`) — a global schema-object namespace that prevents cross-pack name collisions and owner/consumer drift at generation time
- create-prompt-packs: schema⇄spec drift guardrails, including a CI drift gate with a red-then-green acceptance bar

## [0.1.0] - 2026-06-28

### Added

- Restructure into the `salian-claude-toolkit` Claude Code plugin, with plugin manifest and marketplace listing
- Prompt-pack pipeline commands: create-prompt-packs, build, build-verify-review, verify-build, verify-wiring, review-externally
- Workflow commands: changelog, smart-commit, self-improvement-loop
- Skills: learn, manage-skill
- MIT License

### Changed

- Rename repo to `salian/claude-toolkit`, drop redundant owner prefix in URLs
