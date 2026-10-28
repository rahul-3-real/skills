# Mobile Product and Runtime

Apply these principles to React Native, native iOS/Android, and Flutter using the project's corresponding APIs.

## Screens and navigation

- Start with the main user task and a clear screen hierarchy. Use platform-appropriate stacks, tabs, sheets, and confirmation flows. Preserve Android system back and iOS back gestures.
- Account for safe areas, edge-to-edge system UI, keyboard resize/avoidance, small displays, tablets when targeted, orientation changes, and split-screen where supported. Prefer responsive dimensions to device-name checks or a single launch-time width.
- Reuse semantic tokens for typography, surfaces, spacing, color, and motion. Use native feedback and haptics when they communicate useful state; respect reduced motion.
- Support system text scaling, long translations, right-to-left layouts when in scope, and light/dark themes. Keep text readable without fixed-height clipping.
- Use platform-appropriate touch targets, persistent input labels, keyboard/autofill hints, and inline validation. Do not disable font scaling merely to fit a design.
- Give controls accessible names, roles, and states. Check focus order, modal focus, error announcements, and VoiceOver/TalkBack behavior. Icons, gestures, and color must not be the only way to understand or activate an action. See [native accessibility](https://reactnative.dev/docs/accessibility).

Configure typed navigation where supported. Parse and validate deep links, universal/app links, and notification destinations. Handle cold/warm launches, unknown routes, missing records, and authentication redirects. Resume an authorized destination after sign-in without leaking protected content during session hydration. Navigation guards do not replace server authorization.

## Data and lifecycle

Keep the existing server-state layer; prefer TanStack Query for a new React Native app needing remote caching. Define query keys by resource and user scope, bounded retries, cancellation, and mutation invalidation. Distinguish initial loading from background refresh so usable data stays visible.

Connect foregrounding and network reachability to the query library once at the app boundary, with cleanup. Screen focus and app focus differ; use screen refresh where the feature needs it. Avoid duplicate refetch subscriptions across screens. See [TanStack Query's native integration](https://tanstack.com/query/latest/docs/framework/react/react-native).

Choose an offline contract explicitly:

| Need | Implementation |
| --- | --- |
| Online-only action | Explain the connectivity requirement and preserve user input |
| Cached read | Show stale data intentionally, with refresh/error feedback |
| Recoverable local draft | Persist a versioned draft and restore after restart |
| Offline writes required by the product | Persist a queue, use idempotency keys, define ordering/conflict rules, and expose pending/failed status |

Do not call a cached list an offline write implementation. Never blindly retry payments or other non-idempotent mutations. Optimistic updates need rollback or reconciliation. Scope persisted caches and queued operations to the account, migrate stored schemas, and clear sensitive state on logout/account changes. Prevent late responses from the old session repopulating the next user's cache.

Use simple key/value storage for ordinary preferences, and a database when relational queries or substantial offline data warrant it. Handle hydration before rendering auth-sensitive UI. Treat corruption or migration failure as an explicit recoverable state rather than silently destroying irreplaceable user content.

App processes can disappear between steps. Persist important drafts/checkpoints and reconcile on resume. Stop subscriptions, timers, camera sessions, location watches, and playback when their ownership ends. Do not assume a mounted screen is visible or background JavaScript keeps running.

## Authentication and configuration

- Keep public configuration separate from server secrets. API URLs and client identifiers may be bundled; private API keys, signing secrets, and service credentials may not. An environment variable name does not make bundled data secret.
- Store small credentials using platform-backed secure storage, not ordinary preferences or an unencrypted persisted store. Handle unavailable or invalidated credentials. Secure storage is not a backup for irreplaceable user data; uninstall/restore and biometric behavior vary by platform. See [Expo SecureStore](https://docs.expo.dev/versions/latest/sdk/securestore/).
- Use the provider's supported native authentication flow. For OAuth, prefer the system authorization browser with PKCE and verified redirects. Do not embed a confidential client secret or invent token exchange logic in the app.
- Model session startup, authenticated, expired, and signed-out states. Coordinate refresh so concurrent failed requests do not race or loop indefinitely. Revoke/clear the session and user-scoped data where appropriate.
- Keep development API addressing explicit: `localhost` on a physical phone is the phone. Use documented emulator host mapping, a reachable development host, or an authorized tunnel. Do not disable transport security globally to fix a local connection.

## Permissions and device features

For camera, photos, location, microphone, notifications, Bluetooth, and similar capabilities:

1. Check platform/library support, native configuration, and physical-device requirements.
2. Request necessary permissions when the user activates the feature, with a clear purpose. Add required platform usage descriptions and declarations.
3. Handle granted, denied, restricted, limited access, revoked access, and unavailable hardware as applicable. Offer a fallback or settings route without repeatedly prompting.
4. Test cancellation, interruption, backgrounding, and return from system settings. Release acquired resources.

Ask separately for background access only if the product needs it. Use platform scheduling/task APIs for background work and account for operating-system restrictions; never promise exact scheduling from a JavaScript timer.

For push notifications, distinguish permission, token registration, backend delivery, and notification opening. Handle token rotation, account binding/logout, duplicates, and foreground/background/cold-start routing. Do not put secrets in payloads or report delivery verified without exercising real delivery.

Validate media types, sizes, and upload outcomes. Handle temporary-file cleanup, cancellation, and interrupted uploads. Add native billing only when requested; verify current store and SDK requirements before implementing purchases, and keep entitlement verification authoritative on the server.
