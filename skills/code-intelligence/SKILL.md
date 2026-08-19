---
name: code-intelligence
description: Optional provider-neutral structural source-code retrieval for Symphony. Use after canonical init when current-code navigation, symbol lookup, dependency/call analysis, impact analysis, or targeted source retrieval would avoid broad file reads. jCodeMunch is the first reference provider. Never use indexed state as live Git/gate truth or as a substitute for an exact current-source read required for an edit or review.
---

# Optional Code Intelligence

Use code intelligence as an optional **current-source navigation and structural retrieval** layer.

jCodeMunch MCP (`https://github.com/jgravelle/jcodemunch-mcp`) is the first reference provider. Symphony depends on the capability, not the product, MCP, hooks, or any one vendor/client.

> **Index broadly enough to navigate; retrieve narrowly enough to reason; verify live source before changing it.**

This layer is optional. Symphony must work correctly when no code-intelligence provider is installed, configured, licensed, indexed, fresh, reachable, or supported by the current host.

## What this layer owns

Use code intelligence only for questions about the **current codebase structure or source location**, such as:

- where a symbol is defined;
- which class/function/method implements a behavior;
- file/module outlines and signatures;
- callers/callees, imports/importers, inheritance/implementation relationships;
- blast radius and dependency/coupling questions;
- changed-symbol mapping;
- hotspots, complexity, dead-code/test-reachability leads;
- targeted retrieval of one symbol or a small source bundle;
- code-oriented task context assembled from indexed symbols.

Do not use it for:

- historical engineering decisions or old tickets — use `semantic-memory`/canonical history;
- shell/build/test/log stdout/stderr — use `cli-output-optimization` when applicable;
- general document/RAG/reference compression — use `context-assurance` when applicable;
- live route, claim, ticket status, branch/ref/SHA, Repository Sync, Direct-Remote, or other gate state;
- protocol, ticket, `MEMORY.md`, `SKILL.md`, architecture, or other authoritative non-code instructions required by Symphony init/workflow.

## Surface ownership

Keep the layers non-overlapping:

```text
exact / canonical / gate evidence        -> RAW / native exact path
historical engineering knowledge          -> semantic-memory provider
current source-code structural retrieval  -> code-intelligence provider
CLI / shell stdout or stderr              -> cli-output-optimization provider
large reconstructible general context     -> context-assurance provider
anything else                             -> native/direct path
```

**One intelligence/optimization layer owns each surface.**

In particular:

- do not pass jCodeMunch source/context output through Entroly merely to shrink it again;
- do not treat jCodeMunch's optional semantic/context features as Symphony's historical-memory provider;
- do not use RTK to compact code-intelligence results;
- do not let indexed code replace exact live source when Symphony requires an exact current read;
- do not use a code index to satisfy a live repository/gate check.

If classification is ambiguous, use the native exact path.

## Authority model

The **current repository/worktree/ref remains authoritative**.

A code-intelligence index can be authoritative only about what it can prove it indexed at a known revision; it cannot prove that revision is still the active live state.

Treat provider results as one of:

| Class | Condition | Permitted use |
|---|---|---|
| `FRESH_LOCATOR` | Provider proves current-enough index/coverage and identifies source path/symbol | Navigation, structural reasoning, targeted follow-up |
| `FRESH_SOURCE` | Provider returns exact source for a proven-current indexed revision | Read/orientation; exact live verification still required where governing Symphony rules require it |
| `STALE_LOCATOR` | Index/revision is stale or freshness uncertain | Locator only; verify through live/native source/search |
| `INCOMPLETE` | Coverage excludes/withholds relevant files or provider cannot support absence claim | Never conclude absence/completeness; fall back to native search/read |
| `UNUSABLE` | Missing provenance, source path, or trustworthy freshness/coverage status where needed | Ignore for engineering action |

A confidence score ranks retrieval relevance, not authority.

## Exact-source rule

Code intelligence reduces **where and how much to read**, not the standard of evidence.

Preferred flow:

```text
structural/symbol query
    ↓
identify exact symbol/path/range
    ↓
retrieve narrow indexed source for orientation if useful
    ↓
need to edit/review/approve based on exact current bytes?
  yes -> exact-read current canonical source/range through the live repository/worktree path
  no  -> continue from verified-fresh indexed result when sufficient
```

Before modifying source, reviewing an exact patch, approving a correctness conclusion, or relying on a value whose bytes materially matter, obey the governing Symphony exact-read/live-state rules. Do not assume an index watcher made cached source equivalent to the live worktree.

For a targeted edit this still saves tokens: the index identifies the symbol and small range, then the agent exact-reads that range instead of opening a 1,000-line file.

## Freshness and coverage are mandatory signals

Never interpret an indexed result without asking whether it is fresh enough and whether the relevant corpus was covered.

Required behavior:

1. Prefer providers that expose index revision/HEAD, file freshness, coverage and withheld/skipped-file state.
2. A stale index may locate a candidate but cannot settle current absence, impact, caller/reference, or implementation claims.
3. Uncommitted edits require special care. If the provider marks a file edited/uncommitted, stale, degraded, or source-missing, use live/native reads and searches for any decision affected by that file.
4. Absence claims (`symbol does not exist`, `no callers`, `safe to delete`) require both adequate freshness and complete-enough coverage for the claim. Otherwise report the limitation and fall back.
5. Reindex/watch is an optimization, not a gate. Do not block Symphony merely because the index is stale; native tools are the fallback.
6. Do not repeatedly reindex a large repository merely for reassurance. Reindex only when freshness evidence says it is needed and the operation is justified.

## Provider-neutral capability contract

An adapter may use MCP, a local service, an indexed API, an IDE integration, or another mechanism. Canonical Symphony rules must not depend on the transport.

Conceptual capabilities when available:

```text
availability() -> AVAILABLE | UNAVAILABLE
index_status(scope) -> revision + freshness + coverage
repo_map(scope, budget) -> structural overview
search_symbols(query, filters, limit) -> SymbolHit[]
get_symbol_source(identity) -> source + provenance + freshness
structural_query(kind, identity/query, options) -> attributed result
```

An ordinary role agent needs read/query operations only. Index creation, watchers, hooks, tuning, telemetry, or provider configuration are environment/deployment responsibilities unless the user explicitly asks the role to manage them.

A useful normalized result should expose as much as the provider supports:

```text
repository / project
source_path
symbol identity / qualified name
symbol kind/signature
source range or byte offsets
indexed revision
file/index freshness
coverage / withheld status for absence claims
source/provenance channel
confidence/relevance score (ranking only)
```

## jCodeMunch reference provider

jCodeMunch is a strong first adapter because it uses tree-sitter indexing for symbol-aware retrieval and exposes structural code queries rather than relying on grep-and-whole-file reads.

Prefer a **small core surface first** instead of spraying the model with a large tool catalog:

### Orientation/discovery

- repository map / file outline;
- symbol search;
- exact symbol source retrieval.

### Structural follow-up when the task requires it

- importers/references;
- blast radius;
- call hierarchy;
- class/implementation hierarchy;
- changed symbols;
- edit/delete safety or PR-risk queries when their evidence is relevant.

### Composite code context

A task-context or ranked-code-context function may be used when it replaces several independent code-navigation calls and stays within the active task's code surface. Treat its result as **code-intelligence output**, not as Entroly input. If exact current code is required after the capsule identifies the relevant symbols, exact-read those symbols from live canon.

Do not enable or invoke dozens of specialist tools merely because the server exposes them. Tool schemas and unused capabilities also consume context. Prefer provider tiers/gating or the smallest supported tool surface when the environment allows it.

## C/C++ and compiler-derived evidence

For large C/C++ repositories, AST navigation can be especially valuable because grep does not model references, call relationships, inheritance, or blast radius reliably.

When the project already produces compiler/language-index evidence such as SCIP (for example via `scip-clang`) and the provider can import it, compiler-verified reference/implementation edges may strengthen structural queries.

Treat compiler-derived edges as evidence for the indexed revision, not proof of current worktree freshness. A stale compiler index must be labeled stale and verified against current source before a live decision.

Do not require SCIP generation merely to use code intelligence; it is an optional accuracy enhancement whose build/runtime cost must be justified by the project.

## Large-repository policy

Large repositories are the main beneficiary but also the main place where silent coverage gaps are dangerous.

Before relying on structural absence/impact results:

- verify what directories/languages/files were indexed;
- inspect provider file-count/size/exclusion limits and project ignore rules;
- treat skipped/withheld files differently from intentionally excluded generated/vendor content;
- prefer module- or project-scoped indexes when a full monorepo index is operationally expensive;
- raise provider limits only after considering storage, refresh time and privacy—not merely to maximize coverage.

Do not hard-code an upstream provider's default file-count or file-size limit into Symphony policy. Provider defaults change; inspect the installed/reference version during deployment.

## Security and source-cache handling

A source index is another copy/derivative of proprietary source and must be protected accordingly.

For jCodeMunch specifically, current upstream documentation describes local index/cache state under `~/.code-index/` by default and warns that cached bodies are effectively another copy of indexed source. Treat that directory with the same confidentiality expectations as the repository.

Before enabling any provider for private/company repositories:

- approve the local/remote data boundary;
- protect index/cache permissions and backups;
- configure exclusions for secrets, credentials, generated/private data and disallowed paths;
- understand any response-redaction exceptions for raw source tools;
- do not assume a source-code file cannot contain a hardcoded secret;
- use normal repository secret scanning/pre-commit/CI controls independently of the indexer;
- review telemetry, watchers, startup services and agent hooks before enabling them.

Never let a convenience `init` command silently rewrite Symphony's canonical agent files, install hooks, or change vendor configuration as a protocol requirement. Such setup is an explicit environment action outside Symphony core.

## Licensing / deployment boundary

Provider licensing is an environment constraint, not something Symphony bypasses.

At the time this reference adapter was added, jCodeMunch uses a dual-use license: non-commercial use is free under its terms, while commercial/for-profit use requires a paid commercial license. **Verify the current upstream license before use**, especially for office/company repositories.

Symphony may document/integrate the capability without redistributing or automatically installing the provider. Do not bundle jCodeMunch code into Symphony or enable it on a commercial project without the required license/approval.

## Failure semantics

Code intelligence fails open to native repository tools.

When the provider is absent, unlicensed for the environment, unreachable, stale, incomplete, malformed, or unsupported:

```text
indexed path unavailable/insufficient
    ↓
native targeted search (rg/grep/indexed host search/etc.)
    ↓
exact narrow source read
```

Do not block role execution, do not substitute Entroly/RTK/Supermemory for the missing code-index surface, and do not warn during routine provider absence unless the user asked about optimization state or the limitation materially affects a conclusion.

If a provider result conflicts with live source, live source wins immediately.

## Role guidance

| Role | High-value use | Boundary |
|---|---|---|
| Architect | repo/module maps, symbol/dependency topology, implementation discovery, change blast radius | Current architecture/ticket remains authoritative; verify source used for design decisions |
| QA / Tester | changed-symbol impact, caller paths, untested-symbol leads, test-localization clues | Test requirements/results and current source remain canonical |
| Dev / Implementer | symbol lookup, exact target localization, callers/callees, implementation/refactor impact | Exact-read live source before modification where required; no architecture expansion |
| SRTL | changed-symbol/risk orientation, blast radius, reference/call verification leads | Exact diff/current source/tests decide approval |
| Designer / Composer / Critic | normally little use unless the project contains code-driven behavior relevant to their task | Do not load the tool merely because it exists |
| Orchestrator | none for route/claim/state/scheduling | Use deterministic live state only |

## Measurement honesty

Do not repeat jCodeMunch's headline token-savings numbers as Symphony savings.

Benchmark representative project tasks against the existing narrow native workflow, recording:

- files/ranges or symbols retrieved;
- model-bound tokens/bytes if measurable;
- number of tool round trips;
- index/reindex cost and latency;
- stale/incomplete fallbacks;
- whether exact live rereads were needed;
- correctness of symbol/impact/absence conclusions;
- whether the same task completed with fewer irrelevant reads.

A structural query that prevents ten broad reads is valuable even if its own response is not tiny. Conversely, a large tool schema or repeated indexing that outweighs retrieval savings is not an optimization.

## Vendor neutrality

Symphony's canonical rule is capability-based:

> If a fresh-enough code-intelligence provider is available, prefer it for current-source structural discovery and narrow symbol retrieval; verify exact live source whenever governing evidence rules require it. Otherwise use native targeted search/read.

jCodeMunch is the first adapter, not a dependency. A future source indexer, LSP/SCIP service, code graph, or repository intelligence provider may replace it without changing `init`, role lifecycle, tickets, gates, or the user's normal interaction.
