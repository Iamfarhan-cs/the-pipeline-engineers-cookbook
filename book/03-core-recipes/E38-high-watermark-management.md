# E38 — High-Watermark Management

## 1. Problem Recognition

### The production problem

An extraction pipeline needs to remember how far it has safely processed a source.

A source may expose a position such as:

- an integer ID
- a timestamp
- a sequence number
- a Kafka offset
- a database log sequence number

The dangerous mistake is treating the **largest observed source position** as the **largest safely processed position**.

Example:

```text
Source position observed: 1500
Records actually persisted: through 1490
Recorded watermark: 1500   <- unsafe
```

If the process crashes before records 1491–1500 are persisted, the next run may start after 1500 and skip data.

### How to recognize the problem

You need high-watermark management when you see:

- a pipeline storing `last_processed_id`
- a pipeline storing `last_processed_timestamp`
- incremental queries using `WHERE position > watermark`
- multiple workers updating the same progress state
- gaps between observed and safely processed positions
- unexplained missing records after crashes or deployments

---

# 2. Concept and Reasoning

## 2.1 What is a high-watermark?

A **high-watermark** is the furthest source position that the pipeline claims it has safely completed.

If:

```text
current high-watermark = 1000
```

the pipeline is making a strong operational statement: everything required to process source positions through 1000 has been durably completed according to its correctness rules.

That is different from saying that the source currently contains data up to 1000.

## 2.2 Watermark states

| Term | Meaning |
|---|---|
| Low watermark | Lower boundary of a processing window |
| Current watermark | Position currently stored by the pipeline |
| Observed high watermark | Furthest position currently observed in the source |
| Candidate high watermark | Position the current run proposes to complete |
| Safe high watermark | Furthest position proven safely completed |

The key distinction is:

```text
previous safe = 100
observed source maximum = 150
candidate = 150
persisted successfully through = 143

safe watermark remains 100
```

> **Observation is not completion. Completion is not safe completion until persistence has been verified.**

## 2.3 Monotonicity

Normally:

```text
new_safe_watermark >= old_safe_watermark
```

A normal run must not silently move state backward. A backward move is a recovery, correction, or corruption event and must be explicit and auditable.

---

# 3. High-Watermark State Model

A useful state model is:

```text
OBSERVED
   ↓
CANDIDATE
   ↓
PROCESSING
   ↓
PERSISTED
   ↓
VERIFIED
   ↓
SAFE HIGH-WATERMARK
```

The important boundary is:

```text
PERSISTED + VERIFIED
          ↓
SAFE
```

Not:

```text
OBSERVED → SAFE
```

---

# 4. Watermark Ownership and Concurrency

A high-watermark needs a clear owner. Avoid uncontrolled writes from several workers.

A classic race is:

```text
Worker A reads 100
Worker B reads 100

Worker B completes 150 and writes 150
Worker A completes 120 and writes 120

Final watermark = 120
```

Use optimistic concurrency control so stale workers cannot overwrite newer progress.

A practical state table is:

```sql
CREATE TABLE pipeline_watermark (
    pipeline_name TEXT NOT NULL,
    partition_key TEXT NOT NULL DEFAULT '',
    watermark BIGINT NOT NULL,
    version BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (pipeline_name, partition_key)
);
```

The important fields are pipeline identity, partition identity, safe position, version, and update time.

---

# 5. Capture a Finite Upper Boundary

The source may keep growing while a run is executing. Freeze a boundary before processing the window.

For an ID source:

```sql
SELECT COALESCE(MAX(id), 0) AS upper_bound
FROM source_payments;
```

If the safe watermark is 1000 and the upper bound is 1500, the run owns this finite window:

```text
1000 < id <= 1500
```

Rows arriving after the boundary belong to the next run.

---

# 6. Implement the Mechanism from Scratch

The example below uses PostgreSQL and Python.

## 6.1 Durable state and destination

```sql
CREATE TABLE IF NOT EXISTS pipeline_watermark (
    pipeline_name TEXT NOT NULL,
    partition_key TEXT NOT NULL DEFAULT '',
    watermark BIGINT NOT NULL,
    version BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (pipeline_name, partition_key)
);

CREATE TABLE IF NOT EXISTS destination_payments (
    id BIGINT PRIMARY KEY,
    amount NUMERIC NOT NULL,
    status TEXT NOT NULL
);

INSERT INTO pipeline_watermark (pipeline_name, partition_key, watermark)
VALUES ('payments', '', 0)
ON CONFLICT (pipeline_name, partition_key) DO NOTHING;
```

## 6.2 Read current safe state

```python
from dataclasses import dataclass
import psycopg
from psycopg.rows import dict_row


@dataclass(frozen=True)
class WatermarkState:
    pipeline_name: str
    partition_key: str
    watermark: int
    version: int


def read_watermark(conn, pipeline_name, partition_key=""):
    with conn.cursor(row_factory=dict_row) as cur:
        cur.execute(
            """
            SELECT pipeline_name, partition_key, watermark, version
            FROM pipeline_watermark
            WHERE pipeline_name = %s AND partition_key = %s
            """,
            (pipeline_name, partition_key),
        )
        row = cur.fetchone()

    if row is None:
        raise RuntimeError("Watermark state does not exist")

    return WatermarkState(
        row["pipeline_name"],
        row["partition_key"],
        row["watermark"],
        row["version"],
    )
```

## 6.3 Capture and read the source window

```python
def capture_upper_bound(conn):
    with conn.cursor() as cur:
        cur.execute("SELECT COALESCE(MAX(id), 0) FROM source_payments")
        return cur.fetchone()[0]


def read_batch(conn, lower_bound, upper_bound, batch_size):
    with conn.cursor(row_factory=dict_row) as cur:
        cur.execute(
            """
            SELECT id, amount, status
            FROM source_payments
            WHERE id > %s AND id <= %s
            ORDER BY id
            LIMIT %s
            """,
            (lower_bound, upper_bound, batch_size),
        )
        return cur.fetchall()
```

The ordering is important. Without deterministic ordering, the last record in a batch cannot safely define the next position.

## 6.4 Persist idempotently

```python
def persist_batch(conn, rows):
    with conn.cursor() as cur:
        for row in rows:
            cur.execute(
                """
                INSERT INTO destination_payments (id, amount, status)
                VALUES (%s, %s, %s)
                ON CONFLICT (id)
                DO UPDATE SET
                    amount = EXCLUDED.amount,
                    status = EXCLUDED.status
                """,
                (row["id"], row["amount"], row["status"]),
            )
```

Idempotency matters because a crash can leave data committed while the watermark remains unchanged.

## 6.5 Verify persistence

```python
def verify_batch(conn, rows):
    if not rows:
        return

    ids = [row["id"] for row in rows]

    with conn.cursor() as cur:
        cur.execute(
            """
            SELECT COUNT(*)
            FROM destination_payments
            WHERE id = ANY(%s)
            """,
            (ids,),
        )
        count = cur.fetchone()[0]

    if count != len(set(ids)):
        raise RuntimeError(
            f"Persistence verification failed: expected {len(set(ids))}, got {count}"
        )
```

## 6.6 Compare-and-set advancement

```python
def advance_watermark(
    conn,
    pipeline_name,
    partition_key,
    expected_watermark,
    expected_version,
    new_watermark,
):
    if new_watermark < expected_watermark:
        raise ValueError("Watermark regression is not allowed")

    with conn.cursor() as cur:
        cur.execute(
            """
            UPDATE pipeline_watermark
            SET watermark = %s,
                version = version + 1,
                updated_at = NOW()
            WHERE pipeline_name = %s
              AND partition_key = %s
              AND watermark = %s
              AND version = %s
            """,
            (
                new_watermark,
                pipeline_name,
                partition_key,
                expected_watermark,
                expected_version,
            ),
        )

        if cur.rowcount != 1:
            raise RuntimeError("Watermark update lost the concurrency race")
```

This is compare-and-set: advance only if the state is still exactly the state the worker originally read.

---

# 7. Atomic Data and Watermark Updates

When data and watermark state live in the same database, the strongest design is to coordinate them in one transaction:

```text
BEGIN
  ↓
PERSIST DATA
  ↓
VERIFY / CONSTRAINTS
  ↓
UPDATE WATERMARK
  ↓
COMMIT
```

If the transaction rolls back:

```text
DATA      = not committed
WATERMARK = not advanced
```

A simplified SQL pattern is:

```sql
BEGIN;

INSERT INTO destination_payments (id, amount, status)
VALUES (1500, 42.50, 'completed')
ON CONFLICT (id)
DO UPDATE SET
    amount = EXCLUDED.amount,
    status = EXCLUDED.status;

UPDATE pipeline_watermark
SET watermark = 1500,
    version = version + 1,
    updated_at = NOW()
WHERE pipeline_name = 'payments'
  AND partition_key = ''
  AND watermark = 1400
  AND version = 9;

-- verify that exactly one watermark row was updated
COMMIT;
```

The exact transaction boundary depends on whether the watermark represents a batch, a complete window, or another processing boundary. Define that meaning explicitly.

---

# 8. Watermark Types

The mechanism can manage many source positions.

| Position | Example | Main concern |
|---|---|---|
| ID | 1500 | Gaps, reuse, commit visibility |
| Timestamp | 2026-09-26T10:00:00Z | Precision, time zones, backdated updates |
| Sequence | 982341 | Monotonicity and source semantics |
| Kafka offset | Partition 2, offset 900 | Partition ownership |
| Database LSN | PostgreSQL LSN | CDC transaction boundaries |

The implementation changes, but the safety rule does not: only record a position after the corresponding work is safely complete.

---

# 9. Gaps Do Not Necessarily Mean Failure

A numeric high-watermark is a position, not a row count.

Source IDs may be:

```text
100
101
103
104
110
```

A safe high-watermark of 110 can be valid even though IDs 102 and 105–109 do not exist.

Do not wait forever for numeric continuity unless the source contract explicitly guarantees it.

---

# 10. Global vs Partition High-Watermarks

Independent streams need independent state.

```text
partition A → 500
partition B → 900
partition C → 700
```

Store the positions separately:

```text
(A, 500)
(B, 900)
(C, 700)
```

Do not automatically claim global progress of 900. If the partitions share a comparable coordinate system and all are required for global completion, the global safe boundary may be based on the minimum completed position. Otherwise report partition progress independently.

---

# 11. High-Watermark Lag

A useful operational measurement is the distance between the source position observed and the safe position.

For numeric positions:

```text
lag = observed_position - safe_watermark
```

Example:

```text
observed = 10000
safe     = 9200
lag      = 800
```

For timestamps:

```text
lag = observed_time - safe_time
```

Lag can reveal slow extraction, destination contention, retries, blocked workers, or processing backlog.

---

# 12. Rewind and Rollback

A normal high-watermark moves forward. A backward move must be an explicit recovery operation.

Example:

```text
Current safe watermark = 10000
Verified corruption boundary = 9800
```

A controlled recovery may rewind to 9800 and replay to 10000.

Required controls:

1. operator authorization
2. audit record
3. reason for rewind
4. idempotent downstream processing
5. controlled replay
6. post-recovery reconciliation

Never hide a rewind inside normal checkpoint logic.

---

# 13. Manual Override Controls

Do not make manual correction equivalent to an unrestricted SQL update.

Use:

```text
REQUEST OVERRIDE
      ↓
VALIDATE TARGET POSITION
      ↓
RECORD OPERATOR / REASON
      ↓
APPLY CHANGE
      ↓
RECONCILE
      ↓
RESUME
```

Manual correction should be exceptional and auditable.

---

# 14. Watermark History

Current state tells you where the pipeline is. History tells you how it got there.

```sql
CREATE TABLE IF NOT EXISTS pipeline_watermark_history (
    id BIGSERIAL PRIMARY KEY,
    pipeline_name TEXT NOT NULL,
    partition_key TEXT NOT NULL DEFAULT '',
    previous_watermark BIGINT NOT NULL,
    new_watermark BIGINT NOT NULL,
    reason TEXT NOT NULL,
    run_id TEXT,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Useful reasons include `batch_completed`, `window_completed`, `recovery_rewind`, `manual_override`, and `backfill`.

---

# 15. Corruption and Recovery

If watermark state is missing, null, impossible, or otherwise suspicious, do not guess.

Recovery sequence:

1. inspect watermark history
2. inspect pipeline run records
3. inspect destination data
4. identify the last verified boundary
5. choose a safe rewind point
6. replay from that point
7. reconcile

When uncertain, controlled replay is safer than silently advancing state.

---

# 16. High-Watermark and Idempotency

Consider:

```text
Process 1000–1100
Persist successfully
Crash before watermark update
```

The next run may process 1000–1100 again.

Therefore:

```text
High-watermark
      +
Idempotent destination
      =
Safe crash recovery
```

A watermark does not eliminate the need for idempotent writes.

---

# 17. Deletes

A forward watermark normally tracks progress through source positions. It does not automatically detect hard deletes.

If deletes matter, use an explicit mechanism such as:

- tombstones
- soft-delete timestamps
- CDC
- deletion logs
- reconciliation

High-watermark management solves progress tracking, not complete change detection by itself.

---

# 18. Testing

## 18.1 Essential tests

| Test | Expected result |
|---|---|
| New watermark > current | Advance |
| New watermark = current | No-op |
| New watermark < current | Reject |
| Persistence fails | Do not advance |
| Verification fails | Do not advance |
| Stale worker version | Reject update |
| Empty source window | No unsafe advancement |
| Source grows after upper bound | New rows wait for next run |
| Missing numeric IDs | Do not treat gaps as failures |
| Crash before watermark update | Replay safely |
| Manual rewind | Explicitly audited |
| Corrupt state | Stop and recover deliberately |

## 18.2 Crash-recovery integration test

```text
Initial watermark = 100
Source = 101..110

Persist 101..105
Crash

Watermark remains 100

Restart
  ↓
Replay 101..110
  ↓
Idempotent persistence
  ↓
Watermark = 110
```

The final destination must be correct even though records were replayed.

---

# 19. Intentional Failure Drills

## Drill 1 — Fail before persistence

Break the destination write.

Expected:

```text
Data not committed
Watermark unchanged
```

## Drill 2 — Fail after persistence but before watermark update

Expected:

```text
Data committed
Watermark unchanged
Restart replays safely
```

## Drill 3 — Force stale worker update

Two workers read the same version.

Expected:

```text
One update succeeds
One compare-and-set update fails
```

## Drill 4 — Force regression

Attempt:

```text
1500 → 1200
```

Expected: the update is rejected.

## Drill 5 — Corrupt watermark state

Expected: the pipeline stops and follows a deliberate recovery procedure rather than guessing.

---

# 20. Observability

## Logs

Record:

- pipeline name
- partition
- previous safe watermark
- observed source position
- candidate watermark
- new safe watermark
- run ID
- batch count
- record count
- duration
- outcome
- recovery actions

Do not log secrets or sensitive payloads.

## Metrics

Useful metrics include:

```text
pipeline_watermark
pipeline_observed_position
pipeline_watermark_lag
pipeline_watermark_advance_count
pipeline_watermark_regression_attempts
pipeline_watermark_update_conflicts
pipeline_records_processed
pipeline_replay_count
```

## Alerts

Consider alerts for:

- watermark not advancing for an expected interval
- excessive watermark lag
- repeated compare-and-set conflicts
- repeated advancement failures
- unexpected manual rewinds
- corrupt watermark state
- stalled partition progress

---

# 21. Reconciliation

Periodically compare:

```text
SOURCE POSITION
       vs
SAFE PIPELINE POSITION
       vs
DESTINATION DATA
```

For example:

```text
Source max ID       = 20000
Safe watermark      = 19500
Destination max ID  = 19500
```

This may simply indicate current backlog.

But:

```text
Source max ID       = 20000
Safe watermark      = 20000
Destination max ID  = 19400
```

means the watermark claims more completion than the destination supports. Stop advancement and investigate.

---

# 22. Performance

Watermark state itself is tiny. The expensive part is processing the window behind it.

Use:

- indexed source positions
- bounded batches
- keyset pagination where appropriate
- partition-specific state
- controlled concurrency
- efficient destination writes
- transaction sizing based on workload

Verify the source query plan rather than assuming an index is being used.

---

# 23. Common Mistakes

### 1. Advance on observation

```text
MAX(source.position) → watermark
```

Observation is not completion.

### 2. Advance before persistence

A crash can create skipped data.

### 3. No upper boundary

A run can continuously chase a growing source.

### 4. Blind multi-worker writes

A stale worker can overwrite newer progress.

### 5. No idempotent destination

Crash recovery creates duplicates or conflicts.

### 6. Treat watermark as deletion tracking

A forward position does not reveal hard deletes.

### 7. Allow silent backward movement

Operators cannot distinguish recovery from corruption.

### 8. No history

Production diagnosis becomes guesswork.

---

# 24. Production Tools You Should Know

### 1. PostgreSQL

Useful for durable watermark state, transactional data + watermark updates, compare-and-set updates, history, constraints, and indexing.

### 2. Kafka

Useful for partition-specific offsets, consumer progress, and lag monitoring.

### 3. Debezium

Useful for CDC source positions, connector offsets, database change streams, and restartable change extraction.

These tools implement or support the mechanism; they do not replace understanding it.

---

# 25. Production Runbook

## Symptom: Watermark stopped advancing

Check:

1. Is the source producing new data?
2. Is the worker running?
3. Is extraction failing?
4. Is destination persistence slow?
5. Are transactions blocked?
6. Are compare-and-set updates failing?
7. Is one partition stalled?
8. Is lag increasing?

Do not immediately increase the watermark manually.

## Symptom: Watermark is ahead of destination data

Treat this as a correctness incident.

1. Stop further advancement.
2. Determine the last verified destination boundary.
3. Inspect watermark history.
4. Identify the run that advanced the state.
5. Determine whether missing data can be reconstructed.
6. Rewind to a verified safe position if necessary.
7. Replay idempotently.
8. Reconcile source, watermark, and destination.
9. Record the recovery.

## Symptom: Watermark update conflict

1. Reload current state.
2. Discard stale state.
3. Determine remaining work.
4. Replay safely if required.
5. Continue from current state.

Do not overwrite the newer watermark.

## Symptom: Watermark moved backward unexpectedly

1. Stop the affected pipeline.
2. Inspect history.
3. Identify the writer.
4. Check manual intervention.
5. Check worker concurrency.
6. Validate source and destination state.
7. Restore a verified safe position.
8. Reconcile before resuming.

## What not to do

- Do not set the watermark to the source maximum just to make the pipeline catch up.
- Do not delete watermark state without understanding recovery implications.
- Do not overwrite another worker's progress.
- Do not silently rewind.
- Do not assume a high-watermark detects deletes.
- Do not advance state before durable processing.
- Do not remove idempotency because a watermark exists.

---

# 26. Definition of Done

You understand this recipe when you can independently:

- explain what a high-watermark represents
- distinguish observed position from safe position
- design durable watermark state
- capture a finite upper boundary
- process a bounded source window
- advance the watermark safely
- use compare-and-set concurrency control
- handle multiple partitions
- measure watermark lag
- recover from crashes
- perform an explicit rewind
- audit watermark changes
- test failure paths
- reconcile source position with destination state
- explain why idempotency is still required

---

# 27. What You Learned

> **A high-watermark is the furthest source position the pipeline claims to have safely completed. It may move forward, but it must never move forward on observation alone or backward without an explicit recovery procedure.**

The practical mental model is:

```text
READ CURRENT SAFE WATERMARK
        ↓
OBSERVE SOURCE POSITION
        ↓
DEFINE CANDIDATE HIGH-WATERMARK
        ↓
PROCESS THROUGH CANDIDATE
        ↓
VERIFY DURABLE PERSISTENCE
        ↓
COMPARE-AND-SET ADVANCE
        ↓
RECONCILE LAG
```

The same reasoning applies to database incremental extraction, timestamp windows, ID-based extraction, Kafka offsets, CDC positions, partition progress, replay, recovery, and long-running ETL pipelines.

The key question is always:

> **What source position can this pipeline prove it has safely completed?**
