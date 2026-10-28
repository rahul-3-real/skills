# React Native and Expo

## Create a runnable app

1. Inspect the destination before scaffolding; preserve existing files. Choose a stable official TypeScript starter compatible with the requested features. Verify its current invocation instead of copying an old globally installed CLI command.
2. Use the generated compatible React/React Native/Expo versions. Retain required configuration and add only dependencies needed for the first working journey.
3. Establish app identity, navigation root, providers, theme tokens, and environment-specific public API URL. Do not invent production bundle identifiers, signing credentials, or service accounts.
4. Run the app, then implement one complete journey before expanding screens. Keep route entrypoints thin and feature logic outside the route directory so helpers are not discovered as routes.
5. Provide exact commands matching the resulting package scripts and package manager, plus platform prerequisites and environment variable names. Use example environment values without secrets.

Use a development build when required native libraries are absent from Expo Go; a successful Expo Go demo is insufficient evidence for those integrations. Rebuild the native app when native dependencies or native configuration change. See [Expo development builds](https://docs.expo.dev/develop/development-builds/introduction/).

Decide who owns native files before editing them. With Continuous Native Generation, express reproducible changes in app configuration/config plugins. When native projects are maintained by hand, update them intentionally and preserve customizations. Do not run destructive regeneration over unreviewed native edits. See [Expo CNG](https://docs.expo.dev/workflow/continuous-native-generation/).

For dependency changes, use the installed Expo CLI's compatible installation path and documented package-manager support. Run SDK compatibility diagnostics where applicable; do not independently upgrade React Native underneath an Expo SDK. Confirm library support for the app's native architecture, build targets, and engine rather than toggling architecture flags as a generic workaround.

## Organize by responsibility

Follow the existing layout. A new app can use this shape, adding directories only when used:

```text
src/
  app/                   # Expo Router layouts and route entrypoints
  features/<feature>/    # Screens, hooks, schemas, domain logic
  components/ui/         # Reusable native presentation components
  lib/                   # API client, persistence, device adapters
  providers/             # App-wide providers and lifecycle integration
  theme/                 # Semantic visual tokens
```

An existing root-level `app/` is also valid. React Navigation apps should keep their established screen/navigator organization. Route parameters carry small serializable identifiers; fetch records by ID instead of passing mutable objects, functions, or credentials.

## Translate React conventions to native APIs

| Web-specific pattern | Mobile approach |
| --- | --- |
| HTML elements and CSS stylesheets | `View`, `Text`, `Pressable`, `TextInput`, and supported native styles |
| Semantic HTML and browser ARIA assumptions | Native accessibility roles, labels, states, focus, and platform testing |
| Next.js layouts, `next/navigation`, Server Components | Installed native router and native application lifecycle |
| `"use client"`, `next/dynamic`, DOM and `window` | No directive or DOM dependency; verify Metro behavior before claiming bundle splitting |
| `localStorage` and automatic Zustand persistence | Explicit native storage adapter with hydration handling; secure storage for credentials |
| Browser form `register` bindings | Controlled native inputs; use the form library's native controller integration when needed |
| Browser focus/online events | App state, screen focus, and native connectivity signals |

Use the established HTTP client. A shared Axios client or a typed wrapper around supported `fetch` is acceptable; a mobile app does not require Next.js's client instance. Forward cancellation when supported, normalize errors, validate responses at boundaries, and keep transport details out of UI components.

## Component example

Keep styles in the project's token-based styling system. This small presentational control uses core native APIs and receives behavior from its caller:

```tsx
import { Pressable, Text } from "react-native";

interface SaveButtonProps {
  isSaving: boolean;
  onSave: () => void;
}

const SaveButton = ({ isSaving, onSave }: SaveButtonProps) => (
  <Pressable
    accessibilityRole="button"
    accessibilityState={{ disabled: isSaving, busy: isSaving }}
    disabled={isSaving}
    onPress={onSave}
  >
    <Text>{isSaving ? "Saving…" : "Save"}</Text>
  </Pressable>
);

export default SaveButton;
```

Provide adequate padding and visible pressed/disabled styling through shared tokens in the real component. Async callers must handle errors and prevent duplicate work; changing the label alone is not a submission policy.

Use a virtualized list for unbounded collections, stable record IDs for keys, and pagination for remote data. Provide `getItemLayout` only when actual item dimensions are known. Avoid large same-direction virtualized lists inside a `ScrollView`. See [FlatList](https://reactnative.dev/docs/flatlist).

Memoize only to address observed rendering problems or a library's required stable references. Profile production-like builds before adding caching, animation libraries, or list replacements. Verify bundler behavior before promising dynamic imports will reduce the shipped binary.

## Official documentation

- [React Native setup](https://reactnative.dev/docs/environment-setup): framework selection and current platform setup.
- [Expo documentation](https://docs.expo.dev/): version-specific SDK APIs, app configuration, routing, and diagnostics.
- [React Navigation](https://reactnavigation.org/docs/getting-started/): existing navigator integrations and screen lifecycle.

Use documentation matching installed versions. Read platform-specific installation instructions for each native module, including permission strings, config plugins, Pods/Gradle changes, and rebuild requirements.
