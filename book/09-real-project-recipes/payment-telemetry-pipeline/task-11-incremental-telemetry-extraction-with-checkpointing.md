# Recipe 11 — Incremental Telemetry Extraction with Checkpointing

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 11  
> **Source implementation commit:** <code>52c6d53</code> — <code>feat: add incremental telemetry extraction with checkpointing</code>  
> **Source repository:** <code>devops</code>

## 1. What This Recipe Teaches

This recipe transforms the telemetry pipeline from repeated batch reads into an incremental consumer with durable progress tracking.

By completing it, you should understand how to:

- persist pipeline progress in PostgreSQL;
- represent progress as a composite cursor;
- use keyset pagination instead of OFFSET;
- make ordering deterministic when timestamps are identical;
- extract only events after the previous checkpoint;
- build a central <code>process_batch()</code> orchestrator;
- prevent checkpoint advancement when a batch is empty, invalid, or fails during staging;
- make checkpoint persistence idempotent with an upsert;
- connect extraction, validation, staging, and checkpointing into one flow;
- understand why the supporting composite index is deferred to Stage 12.

The Stage 11 implementation creates a checkpoint table, adds composite keyset extraction on <code>(received_at, event_id)</code>, and introduces the first end-to-end batch orchestrator. The source describes this as the transformation from batch processing into an incremental stream consumer. fileciteturn20file0L3-L15

## 2. The Problem

Before Stage 11, the pipeline can extract and stage events, but it has no durable record of where the last successful batch ended.

A naive incremental implementation might use OFFSET:

~~~sql
OFFSET %s
~~~

or timestamp-only filtering:

~~~sql
WHERE received_at > %s
~~~

Both approaches have weaknesses.

OFFSET requires the database to walk past previously processed rows as the dataset grows. Timestamp-only filtering can skip records when several events have the same timestamp.

The Stage 11 source explicitly identifies these problems and replaces them with composite keyset checkpointing. fileciteturn20file0L8-L15

## 3. The Core Idea — A Durable Cursor

The pipeline needs to remember:

~~~text
"What was the last event successfully processed?"
~~~

The checkpoint stores two values:

~~~text
last_received_at
last_event_id
~~~

Together they form the cursor:

~~~text
(last_received_at, last_event_id)
~~~

The ordered event stream therefore looks like:

~~~text
(received_at, event_id)

(10:00:01, A)
(10:00:02, B)
(10:00:02, C)
(10:00:03, D)
             ^
             |
         checkpoint
~~~

The next extraction starts strictly after that pair.

## 4. Target Architecture

~~~text
process_batch()
      |
      +--> get_checkpoint("payment_telemetry")
      |
      +--> extract_events(checkpoint, limit)
      |
      +--> validate_events(events)
      |
      +--> stage_events(valid_events)
      |
      +--> save_checkpoint(last_event_position)
      |
      v
PipelineResult
~~~

The source documents this exact orchestration flow. fileciteturn20file0L57-L76

## 5. Step 1 — Create the Checkpoint Table

Create:

~~~text
migrations/002_create_telemetry_pipeline_checkpoint.sql
~~~

The migration creates:

~~~text
public.telemetry_pipeline_checkpoint
~~~

Schema:

| Column | Type | Constraint | Purpose |
|---|---|---|---|
| <code>pipeline_name</code> | TEXT | PRIMARY KEY | Unique pipeline identifier |
| <code>last_received_at</code> | TIMESTAMPTZ | NOT NULL | Timestamp of last processed event |
| <code>last_event_id</code> | UUID | NOT NULL | UUID of last processed event |
| <code>updated_at</code> | TIMESTAMPTZ | NOT NULL DEFAULT CURRENT_TIMESTAMP | Last checkpoint update time |

The source explicitly defines this schema. fileciteturn20file0L30-L36 fileciteturn20file0L100-L108

## 6. Why pipeline_name Is the Primary Key

A checkpoint table may eventually track multiple pipelines.

For this project:

~~~text
payment_telemetry
~~~

is the pipeline identifier.

Using <code>pipeline_name</code> as the primary key gives each named pipeline exactly one current cursor.

This also enables an upsert instead of creating a new checkpoint row on every execution.

## 7. Step 2 — Build the Checkpoint State Manager

Create:

~~~text
src/payment_telemetry/checkpoint.py
~~~

Define:

~~~text
Checkpoint
get_checkpoint(conn, pipeline_name)
save_checkpoint(conn, checkpoint)
~~~

The source explicitly identifies these as the checkpoint state manager components. fileciteturn20file0L21-L25

### get_checkpoint()

If the pipeline has never run:

~~~text
get_checkpoint(...)
        |
        v
None
~~~

If progress exists, return the saved cursor.

The test suite explicitly verifies both missing and existing checkpoint behavior. fileciteturn20file0L118-L121

### save_checkpoint()

Persist the cursor using:

~~~sql
ON CONFLICT (pipeline_name) DO UPDATE
~~~

This makes checkpoint writes idempotent and guarantees one persistent row per pipeline. fileciteturn20file0L49-L50

## 8. Step 3 — Update the Extractor

Update:

~~~text
src/payment_telemetry/extractor.py
~~~

The extractor now accepts an optional checkpoint.

There are two query modes.

### First run — no checkpoint

Use:

~~~sql
ORDER BY received_at ASC, event_id ASC
LIMIT %s
~~~

This establishes the deterministic initial order.

### Incremental run — checkpoint exists

Use:

~~~sql
WHERE (received_at, event_id) > (%s, %s)
ORDER BY received_at ASC, event_id ASC
LIMIT %s
~~~

The parameters are the checkpoint timestamp, checkpoint event ID, and batch limit.

The source explicitly defines both query forms. fileciteturn20file0L34-L36

## 9. Why the Composite Cursor Is Necessary

Suppose the source contains:

~~~text
received_at          event_id
-------------------  --------
10:00:00.000         A
10:00:00.000         B
10:00:00.000         C
10:00:01.000         D
~~~

If the checkpoint is:

~~~text
(10:00:00.000, B)
~~~

then the next query:

~~~sql
WHERE (received_at, event_id) > (%s, %s)
~~~

returns C and D.

A timestamp-only condition would not provide a unique position when A, B, and C share the same timestamp.

The source explicitly identifies <code>event_id</code> as the tie-breaker that creates a deterministic total ordering. fileciteturn20file0L47-L50

## 10. Keyset Pagination vs OFFSET

OFFSET means:

~~~text
"Skip N rows."
~~~

Keyset pagination means:

~~~text
"Continue after this exact ordered position."
~~~

For a growing telemetry table, keyset pagination avoids repeatedly scanning increasingly large numbers of already-processed rows.

The Stage 11 source specifically identifies OFFSET as a performance problem and establishes composite keyset pagination as the solution. fileciteturn20file0L11-L15

The ordering and cursor must always use the same fields:

~~~text
(received_at, event_id)
~~~

## 11. Step 4 — Create the Pipeline Orchestrator

Create:

~~~text
src/payment_telemetry/pipeline.py
~~~

Define:

~~~text
PIPELINE_NAME = "payment_telemetry"
PipelineResult
process_batch()
~~~

The source identifies these as the Stage 11 orchestration components. fileciteturn20file0L24-L25

The orchestrator becomes the central entry point for processing one incremental batch.

## 12. process_batch() Execution Sequence

The documented Stage 11 sequence is:

1. Fetch the current checkpoint.
2. Extract the next batch.
3. Return without advancement if the batch is empty.
4. Validate the extracted events.
5. Block staging if any event is invalid.
6. Stage valid events.
7. Save the checkpoint at the last event position.
8. Return a <code>PipelineResult</code>.

The source explicitly documents this behavior. fileciteturn20file0L37-L42

The central invariant is:

~~~text
extract
  |
validate
  |
stage
  |
checkpoint
~~~

Never advance the cursor before successful staging.

## 13. Empty Batch Behavior

If extraction returns no events:

~~~text
events = []
~~~

return:

~~~text
events_processed = 0
checkpoint_advanced = False
~~~

Do not stage and do not modify the checkpoint.

The source explicitly defines and tests this behavior. fileciteturn20file0L40-L40 fileciteturn20file0L123-L123

An empty batch means there is no new event position to commit.

## 14. Validation Before Staging

The orchestrator passes the batch through:

~~~text
validate_events(events)
~~~

The result contains:

~~~text
valid_events
invalid_events
~~~

Stage 11 uses an initial all-or-nothing policy:

~~~text
any invalid event
       |
       v
do not stage the batch
       |
       v
do not advance checkpoint
~~~

This was intentionally used before persistent quarantine was introduced in Stage 13. fileciteturn20file0L40-L42 fileciteturn20file0L49-L51

## 15. Why All-or-Nothing at Stage 11?

Stage 10 can identify invalid events, but Stage 11 does not yet have persistent quarantine storage.

Therefore the initial policy avoids silently processing only part of a batch.

The behavior is:

~~~text
valid batch
    |
    +--> stage
    +--> advance checkpoint

invalid batch
    |
    +--> no staging
    +--> no checkpoint advancement
~~~

Stage 13 later introduces persistent quarantine and a different transaction model.

## 16. Step 5 — Stage the Valid Batch

If validation produces no invalid events:

~~~text
validate_events(events)
        |
        v
invalid_events = []
        |
        v
stage_events(valid_events)
~~~

Stage 11 reuses the Stage 9 staging layer.

It does not duplicate Stage 9 responsibilities such as:

- JSONB adaptation;
- bulk insertion;
- event-ID idempotency.

The orchestrator coordinates those capabilities.

## 17. Step 6 — Advance the Checkpoint

After successful staging, save the position of the final ordered event:

~~~text
last_event.received_at
last_event.event_id
~~~

If the batch is:

~~~text
A
B
C
D
~~~

then:

~~~text
checkpoint -> D
~~~

Because extraction orders the batch by:

~~~text
received_at ASC, event_id ASC
~~~

the final element is the greatest position in the processed batch.

The source explicitly requires checkpoint advancement to the last event's position. fileciteturn20file0L41-L42

## 18. Failure Scenario — Staging Fails

Suppose:

~~~text
extract -> success
validate -> success
stage -> failure
~~~

The checkpoint must remain unchanged.

Otherwise the next run could start after events that were never successfully staged.

The Stage 11 test suite explicitly verifies that staging failure does not advance the checkpoint. fileciteturn20file0L127-L127

This is a fundamental pipeline rule:

> Never persist progress beyond work that has not successfully completed.

## 19. Failure Scenario — Checkpoint Save Fails

Suppose:

~~~text
extract -> success
validate -> success
stage -> success
checkpoint save -> failure
~~~

The checkpoint error propagates.

The previous checkpoint remains available for the next run.

That can cause the same events to be extracted again, but Stage 9's event-ID uniqueness and conflict handling make repeated staging safe.

The source explicitly includes a test for checkpoint failure after staging. fileciteturn20file0L128-L128

## 20. Transaction Safety

Stage 11 uses parameterized SQL for extraction and checkpoint operations.

However, the source explicitly records that <code>stage_events</code> and <code>save_checkpoint</code> committed sequentially in this stage.

Conceptually:

~~~text
stage_events()
    |
    v
commit

save_checkpoint()
    |
    v
commit
~~~

This was later refactored in Stage 13 into an atomic transaction covering the complete batch. fileciteturn20file0L81-L83

Do not retroactively describe Stage 11 as having the Stage 13 transaction model.

## 21. Idempotency

Stage 11 has two important idempotency mechanisms.

### Checkpoint upsert

Checkpoint persistence uses:

~~~sql
ON CONFLICT (pipeline_name) DO UPDATE
~~~

so repeated saves update the same pipeline row. fileciteturn20file0L49-L50

### Keyset extraction

The next extraction starts strictly after:

~~~text
(last_received_at, last_event_id)
~~~

so successful forward progress does not repeatedly select the same cursor position.

The source explicitly identifies both behaviors. fileciteturn20file0L87-L90

## 22. Complete Data Flow

~~~text
                         +----------------------+
                         | telemetry checkpoint |
                         +----------+-----------+
                                    |
                                    v
                           get_checkpoint()
                                    |
                                    v
                           extract_events()
                                    |
                                    v
                           validate_events()
                                    |
                         +----------+----------+
                         |                     |
                       invalid                valid
                         |                     |
                         v                     v
                       stop              stage_events()
                                               |
                                               v
                                       save_checkpoint()
                                               |
                                               v
                                        PipelineResult
~~~

The source records the same central orchestration path. fileciteturn20file0L55-L76

## 23. Integration With Existing Stages

Stage 11 connects:

~~~text
Stage 6 -> extractor.py
Stage 9 -> staging.py
Stage 10 -> validation.py
~~~

through:

~~~text
Stage 11 -> pipeline.py
~~~

The source explicitly identifies these integrations. fileciteturn20file0L94-L96

The resulting pipeline is:

~~~text
checkpoint
    |
    v
extractor
    |
    v
validator
    |
    v
stager
    |
    v
checkpoint
~~~

## 24. Database Schema

Stage 11 introduces one table:

~~~text
public.telemetry_pipeline_checkpoint
~~~

| Column | Type | Constraint | Description |
|---|---|---|---|
| <code>pipeline_name</code> | TEXT | PRIMARY KEY | Unique pipeline identifier |
| <code>last_received_at</code> | TIMESTAMPTZ | NOT NULL | Timestamp of last processed event |
| <code>last_event_id</code> | UUID | NOT NULL | UUID of last processed event |
| <code>updated_at</code> | TIMESTAMPTZ | NOT NULL DEFAULT CURRENT_TIMESTAMP | Last checkpoint update time |

No telemetry payload is stored in this table. fileciteturn20file0L100-L108

## 25. Why the Checkpoint Table Is Small

The checkpoint is a cursor, not an event store.

It needs only:

~~~text
pipeline_name
last_received_at
last_event_id
updated_at
~~~

It does not need:

~~~text
event_name
route
properties
user_id
payment data
~~~

This keeps checkpoint state operational and minimal.

The source explicitly states that the table stores only operational cursor coordinates and excludes event properties, payloads, and user identifiers. fileciteturn20file0L139-L141

## 26. Tests

The Stage 11 implementation reports:

~~~text
32 passed in 0.16s
~~~

The new checkpoint and pipeline tests include:

~~~text
test_get_checkpoint_returns_none_when_checkpoint_does_not_exist
test_get_checkpoint_returns_saved_checkpoint
test_save_checkpoint_upserts_checkpoint_and_commits
test_extract_events_uses_checkpoint_for_incremental_extraction
test_process_batch_does_not_advance_checkpoint_when_no_events
test_process_batch_does_not_stage_or_advance_checkpoint_for_invalid_events
test_process_batch_stages_events_then_advances_checkpoint
test_process_batch_passes_existing_checkpoint_to_extractor
test_process_batch_does_not_advance_checkpoint_when_staging_fails
test_process_batch_propagates_checkpoint_failure_after_staging
test_process_batch_advances_checkpoint_to_last_ordered_event
~~~

These tests cover checkpoint state, incremental extraction, and orchestration failure paths. fileciteturn20file0L112-L129

## 27. Test Scenario — First Run

Start with:

~~~text
checkpoint = None
~~~

Expected:

~~~text
extract first ordered batch
        |
        v
validate
        |
        v
stage
        |
        v
save last event position
~~~

Verify that a checkpoint row is created.

## 28. Test Scenario — Incremental Run

Start with:

~~~text
checkpoint = (T1, ID1)
~~~

Expected extraction:

~~~sql
WHERE (received_at, event_id) > (%s, %s)
~~~

with:

~~~text
T1
ID1
~~~

Verify that the existing checkpoint is passed to the extractor. fileciteturn20file0L126-L126

## 29. Test Scenario — Empty Run

Provide a valid checkpoint but no new events.

Expected:

~~~text
events_processed = 0
checkpoint_advanced = False
~~~

Verify:

~~~text
no staging
no checkpoint advancement
~~~

## 30. Test Scenario — Invalid Batch

Provide a batch containing at least one invalid event.

Expected:

~~~text
staging not called
checkpoint not advanced
~~~

This is the Stage 11 all-or-nothing behavior. fileciteturn20file0L123-L125

## 31. Test Scenario — Successful Batch

Provide a fully valid batch.

Expected:

~~~text
validation succeeds
       |
       v
staging succeeds
       |
       v
checkpoint moves to last ordered event
~~~

Verify that the checkpoint matches the final event according to the composite ordering.

## 32. Test Scenario — Staging Failure

Make the staging operation fail.

Expected:

~~~text
checkpoint remains unchanged
~~~

This protects against skipped events. fileciteturn20file0L127-L127

## 33. Test Scenario — Checkpoint Failure

Make checkpoint persistence fail after successful staging.

Expected:

~~~text
checkpoint error propagates
~~~

The old checkpoint remains available for retry, while the staging layer's event-ID idempotency protects against duplicate staging. fileciteturn20file0L128-L128

## 34. PostgreSQL Validation

The source records validation of:

~~~text
migrations/002_create_telemetry_pipeline_checkpoint.sql
~~~

against PostgreSQL 16 ANSI standards. fileciteturn20file0L133-L135

At minimum verify:

- checkpoint table creation;
- primary key on <code>pipeline_name</code>;
- TIMESTAMPTZ checkpoint timestamp;
- UUID event ID;
- default <code>updated_at</code>;
- checkpoint upsert behavior.

The matching composite B-tree index is deliberately deferred to Stage 12.

## 35. Privacy and Data Protection

The checkpoint table contains only:

~~~text
pipeline_name
last_received_at
last_event_id
~~~

It does not contain:

~~~text
event properties
payloads
user identifiers
~~~

The source explicitly documents this privacy boundary. fileciteturn20file0L139-L141

The checkpoint should remain a minimal operational state table.

## 36. Common Implementation Mistakes

### Mistake 1 — Use OFFSET

Use keyset pagination for incremental processing.

### Mistake 2 — Use only the timestamp as the cursor

Use both:

~~~text
received_at + event_id
~~~

### Mistake 3 — Order differently from the cursor

The comparison and ordering must use the same field sequence.

### Mistake 4 — Advance before staging

The correct order is:

~~~text
extract
validate
stage
checkpoint
~~~

### Mistake 5 — Advance on an empty batch

There is no new event position to save.

### Mistake 6 — Stage invalid batches

Stage 11 uses all-or-nothing processing until Stage 13 introduces quarantine.

### Mistake 7 — Store event payloads in the checkpoint table

The checkpoint is a cursor, not a telemetry store.

### Mistake 8 — Create a new checkpoint row every run

Use <code>pipeline_name</code> as the primary key and upsert.

### Mistake 9 — Assume Stage 11 is already atomic

The source explicitly says complete batch transaction atomicity is deferred to Stage 13.

### Mistake 10 — Add the Stage 12 index prematurely

The matching composite B-tree index belongs to Stage 12.

## 37. Independent Implementation Exercise

Build incremental processing for another append-oriented event table.

Requirements:

1. Create a checkpoint table.
2. Give each pipeline one checkpoint row.
3. Store timestamp and unique event ID as the cursor.
4. Implement <code>get_checkpoint()</code>.
5. Implement idempotent <code>save_checkpoint()</code>.
6. Implement first-run extraction.
7. Implement composite keyset extraction.
8. Order by timestamp and event ID ascending.
9. Build a batch orchestrator.
10. Validate before staging.
11. Do not stage invalid batches.
12. Do not advance the checkpoint for empty batches.
13. Advance only after successful staging.
14. Preserve the old checkpoint when staging fails.
15. Propagate checkpoint-save failures.
16. Add tests for each state transition.
17. Validate the migration against PostgreSQL.

Then create several events with exactly the same timestamp.

Run the pipeline repeatedly and verify:

~~~text
no event is skipped
no event is re-extracted after successful checkpointing
checkpoint always identifies the last successfully processed ordered event
~~~

## 38. Validation Checklist

### Checkpoint

- [ ] <code>public.telemetry_pipeline_checkpoint</code> exists.
- [ ] <code>pipeline_name</code> is the primary key.
- [ ] <code>last_received_at</code> is stored.
- [ ] <code>last_event_id</code> is stored.
- [ ] <code>updated_at</code> has a default.
- [ ] One checkpoint row exists per pipeline.

### Checkpoint manager

- [ ] Missing checkpoint returns <code>None</code>.
- [ ] Existing checkpoint can be loaded.
- [ ] Checkpoint writes use an upsert.
- [ ] Repeated saves update the existing row.

### Extractor

- [ ] First run uses ascending <code>received_at, event_id</code>.
- [ ] Incremental run uses the composite cursor.
- [ ] Query uses <code>(received_at, event_id) &gt; (%s, %s)</code>.
- [ ] Ordering matches the cursor fields.
- [ ] Limit remains parameterized.

### Orchestrator

- [ ] Current checkpoint is loaded first.
- [ ] Checkpoint is passed to extraction.
- [ ] Empty batches do not stage.
- [ ] Empty batches do not advance the checkpoint.
- [ ] Validation occurs before staging.
- [ ] Invalid batches do not stage.
- [ ] Invalid batches do not advance the checkpoint.
- [ ] Successful staging precedes checkpoint advancement.
- [ ] Checkpoint points to the last ordered event.
- [ ] Staging failures do not advance the checkpoint.
- [ ] Checkpoint failures propagate.

### Tests

- [ ] Missing checkpoint test passes.
- [ ] Existing checkpoint test passes.
- [ ] Upsert test passes.
- [ ] Incremental extraction test passes.
- [ ] Empty batch test passes.
- [ ] Invalid batch test passes.
- [ ] Successful batch test passes.
- [ ] Existing checkpoint propagation test passes.
- [ ] Staging failure test passes.
- [ ] Checkpoint failure test passes.
- [ ] Last-event checkpoint test passes.
- [ ] Full regression suite passes.

### Privacy

- [ ] Checkpoint stores only cursor coordinates.
- [ ] Event payloads are not stored in the checkpoint table.
- [ ] User identifiers are not stored in the checkpoint table.

## 39. Scope Boundaries

### Implemented in Stage 11

- checkpoint table;
- checkpoint state manager;
- composite keyset extraction;
- first-run extraction;
- incremental extraction;
- central batch orchestrator;
- checkpoint advancement;
- checkpoint and pipeline tests.

### Deferred to Stage 12

- matching composite B-tree index in <code>svc</code>.

### Deferred to Stage 13

- persistent quarantine storage;
- atomic transaction wrapping for the complete batch.

The supplied Stage 11 implementation explicitly defines these boundaries. fileciteturn20file0L145-L149

## 40. What You Should Understand Before the Next Recipe

Stage 11 establishes durable incremental processing:

~~~text
                 checkpoint
                     |
                     v
                 extractor
                     |
                     v
                  validate
                     |
                     v
                   stage
                     |
                     v
             save new checkpoint
~~~

The central lessons are:

- incremental pipelines need durable progress state;
- timestamps alone are not a sufficient cursor when they are not unique;
- composite keyset pagination creates deterministic total ordering;
- the checkpoint must advance only after successful processing;
- empty batches do not advance progress;
- invalid batches are blocked under the Stage 11 all-or-nothing policy;
- checkpoint persistence should be idempotent;
- orchestration should coordinate existing components rather than duplicate them;
- transaction atomicity and persistent quarantine are intentionally left to later stages.

The supplied Stage 11 implementation reports 32 passing tests and PostgreSQL 16 migration validation. fileciteturn20file0L112-L135

## Source Traceability

This recipe is derived from the supplied Stage 11 implementation record:

- Source commit: <code>52c6d53</code>
- Subject: <code>feat: add incremental telemetry extraction with checkpointing</code>
- Repository: <code>devops</code>
- Overview and technical objective: sections 1–2 of the supplied source.
- Technical changes: section 3.
- Design decisions: section 4.
- Architecture and data flow: section 5.
- Transaction safety and idempotency: sections 6–7.
- Integration and schema: sections 8–9.
- Tests and PostgreSQL validation: sections 10–11.
- Privacy and scope boundaries: sections 12–13.
