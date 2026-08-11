---
name: project-onboarding
description: Add a new project to the Symphony Protocol. Implements the `add project <project-folder>` command — registry row, path-integrity marker, MEMORY.md / SKILL.md / ticketorder.md, support directories, git, and verification. Vendor-neutral: any agent of any brand can execute it.
---

# Project Onboarding Skill

Onboarding turns a folder that merely *contains* a project into a folder the Symphony can `init` into. It is a one-time, whole-folder operation — not a role, not a ticket, and never part of normal ticket work.

**Command:** `add project <project-folder>` (case-insensitive). Also triggered by "onboard X to symphony", "add this project to the symphony protocol", or equivalent.

**Canonical reference:** `whatdate-folder/` is the reference implementation. Open its `.symphony-root`, `MEMORY.md`, `SKILL.md` and `ticketorder.md` and mirror their shape. This skill states the rules; whatdate shows the finished article. When the two disagree, whatdate's *shape* wins and this skill's *rules* win.

---

## 1. Authority and its limits

This skill is the **single authorized exception** to one hard rule, and no exception at all to another:

- **`.symphony-root` says "NEVER create it yourself."** That rule binds role agents during ticket work, where a missing marker means you are lost. Onboarding is the one legitimate moment the marker is created. Create exactly one, for a folder that already exists.
- **"Never create project directories" stands absolutely.** If the target folder does not exist, **STOP** and tell the user. Do not `mkdir`, do not clone, do not "restore" a project from git objects or memory. A missing folder is never a licence to make one.

## 2. Five hard stops — check all five before the first write

| # | Condition | Action |
|---|---|---|
| 1 | Target folder does not exist under the Symphony root | **STOP.** Never create it. |
| 2 | `.symphony-root` already present | Read it. `project=` matches → already onboarded; report and stop. `project=` differs → radioactive; **STOP** and report. |
| 3 | Short name already in the Project Registry | **STOP.** Two projects cannot share a short name — `init` would be ambiguous. |
| 4 | A look-alike folder exists elsewhere (similar name or contents) | **STOP** and report. Do not read further, do not merge, do not delete. This is the June-2026 fork incident's prevention rule. |
| 5 | Folder unreadable, or its purpose genuinely undeterminable | **STOP** and ask. |

Anything else — missing docs, no code, no git, odd ticket names — is a finding to record, not a stop.

## 3. Defaults — derive, do not ask

State every derived value in one assumptions block in your final report. Ask nothing unless a hard stop fires.

| Value | Default |
|---|---|
| Short name | Folder name minus a trailing `-folder` (`dbmeter-folder` → `dbmeter`). Lowercase, no spaces. |
| Project type | **`kmp-mobile` — always, for every new project.** See §4. |
| Role set | From the type: `kmp-mobile` / `android-dev` → architect · qa · dev · srtl · orchestrator. `content-web` → composer · critic · designer · tester · implementer · srtl · orchestrator. |
| Ticket prefix | Read it off existing tickets if any exist (`DBM-0` → `DBM`). Otherwise an uppercase abbreviation of the short name. Never renumber or re-prefix tickets that already exist. |
| Git remote | `https://github.com/sudowhat/<short-name>.git` |
| Branch | `main` for a repo with no commits. If commits already exist, keep whatever branch they are on — never rename a branch that has history. |
| Repo exists on GitHub? | Never assume, never ask. Set the remote, attempt the push, and if it fails hand the user the exact push command. |
| Existing tickets | Leave the bodies **exactly** as found. Onboarding seeds route lines and reports anomalies; it never edits, renumbers or splits a ticket. That is Architect work, after init. |

## 4. The cross-platform mandate (standing user ruling, 2026-08-05)

> **Every new project is multi-platform from commit 0.** Kotlin Multiplatform + Compose Multiplatform, Android **and** iOS targets present in the first commit, shared business logic, platform code confined to thin adapters.

The reason is economic: retrofitting iOS onto a finished Android app costs a rewrite; carrying an empty iOS target from day one costs almost nothing. So the type default is `kmp-mobile` and there is nothing to detect — a fresh project usually holds only a few Markdown documents and maybe a ticket or two, with no build files to inspect.

**Enforce it at ticket 0.** During onboarding, look at the project's bootstrap ticket (the lowest-numbered one, typically `<PREFIX>-0`):

- If it exists and already mandates KMP + both targets + a shared source set from the first commit — record that it does, and move on.
- If it exists but is Android-only, or silent on iOS — **do not edit it.** Record it in `MEMORY.md` under known gaps and flag it in your report as the first thing the Architect must fix.
- If no bootstrap ticket exists yet — note in `MEMORY.md` and in your report that ticket 0 must establish the multiplatform structure before any product ticket is designed.

A single-platform bootstrap is the one defect that gets exponentially more expensive with every subsequent ticket. Never let it pass unremarked.

## 5. The procedure

### Step 1 — Survey, read-only

Force-list the folder (dot-directories hide from ordinary listing tools: `Get-ChildItem -Force`). Establish, without writing anything:

- what documents exist, and which are **authoritative** (requirements / architecture / spec) versus incidental;
- whether `tickets/` exists and what is in it (force-list — bracketed names break globs);
- git state: `git remote -v`, `git log --oneline -1`, `git status --short`, current branch. All four; a repo with no commits behaves differently from one with history.

### Step 2 — Resolve identity

Fix the short name, folder, type, role set, ticket prefix, remote and branch per §3. Run the five hard stops from §2 now, before any write.

### Step 3 — Create the four files

Mirror `whatdate-folder/`. All four live at the **project root** — never inside a vendor directory (`.claude/`, `.cursor/`, `.gemini/`, …), because any agent of any brand must be able to read them.

**`.symphony-root`** — verbatim, only the first two lines change:

```
project=<short-name>
canonical_path=C:\Users\pooji\Documents\symphony\<project-folder>\
# SYMPHONY PATH-INTEGRITY MARKER — DO NOT COPY, MOVE, OR RECREATE THIS FILE.
# Agents: verify this file exists at your Active Workspace and that
# project= matches your init command BEFORE your first write of a session.
# If it is missing, STOP and report to the user. NEVER create it yourself,
# NEVER create a project directory, NEVER work in a look-alike folder.
```

**`MEMORY.md`** — the project's live memory. Deliberately short; detail belongs in tickets and canon. Required sections:

1. A one-line statement that the file is deliberately short and where detail actually lives.
2. **Where truth lives** — a table mapping questions to files (spec, architecture, tickets, build, route, protocol).
3. **Document precedence** — which document wins when two disagree, and that a real contradiction is an escalation rather than a judgement call.
4. **Core philosophy** — the project's one-sentence identity, then standing rulings as bullets. Write these from the authoritative documents, in plain language. This is what every agent re-reads on every init.
5. **Current state** — dated. Onboarding facts, the ticket table with statuses, the route.
6. **Known gaps** — every anomaly the survey found (see §6). Nothing swept under the rug.
7. **Lessons worth keeping** — empty at onboarding; incidents get compressed into it later.

Do not create a routine ticket-history archive inside `MEMORY.md`. Completed detail remains canonical in tickets/docs/Git. An optional semantic-memory provider may later index approved history, but the project must initialize and operate completely without it.

**`SKILL.md`** — project-specific technical conventions, with YAML frontmatter (`name: <short-name>-skill`, one-line `description`). Required sections:

1. Project type and role set.
2. **Authoritative documents**, ranked, with the precedence rule restated.
3. **Locked technical baseline** — the decisions no agent may deviate from without a user-approved ticket.
4. **Build system and commands.** If the build does not exist yet, say so plainly and mark the table as the contract ticket 0 must satisfy, plus an instruction to correct it once ticket 0 lands. A command table that silently describes a non-existent build is a trap.
5. **The `rtest` contract** — the single regression entry point: append-only, no agent may weaken a gate, what its minimum phases are, and how it reports a phase this host cannot run (`HOST_SKIPPED`, which is never PASS).
6. **Key source locations** — a path table, plus the placement rules (what belongs in shared code versus platform adapters).
7. **Architecture principles**, short form, pointing at canon.
8. **Git workflow** — repo, remote, branch, commit-message convention, and what to do when a push fails.
9. **Artifact build rule** — version bump, release notes, where the artifact lands, and an explicit statement of which platforms were actually verified.
10. **Environment notes** — OS, dot-directory listing, bracketed-filename renames (`Move-Item -LiteralPath` / `cmd /c ren`, never plain `Rename-Item`), full literal paths.
11. **References.**

**`ticketorder.md`** — the dispatch route:

- **Line 1** is the format legend, and stays a legend forever.
- **Line 2 onward (before the route lines)** is the living batch note: this batch's intent, why the order is what it is, which gates are deliberate, what is waiting on a human. Rewrite it on every batch change — a later reader must not have to reconstruct intent.
- Then one route line per unit of work, `<ticket>-<Role>`, top-to-bottom in dispatch order. Absence of `:DONE` is the only "open" signal.
- Seed lines from the tickets that already exist, respecting their status: a ticket sitting at `[READY_FOR_DEV]` gets only a `-Dev` line; an `[APPROVED]` one gets `-QA` then `-Dev`.
- A route line for a role that cannot start yet is correct, not broken. Roles polling it will classify WAIT — that is the head rule working.

### Step 4 — Support directories

Create `tickets/` (if absent) and `tickets/.claims/`. Git does not track empty directories, so put a short `README.md` in `.claims/` explaining what a claim marker is and that an orphaned claim means resume, never route around.

**Do not create `requests/`.** The Architect's request-intake path (`Agent role.md`, Architect step 4) reads `<project>/requests/` for `[NEW]_REQ-*.md`, but an absent directory simply means no requests — it is not an error, and every live project runs without one. Create it only when the project actually starts using Orchestrator intake.

### Step 5 — Ticket 0 check

Apply §4. Record the outcome; never edit the ticket.

### Step 6 — Register in `Agent role.md`

Two edits to `C:\Users\pooji\Documents\symphony\Agent role.md`:

1. Append a row to the **Project Registry** table: `| <short-name> | <project-folder> | <type> | <roles> |`.
2. Add the project to its **lifecycle** list under "Two Main Lifecycles". If the type is new to that lifecycle, add one bracketed sentence saying what differs — for `kmp-mobile`, that "builds and passes" means both platforms, and that a non-macOS host reports the iOS phase as `HOST_SKIPPED`, which is never PASS.

Until this edit lands, `init <short-name> <role>` cannot resolve. It is the step that actually makes the project real.

### Step 7 — Git

Skip this entire step, silently, if the folder has no `.git` — the file protocol works without git.

1. Ensure `user.name` / `user.email` are set on the repo (match the other projects).
2. Fresh repo → point `HEAD` at `main` before the first commit. Repo with history → leave the branch alone.
3. Stage everything and make one commit: `Onboard <name> to Symphony Protocol + <what else is in this commit>`.
4. Add the remote per §3.
5. Attempt the push. **If it fails on credentials or a missing remote repo, that is expected, not an error to work around.** Do not force, do not invent a workaround, do not switch remotes. Report the state and hand the user the exact command.

### Step 8 — Verify from disk

Re-read everything you wrote, freshly, and confirm:

- the registry row is present and the short name resolves to the right folder;
- `.symphony-root` exists and its `project=` matches;
- all four files plus both support directories exist at the project root;
- route lines parse and reference tickets that actually exist on disk;
- no duplicate ticket numbers (`ls tickets | grep -oE "<PREFIX>-[0-9]+" | sort | uniq -d` → empty);
- no look-alike folder anywhere under the Symphony root;
- git is clean, the commit exists, and the push either succeeded or is reported as pending.

### Step 9 — Report

Give the user: the registry row, the files created, the git state (including the push command if pending), every assumption you defaulted, every known gap recorded, and the exact `init <short-name> <role>` command that now works.

---

## 6. Findings, not fixes

Onboarding **records** problems; it does not repair them. Anything you find goes into `MEMORY.md` under known gaps and into your report — never into a silent edit.

The ones that recur:

- **A ticket references a file that does not exist** (a mockup zip, a spec version, a renamed module). This is exactly what the Ticket Integrity Rules' supersession check catches. Record it. The agent that later executes that ticket must stop rather than adapt a stale plan on the fly.
- **A bootstrap ticket that is single-platform.** See §4 — the expensive one.
- **A ticket that bundles an entire product** into one unit of work. Legitimate as a first baseline ticket; note it so the Architect makes a deliberate decision rather than inheriting it by accident.
- **This host cannot certify one target** (no macOS for iOS, no device for hardware paths). Record it as a `HOST_SKIPPED` obligation in `SKILL.md`, and state plainly in `MEMORY.md` that skipped is not passed.
- **Temporary identifiers** (package name, bundle ID, applicationId) that must be frozen before a store release.

## 7. Environment traps

- **Bracketed filenames** (`[APPROVED]_…`) break shell globs. Force-list to see them; rename only with `Move-Item -LiteralPath` or `cmd /c ren`.
- **Dot-directories hide** from ordinary listing tools. `Get-ChildItem -Force`, always.
- **Empty directories vanish through git.** Always seed a README.
- **Restricted mounts.** Some sandboxes permit create-and-write but not delete. Git then leaves `.git/index.lock` and `tmp_obj_*` files behind, and the *next* git command fails with "Another git process seems to be running". Remove the leftovers — request delete permission from the host if the sandbox refuses — before continuing, and never leave a lock file in the user's repo. If you cannot remove it, say so loudly: it blocks every future git operation on that machine.
- **Never write into a vendor directory.** `.claude/`, `.cursor/`, `.gemini/` and friends are invisible to other vendors' agents, and the Symphony is vendor-neutral by design.
- **Full literal paths only.** The Symphony root is `C:\Users\pooji\Documents\symphony\`.

## 8. After onboarding

Onboarding does not begin work. It ends by handing the user `init <short-name> <role>` — and the Architect's own init sequence takes it from there.

## References

- `Agent role.md` — the `add project` command, the Project Registry, the Path Integrity Protocol, the Ticket Integrity Rules
- `whatdate-folder/` — the reference implementation of all four files
- `skills/ticket-management/SKILL.md` — ticket naming and templates
- `skills/global-skill/SKILL.md` — global rules, repository gates, and live-state freshness
- `skills/token-discipline/SKILL.md` — mandatory targeted retrieval, terse output, and lossless durable-artifact rules for every role
- `skills/semantic-memory/SKILL.md` — optional provider-neutral historical locator; never authoritative or required
- `skills/agent-symphony/SKILL.md` — lifecycle, boundaries, and execution protocol
- `.agent_profiles/architect_profile.md` — §"Standing Expectation: Multi-Platform From Commit 0"
