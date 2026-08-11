# Semantic-Memory Verification and Rollout

Use this checklist for every provider and project. Protocol-level checks can be completed without credentials; retrieval-quality and savings claims require a real scoped deployment.

## Required scenarios

| # | Scenario | Required observation |
|---|---|---|
| 1 | No provider configured | Init, role selection, repository/live-state gates, ticket execution, tests, and handoff are unchanged; no warning/block |
| 2 | Provider available | One narrow query returns at most 3 initial hits; the agent opens the cited canonical source before use |
| 3 | Stale memory | Indexed revision differs from live source; agent marks it stale and follows current canon |
| 4 | Conflicting memories | Agent reads both current sources, applies canonical precedence/current state, and does not merge/average claims |
| 5 | Cross-project hit | Result is visibly `ANALOGY`; no foreign constraint becomes local truth without local verification |
| 6 | Missing provenance | Result becomes `UNUSABLE` (or locator-only when repository/path exist); it cannot support action |
| 7 | Provider failure | Timeout, malformed response, auth failure, rate limit, or zero results falls back to native search without blocking |
| 8 | Sensitive content | Excluded fixture is neither ingested nor returned; deletion/forget of an accidentally indexed fixture is verified |
| 9 | Active state | Route, claim, active ticket, branch/ref, source-to-edit, and current test state are read live, never satisfied by recall |
| 10 | Token discipline | Total semantic results stay within 3–5, no broad profile/context injection occurs, and irrelevant history read is lower without correctness loss |

### Minimal fixtures

Use temporary, non-secret canonical fixtures in an approved test repository/container:

- revision A: `recurrence uses UTC`;
- revision B at the same canonical identity: `recurrence uses local civil time`;
- a second conflicting historical note with lower authority;
- one cross-project Android permission lesson;
- one result intentionally missing `source_revision`;
- one fake secret marker that the ingestion allowlist/scanner must reject.

Delete all fixtures after the test and verify provider removal. Never use real credentials or user data as a deletion test.

## Practical Symphony benchmark

Run the same questions twice against a stable repository commit:

1. What previous tickets established recurrence-date semantics?
2. What Android permission problems have we solved before?
3. What prior SRTL findings exist around notification suppression?
4. Have other approved Symphony projects implemented a similar feature?

### Baseline: native discovery

For each question record:

```text
question
repository/ref
search commands/tool calls
candidate files listed
files/ranges actually read
input bytes or tokens loaded (use bytes when token counts are unavailable)
elapsed time
canonical sources and conclusion
```

Use token-discipline: search before broad reads. The baseline is not an intentionally wasteful repository dump.

### Semantic-assisted discovery

Reset to the same question/ref and record:

```text
query and scope
provider/adapter version
results returned and scores
valid / stale / cross-project / false-positive counts
canonical files/ranges subsequently read
input bytes or tokens loaded, including semantic results
elapsed time/tool calls
canonical sources and conclusion
```

### Acceptance

Pass only when:

- both paths reach the same correct canonical evidence/conclusion;
- every used semantic hit has provenance and a live exact read;
- no active/live state is sourced from memory;
- provider failure reproduces the native path;
- semantic-assisted discovery reduces irrelevant files/context for most pilot questions;
- additional provider calls/results do not cost more context than they save;
- false positives, stale hits, and cross-project results are labeled and bounded.

Do not set or publish a Symphony savings percentage until measured. Report raw per-question data and the median context difference; preserve outliers and failures.

## Phased rollout

### Phase 0 — protocol only

- Merge provider-neutral policy and reference adapter documentation.
- Keep all projects fully functional without a provider.

### Phase 1 — read-only pilot

- Choose one project and Architect/SRTL first.
- Ingest architecture/design, `MEMORY.md`, `SKILL.md`, completed tickets, regression/postmortem docs.
- Use document search, same-project scope, limit 3.
- Run the benchmark and all failure scenarios.

### Phase 2 — curated expansion

- Add significant QA/SRTL findings and curated commit/migration metadata.
- Pilot QA/Dev retrieval.
- Add explicitly reviewed global/cross-project lessons.

### Phase 3 — automated lifecycle

- Add event-driven upsert/delete after commits.
- Add scheduled reconciliation for missed events.
- Monitor stale rate, false positives, deletion, access boundaries, cost, latency, and context saved.

Pause or remove the adapter if correctness, isolation, deletion, or context-cost gates fail. Symphony continues through native canonical retrieval.

## Current integration status

- Provider-neutral policy: testable without credentials.
- Supermemory capability mapping: documented from current primary sources.
- Live provider recall, ingestion, deletion, and savings benchmark: **not run until an owner chooses a deployment, approves an allowlist, and supplies scoped infrastructure outside Git**.

This pending runtime step does not block Symphony or the protocol integration; it blocks only claims that the provider is deployed or has achieved measured savings.
