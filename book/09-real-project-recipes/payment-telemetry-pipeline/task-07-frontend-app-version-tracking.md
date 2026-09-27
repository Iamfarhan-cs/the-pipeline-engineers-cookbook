# Recipe 07 — Frontend App Version Tracking

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 7  
> **Source implementation commit:** <code>0c4982e3b</code> — <code>feat: add frontend app version tracking</code>  
> **Source repository:** <code>emi-frontend</code>

## 1. What This Recipe Teaches

This recipe adds release-version metadata to every frontend telemetry event and hardens the telemetry transport against authentication-store initialization failures.

By completing it, you should understand how to:
- inject an application version at Vite build time;
- use a CI/CD-provided version while retaining a local-development fallback;
- extend a telemetry contract without changing its event identity;
- expose build metadata safely through TypeScript environment typings;
- stamp <code>app_version</code> onto every emitted telemetry event;
- keep authentication-store failures inside the telemetry failure boundary;
- test both version stamping and transport exception containment;
- separate frontend version metadata from later backend-persisted version metadata.

The source identifies the two goals of Stage 7: release-version stamping and stronger transport exception containment. fileciteturn15file0L3-L15

## 2. The Problem

Telemetry tells you what happened, but an event is much more useful when you know which frontend release generated it.

Consider:

~~~text
Event:
identity_verification_completed

Without version:
Which frontend release generated this?

With version:
identity_verification_completed
app_version = 1.5.0
~~~

This makes release-specific debugging and funnel analysis possible.

Stage 7 also addresses a second problem. The previous transport accessed the Pinia authentication store outside the complete exception boundary. If authentication-store initialization failed, that exception could escape the telemetry transport and affect the application flow. fileciteturn15file0L9-L15

## 3. Target Architecture

~~~text
CI/CD or local package metadata
            |
            v
VITE_APP_VERSION || package.json version
            |
            v
import.meta.env.VITE_APP_VERSION
            |
            v
createTelemetryEvent()
            |
            v
TelemetryEvent.app_version
            |
            v
sendTelemetryEvent()
            |
            +--> useAuthStore()
            |       |
            |       +--> token
            |       +--> exception caught
            |
            +--> POST /frontend-telemetry
~~~

The supplied source records this exact build-to-event flow and places authentication-store access inside the transport's exception boundary. fileciteturn15file0L47-L64

## 4. Step 1 — Choose the Version Source

Use two possible sources for the application version:

~~~text
VITE_APP_VERSION
       |
       +-- available -> use it
       |
       +-- unavailable
                |
                v
        package.json version
~~~

The rule is:

~~~text
appVersion = env.VITE_APP_VERSION || packageJson.version
~~~

The CI/CD environment should normally provide the release version. Local developer builds can fall back to the version declared in <code>package.json</code>. fileciteturn15file0L19-L25

### Why Have a Fallback?

Without a fallback, a local build could produce telemetry without version information.

That would make development and debugging less consistent.

The fallback also avoids forcing every local developer to manually define a build variable.

## 5. Step 2 — Inject the Version Through Vite

Update:

~~~text
vite.config.mts
~~~

Resolve the version during the build:

~~~text
VITE_APP_VERSION || package.json version
~~~

Then expose it as:

~~~text
import.meta.env.VITE_APP_VERSION
~~~

The important distinction is that this is **build-time metadata**, not a runtime request to a version service.

The source explicitly states that Vite injects the version at bundle compile time. fileciteturn15file0L30-L35

## 6. Why Build-Time Injection?

Frontend application version is part of the compiled release artifact.

Therefore the desired relationship is:

~~~text
Release build
     |
     v
version metadata
     |
     v
frontend bundle
     |
     v
telemetry events
~~~

This avoids a runtime dependency merely to discover the application's own release version.

The source also verifies that Vite replaces the environment reference with static string literals during bundle optimization. fileciteturn15file0L103-L105

## 7. Step 3 — Add TypeScript Environment Typing

Update:

~~~text
env.d.ts
~~~

Declare:

~~~text
ImportMetaEnv.VITE_APP_VERSION: string
~~~

This keeps TypeScript aware that the build environment exposes the new property.

Without the declaration, the runtime value may exist while TypeScript does not know the property is part of the application's environment contract.

The source explicitly lists the environment typing change as part of Stage 7. fileciteturn15file0L21-L23

## 8. Step 4 — Extend the Telemetry Contract

Update:

~~~text
src/telemetry/index.ts
~~~

Extend <code>TelemetryEvent</code> with:

~~~text
app_version: string
~~~

Then populate the value when <code>createTelemetryEvent()</code> creates the event.

The resulting conceptual event is:

~~~json
{
  "event_id": "...",
  "event_name": "identity_verification_completed",
  "event_version": 1,
  "occurred_at": "...",
  "route": "/verification",
  "app_version": "1.5.0",
  "properties": {}
}
~~~

The exact surrounding event contract should remain consistent with the earlier telemetry stages. The supplied source specifically adds <code>app_version</code> to the existing <code>TelemetryEvent</code>. fileciteturn15file0L21-L25

## 9. Version Metadata Is Not Event Identity

Do not replace or derive <code>event_id</code> from the application version.

The relationship is:

~~~text
event_id      -> identifies the individual event
app_version   -> identifies the frontend release
~~~

For example:

~~~text
event_id = UUID A
app_version = 1.5.0

event_id = UUID B
app_version = 1.5.0
~~~

These are two different events generated by the same application release.

The source explicitly states that version stamping does not alter the event's idempotency key. fileciteturn15file0L76-L78

## 10. Step 5 — Harden the Transport Exception Boundary

Update:

~~~text
src/telemetry/transport.ts
~~~

The previous design already suppressed fetch failures.

Stage 7 moves:

~~~text
useAuthStore()
getApiUrl()
~~~

inside the transport's <code>try/catch</code> boundary.

The source explicitly records this change. fileciteturn15file0L23-L25

## 11. Why Move `useAuthStore()` Inside `try/catch`?

Consider the failure path:

~~~text
sendTelemetryEvent()
       |
       v
useAuthStore()
       |
       X throws
       |
       v
uncaught exception
~~~

If the authentication-store call happens before the exception boundary, the transport's failure-isolation promise is incomplete.

The hardened design is:

~~~text
sendTelemetryEvent()
       |
       v
try {
       |
       +--> useAuthStore()
       |
       +--> get token
       |
       +--> fetch()
       |
       +--> catch failures
}
~~~

This guarantees that authentication-store initialization or hydration failures are treated as telemetry failures rather than application failures. fileciteturn15file0L40-L43

## 12. Total Exception Containment

Telemetry is deliberately non-critical.

The desired invariant is:

~~~text
Any telemetry-specific failure
          |
          v
      contained
          |
          v
application journey continues
~~~

Possible failure points now include:
- authentication-store initialization;
- token extraction;
- API URL resolution;
- fetch execution;
- network failure;
- HTTP failure handling.

The transport boundary should prevent these failures from escaping into the application's main execution path.

The source explicitly calls this total exception containment. fileciteturn15file0L40-L43

## 13. Architecture and Data Flow

~~~text
Vite build
    |
    +--> VITE_APP_VERSION
    |
    +--> package.json version fallback
    |
    v
import.meta.env.VITE_APP_VERSION
    |
    v
createTelemetryEvent()
    |
    +--> event_id
    +--> event_name
    +--> event_version
    +--> occurred_at
    +--> route
    +--> app_version
    +--> properties
    |
    v
sendTelemetryEvent()
    |
    +--> try
    |      +--> useAuthStore()
    |      +--> extract token
    |      +--> fetch POST /frontend-telemetry
    |
    +--> catch
           +--> silently suppress
~~~

The source describes this architecture directly. fileciteturn15file0L47-L64

## 14. Transaction Safety

Stage 7 remains entirely client-side.

Build metadata injection occurs at compile time.

Runtime telemetry dispatch remains non-blocking.

The source explicitly identifies the stage as client-side execution with compile-time metadata injection and non-blocking runtime dispatch. fileciteturn15file0L70-L72

Do not turn application version lookup into a blocking runtime dependency.

## 15. Idempotency

Adding <code>app_version</code> does not change event idempotency.

~~~text
event_id
   |
   v
same event identity

app_version
   |
   v
additional release metadata
~~~

If the same telemetry event is retried, its original <code>event_id</code> remains unchanged.

The source explicitly confirms that version stamping does not modify the idempotency key. fileciteturn15file0L76-L78

## 16. Integration With the Existing Pipeline

Stage 7 affects every telemetry event emitted by the frontend.

~~~text
emi-frontend
     |
     v
TelemetryEvent
     |
     +--> app_version
     |
     v
POST /frontend-telemetry
     |
     v
svc
~~~

The supplied source notes an important boundary: Stage 4's <code>svc</code> ingestion currently ignores the client version, while Stage 8 establishes persistent backend service-version handling. fileciteturn15file0L82-L85

## 17. Database Schema

Stage 7 introduces no database migration.

The source explicitly marks the database schema as:

~~~text
N/A — Client-side build configuration and contract extension.
~~~

Persistence of backend application version is deferred to Stage 8. fileciteturn15file0L89-L91

Do not add a database column as part of this stage unless the implementation scope is deliberately expanded.

## 18. Testing Strategy

Stage 7 adds two important assertions.

### Test 1 — Versioned Telemetry Event

The existing telemetry test verifies that the generated event contains:
- an ID;
- a timestamp;
- a pathname;
- <code>app_version</code>.

The supplied test expects:

~~~text
app_version === "1.5.0"
~~~

The source records this test explicitly. fileciteturn15file0L95-L100

### Test 2 — Authentication Store Failure

Mock the authentication store so that its initialization/access throws.

Expected result:

~~~text
useAuthStore() throws
       |
       v
sendTelemetryEvent() resolves safely
       |
       v
application does not receive an uncaught exception
~~~

The source adds the test <code>does not propagate auth store failures</code>. fileciteturn15file0L97-L100

## 19. Environment Validation

After configuring the Vite build, verify that:

~~~text
import.meta.env.VITE_APP_VERSION
            |
            v
static string in optimized bundle
~~~

The source explicitly states that Vite replacement produces static string literals during bundle optimization. fileciteturn15file0L103-L105

This is useful because it proves the value is actually embedded into the built frontend rather than depending on a runtime environment lookup that the browser cannot perform.

## 20. Privacy and Data Protection

Application version metadata is intentionally low-risk telemetry metadata.

A value such as:

~~~text
1.5.0
~~~

does not contain:
- user-identifiable data;
- hardware fingerprints;
- network attributes.

The source explicitly characterizes static version strings this way. fileciteturn15file0L109-L111

Do not use version metadata as a substitute for user, device, or network identifiers.

## 21. Common Implementation Mistakes

### Mistake 1 — Requiring `VITE_APP_VERSION` everywhere

This can make local development unnecessarily fragile.

Use the project-defined fallback to <code>package.json</code> version.

### Mistake 2 — Looking up the version at runtime

The application version is build metadata.

Inject it during the Vite build rather than making telemetry call another service to discover its own version.

### Mistake 3 — Adding `app_version` only to selected events

The requirement is to stamp the version onto the telemetry contract used by emitted events.

Do not manually add it to individual business-event call sites.

### Mistake 4 — Keeping `useAuthStore()` outside the exception boundary

Bad:

~~~text
useAuthStore()
try {
   fetch()
} catch {
   suppress
}
~~~

The store access itself can throw before the boundary.

Correct:

~~~text
try {
   useAuthStore()
   fetch()
} catch {
   suppress
}
~~~

### Mistake 5 — Changing `event_id` when adding version metadata

Version is descriptive release metadata. It is not event identity.

### Mistake 6 — Adding the Stage 8 database change early

Stage 7 is client-side. Backend persistence is explicitly deferred.

## 22. Independent Implementation Exercise

Add release-version tracking to a different frontend telemetry system.

Requirements:

1. Define a build-time application version variable.
2. Provide a package-version fallback for local builds.
3. Expose the variable through the build system.
4. Add the corresponding TypeScript environment declaration.
5. Add <code>app_version</code> to the telemetry event contract.
6. Populate it centrally when events are created.
7. Preserve the existing <code>event_id</code> semantics.
8. Place authentication-store access inside the complete transport error boundary.
9. Test the generated event's version value.
10. Test authentication-store failure isolation.
11. Verify the optimized build contains the version as static metadata.

Then deliberately make the authentication store throw during telemetry dispatch and verify that the application journey remains unaffected.

## 23. Validation Checklist

### Build configuration
- [ ] <code>VITE_APP_VERSION</code> is supported.
- [ ] <code>package.json</code> version is the fallback.
- [ ] Vite injects the resolved version.
- [ ] The optimized bundle contains the resolved static version.

### TypeScript
- [ ] <code>ImportMetaEnv.VITE_APP_VERSION</code> is declared.
- [ ] Type checking recognizes the variable.

### Telemetry contract
- [ ] <code>TelemetryEvent</code> includes <code>app_version: string</code>.
- [ ] <code>createTelemetryEvent()</code> populates it.
- [ ] Existing event identity remains unchanged.

### Transport
- [ ] <code>useAuthStore()</code> is inside the <code>try/catch</code> boundary.
- [ ] Authentication failures are suppressed.
- [ ] Existing fetch failure containment remains intact.
- [ ] Telemetry remains non-blocking.

### Privacy
- [ ] Version metadata contains no user-identifiable data.
- [ ] No device fingerprinting is added.
- [ ] No network attributes are added.

### Tests
- [ ] Versioned-event test passes.
- [ ] Auth-store failure test passes.
- [ ] Existing telemetry tests continue to pass.

## 24. Scope Boundaries

### Implemented in Stage 7
- frontend build version injection;
- <code>VITE_APP_VERSION</code> environment typing;
- <code>app_version</code> in the telemetry contract;
- version stamping on emitted events;
- transport exception-boundary hardening.

### Deferred to Stage 8
- persistence of backend application version in <code>public.frontend_telemetry_event</code>.

### Deferred to Stage 9
- extraction of version metadata in the Python batch pipeline.

The source explicitly records these boundaries. fileciteturn15file0L115-L119

## 25. What You Should Understand Before the Next Recipe

After Stage 7, telemetry contains an important release dimension:

~~~text
Telemetry event
      |
      +--> event identity
      +--> business event
      +--> occurrence time
      +--> route
      +--> properties
      +--> app_version
~~~

The frontend can now answer an important operational question:

~~~text
Which frontend release generated this event?
~~~

At the same time, the transport boundary is stronger:

~~~text
Auth initialization failure
          |
          v
Telemetry boundary
          |
          v
Suppressed
          |
          v
Application continues
~~~

The source records Stage 7 as complete, hardened, and verified. fileciteturn15file0L123-L125

## Source Traceability

This recipe is derived from the supplied Stage 7 implementation record:
- Source commit: <code>0c4982e3b</code>
- Subject: <code>feat: add frontend app version tracking</code>
- Repository: <code>emi-frontend</code>
- Overview and technical objective: fileciteturn15file0L3-L15
- Implementation sequence: fileciteturn15file0L19-L26
- Technical changes: fileciteturn15file0L30-L36
- Design decisions: fileciteturn15file0L40-L43
- Architecture and data flow: fileciteturn15file0L47-L64
- Transaction safety, idempotency, and integration: fileciteturn15file0L70-L85
- Tests and environment validation: fileciteturn15file0L95-L105
- Privacy and scope boundaries: fileciteturn15file0L109-L119