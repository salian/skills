---
description: Execute a prompt pack — build all features specified in a milestone. Reads the prompt pack, extracts the execution plan, builds each step with tests, and commits incrementally. Resume-safe if context runs out.
argument-hint: "[prompt-pack ID e.g. M12] [--resume | --step N | --plan-only]"
allowed-tools: "*"
---

# Build Prompt Pack

Execute a prompt pack end-to-end: read the spec, build every deliverable, write tests, commit incrementally, and hand off to the verification pipeline.

## Inputs

- `$ARGUMENTS` — the prompt-pack identifier (e.g. `M12`). Required. If missing, ask the user.
- Flags:
  - `--plan-only` — extract and display the execution plan without building anything
  - `--resume` — pick up where a previous run left off (reads progress from the todo list and git log)
  - `--step N` — build only step N from the execution plan
  - `--from N` — start from step N (build N and all subsequent steps)

## Process

### Phase 1: Load context

#### 1a. Find and read the prompt pack

**Normalize the ID before globbing.** Milestone IDs are written `M12` / `M7a`, but pack files are named by zero-padded numeric prefix (`12_LEAD_CAPTURE.prompt.md`, `07a_TESTING_CORE.prompt.md`). Strip a leading `M`/`m` and left-pad the number to two digits (`M12` → `12`, `M7a` → `07a`), then glob for that prefix. Both forms are accepted as `$ARGUMENTS`.

```
docs/prompt-packs/<NN>*
docs/prompts/<NN>*
prompts/<NN>*
.prompts/<NN>*
```

If the glob matches multiple packs (e.g. `12` matching both `12a_` and `12b_`), list the matches and ask which one — never guess.

Read the full prompt pack file.

#### 1b. Read the preamble (if it exists)

Check for a shared preamble file in the same directory (`_PREAMBLE.prompt.md` or similar). If present, read it — it contains shared procedures for Run → Observe → Fix, commit discipline, overbuild prevention, and verification.

#### 1c. Read referenced specs

Read every spec file cited in the prompt pack's "Spec References" section. These contain detailed requirements that the prompt pack summarizes.

#### 1d. Read CLAUDE.md

Load project conventions: testing, commit style, auth patterns, component library, middleware rules. These override any conflicting instructions in the prompt pack.

#### 1e. Read context snapshot

The prompt pack's "Context Snapshot" section describes the expected repo state (prior milestones completed, existing files). Verify the repo matches:

```bash
# Quick check: do the expected files/directories exist?
ls -d src/lib/expected-module/ src/app/expected-route/ 2>/dev/null
```

If the repo state doesn't match (e.g. a prior milestone is missing), warn the user before proceeding.

### Phase 2: Extract the execution plan

Build a numbered step list from **Section 7 (Required Deliverables)**, grouped by logical unit — one group per expected commit (packs budget ≤ 8 commits per pack). Note: Section 14 (Commit Discipline) lives in the shared `_PREAMBLE.prompt.md` and contains *generic* commit rules, not a per-milestone commit list — do not look for a step list there. If a pack does inline an explicit commit plan, use it verbatim.

For each step, extract:
- **Step number**: sequential
- **Commit message**: Conventional Commits format, derived from the step's scope
- **Deliverable files**: from Section 7, matched to this step's scope
- **Tests**: from Section 10, matched to this step's scope (including any state-matrix test file)
- **Wiring requirements**: from Section 8, matched to this step's deliverables
- **Failure-mode obligations**: from Section 11, matched to this step's deliverables — each row's mitigation (guard function/config) AND its named test file belong to the step that delivers the code they protect
- **Acceptance criteria touched**: from Section 12, which criteria this step contributes to

Display the plan:

```
## Execution Plan: M12 — Lead Intelligence

| Step | Commit | Files | Tests |
|------|--------|-------|-------|
| 1 | feat(lead-capture): implement automated lead scoring with 8 parameters | src/lib/leads/scoring.ts | scoring.test.ts |
| 2 | feat(lead-capture): implement per-org scoring config | src/lib/leads/scoring-config.ts | scoring-config.test.ts |
| ... | ... | ... | ... |

Total: N steps, ~M files, ~K test files
```

If `--plan-only` was passed, stop here.

Set up progress tracking with TodoWrite — one todo per step, all `pending`.

### Phase 2.5: Pre-flight — what else does this touch?

Before writing any code, surface the surrounding surface area the spec implicitly touches. The goal is to catch integration points the spec mentions in passing but doesn't enumerate as deliverables — workers that need a new handler registered, deploy/infra that needs an env var, LLM call sites that need a new prompt, schedulers, webhooks, feature flags, etc. **Every hit becomes a checklist item that the build must explicitly satisfy OR explicitly defer with a written reason.**

#### 2.5a. Extract surface-area verbs from the spec

Skim the prompt pack + referenced specs and pull out every domain verb/noun that implies a touchpoint outside the deliverable file list. Typical vocabulary:

- **Background work:** `worker`, `handler`, `job`, `queue`, `scheduled`, `cron`, `batch`, `registerScheduledJobs`
- **AI/LLM:** `LLM`, `prompt`, `model`, `embedding`, `completion`, `tokens`, `Anthropic`, `OpenAI`
- **External I/O:** `webhook`, `publish`, `subscribe`, `consumer`, `producer`, `connector`
- **Delivery:** `deploy`, `migration`, `env var`, `secret`, `Kamal`, `Docker`, `CI`
- **Auth/access:** `auth`, `session`, `CSRF`, `permission`, `role`, `tenant`, `org`
- **Surfacing:** `dashboard`, `nav`, `link`, `page`, `route`, `API`, `endpoint`
- **Observability:** `log`, `metric`, `event`, `analytics`, `telemetry`
- **State:** `migration`, `schema`, `index`, `seed`, `backfill`

Also scrape the spec for its own bespoke nouns (feature names, table names, module names) — these are the highest-signal terms.

Produce a list of 8-15 verbs/nouns most relevant to *this* spec. Don't run the full vocabulary blindly — pick what the spec actually mentions.

#### 2.5b. Grep the repo for each term

For each term, run a fast scoped grep and note where it lands. Examples:

```bash
# Background work registration
grep -rn 'registerScheduledJobs\|registerHandler' src/jobs/ --include='*.ts'

# Existing LLM call sites the new feature might need to plug into
grep -rn '@anthropic-ai/sdk\|openai\|callLLM\|generateCompletion' src/ --include='*.ts'

# Env vars / secrets surface
grep -rn 'process\.env\.' src/ --include='*.ts' | grep -i '<feature-keyword>'

# Nav / sidebar entries
grep -rn '"/dashboard/' src/app/dashboard/_components/ --include='*.tsx'

# CSRF on state-changing client calls
grep -rn 'csrfHeaders' src/app/ --include='*.tsx'
```

Run these in parallel where possible. The point is breadth, not depth — one or two hits per term is enough to know "this concern already has a home in the repo, and the new code will need to plug into it."

#### 2.5c. Produce the touchpoint checklist

For each grep hit that's plausibly relevant, add a line to a checklist. Format:

```
## Pre-flight touchpoints — M[N]

- [ ] **Worker registration** — new `lead-rescore` handler must be added to `registerScheduledJobs()` in `src/jobs/workers/batch.ts:L42`
- [ ] **LLM prompt catalog** — new scoring prompt should live alongside existing prompts in `src/lib/llm/prompts/` (see `lead-enrichment.ts` for shape)
- [ ] **Env var** — spec mentions `SCORING_MODEL_ID`; needs entry in `.env.example` and `kamal/secrets`
- [ ] **Dashboard nav** — new `/dashboard/leads/analytics` page needs an entry in `src/app/dashboard/_components/sidebar.tsx`
- [ ] **CSRF** — new POST `/api/dashboard/leads/rescore` client caller must use `csrfHeaders()`
- [ ] **Migration journal** — schema change requires `pnpm db:generate`, never hand-edit `_journal.json`
- [x] **Webhooks** — DEFERRED: spec explicitly out-of-scope per Section 5 (M14 will add webhook delivery)
```

Each item is either:
1. **A build-time obligation** — the step that delivers the related file must also satisfy this row (cross-reference into the execution plan).
2. **Explicitly deferred** — with a one-line reason citing the spec section or a future milestone. "Deferred because I forgot" is not acceptable; the deferral must be defensible. **Record every deferred touchpoint in `docs/dev/backlog.md`** (create it if absent, mirroring how `CHANGELOG.md` is maintained) — one entry under `## Open` with what / why-deferred / candidate-home / source `<pack-id> build pre-flight (<date>)`, deduped against existing entries. A touchpoint deferred only in the build's prose evaporates; the backlog is its durable home so a later pack (or the project's hardening pass) can pull it.

#### 2.5d. Reconcile with the execution plan

Walk the checklist and, for each non-deferred row, identify which step in the Phase 2 execution plan will satisfy it. If no step covers it, either:
- Add it to the nearest logical step's scope, or
- Add a new step.

Update the displayed execution plan to show which steps now carry which pre-flight obligations. Persist the checklist as a TodoWrite section ("Pre-flight obligations") separate from the per-step todos, so resume runs can see what remains.

#### 2.5e. Sibling write-path parity

Pre-flight (2.5a–d) finds *outward* touchpoints (what the new code must plug INTO). This check finds *inward* obligations: when the new code **writes to a shared persistent entity** (a model/table/aggregate) or **performs an operation other code already performs** (create a user, enqueue a job, send a message, charge an account), it must apply the **same invariants the existing paths already enforce** — limits, validation, authorization, compliance, normalization, idempotency. These are a leading source of "passes self-review and tests, fails external review" bugs, because they live in *how the entity is always written*, not in the spec's deliverable list.

**Skip this** if the pack only reads/transforms, or introduces a brand-new entity with no existing writers. Otherwise:

1. **Name the shared write(s)/operation(s)** the new path performs (e.g. "creates a `Contact`", "inserts a suppression row", "enqueues a send").
2. **Find the existing performers** — grep for other call sites of the same create/update/enqueue/charge, the shared service, or the table.
3. **Diff their guard sets.** For each existing performer, note which of these it applies, then confirm the new path applies every one that's relevant:
   - **Quota / limit** (plan caps, rate limits)
   - **Input validation** (format, length, required fields)
   - **Authorization / tenancy** (scoping, permission, ownership)
   - **Compliance / suppression / PII** (do-not-contact, erasure, consent, redaction)
   - **Normalization / coercion** (schema constraints, enums/whitelist, required-field defaults)
   - **Idempotency / dedup** (unique keys, replay safety)
   - **Audit / observability** (the log/audit trail peers write)
4. Each gap is a **build obligation** for the relevant step, or an **explicit deferral** recorded in `docs/dev/backlog.md` (same rule as 2.5c).

*Example (stack-agnostic):* a new CSV/CRM/API importer that creates accounts should apply the same plan-limit, validation, dedup, and consent checks the signup form and admin-create path already apply — not a fresh happy-path insert.

This phase is intentionally **cheap and fast** — 5-10 minutes of grepping, not a full audit. The deeper audits happen in `/verify-wiring`. The point of pre-flight is to front-load the "oh, I forgot the worker registration" class of miss so it lands in the build, not in the review.

### Phase 2.6: Invariant sketch — stateful-protocol packs only

**Most packs skip this entirely.** Run it only when the pack implements a **stateful protocol**: a multi-step lifecycle with persisted state that more than one actor mutates (concurrently or over time). Signals in the spec/preamble: *claim, lease, lock, cursor, checkpoint, retry, redispatch, reconcile, dedupe, idempotent, exactly-once, state machine, concurrent, backfill, ledger* — or a Failure-Modes section dominated by concurrency/ordering rows.

When it applies, spend ~10–20 minutes writing the model down **before coding**. For this class of code, discovering edges one-at-a-time during review is the slowest, most expensive convergence path — each local patch tends to spawn the next edge. Sketch:

1. **States & transitions** — the states the entity moves through, and for each transition: who triggers it, under what condition.
2. **Invariants** — what must hold across *every* transition, as one-liners (e.g. "at most one worker owns a run at a time", "this counter resets exactly when a pass starts", "flag X set ⟺ a job is queued").
3. **Adversarial conditions the protocol must survive** — walk each and name the mechanism that upholds the invariants under it:
   - **Concurrency** — two actors at once (overlapping ticks, double-claim)
   - **Crash / kill** — process dies at each step; what's left behind, who reclaims it
   - **Retry / replay** — the same operation runs twice (must no-op or converge)
   - **Underlying change** — the entity is modified/deleted mid-operation, OR state captured at enqueue (membership, eligibility, "already done") is stale by run time
   - **Skew / ordering** — clock skew, out-of-order delivery; a delayed/scheduled job's logical time is its scheduled-fire instant, NOT processing time
   - **Partial failure** — step N commits, N+1 fails
   - **External side-effect** — a non-transactional effect (send / charge / write-API call): a DB check before it can be raced (serialize, or use an idempotency key); record it AFTER it succeeds; decide at-least-once vs at-most-once
   - **Blast radius** — when an operation fails, is it this-item-only or system-wide? The retry / alert / abort response must differ

Keep it terse — a comment block or a short note in the pack's working doc — and persist it so resume/verify can see it. It feeds two later phases: each invariant + adversarial condition becomes a **named test** in Phase 3d, and the chosen mechanisms become what Phase 3c must implement. A gap found here is one deliberate design decision in one place, instead of many review rounds re-deriving the same state machine.

### Phase 3: Execute each step

For each step in the plan (or starting from `--step N` / `--from N`, or resuming from last completed):

#### 3a. Announce the step

Mark the current step as `in_progress` in the todo list.

#### 3b. Read existing code

Before writing anything, read the files that this step's deliverables will depend on or integrate with. Understand the patterns already established. Key things to check:
- How do adjacent modules export/import?
- What's the DB schema for tables this step touches?
- What patterns do existing API routes follow (auth guards, error handling, response shape)?
- What do existing tests mock and how?

#### 3c. Build the deliverables

Write the code for this step's files. Follow these rules strictly:

- **Match existing patterns.** Don't invent new conventions — follow what prior milestones established.
- **Use the design system.** If CLAUDE.md specifies UI components, use them. No raw HTML.
- **Wire as you build.** Don't just create the library function — also add the import and call site in the route/handler that uses it (per Section 8 wiring table). If the caller doesn't exist yet (it's a later step), note it as a wiring TODO.
- **Respect non-goals — but capture, don't discard.** Check Section 5 (Non-Goals) and Section 15 (Overbuild Prevention) before adding any feature. If you're tempted to add something not in the spec, don't build it — and don't throw the thought away either. You are closer to the code than the pack author was; genuine improvements you notice (a better design, a missing affordance a real user would want, a simplification, a risky pattern worth revisiting) go into `docs/dev/ideas.md` as a ≤3-line entry: **what / why it's better / where** (`file:line`), tagged with the pack id, phase, and date. Create the file from the template below if absent (same directory as the project's backlog file). Distinction that keeps both files honest: **backlog.md = spec'd-but-deferred obligations** (will be built, has a home); **ideas.md = unspecced suggestions** (may never be built; accepting one is a product decision for the user). Never build directly from ideas.md, and never self-promote an idea into scope.

  Ideas ledger template (when creating it fresh):

  ```markdown
  # Ideas Ledger

  Unspecced improvement ideas noticed during execution — captured instead of
  discarded by overbuild prevention. Companion to `backlog.md`:
  **backlog = spec'd-but-deferred obligations** (will be built; named home) ·
  **ideas = unspecced suggestions** (maybe never; accepting one is a product
  decision for the USER, at triage — never the executing LLM's call).

  Rules: entries ≤3 lines (what / why it's better / where, with source
  `<pack-id> <phase> (<date>)`); dedupe by the "what"; never build directly
  from this file. Accepted → promote to a spec edit or backlog entry with a
  named home; rejected → `## Closed` with a one-line reason.

  ## Open

  _(none yet)_

  ## Closed

  _(none yet)_
  ```
- **Honor the page contract.** For any UI page deliverable, locate the page's row in the spec's page-definition matrix (if one exists) and treat its listed columns, filters, and actions as acceptance criteria alongside the pack text — the pack may have summarized the row; the row wins. Apply the project's list-view baseline (search/filter/sort/pagination/export) unless the pack explicitly non-goals an element.
- **Handle errors.** Every DB query, API call, and external service interaction needs error handling. Don't leave happy-path-only code.
- **Ship mitigations with the code they protect.** If Section 11 assigns a failure-mode row to this step's deliverables, the guard function/config lands in the same commit as the fetch/worker/pipeline it guards — never as a follow-up step.
- **Discharge pre-flight obligations.** Before marking this step complete, check the Phase 2.5 checklist for any rows assigned to this step. Each must be either satisfied in this commit or moved to a later step with a written reason. A row that silently disappears is a bug.

#### 3d. Write tests

Write the test files specified for this step (Section 10). Follow the project's test conventions:
- Co-locate with source (`foo.ts` → `foo.test.ts`)
- Use the project's test framework (Vitest, Jest, pytest, etc.)
- Mock external dependencies (DB, APIs, queues) per existing patterns — EXCEPT state-matrix tests (Section 10), which must hit the real route handler with a real/test DB per the pack's matrix rules; mocks there defeat the purpose
- Cover the scenarios listed in the spec, not just the happy path
- For state-matrix tests: one assertion per cell via `describe.each`/`it.each`, every dimension seeded in `beforeEach`, skipped cells justified in a comment
- For a **stateful-protocol pack** (Phase 2.6), turn each invariant + adversarial condition from the sketch into a named **interleaving** test that drives the hazard directly — two concurrent actors, crash-then-resume, double-fire, stale re-read. These are DISTINCT from state-matrix tests: a matrix asserts per-*state* outputs; an interleaving test asserts the invariant holds across *concurrent or repeated* operations. A matrix alone never exercises a race.

#### 3e. Quality gate

Run the quality checks. ALL must pass before committing:

```bash
make lint       # or the project's lint command
make typecheck  # or tsc --noEmit / mypy / equivalent
make test       # run the full test suite, not just new tests
```

If any fail, fix the issues. Re-run until green. Do NOT commit with failures.

#### 3f. Commit

Stage only the files relevant to this step. Commit with the message from the execution plan:

```bash
git add [specific files]
git commit -m "commit message from plan"
```

Update CHANGELOG.md if the change is user-visible. Commit the changelog alongside or immediately after.

#### 3g. Mark complete

Mark this step as `completed` in the todo list. Move to the next step.

### Phase 4: Post-build

After all steps are complete:

#### 4a. Final quality gate

Run the full suite one more time to catch cross-step regressions:

```bash
make lint && make typecheck && make test
```

#### 4b. Wiring self-check

Walk the Section 8 wiring checklist. For each row, grep to confirm the wiring exists. This is a quick pre-flight before the formal `/verify-wiring` audit.

#### 4c. Hand off to verification pipeline

Print the verification reminder:

```
## Build complete: M[N]

All N steps executed. N commits created. Tests passing.
Ideas captured to docs/dev/ideas.md this build: [list ids/titles, or "none"]

Run the verification pipeline:

1. /verify-build M[N]    — completeness: was everything in the spec built?
   Fix any gaps, then:
2. /verify-wiring M[N]   — connectivity: is everything built actually reachable?
   Fix any gaps, then:
3. /review-externally M[N]    — code quality: bugs, logic errors, security issues?
   Fix any findings.
```

### Phase R: Resume (when `--resume` is passed)

If the previous run was interrupted (context exhaustion, user break, crash):

1. Read the todo list for any `in_progress` or `pending` steps
2. Check git log to confirm which steps were already committed
3. Reconcile: mark committed steps as `completed`, find the first incomplete step
4. Display status:

```
## Resuming M12 build

Steps completed: 1-7 (committed)
Current step: 8 — feat(backlinks): integrate paid API for real DA scores
Remaining: 8-12
```

5. Re-run Phase 2.5 (pre-flight) if the checklist wasn't persisted to the todo list, or load the existing "Pre-flight obligations" todos. Reconcile against commits already made — mark satisfied rows complete.
6. Continue from the first incomplete step using Phase 3.

## Rules

1. **One logical change per commit.** Never bundle multiple steps into one commit. Each step = one commit. This makes git log match the execution plan and makes `/review-externally M12` work correctly.

2. **Tests ship with the code they test.** Don't defer all tests to the end. If the execution plan has a dedicated test step, that's for additional/integration tests — unit tests should land alongside their implementation step.

3. **Don't fight the spec.** If the prompt pack says to implement something a certain way, do it that way — even if you think there's a better approach. The spec was designed holistically. Deviations create wiring mismatches downstream.

4. **Wire as you go.** Every library function should have at least one caller by the time its step is committed. Don't create orphaned code and plan to wire it later — "later" often doesn't happen.

5. **Check overbuild prevention before every step.** Re-read Section 5 (Non-Goals) and Section 15 (Overbuild Prevention). It's easy to drift into implementing features from future milestones, especially when the current step's code naturally suggests them.

6. **Proportionality: does this deliverable's size match its spec line?** Before committing a step, compare what you built to what its §7 text actually asks for. An order-of-magnitude overshoot means STOP and ask the user — do not ship it and let external review sort it out. Ask first whether a well-tested library (ideally one §6 already lists) does this; prefer the boring dependency to a bespoke reimplementation you then own. This is the overbuild that §5 does NOT catch: not the wrong feature, but the right feature built far too elaborately — invisible from the inside, because each increment looks justified. (Observed: a one-line "Pino logger with redaction paths" deliverable shipped as ~700 lines of bespoke redaction engine and then absorbed ~30 review rounds.)

6. **Context awareness.** If you're running low on context (many steps completed, large codebase), tell the user which step you're on and suggest resuming with `/build M12 --resume` in a fresh session. Don't degrade quality by rushing through remaining steps.

7. **Read before you write.** Phase 3b (read existing code) is not optional. The #1 cause of inconsistent implementations is not reading adjacent code before writing new code.

8. **Respect CLAUDE.md over prompt pack.** If CLAUDE.md and the prompt pack conflict (e.g., on auth patterns, component usage, test conventions), CLAUDE.md wins — it reflects the actual state of the project.
