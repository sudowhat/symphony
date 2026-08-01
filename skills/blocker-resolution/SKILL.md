---
name: blocker-resolution
description: Triage a role-boundary block before opening a CANNOT — when to self-fix (logged) vs escalate (with an alarm).
---

# Blocker Resolution — self-fix vs escalate

Strict role boundaries (QA never codes, Dev never touches tests) prevent real damage — but taken too literally they turn a one-line stale-literal fix into a full CANNOT that halts the whole route. This skill draws the line precisely.

*(WD-272: a Dev hit two pre-existing failures — one was a hardcoded schema-version literal made stale by the ticket's **own** migration (mechanical); the other needed picking among several ways to scope an assertion (a judgment call). Both got a full CANNOT even though only one needed a human.)*

**Read this before renaming any ticket `[CANNOT_*]`.** It doesn't change role boundaries — only what you do in the narrow moment you're about to cross the **test↔code** split.

## Scope
ONLY the test-vs-code split: QA↔Dev and Tester↔Implementer. A blocked Dev/Implementer may make a narrow test-file edit; a blocked QA/Tester a narrow production edit — **only** under the test below. Every other boundary stands regardless of how obvious the fix looks (Architect never ships `[DONE]` code; SRTL's dual authority stays SRTL's; the Critic's placement/renumbering stays the Critic's). Those keep a single accountable owner for a decision — "obvious" is not a reason to blur ownership.

## The Straightforward-Fix Test — all five, or escalate
When genuinely unsure, that uncertainty **is** your answer (escalate).
1. **Traceable** — you can point to the exact line(s) in **your own ticket** that unambiguously necessitate this exact change (the literal text, not your reading of intent).
2. **Mechanical** — a small local edit: updating a stale literal to track a value **this ticket** legitimately changed, or narrowly scoping an existing assertion to exclude new ticket-intentional behaviour. Never a new/deleted/`@Ignore`d test or a redesign of what's verified.
3. **Non-weakening** — after the fix the guard checks exactly as much rigor, against an updated correct target. Raising a threshold/limit to make your work fit is **never** eligible. Can't instantly say what changed and why it's not a weakening → not eligible.
4. **Confined** — only the assertion(s) actually broken by your ticket's own change. No drive-by cleanup.
5. **Certain** — not guessing. More than one reasonable way to do it → it's a judgment call → escalate.

All five hold → make the fix, **log it**, continue. Any one fails → don't touch the file; follow your CANNOT procedure.

## Calibration
- ✅ **Self-fix:** ticket added a Room migration, bumping `SCHEMA_VERSION` 14→15 (versioned — no other way); a lock asserted `== 14`. Updating to `15` verifies the same thing (version must be current) and this ticket's own migration made `14` stale. All five hold.
- ❌ **Escalate:** a build assertion caps titles at 28; an agent's output didn't fit, so it widened to 30. Loosening a limit to force work through — never eligible, however small.
- ❌ **Escalate:** a new keeper alarm broke 4 exact-alarm-count assertions; fixing needs *picking* an approach (filter by request code? by action? cancel first?) — traceable and non-weakening, but not mechanical, not certain.

## Logging a self-fix (mandatory, even one character)
Append a `## Boundary Crossing Log` to your own ticket: file + line(s) touched, exact before→after, which requirement justified it, one line confirming all five conditions held. An unlogged self-fix is indistinguishable from an unauthorized edit and will be treated as one.

## Genuine block — CANNOT + alarm
Test fails → real conflict. Follow your role's CANNOT procedure (rename, document fully, STOP, take no other ticket). **The one addition:** sound an audible alarm immediately after renaming — a CANNOT is meant to interrupt someone, not wait to be found by chance.
```powershell
[Console]::Beep(800, 3000)   # or play a WAV; fall back to a desktop notification
```
Windows convenience, not a protocol requirement — on other OSes use an equivalent, and skip silently (never fail or delay the CANNOT) if none.
