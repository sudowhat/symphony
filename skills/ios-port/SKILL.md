---
name: ios-port
description: Audit an existing Android or Kotlin Multiplatform Symphony project, plan the minimum same-repository iOS migration, create dependency-ordered QA/Dev tickets, extend rtest across common/Android/iOS targets, and stop at IOS_READY_FOR_MANUAL_TEST. Load only for `init <project> srtl ios`.
---

# iOS Port

Convert an existing mobile project into one Android+iOS product without duplicating behavior or weakening Android coverage. This SRTL mode plans and routes the migration; Launcher later owns distributable artifacts and store handoff.

## Authority and terminal boundary

- `init <project> srtl ios` authorizes a live repository audit, a complete port plan, ticket creation, route updates, and concise architecture-state documentation.
- The planning pass does not implement production code. After publishing an active batch, SRTL enters its normal review loop while QA and Dev execute it.
- Do not upload, publish, invite testers, answer Apple policy declarations, or create/rotate signing credentials.
- End the implemented migration at `IOS_READY_FOR_MANUAL_TEST`: automated shared, Android, iOS, and simulator gates are green; physical-iPhone acceptance remains human.
- TestFlight/App Store preparation remains `init <project> launcher ios testflight|appstore`.

## Entry gate

1. Complete normal SRTL initialization, Path Integrity, and Repository Sync/Direct-Remote Gate.
2. Read project `MEMORY.md`, `SKILL.md`, build configuration, source-set/module layout, dependency declarations, current `rtest`, active tickets/claims, and `ticketorder.md`.
3. Re-read the live target ref and every file revision immediately before writing.
4. If an unrelated batch is open, do not disturb it: create the completed port tickets as `[DRAFT]`, append no route lines, report `IOS_PORT_PLANNED_DEFERRED`, and continue SRTL's existing review loop.
5. If no unrelated batch is open, create the port tickets in active states and append their QA/Dev route lines in dependency order.

## Idempotent assessment

Classify the live project before proposing work:

- `IOS_ALREADY_ENABLED`: iOS target, wrapper, adapters, target-aware tests, macOS CI, and current simulator evidence all exist. Create no duplicate tickets; report the evidence and next manual/Launcher gate.
- `KMP_IOS_GAPS`: shared source sets exist, but one or more iOS adapters, wrapper, tests, CI, or packaging pieces are missing.
- `ANDROID_ONLY_PORTABLE`: Android code must be reorganized into KMP and platform boundaries before adding iOS.
- `IOS_PORT_BLOCKED`: a required capability or dependency has no acceptable iOS/KMP path. Record the exact blocker and create no speculative replacement ticket until the user must choose among materially different product outcomes.

Search existing open and terminal tickets before numbering. Reuse or supersede matching work; never create a second ticket for the same gap.

## Architecture law

Default to one repository and one product codebase:

```text
shared/src/commonMain/
shared/src/commonTest/
shared/src/androidMain/
shared/src/iosMain/
shared/src/androidTest/ or androidUnitTest/
shared/src/iosTest/
androidApp/
iosApp/
```

- Put domain rules, state, models, persistence contracts, calculations, navigation contracts, and reusable Compose UI in `commonMain`.
- Keep `androidMain`/`iosMain` and `androidApp`/`iosApp` thin: platform APIs, permissions, lifecycle, packaging, signing, and unavoidable native presentation.
- Use interfaces/dependency injection or Kotlin `expect`/`actual`; do not scatter platform macros or target checks through product logic.
- Preserve one observable behavior contract. Do not copy screens, business rules, assets, or tests merely to satisfy a target build.
- Keep the existing Android application working after every ticket. A port is not permission for an Android rewrite.
- A separate iOS repository requires explicit user approval plus a durable reason such as intentionally divergent products, independent ownership/release history, or a legal/security boundary.

## Ticket construction

Create the smallest dependency-ordered slices supported by the audit; do not generate a fixed ceremonial set. Typical slices are:

1. KMP build graph and source-set bootstrap.
2. Shared-code extraction and Android API isolation.
3. Missing iOS service adapters and dependency replacements.
4. Thin `iosApp` wrapper, lifecycle, resources, permissions, and privacy strings.
5. Target-aware `rtest` plus cost-controlled macOS CI.
6. Final simulator build/evidence and project documentation.

For every ticket:

- Follow `skills/ticket-management/SKILL.md`; allocate numbers from a same-operation live scan and verify no collision.
- Name exact files/functions after inspecting current source. Include precise MAY-touch and MUST-NOT-touch lists.
- State dependencies, Android-preservation checks, performance/privacy constraints, platform-specific acceptance, and an explicit regression guard.
- Put testable behavior/structure in an `[APPROVED]` QA→Dev ticket.
- Use a direct `[READY_FOR_DEV]` UI-lane ticket only for genuinely visual/native-wrapper work with no honest pre-implementation automated assertion; include a numbered human manual script.
- Interleave route lines strictly by dependency, normally `<id>-QA` then `<id>-Dev`. Serialize shared-file hotspots.
- If another batch is open, use `[DRAFT]` for every port ticket and leave `ticketorder.md` untouched.

## Required rtest port

The port batch must implement the cross-platform contract in `skills/rtest/SKILL.md` and record exact commands in the project `SKILL.md`:

```text
rtest --platform common
rtest --platform android
rtest --platform ios
rtest --platform all
```

- Move platform-neutral behavior tests to `commonTest` when safe; retain Android-specific coverage for Android adapters and add `iosTest` coverage for iOS adapters.
- Never delete, weaken, ignore, or duplicate an Android regression merely because it is not directly executable on iOS.
- Inject deterministic fakes for microphones, sensors, clocks, files, locations, and notifications so shared behavior can run cheaply without hardware.
- Treat Compose/native UI automation as supporting evidence. Physical microphone, permission, interruption, notification-delivery, accessibility, and device-lifecycle behavior remain manual gates when they cannot be deterministically observed.
- On a non-macOS host, iOS native execution is `HOST_SKIPPED`, never PASS and never sufficient for port closure.

## Cost-controlled CI

- Keep common and Android checks on the project's cheapest supported host.
- Run iOS native tests and simulator build on a pinned macOS/Xcode CI environment.
- Trigger the macOS job manually and on changes to shared, iOS, build, or release files; avoid consuming macOS minutes for Android-only paths.
- Use unsigned simulator builds during migration. Distribution certificates, profiles, and Apple membership are Launcher concerns when TestFlight/App Store is requested.
- Record the CI run, commit, Xcode/macOS version, commands, test results, and simulator artifact metadata without secrets.

## Completion gate

SRTL may record `IOS_READY_FOR_MANUAL_TEST` only when all of these are true at one pinned commit:

- every migration ticket is terminal and independently reviewed;
- common and existing Android regression gates pass;
- Android application build still passes;
- iOS tests pass on macOS;
- the iOS simulator application builds and launches;
- bundle identifier, minimum iOS version, permissions/privacy strings, and wrapper lifecycle are verified;
- project `SKILL.md` documents the exact Android/iOS build and rtest targets;
- remaining physical-iPhone cases are listed as a human checklist;
- skipped device, signing, TestFlight, and App Store phases are stated separately.

Handoff:

```text
IOS_READY_FOR_MANUAL_TEST: <project>
SOURCE: <branch> <commit>
VERIFIED: <common/android/ios tests and simulator build>
SKIPPED: <physical device/signing/distribution>
NEXT: run the physical-iPhone checklist; then init <project> launcher ios testflight
```

A simulator build is never physical-device certification. `HOST_SKIPPED`, an unreviewed ticket, or a failing Android gate blocks this terminal state.
