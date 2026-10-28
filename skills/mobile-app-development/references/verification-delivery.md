# Verification and Delivery

## Verify the behavior that changed

Choose checks proportional to the work and use installed project tools. Do not introduce a test framework just to mirror implementation details.

- Run formatting/linting, strict type checking, and relevant existing tests using actual project scripts.
- Test meaningful domain behavior, persistence migrations, authentication transitions, and error recovery. For React Native interaction tests, prefer React Native Testing Library with the supported runner; Bun tooling alone does not replace native test presets.
- Exercise the changed journey on each targeted platform: keyboard handling, back navigation, safe areas, theme/font scaling, loading/error/retry, session expiration, and relevant offline behavior.
- For device integrations, check permission denial and actual hardware behavior on a physical device where required. Simulator success cannot prove push delivery, camera behavior, biometric security, or real background execution.
- For critical multi-screen journeys, use the existing end-to-end harness, such as Maestro or Detox, if available. Assert user-visible results instead of relying only on snapshots or test IDs.
- Test cold launch, background/resume, process restart, interrupted requests, and duplicate action prevention where the change affects them.

State exactly which environments were checked. If an iOS toolchain, Android SDK, device, or credential is unavailable, distinguish source verification from runtime/build verification and give the remaining command or prerequisite.

## Performance investigations

Reproduce a concrete symptom before changing architecture. Use release or profiling builds on representative hardware; development mode can distort timing. Inspect startup, main/JS thread work, list scrolling, image size, memory growth, network waterfalls, and battery-sensitive listeners according to the symptom.

Bound list/cache growth, resize media appropriately, paginate remote collections, and avoid expensive computation during touch/animation work. Track native resource ownership and leaks. Profile before and after with the same scenario. Do not claim improvements without measurements. See [React Native performance guidance](https://reactnative.dev/docs/performance).

## Build and delivery

Prepare the requested delivery level. A local prototype does not need a store submission pipeline.

For a new release workflow, choose local native builds or the project's build service based on toolchain, credential access, and CI. Check current official commands and prerequisites. Distinguish development clients, internal distribution builds, release binaries, and uploaded store artifacts.

Before a release build, verify the applicable items:

- Stable app identity, display name, icons, splash treatment, scheme/links, version/build numbers, minimum OS versions, and supported devices.
- Correct environment/API endpoint and native permissions, entitlements, capabilities, and platform configuration. Exclude debug menus, mock endpoints, and test credentials.
- Android signing and iOS certificates/profiles managed through approved secrets storage. Never commit signing material or expose it in logs.
- Privacy disclosures and permission declarations matching actual data use and embedded SDKs. Check current store rules instead of relying on remembered requirements.
- Production-like smoke test of authentication, the main journey, error recovery, deep links, and any changed native integration.
- Crash reporting and source maps/native symbols when part of the operational setup; redact personal data and credentials from telemetry.

For CI, retain deterministic lockfile installation, type/lint/test checks, and native builds on appropriate runners. Supply credentials through CI secrets. Do not invent successful CI results or enable external paid jobs without authorization.

## Updates and rollout

Native code/configuration changes require a compatible new binary. Over-the-air JavaScript/assets updates must target a matching native runtime; they cannot supply a missing native module. Check the runtime version policy and channels before publishing. See [Expo runtime compatibility](https://docs.expo.dev/eas-update/runtime-versions/).

When distribution is requested, prepare the artifact and release notes, identify the destination, and use existing authorization. Internal testing, store upload, store submission, public rollout, and OTA publication are distinct actions. Do not claim one implies another. Preserve an appropriate rollback/recovery path and keep backend/storage changes compatible with supported older clients.

For handoff, report artifact location, version/build identifiers, tested platforms, known limitations, and remaining external steps. If publication was not requested, finish with the runnable app/build instructions rather than starting a submission.

## Platform references

- [Expo Build](https://docs.expo.dev/build/introduction/) and [Submit](https://docs.expo.dev/submit/introduction/).
- [Apple distribution](https://developer.apple.com/documentation/xcode/distributing-your-app-for-beta-testing-and-releases).
- [Android release preparation](https://developer.android.com/studio/publish/preparing).

Consult these when the corresponding build/release task is in scope; verify version-specific requirements instead of copying fixed SDK targets or store-policy dates into the skill.
