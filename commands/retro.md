---
description: Run a session/milestone retrospective — write a dated entry to docs/retros/ capturing what was done, what worked, what didn't, and what to change. Route durable lessons to memory and follow-ups to the backlog/ideas ledger.
argument-hint: "[optional focus, e.g. a milestone name or 'this session']"
allowed-tools: Read, Write, Edit, Grep, Glob, Bash(git *), Bash(date *), Bash(ls *)
---

# Retrospective

Capture a retrospective for the work just completed. This is the `/retro` step referenced by the Self-Improvement Loop in `CLAUDE.md`. Run it after a prompt-pack milestone, a significant piece of work, or when the user is wrapping up.

Focus (optional): `$ARGUMENTS`

## Process

1. **Locate the retros directory.** Default to `docs/retros/`. If it doesn't exist, create it (with a `.gitkeep` if empty). If the project keeps retros elsewhere (check `CLAUDE.md`), use that path instead.

2. **Gather what actually happened** — don't rely on memory of the conversation alone:
   - `date +%F` for today's date (never guess it).
   - `git log --oneline` since the last retro (compare against the newest existing `docs/retros/*.md` date, or the last ~20 commits if none).
   - `git status` / `git diff --stat` for uncommitted work in flight.
   - Skim the relevant spec in `docs/specs/` and prompt pack in `docs/prompt-packs/` if the work maps to one, so "what was intended vs. what shipped" is grounded.

3. **Write the entry** to `docs/retros/<YYYY-MM-DD>-<short-slug>.md`. If a file for today already exists, append a new `##`-level section rather than overwriting. Use this shape:

   ```markdown
   # Retro — <YYYY-MM-DD> — <focus>

   ## What was done
   - <shipped items, tied to commits / pack checklist items>

   ## What worked
   - <approaches, tools, patterns that paid off>

   ## What didn't
   - <dead ends, wrong turns, wasted effort — and the pivot>

   ## Lessons
   - <non-obvious takeaways worth remembering next time>

   ## Follow-ups
   - <deferred-but-real work → backlog | unspecced ideas → ideas ledger | bugs → fix now>
   ```

   Keep it terse and specific — a real log, not a status report. Prefer concrete `file:line`, commit hashes, and named specs over generalities.

4. **Route the durable outputs** (the retro is the per-session log; these are the cross-session homes):
   - **Lessons** that will matter beyond this session → write to memory (`~/.claude/projects/<project>/memory/`, one fact per file, add a pointer line to `MEMORY.md`). Skip anything the repo/git history already records.
   - **Deferred-but-real work** → append to `docs/dev/backlog.md` under `## Open` (`what / why-deferred / candidate-home / source`). Dedup by the "what".
   - **Unspecced improvement ideas** → append to `docs/dev/ideas.md` under `## Open` (≤3 lines: what / why better / where). Never build directly from it — user triages.
   - **Architecture/design decisions** made along the way → an ADR in `docs/decisions/` (`NNN-title.md`).

5. **Report** to the user: the retro file path, and a one-line summary of anything routed to memory / backlog / ideas / decisions. Do not commit unless asked.

## Notes

- If nothing substantive happened, say so and skip writing an empty retro.
- One retro per meaningful boundary — don't spam a file per commit.
- This command owns the *writing*; the *triage* of ideas into scope stays a user decision.
