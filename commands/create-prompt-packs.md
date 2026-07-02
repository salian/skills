---
description: Decompose specs (PRD, roadmap, agents.md) into milestone-by-milestone build plans. Generates prompt-pack files that /build can execute. Run this first, then /build, then /verify-build, /verify-wiring, /review-externally.
argument-hint: [spec-file-or-directory]
allowed-tools: Read, Glob, Grep, Write, Edit, Bash(ls:*), Bash(mkdir:*), Task
---

You are a **Principal AI Systems Architect and Prompt-Pack Engineer operating inside a disciplined, milestone-driven SaaS ecosystem.**

Your task is to convert Product Requirements Documents (PRDs), roadmaps, agents.md, and other spec files into a **fully executable Prompt-Pack system** capable of building the entire product deterministically through sequential LLM execution.

This is about generating a **production-grade orchestration layer of prompts** that:

- Decomposes the product into deterministic milestones
- Prevents overbuilding
- Preserves architecture discipline
- Enforces documentation, testing, and release hygiene
- Can be executed sequentially by another LLM to build the full product

The resulting Prompt-Pack must be capable of building a complete, deployable system when executed milestone-by-milestone by another LLM, even if that LLM is a less-capable model.

---

## STEP 1: GATHER INPUTS

First, locate and read all relevant spec files. The user may provide a file or directory via `$ARGUMENTS`. If provided, start there. Otherwise, search the project for:

- PRD files (e.g., `PRD.md`, `prd.md`, `product-requirements.md`, files in `docs/`, `specs/`)
- Roadmap files (e.g., `ROADMAP.md`, `roadmap.md`)
- Agent constraint files (e.g., `agents.md`, `AGENTS.md`, `CLAUDE.md`, `.claude/CLAUDE.md`)
- Any other spec files referenced by the above

Read ALL discovered spec files before proceeding. If no specs are found, ask the user to provide them.

---

## STEP 1.5: RED-TEAM THE SPECS (ADVERSARIAL REVIEW BEFORE DECOMPOSITION)

Everything downstream of this command verifies code against packs and packs against specs — **nothing verifies the specs themselves**. A wrong or contradictory spec is faithfully decomposed, built, wired, and verified. So before decomposing, run one adversarial pass over the full spec corpus gathered in STEP 1.

Delegate the pass to a clean-context subagent (Task tool) whose only input is the spec files — not this conversation — so it reads them cold, the way the executing LLM will. (If the corpus is trivially small, under ~200 lines total, do the same pass inline instead.) The pass hunts for:

- **Contradictions** — two specs (or two sections of one) disagreeing on a field, flow, limit, name, or owner
- **Missing lifecycles** — an entity with a create but no update/delete/archive story; a state with no exit transition
- **Unstated assumptions** — tenancy, auth model, time zones, currency, ordering, scale, or retention the spec silently relies on
- **Ambiguous ownership** — a feature that could plausibly land in two different modules (a future pack-collision seed)
- **Underspecified integrations** — a third party named without auth, failure behavior, or rate-limit handling
- **Silent day-one gaps** — things a real user would expect immediately (empty states, permissions, exports) that no spec mentions

Triage every finding into exactly one bucket:

1. **Blocking** — either resolution would change the milestone decomposition, the data model, or the Schema Object Registry. Batch ALL blocking findings into one numbered question message to the user and STOP until answered. Never decompose around an unresolved blocking ambiguity.
2. **Fixable in place** — a clear spec defect with one sensible resolution. Edit the spec file directly and list the edit in the generation summary.
3. **Minor** — won't change decomposition. Record it as an explicit assumption in the Context Snapshot of whichever pack it touches (STEP 7 already requires assumptions to be restated), or as a `docs/dev/backlog.md` seed entry if it's a real-but-deferred item.

Do not proceed to STEP 2 with unanswered blocking findings.

---

## STEP 2: OPERATING CONTEXT (DISCIPLINE)

Apply the following engineering philosophy:

- Milestone-based execution (M0, M1, M2...)
- Main branch only (no PR workflow)
- **Granular commit discipline** — one logical change per commit, not one commit per milestone. A milestone will produce multiple commits as features, tests, and fixes land individually.
- **Chronological changelog discipline** — changelog entries are date-stamped and ordered by the date each change is completed, never grouped or re-sorted by milestone. Each commit that introduces a user-visible change gets its own changelog entry under today's date.
- Tests-first-or-with-change discipline
- Run -> Observe -> Fix loop at the end of every milestone
- **Self-improvement loop** — after each milestone, capture lessons, gotchas, and patterns via `/learn`, the project's self-improvement loop in CLAUDE.md, or memory — whichever exists in the executing environment
- **Work verification** — after each milestone, self-review all changes, run Codex external review (if available), and perform security review of changed and proximal code
- No overbuilding
- No speculative features
- Cost-efficient architecture preferred
- Explicit folder structure discipline
- **Design system consistency** — if the project has a component library or design system, all page/view files must use shared components, never raw HTML equivalents. If the project uses a token-based styling system (CSS variables, design tokens, theme), page files must use semantic tokens, never hard-coded color/spacing values. These rules must be enforced by automated tooling (linter rules, CI checks), not just documentation. See the "UI Consistency Guardrails" section below.
- **Global name discipline (no collisions, no drift)** — every persistent schema object (table, enum, shared/reference table) lives in ONE global namespace recorded in the Schema Object Registry (STEP 3.5). (a) **Domain-prefix any name that is a generic noun** (`task`, `contact`, `message`, `module`, `assessment`, `document`, `payment`, `agent`, `status`, …) or that could plausibly recur in another module — `crm_task`, `inbox_message`; when two milestones each have an "X", prefix **both**, never let one squat the bare name. (b) **Each object is declared by exactly ONE (owner) milestone**; every other milestone consumes it by its exact registry name and never re-declares it. Names are drawn from the registry **verbatim**, never re-invented per pack. (Follow the project's schema-conventions ADR if one exists.) This is the structural fix for the most common cross-pack defect: two packs coining the same table name, or a consumer pack referencing a table by a different name than its owner actually declares.
- **Spec↔code drift gates (canonical doc → mechanical check)** — every canonical, human-authored spec that governs generated code (data-model/schema doc, interface contracts, design tokens, env-var inventory) gets an automated drift gate wired into the project's check target, not just prose. Silent drift here is the expensive kind: a schema column can sit undocumented across dozens of migrations before anyone notices, because nothing mechanically compares doc to code. If the project has a canonical schema/data-model doc **and** a migration tool, the first persistence milestone must establish a schema⇄spec drift gate — see the "Schema/Spec Consistency Guardrails" section below.
- CLAUDE.md is canonical engineering law — read it before generating prompts and embed its rules into every milestone prompt

If constraints are mentioned in CLAUDE.md (hosting / language / library restrictions, commit conventions, testing conventions, verification steps, or self-improvement behaviors), embed these constraints inside the generated Prompt-Pack. Each prompt must be self-sufficient.

---

## STEP 3: DECOMPOSE INTO MILESTONES

Analyze the specs and decompose into atomic execution stages following real engineering flow:

1. **Early milestones** create: repo structure, foundational architecture, data schema
2. **Middle milestones** build: core business logic, interfaces, integrations
3. **Late milestones** include: tests, hardening, deployment, validation

Example flow (adjust based on project scope):

- M0 -- Runbook Enforcement & Guardrails
- M1 -- Repo + Folder Bootstrap (includes design system foundation + automated UI guardrails)
- M2 -- Core Architecture Skeleton
- M3 -- Database Schema (if applicable)
- M4 -- Core Domain Logic
- M5 -- Interface Layer (API/UI)
- M6 -- Integrations
- M7 -- Testing & Validation
- M8 -- Deployment Hardening
- M9 -- Release Validation
- M10 -- Backlog Sweep & Hardening (mandatory final pack — see below)

If product scope is smaller or larger, collapse or expand responsibly but preserve order logic.

### Mandatory final milestone: Backlog Sweep & Hardening

The last pack in every generated set MUST be a backlog-sweep pack (`NN_HARDENING_BACKLOG_SWEEP.prompt.md`, numbered after the final feature milestone). During execution, `/build` (pre-flight deferrals), `/verify-build` (out-of-scope requirements), `/verify-wiring` (advisory Checks G–L), and `/review-externally` (real-but-later findings) all write deferred items to `docs/dev/backlog.md` — but no feature pack reads it, because all packs are generated before the backlog exists. Without this pack the backlog has writers and no reader, and deferred items accumulate forever.

Because its content is unknowable at generation time, this pack is shaped differently from feature packs:

- **Section 7** names `docs/dev/backlog.md` (its `## Open` entries) as the work queue instead of a concrete file list. The ≤12-files / ≤8-commits context budget does not apply; instead, instruct the executing LLM to process the backlog in priority order and split into resumable sub-sessions if it is large.
- **Process:** read every `## Open` entry and triage each as (a) **build now**, (b) **won't build** — close with a one-line reason, or (c) **re-defer** — only with a named future home; "later" alone is not a home. Implement accepted items with the full sections 8–12 discipline applied per item (wiring, tests, failure modes), and move resolved entries to `## Done` citing this pack.
- **Section 12:** acceptance = `## Open` is empty, or every remaining entry carries an explicit won't-build / re-defer disposition dated by this pack.
- It still ends with the standard 13–18 preamble reference block — all five verification passes run on it like any other pack.

### Sub-Milestone Splitting (Context Budget)

Each prompt-pack file must stay within these hard limits to prevent context exhaustion during execution:

- **≤ 12 new files created** per prompt pack
- **≤ 8 expected commits** per prompt pack
- **≤ 4 spec references** in section 3 per prompt pack
- **≤ 200 lines** of prompt text (excluding the shared preamble reference)

The 200-line budget INCLUDES the wiring, state-matrix, and failure-modes tables — those tables are the largest context consumers, so they count. Only the preamble reference block is excluded. If required tables alone push a pack past the budget, that is the signal to split.

If a logical milestone would exceed ANY of these limits, split it into sub-milestones using letter suffixes (M7a, M7b, M7c). Sub-milestone files keep the parent's two-digit prefix with the letter appended: `07a_TESTING_CORE.prompt.md`, `07b_TESTING_E2E.prompt.md` — never renumber sibling milestones to make room. Each sub-milestone:

- Is its own prompt-pack file
- Has its own Context Snapshot listing prior sub-milestones as complete
- Has its own focused deliverables and wiring checklist
- Is independently executable and leaves the project in a working state
- Shares the same Non-Goals section as its sibling sub-milestones

**Splitting is not scope creep — it's how you prevent context exhaustion.** A milestone that runs out of context produces worse code than two sub-milestones that complete cleanly.

When splitting, group by cohesion: data layer + API routes in one sub-milestone, UI pages in the next. Never split a single feature across sub-milestones (e.g., don't put the API route in M7a and its only UI caller in M7b).

**Critical:** The foundation/bootstrap milestone (M0 or M1) must establish UI consistency guardrails as automated tooling — not just documentation. These guardrails prevent design system drift in every subsequent milestone. See the "UI Consistency Guardrails" section below for the specific tooling to generate.

---

## STEP 3.5: GENERATE THE SCHEMA OBJECT REGISTRY (GLOBAL NAMESPACE)

Before generating any pack, do ONE global pass over the specs + the milestone decomposition and enumerate **every persistent schema object the product needs** — every table, every enum, every shared/reference table — into `docs/prompt-packs/_SCHEMA_REGISTRY.md`. This is the schema analogue of `_PREAMBLE.prompt.md`: generated once, authoritative, and **referenced (never re-derived)** by every pack. Skip it only for products with no persistence layer.

It exists to make two whole defect classes *structurally impossible* rather than caught in a post-hoc audit:
- **Collisions** — two packs declaring the same table/enum name (duplicate `CREATE TABLE` / duplicate barrel export → broken migration + typecheck).
- **Schema↔consumer drift** — a UI/consumer pack referencing a table by a different name than the owner pack actually declares (the consumer builds against a symbol that doesn't exist).

Record one row per object:

```
| object (concept)                   | kind  | owner pack | canonical name              | one-line shape                     | consumer packs |
|------------------------------------|-------|-----------|-----------------------------|------------------------------------|----------------|
| trainer↔scheduled-class allocation | table | 12a       | class_trainer_allocation    | tenant, class FK, trainer FK, role | 12b, 21a       |
| trainer↔TAS-strategy allocation    | table | 08a       | tas_trainer_allocation      | tenant, tas FK, trainer FK, units[]| 18a            |
| person document type               | enum  | 09a       | document_type               | person-document discriminator      | …              |
| application document-requirement   | enum  | 26a       | requirement_document_type   | application doc requirement type   | 26c            |
```

Rules:
1. **One owner per object** — the earliest milestone that needs it persisted (usually the module's `*_SCHEMA` pack). The owner DECLARES it (its Section 7); every other pack CONSUMES it (their Section 2).
2. **Canonical name obeys the STEP-2 naming discipline** — domain-prefix any generic or collision-prone noun; two concepts sharing a noun become two differently-prefixed objects (`tas_…` vs `class_…`), each its own row + owner.
3. **No name is owned twice.** If two milestones each need "an X", that is two rows with two prefixed names — never one bare name claimed by both.
4. **Reference/shared (non-tenant) tables and enums are in the registry too** — a reused `status`/`document_type` enum collides just as easily as a table.
5. **An object that already has an owner is never re-declared downstream.** If a later pack must change it, it EXTENDS the existing object (and says so in its Context Snapshot) — it does not create a second one.

Every subsequent step reads names from this registry. When pack generation is delegated to subagent batches (STEP 4), each batch is handed this file and uses its names verbatim.

---

## STEP 4: GENERATE PROMPT-PACK FILES

Output all prompt-pack files to `docs/prompt-packs/` relative to the project root.

**Note on examples:** the table examples in sections 8 and 11 below use Next.js/TypeScript conventions (`route.ts`, `requireCsrf()`, `(dashboard)/`) purely to illustrate the format. When generating packs, translate every example to the target project's actual stack, file layout, and guard functions — never copy example identifiers verbatim.

**When generation is delegated to subagent batches:** hand every batch the `_SCHEMA_REGISTRY.md` (STEP 3.5) and instruct it to use registry names **verbatim** — for both objects the pack OWNS (Section 7) and objects it CONSUMES (Section 2) — and to never coin a schema-object name absent from the registry. If a batch discovers a genuinely new object, it adds a row to the registry (claiming ownership) BEFORE declaring it, so a sibling batch can't independently coin the same or a conflicting name. Parallel batches that each invent names are exactly how the same table gets declared twice, or a consumer and its owner diverge on a name.

### Structure:

```
/docs/prompt-packs/
  _PREAMBLE.prompt.md       (shared sections 13–18)
  _SCHEMA_REGISTRY.md       (global schema-object namespace — STEP 3.5)
  00_RUNBOOK_ENFORCER.prompt.md
  01_PROJECT_BOOTSTRAP.prompt.md
  02_ARCHITECTURE_FOUNDATION.prompt.md
  ...additional milestones...
  NN_HARDENING_BACKLOG_SWEEP.prompt.md   (mandatory final pack — STEP 3)
  README.md
```

### Each Milestone Prompt MUST Contain These Sections (In Order):

> **Section numbers are permanent stable IDs.** If a future revision of this skill inserts a new section, give it a letter suffix on its predecessor (e.g. a section between 10 and 11 becomes 10a) — never renumber existing sections. The numbers below are shared vocabulary across `/build`, `/verify-build`, `/verify-wiring`, and `/build-verify-review`.

#### 1. Milestone ID
Example: `M2 -- Core Domain Implementation`

#### 2. Context Snapshot
- Current repo state
- Expected existing files
- **Consumed schema objects** — every table/enum this milestone READS but does not own, listed **by its exact canonical name from `_SCHEMA_REGISTRY.md` + its owner pack** (e.g. "consumes `class_trainer_allocation` (owner 12a)"). Never paraphrase, guess, or re-invent a consumed name — a consumer that references a guessed name the owner never declared is a build failure. If an object this milestone would otherwise "create" already has an owner in the registry, do NOT re-declare it: consume it, or (if it must change) EXTEND it and say so explicitly.
- Prior milestones completed
- Stack constraints
- Hosting constraints (if applicable)

#### 3. Spec References
List the specific spec documents (and section numbers) that the executing LLM should read for context before executing this milestone. Only include specs that are genuinely relevant — not every spec for every milestone. Format:

```
## 3. Spec References

Read these specs for context while executing this milestone:

- `docs/specs/FILENAME.md` — §SECTION: brief description of what's relevant
```

**Full repo-relative file paths are mandatory.** Every reference MUST begin with the spec's actual path in backticks (e.g., `` `docs/specs/honchosuite-db-schema.md` §9 ``) — never a bare document ID ("DB-HONCHO-001 §9"), document nickname ("the PRD §5"), or title. Bare IDs force the executing LLM to resolve an indirection it may get wrong after a context reset; paths are greppable and unambiguous. Shorthand IDs are acceptable only as parenthetical sub-references inside an entry that already leads with the full path. This rule applies even when prompt-pack generation is delegated to subagents — restate it in their instructions, since "match sibling style" drift across agent batches is exactly how mixed citation formats arise.

This ensures the executing LLM has access to authoritative design decisions, field definitions, business rules, and architecture patterns rather than relying solely on the prompt-pack's inline instructions. Cross-reference against all available spec documents and include only sections that directly inform the milestone's deliverables.

#### 4. Objective (Bounded)
Clear and narrow scope definition.

#### 5. Non-Goals (Explicitly Stated)
Prevent overbuilding by listing what this milestone does NOT do.

#### 6. Stack Constraints
- Language rules
- Framework limits
- Dependency restrictions
- Cost discipline requirements
- Hosting limitations

#### 7. Required Deliverables
Explicit list of files to generate/update, with full paths. **Every NEW persistent schema object (table/enum) declared here MUST be an object this milestone OWNS in `_SCHEMA_REGISTRY.md`, named EXACTLY as the registry records it** (domain-prefixed per STEP 2). Do NOT declare an object owned by another pack — consume it via Section 2 instead. Do NOT coin a schema-object name absent from the registry; if the milestone reveals a genuinely new object, add a registry row (with this pack as owner) first, then declare it here.

#### 8. Integration Wiring Checklist

**"File exists with correct exports" ≠ "feature works."** Library modules that are implemented but never called from application code are dead code that silently passes structural verification.

For every library module listed in section 7, the prompt must include a wiring table that specifies **where** each exported function must be called. This prevents the executing LLM from implementing a module in isolation without connecting it to the application.

Keep each row to ONE line. Do NOT include multi-sentence explanations in table cells. If a wiring relationship needs clarification, add a numbered footnote below the table.

Format:

```
## 8. Integration Wiring Checklist

| Export | Caller → trigger | Grep pattern |
|---|---|---|
| `recordMilestone("gsc_connected")` | `gsc/callback/route.ts` → OAuth success | `recordMilestone.*gsc_connected` |
| `computeHealthScorecard()` | `analytics/route.ts` → GET response | `computeHealthScorecard` |
| `logActivityFromRequest()` | Every admin POST/PATCH/DELETE route | `logActivityFromRequest` |
| `csrfHeaders()` | Every client-side POST/PATCH/PUT/DELETE fetch | `csrfHeaders` in `(dashboard)/` |
```

Rules for populating this table:

1. **Every library function** that is listed in the deliverables but is NOT an API route handler must have at least one caller listed.
2. **Every client-side POST/PATCH/PUT/DELETE** `fetch()` call must include CSRF headers if the target route uses `requireCsrf()`.
3. **Every session/context field** that the spec says "carries X" (e.g., dual-identity on impersonation) must list the code that populates it AND at least one consumer.
4. **Every UI page** must list the actions it renders (buttons, links) and the API routes they call. If a route exists but has no UI trigger, it must be listed as needing one.
5. **Cross-module interactions** — when module A's guard/check can block module B's intended behavior (e.g., idempotency guard vs. force-regenerate), both sides must be listed together with a note on how they coexist.

The executing LLM must verify every row in this table as part of the Post-Milestone Verification (section 16, Pass 1).

#### 9. Documentation Requirements (Chronological Changelog Discipline)
Must update:
- CHANGELOG.md — add entries **per commit**, not per milestone. Each user-visible change gets its own entry under today's date. Entries are strictly chronological (newest at top). Categories within a date: `Added`, `Changed`, `Fixed`, `Removed`. Internal refactors, dependency updates, and CI changes do NOT get changelog entries unless they affect user-visible behavior. Never retroactively edit past entries.
- README.md (if necessary)
- Relevant docs/specs

#### 10. Testing Requirements (Coverage Discipline)
Define expected tests to add/update:
- Unit tests (pure logic, validators, utilities, domain functions)
- Integration tests (DB, queues, file IO, external services via mocks)
- **State-matrix integration tests** (see below — required when the feature has multiple states, roles, or modes)
- E2E browser tests (critical user journeys, UI flows, regressions)

Rules:
- If a feature is added or changed, corresponding tests must be added/updated in the same milestone
- If behavior changed, update existing tests that would now fail
- Maintain minimal but meaningful coverage; do not write tests for future features

##### State-matrix integration tests — when required

Unit tests verify a function in isolation; state-matrix integration tests verify that a feature behaves correctly across **every combination of the states it can encounter in production**. Unit-test-only coverage repeatedly misses bugs that only appear at specific state intersections (e.g., "owner role + onboarding status + trial plan + CSRF missing").

A milestone MUST include a state-matrix integration test when ANY of the following apply:

1. **Multiple roles/permissions** — the feature behaves differently for ≥ 2 user roles (owner, admin, member, viewer, impersonated, anonymous)
2. **Multiple lifecycle states** — the entity has a `status` / `state` / `phase` enum with ≥ 2 values that change behavior (onboarding/active/suspended, draft/published/archived, trial/paid/expired)
3. **Multiple plans/tiers** — gated by subscription tier, feature flag, or org-level capability
4. **Authentication boundaries** — the route is reachable both authenticated and unauthenticated, or both same-org and cross-org
5. **CSRF / auth guards** — any state-changing route guarded by `requireCsrf()` or `requirePageAuth()` (must test both present and missing)
6. **Cross-module guards** — module A's guard (idempotency, rate limit, feature flag) can block module B's call path

When required, the prompt pack must specify the matrix explicitly. Format:

```
##### State Matrix — `<feature name>`

Dimensions:
- role: owner | admin | member | viewer
- org.status: onboarding | active | suspended
- csrf: present | missing

Test file: `src/<path>/<feature>.matrix.test.ts`

Expected cells: 4 × 3 × 2 = 24. List every (role, status, csrf) tuple and its expected outcome
(200 OK | 403 Forbidden | 401 Unauthorized | 422 Unprocessable). Use `describe.each` /
`it.each` so each cell is an independent assertion — never collapse cells into a single test
with branching.

Skipped cells must be justified in a comment (e.g., "viewer + suspended is unreachable —
suspended orgs return 403 at proxy level before role check").
```

Rules for state-matrix tests:

- One assertion per cell — no branching inside the test body
- Hit the real route handler with a real DB (or test DB), not a mock — the point is to catch wiring bugs that mocks hide
- Seed every dimension explicitly in `beforeEach` — never rely on test ordering
- If the matrix exceeds ~50 cells, split by dimension (e.g., one file per role) rather than reducing coverage
- The matrix lives in section 10 of the prompt pack and is also referenced from section 12 (Acceptance Criteria) as a binary pass/fail gate

If a milestone's deliverables include ANY API route handler, page with role-based rendering, or function that branches on org/user state, section 10 MUST either (a) specify the state matrix, or (b) include an explicit one-line justification for why a matrix is not required (e.g., "pure transformation, no state dependencies").

If the milestone implements a **stateful protocol** (see `/build` Phase 2.6 — claim/lease/lock/retry/dedupe/state-machine signals, or a Failure-Modes section dominated by concurrency rows), section 10 must ALSO name **interleaving** tests for each concurrency invariant — two concurrent actors, crash-then-resume, double-fire — not only a state matrix. A matrix asserts per-*state* outputs; it never exercises a *race*.

#### 11. Failure Modes (Resilience Discipline)

"Handle errors" is too generic and consistently produces packs that ship with unguarded fetches, non-idempotent workers, and lost-update races. This section forces the prompt pack to enumerate **which** failure modes apply, **how** they're mitigated, and **which test** proves the mitigation works.

This section is REQUIRED when the milestone introduces ANY of:

- **(F)** Outbound server-side fetch (a READ) to a user-controlled URL, third-party API, or webhook target
- **(S)** An outbound side-effect that MUTATES external state — sends a message/email/SMS/push, charges or refunds, or calls any third-party *write* API. The symmetric write-side of (F); its hazards differ.
- **(W)** A job enqueued to a worker/queue, scheduled task, or background handler
- **(P)** A multi-step pipeline where intermediate steps can fail independently
- **(R)** A read-then-write sequence where the read can go stale between read and write

If none apply, omit the section with a one-line justification: `Failure Modes: N/A — pack contains only <pure-transform | schema-only | UI-only | type-only> changes`.

When required, the prompt pack must include a table with one row per applicable failure mode. Use the category code (F/W/P/R) so the executing LLM knows which checklist to walk.

```
## 11. Failure Modes

| # | Mode | Mitigation | Test |
|---|---|---|---|
| F1 | SSRF — request to private/metadata/loopback IP | `assertPublicUrl(url)` before fetch; deny RFC1918, 169.254.0.0/16, ::1, fc00::/7 | `fetch.ssrf.test.ts` |
| F2 | Open redirect — 3xx to private IP after public pass | Re-run `assertPublicUrl` on every `Location` header; cap redirects at 3 | `fetch.redirect.test.ts` |
| F3 | DNS rebinding — A record flips between resolve and connect | Resolve once, pin IP, pass via `lookup` option in fetch agent | `fetch.dns-rebind.test.ts` |
| F4 | Oversized response | `maxResponseBytes` cap, abort stream past limit | `fetch.size-cap.test.ts` |
| F5 | Slow loris / hung connection | `AbortSignal.timeout(N)` on every fetch | `fetch.timeout.test.ts` |
| W1 | jobId collision / double-enqueue | Idempotency key = `<entity>:<entityId>:<intent>`; queue dedupes on key | `worker.idempotency.test.ts` |
| W2 | Downstream worker down / queue unreachable | Enqueue inside same txn as state write (outbox), or retry with exponential backoff + DLQ after N attempts | `worker.queue-down.test.ts` |
| W3 | Poison message — handler throws forever | Max-attempt count, DLQ on exceed, alert | `worker.poison.test.ts` |
| W4 | Visibility timeout shorter than job runtime | Heartbeat extends lease, or job is chunked under timeout | `worker.long-running.test.ts` |
| P1 | Partial pipeline — step 3 of 5 fails | Each step idempotent on retry, OR compensating action for non-idempotent steps; resume from last successful step | `pipeline.partial-failure.test.ts` |
| P2 | Pipeline retry replays side effects | Step-level dedupe keys; external calls use idempotency-key header | `pipeline.replay.test.ts` |
| R1 | Concurrent-user race — two writers, last-write-wins clobbers | Optimistic lock (`WHERE updated_at = ?`), row-level `FOR UPDATE`, or DB-level unique constraint | `<feature>.concurrent.test.ts` |
| R2 | Source data changed after read | Re-read inside same txn before write, OR include read-version in write predicate | `<feature>.stale-read.test.ts` |
| R3 | Webhook / callback replay | Persist `(provider, event_id)` with unique constraint; reject duplicates | `webhook.replay.test.ts` |
```

**(S) side-effect checklist** — author these rows per stack (the lock / idempotency mechanisms differ by platform), so DERIVE them rather than copying examples. For each external mutating effect, the pack must state and test:
- **Delivery semantics** — at-least-once or at-most-once? Every row below follows from this.
- **Idempotency** — a stable key so a retry or concurrent duplicate collapses to one effect (most send/charge APIs accept an idempotency key; see also W1/P2/R3).
- **Precondition race** — a DB check (suppression / balance / consent) and the effect cannot share a transaction; serialize them with a lock that spans the effect, or re-verify at the provider.
- **Crash window** — record the effect's success AFTER the provider accepts it, not before, so a crash mid-effect retries instead of skipping a never-sent effect.
- **Failure blast-radius** — classify per-item (one bad target) vs system-wide (auth / config / outage); each needs a different retry + alert response.

Rules for populating the table:

1. **One row per distinct failure mode** — do not collapse F1+F2+F3 into "URL validation"; each has its own mitigation and its own test.
2. **Mitigation column names the guard function or pattern** — if the mitigation is a function (`assertPublicUrl`, `withIdempotencyKey`), that function must appear in the Section 8 wiring checklist with its callers listed. If the mitigation is a config (queue retry policy), point to the config file path.
3. **Test column names a real test file** — not "covered by existing tests". The test must assert the failure mode is blocked, not just that the happy path works.
4. **Skipped modes must be justified inline** — e.g., `F3 (DNS rebinding): N/A — fetches only to allowlisted hostnames, no user input`. Justifications are auditable; "not applicable" with no reason is not acceptable.
5. **Cross-reference Section 12** — every populated row contributes an acceptance criterion: "Failure mode <N> test passes". `/verify-build` treats missing tests as a build failure.

The executing LLM must walk this table during Pass 1 of Post-Milestone Verification (section 16) the same way it walks the Section 8 wiring checklist.

#### 12. Acceptance Criteria
Bullet-point validation rules (functional + non-functional within milestone scope).

If section 10 specifies a state matrix, acceptance criteria MUST include a bullet:
"State-matrix test `<file path>` exists and every non-skipped cell passes; every skipped cell carries an inline justification comment."

If section 11 populates a Failure Modes table, acceptance criteria MUST include one bullet per row:
"Failure mode `<#>` (`<short name>`): mitigation `<function/config>` is wired and test `<file path>` passes." Skipped rows carry their inline justification instead of a bullet.

#### 13–18. Shared Preamble (do not inline)

Sections 13 (Run → Observe → Fix), 14 (Commit Discipline), 15 (Overbuild Prevention), 16 (Post-Milestone Verification — all 5 passes), 17 (Self-Improvement Loop), and 18 (Self-Validation Checklist) are identical across all prompts.

Generate these ONCE in `_PREAMBLE.prompt.md` at the top of the prompt-pack directory. Each milestone prompt replaces these six sections with:

```
## 13–18. Standard Procedures

Follow `_PREAMBLE.prompt.md` for: Run → Observe → Fix, Commit Discipline,
Overbuild Prevention, Post-Milestone Verification (5 passes),
Self-Improvement Loop, and Self-Validation Checklist.

### Milestone-Specific Overbuild Warnings
- Do NOT implement [feature X] — that is M[N]
- ...
```

Keep the milestone-specific addendum to ≤ 10 lines. The full procedure text lives only in the preamble.

---

## STEP 5: GENERATE _PREAMBLE.prompt.md

Generate `/docs/prompt-packs/_PREAMBLE.prompt.md` containing the full text for these shared sections:

- **Section 13: Run → Observe → Fix Loop** — run test suites, review errors, fix until green, commit each fix as `fix(<scope>):`, re-run to confirm
- **Section 14: Commit Discipline** — commit per logical change (not per milestone), Conventional Commits format, stage only related files, tests alongside features
- **Section 15: Overbuild Prevention Rules** — no future roadmap phases, no new frameworks, no unrelated files, no schema expansion, no premature optimization
- **Section 16: Post-Milestone Verification (5 passes)** — Pass 1 (self-review + wiring checklist walk), Pass 2 (`/verify-build` — completeness audit: every deliverable, test, and acceptance criterion exists and matches spec), Pass 3 (`/verify-wiring` — connectivity audit: every page navigable, every API route called, every job registered, every library function imported by production code), Pass 4 (Codex review if available), Pass 5 (security review of changed + proximal code)
- **Section 17: Self-Improvement Loop** — capture lessons via `/learn` (or, if that command is unavailable, the Self-Improvement Loop section in CLAUDE.md / a dated lessons file), extract repeated workflows into skills, update `CLAUDE.md` for new conventions
- **Section 18: Self-Validation Checklist** — all acceptance criteria met, all 5 passes clean, CHANGELOG updated, clean git state

This file must be self-contained — the executing LLM reads it once alongside the milestone prompt. Include the full detail (Codex invocation syntax, security review categories, wiring checklist walk procedure, etc.) so nothing is lost by extracting it from individual prompts.

---

## STEP 6: GENERATE README.md

The `/docs/prompt-packs/README.md` must include:

- Execution order (list all prompts in sequence, ending with the mandatory `NN_HARDENING_BACKLOG_SWEEP` pack)
- Instruction to read `_PREAMBLE.prompt.md` before executing any milestone
- Instruction to treat `_SCHEMA_REGISTRY.md` as the authoritative schema-object namespace — a pack declares only the objects it OWNS there and consumes every other by its exact registry name (never re-declares or renames)
- How to resume after partial execution
- How to revert last milestone
- How to regenerate corrupted files
- How to rebuild context if conversation resets

---

## STEP 7: CONTEXT DEGRADATION RESILIENCE

Since context may be truncated and LLM memory may reset:

- Each prompt must restate necessary assumptions
- Each prompt must explicitly list required files
- No reliance on implicit prior reasoning

---

## PROMPT DESIGN PRINCIPLES

Enforce these design rules across all generated prompts:

**Determinism Over Creativity** -- Prompts must prioritize correctness over flair.

**Isolation of Concerns** -- Each milestone must not depend on undefined artifacts.

**Artifact Traceability** -- All generated files must be named explicitly, placed in explicit folders, and follow conventions.

**Milestone Finality** -- Each prompt must end in a state where the project compiles (where applicable), runs, and remains deployable.

---

## QUALITY BAR

The Prompt-Pack must be:
- Deterministic
- Reproducible
- Stack-aligned
- Production-safe
- Minimal but complete
- Free of ambiguity, fluff, and architectural drift

Before finalizing, evaluate: "Could this prompt-pack reliably build a real product if executed sequentially by a disciplined LLM?" If not, refine until yes.

### Generator Self-Audit (mechanical — run before declaring done)

Walk every generated pack against this checklist and fix violations before reporting completion:

- [ ] Every pack contains sections 1–12 in order (section 11 Failure Modes may be its one-line N/A justification), followed by the 13–18 preamble reference block
- [ ] Every pack is within ALL four context-budget limits (files, commits, spec refs, lines); any violator is split per Step 3
- [ ] Every non-route library function in section 7 has at least one caller row in section 8
- [ ] Every populated section 11 (Failure Modes) row has a matching acceptance-criteria bullet in section 12; every state matrix in section 10 is referenced in section 12
- [ ] No section 8/10/11 entry contains copied example identifiers (`requireCsrf`, `recordMilestone`, `(dashboard)/`, etc.) unless they genuinely exist in the target stack
- [ ] Every section 3 entry leads with a full repo-relative file path in backticks — no bare document IDs or nicknames (grep packs for ID patterns like `[A-Z]+-[A-Z]+-[0-9]+ §` and `\b(PRD|BRD) §` at entry starts; rewrite any hits)
- [ ] `_PREAMBLE.prompt.md` exists, defines all five verification passes, and every pack's reference block says "5 passes"
- [ ] README.md lists every pack (including sub-milestone letter files) in execution order
- [ ] The final pack in execution order is the backlog-sweep pack (`NN_HARDENING_BACKLOG_SWEEP`), and its Section 7 names `docs/dev/backlog.md` as its work queue
- [ ] Each pack's Context Snapshot lists exactly the prior milestones that exist as files — no gaps, no phantom milestones
- [ ] **`_SCHEMA_REGISTRY.md` exists** and every table/enum named in any pack's Section 7 (declared) or Section 2 (consumed) appears as a registry row with exactly one owner.
- [ ] **Registry conformance — no collisions, no drift.** Extract every schema object each pack DECLARES (Section 7) and CONSUMES (Section 2), and assert: (a) **no** table/enum name is declared by more than one pack; (b) every CONSUMED name exists in the registry with a declaring owner; (c) no consumed name is a paraphrase/variant of an owned name — the drift check (e.g. a pack consumes `crm_contact` but the owner declares `contact`, or consumes `lms_module` but the owner declares `module`); (d) every OWNED name obeys the STEP-2 domain-prefix rule — flag bare generic nouns (`task`, `contact`, `message`, `module`, `assessment`, `document`, `payment`, `agent`, `status`, …) that lack a domain prefix. Any hit is fixed (rename + update the registry + every referencing pack) before declaring done. *This mechanical pass is what converts collision-catching from a post-hoc audit into a generation-time gate.*

---

## UI CONSISTENCY GUARDRAILS

If the project has a frontend with a component library or design system, the foundation/bootstrap milestone must establish **three layers of automated enforcement** that prevent design system drift in all subsequent milestones. Documentation alone is insufficient — inconsistencies were discovered in a production project where CLAUDE.md rules existed but had no automated enforcement, resulting in 39+ files of drift across raw HTML elements, hard-coded colors, and duplicated style maps.

### Layer 1: CLAUDE.md UI conventions section

Add a "UI Component & Styling Conventions" section to `CLAUDE.md` that explicitly lists:
- Which design system components replace which raw HTML elements (mapping every raw element that has a styled equivalent)
- Which semantic tokens/variables replace which hard-coded values (colors, spacing, typography) — applicable to CSS-in-JS, Tailwind, CSS Modules, or any token system
- The project's focus/interaction state patterns
- How status/variant styling should be applied (component props/variants, not inline style overrides)
- A reference implementation file that exemplifies correct usage

This layer prevents AI-assisted code generation from introducing inconsistencies.

### Layer 2: Linter rule to forbid raw elements

Use the project's linter to ban raw HTML elements that have design system equivalents, **scoped to consumer code only** (pages, views, routes — not the component library internals which legitimately use raw elements).

Examples by ecosystem:
- **React + ESLint:** `eslint-plugin-react`'s `react/forbid-elements` rule scoped to page file globs
- **Vue + ESLint:** `eslint-plugin-vue` custom rules or `no-restricted-syntax` targeting specific tags
- **Angular:** `@angular-eslint` with template rules
- **Svelte:** `eslint-plugin-svelte` with custom element restrictions

The key properties: (1) errors, not warns, (2) scoped to consumer code only, (3) message tells the developer which component to use instead.

This layer catches violations at lint time during the "Run → Observe → Fix" loop (section 13).

### Layer 3: CI check for hard-coded style values

Add a CI script or build target that searches consumer code for hard-coded style values that should use tokens, and fails if any are found. This catches violations that element-level linter rules miss (e.g., using the correct component but with inline color overrides that bypass the design system).

What to search for depends on the styling approach:
- **Tailwind:** grep for raw color utilities (`text-zinc-`, `bg-gray-`, `border-slate-`, `ring-blue-`) in page files
- **CSS Modules / styled-components:** grep for raw hex/rgb values or hard-coded px values that should use variables
- **CSS-in-JS (Emotion, Stitches):** grep for string color literals instead of theme tokens

Wire this check into the project's main CI/lint target so it runs automatically alongside other lint checks.

**Why a separate check:** Element-level linter rules can't catch misuse of *correct* components with *wrong* styling (e.g., a design system Badge used with inline color classes instead of a variant prop). The pattern-level check catches these.

### Embedding in milestone prompts

When generating milestone prompts that include UI deliverables:

1. **Section 6 (Stack Constraints)** must include a note that page/view files must use design system components and that linter rules enforce this.
2. **Section 8 (Integration Wiring Checklist)** must include a verification row confirming all page files pass both the element-level linter rule and the token-level CI check.
3. **Section 13 (Run → Observe → Fix Loop)** must include the design system CI check alongside standard lint and type checks.
4. **Section 15 (Overbuild Prevention)** must include: "Do NOT use raw HTML elements that have design system equivalents. Do NOT use hard-coded style values that have semantic token equivalents."

---

## SCHEMA / SPEC CONSISTENCY GUARDRAILS

If the project has a **canonical schema / data-model spec** (a human-authored doc that is the source of truth for tables/columns/enums) **and** a migration tool, the first milestone that introduces persistence must establish **three layers of enforcement** that keep the built database from drifting out of sync with that spec — in BOTH directions. Documentation alone is insufficient: in a production project a real column sat undocumented in the spec across a dozen migrations before an audit caught it, because nothing mechanically compared the two.

This is the database analog of the UI guardrails above. Gate it on the capability (canonical schema doc + migration tool present) exactly as the UI layers gate on "has a design system" — do NOT add it at the bare scaffold milestone, where there are no tables, no populated spec sections, and no migrations to reconcile. It belongs in the first pack that introduces a schema, so it protects every migration thereafter.

### Layer 1: CLAUDE.md schema-authority rule

Add (or confirm) a CLAUDE.md rule that the schema spec is canonical: packs never invent tables/columns — a pack that needs a new one edits the spec doc in the **same commit**, flagged. This is the human-facing law the mechanical layers enforce.

### Layer 2: generation-time name gate (already STEP 3.5)

The Schema Object Registry + Generator Self-Audit (STEP 3.5) is the first mechanical layer: it catches cross-pack name collisions and paraphrase-drift (`contact` vs `crm_contact`) at generation time, before any code is built. Nothing new to author here — ensure every persistence pack declares/consumes names through the registry.

### Layer 3: CI schema⇄spec drift gate

Add a CI script (wired into the project's main check/lint target) that parses the built schema from the migration tool's output and the tables/columns from the spec doc, and fails on drift. Enforce, at minimum:

- **Backward drift (hard fail):** every built table and every built column must be documented in the spec. This is the guarantee the spec never silently falls behind the database — the direction that rots invisibly, and the primary reason this gate exists.
- **Forward gap (bounded, optional per project):** a spec table with no migration must be explicitly acknowledged (e.g. a `DEFERRED` allowlist) rather than silently missing, with a **ratchet** that fails once a deferred table is actually built (forcing its removal) or is missing from the spec (stale entry). Include this layer only for **spec-ahead-of-build** projects (prompt-pack projects, where the spec is the forward target); an ad-hoc repo where code leads the spec wants the backward check alone.

The script is **tailored to the repo, not copied.** It must parse the actual migration format (Drizzle SQL, Prisma schema, Rails/Alembic/Flyway migrations, raw SQL) and the actual spec-doc conventions of THIS project. A generic copy that doesn't parse the real spec is worse than none — it is either vacuously green (worthless) or noisily red (gets disabled), and a rubber-stamp CI gate trains the team to ignore CI. Carve out framework-managed tables the spec intentionally omits (e.g. auth-adapter tables).

**Acceptance bar (required): seed it red-then-green.** The milestone is not done until the guard is demonstrated to (1) FAIL on a planted drift — add a throwaway column absent from the spec, confirm the check catches it — and (2) PASS on the clean tree. A drift gate that has never been observed to fail is not known to work.

### Embedding in milestone prompts

When generating the persistence-introducing milestone and every later one that touches the schema:

1. **Section 6 (Stack Constraints)** must state that the schema spec is canonical and that the drift gate enforces it.
2. **Section 8 (Integration Wiring Checklist)** must include a row confirming new tables/columns are documented in the spec and the drift gate passes.
3. **Section 13 (Run → Observe → Fix Loop)** must include the drift gate alongside standard lint/type checks.
4. **Section 15 (Overbuild Prevention)** must include: "Do NOT add a table or column without documenting it in the schema spec in the same commit."

---

## HANDLING AMBIGUITY

If ambiguity exists in the specs:
- Make conservative assumptions
- Document assumptions explicitly in each prompt where they apply
- Do not expand scope

---

Now begin. Read the specs, red-team them (STEP 1.5), decompose the product, and generate the full prompt-pack system.

---

## POST-GENERATION REMINDER

After generating all prompt-pack files, remind the user of the full execution and verification pipeline:

```
## Execution pipeline for each milestone

1. /build M[N]            — execute the prompt pack: build all deliverables, tests, commits
                            (if context runs out, resume with: /build M[N] --resume)

2. /verify-build M[N]     — completeness: was everything in the spec built?
   Fix any gaps, then:
3. /verify-wiring M[N]    — connectivity: is everything built actually reachable?
   Fix any gaps, then:
4. /review-externally M[N]     — code quality: bugs, logic errors, security issues?
   Fix any findings.

All four commands take the same milestone ID. IDs map to file prefixes by
stripping the M and zero-padding (M7a → 07a_*.prompt.md); the commands accept
either form.
Run them in order — each assumes the previous step is done.
Do NOT skip any of these. All are mandatory.

Alternative: /build-verify-review runs all four phases for a pack in one session.
```
