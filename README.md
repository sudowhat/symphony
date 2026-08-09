# Symphony Protocol

**A file-based protocol for running a team of AI coding agents without burning tokens on coordination.**

No orchestration framework. No message bus. No vendor SDK. Agents coordinate by **renaming Markdown files** in a git repo — which means the entire state of your project is `ls`-able, greppable, diffable, and survives every agent crashing at once.

Works with any model that can read files and run a shell: Claude, GPT, Gemini, Grok, Cursor, Codex, or a mix of them at the same time on the same repo.

---

## The idea in one screen

Every agent gets one role and one entry point:

```
init project1 dev
```

It then reads its own boundaries from disk, finds the next job on a plain-text queue, does it, renames a file, and stops. That's the whole protocol.

```
tickets/[APPROVED]_P1-001_search_matches_plurals.md      <- QA's turn
tickets/[READY_FOR_DEV]_P1-002_past_days_muted.md        <- Dev's turn
tickets/[DONE]_P1-003_list_opens_at_top.md               <- finished
```

The filename prefix **is** the state machine. A ticket moves `[APPROVED]` → `[READY_FOR_DEV]` → `[IN_PROGRESS]` → `[DONE]` by being renamed. Nothing else needs to be true.

---

## Why file-based instead of a framework

| | Framework orchestration | Symphony |
|---|---|---|
| Agent crashes mid-task | State lost or stuck | Rename half-done; next agent resumes from disk |
| Two vendors on one project | Usually impossible | Normal — they read the same files |
| "What is the system doing?" | Read logs, query state | `ls tickets/` |
| Cost of coordination | Tokens per message, per hop | Zero. Nobody talks to anybody |
| Reviewing what an agent did | Trust the transcript | `git diff` |
| Onboarding a new model | Port the SDK | Point it at one Markdown file |

**Token efficiency is the design goal, not a side effect.** Agents never pass context to each other — the ticket *is* the context. A specialist receives nothing in-band, ever. It reads the file, does the work, and stops. An agent does not need to know the whole architect to implement a PR or a CR as with solution approach clearly outlined by the architect in the ticket itself.

---

## The work loop

Every non-Architect role runs exactly this:

```
loop: while (true) {
    0. sync:    clean tree + fetch + fast-forward update? -> otherwise report user ; STOP
    1. resume:  any half-finished ticket of MY role?  -> take it
    2. read:    <project>/ticketorder.md
    3. exit:    no open line for MY role anywhere?    -> print "<role>|exit" ; STOP
    4. take:    top open line is mine AND gate open?  -> do it ; mark it ; continue (no sleep)
    5. wait:    print "<role>|waiting on <ticket>" ; sleep 300 ; continue    <- or a silent watcher; see mantra 3
}
```

Three mantras:

1. **Never exit while I still have an open line in the batch.** Blocked is not finished.
2. **Exit the moment nothing on the list is mine.** Don't linger, don't "check in case".
3. **Sleep must resume *this* session with its context intact.** Not a cloud job, not a new agent, not a fresh `init`, not a poll storm. A 300-second blocking sleep qualifies; so does a background watcher that stays silent until there's work.

For a Git project, the sync step runs before every real work entry: init, handoff re-entry, post-wake poll, and bare continuation. It first checks for **any** tracked or untracked worktree change. A dirty tree is reported to the user and stops the agent; it is never stashed, reset, cleaned, or pulled over. A clean tree updates only by fetch plus fast-forward pull. The canonical detail is in `skills/global-skill/SKILL.md`.

A loop that burns tokens waiting is a broken loop even if it breaks no rule.

**You pay per model turn, not per poll.** A sleep *inside* the agent still costs a full turn every tick (~$0.06 at 80K context) — the model has to reload everything just to learn nothing changed — while the same wait handed to a silent background shell loop costs nothing at all, and still honours mantra 3: it wakes *this* session, context intact.

`ticketorder.md` is the queue — one line per unit of work, top-to-bottom:

```
P1-003-QA:DONE
P1-003-Dev:DONE:REVIEWED
P1-001-QA               <- top open line
P1-001-Dev
P1-002-Dev
```

---

## The two lifecycles

**Dev** (`project1`) — four roles, TDD-driven:

| Role | Writes | Never writes |
|---|---|---|
| **Architect** | Tickets, design docs | Production code, ever |
| **QA** | Failing tests | Production code |
| **Dev** | Production code | Tests |
| **SRTL** | Both code *and* tests | Architecture/design docs |

SRTL (Senior Tech Lead) is the release valve. It is the only role with dual authority, and it exists for the two situations that otherwise deadlock: reviewing finished work, and unblocking an agent that hit a wall.

**Content** (`project2`) — five roles for writing and publishing: Composer → Critic → Designer → Tester → Implementer. Status is a filename *suffix* here (`_APPROVED` → `_VERIFIED` → `_RFT` → `_FIXED`), because bracket prefixes are reserved for article states.

---

## The rules that came from real damage

Most of this protocol is scar tissue. Each rule below exists because something broke:

- **Scope lock.** You may touch only the files your ticket names. Before committing: *"can I name the ticket line that authorises each file in `git status`?"* If not, stop. Born from an agent that improvised across 30 files.
- **Behaviour locks.** Every fixed bug gets a test named `lock_<ticket>_<behaviour>` that no agent may delete, weaken or skip. Fixed bugs kept coming back because UI work shipped with zero automated protection.
- **Guards are not obstacles.** No agent may raise a limit or delete an assertion to make its own work pass. Born from an agent that widened a length cap from 28 to 30 so its output would fit.
- **Supersession check.** Before acting on a ticket, verify its factual premises against the *current* repo. A stale plan executed confidently caused more damage than any bug.
- **Path integrity.** One canonical path per project, verified by a `.symphony-root` marker before the first write. An agent once recreated a project in a parallel folder and worked there for days.
- **Self-fix vs escalate.** A five-part test for when an agent may cross the test/code boundary itself (a stale constant its own change made stale) versus when it must stop and escalate (anything requiring a judgement call).

The ticket IDs in the shipped skill files (`WD-…`, `CAP-…`) are real incidents from the project this was built on. They're kept deliberately — the war story is what makes the rule stick.

---

## Repository layout

```
.
├── Agent role.md                  # THE entry point. Every agent reads this first.
├── taskagent.md                   # orchestrator only - see "Status" below, not yet usable
├── orchestrator model map.md      # orchestrator only - model slug legend
├── .agent_profiles/               # one file per role: identity, boundaries, workflow
│   ├── architect_profile.md   qa_profile.md   dev_profile.md   srtl_profile.md
│   ├── orchestrator_profile.md
│   └── composer_ critic_ designer_ tester_ implementer_profile.md
├── skills/                        # shared behaviour, loaded by every agent
│   ├── agent-symphony/            #   the core protocol: lifecycle, boundaries, hard rules
│   ├── global-skill/              #   git workflow, clarification policy, no-keepalive rule
│   ├── ticket-management/         #   ticket naming and templates
│   ├── rtest/                     #   test tiers and TDD conventions
│   ├── blocker-resolution/        #   self-fix vs escalate triage
│   └── stateless-protocol/        #   never rely on chat history
├── project1/                      # SAMPLE dev project
│   ├── .symphony-root  AGENTS.md  MEMORY.md  SKILL.md
│   ├── ticketorder.md  REGRESSION_CHECKLIST.md
│   ├── requests/                  #   raw human requests, pre-ticket
│   └── tickets/                   #   the agent-to-agent API
└── project2/                      # SAMPLE content project
    ├── .symphony-root  AGENTS.md  MEMORY.md  SKILL.md  ticketorder.md
    ├── capsules/                  #   article content
    └── tickets/
```

---

## Getting started

1. **Clone this repo.** It becomes your `<SYMPHONY_ROOT>` — the paths inside the protocol files are written as `<SYMPHONY_ROOT>`; replace them with your absolute path, or keep the placeholder if your agent resolves relative paths reliably.
2. **Add your project** — say `add project <your-folder>` to any agent and it writes the files, registers the project and verifies the result (see [Adding your own project](#adding-your-own-project) below). Prefer to do it by hand? Copy `project1/` (dev) or `project2/` (content), rename it, and do steps 3–4 yourself.
3. **Add a row to the Project Registry** in `Agent role.md`.
4. **Edit three files in your new folder:**
   - `.symphony-root` — set `project=` and `canonical_path=`
   - `SKILL.md` — your real build and test commands. Nothing else tells the agents how to build.
   - `MEMORY.md` — your philosophy in one paragraph. Keep it short; it is not a changelog.
5. **Open an agent and type:** `init <yourproject> architect`

The Architect turns your request into tickets. Then open a second agent: `init <yourproject> qa`, and a third: `init <yourproject> dev`. They will not talk to each other, and that is the point.

---

## Adding your own project

Hand-copying a sample folder works, but the repo ships a skill that does it properly. Point any agent at a folder that **already exists** and say:

```
add project <project-folder>
```

The agent reads `skills/project-onboarding/SKILL.md` and runs a fixed procedure: survey read-only, clear five hard stops, derive every default, write the files, register the project, commit, verify from disk, and hand back the `init` command that now works.

**It refuses rather than improvises.** Five conditions stop it cold: the folder doesn't exist, a `.symphony-root` is already there, the short name is taken, a look-alike folder exists elsewhere, or the folder's purpose is undeterminable. Everything else — missing docs, no git, odd ticket names — is recorded as a finding in `MEMORY.md`, never silently repaired.

**It never creates the project directory.** If the folder isn't there, it stops and tells you. No `mkdir`, no clone, no reconstructing a project from git objects or memory — that rule has no exception, not even here.

**It derives; it doesn't interrogate you.** Short name from the folder (`foo-folder` → `foo`), role set from the project type, ticket prefix from whatever tickets already exist, remote and branch from convention. Every default it chose is listed in one assumptions block in the final report, so you can correct it in one pass instead of answering a questionnaire up front.

**What lands:** `.symphony-root`, `MEMORY.md`, `SKILL.md`, `ticketorder.md`, plus `tickets/` and `tickets/.claims/`. Two things about that set are worth knowing:

- `.symphony-root` is the path-integrity marker, and **onboarding is the only moment it is ever legitimate to create one.** Every other rule in the protocol says a role agent that finds it missing is lost and must stop.
- The registry row appended to `Agent role.md` is **the step that actually makes the project real.** Until that edit lands, `init <short-name> <role>` cannot resolve, no matter how complete the folder looks.

Onboarding does not start work. It ends by handing you `init <short-name> <role>`, and the Architect's own init takes it from there.

---

## Status: the Orchestrator is not there yet

There is an `orchestrator` role in this repo, and the design is sound: a deliberately dumb dispatcher that reads the queue, spawns the right specialist on the right model, and never reasons about the work itself. It follows the Architect's route and Symphony's status gates; it writes no tickets, no code, no tests.

**It is not usable today, and the blocker is not the design — it's vendor capability.**

An orchestrator only pays for itself if a *cheap* model can spawn an *expensive* one on demand — a small dispatcher waking a large coder for one ticket, then going back to sleep. That requires spawning a model of arbitrary level **without a human approving each spawn**. No vendor exposes that cleanly right now: sessions are interactive, spawning is permission-gated, or the sub-agent can't be pinned to a chosen model. An orchestrator that stops every few minutes for a permission click is worse than no orchestrator.

**So the loop is the answer for now, and it is a good one.** You open one agent per role, each runs the five-line loop above, and they self-serialise through `ticketorder.md` — waiting when blocked, exiting the moment nothing is theirs. No dispatcher, no spawn permissions, no second model billing itself to supervise the first; and if you hand the waiting to a watcher, the idle time costs nothing either. In practice this gets you most of the value at a fraction of the complexity.

When vendors do ship unattended, model-selectable spawning, the orchestrator files here are ready and the rest of the protocol does not change — the route file and the status gates are already the contract it would use. Until then, treat `orchestrator_profile.md`, `taskagent.md` and `orchestrator model map.md` as **a design ready for a capability that doesn't exist yet**, not as working parts.

## Things worth knowing before you adopt it

- **This is a protocol, not software.** There is nothing to install and nothing to run. Its entire enforcement mechanism is that agents read the rules and follow them. Capable models follow it well; weaker ones drift and need shorter files.
- **It assumes one agent per role per project at a time.** That guarantee is what makes append-only `:DONE` writes and crash recovery safe without locking.
- **The shared-worktree model is deliberate.** All agents work in one checkout, which is why the pipeline is sequential rather than parallel. Cross-*project* parallelism is fine.
- **Windows-flavoured in places.** The origin project was Windows/PowerShell, so some examples use PowerShell and `Move-Item -LiteralPath`. The protocol itself is OS-agnostic; the commands in `SKILL.md` are yours to replace.
- **Files are long.** Some skill files run to hundreds of lines. That is a real cost at load time and the main thing to trim for your own use. The five-line loop above is the part that matters most.

---

## Contributing

The most useful contributions are **rules that came from something breaking**, written the same way as the ones here: the incident first, then the rule. A rule with a war story attached gets followed. A rule without one gets rationalised away at 2am by an agent in a hurry.

## License

MIT — see [LICENSE](LICENSE).
