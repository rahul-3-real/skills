---
name: mobile-app-development
description: Create, extend, debug, and prepare iOS and Android mobile apps for release. Use for end-to-end mobile app development, including React Native and Expo setup, mobile UX, navigation, device integrations, offline data, testing, and native builds. Adapt to existing Swift, Kotlin, or Flutter projects; exclude browser-only responsive websites.
---

# Mobile App Development

Build a working mobile product, from the first runnable app through its core journeys and platform builds. React Native with TypeScript and Expo is the default for a new cross-platform app when the user has not selected a stack. Preserve existing native, Flutter, or React Native architectures and toolchains.

## Establish the app and delivery target

Inspect project instructions, manifests, lockfiles, neighbouring features, design tokens, native directories, and CI before choosing tools. For existing apps, make the requested change without scaffolding another app or migrating frameworks.

For a new app, identify the primary user journey, target platforms, API availability, authentication needs, device capabilities, and expected deliverable: prototype, locally runnable app, internal build, or store release. Ask only about choices that materially block implementation. Otherwise state reasonable assumptions and build a small vertical slice with navigation, usable states, and persistence or API integration as needed.

| Situation | Approach |
| --- | --- |
| New shared iOS/Android product with no prescribed stack | React Native + Expo + strict TypeScript; prefer Expo Router for navigation |
| Existing React Native app | Retain its router, native ownership model, state libraries, and compatible dependency versions |
| Existing Swift/SwiftUI, Kotlin/Compose, or Flutter app | Work in its language and established architecture; apply the platform-independent mobile guidance here |
| Platform-specific feature, specialized SDK, or performance constraint | Verify native integration support; use a platform adapter or native implementation where justified |

Do not assume Expo prevents custom native code, or choose a framework migration merely to add one capability. Check current official documentation for scaffolding commands, SDK compatibility, native module support, and platform requirements. Do not hardcode a supposedly latest React, React Native, Expo, Xcode, or Android SDK version into project instructions.

## Read only the guidance needed

- For a new React Native app, SDK/configuration work, or React implementation: read [React Native and Expo](references/react-native-expo.md).
- For screen design, navigation, data, authentication, permissions, or device integrations: read [Mobile product and runtime](references/mobile-product-runtime.md).
- For test strategy, performance investigations, builds, CI, distribution, or release readiness: read [Verification and delivery](references/verification-delivery.md).

## Coding standards

These standards adapt this repository's `modern-best-practice-react-components`, `clean-typescript`, `clean-code`, and `bun-first` skills. The essential rules are included here so this skill works when installed alone. Consult companion skills when available and relevant, without importing their web/backend assumptions into a mobile runtime.

- Use `.tsx` for React Native components and `.ts` for other application code, with strict TypeScript. Keep framework-required configuration formats. In native or Flutter projects, use idiomatic Swift, Kotlin, or Dart.
- Use precise types, `unknown` with narrowing at untrusted boundaries, and runtime validation for API, storage, and link payloads. Do not hide errors with `any`, unchecked assertions, or TypeScript suppression comments.
- Prefer named `const` arrow components, hooks, and handlers. Never default-export an anonymous component. Match established export patterns, using the router's required exports in route files.
- Use PascalCase component/type names, descriptive camelCase variables and handlers, kebab-case component filenames, and the project's hook/store naming convention. Prefer double quotes, semicolons, and two-space indentation when no formatter convention exists.
- Keep components focused. Thin screens coordinate feature hooks; reusable presentation components receive typed values and callbacks. Extract custom hooks into separate files and pure business logic into feature modules or utilities. Avoid architectural layers for a single trivial operation.
- Derive values during render. Keep transient UI state local, remote state in the query/cache layer, and shared client state only where needed. Do not duplicate props or query results in a store.
- Put user-triggered work in named event handlers. Use effects for synchronization with external systems, such as app lifecycle, subscriptions, and native resources; clean them up. Do not prohibit effects needed for mobile lifecycle behavior.
- Prefer composition and early returns. Handle asynchronous failures explicitly, preserve useful error context, and never leave unhandled promises, debug logging, dead code, or secrets in logs.
- Follow configured import aliases and module boundaries; verify Metro and test resolution before introducing aliases. Use modern language APIs only when the installed mobile JavaScript engine supports them; TypeScript compilation does not polyfill runtime APIs.
- Reuse existing dependencies. Add only packages needed for the authorized work, with compatible native versions. Prefer Bun for new JavaScript package management and scripts where supported; preserve an existing lockfile/toolchain. Bun is tooling, not the on-device runtime; never import `Bun.*` or Node filesystem APIs into app code.

## Mobile implementation requirements

- Build touch-first screens using actual native components. Respect safe areas, keyboard visibility, back navigation, font scaling, assistive technologies, dark mode, and platform conventions.
- Make loading, empty, error, retry, offline, and pending-write states deliberate. Implement states relevant to the feature rather than adding decorative controls or fake successful actions.
- Treat app suspension, process death, interrupted requests, permission denial, and session expiry as ordinary runtime events. Persist only what the user needs restored.
- Separate ordinary preferences/cache from credentials. Treat bundled configuration as public and enforce authorization on the server.
- Share domain logic where useful; isolate platform-specific capabilities behind focused modules. Preserve platform usability rather than making iOS and Android identical.
- Keep network, authentication, persistence, and native resource ownership explicit. Avoid starting a second backend, collecting analytics, or adding push infrastructure unless the task needs it.

## Finish with evidence

Complete the requested runnable slice and relevant checks. Inspect the actual app where simulator, emulator, or device access is available. For supported platforms, verify the affected journey and appropriate native builds; JavaScript type checking alone does not establish native compatibility.

Report what was built, how to run it, checks performed, and material gaps. Distinguish implementation, simulator checks, physical-device checks, signed builds, and store submission. If tooling or credentials are missing, complete the available work and state the exact remaining step. Building an app does not by itself authorize paid services or publication.
