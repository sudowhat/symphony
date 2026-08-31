# Grok Build CLI Global Preferences (Antigravity / Vendor Independent)

This is the canonical shared version.

## PowerShell / Terminal Title Rule
Always keep the console title exactly equal to the current Grok session title (managed by the TUI via `[ui.notifications.title]`).

## Symphony Auto-Rename Recommendation (New)
For users following the Symphony Protocol (WhatDate, Sulipi, etc.):

**Strongly recommended:** Load the global skill
`<user-home>/.grok/skills/symphony-auto-rename/SKILL.md`

(or the equivalent in your `.grok/skills/`).

When an agent is initialized with `init dev`, `init qa`, `init architect`, etc., this skill causes the agent to end its response with the exact command:

```
/rename <project> <role>
```

Example outputs the agent will produce:
- `/rename whatdate dev`
- `/rename whatdate qa`
- `/rename whatdate architect`

Run that line (or let future orchestrator automation do it) to give the session a stable, discoverable title.

This is essential for the `orchestrate-wd.ps1` script (and future multi-agent orchestration) to reliably locate the right session via `grok sessions search`.

## Other Rules
- Respect `~/.grok/config.toml`
- Do not manually change terminal titles
- Session title is the single source of truth

For full Symphony rules, also load `agent-symphony/SKILL.md`.