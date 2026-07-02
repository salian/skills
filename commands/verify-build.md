---
description: Post-prompt-pack build completeness audit. Verifies every deliverable, test, and acceptance criterion specified in a prompt pack was actually built. Run this BEFORE /verify-wiring.
argument-hint: "[prompt-pack ID, e.g. M12]"
allowed-tools: Read, Glob, Grep, Bash(grep:*), Bash(ls:*), Bash(find:*), Agent
---

# Build Completeness Audit

Verify that everything specified in a prompt pack was **actually built** — every file, function, route, test, background job, and acceptance criterion. This is the completeness check: "did we build what the spec says?" Run this BEFORE `/verify-wiring`, which checks connectivity.

## Why this exists

It's common to finish a prompt pack and believe everything is done because the code compiles, types check, and tests pass. But prompt packs specify dozens of deliverables across multiple sections, and it's easy to miss one entirely — especially when specs reference additional requirements via `docs/specs/` files. This audit walks every section of the prompt pack and checks that each deliverable exists.

## Inputs

- `$ARGUMENTS` — the prompt-pack identifier (e.g. `M12`, `sprint-3`). If empty, ask the user.

## Process

### Phase 0: Locate and read the spec

#### 0a. Find the prompt pack

**Normalize the ID before globbing.** Milestone IDs are written `M12` / `M7a`, but pack files are named by zero-padded numeric prefix (`12_LEAD_CAPTURE.prompt.md`, `07a_TESTING_CORE.prompt.md`). Strip a leading `M`/`m` and left-pad the number to two digits (`M12` → `12`, `M7a` → `07a`), then glob for that prefix. Both forms are accepted as `$ARGUMENTS`.

Search for the prompt pack matching the normalized ID:

```
docs/prompt-packs/<NN>*
docs/prompts/<NN>*
prompts/<NN>*
.prompts/<NN>*
```

If the glob matches multiple packs (e.g. `12` matching both `12a_` and `12b_`), list the matches and ask which one — never guess. If not found, ask the user for the path.

#### 0b. Read referenced specs

Prompt packs have a "Spec References" section listing `docs/specs/` files with section references (e.g. `docs/specs/lead-capture.md — §5`). Read every referenced spec file and the specific sections cited. Features defined in specs but not restated in the prompt pack are still required deliverables.

#### 0c. Read CLAUDE.md

Check for project conventions that affect what "correctly built" means: testing conventions, file naming, component library requirements, security patterns, etc.

### Phase 1: Audit deliverable files (Section 7)

Prompt packs list required files in their "Required Deliverables" section (typically section 7), usually as directory trees with descriptions.

For each file listed:

1. **Check existence**: Does the file exist at the specified path? (Glob for it — prompt packs sometimes use `(dashboard)` route groups that map to actual `dashboard/` dirs, or vice versa.)
2. **Check exports**: Does the file export the functions/handlers described? Grep for the function names mentioned in the spec.
3. **Check spec compliance**: Read the file and verify it implements what the spec describes — correct parameters, correct behavior, correct data model. This is a read + judgment check, not just a grep.

Track results per file:

| File | Exists | Exports correct | Matches spec | Notes |
|------|--------|----------------|-------------|-------|

### Phase 2: Audit tests (Section 10)

Prompt packs list required test files and what they should cover.

For each test file listed:

1. **Check existence**: Does the test file exist?
2. **Check coverage**: Read the test file. Does it have test cases for the scenarios listed in the spec? Match each bullet point in the spec's testing section against actual `it()` / `test()` descriptions.
3. **Check passing**: If feasible (not too slow), run the specific test file and report pass/fail.

4. **State-matrix check**: If the pack's Section 10 specifies a state matrix, verify:
   - The matrix test file exists at the specified path
   - The cell count matches the declared dimensions (e.g. 4 × 3 × 2 = 24) — count the `it.each` / `describe.each` cases
   - Every skipped cell carries an inline justification comment; a skipped cell with no comment is a gap
   - Cells are independent assertions, not one test with branching

Track results:

| Test file | Exists | Scenarios covered | Passing | Missing scenarios |
|-----------|--------|------------------|---------|------------------|

### Phase 3: Audit background jobs and scheduling (Sections 7, 6)

For each background job mentioned in the prompt pack:

1. **Handler exists**: Does the job handler file exist with the correct function?
2. **Job type/queue**: Does it handle the correct job types (e.g. `score_single`, `rescore_batch`, `rescore_periodic`)?
3. **Cadence matches spec**: If the spec says "daily" or "weekly", verify the cron expression matches.

### Phase 3.5: Audit failure modes (Section 11)

If the pack has a populated Failure Modes table, walk every row:

1. **Mitigation exists**: the named guard function/pattern/config exists (grep for the function name or read the config path)
2. **Test exists and asserts the failure is blocked**: the named test file exists and its assertions target the failure mode (e.g. SSRF request rejected), not just the happy path
3. **Skipped rows are justified**: every N/A row carries its inline reason; bare "N/A" is a gap
4. **Stateful-protocol packs carry interleaving tests**: if the pack implements a claim/lock/retry/dedupe protocol, confirm each concurrency invariant has a matching interleaving test (concurrent actors / crash-resume / double-fire), not only a state matrix. A matrix with no interleaving test for a concurrency invariant is a gap.

Missing mitigation tests are **build failures**, not warnings. If the pack omitted Section 11 entirely, check its one-line justification is plausible (pure-transform / schema-only / UI-only / type-only); if the pack ships an outbound fetch, worker, pipeline, or read-then-write sequence with no Section 11 failure-modes table, flag that as a spec defect in the report.

### Phase 3.6: Audit sibling write-path parity

Phases 1–5 verify the spec's *listed* deliverables. This phase verifies an *implicit* requirement the spec rarely restates: a new code path that writes a shared persistent entity (model/table/aggregate) or repeats an existing operation (create a user, enqueue a job, send a message, charge an account) must enforce the **same invariants the existing paths already enforce**. A gap here compiles, lints, and passes the spec's own tests — yet is a real build defect. It's the audit mirror of `/build` Phase 2.5e.

**Skip** if the pack is read-only/transform-only or introduces a brand-new entity with no existing writers. Otherwise:

1. **Identify the new write(s)/operation(s)** the pack added (from the Phase 1 deliverable files).
2. **Find the existing performers** of the same write/operation — grep the shared create/update/enqueue/charge call, service, or table for other call sites.
3. **Diff the guard sets** — read the code (judgment, not just grep). For each guard category an existing performer applies, confirm the new path applies it too:
   - quota/limit · input validation · authorization/tenancy · compliance/suppression/PII · normalization/coercion (schema constraints, enums/whitelist, required-field defaults) · idempotency/dedup · audit/observability
4. Mark each ✅ applied / ⚠️ partial / ❌ missing, citing the `file:line` of the sibling that enforces it.

A ❌ or ⚠️ is a **build gap**, not a warning — count it in the Gaps list and fix before `/verify-wiring`, the same as any Section-7 miss. A guard that genuinely doesn't apply to the new path (e.g. it can only merge, never create, so a create-cap is N/A) is ✅ with a one-line reason — not a gap.

### Phase 4: Audit acceptance criteria (Section 12)

Prompt packs have an "Acceptance Criteria" section — a checklist of user-visible behaviors. Walk each criterion:

1. **Read the criterion** (e.g. "New leads auto-scored on submission, score visible in CRM")
2. **Trace the implementation**: grep/read to confirm the end-to-end path exists that satisfies this criterion
3. **Mark as MET or UNMET** with evidence (file path + line, or "no implementation found")

This is the most judgment-intensive phase. Don't just check that code exists — verify it does what the criterion says.

### Phase 5: Audit spec-referenced requirements

For each spec file read in Phase 0b, check requirements from the specific sections cited that may not be restated in the prompt pack's deliverable list. Common sources of missed requirements:
- Spec subsections with detailed parameter lists (e.g. "8 scoring parameters" — verify all 8)
- Error handling requirements (e.g. "graceful fallback if config missing")
- Data model constraints (e.g. "same email + org within 24h")
- Security requirements (e.g. "formula injection prevention in CSV")
- Performance requirements (e.g. "time-limited signed URLs")

### Phase 6: Report

Output a structured summary:

```markdown
## Build Completeness Audit: [prompt-pack ID]

### Deliverable Files
| # | File | Exists | Exports | Spec match | Issue |
|---|------|--------|---------|-----------|-------|
| 1 | src/lib/leads/scoring.ts | ✅ | ✅ | ✅ | — |
| 2 | src/lib/leads/analytics.ts | ✅ | ✅ | ⚠️ | byPillar returns NULL names |
| 3 | src/app/dashboard/foo/page.tsx | ❌ | — | — | File not created |

### Tests
| # | Test file | Exists | Coverage | Passing | Missing |
|---|-----------|--------|----------|---------|---------|
| 1 | scoring.test.ts | ✅ | 6/8 | ✅ | weight bounds, threshold ordering |

### Background Jobs
| # | Job | Handler | Types | Cadence | Issue |
|---|-----|---------|-------|---------|-------|

### Failure Modes (Section 11)
| # | Mode | Mitigation exists | Test exists | Test asserts block | Issue |
|---|------|------------------|-------------|--------------------|-------|

### Sibling Write-Path Parity
| # | New write path | Guard category | Sibling enforcing it | New path | Issue |
|---|----------------|----------------|----------------------|----------|-------|
| 1 | bulk importer that creates accounts | plan/seat limit | signup + admin-create paths | ❌ | bulk import bypasses the cap |

### Acceptance Criteria
| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | New leads auto-scored on submission | ✅ MET | lead-scoring.ts handles score_single |
| 2 | CSV export filtered + full | ⚠️ PARTIAL | Full export works, filtered missing dateFrom/dateTo |

### Spec-Referenced Requirements
| # | Spec | Requirement | Status | Issue |
|---|------|------------|--------|-------|

---

### Summary
- **Deliverables**: N/M built
- **Tests**: N/M exist, N/M passing, N scenarios missing
- **Acceptance criteria**: N/M met, N partial, N unmet
- **Failure modes**: N/M mitigated and tested
- **Write-path parity**: N/M guards applied, K gaps
- **Spec requirements**: N issues found

### Gaps
[Numbered list of everything missing or incomplete, with file paths and one-line fix descriptions]
```

If gaps are found, end with: **"Fix all N gaps?"**

**Exception:** when invoked by `/build-verify-review` (or any autonomous orchestrator), do NOT ask — report the gaps and end the audit. The orchestrator applies fixes and re-runs; asking "Fix all N gaps?" violates its autonomy contract.

## Rules

1. **Read the spec files, not just the prompt pack.** The prompt pack's "Spec References" section points to detailed specs that contain requirements not restated in the prompt pack. Missing these is the #1 cause of incomplete builds.

2. **Check function signatures, not just file existence.** A file can exist but be missing half its exports. Grep for every function name mentioned in the spec's deliverable descriptions.

3. **Match acceptance criteria against behavior, not code presence.** "CSV export filtered + full" means there must be a way to export with filters AND without — don't just check that an export function exists.

4. **Count parameters explicitly.** If the spec says "8 scoring parameters," count them. If it says "by article, pillar, form variant," verify all three groupings exist. Specs are precise; verification must be equally precise.

5. **Check test scenario coverage, not just test file existence.** A test file with 2 tests doesn't satisfy a spec that lists 6 scenarios. Read the `test()` / `it()` descriptions and match them against the spec's testing bullets.

6. **Flag partial implementations.** If a function exists but is missing a parameter or edge case the spec requires, mark it as ⚠️ PARTIAL, not ✅. Partial is worse than missing — it's harder to spot.

7. **Run all phases before reporting.** The user needs the complete picture.

8. **Don't check for things the prompt pack explicitly excludes.** Read the "Non-Goals" and "Overbuild Prevention" sections. If the spec says "Do NOT implement X," verify X was NOT built.

9. **A deliverable confirmed out-of-scope-by-spec is NOT a gap — but if it's a real future need, backlog it.** When the audit shows a referenced requirement was deliberately deferred (a non-goal, or owned by a later pack), do not list it under Gaps. If no pack yet scopes it and it's a genuine future need, add a one-line entry to `docs/dev/backlog.md` (`## Open`, source `<pack-id> verify-build (<date>)`, deduped) so it isn't silently lost. Things a pack already schedules elsewhere don't need an entry.

10. **Parity gaps are build gaps.** A new shared-entity write that skips a guard its sibling paths enforce (cap, validation, auth, compliance, normalization, idempotency) is a defect even though it compiles and the spec's tests pass — list it in Gaps, don't dismiss it.

## Relationship to other commands

This command is step 3 in the prompt-pack workflow:

1. `/create-prompt-packs` — generate build instructions from specs
2. *(build the code)*
3. **`/verify-build M12`** — verify everything in the spec was built (this command)
4. `/verify-wiring M12` — verify everything built is connected end-to-end
5. `/review-externally M12` — external code-quality review of the milestone's commits
