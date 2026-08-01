# Project Memory

**Keep this file short.** Ticket detail lives in `tickets/`. Canon lives in `ARCHITECTURE.md`. Guarantees live in `REGRESSION_CHECKLIST.md`. If something belongs in a ticket, put it in the ticket.

## Where truth lives

| Question | File |
|---|---|
| Why is it built this way? | `ARCHITECTURE.md` |
| What did ticket N do? | `tickets/[DONE]_P1-00N_*.md` |
| Did this regress or is it new? | `REGRESSION_CHECKLIST.md` |
| How do I build and test? | `SKILL.md` |

## Core philosophy

*(Replace with yours. One paragraph. Every design decision is judged against it.)*

**Example:** this is not a CRUD app - it is a tool people trust with something they cannot re-create. Every decision is judged by whether it strengthens or dilutes that trust.

Standing rulings:
- Data loss is a strict NO. It outranks everything else.
- Performance is a requirement, not a polish step.
- Guards are never weakened to make work pass.

## Current state

Batch P1-001 -> P1-003 in flight. See `ticketorder.md`.