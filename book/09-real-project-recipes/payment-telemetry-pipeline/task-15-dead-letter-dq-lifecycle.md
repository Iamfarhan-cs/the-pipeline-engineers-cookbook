# Recipe 15 — Dead-Letter / Data Quality Lifecycle

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 15  
> **Source implementation:** Task 15 — Dead-Letter / DQ Lifecycle  
> **Source repository:** <code>devops</code>

## 1. What This Recipe Teaches

Task 15 turns the telemetry quarantine table from a passive failure store into an active Dead-Letter / Data Quality management system.

By completing this recipe, you should understand how to:

- model a durable lifecycle for invalid events;
- represent lifecycle state with a database-backed state machine;
- enforce allowed states with PostgreSQL constraints;
- validate application-level state transitions;
- track retry and reprocessing counts;
- store resolution metadata;
- make status updates idempotent;
- execute lifecycle updates transactionally;
- preserve historical batch-accounting snapshots;
- keep replay processing separate from normal incremental checkpoints;
- test valid, invalid, terminal, and repeated state transitions.

The supplied Task 15 documentation defines the lifecycle as an in-database state machine with database-enforced invariants, atomic transitions, idempotency, retry counting, and transaction safety. fileciteturn28file0L5-L12

## 2. The Problem

Before this task, quarantined events could be stored with their validation errors, but quarantine primarily represented a failure destination.

That is not enough for operational remediation.

An invalid event may need to move through several stages:

~~~text
validation failure
       |
       v
QUARANTINED
       |
       v
RETRY_PENDING
       |
       v
REPROCESSING
       |
       v
RESOLVED
~~~

Some events may instead be permanently invalid:

~~~text
QUARANTINED
       |
       v
UNRESOLVABLE
~~~

The lifecycle therefore needs durable state, retry tracking, resolution metadata, and rules preventing impossible transitions. fileciteturn28file0L8-L12 fileciteturn28file0L47-L66

## 3. Implementation Sequence

The supplied implementation followed this sequence:

1. Verify repository state and protected files.
2. Investigate the existing specification and obtain authorization.
3. Analyze the existing quarantine schema and ingestion code.
4. Create migration <code>009_add_quarantine_lifecycle.sql</code>.
5. Add lifecycle fields and database constraints.
6. Implement the lifecycle model and state-transition helpers.
7. Add lifecycle tests.
8. Validate PostgreSQL compatibility.
9. Run the full regression suite.
10. Audit the resulting repository state.

The source explicitly records this implementation sequence. fileciteturn28file0L16-L26

## 4. Step 1 — Extend the Existing Quarantine Table

Task 15 extends:

~~~text
public.telemetry_event_quarantine
~~~

rather than creating a second lifecycle table.

Create:

~~~text
migrations/009_add_quarantine_lifecycle.sql
~~~

The migration adds:

~~~text
status
retry_count
updated_at
resolved_at
resolved_by
resolution_reason
reprocessing_run_id
~~~

It also adds:

~~~text
chk_quarantine_status
chk_quarantine_retry_count
idx_telemetry_quarantine_status
~~~

The supplied implementation explicitly documents this migration. fileciteturn28file0L20-L24 fileciteturn28file0L30-L34

## 5. Why Extend the Same Table?

The documented design chooses direct table extension rather than a separate audit table.

The stated reasons are:

- high query efficiency;
- avoiding unnecessary schema fragmentation;
- backward compatibility with existing batch ingestion paths.

New lifecycle fields receive defaults so existing quarantine insertion continues to work.

The initial lifecycle status is:

~~~text
DEFAULT 'QUARANTINED'
~~~

This design decision is explicitly documented in the supplied source. fileciteturn28file0L38-L41

## 6. Database Schema

The resulting table is:

~~~text
public.telemetry_event_quarantine
~~~

| Column | Type | Constraints | Purpose |
|---|---|---|---|
| <code>id</code> | BIGINT | PRIMARY KEY | Identity surrogate key |
| <code>event_id</code> | UUID | NOT NULL UNIQUE | Ingested event identifier |
| <code>user_id</code> | UUID | NOT NULL | Associated user UUID |
| <code>event_name</code> | TEXT | NOT NULL | Telemetry event name |
| <code>event_version</code> | INTEGER | NOT NULL | Schema version |
| <code>occurred_at</code> | TIMESTAMPTZ | NOT NULL | Event occurrence timestamp |
| <code>route</code> | TEXT | NOT NULL | Frontend route pathname |
| <code>properties</code> | JSONB | NOT NULL | Event property payload |
| <code>received_at</code> | TIMESTAMPTZ | NOT NULL | Ingestion timestamp |
| <code>app_version</code> | TEXT | NOT NULL | App client version |
| <code>errors</code> | JSONB | NOT NULL | Validation error strings |
| <code>quarantined_at</code> | TIMESTAMPTZ | NOT NULL DEFAULT CURRENT_TIMESTAMP | Initial quarantine timestamp |
| <code>status</code> | TEXT | NOT NULL DEFAULT 'QUARANTINED' | Current lifecycle state |
| <code>retry_count</code> | INTEGER | NOT NULL DEFAULT 0 | Retry/reprocessing count |
| <code>updated_at</code> | TIMESTAMPTZ | NOT NULL DEFAULT CURRENT_TIMESTAMP | Last lifecycle update |
| <code>resolved_at</code> | TIMESTAMPTZ | NULL | Resolution timestamp |
| <code>resolved_by</code> | TEXT | NULL | Operator/system resolving the event |
| <code>resolution_reason</code> | TEXT | NULL | Resolution rationale |
| <code>reprocessing_run_id</code> | UUID | NULL FK | Replay/reprocessing execution run |

The supplied documentation defines the complete schema. fileciteturn28file0L92-L115

## 7. Step 2 — Define Lifecycle States

The state machine contains five states:

| State | Meaning |
|---|---|
| <code>QUARANTINED</code> | Initial state after validation failure |
| <code>RETRY_PENDING</code> | Event is waiting for automated or manual retry |
| <code>REPROCESSING</code> | Event is currently being reprocessed |
| <code>RESOLVED</code> | Terminal state for successful remediation or manual resolution |
| <code>UNRESOLVABLE</code> | Terminal state for corrupt or permanently invalid events |

These are the documented lifecycle states. fileciteturn28file0L45-L52

## 8. Lifecycle State Machine

The documented transition graph is:

~~~text
                 QUARANTINED
                 /          \
                /            \
               v              v
      RETRY_PENDING      UNRESOLVABLE
            |
            v
       REPROCESSING
            |
            v
         RESOLVED

REPROCESSING may also return to:

       QUARANTINED
          |
          v
      RETRY_PENDING
~~~

The supplied source specifically documents the retry-failure path from <code>REPROCESSING</code> back to <code>QUARANTINED</code>. fileciteturn28file0L54-L63

The important rule is that terminal states cannot be moved to a different state.

~~~text
RESOLVED       -> terminal
UNRESOLVABLE   -> terminal
~~~

Attempts to move a terminal record to a non-identical state raise <code>ValueError</code>. fileciteturn28file0L66-L66

## 9. Step 3 — Add the Application Lifecycle Model

Update:

~~~text
src/payment_telemetry/quarantine.py
~~~

Add:

~~~text
QuarantineStatus
ALLOWED_TRANSITIONS
QuarantineRecord
~~~

The implementation uses <code>QuarantineStatus</code> as the application representation of the lifecycle and <code>ALLOWED_TRANSITIONS</code> as the transition policy.

The supplied source explicitly identifies these additions. fileciteturn28file0L21-L23

## 10. Why Use ALLOWED_TRANSITIONS?

A state machine should not depend on callers remembering every legal transition.

Instead, define the transition policy centrally:

~~~text
current state
     |
     v
ALLOWED_TRANSITIONS
     |
     v
allowed next states
~~~

Then <code>update_quarantine_status()</code> can reject illegal transitions consistently.

This is an application-level guard in addition to the database's allowed-state constraint.

## 11. Step 4 — Add QuarantineRecord

The lifecycle implementation introduces:

~~~text
QuarantineRecord
~~~

This model represents the stored quarantine record together with its lifecycle metadata.

The record needs to expose the fields required for lifecycle operations, including:

~~~text
event identity
status
retry_count
updated_at
resolved_at
resolved_by
resolution_reason
reprocessing_run_id
~~~

The supplied implementation specifically adds the dataclass and tests its mapping behavior. fileciteturn28file0L22-L23 fileciteturn28file0L121-L126

## 12. Step 5 — Query a Single Quarantined Event

Implement:

~~~python
get_quarantine_record(...)
~~~

This helper retrieves one lifecycle record so callers can inspect its current state before performing remediation.

The implementation adds this query helper as part of the lifecycle API. fileciteturn28file0L21-L23

A typical lifecycle operation should therefore follow:

~~~text
load current record
       |
       v
inspect current status
       |
       v
validate requested transition
       |
       v
apply update
~~~

## 13. Step 6 — List Quarantined Events

Implement:

~~~python
list_quarantined_events(...)
~~~

This provides an operational query path for finding events requiring attention.

The source identifies status-query filtering as part of the test suite, so lifecycle queries are expected to support filtering around the stored state. fileciteturn28file0L22-L23 fileciteturn28file0L121-L126

The status index:

~~~text
idx_telemetry_quarantine_status
~~~

supports this lifecycle-oriented query pattern. fileciteturn28file0L30-L33

## 14. Step 7 — Validate Status Transitions

Implement:

~~~python
update_quarantine_status(...)
~~~

The function must validate:

1. the requested status is a recognized lifecycle state;
2. the transition from the current state is permitted;
3. terminal states cannot be moved to a different state;
4. repeated updates to the current state are handled idempotently.

The supplied source explicitly identifies valid/invalid transition testing and idempotent status updates. fileciteturn28file0L22-L23 fileciteturn28file0L77-L80

## 15. Idempotent Same-State Updates

A request to update an event to the state it already has should not cause unnecessary database work.

Conceptually:

~~~text
current status = QUARANTINED
requested status = QUARANTINED
             |
             v
return existing record
             |
             X
       no redundant SQL
~~~

The source explicitly documents this behavior. fileciteturn28file0L77-L80

## 16. Step 8 — Remediation Helper

Implement:

~~~python
remediate_quarantined_event(...)
~~~

This helper represents the operational path for moving a quarantined event through remediation.

It works with lifecycle status and resolution metadata rather than simply deleting the event.

The supplied source explicitly identifies <code>remediate_quarantined_event</code> as part of the application implementation. fileciteturn28file0L21-L23

## 17. Step 9 — Mark an Event Unresolvable

Implement:

~~~python
mark_unresolvable(...)
~~~

This provides the terminal path for an event that cannot be corrected.

Conceptually:

~~~text
QUARANTINED
     |
     v
UNRESOLVABLE
     |
     v
terminal
~~~

Once an event is <code>UNRESOLVABLE</code>, later lifecycle operations must not move it to another state. fileciteturn28file0L51-L66

## 18. Resolution Metadata

The lifecycle schema introduces:

~~~text
resolved_at
resolved_by
resolution_reason
~~~

These fields preserve the context of a resolution instead of recording only the final status.

For example:

~~~text
status            = RESOLVED
resolved_at       = <timestamp>
resolved_by       = <operator/system>
resolution_reason = <reason>
~~~

The supplied schema explicitly defines these columns. fileciteturn28file0L109-L115

## 19. Retry Counting

The lifecycle adds:

~~~text
retry_count INTEGER NOT NULL DEFAULT 0
~~~

This records the total retry/reprocessing count.

The database also applies a non-negative retry-count constraint.

~~~text
retry_count >= 0
~~~

The source explicitly identifies the retry field and its database check constraint. fileciteturn28file0L30-L34 fileciteturn28file0L109-L111

## 20. Reprocessing Run Association

The lifecycle adds:

~~~text
reprocessing_run_id UUID
~~~

with a foreign-key relationship to:

~~~text
telemetry_pipeline_run
~~~

This allows lifecycle records to identify the execution run associated with replay/reprocessing work. fileciteturn28file0L109-L115

This is distinct from the immutable historical batch accounting introduced earlier.

## 21. Transaction Safety

There are two important transaction boundaries.

### Ingestion

Invalid events are inserted into quarantine inside the orchestrator's atomic batch transaction:

~~~python
with conn.transaction():
    ...
~~~

The source explicitly confirms this behavior. fileciteturn28file0L70-L73

### Remediation

Status and remediation metadata updates execute within explicit database transactions.

This ensures related lifecycle fields change atomically.

For example, these fields must not partially update:

~~~text
status
updated_at
resolution_reason
resolved_at
resolved_by
~~~

The supplied source explicitly documents atomic remediation updates. fileciteturn28file0L70-L73

## 22. Step 10 — Preserve Historical Batch Accounting

Task 14 introduced:

~~~text
telemetry_pipeline_batch
~~~

Its <code>quarantined_count</code> represents the outcome of the original batch.

Task 15 does not rewrite that historical count when a quarantined event is later retried or resolved.

The source explicitly states that historical <code>quarantined_count</code> values remain immutable snapshots of initial batch extraction outcomes. fileciteturn28file0L84-L88

This creates two distinct concepts:

~~~text
Batch accounting
    |
    +--> historical snapshot

Quarantine lifecycle
    |
    +--> current remediation state
~~~

Do not confuse the two.

## 23. Replay Isolation

The source establishes that:

~~~text
process_replay_batch
~~~

operates independently from normal incremental checkpoints.

Replay must not corrupt:

- normal incremental checkpoint state;
- historical lifecycle audit fields.

The supplied documentation explicitly records this separation. fileciteturn28file0L84-L88

## 24. Database-Enforced State Invariants

The migration applies:

~~~text
chk_quarantine_status
chk_quarantine_retry_count
~~~

The purpose is to prevent impossible stored states.

For example:

~~~text
status = UNKNOWN_STATE
~~~

must not be persisted.

Likewise:

~~~text
retry_count = -1
~~~

must be rejected.

The source explicitly defines these database constraints. fileciteturn28file0L30-L34 fileciteturn28file0L40-L41

## 25. Application vs Database Enforcement

Use both layers.

### Application layer

~~~text
current state
      |
      v
allowed transition?
      |
      +--> yes -> update
      |
      +--> no  -> reject
~~~

### Database layer

~~~text
stored status is valid
retry_count >= 0
~~~

The combination prevents both invalid transitions and invalid persisted values. fileciteturn28file0L21-L23 fileciteturn28file0L38-L41

## 26. Tests

The complete regression reports:

~~~text
66 passed in 0.29s
~~~

The source states that:

- 57 existing regression tests continued to pass;
- 9 new lifecycle tests were added.

The new tests cover:

- dataclass mappings;
- status query filtering;
- valid transitions;
- state-transition guards;
- idempotent updates;
- helper methods.

fileciteturn28file0L119-L126

## 27. Test Scenario — Valid Transition

Start:

~~~text
QUARANTINED
~~~

Request:

~~~text
RETRY_PENDING
~~~

Expected:

~~~text
transition accepted
status = RETRY_PENDING
~~~

Then:

~~~text
RETRY_PENDING
       |
       v
REPROCESSING
       |
       v
RESOLVED
~~~

The source defines these states and transition paths. fileciteturn28file0L47-L63

## 28. Test Scenario — Invalid Transition

Attempt a transition that is not in the allowed transition map.

Expected:

~~~text
ValueError
status unchanged
~~~

This verifies the state-machine guard. The source explicitly identifies state transition guards as part of the test suite. fileciteturn28file0L121-L126

## 29. Test Scenario — Terminal State Protection

Start:

~~~text
RESOLVED
~~~

Attempt:

~~~text
RESOLVED -> RETRY_PENDING
~~~

Expected:

~~~text
ValueError
~~~

The same rule applies to <code>UNRESOLVABLE</code>.

Terminal states cannot be moved to non-identical states. fileciteturn28file0L66-L66

## 30. Test Scenario — Idempotent Update

Start:

~~~text
QUARANTINED
~~~

Request:

~~~text
QUARANTINED
~~~

Expected:

~~~text
existing record returned
no redundant SQL
~~~

This protects callers against duplicate lifecycle commands. fileciteturn28file0L77-L80

## 31. PostgreSQL Validation

The source attempted live PostgreSQL validation using:

~~~text
localhost:5432
~~~

The local port was unreachable.

The implementation therefore recorded the connectivity limitation explicitly and validated the migration's:

- DDL syntax;
- CHECK constraints;
- foreign-key declarations;
- default values;

against PostgreSQL specification requirements. fileciteturn28file0L130-L132

Do not describe this task as having completed a successful live database integration test. The supplied source does not support that claim.

## 32. Privacy

Task 15 does not introduce raw payment information, authorization tokens, free-text form inputs, or unredacted PII into quarantine metadata.

The error field remains bounded validation-message data.

The supplied source explicitly documents this privacy boundary. fileciteturn28file0L136-L138

## 33. Common Implementation Mistakes

### Mistake 1 — Treat quarantine as a permanent final state

Quarantine is now a lifecycle starting point, not the entire remediation system.

### Mistake 2 — Allow arbitrary status changes

Use an explicit transition map.

### Mistake 3 — Rely only on application validation

Database constraints must also prevent invalid persisted status values.

### Mistake 4 — Allow negative retry counts

Enforce <code>retry_count >= 0</code> at the database layer.

### Mistake 5 — Mutate terminal states

<code>RESOLVED</code> and <code>UNRESOLVABLE</code> are terminal.

### Mistake 6 — Rewrite Task 14 accounting after remediation

Historical batch accounting remains an immutable snapshot.

### Mistake 7 — Commit lifecycle fields separately

Status and related remediation metadata should update atomically.

### Mistake 8 — Treat replay as normal incremental ingestion

Replay execution must remain separate from normal checkpoint progression.

### Mistake 9 — Claim live PostgreSQL validation when the database was unreachable

The supplied implementation explicitly records that <code>localhost:5432</code> was unreachable.

## 34. Independent Implementation Exercise

Build a Dead-Letter lifecycle for another event-processing system.

Requirements:

1. Extend an existing quarantine table.
2. Add an initial <code>QUARANTINED</code> state.
3. Add retry tracking.
4. Add lifecycle timestamps.
5. Add resolution metadata.
6. Add a reprocessing-run identifier.
7. Define a finite state machine.
8. Define allowed transitions centrally.
9. Reject invalid transitions.
10. Make terminal states immutable.
11. Make same-state updates idempotent.
12. Enforce valid states in PostgreSQL.
13. Enforce non-negative retry counts.
14. Execute remediation updates transactionally.
15. Preserve historical batch-accounting snapshots.
16. Keep replay processing separate from normal checkpoints.
17. Add unit tests for every state transition.
18. Test terminal-state protection.
19. Test repeated same-state requests.
20. Test migration constraints.

Then simulate:

~~~text
QUARANTINED
    |
    v
RETRY_PENDING
    |
    v
REPROCESSING
    |
    v
RESOLVED
~~~

Also simulate:

~~~text
QUARANTINED
    |
    v
UNRESOLVABLE
~~~

Then verify that:

~~~text
RESOLVED -> RETRY_PENDING
UNRESOLVABLE -> RETRY_PENDING
~~~

are rejected.

Finally, repeat an identical status update and verify that it performs no redundant SQL.

## 35. Validation Checklist

### Schema

- [ ] <code>telemetry_event_quarantine</code> contains lifecycle columns.
- [ ] <code>status</code> defaults to <code>QUARANTINED</code>.
- [ ] <code>retry_count</code> defaults to zero.
- [ ] <code>updated_at</code> has a default.
- [ ] Resolution metadata columns exist.
- [ ] <code>reprocessing_run_id</code> is linked to the run table.
- [ ] Status index exists.

### State machine

- [ ] <code>QUARANTINED</code> exists.
- [ ] <code>RETRY_PENDING</code> exists.
- [ ] <code>REPROCESSING</code> exists.
- [ ] <code>RESOLVED</code> exists.
- [ ] <code>UNRESOLVABLE</code> exists.
- [ ] Allowed transitions are explicit.
- [ ] Invalid transitions are rejected.
- [ ] Terminal states are protected.

### Retry and remediation

- [ ] Retry count is non-negative.
- [ ] Resolution timestamp is stored.
- [ ] Resolver identity is stored when applicable.
- [ ] Resolution reason is stored.
- [ ] Reprocessing run ID can be associated.

### Idempotency

- [ ] Same-state updates return the existing record.
- [ ] No redundant SQL is executed for identical status requests.
- [ ] Ingestion retains <code>ON CONFLICT (event_id) DO NOTHING</code>.

### Transactions

- [ ] Quarantine insertion remains inside the batch transaction.
- [ ] Remediation updates are atomic.
- [ ] Lifecycle metadata cannot partially update.

### Integration

- [ ] Task 14 batch accounting remains historical and immutable.
- [ ] Replay does not corrupt normal checkpoints.
- [ ] Lifecycle state is independent from historical batch counts.

### Tests

- [ ] Dataclass mapping tests pass.
- [ ] Status filtering tests pass.
- [ ] Valid transition tests pass.
- [ ] Invalid transition tests pass.
- [ ] Terminal-state tests pass.
- [ ] Idempotency tests pass.
- [ ] Full 66-test regression passes.

### Privacy

- [ ] No raw payment information is introduced.
- [ ] No authorization tokens are introduced.
- [ ] No free-text form inputs are introduced.
- [ ] No unredacted PII is introduced.
- [ ] Validation errors remain bounded messages.

## 36. Scope Boundaries

### Implemented in Task 15

- durable quarantine lifecycle;
- lifecycle status state machine;
- database-enforced status and retry invariants;
- retry counting;
- status queries;
- lifecycle transition updates;
- remediation helper;
- unresolvable helper;
- resolution metadata;
- reprocessing run association;
- idempotent status updates;
- transaction-safe lifecycle changes;
- lifecycle tests.

### Future Tasks

The supplied source explicitly identifies:

- **Task 16:** Late-Arrival & Backfill Handling
- **Task 17:** Data Quality Aggregation Metrics
- **Task 18:** Observability / OTEL Integration
- **Task 19:** System Alerting

fileciteturn28file0L142-L148

## 37. What You Should Understand Before the Next Recipe

Task 15 changes quarantine from:

~~~text
invalid event
      |
      v
passive quarantine row
~~~

into:

~~~text
invalid event
      |
      v
QUARANTINED
      |
      +--> RETRY_PENDING
      |        |
      |        v
      |   REPROCESSING
      |        |
      |        v
      |     RESOLVED
      |
      +--> UNRESOLVABLE
~~~

The key engineering lessons are:

- Dead-Letter storage can become a stateful operational system;
- state transitions should be explicit rather than arbitrary;
- application transition rules and database constraints serve different purposes;
- terminal states protect historical remediation decisions;
- retry counts provide operational lifecycle visibility;
- resolution metadata preserves the reason and ownership of remediation;
- lifecycle updates must be atomic;
- repeated lifecycle commands should be idempotent;
- historical batch accounting and current remediation state are separate concepts;
- replay processing should not corrupt normal incremental checkpoints;
- privacy boundaries must remain intact during remediation.

The supplied implementation reports 66 passing tests, including 9 new lifecycle tests, while explicitly documenting that live PostgreSQL connectivity was unavailable in the validation environment. fileciteturn28file0L119-L132

## Source Traceability

This recipe is derived from the supplied Task 15 implementation documentation:

- Task overview and objective: fileciteturn28file0L5-L12
- Implementation sequence: fileciteturn28file0L16-L26
- Technical changes: fileciteturn28file0L30-L34
- Design decisions: fileciteturn28file0L38-L41
- Lifecycle states and transitions: fileciteturn28file0L45-L66
- Transaction safety and idempotency: fileciteturn28file0L70-L80
- Pipeline integration: fileciteturn28file0L84-L88
- Database schema: fileciteturn28file0L92-L115
- Tests and PostgreSQL validation: fileciteturn28file0L119-L132
- Privacy and future scope: fileciteturn28file0L136-L148
