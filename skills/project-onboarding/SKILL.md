---
name: project-onboarding
description: The `add project <project-folder>` command — how a new project joins the Symphony. Registry row, path marker, MEMORY/SKILL/ticketorder, git, verification.
---

# Project Onboarding

Turns a folder that *contains* a project into one the Symphony can `init` into. One-time, whole-folder. Not a role, not a ticket.

**Command:** `add project <project-folder>` (also "onboard X to symphony"). **Reference:** an already-onboarded project shows the finished shape; this skill states the rules. Shape from the reference, rules from here.

## Authority

- **This skill is the only authorized creator of `.symphony-root`.** The marker's own "NEVER create it yourself" binds role agents mid-work, where a missing marker means you are lost. Onboarding is the one legitimate moment it is written.
- **"Never `mkdir` a project folder" has no exception, not even here.** Folder absent → STOP and tell the user.

## Five hard stops — all five before the first write

| Condition | Action |
|---|---|
| Folder does not exist | **STOP.** Never create it. |
| `.symphony-root` already present | Matching `project=` → already onboarded, report and stop. Different → radioactive, **STOP**. |
| Short name already in the Registry | **STOP** — `init` would be ambiguous. |
| Look-alike folder elsewhere | **STOP** and report. Don't read, merge, or delete. |
| Folder unreadable / purpose undeterminable | **STOP** and ask. |

Everything else — missing docs, no code, no git, odd ticket names — is a **finding to record, not a stop**.

## Defaults — derive, don't ask

State every derived value in one assumptions block in your report.

| Value | Default |
|---|---|
| Short name | Folder name minus a trailing `-folder`. Lowercase, no spaces. |
| Type + roles | Dev type → architect · qa · dev · srtl · orchestrator. Content type → composer · critic · designer · tester · implementer · srtl · orchestrator. A fresh project usually holds only a few Markdown docs and no build files, so **don't try to detect the type from contents** — take it from the workspace's standing convention. |
| Ticket prefix | Read off existing tickets if any (`P1-0` → `P1`); else an abbreviation of the short name. **Never renumber or re-prefix existing tickets.** |
| Remote / branch | The workspace's remote pattern + short name. `main` for a repo with no commits; keep the existing branch if it has history — never rename a branch with history. |
| Remote repo exists? | Never assume, never ask. Set it, attempt the push, hand the user the exact command if it fails. |
| Existing tickets | Bodies stay **exactly** as found. Onboarding seeds route lines and reports anomalies; editing/splitting/renumbering is Architect work, after init. |

## Procedure

**1. Survey, read-only.** Force-list the folder (dot-dirs hide). Identify which documents are *authoritative* (requirements/architecture/spec) vs incidental. Force-list `tickets/` (brackets break globs). Get all four of `git remote -v`, `git log --oneline -1`, `git status --short`, current branch — a repo with no commits behaves differently from one with history.

**2. Resolve identity,** then run the five hard stops.

**3. Create four files at the project root** — never inside a vendor dir (`.claude/`, `.cursor/`, …); the Symphony is vendor-neutral by design.

`.symphony-root` — verbatim from the reference project, only `project=` and `canonical_path=` change.

`MEMORY.md` — the live memory, deliberately short: (1) a line saying detail lives in tickets and canon; (2) **where truth lives** table; (3) **document precedence** — which wins, and that a real contradiction is an escalation, not a judgement call; (4) **core philosophy** — one-line identity + standing rulings, written from the authoritative docs in plain language; (5) **current state**, dated, with the ticket table and route; (6) **known gaps** — every anomaly the survey found; (7) **lessons** — empty at onboarding.

`SKILL.md` — technical conventions, with frontmatter: type + roles · authoritative docs ranked, with the precedence rule · locked technical baseline (what no agent may change without a user-approved ticket) · build commands — **if the build doesn't exist yet, say so and mark the table as the contract ticket 0 must satisfy**, plus an instruction to correct it once ticket 0 lands; a command table silently describing a non-existent build is a trap · the `rtest` contract (append-only, no agent weakens a gate, minimum phases, and how a phase this host can't run is reported — `HOST_SKIPPED`, never PASS) · key source locations + placement rules · architecture principles, short, pointing at canon · git workflow incl. what to do when push fails · artifact rule, stating which platforms were actually verified · environment notes · references.

`ticketorder.md` — line 1 is the format legend, forever. Line 2+ is the **living batch note**: this batch's intent, the order rationale, deliberate gates, what waits on a human. Then route lines, `<ticket>-<Role>`, in dispatch order. Seed from existing tickets *respecting status*: a `[READY_FOR_DEV]` ticket gets only a `-Dev` line; an `[APPROVED]` one gets `-QA` then `-Dev`. A route line for a role that can't start yet is correct — that role will classify WAIT, which is the head rule working.

**4. Support dirs.** `tickets/` and `tickets/.claims/`. Git doesn't track empty dirs — put a short README in `.claims/` (what a claim marker is; an orphaned claim means resume, never route around). **Don't create `requests/`**: the Architect's request-intake path reads it, but an absent directory just means no requests — create it when the project actually starts using intake, not before.

**5. Ticket 0 check.** The bootstrap ticket must establish the project's full target structure in the first commit (see the Architect profile's commit-0 obligation). Already correct → record it. Wrong or silent → **don't edit it**; record in `MEMORY.md` known gaps and flag it as the Architect's first job. Absent → note that ticket 0 must establish it before any product ticket is designed.

**6. Register** in `Agent role.md`: a Project Registry row, and the project added to its lifecycle list. Until this lands, `init` cannot resolve — it's the step that makes the project real.

**7. Git** (skip silently if no `.git`). Set identity → point `HEAD` at `main` if there are no commits → one commit → add the remote → attempt the push. **A push failing on credentials or a missing remote repo is expected, not something to work around**: never force, never switch remotes, never invent a workaround. Report the state and hand over the exact command.

**8. Verify from disk.** Registry row resolves · `.symphony-root` matches · four files + both dirs exist · route lines reference tickets that exist · no duplicate ticket numbers (`ls tickets | grep -oE "<PREFIX>-[0-9]+" | sort | uniq -d` → empty) · no look-alike folder · git clean and the push either done or reported pending.

**9. Report:** registry row · files created · git state incl. any pending push command · every defaulted assumption · every known gap · the `init <short-name> <role>` command that now works.

## Findings, not fixes

Onboarding **records** problems; it never repairs them. Into `MEMORY.md` known gaps and into the report:

- **A ticket referencing a file that doesn't exist** (a mockup archive, a spec version, a renamed module) — exactly what the supersession check catches. Whoever executes that ticket must stop, not adapt a stale plan.
- **A bootstrap ticket missing a target platform** — the expensive one; cost multiplies with every ticket built on it.
- **A ticket bundling an entire product** into one unit — legitimate as a first baseline, but the Architect should decide that deliberately rather than inherit it.
- **A host that can't certify a target** (no macOS for iOS, no device for hardware paths) → a `HOST_SKIPPED` obligation in `SKILL.md`, and `MEMORY.md` stating plainly that skipped is not passed.
- **Temporary identifiers** (package/bundle IDs) that must be frozen before release.

## Environment traps

Bracketed filenames break globs — force-list, rename with `Move-Item -LiteralPath` / `cmd /c ren`. Dot-dirs need `-Force`. Empty dirs vanish through git — seed a README. **Restricted mounts:** some sandboxes allow create/write but not delete, so git leaves `.git/index.lock` and `tmp_obj_*` behind and the *next* git command dies with "Another git process seems to be running" — clear them before continuing, and if you can't, say so loudly: it blocks every future git operation on that machine.

## After

Onboarding does not begin work. It ends by handing over `init <short-name> <role>`.
