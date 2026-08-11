---
name: token-discipline
description: Vendor-neutral token-efficiency rules for every Symphony agent. Compress conversational overhead and repeated context while preserving durable engineering artifacts and evidence.
---

# Symphony Token Discipline

This is a **Symphony-native, vendor-independent** skill. It applies identically to every model, CLI, IDE agent, cloud agent, and future vendor participating in Symphony.

No Symphony workflow may depend on Caveman, Claude Code hooks, Codex configuration, Cursor rules, or any other vendor-specific runtime. External token-saving plugins may additionally be used, but this file is canonical.

## Prime directive

> **Small mouth, full brain. Compress chatter; preserve engineering signal.**

Token reduction must never reduce correctness, reproducibility, test evidence, architecture fidelity, security/privacy requirements, or information required by the next role.

## Compress aggressively

Keep these terse and information-dense:
- progress/status messages;
- tool-call narration;
- readiness/WAIT/EXIT output;
- routine success commentary;
- explanations already evident from a diff/test result;
- handoff summaries;
- repeated restatement of ticket requirements or persisted decisions;
- repeated summaries of files already available to the next agent.

Prefer identifiers, paths, commands, facts, and short fragments over prose when unambiguous.

Bad:
`I have completed my detailed review and found that the resolver appears to use today's date rather than the date selected by the user.`

Good:
`Bug: resolver uses today, not selected date.`

## Preserve losslessly

Do not compress away or paraphrase information whose exact form matters:
- source code;
- required commands;
- paths and filenames;
- symbols, APIs, schemas, fields, IDs and status prefixes;
- diagnostically relevant error text;
- meaningful failing assertions/test evidence;
- patches/diffs when exact changes matter;
- acceptance criteria;
- architectural decisions and constraints;
- security/privacy requirements;
- reproduction steps where omission changes behavior;
- authoritative user wording;
- anything another agent needs to execute without guessing.

Trim irrelevant log noise, not the signal. Prefer targeted excerpts/tails over entire logs.

## Durable artifact rule

**Tickets, project MEMORY/SKILL/architecture docs, code and tests are durable engineering state, not conversational chatter.**

Do not shorten a durable artifact merely to save tokens when doing so would remove useful engineering detail. Instead:
1. write the full required information once to the canonical artifact;
2. thereafter reference the artifact/path/section instead of restating it;
3. do not duplicate the same explanation into multiple handoff files unless the protocol explicitly requires it.

This is the Symphony adaptation of the Caveman idea: **Caveman conversation; lossless engineering artifacts.**

## Read discipline — reduce input tokens too

Before reading, ask what fact is needed for the current decision.

Rules:
1. Prefer targeted search/grep/find over opening many files wholesale.
2. Read only the relevant range when the host supports ranged reads.
3. Do not reread stable files during the same unit of work unless required by Symphony's live-state/remote-movement gates.
4. Volatile state MUST still be reread whenever `global-skill` or the Direct-Remote/Repository Sync Gate requires it. Token saving never overrides freshness/integrity rules.
5. Do not recursively dump directories, full logs, generated trees, lockfiles, binaries, or unrelated source merely for orientation.
6. Start narrow; expand only when evidence requires it.
7. Reuse canonical persisted decisions instead of rebuilding context from conversation.

## Tool discipline

Every tool call should answer a concrete question or perform a required action.

Avoid:
- no-op/heartbeat calls;
- repeated status/list commands without a state-change reason;
- huge raw command output when a filtered query answers the question;
- reading the same file through multiple tools merely to confirm identical content;
- broad repository scans before the ticket/architecture points to the likely area.

Never suppress a mandatory Symphony gate to save tokens.

## Conversation discipline

During work, report only meaningful state changes or useful findings. Do not narrate every file open, grep, edit, or test command.

Default work update:
`<role>|<state> — <essential fact>`

Examples:
- `QA|TAKE WD-346 — reproducing active-occurrence omission.`
- `Dev|TEST — targeted regression now green.`
- `SRTL|BLOCK — ticket approach conflicts with persisted invariant.`

If nothing materially changed, say nothing.

## Handoff discipline

Do not re-explain the ticket to the next role. The ticket is the API.

A normal conversational handoff should contain only what is not already obvious/persisted, usually:

```text
DONE: <ticket/status>
CHANGED: <key paths or one-line scope>
TESTED: <command/result>
ISSUES: <only if any>
```

Omit empty sections. If all detail already lives in the ticket/commit, a one-line handoff is enough.

## Reasoning/output separation

Agents must still reason thoroughly enough for correctness. Token discipline targets **observable verbosity and unnecessary retrieved context**, not the quality of internal analysis.

Never replace analysis with guessing just to be short.

## Priority order

When rules conflict, use this order:
1. user instruction;
2. correctness/security/data integrity;
3. Symphony role boundaries, gates and lifecycle;
4. durable artifact completeness;
5. token efficiency.

Token saving is an optimization, never permission to violate a higher rule.

## Vendor neutrality

Examples may mention generic shell/file operations, but no requirement in this skill may require a specific AI vendor or agent product.

Vendor-specific mirrors/plugins are optional accelerators only. On conflict, this Symphony skill wins.
