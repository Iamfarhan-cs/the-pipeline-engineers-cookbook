# Recipe 04 — Authenticated Frontend Telemetry Ingestion

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 4  
> **Source implementation commit:** 09bcaa585 — feat: add authenticated frontend telemetry ingestion  
> **Source repository:** svc

## 1. What This Recipe Teaches

This recipe builds the backend ingestion boundary between the frontend telemetry system and PostgreSQL.

By completing it, you should understand how to:
- authenticate telemetry requests with the existing Bearer-token system;
- derive user identity from authenticated server context rather than trusting the client;
- validate event IDs, names, versions, routes, and properties;
- enforce an explicit event allow-list;
- restrict properties to flat scalar JSON and a 16 KiB ceiling;
- persist events into PostgreSQL;
- preserve client event time while adding server ingestion time;
- make ingestion idempotent with a unique event ID and ON CONFLICT DO NOTHING;
- test validation and authenticated database attribution.

The source implementation introduced POST /frontend-telemetry, server-side attribution, allow-list validation, scalar-property validation, PostgreSQL persistence, and idempotent event handling. fileciteturn12file0L3-L16

## 2. The Problem

A browser must not be trusted to write directly to the telemetry database or choose the user identity attached to an event.

Use this boundary:

~~~text
Frontend
   | Bearer token
   v
Authenticated backend
   |
   +--> derive authenticated user_id
   +--> validate event
   +--> validate properties
   +--> assign received_at
   v
PostgreSQL
~~~

The dedicated ingestion API exists to authenticate callers, bind events to the authenticated user, reject malformed or oversized payloads, and provide a durable source log. fileciteturn12file0L9-L16

## 3. Target Architecture

~~~text
POST /frontend-telemetry
        |
        v
Authentication Middleware
        |
        +--> validate Bearer token
        +--> extract authenticated client ID
        |
        v
FrontendTelemetryEventRequest.Validate()
        |
        +--> UUID
        +--> event name <= 100
        +--> route <= 2048
        +--> properties <= 16 KiB
        +--> flat scalar JSON object
        |
        v
CoreLogic.IngestFrontendTelemetryEvent()
        |
        +--> event_version == 1
        +--> event_name allow-list
        +--> UserID = authenticated client ID
        +--> OccurredAt = client timestamp
        +--> ReceivedAt = server UTC timestamp
        |
        v
Repository
        |
        v
PostgreSQL
        |
        +--> UNIQUE(event_id)
        +--> ON CONFLICT DO NOTHING
~~~

The source implementation documents this request-to-database flow. fileciteturn12file0L52-L74

## 4. Step 1 — Create the Database Migration

Create:

~~~text
db/migrations/000292_frontend_telemetry_events.up.sql
db/migrations/000292_frontend_telemetry_events.down.sql
~~~

Create the table:

~~~text
public.frontend_telemetry_event
~~~

The migration defines database-level checks for event name, event version, route, and JSON properties. fileciteturn12file0L20-L27

## 5. Database Schema

| Column | Type | Constraints | Purpose |
|---|---|---|---|
| id | UUID | PRIMARY KEY DEFAULT gen_random_uuid() | Internal surrogate key |
| event_id | UUID | NOT NULL UNIQUE | Client-generated event identity |
| user_id | UUID | NOT NULL FK to public.user(id) | Authenticated user identity |
| event_name | TEXT | NOT NULL, length 1–100 | Whitelisted event |
| event_version | INTEGER | NOT NULL, > 0 | Schema version |
| occurred_at | TIMESTAMPTZ | NOT NULL | Client event time |
| route | TEXT | NOT NULL, length 1–2048 | Frontend pathname |
| properties | JSONB | NOT NULL, object | Scalar telemetry properties |
| received_at | TIMESTAMPTZ | NOT NULL, UTC default | Server ingestion time |

The source defines these columns and constraints. fileciteturn12file0L100-L113

### Indexes

Create:

~~~text
frontend_telemetry_event_user_occurred_idx
frontend_telemetry_event_name_occurred_idx
frontend_telemetry_event_occurred_idx
~~~

The source indexes user/time, event-name/time, and event-time access patterns. fileciteturn12file0L115-L118

## 6. Step 2 — Register the Migration

Update db/migrate.go and bump migrationVersion from 291 to 292.

The source implementation explicitly records this migration registration. fileciteturn12file0L22-L24

## 7. Step 3 — Build the Data Access Layer

Create:

~~~text
db/dao/frontend_telemetry.go
db/repo/frontend_telemetry.go
~~~

The DAO represents the persisted event and the repository owns database persistence.

The critical repository behavior is:

~~~go
clause.OnConflict{DoNothing: true}
~~~

This makes repeated delivery of the same event ID harmless. The source implementation explicitly uses this GORM conflict strategy. fileciteturn12file0L22-L27

## 8. Step 4 — Define and Validate the Request DTO

Create:

~~~text
http/rq/frontend_telemetry.go
~~~

Implement FrontendTelemetryEventRequest.Validate().

Validate:

| Field | Rule |
|---|---|
| event_id | Required valid UUID |
| event_name | Maximum 100 characters |
| event_version | Valid positive version; logic requires version 1 |
| occurred_at | Required timestamp |
| route | Maximum 2048 characters |
| properties | JSON object |
| property values | string, number, bool, or null only |
| properties size | Maximum 16 KiB |

These validation rules are documented in the source implementation. fileciteturn12file0L24-L26

## 9. Why Flat Scalar Properties?

Reject arbitrary nested objects and arrays.

Reject conceptually:

~~~json
{
  "form": {
    "email": "user@example.com"
  }
}
~~~

Accept conceptually:

~~~json
{
  "account_type": "PERSONAL",
  "step": 2,
  "completed": true,
  "optional_value": null
}
~~~

This prevents uncontrolled object graphs, keeps the payload bounded, and makes the persisted contract predictable. fileciteturn12file0L44-L48

## 10. Why the 16 KiB Limit?

The properties payload is capped at 16 KiB.

This limits accidental database bloat and oversized request abuse while keeping telemetry payloads small enough for downstream processing. The source explicitly identifies the limit as part of strict payload bounds and privacy/security protection. fileciteturn12file0L44-L48 fileciteturn12file0L142-L146

## 11. Step 5 — Implement the Business Logic Layer

Create:

~~~text
logic/frontend_telemetry.go
~~~

Implement CoreLogic.IngestFrontendTelemetryEvent().

### Rule 1 — Server-side identity

Never trust a client-supplied user_id.

Use:

~~~text
UserID = *client.ID
~~~

The authenticated client identity is authoritative. fileciteturn12file0L34-L38

### Rule 2 — Event version

Accept:

~~~text
event_version == 1
~~~

Reject unsupported versions.

### Rule 3 — Event allow-list

Accept exactly:

~~~text
account_type_selected
identity_verification_completed
liveness_verification_started
liveness_verification_completed
onboarding_form_submitted
~~~

Reject unknown event names with HTTP 400. The source defines this exact five-event allow-list. fileciteturn12file0L34-L38

## 12. Step 6 — Assign Server-Side Ingestion Time

Keep event time and ingestion time separate:

~~~text
occurred_at = when the frontend reports the event happened
received_at = when the backend receives the event
~~~

Set:

~~~text
OccurredAt = payload.OccurredAt.UTC()
ReceivedAt = time.Now().UTC()
~~~

Server-side received_at prevents client clock skew from determining ingestion ordering. fileciteturn12file0L44-L46

## 13. Step 7 — Register the API Endpoint

Create:

~~~text
api/frontend_telemetry.go
~~~

Define FrontendTelemetryAPI with authentication required and register it in svc.go.

Endpoint:

~~~http
POST /frontend-telemetry
~~~

The source implementation configures mandatory Bearer authentication. fileciteturn12file0L26-L27 fileciteturn12file0L34-L38

## 14. Complete Request Flow

~~~text
POST /frontend-telemetry
        |
        v
Bearer authentication
        |
        +---- missing/invalid -> 401
        |
        v
Request DTO validation
        |
        +---- malformed -> 400
        |
        v
Business validation
        |
        +---- unsupported event -> 400
        |
        v
Server attribution
        |
        +--> user_id = authenticated client ID
        +--> received_at = server UTC time
        |
        v
Repository
        |
        v
PostgreSQL
        |
        +---- duplicate event_id -> ignored
        |
        v
HTTP success
~~~

## 15. Idempotency

Use:

~~~text
event_id UUID NOT NULL UNIQUE
~~~

and:

~~~sql
INSERT INTO frontend_telemetry_event (...)
VALUES (...)
ON CONFLICT (event_id) DO NOTHING;
~~~

Therefore repeated delivery of the same event ID produces exactly one persistent row while repeated requests can still return success. fileciteturn12file0L44-L48 fileciteturn12file0L85-L89

## 16. Why Server-Side User Attribution Matters

The secure identity flow is:

~~~text
Bearer token
     |
     v
Authentication middleware
     |
     v
Authenticated client
     |
     v
client.ID
     |
     v
persisted user_id
~~~

This prevents one authenticated client from attributing an event to another user by changing request-body data. fileciteturn12file0L42-L48

## 17. Transaction Safety

Each event insert is a single PostgreSQL statement. The source describes these inserts as atomic and relies on the unique event ID conflict mechanism for duplicates. fileciteturn12file0L79-L81

## 18. Tests

### Request validation

Create:

~~~text
http/rq/frontend_telemetry_test.go
~~~

Cover:

~~~text
TestFrontendTelemetryEventRequestValidateAcceptsScalarProperties
TestFrontendTelemetryEventRequestValidateRejectsNestedProperties
TestFrontendTelemetryEventRequestValidateRejectsArrayProperties
TestFrontendTelemetryEventRequestValidateRejectsInvalidJSON
TestFrontendTelemetryEventRequestValidateRejectsOversizedProperties
TestFrontendTelemetryEventRequestValidateRejectsMissingRequiredFields
~~~

The source lists these exact request-validation tests. fileciteturn12file0L122-L130

### Logic and database

Create:

~~~text
logic/frontend_telemetry_test.go
~~~

Verify authenticated attribution with embedded PostgreSQL:

~~~text
TestIngestFrontendTelemetryEventStoresAuthenticatedUser
~~~

The source states that this test verifies foreign-key attribution against the embedded PostgreSQL harness. fileciteturn12file0L131-L132

## 19. Essential Negative Tests

| Scenario | Expected result |
|---|---|
| Missing Bearer token | 401 |
| Invalid event UUID | Validation failure |
| Missing event name | Validation failure |
| Event name > 100 chars | Validation failure |
| Route > 2048 chars | Validation failure |
| Nested properties | Validation failure |
| Array properties | Validation failure |
| Invalid JSON | Validation failure |
| Properties > 16 KiB | Validation failure |
| Unsupported event name | 400 |
| Unsupported event version | Rejected |
| Duplicate event ID | Success, no second row |

These cases follow the documented validation and allow-list requirements. fileciteturn12file0L24-L27

## 20. Privacy and Security

Authentication is mandatory and unauthenticated requests receive 401 Unauthorized.

User identity is derived exclusively from server authentication tokens.

The properties payload is capped at 16 KiB.

The source explicitly identifies these controls as protections against unauthorized attribution, malformed input, and oversized payload abuse. fileciteturn12file0L142-L146

## 21. Integration With the Larger Pipeline

~~~text
Stage 1 + Stage 2
Frontend event generation
        |
        v
Stage 5
Frontend HTTP transport
        |
        v
Stage 4
POST /frontend-telemetry
        |
        v
public.frontend_telemetry_event
        |
        v
Stage 6
Batch pipeline extractor
~~~

The source identifies the frontend transport as the upstream producer and the Stage 6 batch extractor as the downstream consumer. fileciteturn12file0L93-L96

## 22. Independent Implementation Exercise

Build an ingestion endpoint for a fictional checkout_started event.

Requirements:
1. Require authentication.
2. Derive user_id from the authenticated session.
3. Do not trust client-supplied identity.
4. Require event_version == 1.
5. Allow only explicitly registered event names.
6. Require a UUID event ID.
7. Limit route length.
8. Allow only flat scalar JSON properties.
9. Enforce a property-size limit.
10. Record event time and server ingestion time.
11. Persist to PostgreSQL.
12. Make event_id unique.
13. Use ON CONFLICT DO NOTHING for retries.
14. Test validation and authenticated database attribution.

## 23. Validation Checklist

### API
- [ ] POST /frontend-telemetry exists.
- [ ] Bearer authentication is required.
- [ ] Missing/invalid authentication is rejected.

### Validation
- [ ] Event ID is a valid UUID.
- [ ] Event name is bounded.
- [ ] Route is bounded.
- [ ] Properties are valid JSON objects.
- [ ] Properties contain only scalar values.
- [ ] Properties are capped at 16 KiB.
- [ ] Event version is supported.
- [ ] Event name is allow-listed.

### Attribution
- [ ] User ID comes from authenticated server context.
- [ ] Client cannot choose persisted user ID.
- [ ] received_at is generated server-side.

### Persistence
- [ ] event_id is unique.
- [ ] Duplicate events do not create duplicate rows.
- [ ] Required indexes exist.
- [ ] Foreign key to public.user exists.

### Testing
- [ ] Request validation tests pass.
- [ ] Nested JSON is rejected.
- [ ] Arrays are rejected.
- [ ] Oversized properties are rejected.
- [ ] Missing fields are rejected.
- [ ] Authenticated user attribution is tested with PostgreSQL.

## 24. Scope Boundaries

### Implemented in Stage 4
- authenticated ingestion API;
- request validation;
- event/version allow-listing;
- server-side user attribution;
- PostgreSQL source table;
- idempotent persistence;
- focused request and embedded PostgreSQL tests.

### Deferred to Stage 5
- frontend client HTTP transport connection.

### Deferred to Stage 8
- receiving backend service application-version persistence.

### Deferred to Stage 12
- cursor indexing on (received_at, event_id).

The source explicitly records these boundaries. fileciteturn12file0L150-L154

## 25. What You Should Understand Before Recipe 05

You should now be able to design a trustworthy ingestion boundary between an untrusted frontend and a persistent data platform.

The key pattern is:

~~~text
Untrusted client
      |
      v
Authentication
      |
      v
DTO validation
      |
      v
Business allow-list
      |
      v
Server attribution
      |
      v
Idempotent PostgreSQL persistence
      |
      v
Authoritative source table
~~~

Key lessons:
- authenticate before accepting telemetry as trusted data;
- derive identity server-side;
- validate at multiple boundaries;
- keep the accepted event vocabulary explicit;
- preserve event time while adding server ingestion time;
- use database uniqueness for idempotency;
- keep the source table durable and queryable for downstream extraction.

## Source Traceability

This recipe is derived from the supplied Stage 4 implementation record:
- Source commit: <code>09bcaa585</code>
- Subject: <code>feat: add authenticated frontend telemetry ingestion</code>
- Repository: <code>svc</code>
- Implementation scope: fileciteturn12file0L3-L16
- Implementation sequence: fileciteturn12file0L20-L27
- Technical changes: fileciteturn12file0L32-L38
- Design decisions: fileciteturn12file0L42-L48
- Architecture and persistence: fileciteturn12file0L52-L89
- Database schema: fileciteturn12file0L100-L118
- Tests and validation: fileciteturn12file0L122-L138
- Privacy and scope boundaries: fileciteturn12file0L142-L154