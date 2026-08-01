# P1-002: Past days in the week strip must read quieter

**Lane:** UI (Architect-direct, no QA agent)
**Priority:** P2
**Status:** Ready for Dev

*(SAMPLE TICKET - UI lane. UI-lane tickets are created directly as `[READY_FOR_DEV]`: no QA agent, because pixel assertions proved high-cost and low-yield. The human is QA, via the Manual Test Script. Behaviour that CAN be asserted as state still gets a lock - the lane exempts visuals, never behaviour.)*

## Problem

The month grid dims days that are behind us; the week strip does not. The same fact is styled two different ways on one screen.

## Requirements

1. A day before today renders muted in the week strip.
2. Today and future days are unchanged.
3. A **selected** past day stays legible - selection outranks the past treatment.
4. The muted colour is read from the month grid's binder, not re-invented.

## Architectural Constraints

**MAY touch:** the week-strip binder - the layout for the week cell - the lock test file - `REGRESSION_CHECKLIST.md`

**MUST NOT touch:** the month or year binders (read the token from them, change nothing).

> **Note for ticket authors:** if the Definition of Done names a lock test and a checklist row, the files holding them **must** appear in MAY touch. Omitting them forces the executing agent into a CANNOT it cannot resolve. This is the single most common ticket defect.

## Solution Approach

In the week-cell binder, after the today/selected branches, add a past branch using the month binder's existing colour token. Rank it **below** selected and today so requirement 3 holds by construction.

## Manual Test Script (user)

1. Open the week view with today mid-week. Days before today are visibly quieter. [ ]
2. Today keeps its filled marker. [ ]
3. Tap a past day - it is selected and fully legible, not muted-on-muted. [ ]
4. Compare with the month grid - the colour is identical in both. [ ]

## Retired tests

None.

## Regression Guard

| Behaviour | Symptom if it breaks | Ticket | Lock test | Status |
|---|---|---|---|---|
| Week strip mutes past days with the month grid's token | Two styles for the same fact on one screen | P1-002 | `CalendarBehaviorLocksTest.lock_p1002_past_days_muted` | pinned |
| A selected past day stays legible | Tapping a past day gives grey-on-grey | P1-002 | `CalendarBehaviorLocksTest.lock_p1002_selected_past_legible` | pinned |

## Definition of Done

- [ ] Past days muted with the existing token; today and future untouched
- [ ] Selection outranks the past treatment
- [ ] 2 locks green - checklist rows added - `rtest --fast` green - artifact builds
- [ ] Committed and pushed

## Escalation

If any step cannot be completed EXACTLY as written, rename to `[CANNOT_DEV]`, write what blocked you, and stop.