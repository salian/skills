---
description: Run an external code review (currently via Codex) on prompt-pack commits or recent changes. Encapsulates tool discovery, scope detection, invocation, and result triage. This is NOT optional — run it after every prompt pack build.
argument-hint: "[prompt-pack ID e.g. M12 | --base HEAD~N | --uncommitted]"
allowed-tools: Bash(which:*), Bash(command:*), Bash(codex:*), Bash(npx:*), Bash(ls:*), Bash(bash:*), Bash(git:*), Bash(grep:*), Bash(pkill:*), Read, Write, Edit
---

# Codex External Review

Run an external Codex review on the current changes. This catches bugs, logic errors, and security issues that self-review misses.

**This step is mandatory, not optional.** It consistently finds issues that pass self-review, typecheck, lint, and tests. Do not skip it.

## Process

### Step 1: Find the Codex binary

Codex review can run dozens of times per session in autonomous mode, so the discovery path matters for total wall-clock. Try the cheapest invocations first.

**Cached-path fast path.** If `~/.claude/.codex-path` exists, read it. If the path it points to still exists, use that — skip discovery entirely.

**Discovery (run in order, stop at the first success):**

```bash
# 1. Direct PATH check — fastest when codex is globally installed (`npm install -g @openai/codex`).
#    Returns in ~1ms when present. Use this binary directly: no Node startup overhead per call.
command -v codex

# 2. Common install locations — direct binary, also fast.
ls ~/.local/bin/codex /usr/local/bin/codex /opt/homebrew/bin/codex 2>/dev/null

# 3. Login shell (picks up ~/.bashrc / ~/.zshrc exports that non-login shells miss).
bash -l -c 'command -v codex'

# 4. npx --no-install (cached package, but pays Node + npx resolution overhead per call ~1-3s).
#    Use only if no direct binary is available.
npx --no-install @openai/codex --version

# 5. npx without --no-install (will fetch the package if not yet cached — slow first time).
npx @openai/codex --version
```

Set `CODEX_CMD` to whichever invocation worked. After a successful discovery, **cache the resolved path**: write the absolute binary path (or the full invocation like `npx --no-install @openai/codex`) to `~/.claude/.codex-path`. Future invocations skip discovery and save ~100ms-3s per round.

If ALL paths fail, tell the user: "Codex is not installed. Install with `npm install -g @openai/codex` to enable external review." Then stop — do not silently skip.

### Step 2: Determine review scope

Resolve `$ARGUMENTS` to a `--base` commit using this priority:

#### 2a. Prompt-pack ID given (e.g. `M12`, `M7`, `sprint-3`)

To read the pack file itself, normalize the ID to the file-name prefix first: strip a leading `M`/`m` and zero-pad to two digits (`M12` → `docs/prompt-packs/12_*.prompt.md`, `M7a` → `07a_*.prompt.md`).

Find the first commit belonging to this prompt pack by grepping the git log for the scope or milestone keyword:

```bash
# Find commits matching the prompt-pack scope (e.g. "lead-capture" for M12, "auth" for M1)
# Look at the prompt pack file to determine the conventional commit scope
git log --oneline | grep -i "$SCOPE_OR_MILESTONE"

# Find the oldest such commit — that's the base
git log --oneline --reverse --grep="$SCOPE_OR_MILESTONE" | head -1
```

If the grep finds NO matching commits (scope vocabulary drifted, or commits don't mention the milestone), do not stall — fall through to the 2d auto-detect logic below. Note also that `HEAD~$N` includes any interleaved commits from other scopes; that's acceptable (a slightly wider review is harmless), but mention the widened scope in the report.

If that commit is found, count how many commits from HEAD to that commit:

```bash
BASE_COMMIT=$(git log --oneline --reverse --grep="$SCOPE_OR_MILESTONE" | head -1 | cut -d' ' -f1)
N=$(git rev-list --count $BASE_COMMIT..HEAD)
# Add 1 to include the base commit itself
N=$((N + 1))
```

Use `--base HEAD~$N`.

#### 2b. Explicit `--base HEAD~N` given

Use as-is.

#### 2c. `--uncommitted` given

Review staged + unstaged changes only.

#### 2d. No arguments given — auto-detect

Try to find the prompt-pack scope from recent commits. Fall back in this order:

1. Check if there are uncommitted changes — if yes, review those first
2. Look at the last 20 commits for a common scope pattern (e.g. most commits share `feat(lead-capture):`) — if found, use all commits with that scope as the range
3. Fall back to `--base HEAD~5` (last 5 commits)

#### Run the review

> ⚠️ **Codex CLI quirk**: `codex review --base <BRANCH> "<prompt>"` errors with
> `the argument '--base <BRANCH>' cannot be used with '[PROMPT]'`. The
> `--base` form does NOT accept a custom prompt — Codex uses its built-in
> review prompt and ignores yours. Don't waste a round trip retrying with
> a prompt argument; either drop the prompt entirely (recommended) or use
> `--uncommitted` (which DOES accept a prompt).

```bash
# With base — NO prompt argument allowed:
$CODEX_CMD review --base HEAD~$N

# Uncommitted — prompt IS allowed:
$CODEX_CMD review --uncommitted "Review the following git diff for bugs, logic errors, security issues, and missed edge cases. Be concise — only flag real problems, not style preferences."

# Single commit — NO prompt argument allowed (same constraint as --base):
$CODEX_CMD review --commit <SHA>
```

If you need to bias Codex's attention toward specific concerns on a `--base` review (e.g. "focus on concurrency in archivePillar"), brief it via the spawning agent's prompt or include the guidance inline in the changes themselves (commit message, code comment) — the `--base` form will pick that up from the diff.

Use the default Codex model selection. If you need to force Mini on a ChatGPT-linked Codex CLI, use `-m gpt-5.1-codex-mini`; do not use `--model o4-mini`.

#### Run it to a FILE, `timeout`-wrapped, via your runner's backgrounding

A `codex review` takes minutes and emits a large reasoning trace. Always redirect to a file, bound it with `timeout`, and run it detached:

```bash
# Stable, knowable output path (scratchpad or repo-local tmp).
OUT=/tmp/codex-review-$(git rev-parse --short HEAD).txt
# `timeout` converts a hang into a real exit (code 124) so a stuck run still
# TERMINATES and notifies, instead of waiting forever (see Step 2.6).
timeout 900 $CODEX_CMD review --base HEAD~$N > "$OUT" 2>&1
```

**Launch it through your runner's native backgrounding** (e.g. Claude Code's background Bash / a job that emits a completion event), NOT a hand-rolled poll loop. That is the better completion *trigger* for two reasons: it is event-driven (no `sleep` loop), and it tracks the **specific PID you launched**, so it is immune to the lingering-children trap below — it fires when your wrapped command returns, which is after the verdict is written. Capture the exit code from that completion event.

### Step 2.6: Detect completion reliably (the verdict block, NOT the process)

This is the step that most often goes wrong: runs that wait forever after Codex already answered, or that score a crash as "clean". Two independent questions, two different signals:

| Question | Best signal |
|---|---|
| Has the run finished? | your runner's **backgrounding completion event** + the exit code (primary); the file-watch loop below only as a fallback for runners without native backgrounding |
| Is there a real verdict (vs crash / usage-limit / truncation)? | the **verdict block in the output file** — always required |

**Never use process liveness as the done-signal.** `codex` leaves **lingering child processes** after it has written its verdict, so `pgrep -f "codex review"` / `kill -0 <pid>` / `ps` keep reporting **RUNNING** long after the review is finished. Polling the process tree hangs you indefinitely — this is the #1 cause of "waited forever even though Codex was done." Runner-native backgrounding avoids it (it watches your PID, not a by-name match); a `pgrep` loop does not.

**The exit code alone is also insufficient.** It proves the wrapped command returned, not that a *verdict* was produced — Codex can exit 0 having errored or hit a usage limit, and `timeout` exits 124 on a hang. So on completion you MUST read the file.

**The authoritative content signal is the verdict block at the END of the output file.** Codex prints a structured final block: a lone `codex` line, then either a `Full review comments:` section (with `[P0]`/`[P1]`/`[P2]`/`[P3]` findings) or a no-issues statement (e.g. "did not identify any … issues").

**Fallback only — if your runner has NO native backgrounding,** watch the file directly, with a hard ceiling (~15 min, matching the `timeout`):

```bash
# Stop on a verdict OR a failure marker; cap the wait. Use your runner's
# file-watch/until primitive; this is the condition to test.
until grep -qiE 'Full review comments:|did not identify any|no (discrete|issues)|\[P[0-3]\]|usage limit|rate limit|stream (error|disconnected)|^error:|panic' "$OUT"; do
  sleep 5
done
```

**On completion (either path), classify into exactly one of three terminal states before triaging — never default an ambiguous file to "passed":**

| State | How to recognise it | Action |
|-------|--------------------|--------|
| **Verdict returned** | file contains `Full review comments:` + `[P#]` findings, or an explicit no-issues line | Proceed to Step 3 (triage / clean-pass). |
| **Crashed / blocked** | exit code ≠ 0, OR file contains `usage limit` / `rate limit` / `error:` / `stream error` / `panic` and NO verdict block | Record outcome `blocked-*` (e.g. `blocked-codex-usage-limit`) — do NOT score as passed. Wait for the limit to reset / fix the cause, then re-run. |
| **Hung** | timeout reached with neither a verdict nor a failure marker | Kill the run (and its lingering children: `pkill -f "codex review"`), then retry ONCE. If it hangs again, surface to the user — don't loop silently. |

Only the first state is a real review result. A clean pass requires a *positive* no-issues verdict in the file — silence, a truncated file, or a bare exit code is **not** a pass.

### Step 3: Triage findings

For each issue Codex raises:

| Verdict | Action |
|---------|--------|
| **Legitimate bug or security issue** | Fix it immediately, then re-run Codex to confirm the fix |
| **Logic error or missed edge case** | Fix it immediately |
| **Real issue, but out of this pack's scope** | Don't fix here and don't bury it as "dismissed" — append it to `docs/dev/backlog.md` under `## Open` (what / why-deferred / candidate-home / source `<pack-id> review-externally (<date>)`), deduped. This is for genuine issues that belong to a later pack, not false positives. |
| **Style preference or false positive** | Ignore — note it as dismissed |
| **Unclear** | Read the relevant code and make a judgment call |

> A "backlogged" finding is NOT a "legitimate finding" for loop purposes — like a dismissal, it does not trigger a re-review round. It is fixed later by the pack that owns it, not now.

**When a finding touches a classification, invariant, or state machine, re-derive the whole decision table and fix every cell in one pass** — don't patch only the flagged case. Local case-patching of these is the main driver of refinement-of-refinement review rounds: each patch exposes the next adjacent case.

### Step 4: Report

```
## Codex Review Results

- **Scope**: [uncommitted | HEAD~N]
- **Findings**: N total (M legitimate, K dismissed, J backlogged)
- **Fixed**: [list of fixes applied]
- **Dismissed**: [list with one-line reason each]
- **Backlogged**: [list with the `docs/dev/backlog.md` id + one-line reason each, or "none"]
```

## Rules

- **Read the cached path first.** `~/.claude/.codex-path` holds the resolved invocation from prior runs; use it directly when present. Run the Step 1 discovery only when that file is missing or the path it points to no longer resolves.
- **Don't pass a prompt with `--base` or `--commit`.** Codex's CLI errors out (`'--base' cannot be used with '[PROMPT]'`); see Step 2 quirk note. Drop the prompt for those review modes.
- **Don't route through the `codex:codex-rescue` subagent for a basic review.** It runs its own ready-check that has historically reported "Codex is not installed" even when `npx --no-install @openai/codex` works fine — costing a wasted round trip. Just call `codex review` directly via Bash. (If you genuinely need rescue/diagnosis behaviour, the subagent is still the right tool.)
- **Never silently skip.** If Codex isn't installed, say so explicitly. The user should know this step was missed.
- **Detect completion by the verdict block in the output file, never by process liveness.** `codex` leaves lingering child processes after writing its verdict, so `pgrep`/`kill -0`/`ps` report RUNNING when it's actually done — polling them hangs forever. Watch the file for the terminal verdict (`Full review comments:` + `[P#]`, or an explicit no-issues line) or a failure marker, under a hard timeout (~15 min). See Step 2.6.
- **A pass requires a positive no-issues verdict, not silence.** Exit code 0 alone is not a pass — Codex can exit 0 on an error or usage limit. If the file has no verdict block and no failure marker after the timeout, treat it as hung (kill + retry once), not clean. A usage-limit/error file is `blocked-*`, never `passed`.
- **Fix legitimate findings before moving on.** Don't just report them — fix them. The point of external review is to catch things self-review missed.
- **Re-run after fixing.** Fixes can introduce new issues. Run Codex again on the fixes to confirm they're clean.
