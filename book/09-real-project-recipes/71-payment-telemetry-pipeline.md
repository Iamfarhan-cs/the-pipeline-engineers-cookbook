# Payment Telemetry Pipeline — Measurement Plan Phase

## 1. Task Overview

### Objective

The purpose of this task was to create the **Measurement Plan v1** for the Payment Telemetry Pipeline before starting the production telemetry implementation.

The measurement plan defines:

- What product/process questions telemetry should answer.
- Which frontend journey events should be measured.
- Which backend compliance events should be measured.
- Which operational signals should be collected.
- Which properties are allowed.
- Which data must never be collected.
- Where each type of telemetry should be stored.
- How frontend events and backend compliance states should work together.
- What retention and access-control decisions must be agreed before production rollout.

The main design principle from the implementation discussion was:

> **Collect only the telemetry required to answer defined product and operational questions.**

The project is intentionally starting with PostgreSQL for product/process analytics rather than introducing an S3/data-lake architecture.

## 2. Implementation Sequence

The work was performed in the following order.

### Step 1 — Capture the Measurement Direction

The initial requirement was reviewed and converted into two separate telemetry signal groups.

#### Operational Observability

Used to understand whether the frontend application is working correctly.

Signals include:

- Vue errors
- Unhandled exceptions
- Unhandled promise rejections
- Sanitized console.error
- Sanitized console.warn
- Failed API operations
- Web Vitals
- Release/build version
- Frontend-to-backend trace correlation

These signals are intended for:

~~~text
Frontend
   ↓
OpenTelemetry / Collector
   ↓
Loki / Tempo / VictoriaMetrics
   ↓
Grafana
~~~

#### Product / Process Analytics

Used to understand user journeys and business processes.

Signals include:

- Onboarding journey events
- Account-type selection
- Major onboarding steps
- Application submission
- Liveness verification
- Compliance state transitions

These signals are intended for:

~~~text
Frontend / Backend
      ↓
Telemetry ingestion
      ↓
PostgreSQL
      ↓
Grafana
~~~

## 3. Step 2 — Define Product Questions

Telemetry was not defined by simply collecting everything available.

Instead, the required events were derived from questions that the product and engineering teams need to answer.

### Onboarding Questions

The measurement plan needs to answer:

1. How many users start onboarding?
2. How many users reach each major onboarding step?
3. Where do users abandon the process?
4. Which steps have the highest drop-off?
5. How long does the user take between major steps?
6. Which onboarding steps generate the most frontend/API failures?
7. Are particular releases associated with increased onboarding failures?

### Compliance Questions

The measurement plan needs to answer:

1. How many users reach the compliance stage?
2. How many complete the compliance process?
3. Where does the process stop or fail?
4. How does the frontend journey relate to backend compliance state?
5. How long does the process take between important compliance states?
6. Are frontend/API failures associated with compliance friction?

### Operational Questions

The measurement plan needs to answer:

1. Are frontend errors increasing?
2. Which routes generate the most errors?
3. Which API operations fail most frequently?
4. Are failures concentrated in a particular release?
5. Are Web Vitals degrading after releases?
6. Can a frontend failure be correlated with the corresponding backend request?

## 4. Step 3 — Inspect the Existing Frontend Architecture

The existing frontend application was reviewed before defining implementation points.

Important findings:

- A centralized Axios client already exists.
- Existing verification-session telemetry already exists.
- Vue has a global app.config.errorHandler.
- There were no global window.onerror or unhandledrejection handlers.
- No Web Vitals implementation was found.
- No frontend release-version source was found.
- Existing trace-header extraction already exists.
- Existing onboarding/account flows were inspected to identify real user actions.

This avoided introducing duplicate telemetry mechanisms.

## 5. Step 4 — Map the Actual Frontend Journey

The actual frontend onboarding flow was inspected instead of assuming a generic onboarding sequence.

### Personal Account Flow

The confirmed flow is:

~~~text
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
~~~

The following implementation points were identified.

### Account Type Selection

ChooseAccountType.vue contains the real user action:

~~~text
applyForNewAccount(accountType)
~~~

This is the appropriate location for:

~~~text
account_type_selected
~~~

Allowed property:

~~~text
account_type = PERSONAL | CORPORATE
~~~

The request ID is not included in telemetry.

### Identity Verification

After successful identity information persistence:

~~~text
persistIdentityDraft({
    mode: 'submit',
    showError: true
})
~~~

the user proceeds to the personal account form.

A candidate event was identified:

~~~text
identity_verification_completed
~~~

The event should occur after successful persistence and before routing to the next stage.

### Personal Account Form

The active sections were identified as:

1. Currency Type
2. Account Creation Reason
3. Expected Turnover
4. Source of Funds
5. Tax Declaration
6. Additional Information
7. Terms & Conditions
8. Submit

No field values should be captured.

The successful form submission is performed through the existing backend API.

A candidate event was identified:

~~~text
onboarding_form_submitted
~~~

This event should represent successful backend acceptance rather than simply a button click.

### Liveness Verification

The liveness flow contains:

~~~text
livenessConsent.vue
~~~

and:

~~~text
livenessVerification.vue
~~~

The beginning of the liveness process can support:

~~~text
liveness_verification_started
~~~

Successful liveness completion can support:

~~~text
liveness_verification_completed
~~~

Existing verification-session telemetry should be reused instead of creating duplicate session-state telemetry.

### Final Onboarding Completion

KYCSuccess.vue is not itself treated as the final onboarding completion event.

The strongest completion point identified for personal account onboarding is reaching:

~~~text
/client/personal-account-success
~~~

Liveness-only flows must not incorrectly count as a new-account onboarding completion.

## 6. Step 5 — Inspect the Corporate Flow

The corporate onboarding flow was inspected separately.

The important implementation point is:

~~~text
submitApplicationForReview()
~~~

The successful backend operation is:

~~~text
PUT /user/accounts/create/corporate/:requestId
~~~

A successful response with no blocking tabs errors leads to:

~~~text
client-corporate-stakeholder-links
~~~

The candidate product event is:

~~~text
corporate_application_submitted
~~~

The event should be generated after successful backend acceptance and before navigation.

The following must not be added to telemetry:

- requestId
- stakeholder identifiers
- emails
- names
- submitted form values

## 7. Step 6 — Inspect Existing Verification Telemetry

The existing file:

~~~text
src/utils/verificationSessionTelemetry.ts
~~~

already contains verification-session telemetry.

It supports flows including:

~~~text
personal_authenticated
stakeholder_invite
~~~

Existing functionality includes:

~~~text
trackVerificationSessionState()
setVerificationSessionState()
~~~

The existing mechanism should be reused where applicable.

### Why This Fits Here

Verification-session state already has a dedicated telemetry mechanism.

Creating another independent telemetry implementation for the same session state would:

- duplicate events,
- create inconsistent semantics,
- increase maintenance,
- make downstream analysis harder.

Therefore the measurement plan explicitly prefers reuse over duplication.

## 8. Step 7 — Define Operational Signals

The operational telemetry plan defines the following candidate signals.

| Signal | Purpose | Target |
|---|---|---|
| frontend_error | General frontend errors | Loki |
| frontend_unhandled_exception | Browser/runtime exceptions | Loki |
| frontend_unhandled_rejection | Promise failures | Loki |
| frontend_console_error | Sanitized console errors | Loki |
| frontend_console_warn | Sanitized console warnings | Loki |
| frontend_api_error | Failed API operations | Loki / metrics |
| frontend_web_vital | Frontend performance | VictoriaMetrics |
| Trace information | Frontend/backend correlation | Tempo |
| Release metadata | Release correlation | Grafana/Loki/VictoriaMetrics |

These are measurement candidates defined by the plan; implementation should follow the agreed event contract.

## 9. Step 8 — Define Common Operational Properties

Only properties required for the specific signal should be included.

Potential permitted properties are:

~~~text
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
~~~

Not every event needs every property.

For example, an API failure can use:

~~~text
http_method
sanitized_endpoint
http_status
duration_ms
trace_id
~~~

while a frontend exception can use:

~~~text
error_type
redacted_error_message
redacted_stack_trace
route
release_version
~~~

## 10. Step 9 — Identify the Correct API Failure Integration Point

The existing Axios implementation was inspected.

The centralized file is:

~~~text
src/utils/axios.ts
~~~

It already contains a response interceptor and centralized error handling.

The proposed frontend_api_error telemetry belongs at this centralized layer.

### Why This Fits Here

Using the centralized Axios interceptor means API failures can be measured consistently across the application.

It avoids adding separate failure tracking to individual components.

The telemetry must use only sanitized metadata.

It must not capture:

- request bodies,
- response bodies,
- tokens,
- query parameters,
- raw Axios errors.

## 11. Step 10 — Inspect Trace Correlation

The existing file:

~~~text
src/utils/httpTrace.ts
~~~

already provides:

~~~text
extractTraceIdFromHeaders()
~~~

It checks:

~~~text
x-request-id
x-trace-id
traceparent
~~~

and extracts the relevant trace identifier.

Existing stakeholder flows already use this functionality.

### Design Decision

The telemetry implementation should reuse this existing trace extraction mechanism.

### Why This Fits Here

Trace correlation already has a shared utility.

Reusing it keeps frontend/backend correlation consistent and avoids implementing different trace parsing rules in telemetry code.

Also:

~~~text
request_id
~~~

and:

~~~text
trace_id
~~~

must not be treated as automatically interchangeable identifiers.

## 12. Step 11 — Inspect Web Vitals Support

The frontend was checked for existing Web Vitals support.

No existing implementation was found for:

- LCP
- INP
- CLS
- FCP
- TTFB
- PerformanceObserver
- web-vitals package

Therefore Web Vitals remain an implementation item from the measurement plan.

The intended telemetry category is:

~~~text
frontend_web_vital
~~~

with the resulting metrics routed to the operational observability stack.

## 13. Step 12 — Inspect Release Version Availability

The frontend was checked for an existing release/build identifier.

The following were searched:

~~~text
APP_VERSION
RELEASE_VERSION
release_version
releaseVersion
import.meta.env.*VERSION
CI_COMMIT_SHA
CI_COMMIT_SHORT_SHA
~~~

No existing release-version implementation was identified in the inspected frontend configuration.

Therefore the measurement plan does **not** invent a release identifier.

A deployment/build version or short commit SHA can be introduced during implementation once the actual deployment source is identified.

## 14. Step 13 — Inspect Global Vue Error Handling

The existing global Vue error handler in:

~~~text
src/main.ts
~~~

currently contains:

~~~text
app.config.errorHandler = (error) => {
  console.log(error)
}
~~~

This provides an existing application-level location for Vue errors.

The telemetry implementation must not forward the raw error object.

Instead, error information must be sanitized/redacted before telemetry is emitted.

## 15. Step 14 — Inspect Backend Compliance Architecture

The backend repository was inspected to identify the authoritative source of onboarding compliance state.

Relevant backend areas include:

~~~text
logic/complianceindividualaccount.go
logic/compliancekyc.go
logic/kyc_request_resolution.go
db/repo/reviewrequestkyc.go
~~~

The backend explicitly distinguishes onboarding reviews using:

~~~text
dao.ReviewKindOnboarding
~~~

This is important because onboarding compliance reviews must not be mixed with unrelated/manual review activity.

## 16. Step 15 — Identify Authoritative Backend States

The following backend statuses were observed:

~~~text
UNSUBMITTED
PENDING
AWAITING
ACCEPTED
APPROVED
REJECTED
CANCELLED
ON_HOLD
~~~

The implementation must not assume these statuses are interchangeable.

For example:

~~~text
ACCEPTED != PENDING
~~~

The backend state machine is the authoritative source for compliance-state analytics.

## 17. Step 16 — Inspect Personal Compliance Flow

In:

~~~text
logic/complianceindividualaccount.go
~~~

successful validation causes:

~~~text
SetAccountReviewToSubmitted(...)
~~~

The compliance review is then retrieved.

Depending on its state:

~~~text
Co1Status == AWAITING
~~~

can transition to:

~~~text
PENDING
~~~

or:

~~~text
Co2Status == AWAITING
~~~

can transition to:

~~~text
PENDING
~~~

The updated review is persisted.

### Measurement Decision

The frontend should not attempt to recreate this backend compliance state.

The backend should emit the authoritative compliance transition.

## 18. Step 17 — Inspect Corporate Compliance Flow

Corporate KYC handling was inspected in:

~~~text
logic/compliancekyc.go
~~~

The backend can transition KYC reviews into states including:

~~~text
ACCEPTED
REJECTED
AWAITING
~~~

and other terminal/open states.

The effective corporate stakeholder status also accounts for request-scoped KYC and previously accepted person-linked KYC.

This means frontend status alone cannot reliably represent the final compliance state.

## 19. Step 18 — Define Backend Compliance Event

The primary backend product event identified by the measurement plan is:

~~~text
compliance_state_changed
~~~

Its purpose is to represent an actual authoritative compliance/KYC state transition.

Proposed minimum properties:

~~~text
event_id
occurred_at
event_name
event_version
journey
account_type
previous_state
new_state
~~~

Example semantic structure:

~~~text
journey = onboarding
account_type = PERSONAL | CORPORATE
previous_state = AWAITING
new_state = PENDING
~~~

The exact event emission implementation remains an implementation-phase task.

## 20. Technical Changes Defined by the Measurement Plan

This task primarily produced the measurement design and implementation boundaries rather than introducing the complete telemetry runtime.

The technical areas identified for implementation are:

### Frontend

~~~text
src/utils/axios.ts
src/utils/httpTrace.ts
src/main.ts
src/views/client/ChooseAccountType.vue
src/views/client/personal-account/AccountForm.vue
src/views/client/personal-account/...
src/views/client/corporate-account-registration/CorporateAccountForm.vue
src/views/onboarding/StakeholderKyc.vue
~~~

Existing telemetry utilities should be reused where applicable.

### Backend

Relevant areas identified:

~~~text
logic/complianceindividualaccount.go
logic/compliancekyc.go
logic/kyc_request_resolution.go
db/repo/reviewrequestkyc.go
~~~

The backend compliance state transition is the source for authoritative compliance telemetry.

## 21. Design Decisions

### Decision 1 — Separate Operational and Product Telemetry

#### What

Telemetry is divided into:

~~~text
Operational Observability
~~~

and:

~~~text
Product / Process Analytics
~~~

#### Why This Fits Here

They answer different questions and require different storage/processing paths.

Operational signals belong in the observability stack:

~~~text
Loki
Tempo
VictoriaMetrics
Grafana
~~~

Product/process analytics belongs in:

~~~text
PostgreSQL
~~~

This prevents business analytics from being mixed with operational logs.

### Decision 2 — Use PostgreSQL for Initial Product Analytics

#### What

Product/process telemetry will initially be stored in PostgreSQL.

#### Why This Fits Here

The current expected volume is small, with approximately 100 users.

The existing direction explicitly avoids introducing an S3/data-lake architecture at this stage.

This keeps the implementation simple and aligned with the current scale.

### Decision 3 — Backend Compliance State Is Authoritative

#### What

Frontend events describe the user's journey.

Backend events describe authoritative compliance state.

#### Why This Fits Here

Compliance decisions and state transitions occur in the backend.

The frontend can show a state, but it should not become the source of truth for compliance analytics.

### Decision 4 — Track Meaningful User Actions

#### What

Telemetry should be attached to actual journey actions and successful state transitions.

Examples:

~~~text
account_type_selected
onboarding_form_submitted
corporate_application_submitted
liveness_verification_completed
~~~

#### Why This Fits Here

A route visit does not necessarily mean that a user completed a business step.

For example, displaying a success component does not automatically mean that onboarding is complete.

Events should represent meaningful process milestones.

### Decision 5 — Do Not Capture Form Values

#### What

Telemetry does not capture submitted form values.

#### Why This Fits Here

The application handles sensitive onboarding and compliance information.

Collecting complete form contents would unnecessarily increase privacy and GDPR exposure without being required to answer the defined product questions.

### Decision 6 — Reuse Existing Infrastructure

Existing components should be reused where possible:

~~~text
Axios interceptor
httpTrace utility
verificationSessionTelemetry
Vue global error handler
~~~

#### Why This Fits Here

These components already own the relevant concerns.

Telemetry should integrate with existing application boundaries instead of creating duplicate infrastructure.

## 22. Data Flow / Component Interaction

### Product Analytics

~~~text
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
~~~

Backend compliance:

~~~text
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
~~~

### Operational Telemetry

~~~text
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
~~~

## 23. Privacy and Security Rules

The following information is explicitly prohibited from telemetry.

### Never Capture

- Form values
- Documents
- Passport/ID information
- Tax information
- Names
- Email addresses
- Phone numbers
- Addresses
- Account numbers
- User IDs
- Person IDs
- Stakeholder IDs
- Request IDs
- Tokens
- Credentials
- Free-text fields
- Request bodies
- Response bodies
- URL query parameters
- Arbitrary console output
- Session recordings
- Mouse tracking
- Bot-detection data

### Error Data

Error messages and stack traces must be passed through redaction before storage or transmission.

Production source maps must remain private.

## 24. Performance Considerations

Telemetry must not block the primary user journey.

Important principles:

- Do not make telemetry a prerequisite for navigation.
- Do not block account submission on analytics.
- Do not add unnecessary API calls to individual form fields.
- Reuse centralized infrastructure.
- Capture only required properties.
- Avoid high-volume arbitrary console collection.

Existing verification telemetry already follows a telemetry-only approach that should not block UX.

## 25. Retention and Access Control

Retention must be explicitly agreed before production rollout.

The plan requires separate retention decisions for:

- Operational logs
- Traces
- Metrics
- Product analytics

Retention values were intentionally **not invented** during this task.

Access control must also be established for:

- Raw operational telemetry
- Telemetry PostgreSQL data
- Grafana dashboards
- Traces
- Detailed error information

## 26. Out of Scope

The following are outside the initial implementation scope:

- S3
- Data lake
- Kafka unless later justified
- Session replay
- Mouse tracking
- Bot detection
- Arbitrary console collection
- Form-value collection
- Document collection
- PII collection
- Request-body collection
- Response-body collection
- Token collection
- URL query-parameter collection

## 27. Validation and Verification

The measurement phase was validated by inspecting the actual application and backend implementation.

### Frontend Validation

Verified:

- Existing onboarding routes and components.
- Actual account-type selection handler.
- Actual personal onboarding flow.
- Actual corporate application submission flow.
- Existing liveness flow.
- Existing verification telemetry.
- Centralized Axios error handling.
- Existing trace-header extraction.
- Global Vue error handler.
- Absence of Web Vitals implementation.
- Absence of frontend release-version implementation.
- Absence of global unhandled-error/rejection handlers.

### Backend Validation

Verified:

- dao.ReviewKindOnboarding.
- Actual onboarding compliance state handling.
- Personal compliance status transitions.
- Corporate KYC state transitions.
- Corporate effective stakeholder KYC status logic.
- Repository persistence of review/KYC statuses.
- Existing accepted/rejected/awaiting/cancelled state handling.

No telemetry implementation details were invented where the existing code did not provide them.

## 28. Important Edge Cases

### Personal vs Corporate

The telemetry model must distinguish:

~~~text
PERSONAL
~~~

from:

~~~text
CORPORATE
~~~

because their onboarding flows and backend compliance logic differ.

### Compliance Review Scope

Only:

~~~text
ReviewKindOnboarding
~~~

should be treated as onboarding compliance analytics.

Manual or unrelated review activity must not be mixed into the onboarding funnel.

### Accepted vs Pending

The backend explicitly distinguishes:

~~~text
ACCEPTED
~~~

from:

~~~text
PENDING
~~~

Telemetry must preserve these states rather than normalizing them into a generic success state.

### Liveness-Only Flow

A liveness-only journey must not be incorrectly counted as a complete new-account onboarding journey.

### Missing Trace ID

Trace correlation is optional.

If a trace identifier is unavailable, telemetry should still be usable without inventing an identifier.

### Error Redaction

Raw errors must never bypass the redaction layer.

## 29. Final Implementation State

The **Measurement Plan Phase is complete**.

The completed work established:

- Defined product/process questions.
- Separated operational telemetry from product analytics.
- Mapped the actual frontend onboarding flows.
- Identified meaningful frontend journey events.
- Identified existing telemetry that should be reused.
- Identified the centralized Axios API-failure integration point.
- Identified the existing trace-correlation mechanism.
- Identified Web Vitals as a required operational signal.
- Identified release version as a required operational property, without inventing its source.
- Identified the global Vue error-handling location.
- Inspected backend onboarding compliance logic.
- Identified authoritative compliance states.
- Identified compliance_state_changed as the primary backend compliance telemetry event.
- Defined minimum telemetry properties.
- Defined strict privacy boundaries.
- Defined PostgreSQL as the initial product analytics store.
- Defined Grafana as the reporting layer.
- Defined Loki/Tempo/VictoriaMetrics for operational observability.
- Explicitly excluded S3/data lake and unnecessary high-volume tracking from the initial scope.
- Identified retention and access control as decisions required before production rollout.
