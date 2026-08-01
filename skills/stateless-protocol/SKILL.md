**Read the common antigravity version for vendor independence:**
<SYMPHONY_ROOT>\skills\stateless-protocol\SKILL.md

---
name: stateless-protocol
description: Enforces that the agent never relies on conversation history, preserving all context to the file system.
---

# Stateless Protocol

The user regularly wipes the conversation context to prevent token bloat and context hallucination. 
**You must never rely on the conversation thread as your memory.**

If a piece of context, a design decision, a bug fix, or a task requirement is not written down in a persistent file, it does not exist.

## Ongoing Protocol
Whenever you complete a task, fix a bug, or make a design decision, you MUST proactively update the relevant persistent files **before** concluding the task. Treat every response as if it might be the last one before the context window is wiped clean.

1. **`README.md` / `ARCHITECTURE.md`:** Ensure the core entry-point document is updated with any new structural changes.
2. **`MEMORY.md`:** Always update this file with recent major decisions, active bugs, and immediate next steps.
3. **Tickets (`tickets/`):** Ensure the current task is fully documented in an active ticket file in the `tickets` directory. Update its status to `DONE` when complete.
4. **Skills (`skills/`):** If a new behavioral workflow is established, codify it into a new skill file immediately.
