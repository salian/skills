---
description: Spec→pack coverage audit. Verifies every spec'd capability has an owning prompt pack (or an explicit deferred home) — catches features that silently slipped through pack generation. Run after /create-prompt-packs, after spec changes, and at phase boundaries.
argument-hint: "[optional: spec file or area to scope, e.g. docs/specs/compliance/]"
allowed-tools: Read, Glob, Grep, Bash(grep:*), Bash(ls:*), Bash(find:*), Bash(wc:*), Write, Edit, Agent
---

# Spec→Pack Coverage Audit

Verify that **every capability the specs define has exactly one owning prompt pack** — or an explicit deferred disposition with a named home. This is the reverse of `/verify-build`: that command checks pack→code; this one checks spec→packs.

## Why this exists

Every other command in the pipeline (`/build`, `/verify-build`, `/verify-wiring`, `/review-externally`) takes a **pack** as input. A spec'd feature that landed in **no pack** is therefore invisible to the entire pipeline — nothing ever looks for it, every verifier reports green, and the gap surfaces months later as "wait, where's the login page?" (a real incident: the login page was specced, assumed by every other feature, and owned by no pack).

Two failure mechanisms dominate, and both are mechanical to detect:

1. **Cross-cutting orphans** — capabilities that belong to no module (auth screens, MFA, audit-log viewers, retention jobs, observability, backups), so no module pack claims them. The decomposition is module-shaped; these aren't.
2. **Broken deferral chains** — pack A's Non-Goals say "X → pack B", but pack B never received X (its non-goals defer it elsewhere, or it simply doesn't mention it). Each pack is locally consistent; the chain is broken. Pack-scoped verifiers can never see this.

## Inputs

- `$ARGUMENTS` — optional scope (a spec file or directory). Default: all specs the packs' Section 3 entries reference, plus anything in `docs/specs/`.

## Process

### Phase 0: Locate the corpus

1. Packs: `docs/prompt-packs/*.prompt.md` (same search order as `/build`). Read the README for execution order.
2. Specs: `docs/specs/**` (or the project's spec locations per CLAUDE.md), the build-sequence/roadmap doc, and CLAUDE.md.
3. `docs/prompt-packs/_COVERAGE_REGISTRY.md` if present (prior run or generation-time).
4. `docs/dev/backlog.md` `## Open` titles (a gap already tracked there is TRACKED, not ORPHANED).

### Phase 1: Build the spec capability inventory

Enumerate every buildable capability from the specs, prioritizing **enumerable inventories** (they make the audit mechanical, not judgment-based):

- **Page-definition matrices / nav trees** — one item per page/nav node, including its named columns and actions.
- **End-to-end workflows** — one item per workflow, noting each step.
- **Entity state machines** — one item per machine, noting each state/transition.
- **Module feature sections** — behaviours and engines the page matrix won't show (detection rules, schedulers, imports/exports, integrations).
- **NFR sections** — each concrete requirement needing built artifacts (MFA, audit-log surfaces, retention/purge jobs, file scanning, observability, backups/DR, rate limiting, offline).
- **The mandatory cross-cutting sweep** (run even if the spec doesn't enumerate them): auth screens, password/account lifecycle, session management, app shell/nav, search, notifications, error/empty states, settings, audit logs, permissions admin, public-endpoint rate limiting, backups/DR, observability, legal/consent.

For a large corpus (>2000 spec lines), fan the inventory + resolution out to parallel subagents by spec section — hand each: its slice, the pack directory, the backlog `## Open` titles, and the verdict definitions below. Instruct them to cite pack-file evidence for every verdict and to report empty-grep patterns for every ORPHANED claim.

### Phase 2: Resolve ownership

For each capability, grep the packs for its distinctive nouns (2–3 synonym variants), then **read the candidate pack's Sections 4/5/7** (never verdict from grep hits alone). Assign exactly one verdict:

- **OWNED** — a pack's Section 7/4 delivers it (cite pack + line).
- **DEFERRED** — a pack's Section 5 (or the registry) defers it to a **named home that actually accepts it** — follow the chain: read the target pack and confirm its Section 4/7 receives the scope. A deferral to a pack that doesn't accept it, to a bare phase ("Expansion", "later"), or to "a depth pass" with no pack is NOT deferred — it's ORPHANED-with-a-paper-trail.
- **TRACKED** — already an entry in `docs/dev/backlog.md` `## Open`.
- **PARTIAL** — the area is owned but a concrete named element (a column set, action, transition, sub-feature) appears in no pack's deliverables and no pack's non-goals.
- **ORPHANED** — no pack delivers it, no closed deferral, no backlog entry.

### Phase 2.5: Adversarial verification

Before reporting, re-check every ORPHANED/PARTIAL verdict yourself (or via a fresh subagent): try 2–3 additional grep spellings (snake_case, hyphenated, synonyms, the entity's schema name from `_SCHEMA_REGISTRY.md`), and read the one or two most plausible owner packs end-to-end. Agents' first-pass orphan claims run a meaningful false-positive rate — a finding that survives adversarial re-checking is worth acting on; one that doesn't is noise that erodes trust in the audit.

### Phase 3: Walk the deferral chains

Independently of Phase 1, extract every "→ M[N]" / "belongs to M[N]" / "(M[N])" deferral from every pack's Section 5, and assert the target pack accepts that scope. Report every broken chain — even for capabilities Phase 1 didn't enumerate.

### Phase 4: Persist

1. **Update (or create) `docs/prompt-packs/_COVERAGE_REGISTRY.md`** with the resolved inventory — one row per capability: `| capability | spec ref | kind | phase | owner pack(s) | status | notes |`. This is the durable artifact; the report is a snapshot.
2. **Backlog every confirmed ORPHANED/PARTIAL finding** in `docs/dev/backlog.md` `## Open` (deduped, with candidate home + source `verify-coverage (<date>)`), so the hardening/backlog-sweep pack can consume them.
3. Do NOT edit packs or specs — this command detects; remediation (assigning owners, generating new packs, extending existing ones) is a decision for the user or a follow-up `/create-prompt-packs` pass.

### Phase 5: Report

```markdown
## Spec→Pack Coverage Audit — <date>

**Corpus**: N spec files (~M lines) × K packs
**Inventory**: N capabilities resolved

| Verdict | Count |
|---|---|
| OWNED | … |
| DEFERRED (chain verified) | … |
| TRACKED (backlog) | … |
| PARTIAL | … |
| ORPHANED | … |

### Orphaned (no owner, no home)
| # | Capability | Spec ref | Phase | Candidate home | Confidence |

### Broken deferral chains
| # | From pack | Deferred scope | Claimed target | Why broken |

### Remediation options
[per finding: assign to existing pack / new pack / DEFERRED row + backlog / won't-build]
```

End by asking the user which remediation route to take for the orphans — except when invoked by an orchestrator, in which case report and stop.

## Rules

1. **Never verdict from grep alone.** A grep hit in a Non-Goals section is the opposite of ownership. Read the candidate pack's Sections 4/5/7.
2. **Phase labels don't excuse gaps.** If packs exist for all phases, an Expansion-phase capability with no pack is still orphaned. Only when a phase genuinely has no packs yet is "not yet decomposed" an acceptable disposition — record it as DEFERRED to that phase's future generation run.
3. **Follow every deferral to its terminus.** "→ M15" is only DEFERRED if M15's text accepts the scope. This single rule catches the largest cluster of real-world orphans.
4. **A capability in the backlog is TRACKED, not fine.** Report the TRACKED count so the user sees the total debt; the backlog-sweep pack is its consumer.
5. **Adversarially re-check every orphan before reporting it.** False orphan claims are expensive — they erode trust and waste remediation effort.
6. **The registry is generated from the SPEC side.** Never build the inventory by summarizing packs — that reproduces the exact blindness this command exists to fix.

## Relationship to other commands

- `/create-prompt-packs` STEP 3.6 generates `_COVERAGE_REGISTRY.md` at generation time; this command audits and refreshes it afterwards.
- `/verify-build` consumes the registry per-pack (a pack must deliver the rows it owns).
- Run cadence: once after generation; after any material spec change; at phase boundaries during long build campaigns.
