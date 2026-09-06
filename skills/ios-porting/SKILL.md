---
name: ios-porting
description: Plan and validate Android-to-iOS ports with close UX and behavior parity, using verified implementation lessons. Use during mobile port assessment and after native build, device, or release findings; complements Symphony ios-port planning and release-launch distribution workflows.
---

# iOS porting: evidence and reusable lessons

Keep one product across platforms. Use this skill to turn an audit or a demonstrated porting failure into an actionable plan or a reusable engineering lesson. It does not activate a ticket queue, authorize cloud spending, or authorize Apple account, signing, or publication actions.

For Symphony initialization and ticket lifecycle, follow `../../Agent role.md`. Explicit `srtl ios` planning uses `../ios-port/SKILL.md`; distribution uses `../release-launch/SKILL.md` and its iOS reference. This skill supplies implementation evidence and decision checks, not a competing lifecycle.

## Audit behavior, not filenames

1. Pin the canonical repository commit. Inspect the shared build graph, actual iOS adapter bodies, application entry point, Xcode project, resources, schemes, and tests. An `iosMain` directory or `.xcodeproj` filename is not evidence of a working target.
2. Trace each visible action from the shared screen through its injected callback to its native effect. Look for empty defaults, missing wiring, placeholder files, constant-success returns, Android store URLs, and state flags that never call a native service.
3. Classify each capability separately: present/unverified, observed defect, native-tested, simulator-observed, physical-device-accepted. File presence and a Windows `HOST_SKIPPED` cannot raise a capability to native-tested.
4. Compare current approved requirements, shipped behavior, and historical amendments. Record contradictions; do not silently revive retired features or treat stale prose as a new requirement.
5. Build a parity matrix: Android behavior, intended iOS behavior, evidence, owner, and any explicitly accepted difference. Shared UI should preserve information hierarchy and actions while accommodating safe areas, Dynamic Type, and native sheets. Pixel identity is not the acceptance criterion.

## Make the first native run useful

- Establish an unsigned simulator build before distribution signing. A cloud macOS runner is sufficient for compilation and automated simulator launch; an interactive cloud desktop is a separate capability, not something a CI runner implicitly provides.
- Discover available runner architecture, Xcode, SDK, simulator runtime, JDK, and Kotlin/Compose compatibility. Pin the supported combination. Verify Apple upload requirements from current official documentation when preparing a release; do not bake a dated SDK version into a timeless rule.
- Check that the Xcode project parses and exposes a scheme before diagnosing linker failures. Verify the framework architecture matches the simulator destination. Prove both simulator and device framework builds before describing the port as ready for a physical device.
- Run shared tests on a cheap compatible host and native tests on macOS. Preserve the existing regression entry point and its guards when adding platform selectors; unsupported hosts must report a skip or an explicit unsupported result, never a false pass.
- Limit cloud retries, job duration, artifact retention, and spending to the account owner's settings. Preserve the first failure log and diagnose before retrying. A CI configuration file without a successful run is planned infrastructure.

## Native behavior that needs separate proof

- Microphone permission, actual audio-session state, interruption handling, and app backgrounding must agree. A shared navigation callback is not proof that the OS lifecycle is connected.
- Encoded audio must contain decodable audio, not merely a correctly named file. Test container, codec, sample rate, duration, nonempty payload, and cleanup with synthetic nonpersonal samples. Keep buffering bounded and encoding off the measurement callback.
- Playback must reach a native player and track native completion/error events. A UI `isPlaying` flag alone is not playback.
- Storage and settings need restart, migration, deletion, orphan-file, and invalid-path checks on iOS. Audit backup exposure against the product promise; use explicit product requirements to decide backup policy.
- Map/geocoder and share-sheet behavior have distinct network and disclosure implications. Verify disabled data is absent from exported metadata, text, images, links, and files. Do not infer App Privacy answers from “offline” branding.
- Treat simulator, device, signed archive, TestFlight installation, and public App Store installation as different evidence levels. Use real-device checks for microphone behavior, audible playback, interruptions, and perceived UX; do not promise identical absolute readings from different phone microphones.

## Divide agent work and owner work

The agent owns code, CI configuration, test fixtures, diagnosis, reproducible evidence, draft metadata, and exact owner instructions. The owner handles account identity, contracts, purchases/budgets, credentials under their control, physical-device acceptance, and the final declarations/distribution authorizations required by the task.

If a project uses `[USER]` tickets, document that project-specific extension in its plan. Do not dispatch a human as a fictitious agent role. A dependency on an unanswered human task stays blocked; unrelated agent planning can continue. Do not infer human completion from elapsed time.

## Maintain lessons during the port

After a meaningful build, runtime, device, or release finding, record the lesson while evidence is available. Do not wait until launch to reconstruct it. Record nothing when a ticket produced no reusable finding.

Use this compact structure in a relevant `references/<project>.md` entry:

```text
Evidence level: OBSERVED_SOURCE | REPRODUCED_NATIVE | VERIFIED_FIX | DEVICE_CONFIRMED
Source: repository + relative file/symbol + commit + ticket/run
Symptom and cause: what actually happened and why
Correction: proven correction, or explicitly NOT YET VERIFIED
Regression: test/check that would catch recurrence
Transfer: when this lesson applies elsewhere, and when it does not
```

Keep account IDs, credentials, user recordings/locations, raw private logs, and project-only defaults out of shared lessons. Cross-project references are optional analogies; the active project's exact current files remain authoritative. Promote a technique into the main skill only when evidence supports its applicability beyond one project. Preserve uncertainty instead of writing an untested fix as a rule.

## References

- [dB Meter findings](references/dbmeter.md): initial source audit; read for KMP wrapper, audio-adapter, callback-wiring, and workflow pitfalls. Native fixes are not yet verified.
- [Apple Xcode compatibility](https://developer.apple.com/xcode/system-requirements): verify the build-host/toolchain pair when choosing it.
- [Apple submission requirements](https://developer.apple.com/app-store/submitting/): check current SDK and submission requirements at release time.
- [GitHub Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions): check the owner's current allowance, runner rate, storage, and budget behavior before paid CI use.
