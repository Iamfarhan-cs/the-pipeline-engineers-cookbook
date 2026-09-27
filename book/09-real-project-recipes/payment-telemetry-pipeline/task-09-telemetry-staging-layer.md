# Recipe 09 — Telemetry Staging Layer

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 9  
> **Source implementation commit:** <code>1b78097</code> — <code>feat: add telemetry staging layer</code>  
> **Source repository:** <code>devops</code>

## 1. What This Recipe Teaches

This recipe introduces the persistent staging layer between the operational telemetry source and the downstream data-quality pipeline.

By completing it, you should understand how to:
- create a pipeline-owned PostgreSQL staging table;
- extend an extraction domain model when the source schema gains metadata;
- extract <code>app_version</code> alongside the existing telemetry fields;
- write batches efficiently with Psycopg bulk execution;
- serialize Python properties into PostgreSQL JSONB correctly;
- make staging idempotent with a unique event identifier;
- handle empty batches without unnecessary database work;
- distinguish operational source storage from pipeline-owned storage;
- test bulk insertion, empty input, and duplicate handling;
- understand why validation, checkpointing, orchestration, and quarantine remain separate stages.

The supplied Stage 9 implementation establishes <code>public.telemetry_event_staging</code>, adds <code>app_version</code> to <code>TelemetryEvent</code>, and implements the bulk staging writer. fileciteturn17file0L3-L15

## 2. The Problem

Stage 6 introduced a Python extractor that reads directly from:

~~~text
public.frontend_telemetry_event
~~~

That table belongs to the operational ingestion side of the system.

Downstream reporting and data-quality processing should not continuously operate against that OLTP source.

Stage 9 introduces a pipeline-owned landing area:

~~~text
svc
 |
 v
public.frontend_telemetry_event
 |
 | extract
 v
TelemetryEvent
 |
 | stage
 v
public.telemetry_event_staging
 |
 +--> validation
 +--> orchestration
 +--> downstream processing
~~~

The source explicitly describes the staging table as a pipeline-owned landing table that decouples downstream processing from the <code>svc</code> ingestion table. fileciteturn17file0L11-L15

## 3. Why a Separate Staging Table?

Without a staging layer, downstream systems would need to query the operational table directly.

That creates coupling:

~~~text
Operational database
       |
       +--> API writes
       +--> telemetry ingestion
       +--> analytics queries
       +--> data-quality queries
       +--> batch processing
~~~

With staging:

~~~text
Operational source
       |
       v
Extraction
       |
       v
Pipeline staging
       |
       +--> validation
       +--> quarantine
       +--> analytics
       +--> downstream processing
~~~

The staging layer therefore establishes ownership and workload isolation.

## 4. Target Architecture

~~~text
public.frontend_telemetry_event
            |
            v
     extract_events(conn)
            |
            v
     list[TelemetryEvent]
            |
            v
       stage_events(conn)
            |
            v
public.telemetry_event_staging
            |
            +--> Stage 10 validation
            +--> Stage 11 orchestration
            +--> Stage 13 quarantine
~~~

The supplied source records this extraction-to-staging flow. fileciteturn17file0L52-L70

## 5. Step 1 — Create the Staging Migration

Create:

~~~text
migrations/001_create_telemetry_event_staging.sql
~~~

The migration creates:

~~~text
public.telemetry_event_staging
~~~

The table is owned by the telemetry pipeline rather than the operational <code>svc</code> application.

The source explicitly identifies the migration as the first implementation step. fileciteturn17file0L19-L24

## 6. Step 2 — Design the Staging Table

The Stage 9 table contains:

| Column | Type | Constraints | Purpose |
|---|---|---|---|
| <code>id</code> | BIGINT | GENERATED ALWAYS AS IDENTITY PRIMARY KEY | Staging surrogate key |
| <code>event_id</code> | UUID | NOT NULL UNIQUE | Source event identity |
| <code>user_id</code> | UUID | NOT NULL | Associated user |
| <code>event_name</code> | TEXT | NOT NULL | Telemetry event name |
| <code>event_version</code> | INTEGER | NOT NULL | Event schema version |
| <code>occurred_at</code> | TIMESTAMPTZ | NOT NULL | Event occurrence time |
| <code>route</code> | TEXT | NOT NULL | Frontend pathname |
| <code>properties</code> | JSONB | NOT NULL | Event properties |
| <code>received_at</code> | TIMESTAMPTZ | NOT NULL | Backend ingestion time |
| <code>app_version</code> | TEXT | NOT NULL | Service release version |
| <code>staged_at</code> | TIMESTAMPTZ | NOT NULL DEFAULT CURRENT_TIMESTAMP | Pipeline staging time |

The supplied source documents this complete schema. fileciteturn17file0L95-L109

## 7. Why Use a Surrogate Staging ID?

The staging table has:

~~~text
id = BIGINT identity primary key
~~~

while the business/event identity remains:

~~~text
event_id = UUID UNIQUE
~~~

This separates physical staging-row identity from source event identity.

The pipeline can therefore use the staging primary key internally while still enforcing event-level uniqueness through <code>event_id</code>.

## 8. Why No Foreign Key to `public.user`?

The staging table intentionally does not maintain a foreign key to the operational user table.

The source gives the reason: analytics/staging data should not block operational user deletion or archival operations. fileciteturn17file0L43-L47

The principle is:

~~~text
Operational user lifecycle
          |
          X
          |
should not be blocked by
          |
          v
Pipeline staging history
~~~

Staging should preserve the event needed for downstream processing without unnecessarily coupling its lifecycle to operational user records.

## 9. Step 3 — Extend `TelemetryEvent`

Stage 8 added the backend service version to the source table.

Stage 9 must carry that field into the Python pipeline.

Update:

~~~text
src/payment_telemetry/extractor.py
~~~

Add:

~~~text
app_version: str
~~~

to the <code>TelemetryEvent</code> dataclass.

The extractor query must also select <code>app_version</code>.

The source explicitly identifies both changes. fileciteturn17file0L21-L23

## 10. Why Update the Domain Model First?

The domain model is the contract between extraction and staging.

Without <code>app_version</code>:

~~~text
PostgreSQL source
      |
      v
extractor
      |
      v
TelemetryEvent
      X app_version lost
      |
      v
staging
~~~

With the updated model:

~~~text
source app_version
      |
      v
TelemetryEvent.app_version
      |
      v
staging.app_version
~~~

This preserves the metadata introduced in the earlier application and ingestion stages.

## 11. Step 4 — Update the Extraction Query

The extractor now needs to read the additional source column.

Conceptually:

~~~sql
SELECT
    event_id,
    user_id,
    event_name,
    event_version,
    occurred_at,
    route,
    properties,
    received_at,
    app_version
FROM public.frontend_telemetry_event
ORDER BY received_at ASC
LIMIT %s
~~~

The supplied source explicitly states that the extractor was updated to include <code>app_version</code>. fileciteturn17file0L21-L23

Update row-to-dataclass mapping at the same time.

Do not add the field to the dataclass but forget to map the database value.

## 12. Step 5 — Implement the Staging Writer

Create:

~~~text
src/payment_telemetry/staging.py
~~~

Implement:

~~~python
stage_events(conn, events: list[TelemetryEvent]) -> int
~~~

Its responsibility is narrow:

1. accept a batch of extracted events;
2. serialize their properties correctly;
3. insert the batch into the staging table;
4. ignore duplicate event IDs;
5. return the number of newly inserted rows.

The source explicitly defines this function and its behavior. fileciteturn17file0L31-L38

## 13. Step 6 — Handle Empty Input Early

If the pipeline receives an empty list:

~~~text
events = []
    |
    v
return 0
~~~

Do not execute an empty <code>executemany()</code> operation.

The supplied implementation explicitly returns <code>0</code> without executing SQL for empty input. fileciteturn17file0L33-L36

### Why?

This avoids:
- unnecessary database calls;
- ambiguous empty-batch SQL behavior;
- wasted cursor work;
- needless transaction activity.

An empty batch is a normal pipeline condition and should be handled explicitly.

## 14. Step 7 — Serialize JSONB With Psycopg

The <code>properties</code> field is stored as PostgreSQL JSONB.

Do not rely on arbitrary Python object adaptation.

Use Psycopg's native:

~~~python
Jsonb(...)
~~~

adapter for the event properties.

The source explicitly identifies native <code>psycopg.types.json.Jsonb</code> serialization as the staging implementation. fileciteturn17file0L33-L37

The data flow is:

~~~text
Python dict/scalar properties
          |
          v
psycopg Jsonb adapter
          |
          v
PostgreSQL JSONB
~~~

## 15. Why Native JSONB Adaptation?

Telemetry properties are structured data.

The database column is:

~~~text
properties JSONB NOT NULL
~~~

The adapter ensures the Python representation is serialized into valid PostgreSQL JSONB rather than being treated as an arbitrary string representation.

The source explicitly records this as a design decision. fileciteturn17file0L45-L48

## 16. Step 8 — Use Bulk Execution

Do not issue one database statement per event:

~~~text
for event in events:
    INSERT event
~~~

Instead use Psycopg's batch execution:

~~~text
cursor.executemany(...)
~~~

The supplied implementation explicitly uses <code>cursor.executemany()</code>. fileciteturn17file0L33-L37

This reduces Python/database round trips and is appropriate for the batch-oriented architecture.

## 17. Step 9 — Make Staging Idempotent

The staging table enforces:

~~~text
UNIQUE(event_id)
~~~

and the insert uses:

~~~sql
ON CONFLICT (event_id) DO NOTHING
~~~

The complete behavior is:

~~~text
First staging
    |
    v
event_id X inserted

Same batch staged again
    |
    v
event_id X already exists
    |
    v
DO NOTHING
    |
    v
no duplicate
~~~

The source explicitly defines idempotency through the unique event ID and conflict clause. fileciteturn17file0L45-L48

## 18. Why Idempotency Matters in Data Pipelines

Batch pipelines are commonly retried.

Failures can occur after some rows are written but before the batch is considered complete.

Without idempotency:

~~~text
Batch 1
  |
  +--> event A inserted
  +--> event B inserted
  X failure

Retry
  |
  +--> event A duplicate
  +--> event B duplicate
~~~

With <code>ON CONFLICT DO NOTHING</code>, the retry does not create duplicate staging rows.

This makes the staging boundary safe for repeated processing of historical batches.

## 19. Step 10 — Return the Inserted Row Count

The staging writer returns:

~~~text
cursor.rowcount
~~~

This gives the caller a simple operational result:

~~~text
input events = 1000
newly inserted = 973
duplicates = 27
~~~

The source explicitly states that <code>stage_events()</code> returns the inserted row count through <code>cursor.rowcount</code>. fileciteturn17file0L33-L37

This becomes useful for later pipeline metrics and audit reporting.

## 20. Architecture and Data Flow

~~~text
public.frontend_telemetry_event
            |
            v
     extract_events(conn)
            |
            v
     list[TelemetryEvent]
            |
            | includes app_version
            v
       stage_events(conn)
            |
            +--> Jsonb(properties)
            |
            +--> executemany()
            |
            +--> ON CONFLICT(event_id) DO NOTHING
            |
            v
public.telemetry_event_staging
            |
            +--> Stage 10 validation
            +--> Stage 11 orchestration
            +--> Stage 13 quarantine
~~~

The supplied source documents this overall pipeline flow. fileciteturn17file0L52-L70

## 21. Transaction Safety

The initial Stage 9 implementation calls <code>conn.commit()</code> from the staging writer.

This is important to understand as a historical design point rather than blindly copying it into later orchestration code.

The source explicitly notes that commit ownership changes in Stage 13: the batch orchestrator becomes the owner of commits so multi-table operations can remain atomic. fileciteturn17file0L75-L77

Therefore the Stage 9 lesson is:

~~~text
Stage 9
stage_events()
    |
    +--> writes
    +--> commit

Later orchestration
    |
    v
orchestrator owns commit boundary
~~~

Do not silently attribute the later transaction model to this original Stage 9 implementation.

## 22. Idempotency Model

Staging has two identities:

~~~text
id
 |
 +--> physical staging-row identity

event_id
 |
 +--> source/business event identity
~~~

The important uniqueness constraint is on <code>event_id</code>.

The source explicitly identifies the unique constraint and conflict clause as the mechanism that guarantees re-staging does not create duplicates. fileciteturn17file0L81-L84

## 23. Integration With the Existing Pipeline

### Upstream

Stage 9 consumes:

~~~text
TelemetryEvent instances
~~~

produced by the Stage 6 extractor.

### Downstream

The staging layer feeds later pipeline components:

~~~text
telemetry_event_staging
        |
        +--> validation.py (Stage 10)
        +--> pipeline.py / orchestration
        +--> quarantine (Stage 13)
~~~

The source explicitly identifies these upstream and downstream boundaries. fileciteturn17file0L88-L91

## 24. Database Schema

The staging table is:

~~~text
public.telemetry_event_staging
~~~

Key constraints:

~~~text
id            -> identity primary key
event_id      -> NOT NULL UNIQUE
user_id       -> NOT NULL
event_name    -> NOT NULL
event_version -> NOT NULL
occurred_at   -> NOT NULL
route         -> NOT NULL
properties    -> JSONB NOT NULL
received_at   -> NOT NULL
app_version   -> NOT NULL
staged_at     -> NOT NULL DEFAULT CURRENT_TIMESTAMP
~~~

The complete source schema is documented in the supplied implementation record. fileciteturn17file0L95-L109

## 25. Tests

Stage 9 ends with eight passing tests:

~~~text
8 passed in 0.08s
~~~

The staging-specific tests are:

~~~text
test_stage_events_returns_zero_for_empty_input
test_stage_events_inserts_events_and_commits
test_stage_events_is_idempotent_by_event_id
~~~

The source explicitly records these tests and the full passing suite. fileciteturn17file0L114-L123

### Test 1 — Empty Input

Verify:

~~~text
stage_events(conn, [])
        |
        v
0
~~~

and verify that no SQL is executed.

### Test 2 — Bulk Insertion

Verify:
- event parameters are constructed correctly;
- JSON properties are adapted as JSONB;
- batch execution occurs;
- commit behavior matches the Stage 9 implementation;
- inserted row count is returned.

### Test 3 — Idempotency

Verify the SQL contains:

~~~sql
ON CONFLICT (event_id) DO NOTHING
~~~

and that the duplicate event does not produce another staging row.

## 26. PostgreSQL Validation

Validate the staging migration against PostgreSQL 16.

The supplied source records successful validation of the migration syntax and data types against PostgreSQL 16 standards. fileciteturn17file0L127-L129

At minimum verify:
- table creation succeeds;
- identity primary key is created;
- <code>event_id</code> uniqueness exists;
- JSONB properties are accepted;
- <code>staged_at</code> receives its default;
- duplicate event IDs are handled by the conflict clause.

## 27. Privacy and Data Protection

Stage 9 does not introduce a new source of sensitive data.

The staging table receives events that have already passed through:

~~~text
Stage 1 client sanitization
        |
        v
Stage 4 backend ingestion
        |
        v
Stage 9 staging
~~~

The source explicitly states that staging contains already-sanitized events and that plaintext payment secrets or unredacted credentials do not enter the staging layer. fileciteturn17file0L133-L135

Do not treat staging as a place where privacy rules can be relaxed.

## 28. Common Implementation Mistakes

### Mistake 1 — Querying the operational table forever

Staging exists to establish pipeline ownership and downstream isolation.

### Mistake 2 — Forgetting `app_version`

Stage 8 added backend version metadata to the source table.

Stage 9 must carry it through the domain model and into staging.

### Mistake 3 — Inserting one event at a time

That defeats the batch-processing design.

Use <code>executemany()</code>.

### Mistake 4 — Serializing JSON manually

Do not build ad-hoc JSON strings.

Use Psycopg's <code>Jsonb</code> adapter.

### Mistake 5 — No unique event constraint

Without a uniqueness constraint, application-level duplicate checks can race.

Let PostgreSQL enforce event-level uniqueness.

### Mistake 6 — Forgetting `ON CONFLICT`

Retries are normal in batch systems.

Make repeated staging safe.

### Mistake 7 — Running SQL for an empty batch

Return <code>0</code> immediately.

### Mistake 8 — Adding validation to the staging writer

Stage 9 should not absorb Stage 10's validation responsibilities.

Keep extraction, staging, validation, and quarantine as explicit pipeline boundaries.

### Mistake 9 — Applying later transaction ownership retroactively

The supplied Stage 9 implementation commits inside <code>staging.py</code>, while later Stage 13 moves commit ownership to the orchestrator.

Document the version of the architecture you are implementing rather than mixing stages.

## 29. Independent Implementation Exercise

Build a pipeline-owned staging layer for another event-processing system.

Requirements:

1. Create a dedicated staging table.
2. Give the table its own surrogate primary key.
3. Keep the source event ID unique.
4. Carry all required source metadata into the staging model.
5. Include release/version metadata.
6. Use JSONB for structured properties.
7. Use Psycopg's JSONB adapter.
8. Implement a bulk writer.
9. Return immediately for empty input.
10. Use batch execution rather than one insert per event.
11. Use <code>ON CONFLICT (event_id) DO NOTHING</code>.
12. Return the number of newly inserted rows.
13. Keep staging independent of the operational user foreign-key lifecycle.
14. Add tests for empty input, insertion, and duplicate handling.
15. Validate the schema against the target PostgreSQL version.

Then run the same batch twice and verify that the second execution does not increase the number of staging rows.

## 30. Validation Checklist

### Schema
- [ ] <code>public.telemetry_event_staging</code> exists.
- [ ] Identity surrogate primary key exists.
- [ ] <code>event_id</code> is unique.
- [ ] Required event fields are non-null.
- [ ] <code>properties</code> is JSONB.
- [ ] <code>app_version</code> is present.
- [ ] <code>staged_at</code> has a timestamp default.
- [ ] No unnecessary foreign key to <code>public.user</code> exists.

### Extractor
- [ ] <code>TelemetryEvent</code> contains <code>app_version</code>.
- [ ] Source query selects <code>app_version</code>.
- [ ] Row mapping includes <code>app_version</code>.

### Staging writer
- [ ] Empty input returns <code>0</code>.
- [ ] Empty input performs no SQL.
- [ ] <code>Jsonb</code> is used for properties.
- [ ] <code>executemany()</code> is used.
- [ ] <code>ON CONFLICT (event_id) DO NOTHING</code> is used.
- [ ] Inserted row count is returned.

### Idempotency
- [ ] First staging inserts the event.
- [ ] Re-staging the same event creates no duplicate.
- [ ] Unique event identity is enforced by PostgreSQL.

### Tests
- [ ] Staging test suite passes.
- [ ] Extractor mapping test includes <code>app_version</code>.
- [ ] Full regression suite passes.
- [ ] PostgreSQL migration is validated.

### Privacy
- [ ] Only sanitized telemetry reaches staging.
- [ ] No plaintext payment secrets are introduced.
- [ ] No unredacted credentials are introduced.

## 31. Scope Boundaries

### Implemented in Stage 9
- staging table schema;
- pipeline-owned storage isolation;
- <code>TelemetryEvent.app_version</code> propagation;
- bulk staging writer;
- JSONB serialization;
- event-ID idempotency;
- staging tests.

### Deferred to Stage 10
- in-pipeline event validation.

### Deferred to Stage 11
- keyset checkpointing;
- end-to-end orchestration.

### Deferred to Stage 13
- quarantine isolation for invalid events;
- centralized commit ownership for multi-table atomicity.

The supplied source explicitly defines the first three boundaries and documents the later commit-ownership change. fileciteturn17file0L139-L143 fileciteturn17file0L75-L77

## 32. What You Should Understand Before the Next Recipe

Stage 9 creates the first persistent storage boundary owned by the data pipeline:

~~~text
Operational source
      |
      v
Extraction
      |
      v
TelemetryEvent
      |
      v
Pipeline staging
      |
      +--> future validation
      +--> future orchestration
      +--> future quarantine
~~~

The central lessons are:
- staging is a workload and ownership boundary, not merely another copy of the table;
- source event identity must remain unique across retries;
- batch writes should be optimized for database round trips;
- structured JSON should use the database driver's native adapter;
- metadata added upstream must be carried through the domain model;
- empty batches are normal and should be cheap;
- validation and quarantine belong to later stages;
- transaction ownership can evolve as orchestration becomes more complex.

The supplied Stage 9 implementation records eight passing tests and a verified PostgreSQL schema. fileciteturn17file0L114-L129

## Source Traceability

This recipe is derived from the supplied Stage 9 implementation record:
- Source commit: <code>1b78097</code>
- Subject: <code>feat: add telemetry staging layer</code>
- Repository: <code>devops</code>
- Overview and technical objective: fileciteturn17file0L3-L15
- Implementation sequence: fileciteturn17file0L19-L25
- Technical changes: fileciteturn17file0L29-L38
- Design decisions: fileciteturn17file0L43-L48
- Architecture and data flow: fileciteturn17file0L52-L70
- Transaction safety and idempotency: fileciteturn17file0L75-L84
- Integration and schema: fileciteturn17file0L88-L109
- Tests and PostgreSQL validation: fileciteturn17file0L114-L129
- Privacy and scope boundaries: fileciteturn17file0L133-L143