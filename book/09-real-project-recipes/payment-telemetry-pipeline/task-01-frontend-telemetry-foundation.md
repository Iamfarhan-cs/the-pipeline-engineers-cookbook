# Recipe 01 — Frontend Telemetry Foundation and Journey Events

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 1  
> **Source implementation commit:** 697487e2c — feat: add frontend telemetry foundation and journey events  
> **Source repository:** emi-frontend

## 1. What This Recipe Teaches

This recipe builds the frontend telemetry boundary before network transport or backend ingestion exists.

By completing it, you should understand how to:

- define a versioned telemetry event contract;
- generate event identifiers and timestamps;
- capture the browser pathname without leaking query strings or hashes;
- constrain event properties to safe scalar values;
- sanitize every string property at one central boundary;
- remove credentials, tokens, PII, and sensitive URL parameters;
- instrument meaningful onboarding journey events;
- emit telemetry locally through browser CustomEvent;
- test both the telemetry module and UI integration;
- keep telemetry isolated from network transport.

**Core principle:** telemetry must become privacy-safe before it leaves the application boundary.

---

## 2. The Problem

A frontend application contains useful behavioral signals such as account type selection and identity-verification completion. But telemetry is generated close to user input and authentication state, so careless instrumentation can leak access tokens, passwords, API keys, emails, phone numbers, sensitive query parameters, or arbitrary form data.

Stage 1 therefore does **not** start with a database, API, queue, Kafka topic, or data lake.

It starts with a controlled client-side event boundary.

---

## 3. Real Project Context

The Stage 1 implementation was made in the <code>emi-frontend</code> application and created the frontend telemetry foundation under <code>src/telemetry/</code>.

The initial onboarding events were:

1. <code>account_type_selected</code>
2. <code>identity_verification_completed</code>

Additional journey events were intentionally deferred to later stages. The supplied implementation record identifies source commit <code>697487e2c</code>, based on <code>73f362eda</code>. fileciteturn5file0L3-L15

---

## 4. Target Architecture

~~~text
User interaction
      |
      v
Vue component
      |
      v
trackEvent(name, properties)
      |
      v
createTelemetryEvent()
      |
      +--> event_id = crypto.randomUUID()
      +--> occurred_at = new Date().toISOString()
      +--> route = window.location.pathname
      +--> event_version = 1
      +--> sanitizeProperties()
      |
      v
window.dispatchEvent(
  new CustomEvent("telemetry_event", { detail: event })
)
~~~

There is no network transport in this stage. Transport is isolated for a later stage. fileciteturn5file0L44-L71

---

## 5. Step 1 — Define the Telemetry Contract

A telemetry event needs a stable structure:

| Field | Purpose |
|---|---|
| <code>event_id</code> | Globally unique event identifier |
| <code>event_name</code> | Semantic event name |
| <code>event_version</code> | Version of the event contract |
| <code>occurred_at</code> | Event creation timestamp |
| <code>route</code> | Browser pathname |
| <code>properties</code> | Small set of event attributes |

The implementation restricts properties to:

~~~ts
Record<string, string | number | boolean | null>
~~~

This prevents arbitrary nested objects and arrays from becoming part of the telemetry contract. fileciteturn5file0L32-L39

### Why this matters

Without a contract, telemetry quickly becomes inconsistent. A strict contract gives downstream consumers predictable fields and types.

---

## 6. Step 2 — Create the Privacy Redaction Boundary

Create:

~~~text
src/telemetry/redact.ts
~~~

The implementation defines:

- <code>redactTelemetryText()</code>
- <code>redactError()</code>

The sanitizer is regex-driven and covers:

### Authentication

- Bearer tokens;
- <code>access_token</code>;
- <code>refresh_token</code>;
- <code>id_token</code>;
- <code>authorization</code>.

### Credentials and secrets

- <code>password</code>;
- <code>secret</code>;
- <code>api_key</code>;
- <code>client_secret</code>.

### Personal information

- <code>email</code>;
- <code>phone</code>.

### URL leakage

Sensitive query-string parameters are also sanitized.

Detected values are replaced with:

~~~text
[REDACTED]
~~~

These patterns and the redaction boundary are explicitly documented in the source implementation. fileciteturn5file0L32-L39

### Critical rule

Do not rely on each Vue component to remember what must be redacted.

Use:

~~~text
Component
   |
   v
trackEvent()
   |
   v
createTelemetryEvent()
   |
   v
Central sanitizer
~~~

Boundary redaction prevents individual components from accidentally bypassing sanitization. fileciteturn5file0L44-L49

---

## 7. Step 3 — Protect the Route Field

Capture only:

~~~ts
window.location.pathname
~~~

Do not capture the complete URL because query strings and hashes can contain credentials or other sensitive values.

For example, the intended representation is:

~~~text
/onboarding
~~~

rather than a complete URL containing <code>?token=...</code>.

The source design explicitly discards query strings and hash fragments. fileciteturn5file0L44-L49

---

## 8. Step 4 — Create the Event Builder

Create:

~~~text
src/telemetry/index.ts
~~~

The module is responsible for:

- <code>TelemetryEvent</code>;
- <code>TelemetryProperties</code>;
- event creation;
- property sanitization;
- UUID generation;
- timestamp generation;
- pathname capture;
- browser event dispatch.

The creation sequence is:

~~~text
createTelemetryEvent()
      |
      +-- crypto.randomUUID()
      +-- new Date().toISOString()
      +-- window.location.pathname
      +-- event_version = 1
      +-- sanitizeProperties()
      |
      v
TelemetryEvent
~~~

The source implementation uses <code>crypto.randomUUID()</code> for event identity. fileciteturn5file0L32-L39

---

## 9. Step 5 — Dispatch Locally

Stage 1 does not send telemetry over HTTP.

Instead, <code>trackEvent()</code> dispatches a browser CustomEvent:

~~~ts
window.dispatchEvent(
  new CustomEvent("telemetry_event", {
    detail: event,
  }),
);
~~~

This creates a clean boundary:

~~~text
Telemetry generation
        |
        v
Browser event boundary
        |
        +---- current stage: local listener
        |
        +---- future stage: HTTP transport
~~~

This lets a future transport layer be added without rewriting every component that emits telemetry. fileciteturn5file0L44-L49

---

## 10. Step 6 — Instrument Account Type Selection

The first business event is:

~~~text
account_type_selected
~~~

It is emitted from:

~~~text
src/views/client/ChooseAccountType.vue
~~~

The source places the event inside <code>applyForNewAccount()</code> and records the selected account type.

Conceptually:

~~~ts
trackEvent("account_type_selected", {
  account_type: accountType,
});
~~~

The expected account types are:

~~~text
PERSONAL
CORPORATE
~~~

The event is tied to the business action rather than a generic UI render. fileciteturn5file0L22-L27

---

## 11. Step 7 — Instrument Identity Verification Completion

The second business event is:

~~~text
identity_verification_completed
~~~

It is emitted from:

~~~text
src/views/client/verification/IndividualVerification.vue
~~~

The source places it inside <code>handleNextClick()</code> **after successful draft persistence**.

Conceptually:

~~~ts
trackEvent("identity_verification_completed", {
  account_type: accountType.value,
});
~~~

The intended order is:

~~~text
User submits verification
        |
        v
persistIdentityDraft({ mode: "submit" })
        |
        v
Persistence succeeds
        |
        v
identity_verification_completed
~~~

A completion event should represent successful completion, not merely a button click. fileciteturn5file0L22-L27

---

## 12. Step 8 — Write Unit Tests

Create:

~~~text
src/telemetry/index.spec.ts
~~~

Verify:

- UUID generation;
- timestamp creation;
- pathname extraction;
- event version;
- CustomEvent emission;
- redaction at the telemetry boundary.

The source test suite names these behaviors explicitly. fileciteturn5file0L101-L109

Create:

~~~text
src/telemetry/redact.spec.ts
~~~

Verify:

- Bearer token redaction;
- sensitive key/value redaction;
- sensitive query-parameter redaction. fileciteturn5file0L103-L107

---

## 13. Step 9 — Test UI Integration

Update:

~~~text
src/views/client/ChooseAccountType.spec.ts
~~~

Verify that Personal selection emits <code>account_type_selected</code> with <code>PERSONAL</code>.

Update:

~~~text
src/views/client/verification/__tests__/IndividualVerification.spec.ts
~~~

Verify that successful identity submission emits <code>identity_verification_completed</code>.

The Stage 1 source implementation includes both component-level assertions. fileciteturn5file0L38-L40

---

## 14. Implementation Inventory

The Stage 1 change touches these logical files:

~~~text
src/
├── telemetry/
│   ├── index.ts
│   ├── index.spec.ts
│   ├── redact.ts
│   └── redact.spec.ts
│
└── views/
    └── client/
        ├── ChooseAccountType.vue
        ├── ChooseAccountType.spec.ts
        │
        └── verification/
            ├── IndividualVerification.vue
            └── __tests__/
                └── IndividualVerification.spec.ts
~~~

The supplied implementation record identifies these files and their responsibilities. fileciteturn5file0L22-L27 fileciteturn5file0L34-L40

> **Code-source note:** the supplied Stage 1 document describes the exact implementation and key code expressions but does not contain the complete source contents of every referenced file. This recipe therefore preserves verified code patterns without fabricating omitted full-file code.

---

## 15. Example Event

A runtime event has this shape:

~~~json
{
  "event_id": "generated-uuid",
  "event_name": "account_type_selected",
  "event_version": 1,
  "occurred_at": "2026-09-05T12:00:00.000Z",
  "route": "/client/choose-account-type",
  "properties": {
    "account_type": "PERSONAL"
  }
}
~~~

The UUID, timestamp, and pathname are generated at runtime.

---

## 16. Privacy Rules

Telemetry must not intentionally contain:

- passwords;
- access tokens;
- refresh tokens;
- authorization values;
- API keys;
- client secrets;
- email addresses;
- phone numbers;
- sensitive query-string values;
- arbitrary sensitive user input.

Detected sensitive values are replaced with <code>[REDACTED]</code>. fileciteturn5file0L119-L123

### Defense in depth

Redaction is not permission to collect sensitive information.

Prefer:

~~~text
Do not collect sensitive data
        +
Redact unexpected sensitive values
        =
Defense in depth
~~~

---

## 17. Why Flat Scalar Properties?

The implementation uses:

~~~ts
string | number | boolean | null
~~~

rather than arbitrary JSON.

This reduces the payload surface and prevents accidental inclusion of uncontrolled objects.

Avoid:

~~~ts
trackEvent("some_event", {
  form: entireFormObject,
});
~~~

Prefer explicit fields:

~~~ts
trackEvent("some_event", {
  account_type: "PERSONAL",
  step: 2,
  completed: true,
});
~~~

The flat-scalar restriction is an explicit Stage 1 design decision. fileciteturn5file0L46-L49

---

## 18. Why No Database Yet?

There is no database schema in Stage 1.

The staged design is:

~~~text
Stage 1
Frontend event boundary
        |
        X
No persistence

Stage 4
Backend ingestion
        |
        v
PostgreSQL

Stage 5
HTTP transport
        |
        v
Backend ingestion
~~~

The source explicitly defers PostgreSQL persistence to Stage 4 and HTTP transport to Stage 5. fileciteturn5file0L127-L131

This keeps the first implementation small and allows the telemetry contract to stabilize before persistence and transport.

---

## 19. Idempotency Consideration

The browser layer does not perform database-level deduplication.

However, each event receives a UUID using <code>crypto.randomUUID()</code>. Later transport and persistence layers can use this identifier for deduplication.

The source implementation explicitly identifies UUIDs as the downstream identity used when events are captured, replayed, or resent. fileciteturn5file0L76-L85

**Pipeline principle:** generate event identity as early as possible.

---

## 20. Transaction Safety

Database transaction handling is not applicable at this client layer.

Telemetry emission executes synchronously in the browser event loop and does not require a database transaction. fileciteturn5file0L76-L80

Transaction and persistence guarantees belong to later pipeline stages.

---

## 21. Validation Checklist

### Contract

- [ ] Every event has <code>event_id</code>.
- [ ] Every event has <code>event_name</code>.
- [ ] Every event has <code>event_version</code>.
- [ ] Every event has <code>occurred_at</code>.
- [ ] Every event has pathname-only <code>route</code>.
- [ ] Properties are scalar values.

### Privacy

- [ ] Bearer tokens are redacted.
- [ ] Sensitive key/value pairs are redacted.
- [ ] Sensitive query parameters are redacted.
- [ ] Passwords are not intentionally captured.
- [ ] Authorization values are not intentionally captured.
- [ ] Email and phone values are not intentionally captured.

### Architecture

- [ ] Telemetry is isolated under <code>src/telemetry/</code>.
- [ ] Components call the telemetry API instead of implementing redaction.
- [ ] Stage 1 does not perform HTTP transport.
- [ ] Stage 1 does not require database persistence.

### Journey events

- [ ] <code>account_type_selected</code> is emitted from account selection.
- [ ] <code>identity_verification_completed</code> is emitted only after successful identity persistence.

### Tests

- [ ] Telemetry contract tests pass.
- [ ] Redaction tests pass.
- [ ] Account selection telemetry test passes.
- [ ] Identity verification telemetry test passes.

---

## 22. Expected Test Coverage

The Stage 1 source implementation lists six key test behaviors:

1. Versioned event with ID, timestamp, and pathname.
2. Sensitive string-property redaction.
3. Browser telemetry boundary emission.
4. Bearer token, sensitive key/value, and query-parameter redaction.
5. <code>account_type_selected</code> emission for Personal selection.
6. <code>identity_verification_completed</code> emission after successful identity submission.

fileciteturn5file0L101-L109

---

## 23. Common Implementation Mistakes

### Mistake 1 — Sending the complete URL

Bad:

~~~ts
route: window.location.href
~~~

This can leak secrets embedded in query parameters.

Use pathname-only routing.

### Mistake 2 — Sanitizing inside every component

Bad:

~~~text
ChooseAccountType.vue -> manual sanitization
Verification.vue      -> manual sanitization
AnotherComponent.vue  -> forgotten sanitization
~~~

Better:

~~~text
Every component
      |
      v
trackEvent()
      |
      v
Central sanitizer
~~~

### Mistake 3 — Sending arbitrary objects

Avoid:

~~~ts
properties: entireForm
~~~

Keep the contract scalar and explicit.

### Mistake 4 — Emitting completion before success

Do not emit:

~~~text
submit clicked
   |
   v
completion event
   |
   v
persistence fails
~~~

Use:

~~~text
submit
   |
   v
persist
   |
   v
success
   |
   v
completion event
~~~

### Mistake 5 — Adding HTTP transport too early

Do not combine event creation, redaction, authentication, HTTP, retry, and backend ingestion into one Stage 1 module.

Keep the boundaries independent.

---

## 24. Scope Boundaries

### Implemented in Stage 1

- frontend telemetry event contract;
- UUID generation;
- timestamp generation;
- pathname capture;
- scalar properties;
- centralized regex redaction;
- local browser CustomEvent;
- <code>account_type_selected</code>;
- <code>identity_verification_completed</code>;
- unit and component tests.

### Deferred to Stage 2

- <code>liveness_verification_started</code>;
- <code>liveness_verification_completed</code>;
- <code>onboarding_form_submitted</code>.

### Deferred to Stage 4

- backend ingestion API;
- PostgreSQL persistence.

### Deferred to Stage 5

- HTTP network transport;
- <code>POST /frontend-telemetry</code>.

These scope boundaries are explicitly recorded in the source implementation. fileciteturn5file0L127-L131

---

## 25. Independent Implementation Exercise

Implement the same pattern in another frontend application.

### Event

~~~text
checkout_started
~~~

### Properties

~~~text
cart_size
checkout_type
~~~

### Contract

~~~text
event_id
event_name
event_version
occurred_at
route
properties
~~~

### Security

Redact:

- Bearer tokens;
- passwords;
- authorization values;
- API keys;
- emails;
- phone numbers;
- sensitive query parameters.

### Boundary

Components should only call:

~~~ts
trackEvent(...)
~~~

They should not contain their own redaction logic.

### Verification

Write tests proving that:

1. an event receives a UUID;
2. the event receives a timestamp;
3. the route excludes query parameters;
4. sensitive strings are redacted;
5. the event is emitted through a browser event;
6. the component emits the event at the correct business boundary.

If you can implement this pattern independently, you understand the architecture rather than merely memorizing the project.

---

## 26. What You Should Understand Before Recipe 02

You should now be able to explain:

- why telemetry needs a formal contract;
- why event identity is generated at the frontend boundary;
- why pathname is safer than the complete URL;
- why scalar properties reduce telemetry risk;
- why redaction belongs in the telemetry module;
- why completion events must follow successful business operations;
- why network transport is separated from event generation;
- how browser CustomEvent creates an intermediate telemetry boundary;
- how tests protect both the generic telemetry layer and journey integrations.

The resulting architecture is:

~~~text
Business action
      |
      v
trackEvent()
      |
      v
createTelemetryEvent()
      |
      +--> identity
      +--> timestamp
      +--> route
      +--> version
      +--> redaction
      |
      v
CustomEvent
      |
      v
[Future transport layer]
~~~

---

## Source Traceability

This recipe is derived from the supplied Stage 1 implementation record.

- Source commit: <code>697487e2c</code>
- Subject: <code>feat: add frontend telemetry foundation and journey events</code>
- Repository: <code>emi-frontend</code>
- Implementation scope: fileciteturn5file0L3-L15
- Implementation sequence: fileciteturn5file0L20-L27
- Technical changes: fileciteturn5file0L32-L40
- Architecture and data flow: fileciteturn5file0L44-L71
- Testing: fileciteturn5file0L101-L109
- Privacy and scope boundaries: fileciteturn5file0L119-L131
