---
description: Add a Self-Improvement Loop to this project's CLAUDE.md — automatic learning behaviors, retrospectives, decision logging, and consistent commit/changelog/backlog practices. Idempotent; merges with existing sections rather than duplicating.
allowed-tools: Read, Glob, Grep, Write, Edit, Bash(ls:*), Bash(mkdir:*), Bash(touch:*), Bash(git ls-files:*), Bash(git status:*), Bash(git check-ignore:*)
---

Add a Self-Improvement Loop to this project's CLAUDE.md. This enriches any project with automatic learning behaviors, retrospectives, decision logging, and consistent commit/changelog/backlog practices.

## Instructions

1. Read the project's `CLAUDE.md` file in the current working directory. If no `CLAUDE.md` exists, create one with a basic project header first, then add the sections below.

1a. **Check which referenced commands actually exist** before writing the template. The template references `/learn`, `/manage-skill`, and `/retro`. List `~/.claude/commands/`, `.claude/commands/`, and the available-skills list; for each command that does NOT exist, replace its invocation in the template with the inline fallback behavior (described per-section below) so the enriched CLAUDE.md never instructs a future session to invoke a command that isn't there.

2. Check which of the following sections already exist in the file. Look for these heading patterns:
   - `## Self-Improvement Loop`
   - `### When to capture knowledge`
   - `### When to create or update skills`
   - `### When to run a retrospective`
   - `### When to update this file`
   - `### Decision log`
   - `## Automatic Behaviors`
   - `### Changelog`
   - `### Backlog`
   - `### Ideas Ledger`
   - `### Process Notes`
   - `### Commits`
   - `### Makefile`
   - `## Phase Workflow`
   - `### Post-pipeline maintenance changes`

3. For sections that are **missing**, append them to the end of CLAUDE.md (before any trailing `---` or open items section if one exists). For sections that **already exist**, compare the content and merge any missing bullet points or subsections — do not duplicate content that's already there.

4. After adding the sections, also ensure that `docs/decisions/` directory exists. If it doesn't, create it with a `.gitkeep` file. If the `/retro` fallback was used (step 1a), also create `docs/retros/` with a `.gitkeep`.

5. If a `CHANGELOG.md` doesn't exist yet, create one with the Keep a Changelog format header.

5a. If the project uses prompt packs (a `docs/prompt-packs/` directory exists) and the backlog file doesn't exist, create it with a minimal header — `# Deferred-Items Backlog`, a one-line purpose, and empty `## Open` / `## Done` sections. Use the path chosen per the Backlog adaptation rule below (default `docs/dev/backlog.md`). In projects WITHOUT prompt packs, don't create the file — there are no automatic writers, so let it appear lazily if the team starts a manual one.

5b. Same rule for the ideas ledger: if the project uses prompt packs and `docs/dev/ideas.md` (sibling path to the backlog) doesn't exist, create it with the contract header — a one-line purpose ("unspecced improvement ideas captured instead of discarded by overbuild prevention"), the backlog-vs-ideas distinction table (backlog = spec'd-but-deferred obligations, pack-owned; ideas = unspecced suggestions, **user**-triaged), the ≤3-line entry rule (what / why better / where, with source), the never-build-directly-from-this-file rule, and empty `## Open` / `## Closed` sections. Skip in non-prompt-pack projects.

5c. Create the process-notes ledger regardless of whether the project uses prompt packs: if `docs/dev/process-notes.md` (sibling path to the backlog) doesn't exist, create it with the contract header — a one-line purpose ("workflow/tooling papercuts captured for later fixing, prioritized by repeat frequency"), the entry rule (what / friction / which command or step / source, with a `[×N]` repeat-count marker), the dedup-and-increment rule (recurrences bump the count and append the date, never file a duplicate), the never-fix-inline rule, and empty `## Open` / `## Closed` sections. Unlike backlog and ideas, process notes has a writer in every project (any session that hits workflow friction), so it is not gated on prompt packs — adapt only the path to the project's docs layout.

6. **Verify `.gitignore` enforcement** (the never-commit concern belongs in tooling, not prose). Check that `.gitignore` exists and covers, as applicable to the stack: env/secrets files (`.env*`), OS cruft (`.DS_Store`), dependency dirs (`node_modules/`, `vendor/`, `__pycache__/`), build output, and log directories. Append any missing entries. Then run `git ls-files` against those patterns — if a matching file is already tracked, report it to the user (do NOT `git rm` anything yourself; a tracked `.env` may mean a secret is already in history and needs rotation, which is the user's call).

## Sections to Add

Add the following sections. Adapt project-specific details (like the list of files to never commit) to match what's already in the CLAUDE.md if those conventions are already established:

```markdown
## Self-Improvement Loop

This project maintains a learning system that gets smarter over each session. The following behaviors are **automatic** — do them without being asked.

### When to capture knowledge (`/learn`)

Invoke `/learn` (or save directly to memory/decisions) when any of these happen:

- An approach fails and you pivot — record what didn't work and why
- A library or API behaves unexpectedly — record the gotcha
- The user corrects your approach — record the feedback
- You discover a non-obvious pattern that works well — record it
- A design decision is made during implementation — add an ADR to `docs/decisions/`

### When to create or update skills (`/manage-skill`, or edit `.claude/commands/` directly)

- You perform the same multi-step workflow twice in a session — extract it into a skill
- An existing skill produces suboptimal results — update it
- The user describes a workflow they want repeatable — create a skill for it

### When to run a retrospective (`/retro`, or write a dated entry to `docs/retros/`)

- After completing a phase checklist item
- After completing a significant milestone (even mid-phase)
- When the user says they're done, wrapping up, or stepping away
- At the end of any session where substantial work was done

### When to update this file (CLAUDE.md)

- A new code convention emerges from implementation
- An existing rule proves wrong or unhelpful
- The project structure changes in a way that affects how work is done

### Decision log (`docs/decisions/`)

Record architecture and design decisions as numbered ADRs (`001-title.md`, `002-title.md`). Capture:

- Library/tool choices and why
- Data format decisions
- Patterns adopted or rejected
- Anything where "why did we do it this way?" might come up later

## Automatic Behaviors

### Changelog

After completing any meaningful code change, update `CHANGELOG.md`:

- Entries are date-stamped and chronological (newest date at top) — one entry per user-visible change, added under today's date as the change lands
- Categories within a date: Added, Changed, Fixed, Removed (Keep a Changelog vocabulary)
- Internal refactors, dependency bumps, and CI changes get no entry unless they affect user-visible behavior
- Never retroactively edit past entries

### Backlog

`docs/dev/backlog.md` is the durable home for **deferred-but-real** items — things worth doing that aren't yet scoped into the current unit of work. Maintain it the same way `CHANGELOG.md` is maintained: when work surfaces a deferral, it lands here rather than evaporating into a report, a comment, or a commit body.

- A backlog entry is NOT a bug (fix those now) and NOT a blocker (raise those now). It is real, worth doing, and out of current scope.
- Entries are terse: a short id, title + priority, and `what / why-deferred / candidate-home / source`. Dedup by the "what" — never file the same concern twice.
- When later work resolves an item, move it to a `## Done` section (or delete it). Items already scheduled elsewhere are recorded as resolved-by-design so audits don't re-raise them.
- **If the project uses prompt packs, the build pipeline maintains this automatically:** `/verify-wiring` writes its advisory (Check G–L) findings, `/build` writes explicitly-deferred pre-flight touchpoints, `/review-externally` writes real-but-out-of-scope findings, `/verify-build` writes confirmed out-of-scope-by-spec future needs, and `/build-verify-review` reconciles resolved items at pack-complete. Packs *pull* from this list when built; scope is never *pushed* into a pack speculatively.

### Ideas Ledger

`docs/dev/ideas.md` is the companion to the backlog for **unspecced suggestions** — genuine improvements noticed during execution (a better design, a missing affordance, a simplification) that scope discipline forbids building now. The rule is **capture, don't build — and don't discard**: the executor is closer to the code than the spec author was, and that intelligence is lost unless it has somewhere to land.

- Entries are ≤3 lines: **what / why it's better / where** (`file:line` or page), with source (phase + date). Dedup by the "what".
- The split that keeps both files honest: **backlog = spec'd-but-deferred obligations** (will be built; has or needs a named home) vs **ideas = unspecced suggestions** (may never be built; accepting one is a product decision for the USER).
- Never build directly from this file, and never self-promote an idea into scope. Ideas are triaged by the user (at natural boundaries, or by a hardening/sweep pass); accepted → promoted to a spec edit or backlog entry with a named home; rejected → moved to `## Closed` with a one-line reason.

### Process Notes

`docs/dev/process-notes.md` is the third sibling ledger — the durable home for **workflow/tooling papercuts**: friction in the *process itself* (an ambiguous command instruction, an easy-to-forget step, a pipeline stage that keeps re-prompting, a confusing hand-off between commands), as distinct from product/code work (backlog) or product suggestions (ideas). The rule is **capture, don't fix inline — and don't discard**: a papercut noticed mid-task rarely justifies breaking flow to fix, but it evaporates unless it lands somewhere.

- Entries are terse: **what / friction observed / which command or step / source** (phase + date), prefixed with a `[×N]` repeat-count marker (starts at `[×1]`).
- **Track repeat frequency — it is the prioritization signal.** Dedup by the "what": when the same papercut recurs, do NOT file a second entry — increment `N` and append the new date to that entry. Impact ≈ *per-occurrence cost × frequency*, so a small annoyance hit every session can outrank a painful one seen once. A rising `N` is the cue to promote a fix.
- Never fix a papercut inline just because you noticed it (that is scope creep on the current task). Process notes are triaged at natural boundaries — accepted → becomes a concrete fix (a skill/command edit, or a backlog entry if the fix is larger) and the note moves to `## Closed` referencing the fix; rejected → moved to `## Closed` with a one-line reason.

### Commits

When asked to commit or after completing a task that warrants a commit:

- Commit message style: Conventional Commits format — `type(scope): summary` (e.g., `feat(spintax): add engine with nested support`)
- One logical change per commit; stage only the related files
- Always update CHANGELOG.md before committing
- Sensitive/generated files are excluded via `.gitignore` (enforced, not remembered). If `git status` ever shows a secrets file, env file, or build artifact as tracked or staged, stop and flag it — that means `.gitignore` has a gap or the file was tracked before being ignored; fix the cause, never just skip the file this once

### Makefile

The project uses a `Makefile` as the single entry point for all commands. When adding new scripts, compositions, or workflows, **always update the Makefile** with a corresponding target and help comment.

## Phase Workflow

1. Read the relevant spec in `docs/specs/` before starting work
2. Break work into the checklist items from the spec
3. Each checklist item should produce a visible result
4. Update CHANGELOG.md after each meaningful change
5. Commit at natural milestones
6. Run `/retro` after completing each checklist item or milestone

### Post-pipeline maintenance changes

Once the build pipeline (prompt packs, if used) is complete, ad-hoc changes still follow the full discipline — Conventional Commits, chronological changelog entries, tests shipped with the change, decision log for anything architectural. A change too small for a prompt pack is NOT too small for the discipline. If a change is large enough that it touches multiple modules or adds a feature, consider generating a new prompt pack for it instead of building ad-hoc.
```

## Adaptation Rules

When adding these sections, make smart adaptations based on what's already in the CLAUDE.md:

- **Command availability**: For each of `/learn`, `/manage-skill`, `/retro` that doesn't exist in this environment (checked in step 1a), drop the slash-command mention and keep only the fallback behavior (save to memory/decisions, edit `.claude/commands/` directly, write a dated entry to `docs/retros/`). Never instruct future sessions to invoke a command that isn't installed.
- **Never-commit concern**: enforcement lives in `.gitignore` (step 6), not in CLAUDE.md prose. If the existing CLAUDE.md already carries a never-commit list, fold its entries into `.gitignore` and replace the list with the tracked-file tripwire bullet from the template.
- **Commit style**: If a commit convention already exists, keep the existing one. Only add the commit section if no convention is documented. **Exception**: if the project uses prompt packs (a `docs/prompt-packs/` directory exists), the convention MUST be Conventional Commits — `/review-externally`'s scope detection greps the git log for `type(scope):` patterns, and /build's plan-to-log mapping depends on it. Flag any existing non-conventional convention to the user instead of silently keeping it.
- **Changelog style**: If a changelog convention already exists, merge rather than replace — but preserve the chronological, per-change discipline if the project uses prompt packs (the pack pipeline requires date-stamped per-commit entries).
- **Backlog path & framing**: Default the backlog to `docs/dev/backlog.md`, but adapt to the project's docs layout (the same way the decision-log path adapts): if `docs/` is organized into typed subdirs (`specs/`, `retros/`, etc.), keep it in a dev/working subdir; if `docs/` is flat, `docs/backlog.md` is fine; a repo-root `BACKLOG.md` next to `CHANGELOG.md` is also acceptable. If a backlog already exists at another path, keep that path and only merge missing content. The pipeline commands write to whatever path this CLAUDE.md documents — if you choose a non-default path, the project's `~/.claude/commands` (or `.claude/commands`) pipeline commands must reference the same one, so flag any mismatch. **Drop the "build pipeline maintains this automatically" bullet entirely if the project has no `docs/prompt-packs/` directory** — without the pipeline there are no automatic writers, so frame the Backlog purely as a manual triage list.
- **Ideas Ledger path & framing**: keep it a sibling of the backlog (same directory, `ideas.md`), moving with whatever backlog path was chosen. In non-prompt-pack projects, keep the section (the capture-don't-build rule is valuable everywhere) but drop pipeline-specific mentions and frame triage as "at natural boundaries with the user".
- **Process Notes path & framing**: keep it a sibling of the backlog (same directory, `process-notes.md`), moving with whatever backlog path was chosen. Unlike the backlog and ideas ledger, this section applies to **every** project regardless of prompt-pack use — the writer is any session that hits workflow friction, which every project has. Always include it. Preserve the `[×N]` repeat-count mechanic verbatim; it is the section's whole point (frequency drives prioritization). If the project has no `docs/` at all, a repo-root `PROCESS-NOTES.md` next to `CHANGELOG.md` is acceptable, matching wherever the backlog landed.
- **Makefile section**: Only include if the project actually uses a Makefile. If it uses a different build system (npm scripts, just, etc.), adapt the section accordingly.
- **Phase Workflow**: If the project has specs in a different location than `docs/specs/`, adapt the path. If no spec directory exists, still add the workflow but note the spec path should be updated.
- **Decision log path**: Use `docs/decisions/` unless the project already has an ADR directory elsewhere.

## After Adding

Tell the user what was added and what was already present. If any sections needed adaptation, explain what you changed and why.
