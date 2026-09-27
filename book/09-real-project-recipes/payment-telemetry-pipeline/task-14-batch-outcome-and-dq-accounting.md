# Recipe 14 — Batch Outcome and Data Quality Accounting

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 14  
> **Source implementation commit:** <code>d97f547</code> — <code>feat: add batch outcome and DQ accounting</code>  
> **Source repository:** <code>devops</code>

## 1. What This Recipe Teaches

Stage 14 adds a durable per-batch data-quality ledger to the Payment Telemetry Pipeline.

By completing this recipe, you should understand how to:

- record an immutable historical outcome for every processed batch;
- track exact first/last event boundaries;
- distinguish classification counts from actual persistence counts;
- enforce DQ arithmetic invariants in PostgreSQL;
- make accounting idempotent;
- keep accounting inside the same transaction as staging, quarantine, and checkpoint advancement;
- prevent checkpoint advancement when accounting fails;
- keep the accounting ledger free of event-level user data.

The supplied implementation introduces migration <code>004_create_telemetry_pipeline_batch.sql</code>, frozen <code>BatchOutcome</code>, <code>record_batch_outcome()</code>, and accounting immediately before checkpoint persistence. fileciteturn25file0L3-L15

## 2. The Problem

Stage 13 already separates valid and invalid events:

~~~text
extract
   |
validate
   |
   +--> valid   -> staging
   |
   +--> invalid -> quarantine
   |
   +--> checkpoint
~~~

Operational systems also need durable answers to:

- How many events were extracted?
- How many were valid?
- How many were invalid?
- How many valid rows were actually inserted?
- How many invalid rows were actually quarantined?
- What exact event boundary did the batch cover?

Stage 14 creates a PostgreSQL ledger for durable auditability and reconciliation. fileciteturn25file0L8-L15

## 3. The Core DQ Invariants

The database enforces:

~~~text
extracted = valid + invalid
staged <= valid
quarantined <= invalid
~~~

It also rejects negative counters.

These are database-enforced <code>CHECK</code> constraints, so inconsistent accounting cannot be persisted even if application code contains a bug. fileciteturn25file0L31-L38 fileciteturn25file0L44-L48

## 4. Classification Count vs Persistence Count

These counts represent different facts.

Example:

~~~text
10 extracted
8 valid
2 invalid
7 staged
2 quarantined
~~~

This is valid.

A <code>staged_count</code> of 7 with <code>valid_count</code> of 8 can occur when one valid event already exists and staging uses <code>ON CONFLICT DO NOTHING</code>.

Therefore:

~~~text
valid_count  = events classified as valid
staged_count = rows actually inserted
~~~

The same distinction applies to invalid versus quarantined events. fileciteturn25file0L44-L48

## 5. Implementation Sequence

The source implementation follows this order:

1. Create the batch ledger migration.
2. Implement frozen <code>BatchOutcome</code>.
3. Implement <code>record_batch_outcome()</code>.
4. Integrate accounting into <code>process_batch()</code>.
5. Record accounting before the checkpoint.
6. Add unit and transaction tests.
7. Run the full regression suite.

The supplied implementation reports 38 passing tests. fileciteturn25file0L19-L25

## 6. Step 1 — Create the Batch Ledger

Create:

~~~text
migrations/004_create_telemetry_pipeline_batch.sql
~~~

The migration creates:

~~~text
public.telemetry_pipeline_batch
~~~

This table is the durable ledger for processed batch outcomes. fileciteturn25file0L21-L24

### Schema

| Column | Type | Purpose |
|---|---|---|
| <code>id</code> | BIGINT | Identity primary key |
| <code>pipeline_name</code> | TEXT | Pipeline identifier |
| <code>first_received_at</code> | TIMESTAMPTZ | First event ingestion time |
| <code>first_event_id</code> | UUID | First event identifier |
| <code>last_received_at</code> | TIMESTAMPTZ | Last event ingestion time |
| <code>last_event_id</code> | UUID | Last event identifier |
| <code>extracted_count</code> | INTEGER | Events extracted |
| <code>valid_count</code> | INTEGER | Events classified valid |
| <code>invalid_count</code> | INTEGER | Events classified invalid |
| <code>staged_count</code> | INTEGER | Valid rows actually inserted |
| <code>quarantined_count</code> | INTEGER | Invalid rows actually inserted |
| <code>created_at</code> | TIMESTAMPTZ | Ledger recording time |

All counters are non-negative. The source documents the complete schema and constraints. fileciteturn25file0L96-L117

## 7. Step 2 — Enforce Boundary Uniqueness

The ledger defines:

~~~text
UNIQUE (pipeline_name, last_received_at, last_event_id)
~~~

The named constraint is:

~~~text
telemetry_pipeline_batch_boundary_unique
~~~

This makes the last event boundary the idempotency identity for a processed batch. fileciteturn25file0L31-L38 fileciteturn25file0L81-L85

## 8. Step 3 — Enforce Arithmetic Constraints

The migration defines:

~~~sql
CHECK (extracted_count = valid_count + invalid_count)
~~~

and:

~~~sql
CHECK (
    staged_count <= valid_count
    AND quarantined_count <= invalid_count
)
~~~

There is also a non-negative-count constraint.

This makes PostgreSQL the final enforcement point for the accounting model. fileciteturn25file0L35-L38

## 9. Step 4 — Implement BatchOutcome

Create:

~~~text
src/payment_telemetry/accounting.py
~~~

Implement a frozen <code>BatchOutcome</code> dataclass containing the batch boundary and five outcome counts:

~~~text
pipeline_name
first_received_at
first_event_id
last_received_at
last_event_id
extracted_count
valid_count
invalid_count
staged_count
quarantined_count
~~~

The model is frozen so its values cannot be mutated after construction. The source explicitly identifies the frozen model. fileciteturn25file0L21-L24

## 10. Step 5 — Implement record_batch_outcome()

Implement:

~~~python
record_batch_outcome(conn, outcome)
~~~

It inserts into:

~~~text
public.telemetry_pipeline_batch
~~~

using:

~~~sql
ON CONFLICT (pipeline_name, last_received_at, last_event_id)
DO NOTHING
~~~

This makes repeated processing of the same boundary idempotent. fileciteturn25file0L38-L39 fileciteturn25file0L81-L85

### No Internal Commit

The accounting helper performs database work but does not commit.

~~~text
record_batch_outcome()
    |
    +--> execute INSERT
    |
    +--> return
    |
    X--> do not commit
~~~

The transaction remains owned by <code>process_batch()</code>. The source includes a test confirming no internal commit. fileciteturn25file0L121-L129

## 11. Step 6 — Integrate Accounting Into process_batch()

The orchestrator already knows:

~~~text
events
valid events
invalid events
staged_count
quarantined_count
~~~

Construct:

~~~text
BatchOutcome(
    extracted_count = len(events),
    valid_count = len(valid),
    invalid_count = len(invalid),
    staged_count = staged_count,
    quarantined_count = quarantined_count
)
~~~

The source explicitly documents these values. fileciteturn25file0L54-L68

Use the actual row counts returned by staging and quarantine. Do not replace them with <code>len(valid)</code> or <code>len(invalid)</code>. fileciteturn25file0L44-L48

## 12. Step 7 — Accounting Before the Checkpoint

The exact ordering is:

~~~text
stage valid events
        |
        v
quarantine invalid events
        |
        v
record_batch_outcome()
        |
        v
save_checkpoint()
~~~

Accounting must occur immediately before checkpoint persistence. fileciteturn25file0L8-L9 fileciteturn25file0L38-L39

This creates:

~~~text
checkpoint advances
        only after
batch accounting succeeds
~~~

## 13. Complete Transaction Flow

~~~text
process_batch() inside conn.transaction()
   |
   +--> extract events
   |
   +--> validate events
   |
   +--> stage valid events
   |       +--> staged_count
   |
   +--> quarantine invalid events
   |       +--> quarantined_count
   |
   +--> construct BatchOutcome
   |
   +--> record_batch_outcome()
   |
   +--> save_checkpoint()
   |
   v
COMMIT
~~~

The source documents this flow directly. fileciteturn25file0L52-L77

## 14. Transaction Safety

Suppose accounting fails:

~~~text
stage             -> success
quarantine        -> success
accounting        -> FAILURE
checkpoint        -> not reached
~~~

The transaction rolls back.

Therefore the batch cannot leave committed staging/quarantine work while the checkpoint advances without accounting.

The source explicitly states that accounting failure aborts the transaction and prevents checkpoint advancement. fileciteturn25file0L44-L48

## 15. Idempotency

First processing:

~~~text
batch boundary
      |
      v
ledger row inserted
~~~

Retry with the same boundary:

~~~text
same boundary
      |
      v
ON CONFLICT DO NOTHING
      |
      v
no duplicate ledger row
~~~

The uniqueness constraint and conflict clause provide this behavior. fileciteturn25file0L81-L85

## 16. Why Store First and Last Boundaries?

Counts tell you what happened. Boundaries tell you which events the batch covered.

The ledger stores:

~~~text
first_received_at
first_event_id
last_received_at
last_event_id
~~~

This creates a durable processing boundary for audit and reconciliation. fileciteturn25file0L11-L15 fileciteturn25file0L31-L37

## 17. Integration With Existing Stages

~~~text
Stage 9  -> staged_count
Stage 10 -> valid_count + invalid_count
Stage 13 -> quarantined_count
Stage 11 -> checkpoint, executed after accounting
~~~

The source explicitly identifies these upstream and downstream relationships. fileciteturn25file0L89-L92

## 18. Example: Valid Outcome

~~~text
extracted       = 10
valid           = 8
invalid         = 2
staged          = 7
quarantined     = 2
~~~

Accepted because:

~~~text
10 = 8 + 2
7 <= 8
2 <= 2
~~~

## 19. Example: Invalid Outcome

~~~text
extracted = 10
valid     = 7
invalid   = 2
~~~

Rejected because:

~~~text
10 != 7 + 2
~~~

The supplied implementation validates these constraints against PostgreSQL 16. fileciteturn25file0L134-L136

## 20. Tests

The Stage 14 regression reports:

~~~text
38 passed in 0.18s
~~~

Important new tests include:

~~~text
test_record_batch_outcome_inserts_without_committing
test_batch_outcome_is_immutable
test_process_batch_propagates_accounting_failure_before_checkpoint
~~~

They verify:

- no internal accounting commit;
- immutable <code>BatchOutcome</code>;
- checkpoint protection when accounting fails.

The source documents these tests and the full regression. fileciteturn25file0L121-L130

## 21. PostgreSQL Validation

The migration was validated against PostgreSQL 16.

Verify:

- the table is created correctly;
- negative counts are rejected;
- mismatched classification counts are rejected;
- staged counts above valid counts are rejected;
- quarantined counts above invalid counts are rejected;
- duplicate boundaries cannot create duplicate ledger rows.

The source explicitly records PostgreSQL 16 arithmetic-constraint validation. fileciteturn25file0L134-L136

## 22. Privacy and Data Protection

The ledger records operational information:

~~~text
pipeline name
event boundaries
integer outcome counts
recording timestamp
~~~

It does not store:

~~~text
user identifiers
routes
event properties
~~~

The source explicitly states that no user identifiers, routes, or properties are stored in the accounting ledger. fileciteturn25file0L140-L142

## 23. Common Implementation Mistakes

### Mistake 1 — Treat classification as persistence

Do not use <code>len(valid)</code> as <code>staged_count</code>. Use the actual inserted row count.

### Mistake 2 — Commit accounting separately

Accounting must participate in the same transaction as staging, quarantine, and checkpoint advancement.

### Mistake 3 — Save the checkpoint first

Accounting must succeed before checkpoint advancement.

### Mistake 4 — Enforce DQ rules only in Python

Use PostgreSQL constraints as the final guardrail.

### Mistake 5 — Omit boundaries

Counts without first/last event boundaries are weaker for reconciliation.

### Mistake 6 — Make accounting non-idempotent

Use the documented unique boundary and <code>ON CONFLICT DO NOTHING</code> strategy.

### Mistake 7 — Store event-level telemetry in the ledger

Keep the accounting table operational and scalar-focused.

## 24. Independent Implementation Exercise

Build batch accounting for another event pipeline.

Requirements:

1. Create a batch ledger table.
2. Store first and last event boundaries.
3. Store extracted, valid, invalid, staged, and quarantined counts.
4. Reject negative counters.
5. Enforce <code>extracted = valid + invalid</code>.
6. Enforce <code>staged <= valid</code>.
7. Enforce <code>quarantined <= invalid</code>.
8. Make the batch boundary unique.
9. Implement an immutable batch-outcome model.
10. Implement a writer with no internal commit.
11. Make duplicate accounting idempotent.
12. Record actual persistence counts.
13. Call accounting before checkpoint persistence.
14. Keep all operations in one transaction.
15. Roll back if accounting fails.
16. Test PostgreSQL constraints.
17. Test immutability.
18. Test checkpoint protection.

Simulate:

~~~text
20 extracted
15 valid
5 invalid
14 staged
5 quarantined
~~~

Verify:

~~~text
20 = 15 + 5
14 <= 15
5 <= 5
~~~

Then attempt:

~~~text
20 extracted
15 valid
4 invalid
~~~

Expected result:

~~~text
database rejects the row
~~~

Finally, retry the same batch boundary and verify that no second accounting row is created.

## 25. Validation Checklist

### Ledger

- [ ] <code>public.telemetry_pipeline_batch</code> exists.
- [ ] First and last boundaries are stored.
- [ ] All five counts are stored.
- [ ] Counts cannot be negative.
- [ ] Batch boundary is unique.
- [ ] Arithmetic invariants are enforced.

### Accounting model

- [ ] <code>BatchOutcome</code> is frozen.
- [ ] Boundary fields are present.
- [ ] All five counts are present.

### Writer

- [ ] <code>record_batch_outcome()</code> exists.
- [ ] SQL is parameterized.
- [ ] No internal commit occurs.
- [ ] Duplicate boundaries are ignored.
- [ ] Actual persistence counts are recorded.

### Pipeline ordering

- [ ] Staging occurs before accounting.
- [ ] Quarantine occurs before accounting.
- [ ] Accounting occurs before checkpoint.
- [ ] Accounting is inside the atomic transaction.
- [ ] Accounting failure prevents checkpoint advancement.

### Data quality

- [ ] <code>extracted = valid + invalid</code>.
- [ ] <code>staged <= valid</code>.
- [ ] <code>quarantined <= invalid</code>.
- [ ] PostgreSQL rejects invalid arithmetic.

### Privacy

- [ ] No user identifiers in the ledger.
- [ ] No routes in the ledger.
- [ ] No event properties in the ledger.

### Tests

- [ ] Accounting helper does not commit.
- [ ] BatchOutcome immutability test passes.
- [ ] Accounting failure protects checkpoint.
- [ ] Full 38-test regression passes.

## 26. Scope Boundaries

### Implemented in Stage 14

- batch accounting table;
- immutable <code>BatchOutcome</code>;
- arithmetic DQ constraints;
- boundary uniqueness;
- idempotent accounting insertion;
- transactionally bound accounting;
- accounting and rollback/order tests.

### Deferred to Stage 15

- associating batches with execution runs through <code>run_id</code>.

### Deferred to Stage 16

- dual-mode accounting for <code>NORMAL</code> versus <code>REPLAY</code> processing.

These boundaries are explicitly defined in the supplied source. fileciteturn25file0L146-L152

## 27. What You Should Understand Before the Next Recipe

Stage 14 adds this durable layer:

~~~text
events
  |
  +--> classification
  |       +--> valid
  |       +--> invalid
  |
  +--> persistence
  |       +--> staged
  |       +--> quarantined
  |
  +--> exact boundaries
          |
          v
telemetry_pipeline_batch
~~~

The central lessons are:

- data pipelines need durable batch accounting, not only logs;
- classification and persistence counts represent different facts;
- PostgreSQL can enforce data-quality mathematics;
- first/last boundaries make batch outcomes auditable;
- accounting should be idempotent;
- accounting must commit with the batch;
- accounting must succeed before checkpoint advancement;
- the ledger should avoid duplicating event-level telemetry;
- run tracking and replay accounting are separate later-stage concerns.

The supplied Stage 14 implementation reports 38 passing tests and PostgreSQL 16 validation. fileciteturn25file0L121-L136

## Source Traceability

This recipe is derived from the supplied Stage 14 implementation record:

- Source commit: <code>d97f547</code>
- Subject: <code>feat: add batch outcome and DQ accounting</code>
- Repository: <code>devops</code>
- Overview and objective: fileciteturn25file0L3-L15
- Implementation sequence: fileciteturn25file0L19-L25
- Technical changes: fileciteturn25file0L29-L39
- Design decisions: fileciteturn25file0L44-L48
- Architecture and transaction flow: fileciteturn25file0L52-L77
- Idempotency and integration: fileciteturn25file0L81-L92
- Database schema: fileciteturn25file0L96-L117
- Tests and PostgreSQL validation: fileciteturn25file0L121-L136
- Privacy and scope: fileciteturn25file0L140-L152
