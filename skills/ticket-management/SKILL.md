---
name: ticket-management
description: Ticket creation, naming, and templates. Coding projects only.
---

# Ticket Management

Applies to **coding projects only** (not pure-content repos). Every requested change/feature/fix becomes a ticket.

## Naming
`[STATUS]_<PREFIX>-<number>_<brief_description>.md`, in `tickets/`. E.g. `[APPROVED]_P1-078_title_focus_prefill.md`, `[DONE]_P1-077_crash_on_launch.md`.

**Statuses:** `[DRAFT]` (Architect design-in-progress — not in the active batch, not pickable by QA/Dev) · `[APPROVED]` · `[READY_FOR_DEV]` · `[IN_PROGRESS]` · `[DONE]` · `[DEFER]`/`[HOLD]`/`[CANCELLED]`/`[CANNOT_*]`.

Next number = scan existing tickets (none → `001`). Create as `[APPROVED]` when the Solution Approach + QA instructions are final; `[DRAFT]` while still designing.

## Template
```markdown
# <PREFIX>-<number>: <Title>

**Lane:** backend | UI

## Problem
<current vs expected behaviour, with code references>

## Requirements
<numbered; one verifiable thing each>

## Architectural Constraints
<numbered: MAY-touch files, MUST-NOT-touch files, patterns to preserve, scope>

## QA / Testing Instructions
<happy path, edges, regressions, exact expected values, what "done" means>
<mark OS-level scenarios: [NOT ROBOLECTRIC-TESTABLE — …]>

## Solution Approach
<precise numbered steps + code sketches Dev follows exactly>

## Regression Guard
<behaviour · symptom if broken · lock test name — see agent-symphony §Regression Guard>

## Definition of Done
<tests green · lock added · checklist updated · scope respected · committed+pushed>

## Escalation
If any step cannot be completed EXACTLY as written, rename to [CANNOT_*], write
what blocked you, and stop. Do not improvise, widen scope, or weaken a guard.
```
(UI-lane: replace QA Instructions with `## Manual Test Script (user)`, add `## Retired tests` if any.)

## Progression
Don't start implementing on creation unless told. Rename the file's status prefix and update the internal Status field on each transition; add brief `## Progress Notes`. On completion → `[DONE]`, append the commit hash.

## Write for small-model executors (mandatory standard)
QA/Dev seats are deliberately small models: superb from precise instructions, bad at improvising. Every ticket must be executable **without judgment calls**:
1. **Exact coordinates, not descriptions** — `…/MainActivity.kt` → `fetchDates()` → the `submitList` callback, not "the home list code".
2. **Copy-pasteable code** — show the replacement block / test skeleton verbatim. If the agent must invent structure, the ticket isn't finished.
3. **Exact assertions with exact values** — "next = 2026-10-15", never "verify it works".
4. **Explicit MAY-touch / MUST-NOT-touch** lists — unlisted = out of scope.
5. **A `## Regression Guard`** naming the lock test.
6. **A Definition of Done** checklist.
7. **An escalation line** (above).
8. **No compound requirements** — one numbered item = one verifiable thing; split any hidden "and".
