---
name: project1-skill
description: Project-specific build commands and conventions for the sample dev project.
---

# Project1 Skill

*(Replace every command below with your own. Agents read this file to learn how to build and test; nothing else tells them.)*

## Build & test commands

| Mode | Command | When |
|---|---|---|
| `rtest --fast` | `<your fast, pure-unit command>` | QA proving RED; Dev's inner loop |
| `rtest --targeted` | `<your single-class command>` | QA's final RED check; Dev's GREEN check |
| `rtest` (incremental) | `<your full suite, cache ON, no clean>` | Dev, ONCE, before `[DONE]` |
| `rtest --full-cold` | `<your clean, no-cache command>` | ONCE per batch, at batch close |

**Hard rule:** never `clean` during a ticket. Cold runs happen once per batch.

## Artifact build

At batch close the SRTL bumps the version, writes one line to `RELEASE_NOTES.md`, builds the artifact and copies it to the project root. See `srtl_profile.md` Function 3.

## Key locations

| Path | Purpose |
|---|---|
| `src/` | production code |
| `test/` | the rtest suite |
| `tickets/` | the agent-to-agent API |
| `locks/` | behaviour locks (one file per area) |