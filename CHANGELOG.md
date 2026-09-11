# Changelog

All notable changes to this plugin are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); version headings track
`version` in `.claude-plugin/plugin.json`.

## [Unreleased]

## [0.5.0] - 2026-08-05

### Added

- self-improvement-loop: Process Notes ledger (`docs/dev/process-notes.md`) — a third sibling to backlog/ideas that captures **workflow/tooling papercuts** (ambiguous command instructions, easy-to-forget steps, clunky hand-offs) instead of leaving them to evaporate in a retro. Entries carry a `[×N]` repeat-count marker; recurrences increment `N` and append a date rather than filing a duplicate, so repeat frequency becomes the prioritization signal (impact ≈ per-occurrence cost × frequency). Unlike backlog/ideas it is created in **every** project, not just prompt-pack ones, since every project has a friction writer (any session). Includes the section template, create step (5c), heading-detection entry, and a path/framing adaptation rule.

## [0.4.0] - 2026-07-04

### Added

- verify-coverage: new command — spec→pack coverage audit (the reverse of verify-build). Finds spec'd capabilities no pack owns (the "login slipped through" class), walks Non-Goals deferral chains to their terminus, adversarially re-checks orphan claims, and persists results to `_COVERAGE_REGISTRY.md` + the backlog
- create-prompt-packs: STEP 3.6 Feature Coverage Registry (`_COVERAGE_REGISTRY.md`) — every spec'd capability gets exactly one owning pack or a DEFERRED row with a named home, with a mandatory cross-cutting sweep (auth screens, audit logs, retention, observability, …) for the capabilities no module pack claims
- create-prompt-packs: generator self-audit gains closed-deferral-chain and registry-coverage checks; a "→ M[N]" deferral must land in a pack that actually accepts the scope
- create-prompt-packs: UI Depth Guardrails — packs transcribe page-definition-matrix rows verbatim, add a persona pass derived from the spec's target-users table, apply a list-view baseline (search/filter/sort/pagination/count/empty state/export), and push the baseline upstream into the spec's matrix intro as a fix-in-place edit
- build, verify-wiring, review-externally, self-improvement-loop: ideas ledger (`docs/dev/ideas.md`) — out-of-scope improvement ideas are captured (≤3-line entries, user-triaged) instead of discarded by overbuild prevention; distinct from the backlog (spec'd-but-deferred obligations); canonical template lives in build.md; self-improvement-loop installs the section + file
- create-prompt-packs: generated `_PREAMBLE` Section 15 carries the capture-don't-build rule; the mandatory backlog-sweep pack also triages the ideas ledger with the user; working-ledger stubs (backlog + ideas) are created at generation time

### Changed

- verify-build: Phase 0d cross-checks the pack against its `_COVERAGE_REGISTRY.md` rows (registry rows are deliverables even when the pack text omits them); Phase 5 diffs built UI pages against their page-matrix rows
- build: the spec's page-matrix row + list-view baseline are binding acceptance criteria for UI deliverables even where the pack summarizes them
- build-verify-review: warns when `_COVERAGE_REGISTRY.md` is missing; pack-complete summary lists newly captured ideas alongside spec-gap questions
- create-prompt-packs: post-generation pipeline reminder now includes `/verify-coverage` (step 0 + phase-boundary cadence)

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
