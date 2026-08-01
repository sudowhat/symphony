# Orchestrator Model Map — Legend  (STATIC · user reference)

Defines what each model **slug** launches as. The live **rotation rings** per role live in
`taskagent.md` (leftmost = next; the Orchestrator rotates head → tail in place, so that file
never grows). This file changes only when you add a new model slug.

## Legend (slug → launch)

| Slug | Provider | Launch model | Effort |
|---|---|---|---|
| `fable5-high` | anthropic | `fable` / claude-fable-5 | high |
| `opus4.8-max` | anthropic | `opus` / claude-opus-4-8 | max |
| `opus4.8-medium` | anthropic | `opus` / claude-opus-4-8 | medium |
| `sonnet5-low` | anthropic | `sonnet` / claude-sonnet-5 | low |
| `sonnet5-medium` | anthropic | `sonnet` / claude-sonnet-5 | medium |
| `sonnet5-high` | anthropic | `sonnet` / claude-sonnet-5 | high |
| `sonnet5-max` | anthropic | `sonnet` / claude-sonnet-5 | max |
| `grok-medium` | xai | ⟨your Grok model id⟩ | medium\* |
| `grok-high` | xai | ⟨your Grok model id⟩ | high\* |
| `haiku4.5-medium` | anthropic | `haiku` / claude-haiku-4-5-20251001 | medium |
| `flash3.6-medium` | google | `flash` / gemini-3.6-flash | medium |

**Slug rule:** `<model><ver>-<effort>` — no spaces, no colons, no commas (colon separates role from ring, comma separates slugs in `taskagent.md`).

Notes:
- Anthropic launch → `claude --model <alias> --effort <effort>` (e.g. `claude --model opus --effort max`).
- \*Grok: fill your xAI CLI/model id; Grok's effort names differ — the adapter maps `medium`/`high` to Grok's own reasoning setting.
- `sonnet5-max`: if Sonnet 5 doesn't support `max`, Claude Code auto-falls back to its highest supported level (no crash).
