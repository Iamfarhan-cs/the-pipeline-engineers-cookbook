# Recipe 08 — Persist Backend Application Version in Telemetry

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 8  
> **Source implementation commit:** <code>718ba47b2</code> — <code>feat: persist backend application version in telemetry</code>  
> **Source repository:** <code>svc</code>

## 1. What This Recipe Teaches

This recipe adds authoritative backend application-version attribution to the telemetry ingestion database.

By completing it, you should understand how to:
- add a production database column through a migration;
- make the new column non-null while preserving existing rows;
- enforce non-empty values with PostgreSQL;
- expose service version through backend configuration;
- fall back from an environment default to a build version;
- map the new field through the Go DAO layer;
- populate the field during ingestion rather than trusting the client;
- keep telemetry event idempotency unchanged;
- synchronize a schema snapshot;
- harden an embedded PostgreSQL test harness;
- verify persistent version attribution with integration tests.

The source describes Stage 8 as persistent backend version tracking in <code>public.frontend_telemetry_event</code>, with the receiving <code>svc</code> version recorded as the authoritative value. fileciteturn16file0L3-L15

## 2. The Problem

Stage 7 introduced a frontend <code>app_version</code> into emitted telemetry.

However, a client-provided version cannot be treated as an authoritative record of which backend deployment received and persisted the event.

Therefore Stage 8 establishes a separate backend-derived version:

~~~text
HTTP telemetry request
        |
        v
svc ingestion process
        |
        +--> service configuration
        |
        v
backend app_version
        |
        v
PostgreSQL
~~~

The source explicitly identifies the backend service version as the authoritative audit trail rather than trusting client-provided version strings. fileciteturn16file0L8-L15

## 3. Important Terminology

Stage 8 uses the name <code>app_version</code> for the value persisted by the backend.

In this stage, the persisted value represents:

~~~text
receiving svc backend application version
~~~

Do not silently reinterpret this as the browser's frontend release version.

Stage 7 introduced frontend build-version metadata. Stage 8 persists the backend service version. The supplied source explicitly distinguishes the backend attribution from the client-provided value. fileciteturn16file0L43-L47

## 4. Target Architecture

~~~text
svc
Config
APP_VERSION = "2.14.0"
        |
        v
IngestFrontendTelemetryEvent()
        |
        | AppVersion = l.cfg.AppVersion
        v
CreateFrontendTelemetryEvent()
        |
        v
public.frontend_telemetry_event
        |
        +--> app_version = "2.14.0"
        +--> event_id uniqueness preserved
~~~

The source records this backend-config-to-database flow directly. fileciteturn16file0L51-L67

## 5. Step 1 — Create the Database Migration

Create:

~~~text
db/migrations/000293_frontend_telemetry_app_version.up.sql
db/migrations/000293_frontend_telemetry_app_version.down.sql
~~~

The migration adds:

~~~sql
ALTER TABLE public.frontend_telemetry_event
    ADD COLUMN app_version TEXT NOT NULL DEFAULT 'unknown',
    ADD CONSTRAINT frontend_telemetry_event_app_version_ck
    CHECK (app_version <> '');
~~~

The supplied implementation uses exactly this schema strategy. fileciteturn16file0L19-L22

## 6. Why Use `DEFAULT 'unknown'`?

The existing table already contains historical telemetry rows.

Adding a new <code>NOT NULL</code> column without a compatible backfill strategy could make the migration incompatible with those records.

The migration therefore establishes:

~~~text
Existing rows
     |
     v
app_version = 'unknown'
~~~

while all new ingestion rows receive the actual configured backend version.

This is the source's non-breaking backfill strategy. fileciteturn16file0L43-L47

### Important Distinction

<code>unknown</code> is a migration/backfill compatibility value.

It should not be treated as proof that a deployment was literally named <code>unknown</code>.

## 7. Step 2 — Enforce Non-Empty Values

The database constraint is:

~~~sql
CHECK (app_version <> '')
~~~

This rejects:

~~~text
app_version = ''
~~~

but allows meaningful values such as:

~~~text
2.14.0
dev
unknown
~~~

The source explicitly validates the empty-string constraint against PostgreSQL. fileciteturn16file0L123-L125

### Why Enforce This in PostgreSQL?

Application validation alone is not enough.

Other write paths could potentially bypass Go business logic.

The database therefore protects its own invariant:

~~~text
Any persisted app_version
        |
        v
must not equal ''
~~~

## 8. Step 3 — Register Migration Version

Update:

~~~text
db/migrate.go
~~~

Increase the migration version:

~~~text
292 -> 293
~~~

The supplied implementation explicitly records this migration registration step. fileciteturn16file0L21-L22

Always keep the migration version and migration files synchronized.

## 9. Step 4 — Add Backend Configuration

Update backend configuration so the service exposes its application version.

The supplied configuration adds:

~~~go
AppVersion string envconfig:"APP_VERSION" default:"dev"
~~~

The source identifies this exact configuration field and default. fileciteturn16file0L33-L35

The conceptual flow is:

~~~text
APP_VERSION
    |
    +--> supplied -> use supplied value
    |
    +--> default -> dev
    |
    v
cfg.AppVersion
~~~

## 10. Step 5 — Resolve the Build Version

The service configuration also falls back to <code>shared.BuildVersion</code> when <code>APP_VERSION</code> remains at its default or is empty.

The resulting concept is:

~~~text
APP_VERSION
    |
    +--> meaningful value -> cfg.AppVersion
    |
    +--> default/empty
             |
             v
       shared.BuildVersion
~~~

The source explicitly records this fallback in <code>svc/svc.go</code>. fileciteturn16file0L33-L35

This allows deployment infrastructure to provide a release version while retaining a sensible development/build fallback.

## 11. Step 6 — Extend the DAO Model

Update:

~~~text
db/dao/frontend_telemetry.go
~~~

Add:

~~~go
AppVersion string `gorm:"column:app_version;not null"`
~~~

This maps the Go data model to the new PostgreSQL column.

The source explicitly records this GORM mapping. fileciteturn16file0L23-L24

The resulting model relationship is:

~~~text
Go DAO field
     |
     | GORM mapping
     v
public.frontend_telemetry_event.app_version
~~~

## 12. Step 7 — Populate Version During Ingestion

Update:

~~~text
logic/frontend_telemetry.go
~~~

During event ingestion, assign:

~~~go
AppVersion: l.cfg.AppVersion
~~~

This is the critical trust boundary.

Do not derive the authoritative backend value from:
- request headers;
- client telemetry properties;
- frontend JavaScript state;
- query parameters;
- user-controlled payload data.

Instead:

~~~text
Authenticated request
       |
       v
svc process configuration
       |
       v
AppVersion
       |
       v
database row
~~~

The supplied source explicitly records the assignment from service configuration during ingestion. fileciteturn16file0L23-L24

## 13. Why Backend Attribution Is Authoritative

Suppose a browser sends:

~~~text
frontend app_version = "1.0.0"
~~~

but the request is actually received by:

~~~text
svc = "2.14.0"
~~~

The backend record should be able to say:

~~~text
receiving backend version = 2.14.0
~~~

This answers the operational question:

~~~text
Which backend deployment actually received and persisted this event?
~~~

The source explicitly identifies this as the authoritative backend attribution model. fileciteturn16file0L43-L46

## 14. Step 8 — Synchronize the Schema Snapshot

Regenerate:

~~~text
db/schema/0-schema.sql
~~~

The snapshot should now include the new <code>app_version</code> column and constraint.

The source records schema snapshot synchronization through migration 293. fileciteturn16file0L21-L26

Do not update a generated schema snapshot manually if the repository's normal generation process is available.

## 15. Updated Database Schema

The table now contains:

| Column | Type | Constraints | Description |
|---|---|---|---|
| <code>id</code> | UUID | PRIMARY KEY | Surrogate identifier |
| <code>event_id</code> | UUID | NOT NULL UNIQUE | Client event UUID |
| <code>user_id</code> | UUID | NOT NULL FK | Authenticated client ID |
| <code>event_name</code> | TEXT | NOT NULL CHECK | Whitelisted event name |
| <code>event_version</code> | INTEGER | NOT NULL CHECK | Event schema version |
| <code>occurred_at</code> | TIMESTAMPTZ | NOT NULL | Occurrence timestamp |
| <code>route</code> | TEXT | NOT NULL CHECK | Source route path |
| <code>properties</code> | JSONB | NOT NULL CHECK | Scalar property payload |
| <code>received_at</code> | TIMESTAMPTZ | NOT NULL | Server ingestion timestamp |
| <code>app_version</code> | TEXT | NOT NULL CHECK (<code>&lt;&gt; ''</code>) | Receiving backend service version |

The supplied source documents this updated schema. fileciteturn16file0L93-L112

## 16. Architecture and Data Flow

~~~text
HTTP telemetry request
        |
        v
svc configuration
        |
        +--> cfg.AppVersion
        |
        v
IngestFrontendTelemetryEvent()
        |
        | AppVersion = l.cfg.AppVersion
        v
DAO / Repository
        |
        v
INSERT frontend_telemetry_event
        |
        +--> event_id
        +--> user_id
        +--> event_name
        +--> event_version
        +--> occurred_at
        +--> route
        +--> properties
        +--> received_at
        +--> app_version
        |
        v
PostgreSQL
~~~

The supplied implementation shows the version assignment and insert flow directly. fileciteturn16file0L51-L67

## 17. Transaction Safety

The schema change is applied through the project's PostgreSQL DDL migration mechanism.

Ingestion continues to use atomic single-statement inserts.

The source explicitly describes the migration as transactional and ingestion inserts as atomic. fileciteturn16file0L70-L74

The important separation is:

~~~text
Schema change
     |
     v
Migration transaction

Telemetry event
     |
     v
Atomic INSERT
~~~

## 18. Idempotency

Adding <code>app_version</code> must not alter the existing telemetry idempotency behavior.

The ingestion path continues to use:

~~~text
ON CONFLICT (event_id) DO NOTHING
~~~

The migration itself has clean up/down behavior, while event-level idempotency remains tied to <code>event_id</code>.

The source explicitly states that ingestion idempotency on <code>event_id</code> is preserved. fileciteturn16file0L78-L80

## 19. Test Harness Hardening

Stage 8 also changes the embedded PostgreSQL test environment.

Update:

~~~text
internal/testutil/embedded_postgres_main.go
~~~

Configure explicit UTF-8 encoding:

~~~text
Encoding("UTF8")
~~~

The source records this as part of the test-harness hardening. fileciteturn16file0L25-L26

### Why Make Encoding Explicit?

Integration tests should not depend on an implicit or host-specific database encoding.

Making the encoding explicit makes the embedded PostgreSQL environment closer to the expected application database assumptions and makes test behavior more deterministic.

## 20. Timestamp Equality Hardening

The supplied implementation also aligned timestamp equality checks using:

~~~go
now.Equal(stored.OccurredAt)
~~~

This avoids relying on representation-level equality when comparing time values.

The source explicitly records this timestamp comparison adjustment alongside the embedded PostgreSQL configuration change. fileciteturn16file0L25-L26

## 21. Testing Strategy

Stage 8 updates the integration test:

~~~text
logic/frontend_telemetry_test.go
~~~

The important assertion is that the persisted row's version matches the service configuration:

~~~text
cfg.AppVersion
      |
      v
ingestion logic
      |
      v
stored.AppVersion
~~~

The source specifically records <code>TestIngestFrontendTelemetryEventStoresAuthenticatedUser</code> as verifying persistent version attribution and timestamp accuracy. fileciteturn16file0L116-L119

## 22. PostgreSQL Validation

Validate migration 293 against PostgreSQL.

At minimum verify:

~~~text
migration 000293 executes
        |
        v
app_version exists
        |
        v
app_version cannot be NULL
        |
        v
app_version != ''
~~~

The supplied source explicitly reports that PostgreSQL 16 validation confirmed the check-constraint violation for <code>app_version = ''</code>. fileciteturn16file0L123-L125

## 23. Privacy and Data Protection

Backend service-version strings such as:

~~~text
2.14.0
dev
~~~

describe software releases rather than users or devices.

The source explicitly states that service version strings carry no client or user-identifiable data. fileciteturn16file0L129-L131

Do not use <code>app_version</code> as a proxy for identity, device fingerprinting, or network metadata.

## 24. Common Implementation Mistakes

### Mistake 1 — Trusting the frontend version as the authoritative backend version

The client can provide a version value, but Stage 8's persisted backend value must come from the receiving service configuration.

### Mistake 2 — Adding a nullable column

The source requires:

~~~text
NOT NULL
~~~

Use the non-breaking default/backfill strategy instead of leaving the invariant optional.

### Mistake 3 — Forgetting the empty-string constraint

<code>NOT NULL</code> prevents <code>NULL</code>, but it does not prevent:

~~~text
app_version = ''
~~~

That is why the PostgreSQL <code>CHECK</code> constraint is required.

### Mistake 4 — Forgetting migration registration

Creating the SQL file is not enough if the project's migration registry remains at version 292.

### Mistake 5 — Updating the DAO but not ingestion logic

The field must travel through the complete chain:

~~~text
configuration
    -> logic
    -> DAO
    -> PostgreSQL
~~~

### Mistake 6 — Breaking event idempotency

Do not change the existing <code>event_id</code> conflict behavior while adding version attribution.

### Mistake 7 — Updating generated schema manually

Regenerate the schema snapshot through the project's established process.

### Mistake 8 — Ignoring test-environment encoding

If integration tests use embedded PostgreSQL, make environment assumptions explicit rather than relying on host defaults.

## 25. Independent Implementation Exercise

Add authoritative backend release-version tracking to another event-ingestion service.

Requirements:

1. Add an <code>app_version</code> column to the existing event table.
2. Make it <code>TEXT NOT NULL</code>.
3. Use a compatibility default for historical rows.
4. Add a database <code>CHECK</code> preventing empty strings.
5. Register the migration.
6. Add an application configuration field.
7. Provide an environment variable for the service version.
8. Define a sensible build-version fallback.
9. Map the field in the persistence model.
10. Populate it from server configuration during ingestion.
11. Do not trust client-provided version metadata for the authoritative backend field.
12. Preserve existing event-idempotency behavior.
13. Update the schema snapshot if the project uses one.
14. Add an integration test proving the persisted version equals the service configuration.
15. Validate the empty-string constraint against PostgreSQL.

Then test the migration against a database containing existing telemetry rows and verify that historical rows receive the compatibility default.

## 26. Validation Checklist

### Migration
- [ ] Migration <code>000293</code> exists.
- [ ] Up migration adds <code>app_version</code>.
- [ ] Down migration removes the change.
- [ ] Migration registry is updated from 292 to 293.
- [ ] Existing rows receive the compatibility default.
- [ ] Column is <code>NOT NULL</code>.
- [ ] Empty string is rejected.

### Backend configuration
- [ ] <code>APP_VERSION</code> is supported.
- [ ] Development default is defined.
- [ ] Build-version fallback is configured.
- [ ] <code>cfg.AppVersion</code> contains the authoritative service version.

### Data layer
- [ ] DAO maps <code>app_version</code>.
- [ ] Ingestion populates it from <code>cfg.AppVersion</code>.
- [ ] Repository persistence includes the field.
- [ ] Schema snapshot is synchronized.

### Idempotency
- [ ] <code>event_id</code> uniqueness remains unchanged.
- [ ] <code>ON CONFLICT (event_id) DO NOTHING</code> behavior remains intact.

### Tests
- [ ] Embedded PostgreSQL uses explicit UTF-8.
- [ ] Timestamp comparisons use the appropriate time equality semantics.
- [ ] Integration test verifies stored version attribution.
- [ ] Migration runs successfully on PostgreSQL.
- [ ] Empty-string constraint is verified.

### Privacy
- [ ] Service version contains no user-identifiable information.
- [ ] Client-provided version is not used as the authoritative backend value.

## 27. Scope Boundaries

### Implemented in Stage 8
- source-table <code>app_version</code> column;
- non-empty database constraint;
- backend application-version configuration;
- build-version fallback;
- Go DAO mapping;
- ingestion-time version attribution;
- schema snapshot synchronization;
- embedded PostgreSQL UTF-8 hardening;
- persistent version attribution testing.

### Deferred to Stage 9
- Python pipeline extractor changes;
- staging-table adaptation for version metadata.

The supplied source explicitly defines this Stage 8 boundary. fileciteturn16file0L135-L138

## 28. What You Should Understand Before the Next Recipe

Stage 8 creates the authoritative backend release dimension in the operational telemetry store:

~~~text
svc deployment
     |
     v
cfg.AppVersion
     |
     v
telemetry ingestion
     |
     v
frontend_telemetry_event.app_version
~~~

The key engineering lessons are:
- client metadata and server attribution are different trust boundaries;
- database constraints should enforce important invariants;
- historical data requires an explicit migration/backfill strategy;
- application configuration should feed authoritative service metadata;
- schema, DAO, business logic, and tests must evolve together;
- adding metadata must not accidentally change idempotency;
- generated schema artifacts must remain synchronized with migrations.

The source records Stage 8 as complete, schema-verified, and active in <code>svc</code>. fileciteturn16file0L142-L144

## Source Traceability

This recipe is derived from the supplied Stage 8 implementation record:
- Source commit: <code>718ba47b2</code>
- Subject: <code>feat: persist backend application version in telemetry</code>
- Repository: <code>svc</code>
- Overview and technical objective: fileciteturn16file0L3-L15
- Implementation sequence: fileciteturn16file0L19-L26
- Technical changes: fileciteturn16file0L31-L39
- Design decisions: fileciteturn16file0L43-L47
- Architecture and data flow: fileciteturn16file0L51-L67
- Transaction safety and idempotency: fileciteturn16file0L72-L80
- Database schema: fileciteturn16file0L93-L112
- Tests and PostgreSQL validation: fileciteturn16file0L116-L125
- Privacy and scope boundaries: fileciteturn16file0L129-L138