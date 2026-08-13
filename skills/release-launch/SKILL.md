---
name: release-launch
description: Prepare, verify, and hand off release artifacts without leaking signing secrets or bypassing quality gates. Use for Android App Bundles/APKs, internal testing, release signing, version/package/SDK audits, artifact checksums and certificate fingerprints, bundle-size/privacy audits, Play Console guidance, and reusable launch checklists.
---

# Release Launch

Prepare a reproducible release artifact from a clean, reviewed project state. Keep store publication human-gated and credentials local.

For Android or Google Play work, read `references/android-play.md` before acting.

## Authority and boundaries

- Own release readiness, release-only configuration, signing integration, build execution, artifact verification, launch records, and store handoff instructions.
- Do not alter product behavior, tests, architecture, or guards. Route product/test defects to SRTL through the project workflow.
- Make only narrow release-build corrections whose effect is confined to packaging, signing, dependency compatibility, version metadata, or release tooling. Re-run all gates after a correction.
- Never upload, publish, start a rollout, invite testers, change store policy answers, or rotate/revoke keys without explicit user authorization for that exact action.
- Never print, echo, log, commit, or request passwords in chat. Never commit keystores or credential properties.
- Treat an ignored directory as untracked, not encrypted. Keep secrets outside the repository.

## Launch workflow

### 1. Establish live state

1. Resolve the canonical project path from `Agent role.md`.
2. Verify `.symphony-root`.
3. Pass the Repository Sync or Direct-Remote Gate before reading project state.
4. Read project `MEMORY.md`, `SKILL.md`, release documentation, active tickets, route, and launch checklist.
5. Record the exact branch and commit intended for release.

Stop on dirtiness, divergence, stale work, open CANNOT state, or an unexplained unreviewed change.

### 2. Audit release identity

Verify from effective build output—not assumptions:

- application/package/bundle identifier;
- version code/build number and version name;
- minimum and target SDK/platform versions;
- real release task/scheme and output path;
- release channel and intended audience;
- current commit suitability, including required review markers and test state.

Do not silently bump versions. Ask when the intended release version is not already authorized.

### 3. Verify quality gates

Run the project-prescribed release checks in the cheapest valid order:

1. targeted configuration/task discovery;
2. project regression suite;
3. release lint/static analysis;
4. release compilation/package task;
5. host/device phases where available.

Report `HOST_SKIPPED` precisely. It is never PASS or certification.

Run commands expected to exceed ten seconds in the background per `global-skill`.

### 4. Configure signing securely

If no permanent upload/distribution key exists, stop and guide the user through interactive creation. The user enters secrets directly in their terminal or secret manager.

Signing configuration must read from one of:

- environment variables;
- an ignored local properties file; or
- an approved external secret manager.

Environment variables take precedence over local files. Fail release builds when required signing values are missing; never fall back to debug or unsigned output.

Verify all credential files are ignored and absent from `git status`, `git ls-files`, build logs, and committed history before proceeding.

### 5. Build and verify the artifact

Build the real release target. Then verify and record:

- exact absolute artifact path and byte size;
- SHA-256 digest;
- package/bundle identifier;
- version code/build number and version name;
- target SDK/platform;
- signing status;
- signing certificate SHA-256 fingerprint;
- source branch and commit;
- tests/lint/build results and host-skipped phases.

A build command returning zero is necessary but insufficient. Inspect the artifact or generated release manifest and verify the signature independently.

### 6. Audit size and bundled data

When size is questioned, inspect the artifact rather than guessing:

- group compressed size by bytecode, native libraries, resources, assets, and metadata;
- list the largest entries;
- search for bundled databases, recordings, archives, seed data, or user-derived files;
- distinguish bundle upload size from device-specific download size;
- identify safe optimization opportunities without weakening functionality or guards.

Never claim that an artifact contains no user data until its entries and packaged assets have been inspected.

### 7. Hand off to the store

Before any external action, tell the user exactly what remains human-controlled. For Play internal testing:

1. confirm Play App Signing state;
2. upload the verified AAB only after user authorization;
3. confirm package/version/certificate acceptance;
4. review warnings and release notes;
5. preview and confirm;
6. start rollout only after explicit user approval;
7. install through Play on a physical device and run the project smoke test.

Do not treat upload acceptance as runtime validation.

## Launch checklist record

Maintain `<project>/LAUNCH_CHECKLIST.md` as the live, reusable evidence ledger. Keep secrets out. Use these states:

- `[ ]` not started;
- `[~]` in progress or awaiting human action;
- `[x]` verified with evidence;
- `[!]` blocked, with exact cause;
- `[S]` host/device skipped, with exact requirement.

For each completed line include the date, commit, command/result or artifact fact. Record reusable lessons under `## Lessons for future launches`; keep project-specific values in the project checklist, not this shared skill.

## Completion

A prepared release ends with a clean/current repository and this concise handoff:

```text
PREPARED: <channel> <package> <version/build>
ARTIFACT: <absolute path>
SHA256: <digest>
SIGNER_SHA256: <fingerprint>
SOURCE: <branch> <commit>
VERIFIED: <tests/lint/build>
SKIPPED: <host/device phases, if any>
NEXT: <single human-controlled action>
```

Never mark `PUBLISHED` from a prepared artifact. Publication requires observed store acceptance and explicit user authorization.
