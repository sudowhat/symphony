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

## Standard commands

Typical evidence operations are:

- `adb -s <serial> shell uiautomator dump /sdcard/window.xml`
- `adb -s <serial> pull /sdcard/window.xml <ignored-run-dir>/...`
- `adb -s <serial> exec-out screencap -p > <ignored-run-dir>/...`

Adapt only the target package, activity launch command, and host shell syntax from the project’s own ADB sheet/skill. Do not place device-specific assumptions in this shared skill.
