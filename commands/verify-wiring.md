---
description: Post-prompt-pack integration wiring audit. Verifies every feature built by a prompt pack is reachable end-to-end — not just structurally present, but actually wired into navigation, API callers, schedulers, and UI. Works across any web project that uses prompt packs.
argument-hint: "[prompt-pack ID, e.g. M12]"
allowed-tools: Read, Glob, Grep, Bash(grep:*), Bash(ls:*), Bash(find:*), Agent
---

# Integration Wiring Audit

Verify that every feature in a prompt pack is **reachable end-to-end** — not just structurally correct. This catches the "built but unwired" class of bugs: orphaned pages, orphaned API routes, unregistered scheduled jobs, UI fields that never send their data, and library functions with zero callers.

## Why this exists

Structural verification (files exist, types check, tests pass) misses a critical class of bugs where code is **built but never connected**. Common patterns:
- A page exists but no navigation links to it
- An API route exists but no UI or worker calls it
- A job handler exists but isn't registered in the scheduler
- A UI component has inputs that are never included in the form submission
- A library function is exported but never imported by production code

## Inputs

- `$ARGUMENTS` — the prompt-pack identifier (e.g. `M12`, `sprint-3`, `auth-phase-2`). If empty, ask the user.

## Process

### Phase 0: Discover project structure

Before running checks, detect the project's framework and conventions. This makes the audit work across different stacks.

#### 0a. Locate prompt packs

**Normalize the ID before globbing.** Milestone IDs are written `M12` / `M7a`, but pack files are named by zero-padded numeric prefix (`12_LEAD_CAPTURE.prompt.md`, `07a_TESTING_CORE.prompt.md`). Strip a leading `M`/`m` and left-pad the number to two digits (`M12` → `12`, `M7a` → `07a`), then glob for that prefix. Both forms are accepted as `$ARGUMENTS`.

```
# Common locations — try in order
docs/prompt-packs/<NN>*
docs/prompts/<NN>*
prompts/<NN>*
.prompts/<NN>*
```

If the glob matches multiple packs (e.g. `12` matching both `12a_` and `12b_`), list the matches and ask which one — never guess. If not found, ask the user for the path.

#### 0b. Detect framework and conventions

Read `package.json`, `CLAUDE.md`, project config files, and the directory tree to determine:

| Convention | How to detect | Examples |
|-----------|--------------|---------|
| **Page files** | Framework config / directory structure | Next.js: `page.tsx` in `app/`. SvelteKit: `+page.svelte` in `routes/`. Nuxt: `.vue` in `pages/`. Remix: `route.tsx` in `routes/`. Plain React/Vue SPA: component files registered in router config. |
| **API routes** | Framework config / directory structure | Next.js: `route.ts` in `app/api/`. Express/Fastify: files in `routes/` or `controllers/`. Django: `views.py` + `urls.py`. Rails: `controllers/` + `routes.rb`. |
| **Source directories** | Top-level folders | `src/`, `app/`, `lib/`, `server/`, `packages/` |
| **Job/scheduler files** | Grep for cron patterns, BullMQ, Agenda, node-cron, Celery, Sidekiq | Files containing cron expressions (`* * * * *`), `registerScheduledJobs`, `schedule.every`, `@periodic_task` |
| **Security middleware** | Grep for CSRF, auth guard, middleware patterns | `requireCsrf`, `csrfHeaders`, `csrf_protect`, `@login_required`, `authenticate`, `authGuard` |
| **Navigation/routing** | Framework link components | `<Link>`, `<a>`, `<router-link>`, `<NuxtLink>`, `navigate()`, `router.push()` |

Store these as variables for use in all subsequent checks.

#### 0c. Read CLAUDE.md for project-specific conventions

If `CLAUDE.md` exists, scan for:
- Middleware / request handling rules
- Auth architecture patterns
- CSRF requirements
- Job scheduling patterns
- Any custom wiring conventions

These inform what "correctly wired" means for this project.

### Phase 1: Extract the feature inventory

1. Read the prompt pack file.
2. Read every spec file referenced in the prompt pack (look for "Spec References", "Related specs", "See also" sections, or file path references like `docs/specs/...`).
3. **Walk the pack's own Section 8 wiring checklist first.** Every row (export → caller → grep pattern) goes straight into the inventory — the pack already did the hard work of specifying expected callers and grep patterns; verify each row exactly as written before deriving anything yourself. A Section 8 row whose grep pattern returns no production hits is an automatic FAIL.
4. **Add Section 11 mitigations to the inventory.** Every guard function/pattern named in the pack's Failure Modes table (e.g. `assertPublicUrl`, `withIdempotencyKey`) is a `library-function` entry — it must be called by the code it protects (Check E), not merely exist.
5. Build the rest of the **feature inventory** — a flat list of every distinct user-facing or system-facing feature the prompt pack specifies beyond what sections 8/11 already cover. For each feature, record:
   - **What**: one-line description
   - **Type**: `page` | `api-route` | `scheduled-job` | `ui-action` | `library-function` | `webhook` | `cli-command`
   - **Expected location**: predicted file path based on detected project conventions

### Phase 2: Run the wiring checks

For every feature in the inventory, run the applicable checks. Track results in a table.

---

#### Check A — Navigation reachability (for `page` features)

**Question**: Can a user navigate to this page from somewhere in the app?

For each new page file, grep the source tree for links or navigation calls pointing to that route:

```
# Adapt pattern to framework:
# Next.js / React Router:  href="/path" or to="/path"
# Vue Router:              :to="/path" or router.push('/path')
# SvelteKit:               href="/path"
grep -r '"/path/to/page"' [source-dirs] --include='*.tsx' --include='*.vue' --include='*.svelte' -l
```

**PASS** = at least one other page/component links to it.
**FAIL** = zero inbound links → orphaned page.

---

#### Check B — API route reachability (for `api-route` and `webhook` features)

**Question**: Does any code actually call this endpoint?

For each API route handler, grep for calls to that endpoint:

```
# Client-side fetches
grep -r '"/api/path/to/route"' [source-dirs] --include='*.ts' --include='*.tsx' --include='*.vue' --include='*.js' -l

# Server-side / worker calls
grep -r '"/api/path/to/route"' [job-dirs] [worker-dirs] --include='*.ts' --include='*.js' -l
```

Also check: SDK client files, webhook registration code, external service configs.

**PASS** = at least one caller exists outside the route file itself.
**FAIL** = zero callers → orphaned route.

---

#### Check C — Scheduler registration (for `scheduled-job` features)

**Question**: Is this job actually registered to run, not just defined?

The detection depends on the stack:

| Stack | Where jobs are registered | What to grep for |
|-------|--------------------------|-----------------|
| BullMQ | Scheduler/worker setup file | Job name in `add()`, `upsert()`, or registration array |
| node-cron | Startup file or cron config | `cron.schedule('pattern', handler)` |
| Agenda | Agenda setup file | `agenda.define()` + `agenda.every()` |
| Celery | `celery.py` or `tasks.py` | `@periodic_task` or `CELERYBEAT_SCHEDULE` |
| Sidekiq | `sidekiq.yml` or initializer | Job class in schedule config |
| GitHub Actions | `.github/workflows/` | Workflow file with `schedule:` trigger |
| Custom | Varies | Grep for cron patterns near job name |

**PASS** = job name/handler appears in both definition AND scheduling/registration.
**FAIL** = handler exists but isn't registered to run → will never execute.

---

#### Check D — UI-to-API completeness (for `ui-action` features)

**Question**: Does clicking this button actually do something end-to-end?

For each user-triggered action, trace the full chain:

1. **Entry point**: a button, link, or form in a UI component. Grep for the label text or handler function name.
2. **Client action**: a fetch/axios/http call to an API endpoint. Grep for the URL in the component file.
3. **Security middleware**: For mutating requests (POST/PATCH/PUT/DELETE), verify the appropriate security pattern is applied:
   - CSRF tokens/headers on the client call (if project requires it — check CLAUDE.md)
   - Auth guards on the server handler
   - Input validation at the boundary
4. **Server handler**: the API route file exports the matching HTTP method. Read the file and confirm.
5. **Persistence/effect**: the handler produces a side effect (DB write, queue enqueue, external API call, file write). Read the handler to confirm.

**PASS** = every link in the chain exists and connects correctly.
**FAIL** = any broken link. Report which specific link is broken.

---

#### Check E — Zero-caller detection (for `library-function` features)

**Question**: Is this function actually used by production code?

For each new exported function:

```
grep -r 'functionName' [source-dirs] --include='*.ts' --include='*.tsx' --include='*.js' --include='*.py' --include='*.rb' -l
```

**PASS** = imported/called by at least one non-test production file (other than the file that defines it).
**FAIL** = only matches are the defining file and/or test files → dead code in production.

> **Important**: test files importing a function do NOT count. Only production code callers count.

---

#### Check F — Browser preview spot-check (when dev server is available)

**Question**: Does the feature actually render and load data?

Check for a running preview server using `preview_list`. If one is available:

1. Navigate to each new/modified page
2. Verify new buttons, links, cards, and sections appear in the DOM snapshot
3. Check browser console for errors
4. Verify data loads — not stuck on "Loading..." or empty state when data should exist

If no preview server is running, mark as **SKIPPED** (not FAIL).

---

### Phase 2.5: Spec Gap Audit (advisory)

Checks A–F verify the spec was implemented correctly. This phase asks the harder question: **was the spec itself complete?** A feature can be perfectly wired per the prompt pack and still ship with obvious holes a user would hit on day one.

Unlike A–F, these checks are judgment-based, not grep-deterministic. Treat findings as **questions for the user**, not failures. Do not auto-fix them. Do not block `/build-verify-review` on them.

Run each check against the feature inventory AND the surrounding product context (sibling features, parent flows, the broader codebase).

#### Check G — Sibling parity

For each new entity or feature, scan for analogous entities that already exist in the codebase. List what they have that this one doesn't.

Example prompts:
- "Leads have an export endpoint, bulk delete, and an analytics page. This new `citations` entity has none of those — intentional?"
- "Every other dashboard list page has filter + sort + pagination. The new `/dashboard/audits` page has only a static list — intentional?"

#### Check H — CRUD + lifecycle completeness

For each new entity, walk the lifecycle: **create → read (list) → read (detail) → update → delete → archive/restore → permissions → audit log**. Flag missing operations that the entity's nature suggests should exist.

Don't be pedantic — a read-only computed view legitimately has no update path. Use judgment.

#### Check I — Cross-cutting concerns

For each new feature, ask whether it should have hooked into existing cross-cutting systems but didn't:
- **Observability**: logging, metrics, error reporting
- **Notifications**: email, in-app, Slack/webhooks where similar features notify
- **Search / global index**: if the project has a search surface, is the new entity indexed?
- **Permissions / roles**: does the feature respect the project's role model, or is it implicitly admin-only?
- **Rate limiting**: do similar endpoints rate-limit and this one doesn't?
- **Caching / invalidation**: does the new write path bust the caches the read path depends on?
- **Internationalization**: if the app is translated, are new strings in the translation pipeline?

#### Check J — User journey reachability

For each new feature, ask: **what's the realistic path a user takes to discover and use this?** Trace from the most likely entry point (dashboard home, onboarding, a notification) to the feature. If the only path is "type the URL directly," that's a gap even if Check A passed (a single buried link technically satisfies A).

#### Check K — Failure-mode coverage

Scan the new code for happy-path-only assumptions:
- What happens on empty state? (zero rows, never-configured)
- What happens on error? (network failure, validation failure, permission denied)
- What happens on partial data? (entity exists but related entity doesn't)
- What happens on the second click? (idempotency, double-submit)

A feature that only renders correctly when everything succeeds is half-built, even if the spec didn't say so.

#### Check L — Spec-vs-product drift

Re-read the prompt pack's stated *purpose* (the "why" or user-story section, not the deliverable list). Ask: does the built feature actually accomplish that purpose, or does it just satisfy the literal deliverables? Flag mismatches.

Example: a pack titled "let users recover from failed payments" that ships a `/billing/failures` page with no recovery action — the deliverable is built, the purpose is not.

---

### Phase 2.6: Persist advisory findings to the backlog

The Phase 2.5 (G–L) findings are advisory and survive past this session ONLY if written down — otherwise they evaporate into the report. Record them durably the same way `CHANGELOG.md` is maintained automatically.

After running Checks G–L, append every **new, real** advisory finding to the project backlog at `docs/dev/backlog.md` (create the file from the template below if it doesn't exist — mirror the changelog convention).

1. **Read `docs/dev/backlog.md` first** (if present). Build the set of titles already tracked under `## Open` and `## Done`.
2. **For each G–L finding that represents a real-but-deferred item**, add an entry under `## Open` — UNLESS an equivalent entry already exists (dedup by the "what", not exact string match: don't re-file the same concern under a new id). Each entry carries:
   - a short stable id (`B<n>`) + one-line title + priority `(low|med|high)`
   - **What** (one or two sentences)
   - **Why deferred** (cite the spec section / future pack that owns it, or "no pack scopes it — by design")
   - **Candidate home** (the pack or lane most likely to consume it — often the project's hardening/housekeeping pass)
   - **Source** (`<pack-id> verify-wiring (<YYYY-MM-DD>)`)
3. **Do NOT backlog** a finding that is (a) already a tracked open item, (b) confirmed resolved-by-design (instead record it as `## Done` / a `resolved-by-design` note so the next audit doesn't re-raise it), or (c) too speculative to act on (leave it in the report only).
4. **Reconcile**: if this pack's build resolved an item already in `## Open`, move it to `## Done` (or delete it) with the resolving pack noted.

Backlog file template (when creating it fresh):

```markdown
# Deferred-Items Backlog

Real-but-not-yet-built items, mostly from `/verify-wiring`'s advisory Checks G–L.
Not bugs (those get fixed in-pack) and not blockers (those halt the pack). Packs
*pull* from this list when built; scope is never *pushed* into a pack speculatively.

## Open

### B1 — <title> (low|med|high)
- **What:** ...
- **Why deferred:** ...
- **Candidate home:** ...
- **Source:** <pack-id> verify-wiring (<date>)

## Done

_(none yet)_
```

This is the only place the audit WRITES; it never fixes the underlying finding. Keep the entries terse — the backlog is a triage queue, not a design doc.

---

### Phase 3: Report

Output a structured summary:

```markdown
## Wiring Audit: [prompt-pack ID]
**Project**: [project name]
**Framework**: [detected framework]
**Features audited**: N

| # | Feature | Type | Check | Result | Detail |
|---|---------|------|-------|--------|--------|
| 1 | User profile page | page | A nav | ✅ | linked from /settings |
| 2 | Export API | api-route | B caller | ❌ | zero fetch() callers |
| 3 | Nightly cleanup job | sched-job | C sched | ❌ | handler exists, not registered |
| ... | ... | ... | ... | ... | ... |

### Gaps Found: N (Checks A–F — these are failures)

[For each gap:]
**Gap N: [title]**
- What's missing: [specific broken link in the chain]
- Fix location: [file path]
- Suggested fix: [one-sentence description]
- Severity: Critical | High | Low

### Spec Gap Questions: M (Checks G–L — these are advisory, not failures)

[For each question:]
**Question M: [title]**
- Observation: [what's present vs. what sibling features / lifecycle / journeys suggest]
- Possible gap: [what the user may have intended but the spec didn't capture]
- Confidence: High | Medium | Low
- Action: ask the user — do not auto-fix

### Summary
[One-line assessment for A–F: "All features wired" or "N critical gaps need fixing before ship"]
[Separate one-line for G–L: "M spec-gap questions to review" or "no spec gaps surfaced"]
```

If A–F gaps exist, end with: **"Fix all N gaps?"**
If only G–L questions exist, end with: **"Review M spec-gap questions? (Recorded in `docs/dev/backlog.md`.)"** — never auto-fix these.

**Exception:** when invoked by `/build-verify-review` (or any autonomous orchestrator), do NOT ask either question — report the gaps and questions and end the audit. The orchestrator applies A–F fixes itself and surfaces G–L questions at the pack boundary; asking violates its autonomy contract.

State which G–L findings were written to `docs/dev/backlog.md` (new ids), which were skipped as already-tracked or resolved-by-design, and which were left report-only as too speculative.

## Rules

These rules are non-negotiable and exist because past audits missed bugs by violating them:

1. **Never report PASS based on file existence.** Always grep for inbound references. A file can exist and be perfectly correct yet completely orphaned.

2. **Test imports don't count as callers.** For Check E, only production code callers matter. A function with 20 test callers and zero production callers is dead code.

3. **Missing security middleware is itself a gap.** For Check D, if the project requires CSRF tokens (or auth guards, or input validation) on mutating routes and a fetch omits them, that's a FAIL — even if the fetch works without them in dev.

4. **Read the specs, not just the prompt pack.** Prompt packs reference spec files. Features are often defined in specs, not restated in the prompt pack. Missing a spec = missing features from the inventory.

5. **Run ALL checks before reporting.** Don't stop at the first failure. The user needs the complete picture to prioritize fixes.

6. **Flag false positives rather than miss real gaps.** A false positive wastes 5 seconds of the user's attention. A missed gap wastes hours of debugging in production.

7. **Trace cross-module boundaries.** The most common wiring bugs live at boundaries: a UI component that calls an API that calls a library function. Check each hop, not just the endpoints.

8. **Check both directions for scheduled jobs.** A job needs (a) a handler that does the work AND (b) a registration that triggers it on schedule. Either one alone = broken.

9. **Never auto-fix Phase 2.5 findings.** Checks G–L produce judgment-based questions, not deterministic failures. Surface them to the user with confidence levels and let the user decide. Auto-fixing spec gaps will (a) bloat the prompt pack's scope, (b) stall the build pipeline on debatable findings, and (c) create churn when the original spec was intentionally narrow.

10. **Keep Phase 2.5 findings separate in the report.** Phase 2.5 questions must never appear in the "Gaps Found" table. They live in their own "Spec Gap Questions" section. `/build-verify-review` and similar pipelines key off the Gaps Found count to decide whether to loop — mixing the two will cause infinite loops on subjective findings.

11. **Persist advisory findings, don't just report them (Phase 2.6).** Every real G–L finding is appended to `docs/dev/backlog.md`, deduped against what's already there, so it survives the session. Writing to the backlog is NOT auto-fixing — it changes no code and does not affect the A–F gap count or the loop decision. A G–L finding that only ever appears in a report is a process bug.
