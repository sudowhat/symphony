# dB Meter: initial iOS audit, 2026-09-06

Repository: `sudowhat/dbmeter`; source commit `290040494564496a018cba2cd572479269244bb4`.
Plan: canonical `dbmeter-folder/ios-plan.md`, DBM-42. Ongoing lesson consolidation: DBM-61.
These are source observations, not results of a native build or an iPhone test. Refresh the active source before using them. No iOS compiler fix has been demonstrated in this audit.

## 1. A wrapper path can be a placeholder

- Evidence level: **OBSERVED_SOURCE**.
- Source: `iosApp/iosApp.xcodeproj/project.pbxproj` contains two comments and no PBX project objects. `iosApp/iosApp/iOSApp.swift` exists separately. DBM-34.
- Cause: source-presence checks accepted packaging that had never been parsed by Xcode.
- Correction: **NOT YET VERIFIED**. Build a real project, configurations, scheme, framework integration, resources, and simulator destination.
- Regression: successful `xcodebuild -list`, native framework link, simulator build and launch, all pinned to one commit.
- Transfer: verify meaningful file contents and executable build graph on any nominally supported platform. Do not assume every existing wrapper needs replacement.

## 2. File existence and MIME metadata do not prove recording

- Evidence level: **OBSERVED_SOURCE**.
- Source: `composeApp/src/iosMain/kotlin/com/dbmeter/app/evidence/IosEvidenceAdapters.kt`, `IosEvidenceAudioEncoder.writeFrame/finish`; DBM-46.
- Symptom/cause: samples accumulate in `ArrayList<Short>`; `finish` creates a file with null content while returning M4A/AAC metadata. No native encoder writes encoded audio. Memory grows with recording duration.
- Correction: **NOT YET VERIFIED**. Implement bounded native AAC encoding using the actual stream format and temp-to-final ownership semantics.
- Regression: decode synthetic recorded frames with native APIs; assert real AAC track, duration tolerance, bounded buffering, abort/error cleanup, and no file when audio saving is off.
- Transfer: a non-null attachment record or existing file cannot establish a functioning media adapter. Do not generalize the project's chosen codec to other products.

## 3. Playback state can be disconnected from playback

- Evidence level: **OBSERVED_SOURCE**.
- Source: same file, `IosEvidenceAudioPlayer.prepare/play/pause/stop`; DBM-46.
- Symptom/cause: `prepare` checks file existence; playback methods update flows without invoking a native player. Audible playback has no implementation here.
- Correction: **NOT YET VERIFIED**. Native player integration with readiness, duration, progress, completion, interruption, and release.
- Regression: native player integration checks plus audible physical-iPhone acceptance; invalid/empty files fail preparation.
- Transfer: trace observable native effects behind UI flags; keep hardware acceptance distinct from fake-driven tests.

## 4. Shared screens can hide missing platform callback wiring

- Evidence level: **OBSERVED_SOURCE**.
- Source: `composeApp/src/commonMain/kotlin/com/dbmeter/app/RuntimeApp.kt`, `App` defaults and `ShareFormatting`; `composeApp/src/iosMain/kotlin/com/dbmeter/app/MainViewController.kt`; `.../platform/IosRateAppAction.kt`; DBM-47.
- Symptom/cause: iOS entry point supplies microphone permission only; several visible action callbacks have empty shared defaults. Rating URL defaults to null. Shared share formatting hardcodes Google Play.
- Correction: **NOT YET VERIFIED**. Explicit iOS callback wiring and platform-specific store identity injection while retaining shared screen logic.
- Regression: drive visible actions through native integration; verify iOS payloads have the assigned Apple identity and Android payloads retain theirs.
- Transfer: shared UI reuse does not establish platform action parity. An unavailable prelaunch listing is also not proof of a broken URL; distinguish configuration from service availability.

## 5. A test wrapper may contain unreachable native intent

- Evidence level: **OBSERVED_SOURCE**.
- Source: `rtest.ps1`, `Invoke-Gradle` invokes `gradlew.bat`; native phase calls Xcode on macOS; parameter block has no platform selector. `.github/` and `composeApp/src/iosTest/` were absent at the pinned commit. DBM-44.
- Correction: **NOT YET VERIFIED**. Host-correct wrapper dispatch plus real native tests/CI and per-platform results, preserving existing guards.
- Regression: run the wrapper on each supported host; prove requested tests executed and no zero-test/native-skip result was counted as a pass.
- Transfer: read and execute the actual command path; a platform branch in a script does not prove the host can reach it.

## 6. Permission copy can lag behind shared product changes

- Evidence level: **OBSERVED_SOURCE**.
- Source: `iosApp/iosApp/Info.plist` refers to explicit “Evidence” recording, while current requirements use Settings switches and ordinary GO/STOP; DBM-47.
- Correction: **NOT YET VERIFIED**. Reconcile iOS usage descriptions, actual prompts, privacy manifest, policy, and store answers from the current capture/egress contract.
- Regression: inspect actual prompt copy and packaged metadata in the signed build; owner reviews final policy declarations.
- Transfer: audit dormant platform resources when shared behavior changes. Do not copy privacy answers across platforms without inspecting their adapters and SDKs.
