# Agents

This project uses the **Symphony Protocol**. Any agent, any vendor, starts the same way:

```
init project1 <role>        # architect | qa | dev | srtl | orchestrator
```

Source of truth, in load order:

1. `<SYMPHONY_ROOT>/Agent role.md` - the entry point and role registry
2. `<SYMPHONY_ROOT>/.agent_profiles/<role>_profile.md` - your boundaries
3. `<SYMPHONY_ROOT>/skills/` - shared behaviour rules
4. `MEMORY.md` (this folder) - live state
5. `SKILL.md` (this folder) - build and test commands

Do not read source code or change anything before that sequence is complete.