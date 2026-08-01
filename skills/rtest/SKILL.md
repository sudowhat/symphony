---
name: rtest
description: Common regression test conventions and TDD principles for all Symphony projects.
---

# rtest — Regression Test Skill

## What is rtest?

`rtest` is the regression test suite for Symphony Protocol projects. It is the quality gate that all code must pass before a ticket can be promoted to `[DONE]`.

The term "rtest" is used consistently across all projects. The exact command to run it is project-specific and lives in `<project-folder>/SKILL.md` (e.g., `rtest.bat` for Project1).

---

## TDD Principles

1. **Tests come first**: For `[APPROVED]` tickets, QA writes failing tests BEFORE Dev writes code.
2. **Red → Green**: Tests must fail on the current code (proving the bug/feature is not yet implemented), then pass after Dev's implementation.
3. **Tests are executable specifications**: They describe the expected behavior more precisely than prose.
4. **Full suite green**: Before promoting a ticket to `[DONE]`, the incremental `rtest` run (build cache on, **no `clean`**) must be green — this executes every test class whose inputs changed, not just the new tests. Pre-existing failures must be noted and distinguished from new failures. The **cold** full suite (`rtest --full-cold`) is the per-**batch** regression backstop, not a per-ticket step. See the **Test Execution Policy** below.

---

## Writing Good Tests

- **Cover the matrix**: happy path, edge cases, regressions, error conditions, boundary values.
- **Test the behavior, not the implementation**: Tests should still pass if the internal implementation changes.
- **One concept per test**: Each test method should verify one specific behavior. Use multiple test methods for different scenarios.
- **Descriptive names**: `testClone_fromPastSource_forcesFutureStart` is better than `testClone1`.
- **Lifecycle Testing (WD-221)**: Every new custom View must have a lifecycle test calling `assertSurvivesDetachCycle`.
- **Follow the Architect's QA/Testing Instructions**: The ticket's `## QA / Testing Instructions` section is your guide. Cover all scenarios mentioned.
- **Make tests fail first**: Verify the test fails on the current (pre-change) code before promoting the ticket. A test that passes immediately may be testing the wrong thing.

---

## Running Tests

**Project-specific commands live in `<project-folder>/SKILL.md`.**

Common patterns across projects:
- Full suite: `.\rtest.bat` (Windows) or `./rtest.sh` (Unix)
- Gradle: `.\gradlew.bat testDebugUnitTest --console=plain`
- Targeted: `.\gradlew.bat testDebugUnitTest --tests "com.example.ui.CalendarBarUiTest"`
- For web projects: `npm test` or equivalent (see project SKILL.md)

## Test Execution Policy (MANDATORY — overrides any "run the full suite" phrasing elsewhere)

There are four run modes. Always use the **cheapest mode valid for the step you are on**. The exact command for each mode lives in `<project-folder>/SKILL.md`.

| Mode | Runs | Who / When |
|---|---|---|
| `rtest --fast` | Small (pure-JVM) tests only; may add `--tests` to narrow to the changed package | QA proving RED; Dev's inner loop while coding |
| `rtest --targeted` | The specific new/affected test classes (Small **and** Medium) via `--tests` | QA's final RED check; Dev's "now GREEN" check |
| `rtest` (incremental) | Whole suite, build cache **on**, **NO `clean`** — only changed-input classes actually execute | Dev, ONCE, before `[DONE]` |
| `rtest --full-cold` | Whole suite after `clean`, cache disabled | ONCE per **batch**, at the Artifact Build step |

**Hard rules:**
1. **NEVER run `clean` before rtest during a ticket.** `clean` destroys Gradle incrementality and forces every test class to re-execute. Cold runs happen only at `--full-cold`.
2. Per ticket, the suite is verified **exactly once**, via the incremental `rtest` (cached). The cold full suite runs once per **batch**, not once per ticket.
3. The `[DONE]` "full suite green" gate is satisfied by the incremental `rtest` run. Do **not** additionally run a cold full suite per ticket.

---

## QA Responsibilities

- Read `[APPROVED]` tickets carefully, especially the `## QA / Testing Instructions` section.
- Extract test scenarios and write failing tests in the rtest suite. **Tag each test by size:** `@SmallTest` for pure-JVM logic tests (no Robolectric/Android runtime), `@MediumTest` for Robolectric/UI tests.
- **Verify the new tests FAIL using `rtest --fast`** (or `rtest --targeted` if the scenario needs a Medium/Robolectric test) on the ticket's package. Do **NOT** run a cold full `rtest` suite — see the Test Execution Policy above.
- Rename the ticket to `[READY_FOR_DEV]` (or `[READY_FOR_IMPL]` for content web).
- Commit and push the tests + renamed ticket. Commit message: `<ticket_name_without_status_prefix>.md`.

---

## Dev Responsibilities

- Read `[READY_FOR_DEV]` tickets carefully, especially `## Solution Approach` and `## Architectural Constraints`.
- Follow the Solution Approach exactly. Do not improvise.
- **Run `rtest --fast`** (optionally `--tests` on the changed package) while iterating on your code changes. Do NOT run a cold full suite during active development.
- Fix production code (not tests) until the ticket-specific tests pass; confirm them GREEN with `rtest --targeted`.
- If tests appear wrong or the Solution Approach is unworkable, rename to `[CANNOT_DEV]` and document findings. Do not modify tests.
- **Final Verification:** Only after the ticket-specific tests pass, run the **incremental** `rtest` (build cache on, **NO `clean`**) ONCE to ensure no regressions were introduced. Do NOT run a cold full suite per ticket — the cold suite is the batch-end gate.
- Promote to `[DONE]` only when the full suite is green.

---

## Robolectric UI Testing (Android Specific)

- **Avoid Complex Navigation via Clicks:** When testing deeply nested or complex UI states (e.g. drilling down from Year View -> Month View -> Week View via `RecyclerView` item clicks), avoid simulating UI clicks. Robolectric often fails to lay out `RecyclerView` items properly without tedious manual `measure`/`layout` calls, leading to `CANNOT_DEV` blockers when Dev's code is correct but the test fails to navigate.
- **Use Reflection for State:** Instead, use Kotlin reflection (`javaClass.getDeclaredField(...)` and `javaClass.getDeclaredMethod(...)`) to directly set internal state variables (like `selectedDate`) and invoke internal navigation methods (like `showWeekView()`) before making assertions on the resulting UI. This ensures tests are robust and accurately verify the intended UI state.

---

## CRITICAL: Never Block on Test Execution

**Rule:** ALL test commands (rtest, gradlew, any build/test script) that take more than ~10 seconds MUST be run in the background. The agent must NEVER run a test script synchronously in a single blocking call.

**Why:** When a test script runs synchronously, the agent is frozen — it cannot respond to the user, cannot report progress, and cannot do other work. Users experience long silences (5-10+ minutes) with no feedback.

**How:** Use the background-launch pattern from `skills/global-skill/SKILL.md` (Long-Running Commands section):

1. **Launch** the test in background with output redirected to temp files, return immediately.
2. **Tell the user** the test is running in background.
3. **Poll** on subsequent prompts (user message or agent continuation) — check `Test-Path $doneFile`, read tail of log if done.
4. **Report results** when complete.

**Never do this:**
```
# BAD — blocks the agent for minutes
.\rtest.bat --tests "com.example..."
```

**Always do this:**
```
# GOOD — returns immediately, poll later
Start-Process -FilePath ".\rtest.bat" -ArgumentList "..." -NoNewWindow -RedirectStandardOutput $logFile ...
Write-Host "Launched in background. Poll with: Test-Path $doneFile"
```

This applies to QA (proving RED), Dev (proving GREEN), and any other role running build/test commands.
