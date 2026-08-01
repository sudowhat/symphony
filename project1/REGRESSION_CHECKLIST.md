# Regression Checklist (Behaviour Locks)

**Purpose:** every bug reported at least once is pinned here by a test that must never be deleted, weakened or ignored. When someone says "this came back", read this file first: if the lock is green, the report is something new; if the lock is missing, that gap is the bug.

**Rules**
- Locks live in `test/locks/<Area>BehaviorLocksTest`, named `lock_<ticket>_<behaviour>`.
- Append-only. Changing a lock needs an explicit Architect-approved line in the ticket that changes it.
- A ticket that fixes a reported bug adds its lock in the same commit as the fix.
- Batch close is blocked while any lock is red.

| Behaviour that must never break again | Symptom if it breaks | Ticket | Lock test | Status |
|---|---|---|---|---|
| The list opens at the top | "It loads halfway down the page" | P1-003 | `HomeBehaviorLocksTest.lock_p1003_list_opens_at_top` | pinned |