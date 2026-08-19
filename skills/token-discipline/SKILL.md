---
name: token-discipline
description: Vendor-neutral token-efficiency rules for every Symphony agent. Reduce retrieved context and conversational overhead while preserving complete engineering artifacts, required evidence, live-state freshness, gates, and role boundaries.
---

# Symphony Token Discipline

This is a mandatory, Symphony-native, vendor-independent skill for every current and future role.

No Symphony workflow may depend on an external optimizer, vendor hook, vendor configuration file, IDE rule, plugin, or installation. Vendor-specific helpers may be optional accelerators only; the vendor-neutral Symphony files remain canonical.

## Prime directive

> **Small mouth, full brain. Compress chatter; preserve engineering signal.**

Token discipline reduces unnecessary retrieval and observable verbosity. It never reduces the reasoning, exact evidence, or durable detail needed to execute, review, reproduce, secure, or maintain the work.

## Non-negotiable safety floor

Token optimization NEVER overrides:

1. user instructions;
2. correctness, security, privacy, data integrity, or architecture fidelity;
3. the Repository Sync Gate, Direct-Remote Gate, optimistic concurrency, or live-state freshness;
4. Symphony role boundaries, ticket lifecycle, scope locks, or exact Role Work Loop behavior;
5. ticket, design, and durable-artifact completeness;
6. required code, tests, commands, exact errors, diffs, logs, assertions, reproduction steps, and execution/review evidence.

If saving tokens would weaken any item above, spend the tokens.

## Relationship to existing Symphony rules

This skill complements existing rules; it does not restate or supersede their semantics:

- `global-skill` Cost-Aware Execution owns ask-versus-assume decisions.
- `global-skill` and `Agent role.md` No Keepalive / No Tool Spam own WAIT and EXIT behavior.
- `Agent role.md` owns the exact one-line Role Work Loop status strings.
- `global-skill` owns Repository Sync, Direct-Remote, and live-state reread gates.
- `agent-symphony`, role profiles, and active tickets own boundaries, lifecycle, scope, and completeness.

When this skill and a governing rule appear to differ, follow the governing rule and report the documentation conflict.

## Compress aggressively

Keep these terse and information-dense:

- progress and routine status messages;
- tool-call narration;
- readiness, WAIT, EXIT, and handoff output;
- routine success commentary;
- explanations already evident from an exact diff or test result;
- repeated restatement of ticket requirements or persisted decisions;
- repeated summaries of files already available to the next role.

Prefer identifiers, paths, commands, facts, and short fragments over prose when unambiguous.

Bad:

`I completed my detailed review and found that the resolver appears to use today's date rather than the date selected by the user.`

Good:

`Bug: resolver uses today, not selected date.`

## Preserve losslessly

Do not compress away or loosely paraphrase information whose exact form matters:

- source code and patches;
- required commands, paths, filenames, symbols, APIs, schemas, fields, IDs, and status prefixes;
- diagnostically relevant errors, failing assertions, and test evidence;
- meaningful diffs and review evidence;
- acceptance criteria, architectural decisions, constraints, and authoritative user wording;
- security/privacy requirements;
- reproduction steps where omission changes behavior;
- anything another role needs to execute without guessing.

Trim irrelevant noise, not signal. Prefer a targeted excerpt or bounded tail when it contains all required evidence.

## Durable artifact rule

Tickets, project `MEMORY.md`/`SKILL.md`, architecture documents, code, tests, and exact evidence are durable engineering state—not conversational chatter.

1. Write the complete required information once to the canonical artifact.
2. Afterwards, reference its path/section/status instead of restating it in chat or another handoff.
3. Do not duplicate the same explanation across files unless the protocol requires it.
4. Never shorten a durable artifact merely to reduce tokens when that would remove execution, review, or maintenance value.

Lossless artifacts are the compression boundary: terse conversation, complete engineering state.

## Read discipline — reduce input tokens too

Before reading, name the fact needed for the current decision.

1. Fully read every file that the init sequence, role profile, active ticket, or governing skill explicitly requires in full. Ranged reads are not a substitute for completing a mandatory context load.
2. After mandatory context is loaded, inspect the ticket/architecture for named paths, symbols, and constraints before exploring the repository.
3. Prefer targeted search (`rg`/`grep`/indexed search/`find`) before directory surveys or opening many files.
4. Read the smallest relevant range when the host supports ranged reads; expand only when surrounding context is required.
5. Reuse stable information during the same unit of work. Keep the path/revision and resolved fact; do not reread merely for reassurance.
6. Re-read volatile state whenever `global-skill`, Repository Sync, Direct-Remote, a real WAIT wake, elapsed time, or possible external movement requires freshness. Stable-read reuse never overrides a live-state gate.
7. Do not recursively dump directories, complete logs, generated trees, lockfiles, binaries, or unrelated source for orientation.
8. Reuse canonical persisted decisions instead of reconstructing them from conversation.

## Tool discipline

Every tool call must answer a concrete question, verify a required condition, or perform an authorized action.

Avoid:

- no-op or heartbeat calls;
- repeated status/list calls without a state-change reason;
- huge raw output when a filtered query is sufficient;
- reading the same stable file through multiple tools;
- broad repository scans before ticket-directed search;
- rerunning successful checks without a freshness, mutation, or gate reason.

Never suppress or narrow a mandatory Symphony gate to save tokens.

## Optional semantic-memory retrieval

A semantic-memory provider may locate relevant historical engineering artifacts. It is an **optional advisory index**, never a source of truth or dependency. Symphony behaves identically when no provider exists, the host/vendor cannot access it, or a request fails.

Binding regardless of whether you load the detail:

1. Complete canonical init and understand active/live work before any semantic query.
2. Query only when history would materially help; use one narrow same-project query, 3 results by default and 5 maximum across expansions/scopes.
3. Require repository/path provenance, exact-read the current canonical source, and resolve revision/supersession from live canon before acting.
4. Never source route, claim, ticket status, branch/ref, current source, test state, or another volatile fact from semantic memory.
5. Never inject a broad memory/profile at init or auto-store conversation/tool chatter.
6. Fail open to normal repository search. Provider absence, timeout, malformed/conflicting output, or missing provenance must not block role execution.
7. Label cross-project results as analogies and preserve repository/privacy boundaries.

Operational policy, provider contract, ingestion allowlist, role boundaries, verification, and the first reference adapter:
`skills/semantic-memory/SKILL.md`. Load it on demand only after active context is known.

## Optional code intelligence

A code-intelligence provider (currently jCodeMunch) may answer **current-source structural questions** and retrieve narrow symbol-level code after mandatory init and active-work discovery. It is an optional navigation/index accelerator, never live Git/gate truth or a required dependency.

Use surface ownership:

```text
exact / canonical / gate evidence        -> RAW / native exact path
historical engineering knowledge         -> semantic-memory provider
current source-code structural retrieval -> code-intelligence provider
CLI / shell stdout or stderr             -> cli-output-optimization provider
large reconstructible general context    -> context-assurance provider
anything else                            -> native/direct path
```

Binding regardless of whether you load the detail:

1. Use code intelligence for symbol discovery, outlines, callers/callees, imports/importers, implementations/inheritance, blast radius, changed-symbol mapping, and narrow source retrieval—not for tickets, history, shell output, or live repository state.
2. Require freshness and coverage awareness. A stale/uncertain index is a locator only; incomplete/withheld coverage cannot support absence or safety claims.
3. Current repository/worktree/ref remains authoritative. Exact-read current source when the governing role/ticket requires exact bytes before an edit, review, approval, or correctness decision.
4. **One layer per surface.** Do not send jCodeMunch source/context through Entroly merely to compress it again, do not run it through RTK, and do not treat optional jCodeMunch memory/context features as Symphony semantic memory.
5. If the provider is unavailable, stale, incomplete, unlicensed for the environment, or insufficient for the question, fall back immediately to native targeted search/read. Do not block role execution.
6. Prefer the smallest useful code-intelligence tool surface; do not load a large MCP/tool catalog merely because the provider exposes it.
7. Treat local indexes/caches as source-derived confidential data and verify provider licensing before use on commercial/company repositories.

Operational policy, freshness/coverage classes, C/C++/SCIP guidance, security, licensing, role boundaries, and the first reference adapter:
`skills/code-intelligence/SKILL.md`. Load it on demand only when the active task benefits from current-code structural retrieval.

## Optional context assurance

A context-assurance provider (currently Entroly) may reduce a **large, already-narrowed, model-bound evidence bundle** while preserving provenance, recovery, and raw fallback. It is an optional accelerator, never a source of truth or dependency.

Use **surface ownership**, not file extension or command type, to choose an optimizer:

```text
exact / canonical / gate evidence        -> RAW; no optimizer
historical knowledge lookup              -> semantic-memory provider
current source-code structural retrieval -> code-intelligence provider
CLI / shell stdout or stderr             -> cli-output-optimization provider
large reconstructible general context    -> context-assurance provider
anything else                            -> native/direct path
```

Binding regardless of whether you load the detail:

1. **One optimizer/intelligence layer per surface.** Never chain RTK or code-intelligence output through Entroly, or use Entroly as semantic memory merely because it also implements memory features.
2. Mandatory init/protocol files, active ticket requirements, authoritative architecture, exact source/config needed for decisions/edits, repository gates, exact diffs, and material test/security evidence stay raw.
3. Token discipline runs first: narrow retrieval before considering context assurance. Do not collect broad context merely to compress it afterwards.
4. Use context assurance only when the remaining necessary non-authoritative/reconstructible **general** evidence is still materially large and the active environment positively exposes a policy-compliant provider path.
5. Require sufficient receipt/provenance and the ability to recover/reacquire exact originals when omitted evidence later becomes necessary.
6. If classification is ambiguous, the provider is unavailable, or the reduced result is insufficient, use/recover the native exact context. Never guess and never silently switch ownership to another optimizer.
7. The absence of RTK does not make CLI output an Entroly surface; the absence of semantic memory does not make historical recall an Entroly surface; the absence of code intelligence does not make source-code retrieval an Entroly surface.

Operational policy, surface classification, recovery, privacy, role guidance, and the first reference adapter:
`skills/context-assurance/SKILL.md`. Load it on demand only for a qualifying large model-bound context bundle.

## Optional CLI-output compression

An external compressor (currently RTK) may reduce noisy build/test/log output before it enters
context. It is an **optional accelerator**, never a dependency: Symphony behaves identically when it
is installed, absent, unsupported by the host, or unavailable to a cloud agent. When it is absent,
run the native command — no warning, no workflow change.

Binding regardless of whether you load the detail:

1. Never compress a mandatory gate's evidence. `git status --porcelain=v1 --untracked-files=all` and
   every other command whose exact output or exit status is consumed programmatically stay raw.
2. Never substitute compressed or structural output for an authoritative read of protocol files,
   tickets, `MEMORY.md`/`SKILL.md`, or source you are about to modify or review.
3. Detect by positive identification, never by binary name alone — unrelated tools publish the same
   name, and one of them writes Git tags.
4. When compressed output is insufficient, read the persisted raw output. Never rerun an expensive
   build or test suite merely because output was compressed, and never guess.

Operational detail — command classification, recovery, per-tool guidance, privacy configuration:
`skills/cli-output-optimization/SKILL.md`. Load it on demand, only after positive detection.

## Conversation discipline

Report only meaningful state changes, decisions, blockers, or evidence. Do not narrate every read, search, edit, or test command.

When `Agent role.md` defines an exact Role Work Loop line, print it verbatim—for example:

- `<role>|exit`
- `<role>|waiting on <ticket>`
- `<role>|blocked on <ticket> — needs SRTL`

Do not replace those strings with generic labels or explanatory paragraphs.

Outside an exact loop status, default to:

`<role>|<state> — <essential fact>`

If nothing materially changed, say nothing.

## Handoff discipline

The ticket is the inter-role API. Do not re-explain information already persisted there.

A normal conversational handoff contains only non-obvious execution facts:

```text
DONE: <ticket/status>
CHANGED: <key paths or one-line scope>
TESTED: <exact command/result>
ISSUES: <only if any>
```

Omit empty sections. If all detail and evidence already live in the ticket/commit, a one-line handoff is enough.

## Reasoning/output separation

Reason thoroughly enough for correctness. Token discipline targets unnecessary retrieved context and observable verbosity—not analysis quality.

Never replace verification with guessing, or omit evidence to appear concise.

## Priority order

When rules conflict:

1. user instruction;
2. correctness, security, privacy, and data integrity;
3. live-state/repository gates and optimistic concurrency;
4. role boundaries, lifecycle, and scope;
5. durable artifact and ticket completeness;
6. token efficiency.

## Vendor neutrality

All requirements in this skill are implementable through ordinary file reads, searches, edits, commands, and the repository access mode already authorized by Symphony.

Vendor-specific mirrors may point to this skill but may not contain exclusive protocol content. On conflict, the vendor-neutral Symphony files win.
