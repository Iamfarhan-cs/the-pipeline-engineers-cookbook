# Recipe 05 — Authenticated Frontend Telemetry Transport

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 5  
> **Source implementation commit:** `9684b6aba` — `feat: add authenticated frontend telemetry transport`  
> **Source repository:** `emi-frontend`

## 1. What This Recipe Teaches

This recipe connects the frontend telemetry boundary from Stages 1–3 to the authenticated backend ingestion API from Stage 4.

By completing it, you should understand how to:
- build a dedicated telemetry HTTP transport;
- authenticate telemetry using the existing access token;
- avoid coupling background telemetry to the application's central API client;
- use `keepalive: true` for navigation/unload durability;
- drop telemetry when there is no authenticated user;
- prevent telemetry failures from affecting the application journey;
- preserve the existing local `CustomEvent` boundary;
- use the same `event_id` across retries so backend idempotency can work;
- test authenticated dispatch, unauthenticated dropping, and failure isolation.

The source commit introduces `sendTelemetryEvent()` and connects `trackEvent()` to `POST /frontend-telemetry`. fileciteturn13file0L3-L16

## 2. The Problem

Stages 1–3 generated privacy-safe telemetry but kept it inside the browser:

~~~text
trackEvent()
    |
    v
window.dispatchEvent(CustomEvent)
~~~

Stage 4 created the backend ingestion API:

~~~text
POST /frontend-telemetry
    |
    v
authenticated ingestion
    |
    v
PostgreSQL
~~~

Stage 5 connects those two boundaries.

The critical requirement is that telemetry is **non-essential background work**. If telemetry fails, the onboarding journey must continue normally. fileciteturn13file0L9-L16

## 3. Target Architecture

~~~text
Vue / React Component
        |
        v
trackEvent(name, properties)
        |
        +------------------------------+
        |                              |
        v                              v
sendTelemetryEvent(event)       CustomEvent
        |
        v
access token check
        |
        +---- absent -> drop
        |
        v
POST /frontend-telemetry
        |
        +--> Bearer token
        +--> JSON event
        +--> keepalive: true
        |
        v
Stage 4 backend
        |
        v
PostgreSQL
~~~

The source records this dual path from `trackEvent()` to both network transport and local browser dispatch. fileciteturn13file0L53-L73

## 4. Step 1 — Design the Transport Boundary

Create:

~~~text
src/telemetry/transport.ts
~~~

The transport API is:

~~~ts
sendTelemetryEvent(event: TelemetryEvent): Promise<void>
~~~

The module should own only network delivery concerns:
- endpoint resolution;
- authentication header construction;
- request serialization;
- browser keepalive behavior;
- failure containment.

Event creation, privacy redaction, and business-event instrumentation remain in the existing telemetry modules.

The source implementation explicitly introduces this dedicated transport module. fileciteturn13file0L22-L26

## 5. Step 2 — Resolve the Backend Endpoint

The endpoint is constructed as:

~~~text
${getApiUrl()}/frontend-telemetry
~~~

Use the application's existing API URL helper rather than hard-coding an environment-specific host.

The source explicitly states that `sendTelemetryEvent()` resolves the endpoint using `getApiUrl()`. fileciteturn13file0L33-L35

This allows the same telemetry transport code to operate against the configured development, staging, or production API.

## 6. Step 3 — Retrieve the Access Token

The transport reads the active user's `accessToken` from the existing authentication store.

The flow is:

~~~text
sendTelemetryEvent(event)
        |
        v
auth store
        |
        v
accessToken()
        |
        +---- token exists -> continue
        |
        +---- no token -> return
~~~

The source explicitly requires unauthenticated events to be dropped rather than sent to the backend. fileciteturn13file0L33-L38

## 7. Why Drop Unauthenticated Events?

Stage 4 requires authentication.

Therefore sending an event without a token would only produce an unnecessary `401` request.

Instead:

~~~text
No access token
      |
      v
Do not call fetch()
~~~

This has several benefits:
- no unnecessary network request;
- no expected 401 noise;
- no background authentication behavior;
- simpler telemetry failure semantics.

The source identifies dropping unauthenticated events as an explicit design decision. fileciteturn13file0L46-L49

## 8. Step 4 — Use Native Fetch

The transport uses browser-native `fetch` rather than the application's central Axios/API client.

Conceptually:

~~~ts
await fetch(`${getApiUrl()}/frontend-telemetry`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${token}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify(event),
  keepalive: true,
});
~~~

The source specifies this exact request shape. fileciteturn13file0L33-L38

## 9. Why Not the Central API Client?

The source deliberately avoids Axios/application `apiClient`.

The reason is isolation.

A central API client may contain application-wide behavior such as:
- automatic 401 handling;
- token refresh;
- logout behavior;
- user-facing error notifications;
- shared request interceptors.

Those behaviors are appropriate for business requests but undesirable for background telemetry.

Imagine telemetry fails while the user is onboarding:

~~~text
Telemetry request -> 401
        |
        v
Global API interceptor
        |
        +--> refresh token
        +--> show error
        +--> logout
        v
User journey disrupted
~~~

The dedicated native `fetch` boundary avoids this coupling. The source explicitly gives circular dependencies and global interceptor behavior as reasons for using native fetch. fileciteturn13file0L44-L48

## 10. Step 5 — Enable `keepalive`

Set:

~~~ts
keepalive: true
~~~

This is important for telemetry because users may navigate immediately after an event.

Without keepalive:

~~~text
trackEvent()
    |
    v
fetch()
    |
    v
User navigates
    |
    v
Browser may cancel request
~~~

With keepalive:

~~~text
trackEvent()
    |
    v
fetch({ keepalive: true })
    |
    v
Navigation
    |
    v
Browser can continue the request
~~~

The source explicitly identifies navigation durability as the reason for `keepalive: true`. fileciteturn13file0L46-L48

## 11. Step 6 — Contain Every Transport Failure

Telemetry must not create uncaught promise rejections.

The source implementation uses both:

~~~ts
.catch(() => {})
~~~

and an outer:

~~~ts
try/catch
~~~

around the fetch operation.

The goal is:

~~~text
Telemetry succeeds
    -> data delivered

Telemetry gets HTTP error
    -> ignored

Network fails
    -> ignored

fetch throws
    -> ignored

Application continues
    -> always
~~~

The source explicitly describes the dual failure boundary and the requirement for zero uncaught promise rejections. fileciteturn13file0L22-L25

## 12. Why Suppress HTTP Errors?

Telemetry is observability data, not the business transaction itself.

For example:

~~~text
User submits onboarding form
        |
        +--> business API succeeds
        |
        +--> telemetry POST fails
        |
        v
User should still continue
~~~

Do not show an error modal because telemetry failed.

The source explicitly requires network failures and HTTP error responses to be discarded without affecting the UI. fileciteturn13file0L46-L48

## 13. Step 7 — Connect Transport to `trackEvent()`

Update:

~~~text
src/telemetry/index.ts
~~~

After the event is created, invoke:

~~~ts
sendTelemetryEvent(event)
~~~

The existing local browser event should continue to be dispatched.

The resulting structure is:

~~~text
trackEvent()
     |
     v
create event
     |
     +----------------------+
     |                      |
     v                      v
sendTelemetryEvent()   CustomEvent dispatch
     |
     v
HTTP backend
~~~

The source specifies that `trackEvent()` invokes the transport asynchronously before dispatching the existing `CustomEvent`. fileciteturn13file0L22-L26

## 14. Do Not Replace the CustomEvent Boundary

A common mistake is to turn the telemetry system into only an HTTP call.

Do not remove:

~~~ts
window.dispatchEvent(new CustomEvent("telemetry_event", ...))
~~~

The local event boundary remains useful for:
- browser-level consumers;
- tests;
- future local telemetry processing;
- separation between event generation and transport.

Stage 5 adds transport; it does not eliminate the existing telemetry abstraction.

## 15. Complete Data Flow

~~~text
Business event
      |
      v
trackEvent(name, properties)
      |
      v
createTelemetryEvent()
      |
      +--> event_id
      +--> event_version
      +--> occurred_at
      +--> route
      +--> sanitized properties
      |
      +-------------------------------+
      |                               |
      v                               v
sendTelemetryEvent(event)       CustomEvent
      |
      v
access token?
      |
      +---- no --> drop
      |
      +---- yes
      |
      v
fetch POST /frontend-telemetry
      |
      +--> Bearer token
      +--> JSON body
      +--> keepalive
      |
      v
svc
      |
      v
PostgreSQL
~~~

## 16. Idempotency

Stage 5 does not create a new idempotency mechanism.

Instead, it reuses the `event_id` generated by the Stage 1 telemetry event builder.

~~~text
createTelemetryEvent()
        |
        v
event_id = UUID X
        |
        v
sendTelemetryEvent()
        |
        v
backend receives UUID X
        |
        v
Stage 4 UNIQUE(event_id)
~~~

If a network retry sends the same event again, Stage 4 can safely discard the duplicate.

The source explicitly states that retries carry the same client-generated UUID for backend deduplication. fileciteturn13file0L85-L87

## 17. Transaction Safety

The browser transport is fire-and-forget from the application's perspective.

The client does not wait for PostgreSQL transaction completion.

This is intentional:

~~~text
UI action
  |
  +--> business operation
  |
  +--> telemetry transport
           |
           v
        background
~~~

The source explicitly describes the transport as fire-and-forget so the user experience remains fast and fluid. fileciteturn13file0L79-L81

## 18. Tests

Create:

~~~text
src/telemetry/transport.spec.ts
~~~

Test:

~~~text
sendTelemetryEvent › sends an authenticated telemetry event to the dedicated endpoint
sendTelemetryEvent › does not send telemetry when there is no access token
sendTelemetryEvent › does not propagate fetch failures
~~~

The source lists these three focused transport tests. fileciteturn13file0L104-L109

### Test 1 — Authenticated dispatch

Verify:
- an access token exists;
- `fetch` is called;
- method is POST;
- endpoint ends in `/frontend-telemetry`;
- Authorization uses `Bearer <token>`;
- Content-Type is JSON;
- the telemetry event is serialized as the body;
- `keepalive` is enabled.

### Test 2 — No access token

Verify:

~~~text
accessToken = undefined
        |
        v
fetch() NOT called
~~~

### Test 3 — Fetch failure

Mock fetch rejection and verify:

~~~text
sendTelemetryEvent()
        |
        v
does not reject
~~~

## 19. Integration Testing

Stage 5 should verify the full path:

~~~text
UI interaction
    |
    v
trackEvent()
    |
    v
sendTelemetryEvent()
    |
    v
POST /frontend-telemetry
    |
    v
svc
    |
    v
PostgreSQL row
~~~

The source explicitly records end-to-end verification from UI interaction through the backend endpoint. fileciteturn13file0L22-L27

## 20. Privacy and Authentication

The Bearer token belongs only in the standard HTTP Authorization header.

Do not place it in:
- URL query parameters;
- telemetry properties;
- event route;
- JSON body.

The source explicitly states that Bearer tokens are never included in the URL or payload body. fileciteturn13file0L119-L121

The payload remains the same telemetry event contract established earlier.

## 21. No Database Changes

Stage 5 does not modify the database.

The database schema is:

~~~text
N/A — Client-side transport module.
~~~

The source explicitly marks database schema changes as not applicable. fileciteturn13file0L98-L100

Stage 4 already created the authoritative persistence boundary.

## 22. Common Implementation Mistakes

### Mistake 1 — Using the central API client

This can accidentally trigger global authentication refreshes, logout behavior, or user-facing error handling.

Use dedicated native `fetch`.

### Mistake 2 — Sending without checking authentication

Bad:

~~~text
always fetch()
~~~

Correct:

~~~text
access token exists?
    |
    +-- no -> drop
    +-- yes -> fetch
~~~

### Mistake 3 — Letting telemetry errors escape

Bad:

~~~ts
return fetch(...);
~~~

when callers do not handle the resulting rejection.

Correctly contain the failure inside the telemetry module.

### Mistake 4 — Blocking the UI

Do not await telemetry as part of the user's business operation.

### Mistake 5 — Removing CustomEvent

Transport and local event dispatch serve different boundaries. Keep both.

### Mistake 6 — Generating a new event ID during transport

Do not create a second UUID in `sendTelemetryEvent()`.

Use the event ID already generated by `createTelemetryEvent()` so retries remain idempotent.

## 23. Independent Implementation Exercise

Build a dedicated telemetry transport for another frontend application.

Requirements:

1. Accept a completed telemetry event object.
2. Resolve the backend URL through the application's existing URL configuration.
3. Read the current access token from the authentication state.
4. Drop the event if no token exists.
5. Send POST JSON to `/telemetry`.
6. Use `Authorization: Bearer <token>`.
7. Set `keepalive: true`.
8. Do not use the application's central API client.
9. Suppress network failures.
10. Do not block the business operation.
11. Preserve the original event ID.
12. Write tests for authenticated delivery, anonymous dropping, and fetch failure.

Then connect the transport to your existing `trackEvent()` function while retaining its local event boundary.

## 24. Validation Checklist

### Transport
- [ ] Dedicated transport module exists.
- [ ] Endpoint resolves through existing API configuration.
- [ ] Native fetch is used.
- [ ] POST is used.
- [ ] JSON body contains the telemetry event.
- [ ] Bearer authentication is present.
- [ ] `keepalive: true` is enabled.

### Authentication
- [ ] Access token is read from existing auth state.
- [ ] No token means no network request.
- [ ] Token is never placed in the URL or body.

### Failure isolation
- [ ] Fetch failures are swallowed.
- [ ] HTTP errors do not surface to the user.
- [ ] No telemetry failure interrupts the onboarding flow.

### Event architecture
- [ ] `trackEvent()` still creates the canonical event.
- [ ] Existing CustomEvent dispatch remains.
- [ ] Transport uses the existing `event_id`.

### Tests
- [ ] Authenticated transport test passes.
- [ ] Anonymous transport test passes.
- [ ] Fetch failure test passes.
- [ ] End-to-end UI-to-backend transmission is verified.

## 25. Scope Boundaries

### Implemented in Stage 5
- client HTTP transport;
- Bearer authentication integration;
- `keepalive: true`;
- failure containment;
- connection from `trackEvent()` to backend ingestion.

### Deferred to Stage 7
- frontend application-version tracking;
- Pinia authentication-store hardening.

The source explicitly records these scope boundaries. fileciteturn13file0L125-L128

## 26. What You Should Understand Before Recipe 06

You should now understand how a production telemetry system crosses the browser-to-backend boundary without allowing observability work to become part of the critical user journey.

The core pattern is:

~~~text
Business event
      |
      v
Canonical telemetry event
      |
      v
Dedicated transport
      |
      +--> authenticated?
      |       |
      |       +-- no -> drop
      |
      +--> fetch + keepalive
      |
      +--> failure -> suppress
      |
      v
Backend ingestion
      |
      v
Idempotent PostgreSQL source
~~~

Most importantly:
- telemetry is background work;
- authentication belongs in the transport boundary;
- business API infrastructure should not automatically control telemetry behavior;
- failures in telemetry must not become failures in the product journey;
- event identity must survive transport retries;
- `keepalive` matters when events occur immediately before navigation.

## Source Traceability

This recipe is derived from the supplied Stage 5 implementation record:
- Source commit: <code>9684b6aba</code>
- Subject: <code>feat: add authenticated frontend telemetry transport</code>
- Repository: <code>emi-frontend</code>
- Implementation scope: fileciteturn13file0L3-L16
- Implementation sequence: fileciteturn13file0L20-L27
- Technical changes: fileciteturn13file0L31-L40
- Design decisions: fileciteturn13file0L44-L49
- Architecture and transport behavior: fileciteturn13file0L53-L87
- Tests and validation: fileciteturn13file0L104-L115
- Privacy and scope boundaries: fileciteturn13file0L119-L128