# E35 — Watermark-Based Extraction

## 1. Problem Recognition

A recurring database extractor needs a durable answer to:

> What source position did the last successful run safely process?

That position is the **watermark**.

A watermark can be a timestamp, sequence number, ID, version, log position, or composite value.

```text
LOAD LAST SAFE WATERMARK
          ↓
DEFINE CURRENT EXTRACTION BOUNDARY
          ↓
READ SOURCE CHANGES
          ↓
VALIDATE
          ↓
PERSIST DURABLY
          ↓
ADVANCE WATERMARK
          ↓
RECONCILE
```

The difficult part is not storing a timestamp or number. The difficult part is proving that the watermark represents safe durable progress.

### Recognize the problem

Use a watermark when:

- full scans are becoming too expensive;
- the source exposes a reliable ordered change signal;
- extraction runs repeatedly;
- the pipeline must resume after failure;
- multiple partitions need independent progress;
- the current run must have a finite source window.

### Critical principle

> A watermark is a claim about durable progress, not merely a value observed in the source.

---

## 2. Watermark Contract

Before implementation, document:

| Property | Question |
|---|---|
| Identity | What identifies the source row/change? |
| Position | What value represents progress? |
| Ordering | Is the value reliably ordered? |
| Precision | How precise is it? |
| Monotonicity | Can it move backward? |
| Inserts | Does it detect inserts? |
| Updates | Does it detect updates? |
| Deletes | Does it detect deletes? |
| Visibility | When does the value become observable? |
| Retention | Can old positions still be queried? |
| Scope | Is the watermark global or partition-specific? |

A column named `updated_at` is not automatically a correct watermark.

---

## 3. Watermark Types

| Type | Example | Main concern |
|---|---|---|
| Timestamp | `updated_at` | Precision, late visibility, backdated changes |
| ID | `id` | Usually detects inserts, not updates/deletes |
| Sequence | `sequence_number` | Must be reliably ordered |
| Version | `version` | Must advance on every relevant mutation |
| CDC position | LSN/SCN | Database-specific semantics |
| Composite | `(updated_at, id)` | Stronger ordering, more state |

The correct watermark is the one whose semantics match the source mutation model.

### Watermark vs current source position

```text
CURRENT SOURCE POSITION
        ≠
LAST SAFE WATERMARK
```

The source can be ahead of the pipeline.

---

## 4. Start, Upper, and Committed Watermarks

A robust run distinguishes three positions.

### Starting watermark

The last safe position from the previous run.

`START = 20:00`

### Upper watermark

A source-derived boundary defining the current finite window.

`END = 20:30`

### Committed watermark

The position actually processed and durably persisted.

`COMMITTED = 20:25`

Safe relationship:

`START ≤ COMMITTED ≤ END`

Do not set the committed watermark to the upper watermark unless the entire window was successfully processed.

---

## 5. Capture a Finite Upper Boundary

For a timestamp source:

```sql
SELECT MAX(updated_at)
FROM customers;
```

Then:

```sql
SELECT id, customer_name, status, updated_at
FROM customers
WHERE updated_at > %s
  AND updated_at <= %s
ORDER BY updated_at, id;
```

A finite upper boundary prevents a continuously changing source from becoming an endless moving target.

Do not use the application clock as a source watermark unless the source contract explicitly defines that behavior.

---

## 6. Durable Watermark Storage

```sql
CREATE TABLE extraction_watermark (
    pipeline_name TEXT NOT NULL,
    source_system TEXT NOT NULL,
    source_object TEXT NOT NULL,
    partition_key TEXT NOT NULL DEFAULT '',

    watermark_time TIMESTAMPTZ,
    watermark_id BIGINT,
    watermark_version BIGINT,

    run_id UUID,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (
        pipeline_name,
        source_system,
        source_object,
        partition_key
    )
);
```

Load it with:

```sql
SELECT watermark_time, watermark_id, watermark_version, run_id
FROM extraction_watermark
WHERE pipeline_name = %s
  AND source_system = %s
  AND source_object = %s
  AND partition_key = %s;
```

If no watermark exists, use an explicit initial-load policy:

- full extraction;
- configured start date;
- minimum source position;
- provider snapshot.

Do not silently choose one.

---

## 7. Composite Watermarks

Timestamp-only state is ambiguous when many rows share the same timestamp.

Example:

```text
101 → 10:00:00
102 → 10:00:00
103 → 10:00:00
```

Use:

`(updated_at, id)`

Query:

```sql
SELECT id, customer_name, updated_at
FROM customers
WHERE (updated_at, id) > (%s, %s)
  AND updated_at <= %s
ORDER BY updated_at, id
LIMIT %s;
```

Index:

```sql
CREATE INDEX idx_customers_watermark
ON customers (updated_at, id);
```

The second component resolves ties deterministically.

---

## 8. Python Watermark Model

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass(frozen=True)
class Watermark:
    updated_at: datetime
    row_id: int


def is_after(candidate: Watermark, current: Watermark) -> bool:
    return (
        candidate.updated_at,
        candidate.row_id,
    ) > (
        current.updated_at,
        current.row_id,
    )
```

Represent the watermark as a domain concept rather than passing unrelated timestamp and ID variables throughout the application.

---

## 9. Incremental Read

```python
def read_batch(connection, watermark, upper_bound, batch_size):
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
                watermark.updated_at,
                watermark.row_id,
                upper_bound,
                batch_size,
            ),
        )
        return cur.fetchall()
```

The query provides a safe lower boundary, finite upper boundary, deterministic ordering, and bounded batch size.

---

## 10. Advance Only After Persistence

Suppose a batch ends at:

`(10:02, 102)`

Correct:

```text
READ
 ↓
VALIDATE
 ↓
PERSIST
 ↓
VERIFY / COMMIT
 ↓
WATERMARK = (10:02, 102)
```

Incorrect:

```text
READ
 ↓
WATERMARK = (10:02, 102)
 ↓
PERSIST
```

If persistence fails after the second sequence, the next run can skip data.

---

## 11. Atomic Data and Watermark Updates

When destination data and watermark storage share a database:

```sql
BEGIN;

INSERT INTO staging.customer_changes (
    id,
    customer_name,
    status,
    updated_at
)
VALUES (...);

INSERT INTO extraction_watermark (
    pipeline_name,
    source_system,
    source_object,
    partition_key,
    watermark_time,
    watermark_id,
    run_id
)
VALUES (...);

COMMIT;
```

This creates one transactional boundary.

When the systems are different, do not pretend they are atomic. Use idempotency, replay, and explicit recovery semantics.

---

## 12. Idempotency

A crash can occur after data persistence but before watermark persistence:

```text
persist data
 ↓
COMMIT
 ↓
crash
 ↓
watermark unchanged
```

The next run may read the same records again. That is acceptable if destination writes are idempotent:

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

Desired failure mode:

```text
duplicate work
    ↓
idempotent persistence
    ↓
correct final state
```

---

## 13. Watermark vs Checkpoint

A checkpoint is a general progress marker:

```text
page 25
file 7
Kafka offset 8123
cursor ABC
```

A watermark represents progress through an ordered source change domain:

```text
updated_at = 20:30
sequence = 99182
version = 5002
```

Therefore:

> A watermark is a specialized form of progress state.

---

## 14. Overlap Windows

When source visibility or timestamp precision is uncertain, use an overlap.

Example:

```text
committed watermark = 20:00
overlap = 5 minutes
effective query start = 19:55
```

Repeated records are expected.

The stored watermark should **not** move backward to 19:55.

Keep these concepts separate:

```text
query start position
        ≠
committed watermark
```

The destination must safely deduplicate or upsert overlapping records.

---

## 15. Late Visibility

Consider:

```text
20:00 → extractor captures upper boundary
20:01 → transaction begins
20:05 → transaction commits
20:06 → row becomes visible
```

That row may belong to a later run.

Overlap can reduce boundary risk, but it cannot fix a fundamentally broken source change contract.

---

## 16. Timestamp Precision

Suppose the source stores timestamps only to seconds. Many records can have:

`10:00:00`

If the pipeline stores only that timestamp and advances incorrectly inside the group, rows can be skipped.

A composite watermark solves this:

`(updated_at, id)`

Use the same ordering in:

- SELECT;
- ORDER BY;
- checkpoint;
- resume logic.

---

## 17. Monotonicity

A watermark should not move backward.

Bad:

```text
current = 20:30
new = 20:15
```

Use an optimistic condition:

```sql
UPDATE extraction_watermark
SET
    watermark_time = %s,
    watermark_id = %s,
    run_id = %s,
    updated_at = now()
WHERE pipeline_name = %s
  AND source_system = %s
  AND source_object = %s
  AND partition_key = %s
  AND (
      watermark_time < %s
      OR (
          watermark_time = %s
          AND watermark_id < %s
      )
  );
```

If zero rows are updated, investigate whether another worker is already ahead or the proposed position is invalid.

---

## 18. Inserts, Updates, and Deletes

A watermark must be evaluated against the actual mutation model.

### Inserts

Usually captured by increasing IDs or timestamps.

### Updates

Captured only if the selected position changes on every relevant update.

### Hard deletes

Usually invisible when polling the current table.

Possible delete mechanisms:

- soft-delete flag;
- deletion/change table;
- trigger-based change log;
- CDC;
- periodic reconciliation.

Never claim a normal `updated_at` watermark captures hard deletes unless the source contract explicitly supports it.

---

## 19. Backdated Changes

This is dangerous:

```sql
UPDATE customers
SET updated_at = '2026-09-20 10:00:00'
WHERE id = 42;
```

If the watermark is already after that value, the change may never be observed.

Solutions include:

- enforce source timestamp semantics;
- use a mutation sequence;
- use a version;
- use a change table;
- use CDC;
- perform periodic reconciliation.

---

## 20. Multiple Partitions

Independent streams need independent watermarks.

Example:

```text
Tenant A → 20:00
Tenant B → 19:42
Tenant C → 20:17
```

Do not collapse them into one global position unless the source guarantees that a shared position is valid.

A useful scope is:

`pipeline + source + object + partition`

---

## 21. Concurrent Workers

For independent partitions:

```text
worker 1 → tenant A
worker 2 → tenant B
worker 3 → tenant C
```

For one shared stream, coordinate watermark ownership using:

- one checkpoint owner;
- range partitioning;
- row locking;
- optimistic concurrency;
- a work queue.

Never let two workers independently overwrite the same watermark.

---

## 22. Watermark Versioning

Watermark representation can evolve.

Example:

```text
Version 1 → timestamp
Version 2 → (timestamp, ID)
```

If the representation changes, migrate it deliberately.

Possible strategies:

- compatible migration;
- dual-read;
- controlled reset;
- backfill.

A watermark is state. State changes require migration discipline.

---

## 23. Validate Watermark State

Before using stored state, validate:

- correct pipeline;
- correct source;
- correct object;
- correct partition;
- supported schema version;
- valid timestamp;
- valid ID;
- expected ordering;
- no impossible future position;
- no malformed representation.

Example:

```python
def validate_watermark(watermark):
    if watermark.row_id < 0:
        raise ValueError("Invalid watermark ID")

    if watermark.updated_at.tzinfo is None:
        raise ValueError("Watermark must be timezone-aware")
```

A corrupt watermark should stop unsafe progress rather than silently resetting to zero.

---

## 24. Recovery

If the watermark is corrupt:

1. Stop incremental advancement.
2. Inspect the last successful run.
3. Find the last verified persisted position.
4. Restore that position.
5. Replay from a safe overlap boundary.
6. Reconcile the destination.

Never do:

```text
corrupt watermark
      ↓
watermark = now()
```

That can permanently skip changes.

---

## 25. When Watermarks Are Not Enough

When requirements include reliable inserts, updates, deletes, ordering, and high change volume, CDC may be more appropriate.

```text
database transaction log
          ↓
CDC reader
          ↓
change events
          ↓
durable processing
          ↓
destination
```

The engineering skill is recognizing when a simple watermark no longer provides sufficient guarantees.

---

## 26. Reconciliation

Record:

```text
run_id
starting watermark
upper watermark
ending watermark
records read
records persisted
records duplicated
records rejected
duration
status
```

Inspect current state:

```sql
SELECT
    pipeline_name,
    source_object,
    partition_key,
    watermark_time,
    watermark_id,
    run_id,
    updated_at
FROM extraction_watermark
ORDER BY updated_at DESC;
```

Compare with source state:

```sql
SELECT MAX(updated_at)
FROM customers;
```

A persistent gap can indicate extraction failure, source lag, checkpoint blockage, incorrect source semantics, or insufficient processing capacity.

---

## 27. Testing

### Unit tests

Test:

- watermark comparison;
- equal timestamps;
- monotonic advancement;
- invalid watermark rejection;
- upper-bound behavior;
- overlap calculation;
- partition-specific state.

### Integration tests

Test:

- insert after watermark;
- update after watermark;
- duplicate timestamp groups;
- persistence + watermark transaction;
- crash before watermark update;
- checkpoint reload;
- concurrent workers;
- source failure;
- destination failure.

### Boundary cases

```text
watermark exactly equals row timestamp
row exactly equals upper bound
row just after upper bound
many rows with identical timestamps
empty source window
first-ever run
corrupt watermark
watermark ahead of source
watermark regression
```

---

## 28. Intentional Failure Drills

### Drill 1 — Crash after persistence

Expected:

```text
data persisted
watermark unchanged
next run safely reprocesses
```

### Drill 2 — Crash before persistence

Expected:

```text
watermark unchanged
data absent
```

### Drill 3 — Corrupt watermark

Expected:

```text
unsafe progress stops
known-safe position is restored
```

### Drill 4 — Attempt watermark regression

Try:

```text
20:30 → 20:15
```

Expected: rejection or safe no-op.

### Drill 5 — Two workers update one watermark

Expected: coordination prevents unsafe overwrite.

### Drill 6 — New row after upper boundary

Expected:

```text
current run → excludes it
later run → captures it
```

---

## 29. Observability

Track:

```text
watermark_value
watermark_age
watermark_lag_seconds
watermark_advance_total
watermark_regression_attempts_total
watermark_write_failures_total
incremental_runs_started_total
incremental_runs_succeeded_total
incremental_runs_failed_total
records_read_total
records_persisted_total
records_duplicated_total
```

Alert on:

- watermark not advancing unexpectedly;
- growing source-to-watermark lag;
- repeated watermark write failures;
- unexpected extraction-window growth;
- regression attempts;
- unusual zero-change runs.

A zero-record run is not automatically an error.

---

## 30. Production Runbook

### Before deployment

- [ ] Document watermark semantics.
- [ ] Verify source ordering.
- [ ] Verify insert/update/delete coverage.
- [ ] Verify precision.
- [ ] Verify source indexes.
- [ ] Create durable watermark storage.
- [ ] Define initial watermark policy.
- [ ] Define overlap policy.
- [ ] Define upper-bound policy.
- [ ] Define recovery procedure.
- [ ] Define reconciliation checks.

### Before each run

Check:

1. Starting watermark.
2. Source maximum position.
3. Source-to-watermark lag.
4. Expected change volume.
5. Previous run status.

### During the run

Monitor:

- rows read;
- rows persisted;
- duplicate rate;
- source query latency;
- destination latency;
- watermark movement.

### If the watermark does not advance

Trace:

```text
source query
    ↓
rows returned?
    ↓
validation
    ↓
destination persistence
    ↓
transaction commit
    ↓
watermark update
```

Do not manually advance the watermark until the missing stage is understood.

### If the watermark is ahead of the destination

Stop processing.

Determine whether data was actually persisted, whether the watermark was incorrectly advanced, and whether replay from a prior safe position is required.

### What not to do

- Do not set the watermark to current time.
- Do not casually move it backward.
- Do not use one global watermark for independent streams.
- Do not assume `updated_at` is a complete change feed.
- Do not advance before persistence.
- Do not discard corrupted watermark state before recovering the last safe value.

---

## 31. Common Mistakes

1. Treating a watermark as an ordinary timestamp variable.
2. Advancing it before persistence.
3. Using application wall-clock time as source progress.
4. Running without a finite upper boundary.
5. Using timestamp-only state when timestamps are not unique.
6. Ignoring late visibility.
7. Assuming deletes are captured.
8. Ignoring idempotency.
9. Sharing one watermark across independent partitions.
10. Allowing stale workers to regress state.
11. Changing watermark representation without migration.
12. Resetting corrupted state to "now".
13. Using a watermark mechanism when CDC is required by the source contract.

---

## 32. Definition of Done

You are done when you can:

- [ ] Explain what a watermark represents.
- [ ] Distinguish source position from safe committed position.
- [ ] Choose timestamp, ID, sequence, version, or composite watermarks.
- [ ] Define a watermark contract.
- [ ] Store watermarks durably.
- [ ] Capture finite upper boundaries.
- [ ] Implement composite ordering.
- [ ] Advance only after durable persistence.
- [ ] Prevent watermark regression.
- [ ] Make destination processing idempotent.
- [ ] Handle overlap safely.
- [ ] Explain why hard deletes require additional mechanisms.
- [ ] Maintain independent watermarks for independent streams.
- [ ] Validate and migrate watermark state.
- [ ] Recover corrupted watermark state.
- [ ] Test crash and concurrency scenarios.
- [ ] Monitor watermark lag and advancement.
- [ ] Reconcile watermark state with source and destination.
- [ ] Operate the extractor using a production runbook.

---

## 33. What You Learned

A watermark is one of the central state concepts in incremental Data Engineering.

You learned to:

1. Treat the watermark as durable progress state.
2. Separate starting, upper, and committed positions.
3. Match the watermark type to source semantics.
4. Use composite positions when a single value is ambiguous.
5. Define finite extraction windows.
6. Advance state only after durable persistence.
7. Use idempotency to make crash recovery safe.
8. Recognize when timestamp polling cannot capture deletes.
9. Coordinate watermark ownership across workers.
10. Validate and migrate watermark state.
11. Recover corrupted state from known-safe evidence.
12. Monitor source-to-watermark lag.

The key mental model is:

```text
LAST SAFE WATERMARK
        ↓
FINITE UPPER BOUND
        ↓
READ SOURCE CHANGES
        ↓
VALIDATE
        ↓
PERSIST
        ↓
VERIFY
        ↓
ADVANCE WATERMARK
        ↓
RECONCILE
```

A production Data Engineer should be able to answer not only "What is our current watermark?" but "Why is this watermark safe, what data does it represent, how was it advanced, and how would we recover if it were wrong?"
