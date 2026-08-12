---
name: marathon
description: Single-seat batch completion mode. One agent reviews what is already done, then executes every remaining role step in an open batch to the end, unblocking itself with Architect judgment and stopping only for decisions that are genuinely the user's.
---

# Marathon Mode

**Trigger:** the user says **"start marathon"** (or "run the marathon", "marathon the batch").

Established by user ruling 2026-08-12. This skill is vendor-neutral and lives in the Symphony tree
because every agent must read the same definition — never in a vendor-local memory or skill store.

---

## What it is

Normally each role runs in its own seat and hands off through the ticket files. Marathon mode
collapses that into **one seat carrying an open batch from its current head to the end**: review
whatever is already finished, then perform every remaining role step in route order, deciding as an
Architect would when something needs deciding, and interrupting the user only when the decision is
genuinely theirs.

The user's framing: *"review already DONE and finish remaining in the batch."*

---

## The loop

```
0. init + Repository Sync Gate (or Direct-Remote Gate). A failed gate ends the marathon.
1. HEAD REVIEW: if the route head is a pending review of work you did NOT implement,
   you are a legitimately fresh seat — review it properly and write its terminal.
2. WALK: read ticketorder.md top-down. Take every open line in order, whatever its role
   token. Never skip the head. One seat plays QA, then Dev, then SRTL on each ticket.
3. TERMINAL: work you implemented yourself ends at :DONE on your own line, -SRTL left OPEN.
4. UNBLOCK: if blocked in a way an Architect could resolve, resolve it inline, record the
   decision in the ticket, continue. Do not raise a CANNOT for something you can decide.
5. RING: only for a decision that is genuinely the user's. Otherwise keep going.
6. CLOSE: attempt batch close. Any open -SRTL line blocks it — stage everything, then ring.
```

Marathon mode's single deviation from the Role Work Loop (`Agent role.md`) is step 2: one seat may
take consecutive lines belonging to different roles. Everything else in that loop still binds —
strict top-down order, never skipping the head, sync gate before each fresh queue read.

---

## Hard limit — the one thing marathon mode may never optimise away

**WD-334 / `<project>/SKILL.md` §"Independent SRTL Review Attestation": a seat that implements or
materially corrects a ticket may not write its own `:REVIEWED` or `SRTL Review: PASS`.**

| Situation | What you write |
|---|---|
| You implemented it | `:DONE` on your own QA/Dev line, and **leave the `<id>-SRTL` line open** |
| You did **not** implement it (genuinely fresh seat) | `:REVIEWED` on the reviewed line + `<id>-SRTL:DONE` |
| User personally signed off | `:REVIEWED:USER` |
| User explicitly waived review | `:REVIEW_WAIVED:USER` — never recorded as reviewed |

**One signal:** an open `<id>-SRTL` line is the single statement that review is owed. Do not invent a
second encoding for it (the retired `:REVIEW_PENDING` suffix was exactly that mistake, and the
retired `:SRTL` suffix collided with the line form).

Speed is never a reason to self-certify. A marathon that ends with honest open `-SRTL` lines
succeeded; one that ends with self-signed `:REVIEWED` failed, however green the tests are.

An open `-SRTL` line blocks batch close, version bump and build. That wall is the normal, expected
end of a marathon the same seat implemented — reaching it is completion, not failure.

---

## Self-unblocking (the Architect hat, without re-initing)

Resolve inline and keep moving when the blocker is a **design or specification** question you can
answer from the ticket, `ARCHITECTURE.md`, the UTS, and project canon:

- an ambiguous or under-specified Solution Approach;
- a stale premise (ticket cites a file/count that has since moved) that is unambiguously repairable;
- a conflict between two constraints where canon clearly prefers one;
- a missing test seam that the ticket's own scope authorises.

Record every such decision in the ticket under `## Architect Decision (marathon, <date>)` with the
reasoning. The next reader must be able to see that a judgment was made and why.

**Ring the bell instead when:**

1. the decision is product intent, scope change, or priority — the user's call, not yours;
2. resolving it would weaken a guard, lock, or build assertion (never permitted regardless);
3. an open `-SRTL` line blocks batch close and you need review or an explicit waiver;
4. the premise is stale in a way that needs a human (`*_STALE` territory);
5. anything risks data loss, or contradicts a standing ruling in the project `MEMORY.md`;
6. repeated failure you cannot diagnose — report evidence rather than improvising.

Bell procedure: `global-skill/SKILL.md` §"Audible Attention Signal" (6 seconds).

---

## What still binds during a marathon

Marathon mode changes **who** does the steps, never **what the steps are**:

- scope lock — only files the ticket authorises;
- one ticket = one commit; commit **and push** on each terminal, never batch them up;
- guards, locks and build assertions are never weakened to make work pass;
- encoding/line-ending discipline; targeted edits, never wholesale rewrites;
- the project's test-execution ruling (for WhatDate: no bare `.\rtest.bat`, no full/cold suite —
  targeted runs only);
- TDD order per ticket: the failing test exists and is proven RED before the production fix;
- data loss is a strict NO;
- `ticketorder.md` hygiene — append `:DONE` to your own line only; prune only at batch close.

---

## Reporting

One line per state change, per `token-discipline`. At the end, a single report: tickets completed,
terminals written, test evidence, decisions made under the Architect hat, and exactly what remains
blocked and on whom.
