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

---

# Staging pass, 2026-09-07 (DBM-61)

Repository: `sudowhat/dbmeter`, commits `75537ad..ade262d`. Tickets DBM-34, 44, 45, 46, 47, 48, all
`[IN_PROGRESS]`.

**Status of entries 1-6 above: corrections are now DRAFTED, and every one of them is still
`NOT YET VERIFIED`.** They were written on a Windows workstation with no Kotlin/Native toolchain,
under an explicit user instruction to stage the work without building or testing. No Kotlin was
compiled, no test executed, no simulator or device touched. Nothing here raises an evidence level;
it records that a proposed correction exists and where to find it.

One amendment to entry 4, from writing the wiring: the sharper failure was not the empty defaults.
`hasLocationPermission` defaults to `{ true }` — a **non-empty default that returns a wrong answer**.
An unwired empty callback does nothing visible and is eventually noticed; a defaulted predicate feeds
confident false state into shared UI and looks like working software. When auditing shared
callbacks, sort them by "what does the default *assert*", not by "which ones are unwired".

## 7. A platform's default backup posture can contradict the product promise

- Evidence level: **OBSERVED_SOURCE** (both halves verifiable without a build).
- Source: `androidApp/src/main/AndroidManifest.xml` sets `android:allowBackup="false"`; the iOS app
  stored its database and audio attachments under `Documents/`, which iOS includes in iCloud backup
  by default. `privacy-policy.md` states "deleting locally is deleting everywhere this data exists".
  DBM-46.
- Symptom and cause: the two platforms would have shipped **opposite** data-egress postures behind
  one written promise, and the policy sentence would have been false on iOS — a deleted measurement
  would still sit in the user's iCloud backup. Nobody chose this; it is the platform default, and
  platform defaults do not read the privacy policy.
- Correction: **NOT YET VERIFIED**. Mark the attachment directory and the database
  `NSURLIsExcludedFromBackupKey`. Note the ordering trap that came with it: an ORM that creates its
  database file lazily has nothing to mark on a first-ever launch, so the flag lands one session
  late unless a post-create hook is added.
- Regression: inspect the platform backup footprint after writing several attachments; assert it
  does not grow with them. This is a device check, not a unit test.
- Transfer: applies whenever one product ships to two platforms with different default backup,
  sync or cloud behaviour, and any user-facing claim depends on data staying local. Derive the
  policy from the **already-decided** platform, rather than treating the second platform's default
  as a fresh product decision. It does not apply where the product intends cloud sync.

## 8. Host detection in a shared test runner can silently read null

- Evidence level: **OBSERVED_SOURCE**.
- Source: `rtest.ps1` gated its native iOS phase on `if ($IsMacOS)`. DBM-44.
- Symptom and cause: `$IsMacOS` and `$IsWindows` were introduced in PowerShell 6 and **do not exist
  at all** in Windows PowerShell 5.1, which is what actually ran this script. The branch read
  `$null`, took the else path, printed `HOST_SKIPPED`, and looked correct — because on that host the
  answer happened to be right. The bug is invisible until the same script runs somewhere else,
  which is exactly when a cross-platform runner is being relied on. Two smaller instances of the
  same class turned up alongside it: the wrapper invoked `gradlew.bat` unconditionally, and a
  Windows-authored repository stores `gradlew` as mode `100644`, so a macOS runner fails with
  "permission denied" before compiling anything.
- Correction: **NOT YET VERIFIED**. Ask the runtime
  (`RuntimeInformation.IsOSPlatform(...)`) instead of trusting an automatic variable that may not
  exist; resolve the wrapper filename per host; `chmod +x` in CI rather than depending on a stored
  mode bit.
- Regression: run the wrapper on every supported host and assert the *intended* phase actually
  executed. A skip that was never reachable reports identically to a skip that was chosen.
- Transfer: any script whose platform branch has only ever run on one platform. Prefer a runtime
  query over a magic variable, and prove the other branch by executing it — reading it is not
  enough.

## 9. Write the guard so the known-bad implementation fails it

- Evidence level: **OBSERVED_SOURCE**, applied while writing tests for entries 2 and 3.
- Source: the stub encoder returned `EvidenceEncodedAudio(mimeType = "audio/mp4", durationMs = ...)`
  for a zero-byte file. DBM-46.
- Symptom and cause: the natural test — assert `finish()` returns non-null with the right MIME type
  and duration — passes against the stub. A test written from the *contract* rather than from the
  *defect* would have shipped green over a feature that did not exist.
- Correction: **VERIFIED as a method, not as a fix.** Every assertion checks the produced bytes:
  non-zero file size, reopenable by the platform decoder, non-zero decoded frame count, decoded
  sample rate matching the negotiated rate, duration within tolerance. Separately, for a static
  guard that could be executed on the authoring host, the guard was run against nine sandboxed
  mutations of the thing it protects and all nine were confirmed to fail it.
- Regression: before trusting a new guard, state the specific defect it exists to catch and
  demonstrate it failing on that defect. A guard that has never failed has never been tested.
- Transfer: universal, and cheapest at the moment the defect is still in front of you. It matters
  most for adapters that can return plausible success — media, storage, network, permissions.
