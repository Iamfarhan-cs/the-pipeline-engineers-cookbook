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
