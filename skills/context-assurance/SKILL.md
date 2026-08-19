---
name: context-assurance
description: Optional provider-neutral control for reducing large model-bound evidence/context while preserving authority, recoverability, provenance, and exact fallback. Entroly is the first reference implementation. Never use it for canonical gates, exact required evidence, current-code structural retrieval, or CLI output already owned by another layer.
---

# Optional Context Assurance

Use this skill only when an agent has already completed mandatory Symphony context loading, understands the active work, and has a **large model-bound evidence bundle** whose size materially harms context efficiency.

Entroly (`https://github.com/juyterman1000/entroly`) is the first reference implementation. Symphony depends on the capability, not the product.

> **Select/compress reconstructible context; preserve engineering truth exactly.**

This layer is optional. Symphony must behave correctly and identically when no context-assurance provider is installed, configured, reachable, or controllable from the current host.

## Surface ownership — one optimizer per surface

Do not stack optimizers merely because they are available.

Use this routing order:

```text
candidate input/context
    ↓
canonical / exact / gate evidence?
  yes → RAW / exact; no optimizer
  no
    ↓
historical knowledge lookup?
  yes → semantic-memory provider; verify canonical source
  no
    ↓
current source-code structural retrieval?
  yes → code-intelligence provider; verify live source where required
  no
    ↓
CLI / shell stdout or stderr?
  yes → cli-output-optimization provider (currently RTK)
  no
    ↓
large reconstructible general model-bound evidence bundle?
  yes → context-assurance provider (currently Entroly)
  no  → direct / native context
```

**One optimizer/intelligence layer owns each surface.** In particular:

- do not send RTK-compressed command output through Entroly for another compression pass;
- do not send code-intelligence source/context output through Entroly merely to shrink it again;
- do not use Entroly as a semantic-memory provider merely because it also has memory features;
- do not use Entroly to replace exact canonical reads surfaced by semantic memory or code intelligence;
- do not run canonical/gate evidence through any optimizer.

If classification is ambiguous, use the raw/native path.

## What this layer owns

Context assurance may operate on a large **assembled general-context payload** headed toward a model when all of these hold:

1. the material is not itself a live gate or exact machine contract;
2. it is not a current-source structural/code-retrieval question owned by `code-intelligence`;
3. omission/compression is allowed by the active role/ticket and does not weaken required evidence;
4. omitted material remains reconstructible or recoverable where the provider claims recovery;
5. the provider exposes what it selected/omitted with sufficient receipt/provenance information;
6. the agent can obtain the exact original when the reduced result is insufficient;
7. the transformation does not alter tool definitions, forced tool choices, message ordering, provider semantics, or security boundaries unless the user/project explicitly permits that transformation.

Typical candidates:

- a large set of non-authoritative reference documents assembled for one question;
- many non-code search/find results after mandatory exact context is already known;
- large RAG/document evidence sets where the source documents remain available;
- verbose API/tool payloads that are not current-source code-intelligence results, not already owned by the CLI-output optimizer, and not programmatic gate contracts;
- accumulated background/reference material where only a bounded subset is relevant to the current decision.

This skill is not permission to gather broad context first and compress later. `token-discipline` still requires narrow retrieval. Context assurance is a second-stage optimization only when the **necessary** candidate context remains large.

## CLASS A — never reduce

Keep these raw/exact and outside context assurance:

- `Agent role.md`, role profiles, mandatory skills, and protocol instructions required by init;
- active ticket acceptance criteria, ticket lifecycle/status, claims, `ticketorder.md`, and `.symphony-root`;
- project `MEMORY.md` / `SKILL.md` and architecture/design material when being used as authoritative instruction;
- source/config/schema/API content whose exact form is required for an implementation, edit, review, or decision;
- code-intelligence symbol/source/context output when that output is already the narrow source retrieval for the task;
- Repository Sync / Direct-Remote / optimistic-concurrency evidence;
- exact branch/ref/SHA/divergence state;
- exact patches/diffs required for QA/SRTL/code review;
- test/assertion/error evidence when exact text is material to diagnosis or promotion;
- security/privacy/compliance evidence whose omission can change the conclusion;
- machine-readable output consumed by another step;
- any evidence explicitly required raw by the user, ticket, governing skill, or role profile.

A file is not compressible merely because it is large. Authority and decision use determine classification.

## CLASS B — candidate model-bound context

Potentially reducible only after normal token-discipline has already narrowed retrieval:

- redundant or overlapping background references;
- large document/RAG evidence sets with stable canonical originals;
- broad but necessary non-code search result sets where each hit has a retrievable source;
- repeated explanatory prose from non-authoritative sources;
- large structured tool/API payloads used for human/model interpretation rather than as control-plane contracts or code-intelligence source capsules;
- historical/contextual material whose exact bytes are not the decision contract.

Provider support does not make an item CLASS B. Symphony classification wins.

## Relationship to semantic memory

`skills/semantic-memory/SKILL.md` answers:

> **What historical source is likely relevant?**

This skill answers:

> **Given a still-large set of non-authoritative/reconstructible general evidence, what bounded context should reach the model?**

Normal historical flow:

```text
semantic query
    ↓
small locator set
    ↓
exact-read current canonical source
    ↓
act from canon
```

Do **not** insert context assurance into that path when the exact canonical source is itself required evidence.

If semantic recall leads to many optional supporting references after the authoritative source is already known, that secondary bundle may be a CLASS B candidate.

Entroly's own memory subsystem is intentionally out of scope for this skill. Supermemory remains the first semantic-memory reference adapter unless Symphony explicitly changes that provider separately.

## Relationship to code intelligence

`skills/code-intelligence/SKILL.md` owns current-source structural discovery and narrow symbol retrieval.

Examples:

```text
find Foo::bar + callers + exact symbol source → code-intelligence path
large design/research/document evidence set   → context-assurance path
```

Do not pass an already narrow code-intelligence result through Entroly. The code-intelligence layer has already solved the source-context selection problem for that surface.

If a code-intelligence provider is absent, stale, incomplete, or insufficient, fall back to native targeted code search/read according to that skill. **Do not transfer source-code ownership to Entroly.**

If a task also has a large set of non-code supporting documents, that separate general-document bundle may independently qualify for context assurance.

## Relationship to CLI-output optimization

`skills/cli-output-optimization/SKILL.md` owns shell/CLI output when its provider is available and the command is safe to compress.

Examples:

```text
Gradle/test/git/log stdout → RTK path
large assembled document/RAG context → Entroly path
```

Never use Entroly as a fallback compressor merely because RTK is absent. If a CLI command is classified by the CLI-output policy and RTK is unavailable, run the native command according to that skill. The absence of one provider does not transfer ownership to another.

Likewise, once RTK has produced compressed output, do not feed that output through Entroly simply to reduce it again. If the RTK result is insufficient, recover/read the raw output using the CLI-output policy.

## Provider-neutral capability contract

A context-assurance adapter may use a local process, proxy, MCP, SDK, HTTP service, or another host-specific integration. Canonical Symphony rules must not depend on any transport or vendor hook.

Conceptual capability:

```text
availability() -> AVAILABLE | UNAVAILABLE
assure(payload, task, budget, policy) -> AssuredContext
recover(receipt_or_ref) -> ExactOriginal | UNAVAILABLE
```

`AssuredContext` should expose, when supported:

```text
selected_context
receipt_id / recovery_ref
source/fragment provenance
selected vs omitted accounting
warnings / uncertainty / fallback status
transformation type
```

A provider that cannot expose enough information to determine what happened must not be used for correctness-sensitive engineering context.

## Entroly reference adapter

Entroly is currently the first reference implementation because it provides task-conditioned evidence selection/compression, Context Receipts, content-addressed recovery, and explicit fallback/passthrough behavior.

Use it only when the execution environment actually controls a supported Entroly boundary. Possible environments may expose Entroly through a proxy, MCP, SDK, wrapper, or another configured adapter. Do not assume availability merely because an `entroly` executable or package exists somewhere.

Positive availability means all of the following are known for the current path:

1. the adapter is reachable;
2. this request/context will actually traverse the adapter;
3. recovery/receipt behavior required by this policy is enabled for the selected mode;
4. the project's privacy/security boundary permits the payload;
5. the adapter can fail back to unchanged input when optimization is uncertain or unavailable.

If any item is unknown, treat Entroly as unavailable for that payload.

Do not encode `entroly go`, client-specific installs, provider base-URL rewrites, or vendor plugin setup as mandatory Symphony behavior. Those are environment/deployment choices.

## Recovery and sufficiency

A reduced context must never strand the agent without evidence that was required after all.

```text
assured/reduced context
    ↓
enough evidence for the decision?
  yes → continue
  no  → recover/read exact original evidence
```

If the provider claims exact recovery, verify/use the provider's receipt/recovery reference rather than rerunning expensive upstream work or guessing from the summary.

If recovery is unavailable, corrupted, expired, or unverifiable, stop relying on the reduced representation for that decision and reacquire the exact canonical source through the normal Symphony path.

Never promote a ticket, approve a review, or make a security/correctness conclusion from a reduced representation when the omitted material could change the result.

## Receipts and provenance

Treat receipts as operational evidence about the transformation, not as new project truth.

A useful receipt should make it possible to answer:

- what input/context was considered;
- what was retained, omitted, or transformed;
- what source/fragment each retained item came from;
- what warnings/uncertainty/fallback occurred;
- how to recover exact omitted/original material where recovery is promised.

Do not paste entire receipts into tickets or chat by default. Persist only the small evidence needed to reproduce/justify a decision when the ticket requires it.

Provider token/cost estimates are measurements or models from that provider, not billing truth and not Symphony-wide savings.

## Security and privacy

A context-assurance provider may create indexes, caches, receipts, recovery bundles, or other source-derived artifacts. Treat these as data-bearing state.

Before enabling a provider for private/sensitive work, account for:

- local/remote data boundary;
- retention and secure deletion;
- encryption/access control;
- secrets and PII in payloads;
- prompt-injection/untrusted-document handling;
- receipt/recovery-store location;
- project/client isolation.

Never route credentials, secret manifests, private keys, tokens, or other disallowed material through an optimizer merely to save tokens.

Prefer least-privilege, project-scoped configuration. Provider telemetry or community reporting must not become a Symphony requirement.

## Failure semantics

Context assurance fails open to the **native/raw context path**, except where security itself fails closed.

On optimizer unavailability, timeout, ranking/compression uncertainty, malformed output, missing receipt, or unsupported host:

- preserve/use the original context;
- do not block normal Symphony role execution;
- do not switch automatically to RTK, code intelligence, or semantic memory as a substitute;
- do not warn the user during routine absence unless it materially affects requested diagnostics/measurement.

Security/authorization/path-safety failures are different: do not silently weaken a security control to obtain compression.

## Role guidance

| Role | Good candidates | Boundary |
|---|---|---|
| Architect | large sets of supporting research/design references after canon is loaded | architecture canon and accepted constraints remain exact; code navigation belongs to code intelligence |
| QA / Tester | broad supporting diagnostics/reference material | failure identity, assertions, acceptance criteria, test gates stay exact; current-code impact belongs to code intelligence |
| Dev / Implementer | large non-authoritative docs/tool payloads used for orientation | exact source/config before edit stays raw; source discovery belongs to code intelligence |
| SRTL | supporting evidence bundles during broad investigation | exact diff, tests, gates, architecture and review evidence stay raw; changed-code structure belongs to code intelligence |
| Composer / Critic / Designer | large reference/source collections where originals remain available | authoritative source wording required by task stays exact |
| Orchestrator | generally avoid | route/claim/state/scheduling are live deterministic contracts |

## Decision algorithm

Before using context assurance, answer in order:

1. **Is this exact/canonical/gate evidence?** → raw; stop.
2. **Is this historical lookup?** → semantic-memory policy; stop.
3. **Is this current-source structural/code retrieval?** → code-intelligence policy; stop.
4. **Is this CLI stdout/stderr?** → CLI-output policy; stop.
5. **Has token-discipline already narrowed retrieval?** If no, narrow first.
6. **Is the remaining general model-bound bundle still materially large?** If no, send directly.
7. **Is a positively identified, policy-compliant context-assurance adapter available?** If no, send directly.
8. **Are receipt/provenance and required recovery available?** If no, send directly.
9. Use the assurance path once. Do not chain a second compressor/intelligence layer.
10. If the reduced result is insufficient, recover exact evidence immediately.

## Measurement honesty

Evaluate this layer on representative Symphony work, not vendor headline percentages.

For each pilot payload record only what is useful:

- original context size;
- reduced context size;
- whether any required evidence was omitted;
- whether recovery was needed;
- whether recovery was exact/sufficient;
- extra latency/tool calls;
- whether the final engineering decision matched the raw-context baseline.

A smaller payload that causes another model turn, repeated retrieval, wrong classification, or a raw reread may have no net benefit.

## Vendor neutrality

Symphony's canonical rule is capability-based:

> If a context-assurance provider is available and the payload is CLASS B general context, it may reduce the model-bound evidence once, with provenance/recovery and raw fallback. Otherwise use the native context.

Entroly is the first adapter, not a dependency. A future provider may replace it without changing role lifecycle, `init`, tickets, gates, or the user's normal interaction.
