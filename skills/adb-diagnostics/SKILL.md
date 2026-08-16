---
name: adb-diagnostics
description: Optional SRTL protocol for deterministic Android Debug Bridge diagnostics on a user-connected USB-debugging device. Use only after init <project> srtl adb [serial] to run project-owned ADB cases that can be driven and proven without manual judgment.
---

# ADB Diagnostics

Use this skill only as the explicit SRTL ADB overlay. It never auto-starts because a project is Android or a device happens to be connected.

## Availability gate

Run this only after the normal init sequence has passed the repository/direct-remote gate and project context has identified the target package.

1. Run `adb devices -l`.
2. Accept exactly one device whose state is `device`, unless the init command supplied a serial that matches one such device.
3. Verify the project-owned target package is installed through ADB package state.
4. If any check fails, report one terse `ADB_UNAVAILABLE: <reason>` and continue normal SRTL work. Do not open a case, change device state, create a CANNOT, or retry/poll.
5. If eligible, select the device once with `adb -s <serial>`; never send an unqualified ADB command later in the run.

Do not record raw device serials in tickets, ledgers, or committed files.

## Case admission contract

Run only a case that has all six:

1. **Deterministic fixture:** all required data can be created in the app UI with a unique disposable prefix; it never needs an existing user item.
2. **Public interaction:** every action uses ADB-visible UI interaction. Use a fresh UI dump and a unique resource-id/content description; visible text is fallback. Use coordinates only for a documented bounds assertion, with screenshot/dump evidence.
3. **Deterministic oracle:** result is exact text/state/bounds from a UI dump or explicit package/process state—not a human impression.
4. **Controlled dependency:** no device clock/settings changes, network content, third-party picker content, camera hardware, notification delivery, audible output, or permission state is needed.
5. **Visible cleanup:** fixtures are removed/archived through the app UI before the next case.
6. **Concise evidence plan:** names the before/after dump or screenshot under `.workspace-temp/adb-diagnostics/<run-id>/`.

If a promised selector, fixture route, or oracle is missing, report `PRECONDITION_DEFECT`. Repair or retire the script; never relabel that as “blocked” or invent an unobservable substitute.

## Safe operating rules

- Never run `pm clear`, uninstall, data/database/file edits, root commands, device time/time-zone/settings changes, permission grants/revokes, or broadcasts/intents that bypass the visible app journey.
- Do not install/replace an APK as a diagnostic shortcut. Installation belongs to the normal approved build/release flow.
- `am force-stop` followed by the normal app launch is allowed only when the case says so; it must not clear data.
- Do not interact with user-owned entries/categories. If fixture cleanup fails, mark FAIL and stop mutation until the user decides how to recover it.
- ADB diagnostics do not prove audio, color/taste, typography, animation quality, network rendering, OS notification delivery, or any behavior outside the stated oracle. Keep those in manual acceptance or code tests.
- Preserve all repository sync, scope, ticket, security/privacy, regression-lock, rtest/build, and final-human-acceptance rules. This skill adds evidence; it removes no gate.

## Efficient execution

- Create one run directory, then take fresh targeted dumps only at state transitions. Avoid whole-screen discovery loops and repeated screenshots of unchanged state.
- Store raw XML and PNG evidence in the ignored run directory. In the ledger/ticket record only case result, evidence filenames, and exact mismatch; do not restate the script.
- Use `PASS`, `FAIL`, `PRECONDITION_DEFECT`, and `READY` inside ADB sheets. `ADB_UNAVAILABLE` applies to the whole requested overlay, not an individual case.
- Stop the overlay after a destructive-risk or cleanup failure; give the user the exact fixture title and evidence names.

## Defects found during a case — fix small, draft big (user ruling 2026-08-16)

When a case's oracle turns up a genuine, confirmable app defect (not a matter of taste), triage the
fix by size and certainty before touching anything, the same way `skills/blocker-resolution/SKILL.md`
triages a role-boundary block:

1. **Quick fix (roughly 2-3 minutes of work, small and certain):** fix it directly under SRTL's
   normal code authority. Commit as `SRTL fix: <short description>`, retest the case on-device, and
   record the fix SHA in that case's Run record row exactly as the sheet's ledger conventions already
   require.
2. **Not a quick fix** (needs real investigation, touches a shared/product decision, or otherwise
   needs more than a fast pass affords): do not attempt it live. Instead create
   `tickets/[DRAFT]_WD-<next>_<slug>.md` containing ONLY:
   - the exact originating test case (sheet filename + case number), so the evidence trail is
     traceable back to the ADB run that found it;
   - the problem description — what was observed vs. the case's stated oracle, in the case's own
     terms. No `## Solution Approach`, `## Architectural Constraints`, or `## QA / Testing
     Instructions` — a `[DRAFT]` ticket is explicitly unfinished, and those sections are the
     Architect's job when converting it to a full ticket.
   Leave the case's own Run record row as `FAIL` with a one-line pointer to the new ticket number;
   never mark it PASS or paper over it with a workaround.

This does not change the sheet's own "commit the plan before the fix" rule for anything actually
attempted (see "Ledger, evidence and resume conventions" rule 5) — it only decides, before that,
whether SRTL attempts the fix at all.

## Run resumability

A long ADB run is expected to be interrupted — the session ends, the user stops it, the device drops. A stateless seat re-entering through `init <project> srtl adb [serial]` must be able to continue from the files alone, without asking what already happened.

- **The Run record table is the resume pointer.** `READY` means unrun; anything else is terminal. A fresh seat continues at the first `READY` row in the lowest-numbered sheet. Never pre-fill a row before its case finishes, and never leave a finished case as `READY` — an inaccurate table sends the next seat to the wrong place.
- **Commit per case, not per sheet.** Write the case's row and commit it as soon as that case reaches a terminal result. An interruption must never cost more than the case in flight.
- **Keep a short Run state block beside the Run record**, current as you go: run-id, target package with `versionName`/`versionCode`, device model/product, the case in flight, and **every disposable fixture still live on the device, by exact name**. Identify the device by model, never by serial (see the serial rule above). The fixture list is the part that matters most: an abrupt stop otherwise strands prefixed fixtures on the user's real device with nothing recording that they exist, and the suite's final cleanup sweep never runs.
- **Evidence is ephemeral; the commit is the durable record.** Delete a case's evidence once its row is committed **and pushed** — never while a result exists only locally, and never before everything needed from it is written into the row, since it is not recoverable afterward.
- **Record the fix commit's short SHA in the row of the case that prompted it.** This is what survives evidence deletion and lets anyone trace a result back to the change that caused it.
- **Commit the plan before the fix.** When a case FAILs and you intend to repair it, first write the observed mismatch, the suspected root cause, and the files you intend to touch into that case's row, and commit that. Then implement, `SRTL fix:` commit, retest the case on-device, and record the SHA. A few lines is enough. This keeps the scope pinned in writing before editing begins, and means an interruption between diagnosis and repair hands the next seat your reasoning instead of an unexplained FAIL.

## Standard commands

Typical evidence operations are:

- `adb -s <serial> shell uiautomator dump /sdcard/window.xml`
- `adb -s <serial> pull /sdcard/window.xml <ignored-run-dir>/...`
- `adb -s <serial> exec-out screencap -p > <ignored-run-dir>/...`

Adapt only the target package, activity launch command, and host shell syntax from the project’s own ADB sheet/skill. Do not place device-specific assumptions in this shared skill.
