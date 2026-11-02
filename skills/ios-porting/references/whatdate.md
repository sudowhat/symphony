# WhatDate: native validation without a Mac, 2026-11-02

Repository: `sudowhat/whatdate-android`, commit `15fc7d6a0`, ticket WD-912. The owner has no Mac and has an iPhone 15.
The iOS source (SwiftUI app, `iosMain`, `commonMain`) was written across about 40 tickets without a native build.

## 1. Apple klibs compile on Windows; only linking needs a Mac

- Evidence level: **REPRODUCED_NATIVE** for the compile, on a Windows 11 host. The link step is SKIPPED there.
- Source: `shared/build.gradle` (`iosArm64()`, `iosSimulatorArm64()`), Kotlin 2.3.21; `rtest.bat` `:run_ios`.
- **Symptom and cause.** The project treated every iOS step as macOS-only and reported `HOST_SKIPPED`. In fact, the Kotlin/Native Windows distribution ships prebuilt Apple platform klibs (`~/.konan/kotlin-native-prebuilt-windows-x86_64-<ver>/klib/platform/ios_arm64`). These tasks therefore run and type-check against `platform.Foundation`, `platform.Security` and the other Apple frameworks:
  - `compileKotlinIosArm64`;
  - `compileKotlinIosSimulatorArm64`;
  - `compileTestKotlinIosArm64`.

  `linkDebugFrameworkIosArm64` reports SKIPPED. So do the generated Objective-C header and everything Swift.
- **Correction (VERIFIED_FIX).** The Windows runner compiles the three klibs, and a compile error is FAIL. A planted `val x: Int = platform.Foundation.NSDate()` in `iosMain` failed it with a type mismatch. The same source minus the probe passed.
- **Regression.** A lock asserts that the runner calls the three tasks and maps a failure to FAIL, never PASS.
- **Transfer.**
  - Applies to any KMP project on Kotlin 2.x without cinterop of its own C headers. Custom cinterop still needs a Mac.
  - It is a cheaper, stronger form of the "compile `commonMain` for a native target" rule (dbmeter entry 17), because `iosMain` is checked too.
  - It is not evidence about linking, Swift interop names or runtime behaviour.

## 2. Check that CI can start before planning on it

- Evidence level: **OBSERVED_PROCESS**.
- **Source.** `gh run list`: since 2026-10-17, every run of the repository ended `startup_failure` with no job created. That covered 7 runs across 4 workflows, including a Linux workflow dispatch. The last success was 2026-08-11. The macOS workflow had never run.
- **Symptom and cause.** The port plan assumed hosted macOS CI. The account could not start any job. Without the `user` scope the agent cannot see billing, and the pattern (every workflow, no job) points to the account rather than a workflow file.
- **Correction.** Ask the owner to check billing before the first macOS dispatch. Prepare an alternative host (Codemagic's free macOS minutes) and keep the build commands host-neutral.
- **Transfer.** Run `gh run list` across all workflows before relying on Actions for a port. One startup failure with a job is a workflow problem; many with none is an account problem.

## 3. A simulator build without an `iosX64` framework must exclude x86_64

- Evidence level: **OBSERVED_SOURCE**. *Not yet verified on a Mac.*
- **Source.** `iosApp/project.yml`, and the CI step `xcodebuild -sdk iphonesimulator` with no destination.
- **Symptom and cause.** dbmeter entry 14 dropped `iosX64` because Compose no longer publishes it. An `xcodebuild` simulator build with no run destination builds every simulator architecture, so the x86_64 slice has no Kotlin framework to link.
- **Correction (NOT YET VERIFIED).** Set `EXCLUDED_ARCHS[sdk=iphonesimulator*]: x86_64`, or pass an arm64 destination.
- **Transfer.** Applies wherever `iosX64` is absent. Pair the exclusion with entry 14's assertion.

## 4. A no-Mac owner route: unsigned device IPA, then sideload

- Evidence level: **OBSERVED_SOURCE** here. The method is dbmeter entry 11 (VERIFIED_FIX there).
- **Route.**
  1. A manual workflow stage archives `generic/platform=iOS` with `CODE_SIGNING_ALLOWED=NO`.
  2. It packs `Payload/<App>.app` as an `.ipa` (short artifact retention).
  3. The owner signs and installs it from Windows with Sideloadly or AltStore and their own Apple ID.
- **Limits of a free Apple ID.**
  - The app stops opening after 7 days; reinstalling over it keeps its data.
  - At most 3 sideloaded apps and 10 new App IDs per 7 days.
  - Developer Mode must be on.
  - The unsigned IPA carries no entitlements, so Associated Domains (Universal Links) and Time Sensitive notifications are expected to be absent.
  - The owner types the Apple ID password into the signer; agents never handle it.
- **Transfer.** Any pre-membership device test. Not a substitute for TestFlight evidence.
