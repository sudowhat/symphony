---
name: rtest
description: Regression-test conventions and TDD tiers for all Symphony projects.
---

# rtest — Regression Test Skill

`rtest` is the regression suite every ticket passes before `[DONE]`. The term is universal; the exact command is per-project in `<project>/SKILL.md`.

## TDD
QA writes **failing** tests before Dev codes. Red (fails on current code, proving the gap) → Green (passes after Dev). Tests are executable specs — more precise than prose. Before `[DONE]`, the **incremental** `rtest` (cache on, no `clean`) must be green.

## Writing good tests
Cover the matrix (happy path, edges, regressions, boundaries). Test **behaviour, not implementation** (survives an internal refactor). One concept per method; descriptive names (`testClone_fromPastSource_forcesFutureStart`). Every new custom View gets a lifecycle test (detach→re-attach). Follow the ticket's QA Instructions. **Verify RED first** — a test that passes immediately is testing the wrong thing.

## Execution tiers (overrides any "run the full suite" phrasing elsewhere)
Use the cheapest valid mode for the step. Exact commands per-project.

| Mode | Runs | Who / when |
|---|---|---|
| `rtest --fast` | Small (pure-JVM) only; `--tests` to narrow | QA proving RED; Dev's inner loop |
| `rtest --targeted` | The specific new/affected classes (Small + Medium) | QA's final RED; Dev's "now GREEN" |
| `rtest` (incremental) | Whole suite, cache **on**, **no `clean`** — only changed-input classes execute | Dev, ONCE, before `[DONE]` |
| `rtest --full-cold` | Whole suite after `clean`, cache off | ONCE per **batch**, at the artifact build |

**Hard rules:** never `clean` during a ticket (it destroys incrementality and re-runs everything). Per ticket the suite is verified exactly once (incremental); the `[DONE]` "full green" gate is that incremental run — don't add a cold run per ticket. The cold suite is the batch backstop.

## QA / Dev split
- **QA:** write failing tests from the QA Instructions, **tag by size** (`@SmallTest` pure-JVM, `@MediumTest` Robolectric/UI), verify RED with `--fast`/`--targeted` (never cold full), promote, commit `<ticket>.md`, push.
- **Dev:** follow the Solution Approach exactly; iterate `--fast`; confirm GREEN with `--targeted`; incremental `rtest` once before `[DONE]`. Never edit tests — a wrong-looking test or unworkable approach → `[CANNOT_DEV]` with findings.

## Robolectric (Android)
Don't simulate `RecyclerView` click-chains to reach a UI state — Robolectric doesn't reliably lay out items, causing false CANNOTs. **Drive state via reflection** instead (`getDeclaredField`/`getDeclaredMethod` to set `selectedDate`, invoke `showWeekView()`), then assert.

## Never block on tests
Any test/build command over ~10s runs in the **background** (`global-skill` §Long-running commands): launch → tell the user → poll on later turns → report trimmed result. Never run a suite synchronously in one blocking call — it freezes the agent for minutes. Applies to QA, Dev, and every role running build/test commands.
