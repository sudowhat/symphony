# Android / Google Play release reference

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
