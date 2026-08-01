---
name: agent-symphony
description: The core Symphony Protocol — ticket lifecycle, agent boundaries, and workflow rules.
---

# The Agent Symphony Protocol

Role-based multi-agent TDD. Agents keep to their role and communicate **only** through ticket files. Lifecycle states and the work loop are in `Agent role.md`; this file is the rules behind them.

**Nothing is `[DONE]` (Dev) or `[READY_FOR_DEV]` (QA) until committed AND pushed.** That is the final, non-negotiable step.

---

## One agent per role per project (orphan recovery)

No two agents of the same role ever run in parallel on one project (different roles may overlap; cross-project is fine). So a `.claims/<T>-<Role>.claim`, a claim marked `IN_PROGRESS`, or an in-flight ticket of **your** role means one thing: **the previous you died** — never a live peer. Resume it before reading `ticketorder.md`.

*(WD-236: a Dev saw an orphaned claim, assumed a live peer, and took a different ticket — stranding half-finished work. Hence: resume first, always.)*

Recovery: (1) the orphan is your first priority; (2) **audit before trusting** — read the ticket, inspect `git status`/`git diff` and unpushed commits, verify against the Solution Approach (orphaned work is a lead, not gospel); (3) finish cleanly — complete what's missing, run remaining gates, commit+push; unrecoverable → escalate via CANNOT, never silently discard; (4) delete the `.claim` on terminal handoff.

---

## Agent boundaries

**Dev lifecycle (project1):**
- **Architect** — docs and `[APPROVED]` tickets only. May read code / run rtest / make *experimental* changes **only** to diagnose a CANNOT, then rolls them back or documents them as directions — never promotes to `[DONE]` via own code. Every behaviour change goes through a ticket.
- **QA** — failing tests for `[APPROVED]`, promote to `[READY_FOR_DEV]`, commit `<ticket>.md`, push. Blocked → `[CANNOT_QA]` (findings, stop). Never touches production code.
- **Dev** — pull; claim `[READY_FOR_DEV]`→`[IN_PROGRESS]`; implement **exactly** per Solution Approach (numbered steps are binding); rtest green; `[DONE]`; one squashed commit + hash appended, push. Blocked → `[CANNOT_DEV]` (findings, stop). Never writes tests or changes architecture.
- **SRTL** — reviews `[DONE]` against the ticket (Solution Approach, Constraints, Regression Guard, DoD; diff matches the ticket's law). Corrects **code AND tests** (sole dual authority). Unblocks `[CANNOT_*]` at root cause. Appends `**SRTL Review:** ✅ PASS` / `🔧 CORRECTED — <details>`. Never touches `ARCHITECTURE.md`/design docs/templates (only brief MEMORY status lines) unless asked; never creates tickets or refactors beyond the ticket.

**`[APPROVED_UI]` fast-track:** for purely visual tickets (tokens, fonts, icons; no business logic) the Architect may label `[APPROVED_UI]`; QA or Dev then writes verification + implementation in one pass. Avoids handoff overhead for boilerplate visuals.

**Content lifecycle (project2):**
- **Composer** — raw article → `[DRAFT]` capsule; voice preserved 100%; title ≤28 chars, crux-hiding. Never authors from scratch/reviews/numbers/designs/codes.
- **Critic** — editorial gate + staircase curator. `[DRAFT]`→`[REVISION]`/`[FINAL]`. Owns placement: assigns N, renumbers `[COVERED]` on insert, fixes cross-refs, updates inventory + Staircase Map, logs to MEMORY.
- **Designer** — integration/UI tickets (`*_APPROVED`) with Notes to Tester/Implementer; `[FINAL]`→`[COVERED]`. Specs cover phone/tablet/desktop + dark. No content, no code.
- **Tester** — owns `rtest.py`. `*_APPROVED`→failing→`*_VERIFIED`; `*_RFT`→`*_FIXED`/`*_FIX_FAILS`. Standing assertions: contiguous numbers, unbroken prev/next, matching library/sitemap/index counts. Blocked → `*_CANNOT_TEST`.
- **Implementer** — `npm run build`; edit `src/`/`build.js` per ticket until assertions+rtest pass; `*_VERIFIED`/`*_FIX_FAILS`→`*_RFT`. Never hand-edits `dist/`, never touches rtest or content. Blocked → `*_CANNOT_IMPL`.

Content status is a filename **suffix** (`_APPROVED`, `_VERIFIED`…); bracket **prefixes** are reserved for capsule content states.

---

## "proceed" / "next" / "continue"

A bare continuation word means exactly: **resume my own current task, within my own role**. It is a recovery signal, **not** a "yes" to whatever the agent last floated, and **never** licence to cross a role boundary.

If a QA agent ends a turn asking "shall I fix the `[READY_FOR_DEV]` ticket?" (that's Dev's job) and the user says "continue", it means "do your next QA-appropriate thing" — never "write the code you floated". The prior question was itself the mistake. On resume: never re-offer or take an out-of-role action; re-run your own work loop; if genuinely ambiguous, ask — never guess toward the boundary-crossing reading. Role isolation is the protocol's single most important invariant; one careless musing + one "continue" must never breach it.

Only the Architect/Designer may step outside their lane, and only to *investigate* a CANNOT.

---

## CANNOT resolution (Architect owns it)

First run `skills/blocker-resolution/SKILL.md` triage — a straightforward, ticket-traceable, non-weakening fix (e.g. a stale literal your own change made stale) is made and logged, not escalated. Only a genuine block becomes CANNOT. On CANNOT: rename, write findings, **sound the alarm** (blocker-resolution §"Genuine Block"), STOP — do not touch another ticket until it clears.

Architect's four paths: **CANCEL** (`[CANCELLED]`, keep the file), **REVISE** in place (fix flawed sections, back to `[APPROVED]`, same number), **APPROVED + Supplemental Directions** (append clarifications, back to `[APPROVED]`), **NEW ticket** (cancel old with cross-ref, fresh number). All CANNOT communication lives in the ticket file — no side channels.

---

## Batch / interactive rule (Architect & Designer)

One continuous user session = one batch. Create multiple `[APPROVED]` tickets; if a later insight breaks an earlier one, **go edit that earlier file** (keep the `[APPROVED]` prefix). Reorder within the active batch freely; never reorder `[DONE]`. Tickets move to QA/Dev only when the user explicitly signals the batch is done.

---

## Shared worktree → pipeline, not parallel

All agents share **one checkout/branch** — no per-agent sandbox. Shared worktree = shared consequences, which is why only one agent works at a time.

**Order is `ticketorder.md`** (Architect authors; QA/Dev append `:DONE` to their own finished line only). The work loop (`Agent role.md`) drives it: resume orphans → EXIT if no open line is yours → WAIT if yours exists but isn't the head → TAKE the head when it's yours and gate-open, then chain same-role heads. Never mark `:DONE` on a CANNOT.

**Why not parallel — the rtest conflict:** if QA promotes N+1's failing tests while Dev works N, Dev's rtest shows N+1's failures too (noise masking real regressions), and a shared-file ticket's tests may not even compile against Dev's mid-refactor. The pipeline keeps Dev's signal clean: only their own ticket's tests are non-green.

**Strict pipeline (default):** QA does not promote the next ticket until Dev `[DONE]`s the current one.
**Staged-one-ahead (only if ALL hold):** next ticket's deps are `[DONE]`; shares no production file with Dev's current ticket; its tests compile against the committed state; user hasn't said otherwise. Any doubt → strict.

**Test tiers** (see `rtest/SKILL.md`): QA proves RED with `rtest --fast`/`--targeted` — never a cold full suite. Dev iterates with `--fast`, confirms GREEN with `--targeted`, then runs **incremental** `rtest` (cache on, no `clean`) once before `[DONE]`. The **cold** full suite (`rtest --full-cold`) runs once per batch (folded into the artifact build). Choosing a tier is an execution decision — it never moves role boundaries.

**Compile-blockers land first.** A non-compiling suite blocks everyone; the Architect places compile-blocker tickets first in `ticketorder.md`.

**Route order wins.** With `ticketorder.md` present, QA/Dev never independently pick lowest-numbered. Absent it (legacy), fall back to lowest-numbered `[APPROVED]`/`[READY_FOR_DEV]`; a late high-numbered blocker → renumber or add a route file. (`138A` sorts below `139` only because `8<9`; never use `139A` to sort below `139`.)

**Shared-file hotspots** (Architect's duty): tickets touching the same production file must never be in Dev at once (they conflict at merge). Publish a required Dev order, ensure each claim starts with `git pull`, and never release a dependent ticket to QA before its prerequisite is `[DONE]`.

**Batch-fast mode (optional):** for large (8+), *independent* batches the user may declare it: per ticket, Dev runs a targeted rtest + a compile check (`compileDebugUnitTestKotlin`) instead of full rtest, then **one full rtest at batch end**. The compile check catches the common cross-ticket break (a test referencing something you removed) in ~10s. **Avoid** when tickets share files, involve DB migrations or layout-ID renames, have open CANNOTs, or the batch is small. The end-of-batch full run is non-negotiable; failures → Dev bisects and fixes forward before anything ships. It trades early regression detection for speed — worth it only when tickets are genuinely independent.

---

## Hard rules (from the WD-170 incident: a corrupted-index multi-ticket commit put 32 test files on origin as 0-byte blobs)

1. **Scope lock.** Modify only files in your ticket's "MAY touch" list (plus tests your role owns). Need any other file → STOP, `[CANNOT_*]`, findings. Before committing, read `git status`; an unmappable file → do not commit, escalate.
2. **Commit discipline.** One ticket = one commit (Dev squashes QA's test commit + own work). Commit+push immediately at your terminal state — an uncommitted `[DONE]` is a violation. Delete every scratch file before staging.
3. **Encoding.** Windows repo: UTF-8, mostly CRLF — preserve both. Never rewrite a file wholesale when a targeted edit suffices (a diff showing every line changed = broken line endings; revert). Never `>`/`>>` onto a source file (writes UTF-16). Check `git diff --stat` before commit; wholesale-rewrite or "Bin" on a text file = encoding damage.
4. **Git integrity.** Index errors, or a known-nonempty file showing 0 bytes → STOP all work, report. Pre-push gate (must be empty):
   ```powershell
   git ls-tree -r -l HEAD | Select-String "\.(kt|java|xml|proto)" | Where-Object { ($_ -split "\s+")[3] -eq "0" }
   ```
   Never kill a git process mid-op. A stale `.git/index.lock` at session start → confirm no git running, delete it, then sync.
5. **Working dir hygiene.** Verify cwd before every bulk op (a wrong-cwd agent created a nested `app/app/`). A dirty worktree with out-of-scope files at session start → report and wait; never absorb or revert it.
6. **Berserk brake.** Before every commit: *"Can I name the ticket line that authorises each file in `git status`?"* Any "no" → stop, revert, escalate. A slow wrong plan is recoverable; fast improvisation is not.

---

## Two-lane lifecycle

Robolectric UI tests proved high-cost/low-yield, so the Architect declares `**Lane:** backend` or `**Lane:** UI` in every ticket.

- **Backend lane** (engine, data, serialization, alerts, privacy): full TDD — QA writes failing pure-JVM tests, Dev passes them. Cheap and precise; guards what manual testing can't.
- **UI lane** (screens, layout, nav, animation): Architect creates it **directly as `[READY_FOR_DEV]`** (no QA, ever). It carries a `## Manual Test Script (user)` — the human is QA. Dev's `[DONE]` gate: `rtest --fast` green + `compileDebugUnitTestKotlin` + `assembleDebug`. Final acceptance is the user's manual pass; a bug against a `[DONE]` UI ticket → new follow-up ticket, original stays `[DONE]`.
- **Retiring old UI tests:** a UI ticket may carry a `## Retired tests` list Dev deletes in the same commit (authorized, not "writing tests"). Dev never touches a test not on that list — a break in an unlisted test is `[CANNOT_DEV]`.

Everything else (boundaries, scope lock, commit/encoding/integrity, CANNOT, one-at-a-time) applies identically in both lanes.

---

## Regression Guard (every ticket)

Fixed bugs kept returning because UI tickets shipped with zero automated protection. So:

1. Every ticket carries a `## Regression Guard` section: one line each for the behaviour fixed and the exact test pinning it. No `[DONE]` without that test existing and passing. Both lanes.
2. UI **behaviour** is now tested (visuals stay manual). Anything expressible as a state assertion gets a lock: scroll/anchor, visibility, enable/disable, ordering, nav/back-press precedence, dismissal, post-event binding, lifecycle survival.
3. Locks live in `test/…/locks/<Area>BehaviorLocksTest`, one method `lock_<ticket>_<behaviour>()` with a one-line symptom comment. **Append-only** — no agent deletes/weakens/`@Ignore`s a lock; a genuine contradiction needs written Architect approval in the ticket. The lock lands in the same commit as the fix.
4. `REGRESSION_CHECKLIST.md` (project root) is the human index: symptom · ticket · test, updated in the same commit. First thing to read on "this came back".
5. Batch-close gate: full suite green, every lock green, before the artifact builds. A red lock blocks the build — no "unrelated failure" reasoning.

---

## Memory hygiene & artifact delivery

- **MEMORY.md stays small.** On any terminal state, move that ticket's log line into a `# HISTORY` section at the file's bottom. Agents **ignore HISTORY** on init unless the user asks. Keep only the 3–4 most recent activities active.
- **Artifact on empty cycle.** When the last active ticket reaches `[DONE]` (no `[READY_FOR_DEV]`/`[IN_PROGRESS]` left), build the artifact and drop it in the project root automatically. For dev projects: run `rtest --full-cold` (the per-batch regression backstop); **only if fully green**, build (`assembleDebug` or per `SKILL.md`), copy to the root, report size. Not green → do not build; bisect the culprit and escalate. Other projects: adapt the build+copy from the project `SKILL.md`.

---

## Orchestrator (canonical: `.agent_profiles/orchestrator_profile.md`)

Per-project role (`init <project> orchestrator`), one instance per project. Dumb by design: follows the Architect's route and Symphony's gates; never reasons technically, writes tickets/code/tests, or weakens guards. **Not usable today** — it needs unattended, model-selectable spawning that no vendor exposes cleanly; the loop covers the same ground meanwhile (see repo README).

Three-file contract: `ticketorder.md` (route — Architect authors/reorders/prunes; QA/Dev only append `:DONE`) + `taskagent.md` (model rotation rings) + `orchestrator model map.md` (slug legend). Route says WHAT next; ticket status says WHEN; ring says on WHICH model. QA/Dev also read `ticketorder.md` themselves so manual seats follow the same order.

Only Orchestrator-written state: `.claims/<T>-<Role>.claim` (in-flight + serialization; timeout → bounded retry → escalate). Position is re-derived from ticket statuses each pass (crash-resumable, no pointer file). Adding a role = Registry row + one gate row + a ring line.

**Route hygiene (Architect duty):** prune completed lines **only at batch close or new batch — never mid-batch**. The route is both queue and live record of the open batch. Never prune: `[HOLD]`/`[DEFER]`, in-flight lines, a line with a role suffix lacking its own `:DONE` (e.g. `WD-279-Dev:DONE:REVIEWED` still has an open SRTL request unless a `WD-279-SRTL:DONE` line exists), or any line whose human gate is unanswered. Always re-scan `tickets/` live; never prune from memory. QA/Dev write only the `:DONE` suffix; the Orchestrator never authors route content.
