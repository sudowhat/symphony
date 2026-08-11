# Supermemory Reference Adapter

Current-provider review: 2026-08-12. Re-check linked primary documentation before deployment because API and plan details can change.

## Fit decision

Supermemory is a sound first adapter, not a Symphony dependency. It currently provides document/memory ingestion, scoped semantic search, metadata filters, update/delete operations, MCP, a GitHub connector, and API-compatible hosted/self-hosted deployments.

Use its **document retrieval** surface first for Symphony engineering history. Raw document chunks with canonical metadata are easier to provenance than automatically extracted personal-memory facts. Enable hybrid/extracted-memory results only when every returned fact can be traced to source document IDs and normalized to Symphony's `MemoryHit` contract.

Primary references:

- [API overview](https://supermemory.ai/docs/api-reference/overview)
- [Search](https://supermemory.ai/docs/recall/search)
- [Container tags](https://supermemory.ai/docs/concepts/container-tags)
- [Metadata filtering](https://supermemory.ai/docs/concepts/filtering)
- [Document operations](https://supermemory.ai/docs/ingestion/document-operations)
- [Authentication and scoped keys](https://supermemory.ai/docs/authentication)
- [MCP](https://supermemory.ai/docs/supermemory-mcp/mcp)
- [GitHub connector](https://supermemory.ai/docs/connectors/github)
- [Self-hosting](https://supermemory.ai/docs/self-hosting/overview)
- [Security overview](https://supermemory.ai/docs/overview/security)
- [Open-source repository and MIT license](https://github.com/supermemoryai/supermemory)

## Provider mapping

| Symphony operation | Supermemory mapping |
|---|---|
| `availability` | bounded health/probe against configured base URL; any failure becomes `UNAVAILABLE` |
| `search` | current `/v4/search`; TypeScript `client.search`; Python `client.search.memories`; `searchMode: "documents"` initially |
| scope | one current `containerTag` per request; deterministic `symphony:project:<short-name>` or `symphony:global` |
| metadata filter | `filters` on repository, source kind, status, branch/ref, or other scalar metadata |
| result budget | `limit: 3` by default; hard Symphony cap 5 across all searches for one decision |
| ingest/upsert | `client.add` plus deterministic `customId`; retain provider document ID; update changed content through document update/reprocessing |
| delete | document delete by provider ID/custom identity; verify absence after deletion |
| supersession | update content/metadata and optionally mark prior state; still verify current Git canon on recall |

The current API uses singular `containerTag`; plural `containerTags` is deprecated for new v4 integrations. Search one approved scope at a time.

### Suggested metadata

Store provider-neutral names as scalar metadata where supported:

```json
{
  "symphony_project": "whatdate",
  "symphony_scope": "project:whatdate",
  "repository": "sudowhat/whatdate-android",
  "source_path": "tickets/[DONE]_WD-184_example.md",
  "source_kind": "ticket",
  "source_commit": "<git-commit-sha>",
  "source_content_sha": "<blob-or-content-sha>",
  "source_locator": "WD-184",
  "observed_at": "<ISO-8601>",
  "lifecycle_status": "DONE"
}
```

Do not store credentials, user data, raw secrets, or provider access details in metadata. Validate returned scope/repository/path against the query policy rather than accepting provider-generated labels.

## Initial retrieval profile

Use this conceptual request:

```json
{
  "q": "What prior tickets established recurrence date semantics?",
  "containerTag": "symphony:project:whatdate",
  "searchMode": "documents",
  "limit": 3,
  "filters": {
    "AND": [
      { "key": "repository", "value": "sudowhat/whatdate-android" }
    ]
  }
}
```

Normalize returned results, discard missing/mismatched provenance, then fetch the current source path from the live target ref. Never inject a Supermemory profile or broad `/context` payload during Symphony init.

## Ingestion choice

### Recommended Phase 1: allowlist API ingestion

Use a small event-driven ingestion worker or explicit admin sync that:

1. observes a committed change to an allowlisted canonical artifact;
2. reads it from the approved repository/ref;
3. rejects excluded paths and scans for secrets/sensitive content;
4. writes/upserts the deterministic canonical identity with provenance metadata;
5. records the provider document ID outside source code/secrets;
6. verifies processing completion;
7. deletes the provider record when canon is deleted or access is revoked.

This preserves file-level allowlisting and metadata quality.

### GitHub connector: optional later adapter

The current hosted connector can select repositories, sync documentation/text extensions, and process updates through webhooks plus scheduled/manual sync. It is not the recommended first rollout because:

- it requires a Scale/Enterprise plan;
- OAuth requests `repo`, `user:email`, and `admin:repo_hook` scopes;
- selection is repository-level, while Symphony's ingestion policy is file/path allowlist-oriented;
- it focuses on documentation/text files and does not replace source-code inspection;
- webhook ingestion is delayed/batched and therefore cannot satisfy live-state gates;
- connector removal/document-retention choices require deliberate cleanup handling.

If adopted, approve each repository explicitly, use a dedicated least-privilege account/app where practical, inspect the actual indexed file set, preserve Symphony provenance metadata, and continue every Direct-Remote check unchanged.

## Hosted, MCP, and self-hosted choices

### Hosted API

Use only after approving data residency, retention/deletion, plan, access controls, and private-repository exposure. Mint a container-scoped key rather than distributing an organization master key. Treat published SOC 2/GDPR/HIPAA statements as vendor assurances to be reviewed under the organization's normal security process, not as automatic approval.

### MCP

MCP is an optional transport for hosted Supermemory. Do not require it, auto-install it, or use its automatic context/profile injection. Map only narrow search/list/get operations into the provider contract. Engineering writes should still originate from finalized canonical artifacts, not automatic conversation capture. Use forget/delete for verified removal workflows.

### Self-hosted/local API

The self-hosted server exposes the same Memory API at a configurable base URL and can use local embeddings plus a local/OpenAI-compatible model. It keeps data under its local `.supermemory` directory, which must be included in backup, access-control, retention, and secure-deletion policy.

As of this review, the documented native installer lists macOS and Linux. A Windows Symphony host should use an approved WSL/Linux environment or the hosted API if deployment is chosen; Symphony itself must not require either. Self-hosted MCP/connectors are not listed as equivalent to the hosted platform, so use the API adapter directly.

Self-hosting improves control but creates operational responsibility for model/embedding privacy, patching, encryption, availability, backups, restore tests, retention, access logs, and deletion verification.

## Deployment steps still requiring an owner

Protocol integration requires no credentials. To activate one project later:

1. choose hosted API or approved self-hosted endpoint;
2. approve repository/data classification and exact Phase-1 allowlist;
3. create one least-privilege, container-scoped credential per trust boundary;
4. configure the adapter outside committed Symphony files;
5. seed a small set of finalized documents with full provenance;
6. run every scenario in `verification.md` plus the baseline/retrieval benchmark;
7. enable read-only semantic search for one pilot role/project;
8. add event-driven upsert/delete only after retrieval quality and deletion are proven;
9. expand projects or cross-project scope only through explicit review.

Do not enable automatic conversation capture, full-profile injection, or broad repository connectors as part of the pilot.
