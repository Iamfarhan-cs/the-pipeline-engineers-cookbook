# Chapter 71 — Payment Telemetry Pipeline

> Source note: This chapter is reconstructed from the saved Payment Telemetry Pipeline handovers and implementation documentation available for the project. It deliberately distinguishes verified implementation from roadmap status where detailed evidence was not available.

## Project Overview

The Payment Telemetry Pipeline is a production-oriented Data Engineering pipeline for Zolvat telemetry.

The pipeline processes frontend telemetry events from PostgreSQL and progressively adds:

- source investigation
- extraction
- staging
- validation
- persistent quarantine
- incremental processing
- checkpointing
- batch Data Quality accounting
- pipeline execution metadata
- replay and reprocessing

The project uses Python 3.13, PostgreSQL, psycopg, pytest, and Docker.

V1 priorities are:

1. Correctness
2. Durability
3. Transaction safety
4. Privacy
5. Clear architecture
6. Maintainability

The project intentionally avoids premature optimization for the possible future volume of millions of users or events per day.

---

# 1. Task Overview

The project was implemented incrementally.

Each task added one responsibility instead of attempting to build the complete pipeline at once.

The resulting processing model is:

~~~text
Frontend Telemetry
        |
        v
PostgreSQL Source
        |
        v
Incremental Extraction
        |
        v
Checkpoint
        |
        v
Validation
      /   \
     /     \
 Valid     Invalid
   |          |
   v          v
Staging   Quarantine
   \          /
    \        /
     v      v
 Batch Outcome / DQ Accounting
            |
            v
     Pipeline Run Metadata
            |
            v
   Replay / Reprocessing
~~~

The main engineering theme is controlled state transition. Data should not disappear silently, progress should not move ahead of durable work, and historical processing should not corrupt live incremental state.

---

# 2. Project Repository and Working Context

Repository:

~~~text
~/OneDrive/Desktop/Zolvat/devops/payment-telemetry-pipeline
~~~

Branch used for the main pipeline work:

~~~text
feature/payment-telemetry-pipeline
~~~

Remote:

~~~text
https://github.com/zolvat/devops.git
~~~

A pre-existing unrelated file was protected throughout the work:

~~~text
../docker-compose.dev.yml
~~~

It was deliberately kept outside the pipeline commits.

---

# 3. Technology Stack

| Area | Technology |
|---|---|
| Language | Python 3.13 |
| Database | PostgreSQL |
| Driver | psycopg |
| Tests | pytest |
| Containers | Docker |
| Source | PostgreSQL telemetry table |
| Event properties | JSONB |
| Wider observability stack | OpenTelemetry Collector, VictoriaMetrics, Grafana, Loki, Tempo |

The pipeline repository uses direct SQL migrations rather than an ORM.

---

# 4. Privacy Boundary

Telemetry is privacy-sensitive.

The project explicitly avoids storing:

- payment payloads
- form values
- uploaded documents
- passwords
- free text
- unnecessary PII

The telemetry contract contains operational fields such as:

~~~text
event_id
event_name
event_version
occurred_at
route
properties
~~~

Execution metadata is also operational metadata. It should not become an additional location for sensitive user payloads.

---

# 5. Roadmap Status

| Task | Name | Status |
|---:|---|---|
| 1 | Source Investigation / Frontend Telemetry Boundary | Complete |
| 2 | Acquisition Strategy | Complete |
| 3 | Raw Storage Architecture | Complete |
| 4 | MinIO / Raw Storage | Complete |
| 5 | PostgreSQL Acquisition Metadata | Complete |
| 6 | Acquisition Engine | Complete |
| 7 | Raw Artifact Validation | Complete |
| 8 | Quarantine / Staging Foundation | Complete |
| 9 | Parser / Validation Framework | Complete |
| 10 | Incremental Processing / Checkpoint | Complete |
| 11 | Persistent Telemetry Quarantine | Complete |
| 12 | Batch Outcome & DQ Accounting | Complete |
| 13 | Pipeline Run / Execution Metadata | Complete |
| 14 | Reprocessing & Replay | Complete |

## Evidence boundary for Tasks 2–7

The saved project handovers establish that Tasks 2–7 were completed.

However, the detailed implementation records for those individual tasks were not present in the retrieved material used to reconstruct this chapter.

Therefore this chapter does not invent:

- file names
- migration numbers
- commands
- test counts
- commit hashes
- implementation details

for Tasks 2–7.

Their completion status is recorded, but their detailed recipes should be reconstructed from their original task documentation if those records are later supplied.

---

# 6. Task 1 — Frontend Telemetry Boundary

## Objective

Task 1 established the first working frontend telemetry boundary.

The task created a telemetry abstraction and connected it to frontend navigation. It did not attempt to build the complete backend pipeline.

## Implementation

Created:

~~~text
src/telemetry/index.ts
~~~

Modified:

~~~text
src/router/index.ts
~~~

The telemetry contract contains:

~~~text
event_id
event_name
event_version
occurred_at
route
properties
~~~

Event IDs use crypto.randomUUID().

Event timestamps use new Date().toISOString().

The router integration uses router.afterEach and records:

~~~text
event_name = page_view
event_version = 1
route = to.path
~~~

The implementation intentionally uses to.path instead of the complete URL containing query parameters.

## Data Flow

~~~text
User navigation
      |
      v
Vue Router
      |
      v
router.afterEach()
      |
      v
trackEvent()
      |
      v
TelemetryEvent
      |
      v
Development console
~~~

## Design Decision

The first task stopped at event generation.

It did not implement:

- backend telemetry ingestion
- HTTP telemetry transport
- message broker integration
- browser OpenTelemetry SDK
- database persistence
- batching
- retry
- production telemetry delivery

This kept the first implementation focused on the event contract and frontend measurement boundary.

## Validation

The documented validation included:

~~~text
npm ci
npm run type-check
npm run build
git diff --check
~~~

The first npm installation encountered ECONNRESET and succeeded after retrying.

Browser verification confirmed the telemetry event.

The implementation was committed locally on feature/frontend-telemetry. It was not pushed to the remote at that stage.

---

# 7. Tasks 2–7 — Foundation Work

The authoritative project handover records the following completed foundation tasks:

### Task 2 — Acquisition Strategy

Completed.

### Task 3 — Raw Storage Architecture

Completed.

### Task 4 — MinIO / Raw Storage

Completed.

### Task 5 — PostgreSQL Acquisition Metadata

Completed.

### Task 6 — Acquisition Engine

Completed.

### Task 7 — Raw Artifact Validation

Completed.

The available source material does not provide sufficient task-specific implementation evidence for these tasks in the current reconstruction.

Therefore the chapter deliberately does not claim particular files, tables, commands, test results, or commits for them.

This is an intentional documentation boundary rather than an assumption that the work did not happen.

---

# 8. Task 8 — Telemetry Staging Layer

## Objective

Task 8 established a durable staging boundary between extraction and downstream processing.

The staging layer provides:

- replayability
- recoverability
- idempotency
- duplicate handling
- debugging
- source-to-stage reconciliation
- Data Quality isolation
- downstream decoupling
- intermediate persistence
- operational visibility

## Database Change

Migration:

~~~text
migrations/001_create_telemetry_event_staging.sql
~~~

Table:

~~~text
public.telemetry_event_staging
~~~

## Implementation

Created:

~~~text
src/payment_telemetry/staging.py
~~~

Updated extraction support in:

~~~text
src/payment_telemetry/extractor.py
~~~

Tests:

~~~text
tests/test_extractor.py
tests/test_staging.py
~~~

The staging operation uses:

~~~text
ON CONFLICT (event_id) DO NOTHING
~~~

and uses psycopg JSONB support for properties.

## Integration Verification

First controlled run:

~~~text
EXTRACTED: 1
STAGED: 1
~~~

The staging row was directly inspected in PostgreSQL.

The same event was processed again:

~~~text
EXTRACTED: 1
STAGED: 0
~~~

Database verification:

~~~text
staging_rows = 1
unique_events = 1
~~~

Synthetic test data was removed after validation.

## Tests

Task 8:

~~~text
8 passed
~~~

Commit:

~~~text
1b78097
feat: add telemetry staging layer
~~~

The commit was pushed.

---

# 9. Task 9 — Telemetry Validation and Data Quality

## Objective

Task 9 added application-level validation after extraction and before persistence.

Created:

~~~text
src/payment_telemetry/validation.py
~~~

Tests:

~~~text
tests/test_validation.py
~~~

## Validation Rules

The documented validation covers:

- event UUID validity
- user UUID validity
- event name presence
- event name length
- positive event version
- route presence
- route length
- properties being an object
- occurred_at validity
- received_at validity
- app_version presence
- multiple simultaneous validation failures
- valid and invalid batch separation
- multiple invalid events
- empty batch handling

The validator preserves multiple errors for the same event.

## Database Constraint Relationship

The source PostgreSQL table already enforces several application-level conditions, including:

- non-empty app_version
- valid event name length
- properties being a JSON object
- valid route length
- positive event version

This means some invalid states cannot naturally be inserted into the source database.

The application validation layer is still required because the pipeline contract should be explicit and testable.

## Tests

Validation tests:

~~~text
13 passed
~~~

Full regression:

~~~text
21 passed in 0.17s
~~~

Compilation:

~~~text
python -m compileall -q src tests
~~~

completed successfully.

Commit:

~~~text
3d6f45a
feat: add telemetry event validation
~~~

---

# 10. Task 10 — Incremental Extraction and Checkpointing

## Objective

Task 10 changed the pipeline from repeated extraction to durable incremental processing.

The pipeline needed a reliable answer to:

> Where should the next execution continue?

The selected cursor is:

~~~text
(received_at, event_id)
~~~

This gives deterministic ordering.

## Source Table

Verified source table:

~~~text
public.frontend_telemetry_event
~~~

Relevant fields:

~~~text
id UUID
event_id UUID UNIQUE
user_id UUID
event_name TEXT
event_version INTEGER
occurred_at TIMESTAMPTZ
route TEXT
properties JSONB
received_at TIMESTAMPTZ
app_version TEXT
~~~

The source cursor index is:

~~~text
frontend_telemetry_event_received_event_idx
~~~

on:

~~~text
(received_at, event_id)
~~~

## Checkpoint

Migration:

~~~text
migrations/002_create_telemetry_pipeline_checkpoint.sql
~~~

Table:

~~~text
public.telemetry_pipeline_checkpoint
~~~

Fields:

~~~text
pipeline_name
last_received_at
last_event_id
updated_at
~~~

Current pipeline name:

~~~text
payment_telemetry
~~~

## Incremental Query

The cursor query is:

~~~text
WHERE (received_at, event_id) > (%s, %s)
ORDER BY received_at, event_id
LIMIT %s
~~~

This provides deterministic incremental extraction.

## Checkpoint Module

File:

~~~text
src/payment_telemetry/checkpoint.py
~~~

Model:

~~~text
Checkpoint
~~~

Functions:

~~~text
get_checkpoint()
save_checkpoint()
~~~

The helper does not hide transaction ownership from the pipeline.

## Testing

Task 10 final regression:

~~~text
32 passed in 1.88s
~~~

Compilation passed.

Whitespace validation passed.

Integration validation demonstrated:

- initial extraction
- deterministic ordering
- batch boundaries
- incremental continuation
- checkpoint advancement
- no-new-data behavior
- no normal reprocessing after checkpoint completion

## Known Limitation

The strict cursor can miss a late-arriving or backfilled event.

That behavior was intentionally left for later work rather than silently adding a partial solution to Task 10.

Commit:

~~~text
52c6d53
feat: add incremental telemetry extraction with checkpointing
~~~

Backend cursor index commit:

~~~text
01218b951
feat: add telemetry cursor index
~~~

---

# 11. Task 11 — Persistent Telemetry Quarantine

## Objective

Task 11 made invalid telemetry durable.

The processing model became:

~~~text
Extract
   |
   v
Validate
  / \
 /   \
Valid Invalid
 |      |
 v      v
Stage Quarantine
   \    /
    \  /
     v
Checkpoint
~~~

The checkpoint may advance only after every extracted event has been durably handled.

## Database Change

Migration:

~~~text
migrations/003_create_telemetry_event_quarantine.sql
~~~

Table:

~~~text
public.telemetry_event_quarantine
~~~

Implementation:

~~~text
src/payment_telemetry/quarantine.py
~~~

The persistence operation uses event_id idempotency with:

~~~text
ON CONFLICT (event_id) DO NOTHING
~~~

The function does not commit internally.

## Idempotency Verification

A controlled invalid event was inserted.

The same event was inserted again with a different validation error.

The second insertion was ignored:

~~~text
duplicate_inserted = 0
~~~

The original validation errors remained.

This verified PostgreSQL-level event idempotency.

## Atomic Rollback Verification

A controlled source event was processed while checkpoint persistence was deliberately made to fail.

A separate connection then verified:

~~~text
staging_count = 0
quarantine_count = 0
~~~

The checkpoint remained unchanged.

This demonstrated that staging, quarantine, and checkpoint advancement are part of one transaction boundary.

## Git State

Commit:

~~~text
9211aa3
feat: add persistent telemetry quarantine
~~~

The commit contained:

~~~text
9 files changed
213 insertions
56 deletions
~~~

The commit was pushed successfully.

---

# 12. Task 12 — Batch Outcome and DQ Accounting

## Objective

Task 12 added durable batch-level accounting.

Task 11 answered whether individual events were valid or invalid.

Task 12 added the durable answer to:

> What happened to this batch?

## Database Change

Migration:

~~~text
004_create_telemetry_pipeline_batch.sql
~~~

Table:

~~~text
public.telemetry_pipeline_batch
~~~

Verified fields:

~~~text
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
pipeline_name TEXT NOT NULL
first_received_at TIMESTAMPTZ NOT NULL
first_event_id UUID NOT NULL
last_received_at TIMESTAMPTZ NOT NULL
last_event_id UUID NOT NULL
extracted_count INTEGER NOT NULL
valid_count INTEGER NOT NULL
invalid_count INTEGER NOT NULL
staged_count INTEGER NOT NULL
quarantined_count INTEGER NOT NULL
created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
~~~

Unique constraint:

~~~text
pipeline_name + last_received_at + last_event_id
~~~

Checks enforce:

~~~text
all counters >= 0
extracted_count = valid_count + invalid_count
staged_count <= valid_count
quarantined_count <= invalid_count
~~~

## Accounting Module

File:

~~~text
src/payment_telemetry/accounting.py
~~~

Dataclass:

~~~text
BatchOutcome
~~~

Persistence function:

~~~text
record_batch_outcome(conn, outcome)
~~~

The function does not commit.

The pipeline transaction owns commit and rollback.

The insert is idempotent through the batch uniqueness key.

## Transaction Order

The critical flow is:

~~~text
get checkpoint
    |
    v
extract
    |
    v
validate
    |
    v
stage valid
    |
    v
quarantine invalid
    |
    v
build BatchOutcome
    |
    v
record batch outcome
    |
    v
save checkpoint
    |
    v
COMMIT
~~~

The invariant is:

> Accounting must succeed before checkpoint advancement.

If accounting fails, the checkpoint must not advance.

If checkpoint persistence fails, the accounting record must roll back.

## Real PostgreSQL Verification

Successful execution produced:

~~~text
extracted_count=1
valid_count=1
invalid_count=0
staged_count=1
checkpoint_advanced=True
invalid_events=()
~~~

Durable accounting contained:

~~~text
extracted_count = 1
valid_count = 1
invalid_count = 0
staged_count = 1
quarantined_count = 0
~~~

A second execution produced zero new work.

Rollback testing showed:

~~~text
before: 0
inside: 1
after: 0
~~~

## Tests

Task 12 final full regression:

~~~text
38 passed in 0.30s
~~~

Pipeline-specific tests:

~~~text
8 passed
~~~

Commit:

~~~text
d97f547
feat: add batch outcome and DQ accounting
~~~

The commit was pushed successfully.

---

# 13. Task 13 — Pipeline Run and Execution Metadata

## Objective

Task 13 added durable execution-level metadata.

Task 12 answered:

> What happened to this processed batch?

Task 13 answers:

> Which execution or attempt performed the processing?

The architecture deliberately separates:

~~~text
Pipeline Run
     |
     v
Batch
     |
     v
Checkpoint
     |
     v
Source Position
~~~

## Run Lifecycle

Statuses:

~~~text
RUNNING
SUCCEEDED
FAILED
~~~

A run moves from RUNNING to either SUCCEEDED or FAILED.

## Database Change

Migration:

~~~text
migrations/005_create_telemetry_pipeline_run.sql
~~~

Table:

~~~text
public.telemetry_pipeline_run
~~~

Fields:

~~~text
id UUID PRIMARY KEY
pipeline_name TEXT NOT NULL
status TEXT NOT NULL
started_at TIMESTAMPTZ NOT NULL
last_heartbeat_at TIMESTAMPTZ NOT NULL
completed_at TIMESTAMPTZ NULL
error_code TEXT NULL
error_message TEXT NULL
created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
~~~

Constraints enforce valid lifecycle states.

For example:

~~~text
RUNNING -> completed_at must be NULL
SUCCEEDED/FAILED -> completed_at must be present
FAILED -> error_code must be present
RUNNING/SUCCEEDED -> failure metadata must be NULL
~~~

An index exists on:

~~~text
(pipeline_name, started_at DESC)
~~~

## Batch Association

Migration:

~~~text
migrations/006_link_telemetry_pipeline_batch_to_run.sql
~~~

adds:

~~~text
run_id UUID
~~~

to:

~~~text
public.telemetry_pipeline_batch
~~~

The foreign key points to:

~~~text
telemetry_pipeline_run.id
~~~

The field remains nullable so historical Task 12 batches do not receive fabricated run IDs.

## Run Module

Created:

~~~text
src/payment_telemetry/run.py
~~~

Functions:

~~~text
create_run()
heartbeat_run()
complete_run()
fail_run()
~~~

The run model is immutable.

The lifecycle helpers do not commit internally.

## Runner

Created:

~~~text
src/payment_telemetry/runner.py
~~~

Main entry point:

~~~text
run_pipeline(conn, limit=1000)
~~~

## Successful Run Flow

~~~text
run_pipeline()
    |
    +-- generate run_id
    |
    +-- create RUNNING run
    |
    +-- COMMIT
    |
    +-- process_batch(run_id)
    |
    +-- heartbeat
    |
    +-- COMMIT
    |
    +-- mark SUCCEEDED
    |
    +-- COMMIT
~~~

## Failure Flow

~~~text
create RUNNING run
       |
       v
COMMIT
       |
       v
process batch
       |
       X
batch transaction rolls back
       |
       v
separate failure transaction
       |
       v
RUNNING -> FAILED
       |
       v
COMMIT
       |
       v
original exception re-raised
~~~

This ensures failure metadata survives the failed batch transaction.

## Heartbeat

Task 13 adds a run-level last_heartbeat_at field.

This is a simple V1 foundation.

It does not implement:

- stale-run detection
- alerting
- concurrency management
- dashboarding
- heartbeat scheduling
- event-level heartbeat

Those concerns belong to later operational work.

## Tests

Created:

~~~text
tests/test_run.py
tests/test_runner.py
~~~

Updated:

~~~text
tests/test_accounting.py
tests/test_pipeline.py
~~~

Task 13 focused tests:

~~~text
18 passed in 1.29s
~~~

Full regression:

~~~text
46 passed in 0.64s
~~~

Real PostgreSQL validation confirmed the run lifecycle.

A real failure execution confirmed:

~~~text
run.status = FAILED
error_code = PIPELINE_RUN_FAILED
batch_rows = 0
checkpoint unchanged
~~~

Commit:

~~~text
341a655
feat: add pipeline run execution metadata
~~~

The commit was pushed successfully.

---

# 14. Task 14 — Reprocessing and Replay

## Objective

Task 14 provides controlled reprocessing of previously processed telemetry without corrupting normal incremental processing state.

The central decision is:

> Replay is an independent historical execution. It does not move, overwrite, or reuse the normal checkpoint.

## Architecture

Normal processing:

~~~text
Pipeline Run
    |
    v
Batch
    |
    v
Checkpoint
    |
    v
Source Position
~~~

Replay:

~~~text
Replay Run
    |
    v
Replay Scope
    |
    v
Historical Source Extraction
    |
    v
Validation
    |
    v
Staging / Quarantine
    |
    v
Replay Batch Accounting
~~~

The normal checkpoint is intentionally outside the replay flow.

---

# 15. Replay Scope

Replay uses the deterministic source ordering:

~~~text
(received_at, event_id)
~~~

Replay scope contains:

~~~text
start_received_at
start_event_id
end_received_at
end_event_id
~~~

The start must be less than or equal to the end.

The first replay batch is inclusive:

~~~text
(received_at, event_id) >= start
AND
(received_at, event_id) <= end
~~~

Later batches use:

~~~text
(received_at, event_id) > after
AND
(received_at, event_id) <= end
~~~

This creates deterministic pagination through a fixed historical range.

## Replay Scope Model

The implementation contains an immutable ReplayScope model with:

~~~text
start_received_at
start_event_id
end_received_at
end_event_id
~~~

Invalid ranges raise ValueError.

Replay extraction is provided by:

~~~text
extract_events_for_replay(...)
~~~

It supports:

- deterministic ordering
- inclusive historical range
- batch limits
- execution-local after cursors

It does not modify the normal checkpoint.

---

# 16. Why Replay Uses a Separate Cursor

The normal checkpoint represents:

> Persisted live incremental processing state.

The replay scope represents:

> An explicitly requested historical interval.

They have different meanings.

Therefore the implementation deliberately does not reuse the normal Checkpoint model for replay.

This prevents a historical operation from moving the live pipeline backward.

---

# 17. Replay Batch Processing

Replay processing uses:

~~~text
process_replay_batch(...)
~~~

and a ReplayBatchResult containing the batch result and the last replay position.

The replay position is execution-local.

It is not written into the normal live checkpoint.

This makes it possible to stop and resume a replay without changing the live incremental cursor.

---

# 18. Replay and Idempotency

Replay can encounter events already processed normally.

Existing persistence remains event-idempotent.

Staging and quarantine use event identity with conflict handling.

Therefore:

~~~text
same event
    |
    +--> normal processing
    |
    +--> replay
    |
    v
idempotent persistence
~~~

The replay implementation does not require the normal checkpoint to be reset.

---

# 19. Replay Accounting and Metadata

Task 14 added:

~~~text
migrations/007_add_replay_batch_accounting.sql
migrations/008_create_telemetry_pipeline_replay.sql
~~~

The first extends accounting for replay batches.

The second creates durable replay metadata.

This gives replay its own durable operational identity while reusing the existing processing components.

---

# 20. Task 14 Implementation Files

The Task 14 commit contains:

~~~text
migrations/007_add_replay_batch_accounting.sql
migrations/008_create_telemetry_pipeline_replay.sql

src/payment_telemetry/accounting.py
src/payment_telemetry/extractor.py
src/payment_telemetry/pipeline.py
src/payment_telemetry/run.py
src/payment_telemetry/runner.py

tests/test_accounting.py
tests/test_extractor.py
tests/test_pipeline.py
tests/test_run.py
tests/test_runner.py
~~~

Commit:

~~~text
b099769
feat: add telemetry replay and reprocessing
~~~

The commit contained:

~~~text
12 files changed
736 insertions
21 deletions
~~~

The protected Docker Compose modification remained outside the commit.

---

# 21. Replay Edge Cases and Failure Handling

## Invalid replay range

A start position after the end position is rejected.

## Empty replay

An empty replay result is handled as an empty operation rather than incorrectly advancing replay state.

A regression test was added for this behavior.

## Normal checkpoint

Replay must not move the normal checkpoint.

## Duplicate event

Existing event-level idempotency protects persistence.

## Failed replay

Failure must not be recorded as successful replay completion.

## Transaction failure

Existing transaction ownership is preserved so replay processing remains rollback-safe.

---

# 22. Task 14 Validation

Final full regression:

~~~text
57 passed in 2.07s
~~~

The final suite included:

~~~text
tests/test_accounting.py
tests/test_checkpoint.py
tests/test_config.py
tests/test_db.py
tests/test_extractor.py
tests/test_package.py
tests/test_pipeline.py
tests/test_quarantine.py
tests/test_run.py
tests/test_runner.py
tests/test_staging.py
tests/test_validation.py
~~~

Final result:

~~~text
57 passed
~~~

The documented test result was obtained after the final implementation and test changes and before the Task 14 commit.

---

# 23. Complete Data Flow After Task 14

## Normal Incremental Processing

~~~text
PostgreSQL Source
       |
       v
Checkpoint Cursor
       |
       v
Incremental Extraction
       |
       v
Validation
     /   \
    /     \
 Valid   Invalid
   |        |
   v        v
Staging  Quarantine
    \       /
     \     /
      v   v
Batch Accounting
       |
       v
Checkpoint Advancement
       |
       v
Next Incremental Run
~~~

## Historical Replay

~~~text
PostgreSQL Source
       |
       v
Explicit Replay Scope
       |
       v
Replay Cursor
       |
       v
Historical Extraction
       |
       v
Validation
     /   \
    /     \
 Valid   Invalid
   |        |
   v        v
Staging  Quarantine
     \      /
      \    /
       v  v
Replay Accounting
       |
       v
Replay Completion
~~~

The most important boundary is:

~~~text
Normal checkpoint != Replay cursor
~~~

---

# 24. Cross-Task Design Decisions

## Decision 1 — PostgreSQL owns durable pipeline state

Checkpoint, staging, quarantine, batch accounting, run metadata, and replay metadata are represented in PostgreSQL.

### Why

The project already uses PostgreSQL and needs transactional state.

### Why this layer

Processing state must be close to the database operations whose correctness depends on it.

---

## Decision 2 — Low-level helpers do not hide commits

### Implemented

Staging, quarantine, accounting, checkpoint, and run helpers operate within caller-owned transactions.

### Why

The pipeline needs to decide exactly which operations commit together.

### Problem solved

Hidden commits would break atomic rollback.

---

## Decision 3 — Accounting precedes checkpoint advancement

### Implemented

Batch accounting is persisted before the checkpoint is advanced.

### Why

A checkpoint is a statement about completed work.

### Problem solved

The pipeline cannot move past a batch whose durable accounting failed.

---

## Decision 4 — Invalid events are preserved

### Implemented

Invalid telemetry is written to persistent quarantine.

### Why

Invalid events may need investigation or later remediation.

### Problem solved

Data Quality failures do not silently disappear.

---

## Decision 5 — Idempotency is database-enforced

### Implemented

Staging and quarantine use event identity and PostgreSQL conflict handling.

### Why

Retries and replay can process the same event more than once.

### Problem solved

Repeated processing does not automatically create duplicate persistent state.

---

## Decision 6 — Run, batch, and checkpoint are separate

### Implemented

Each has a distinct responsibility.

### Why

Execution identity, batch outcome, and source position are different questions.

### Problem solved

One state record does not become overloaded with unrelated meanings.

---

## Decision 7 — Replay is independent of live processing

### Implemented

Replay has explicit historical scope and an execution-local cursor.

### Why

Historical processing must not move live incremental progress.

### Problem solved

A replay cannot accidentally reset or advance the normal checkpoint.

---

## Decision 8 — Historical batches are not assigned fabricated run IDs

### Implemented

The Task 13 batch run association remains nullable.

### Why

Historical batches predate the run metadata model.

### Problem solved

The system does not invent historical execution information.

---

# 25. Transaction Architecture

Normal batch processing follows the general model:

~~~text
BEGIN
  |
  +-- read checkpoint
  |
  +-- extract
  |
  +-- validate
  |
  +-- stage valid
  |
  +-- quarantine invalid
  |
  +-- record accounting
  |
  +-- advance checkpoint
  |
COMMIT
~~~

If a processing step fails:

~~~text
ROLLBACK
~~~

Run-level failure handling uses a separate transaction:

~~~text
Run creation
    |
    +-- COMMIT
         |
         v
Batch transaction
    |
    +-- failure
         |
         +-- ROLLBACK
         |
         v
Separate failure transaction
         |
         v
RUNNING -> FAILED
         |
         +-- COMMIT
~~~

The separation exists because failure metadata must survive the transaction that failed.

---

# 26. Failure Scenarios Covered

The implementation addresses:

### Validation failure

Invalid events are routed to quarantine.

### Duplicate event

Database conflict handling prevents duplicate persistence.

### Staging failure

The checkpoint does not advance.

### Quarantine failure

The checkpoint does not advance.

### Accounting failure

The checkpoint does not advance.

### Checkpoint failure

Earlier transactional work rolls back.

### Pipeline execution failure

The run is marked FAILED while preserving failure metadata.

### Invalid replay range

The replay request is rejected.

### Empty replay

Replay completes as an empty operation without corrupting state.

### Replay duplication

Existing event-level idempotency protects persistence.

---

# 27. Database and Environment Notes

The correct telemetry integration database was identified as:

~~~text
telemetry_integration_db
~~~

PostgreSQL service/container:

~~~text
zolvat-postgres
~~~

The repository environment configuration can contain:

~~~text
DB_NAME=emi_db
~~~

but the project documentation explicitly distinguishes emi_db from the telemetry integration database.

Therefore validation must use the correct integration database and must not assume the environment default points to the telemetry integration database.

The project also deliberately avoided dropping or recreating the integration database during these tasks.

---

# 28. Testing Strategy

The project uses several levels of verification.

## Unit Testing

Individual modules were tested for:

- validation
- staging
- checkpoint
- quarantine
- accounting
- run lifecycle
- runner
- extractor
- pipeline behavior

## PostgreSQL Integration Testing

Important persistence and transaction behavior was verified against PostgreSQL.

Examples include:

- staging persistence
- quarantine persistence
- checkpoint persistence
- batch accounting
- rollback behavior
- run lifecycle
- failure persistence

## Regression Testing

The documented regression progression was:

~~~text
Task 8 -> 8 passed
Task 9 -> 21 passed
Task 10 -> 32 passed
Task 12 -> 38 passed
Task 13 -> 46 passed
Task 14 -> 57 passed
~~~

These counts are the documented results at the corresponding task checkpoints.

---

# 29. Security, Privacy, and Performance

## Security and Privacy

The pipeline avoids adding sensitive payloads to processing metadata.

The documented privacy boundary excludes:

- passwords
- documents
- free text
- form values
- payment payloads
- unnecessary PII

## Performance

V1 favors correctness and durability over speculative optimization.

Potential future optimization areas include:

- batch throughput
- connection usage
- heartbeat frequency
- concurrency
- worker architecture
- storage optimization

These should be justified by measured workload characteristics.

The project deliberately does not introduce Kafka, Airflow, Spark, dbt, or S3 simply because they might become useful at larger scale.

---

# 30. Final Implementation State

At Task 14, the project has the following durable processing foundation:

~~~text
Frontend Telemetry Contract
        |
        v
PostgreSQL Telemetry Source
        |
        v
Deterministic Incremental Extraction
        |
        v
Persistent Checkpoint
        |
        v
Validation
     /     \
    /       \
Staging   Quarantine
    \       /
     \     /
      Batch Accounting
           |
           v
     Pipeline Run Metadata
           |
           +------------------+
           |                  |
           v                  v
     Normal Processing     Historical Replay
                              |
                              v
                       Replay Scope
                              |
                              v
                       Replay Cursor
                              |
                              v
                       Replay Accounting
~~~

Task 14 final state:

~~~text
Task 14: COMPLETE
Commit: b099769
Final regression: 57 passed in 2.07s
~~~

The next documented task after Task 14 is Task 15, but its exact implementation requirements must be established from the authoritative task specification before coding.

---

# 31. What This Real Project Teaches

The Payment Telemetry Pipeline demonstrates how production pipeline behavior can be built incrementally.

The progression is:

~~~text
Telemetry Contract
      |
      v
Staging
      |
      v
Validation
      |
      v
Incremental Processing
      |
      v
Persistent Quarantine
      |
      v
Batch Accounting
      |
      v
Run Metadata
      |
      v
Replay
~~~

Each stage solves a different operational problem.

### Staging

Makes intermediate data durable and replayable.

### Validation

Defines what acceptable telemetry looks like.

### Quarantine

Preserves invalid events instead of discarding them.

### Checkpointing

Defines where live incremental processing can continue.

### Batch Accounting

Records what happened to each processed batch.

### Run Metadata

Records which execution performed the work.

### Replay

Allows historical processing without corrupting live incremental state.

The central engineering lesson is:

> A production pipeline is not just a sequence of transformations. It is a system of durable states, transaction boundaries, failure paths, and recovery mechanisms.

That is the practical value of this project as a real-project recipe.
