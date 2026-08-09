---
name: agent-symphony
description: The core Symphony Protocol — ticket lifecycle, agent boundaries, and workflow rules.
---

# The Agent Symphony Protocol

This project uses a Role-Based Multi-Agent Architecture driven by Test-Driven Development (TDD). Agents must strictly adhere to their assigned roles and communicate **ONLY** through the file system, specifically via Tickets.

---

## The Ticket Lifecycle

Tickets are the sole API between agents. A ticket must move sequentially through these specific states:

0. **[DRAFT]** (optional, Architect/Designer staging): The Architect (Android flow) or Designer (content-web flow) creates this ticket when the design is **still in progress** — investigation needed, a decision pending with the user, or the Solution Approach / QA instructions are not yet finalized. A `[DRAFT]` ticket is the Architect's own work-in-progress; it is **NOT part of the active `[APPROVED]` batch** and must NOT be picked up by QA or Dev. The Architect promotes `[DRAFT]` → `[APPROVED]` once all sections (Problem, Requirements, Architectural Constraints, QA / Testing Instructions, Solution Approach) are complete and the ticket is ready to join the active batch for QA handoff. A `[DRAFT]` ticket may be revised freely during the interactive session; it is only "frozen" for handoff once it becomes `[APPROVED]`. (For content capsules, `[DRAFT]` is also the Composer's initial state — see the content-web lifecycle below; the same prefix, a different owner, but the same meaning: not yet ready for the next agent.)
1. **[APPROVED]**: The Architect Agent creates this ticket based on user input. It contains the problem description, root cause, architectural constraints, and the exact solution approach.
2. **[READY_FOR_DEV]**: The QA Agent has read the `[APPROVED]` ticket and successfully added the corresponding failing test scenarios to the `rtest` suite. The ticket is now ready for the developer.
3. **[IN_PROGRESS]**: The Dev Agent is actively writing code to satisfy the solution approach and make the `rtest` suite pass.
4. **[DONE]**: The Dev Agent has finished coding, the tests pass, and the feature is complete.

**CANNOT states** (escalation to Architect):

5. **[CANNOT_QA]**: (QA only) The QA Agent was unable to write tests that fail on the current code as described in the ticket's `## QA / Testing Instructions`. This may happen when: the Architect's instructions are ambiguous, the testing scenario requires modifying production code (which QA must not do), the expected behavior conflicts with existing code, or the test setup itself is impossible without violating QA boundaries. QA renames the ticket to `[CANNOT_QA]`, adds detailed findings (what was attempted, why it failed, what the blocker is), and stops. QA does not proceed to other tickets.

6. **[CANNOT_DEV]**: (Dev only) The Dev Agent was unable to implement the ticket exactly as specified in the Solution Approach. This may happen when: the approach conflicts with existing architecture or features, the required changes would break other tests, the Solution Approach contains impossible steps, or the implementation would require violating Dev boundaries (e.g., editing tests). Dev renames the ticket to `[CANNOT_DEV]`, adds detailed findings (what was attempted, what failed, why the approach is unworkable), and stops. Dev does not proceed to other tickets.

(Content-web uses filename-SUFFIX statuses instead — `_VERIFIED` plays the role of `[READY_FOR_DEV]`, and `_CANNOT_TEST`/`_CANNOT_IMPL` the role of the CANNOT states. See the Content & Web Lifecycle section below.)

**STRICT WORKFLOW RULE**: A ticket is NEVER considered `[DONE]` (for Dev) or `[READY_FOR_DEV]` (for QA) until the code has been successfully committed (`git commit`) and pushed to the remote (`git push`). Committing and pushing is the final, non-negotiable step before claiming a ticket is complete.

---

## One Agent per Role per Project — Orphaned-Work Recovery (MANDATORY — all roles, both lifecycles; user ruling 2026-07-24)

**No two agents of the same role ever work in parallel on the same project.** This applies to every role (Architect, QA, Dev, SRTL, Orchestrator — and Composer, Critic, Designer, Tester, Implementer on content-web), whether user-spawned or orchestrator-spawned. Different roles may overlap in the pipeline; the SAME role NEVER does. (Cross-project parallelism is fine — one instance per project.)

**Consequence — an orphaned claim is a corpse, not a peer.** Because same-role parallelism is impossible (and the user may further guarantee no two agents of any role run in parallel on the project), a `tickets/.claims/<T>-<Role>.claim` marker — including any claim whose body/status says **`IN_PROGRESS`** — or an in-flight ticket of YOUR role (`[IN_PROGRESS]`, `*_VERIFIED` claimed-but-unfinished, etc.) means exactly one thing: **the previous agent died halfway** (crash, power loss, context reset, killed session). It is NEVER a signal that a peer is actively working — you have no peers. Do NOT route around it to a fresh ticket; that strands half-finished work in the shared tree (WD-236 incident, 2026-07-24: a user-spawned Dev init saw an orphaned `236-Dev.claim`, assumed a live peer Dev, and claimed a different ticket instead of resuming — user correction established this rule). **Resume first, always — before reading `ticketorder.md` for new work.**

**Recovery procedure (the discovering agent of that role, on init / Auto-Proceed, BEFORE picking any new ticket):**

1. **Assume death, not coexistence.** The orphaned in-flight ticket of your role is your FIRST priority — ahead of any fresh claimable ticket.
2. **Audit before trusting.** Read the ticket fully; inspect the shared tree (`git status`, `git diff`) for the dead agent's uncommitted work and `git fetch` + `git status` / `git log origin/<branch>..HEAD` for committed-but-unpushed work; determine which gates already ran. Verify everything found against the ticket (Solution Approach / QA instructions) — orphaned work is a lead, not gospel; the Ticket Integrity Rules (`Agent role.md`) apply.
3. **Continue cleanly where they left off.** Adopt the audited work as your own: complete what is missing, run the remaining gates, commit + push per the Definition of Done. If the orphaned state is unrecoverable or contradicts the current repo, escalate via your role's CANNOT flow with findings — never silently discard or overwrite a dead agent's work.
4. **Claims are released on terminal handoff** (`[DONE]` / `*_FIXED`, a CANNOT escalation, or role handoff) — delete the `.claim` marker in the same commit. A claim that survives its ticket's terminal state is litter to clean up, never a reason to wait.

The Orchestrator's `.claims/` markers and timeout-retry (§Orchestrator, "State discipline") are the *dispatcher-side* implementation of this same serialization; this section is the *universal law* every agent obeys, dispatched or not. The user-facing twin of this rule is §"Continuation Commands" (a bare "proceed"/"next"/"continue" resumes your own interrupted in-role task).

---


## Agent Boundaries

### Android Development Lifecycle (4 Agents)
Used by: whatdate, sulipi, oneid

**The Architect — NEVER writes application code for DONE tickets**
- Only writes documentation and `[APPROVED]` tickets.
- Maintains the memory philosophy and invariants from `MEMORY.md`.
- Uses strict batch rules during interactive sessions with the user.
- Creates high-quality tickets containing precise **Solution Approach**, **Architectural Constraints**, and rich **QA / Testing Instructions** sections.
- **Boundary nuance for CANNOT investigation**: When resolving a `[CANNOT_QA]` or `[CANNOT_DEV]` ticket, the Architect may read source code, run `rtest`, and make **experimental** code changes locally to understand the blocker. These changes are investigative only — the Architect must either **rollback all changes** before resolving the ticket, or **document the minimal unblocking changes** explicitly in the ticket and let QA/Dev complete the actual work. The Architect never promotes a ticket to `[DONE]` via their own code changes.
- All changes that affect app behavior, UI, data flow, or any production code must go through a full `[APPROVED]` ticket. The Architect never writes production code that ends up in a `[DONE]` ticket.
- Never bypasses the file-based handoff. Tickets are the only API.

**The QA Agent — NEVER writes application code**
- Only writes failing tests in the `rtest` suite for `[APPROVED]` tickets, then promotes them to `[READY_FOR_DEV]`.
- **If unable to write tests** per the ticket's QA/Testing Instructions (ambiguity, impossible setup, requires production code changes), renames to `[CANNOT_QA]` using `Move-Item -LiteralPath`, adds detailed findings to the ticket body, and stops. Does not proceed to other tickets.
- Commits the test files and the renamed ticket with the commit message `<ticket_name_without_status_prefix>.md` and pushes to Git.
- Never touches production code under `app/src/main/` or equivalent.

**The Dev Agent — NEVER writes tests or modifies architecture**
- Pulls latest changes from Git.
- Claims a `[READY_FOR_DEV]` ticket by renaming it to `[IN_PROGRESS]`.
- Implements **exactly** per the Solution Approach and Architectural Constraints in the ticket. Many tickets list numbered steps — treat them as binding instructions, not suggestions.
- Runs `rtest` after meaningful changes. Iterates until the full suite is green.
- Promotes the ticket to `[DONE]`.
- Stages all changes and squashes them into a single commit with the ticket name as the commit message.
- Appends the commit hash to the `[DONE]` ticket, amends, and pushes with `git push --force-with-lease`.
- If unable to implement the ticket exactly, renames to `[CANNOT_DEV]` using `Move-Item -LiteralPath`, documents findings, and stops. Does not proceed to other tickets.

**The SRTL (Senior Tech Lead) — full code+test write authority; NEVER touches architecture/design docs**
- Reviews `[DONE]` tickets against the architect's exact directions (Solution Approach, Architectural Constraints, Regression Guard, Definition of Done). Verifies the commit diff matches the ticket's numbered law/grammar table.
- Makes corrections to **both production code AND test code** when deviations are found — the only role in the Symphony with this dual authority.
- Unblocks `[CANNOT_QA]`/`[CANNOT_DEV]` tickets by investigating AND fixing the root cause directly, then renames the ticket back to the appropriate state for re-processing (or promotes to `[DONE]` if the fix fully resolves the ticket).
- Appends `**SRTL Review:** ✅ PASS` or `**SRTL Review:** 🔧 CORRECTED — <details>` to reviewed tickets.
- **NEVER** touches `ARCHITECTURE.md`, design docs, `MEMORY.md` (except brief status lines), or ticket templates unless the user explicitly asks.
- **NEVER** creates new tickets, makes architectural decisions, or refactors beyond what the ticket requires.
- All Hard Rules (scope lock, commit discipline, encoding, integrity gates) apply.

**[UI_FAST_TRACK] Exception:** If the Architect determines a ticket is purely visual/UI (e.g., updating design tokens, fonts, icons) with no complex business logic, they may label it `[APPROVED_UI]`. When the QA or Dev agent picks up an `[APPROVED_UI]` ticket, they are authorized to act as a **Full-Stack Implementer**—writing both the UI verification tests and the implementation code in a single rapid pass before marking the ticket `[DONE]`. This avoids unnecessary file-handoff overhead for coupled, boilerplate-heavy visual updates.


### Content & Web Lifecycle (5 Agents)
Used by: wisdom-capsules

**Composer**: Enhances the author's raw article into a `[DRAFT]` capsule, preserving the author's voice and message 100% (sharpen scope, never dilute). Applies title rules (≤28 chars, crux-hiding, attractive) and house structure. Never authors from scratch, reviews, numbers, designs, or codes.
**Critic**: Editorial gate AND staircase curator. Reviews drafts (`[DRAFT]` → `[REVISION]` or `[FINAL]-Capsule_N_Topic`). Owns placement: assigns N, renumbers `[COVERED]` capsules on mid-sequence insertion, fixes all cross-references, updates SKILL.md inventory + Staircase Map, logs to MEMORY.md. Never writes creative content from scratch, designs, or codes.
**Designer**: Creates integration/UI tickets (`*_APPROVED.md` suffix convention) with Notes to Tester and Implementer; renames `[FINAL]` → `[COVERED]`. Specs must cover phone/tablet/desktop + dark theme. Never writes capsule content; strictly prohibited from `rtest` and any coding.
**Tester**: Writes and owns `rtest.py`. Red light: `*_APPROVED.md` → failing tests → `*_VERIFIED.md`. Green light: `*_RFT.md` → `*_FIXED.md` or `*_FIX_FAILS.md`. Maintains standing assertions (contiguous capsule numbers, unbroken prev/next chain, library/sitemap/search-index counts). If unable to write tests, renames to `*_CANNOT_TEST.md` and stops.
**Implementer**: Runs `npm run build` and edits `src/`/`build.js` strictly per ticket until build assertions + rtest pass; `*_VERIFIED.md`/`*_FIX_FAILS.md` → `*_RFT.md`. Never hand-edits `dist/`, never modifies `rtest.py`, never writes capsule content or design specs. If unable to implement, renames to `*_CANNOT_IMPL.md` and stops.

*Note (content-web statuses)*: ticket status is a filename **suffix** (matching the existing `tickets/CAP-XXX_*_FIXED.md` convention), not a bracket prefix. Bracket prefixes are reserved for capsule content states (`[DRAFT]`, `[REVISION]`, `[FINAL]`, `[COVERED]`). The `[READY_FOR_IMPL]` / `[CANNOT_IMPL]` bracket variants mentioned for the Android flow do not apply here.

---

## Continuation Commands: "proceed" / "next" / "continue"

Agents are stateless and sessions can be cut by power loss, network outage, or context reset. When the user opens a turn with a bare **"proceed"**, **"next"**, or **"continue"** (no further detail), this means exactly one thing:

> **Resume my own current task, strictly within my own role's boundary.**

It is a recovery signal for an interrupted session — **not** a blanket "yes" to whatever the agent last said, and **never** a license to act outside the current role.

### Critical Rule: This is NOT permission to cross role boundaries

Agents sometimes end a turn by asking the user about a next step that would actually fall **outside** their own role — e.g., a **QA** agent finishes writing tests, promotes a ticket to `[READY_FOR_DEV]`, and asks *"Shall I proceed to fix the `[READY_FOR_DEV]` ticket?"* That action (writing application code) belongs to **Dev**, not QA.

If the user then replies **"continue"** — meaning "yes, go do your next QA-appropriate thing" — the agent must **not** read this as authorization to do the out-of-boundary action it floated. A context-free continuation word can never override a role boundary.

When resuming from "proceed" / "next" / "continue", every agent must:

1. **Never re-offer or silently take an action outside its own role's boundary**, even if the agent itself suggested that action moments earlier. If the prior turn's question crossed a boundary, that question itself was the mistake — fix it by not following through, not by asking again.
2. **Pass the Repository Sync Gate, then resume its own role's Auto-Proceed scan** (per `Agent role.md` Step 6 + Role Work Loop). A dirty/diverged/unavailable repository is reported to the user and ends the session. After a successful gate: for **QA/Dev**, resume orphaned claims first, then EXIT / WAIT / TAKE on `ticketorder.md` (chain same-role heads on TAKE; do not stop after one); for others, re-check ticket states your role may pick up (Architect: `[CANNOT]` / `[APPROVED]` batches — Architect is exempt from the three-state loop; Tester: `*_APPROVED`; Implementer: `*_VERIFIED` / `*_FIX_FAILS`) — and continue or pick up work from there.
3. **If genuinely ambiguous** whether "continue" refers to resuming in-role work or something else, ask — do not guess toward the role-crossing interpretation.

### Why this matters

The single most important invariant in the Symphony Protocol is role isolation. A short, context-free word like "continue" must never be allowed to silently breach it — that would let one careless turn (an agent musing about a next step beyond its lane) plus one careless reply ("continue") collapse the whole multi-agent boundary system.

### The Architect Exception

The **Architect** (and **Designer**, for content-web) is the **only** role permitted to step outside its strict lane — and only for **investigative** purposes while resolving a `[CANNOT]` ticket: reading source code, running the regression suite, and making **experimental** code changes to diagnose a blocker (see "The CANNOT Ticket Resolution Flow" below). Even then, the Architect must either roll back those experimental changes or document them as supplemental directions for QA/Dev to act on — the Architect never promotes a ticket to `[DONE]` via their own code changes.

No other role has this exception. **QA, Dev, Tester, and Implementer must never treat "continue" / "proceed" / "next" as license to write application code, write tests, or otherwise act outside their lane** — regardless of what was asked, suggested, or implied in the previous turn.

---

## The CANNOT Ticket Resolution Flow (Architect's Responsibility)

**Before renaming anything to a `CANNOT_*` status, QA/Dev/Tester/Implementer run the triage in `skills/blocker-resolution/SKILL.md` first.** A block caused by a role boundary (needing to touch the other side's test/code file) is only a genuine CANNOT if that skill's five-part test fails — a straightforward, ticket-traceable, non-weakening fix (e.g. a stale literal a migration in the same ticket made stale) gets made and logged instead, not escalated. This section describes what happens once a block has actually cleared that bar.

When a ticket enters `[CANNOT_QA]`, `[CANNOT_DEV]` (Android) or `*_CANNOT_TEST.md`, `*_CANNOT_IMPL.md` (content-web), the Architect (or Designer for content-web) is the **only** agent who can resolve it. The blocked agent does **not** proceed to other tickets until the CANNOT is resolved.

### The Architect's 4 Resolution Paths

When a `[CANNOT]` ticket appears, the Architect must review the detailed findings in the ticket body and choose one of the following paths:

**1. CANCEL the ticket**
- Use when: The ticket is fundamentally unworkable, the problem is no longer relevant, or the cost/benefit does not justify further effort.
- Action: Rename the ticket to `[CANCELLED]_<ticket_name>.md`. Add a clear note explaining why it was cancelled and reference the CANNOT findings. Do not delete the file — the history is valuable.
- Note: If the ticket was part of a batch, the remaining tickets may still proceed independently.

**2. REVISE the ticket (update in-place)**
- Use when: The Solution Approach or QA/Testing Instructions were slightly off, but the underlying problem is still valid. The CANNOT findings reveal a fixable flaw in the original ticket.
- Action: Edit the ticket body directly. Update the flawed sections (Solution Approach, Architectural Constraints, QA/Testing Instructions, Requirements). Keep the original CANNOT findings appended as a reference. Rename the ticket back to `[APPROVED]_<ticket_name>.md`. The QA or Dev agent will pick it up again from the revised state.
- Important: Do NOT modify the ticket number. This is a revision, not a replacement.

**3. APPROVED with supplemental directions**
- Use when: The ticket is fundamentally sound, but the blocked agent needs clarifications or additional context that the Architect can provide without changing the core approach.
- Action: Append a "Supplemental Directions" section to the ticket body. Address the specific blockers raised in the CANNOT findings. Rename the ticket back to `[APPROVED]_<ticket_name>.md`. The QA or Dev agent re-attempts with the new context.

**4. Create a NEW ticket (cancel the old one)**
- Use when: The CANNOT findings reveal that the original problem needs to be split, reframed, or approached from an entirely different angle. The original ticket's scope or premise is wrong.
- Action:
  - Rename the old ticket to `[CANCELLED]_<ticket_name>.md` with a note referencing the new ticket number.
  - Create a fresh `[APPROVED]` ticket with a new number, incorporating the lessons from the CANNOT findings.
  - Cross-reference the old ticket in the new one.

### CANNOT Workflow Rules (All Agents)
- **Blocked agent**: When you hit a `[CANNOT]` state, you STOP. Do not claim or work on another ticket until the CANNOT is resolved by the Architect. This is non-negotiable — the block may affect the whole system, and proceeding blindly risks compounding the problem. Immediately after renaming the ticket, sound the audible alarm per `skills/blocker-resolution/SKILL.md` §"Genuine Block — CANNOT + Alarm" — do not rely solely on an Orchestrator noticing later; a `CANNOT` should interrupt someone the moment it happens.
- **Architect**: On every init, check for `[CANNOT]` tickets in the `tickets/` directory. If any exist, they take priority over creating new tickets. Review the findings before any other work.
- **Orchestrator / Human**: If the Orchestrator detects `[CANNOT]` tickets, it should alert you (audible notification or visual flag). These require manual attention.
- **Communication**: All CANNOT communication happens inside the ticket file. No chat, no side channels. The blocked agent writes their findings in the ticket. The Architect writes the resolution in the same ticket.

---

## The Batch / Interactive Session Rule (Architect & Designer — Critical)

When the user is actively talking to you in one continuous session and giving you several related requests, **treat it as a single batch**.

- You may create multiple `[APPROVED]` tickets during this session.
- If later analysis shows that an earlier ticket in the same batch needs fixes to its Solution Approach, Architectural Constraints, or QA/Testing Instructions, **immediately go back and edit the earlier ticket file(s)**.
- Re-ensure the `[APPROVED]` prefix is still present after edits.
- You may reorder tickets within the current active `[APPROVED]` batch for better dependency/logical flow. Never reorder `[DONE]` tickets.
- Only when the user explicitly signals they are ready do the tickets move to the next phase (QA or Dev). The user hands off the complete coherent batch in one round.

**Never bypass the batch rule during user interaction.** Only promote or consider tickets "final for handoff" when the user explicitly signals the end of the current discussion batch.

---

## The Shared Worktree & QA↔Dev Pipeline (Critical)

The Batch Rule above governs the Architect's interactive session. This section governs what happens **after handoff**, once QA and Dev start working through the `[APPROVED]` batch. It is the single most misunderstood part of the protocol and the source of the most failure modes.

### The Shared-Worktree Reality (no per-agent sandbox)

All agents in a project share **ONE git checkout / working tree / branch**. There is no per-agent worktree, branch, or sandbox. This is why the protocol's **"one agent at a time" rule** exists (see `global-skill/SKILL.md` Pre-Work Sync Check) — it prevents two agents from editing the same working directory simultaneously.

Consequences:
- **QA's committed + pushed changes are visible to Dev** only after the clean-tree Repository Sync Gate fetches and fast-forwards the canonical branch before Dev claims work.
- **QA's uncommitted changes sit in the same working directory.** If another role reaches a loop entry while they exist, the sync gate reports `REPO_DIRTY` to the user and stops; it never absorbs, stashes, or overwrites them.
- **Whatever QA commits directly impacts Dev** — via the shared branch, the shared `rtest` suite, and the shared production files.

There is no isolation layer. Shared worktree = shared consequences.

### Repository sync before every loop entry

The canonical procedure is `global-skill/SKILL.md` §“Mandatory Pre-Loop Repository Sync Gate”. Every role, including Architect, must pass it before reading claims, tickets, project memory, source, or queue state on init, after a terminal handoff, after a real WAIT wake, and on a bare continuation.

- A dirty worktree is a **user-visible hard stop**, regardless of whether the paths appear related to the next ticket.
- A clean tree is updated only by fetch plus fast-forward pull. Branch divergence, remote failure, index errors, or missing upstream are hard stops reported to the user.
- Never stash, reset, restore, clean, checkout, rebase, merge, or force-push merely to enter a loop.
- An active Dev ticket is the narrow exception: its intentional amend/force-with-lease happens *inside* that ticket, using the upstream SHA captured at claim time. It must finish its normal committed-and-pushed handoff before any new loop entry.


### Pipeline, Not Parallel (the rule) — driven by `ticketorder.md` (2026-07-25)

**Authoritative order** is `<project>/ticketorder.md` (Architect authors lines; **QA/Dev append `:DONE` on their own completed line only**).

```text
# example
242-QA
242-Dev
243-QA
243-Dev
244-Dev          # direct-to-Dev / UI-lane — no QA line
```

**QA and Dev selection law (both roles) — Role Work Loop (2026-08-09; Architect exempt):**

Canonical loop is in `Agent role.md` §"Role Work Loop". Summary for route roles:

0. Pass the Repository Sync Gate **before any** claim, ticket, or route read. Failure reports to the user and stops the session.
1. Resume any orphaned claim / `[IN_PROGRESS]` for **your** role first (**TAKE**).
2. Read `ticketorder.md`. **Head** = first line **not** already ending in `:DONE`. Scan all open lines for **your** role token.
3. **EXIT** if **no** open `*-<YourRole>` lines remain on the list. Stop looping / cancel scheduled polls for this role. **End the turn — no keepalive tools.**
4. **WAIT** if open lines for you exist **but** head is not your role (or gate closed). Keep the loop alive via **one** scheduled sleep, then pass the Repository Sync Gate before re-reading; do **not** skip down the file. **After the status line, no empty shell noops or tight tool loops.** Vendor-neutral: Claude/Grok/Codex/etc. all obey the same ban.
5. **TAKE** if head **is** your role and gate-open (`[APPROVED]` for QA, `[READY_FOR_DEV]`/`[IN_PROGRESS]` for Dev). On successful handoff (commit+push): rewrite that one line to `<id>-<Role>:DONE`, then **immediately re-enter the loop through the sync gate** (chain consecutive same-role heads in the same session — do **not** stop after one ticket).
6. Never mark `:DONE` on a CANNOT terminal state — the open line keeps the route blocked.

This **replaces** “pick lowest-numbered `[APPROVED]` / `[READY_FOR_DEV]`” and **supersedes** the old “orchestrator-spawned stops after one ticket” rule (2026-07-25). Architect-written interleaving (QA→Dev→QA→Dev or direct-to-Dev) is followed exactly; QA cannot race ahead of a Dev line that is still the head; Dev chains UI-lane `*-Dev` heads when they become head.

```
QA:   head 242-QA → tests → READY_FOR_DEV → push → mark 242-QA:DONE
Dev:  head 242-Dev → implement → DONE → push → mark 242-Dev:DONE
QA:   head 243-QA → ...
```

### Why Not Parallel — the `rtest` Conflict

This is the core reason. If QA promotes WD-N+1 to `[READY_FOR_DEV]` (committing its failing tests) **while Dev is still working on WD-N**, then Dev's full `rtest` run shows **WD-N+1's failures too** — noise that pollutes Dev's signal ("are *my* tests green?") and risks masking real regressions. Worse, if two tickets share a production file (a hotspot), QA's tests for N+1 may be written against a view/API that Dev is mid-refactor on — the tests won't even compile, blocking Dev's run entirely.

The pipeline eliminates this: when Dev runs full `rtest`, the only non-green items are Dev's own ticket's tests (going from red → green as work progresses). The existing suite stays green. The next ticket's tests don't exist yet.

### Strict Pipeline (preferred) vs. Staged-One-Ahead (looser)

**Strict pipeline (default, preferred):** QA does **not** promote the next ticket (or commit its tests) until Dev `[DONE]`s the current one. Dev's full `rtest` then only ever shows: their own ticket's tests (going green) + the existing green suite. No noise. No compile conflicts.

**Staged-one-ahead (allowed only when safe):** QA may write the next ticket's tests and promote it to `[READY_FOR_DEV]` **one step ahead** of Dev, but **only when ALL of these are true**:
1. The next ticket's deps are `[DONE]` (its prerequisite tickets have landed).
2. The next ticket shares **no production file** with the ticket Dev is currently implementing (no shared-file hotspot).
3. The next ticket's tests **compile cleanly** against the current committed state (so Dev's `rtest` still compiles).
4. The user has not directed otherwise.

If any of these fail, fall back to the strict pipeline. When in doubt, strict.

### Test Execution Tiers in the Pipeline (see `skills/rtest/SKILL.md` Test Execution Policy)

Independently of strict vs. batch-fast mode, every agent uses the **four run modes** from `rtest/SKILL.md` instead of cold-running the whole suite at every step:
- **QA** proves RED with `rtest --fast` / `rtest --targeted` — **never** a cold full suite.
- **Dev** iterates with `rtest --fast`, confirms GREEN with `rtest --targeted`, then runs the **incremental** `rtest` (build cache **on**, **no `clean`**) once before `[DONE]`. Because the build cache skips unchanged test classes, this "full" run executes only the classes the ticket actually touched — cheap enough that it need not be deferred in the strict pipeline.
- The **cold** full suite (`rtest --full-cold`, i.e. `clean` + `--no-build-cache`) runs **ONCE per batch** as the regression backstop — see "Common Dev Convention" below, where it is folded into the artifact build.

This removes the two cold full-suite runs that previously happened per ticket (QA verify-fail, Dev confirm-fail). It **complements** Batch-Fast Mode below: even in the strict pipeline the per-ticket run is now incremental, and batch-fast mode's "one full run at end of batch" becomes the `rtest --full-cold` gate. Choosing a run mode is an **execution** decision — it does **not** move role boundaries: QA still authors every test and size tag, Dev still authors none.

### Compile-Blockers Must Land First

If the `rtest` suite **does not compile** (stale test refs, removed production IDs, signature drift), **no other QA or Dev work can proceed** — QA can't verify new tests fail, Dev can't verify fixes pass. A compile-blocker ticket must be `[DONE]` **before the pipeline starts**. The Architect is responsible for identifying compile-blockers in the batch and placing them **first in `ticketorder.md`** (and/or renumbering — see the numbering note below).

### Ticket Numbering & route order (supersedes bare "pick lowest")

When `<project>/ticketorder.md` exists, **route order wins** — QA/Dev do not independently pick lowest-numbered tickets. The Architect encodes dependency order as explicit `<id>-QA` / `<id>-Dev` lines (and may insert a compile-blocker line first).

If `ticketorder.md` is **absent** (legacy project), fall back to: QA picks lowest-numbered `[APPROVED]`, Dev picks lowest-numbered `[READY_FOR_DEV]`. That fallback assumes lower number = earlier dependency. If a blocker is added late with a high number, **renumber** or **add a route file** — prefer adding/updating `ticketorder.md`. The `A` suffix (e.g., `WD-138A`) works **only if the base number is already lower** — `138A` < `139` because `8 < 9`. Do **not** use `WD-139A` to sort below `WD-139`.

### Shared-File Hotspots (Architect's responsibility)

When multiple tickets in a batch edit the **same production file** (e.g., `AddEditActivity.kt`, `MainActivity.kt`, `DetailActivity.kt`), they must **never** be in Dev simultaneously — two Dev changes to the same file conflict at `git merge`/`pull` time and one overwrites the other. The Architect must:
1. Identify shared-file hotspots when handing off the batch.
2. Publish a required Dev order for those tickets (e.g., "WD-123 first → then WD-125/126/127/129, never in parallel with each other").
3. Ensure every Dev claim starts only after the Repository Sync Gate fast-forwards the clean worktree, so the prior ticket's edits to the shared file are present.
4. Never release a dependent ticket to QA before its prerequisite is `[DONE]` (else QA writes tests against code/views that don't exist yet).

### Fallback: Targeted `rtest` Runs

If the pipeline can't be strictly maintained (e.g., QA already promoted the next ticket and Dev is mid-work), Dev may use **targeted test runs** during development to avoid the noise:
```
.\rtest.bat --tests "com.whatdate...<DevTicketClassName>"
```
Dev runs the **full** `rtest` only at the end (before promoting to `[DONE]`). At that point any `[READY_FOR_DEV]` failures from tickets QA staged ahead are expected and ignorable — Dev's own ticket's tests must be green, and no previously-green test may have gone red. This is a fallback, not the default — the strict pipeline is always preferred.

### Batch-Fast Mode (optional — large independent batches)

When the active batch is **large** (e.g., 8+ tickets) AND the tickets are **independent** (no shared-file hotspots, no DB schema/migration interactions, no shared layout IDs — see "When to use vs. avoid" below), the user may **declare batch-fast mode** at handoff. This replaces the per-ticket full `rtest` with a lighter per-ticket check + one full run at end of batch. The time savings are significant: a 10-ticket batch drops from ~10 × full-rtest to ~10 × (targeted + compile-check) + 1 × full-rtest — typically ~15 minutes saved on a 120s-per-full-run suite.

**Batch-Fast Mode flow (replaces the Dev side of the pipeline when active):**
```
For each ticket N in the batch:
  Dev:  git pull → claim N (IN_PROGRESS) → implement
  Dev:  targeted rtest:  .\rtest.bat --tests "com.whatdate...<DevTicketClassName>"  → green
  Dev:  compile check:   .\gradlew.bat compileDebugUnitTestKotlin --console=plain   → BUILD SUCCESSFUL
  Dev:  promote N → [DONE] → commit → push
  (NO full rtest per ticket — that's the optimization)
QA:    write tests for N+1 → [READY_FOR_DEV] → commit → push  (one ahead, as usual)

After the last ticket in the batch is [DONE]:
  Dev:  ONE full  .\rtest.bat  → must be BUILD SUCCESSFUL, all green
  If failures: bisect (git stash + targeted run per ticket, or git bisect on the ticket commits) to find which change broke what. Fix forward (new commit or amend the culprit ticket's commit).
```

**The per-ticket compile check is the key guardrail.** It catches the most common cross-ticket breakage — a test referencing something your change removed (an ID, a method, a field) — in ~10 seconds instead of a full 120s+ run, without the regression-compounding risk of skipping all verification. It does NOT catch runtime regressions (a test that compiles but fails because behavior changed) — those are caught by the end-of-batch full run.

**When to use Batch-Fast Mode (all must be true):**
1. The user explicitly declares "batch-fast mode" at handoff (this is never the default — the strict pipeline with full-rtest-per-ticket is the default).
2. The batch is large enough that the time savings matter (typically 8+ tickets).
3. The tickets are **independent**: no shared-file hotspots (check the Architect's shared-file map for the batch), no DB schema/migration interactions, no shared layout IDs that could collide.
4. No `[CANNOT]` tickets are blocking (those must resolve first, per normal flow).

**When to AVOID Batch-Fast Mode (use strict pipeline with full-rtest-per-ticket instead):**
- Multiple tickets touch the same production file (shared-file hotspot) — two changes to the same file can interact in ways a compile check won't catch.
- DB schema migrations are in the batch (migration A might break migration B's tests; a compile check won't catch runtime migration logic errors).
- Layout ID renames are in the batch (a renamed ID can break another ticket's tests at runtime, not compile time, if the test uses runtime `resId()` lookups instead of `R.id.`).
- The batch has `[CANNOT]` tickets or unresolved dependencies.
- The batch is small (≤4 tickets) — the time savings don't justify the deferred-regression risk.

**The end-of-batch full run is non-negotiable.** Even in batch-fast mode, the batch is not "done" until one full `rtest` confirms all green. If the end-of-batch run finds failures, the batch is not shipped (no APK build, no "batch complete" report) until the failures are bisected and fixed. The bisect is Dev's responsibility: `git stash` the current state, `git checkout <ticket-N-commit>`, run targeted tests, find the culprit, fix forward.

**Risk honesty (why this is optional, not default):** if ticket 5's changes break ticket 2's tests, you don't find out until after ticket 10. Then you're bisecting 6 tickets' diffs to find the culprit — harder than catching it at ticket 5's promotion (which the strict pipeline does). Batch-fast mode trades early regression detection for speed. It's worth it when tickets are genuinely independent (the risk is low) and the batch is large (the savings are real). It's not worth it when tickets share files or interact (the risk is high) or the batch is small (the savings are trivial).

### Summary (the rule, in one line)

**Shared worktree, shared consequences → pipeline, not parallel → QA at most one ticket ahead → strict pipeline (full rtest per ticket) unless all four staged-ahead safety conditions hold → batch-fast mode (targeted + compile-check per ticket, full rtest at end) only when user-declared + tickets independent + batch large → compile-blockers first → shared-file tickets never concurrent.**

---

## Workspace Hygiene & Memory Maintenance (All Agents)

To prevent `MEMORY.md` from becoming overloaded with stale ticket logs and history, all agents (Architect, QA, Dev) must follow this cleanup rule:

- **When a task is done:** Whenever you complete a batch of tickets, or whenever an active ticket reaches a terminal state (`[DONE]`, `[CANCELLED]`, `[REVERTED]`, `[DEFER]`), you must **move** its status log entry from the active pending section of `MEMORY.md` into a separate `# HISTORY` section at the very bottom of the file.
- **HISTORY Section:** The `# HISTORY` section should be clearly segregated at the end of `MEMORY.md`.
- **Do Not Read History Unprompted:** All agents are strictly instructed to **ignore** the `# HISTORY` section during their initialization reads. Only read or summarize the history section if the user explicitly asks you to. The active `MEMORY.md` should only contain the 3-4 most recent ongoing activities.

---

## Common Dev Convention (All Symphony Projects)

Every time the last active ticket for the current cycle is promoted to `[DONE]` (i.e., no more `[READY_FOR_DEV]` or `[IN_PROGRESS]` remain), **immediately build the project artifact** and place the output in the project root folder.

For **WhatDate Android**: run `rtest --full-cold` first to confirm green — this is the per-batch regression backstop (`clean` + `--no-build-cache`); **only if it is fully green**, run `./gradlew.bat clean assembleDebug --console=plain`, then copy `app\build\outputs\apk\debug\whatdate-v2.apk` to `WhatDate-debug.apk` in the project root. Report the file size. If `rtest --full-cold` is not green, do NOT build the artifact — bisect to the culprit ticket and escalate the regression.
test.bat`For **WhatDate Android**: run `rtest --full-cold` first to confirm green — this is the per-batch regression backstop (`clean` + `--no-build-cache`); **only if it is fully green**, run `./gradlew.bat clean assembleDebug --console=plain`, then copy `app\build\outputs\apk\debug\whatdate-v2.apk` to `WhatDate-debug.apk` in the project root. Report the file size. If `rtest --full-cold` is not green, do NOT build the artifact — bisect to the culprit ticket and escalate the regression.

For **other projects**: adapt the exact build + copy command from the project `SKILL.md`, but always deliver the build artifact automatically — do not wait for the user to request it.

---

## WhatDate Android — Architect Role (Primary Flow)

For the WhatDate Android project the active lifecycle is the 3-agent Android flow.

**Architect responsibilities**:
- Act as gatekeeper. Analyze user requests for root cause and alignment with architecture + the core philosophy ("WhatDate is a memory attached to one or more dates...").
- On every init: re-read the philosophy + invariants from `whatdate-folder/MEMORY.md`.
- Produce `[APPROVED]_WD-XXX_...md` tickets only after deep analysis. Provide rich `## QA / Testing Instructions` and `## Architectural Constraints`.
- Follow the critical **Batch / Interactive Session Rule** (detailed above and in your profile).
- All changes (no matter the size) that affect app behavior, UI, or runtime must go through a full `[APPROVED]` ticket. The Architect never writes production code directly.
- Never bypass the file-based handoff. Tickets are the only API.

See the expanded `.agent_profiles/architect_profile.md` for the complete current playbook.

---

## Content Project Variant — Wisdom Capsules Symphony

For content-based projects (e.g., Wisdom Capsules), the pipeline uses 5 agents split across two completely separate lifecycles: **Content Creation** and *
---

## Role Discipline & Repo Safety — HARD RULES (added 2026-07-05 after the WD-170/170A incident)

These rules exist because of real damage: a combined multi-ticket commit made through a corrupted git index, 32 test files silently stored as 0-byte blobs on origin, stale `index.lock` files blocking all git operations, wholesale CRLF→LF rewrites polluting diffs, a QA scratch file committed to main, and edits to files no ticket authorized. Every rule below is non-negotiable for ALL agents (QA, Dev, Tester, Implementer — and the Architect where applicable).

### 1. Scope lock (the anti-berserk rule)
- You may modify ONLY the files named in your ticket's `## Architectural Constraints` "MAY touch" list, plus test files your role owns (QA/Tester) or files the ticket's compile-sweep step explicitly authorizes.
- If completing the ticket seems to require touching ANY other file: STOP. Rename to `[CANNOT_QA]`/`[CANNOT_DEV]`, write findings, end your turn. Improvising outside scope is a protocol breach even if the code would work.
- Hard tripwire: before committing, run `git status` and read the file list. If it contains a file you cannot map to your ticket's allowed list, DO NOT COMMIT — escalate.

### 2. Commit discipline
- **One ticket = one commit** (Dev squashes QA's test commit + own work per the global-skill Git workflow). NEVER combine multiple tickets into one commit.
- Commit and push IMMEDIATELY on reaching your ticket's terminal state (QA: after RED verify + promote; Dev: after GREEN + promote). A `[DONE]` rename sitting uncommitted in the worktree is a protocol violation — a crash loses finished work.
- Delete every scratch/temporary file you created (fingerprint-capture scratch tests, probe scripts, log dumps) BEFORE staging. `git add -A` with scratch files present = pollution on main.

### 3. Encoding & line-ending discipline
- This is a Windows repo: source files are UTF-8, most with CRLF. Preserve BOTH on every file you touch.
- NEVER rewrite a file wholesale when a targeted edit suffices. A diff that shows every line changed means you broke line endings — revert and redo.
- PowerShell landmines: `>` / `>>` redirection writes UTF-16 — never redirect onto a source file. Use `Set-Content -Encoding utf8` only when explicitly writing whole files, and prefer editor-style targeted edits always.
- Before committing, sanity-check your diff: `git diff --stat` file count must match the files you intentionally changed. Wholesale-rewrite entries (or "Bin" markers on .kt/.xml text files) mean encoding damage — fix before commit.

### 4. Git integrity gates
- If ANY git command reports index errors (`unknown index entry format`, `bad index`), or a file you know has content shows as 0 bytes: STOP ALL WORK. Do not commit, do not "fix" by re-adding. Report to the Architect/user. A commit through a corrupted index poisons origin.
- Post-commit, pre-push integrity check (mandatory):
  ```powershell
  git ls-tree -r -l HEAD | Select-String "\.(kt|java|xml|proto)" | Where-Object { ($_ -split "\s+")[3] -eq "0" }
  ```
  Output must be EMPTY. Any 0-byte source blob = do not push, escalate.
- Never kill a git process mid-operation. If you find `.git/index.lock` at session start: verify no git process is running, then delete the lock, then run the standard fetch/status sync check.

### 5. Working-directory hygiene
- Verify your current directory before EVERY script or bulk file operation. The `app/app/` nested-duplicate directory in whatdate-folder was created by an agent running with the wrong cwd — never create paths like that; if your op would create a directory that mirrors an existing tree, your cwd is wrong.
- If the worktree is dirty at **any session or loop entry**, regardless of whether paths appear inside your ticket's scope: do NOT absorb, stage, commit, revert, stash, or overwrite it. Report the exact dirty status to the user and stop before claim/queue work. The pipeline's expected terminal state is clean by definition.

### 6. Berserk brake (self-check)
Ask yourself before every commit: "Can I name the ticket line item that authorizes each file in `git status`?" If the answer is no for any file — you have gone off-script. Stop, revert the unauthorized changes, escalate what you learned. An agent that follows a wrong plan slowly is recoverable; an agent that improvises quickly is not.

---

## Two-Lane Lifecycle (adopted 2026-07-05 — supersedes single-lane flow for UI work)

Robolectric UI tests proved high-cost/low-yield (constant reconciliation churn on redesigns, missed real UX bugs). The lifecycle now has two lanes; **the Architect declares the lane in every ticket header** (`**Lane:** backend` or `**Lane:** UI`).

### Backend lane (engine, data model, serialization, wire/proto, alerts, backup, privacy invariants)
Unchanged full Symphony: `[APPROVED]` → QA writes failing pure-JVM tests → `[READY_FOR_DEV]` → Dev → `[DONE]`. TDD stays mandatory here because these tests are cheap (seconds), precise, and guard what manual APK testing cannot catch (recurrence math, fingerprints, redaction).

### UI lane (screens, layouts, navigation, animations, styling)
- The Architect creates the ticket **directly as `[READY_FOR_DEV]`** — no QA agent involvement, ever. QA agents must not pick up UI-lane tickets (they will never see them in `[APPROVED]`).
- The ticket contains a **`## Manual Test Script (user)`** section instead of QA instructions: a numbered install-and-tap checklist the human runs on the APK. The user is QA for this lane.
- Dev's promotion gate to `[DONE]`: (1) `rtest --fast` green (pure-JVM tier must never regress), (2) `compileDebugUnitTestKotlin` BUILD SUCCESSFUL, (3) `assembleDebug` builds. NO full Robolectric run required.
- **Final acceptance is the user's manual pass.** If the user reports a bug against a `[DONE]` UI ticket, the Architect opens a follow-up ticket; the original stays `[DONE]` (history is truth).

### Retiring obsolete UI tests (controlled demolition)
Legacy Robolectric UI tests are retired screen-by-screen as the redesign replaces each screen:
- Each UI-lane redesign ticket carries a **`## Retired tests`** list naming the exact test files/methods Dev must DELETE in the same commit. Deleting Architect-listed obsolete tests is authorized for Dev and is not "writing tests."
- Dev never deletes or edits a test that is not on the ticket's Retired list. A compile break in a non-listed test = `[CANNOT_DEV]`, as always.
- Until a screen is redesigned, its existing Medium tests remain and the periodic full suite may still run them — but full-suite green is no longer a per-ticket gate for UI-lane work.

### What does NOT change
Role boundaries, scope lock, commit discipline, encoding rules, integrity gates, CANNOT flow, one-agent-at-a-time — all Hard Rules above apply identically in both lanes.

---

## Regression Guard — MANDATORY on every ticket (added 2026-07-18 after repeated re-appearance of fixed bugs)

**The problem this solves:** fixed bugs kept coming back (import crash, Home opening at the bottom, composer visibility, blank calendar). Root cause was structural, not carelessness: **UI-lane tickets shipped with zero automated protection**, so the next agent touching a hotspot file silently undid earlier fixes, and nobody noticed until the user hit it again.

### 1. Every ticket carries a `## Regression Guard` section
It names, in one line each: **the behaviour being fixed** and **the exact test that pins it forever**. No ticket reaches `[DONE]` without its guard test existing and passing. This applies to BOTH lanes.

### 2. Two-Lane amendment — UI *behaviour* is now testable-and-tested
The Two-Lane rule stands for **visuals** (spacing, colour, typography, animation → manual only). It NO LONGER exempts **behaviour**. If a UI fix can be expressed as a state assertion, it MUST get a lock test:
- scroll/anchor positions · visibility rules · enable/disable states · ordering/sorting · navigation and back-press precedence · dismissal paths · which view is bound after an event · lifecycle survival (detach → re-attach).
These are cheap Robolectric assertions (milliseconds), not the brittle pixel tests we retired.

### 3. Behaviour Locks live in one accumulating place per area
`app/src/test/java/com/whatdate/locks/<Area>BehaviorLocksTest.kt` (e.g. `HomeBehaviorLocksTest`, `CalendarBehaviorLocksTest`, `ImportBehaviorLocksTest`). Rules:
- One test method per locked behaviour, named `lock_<wdTicket>_<behaviour>()`, with a one-line comment stating the user-visible symptom it prevents.
- **Locks are append-only.** No agent may delete, weaken, or `@Ignore` a lock. If a lock genuinely contradicts a new approved requirement, the ticket must say so explicitly and the Architect must approve the change in writing inside that ticket.
- A ticket that fixes a user-reported bug adds its lock in the SAME commit as the fix.

### 4. `REGRESSION_CHECKLIST.md` (project root) is the human-readable index
One line per locked behaviour: symptom · ticket · test method. Updated in the same commit. It is the first thing to read when a user reports "this came back".

### 5. Batch-close gate
Before the artifact build, the closing agent runs the full suite and confirms **every lock is green**. A red lock blocks the APK — no exceptions, no "unrelated failure" reasoning.

---

## Orchestrator (added 2026-07-21 — the automated dispatcher; canonical: `.agent_profiles/orchestrator_profile.md`)

A first-class role invoked per project like any other (`init <project> orchestrator`) that automates today's manual role invocation. Cross-project orchestration = one instance per project, parallel by construction. Dumb-by-design: it follows the Architect's route and Symphony's status gates; it never reasons technically, never writes tickets/code/tests, never weakens guards.

**The three-file contract:** `<project>/ticketorder.md` (Architect-authored route: `<ticket>-<role>` per line, top-down; **Architect authors/reorders/prunes lines; QA and Dev may only append `:DONE` to the single line they just completed**) + `taskagent.md` (constant-size model rotation rings; Orchestrator rotates head→tail atomically before each spawn) + `orchestrator model map.md` (static slug legend). Route says WHAT next, ticket status says WHEN (gate), ring says on WHICH model. **QA/Dev themselves read `ticketorder.md` only after passing the Repository Sync Gate on every loop entry** (not only the Orchestrator), so manual pure-Grok/Claude seats follow the same current order.

**Role Work Loop (2026-07-25 — all roles except Architect):** EXIT if no open work for your role remains on the list; WAIT if your work exists but is not the head; TAKE when head is yours and **repeat** (chain same-role heads). On WAIT/EXIT: **no keepalive tool spam** — end the turn after one status line (`Agent role.md` + `global-skill` §"No Keepalive"). Canonical text: `Agent role.md` §"Role Work Loop". Supersedes "orchestrator-spawned stops after one ticket."

**Gates (android):** QA runs at `[APPROVED]`→`[READY_FOR_DEV]`; Dev at `[READY_FOR_DEV]`→`[DONE]`; SRTL review ONLY via a human-added `<T>-SRTL` route line at `[DONE]`. **Content-web:** per the existing suffix lifecycle incl. the Designer bridge (`[FINAL]` capsule → `*_APPROVED` ticket). CANNOT states are AUTO-routed (Architect/Designer → SRTL → human escalation, bounded at 3); `*_STALE` is ALWAYS human-escalated; `[DONE]`/`*_FIXED` review is ALWAYS human-gated.

**State discipline:** position in the route is DERIVED from ticket statuses each pass (crash-resumable, no pointer file). The only Orchestrator-written state: `tickets/.claims/<T>-<Role>.claim` markers (in-flight + same-role serialization; timeout → bounded retry with the next ring model → escalation) and the rotated ring line. Per-project dispatch is strictly serial (this IS the git isolation); different projects run in parallel. Logs (`orchestrator/LOG.md`, `orchestrator/INBOX.md`) are bounded/rotated and never loaded wholesale into context.

**Intake:** human requests land as `<project>/requests/[NEW]_REQ-<id>.md` (verbatim + `priority:` + optional `depends:`); the Architect/Composer self-selects the oldest on init, tickets it, appends route lines, renames `[TICKETED]`. Specialists receive NOTHING in-band — ever.

**Route hygiene (Architect duty, 2026-07-25; amended 2026-08-01):** the Architect prunes completed lines from `ticketorder.md` **only at batch close or when opening a new batch — never mid-batch.** The route is both a dispatch queue *and* the live record of the open batch; mid-batch writes (unblocking a `[CANNOT]`, reordering, revising the note) leave every existing line intact and touch only the line(s) concerned plus line 2. **Line notation (settled 2026-08-01):** `WD-294-QA` = queued/unfinished (**the absence of `:DONE` is the only open signal**) · `WD-294-QA:DONE` = that role finished · `WD-283-Dev:DONE:SRTL` = Dev finished **and SRTL already reviewed it** — terminal, since SRTL fixes and re-tests directly rather than queuing · `WD-284-SRTL:DONE` = an explicitly routed SRTL pass completed. **Never prune:** any line without `:DONE` · `[HOLD]`/`[DEFER]` lines · in-flight lines (`[IN_PROGRESS]` or an open `.claims/` marker) · any line whose human gate is unanswered. Always re-scan `tickets/` live in the same turn; never prune from memory. Line 2 is a living batch note, rewritten on every write. **QA/Dev write only the `:DONE` suffix on their finished line** — they never prune or reorder. The Orchestrator never authors route content (it may read `:DONE` markers as completion signals). Full rationale and the incident that forced the amendment: `.agent_profiles/architect_profile.md` §"ticketorder.md Hygiene".

**Adding a role later** = Registry row + one gate row in the profile + a ring line. No structural change.
