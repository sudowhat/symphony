---
name: cli-output-optimization
description: Optional, vendor-neutral policy for reducing noisy CLI/build/test output before it enters model context (currently RTK). Never required. Defines which commands may be compressed, which must stay raw, and how to recover full output without rerunning work.
---

# Optional CLI-Output Optimization

This skill is an **optional execution accelerator**. It is never required.

Canonical binding rules live in `skills/token-discipline/SKILL.md` §"Optional CLI-output
compression". This file is the operational detail, loaded on demand — only by an agent that has
already positively detected the tool and is about to run a high-volume command.

> **Symphony must behave identically whether the tool is installed, absent, unsupported by the
> host, or unavailable to a cloud/direct-repository agent.** No `init`, onboarding, role, ticket,
> test, Git, or orchestration path may depend on it.

The current implementation is RTK (Rust Token Killer, `https://github.com/rtk-ai/rtk`). Everything
below is written as policy about *a* compressor; if RTK is replaced, only this file changes.

## Layer position

```
Layer 1  token-discipline   retrieve only necessary context
Layer 2  THIS SKILL         reduce noisy CLI output (optional, when available and safe)
Layer 3  token-discipline   keep narration/handoffs compact
Layer 4  durable artifacts  don't regenerate context across agents
```

Layer 2 is the only optional layer. Layers 1, 3, and 4 are mandatory and unaffected by it.

## Detection — positive identification required

**A name match on `PATH` is NOT detection.** At least one unrelated npm package publishes the same
`rtk` binary name (`cliffano/rtk`, a release/version-tagging tool that writes changelogs and
**creates Git tags**). Routing a Symphony command into it would be a repository-integrity incident,
not a missed optimization.

Treat the tool as available only when **both** hold:

1. `rtk --version` exits 0; and
2. the help surface identifies this specific tool (e.g. `rtk --help` advertises RTK-specific
   surface such as `gain`, `telemetry`, or `--ultra-compact`).

If identification is ambiguous → **treat as absent**. Do not probe further, do not guess.

```powershell
# One-time per session; cache the boolean, do not re-probe per command.
$rtk = $false
if (Get-Command rtk -ErrorAction SilentlyContinue) {
    $v = (rtk --version 2>&1); $h = (rtk --help 2>&1) -join "`n"
    $rtk = ($LASTEXITCODE -eq 0) -and ($h -match 'ultra-compact|\bgain\b|telemetry')
}
```

When absent: run the native command. **Do not warn, do not block, do not alter workflow
semantics, do not mention it.** Absence is the normal case.

## The decision

```
if tool available AND command is CLASS B:
    compressed invocation
else:
    native command
```

Never invert the default. An unclassified command is CLASS A until deliberately classified.

## CLASS A — raw, never compressed

Exact output/content is part of correctness, synchronization, review, or protocol behavior:

- `Agent role.md`, role profiles, `skills/global-skill`, `skills/token-discipline`,
  `skills/agent-symphony`, and this file
- project `MEMORY.md`, project `SKILL.md`, architecture/design docs used as authoritative instruction
- tickets, `.claims/`, `ticketorder.md`, `.symphony-root`
- source reads immediately preceding a modification, review, or decision
- **Repository Sync Gate evidence** — in particular `git status --porcelain=v1 --untracked-files=all`
- Direct-Remote Gate evidence, optimistic-concurrency SHAs/revisions, remote-movement checks,
  branch-divergence decisions, commit/SHA verification
- exact patches/diffs for SRTL or code review
- repository-integrity checks (e.g. the zero-byte blob scan)
- security/privacy-sensitive evidence
- test evidence whenever compressed output is insufficient to decide
- anything the user or another agent asked for raw

**The porcelain status gate must never pass through a compact-status transformation.** Its exact
bytes are consumed as a gate, not read as prose. The same rule extends to *any* command whose exact
output or exit semantics are used programmatically as a gate, unless equivalence has been explicitly
proven and recorded.

This is demonstrated, not precautionary. On a **clean** tree the raw gate emits **zero bytes** — the
pass condition the gate tests for — while `rtk git status` emits the prose line
`clean — nothing to commit`. Since the gate treats *any* output as a dirty worktree, substituting
the compact form inverts it: every clean tree would report `REPO_DIRTY`. Compact status is a
human summary, never a gate contract.

## CLASS B — compressible

High-volume, repetitive, diagnostic, reconstructible, and not itself an integrity contract:

- Gradle build/test/lint/dependencies; Android/KMP compilation
- npm/pnpm builds; Jest/Vitest/Playwright; pytest and similar runners
- Docker/Kubernetes/OpenShift listings and logs; repetitive application logs
- non-authoritative Git orientation (exploratory `log`, history/branch summaries, push progress)
- broad grep/find/tree/listing **during exploration only**
- other large diagnostic output where raw recovery exists

Membership in the tool's supported-command list does **not** by itself make a command CLASS B.

## Recovery is mandatory

A compressed result must never strand an agent without the omitted evidence.

```
compressed signal first
    ↓
enough evidence to decide?
  yes → continue
  no  → read the persisted raw output
```

RTK persists unfiltered output on failure (default `[tee] mode = "failures"`) and prints a
`[full output: <path>]` hint. **Read that file. Do not rerun an expensive build or test suite merely
because the output was compressed** — the raw bytes are already on disk.

If a compressed result is ambiguous, obtain the raw evidence immediately. Never guess, and never
report a result whose evidence you could not read.

Raw escape hatches, in order of preference: the tee file → `rtk proxy <cmd>` (tracked passthrough,
no filtering) → the native command.

Retention is **count-bounded, not time-bounded** (oldest files pruned beyond `max_files`). A tee file
is not durable evidence: if raw output matters to a ticket, copy the relevant excerpt into the
ticket, which is the durable artifact.

## Gradle / Android / KMP

Verified behavior of the Gradle filter (`src/cmds/jvm/gradlew_cmd.rs`): recognizes `./gradlew`,
`gradlew.bat`, and `gradle`; routes Build / Test / ConnectedTest / Lint / Dependencies; keeps
`BUILD SUCCESSFUL` / `BUILD FAILED` lines, actionable errors, and the **first user-code stack frame**
while dropping framework frames; preserves Android lint violation code context; strips
`INSTRUMENTATION_STATUS` noise from connected-test runs.

**This does not change Symphony's long-running-command policy.** Output-size reduction and
background execution are orthogonal concerns. `global-skill` §"Long-Running Commands" still governs
anything expected to exceed ~30s: launch in background, redirect to a log, poll in short calls. Do
not regress background execution or user responsiveness to gain compression.

## rtest

**Do not rewrite `rtest.bat` or any project test wrapper to embed a compressor.** Those wrappers are
project gates; changing them changes the gate for every agent and every vendor, including hosts
where the tool is absent. That would violate the optional-accelerator rule.

Exit-code propagation has been verified in RTK's runner: filtered, streamed, and passthrough modes
all return the wrapped process's exit code. That makes an *agent-side* compressed invocation
acceptable where a project documents one. Any such pattern is optional and must be recorded in the
project's own `SKILL.md`, never assumed.

Test evidence remains CLASS A whenever the compressed form is insufficient to identify the failing
test, its assertion, or its cause. Preserve failure identity, assertion text, and exit status; when
in doubt, read the tee file.

## Git

Compressed: exploratory `log`, history/branch orientation, routine push progress, non-authoritative
diff skimming.

Raw, always: the Repository Sync Gate, remote-movement checks, divergence decisions, exact porcelain
status, exact patch/review diffs, commit/SHA verification — and anything where compaction could hide
a fact a Symphony invariant depends on.

For exact diffs prefer native Git.

## Source and file reading

Compressed/structural reads (`rtk read`, `rtk smart`, compressed grep/find) are for **orientation
only**.

```
compressed search for discovery
    ↓
identify the relevant file and range
    ↓
exact read before any decision or edit
```

Some compressed search commands shell out to external binaries (`rtk grep` requires `grep` on
`PATH`) and simply fail on hosts that lack them — Windows/PowerShell hosts typically do. A failed
compressed search is not a fallback signal to investigate; use the host's native search
(`rg`, `Select-String`, indexed search) and move on.

Never substitute a structural or signature-level summary for an authoritative read when following a
ticket, implementing, reviewing, reading protocol rules, verifying exact behavior, or writing based
on the file. This reinforces token-discipline §"Read discipline": start narrow, expand when evidence
requires it — narrower is not the same as lossier.

## Configuration and privacy (recommendation, not protocol)

Recommended for Symphony-managed and local development environments:

```toml
[telemetry]
enabled = false
```

Equivalent: `rtk telemetry disable`, or `RTK_TELEMETRY_DISABLED=1`.

Tee writes **raw command output to disk**. On failure that can include database contents, decrypted
fixtures, tokens, or personal data — a live concern for privacy-sensitive projects whose test
failures may print user entry content. Review `[tee] mode` and `max_files` against the project's
privacy posture, and never copy secret-bearing raw output into a ticket or commit.

None of this is a Symphony dependency. It is the privacy-preserving configuration recommended *if*
the tool is used.

## Custom filters

Do **not** author project-local filters to chase token percentages.

A custom filter is justified only when all hold: a demonstrated high-volume command; inadequate
built-in handling; formally preserved diagnostic signal; existing raw recovery; and no weakening of
tests or guards.

Treat a project-local filter config as **executable, behavior-affecting tooling configuration**
subject to normal review. Do not auto-trust arbitrary filters from any source.

## Vendor neutrality

Symphony states only the desired behavior:

> If a CLI-output compressor is available, prefer it for approved CLASS B commands. Otherwise run
> the native command.

Symphony **never** encodes vendor integration mechanics — no agent-specific `init` variants, hooks,
plugins, IDE rules, or vendor config files. Installing native hooks is an individual user's
environment choice and sits outside the protocol.

Cloud/direct-repository agents using GitHub APIs or connector-native reads gain little or nothing
here; do not force adoption. `token-discipline` applies to them unchanged.

## Measurement honesty

Do not claim a Symphony-wide token saving. The upstream project's percentages describe **reduction
in shell-command output**, not total model cost — output size dilutes through conversation history
and output tokens.

If measuring, record per representative command: raw size, compressed size, reduction %, whether any
essential evidence was lost, and whether raw recovery was needed. A reduction that forced a raw
re-read is not a saving. The objective is useful-context reduction, not a marketing percentage.

Observed baseline (RTK 0.45.0, Windows x86_64, bytes of captured output):

| Command | Raw | Compressed | Reduction |
|---|---|---|---|
| `git log -n 30` | 9,156 | 3,090 | 66% |
| `git diff HEAD~1` | 15,336 | 8,187 | 47% |
| `gradlew --version` | 455 | 47 | 90% |
| `git status` (illustrative — gate stays raw) | 104 | 49 | 53% |

Real-world Git reduction sits nearer 50–65% than the headline figure. Exit-code propagation was
verified empirically across `rtk err` / `rtk test` / `rtk proxy` (7→7, 3→3, 5→5, 0→0). A full Gradle
build/test measurement has not been taken; `rtk grep` is unavailable on this host class (see above).
