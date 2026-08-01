---
name: global-skill
description: Global workspace rules, behavior conventions, and Git workflow for all agents.
---

# Global Skill

Behaviour rules for **all** agents. Read after your role profile, before any work.

## No work before context load

Do not list, read source, or edit until you've finished the `Agent role.md` init sequence and read this file. Dot-dirs (`.agent_profiles`, `.git`) need `Get-ChildItem -Force`. Prefer full paths. "Main folder" = the project root (e.g. `project1/`), never an internal module dir.

---

## Token efficiency (the point of Symphony)

**Don't checkpoint conversation context.** Everything durable is already saved by hand — in tickets, `ARCHITECTURE.md`, `MEMORY.md`, and the code. Do not spend tokens re-summarising the conversation, writing progress recaps, or building "context to carry forward". When a session ends or context is cleared, the next agent reloads truth from files (see `stateless-protocol`). If something matters beyond this session, write it to the right file (ticket / ARCHITECTURE / MEMORY) in a few words and move on — never narrate it into the chat "so it isn't lost". The filesystem is the memory; the conversation is disposable.

**No keepalive / no tool spam** *(agents once burned tokens firing empty shell commands every fraction of a second to "stay alive")*: after any WAIT/EXIT/"nothing to do" status, **end the turn with zero further tool calls**. Never use tools as a heartbeat (empty commands, `echo` noops, tight sleep loops, unchanging status commands). WAIT = one scheduled sleep, then a real re-read. EXIT cancels the poll; don't re-arm. Full detail: `Agent role.md` §Work Loop.

**Clarify only when guessing costs more than asking.** Classify each request: **Clear** (90%+) → proceed, state assumptions if useful. **Minor ambiguity** (70–90%) → proceed on sensible defaults, name them, don't stop. **Material** (40–70%, different readings change architecture/cost/security/API) → pause, ask ≤3 targeted questions. **Critical** (<40%) → ask ≤5 high-leverage questions. Questions must be specific, decision-oriented, one-sentence, non-overlapping. Max 2 rounds / 5 questions total; then state assumptions and proceed on safest defaults. Bad: "tell me everything you want." Good: "JWT or session auth?"

---

## Performance is a first-class constraint (user mandate)

A feature that janks, stutters, or makes the user wait is **not done**.
1. **Never block the UI/main thread** with I/O, DB, disk, network, serialization, or heavy compute — push it off-main and render the shell immediately.
2. **Warm the next screen** while the current loads (prefetch predictable next actions). Latency the user never sees is defeated.
3. **Cache hot paths; compute once.** Don't re-read/recompute stable data on every open/scroll/redraw; invalidate precisely.
4. The Architect states a **performance approach** in every ticket touching a user-facing hot path (async/cached/prefetched/paginated), under Architectural Constraints. QA/Dev must not regress latency; "it works" is insufficient if slower.
5. **Measure** on perf-sensitive work — state before/after, don't assume.
6. **On conflict, performance wins.** Only the user may trade it away, in writing.

---

## Read live state from disk

Ticket statuses, `MEMORY.md`, git state, and build artifacts change constantly and externally. A cached/sandbox filesystem view may be a stale snapshot.
- Read ticket listings/statuses/file contents from the **actual project disk** at the moment you need truth — use a direct file reader over a sandbox shell where both exist. Never assert from an earlier in-session listing or from memory.
- Git state: the live repo is truth; if you can't read it reliably, don't assert it — defer to the user.
- Never claim a build's age from a cached mount; if you can't stat the real file, ask.
- **Re-read immediately before asserting** anything about current state whenever time has passed or another agent may have acted.

---

## Status command (all roles)

On "status" / "update" / "what's the status" (any role, already initialized): don't re-init. Scan `tickets/` with `-Force`, filter to **pending only** (everything except `[DONE]`/`[CANCELLED]`/`[REVERTED]`). Per ticket, output filename + a 2–3 line summary (what it is, its stage, blocker/next owner) — **never dump ticket bodies**. No pending → say so. Read-only: no renames, no edits, no auto-proceed.

---

## Long-running commands → background

A synchronous `gradlew`/`rtest`/`assembleDebug` call blocks the tool for minutes — the agent looks frozen and can't message the user. **Any command expected to exceed ~30s runs in the background**, output redirected to a temp file, polled in short later calls.

Pattern (PowerShell): `Start-Process ... -NoNewWindow -RedirectStandardOutput <log> -RedirectStandardError <err>`, writing a `.done` marker on exit; poll with `Test-Path <done>`; read results with `Get-Content -Tail 40`. **Always trim output** (`-Tail N`) — never dump a 200-line stack trace; BUILD SUCCESSFUL/FAILED + the error line is in the last ~40.

| Command | Duration | Mode |
|---|---|---|
| `git`, `Move-Item`, reads/edits | <5s | foreground |
| targeted `rtest`, warm compile | 30–120s | background |
| incremental full `rtest`, `assembleDebug` | 2–5min | **always background** |
| `rtest --full-cold` | 5–15min | **always background** |

Contract: after launching, tell the user "launched `<cmd>` in the background; I'll report when done" and **end the turn** — don't poll-loop within one turn (that re-creates the block). Next turn: poll once; still running → one line + end turn; done → trimmed result. Never run two background gradle/rtest jobs on one project (shared daemon). Use `<TEMP_DIR>\` for markers.

---

## Git workflow (only if the project has a `.git/`)

Not every project is git-ified. Check first (`git status` from the project root); no repo → **skip all git steps silently** and proceed with renames/edits/rtest as normal. Re-check each init. All Hard Rules in `agent-symphony` §"Hard rules" bind every git op (scope lock, one-ticket-one-commit, UTF-8/CRLF, stop on index error or 0-byte blob, delete scratch, pre-push zero-byte gate).

**Pre-work sync check (every init).** The one-agent rule prevents simultaneous edits but not a session that committed locally then died before pushing — leaving a commit stranded while origin stays behind (this happened: two sessions redid the same WD-116 work and diverged).
0. **Clear a stale lock:** one agent runs at a time, so any `.git/index.lock` at session start is a dead leftover — delete it, then continue. (Still STOP for real index *corruption* — `unknown index entry format` — or a tree dirty outside your scope.)
1. `git fetch`.
2. `git status` — errors or out-of-scope dirty tree → STOP and report.
3. **Ahead only** → `git push` before starting new work.
4. **Diverged** → stop and reconcile. Diff both tips; if one is a strict continuation of the other, keep the further one; genuine conflict → escalate like a CANNOT.

**QA after test authoring:** `git add .` → `git commit -m "<ticket>.md"` (name minus status prefix) → `git push`.
**Dev after implementation:** `git pull` (before claiming) → implement → squash QA's test commit + own work into **one** commit named `<ticket>.md` → append the commit hash to the `[DONE]` ticket → amend → `git push --force-with-lease`.

---

## Vendor neutrality (permanent)

Any agent (Claude, Gemini, Grok, GPT, Cursor, Codex…) must be able to participate using only the neutral files: `Agent role.md`, `skills/*`, `.agent_profiles/*`, and each project's `MEMORY.md`/`SKILL.md`/`ARCHITECTURE.md`/`AGENTS.md`/`tickets/`.
1. **No protocol content lives only in a vendor file** (`CLAUDE.md`, `GEMINI.md`, `.cursorrules`…) — those are convenience mirrors, nothing more.
2. **Change order:** write rules to neutral files first, then mirror. A mirror ahead of canon is a defect.
3. **On conflict, neutral files win** — every mirror says so.
4. Each project root carries `AGENTS.md` pointing new agents at `Agent role.md`.
5. Skills/profiles never move into vendor dirs (`.claude/`, `.gemini/`).

**Terminology:** *project root* = the workspace dir · *tickets* = status-prefixed files in `tickets/` · *rtest* = the regression suite (command in the project `SKILL.md`) · *Symphony Protocol* = this file-based coordination system.
