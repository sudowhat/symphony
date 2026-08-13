---
name: global-skill
description: Global workspace rules, behavior conventions, agent initialization macros, and Git workflow.
---

# Global Skill

This skill defines general behavior rules and project-wide conventions used by **ALL** agents in the Symphony Protocol. Every agent must read this file after their role profile and before any work.

---

## CRITICAL: No Exploratory Work Before Context Load

**Strict Rule**: You MUST NOT call listing tools, read source code, edit files, or perform any exploratory work until you have completed the full initialization sequence from `Agent role.md` (Steps 1–4) and read this file.

### Discovering Hidden / Dot Directories

Directory listing tools often hide dot-directories (e.g., `.agent_profiles`, `.git`, `.idea`). When you need to explore them, use terminal commands with PowerShell:

```powershell
Get-ChildItem -Path "C:\Users\pooji\Documents\symphony" -Force
Get-ChildItem -Path "C:\Users\pooji\Documents\symphony\whatdate-folder" -Recurse -Force
```

Always prefer explicit full paths.

### The "Main Folder" Rule

Whenever the user requests to place, copy, or move a file to the "main folder", this strictly refers to the **root project folder** (e.g., `whatdate-folder/`), and NOT any internal module directories like `app/src/main/`.

---

## Performance Is a First-Class Constraint — The Mantra (All Agents, All Projects — added 2026-07-16, user mandate)

**We breathe performance.** Performance is a requirement on equal footing with correctness — never an afterthought, never a "we'll optimize later." **No feature may ship that compromises perceived or actual performance.** A feature that works but janks, stutters, or makes the user wait is **not done**.

Binding rules for every agent, every project:

1. **Never block the UI / main thread** with I/O, database, disk, network, serialization, or heavy computation. That work goes off-main (background dispatcher / worker / async) with the result delivered back to the UI. A screen must render its shell immediately; data fills in without freezing interaction.
2. **Warm the next screen while the current one loads.** Where the user's next action is predictable, prefetch/cache it in the background during the current load — e.g., pre-cache the detail of the first few list items while the list itself is loading; pre-cache a week/month's entries while that calendar page is loading. Latency the user never sees is latency defeated.
3. **Cache on hot paths; compute once.** Do not recompute or re-read the same data on every open, scroll, or redraw. Materialize/memoize what is stable; invalidate precisely.
4. **The Architect must state a performance budget/approach in every ticket that touches a user-facing hot path** (open/scroll/interaction): explicitly say how it stays fast (async, cached, prefetched, paginated) under `## Architectural Constraints`. QA/Dev must not regress load, scroll, or interaction latency; "it works" is insufficient if it is slower.
5. **Measure on anything perf-sensitive.** State the before/after (cold open, scroll jank, interaction delay) rather than assuming. Prove it's fast, don't hope.
6. **On conflict, performance wins.** If a feature cannot be delivered without degrading performance, that is an architectural problem to solve or escalate — not a tax to quietly pass to the user. Only the user may explicitly trade performance away, in writing.

This mantra is vendor-neutral and applies to every project under `Symphony/`. Mirror the one-liner into each project's standing decisions where useful, but this file is canonical.

---

## Read Live State From the Authoritative Project Source (All Agents — clarified 2026-08-09)

Symphony is multi-agent and stateless: ticket filenames/statuses, `MEMORY.md`, Git state, and built artifacts can change **constantly and externally**. A local filesystem mount can be stale, and a cloud agent can be stale if it relies on a previously fetched GitHub file. Reading either as current truth yields wrong ticket state, branch state, or artifact claims.

**Hard rule — re-read volatile state from its authoritative source immediately before asserting it:**
1. **Project files** (`MEMORY.md`, ticket bodies/listings, claims, source): a **local CLI** reads the canonical on-disk project path; a **direct-repo/cloud agent** fetches the exact path from the target project's current remote branch/ref. Neither may rely on conversational memory, an earlier listing, or a sandbox/cache snapshot.
2. **Git state:** a local CLI reads the live worktree (`status`, branch, upstream, `index.lock`). A direct-repo/cloud agent reads the live target ref/commit and the returned revision of every file it will assert or edit. A cloud agent cannot see uncommitted local disk work and must never invent a `REPO_DIRTY` report for it.
3. **APK / build-artifact freshness (timestamps):** never claim a build's age from a cached mount. A direct file reader returns contents, not `mtime`; if the authoritative host cannot stat the real file, ask the user rather than reporting a possibly-stale timestamp.
4. **Re-read before you assert:** whenever time has passed or another agent may have acted, re-read the relevant disk path or remote ref/file immediately before making a claim. Do not repeat an earlier in-session read as if it were current.

## Skill: Clarify Before Acting (Token-Efficient Mode)

Before executing any task, classify the request into one of four confidence levels. This is one of the highest-ROI optimizations you can make.

### Level 1 – Clear (90–100% confidence)
Requirements are sufficiently specified. **Proceed immediately.** State assumptions only if helpful.

### Level 2 – Minor Ambiguity (70–90% confidence)
Some details are missing but reasonable defaults exist. **Proceed using sensible assumptions.** Explicitly mention them. Do not stop to ask questions.

### Level 3 – Material Ambiguity (40–70% confidence)
Different interpretations would significantly change implementation, architecture, cost, security, APIs, schemas, or user experience. **Pause.** Ask only the minimum targeted questions required. **Maximum: 3 concise questions.**

Good questions: "Should authentication use JWT or session-based?" "Is backward compatibility required?"
Bad questions: "Tell me everything you want." "Any other requirements?"

### Level 4 – Critical Ambiguity (<40% confidence)
The task is too underspecified and proceeding would likely waste work. **Ask a small set of high-leverage questions.** Maximum: 5 questions. They must unblock major decisions.

### Question Quality Rules

Questions must be:
- Specific
- Decision-oriented
- High impact
- Answerable in one sentence
- Non-overlapping

### Prevent Clarification Loops

Never enter infinite clarification cycles.
- **Maximum clarification rounds: 2**
- **Maximum total questions per task: 5**
- If questions remain unanswered: state assumptions explicitly, proceed with safest defaults, mark assumptions as needing validation later.

### Cost-Aware Execution

Before asking a question, estimate:
- **Cost of asking**: Additional tokens, waiting for user response
- **Cost of guessing**: Potential rework, architecture mistakes, security risks, breaking changes

Ask questions only when: `Cost(guessing) > Cost(asking)`. Otherwise proceed.

---

## No Keepalive / No Tool Spam (All Roles, All Vendors — added 2026-07-30)

**Incident:** agents in WAIT (or after EXIT) burned large volumes of tokens by issuing empty shell commands (`exit 0`, noops, tight sleep loops) every fraction of a second to "stay alive" instead of performing the Role Work Loop's single real 300s sleep and then re-reading.

**Hard rule — vendor-neutral (Claude, Grok, Codex, Cursor, Gemini, …):**

1. After reporting **WAIT**, run exactly one real 300s same-session sleep (`Start-Sleep -Seconds 300` or equivalent), then re-read the work list. Do not end the turn before the sleep unless the host provides a true same-session scheduled wake primitive.
2. After reporting **EXIT** (or a non-loop "nothing to do right now" status), **end the turn with zero further tool calls**.
3. Never use tools as a heartbeat: empty commands, `exit 0`, `echo`/`Write-Host` noops, tight short sleep loops, or repeated status commands that do not change decisions.
4. WAIT sleep is one scheduled/blocking wait for the configured interval (`Agent role.md` § Role Work Loop), then a real re-read of the work list — not continuous tool chatter between ticks.
5. EXIT cancels scheduled polls for that role; do not re-arm until new work exists or the user re-inits.
6. Canonical detail lives in `Agent role.md` §"Role Work Loop" → **"No keepalive / no tool spam while WAIT or after EXIT"**. This section is the global pointer so every init load hits the rule even before the full loop text is re-applied.

This complements Cost-Aware Execution: wasted tool turns have the same cost profile as wasted clarification rounds.

---

## Audible Attention Signal — Ring When You Stop For The User (All Roles, All Vendors — user ruling 2026-08-09)

The user is not watching the terminal. An agent that finishes, blocks, or asks a question and then
sits silently has effectively stopped without telling anyone, and the time between "agent needs a
decision" and "user notices" is dead time.

**Ring a 6-second bell whenever you stop and need the user.** That means: you have finished the
work and are handing back; you are blocked and cannot proceed; you have asked a question and are
waiting on the answer; or you are ending a session.

```powershell
try {
    $player = New-Object System.Media.SoundPlayer "C:\Windows\Media\Alarm01.wav"
    $player.Play(); Start-Sleep -Seconds 6; $player.Stop()
} catch {
    [Console]::Beep(800, 6000)
}
```

**Do not ring for:**
- routine progress inside a turn you are still working through;
- a background build or test finishing when you intend to keep going;
- anything the user does not have to act on.

A bell that fires constantly stops meaning "you are needed" and becomes noise the user learns to
ignore, which costs more than the silence it replaced.

**Related but separate:** `skills/blocker-resolution/SKILL.md` §"Genuine Block — CANNOT + Alarm"
defines its own alarm for the moment a ticket is renamed `CANNOT_*`. That one marks a pipeline
blocker for whoever is watching; this one marks that *you* are waiting on the user. Both may fire
for the same event — a CANNOT you cannot resolve is also a stop.

Windows-environment convenience, not a cross-vendor requirement: on another OS use the equivalent
audible or desktop notification, and skip silently if none exists. Never fail, delay, or withhold
the actual report because a sound could not play.

---

## Skill: Generic Status Command (All Roles)

**Trigger phrases**: "status", "update", "tell me the status", "what's the status", "give me an update", or close variants — from any user message, to **any agent regardless of role** (Architect, QA, Dev, Designer, Tester, Implementer).

When triggered, respond immediately without re-running the full init sequence (you should already be initialized into a project). Do the following:

1. **Scan** the current project's `tickets/` directory with `Get-ChildItem -Force` (brackets in filenames break globs without `-Force`/`-LiteralPath`).
2. **Filter to pending tickets only** — i.e., everything **except** `[DONE]`, `[CANCELLED]`, and `[REVERTED]`. Pending statuses include (across both lifecycles): `[DRAFT]` (Architect/Designer or Composer design-in-progress), `[APPROVED]`, `[READY_FOR_DEV]`, `[READY_FOR_IMPL]`, `[IN_PROGRESS]`, `[CANNOT_QA]`, `[CANNOT_DEV]`, `[CANNOT_TEST]`, `[CANNOT_IMPL]`, `[DEFER]`, `[HOLD]`. A `[DRAFT]` ticket is pending (not done) but belongs to the Architect/Designer/Composer — QA/Dev must not pick it up.
3. **Never dump full ticket bodies.** For each pending ticket, output only:
   - The ticket filename (status prefix + ticket number + short name)
   - A **2-3 line summary**: what the ticket is about, what stage/status it is currently in, and (if applicable) what is blocking it or what the next action/owner is.
4. If there are **no pending tickets**, say so in one line (e.g., "No pending tickets — project is stable.").
5. Do not take any action beyond reporting (no renames, no edits, no auto-proceed) — a status request is read-only.

This behavior is project-agnostic and role-agnostic: it applies to whichever project/role the agent is currently initialized into, using that project's own `tickets/` directory.

---

## Skill: Long-Running Commands (Gradle / rtest / assembleDebug) — Don't Block the User

**Problem:** When an agent runs `.\gradlew.bat ...`, `.\rtest.bat`, `assembleDebug`, or any command that takes 30s–10min **synchronously** via a single bash tool call, the tool call blocks until the command finishes. During that time the agent **cannot send any message to the user** — it appears frozen/unresponsive. If the command throws a huge Java stack trace, output-capture/processing adds further latency on top of the build time itself. Users have reported 10-minute silences this way.

**Rule: Any command expected to take more than ~30 seconds MUST be run in the background** with output redirected to a temp file, then polled in short subsequent tool calls. This keeps each individual bash call short (seconds, not minutes) so the agent regains control frequently and can update the user between polls.

### Background-launch pattern (Windows PowerShell)

```powershell
# 1. Launch in background, redirect output to a temp file, return immediately
$logFile = "$env:TEMP\opencode\gradle_build.log"
$errFile = "$env:TEMP\opencode\gradle_build.err"
$doneFile = "$env:TEMP\opencode\gradle_build.done"

# Clean any prior done-marker
Remove-Item -LiteralPath $doneFile -ErrorAction SilentlyContinue

Start-Process -FilePath ".\gradlew.bat" `
  -ArgumentList "compileDebugUnitTestKotlin","--console=plain" `
  -NoNewWindow -RedirectStandardOutput $logFile -RedirectStandardError $errFile `
  -PassThru | ForEach-Object {
    # When the process exits, write a done-marker. Register a background job.
    Register-ObjectEvent -InputObject $_ -EventName Exited -Action {
      Set-Content -LiteralPath $doneFile -Value $EventArgs.SourceEventArgs.Id
    } | Out-Null
  }

Write-Host "Launched gradle in background. Poll with: Test-Path $doneFile"
```

### Poll pattern (short calls, run between other work)

```powershell
# 2. Poll — returns instantly either way
if (Test-Path -LiteralPath $doneFile) {
  Write-Host "DONE. Exit status recorded. Tail of output:"
  Get-Content -LiteralPath $logFile -Tail 40
  Get-Content -LiteralPath $errFile -Tail 20 -ErrorAction SilentlyContinue
} else {
  Write-Host "Still running. Current tail:"
  Get-Content -LiteralPath $logFile -Tail 5 -ErrorAction SilentlyContinue
}
```

### Output trimming (mandatory for large outputs)

Even when reading the final output, **always trim** with `Select-Object -Last N` or `Get-Content -Tail N` (e.g. last 40 lines). Never dump a raw Java stack trace — those run 200+ lines and the tool's output-handling adds significant latency. The useful signal (BUILD SUCCESSFUL / FAILED + the actual error line) is always in the last ~40 lines for gradle, and the report file path is on the final line.

### When to use background vs. foreground

| Command | Expected duration | Mode |
|---|---|---|
| `git status`, `git fetch`, `git log`, `Move-Item`, file reads/edits | <5s | Foreground (single call) |
| `.\gradlew.bat compileDebugUnitTestKotlin` (incremental, warm cache) | 20–90s | Background if >30s expected; foreground ok if warm |
| `.\rtest.bat --tests "..."` (targeted, 1–3 classes) | 30–120s | Background |
| `.\rtest.bat` (incremental full suite, no clean) | 2–5min | **Always background** |
| `.\rtest.bat --full-cold` (clean + no cache) | 5–15min | **Always background** |
| `.\gradlew.bat clean assembleDebug` | 2–5min | **Always background** |

### Communication contract while background jobs run

- After launching a background job, **immediately tell the user**: "Launched `<cmd>` in the background. I'll report when it finishes." Then end the turn — do not spin in a poll loop within one turn (that re-creates the blocking problem).
- On the next turn (user prompt or your own continuation), poll once. If still running, say so in one line and end the turn. If done, report the trimmed result and proceed.
- Never run more than one background gradle/rtest job at a time on the same project — they share the Gradle daemon and build cache and will conflict.

### Temp directory

Use `C:\Users\pooji\AppData\Local\Temp\opencode\` (pre-approved for external access) for all background-job log/err/done files. Clean up old markers before launching a new job with the same name.

---

## Global Rule: Git Workflow for All Coding Projects

This applies only to projects that **already have an initialized Git repository**. A local CLI verifies that through a `.git/` directory at the canonical project root; a direct-repo/cloud agent verifies it by reading the selected project repository and its target ref. Not every project in `Symphony\` is git-ified yet. If the project has no repository, **skip all Git steps silently** (no fetch/pull/commit/push) and proceed with the rest of the workflow (ticket renames, file edits, rtest) exactly as normal — Git is a delivery/sync mechanism layered on top of the file-based ticket protocol, not a prerequisite for it. Re-check on each new init, since a project may be Git-ified between sessions.

> **HARD RULES:** `skills/agent-symphony/SKILL.md` §"Role Discipline & Repo Safety" (added 2026-07-05) is binding for every git operation below: scope lock (touch only ticket-listed files), one ticket = one commit, preserve UTF-8/CRLF encodings (never PowerShell `>` onto source files), STOP on any index error or 0-byte source blob, delete scratch files before staging, and the post-commit zero-byte integrity check before every push. Read it before your first commit of the session.

### Mandatory Pre-Loop Repository Sync Gate — Local CLI (All Roles, Every Loop Entry)

(Applies only when the project has a Git repository — see above.)

Cloud-based Architects and other authorised contributors can advance the remote branch while a CLI agent is idle or between sessions. A local worktree is therefore not current merely because it exists. **The clean, updated repository is the admission gate to every new unit of work.**

Run this gate from the canonical project root **after Path Integrity succeeds and before** reading any project-specific `MEMORY.md`, `SKILL.md`, ticket, claim, source file, build artifact, or queue entry:

- once during `init`;
- on every re-entry to Auto-Proceed after a terminal handoff;
- after each real 300-second WAIT wake, before re-reading the queue; and
- before a bare `proceed` / `next` / `continue` resumes work.

Do **not** run the gate while actively executing a claimed ticket: the worktree is expected to become dirty during that work. The ticket must reach its normal committed-and-pushed terminal handoff first; the next loop entry then uses this gate.

### Gate procedure

1. Confirm the current directory is the canonical project Git root. If Git cannot identify a worktree, report `REPO_SYNC_BLOCKED` and stop; never search for or create another checkout.
2. If Git reports an `index.lock`, index corruption, missing upstream, detached/unknown branch, or any other repository-state error, report `REPO_SYNC_BLOCKED` to the user and stop. Do not remove locks, repair the index, or rewrite Git metadata in this gate.
3. **Before any fetch or pull**, run:
   ```powershell
   git status --porcelain=v1 --untracked-files=all
   ```
   Any output — modified, staged, deleted, renamed, conflicted, or untracked files — is a dirty worktree. Report it to the user and stop immediately:
   ```text
   REPO_DIRTY: <project> (<branch>)
   <verbatim porcelain status>
   No repository update or ticket loop was started.
   ```
   Do not stash, reset, restore, clean, checkout, commit, stage, discard, or otherwise alter those files. Do not claim a ticket or enter the loop.
4. With a clean tree only, run `git fetch --prune`. A fetch failure is `REPO_SYNC_BLOCKED`: report the command failure and stop.
5. Compare `HEAD` with its configured upstream:
   - **behind only** → run `git pull --ff-only`;
   - **up to date** → continue;
   - **ahead only** → push the already committed local work with ordinary `git push`, then verify it is current;
   - **diverged** → report `REPO_DIVERGED` with branch/upstream and ahead/behind counts, then stop. Do not merge, rebase, force-push, or choose a side.
6. After any pull or push, re-run the porcelain check and verify `HEAD` equals its upstream. Any failure, remaining dirtiness, or non-fast-forward condition is `REPO_SYNC_BLOCKED`: report it and stop.

The gate is deliberately conservative. It updates only a clean worktree by fast-forward pull, and it never hides another person's local changes. A successful gate is the only condition under which the local CLI agent may inspect current project state or proceed to the role loop.

### Mandatory Direct-Remote Gate — Cloud / GitHub Access (All Roles, Every Loop Entry)

A direct-repo/cloud agent has no local project worktree. It does **not** run local-only `git status`, `git pull`, or a dirty-tree check against a nonexistent checkout. It instead treats the selected project's live target branch as its workspace and must complete this gate at the same entry points as the local CLI gate: during `init`, after a terminal handoff, after a real WAIT wake, and before `proceed` / `next` / `continue`.

1. Confirm the target repository and target branch/ref from the project bootstrap or the project's registered remote. Never use `sudowhat/symphony` as the project repository.
2. Fetch the current target ref/commit live, then fetch `.symphony-root`, `MEMORY.md`, `SKILL.md`, the active ticket/claim, `tickets/`, `ticketorder.md`, and any source file needed for the current decision from that same live branch. Retain the returned file revision/SHA for every file intended for edit.
3. Immediately before a claim, a current-state assertion, or a ticket/document/source edit, re-read the target ref and relevant file revisions. If the ref or relevant file revision has moved, stop and report:
   ```text
   REPO_REMOTE_MOVED: <project> (<target-ref>)
   Expected <recorded-ref-or-file-sha>; now <current-sha>.
   No claim, overwrite, or ticket-loop action was made.
   ```
   Do not attempt to merge, rebase, force-update, or silently replay the change.
4. Write only through an optimistic-concurrency operation that supplies the fetched file SHA/revision and/or advances the expected target ref without force. A revision mismatch or non-fast-forward failure is the same `REPO_REMOTE_MOVED` hard stop. Never use a blind overwrite or force ref update.
5. After a successful write, fetch the target ref and changed file(s) again and verify that the intended commit/content is present before reporting the handoff.

The direct-remote gate is the cloud equivalent of the local CLI gate. It cannot diagnose local uncommitted changes; only the CLI agent that can inspect the canonical worktree can issue `REPO_DIRTY`. Both modes fail closed before fresh work and follow the same ticket lifecycle.

### Dev amend exception

Dev may intentionally amend the QA commit for its **currently active** ticket, which temporarily makes its local branch differ from the remote. That is not a new loop entry. Record the upstream remote, remote ref, and SHA at claim time in the Dev claim, fetch immediately before the final push, and use `git push --force-with-lease=<recorded-remote-ref>:<recorded-upstream-sha>` against that recorded remote. If the remote SHA changed, the lease must fail and Dev reports the conflict to the user; it must never overwrite a cloud Architect commit. Once that push succeeds and the tree is clean/current again, the next loop entry runs the normal gate.

### QA Agent — After Test Authoring

After writing failing tests and promoting the ticket from `[APPROVED]` to `[READY_FOR_DEV]`:
1. Stage all changed files (tests + renamed ticket):
   ```bash
   git add .
   ```
2. Commit with the ticket file name **minus the status prefix** as the commit message:
   ```bash
   git commit -m "WD-XXX_description_of_ticket.md"
   ```
3. Push to the remote repository:
   ```bash
   git push
   ```

### Dev Agent — After Implementation

The Repository Sync Gate has already fast-forwarded the clean worktree before Dev claims a ticket. Do not run `git pull` again while the active ticket has uncommitted work or an intentional amended commit.

After implementing the solution, running all tests to green, and promoting the ticket from `[IN_PROGRESS]` to `[DONE]`:
1. Stage all changes (production code + renamed `[DONE]` ticket):
   ```bash
   git add .
   ```
3. Combine all commits for this ticket (QA test commit + all Dev commits) into **one single commit** using squash/amend. The commit message must be the ticket file name minus the status prefix:
   ```bash
   git commit --amend --no-edit
   ```
   (Or use interactive rebase `git rebase -i HEAD~N` if needed.)
4. Retrieve the final commit hash:
   ```bash
   git rev-parse HEAD
   ```
5. Append the commit hash to the `[DONE]_<ticket_name>.md` ticket file.
6. Stage the updated ticket and amend the commit:
   ```bash
   git add tickets/[DONE]_<ticket_name>.md
   git commit --amend --no-edit
   ```
7. Before final push, fetch only (`git fetch --prune`) and compare the configured upstream SHA with the SHA recorded when this ticket was claimed. If it changed, stop and report; do not pull, merge, rebase, or force-push over it. If unchanged, push the single combined commit with the exact lease:
   ```bash
   git push --force-with-lease=<branch>:<recorded-upstream-sha>
   ```
   Verify the tree is clean and the branch is current after push. The next ticket starts only through the Repository Sync Gate.

---

## Terminology Preference

1. **Project root**: The active workspace directory (e.g., `whatdate-folder/`). All project work happens here.
2. **Tickets**: Markdown files in `tickets/` with status prefixes like `[APPROVED]`, `[READY_FOR_DEV]`, etc.
3. **rtest**: The regression test suite (project-specific command lives in `<project-folder>/SKILL.md`).
4. **Symphony Protocol**: The file-based multi-agent coordination system described in `agent-symphony/SKILL.md`.

---

## WhatDate Android Project Specifics

- **Primary active project**: `C:\Users\pooji\Documents\symphony\whatdate-folder/`
- `MEMORY.md` in the above folder is the source of truth for architecture decisions, core philosophy ("WhatDate is a memory..."), model invariants, and recent ticket status.
- On **every** Architect initialization, you **must** re-read the Fundamental Definition + Core Model Invariants sections from `MEMORY.md`.
- Real tickets use the prefix style `[APPROVED]_WD-078_brief_description.md`, `[READY_FOR_DEV]_...`, `[DONE]_...`.
- There is no dual-mode or trivial exception for the Architect. All app modifications require tickets.
- The meta folder `Symphony\` contains cross-project profiles, skills, and the universal `Agent role.md`. The canonical profile for each role is under `.agent_profiles\<role>_profile.md`.

---

## Symphony Context Resolution Order (All Agents)

To ensure synchronization across different agents, every init loads context in this order before acting:

1. `Agent role.md` (universal entry)
2. Role profile (`.agent_profiles/<role>_profile.md`)
3. `skills/global-skill/SKILL.md` (global rules and repository/live-state gates)
4. `skills/token-discipline/SKILL.md` (mandatory lossless input/output discipline)
5. `skills/agent-symphony/SKILL.md` (protocol)
6. Project `MEMORY.md` (live state; only after the repository/direct-remote gate)
7. Project `SKILL.md` (technical conventions)
8. Conditional role skills such as `ticket-management`, `release-launch`, `rtest`, and `blocker-resolution`
9. Active route, claim, ticket, and work state
10. Optional `skills/semantic-memory/SKILL.md` historical enrichment, only when the active work needs it and only after step 9

Mandatory governing files are read completely once during init. Token discipline governs targeted exploration afterwards and never permits partial loading of a required file, stale project state, or a skipped gate. Semantic memory is query-driven and optional: it locates possible historical sources after live context is known, never injects broad context during init, and never supplies current state or authority.

Always prefer the Symphony common versions for protocol consistency.

---

## Vendor Neutrality Rule (2026-07-05 — permanent)

The Symphony Protocol is vendor-independent by design: any agent (Claude, Gemini, Grok, GPT, Cursor, Codex, …) must be able to participate using ONLY the vendor-neutral files: `Agent role.md`, `skills/*`, `.agent_profiles/*`, and each project's `MEMORY.md` / `SKILL.md` / `ARCHITECTURE.md` / `AGENTS.md` / `tickets/`.

Hard rules:
1. **No protocol content may live exclusively in a vendor-specific file** (`CLAUDE.md`, `GEMINI.md`, `.cursorrules`, `.github/copilot-instructions.md`, etc.). Those files are convenience mirrors/pointers for tools that auto-load them — nothing more.
2. **Change order:** protocol/rule/lifecycle changes are written to the vendor-neutral files FIRST, then mirrored (Architect's responsibility). A mirror that is ahead of the canonical files is a defect — fix immediately.
3. **On conflict, vendor-neutral files win.** Every mirror must carry a notice saying so.
4. Each project root carries an `AGENTS.md` (the cross-vendor convention many tools auto-read) that points new agents at `Agent role.md` and lists the source-of-truth hierarchy.
5. Skills and profiles never move into vendor-specific directories (e.g., `.claude/`, `.gemini/`) — they stay in the shared `skills/` and `.agent_profiles/` trees.
