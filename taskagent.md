# Model rotation rings — `role: slugA, slugB, …` (leftmost = next; the Orchestrator moves head→tail after each dispatch, so this file never grows). Slugs are defined in `orchestrator model map.md`.
# Tune per batch: put a stronger model first for migration/lock-heavy work, a cheaper one first for spec-following UI work. Keep a backup of the previous rings when you retune.
architect: opus4.8-max, fable5-high
qa: sonnet5-high, grok-medium
dev: sonnet5-high, sonnet5-max
srtl: opus4.8-max
composer: sonnet5-medium, grok-medium
critic: opus4.8-medium, sonnet5-high
designer: opus4.8-max, grok-high
tester: grok-medium, sonnet5-low
implementer: sonnet5-max, opus4.8-medium, grok-high
orchestrator: haiku4.5-medium
default: sonnet5-medium
