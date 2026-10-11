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

---

# CI & Physical Device Execution Pass, 2026-09-09 (DBM-34 / DBM-55)

Repository: `sudowhat/dbmeter`, build `34369193304`. Toolchain: Xcode 26, macOS-15 runner.

## 10. Cloud macOS CI testing vs. physical device validation economics

- Evidence level: **OBSERVED_PROCESS** / **VERIFIED_METHOD**.
- Source: GitHub Actions macOS runner billing and CI execution logs.
- Symptom and cause: booting iOS simulators and running automated tests on cloud macOS runners
  consumes substantial runner time and burns limited monthly/paid CI minutes ($0.08–$0.16/min).
  Furthermore, iOS simulators do not have real hardware microphones, cannot simulate actual phone
  audio interruptions, lock button suspends, route yanking, or real device microphone sensitivity.
- Correction: **VERIFIED DIRECTIVE**. Standing ruling: **No automated testing on CI**.
  CI is strictly for fast compilation verification (`xcodebuild` clean build and framework link).
  All functional, audio, microphone, storage, and UI testing is conducted directly on a physical
  device (iPhone 15).
- Regression: enforce compilation-only CI workflows with minimal runner minutes; route all
  functional test cases to physical device checklists (`MANUAL_RELEASE_CHECKLIST.md`).
- Transfer: applies to any mobile multiplatform project balancing cloud CI spend against physical
  hardware testing.

## 11. Unsigned physical device IPA packaging for free-tier sideloading

- Evidence level: **VERIFIED_FIX**.
- Source: `.github/workflows/ios.yml`, output artifact `dbmeter-unsigned.ipa` (18.2 MB).
- Symptom and cause: testing on a physical iPhone usually requires a paid $99/year Apple Developer
  Program account for TestFlight or development provisioning profiles. Waiting for enrollment or
  paying upfront can stall initial physical-device testing.
- Correction: **VERIFIED FIX**. Build an unsigned ARM64 physical device binary via CI:
  `xcodebuild -project iosApp/iosApp.xcodeproj -scheme iosApp -destination 'generic/platform=iOS' archive ... CODE_SIGNING_ALLOWED=NO CODE_SIGNING_REQUIRED=NO`.
  Package the resulting `.xcarchive/Products/Applications/iosApp.app` into a standard `Payload/iosApp.app`
  directory, zip it, and name it `.ipa`. The developer can install this directly onto an iPhone 15
  using free tools like Sideloadly with a standard Apple ID.
- Regression: verify generated IPA contains valid ARM64 Mach-O binary and Compose framework assets
  without requiring code signing entitlements during build.
- Transfer: ideal for bootstrapping pre-enrollment physical-device testing on iOS without incurring
  Apple Developer fees.


---

# Compilation pass, 2026-09-09 (DBM-34): moved here from the project handoff on 2026-10-11

Repository: `sudowhat/dbmeter`, `MEMORY.md` → "iOS port — HANDOFF STATE (2026-09-09)", CI runs
34141641058, 34146772472, 34149062240 and later. These findings were recorded in the project but had
not reached this shared reference (DBM-61, the consolidation ticket, is still a draft). Compile
results come from macOS CI; nothing below is device-confirmed unless stated.

## 12. The runner's default Xcode could not link the app

- Evidence level: **VERIFIED_FIX** (CI).
- Source: `.github/workflows/ios.yml` input `xcode_version` (default now `26.0`); DBM-34.
- Symptom and cause:
  - the `macos-15` image defaulted to Xcode 16.4 / iOS SDK 18.5;
  - Compose Multiplatform 1.11.1 ships an object referencing `UIViewLayoutRegion`, which is absent
    from that SDK;
  - the Kotlin framework linked anyway, because it compiles against its own sysroot, so the failure
    appeared only at the **app** link.
- Correction: pin Xcode explicitly; a blank value silently restores the broken toolchain. App Store
  uploads also need the current SDK, so the same pin serves release.
- Regression: the workflow fails loudly if the selected Xcode differs from the pin.
- Transfer: any KMP project whose dependencies track the newest Apple SDK. "The framework links" is
  not "the app links".

## 13. The Kotlin/Native cache key must include the Xcode version

- Evidence level: **VERIFIED_FIX** (CI timing).
- Symptom and cause: the cache key lacked the Xcode version. After switching Xcode, CI restored
  `~/.konan` built for the old SDK, spent 16m39s regenerating platform libraries, and then skipped
  the cache save, because the exact key had hit. Every run paid it again.
- Correction: include the Xcode version (and Kotlin version) in the key.
- Regression: a second run on an unchanged toolchain restores the cache and skips platform-library
  generation.
- Transfer: every Kotlin/Native CI on macOS, where minutes are billed at a multiplier.

## 14. One unsupported target fails the whole iOS configuration

- Evidence level: **VERIFIED_FIX** (82 errors collapsed to 74 real ones once removed).
- Symptom and cause:
  - `iosX64` (the Intel-Mac simulator) was declared;
  - Compose Multiplatform stopped publishing it after `1.11.0-alpha01`;
  - Gradle resolves every declared target, so one missing artifact failed iOS dependency
    resolution for all targets.
- Correction: drop the target, with the owner's approval. The project runner asserts `iosArm64` +
  `iosSimulatorArm64` exist and `iosX64` stays absent.
- Regression: that runner assertion.
- Transfer: declare only targets that every dependency publishes; check before adding a library.

## 15. Opt-ins must reach the leaf Apple source sets

- Evidence level: **VERIFIED_FIX** (CI compile).
- Symptom and cause: `ExperimentalForeignApi` declared only on the intermediate
  `appleMain`/`appleTest` tier was a **configuration** failure, worse than no opt-in: 22 errors plus
  inference cascades.
- Correction: declare it on the whole Apple subtree, leaves included.
- Transfer: any language-feature or opt-in setting in a hierarchical KMP source-set tree.

## 16. An "unresolved reference" to an obvious Apple API is usually a missing category import

- Evidence level: **VERIFIED_FIX** (the last 23 compile errors).
- Symptom and cause: Objective-C **category** members (`setActive`, `timeIntervalSince1970`,
  `sizeWithAttributes`, …) are exposed by Kotlin/Native as extensions. Each needs its own import
  (`platform.AVFAudio.setActive`, `platform.Foundation.timeIntervalSince1970`,
  `platform.UIKit.sizeWithAttributes`).
- Correction: also, `CLLocation.coordinate` is a by-value C struct and needs `useContents { }`.
- Transfer: check the import list before doubting the platform API.

## 17. "Shared" code that only ever compiled for Android was not portable

- Evidence level: **VERIFIED_FIX** (CI compile).
- Symptom and cause: three `commonMain` files had shipped on Android for months while using
  JVM-only APIs: `Math.PI` (java.lang), `toSortedMap()` (java.util) and `Dispatchers.IO`. The
  Windows/Android lane compiled `commonMain` only for the JVM, so nothing objected.
- Correction:
  - replace them with Kotlin-common APIs;
  - make the dispatcher `expect`/`actual` so Android keeps `Dispatchers.IO` exactly. Moving the
    shipping product onto `Default` for an iOS compile convenience was forbidden.
- Regression: compile `commonMain` for a non-JVM target (or its metadata) on every change, on the
  cheap host.
- Transfer: universal for Android-first KMP. Until `commonMain` has compiled for a native target,
  "it is in commonMain" proves nothing.

## 18. Source-set accessors that exist only on macOS break the Windows lane

- Evidence level: **VERIFIED_FIX** (Android lane green after the change).
- Symptom and cause: `appleMain`/`appleTest` exist only when iOS targets are enabled (macOS hosts).
  Touching their accessors unguarded broke the Android build on Windows.
- Correction: guard with the project's `iosTargetsEnabled` flag.
- Transfer: any build script shared between hosts with different target sets.

## 19. Order CI so its own evidence is not gated by unrelated, uncompiled tests

- Evidence level: **VERIFIED_FIX**.
- Symptom and cause: the regression step ran before the app build, so a never-compiled test source
  set (another ticket's) blocked the build-and-launch evidence this ticket needed.
- Correction: build and launch first; run tests afterwards under `always()`, each step capped. The
  simulator test step is capped at 12 minutes and the job at 60.
- Transfer: any pipeline where several in-flight tickets share one workflow.

## 20. Simulator tests can hang on a headless runner; budget decides the testing model

- Evidence level: **REPRODUCED_NATIVE** for the hang (not fixed). **VERIFIED_METHOD** for the cost
  model.
- Symptom and cause:
  - simulator boot cost about 4m30s;
  - the iOS test suite hung for over 21 minutes with no log and took a whole job down; the prime
    suspect is audio tests touching `AVAudioSession` on a headless simulator;
  - on a private repository, macOS minutes bill at 10x, which left roughly 200 macOS minutes a
    month on the free plan, shared by every private repository of the account.
- Correction (project ruling, see entry 10):
  - CI is compile/link only, with a `stop_after` input (`discovery` ≈ 2 min < `link` ≈ 1–2 min
    cached < `compile_app` ≈ 3 min);
  - `workflow_dispatch` only, never on push/PR;
  - native compile errors are triaged in batches per run, not one run per fix.
- Transfer: weigh the macOS minute budget per account before planning native test runs. Hardware-
  bound behaviour belongs on a device either way.

## 21. Two native-adapter defects found while wiring (fixes not yet device-verified)

- Evidence level: **OBSERVED_SOURCE**; corrections **NOT YET VERIFIED** on a device (DBM-46).
- `CLLocationManager.delegate` is a **weak** reference. An adapter that did not keep its own strong
  reference could lose the delegate before the fix arrived. The request then completed into
  nothing, which is indistinguishable from "location unavailable".
- A player created while the audio session is still in `.record` (left there by the meter) produces
  **no audible output**. The session must switch to `.playback` first.
- Transfer: weak delegates and shared audio-session state are classic silent-failure sources in any
  iOS adapter.
