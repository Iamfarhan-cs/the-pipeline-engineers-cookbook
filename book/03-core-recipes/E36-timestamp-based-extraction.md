# E36 — Timestamp-Based Extraction

## Problem Recognition

A source changes continuously, but scanning the complete table on every run is expensive. Timestamp-based extraction uses a reliable source change timestamp such as `updated_at` to process a finite window.

```text
LOAD LAST SAFE TIMESTAMP
        ↓
CAPTURE SOURCE UPPER BOUND
        ↓
QUERY TIMESTAMP WINDOW
        ↓
VALIDATE
        ↓
PERSIST
        ↓
ADVANCE SAFE TIMESTAMP
        ↓
RECONCILE
```

The key question is not whether a timestamp exists. It is whether that timestamp reliably represents every source change the pipeline must capture.

## Concept and Reasoning

### Timestamp contract

Before implementation, document:

| Property | Question |
|---|---|
| Column | Which timestamp represents change? |
| Time zone | UTC or local? |
| Precision | Seconds, milliseconds, microseconds? |
| Insert rule | Is it populated on insert? |
| Update rule | Does every relevant update change it? |
| Mutability | Can it be backdated? |
| Visibility | When does a change become queryable? |
| Deletes | How are hard deletes represented? |
| Ordering | Is timestamp alone sufficient? |
| Index | Is the field indexed? |

`updated_at` is not automatically a valid watermark.

### Use UTC

Use timezone-aware timestamps and normally standardize on UTC.

```python
from datetime import datetime, timezone
now = datetime.now(timezone.utc)
```

PostgreSQL should normally use `TIMESTAMPTZ`.

### Durable state

```sql
CREATE TABLE extraction_timestamp_state (
    pipeline_name TEXT NOT NULL,
    source_system TEXT NOT NULL,
    source_object TEXT NOT NULL,
    partition_key TEXT NOT NULL DEFAULT '',
    last_safe_timestamp TIMESTAMPTZ NOT NULL,
    run_id UUID,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (pipeline_name, source_system, source_object, partition_key)
);
```

The initial position must be explicit: historical minimum, configured cutoff, or current source position.

## Implementation

### 1. Define a finite extraction window

Suppose:

```text
last_safe = 10:00:00
upper_bound = 10:30:00
```

Use:

```sql
SELECT id, customer_name, status, updated_at
FROM customers
WHERE updated_at > %s
  AND updated_at <= %s
ORDER BY updated_at, id;
```

This creates `(10:00:00, 10:30:00]`.

Prefer a source-derived upper boundary when the source contract permits it:

```sql
SELECT MAX(updated_at) FROM customers;
```

Do not substitute application wall-clock time unless that is explicitly part of the source contract.

### 2. Handle boundaries deliberately

Exclusive:

```sql
WHERE updated_at > :last_safe
```

Inclusive:

```sql
WHERE updated_at >= :last_safe
```

Inclusive boundaries intentionally re-read boundary records. That can be safer when precision or visibility is uncertain, provided destination writes are idempotent.

> Duplicate work is generally safer than silently skipping source changes.

### 3. Handle timestamp precision

If many records share `10:00:00`, timestamp-only state cannot tell which record inside that timestamp group was processed.

Use a composite position:

```text
(updated_at, id)
```

```sql
SELECT id, customer_name, updated_at
FROM customers
WHERE (updated_at, id) > (%s, %s)
  AND updated_at <= %s
ORDER BY updated_at, id;
```

```sql
CREATE INDEX idx_customers_updated_at_id
ON customers (updated_at, id);
```

Python model:

```python
from dataclasses import dataclass
from datetime import datetime

@dataclass(frozen=True)
class TimestampPosition:
    timestamp: datetime
    row_id: int
```

The ID is the deterministic tie-breaker.

### 4. Understand transaction visibility

Example:

```text
10:00 → transaction starts
10:01 → updated_at = 10:01
10:05 → transaction commits
10:02 → extractor captures upper boundary
```

The row may not have been visible when the boundary was captured. A later run must be able to discover it.

Possible protections include overlap windows, source transaction/snapshot semantics, change tables, or CDC.

### 5. Use overlap when justified

```text
last safe = 10:00
overlap = 5 minutes
effective start = 09:55
```

The extractor intentionally re-reads part of the previous window.

> The query start can move backward for safety; the durable watermark must not move backward.

### 6. Advance only after persistence

Correct:

```text
READ
 ↓
VALIDATE
 ↓
PERSIST
 ↓
COMMIT
 ↓
ADVANCE TIMESTAMP
```

Incorrect:

```text
READ
 ↓
ADVANCE TIMESTAMP
 ↓
PERSIST
```

The second design can permanently skip data if persistence fails.

### 7. Make destination writes idempotent

```sql
INSERT INTO customer_target (
    id, customer_name, status, updated_at
)
VALUES (%s, %s, %s, %s)
ON CONFLICT (id)
DO UPDATE SET
    customer_name = EXCLUDED.customer_name,
    status = EXCLUDED.status,
    updated_at = EXCLUDED.updated_at
WHERE customer_target.updated_at <= EXCLUDED.updated_at;
```

A replayed older source version must not overwrite newer state.

### 8. Handle late and backdated changes

If a row receives a timestamp older than the current watermark, a strict timestamp filter may never see it.

If applications can do this:

```sql
UPDATE customers
SET updated_at = '2026-01-01'
WHERE id = 42;
```

timestamp order no longer represents mutation order.

Possible solutions:

- reliable commit/change sequence;
- source change table;
- CDC;
- controlled overlap;
- periodic reconciliation;
- source-side enforcement.

Do not compensate for a broken source contract by increasing overlap indefinitely.

### 9. Handle nulls

If `updated_at IS NULL`, define an explicit policy: reject, quarantine, use another documented timestamp, or handle through a separate initial-load path.

Do not silently convert null to an arbitrary epoch value.

### 10. Handle deletes

A current-table query normally cannot discover hard deletes.

```text
row exists → row deleted → no row to query
```

Use soft deletes, a deletion audit table, a change log, CDC, or periodic reconciliation when deletes matter.

### 11. Index and batch large tables

Timestamp-only:

```sql
CREATE INDEX idx_customers_updated_at ON customers (updated_at);
```

Composite:

```sql
CREATE INDEX idx_customers_updated_at_id ON customers (updated_at, id);
```

Verify with:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

Use controlled batches:

```python
while True:
    rows = read_batch(connection, position, upper_bound, batch_size=5_000)
    if not rows:
        break
    persist(rows)
    position = TimestampPosition(
        timestamp=rows[-1].updated_at,
        row_id=rows[-1].id,
    )
```

Only successfully persisted data may advance the position.

### 12. Partition and concurrency

For tenant-partitioned extraction:

```text
tenant A → timestamp A
tenant B → timestamp B
tenant C → timestamp C
```

Store independent positions using `pipeline + source + object + partition`.

Do not allow two workers to overwrite one timestamp state without coordination. Use partition ownership, row locking, optimistic concurrency, or a single checkpoint owner.

## Testing

### Unit tests

Test:

- timestamp comparison;
- inclusive/exclusive boundaries;
- timezone normalization;
- precision;
- composite ordering;
- null handling;
- overlap calculation.

### Integration tests

Test:

- new row after watermark;
- updated row after watermark;
- many rows with identical timestamps;
- row exactly on upper bound;
- row just after upper bound;
- crash after persistence;
- crash before persistence;
- duplicate replay;
- concurrent workers;
- source connection failure.

### Source-contract tests

Verify insert/update timestamp behavior, precision, timezone, backdating behavior, and delete behavior.

## Observability

Track:

```text
starting_timestamp
upper_timestamp
ending_timestamp
timestamp_lag_seconds
records_read
records_persisted
records_rejected
records_duplicated
run_duration_seconds
checkpoint_write_failures
source_query_duration
```

Monitor source-to-watermark lag, rows per minute, extraction-window size, duplicate rate, checkpoint advancement, failed runs, and query latency.

Useful logs include run ID, pipeline, source, partition, timestamps, counts, and status. Never log sensitive source values merely for debugging convenience.

## Intentional Failure

### Drill 1 — Duplicate boundary

Create multiple records with the same timestamp. Verify that no records are skipped.

### Drill 2 — Crash after persistence

Persist a batch, prevent checkpoint advancement, restart, and verify idempotent recovery.

### Drill 3 — Timestamp regression

Have two workers attempt conflicting updates. Verify the older position cannot overwrite the newer one.

### Drill 4 — Late visibility

Delay a source transaction until after the upper boundary. Verify a later run captures it.

### Drill 5 — Backdated update

Change a row to an older timestamp. Verify whether the source contract can lose the mutation.

The purpose is to expose missing guarantees, not hide the failure.

## Recovery

If a run persists data through `10:15` but crashes before advancing the timestamp:

```text
last safe = 10:00
persisted = through 10:15
checkpoint = still 10:00
```

Restart from `10:00`. Reprocessing is expected and safe when destination writes are idempotent.

If the timestamp state is corrupted:

1. stop incremental advancement;
2. inspect the last successful run;
3. recover the last verified safe position;
4. replay from that position or a safe overlap boundary;
5. reconcile source and destination.

Never reset corrupted state to `now`.

## Production Tools You Should Know

### PostgreSQL

Know `TIMESTAMPTZ`, composite indexes, transactions, `EXPLAIN`, locking, and upserts.

### Debezium

Know it as a production CDC option when timestamp polling cannot provide sufficient change guarantees.

### Apache Airflow

Know it as an orchestration platform for scheduling, monitoring, retrying, and backfilling timestamp-based incremental tasks.

The underlying mechanism should remain understandable without these tools.

## Production Runbook

### Before deployment

- [ ] Verify timestamp semantics.
- [ ] Verify timezone.
- [ ] Verify precision.
- [ ] Verify insert/update behavior.
- [ ] Determine delete strategy.
- [ ] Add required indexes.
- [ ] Define initial timestamp.
- [ ] Define upper-bound strategy.
- [ ] Define overlap policy.
- [ ] Make destination writes idempotent.
- [ ] Test crash recovery.
- [ ] Test boundary behavior.

### Before each run

Check:

1. Last safe timestamp.
2. Source maximum timestamp.
3. Source-to-watermark lag.
4. Previous run status.
5. Expected change volume.

### If no records are returned

Check source activity, timezone, precision, boundaries, permissions, previous checkpoint, and source clock behavior. Do not immediately advance the watermark merely because the query returned zero rows.

### If duplicate records appear

Check overlap, inclusive boundaries, retry behavior, destination uniqueness, and source mutation behavior.

### If the watermark is ahead of the destination

Stop the pipeline. Recover the last known safe position and replay.

### What not to do

- Do not set the watermark to current application time.
- Do not ignore timezone semantics.
- Do not assume timestamp precision.
- Do not use timestamp-only state when ties matter.
- Do not advance before persistence.
- Do not assume hard deletes are captured.
- Do not silently reset corrupted state.

## Common Mistakes

1. Treating `updated_at` as automatically reliable.
2. Mixing local and UTC timestamps.
3. Ignoring timestamp precision.
4. Using boundaries without understanding visibility.
5. Advancing before persistence.
6. Failing to use a finite upper boundary.
7. Ignoring duplicate timestamp values.
8. Allowing backdated changes.
9. Assuming every update changes `updated_at`.
10. Assuming timestamp extraction captures deletes.
11. Using one timestamp state for independent partitions.
12. Allowing concurrent workers to regress state.
13. Loading huge windows into memory.
14. Resetting state to `now` after failure.
15. Failing to make duplicate work idempotent.

## Definition of Done

- [ ] Explain when timestamp extraction is appropriate.
- [ ] Document the source timestamp contract.
- [ ] Normalize timestamps to UTC.
- [ ] Understand source precision.
- [ ] Store progress durably.
- [ ] Capture a finite upper boundary.
- [ ] Implement boundary semantics correctly.
- [ ] Use composite timestamp + ID ordering when required.
- [ ] Handle overlap safely.
- [ ] Advance only after durable persistence.
- [ ] Make destination writes idempotent.
- [ ] Explain transaction visibility.
- [ ] Identify late and backdated changes.
- [ ] Explain why deletes need additional mechanisms.
- [ ] Maintain independent state per partition.
- [ ] Prevent concurrent regression.
- [ ] Test crash, boundary, and precision failures.
- [ ] Monitor timestamp lag and checkpoint movement.
- [ ] Operate the pipeline using the runbook.

## What You Learned

Timestamp-based extraction is simple at the SQL level but subtle at production scale. You learned to treat the timestamp as a source contract, use UTC, capture finite windows, understand boundary and precision behavior, account for transaction visibility, use overlap deliberately, advance only after durable persistence, make repeated processing idempotent, recognize delete/backdating limitations, and coordinate state across partitions and workers.

Core mental model:

```text
LAST SAFE TIMESTAMP
        ↓
CAPTURE SOURCE UPPER BOUND
        ↓
DEFINE FINITE WINDOW
        ↓
EXTRACT
        ↓
VALIDATE
        ↓
PERSIST
        ↓
VERIFY
        ↓
ADVANCE TIMESTAMP
        ↓
RECONCILE
```
