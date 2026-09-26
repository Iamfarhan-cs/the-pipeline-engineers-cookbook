# ask 1 — Frontend Telemetry Event Boundary

## 1. Task Overview

### Objective

The objective of this task was to establish the **first frontend telemetry boundary** in the `emi-frontend` application.

The implementation needed to:

1.  Create a small telemetry abstraction. 
2.  Define a consistent telemetry event structure. 
3.  Generate a unique event ID and timestamp. 
4.  Track successful frontend page navigation. 
5.  Make the generated event observable during development. 
6.  Avoid capturing sensitive query-string data. 
7.  Validate the implementation in the real browser. 
8.  Keep the implementation independent from the future telemetry transport/ingestion layer. 

### Project Rule

> **Discover on demand, implement continuously.**

Frontend discovery had already been completed before this task. Therefore, this task did not repeat broad frontend discovery. Only the specific router location required for implementation was inspected.

### Repository

```
```

```
~/OneDrive/Desktop/Zolvat/emi-frontend
```

### Branch

```
```

```
feature/frontend-telemetry
```

---

# 2. Starting Point

Before implementation, the frontend already used:

-  Vue 3 
-  TypeScript 
-  Vite 
-  Vue Router 4 
-  Pinia 
-  Axios 

The application already had a centralized router implementation in:

```
```

```
src/router/index.ts
```

The router was created using:

```
```

```
createRouter({
  history: createWebHistory(),
  routes,
  ...
})
```

The existing router also contained an `afterEach()` hook.

The existing hook was responsible for refreshing authentication activity:

```
```

```
router.afterEach(() => {
  const authStore = useAuthStore()
  authStore.touchActivity()
})
```

This existing hook became the natural location for page-view telemetry.

---

# 3. Implementation Sequence

The implementation was performed in the following order.

## Step 1 — Create the implementation branch

The frontend work was isolated from `develop`.

```
```

```
git switch -c feature/frontend-telemetry
```

The resulting branch was:

```
```

```
feature/frontend-telemetry
```

No implementation changes were made directly on `develop`.

---

## Step 2 — Inspect the existing frontend structure

The source tree was inspected only far enough to identify the application shell and routing structure.

Relevant files identified included:

```
```

```
src/App.vue
src/router/index.ts
src/components/
src/components/client/
src/components/onboarding/
src/components/common/
```

`App.vue` uses `RouterView` and `useRouter`, but the actual router instance is created centrally in:

```
```

```
src/router/index.ts
```

Therefore, telemetry was not placed directly inside individual components or pages.

---

## Step 3 — Confirm the existing router lifecycle

The router implementation was inspected.

The existing router contained:

```
```

```
router.afterEach(() => {
  const authStore = useAuthStore()
  authStore.touchActivity()
})
```

This established that successful navigation already had a centralized lifecycle hook.

### Implementation decision

The existing hook was extended rather than creating another `router.afterEach()`.

This prevents multiple independent navigation hooks from being introduced for the same lifecycle event.

---

# 4. Create the Telemetry Module

A new directory was created:

```
```

```
src/telemetry/
```

The telemetry implementation was added to:

```
```

```
src/telemetry/index.ts
```

The module defines the telemetry event contract.

## Event contract

```
```

```
export interface TelemetryEvent {
  event_id: string
  event_name: string
  event_version: number
  occurred_at: string
  route?: string
  properties?: Record<string, unknown>
}
```

The tracking function accepts:

```
```

```
export interface TrackEventInput {
  event_name: string
  route?: string
  properties?: Record<string, unknown>
}
```

---

# 5. Event Generation

The `trackEvent()` function creates the telemetry event.

```
```

```
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
```

### Result

Every generated event contains:

| FieldImplementation |                                |
| ------------------- | ------------------------------ |
| `event_id`          | `crypto.randomUUID()`          |
| `event_name`        | Supplied by caller             |
| `event_version`     | `1`                            |
| `occurred_at`       | `new Date().toISOString()`     |
| `route`             | Optional route path            |
| `properties`        | Optional additional properties |

---

# 6. First Telemetry Event

The first implemented event was:

```
```

```
page_view
```

The event represents a successfully completed frontend navigation.

The existing router hook was changed to:

```
```

```
router.afterEach((to) => {
  const authStore = useAuthStore()
  authStore.touchActivity()

  trackEvent({
    event_name: 'page_view',
    route: to.path,
  })
})
```

The telemetry import was added:

```
```

```
import { trackEvent } from '@/telemetry'
```

---

# 7. Why `page_view` Was Implemented First

`page_view` was used as the first event because it can be generated centrally from the existing Vue Router lifecycle.

This means individual pages do not need to contain telemetry calls such as:

```
```

```
trackEvent(...)
```

in every component.

Instead:

```
```

```
Any successful route navigation
        ↓
Vue Router
        ↓
router.afterEach()
        ↓
trackEvent()
        ↓
page_view
```

This provides a single implementation boundary for navigation telemetry.

---

# 8. Why Telemetry Belongs in the Router

The router is responsible for application navigation.

The telemetry event represents successful navigation.

Therefore, the router already knows the information required to generate the first event:

```
```

```
destination route
navigation completion
```

Putting the tracking logic at the router lifecycle avoids:

-  duplicating code across pages 
-  relying on individual components to remember tracking 
-  implementing separate page-view logic for every route 
-  coupling individual business components to telemetry 

The telemetry module itself remains independent from Vue Router. The router only calls:

```
```

```
trackEvent(...)
```

This keeps the event-generation abstraction reusable.

---

# 9. Privacy Boundary

The implementation deliberately uses:

```
```

```
route: to.path
```

instead of:

```
```

```
route: to.fullPath
```

## Why

`to.fullPath` can contain query parameters.

The application contains routes where query parameters may contain invitation/token-related values.

Using:

```
```

```
to.path
```

means the telemetry event records the route path without automatically including the query string.

Example:

```
```

```
/client/transfer
```

is captured.

The implementation does not automatically capture:

```
```

```
?token=...
?invite_id=...
?sig=...
```

### Privacy rule for this task

The first `page_view` event does not intentionally capture:

-  passwords 
-  access tokens 
-  email addresses 
-  account identifiers 
-  payment information 
-  query parameters 

This establishes a safer boundary before future telemetry properties are introduced.

---

# 10. Development Visibility

No telemetry package was installed.

Instead, the event is exposed during development using:

```
```

```
if (import.meta.env.DEV) {
  console.debug('[telemetry]', event)
}
```

This provides an immediate way to inspect generated events without introducing a transport dependency.

### Result

During development:

```
```

```
[telemetry] {
  event_id: "...",
  event_name: "page_view",
  event_version: 1,
  occurred_at: "...",
  route: "/client/transfer"
}
```

can be inspected through Chrome DevTools.

---

# 11. Dependency Environment Issue

Initial type checking failed because the local repository did not have `node_modules` installed.

The project already declared:

```
```

```
vue-tsc
```

as a development dependency, but the executable was not available locally.

The repository contained:

```
```

```
package-lock.json
```

Therefore, the dependency environment was restored using:

```
```

```
npm ci
```

The first `npm ci` attempt encountered transient registry connection failures:

```
```

```
ECONNRESET
```

A retry completed successfully.

The successful installation reported:

```
```

```
added 1106 packages, and audited 1107 packages
```

`vue-tsc` was then available:

```
```

```
vue-tsc@3.1.4
```

No manual dependency modification was made for telemetry.

---

# 12. Validation and Testing

## 12.1 Type Check

Command:

```
```

```
npm run type-check
```

Result:

```
```

```
PASS
```

No TypeScript errors were reported.

---

## 12.2 Production Build

Command:

```
```

```
npm run build
```

Result:

```
```

```
PASS
```

The production build completed successfully.

---

## 12.3 Git Diff Check

Command:

```
```

```
git diff --check
```

Result:

```
```

```
PASS
```

No whitespace errors were reported.

---

## 12.4 Browser Validation

The development frontend was started with:

```
```

```
npm run dev
```

The application was opened in Chrome.

Chrome DevTools Console was configured to display verbose messages because telemetry uses:

```
```

```
console.debug()
```

The actual browser event was observed.

Example observed event:

```
```

```
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
```

A second telemetry event was also observed after navigation.

This verified that the event is not merely constructed by unit-level code; it is generated by actual frontend navigation.

---

# 13. Browser Data-Flow Test

The tested flow was:

```
```

```
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
```

A navigation to the transfer route produced:

```
```

```
page_view
/client/transfer
```

Subsequent navigation generated another `page_view` event.

This confirms that the telemetry hook is connected to the existing router lifecycle.

---

# 14. Edge Cases and Failure Handling

## Query parameters

The implementation uses:

```
```

```
to.path
```

rather than:

```
```

```
to.fullPath
```

to avoid automatically capturing query parameters.

---

## Development-only logging

Console output is guarded by:

```
```

```
import.meta.env.DEV
```

Therefore the current implementation does not intentionally expose the debug event through `console.debug()` in production builds.

---

## Existing navigation behavior

The existing authentication activity behavior remains intact:

```
```

```
authStore.touchActivity()
```

Telemetry was added after this existing behavior.

The telemetry change therefore does not replace or remove the existing authentication activity update.

---

## Backend unavailable

During browser testing, unrelated backend connection errors were visible in the console.

Those errors did not prevent verification of the frontend telemetry event because the `page_view` event was generated locally by the frontend.

No backend telemetry transport exists in this task, so backend availability is not currently required for local event generation.

---

# 15. Security, Privacy and Performance Considerations

## Security

The first event does not intentionally capture authentication credentials or payment information.

The route value is restricted to:

```
```

```
to.path
```

instead of the full route including query parameters.

---

## Privacy

The implementation does not add user-identifying fields to `page_view`.

The generic `properties` field exists in the event contract, but the current `page_view` implementation does not populate it.

Future telemetry events must continue to follow the project's existing telemetry privacy rules.

---

## Performance

The implementation is lightweight:

```
```

```
crypto.randomUUID()
new Date().toISOString()
object creation
console.debug() in development
```

There is no network request, queue, broker, or external SDK involved in this task.

Therefore, the first implementation does not introduce a telemetry network dependency into every page navigation.

---

# 16. Git Changes

The implementation modified:

```
```

```
src/router/index.ts
```

and created:

```
```

```
src/telemetry/index.ts
```

The relevant router diff was:

```
```

```
 import { useAuthStore } from '@/stores/auth'
+import { trackEvent } from '@/telemetry'

-router.afterEach(() => {
+router.afterEach((to) => {
   const authStore = useAuthStore()
   authStore.touchActivity()
+
+  trackEvent({
+    event_name: 'page_view',
+    route: to.path,
+  })
 })
```

No unrelated application files were modified for this implementation.

---

# 17. Commit

The implementation was committed locally on:

```
```

```
feature/frontend-telemetry
```

The working tree was verified clean:

```
```

```
## feature/frontend-telemetry
```

The branch was **not pushed to the remote repository**.

---

# 18. Final Implementation State

Task 1 now provides the following working frontend telemetry path:

```
```

```
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
```

### Current event

```
```

```
event_name = page_view
event_version = 1
route = to.path
```

### What is not yet implemented

This task intentionally does **not** include:

-  backend telemetry ingestion 
-  HTTP telemetry transport 
-  message broker integration 
-  OpenTelemetry browser SDK 
-  telemetry database persistence 
-  batching/retry mechanisms 
-  production telemetry delivery 

Those belong to subsequent implementation work.

---

# 19. Task Completion Criteria

| RequirementStatus               |               || ------------------------------- | ------------- |
| Dedicated implementation branch | Complete      |
| Telemetry abstraction           | Complete      |
| Event contract                  | Complete      |
| First telemetry event           | Complete      |
| Router integration              | Complete      |
| Privacy-safe route handling     | Complete      |
| Browser event verification      | Complete      |
| Type-check                      | Passed        |
| Production build                | Passed        |
| Git diff validation             | Passed        |
| Local commit                    | Complete      |
| Remote push                     | Not performed |

## Final Status

**Task 1 — Frontend Telemetry Event Boundary: COMPLETE**

The frontend can now generate a standardized `page_view` telemetry event from successful Vue Router navigation, with a defined event contract and a privacy-conscious route boundary.

---

# Payment Telemetry Pipeline — Measurement Plan Phase

## 1. Task Overview

### Objective

The purpose of this task was to create the **Measurement Plan v1** for the Payment Telemetry Pipeline before starting the production telemetry implementation.

The measurement plan defines:

-  What product/process questions telemetry should answer. 
-  Which frontend journey events should be measured. 
-  Which backend compliance events should be measured. 
-  Which operational signals should be collected. 
-  Which properties are allowed. 
-  Which data must never be collected. 
-  Where each type of telemetry should be stored. 
-  How frontend events and backend compliance states should work together. 
-  What retention and access-control decisions must be agreed before production rollout. 

The main design principle from the implementation discussion was:

> **Collect only the telemetry required to answer defined product and operational questions.**

The project is intentionally starting with PostgreSQL for product/process analytics rather than introducing an S3/data-lake architecture.

---

# 2. Implementation Sequence

The work was performed in the following order.

## Step 1 — Capture the Measurement Direction

The initial requirement was reviewed and converted into two separate telemetry signal groups:

### Operational Observability

Used to understand whether the frontend application is working correctly.

Signals include:

-  Vue errors 
-  Unhandled exceptions 
-  Unhandled promise rejections 
-  Sanitized `console.error` 
-  Sanitized `console.warn` 
-  Failed API operations 
-  Web Vitals 
-  Release/build version 
-  Frontend-to-backend trace correlation 

These signals are intended for:

```
```

```
Frontend
   ↓
OpenTelemetry / Collector
   ↓
Loki / Tempo / VictoriaMetrics
   ↓
Grafana
```

### Product / Process Analytics

Used to understand user journeys and business processes.

Signals include:

-  Onboarding journey events 
-  Account-type selection 
-  Major onboarding steps 
-  Application submission 
-  Liveness verification 
-  Compliance state transitions 

These signals are intended for:

```
```

```
Frontend / Backend
      ↓
Telemetry ingestion
      ↓
PostgreSQL
      ↓
Grafana
```

---

# 3. Step 2 — Define Product Questions

Telemetry was not defined by simply collecting everything available.

Instead, the required events were derived from questions that the product and engineering teams need to answer.

## Onboarding Questions

The measurement plan needs to answer:

1.  How many users start onboarding? 
2.  How many users reach each major onboarding step? 
3.  Where do users abandon the process? 
4.  Which steps have the highest drop-off? 
5.  How long does the user take between major steps? 
6.  Which onboarding steps generate the most frontend/API failures? 
7.  Are particular releases associated with increased onboarding failures? 

## Compliance Questions

The measurement plan needs to answer:

1.  How many users reach the compliance stage? 
2.  How many complete the compliance process? 
3.  Where does the process stop or fail? 
4.  How does the frontend journey relate to backend compliance state? 
5.  How long does the process take between important compliance states? 
6.  Are frontend/API failures associated with compliance friction? 

## Operational Questions

The measurement plan needs to answer:

1.  Are frontend errors increasing? 
2.  Which routes generate the most errors? 
3.  Which API operations fail most frequently? 
4.  Are failures concentrated in a particular release? 
5.  Are Web Vitals degrading after releases? 
6.  Can a frontend failure be correlated with the corresponding backend request? 

---

# 4. Step 3 — Inspect the Existing Frontend Architecture

The existing frontend application was reviewed before defining implementation points.

Important findings:

-  A centralized Axios client already exists. 
-  Existing verification-session telemetry already exists. 
-  Vue has a global `app.config.errorHandler`. 
-  There were no global `window.onerror` or `unhandledrejection` handlers. 
-  No Web Vitals implementation was found. 
-  No frontend release-version source was found. 
-  Existing trace-header extraction already exists. 
-  Existing onboarding/account flows were inspected to identify real user actions. 

This avoided introducing duplicate telemetry mechanisms.

---

# 5. Step 4 — Map the Actual Frontend Journey

The actual frontend onboarding flow was inspected instead of assuming a generic onboarding sequence.

## Personal Account Flow

The confirmed flow is:

```
```

```
/client/accounts
       ↓
/choose-account-type
       ↓
/individual-verification/:requestId
       ↓
/personal/form/:requestId
       ↓
/verification/liveness-consent
       ↓
/verification/liveness
       ↓
/personal/account/success
```

The following implementation points were identified.

### Account Type Selection

`ChooseAccountType.vue` contains the real user action:

```
```

```
applyForNewAccount(accountType)
```

This is the appropriate location for:

```
```

```
account_type_selected
```

Allowed property:

```
```

```
account_type = PERSONAL | CORPORATE
```

The request ID is not included in telemetry.

### Identity Verification

After successful identity information persistence:

```
```

```
persistIdentityDraft({
    mode: 'submit',
    showError: true
})
```

the user proceeds to the personal account form.

A candidate event was identified:

```
```

```
identity_verification_completed
```

The event should occur after successful persistence and before routing to the next stage.

### Personal Account Form

The active sections were identified as:

1.  Currency Type 
2.  Account Creation Reason 
3.  Expected Turnover 
4.  Source of Funds 
5.  Tax Declaration 
6.  Additional Information 
7.  Terms & Conditions 
8.  Submit 

No field values should be captured.

The successful form submission is performed through the existing backend API.

A candidate event was identified:

```
```

```
onboarding_form_submitted
```

This event should represent successful backend acceptance rather than simply a button click.

### Liveness Verification

The liveness flow contains:

```
```

```
livenessConsent.vue
```

and

```
```

```
livenessVerification.vue
```

The beginning of the liveness process can support:

```
```

```
liveness_verification_started
```

Successful liveness completion can support:

```
```

```
liveness_verification_completed
```

Existing verification-session telemetry should be reused instead of creating duplicate session-state telemetry.

### Final Onboarding Completion

`KYCSuccess.vue` is not itself treated as the final onboarding completion event.

The strongest completion point identified for personal account onboarding is reaching:

```
```

```
/client/personal-account-success
```

Liveness-only flows must not incorrectly count as a new-account onboarding completion.

---

# 6. Step 5 — Inspect the Corporate Flow

The corporate onboarding flow was inspected separately.

The important implementation point is:

```
```

```
submitApplicationForReview()
```

The successful backend operation is:

```
```

```
PUT /user/accounts/create/corporate/:requestId
```

A successful response with no blocking `tabs` errors leads to:

```
```

```
client-corporate-stakeholder-links
```

The candidate product event is:

```
```

```
corporate_application_submitted
```

The event should be generated after successful backend acceptance and before navigation.

The following must not be added to telemetry:

- `requestId` 
-  stakeholder identifiers 
-  emails 
-  names 
-  submitted form values 

---

# 7. Step 6 — Inspect Existing Verification Telemetry

The existing file:

```
```

```
src/utils/verificationSessionTelemetry.ts
```

already contains verification-session telemetry.

It supports flows including:

```
```

```
personal_authenticated
stakeholder_invite
```

Existing functionality includes:

```
```

```
trackVerificationSessionState()
setVerificationSessionState()
```

The existing mechanism should be reused where applicable.

### Why This Fits Here

Verification-session state already has a dedicated telemetry mechanism.

Creating another independent telemetry implementation for the same session state would:

-  duplicate events, 
-  create inconsistent semantics, 
-  increase maintenance, 
-  make downstream analysis harder. 

Therefore the measurement plan explicitly prefers reuse over duplication.

---

# 8. Step 7 — Define Operational Signals

The operational telemetry plan defines the following candidate signals.

| SignalPurposeTarget            |                              |                              |
| ------------------------------ | ---------------------------- | ---------------------------- |
| `frontend_error`               | General frontend errors      | Loki                         |
| `frontend_unhandled_exception` | Browser/runtime exceptions   | Loki                         |
| `frontend_unhandled_rejection` | Promise failures             | Loki                         |
| `frontend_console_error`       | Sanitized console errors     | Loki                         |
| `frontend_console_warn`        | Sanitized console warnings   | Loki                         |
| `frontend_api_error`           | Failed API operations        | Loki / metrics               |
| `frontend_web_vital`           | Frontend performance         | VictoriaMetrics              |
| Trace information              | Frontend/backend correlation | Tempo                        |
| Release metadata               | Release correlation          | Grafana/Loki/VictoriaMetrics |

These are measurement candidates defined by the plan; implementation should follow the agreed event contract.

---

# 9. Step 8 — Define Common Operational Properties

Only properties required for the specific signal should be included.

Potential permitted properties are:

```
```

```
event_id
occurred_at
event_name
event_version
route
release_version
error_type
redacted_error_message
redacted_stack_trace
http_method
sanitized_endpoint
http_status
duration_ms
trace_id
```

Not every event needs every property.

For example, an API failure can use:

```
```

```
http_method
sanitized_endpoint
http_status
duration_ms
trace_id
```

while a frontend exception can use:

```
```

```
error_type
redacted_error_message
redacted_stack_trace
route
release_version
```

---

# 10. Step 9 — Identify the Correct API Failure Integration Point

The existing Axios implementation was inspected.

The centralized file is:

```
```

```
src/utils/axios.ts
```

It already contains a response interceptor and centralized error handling.

The proposed `frontend_api_error` telemetry belongs at this centralized layer.

### Why This Fits Here

Using the centralized Axios interceptor means API failures can be measured consistently across the application.

It avoids adding separate failure tracking to individual components.

The telemetry must use only sanitized metadata.

It must not capture:

-  request bodies, 
-  response bodies, 
-  tokens, 
-  query parameters, 
-  raw Axios errors. 

---

# 11. Step 10 — Inspect Trace Correlation

The existing file:

```
```

```
src/utils/httpTrace.ts
```

already provides:

```
```

```
extractTraceIdFromHeaders()
```

It checks:

```
```

```
x-request-id
x-trace-id
traceparent
```

and extracts the relevant trace identifier.

Existing stakeholder flows already use this functionality.

### Design Decision

The telemetry implementation should reuse this existing trace extraction mechanism.

### Why This Fits Here

Trace correlation already has a shared utility.

Reusing it keeps frontend/backend correlation consistent and avoids implementing different trace parsing rules in telemetry code.

Also:

```
```

```
request_id
```

and

```
```

```
trace_id
```

must not be treated as automatically interchangeable identifiers.

---

# 12. Step 11 — Inspect Web Vitals Support

The frontend was checked for existing Web Vitals support.

No existing implementation was found for:

-  LCP 
-  INP 
-  CLS 
-  FCP 
-  TTFB 
- `PerformanceObserver` 
- `web-vitals` package 

Therefore Web Vitals remain an implementation item from the measurement plan.

The intended telemetry category is:

```
```

```
frontend_web_vital
```

with the resulting metrics routed to the operational observability stack.

---

# 13. Step 12 — Inspect Release Version Availability

The frontend was checked for an existing release/build identifier.

The following were searched:

```
```

```
APP_VERSION
RELEASE_VERSION
release_version
releaseVersion
import.meta.env.*VERSION
CI_COMMIT_SHA
CI_COMMIT_SHORT_SHA
```

No existing release-version implementation was identified in the inspected frontend configuration.

Therefore the measurement plan does **not** invent a release identifier.

A deployment/build version or short commit SHA can be introduced during implementation once the actual deployment source is identified.

---

# 14. Step 13 — Inspect Global Vue Error Handling

The existing global Vue error handler in:

```
```

```
src/main.ts
```

currently contains:

```
```

```
app.config.errorHandler = (error) => {
  console.log(error)
}
```

This provides an existing application-level location for Vue errors.

The telemetry implementation must not forward the raw error object.

Instead, error information must be sanitized/redacted before telemetry is emitted.

---

# 15. Step 14 — Inspect Backend Compliance Architecture

The backend repository was inspected to identify the authoritative source of onboarding compliance state.

Relevant backend areas include:

```
```

```
logic/complianceindividualaccount.go
logic/compliancekyc.go
logic/kyc_request_resolution.go
db/repo/reviewrequestkyc.go
```

The backend explicitly distinguishes onboarding reviews using:

```
```

```
dao.ReviewKindOnboarding
```

This is important because onboarding compliance reviews must not be mixed with unrelated/manual review activity.

---

# 16. Step 15 — Identify Authoritative Backend States

The following backend statuses were observed:

```
```

```
UNSUBMITTED
PENDING
AWAITING
ACCEPTED
APPROVED
REJECTED
CANCELLED
ON_HOLD
```

The implementation must not assume these statuses are interchangeable.

For example:

```
```

```
ACCEPTED != PENDING
```

The backend state machine is the authoritative source for compliance-state analytics.

---

# 17. Step 16 — Inspect Personal Compliance Flow

In:

```
```

```
logic/complianceindividualaccount.go
```

successful validation causes:

```
```

```
SetAccountReviewToSubmitted(...)
```

The compliance review is then retrieved.

Depending on its state:

```
```

```
Co1Status == AWAITING
```

can transition to:

```
```

```
PENDING
```

or:

```
```

```
Co2Status == AWAITING
```

can transition to:

```
```

```
PENDING
```

The updated review is persisted.

### Measurement Decision

The frontend should not attempt to recreate this backend compliance state.

The backend should emit the authoritative compliance transition.

---

# 18. Step 17 — Inspect Corporate Compliance Flow

Corporate KYC handling was inspected in:

```
```

```
logic/compliancekyc.go
```

The backend can transition KYC reviews into states including:

```
```

```
ACCEPTED
REJECTED
AWAITING
```

and other terminal/open states.

The effective corporate stakeholder status also accounts for request-scoped KYC and previously accepted person-linked KYC.

This means frontend status alone cannot reliably represent the final compliance state.

---

# 19. Step 18 — Define Backend Compliance Event

The primary backend product event identified by the measurement plan is:

```
```

```
compliance_state_changed
```

Its purpose is to represent an actual authoritative compliance/KYC state transition.

Proposed minimum properties:

```
```

```
event_id
occurred_at
event_name
event_version
journey
account_type
previous_state
new_state
```

Example semantic structure:

```
```

```
journey = onboarding
account_type = PERSONAL | CORPORATE
previous_state = AWAITING
new_state = PENDING
```

The exact event emission implementation remains an implementation-phase task.

---

# 20. Technical Changes Defined by the Measurement Plan

This task primarily produced the measurement design and implementation boundaries rather than introducing the complete telemetry runtime.

The technical areas identified for implementation are:

### Frontend

```
```

```
src/utils/axios.ts
src/utils/httpTrace.ts
src/main.ts
src/views/client/ChooseAccountType.vue
src/views/client/personal-account/AccountForm.vue
src/views/client/personal-account/...
src/views/client/corporate-account-registration/CorporateAccountForm.vue
src/views/onboarding/StakeholderKyc.vue
```

Existing telemetry utilities should be reused where applicable.

### Backend

Relevant areas identified:

```
```

```
logic/complianceindividualaccount.go
logic/compliancekyc.go
logic/kyc_request_resolution.go
db/repo/reviewrequestkyc.go
```

The backend compliance state transition is the source for authoritative compliance telemetry.

---

# 21. Design Decisions

## Decision 1 — Separate Operational and Product Telemetry

### What

Telemetry is divided into:

```
```

```
Operational Observability
```

and:

```
```

```
Product / Process Analytics
```

### Why This Fits Here

They answer different questions and require different storage/processing paths.

Operational signals belong in the observability stack:

```
```

```
Loki
Tempo
VictoriaMetrics
Grafana
```

Product/process analytics belongs in:

```
```

```
PostgreSQL
```

This prevents business analytics from being mixed with operational logs.

---

## Decision 2 — Use PostgreSQL for Initial Product Analytics

### What
Product/process telemetry will initially be stored in PostgreSQL.

### Why This Fits Here

The current expected volume is small, with approximately 100 users.

The existing direction explicitly avoids introducing an S3/data-lake architecture at this stage.

This keeps the implementation simple and aligned with the current scale.

---

## Decision 3 — Backend Compliance State Is Authoritative

### What

Frontend events describe the user's journey.

Backend events describe authoritative compliance state.

### Why This Fits Here

Compliance decisions and state transitions occur in the backend.

The frontend can show a state, but it should not become the source of truth for compliance analytics.

---

## Decision 4 — Track Meaningful User Actions

### What

Telemetry should be attached to actual journey actions and successful state transitions.

Examples:

```
```

```
account_type_selected
onboarding_form_submitted
corporate_application_submitted
liveness_verification_completed
```

### Why This Fits Here

A route visit does not necessarily mean that a user completed a business step.

For example, displaying a success component does not automatically mean that onboarding is complete.

Events should represent meaningful process milestones.

---

## Decision 5 — Do Not Capture Form Values

### What

Telemetry does not capture submitted form values.

### Why This Fits Here

The application handles sensitive onboarding and compliance information.

Collecting complete form contents would unnecessarily increase privacy and GDPR exposure without being required to answer the defined product questions.

---

## Decision 6 — Reuse Existing Infrastructure

Existing components should be reused where possible:

```
```

```
Axios interceptor
httpTrace utility
verificationSessionTelemetry
Vue global error handler
```

### Why This Fits Here

These components already own the relevant concerns.

Telemetry should integrate with existing application boundaries instead of creating duplicate infrastructure.

---

# 22. Data Flow / Component Interaction

## Product Analytics

```
```

```
Frontend Journey
      │
      ├── account_type_selected
      ├── identity_verification_completed
      ├── onboarding_form_submitted
      ├── liveness_verification_completed
      └── corporate_application_submitted
      │
      ▼
Telemetry Ingestion
      │
      ▼
PostgreSQL
      │
      ▼
Grafana
```

Backend compliance:

```
```

```
Backend Compliance Logic
      │
      └── compliance_state_changed
                │
                ▼
        Telemetry Ingestion
                │
                ▼
           PostgreSQL
                │
                ▼
             Grafana
```

## Operational Telemetry

```
```

```
Frontend
   │
   ├── Vue errors
   ├── JS exceptions
   ├── Promise rejections
   ├── API failures
   ├── Web Vitals
   └── release metadata
   │
   ▼
Collector
   │
   ├── Loki
   ├── Tempo
   └── VictoriaMetrics
           │
           ▼
        Grafana
```

---

# 23. Privacy and Security Rules

The following information is explicitly prohibited from telemetry.

### Never Capture

-  Form values 
-  Documents 
-  Passport/ID information 
-  Tax information 
-  Names 
-  Email addresses 
-  Phone numbers 
-  Addresses 
-  Account numbers 
-  User IDs 
-  Person IDs 
-  Stakeholder IDs 
-  Request IDs 
-  Tokens 
-  Credentials 
-  Free-text fields 
-  Request bodies 
-  Response bodies 
-  URL query parameters 
-  Arbitrary console output 
-  Session recordings 
-  Mouse tracking 
-  Bot-detection data 

### Error Data

Error messages and stack traces must be passed through redaction before storage or transmission.

Production source maps must remain private.

---

# 24. Performance Considerations

Telemetry must not block the primary user journey.

Important principles:

-  Do not make telemetry a prerequisite for navigation. 
-  Do not block account submission on analytics. 
-  Do not add unnecessary API calls to individual form fields. 
-  Reuse centralized infrastructure. 
-  Capture only required properties. 
-  Avoid high-volume arbitrary console collection. 

Existing verification telemetry already follows a telemetry-only approach that should not block UX.

---

# 25. Retention and Access Control

Retention must be explicitly agreed before production rollout.

The plan requires separate retention decisions for:

-  Operational logs 
-  Traces 
-  Metrics 
-  Product analytics 

Retention values were intentionally **not invented** during this task.

Access control must also be established for:

-  Raw operational telemetry 
-  Telemetry PostgreSQL data 
-  Grafana dashboards 
-  Traces 
-  Detailed error information 

---

# 26. Out of Scope

The following are outside the initial implementation scope:

-  S3 
-  Data lake 
-  Kafka unless later justified 
-  Session replay 
-  Mouse tracking 
-  Bot detection 
-  Arbitrary console collection 
-  Form-value collection 
-  Document collection 
-  PII collection 
-  Request-body collection 
-  Response-body collection 
-  Token collection 
-  URL query-parameter collection 

---

# 27. Validation and Verification

The measurement phase was validated by inspecting the actual application and backend implementation.

## Frontend Validation

Verified:

-  Existing onboarding routes and components. 
-  Actual account-type selection handler. 
-  Actual personal onboarding flow. 
-  Actual corporate application submission flow. 
-  Existing liveness flow. 
-  Existing verification telemetry. 
-  Centralized Axios error handling. 
-  Existing trace-header extraction. 
-  Global Vue error handler. 
-  Absence of Web Vitals implementation. 
-  Absence of frontend release-version implementation. 
-  Absence of global unhandled-error/rejection handlers. 

## Backend Validation

Verified:

- `dao.ReviewKindOnboarding`. 
-  Actual onboarding compliance state handling. 
-  Personal compliance status transitions. 
-  Corporate KYC state transitions. 
-  Corporate effective stakeholder KYC status logic. 
-  Repository persistence of review/KYC statuses. 
-  Existing accepted/rejected/awaiting/cancelled state handling. 

No telemetry implementation details were invented where the existing code did not provide them.

---

# 28. Important Edge Cases

## Personal vs Corporate

The telemetry model must distinguish:

```
```

```
PERSONAL
```

from:

```
```

```
CORPORATE
```

because their onboarding flows and backend compliance logic differ.

## Compliance Review Scope

Only:

```
```

```
ReviewKindOnboarding
```

should be treated as onboarding compliance analytics.

Manual or unrelated review activity must not be mixed into the onboarding funnel.

## Accepted vs Pending

The backend explicitly distinguishes:

```
```

```
ACCEPTED
```

from:

```
```

```
PENDING
```

Telemetry must preserve these states rather than normalizing them into a generic success state.

## Liveness-Only Flow

A liveness-only journey must not be incorrectly counted as a complete new-account onboarding journey.

## Missing Trace ID

Trace correlation is optional.

If a trace identifier is unavailable, telemetry should still be usable without inventing an identifier.

## Error Redaction

Raw errors must never bypass the redaction layer.

---

# 29. Final Implementation State

The **Measurement Plan Phase is complete**.

The completed work established:

-  Defined product/process questions. 
-  Separated operational telemetry from product analytics. 
-  Mapped the actual frontend onboarding flows. 
-  Identified meaningful frontend journey events. 
-  Identified existing telemetry that should be reused. 
-  Identified the centralized Axios API-failure integration point. 
-  Identified the existing trace-correlation mechanism. 
-  Identified Web Vitals as a required operational signal. 
-  Identified release version as a required operational property, without inventing its source. 
-  Identified the global Vue error-handling location. 
-  Inspected backend onboarding compliance logic. 
-  Identified authoritative compliance states. 
-  Identified `compliance_state_changed` as the primary backend compliance telemetry event. 
-  Defined minimum telemetry properties. 
-  Defined strict privacy boundaries. 
-  Defined PostgreSQL as the initial product analytics store. 
-  Defined Grafana as the reporting layer. 
-  Defined Loki/Tempo/VictoriaMetrics for operational observability. 
-  Explicitly excluded S3/data lake and unnecessary high-volume tracking from the initial scope. 
-  Identified retention and access control as decisions required before production rollout.

---

__TASK3__
