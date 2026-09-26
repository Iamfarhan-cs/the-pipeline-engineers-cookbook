# E34 — Incremental Database Extraction

## 1. Problem Recognition

A full database extraction rereads the complete source. That becomes expensive when a table contains millions of rows but only a small number changed since the previous run.

Incremental extraction means:

```text
previous safe position
        ↓
identify source changes
        ↓
extract only the changes
        ↓
persist them safely
        ↓
advance the position
```

The difficult part is not writing a `WHERE` clause. The difficult part is defining what counts as a change, proving that changes are not skipped, handling updates and deletes, and advancing state only after durable processing.

### Recognize the problem

Use incremental database extraction when:

- the source is a relational database;
- full scans are becoming expensive;
- the source has a reliable change indicator;
- the pipeline runs repeatedly;
- downstream systems need inserts/updates/deletes rather than repeated full snapshots;
- extraction windows must be restartable;
- source changes can occur while extraction is running.

### Core pattern

```text
LOAD LAST SAFE POSITION
        ↓
DEFINE FINITE EXTRACTION WINDOW
        ↓
READ CHANGES
        ↓
VALIDATE
        ↓
PERSIST IDEMPOTENTLY
        ↓
VERIFY
        ↓
ADVANCE POSITION
        ↓
RECONCILE
```

---

## 2. Concept and Reasoning

### What makes database extraction incremental?

A source needs a way to distinguish records that may have changed since a known position.

Common mechanisms:

| Mechanism | Example | Main concern |
|---|---|---|
| Updated timestamp | `updated_at` | Precision, clock/update correctness |
| Increasing ID | `id` | Usually detects inserts, not updates/deletes |
| Version number | `version` | Must be updated reliably |
| Change sequence | database LSN/SCN | Database-specific semantics |
| CDC | transaction log | More complete change capture |
| Trigger/change table | audit table | Trigger correctness and overhead |

An incremental extractor is therefore a **stateful synchronization process**.

### Critical principle

> Never advance the extraction position beyond the changes the pipeline can prove it has durably processed.

---

## 3. Define the Change Contract

Before writing SQL, answer:

1. What identifies a source row?
2. What identifies a change?
3. Does the mechanism detect inserts?
4. Does it detect updates?
5. Does it detect deletes?
6. What is the timestamp/sequence precision?
7. Can values arrive late?
8. Can the change value move backward?
9. Can multiple rows share the same value?
10. Can source transactions commit out of order?
11. How are old rows retained or deleted?

A timestamp column is not automatically a correct change feed.

For example:

```sql
updated_at TIMESTAMPTZ
```

is useful only if every relevant mutation reliably updates it.

---

## 4. Choose the Incremental Boundary

Suppose the last completed watermark is:

```text
2026-09-26 10:00:00
```

A naive query is:

```sql
SELECT *
FROM customers
WHERE updated_at > %s
ORDER BY updated_at;
```

This can miss records when timestamp precision, transaction timing, or late visibility creates boundary ambiguity.

A safer model is a finite window with overlap:

```text
last_safe_watermark
        ↓
overlap window
        ↓
extract through current upper bound
        ↓
deduplicate
        ↓
advance only after complete success
```

For example:

```text
previous watermark = 10:00
lookback = 5 minutes
query from = 09:55
upper bound = 10:30
```

Repeated records are expected. Idempotent persistence makes the overlap safe.

---

## 5. Freeze an Upper Bound

Do not continually query "everything newer than the last watermark" while the extraction is running.

Capture a finite upper bound first.

For a timestamp source:

```sql
SELECT MAX(updated_at)
FROM customers;
```

Then extract:

```sql
SELECT id, customer_name, updated_at
FROM customers
WHERE updated_at > %s
  AND updated_at <= %s
ORDER BY updated_at, id;
```

Conceptually:

```text
lower bound = last safe position
upper bound = captured source position
```

This makes the current run finite.

### Why it matters

Without an upper bound, a continuously changing source can keep producing new records while the extractor is running. The extraction can chase a moving target indefinitely and the checkpoint semantics become unclear.

---

## 6. Timestamp-Based Incremental Extraction

Example source table:

```sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY,
    customer_name TEXT NOT NULL,
    status TEXT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);
```

Basic extraction:

```sql
SELECT
    id,
    customer_name,
    status,
    updated_at
FROM customers
WHERE updated_at > %s
  AND updated_at <= %s
ORDER BY updated_at, id;
```

This requires:

```sql
CREATE INDEX idx_customers_updated_id
ON customers (updated_at, id);
```

The composite index supports both the time boundary and deterministic ordering.

---

## 7. Why `>` vs `>=` Matters

Suppose the last watermark is:

```text
10:00:00.000
```

A row also has:

```text
updated_at = 10:00:00.000
```

Using:

```sql
updated_at > '10:00:00.000'
```

excludes it.

Using:

```sql
updated_at >= '10:00:00.000'
```

includes it again.

In production, overlap plus idempotent processing is often safer than assuming perfect timestamp precision.

The correct operator depends on the source's update and checkpoint semantics.

---

## 8. Timestamp Precision

Do not assume all systems provide microsecond or nanosecond precision.

A source may effectively store:

```text
2026-09-26 10:00:00
```

for many updates.

If ten records share the same timestamp and the extractor checkpoints incorrectly inside that group, records can be skipped.

Prefer a compound position when possible:

```text
(updated_at, id)
```

Query:

```sql
SELECT id, customer_name, updated_at
FROM customers
WHERE (updated_at, id) > (%s, %s)
ORDER BY updated_at, id
LIMIT %s;
```

This creates a deterministic position inside equal timestamps.

---

## 9. The Watermark Must Represent Safe Progress

Suppose a batch contains:

```text
id 101 → updated 10:01
id 102 → updated 10:02
id 103 → updated 10:03
```

If the pipeline persists only 101 and 102, it must not record:

```text
watermark = 10:03
```

The safe state is the position represented by successfully processed data.

For composite positions:

```text
last_safe_position = (10:02, 102)
```

This is the same checkpoint principle used elsewhere in the cookbook, applied specifically to database change extraction.

---

## 10. Python Incremental Extractor

```python
from dataclasses import dataclass
from datetime import datetime, timezone


@dataclass(frozen=True)
class Position:
    updated_at: datetime
    row_id: int


def extract_window(connection, start: Position, end: datetime, batch_size=10_000):
    position = start

    while True:
        with connection.cursor() as cur:
            cur.execute(
                """
                SELECT id, customer_name, status, updated_at
                FROM customers
                WHERE (updated_at, id) > (%s, %s)
                  AND updated_at <= %s
                ORDER BY updated_at, id
                LIMIT %s
                """,
                (
                    position.updated_at,
                    position.row_id,
                    end,
                    batch_size,
                ),
            )
            rows = cur.fetchall()

        if not rows:
            break

        yield rows

        last = rows[-1]
        position = Position(
            updated_at=last[3],
            row_id=last[0],
        )
```

The extractor should persist its position only after the corresponding batch has been safely handled downstream.

---

## 11. Durable Watermark Storage

Do not keep the last position only in a Python variable or local file.

Use durable storage.

```sql
CREATE TABLE database_extraction_checkpoint (
    pipeline_name TEXT NOT NULL,
    source_system TEXT NOT NULL,
    source_table TEXT NOT NULL,
    partition_key TEXT NOT NULL DEFAULT '',
    watermark_time TIMESTAMPTZ,
    watermark_id BIGINT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (
        pipeline_name,
        source_system,
        source_table,
        partition_key
    )
);
```

Load it:

```sql
SELECT watermark_time, watermark_id
FROM database_extraction_checkpoint
WHERE pipeline_name = %s
  AND source_system = %s
  AND source_table = %s
  AND partition_key = %s;
```

---

## 12. Atomic Data and Checkpoint Progress

The ideal boundary is:

```text
READ CHANGES
     ↓
PERSIST DATA
     ↓
COMMIT DATA + CHECKPOINT
```

If source and destination checkpoint storage share a database, this can sometimes be implemented in one transaction.

Example:

```sql
BEGIN;

INSERT INTO staging.customer_changes (...)
VALUES (...);

INSERT INTO database_extraction_checkpoint (
    pipeline_name,
    source_system,
    source_table,
    partition_key,
    watermark_time,
    watermark_id
)
VALUES (...)
ON CONFLICT (
    pipeline_name,
    source_system,
    source_table,
    partition_key
)
DO UPDATE SET
    watermark_time = EXCLUDED.watermark_time,
    watermark_id = EXCLUDED.watermark_id,
    updated_at = now();

COMMIT;
```

If data and checkpoint live in different systems, use idempotent processing and carefully defined recovery semantics instead of pretending the systems have one atomic transaction.

---

## 13. Idempotency Is Required

Incremental extraction commonly uses overlap.

Suppose run 1 extracts through:

```text
10:00:00
```

Run 2 starts from:

```text
09:55:00
```

The same rows may appear again.

The destination must safely handle repeated delivery.

Example:

```sql
INSERT INTO customer_stage (id, customer_name, status, updated_at)
VALUES (%s, %s, %s, %s)
ON CONFLICT (id)
DO UPDATE SET
    customer_name = EXCLUDED.customer_name,
    status = EXCLUDED.status,
    updated_at = EXCLUDED.updated_at
WHERE customer_stage.updated_at <= EXCLUDED.updated_at;
```

This prevents an older overlapping record from overwriting a newer version.

---

## 14. Inserts, Updates, and Deletes

A timestamp-based query can detect inserts and updates if the source updates the timestamp correctly.

It does not automatically detect hard deletes.

Example:

```text
source
id 1
id 2
id 3

id 2 deleted
```

A query for rows with a newer `updated_at` cannot return row 2 if row 2 no longer exists.

Possible solutions:

- soft-delete flag;
- deletion audit table;
- trigger-based change table;
- CDC;
- periodic reconciliation/full scan.

Never claim timestamp incremental extraction provides delete capture unless the source contract supports it.

---

## 15. Soft Deletes

If the source uses:

```sql
is_deleted BOOLEAN NOT NULL DEFAULT FALSE
```

and updates `updated_at` when deletion occurs:

```sql
UPDATE customers
SET is_deleted = TRUE,
    updated_at = now()
WHERE id = 42;
```

then incremental extraction can observe the deletion event.

Downstream can apply:

```sql
UPDATE customer_target
SET is_deleted = TRUE
WHERE id = %s;
```

The important rule is that deletion must participate in the source's change contract.

---

## 16. Hard Delete Capture with a Change Table

A trigger or application process can write changes into a dedicated table.

Example conceptual table:

```sql
CREATE TABLE customer_change_log (
    change_id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    operation TEXT NOT NULL,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Possible events:

```text
INSERT
UPDATE
DELETE
```

The extractor reads the change table instead of trying to infer deletes from the current table.

This introduces another component that must itself be reliable.

---

## 17. Database CDC

When the requirements include reliable inserts, updates, deletes, transaction ordering, and high change volume, database CDC may be more appropriate than polling.

Conceptually:

```text
transaction log
      ↓
CDC reader
      ↓
change events
      ↓
durable processing
      ↓
destination
```

CDC is a separate extraction mechanism and is covered by later recipes. The key decision here is knowing when timestamp polling is insufficient.

---

## 18. Source Transaction Timing

Consider this sequence:

```text
10:00:00  extractor reads MAX(updated_at)
10:00:01  transaction begins
10:00:05  transaction updates row
10:00:10  transaction commits
```

If the upper bound was captured before the commit, the row may not appear in the current window.

That is acceptable only if the next run is guaranteed to include it.

This is one reason an overlap window is useful.

The design must account for **visibility time**, not only the timestamp stored in the row.

---

## 19. Overlap Windows

Suppose:

```text
last safe watermark = 10:00
lookback = 5 minutes
```

Extract from:

```text
09:55 → current upper bound
```

Then deduplicate/upsert downstream.

The overlap protects against:

- timestamp precision issues;
- transaction commit delays;
- late visibility;
- clock differences;
- uncertain boundary behavior.

Overlap does not replace a correct source contract. It is a safety mechanism.

---

## 20. Finite Windows

A run should normally have both:

```text
lower bound
upper bound
```

Example:

```text
lower = 09:55
upper = 10:30
```

At the end of the run, advance the checkpoint only to the safe upper position represented by successfully processed data.

Do not move the checkpoint to wall-clock time merely because the run finished.

Bad:

```python
checkpoint = datetime.now(timezone.utc)
```

Better:

```text
checkpoint = last source position actually processed safely
```

---

## 21. Composite Watermarks

A timestamp alone can be ambiguous.

A stronger logical position can be:

```text
(updated_at, id)
```

Query:

```sql
WHERE (updated_at, id) > (%s, %s)
ORDER BY updated_at, id
```

Checkpoint:

```text
2026-09-26T10:30:00Z, 123456
```

This handles multiple rows sharing the same timestamp.

---

## 22. Monotonicity

A good incremental position should move forward predictably.

Bad source behavior:

```text
row A → updated_at 10:00
row B → updated_at 09:59
```

If a row can legitimately receive an older change value, timestamp extraction can miss it.

Test the source contract.

Do not assume `updated_at` is monotonic globally merely because the column exists.

---

## 23. Backdated Updates

Some applications allow:

```sql
UPDATE customers
SET updated_at = '2026-09-20 10:00:00'
WHERE id = 10;
```

An extractor whose checkpoint is already after that timestamp may never see the update.

Possible solutions:

- enforce source-side timestamp semantics;
- use an application-generated version;
- use a change sequence;
- use CDC;
- use a sufficiently broad reconciliation strategy.

The extraction mechanism must match the source's mutation semantics.

---

## 24. Index Design

An incremental query must have an appropriate access path.

For:

```sql
WHERE (updated_at, id) > (%s, %s)
ORDER BY updated_at, id
```

use:

```sql
CREATE INDEX idx_customers_updated_id
ON customers (updated_at, id);
```

Inspect the plan:

```sql
EXPLAIN
SELECT id, customer_name, status, updated_at
FROM customers
WHERE (updated_at, id) > ('2026-09-26 10:00:00+00', 1000)
ORDER BY updated_at, id
LIMIT 10000;
```

Do not add indexes blindly. Indexes improve extraction but add storage and write overhead to the source.

---

## 25. Batch Processing

Never assume the entire incremental window fits in memory.

Use batches:

```text
window
 ↓
batch 1
 ↓
batch 2
 ↓
batch 3
 ↓
...
```

The position should advance at batch boundaries after durable processing.

For a composite position:

```text
batch last row = (updated_at, id)
```

That becomes the candidate next position.

---

## 26. Multiple Partitions or Tenants

Checkpoint state must match the extraction scope.

Do not use one global watermark when tenants have independent source streams.

Prefer:

```text
pipeline
source
resource
tenant/partition
watermark
```

Example:

```text
Tenant A → 10:00
Tenant B → 09:55
Tenant C → 10:12
```

Each position advances independently.

---

## 27. Concurrent Incremental Workers

Parallel workers can corrupt checkpoint semantics if they update the same position without coordination.

Possible approaches:

- partition by independent key ranges;
- one checkpoint owner per stream;
- transactional row locking;
- optimistic concurrency with version checks.

Example optimistic update:

```sql
UPDATE database_extraction_checkpoint
SET watermark_time = %s,
    watermark_id = %s,
    updated_at = now()
WHERE pipeline_name = %s
  AND source_system = %s
  AND source_table = %s
  AND partition_key = %s
  AND watermark_time = %s
  AND watermark_id = %s;
```

If zero rows are updated, another worker may have moved the position. Treat this as a coordination event, not a success.

---

## 28. Reconciliation

Incremental extraction should not run forever without checking correctness.

Useful reconciliation methods include:

- periodic source/destination counts;
- checksums by partition;
- maximum source update position vs checkpoint;
- sampled record comparison;
- periodic full snapshots;
- delete reconciliation.

Example:

```sql
SELECT MAX(updated_at)
FROM customers;
```

Compare it with the checkpoint.

A large persistent gap may indicate extraction failure or source behavior outside the assumed contract.

---

## 29. Recovery

### Failure before persistence

Do not advance the checkpoint.

Restart from the previous safe position.

### Failure after persistence but before checkpoint

The next run may read the same records again.

This is why idempotent destination writes are required.

```text
persisted
   ↓
crash
   ↓
checkpoint unchanged
   ↓
re-read overlap
   ↓
idempotent upsert
```

This is a normal and safe recovery pattern.

### Corrupt checkpoint

If the checkpoint is invalid:

1. stop incremental advancement;
2. inspect the last successful run;
3. identify the last known safe position;
4. restore that position;
5. replay from an overlap boundary if appropriate;
6. reconcile the destination.

Never invent a new watermark just to make the pipeline start.

---

## 30. Testing

### Unit tests

Test:

- `>` and `>=` boundary behavior;
- equal timestamps;
- composite watermark ordering;
- finite upper bounds;
- empty windows;
- duplicate overlap records;
- checkpoint advancement;
- retry after persistence failure;
- multiple tenants/partitions.

### Integration tests

Use a real database to test:

- inserts after the previous watermark;
- updates after the watermark;
- soft deletes;
- keyset traversal;
- index usage;
- transaction visibility;
- checkpoint persistence;
- restart behavior.

### Edge cases

Test:

```text
0 changes
1 change
many changes at identical timestamp
change exactly at lower boundary
change exactly at upper boundary
late commit
backdated update
hard delete
soft delete
source connection loss
destination failure
checkpoint write failure
```

---

## 31. Intentional Failure Drills

### Drill 1 — Fail after data persistence but before checkpoint

Expected:

```text
checkpoint unchanged
next run reprocesses data safely
no duplicate target state
```

### Drill 2 — Corrupt the checkpoint

Expected:

```text
pipeline refuses unsafe progress
operator restores a known-safe position
```

### Drill 3 — Insert a record after the upper bound

Expected:

```text
record excluded from current run
record included by a later run
```

### Drill 4 — Update a row during extraction

Observe whether the source snapshot/overlap semantics produce the documented result.

### Drill 5 — Delete a row

Verify whether the chosen change mechanism can detect the deletion.

### Drill 6 — Run two workers against one checkpoint

Expected:

```text
checkpoint coordination prevents unsafe regression or overlap corruption
```

---

## 32. Observability

Track:

```text
database_incremental_runs_started_total
database_incremental_runs_succeeded_total
database_incremental_runs_failed_total
records_read_total
records_persisted_total
records_duplicated_total
records_rejected_total
checkpoint_advance_total
checkpoint_write_failures_total
incremental_window_lag_seconds
extraction_duration_seconds
source_query_latency
```

Useful run metadata:

```text
run_id
source table
lower bound
upper bound
starting checkpoint
ending checkpoint
rows read
rows persisted
rows rejected
rows duplicated
status
duration
```

Alert on:

- checkpoint not advancing;
- increasing source-to-checkpoint lag;
- repeated checkpoint failures;
- unexpected zero-change runs;
- unusually large extraction windows;
- source query latency spikes.

A zero-change run is not automatically an error. Interpret it using the source's normal behavior.

---

## 33. Production Runbook

### Before deployment

- [ ] Define the source change contract.
- [ ] Confirm insert/update/delete behavior.
- [ ] Confirm timestamp/version precision.
- [ ] Confirm stable source identity.
- [ ] Add the required source index.
- [ ] Define checkpoint storage.
- [ ] Define overlap policy.
- [ ] Define finite upper-bound behavior.
- [ ] Define destination idempotency.
- [ ] Define reconciliation strategy.

### Before each run

Check:

1. Last safe checkpoint.
2. Source health.
3. Source-to-checkpoint lag.
4. Current upper bound.
5. Expected extraction volume.

### During the run

Monitor:

- rows read;
- rows persisted;
- duplicate rate;
- query latency;
- destination latency;
- checkpoint progress.

### If the checkpoint does not advance

1. Inspect the current run.
2. Check source query results.
3. Check destination writes.
4. Check checkpoint transaction.
5. Determine whether data was persisted without checkpoint advancement.
6. Resume using idempotent processing.

### If source changes cannot be captured reliably

Stop pretending the polling design is complete.

Consider:

- change table;
- CDC;
- periodic full reconciliation;
- source-side contract correction.

### What not to do

- Do not advance the watermark to wall-clock time.
- Do not assume `updated_at` detects deletes.
- Do not use an unindexed timestamp scan on a huge table without source-impact analysis.
- Do not use one checkpoint for independent partitions.
- Do not remove overlap merely to eliminate duplicates.
- Do not treat duplicate delivery as corruption when the design intentionally uses overlap.

---

## 34. Common Mistakes

### Mistake 1 — Treating incremental extraction as `WHERE updated_at > watermark`

The hard part is defining the source change contract and safe position.

### Mistake 2 — No finite upper bound

The run can chase a moving source indefinitely.

### Mistake 3 — Checkpointing before persistence

Failures can permanently skip records.

### Mistake 4 — No overlap

Boundary and visibility timing can create gaps.

### Mistake 5 — No idempotency

Overlap and retries create duplicate effects.

### Mistake 6 — Assuming timestamps capture deletes

Hard deletes disappear from the current table.

### Mistake 7 — Ignoring timestamp precision

Multiple records can share the same value.

### Mistake 8 — Trusting source clocks

Application and database timing can differ.

### Mistake 9 — One global checkpoint for independent streams

A slow partition can block or corrupt progress for another.

### Mistake 10 — No reconciliation

A pipeline can silently drift for months.

### Mistake 11 — Blindly adding an index

Indexes improve reads but add source write and storage cost.

### Mistake 12 — Treating CDC and polling as interchangeable

They have different guarantees and operational mechanics.

---

## 35. Definition of Done

You are done when you can:

- [ ] Explain full vs incremental database extraction.
- [ ] Define a source change contract.
- [ ] Identify whether a source detects inserts, updates, and deletes.
- [ ] Implement timestamp-based extraction.
- [ ] Implement a composite `(timestamp, ID)` position.
- [ ] Define a finite upper bound.
- [ ] Use an overlap window safely.
- [ ] Store checkpoints durably.
- [ ] Advance checkpoints only after durable persistence.
- [ ] Make destination writes idempotent.
- [ ] Handle source transaction visibility.
- [ ] Explain why hard deletes require additional change information.
- [ ] Recognize when CDC is more appropriate.
- [ ] Handle multiple tenants or partitions independently.
- [ ] Test boundary, overlap, late-commit, and failure cases.
- [ ] Recover after persistence succeeds but checkpointing fails.
- [ ] Reconcile incremental progress with the source.
- [ ] Operate the extractor using a production runbook.

---

## 36. What You Learned

Incremental database extraction is not merely a performance optimization. It is a stateful synchronization problem.

You learned to:

1. Define exactly what constitutes a source change.
2. Select an appropriate incremental position.
3. Use finite extraction windows.
4. Handle timestamp precision and boundary ambiguity.
5. Use composite watermarks when necessary.
6. Protect against late visibility with overlap.
7. Persist data before advancing checkpoints.
8. Make repeated delivery safe with idempotent writes.
9. Understand why timestamp polling cannot automatically detect hard deletes.
10. Recognize when change tables or CDC are required.
11. Maintain independent checkpoints for independent streams.
12. Reconcile incremental state rather than trusting the pipeline indefinitely.

The key mental model is:

```text
LAST SAFE POSITION
       ↓
FINITE SOURCE WINDOW
       ↓
READ CHANGES
       ↓
VALIDATE
       ↓
IDEMPOTENT PERSISTENCE
       ↓
VERIFY
       ↓
ADVANCE CHECKPOINT
       ↓
RECONCILE
```

A production Data Engineer should be able to explain not only how to retrieve changed rows, but why the selected change signal is complete enough, how boundary failures are recovered, and what evidence proves that no change was silently skipped.