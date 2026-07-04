---
description: Build every prompt pack end-to-end with the full verify-build → verify-wiring → review-externally pipeline, fixing gaps autonomously. One pack per session — all four phases run in a single turn. Halts only at pack boundaries.
argument-hint: "[--reset] [--status] [--from <pack-id>] [--only <pack-id>] [--skip-questions]"
allowed-tools: "*"
---

# Build → Verify → Review (orchestrator)

Walk through every prompt pack in `docs/prompt-packs/` (or `prompts/`, `.prompts/` — same search order as `/build`). For each pack run the four-phase pipeline:

1. `/build <pack>` — build all deliverables
2. `/verify-build <pack>` — completeness audit; fix any gaps it finds
3. `/verify-wiring <pack>` — connectivity audit; fix any gaps it finds
4. `/review-externally <pack>` — external code review; fix any findings it surfaces

Persist progress to `.claude/.bvr-state.json` in the project root so a fresh session picks up exactly where the prior run left off.

## Inputs

- (no args) — start a new run, or resume an interrupted one if state exists
- `--reset` — delete the state file and start from the first pack
- `--status` — print the state file and exit without doing any work
- `--from <pack-id>` — start (or resume) from this pack, skipping earlier ones (e.g. `--from 04a`)
- `--only <pack-id>` — run the full pipeline for one pack and stop
- `--skip-questions` — skip the upfront clarifying-questions sweep for this session and go straight to building. By default the sweep runs at the start of **every** pack's session (see Phase A.5).

## Session boundaries (designed flow)

**One pack per session.** All four phases (`build` → `verify-build` → `verify-wiring` → `review-externally`) run in a single turn for the current pack. The orchestrator never halts between phases within a pack — it pushes through to `pack-complete`.

The only designed halt is **at the pack boundary**: after `review-externally` passes and the pack completes, end the turn. The user starts the next session manually to process the next pack.

Rationale: a pack's worth of work (4 phases, including Codex's multi-round review iteration) is the unit of work that fits cleanly inside one session's context budget while keeping responsiveness high. Splitting mid-pack creates resume-state complexity and risks Codex losing the build's working set; bundling multiple packs into one turn risks auto-compaction mid-pipeline.

Legitimate halts:

1. **Pack complete** (after `review-externally` passes) — primary designed boundary. End the turn cleanly.
2. **Hard halt** (`halted` set) — a real technical problem that needs the user. Loud message, state preserved for inspection.
3. **Real technical decision** — a substantive question the user must answer (e.g. conflicting spec interpretations). Ask the question directly. Never ask "continue?" / "proceed?".

## State file

Path: `.claude/.bvr-state.json`. Shape:

```json
{
  "completedPacks": ["01_PROJECT_BOOTSTRAP", "02_SITE_CHROME"],
  "currentPack": "03_LEAD_CAPTURE",
  "currentPhase": "verify-wiring",
  "phaseAttempts": 2,
  "lastUpdated": "2026-05-20T09:14:00Z",
  "history": [
    { "pack": "01_PROJECT_BOOTSTRAP", "phase": "build", "attempt": 1, "outcome": "passed", "at": "..." }
  ],
  "halted": null
}
```

- `completedPacks` — pack IDs that finished all four phases cleanly. Replaces the old positional `currentPackIndex` so that newly-added packs (e.g. 31–40 dropped on disk weeks after the original run) are picked up automatically without index drift.
- `currentPack` — the pack ID currently being processed, or `null` between packs.
- `currentPhase` ∈ `build`, `verify-build`, `verify-wiring`, `review-externally`. (No `done` — the state file doesn't claim "all packs done"; that's derived from `pending = enumerated_packs − completedPacks`.)
- `phaseAttempts` — number of times the current phase has been entered for the current pack. Observational only — no longer a cap. Resets to 0 when the phase or pack advances. Useful for spotting runaway loops in `--status`.
- `history` — append-only log of phase outcomes.
- `halted` — `null` while running; an object `{ "reason", "pack", "phase" }` when stopped for a real technical problem (not a designed boundary).

## Autonomy contract

This command exists so the user does **not** have to babysit four sequential slash commands. Honor that:

- **Never ask the user to type "continue", "ok", "proceed", or "yes" to move between phases.** Phase advances are automatic. If you find yourself about to write "Let me know if you want me to continue" — don't.
- **Never pause for confirmation between `/build`, `/verify-build`, `/verify-wiring`, `/review-externally`.** The whole point of this orchestrator is to remove those handoffs.
- **Only stop for user input when a technical decision genuinely cannot be safely assumed** — e.g. a verifier surfaces a finding whose fix would conflict with an explicit spec requirement and either interpretation is plausible; credentials are missing for an external service; a destructive action (data loss, force-push, secret rotation) would be needed. Cosmetic choices, naming, refactor scope, etc. are NOT this — make a reasonable call and move on.
- **Halt only at the pack boundary.** Within a pack, never halt — push through `build` → `verify-build` → `verify-wiring` → `review-externally` in a single turn. The only legitimate halts are: (a) pack complete (after `review-externally` passes); (b) hard halt (`halted` set); (c) a real technical decision question for the user.
- **Halts may ask a real technical question, but never a "continue?" question.** If a halt needs the user to make a substantive decision (which of two architectural paths to take, how to resolve a spec conflict), ask that decision directly and wait. Never convert a halt into "should I keep trying?" / "want me to proceed?" / "continue?".

## Process

### Phase A: Initialize and resume

1. If `--status`: read the state file, print it as a table, exit without touching state.
2. If `--reset`: delete `.claude/.bvr-state.json`.
3. If state file exists and `halted` is set: print the halt reason and stop. The user must investigate, then either re-run with `--reset` or manually clear `halted` in the JSON.
4. **Enumerate prompt packs from disk every session** (not just the first run):
   - Search the same locations `/build` searches: `docs/prompt-packs/*.prompt.md`, `docs/prompts/*.prompt.md`, `prompts/*.prompt.md`, `.prompts/*.prompt.md`. Use whichever directory has matches.
   - Exclude any file starting with `_` (preamble) or `00_` (runbook enforcer is typically a meta-spec, not a buildable pack — confirm by skimming its first 30 lines; if it actually has buildable deliverables — a populated Section 7 with file paths — include it).
   - Sort lexicographically (the `NN_` / `NNa_` prefixes give correct order).
   - Strip the `.prompt.md` suffix to get the pack ID list.
   - This is the **canonical pack list** for this session. The state file does NOT cache it — pack IDs that landed on disk between sessions (e.g. user ran `/create-prompt-packs` last week to add 31–40) are picked up automatically.
5. Compute `pending = enumerated_packs − completedPacks`. Preserve disk order. The first element is `currentPack`.
6. If `pending` is empty:
   - Print: `No pending packs (N completed; 0 pending). Exiting.`
   - Save and exit.
7. If `--from <id>`: drop everything in `pending` before `<id>`. Treat the rest as the work queue.
8. If `--only <id>`: replace `pending` with just `<id>`. Record an internal flag to halt cleanly after that pack's `pack-complete`.
9. If state exists and was mid-pack (`currentPack` set, `currentPhase` set, `halted` null): resume that exact (pack, phase). Otherwise start at `currentPack = pending[0]`, `currentPhase = "build"`, `phaseAttempts = 0`.

Print the resume banner:

```
## build-verify-review — running
Pack:    03_LEAD_CAPTURE (3 of 99 enumerated; 2 completed, 96 pending after this)
Phase:   verify-wiring
```

If `docs/prompt-packs/_COVERAGE_REGISTRY.md` does not exist, add one warning line to the banner: `⚠ No _COVERAGE_REGISTRY.md — spec→pack coverage has never been audited; features may exist in the specs that NO pack builds (run /verify-coverage).` Do not halt — warn and continue.

### Phase A.5: Upfront clarifying-questions sweep

**Run this by default at the start of every pack's session** — not just the first run. Because the designed flow is one-pack-per-session (the user manually starts each pack's session), a leftover `.bvr-state.json` from the prior pack is **not** a reason to skip the sweep. Asking clarifying questions up front, before any building, is the point of this phase: it surfaces ambiguity early and lets the user and the model align before code is written.

Scope the sweep to **the pack about to be built** (`currentPack`), plus project context. Do not re-interrogate the whole pending queue every session — questions about pack 40 are noise when you're about to build pack 4; they'll be asked when pack 40's session starts.

**Skip the sweep only when:**
- `--skip-questions` was passed (explicit opt-out for this session).
- `--status` (no work is done at all).
- `currentPack` already has answers recorded for it in `.claude/.bvr-answers.md` (the questions were asked in an earlier, interrupted session for this same pack — don't re-ask).

Note: a leftover state file / `currentPack` being set is **no longer** a skip condition. The sweep runs on resume too, as long as the conditions above don't apply.

1. Read `currentPack`'s pack file end-to-end. Also read project-level context: `CLAUDE.md`, `docs/PRD*.md`, `docs/ROADMAP*.md`, `docs/AGENTS.md` if present.
2. Collect every question that would require user judgment to answer safely. Examples worth asking: ambiguous brand/voice direction, missing API keys, choice between two equally valid architectural paths the spec leaves open. Examples NOT worth asking: file naming, code style, test framework selection when the project already uses one.
3. If zero questions, print `No upfront questions for <currentPack> — starting build.` and proceed to Phase B.
4. If there are questions, present them as a single grouped message — numbered, with the pack ID and a one-line reason. Wait for the user's answers, then append them to `.claude/.bvr-answers.md` (markdown, one section per pack, one Q&A per sub-section) so they survive context resets. Reference this file from each `/build` invocation.
5. After answers are in, proceed to Phase B without further confirmation.

### Phase B: Main loop

This loop processes **exactly one pack per turn**: all four phases run sequentially, then halt at `pack-complete`. There is no within-turn pack chaining.

While `currentPack` is set:

1. Increment `phaseAttempts`, save state.

2. **Run the current phase as a slash command.** Invoke it via the Skill tool, passing `currentPack` as the argument:
   - `build` → `/build <pack>` (use `--resume` if `phaseAttempts > 1`, since prior attempts may have committed partial work)
   - `verify-build` → `/verify-build <pack>`
   - `verify-wiring` → `/verify-wiring <pack>`
   - `review-externally` → `/review-externally <pack>`

3. **Interpret the result:**
   - **Passed**: append a `passed` history entry, advance to the next phase (or to `pack-complete` after `review-externally`), reset `phaseAttempts` to 0, save state, continue. There is NO attempt cap — convergence happens when the phase reports clean.
   - **Findings exist**: append a `findings` history entry summarizing them. Fix the findings in this same turn — read the verifier's report, apply edits, run the project's lint+build commands to confirm no regressions, commit each logical fix as its own commit (per project CLAUDE.md commit conventions). Do NOT advance the phase. Loop back to step 1 to re-run the same phase. Keep iterating until clean. When a finding touches a classification/invariant/state-machine, re-derive the full decision table and fix every case in that one iteration rather than patching only the flagged case — this is what stops the refinement-of-refinement loop. `phaseAttempts` is observational only — its growing value is a signal for the user (visible in the per-iteration log line) to manually intervene if the loop looks runaway.

     **What counts as a finding (per phase):**
     - `verify-wiring`: ONLY the "Gaps Found" count (Checks A–F). "Spec Gap Questions" (Checks G–L) are advisory by design — never fix them, never loop on them. `/verify-wiring` already persists them to `docs/dev/backlog.md` (its Phase 2.6); collect them and ALSO surface them in the pack-complete summary for the user to review between sessions. A verify-wiring run with 0 A–F gaps and 5 G–L questions is **passed**. Writing the backlog never affects the gap count or the loop decision.
     - `review-externally`: ONLY legitimate findings fixed this round. Dismissed findings (false positives, style preferences) AND backlogged findings (real, but owned by a later pack — written to `docs/dev/backlog.md`) do not trigger a retry — a run reporting "3 total (0 legitimate, 2 dismissed, 1 backlogged)" is **passed**.
     - `verify-build`: all reported gaps count, including failure-mode (Section 11) test gaps — those are build failures, not warnings.
   - **Environmental error** (skill couldn't find the pack, build broken in a way unrelated to the current pack, etc.): set `halted = { reason, pack, phase }`, save state, exit loudly per the "Halting is loud" rule.

4. **Pack-complete halt:** when phase advances past `review-externally`:
   - Append `pack-complete` to history.
   - Add `currentPack` to `completedPacks`.
   - **Reconcile `docs/dev/backlog.md`:** if this just-completed pack resolved any item open in the backlog (it built what an entry was waiting for), move that entry to `## Done` (or delete it) with the resolving pack noted. This is the only backlog write the orchestrator itself makes; new items are written by the phases (`/verify-wiring` Phase 2.6, `/build` pre-flight, `/review-externally` triage).
   - Clear `currentPack = null`, `currentPhase = null`, `phaseAttempts = 0`, save state.
   - If `--only` was set and this was its pack, halt cleanly with "only-mode complete".
   - Recompute `pending`. If empty, fall through to Phase C.
   - Otherwise halt cleanly — the next pack starts in a new session.

5. **Clean-halt actions** (any designed boundary):
   - Save state.
   - Print a one-line summary: e.g. `Pack 03_LEAD_CAPTURE complete (build + verify-build + verify-wiring + review-externally). Next: 04_FOO_BAR. Resume with /build-verify-review.`
   - If verify-wiring surfaced Spec Gap Questions (Checks G–L), list them below the summary line for the user to review before the next session — these were deliberately not auto-fixed. Note that they are recorded in `docs/dev/backlog.md` (with their ids) so they persist beyond this summary.
   - If any phase appended entries to `docs/dev/ideas.md` during this pack, list them (one line each) below the summary — ideas await the user's product decision at exactly this boundary; surfacing them here is what makes capture-instead-of-discard worth the discipline.
   - Exit.

### Phase C: All packs complete

When `pending` is empty after recomputation:
- Print a final summary: total packs in `completedPacks`, commits added during this run (`git log --since=...` filtered), any halts in history.
- Save and exit.

## Rules

1. **One state save per state transition.** Save after: starting a phase, finishing a phase, advancing a pack, halting. Never let in-memory state diverge from the file — a kill -9 mid-step should still leave the orchestrator able to resume sensibly.

2. **No attempt cap.** Phases stop only when they report clean. `phaseAttempts` is observational — track it, print it, but never use it to halt. A long convergence loop is information, not a failure. The user has visibility (per-iteration log lines) and can intervene manually if a loop looks runaway.

3. **Fix findings in the orchestrator turn, then re-run the verifier.** Do not invoke the verifier twice in a row hoping for a different result without changing code in between.

4. **Commit per fix, not per pack.** Every fix lands as its own commit with Conventional Commits format. The project's CLAUDE.md governs scope vocabulary.

5. **Respect CLAUDE.md and the spec.** If a verifier suggests a fix that conflicts with CLAUDE.md or the prompt pack's non-goals, do NOT apply it. Record the conflict in `history` with outcome `acknowledged-conflict` and treat the verifier finding as resolved.

6. **Halting is loud only for real problems.** When you halt for a designed boundary, the message is one terse line. When you halt for `halted = {...}` (real failure), print the pack, phase, attempt count, the verifier's last report verbatim, and the suggested next step.

7. **Never edit `.bvr-state.json` from outside the JSON shape above.** Don't add ad-hoc fields. Don't delete history. Append-only for `history`.

8. **Pack list re-derives every session.** Never cache `packs` in the state file. The canonical list is always whatever's on disk at session start, intersected with the work-tracking via `completedPacks`. Newly-added packs flow in automatically.

## Resume semantics

A session ends in one of three ways:

1. **Clean designed-boundary halt** — next session picks up at `currentPack` / `currentPhase`. Run `/build-verify-review` to continue.
2. **Hard halt** (`halted` set) — the user must investigate and either `--reset` or clear `halted` manually in the JSON.
3. **Crash / context exhaustion mid-phase** — on resume, `phaseAttempts` was incremented BEFORE the phase ran, so the interrupted attempt is reflected in the count.

For the `build` phase specifically, pass `--resume` when re-entering so `/build` reads the todo list and git log to skip already-committed steps.

## Output

While running, after each phase completion print one line:

```
[03_LEAD_CAPTURE] verify-wiring attempt 2 → passed
```

After each fix iteration print:

```
[03_LEAD_CAPTURE] verify-wiring attempt 1 → 3 findings, applied 3 fixes, 2 commits, retrying
```

After every 5 attempts on the same phase, also print a runaway-watch line:

```
[03_LEAD_CAPTURE] verify-wiring on attempt 5 — convergence is slow. /build-verify-review --status to inspect; kill this session if it looks runaway.
```

On pack-complete halt, one line:

```
Pack 03_LEAD_CAPTURE complete (build → verify-build → verify-wiring → review-externally). Next: 04_FOO_BAR. Resume with /build-verify-review when ready.
```

On hard halt, a multi-line block with the halt reason, pack, phase, and verifier output.
