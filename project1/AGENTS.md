# Agents

This project uses the **Symphony Protocol**. Any agent, any vendor, starts the same way:

```
init project1 <role>        # architect | qa | dev | srtl | orchestrator
```

Source of truth, in mandatory load order:

1. `<SYMPHONY_ROOT>/Agent role.md` — universal entry and role registry
2. `<SYMPHONY_ROOT>/.agent_profiles/<role>_profile.md` — role boundaries
3. `<SYMPHONY_ROOT>/skills/global-skill/SKILL.md` — global rules and repository/live-state gates
4. `<SYMPHONY_ROOT>/skills/token-discipline/SKILL.md` — lossless input/output discipline
5. `<SYMPHONY_ROOT>/skills/agent-symphony/SKILL.md` — lifecycle and execution protocol
6. Pass Path Integrity and the Repository Sync/Direct-Remote Gate
7. `MEMORY.md` — current project state
8. `SKILL.md` — technical, build, and test conventions
9. Load conditional role skills
10. Read the active route, claim, ticket, and work state

Mandatory governing files are read completely. Token discipline applies to subsequent exploration and reporting; it never permits skipping a gate, fresh state, complete ticket, code, tests, exact errors, diffs, or evidence.
