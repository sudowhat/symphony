---
name: ticket-management
description: Ticket creation, naming conventions, and workflow rules. Applies ONLY to coding projects.
---

# Ticket Management Skill

## Applicability

This skill applies **ONLY to coding and software projects** (projects containing source code, build systems, or software configurations). It does **NOT** apply to content-only repositories (such as wisdom capsules, markdown articles, text documentation, or pure-content databases).

This skill enforces a standard ticket tracking system across all coding projects. Whenever the user requests a new change, feature, or bug fix, you MUST create and manage a ticket for it.

---

## 1. Ticket Location

All tickets must be stored in a `tickets/` directory at the root of the current project workspace.
If the `tickets/` directory does not exist, create it.

---

## 2. Ticket Naming Convention

The filename must include the **current status as a prefix**, a project prefix, a ticket number, and a brief description.

**Preferred / Active usage (WhatDate Android):**
Format: `[STATUS]_<PROJECT_PREFIX>-<number>_<brief_description>.md`

Examples:
- `[APPROVED]_WD-078_add_title_focus_dtm_prefill_selected.md`
- `[READY_FOR_DEV]_WD-060_clone_auto_increment_title_suffix.md`
- `[DONE]_WD-077_crash_on_launch.md`
- `[DEFER]_WD-074_....md`
- `[HOLD]_WD-033_....md`

**Status values** (WhatDate style):
- `[DRAFT]` (Architect pre-`[APPROVED]` staging — design in progress; NOT yet in the active batch, NOT pickable by QA/Dev)
- `[APPROVED]`
- `[READY_FOR_DEV]`
- `[IN_PROGRESS]`
- `[DONE]`
- `[DEFER]`, `[HOLD]`, `[CANNOT]` (for non-active items)

When creating for WhatDate, use the `[STATUS]_WD-XXX_...` style and place the file in `whatdate-folder/tickets/`. Use `[APPROVED]` for a ticket whose Solution Approach + QA instructions are complete and ready for the active batch; use `[DRAFT]` when the design is still being worked out (investigation pending, a decision awaited from the user, or sections not yet finalized) — promote to `[APPROVED]` once finalized.

---

## 3. Creating a Ticket

When a new change is requested, create a new markdown file in the `tickets/` directory.
- Determine the next available ticket number by checking existing tickets. If none exist, start with `001`.
- The initial status in the filename should be `[APPROVED]` (for Architect-created tickets ready for the active batch), `[DRAFT]` (for Architect-created tickets still being designed, OR for content capsules), or `[DRAFT]` (Composer content capsules). `[DRAFT]` is the universal "design in progress, not yet ready for the next agent" state.

### Ticket Template (WhatDate Android)

```markdown
# <PROJECT_PREFIX>-<number>: <Title>

## Problem
<Briefly describe the requested change, feature, or bug. Include current behavior and expected behavior.>

## Requirements
<Numbered list of what must be true for this ticket to be considered complete.>

## Architectural Constraints
<Strict numbered rules the Dev must obey. Include:
- Files that MAY be touched
- Files that MUST NOT be touched
- Patterns to preserve
- Scope boundaries
- Any "follow exactly these steps" rules>

## QA / Testing Instructions
<Detailed test scenarios for the QA agent. Include:
- Happy path tests
- Edge cases
- Regression checks
- Specific UI interactions to verify
- What "done" looks like for QA>

## Solution Approach
<Precise, often numbered steps with code sketches that the Dev agent will follow exactly.>

## Progress Notes
<Updated by Dev during implementation.>
```

---

## 4. Work Flow & Progression

- **Do Not Start Immediately:** When a ticket is created (with `[APPROVED]` status), do **NOT** start solving it or working on the implementation/fix immediately unless the user explicitly instructs you to proceed.
- **Include a Plan:** Every newly created ticket MUST include a proposed **Plan** detailing how you intend to address the ticket.
- **Updating a Ticket:** As work progresses:
  - **Rename** the ticket file to reflect the new status (e.g., `[APPROVED]` → `[READY_FOR_DEV]`).
  - Update the **Status** field inside the file content to match.
  - Add brief updates to the **Progress Notes** section inside the ticket file.

---

## 5. Closing a Ticket

Once the requested change is fully completed and verified:
- **Rename** the ticket file to reflect the `[DONE]` status.
- Update the **Status** field inside the file to `Done`.
- Add a final note to the **Progress Notes** summarizing the completion and any commit hashes.

---

## Writing Tickets for Junior/Small-Model Agents (mandatory standard — added 2026-07-18)

Our QA and Dev seats are deliberately small models (e.g. grok-medium QA, cursor-composer Dev). They execute superbly from precise instructions and improvise badly from vague ones. Every ticket MUST therefore be executable without judgment calls:

1. **Exact coordinates, never descriptions.** `app/src/main/java/com/whatdate/ui/MainActivity.kt` → function `fetchDates()` → the `submitList` callback. Not "the home list code".
2. **Copy-pasteable code.** Show the replacement block or the test skeleton verbatim. If the agent has to invent structure, the ticket is not finished.
3. **Exact assertions with exact expected values.** "next = 2026-10-15" — never "verify it works".
4. **Explicit MAY-touch / MUST-NOT-touch file lists** in Architectural Constraints. Anything not listed is out of scope by definition.
5. **A `## Regression Guard` section** naming the lock test (see agent-symphony §Regression Guard).
6. **A Definition of Done checklist** at the end: tests green · lock added · checklist updated · scope respected · committed + pushed.
7. **An escalation line:** "If any step cannot be completed EXACTLY as written, rename to `[CANNOT_QA]`/`[CANNOT_DEV]`, write what blocked you, and stop. Do not improvise, do not widen scope, do not weaken a test or a guard to make something pass."
8. **No compound requirements.** One numbered requirement = one verifiable thing. Split anything with an "and" that hides a second decision.
