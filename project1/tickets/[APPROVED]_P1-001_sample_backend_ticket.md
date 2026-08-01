# P1-001: Search must match singular and plural the same way

**Lane:** backend
**Priority:** P2
**Status:** Approved

*(SAMPLE TICKET - backend lane. Backend means the behaviour is cheap to assert in a pure unit test, so it goes through full TDD: QA writes failing tests first, then Dev makes them pass.)*

## Problem

Searching `birthdays` returns nothing although entries tagged `birthday` exist. Tags are stored singular by convention, but people type either form.

## Requirements

1. A query and a stored word match when their normalised forms are equal.
2. Normalisation lowercases and strips a trailing `s` or `es` from words longer than 3 characters.
3. Normalisation is applied to **both** sides of the comparison, from **one** helper. Do not duplicate the rule.

## Architectural Constraints

**MAY touch:** `src/search/Matcher`, `test/` , `REGRESSION_CHECKLIST.md`

**MUST NOT touch:** the index schema, any query signature, any migration.

**Rules:**
- No schema change, no migration.
- One helper, both sides. A second copy of the rule is free to drift from the first.

## Solution Approach

```
fun normalizeWord(w: String): String {
    val s = w.trim().lowercase()
    return when {
        s.length > 3 && s.endsWith("es") -> s.dropLast(2)
        s.length > 3 && s.endsWith("s")  -> s.dropLast(1)
        else -> s
    }
}
```

Call it from the single tag-comparison site. Nowhere else.

## QA / Testing Instructions

1. `lock_p1001_singular_plural_match` - `normalizeWord("birthdays") == "birthday"`; matching succeeds in **both** directions.
2. Assert the helper is called on both sides of the comparison, so a one-sided fix cannot pass.

## Regression Guard

| Behaviour | Symptom if it breaks | Ticket | Lock test | Status |
|---|---|---|---|---|
| Search matches singular and plural both ways | `birthdays` finds nothing though `birthday` entries exist | P1-001 | `SearchBehaviorLocksTest.lock_p1001_singular_plural_match` | pinned |

## Definition of Done

- [ ] One helper, applied to both sides
- [ ] Lock green - checklist row added - incremental `rtest` green
- [ ] Committed and pushed

## Escalation

If any step cannot be completed EXACTLY as written, rename to `[CANNOT_QA]` / `[CANNOT_DEV]`, write what blocked you, and stop. Do not improvise, do not widen scope, do not weaken a test or a guard to make something pass.