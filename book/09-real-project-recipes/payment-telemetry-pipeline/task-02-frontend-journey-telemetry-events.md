# Recipe 02 — Frontend Journey Telemetry Events

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 2  
> **Source implementation commit:** `6e6579888` — `feat: add frontend journey telemetry events`  
> **Source repository:** `emi-frontend`

## 1. What This Recipe Teaches

This recipe extends the Stage 1 telemetry foundation with the remaining three onboarding journey milestones.

By completing it, you should understand how to:

- instrument events at verified business-success boundaries;
- distinguish a started operation from a completed operation;
- instrument a React-based liveness component alongside Vue components;
- avoid sending ephemeral biometric session identifiers;
- track a corporate onboarding submission without collecting the submitted form;
- keep event properties empty when standard event metadata is sufficient;
- test telemetry under both success and non-success conditions;
- extend an existing event whitelist without changing the telemetry contract.

Stage 1 established the telemetry boundary and the first two onboarding events. Stage 2 adds:

1. `liveness_verification_started`
2. `liveness_verification_completed`
3. `onboarding_form_submitted`

The source implementation describes this as the remaining three critical onboarding journey events. fileciteturn10file0L3-L15

---

## 2. The Problem

The onboarding flow contains several meaningful milestones:

~~~text
Account selection
      |
      v
Identity verification
      |
      v
Liveness verification
      |
      v
Corporate onboarding submission
~~~

Without telemetry at these boundaries, a pipeline cannot distinguish where users stop progressing.

For example, these states are materially different:

~~~text
Liveness session created
       !=
Liveness completed successfully
       !=
Corporate application submitted successfully
~~~

Stage 2 captures those milestones while deliberately excluding biometric payloads, cloud session credentials, and corporate registration details.

---

## 3. Real Project Context

Stage 2 extends the frontend telemetry module created in Stage 1.

The implementation was made in `emi-frontend` and instrumented:

- `FaceLivenessReact.jsx`
- `livenessVerification.vue`
- `CorporateAccountForm.vue`

The source commit is `6e6579888`, dated 2026-09-07. fileciteturn10file0L3-L9

The Stage 2 objective was to place each event at an exact, verified user-action success boundary and avoid ephemeral session IDs or biometric payloads. fileciteturn10file0L11-L15

---

# 4. Stage 1 Foundation

Stage 2 assumes that Stage 1 already provides:

~~~text
trackEvent(name, properties?)
        |
        v
createTelemetryEvent()
        |
        +--> event_id
        +--> event_version
        +--> occurred_at
        +--> route
        +--> redaction
        |
        v
window.dispatchEvent(CustomEvent)
~~~

Do not recreate the telemetry system in each component.

Stage 2 only adds business-event instrumentation.

---

# 5. Step 1 — Analyze the Liveness Journey

Before adding telemetry, identify the actual lifecycle.

The source implementation identified two important liveness boundaries:

1. session initialization in `FaceLivenessReact.jsx`;
2. completion callback handling in `livenessVerification.vue`.

The implementation sequence explicitly starts by locating these boundaries before inserting events. fileciteturn10file0L19-L24

The resulting lifecycle is:

~~~text
Create liveness session
        |
        | successful session creation
        v
liveness_verification_started
        |
        v
User completes liveness flow
        |
        | successful result
        v
liveness_verification_completed
~~~

This distinction is important for funnel analysis.

---

# 6. Step 2 — Instrument Liveness Start

File:

~~~text
src/components/FaceLivenessReact.jsx
~~~

Event:

~~~text
liveness_verification_started
~~~

The source implementation emits this event after the liveness session creation API returns a valid `session_id`.

Conceptually:

~~~jsx
trackEvent("liveness_verification_started");
~~~

The event intentionally has no additional properties.

The important boundary is:

~~~text
Session creation request
        |
        v
API response
        |
        +---- failure -> no telemetry milestone
        |
        +---- valid session_id -> liveness_verification_started
~~~

The source specifically records that the event is called after a successful session creation response. fileciteturn10file0L21-L23

---

# 7. Why the Session ID Is Not a Telemetry Property

The liveness provider returns a short-lived `session_id`.

It may be tempting to send:

~~~ts
trackEvent("liveness_verification_started", {
  session_id: sessionId,
});
~~~

Do **not** do this.

Stage 2 intentionally excludes the AWS Rekognition `session_id`.

The source design calls this out as a zero-ephemeral-session-ID rule because the value represents short-lived cloud infrastructure/session information that does not belong in behavioral telemetry. fileciteturn10file0L38-L42

The telemetry question is:

> Did liveness verification start?

It is not:

> What cloud session identifier was used?

This is a useful data-engineering principle:

**Collect the minimum attribute needed to answer the analytical question.**

---

# 8. Step 3 — Instrument Liveness Completion

File:

~~~text
src/views/client/verification/livenessVerification.vue
~~~

Event:

~~~text
liveness_verification_completed
~~~

The source places the event inside `handleCallback` only when:

~~~ts
isSuccessfulLivenessResult(data)
~~~

returns true.

Conceptually:

~~~ts
if (isSuccessfulLivenessResult(data)) {
  trackEvent("liveness_verification_completed");
}
~~~

The event has no additional properties.

The critical sequence is:

~~~text
Liveness callback
      |
      v
isSuccessfulLivenessResult(data)
      |
      +---- false -> no completion event
      |
      +---- true -> liveness_verification_completed
~~~

The source explicitly defines this success predicate as the emission boundary. fileciteturn10file0L21-L24

---

# 9. Why Success Must Be Verified

A common telemetry mistake is:

~~~text
callback received
      |
      v
completion event
~~~

A callback does not necessarily mean the business operation succeeded.

The correct model is:

~~~text
callback received
      |
      v
validate result
      |
      v
confirmed success
      |
      v
completion event
~~~

This prevents false-positive funnel metrics.

The Stage 2 design explicitly requires confirmed upstream success before dispatching milestone events. fileciteturn10file0L38-L42

---

# 10. Step 4 — Instrument Corporate Onboarding Submission

File:

~~~text
src/views/client/corporate-account-registration/CorporateAccountForm.vue
~~~

Event:

~~~text
onboarding_form_submitted
~~~

The source places the event in:

~~~text
submitApplicationForReview()
~~~

after a successful API response.

Conceptually:

~~~ts
// successful API response
trackEvent("onboarding_form_submitted");
~~~

The source notes that the event is emitted before navigating to stakeholder links. fileciteturn10file0L21-L24

The correct business sequence is:

~~~text
User submits corporate application
        |
        v
submitApplicationForReview()
        |
        v
API request
        |
        +---- failure -> no submission milestone
        |
        +---- success -> onboarding_form_submitted
        |
        v
Navigate to stakeholder links
~~~

---

# 11. Do Not Capture the Corporate Form

The corporate form may contain many fields.

Do not turn the telemetry event into:

~~~ts
trackEvent("onboarding_form_submitted", {
  form: formData,
});
~~~

The event only represents the milestone.

It does not need:

- corporate registration details;
- names;
- addresses;
- contact details;
- uploaded documents;
- arbitrary form state.

The source explicitly states that corporate registration details are omitted and only the submission milestone is tracked. fileciteturn10file0L112-L114

---

# 12. Why These Events Have No Properties

All three Stage 2 events use only the standard telemetry metadata.

They do not add event-specific properties:

~~~text
liveness_verification_started
        |
        +-- event_id
        +-- event_version
        +-- occurred_at
        +-- route
        +-- properties = empty

liveness_verification_completed
        |
        +-- event_id
        +-- event_version
        +-- occurred_at
        +-- route
        +-- properties = empty

onboarding_form_submitted
        |
        +-- event_id
        +-- event_version
        +-- occurred_at
        +-- route
        +-- properties = empty
~~~

This is an intentional design decision. fileciteturn10file0L38-L42

The analytical questions are simply:

- Did liveness start?
- Did liveness complete?
- Was onboarding submitted?

Additional sensitive context is not required to answer those questions.

---

# 13. Complete Stage 2 Event Set

After Stage 2, the frontend journey event set contains five events:

| Stage | Event |
|---|---|
| Stage 1 | `account_type_selected` |
| Stage 1 | `identity_verification_completed` |
| Stage 2 | `liveness_verification_started` |
| Stage 2 | `liveness_verification_completed` |
| Stage 2 | `onboarding_form_submitted` |

The source states that these five exact event names form the whitelist used by the Stage 4 backend ingestion API. fileciteturn10file0L78-L87

This creates an important contract:

~~~text
Frontend event names
        |
        v
Canonical allowed-event list
        |
        v
Backend ingestion validation
~~~

The frontend should not invent arbitrary event names outside the agreed contract.

---

# 14. Architecture and Data Flow

The complete Stage 2 flow is:

~~~text
                         User Actions
                              |
          +-------------------+--------------------+
          |                   |                    |
          v                   v                    v
FaceLivenessReact       Liveness Callback      Corporate Form
          |                   |                    |
Session creation OK      Result success       Submit API OK
          |                   |                    |
          v                   v                    v
liveness_             liveness_             onboarding_
verification_started  verification_completed form_submitted
          |                   |                    |
          +-------------------+--------------------+
                              |
                              v
                    createTelemetryEvent()
                              |
                              v
                  window.dispatchEvent()
                              |
                              v
                         CustomEvent
~~~

The source architecture records these three trigger paths explicitly. fileciteturn10file0L46-L61

---

# 15. Step 5 — Add Vitest Coverage

Stage 2 adds or updates tests for each journey event.

## Corporate onboarding

Create/update:

~~~text
CorporateAccountForm.spec.ts
~~~

Verify:

~~~text
successful submission
       |
       v
onboarding_form_submitted
~~~

The source implementation created a dedicated `CorporateAccountForm.spec.ts` test file. fileciteturn10file0L29-L34

## Liveness completion

Update:

~~~text
LivenessVerification.spec.ts
~~~

Verify that completion telemetry occurs after a successful liveness result.

## Liveness start

Update:

~~~text
FaceLivenessReact.spec.jsx
~~~

Verify that start telemetry is emitted when the valid liveness session is established.

The source lists these three test behaviors explicitly. fileciteturn10file0L97-L102

---

# 16. Test the Negative Cases

Telemetry should not fire merely because a handler executed.

At minimum, reason about these cases:

### Liveness session creation fails

~~~text
API failure
    |
    v
NO liveness_verification_started
~~~

### Liveness result is unsuccessful

~~~text
isSuccessfulLivenessResult(data) === false
    |
    v
NO liveness_verification_completed
~~~

### Corporate submission fails

~~~text
submitApplicationForReview() fails
    |
    v
NO onboarding_form_submitted
~~~

These conditions follow directly from the source requirement that events fire only under verified success conditions. fileciteturn10file0L38-L42

---

# 17. Privacy and Data Protection

Stage 2 adds biometric-related telemetry, so the privacy boundary becomes particularly important.

Do not send:

- facial images;
- biometric payloads;
- AWS Rekognition session IDs;
- cloud session credentials;
- corporate registration details;
- arbitrary form values.

Instead, emit only the milestone event.

The source explicitly confirms that biometric payloads, facial images, cloud session credentials, and corporate registration details are excluded. fileciteturn10file0L112-L114

---

# 18. Transaction Safety

This stage is client-side only.

Telemetry calls happen inside response handlers after the relevant business transaction or API mutation has completed. fileciteturn10file0L66-L68

Therefore:

~~~text
Business operation
       |
       v
API result
       |
       v
Verified success
       |
       v
Telemetry emission
~~~

Telemetry should not determine whether the business transaction succeeds.

If telemetry fails, the user-facing business operation should remain independent.

---

# 19. Idempotency

Each call to <code>trackEvent()</code> generates a new cryptographically random UUID.

This provides unique event identity across multiple submissions or retries. fileciteturn10file0L72-L74

Later backend persistence can use this identifier for deduplication.

Stage 2 itself does not implement database-level idempotency.

---

# 20. Database Schema

No database changes are made in Stage 2.

The source explicitly identifies this stage as client-side instrumentation only. fileciteturn10file0L91-L93

Database persistence remains a later pipeline stage.

---

# 21. Implementation Inventory

The Stage 2 logical changes are:

~~~text
src/
├── components/
│   └── FaceLivenessReact.jsx
│
└── views/
    └── client/
        ├── verification/
        │   └── livenessVerification.vue
        │
        └── corporate-account-registration/
            └── CorporateAccountForm.vue
~~~

Test coverage includes:

~~~text
CorporateAccountForm.spec.ts
FaceLivenessReact.spec.jsx
LivenessVerification.spec.ts
~~~

The source implementation identifies these files and their responsibilities. fileciteturn10file0L29-L34

> **Code-source note:** the supplied Stage 2 document provides the exact event calls, locations, success conditions, tests, and design constraints, but not the complete contents of every source file. This recipe therefore does not invent omitted full-file code.

---

# 22. Common Implementation Mistakes

## Mistake 1 — Emitting liveness start before session creation succeeds

Bad:

~~~text
start request
   |
   v
liveness_verification_started
   |
   v
session creation fails
~~~

Correct:

~~~text
session creation
   |
   v
valid session response
   |
   v
liveness_verification_started
~~~

---

## Mistake 2 — Capturing the liveness session ID

Bad:

~~~ts
trackEvent("liveness_verification_started", {
  session_id: sessionId,
});
~~~

The session ID is intentionally excluded.

---

## Mistake 3 — Treating every callback as successful completion

Bad:

~~~ts
handleCallback(data) {
  trackEvent("liveness_verification_completed");
}
~~~

Correct:

~~~ts
handleCallback(data) {
  if (isSuccessfulLivenessResult(data)) {
    trackEvent("liveness_verification_completed");
  }
}
~~~

---

## Mistake 4 — Sending the corporate form

Bad:

~~~ts
trackEvent("onboarding_form_submitted", {
  form: formData,
});
~~~

Correct:

~~~ts
trackEvent("onboarding_form_submitted");
~~~

---

## Mistake 5 — Adding unnecessary properties

Do not add session IDs, biometric metadata, or form values simply because they are available.

If the analytical question is binary, the event can remain property-free.

---

# 23. Independent Implementation Exercise

Implement the same pattern for a different multi-step application.

Create three events:

~~~text
document_upload_started
document_upload_completed
application_submitted
~~~

### Requirements

#### Document upload start

Emit only after the upload session has been successfully initialized.

#### Document upload completion

Emit only after the backend confirms successful processing.

#### Application submission

Emit only after the application API succeeds.

### Privacy

Do not send:

- document contents;
- document images;
- storage URLs;
- temporary upload session IDs;
- personal form data.

### Tests

Write tests proving:

1. failed initialization emits no start event;
2. successful initialization emits the start event;
3. unsuccessful processing emits no completion event;
4. successful processing emits the completion event;
5. failed application submission emits no submission event;
6. successful submission emits the submission event;
7. no sensitive properties are attached.

This exercise tests whether you understand **business-boundary instrumentation**, rather than merely the syntax of <code>trackEvent()</code>.

---

# 24. What You Should Understand Before Recipe 03

You should now understand the difference between:

~~~text
Technical callback
       !=
Business success
~~~

Telemetry should represent the latter.

You should also be able to explain:

- how to locate meaningful instrumentation points;
- why a session-created event differs from a completed event;
- why success predicates must be evaluated before completion telemetry;
- why ephemeral infrastructure/session identifiers should stay out of analytics;
- why milestone events can be property-free;
- how frontend event names become part of a backend ingestion whitelist;
- how to test both positive and negative telemetry paths.

The Stage 2 architecture is:

~~~text
Verified business result
          |
          v
      trackEvent()
          |
          v
createTelemetryEvent()
          |
          +--> event identity
          +--> timestamp
          +--> route
          +--> standard metadata
          |
          v
    CustomEvent boundary
          |
          v
     Future transport
~~~

---

## Source Traceability

This recipe is derived from the supplied Stage 2 implementation record:

- Source commit: <code>6e6579888</code>
- Subject: <code>feat: add frontend journey telemetry events</code>
- Repository: <code>emi-frontend</code>
- Implementation scope: fileciteturn10file0L3-L15
- Implementation sequence: fileciteturn10file0L19-L24
- Technical changes: fileciteturn10file0L29-L34
- Design decisions: fileciteturn10file0L38-L42
- Architecture: fileciteturn10file0L46-L61
- Tests and validation: fileciteturn10file0L97-L108
- Privacy and scope boundaries: fileciteturn10file0L112-L121
