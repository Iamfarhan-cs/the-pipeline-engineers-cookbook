# Task 1 — Frontend Telemetry Event Boundary

## 1. Task Overview

### Objective

The objective of this task was to establish the **first frontend telemetry boundary** in the \`emi-frontend\` application.

The implementation needed to:

1. Create a small telemetry abstraction.
2. Define a consistent telemetry event structure.
3. Generate a unique event ID and timestamp.
4. Track successful frontend page navigation.
5. Make the generated event observable during development.
6. Avoid capturing sensitive query-string data.
7. Validate the implementation in the real browser.
8. Keep the implementation independent from the future telemetry transport/ingestion layer.

### Project Rule

> **Discover on demand, implement continuously.**

Frontend discovery had already been completed before this task. Therefore, this task did not repeat broad frontend discovery. Only the specific router location required for implementation was inspected.

### Repository

~~~text
~/OneDrive/Desktop/Zolvat/emi-frontend
~~~

### Branch

~~~text
feature/frontend-telemetry
~~~

## 2. Starting Point

Before implementation, the frontend already used:

- Vue 3
- TypeScript
- Vite
- Vue Router 4
- Pinia
- Axios

The application already had a centralized router implementation in:

~~~text
src/router/index.ts
~~~

The router was created using:

~~~text
createRouter({
  history: createWebHistory(),
  routes,
  ...
})
~~~

The existing router also contained an \`afterEach()\` hook.

The existing hook was responsible for refreshing authentication activity:

~~~text
router.afterEach(() => {
  const authStore = useAuthStore()
  authStore.touchActivity()
})
~~~

This existing hook became the natural location for page-view telemetry.

## 3. Implementation Sequence

The implementation was performed in the following order.

### Step 1 — Create the implementation branch

The frontend work was isolated from \`develop\`.

~~~text
git switch -c feature/frontend-telemetry
~~~

The resulting branch was:

~~~text
feature/frontend-telemetry
~~~

No implementation changes were made directly on \`develop\`.

### Step 2 — Inspect the existing frontend structure

The source tree was inspected only far enough to identify the application shell and routing structure.

Relevant files identified included:

~~~text
src/App.vue
src/router/index.ts
src/components/
src/components/client/
src/components/onboarding/
src/components/common/
~~~

\`App.vue\` uses \`RouterView\` and \`useRouter\`, but the actual router instance is created centrally in:

~~~text
src/router/index.ts
~~~

Therefore, telemetry was not placed directly inside individual components or pages.

### Step 3 — Confirm the existing router lifecycle

The router implementation was inspected.

The existing router contained:

~~~text
router.afterEach(() => {
  const authStore = useAuthStore()
  authStore.touchActivity()
})
~~~

This established that successful navigation already had a centralized lifecycle hook.

#### Implementation decision

The existing hook was extended rather than creating another \`router.afterEach()\`.

This prevents multiple independent navigation hooks from being introduced for the same lifecycle event.

## 4. Create the Telemetry Module

A new directory was created:

~~~text
src/telemetry/
~~~

The telemetry implementation was added to:

~~~text
src/telemetry/index.ts
~~~

The module defines the telemetry event contract.

### Event contract

~~~text
export interface TelemetryEvent {
  event_id: string
  event_name: string
  event_version: number
  occurred_at: string
  route?: string
  properties?: Record<string, unknown>
}
~~~

The tracking function accepts:

~~~text
export interface TrackEventInput {
  event_name: string
  route?: string
  properties?: Record<string, unknown>
}
~~~

## 5. Event Generation

The \`trackEvent()\` function creates the telemetry event.

~~~text
export function trackEvent({
  event_name,
  route,
  properties,
}: TrackEventInput): TelemetryEvent {
  const event: TelemetryEvent = {
    event_id: crypto.randomUUID(),
    event_name,
    event_version: 1,
    occurred_at: new Date().toISOString(),
    ...(route ? { route } : {}),
    ...(properties ? { properties } : {}),
  }

  if (import.meta.env.DEV) {
    console.debug('[telemetry]', event)
  }

  return event
}
~~~

### Result

Every generated event contains:

| Field | Implementation |
|---|---|
| \`event_id\` | \`crypto.randomUUID()\` |
| \`event_name\` | Supplied by caller |
| \`event_version\` | \`1\` |
| \`occurred_at\` | \`new Date().toISOString()\` |
| \`route\` | Optional route path |
| \`properties\` | Optional additional properties |

## 6. First Telemetry Event

The first implemented event was:

~~~text
page_view
~~~

The event represents a successfully completed frontend navigation.

The existing router hook was changed to:

~~~text
router.afterEach((to) => {
  const authStore = useAuthStore()
  authStore.touchActivity()

  trackEvent({
    event_name: 'page_view',
    route: to.path,
  })
})
~~~

The telemetry import was added:

~~~text
import { trackEvent } from '@/telemetry'
~~~

## 7. Why \`page_view\` Was Implemented First

\`page_view\` was used as the first event because it can be generated centrally from the existing Vue Router lifecycle.

This means individual pages do not need to contain telemetry calls such as:

~~~text
trackEvent(...)
~~~

in every component.

Instead:

~~~text
Any successful route navigation
        ↓
Vue Router
        ↓
router.afterEach()
        ↓
trackEvent()
        ↓
page_view
~~~

This provides a single implementation boundary for navigation telemetry.

## 8. Why Telemetry Belongs in the Router

The router is responsible for application navigation.

The telemetry event represents successful navigation.

Therefore, the router already knows the information required to generate the first event:

~~~text
destination route
navigation completion
~~~

Putting the tracking logic at the router lifecycle avoids:

- duplicating code across pages
- relying on individual components to remember tracking
- implementing separate page-view logic for every route
- coupling individual business components to telemetry

The telemetry module itself remains independent from Vue Router. The router only calls:

~~~text
trackEvent(...)
~~~

This keeps the event-generation abstraction reusable.

## 9. Privacy Boundary

The implementation deliberately uses:

~~~text
route: to.path
~~~

instead of:

~~~text
route: to.fullPath
~~~

### Why

\`to.fullPath\` can contain query parameters.

The application contains routes where query parameters may contain invitation/token-related values.

Using:

~~~text
to.path
~~~

means the telemetry event records the route path without automatically including the query string.

Example:

~~~text
/client/transfer
~~~

is captured.

The implementation does not automatically capture:

~~~text
?token=...
?invite_id=...
?sig=...
~~~

### Privacy rule for this task

The first \`page_view\` event does not intentionally capture:

- passwords
- access tokens
- email addresses
- account identifiers
- payment information
- query parameters

This establishes a safer boundary before future telemetry properties are introduced.

## 10. Development Visibility

No telemetry package was installed.

Instead, the event is exposed during development using:

~~~text
if (import.meta.env.DEV) {
  console.debug('[telemetry]', event)
}
~~~

This provides an immediate way to inspect generated events without introducing a transport dependency.

### Result

During development:

~~~text
[telemetry] {
  event_id: "...",
  event_name: "page_view",
  event_version: 1,
  occurred_at: "...",
  route: "/client/transfer"
}
~~~

can be inspected through Chrome DevTools.

## 11. Dependency Environment Issue

Initial type checking failed because the local repository did not have \`node_modules\` installed.

The project already declared:

~~~text
vue-tsc
~~~

as a development dependency, but the executable was not available locally.

The repository contained:

~~~text
package-lock.json
~~~

Therefore, the dependency environment was restored using:

~~~text
npm ci
~~~

The first \`npm ci\` attempt encountered transient registry connection failures:

~~~text
ECONNRESET
~~~

A retry completed successfully.

The successful installation reported:

~~~text
added 1106 packages, and audited 1107 packages
~~~

\`vue-tsc\` was then available:

~~~text
vue-tsc@3.1.4
~~~

No manual dependency modification was made for telemetry.

## 12. Validation and Testing

### 12.1 Type Check

Command:

~~~text
npm run type-check
~~~

Result:

~~~text
PASS
~~~

No TypeScript errors were reported.

### 12.2 Production Build

Command:

~~~text
npm run build
~~~

Result:

~~~text
PASS
~~~

The production build completed successfully.

### 12.3 Git Diff Check

Command:

~~~text
git diff --check
~~~

Result:

~~~text
PASS
~~~

No whitespace errors were reported.

### 12.4 Browser Validation

The development frontend was started with:

~~~text
npm run dev
~~~

The application was opened in Chrome.

Chrome DevTools Console was configured to display verbose messages because telemetry uses:

~~~text
console.debug()
~~~

The actual browser event was observed.

Example observed event:

~~~text
[telemetry]

event_id:
31123d2a-03a7-4881-82fd-c6c8604676df

event_name:
page_view

event_version:
1

occurred_at:
2026-09-01T15:36:01.607Z

route:
/client/transfer
~~~

A second telemetry event was also observed after navigation.

This verified that the event is not merely constructed by unit-level code; it is generated by actual frontend navigation.

## 13. Browser Data-Flow Test

The tested flow was:

~~~text
Open application
      ↓
Vue Router navigation
      ↓
router.afterEach()
      ↓
authStore.touchActivity()
      ↓
trackEvent()
      ↓
TelemetryEvent generated
      ↓
console.debug()
      ↓
Chrome DevTools
~~~

A navigation to the transfer route produced:

~~~text
page_view
/client/transfer
~~~

Subsequent navigation generated another \`page_view\` event.

This confirms that the telemetry hook is connected to the existing router lifecycle.

## 14. Edge Cases and Failure Handling

### Query parameters

The implementation uses:

~~~text
to.path
~~~

rather than:

~~~text
to.fullPath
~~~

to avoid automatically capturing query parameters.

### Development-only logging

Console output is guarded by:

~~~text
import.meta.env.DEV
~~~

Therefore the current implementation does not intentionally expose the debug event through \`console.debug()\` in production builds.

### Existing navigation behavior

The existing authentication activity behavior remains intact:

~~~text
authStore.touchActivity()
~~~

Telemetry was added after this existing behavior.

The telemetry change therefore does not replace or remove the existing authentication activity update.

### Backend unavailable

During browser testing, unrelated backend connection errors were visible in the console.

Those errors did not prevent verification of the frontend telemetry event because the \`page_view\` event was generated locally by the frontend.

No backend telemetry transport exists in this task, so backend availability is not currently required for local event generation.

## 15. Security, Privacy and Performance Considerations

### Security

The first event does not intentionally capture authentication credentials or payment information.

The route value is restricted to:

~~~text
to.path
~~~

instead of the full route including query parameters.

### Privacy

The implementation does not add user-identifying fields to \`page_view\`.

The generic \`properties\` field exists in the event contract, but the current \`page_view\` implementation does not populate it.

Future telemetry events must continue to follow the project's existing telemetry privacy rules.

### Performance

The implementation is lightweight:

~~~text
crypto.randomUUID()
new Date().toISOString()
object creation
console.debug() in development
~~~

There is no network request, queue, broker, or external SDK involved in this task.

Therefore, the first implementation does not introduce a telemetry network dependency into every page navigation.

## 16. Git Changes

The implementation modified:

~~~text
src/router/index.ts
~~~

and created:

~~~text
src/telemetry/index.ts
~~~

The relevant router diff was:

~~~text
 import { useAuthStore } from '@/stores/auth'
+import { trackEvent } from '@/telemetry'

-router.afterEach(() => {
+router.afterEach((to) => {
   const authStore = useAuthStore()
   authStore.touchActivity()

+  trackEvent({
+    event_name: 'page_view',
+    route: to.path,
+  })
 })
~~~

No unrelated application files were modified for this implementation.

## 17. Commit

The implementation was committed locally on:

~~~text
feature/frontend-telemetry
~~~

The working tree was verified clean:

~~~text
## feature/frontend-telemetry
~~~

The branch was **not pushed to the remote repository**.

## 18. Final Implementation State

Task 1 now provides the following working frontend telemetry path:

~~~text
┌─────────────────────┐
│    User navigates   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│     Vue Router      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   router.afterEach  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│     trackEvent()    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  TelemetryEvent     │
│                     │
│  event_id           │
│  event_name         │
│  event_version      │
│  occurred_at        │
│  route              │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Chrome DevTools     │
│ [telemetry]         │
└─────────────────────┘
~~~

### Current event

~~~text
event_name = page_view
event_version = 1
route = to.path
~~~

### What is not yet implemented

This task intentionally does **not** include:

- backend telemetry ingestion
- HTTP telemetry transport
- message broker integration
- OpenTelemetry browser SDK
- telemetry database persistence
- batching/retry mechanisms
- production telemetry delivery

Those belong to subsequent implementation work.

## 19. Task Completion Criteria

| Requirement | Status |
|---|---|
| Dedicated implementation branch | Complete |
| Telemetry abstraction | Complete |
| Event contract | Complete |
| First telemetry event | Complete |
| Router integration | Complete |
| Privacy-safe route handling | Complete |
| Browser event verification | Complete |
| Type-check | Passed |
| Production build | Passed |
| Git diff validation | Passed |
| Local commit | Complete |
| Remote push | Not performed |

## Final Status

**Task 1 — Frontend Telemetry Event Boundary: COMPLETE**

The frontend can now generate a standardized \`page_view\` telemetry event from successful Vue Router navigation, with a defined event contract and a privacy-conscious route boundary.
