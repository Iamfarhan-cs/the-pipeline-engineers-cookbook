# Recipe 13 — Persistent Telemetry Quarantine and Atomic Batch Processing

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 13  
> **Source implementation commit:** <code>9211aa3</code> — <code>feat: add persistent telemetry quarantine</code>  
> **Source repository:** <code>devops</code>

## 1. What This Recipe Teaches

Stage 13 introduces durable quarantine storage and moves transaction ownership to the batch orchestrator.

By completing this recipe, you should understand how to:

- persist invalid events with their validation diagnostics;
- keep valid events moving while invalid events are isolated;
- store validation errors as JSONB;
- make quarantine insertion idempotent;
- wrap staging, quarantine, and checkpoint advancement in one PostgreSQL transaction;
- remove commit authority from helper modules;
- test mixed valid/invalid batches and rollback behavior;
- separate quarantine storage from later remediation lifecycle work.

The source explicitly identifies the quarantine table, quarantine writer, atomic transaction wrapper, helper commit removal, and 35-test regression. fileciteturn22file0L3-L15

## 2. The Problem

Stage 11 blocked an entire batch when any event failed validation:

~~~text
batch contains invalid event
          |
          v
      stop batch
          |
          +--> no staging
          +--> no checkpoint advancement
~~~

That protects against silently skipping invalid data, but one bad event can stop the entire stream.

Stage 13 changes the behavior:

~~~text
extract
   |
validate
   |
   +--> valid events   -> staging
   |
   +--> invalid events -> quarantine
                              |
                              v
                         checkpoint
~~~

Invalid events are now isolated into durable quarantine storage while valid events continue. fileciteturn22file0L8-L15

## 3. The Atomic Batch Boundary

The complete database mutation phase is wrapped in:

~~~python
with conn.transaction():
    ...
~~~

The intended unit is:

~~~text
BEGIN
  |
  +--> stage valid events
  +--> quarantine invalid events
  +--> save checkpoint
  |
COMMIT
~~~

If an unhandled database or system exception occurs during the transactional work, the batch rolls back. fileciteturn22file0L48-L52 fileciteturn22file0L81-L83

This creates the key invariant:

> Staging, quarantine, and checkpoint progress are committed together.

## 4. Step 1 — Create the Quarantine Table

Create:

~~~text
migrations/003_create_telemetry_event_quarantine.sql
~~~

The table is:

~~~text
public.telemetry_event_quarantine
~~~

The source defines it using the event fields represented by staging plus:

~~~text
errors JSONB NOT NULL
quarantined_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
~~~

The source explicitly records this schema change. fileciteturn22file0L30-L37

### Schema

| Column | Type | Constraint | Purpose |
|---|---|---|---|
| <code>id</code> | BIGINT | GENERATED ALWAYS AS IDENTITY PRIMARY KEY | Surrogate identifier |
| <code>event_id</code> | UUID | NOT NULL UNIQUE | Event identifier |
| <code>user_id</code> | UUID | NOT NULL | Associated user UUID |
| <code>event_name</code> | TEXT | NOT NULL | Telemetry event name |
| <code>event_version</code> | INTEGER | NOT NULL | Event schema version |
| <code>occurred_at</code> | TIMESTAMPTZ | NOT NULL | Event occurrence time |
| <code>route</code> | TEXT | NOT NULL | Frontend route |
| <code>properties</code> | JSONB | NOT NULL | Event properties |
| <code>received_at</code> | TIMESTAMPTZ | NOT NULL | Ingestion timestamp |
| <code>app_version</code> | TEXT | NOT NULL | Service release version |
| <code>errors</code> | JSONB | NOT NULL | Validation error strings |
| <code>quarantined_at</code> | TIMESTAMPTZ | NOT NULL DEFAULT CURRENT_TIMESTAMP | Quarantine insertion time |

These columns and constraints are documented in the supplied source. fileciteturn22file0L102-L116

## 5. Why Preserve the Entire Event?

Quarantine is durable diagnostic storage, not merely an error log.

The stored record keeps:

~~~text
original event
      +
validation errors
      +
quarantine timestamp
~~~

This gives later data-quality operations both the rejected event and the reason it was rejected.

## 6. Step 2 — Build the Quarantine Writer

Create:

~~~text
src/payment_telemetry/quarantine.py
~~~

Implement:

~~~python
quarantine_events(conn, invalid_events)
~~~

The documented behavior is:

1. Return <code>0</code> for an empty list without executing SQL.
2. Convert each error list with <code>psycopg.types.json.Jsonb(list(item.errors))</code>.
3. Insert the batch with <code>cursor.executemany()</code>.
4. Use <code>ON CONFLICT (event_id) DO NOTHING</code>.
5. Return the inserted row count.

The source explicitly documents these behaviors. fileciteturn22file0L33-L37

## 7. Empty Input

For:

~~~text
invalid_events = []
~~~

return:

~~~text
0
~~~

without executing SQL.

This avoids unnecessary database work when the batch is completely valid.

The source includes a dedicated test for this behavior. fileciteturn22file0L126-L128

## 8. Structured Error Preservation

Validation errors are stored as a JSONB list.

Conceptually:

~~~text
[
  "validation error 1",
  "validation error 2"
]
~~~

The source explicitly uses the Psycopg JSONB adapter and identifies structured error preservation as a design decision. JSONB enables later querying, indexing, and operational triage. fileciteturn22file0L33-L36 fileciteturn22file0L48-L52

## 9. Quarantine Idempotency

The quarantine table enforces unique <code>event_id</code>.

The writer uses:

~~~sql
ON CONFLICT (event_id) DO NOTHING
~~~

Therefore:

~~~text
first attempt  -> insert
retry          -> existing event ignored
~~~

The source explicitly defines this as the quarantine idempotency mechanism. fileciteturn22file0L87-L89

## 10. Step 3 — Refactor process_batch()

Update:

~~~text
src/payment_telemetry/pipeline.py
~~~

Wrap batch processing in:

~~~python
with conn.transaction():
    ...
~~~

The source explicitly records this orchestration change. fileciteturn22file0L23-L24 fileciteturn22file0L38-L42

## 11. New Execution Sequence

The Stage 13 architecture is:

~~~text
process_batch()
   |
   v
BEGIN transaction
   |
   +--> extract_events(checkpoint)
   |
   +--> validate_events(events)
   |        +--> valid_events
   |        +--> invalid_events
   |
   +--> stage_events(valid_events)
   |
   +--> quarantine_events(invalid_events)
   |
   +--> save_checkpoint(last_event)
   |
   v
COMMIT
~~~

This sequence is documented directly in the source. fileciteturn22file0L56-L76

## 12. Step 4 — Stage Valid Events

The orchestrator sends:

~~~text
validation.valid_events
~~~

to the existing staging helper:

~~~python
stage_events(conn, validation.valid_events)
~~~

The source explicitly records this operation. fileciteturn22file0L38-L42

## 13. Step 5 — Quarantine Invalid Events

The same validation result provides:

~~~text
validation.invalid_events
~~~

The orchestrator sends those records to:

~~~python
quarantine_events(conn, validation.invalid_events)
~~~

The source explicitly records this operation. fileciteturn22file0L38-L42

A mixed batch can now behave like:

~~~text
10 extracted
   |
   +--> 8 valid   -> staging
   |
   +--> 2 invalid -> quarantine
   |
   +--> checkpoint advances
~~~

This is the fundamental continuous-ingestion change.

## 14. Step 6 — Advance the Checkpoint

After valid events are staged and invalid events are quarantined, the checkpoint is saved using the last extracted event:

~~~text
last_event.received_at
last_event.event_id
~~~

The source explicitly records that <code>process_batch()</code> saves the checkpoint at the last extracted event. fileciteturn22file0L38-L43

Because checkpoint persistence is inside the same transaction, the cursor cannot commit independently of the batch's data-processing work.

## 15. Step 7 — Remove Helper-Level Commits

Remove internal <code>conn.commit()</code> calls from:

~~~text
src/payment_telemetry/staging.py
src/payment_telemetry/checkpoint.py
~~~

The source explicitly documents this helper commit decoupling. fileciteturn22file0L24-L24 fileciteturn22file0L43-L43

The new responsibility boundary is:

~~~text
helper
  -> perform database operation

process_batch()
  -> begin transaction
  -> call helpers
  -> commit or rollback
~~~

This is required for atomic multi-step processing.

## 16. Failure Scenario — Staging Fails

Suppose:

~~~text
extract      -> success
validate     -> success
stage        -> failure
~~~

The transaction rolls back.

Therefore the batch's transactional changes are not partially committed.

The source explicitly states that staging, quarantine, and checkpoint work rolls back when an unhandled exception occurs. fileciteturn22file0L81-L83

## 17. Failure Scenario — Quarantine Fails

Suppose valid staging succeeds but quarantine fails:

~~~text
stage valid events
        |
        v
quarantine failure
        |
        v
ROLLBACK
~~~

The valid staging operation is not committed independently because both operations share the same transaction.

## 18. Failure Scenario — Checkpoint Save Fails

Suppose:

~~~text
stage       -> success
quarantine  -> success
checkpoint  -> failure
~~~

The transaction rolls back.

This prevents the checkpoint from moving ahead of the committed data state.

This is the key reason transaction ownership belongs to <code>process_batch()</code> rather than individual helpers.

## 19. Continuous Ingestion

Stage 13 changes the failure model from:

~~~text
invalid event -> entire batch blocked
~~~

to:

~~~text
invalid event -> quarantine
valid event   -> staging
both          -> one transaction
checkpoint    -> continue stream
~~~

The source explicitly describes this as continuous ingestion. fileciteturn22file0L48-L52

## 20. Architecture and Data Flow

~~~text
process_batch()
   |
   v
BEGIN (conn.transaction())
   |
   +--> extract_events(checkpoint)
   |
   +--> validate_events(events)
   |        |
   |        +--> valid_events
   |        +--> invalid_events
   |
   +--> stage_events(valid_events)
   |        |
   |        v
   |   telemetry_event_staging
   |
   +--> quarantine_events(invalid_events)
   |        |
   |        v
   |   telemetry_event_quarantine
   |
   +--> save_checkpoint(last_event)
            |
            v
    telemetry_pipeline_checkpoint
   |
   v
COMMIT
~~~

This is the complete transaction flow documented by the source. fileciteturn22file0L56-L76

## 21. Integration With Previous Stages

Stage 13 consumes <code>invalid_events</code> from Stage 10 and integrates directly with the staging and checkpoint components. fileciteturn22file0L93-L96

The resulting architecture is:

~~~text
Stage 10 validation
        |
        +--> valid_events ----> Stage 9 staging
        |
        +--> invalid_events -> Stage 13 quarantine
                                      |
Stage 11 checkpoint <-----------------+
~~~

Stage 13 therefore extends the existing pipeline rather than replacing its validation or staging logic.

## 22. Database Schema

Stage 13 introduces:

~~~text
public.telemetry_event_quarantine
~~~

The table stores:

~~~text
event identity
event metadata
event properties
validation errors
quarantine timestamp
~~~

The complete schema is documented in Section 4.

No remediation state machine or resolution metadata is introduced here.

Those concerns are explicitly deferred to later stages. fileciteturn22file0L148-L152

## 23. Tests

The Stage 13 source reports:

~~~text
35 passed in 0.17s
~~~

New quarantine tests include:

~~~text
test_quarantine_events_returns_zero_for_empty_input
test_quarantine_events_inserts_invalid_event_and_preserves_errors
test_quarantine_events_is_idempotent_by_event_id
~~~

The pipeline test verifies:

~~~text
test_process_batch_stages_valid_and_quarantines_invalid_events
~~~

This verifies simultaneous staging, quarantine, and checkpoint advancement in one transaction. fileciteturn22file0L120-L132

## 24. Test Scenario — Empty Quarantine Input

Input:

~~~text
invalid_events = []
~~~

Expected:

~~~text
return 0
no SQL execution
~~~

The source explicitly tests this behavior. fileciteturn22file0L126-L128

## 25. Test Scenario — Error Preservation

Create an invalid event with multiple validation errors.

Verify that the quarantine row preserves:

~~~text
event_id
event fields
errors JSONB array
~~~

The source explicitly tests correct JSONB error formatting. fileciteturn22file0L128-L128

## 26. Test Scenario — Quarantine Idempotency

Insert the same invalid event twice.

Expected:

~~~text
first insert  -> one quarantine row
second insert -> no additional row
~~~

because of:

~~~sql
ON CONFLICT (event_id) DO NOTHING
~~~

The source explicitly verifies this behavior. fileciteturn22file0L129-L129

## 27. Test Scenario — Mixed Batch

Create a batch containing valid and invalid events.

Expected:

~~~text
valid events   -> staging
invalid events -> quarantine
checkpoint     -> advances
~~~

and all three operations occur inside one transaction.

The source explicitly identifies this pipeline test. fileciteturn22file0L131-L132

## 28. PostgreSQL Validation

The source records verification of:

~~~text
migrations/003_create_telemetry_event_quarantine.sql
~~~

against PostgreSQL 16, including JSONB query capabilities. fileciteturn22file0L136-L138

Verify at minimum:

- quarantine table creation;
- unique event ID constraint;
- JSONB event properties;
- JSONB validation errors;
- default quarantine timestamp;
- insertion and conflict behavior.

## 29. Privacy and Data Protection

Quarantine preserves original event properties for debugging.

The source states that Stage 1 client-side sanitization has already sanitized sensitive values, while the quarantine <code>errors</code> field contains validation messages only. fileciteturn22file0L142-L144

The intended boundary is:

~~~text
sanitized event + diagnostics
~~~

not unsanitized sensitive input.

## 30. Common Implementation Mistakes

### Mistake 1 — Keep commits inside helper functions

Internal commits break the batch-level atomic transaction.

### Mistake 2 — Quarantine outside the transaction

Quarantine state can diverge from staging and checkpoint state.

### Mistake 3 — Continue blocking the batch on invalid events

Stage 13 exists specifically to isolate invalid events and keep valid ingestion moving.

### Mistake 4 — Drop invalid events after validation

Persist them with their diagnostic error list.

### Mistake 5 — Store errors as an opaque string

The documented design uses JSONB for the error list.

### Mistake 6 — Allow duplicate quarantine rows

Use unique <code>event_id</code> and <code>ON CONFLICT (event_id) DO NOTHING</code>.

### Mistake 7 — Advance the checkpoint outside the transaction

The cursor must commit together with the corresponding staging/quarantine work.

### Mistake 8 — Store unsanitized values

Quarantine relies on the Stage 1 sanitization boundary.

### Mistake 9 — Add remediation lifecycle fields now

Remediation state, retry counters, and resolution metadata are explicitly deferred.

## 31. Independent Implementation Exercise

Build a durable quarantine system for another event pipeline.

Requirements:

1. Create a quarantine table containing original event fields.
2. Add a structured JSON/JSONB error field.
3. Add a quarantine timestamp.
4. Make the event identifier unique.
5. Implement a batch quarantine writer.
6. Return zero for empty input without SQL.
7. Use bulk insertion.
8. Make duplicate quarantine writes idempotent.
9. Remove commits from lower-level database helpers.
10. Make the batch orchestrator own the transaction.
11. Stage valid events.
12. Quarantine invalid events.
13. Advance the checkpoint inside the same transaction.
14. Roll back the batch if a transactional operation fails.
15. Test mixed valid/invalid batches.
16. Test retry behavior.
17. Verify the migration on PostgreSQL.

Then create:

~~~text
3 valid events
2 invalid events
~~~

Verify:

~~~text
staging rows    = 3
quarantine rows = 2
checkpoint      = last extracted event
all committed together
~~~

Finally, force a quarantine failure and verify that the staging rows and checkpoint update roll back with the same transaction.

## 32. Validation Checklist

### Quarantine storage

- [ ] <code>public.telemetry_event_quarantine</code> exists.
- [ ] Original event fields are preserved.
- [ ] <code>errors</code> is JSONB and NOT NULL.
- [ ] <code>quarantined_at</code> has a default.
- [ ] <code>event_id</code> is unique.

### Quarantine writer

- [ ] Empty input returns zero.
- [ ] Empty input performs no SQL.
- [ ] Validation errors are stored as JSONB.
- [ ] Bulk insertion is used.
- [ ] Duplicate event IDs are ignored.
- [ ] Inserted row count is returned.

### Transaction orchestration

- [ ] <code>process_batch()</code> owns the transaction.
- [ ] Valid events are staged inside the transaction.
- [ ] Invalid events are quarantined inside the transaction.
- [ ] Checkpoint advancement is inside the transaction.
- [ ] Helper modules do not call <code>conn.commit()</code>.
- [ ] Staging failure rolls back the batch.
- [ ] Quarantine failure rolls back the batch.
- [ ] Checkpoint failure rolls back the batch.

### Continuous ingestion

- [ ] Invalid events no longer block valid events.
- [ ] Valid events reach staging.
- [ ] Invalid events reach quarantine.
- [ ] Checkpoint advances after successful batch completion.

### Tests

- [ ] Empty quarantine test passes.
- [ ] Error preservation test passes.
- [ ] Quarantine idempotency test passes.
- [ ] Mixed staging/quarantine/checkpoint test passes.
- [ ] Full 35-test regression passes.

### Privacy

- [ ] Quarantine receives sanitized telemetry.
- [ ] Sensitive values remain sanitized.
- [ ] Error JSONB contains validation messages only.

## 33. Scope Boundaries

### Implemented in Stage 13

- persistent quarantine table;
- quarantine insertion helper;
- structured validation-error storage;
- event-ID idempotency;
- atomic batch transaction;
- removal of helper-level commits;
- mixed valid/invalid continuous ingestion;
- transaction and quarantine tests.

### Deferred to Stage 14

- per-batch data-quality accounting;
- boundary tracking.

### Deferred to Task 15 / Stage 17

- remediation state machine;
- retry counters;
- resolution lifecycle metadata.

The source explicitly defines these boundaries. fileciteturn22file0L148-L152

## 34. What You Should Understand Before the Next Recipe

Stage 13 changes the pipeline failure model:

~~~text
BEFORE
invalid event
     |
     v
batch blocked

AFTER
invalid event
     |
     v
persistent quarantine
     |
     v
batch continues
~~~

The transaction boundary guarantees:

~~~text
valid staging
+
invalid quarantine
+
checkpoint advancement
        |
        v
     COMMIT
~~~

The central lessons are:

- quarantine is durable data-quality infrastructure, not just logging;
- invalid events remain available for diagnosis;
- valid and invalid events can be processed in one batch;
- the orchestrator should own transaction boundaries when several helpers participate in one atomic operation;
- helper-level commits undermine multi-step atomicity;
- idempotent quarantine protects retries;
- checkpoint advancement is safe when it commits together with the corresponding data-processing work;
- remediation and lifecycle management are separate later-stage concerns.

The supplied Stage 13 implementation reports 35 passing tests and PostgreSQL 16 validation. fileciteturn22file0L120-L138

## Source Traceability

This recipe is derived from the supplied Stage 13 implementation record:

- Source commit: <code>9211aa3</code>
- Subject: <code>feat: add persistent telemetry quarantine</code>
- Repository: <code>devops</code>
- Overview and technical objective: fileciteturn22file0L3-L15
- Implementation sequence: fileciteturn22file0L19-L26
- Technical changes: fileciteturn22file0L30-L44
- Design decisions: fileciteturn22file0L48-L52
- Architecture and data flow: fileciteturn22file0L56-L76
- Transaction safety and idempotency: fileciteturn22file0L81-L89
- Integration and schema: fileciteturn22file0L93-L116
- Tests and PostgreSQL validation: fileciteturn22file0L120-L138
- Privacy and scope boundaries: fileciteturn22file0L142-L152
