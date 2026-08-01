---
name: blocker-resolution
description: How to triage a role-boundary block before opening a CANNOT — when a straightforward, ticket-traceable fix is safe to make yourself, and when it's a genuine design conflict that must escalate (with an audible alarm).
---

# Blocker Resolution — Self-Fix vs. Escalate (added 2026-07-29)

**The problem this solves:** the strict role boundaries (QA never writes code, Dev never writes tests, etc.) exist to prevent exactly the kind of damage recorded elsewhere in this protocol (WD-170/170A, CAP-020) — but taken too literally, they also turn a one-line stale-literal fix into a full `CANNOT` halt that blocks the whole route behind it. Real incident, 2026-07-29 (WD-272): a Dev implementing a P0 ticket hit two pre-existing test failures — one was a hardcoded schema-version literal made stale by the ticket's own migration (genuinely mechanical), the other required picking between several plausible ways to scope an assertion around new behavior (genuinely a judgment call). Both got the same treatment (full `CANNOT_DEV`) even though only one of them actually needed a human. This skill draws that line precisely, so it doesn't get drawn by gut feel next time.

**Read this before renaming any ticket to a `CANNOT_*` status.** It does not change what counts as a role boundary — it changes what you do in the narrow moment you're about to cross one for a test file.

---

## Scope — which boundary this applies to

This applies ONLY to the **test-vs-code split**: QA↔Dev (Android lifecycle) and Tester↔Implementer (content-web lifecycle). A blocked Dev/Implementer may, under the test below, make a narrow edit to a test file; a blocked QA/Tester may, under the same test, make a narrow edit to production code.

**It does NOT extend to any other boundary**, regardless of how obvious the fix looks:
- The Architect never ships production code in a `[DONE]` ticket.
- SRTL's full code+test dual authority stays SRTL-exclusive (this skill gives Dev/QA a narrow slice of what SRTL can already do everywhere — it does not give anyone SRTL's full authority).
- The Critic's exclusive placement/renumbering/slug-registry authority stands.
- Designer/Composer/Implementer's content-ownership walls stand.

Those boundaries exist to keep a single accountable owner for an architectural or editorial decision, not merely to gate mechanical edits — "obvious" is not a reason to blur who owns a decision. Only the test-vs-code split, where the blocking artifact is a literal test assertion rather than a design decision, is in scope here.

---

## The Straightforward-Fix Test

All five must hold, or you escalate instead. This is deliberately strict — when genuinely unsure, that uncertainty IS your answer (escalate):

1. **Traceable** — you can point to the exact line(s) in **your own ticket** (Requirements / Solution Approach / MAY-touch list) that directly, unambiguously necessitates this exact change. Not your interpretation of intent — the literal text of your own ticket.
2. **Mechanical** — the fix is a small, local edit: updating a stale literal/constant to track a value **this ticket** legitimately changed, or narrowly scoping an existing assertion to exclude a new, ticket-intentional behavior it never anticipated. It is NOT a new test, NOT a deleted test, NOT an `@Ignore`, NOT a redesign of what's being verified.
3. **Non-weakening** — after your fix, the guard checks exactly as much rigor as before, just against an updated, correct target. Raising a numeric threshold/limit to make your own work fit is never eligible, even if it looks small. If you can't immediately say what changed and why it isn't a weakening, it's not eligible.
4. **Confined** — touches only the specific assertion(s)/line(s) actually broken by your ticket's own change. No drive-by cleanup, no touching anything else in that file.
5. **Certain** — you are not guessing, inferring, or assuming. If there's more than one reasonable way to make the fix and you're picking one, you are not certain — that's a judgment call, which means escalate.

**If all five hold:** make the fix, log it (below), and continue toward `[DONE]` — do not open a CANNOT for this.
**If any one fails:** do not touch the file. Follow your role's existing CANNOT procedure exactly as written in your profile — nothing about that procedure changes — then see "Genuine Block" below for the one addition.

## Calibration examples (real cases — use these to gauge borderline calls)

- ✅ **Self-fix.** WD-272 added a new Room migration, legitimately bumping `SCHEMA_VERSION` 14→15 (Room migrations are version-numbered — there is no other way to add one). A regression lock asserted `assertEquals(14, Project1Database.SCHEMA_VERSION)`. Updating the literal to `15` (or to reference the constant) changes nothing about what's being verified — schema version must be current — and it was this exact ticket's own migration that made `14` stale. Traceable, mechanical, non-weakening, confined, certain.
- ❌ **Escalate** (CAP-020 incident, 2026-07): a build assertion capped title length at 28 chars; an agent's work didn't fit, so it widened the limit to 30 to force the work through. This is not tracking a value the ticket legitimately changed — it's loosening a limit because the agent's own output didn't satisfy it. Never eligible, however small the number looks.
- ❌ **Escalate** (WD-272's other blocker, same session): a new keeper alarm now exists after every app boot, which broke 4 pre-existing tests' exact-alarm-count assertions. Fixing this requires *picking* an approach — filter by the keeper's request code? by its broadcast action? cancel it before asserting? — and that's a test-authorship style call with more than one reasonable answer. Traceable and non-weakening, but not mechanical, and not certain. Correctly escalated.

## Logging a self-fix (mandatory, no exceptions)

Before promoting the ticket further, append a `## Boundary Crossing Log` section to **your own ticket** (create it if absent):
- File + line(s) touched, and the exact before → after.
- Which requirement in this ticket justified it.
- A one-line confirmation you checked all five test conditions above.

This is not optional for even a one-character fix. The whole point of a stateless, file-based protocol is that the next agent — or the user — can see exactly what happened without digging through git blame. An unlogged self-fix is indistinguishable from an unauthorized edit, and will be treated as one if found later.

---

## Genuine Block — CANNOT + Alarm

If the test above fails, this is a real design conflict, not a paperwork problem. Do exactly what your role profile's existing CANNOT procedure already says (rename the ticket, document findings in full, STOP, do not claim other tickets) — that procedure is unchanged by this skill.

**The one addition:** immediately after renaming the ticket, sound an audible alarm. A CANNOT is meant to interrupt someone, not wait silently in a file that only gets read by chance. On Windows (PowerShell):

```powershell
$player = New-Object System.Media.SoundPlayer "C:\Windows\Media\Alarm01.wav"
$player.Play()
Start-Sleep -Milliseconds 3000
$player.Stop()
```

If `Alarm01.wav` is unavailable, fall back to a plain tone of the same length:
```powershell
[Console]::Beep(800, 3000)
```

This is a Windows-environment convenience, not a cross-vendor protocol requirement — agents on other OSes should use an equivalent audible or desktop notification if one is available, and skip silently (never fail or delay the CANNOT over a missing sound) if not.
