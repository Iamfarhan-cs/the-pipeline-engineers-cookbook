# Task 1 — Frontend Telemetry Event Boundary

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

# Payment Telemetry Pipeline

## Phase: Frontend Telemetry Foundation & Journey Events

**Branch:** `feature/frontend-telemetry`
**Commit:** `697487e2c`
**Commit Message:** `feat: add frontend telemetry foundation and journey events`
**Status:** Completed

---

# 1. Task Overview

## Objective

This phase implemented the first frontend telemetry layer for the Payment Telemetry Pipeline.

The objective was to create a reusable and privacy-safe telemetry foundation and use it to track meaningful user journey events in the frontend.

The implementation focused on two journey events:

- `account_type_selected`
- `identity_verification_completed`

The telemetry foundation provides a common event structure, event ID generation, timestamps, route information, property sanitization, and a browser event boundary.

The implementation was intentionally limited to meaningful journey milestones. It does not attempt to track every user interaction.

## Expected Direction

The broader telemetry architecture is:

```text
Frontend Product Events
        +
Backend Authoritative State
        ↓
Telemetry Ingestion
        ↓
PostgreSQL
        ↓
Grafana

```

This phase only implemented the frontend event-generation portion.

Database ingestion, PostgreSQL telemetry storage, Grafana dashboards, and backend authoritative telemetry were not implemented in this phase.

---

# 2. Implementation Sequence

The implementation was completed incrementally in the following order.

## Step 1 — Create the Telemetry Foundation

Created:

```text
src/telemetry/index.ts

```

The purpose of this file is to provide one common telemetry API instead of implementing event creation separately inside every Vue component.

The foundation introduced:

- telemetry property types
- product event names
- operational event names
- event creation
- event IDs
- timestamps
- route capture
- property sanitization
- browser event emission

The main functions are:

```text
createTelemetryEvent()
trackEvent()

```

---

## Step 2 — Define the Telemetry Event Structure

The telemetry event structure was defined as:

```ts
type TelemetryEvent = {
  event_id: string
  event_name: string
  event_version: number
  occurred_at: string
  route?: string
  properties?: TelemetryProperties
}

```

Every telemetry event therefore has:

- `event_id`
- `event_name`
- `event_version`
- `occurred_at`
- optional `route`
- optional `properties`

The initial event version is:

```text
1

```

This provides a stable structure that can later be extended without changing the basic event contract.

---

## Step 3 — Add Centralized Privacy Redaction

Created:

```text
src/telemetry/redact.ts

```

and:

```text
src/telemetry/redact.spec.ts

```

The redaction layer was added before instrumenting application events so that event callers do not have to implement their own sensitive-data protection.

Sensitive values are replaced with:

```text
[REDACTED]

```

The implementation covers sensitive patterns including:

- bearer tokens
- access tokens
- refresh tokens
- ID tokens
- authorization values
- passwords
- secrets
- API keys
- client secrets
- email values
- phone values
- selected sensitive query parameters

An error-specific helper was also added:

```text
redactError()

```

It returns sanitized error information rather than exposing raw sensitive error content.

---

## Step 4 — Add Telemetry Foundation Tests

Created:

```text
src/telemetry/index.spec.ts

```

The tests verify:

- telemetry event creation
- event ID generation
- timestamp creation
- route handling
- browser `telemetry_event` emission
- sensitive property redaction

Redaction tests verify:

- bearer token removal
- sensitive key/value removal
- sensitive query parameter removal
- error message and stack sanitization

The foundation and redaction tests passed:

```text
7/7 passed

```

---

## Step 5 — Discover the Canonical Account Type

Before implementing the account-selection event, the existing frontend account type was checked.

The canonical definition is in:

```text
src/types/accounts.ts

```

The type is:

```ts
export type AccountType = 'PERSONAL' | 'CORPORATE'

```

The existing account-selection flow was located in:

```text
src/views/client/ChooseAccountType.vue

```

The existing selection function is:

```text
applyForNewAccount(accountType)

```

This existing function was used as the telemetry insertion point.

---

# 3. Technical Changes

## 3.1 Telemetry Event Names

The telemetry foundation defines product events including:

```text
account_type_selected
identity_verification_completed
onboarding_form_submitted
liveness_verification_started
liveness_verification_completed
corporate_application_submitted

```

Operational event names were also defined:

```text
frontend_error
frontend_unhandled_exception
frontend_unhandled_rejection
frontend_console_error
frontend_console_warn
frontend_api_error

```

Only the first two product events were implemented in this phase.

The remaining event names are part of the telemetry contract but were not instrumented yet.

---

# 4. Account Type Selection Implementation

## File Changed

```text
src/views/client/ChooseAccountType.vue

```

The telemetry event was added to the existing account-selection function:

```ts
trackEvent('account_type_selected', {
  account_type: accountType,
})

```

## Event Payload

The event contains:

```text
account_type

```

Possible values are:

```text
PERSONAL
CORPORATE

```

No additional account information is captured.

## Why This Fits Here

The `applyForNewAccount()` function is already the canonical point where the selected account type reaches the application logic.

Putting telemetry here means:

1. The event represents the actual account selection.
2. Both Personal and Corporate selections use the same implementation point.
3. No duplicate click handlers are required.
4. The telemetry code does not need to know how the visual option card works.
5. The existing navigation/business logic remains unchanged.

This is preferable to attaching telemetry directly to the UI component because the function already represents the application's account-selection action.

---

# 5. Account Selection Test

Created:

```text
src/views/client/ChooseAccountType.spec.ts

```

The focused test verifies that selecting Personal emits:

```text
account_type_selected

```

with:

```text
account_type: PERSONAL

```

The test observes the existing browser telemetry event boundary.

## Result

```text
1/1 passed

```

---

# 6. Identity Verification Implementation

## File Changed

```text
src/views/client/verification/IndividualVerification.vue

```

The actual identity submission flow was inspected before adding telemetry.

The relevant function is:

```text
handleNextClick()

```

The function validates:

- phone
- recovery email
- tax information
- residency
- citizenships

If validation fails, the existing error behavior continues and telemetry is not emitted.

For Personal accounts, the existing flow calls:

```text
persistIdentityDraft({
  mode: 'submit',
  showError: true
})

```

The telemetry event was placed after the persistence operation successfully completes.

---

# 7. Identity Verification Event

The implemented event is:

```ts
trackEvent('identity_verification_completed', {
  account_type: accountType.value,
})

```

The event is emitted only after:

```text
saved === true

```

The existing router navigation then continues.

Conceptually, the flow is:

```text
User clicks Next
        ↓
Validate required fields
        ↓
Validation successful?
   ┌────┴────┐
   No        Yes
   ↓          ↓
Show error   Persist identity submission
              ↓
          Save successful?
           ┌────┴────┐
           No        Yes
           ↓          ↓
        Stop       Emit telemetry
                      ↓
                 Existing navigation

```

---

# 8. Why Telemetry Is After Successful Persistence

The telemetry event is named:

```text
identity_verification_completed

```

Therefore, it should not represent simply clicking the Next button.

If telemetry were emitted before the persistence request succeeded, the event could report completion even when the backend save failed.

The implementation therefore uses the existing success result:

```text
saved === true

```

as the condition for emitting the event.

This makes the event represent the successful application flow rather than an attempted action.

---

# 9. Identity Verification Failure Handling

The implementation intentionally does not emit the completion event when:

### Validation fails

If any required validation fails:

```text
identity_verification_completed

```

is not emitted.

### Persistence fails

If:

```text
persistIdentityDraft()

```

does not successfully save the identity information:

```text
identity_verification_completed

```

is not emitted.

### Successful submission

Only after:

```text
saved === true

```

is the event emitted.

This avoids reporting false-positive completion events.

---

# 10. Identity Verification Test Changes

Updated:

```text
src/views/client/verification/__tests__/IndividualVerification.spec.ts

```

A focused test was added for successful identity submission.

The test verifies:

- the identity form loads
- the Next button exists
- validation succeeds
- the PUT request is performed
- the telemetry event is emitted
- the event name is correct
- the event version is `1`
- the account type is `PERSONAL`
- existing router navigation still happens

Supporting mocks were updated so the child validation components return successful validation results.

The identity store mock was also updated so the loaded identity data is reflected in the reactive test state.

## Result

```text
13/13 passed

```

---

# 11. Data Flow / Component Interaction

## Before This Phase

The frontend already contained the application journey and existing API/navigation logic, but there was no common implementation for the new telemetry event contract.

Individual components handled their normal application behavior independently.

---

## After This Phase

The flow is:

```text
Vue Component
     ↓
Existing application action
     ↓
trackEvent()
     ↓
createTelemetryEvent()
     ↓
sanitizeProperties()
     ↓
Browser CustomEvent
     ↓
telemetry_event

```

The important point is that telemetry is generated from the actual application action.

For example:

```text
ChooseAccountType.vue
        ↓
applyForNewAccount()
        ↓
trackEvent('account_type_selected')
        ↓
Telemetry Event

```

And:

```text
IndividualVerification.vue
        ↓
handleNextClick()
        ↓
validation
        ↓
persistIdentityDraft()
        ↓
successful save
        ↓
trackEvent('identity_verification_completed')
        ↓
Telemetry Event
        ↓
existing router navigation

```

---

# 12. Why a Browser CustomEvent Is Used

The current telemetry foundation emits:

```text
telemetry_event

```

using:

```ts
window.dispatchEvent(
  new CustomEvent('telemetry_event', {
    detail: event,
  })
)

```

## Why This Fits Here

At this phase, the telemetry contract needed to be separated from the eventual transport mechanism.

Using a browser event provides:

- a simple local event boundary
- easy testability
- no dependency on a backend ingestion endpoint
- no coupling between Vue components and a future transport implementation

This allows frontend journey events to be implemented and tested before the ingestion layer is finalized.

The external transport is therefore intentionally deferred to a later phase.

---

# 13. Route Handling

The telemetry foundation captures:

```text
window.location.pathname

```

The pathname is used rather than the complete URL.

This prevents query-string values from being directly included in the route field.

The telemetry foundation tests also verify that sensitive query information is not retained in the route.

---

# 14. Existing Infrastructure Considerations

The implementation was designed around existing frontend infrastructure.

## HTTP Trace Utility

Existing:

```text
src/utils/httpTrace.ts

```

This utility already handles request/trace ID extraction.

A new trace extraction implementation was not created.

---

## Axios

Existing:

```text
src/utils/axios.ts

```

contains authentication refresh/retry behavior.

Future API-error telemetry must account for this behavior so that recoverable intermediate `401` responses are not incorrectly counted as final failures.

This was not implemented in this phase.

---

## Verification Telemetry

Existing:

```text
src/utils/verificationSessionTelemetry.ts

```

already provides verification-related telemetry/session state functionality.

A duplicate implementation was not introduced.

---

## Existing Logger

Existing:

```text
src/utils/logger.ts

```

was not changed.

The logger is not used as the telemetry privacy boundary because raw error message/stack information can still be preserved by that logging path.

The new telemetry system therefore performs its own redaction before emitting telemetry.

---

# 15. Design Decisions

| DecisionImplementationWhy                      |                                                |                                                                    |
| ---------------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------ |
| Central telemetry module                       | `src/telemetry/index.ts`                       | Keeps event creation consistent                                    |
| Version every event                            | `event_version: 1`                             | Provides a stable event contract                                   |
| Generate unique event IDs                      | `crypto.randomUUID()` when available           | Gives each event a unique identifier                               |
| Capture pathname only                          | `window.location.pathname`                     | Avoids putting query values into route telemetry                   |
| Centralized redaction                          | `src/telemetry/redact.ts`                      | Prevents each caller from implementing privacy rules independently |
| Use existing application functions             | `applyForNewAccount()` and `handleNextClick()` | Tracks actual business actions rather than raw UI clicks           |
| Emit identity completion after successful save | `saved === true`                               | Prevents false completion events                                   |
| Use browser CustomEvent                        | `telemetry_event`                              | Provides a transport-neutral and testable boundary                 |
| Keep payloads minimal                          | `account_type` only for these events           | Reduces unnecessary data collection                                |
| Do not implement ingestion yet                 | Deferred                                       | Keeps this phase focused on frontend event generation              |

---

# 16. Security and Privacy Considerations

Privacy was treated as a core implementation requirement.

The telemetry system explicitly avoids collecting:

- passwords
- tokens
- authorization information
- email values
- phone values
- form values
- request bodies
- response bodies
- documents
- raw sensitive query values
- session replay data
- mouse/keyboard tracking

Sensitive strings are processed through:

```text
redactTelemetryText()

```

before being included in a telemetry event.

Errors can be processed through:

```text
redactError()

```

to sanitize messages and stack traces.

The telemetry boundary therefore does not rely on individual feature developers to manually sanitize every string.

---

# 17. Performance Considerations

The implementation is intentionally lightweight.

The current foundation:

- creates a small in-memory event object
- performs synchronous string redaction
- dispatches a browser `CustomEvent`
- does not make a network request
- does not add a telemetry dependency
- does not capture large payloads

This keeps the initial frontend telemetry layer low overhead.

Actual transport and batching considerations can be addressed when the ingestion layer is implemented.

---

# 18. Validation and Testing

The implementation was validated incrementally.

## Telemetry Foundation and Redaction

Result:

```text
7/7 passed

```

Covered:

- event creation
- event structure
- browser event emission
- sensitive property redaction
- bearer tokens
- sensitive key/value patterns
- query parameter redaction
- error message/stack sanitization

---

## Account Type Selection

Result:

```text
1/1 passed

```

Validated:

```text
account_type_selected

```

with:

```text
account_type: PERSONAL

```

---

## Identity Verification

Result:

```text
13/13 passed

```

Validated:

- successful submission
- telemetry emission
- correct event name
- correct event version
- correct account type
- API persistence
- existing navigation

---

# 19. Regression Considerations

The implementation was inserted into existing application paths instead of replacing them.

For account selection:

```text
existing account-selection behavior

```

continues after telemetry.

For identity verification:

```text
existing persistence
        ↓
telemetry
        ↓
existing navigation

```

The telemetry addition therefore does not replace the existing business logic.

The focused tests confirmed that existing navigation and API behavior continue to execute.

---

# 20. Edge Cases and Failure Handling

## Invalid Identity Form

If required validation fails:

```text
identity_verification_completed

```

is not emitted.

The existing validation error behavior remains active.

---

## Failed Identity Persistence

If the identity save operation fails:

```text
identity_verification_completed

```

is not emitted.

This prevents false completion metrics.

---

## Sensitive Property Passed to Telemetry

String properties pass through the centralized redaction layer.

Sensitive values are converted to:

```text
[REDACTED]

```

---

## Sensitive Query Parameters

The telemetry route uses:

```text
pathname

```

rather than the full URL.

Sensitive query values are also handled by the redaction layer when present in telemetry strings.

---

## Recoverable Axios Authentication Retry

The existing Axios layer performs authentication refresh/retry behavior.

No new API-error telemetry was added in this phase.

When API-error telemetry is implemented later, it must avoid counting recoverable intermediate authentication failures as final API failures.
---

# 21. Git and Commit

The work was completed on:

```text
feature/frontend-telemetry

```

The changes were staged and reviewed with:

```bash
git diff --cached --stat

```

The staged implementation contained:

```text
8 files changed

```

The final commit was:

```text
697487e2c

```

with:

```text
feat: add frontend telemetry foundation and journey events

```

Final commit result:

```text
8 files changed
416 insertions(+)
8 deletions(-)

```

The repository was not pushed to the remote as part of this task.

---

# 22. Files Added

```text
src/telemetry/index.ts
src/telemetry/index.spec.ts
src/telemetry/redact.ts
src/telemetry/redact.spec.ts
src/views/client/ChooseAccountType.spec.ts

```

---

# 23. Files Modified

```text
src/views/client/ChooseAccountType.vue

src/views/client/verification/IndividualVerification.vue

src/views/client/verification/__tests__/IndividualVerification.spec.ts

```

---

# 24. What Was Not Implemented

The following were intentionally left for later phases:

```text
onboarding_form_submitted

liveness_verification_started

liveness_verification_completed

corporate_application_submitted

compliance_state_changed

frontend global error instrumentation

unhandled exception instrumentation

unhandled rejection instrumentation

console.error / console.warn telemetry

frontend API failure telemetry

Web Vitals

backend telemetry ingestion

PostgreSQL telemetry storage

Grafana dashboards

```

No implementation should be assumed for these items until they are separately discovered and completed.

---

# 25. Next Implementation Step

The next telemetry event is:

```text
onboarding_form_submitted

```

The next implementation should follow the same process:

```text
1. Find the exact onboarding submission path.
2. Identify the successful submission point.
3. Identify the minimum approved non-PII properties.
4. Add trackEvent().
5. Add focused test coverage.
6. Run the relevant tests.
7. Verify existing behavior remains unchanged.
8. Commit the completed increment.

```

The next step should begin with targeted discovery only.

Do not broadly inspect unrelated frontend or backend code before the onboarding submission path is required.

---

# 26. Final Implementation State

At the end of this phase, the frontend has a reusable telemetry foundation with:

```text
Telemetry event contract
        ↓
Event versioning
        ↓
Unique event IDs
        ↓
Timestamp
        ↓
Sanitized route
        ↓
Centralized property redaction
        ↓
Browser telemetry_event boundary

```

Two meaningful product journey events are currently implemented:

```text
account_type_selected
identity_verification_completed

```

Both have automated test coverage.

The identity verification event is specifically tied to successful persistence rather than button interaction, which prevents false completion reporting.

The implementation is committed to:

```text
feature/frontend-telemetry

```

at:

```text
697487e2c

```

This completes the current frontend telemetry foundation phase and provides the base required for the next incremental journey event implementation.


---

# Payment Telemetry Pipeline — Frontend Telemetry Implementation

## 1. Task Overview

### Task

Implement the frontend telemetry layer for the Payment Telemetry Pipeline and add privacy-safe tracking for important user journey events in the Zolvat EMI frontend.

### Technical Objective

The objective of this implementation was to capture meaningful product journey milestones from the frontend without collecting sensitive user information.

The telemetry should answer questions such as:

- Which account type did the user select?
- Did the identity verification process complete successfully?
- Was the corporate onboarding form successfully submitted?
- Did a liveness verification session start?
- Did the liveness verification complete successfully?

The implementation must not behave like session replay or generic frontend logging.

The telemetry should contain **business/product events**, not:

- Form values
- KYC information
- Documents
- Passwords
- Tokens
- Authentication headers
- Request/response bodies
- Keystrokes
- Mouse movements
- Session replay data
- Other sensitive or personally identifiable information

### Implementation Principle

The work followed:

> **Discover on demand, implement continuously.**

Instead of performing broad discovery of the entire backend, database, DevOps, and Grafana infrastructure, the required frontend code was investigated only when necessary for the current implementation step.

---

# 2. Implementation Sequence

## Step 1 — Establish the telemetry foundation

The existing telemetry foundation was used as the central mechanism for creating and dispatching telemetry events.

The foundation provides:

- Event ID generation
- Event name
- Event version
- Timestamp
- Current frontend route
- Optional properties
- Property sanitization
- Browser event dispatch

The main implementation is located under:

```text
src/telemetry/

```

The main API is:

```text
trackEvent(...)

```

Events are dispatched through:

```text
CustomEvent('telemetry_event')

```

This provides a frontend-level event boundary that can later be connected to the actual telemetry transport.

---

## Step 2 — Add account type selection telemetry

The account selection flow was inspected and the telemetry event was placed at the actual account-type selection boundary.

Event:

```text
account_type_selected

```

The implementation uses the application's existing canonical account types, such as:

```text
PERSONAL
CORPORATE

```

The event is generated when the user actually selects an account type.

### Why this was implemented here

The account-type component is the layer where the user decision actually occurs.

Tracking it here means:

```text
User selects account type
        ↓
account_type_selected

```

rather than attempting to infer the selection later from navigation or API calls.

This gives the telemetry a clear relationship with the user's actual product interaction.

---

## Step 3 — Add identity verification completion telemetry

The identity verification flow was inspected to find the actual successful submission boundary.

The implementation is in:

```text
src/views/client/verification/IndividualVerification.vue

```

Event:

```text
identity_verification_completed

```

The event is emitted only after:

```text
persistIdentityDraft({
    mode: 'submit',
    showError: true
})

```

successfully completes.

### Why this boundary was selected

A button click does not prove that identity verification was successfully completed.

The existing persistence operation provides a stronger business boundary:

```text
User submits verification
        ↓
persistIdentityDraft(...)
        ↓
Successful result
        ↓
identity_verification_completed

```

If persistence fails, the event is not emitted.

This prevents false-positive completion telemetry.

---

## Step 4 — Investigate the corporate onboarding submission flow

The corporate onboarding component was inspected to identify where the application is actually considered successfully submitted.

Relevant component:

```text
src/views/client/corporate-account-registration/CorporateAccountForm.vue

```

The relevant function is:

```text
submitApplicationForReview()

```

The existing flow performs:

```text
forceSaveDeclarationsNow()

```

followed by:

```text
PUT /user/accounts/create/corporate/:requestId

```

The response is then checked for validation tabs/errors.

Telemetry was not placed on the submit button.

Instead, the successful response path was selected.

---

## Step 5 — Add corporate onboarding telemetry

Event:

```text
onboarding_form_submitted

```

The event was added inside the successful branch of the corporate submission API flow.

The resulting sequence is:

```text
User submits corporate onboarding
        ↓
forceSaveDeclarationsNow()
        ↓
PUT /user/accounts/create/corporate/:requestId
        ↓
Server response
        ↓
Check validation tabs/errors
        ↓
No validation errors
        ↓
Update existing signatory state
        ↓
trackEvent('onboarding_form_submitted')
        ↓
Navigate to stakeholder links page

```

### Why this boundary was selected

The API response is already the existing business decision point that determines whether the corporate onboarding submission succeeded.

Placing telemetry there means the event represents:

> The application was successfully submitted.

It does not represent:

> The user clicked Submit.

This distinction is important because the API may reject the submission due to validation errors.

---

## Step 6 — Investigate the liveness verification start boundary

The liveness implementation was inspected to determine where a reliable "start" event could be generated.

Relevant component:

```text
src/components/FaceLivenessReact.jsx

```

The component creates a liveness session through the existing API flow.

The response contains:

```text
session_id

```

The telemetry event is generated only when the liveness session creation succeeds and a valid `session_id` is present.

Event:

```text
liveness_verification_started

```

The flow is:

```text
Create liveness session
        ↓
API succeeds
        ↓
session_id exists
        ↓
liveness_verification_started

```

### Why this boundary was selected

The component does not expose a separate explicit user-start callback that could be used as a reliable telemetry boundary.

A successfully created liveness session is therefore the earliest reliable implementation point available in the current component.

The actual session ID is not added to the telemetry properties.

---

## Step 7 — Investigate the liveness completion condition

The liveness completion flow was inspected before adding telemetry.

The existing helper:

```text
isSuccessfulLivenessResult(...)

```

was checked to determine what the application considers a successful liveness result.

The actual success condition uses:

```text
message: "SUCCEEDED"

```

This was important because using a different field such as:

```text
status: "SUCCEEDED"

```

would not match the existing application behavior.

---

## Step 8 — Add liveness completion telemetry

Relevant component:

```text
src/views/client/verification/livenessVerification.vue

```

Event:

```text
liveness_verification_completed

```

The event is emitted only after:

```text
isSuccessfulLivenessResult(data)

```

returns true.

The resulting flow is:

```text
Liveness callback
        ↓
isSuccessfulLivenessResult(data)
        ↓
Successful result
        ↓
scanSuccess = true
        ↓
liveness_verification_completed
        ↓
Existing liveness success processing continues

```

### Why this boundary was selected

The existing helper already defines the application's successful liveness condition.

Reusing that condition avoids duplicating or changing business logic inside the telemetry implementation.

Telemetry observes the existing success state instead of defining a second success rule.

---

## Step 9 — Investigate `corporate_application_submitted`

The telemetry event definitions contained:

```text
corporate_application_submitted

```

Before implementing it, the frontend source was searched for a real application-submission boundary.

The search found:

```text
src/telemetry/index.ts

```

but no actual usage elsewhere.

The corporate form also contained:

```text
const applicationSubmitted = ref(false)

```

but the inspected implementation did not contain:

```text
applicationSubmitted = true

```

or:

```text
applicationSubmitted.value = true

```

### Final decision

`corporate_application_submitted` was **not artificially implemented**.

The existing:

```text
onboarding_form_submitted

```

already represents the successful corporate onboarding submission boundary.

Adding another event to the same action would produce duplicate telemetry for the same business milestone.

### Design rule

Do not create telemetry events merely because an event name exists in a type definition.

A telemetry event should correspond to a real product/business boundary.

---

# 3. Technical Changes

## 3.1 Frontend telemetry foundation

The telemetry foundation provides:

```text
event_id
event_name
event_version
occurred_at
route
properties

```

The event version is currently:

```text
1

```

The event ID is generated using the browser UUID mechanism.

The timestamp uses an ISO-formatted timestamp.

The route is based on the current browser pathname.

---

## 3.2 Telemetry event dispatch

Events are dispatched through:

```text
CustomEvent('telemetry_event')

```

The event is available to frontend consumers without coupling individual product components to a future backend transport implementation.

This keeps product components focused on:

```text
When did this business event happen?

```

while the telemetry foundation handles:

```text
How is the event represented and dispatched?

```

---

## 3.3 Privacy sanitization

Telemetry properties are sanitized through the centralized redaction mechanism.

The redaction layer covers sensitive patterns including:

- Bearer/access tokens
- Refresh tokens
- ID tokens
- Authentication values
- Passwords
- Secrets
- API keys
- Client secrets
- Email addresses
- Phone numbers
- Sensitive query parameters

This provides a common privacy boundary instead of requiring every component to implement its own redaction logic.

---

## 3.4 Product event implementation

The current implemented product events are:

| EventLocation / BoundaryPurpose   |                                              |                                                     |
| --------------------------------- | -------------------------------------------- | --------------------------------------------------- |
| `account_type_selected`           | Account type selection component             | Records account type selection                      |
| `identity_verification_completed` | Successful identity draft persistence        | Records successful identity verification completion |
| `onboarding_form_submitted`       | Successful corporate onboarding API response | Records successful corporate form submission        |
| `liveness_verification_started`   | Successful liveness session creation         | Records liveness session start                      |
| `liveness_verification_completed` | Successful liveness result                   | Records successful liveness completion              |

---

# 4. Design Decisions

## Decision 1 — Use business-event boundaries instead of UI clicks

### What was implemented

Telemetry was placed at successful application/business boundaries.

Examples:

```text
identity_verification_completed

```

is emitted after successful persistence rather than when the user clicks Submit.

Similarly:

```text
onboarding_form_submitted

```

is emitted after successful server validation rather than at button click.

### Why this fits here

UI clicks do not necessarily represent successful business operations.

A click can be followed by:

- Validation errors
- API failure
- Authentication failure
- Network failure
- Other application errors

Tracking the successful operation gives more meaningful telemetry.

---

## Decision 2 — Keep telemetry properties minimal

### What was implemented

The newly added journey events do not send sensitive form values or request data.

Several events intentionally have no properties.

### Why this fits here

The purpose of these events is to measure product journey progression, not collect user content.

For example:

```text
liveness_verification_completed

```

only needs to indicate that liveness completed successfully.

It does not need:

- document data
- biometric information
- session details
- user information

This reduces privacy risk and keeps events lightweight.

---

## Decision 3 — Centralize redaction

### What was implemented

Sensitive-value sanitization is handled by the shared telemetry foundation.

### Why this fits here

Redaction belongs at the telemetry boundary because this is the last controlled point before telemetry leaves the feature-specific code.

This protects against accidental leakage when telemetry properties are introduced later.

It also avoids duplicating redaction logic across components.

---

## Decision 4 — Reuse existing application success logic

### What was implemented

Liveness completion telemetry uses:

```text
isSuccessfulLivenessResult(...)

```

instead of creating a new success condition.

Corporate onboarding telemetry uses the existing API validation/result flow.

Identity verification telemetry uses the existing persistence result.

### Why this fits here

The application already has established business rules.

Telemetry should observe those rules rather than redefine them.

This reduces the chance of telemetry becoming inconsistent with actual product behavior.

---

## Decision 5 — Do not implement duplicate corporate application telemetry

### What was implemented

`corporate_application_submitted` was left unused because there is currently no distinct frontend business boundary for it.

### Why this fits here

`onboarding_form_submitted` already represents successful corporate onboarding submission.

Adding both events to the same action would make downstream analytics ambiguous and potentially count one business action twice.

---

# 5. Data Flow / Component Interaction

## Before this implementation

The frontend product components performed their existing business actions:

```text
User interaction
      ↓
Vue/React component
      ↓
Existing application logic
      ↓
API / state update

```

There was no complete product-event telemetry signal at each of these journey boundaries.

---

## After this implementation

The flow becomes:

```text
User interaction
      ↓
Frontend component
      ↓
Existing business logic
      ↓
Successful business boundary
      ↓
trackEvent(...)
      ↓
TelemetryEvent
      ↓
CustomEvent('telemetry_event')

```

The product flow remains responsible for the actual business operation.

The telemetry layer only records the resulting product event.

---

## Example — Corporate onboarding

```text
CorporateAccountForm.vue
        |
        | submitApplicationForReview()
        v
forceSaveDeclarationsNow()
        |
        v
PUT /user/accounts/create/corporate/:requestId
        |
        v
Server response
        |
        +---- validation errors ----> existing error handling
        |
        +---- success -------------> trackEvent()
                                     |
                                     v
                               onboarding_form_submitted
                                     |
                                     v
                                 router.push(...)

```

This means telemetry does not interfere with the existing submission flow.

---

# 6. Validation and Testing

## 6.1 Telemetry foundation tests

Validated:

- Telemetry event creation
- Event structure
- Event dispatch
- Event versioning
- Redaction behavior

---

## 6.2 Redaction tests

Validated sensitive-value handling for the centralized telemetry redaction logic.

The purpose is to ensure that sensitive values are not accidentally included in telemetry properties.

---

## 6.3 Corporate onboarding test

A new test was created:

```text
src/views/client/corporate-account-registration/__tests__/CorporateAccountForm.spec.ts

```

The test validates that:

1. Corporate onboarding submission succeeds.
2. The successful response does not contain validation tabs.
3. Existing submission processing runs.
4. `onboarding_form_submitted` is emitted.

---

## 6.4 Liveness start test

Updated:

```text
src/components/__tests__/FaceLivenessReact.spec.jsx

```

The test validates:

- A successful liveness session creation produces the start event.
- Exactly one telemetry event is emitted.
- The event name is correct.
- Event version is correct.
- No telemetry properties are attached.

---

## 6.5 Liveness completion test

Updated:

```text
src/views/client/verification/__tests__/LivenessVerification.spec.ts

```

The test invokes the existing callback with:

```text
{ message: 'SUCCEEDED' }

```

and validates:

```text
trackEvent('liveness_verification_completed')

```

is called exactly once.

---

## 6.6 Final targeted test run

The complete targeted telemetry suite was executed with:

```bash
npx vitest run \
src/telemetry/index.spec.ts \
src/telemetry/redact.spec.ts \
src/views/client/choose-account-type/__tests__/ChooseAccountType.spec.ts \
src/views/client/verification/__tests__/IndividualVerification.spec.ts \
src/views/client/corporate-account-registration/__tests__/CorporateAccountForm.spec.ts \
src/views/client/verification/__tests__/LivenessVerification.spec.ts \
src/components/__tests__/FaceLivenessReact.spec.jsx

```

Final result:

```text
Test Files  6 passed (6)
Tests       25 passed (25)

```

### Result

**25/25 tests passed.**

No telemetry test failures occurred.

---

# 7. Edge Cases and Failure Handling

## Identity verification failure

If:

```text
persistIdentityDraft(...)

```

fails, the completion event is not emitted.

This prevents:

```text
identity_verification_completed

```

from being reported for a failed submission.

---

## Corporate onboarding validation failure

If the corporate API response contains validation tabs/errors:

```text
tabs.length > 0

```

the existing validation handling continues.

Telemetry is not emitted.

Therefore:

```text
Invalid submission
        ↓
Validation errors
        ↓
No onboarding_form_submitted

```

---

## Corporate onboarding API failure

If the API request rejects:

```text
.catch(...)

```

handles the failure.

The successful onboarding telemetry event is not emitted.

---

## Declaration-save failure

Before the corporate API request, the existing flow calls:

```text
forceSaveDeclarationsNow()

```

If this operation throws, the function returns before submitting the application.

Therefore:

```text
Declaration save failure
        ↓
No application submission
        ↓
No onboarding_form_submitted

```

---

## Liveness session creation failure

The liveness start event is only emitted when:

```text
data?.session_id

```

exists after successful session creation.

A failed session creation does not generate:

```text
liveness_verification_started

```

---

## Liveness verification failure

The completion event is only emitted when:

```text
isSuccessfulLivenessResult(data)

```

returns true.

Failed liveness results do not produce:

```text
liveness_verification_completed

```

---

## Duplicate corporate event prevention

`corporate_application_submitted` was not added because no distinct business boundary exists.

This prevents duplicate telemetry for the same submission action.

---

# 8. Security, Privacy and Performance Considerations

## Privacy

The telemetry implementation is intentionally privacy-safe.

The new journey events do not capture:

- PII
- KYC data
- Documents
- Passwords
- Tokens
- Authentication headers
- Request bodies
- Response bodies
- Session replay
- Keystrokes
- Mouse movement

---

## Security

The centralized redaction layer protects telemetry properties from known sensitive-value patterns.

This is important because telemetry can eventually leave the browser and enter external pipeline infrastructure.

Sensitive values should therefore be removed before dispatch rather than relying on downstream systems to remove them.

---

## Performance

The events are lightweight product events.

The implementation does not introduce:

- continuous event streams for mouse movement
- keystroke capture
- DOM mutation recording
- session replay
- large request/response payloads

The events contain only the information required to identify the product journey milestone.

---

# 9. Git / Version Control

The implementation was completed on:

```text
feature/frontend-telemetry

```

The changes were staged and committed.

Commit:

```text
6e6579888

```

Commit message:

```text
feat: add frontend journey telemetry events

```

Git hooks completed successfully during the commit process.

Remote push was not performed.

---

# 10. Final Implementation State

The frontend telemetry phase is complete.

The frontend currently provides telemetry for:

```text
account_type_selected
identity_verification_completed
onboarding_form_submitted
liveness_verification_started
liveness_verification_completed

```

The implementation:

- Uses a shared telemetry foundation.
- Uses versioned telemetry events.
- Dispatches browser telemetry events through `CustomEvent`.
- Sanitizes telemetry properties.
- Tracks meaningful business boundaries.
- Avoids sensitive user data.
- Handles failed operations without generating false success events.
- Includes automated test coverage.
- Passed 25/25 targeted telemetry tests.
- Has been committed to Git.

The event:

```text
corporate_application_submitted

```

remains defined but intentionally unused because there is currently no distinct frontend business boundary for it.

---

# 11. Final Acceptance Checklist

| RequirementStatus                 |               |
| --------------------------------- | ------------- |
| Frontend telemetry foundation     | Complete      |
| Privacy-safe redaction            | Complete      |
| Account type telemetry            | Complete      |
| Identity verification telemetry   | Complete      |
| Corporate onboarding telemetry    | Complete      |
| Liveness start telemetry          | Complete      |
| Liveness completion telemetry     | Complete      |
| Automated tests                   | Complete      |
| Targeted test result              | 25/25 passed  |
| Duplicate corporate event avoided | Complete      |
| Git commit                        | Complete      |
| Remote push                       | Not performed |

## Final Status

**Frontend Telemetry Implementation Phase — COMPLETE**

The frontend is now ready for the next pipeline stage, where the existing `telemetry_event` output can be connected to the downstream telemetry ingestion/transport layer.

The next stage should be implemented using the same principle:

> **Discover only what is required for the next implementation step, then continue building the pipeline incrementally.**

---

# Payment Telemetry Pipeline — Backend Frontend Telemetry Ingestion

## 1. Task Overview

### Objective

Implement the **backend ingestion layer** for frontend telemetry in the Payment Telemetry Pipeline.

The objective was to create a dedicated, authenticated backend API that receives privacy-safe frontend telemetry events, validates them server-side, associates each event with the authenticated client, and stores the accepted events in PostgreSQL.

The implementation follows the current project direction:

```
Frontend
    ↓
Authenticated Telemetry API
    ↓
PostgreSQL
    ↓
Future ETL / Analytics
    ↓
Grafana
```

No S3/MinIO or data-lake layer was introduced because the current telemetry volume is expected to be small and the agreed direction is PostgreSQL-first.

---

# 2. Implementation Sequence

The implementation was performed in the following order.

## Step 1 — Confirm frontend event contract

Before implementing the backend, the frontend was checked to determine which telemetry events actually exist.

The frontend currently emits exactly five events:

```
account_type_selected
identity_verification_completed
liveness_verification_started
liveness_verification_completed
onboarding_form_submitted
```

This list became the backend event allow-list.

The backend was intentionally not designed to accept arbitrary event names.

---

## Step 2 — Investigate existing backend event systems

Existing backend systems were inspected to determine whether telemetry should reuse an existing event mechanism.

Two existing systems were specifically evaluated:

### Tenant event/outbox system

The existing `tenant_event`, `tenant_event_payload`, and `tenant_event_outbox` infrastructure is designed for domain/business events and webhook delivery.

It was therefore not reused for browser telemetry.

### Activity logs

The existing activity log system is employee-oriented and contains operational metadata such as browser, OS, device, IP, and user-agent information.

It was also not reused for product telemetry.

### Result

A dedicated telemetry ingestion path was chosen.

---

# 3. Database Implementation

## Step 3 — Create telemetry migration

Added:

```
db/migrations/000292_frontend_telemetry_events.up.sql
db/migrations/000292_frontend_telemetry_events.down.sql
```

The migration version was changed from:

```
291
```

to:

```
292
```

The migration was applied successfully and verified as:

```
292 | false
```

where `false` means the migration is not dirty.

---

## Step 4 — Create PostgreSQL telemetry table

Created:

```
public.frontend_telemetry_event
```

### Columns

| Column | Purpose |
| --- | --- |
| `id` | Internal database UUID |
| `event_id` | Frontend-generated unique event ID |
| `user_id` | Authenticated client ID |
| `event_name` | Telemetry event name |
| `event_version` | Telemetry contract version |
| `occurred_at` | Time the event occurred |
| `route` | Frontend route |
| `properties` | Telemetry properties stored as JSONB |
| `received_at` | Server-side ingestion timestamp |

### Database constraints

The table enforces:

- non-empty event name
- event name maximum length of 100 characters
- positive event version
- non-empty route
- route maximum length of 2048 characters
- properties must be a JSON object
- `event_id` must be unique
- `user_id` must reference `public.user(id)`

The user relationship uses:

```
ON DELETE RESTRICT
```

---

## Step 5 — Add indexes

Added indexes for expected telemetry queries:

```
(user_id, occurred_at DESC)
(event_name, occurred_at DESC)
(occurred_at DESC)
```

These support future queries involving:

- user journeys
- event history
- event frequency
- event trends
- time-based analytics

---

# 4. DAO and Repository

## Step 6 — Add DAO

Created:

```
db/dao/frontend_telemetry.go
```

Added:

```
FrontendTelemetryEvent
```

The DAO represents the PostgreSQL telemetry record and maps the `properties` field to the existing JSONB mapping convention.

---

## Step 7 — Add repository

Created:

```
db/repo/frontend_telemetry.go
```

Implemented:

```
CreateFrontendTelemetryEvent()
```

The repository uses:

```
ON CONFLICT DO NOTHING
```

for the unique `event_id`.

### Why this fits here

Idempotency belongs at the persistence boundary because the database is the final authority on whether an event already exists.

This prevents duplicate rows if the same event is submitted more than once.

---

# 5. Request DTO and Validation

## Step 8 — Create request DTO

Created:

```
http/rq/frontend_telemetry.go
```

Added:

```
FrontendTelemetryEventRequest
```

The request contains:

```
event_id
event_name
event_version
occurred_at
route
properties
```

---

## Step 9 — Add request validation

Validation was implemented before persistence.

The request validates:

### Event ID

Must be a non-zero UUID.

### Event name

Must:

- exist
- not be empty
- be no longer than 100 characters

### Event version

Must be greater than zero.

### Occurred timestamp

Must be present.

### Route

Must:

- exist
- not be empty
- be no longer than 2048 characters

### Properties

Must:

- be valid JSON
- be a JSON object
- be no larger than 16 KB

---

# 6. Property Contract

## Step 10 — Enforce scalar telemetry properties

The frontend telemetry contract allows only:

```
string
number
boolean
null
```

The backend now enforces the same contract.

### Valid example

```json
{
  "account_type": "personal",
  "step": 1,
  "completed": true,
  "optional": null
}
```

### Rejected example — nested object

```json
{
  "account": {
    "type": "personal"
  }
}
```

### Rejected example — array

```json
{
  "steps": [
    "start",
    "complete"
  ]
}
```

### Why this fits here

This validation belongs at the request boundary because the backend must not rely exclusively on frontend validation.

The backend is the final input boundary before data enters the telemetry dataset.

---

# 7. Event Allow-List

## Step 11 — Add supported event names

The backend was configured to accept only:

```
account_type_selected
identity_verification_completed
liveness_verification_started
liveness_verification_completed
onboarding_form_submitted
```

The supported event version is:

```
1
```

Unsupported event names and versions are rejected.

### Why this fits here

The allow-list belongs in backend business logic because it protects the stored telemetry dataset from arbitrary or accidental event types.

It also keeps the backend contract synchronized with the currently implemented frontend measurement plan.

---

# 8. Authentication and User Attribution

## Step 12 — Require authentication

The endpoint was configured with:

```
AuthRequired: true
```

The backend obtains the authenticated client from the existing request context:

```
rq.GetClientFromContext(ctx)
```

The database `user_id` is taken from:

```
client.ID
```

### Important behavior

The frontend does **not** provide the authoritative database `user_id`.

The resulting flow is:

```
Frontend event
      ↓
Authenticated request
      ↓
Backend auth context
      ↓
Authenticated client
      ↓
client.ID
      ↓
frontend_telemetry_event.user_id
```

### Why this fits here

Identity attribution belongs on the server because the authenticated backend context is authoritative.

It prevents the client from claiming that telemetry belongs to another user.

---

# 9. Backend Business Logic

## Step 13 — Implement ingestion logic

Created:

```
logic/frontend_telemetry.go
```

Implemented:

```
IngestFrontendTelemetryEvent()
```

The final processing sequence is:

```
1. Get authenticated client
2. Reject unauthenticated request
3. Read request DTO
4. Validate request
5. Validate event version
6. Validate event name
7. Parse properties
8. Obtain user ID from authenticated context
9. Normalize event name and route
10. Convert occurred_at to UTC
11. Set received_at on the server
12. Persist through repository
```

---

# 10. API Implementation

## Step 14 — Add dedicated API

Created:

```
api/frontend_telemetry.go
```

API name:

```
FrontendTelemetry
```

Endpoint:

```
POST /frontend-telemetry
```

The endpoint requires authentication and delegates processing to:

```
IngestFrontendTelemetryEvent()
```

---

## Step 15 — Register API at runtime

Updated:

```
svc.go
```

Added:

```
api.NewFrontendTelemetry(businessLogic)
```

This makes the telemetry endpoint part of the running backend API definitions.

---

# 11. Normalization

## Step 16 — Normalize event name and route

The ingestion logic trims surrounding whitespace from:

```
event_name
route
```

before storing them.

### Why this fits here

Normalization is performed at the storage boundary so that persisted telemetry has consistent values.

The request validation itself uses a value receiver, so normalization was explicitly performed in the ingestion logic before creating the DAO record.

---

# 12. Testing

## Step 17 — Add focused request tests

Created:

```
http/rq/frontend_telemetry_test.go
```

Tests cover:

- valid scalar properties
- nested property rejection
- array property rejection
- invalid JSON
- oversized properties
- missing required event ID

The focused test suite passed:

```
ok github.com/zolvat/svc/http/rq 0.215s
```

---

## Step 18 — Compile verification

Backend logic was formatted and compile-tested with:

```bash
go test ./logic -run '^$'
```

Result:

```
ok github.com/zolvat/svc/logic
```

---

## Step 19 — Database integration test limitation

A focused integration test was also created to verify that the authenticated client's ID is stored as the telemetry `user_id`.

However, the existing Windows embedded PostgreSQL test harness failed during database setup with:

```
character with byte sequence 0xe2 0x86 0x92
in encoding "UTF8" has no equivalent in encoding "WIN1252"
```

The failure occurs before the telemetry test itself executes.

The running PostgreSQL container was separately verified as UTF-8.

The shared embedded PostgreSQL test infrastructure was not modified because the encoding problem is an existing test-harness/environment issue rather than a telemetry implementation issue.

---

# 13. Edge Cases and Failure Handling

## Unauthenticated request

Rejected before processing.

```
authenticated client is required
```

## Missing event ID

Rejected during request validation.

## Unsupported event version

Rejected by the business logic.

## Unsupported event name

Rejected by the backend allow-list.

## Invalid JSON

Rejected during request validation.

## Nested property

Rejected.

## Array property

Rejected.

## Properties larger than 16 KB

Rejected.

## Duplicate event ID

Handled by:

```
ON CONFLICT DO NOTHING
```

This prevents duplicate database records.

## Database failure

The ingestion logic returns an internal error rather than exposing the database failure details to the client.

---

# 14. Security and Privacy Considerations

The implementation intentionally creates multiple privacy boundaries.

### Server-side identity

`user_id` is derived from authenticated backend context.

### Event allow-list

Only approved telemetry events can enter the dataset.

### Property restrictions

Only scalar values and `null` are allowed.

### Payload size

Properties are limited to 16 KB.

### No application payload reuse

Telemetry does not reuse arbitrary request or response bodies.

### No tenant event reuse

The business-event/outbox infrastructure is not used for frontend analytics.

### No activity-log reuse

Employee-oriented operational logging is not used as the telemetry store.

### No S3/MinIO

No unnecessary raw data lake or object-storage layer was introduced.

---

# 15. Performance Considerations

The implementation is intentionally simple for the current expected telemetry volume.

PostgreSQL is used as the ingestion store.

Indexes were added for common access patterns:

```
user + occurred_at
event_name + occurred_at
occurred_at
```

Event insertion uses a single database record per telemetry event.

Idempotency is handled by the unique `event_id` constraint and conflict-safe insertion.

Batching and queueing were intentionally not introduced at this stage.

---

# 16. Data Flow

## Before this task

The frontend could generate telemetry events locally, but there was no generic production transport connecting those events to a backend telemetry dataset.

```
Frontend
   ↓
trackEvent()
   ↓
Browser CustomEvent
```

There was no completed generic ingestion path.

---

## After this task

The backend ingestion path now exists:

```
Frontend
   ↓
Telemetry event
   ↓
POST /frontend-telemetry
   ↓
Authentication
   ↓
Request validation
   ↓
Event/version validation
   ↓
Property validation
   ↓
Server-side user attribution
   ↓
Repository
   ↓
PostgreSQL
```

The remaining missing connection is the frontend network transport.

---

# 17. Important Design Decisions

| Decision | Why |
| --- | --- |
| Dedicated telemetry endpoint | Keeps telemetry separate from normal business APIs |
| PostgreSQL storage | Matches current small-volume telemetry direction |
| Server-side user ID | Prevents client-controlled identity attribution |
| Event allow-list | Prevents arbitrary event types entering the dataset |
| Version validation | Creates an explicit telemetry contract |
| Scalar-only properties | Keeps telemetry predictable and privacy-safe |
| 16 KB property limit | Prevents oversized telemetry payloads |
| Unique event ID | Provides event-level idempotency |
| `ON CONFLICT DO NOTHING` | Makes duplicate ingestion safe |
| Dedicated DAO/repository | Keeps persistence concerns separate from API/business logic |
| No tenant-event reuse | Existing system serves business events/webhooks, not browser analytics |
| No activity-log reuse | Existing logs are employee-oriented and operational |
| No S3/MinIO | Not required for the current small-volume PostgreSQL architecture |
| Focused request tests | Allows validation testing without the problematic PostgreSQL harness |

---

# 18. Git Implementation State

### Backend repository

```
Zolvat/svc
```

### Branch

```
feature/backend-telemetry
```

### Commit

```
09bcaa585
```

### Commit message

```
feat: add authenticated frontend telemetry ingestion
```

### Commit contents

The commit added/modified 11 files covering:

- API
- DAO
- migration
- repository
- request DTO
- request tests
- business logic
- logic integration test
- runtime registration
- migration version

The branch was successfully pushed to:

```
origin/feature/backend-telemetry
```

No pull request was created as part of this task.

---

# 19. Final Implementation State

The backend frontend-telemetry ingestion layer is complete.

Implemented:

```
Frontend telemetry contract       ✓
Backend PostgreSQL table          ✓
Database migration 292            ✓
DAO                               ✓
Repository                        ✓
Authenticated API                 ✓
Server-side user attribution      ✓
Event allow-list                  ✓
Event version validation          ✓
Property validation               ✓
Property size limit               ✓
Event normalization               ✓
Idempotent event storage          ✓
Focused validation tests          ✓
Compile verification              ✓
Git commit                        ✓
Remote branch pushed              ✓
```

The backend is now ready for the next phase:

```
Frontend trackEvent()
        ↓
Dedicated authenticated transport
        ↓
POST /frontend-telemetry
        ↓
Backend ingestion
        ↓
PostgreSQL
```

After end-to-end ingestion is verified, the project can move into the actual Data Engineering phase:

```
PostgreSQL ingestion
        ↓
Python ETL / ELT
        ↓
Staging
        ↓
Curated analytics tables
        ↓
Incremental processing
        ↓
Data quality checks
        ↓
Funnel / product metrics
        ↓
Grafana
```

The implementation deliberately stops at the backend ingestion boundary so the next phase can focus on connecting the frontend transport and then building the ETL pipeline