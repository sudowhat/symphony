# iOS / App Store release reference

## Target and host

- Keep one shared multiplatform codebase. Shared product behavior belongs in `commonMain`; `iosMain` and the Xcode wrapper contain only platform adapters, entitlements, signing, and packaging needed by iOS.
- Select the artifact explicitly: `simulator` for an unsigned simulator build, `testflight` for an archive exported for internal distribution, or `appstore` for an archive prepared for App Store submission.
- Native iOS compilation, archiving, signing, and export require macOS with the project-supported Xcode version. A Windows/Linux host may run shared checks but must report the native phase as `HOST_SKIPPED`, never PASS.
- No Gradle argument, cross-compiler, container or CI trick removes that constraint. And if the project's regression suite has only ever run on Windows, the iOS side is **unverified, not working** — say so in exactly those words rather than implying it functions.
- A macOS CI runner is a valid build host. Pin Xcode, cache conservatively, and trigger expensive iOS jobs manually or only when shared/iOS/release files change.
- Two things make that advice concrete on GitHub Actions: macOS minutes are billed at a **multiplier** over Linux (the most expensive runner class), which is why iOS jobs stay off the per-push path; and `xcode-select -s` should pin the version explicitly, because an image update silently changing the compiler is a classic mystery breakage. Cache the Gradle and Kotlin/Native caches — a cold Kotlin/Native toolchain download is slow and charged at the macOS rate.
- A CI job may compile, test, archive and upload to TestFlight. It may **never** publish to the App Store or release to external testers without explicit human authorization for that exact action.

## Apple account and identifiers

- The **Apple Developer Program membership ($99/yr) is mandatory** for TestFlight or the App Store. There is no free path; a free account signs only to a personal device for 7 days.
- The bundle identifier is **permanent once the app record exists in App Store Connect**. Freeze it deliberately before the first upload — renaming later means a new app record, a new listing, and the loss of every review and all TestFlight history. This is the one preflight decision that cannot be undone later.
- Register the App ID with only the capabilities actually used. An unnecessary capability is the iOS equivalent of an unnecessary Android permission, and is refused on the same grounds.

## Preflight

- Read the shared module, `iosMain`, Xcode project/workspace, scheme, project `SKILL.md`, and `LAUNCH_CHECKLIST.md`.
- Confirm the real shared-framework task and Xcode scheme/configuration instead of assuming names.
- Verify bundle identifier, marketing version, build number, minimum iOS version, supported devices, privacy strings, entitlements, and release signing from effective build/archive output.
- Confirm the source commit is clean, pushed, reviewed as required, and regression-green on shared code.

## Signing and provisioning

- Keep distribution certificates, private keys, provisioning profiles, App Store Connect API keys, and signing passwords in the CI secret store or user-controlled keychain; never commit or print them.
- Use automatic or manual signing consistently with the project. Record the Team ID, certificate fingerprint, profile UUID/name, and expiry without recording secrets.
- Fail closed when distribution signing is unavailable. Never substitute ad hoc, development, or simulator output for a TestFlight/App Store artifact.
- Import ephemeral CI credentials only for the job, use a temporary keychain when appropriate, and remove them during job cleanup.

## Build and verification

1. Run project-prescribed shared tests and static checks.
2. For `simulator`, build the declared scheme for an iOS Simulator destination and record that no distribution signing or physical microphone certification occurred.
3. For `testflight` or `appstore`, archive the Release scheme with `xcodebuild archive`, then export using the authorized distribution method.
4. Verify the archive/export with `xcodebuild -exportArchive` results and inspect the app/IPA metadata and code signature independently.
5. Record SHA-256, byte size, bundle identifier, marketing version, build number, minimum OS, architectures, entitlements, signer identity/fingerprint, source branch, and commit.
6. Preserve the artifact path; do not commit archives or IPAs unless the project explicitly tracks release binaries.

## Size interpretation

- IPA size is **not** the App Store download size. Apple applies app thinning, and the store page figure differs again. Quote the App Store Connect build-detail per-device estimates, not the local IPA size.
- Kotlin/Native produces a static framework whose size is dominated by shared Kotlin code and the Compose runtime; Release symbol stripping matters more here than on Android.
- Before answering whether data is bundled, search the `.app` bundle for `.db`, `.sqlite*`, audio, archives and seed data rather than assuming.

## Honest device validation

- Simulator success validates packaging and simulator runtime only. It is not physical-device certification.
- For microphone or audio-measurement apps, final release evidence requires a real iPhone smoke test covering permission flow, live input, background/interruption behavior where relevant, and representative readings.
- Minimize hardware cost by using a borrowed device or a limited external tester. TestFlight and App Store distribution require an active Apple Developer Program membership; defer that purchase until distribution is actually needed.
- Never present screenshots or measurements from a generated mockup as observed app evidence. Capture store screenshots from the real app on a simulator or device, and label simulator-only evidence accurately.

## App Store Connect handoff

1. Confirm bundle ID, App Store record, agreements, tax/banking state where applicable, and required privacy declarations.
2. Upload only after explicit user authorization.
3. Confirm processing reports the expected version/build and signing identity.
4. Review export-compliance, privacy, age-rating, and permission declarations with the human owner; do not guess policy answers.
5. For TestFlight, select the authorized internal/external group and submit for beta review when required only after explicit approval.
6. For App Store, attach real screenshots/metadata, select the verified build, and stop before submission or release unless explicitly authorized.
7. Run the physical-device smoke test from the distributed build and record the result.

## Rejection risks for this class of app

- Missing or vague privacy usage strings — the single most common avoidable rejection.
- Requesting a permission the visible feature set does not justify.
- Claiming measurement accuracy that implies a certified instrument.
- Tips or donations to the developer must go through in-app purchase; an external payment link is a rejection risk on iOS in a way it is not on Play.

## Required evidence

- target artifact: simulator, TestFlight, or App Store;
- absolute app/archive/IPA path and size;
- SHA-256;
- bundle ID, marketing version/build, minimum iOS, and architectures;
- signing identity/fingerprint, profile, and entitlements without secrets;
- branch and commit;
- shared tests, Xcode build/archive/export results;
- macOS/Xcode version;
- simulator and physical-device results stated separately;
- skipped Android, macOS, signing, or device phases stated separately.
