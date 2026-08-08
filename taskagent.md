# role: model-ring   (task → agent/model rotation; leftmost = next; Orchestrator moves head → tail after each dispatch; size never grows)
# Tuned 2026-07-27 for the POSTURE/DURATION batch (260-270): FOUR schema migrations (265,267,268,269) + one large UI ticket (270).
# Rationale: migrations and lock-bearing tests are where a cheap model costs more than it saves; UI-lane work is spec-following.
# Restore the previous rings from taskagent.prev-batch.bak once this batch is [DONE].
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
