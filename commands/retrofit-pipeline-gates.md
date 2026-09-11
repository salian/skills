---
description: Retrofit the pipeline upgrades (Schema Object Registry, spec red-team, backlog sweep, coverage gates, ideas ledger, UI depth guardrails) onto an in-flight prompt-pack project whose packs predate them. Upgrades unbuilt packs in-situ; routes built-pack gaps to the backlog; never regenerates.
argument-hint: "[optional: --skip-phase N | --only-phase N]"
allowed-tools: "*"
---

# Retrofit Pipeline Gates onto an In-Flight Prompt-Pack Project

This project's prompt packs were generated before three upgrades to the global pipeline commands (Schema Object Registry + spec-drift guardrails; spec red-team + mandatory backlog-sweep pack; spec→pack coverage gates + ideas ledger + UI depth guardrails). The commands themselves (global `~/.claude/commands/` or the claude-toolkit plugin) are already current — this session retrofits the **project-side artifacts** they now expect.

**Prime directive — respect build history.** Upgrade UNBUILT packs in-situ. Never regenerate the pack set. Never rewrite a built pack's text (its text must keep matching what its commits actually did). Gaps discovered in already-built work become `docs/dev/backlog.md` entries, not pack edits. Determine built vs unbuilt from `.claude/.bvr-state.json` (`completedPacks`) if present, else from `git log` (conventional-commit scopes per pack); when uncertain, treat a pack as built — the safe side routes findings to the backlog.

Every phase is **idempotent**: check whether the artifact already exists/conforms first, skip and say so if it does. `$ARGUMENTS` may skip or isolate phases.

## Phase 0 — Inventory

Read: `CLAUDE.md`; `docs/prompt-packs/README.md` + `_PREAMBLE.prompt.md`; `_SCHEMA_REGISTRY.md` / `_COVERAGE_REGISTRY.md` if present; `docs/dev/backlog.md` / `docs/dev/ideas.md` if present; the spec corpus cited by the packs' §3 sections; the build state. Produce: the pack list split built/unbuilt, the spec file list, and a per-phase "already present?" checklist.

## Phase 1 — Schema Object Registry + drift guardrails

1. If `_SCHEMA_REGISTRY.md` is missing, generate it per the global create-prompt-packs command's STEP 3.5: one row per persistent object (table/enum) — owner pack, canonical (domain-prefixed) name, one-line shape, consumer packs. Sources: every pack's §7 (declared) + §2 (consumed), reconciled against the **actual built schema** (migrations/schema files) — for built objects, the built name wins.
2. Run the registry conformance audit (STEP 3.5 rules): (a) no name declared by two packs; (b) every consumed name has a declaring owner; (c) paraphrase-drift — a consumer referencing a variant of the owner's name; (d) bare generic nouns (`task`, `contact`, `message`, `module`, …) lacking a domain prefix. Fix violations in UNBUILT pack texts (rename to registry names, update the registry). Violations baked into BUILT code get a backlog entry each — a rename there is a migration decision, not a text edit.
3. Drift gate — only if the project has BOTH a canonical human-authored schema/data-model doc AND a migration tool: if persistence is already built and no schema⇄spec CI drift gate exists, add one per create-prompt-packs' "Schema/Spec Consistency Guardrails" — parse the real migration format and the real spec-doc conventions, backward-drift hard-fail, carve-outs for framework-managed tables, wired into the project's main check target, and **proven red-then-green** (plant a throwaway drift, watch it fail, remove it, watch it pass). No canonical schema doc → skip and say so.

## Phase 2 — Spec red-team + backlog machinery

1. Retroactive spec red-team (create-prompt-packs STEP 1.5): delegate to a clean-context subagent whose ONLY input is the spec files, hunting contradictions, missing lifecycles, unstated assumptions, ambiguous ownership, underspecified integrations, and silent day-one gaps. Triage: **blocking** (would change decomposition or the data model) → batch ALL into one numbered question message to the user and stop that thread until answered; **fixable-in-place** → edit the spec and list the edit; **minor** → backlog entry or an assumption note in the affected UNBUILT pack's §2.
2. Ensure `docs/dev/backlog.md` exists (template: global verify-wiring command, Phase 2.6).
3. Ensure the mandatory final backlog-sweep pack exists (`NN_HARDENING_BACKLOG_SWEEP.prompt.md`, numbered after the last feature pack — never renumber existing packs), shaped per create-prompt-packs' "Mandatory final milestone": §7 names the backlog `## Open` as the work queue, the context budget is waived, triage dispositions are build-now / won't-build-with-reason / re-defer-with-named-home, and it includes the ideas-ledger user-triage table (Phase 3). Add it to the README execution order.

## Phase 3 — Coverage gates, ideas ledger, UI depth

1. Create `docs/dev/ideas.md` if missing (template: global build command) — capture-don't-build; **backlog = spec'd-but-deferred obligations vs ideas = unspecced suggestions, user-triaged**; never build directly from it.
2. Add the capture-don't-build rule to this project's `_PREAMBLE.prompt.md` §15 (Overbuild Prevention) if absent.
3. Run **`/verify-coverage`**: it builds the spec-side capability inventory (including the mandatory cross-cutting sweep — auth screens, password/account lifecycle, session management, shell/nav, search, notifications, error/empty states, settings, audit logs, permissions admin, public-endpoint rate limiting, backups/DR, observability, legal/consent), resolves ownership, **walks every §5 deferral chain to its terminus** (a "→ M[N]" only counts if M[N] actually accepts the scope), adversarially re-checks every orphan claim, writes `_COVERAGE_REGISTRY.md`, and backlogs confirmed orphans. Expect orphans to cluster in cross-cutting/NFR capabilities and broken deferral chains — that is the class this gate exists to catch.
4. UI depth retrofit — **UNBUILT UI packs only**, per create-prompt-packs' "UI Depth Guardrails": (a) transcribe (or cite as binding) the spec's page-definition-matrix row for each page the pack delivers — verbatim columns and actions, no summarizing; (b) add a 3–6 line persona pass derived from the spec's target-users/personas section (who / top jobs on this screen / affordances those jobs need); (c) state that the list-view baseline applies (search, status + top-facet filters, sort, pagination, row count, empty state, export), any omission as an explicit §5 non-goal with a home. Keep each pack within its line budget — trim prose, not tables. For BUILT UI pages, run a quick baseline sweep (does each built list page meet the baseline?) and backlog the gaps — do not edit built packs.
5. If the spec has a page-definition matrix whose intro lacks a list-page baseline paragraph, add one there (fix-in-place spec edit, recorded in the report).
6. Update the prompt-packs README (registries, sweep pack, `/verify-coverage` cadence) and CLAUDE.md's build-pipeline section (coverage-registry law, ideas-vs-backlog distinction, sweep pack) — merge with what's there, never duplicate.

## Phase 4 — Report + commit

- Granular conventional commits, one logical change each (schema registry; red-team spec edits; sweep pack; ideas ledger + preamble; coverage registry + backlogged orphans; UI pack upgrades batched sensibly). Process/docs changes get no CHANGELOG entry unless this project logs them.
- Final report: per phase — created / updated / skipped-as-already-present; built-pack findings routed to the backlog (count + ids); orphan clusters from `/verify-coverage`; blocking spec questions awaiting answers; recommended next step (orphan clusters big enough to be coherent packs are better served by a `/create-prompt-packs` extension run than by the sweep pack alone).
- Capture lessons per the project's self-improvement loop.

## Rules

1. **Adversarially verify before reporting any collision/orphan/drift finding** — first-pass claims run a high false-positive rate; a finding you can't re-confirm with a second grep spelling + a pack read is noise.
2. **Never regenerate packs wholesale; never renumber.** The retrofit is surgical.
3. **Built work is history.** Its gaps are backlog entries with candidate homes, not retroactive pack edits.
4. **Ask the user only for blocking decisions, batched once.** Everything else proceeds with recorded assumptions.
