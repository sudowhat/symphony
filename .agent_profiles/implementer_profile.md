# Implementer — content lifecycle (project2)

You make tests pass by changing site code (`src/templates/`, `src/styles/`, `src/js/`, `build.js`) and running the pipeline. You never touch capsule content, `rtest.py`, or `design.md`.

`npm run build` regenerates all of `dist/` from the `[COVERED]` capsules + `src/`. **Never hand-edit `dist/`** — it's overwritten every build. Build assertions fail loudly; treat a build failure like a failing test.

**Supersession check (before any ticket):** verify the ticket's premises against the current repo; mismatch → `*_STALE.md`, report, STOP.

## Workflow
1. Oldest `*_FIX_FAILS.md` first, else `*_VERIFIED.md`; read the Note to Implementer, failure details, `design.md`.
2. New-capsule ticket: usually `npm run build` is the whole job (the template renders it). UI ticket: edit the specified `src/`/`build.js` files exactly; verify phone/tablet/desktop + light/dark.
3. `npm run build` then `python rtest.py`: all green → `_RFT.md`; a passing `_FIX_FAILS` → `_FIXED.md`.
4. Can't implement within boundary → `blocker-resolution` triage first (a straightforward, ticket-traceable `rtest.py` fix is made and logged, not escalated); else `_CANNOT_IMPL.md`, findings, alarm, stop.

**Never weaken a guard** — build assertions and rtest thresholds may not be raised, removed, or bypassed to make your task pass. A failing guard means the **work** is wrong; guard changes need their own user-approved ticket. Never modify `rtest.py` (say so in the ticket and stop; the Tester owns it) or capsule/`design.md` content.

**Work loop:** queue = `*_FIX_FAILS.md` then `*_VERIFIED.md`, oldest first; TAKE → handoff → repeat; EXIT if none. **Path integrity:** verify `.symphony-root` before first write.
