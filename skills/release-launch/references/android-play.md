# Android / Google Play release reference

## Target and shared-code boundary

- Keep one shared multiplatform codebase. Shared product behavior belongs in `commonMain`; `androidMain` and the Android application wrapper contain only platform adapters, manifest/resources, signing, and packaging needed by Android.
- Select the artifact explicitly: `apk` for direct installation/smoke testing or `aab` for Google Play. The target changes the toolchain task and verification, not the Launcher role or product implementation.
- In a mixed Android/iOS repository, run the Android release task from the same pinned commit used by shared checks. State the iOS phase separately as PASS, FAIL, or `HOST_SKIPPED`.


## Preflight

- Read the effective Android application module, version catalog, manifest, project `SKILL.md`, and `LAUNCH_CHECKLIST.md`.
- Confirm the real task with Gradle (`:<application-module>:bundleRelease`) rather than assuming `app`.
- Verify package, version code/name, min SDK, target SDK, and release signing from generated release outputs.
- Confirm the release commit is clean, pushed, reviewed as required, and regression-green.

## Upload-key handling

- Create one permanent upload key per app in a user-controlled path outside the repository.
- Use an interactive `keytool -genkeypair`; never pass passwords on the command line.
- Keep an encrypted offline backup plus a separately stored recovery record for alias, certificate fingerprints, creation date, and validity. Do not store passwords in the launch checklist.
- JKS and PKCS12 are both usable for Android upload keys. Do not migrate formats mid-launch merely because `keytool` prints an informational recommendation.
- An upload certificate is normally self-signed. `jarsigner -strict` may report an untrusted self-signed chain and missing timestamp even when every AAB entry is correctly signed. Verify the signer identity/fingerprint and signature coverage; do not mislabel the expected trust warning as an unsigned bundle.

## Local signing properties

Prefer environment variables, with an ignored properties file as a local fallback. Typical non-secret keys:

```properties
storeFile=C:/absolute/path/outside/repository/upload-key.jks
storePassword=<local secret>
keyAlias=<alias>
keyPassword=<local secret>
```

Before building, prove the properties file is ignored with `git check-ignore` and absent from `git ls-files`.

## Build and verification

1. Run the project regression command.
2. Run the real `bundleRelease` task; do not disable `lintVitalRelease` or create a baseline to force a pass.
3. Resolve genuine lint/dependency failures; product-code or test failures belong to SRTL.
4. Record the AAB SHA-256 with `Get-FileHash -Algorithm SHA256`.
5. Verify signing with `jarsigner -verify` and inspect the signer with `keytool -printcert -jarfile`.
6. Read package/version/SDK from the generated release bundle manifest or a compatible Android bundle inspection tool.
7. Preserve the artifact path; do not commit the AAB unless the project explicitly tracks release binaries.

## Bundle-size interpretation

- AAB upload size is not the installed or download size. Google Play serves device-specific splits.
- Compose/Kotlin bytecode commonly dominates DEX size.
- Room/SQLite and other native dependencies include several ABIs in the AAB; one device receives only its matching ABI.
- Large launcher images or duplicated density resources can materially increase resources.
- Search artifact entries for `.db`, `.sqlite*`, audio, archives, JSON/CSV seed data, and project-specific evidence/history names before answering whether data is bundled.

## Play Console internal testing

1. Confirm the page states that releases are signed by Google Play; if so, Play App Signing is already active.
2. Upload the verified AAB.
3. Confirm Play reports the expected package and version code/name. A reused version code blocks upload.
4. Review all errors/warnings. Do not proceed through policy declarations by guessing.
5. Add release notes appropriate to the internal audience.
6. Preview and confirm.
7. Start the internal rollout only with explicit user approval.
8. Add testers or tester lists, share the opt-in link, install from Play, and execute the physical-device smoke script.

## Play Console store-side release

The build is the easy half. These are the store-side steps, in the order that unblocks the rest.

1. **Clear `Draft` first.** While an app is `Draft`, Play withholds the binary from every tester
   regardless of a correctly configured tester list and account. The symptom is a working opt-in page
   whose "Download test app" returns **"App not found."** Draft clears only when Store listing and
   App content are both complete. Do not debug this device-side; it is not an adb or install problem.
2. **App content declarations** (Policy → App content): privacy policy URL, Data safety, content
   rating, target audience, ads, and any news/government/financial/health declarations.
3. **Data safety is a transmission question, not an intent question.** "No data collected" is wrong
   if *any* optional code path sends data off-device — a map tile fetch, a platform geocoder lookup,
   a crash reporter. Declare what leaves, with purpose and optionality.
   - A user-initiated share through the OS chooser is Google's **specific user-initiated transfer
     exception**; do not declare data categories solely because the user can share them.
   - The same applies to flows that run inside another app's process (Play In-App Review, a `mailto:`
     compose) — the app neither sees nor stores the result.
   - Answering **Yes** to "users can request data deletion" requires a **Delete data URL**. The
     hosted policy must actually contain deletion instructions, or the URL is a false pointer.
4. **Republish the hosted privacy policy and byte-check it** against the repository copy every
   release. A hosted copy drifts silently; compare normalized SHA-256, don't eyeball it.
5. **Store listing copy.** The **App name is Play's highest-weighted ASO field** — a bare brand name
   carries no keyword weight. Short description is the second lever. Verify every product claim
   against the code before pasting it; a store listing is a public assertion about behavior.
6. **Graphic asset specs, checked locally before upload** (PIL or equivalent, not by rejection):
   - Screenshots: each side 320–3840 px, aspect ratio ≤ 2:1, 24-bit PNG/JPEG **with no alpha**.
   - Feature graphic: **fixed** 1024×500 — a size, not a ratio.
   - Icon: 512×512 32-bit PNG **with** alpha.
   - Label AI-generated or AI-edited assets in the AI declaration.
7. **Submission is batched.** There is no per-change rollout button. Console collects every pending
   edit plus the release into one **"Submit N changes for review"** on Publishing overview. Confirm
   the count matches what you changed. With **Managed publishing off**, approval auto-publishes; with
   it on, approval leaves a second manual gate.
8. **Closed testing before production** (accounts created after Nov 2023): ≥12 testers continuously
   opted in for ≥14 days. Push 2–3 minor updates during the window — review favors visible iteration.
   Answer the production questionnaire from what the app actually does on the submission date; paid
   testing services supply boilerplate answers that are frequently false for the specific app.

## Traps with a track record

Each of these has cost a real launch cycle at least once.

- **A version code is burned only by upload.** Built-but-never-uploaded codes are free to reuse.
  Bump at upload time, not at build time, or you leak version numbers on every rebuild.
- **Source drift with no version bump blocks everything.** Check the live version code against the
  last uploaded one before any other release work.
- **Unused dependencies are not free.** They ship native `.so` files that fail Play's 16 KB
  memory-page check, add permissions, and start background telemetry that contradicts the listing's
  privacy claims. Audit dependency usage before the release build, every release.
- **A dozing device fails connected instrumentation in a way that looks like a code defect** —
  `No compose hierarchies found`, `Activity never becomes RESUMED`. Wake and pin the screen
  (`input keyevent KEYCODE_WAKEUP`, `svc power stayon usb`) before blaming the app.
- **A release-signed install blocks debug-signed instrumentation** (`INSTALL_FAILED_UPDATE_INCOMPATIBLE`).
  Either uninstall first — only with authorization, since it destroys the user's data — or skip the
  phase and record it honestly.
- **Play-only behavior needs a Play-delivered install.** Billing and other Play services cannot
  resolve for a sideloaded build (`installerPackageName=null`). This is an environment limitation,
  not a code defect; don't debug the code.
- **Verify that a "cold" or "full" test flag actually does what it claims** from the tool's own
  output. A declared-but-unused flag can produce years of falsely-recorded clean runs.
- **Expected signing warnings are not failures.** A self-signed upload certificate with no timestamp,
  plus POSIX-attribute warnings, is the normal `jarsigner` output for a valid AAB.
- **Upload size is not download size.** Play serves per-device ABI and resource splits.
- **An unhedged inference propagates as fact.** "Console shows X," written once without anyone
  looking, spreads into every document that cites it and is quoted back as verified. Screenshot-verify
  console state before recording it; hedge explicitly when you have not.

## Required evidence

- absolute AAB path and size;
- SHA-256;
- package, version code/name, target SDK;
- upload-certificate SHA-256 fingerprint;
- branch and commit;
- regression, lint, bundle, and signing results;
- Play processing result;
- device smoke-test result;
- skipped iOS/macOS or Android-device phases stated separately.
