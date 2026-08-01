---
name: stateless-protocol
description: Never rely on conversation history; the filesystem holds all context.
---

# Stateless Protocol

The user wipes conversation context regularly to avoid token bloat and hallucination. **The chat is never your memory.** If a decision, fix, or requirement isn't in a persistent file, it does not exist.

**The two-way rule:**
1. **Don't read from the conversation for durable truth** — reload it from files each session.
2. **Don't write durable truth into the conversation** — no progress recaps, no "context to carry forward", no re-summarising what you did. That's wasted tokens; the next agent won't see the chat anyway.

Instead, before concluding any task, put what matters in the right file — in as few words as possible:
- **`ARCHITECTURE.md` / `README.md`** — structural or design changes.
- **`MEMORY.md`** — major decisions, active bugs, next steps (keep it short; move finished items to `# HISTORY`).
- **`tickets/`** — the task itself; update the status prefix to terminal when done.
- **`skills/`** — a new reusable workflow.

Treat every response as if the context is wiped right after it. The filesystem is the memory; the conversation is disposable.
