# E44 — Database Extraction Batching

## 1. Problem Recognition

Large database extraction should not be one unbounded result. It can exhaust memory, hold connections too long, create huge transfers, delay checkpoints, and make recovery expensive. Row-by-row extraction creates excessive round trips.

Batching creates bounded units:
````
SOURCE → READ BATCH → VALIDATE → PERSIST → CHECKPOINT → NEXT BATCH
````

Recognize the problem when source tables are large, result sets exceed memory, extraction has no durable progress boundary, or row-by-row queries are too slow.

---

# 2. Concept and Reasoning

A batch is a bounded set of source records processed as one extraction unit. Batch size is a resource-control decision, not merely a SQL LIMIT.

Small batches reduce failure scope but increase round trips and checkpoint overhead. Large batches improve throughput potential but increase memory, query duration, network transfer, retry cost, and source pressure.

Batching controls memory, network transfer, query duration, failure scope, checkpoint frequency, and parallel work.

---

# 3. Offset Batching
````sql
SELECT id, amount, updated_at
FROM transactions
ORDER BY id
LIMIT :batch_size
OFFSET :offset;
````

Offset batching is simple but deep offsets can become expensive because the database may need to skip many preceding rows. For large production tables, prefer keyset/range batching when a suitable ordered key exists.

---

# 4. Keyset Batching
````sql
SELECT id, amount, updated_at
FROM transactions
WHERE id > :last_id
ORDER BY id
LIMIT :batch_size;
````

After a successful batch, persist the highest processed ID as the safe boundary.
````text
LAST SAFE ID → READ NEXT RANGE → PERSIST → VERIFY → CHECKPOINT → NEXT RANGE
````

This avoids the deep-offset problem and gives the pipeline an explicit progress boundary.

---

# 5. Deterministic Boundaries

A batch needs stable ordering. If the ordering column is not unique, use a composite boundary.
````sql
SELECT id, updated_at
FROM transactions
WHERE updated_at > :last_ts
   OR (updated_at = :last_ts AND id > :last_id)
ORDER BY updated_at, id
LIMIT :batch_size;
````

The pair `(updated_at, id)` is the progress boundary. This prevents equal timestamps from causing skipped records.

---

# 6. Finite Upper Bounds

For a changing source, freeze a run boundary when the extraction contract requires a finite snapshot-like run.
````sql
SELECT MAX(id) AS upper_id FROM transactions;
````

Then extract only `id > :last_id AND id <= :upper_id`. This prevents newly inserted rows from continuously extending the same run.

---

# 7. Batch Size Trade-Off

Tune batch size using measurements such as rows, bytes, query duration, total duration, memory, database CPU/I/O, and retry frequency.

Do not assume that more rows per batch means higher throughput. The useful target is a predictable resource envelope.

---

# 8. Batch Size vs Fetch Size

Batch size is the logical extraction unit. Fetch size controls how many rows the client retrieves at a time.
````text
logical batch = 10,000 rows
fetch size    = 1,000 rows
````

This lets the pipeline keep a 10,000-row checkpoint boundary while keeping client memory bounded.

---

# 9. Python Implementation
````python
import psycopg


def extract_batches(conn, batch_size, upper_id):
    last_id = 0
    while True:
        rows = conn.execute(
            """
            SELECT id, amount, updated_at
            FROM transactions
            WHERE id > %s AND id <= %s
            ORDER BY id
            LIMIT %s
            """,
            (last_id, upper_id, batch_size),
        ).fetchall()
        if not rows:
            return
        yield rows
        last_id = rows[-1][0]
````

This demonstrates the mechanism. For very large rows or batches, use bounded fetch/streaming techniques rather than unsafe `fetchall()` usage.

---

# 10. Safe Checkpoint Sequence
````text
READ BATCH
   ↓
VALIDATE
   ↓
PERSIST
   ↓
VERIFY
   ↓
ADVANCE CHECKPOINT
   ↓
NEXT BATCH
````

Never checkpoint before persistence. If the process crashes after persistence but before checkpoint advancement, replay the batch safely using idempotent destination writes.

---

# 11. Batch State and Audit
````sql
CREATE TABLE extraction_batches (
    pipeline_name TEXT NOT NULL,
    run_id UUID NOT NULL,
    batch_number BIGINT NOT NULL,
    lower_boundary TEXT,
    upper_boundary TEXT,
    row_count BIGINT,
    status TEXT NOT NULL,
    started_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    PRIMARY KEY (pipeline_name, run_id, batch_number)
);
````

Useful states are `PENDING`, `RUNNING`, `SUCCEEDED`, `FAILED`, and `RETRYING`.

A checkpoint answers **where can I safely continue?** A batch audit answers **what happened in this batch?**

---

# 12. Batch Atomicity

Choose explicitly between batch-atomic transactions, item-level persistence, or stage-then-publish designs. Batching itself does not provide atomicity.

If a batch can partially succeed, the destination must support idempotency and replay or item-level status.

---

# 13. Indexing and Query Plans

Keyset batching depends on suitable indexes.
````sql
CREATE INDEX idx_transactions_updated_id
ON transactions (updated_at, id);
````

Inspect the plan with `EXPLAIN`. Look for sequential scans, unnecessary sorts, excessive rows examined, and unexpected execution cost. Use `EXPLAIN ANALYZE` carefully because it executes the query.

---

# 14. Mutable Rows and Deletes

Batching does not solve source consistency. If rows change during extraction, use an appropriate snapshot, timestamp-window/overlap, or CDC strategy.

A simple `id > last_id` query also cannot discover rows deleted after an earlier extraction. Deletes require soft-delete markers, deletion logs, CDC, tombstones, or reconciliation.

---

# 15. Parallel Batches
````text
0–1M       → worker A
1M–2M      → worker B
2M–3M      → worker C
3M–4M      → worker D
````

Every range needs deterministic ownership. Track range, worker, status, and boundaries so two workers do not accidentally process the same range.

Parallelism increases database connections, CPU, I/O, network traffic, and destination pressure. Batching makes parallelism manageable; it does not make it free.

---

# 16. Batch Skew

Two batches with the same row count can have very different costs if record sizes differ.
````text
10,000 small rows → 5 MB
10,000 large rows → 2 GB
````

Track rows, bytes, duration, database time, and destination time. Row count alone is not always a good workload metric.

---

# 17. Adaptive Batch Size
````text
measure batch
     ↓
too slow? ──yes──→ reduce batch
     │
     no
     ↓
consider bounded increase
````

Define minimum and maximum limits. Never allow an uncontrolled feedback loop to increase batch size indefinitely.

---

# 18. Testing

Test first, middle, and final batches; empty sources; short final batches; exact boundaries; composite boundaries; upper bounds; checkpoint movement; retries; duplicate replay; concurrent source writes; large records; and parallel range ownership.

Run integration tests against a real database. Verify that a crash after persistence but before checkpoint advancement causes safe replay rather than data loss.

---

# 19. Intentional Failure Drills

### Drill 1 — Tiny batch
Measure query count, checkpoint overhead, and throughput.

### Drill 2 — Huge batch
Measure memory, query duration, network transfer, retry cost, and source load.

### Drill 3 — Crash after persistence
Verify the batch is replayed safely and the checkpoint eventually advances correctly.

### Drill 4 — Remove the supporting index
Compare execution plans and runtime.

### Drill 5 — Increase worker count
Observe connection usage, database CPU/I/O, query latency, and throughput.

### Drill 6 — Create large records
Observe why row-count-only sizing can produce unsafe memory or network usage.

---

# 20. Observability
````text
extraction_batches_total
extraction_batches_succeeded_total
extraction_batches_failed_total
extraction_batch_rows
extraction_batch_bytes
extraction_batch_duration_seconds
extraction_batch_query_duration_seconds
extraction_batch_retries_total
````

Every batch log should include pipeline name, run ID, batch ID, worker ID, boundaries, row count, duration, and status. Do not log sensitive row contents merely to make a batch traceable.

Interpret batch metrics together with database CPU, I/O, locks, connection-pool wait, and query latency.

---

# 21. Recovery

For a failed batch: identify the exact boundary; determine whether destination persistence occurred; check the checkpoint; confirm replay is safe; retry the same boundary when valid; reconcile destination state; then advance the checkpoint only after successful persistence.

If the source snapshot or boundary is no longer valid, restart under a new explicitly defined extraction boundary rather than mixing incompatible states.

---

# 22. Production Runbook

## Extraction is slow
Check batch duration, query plan, indexes, source CPU/I/O, locks, network transfer, pool wait, and worker concurrency. Optimize the query or tune batch size experimentally before adding workers.

## Memory is high
Check logical batch size, fetch size, record width, serialization, and destination buffers. Reduce batch/fetch size and stream records where appropriate.

## Too many database queries
Check batch size, retry rate, and accidental nested queries. Increase batch size only after verifying source capacity.

## Duplicates after recovery
Check checkpoint timing, destination idempotency, batch replay, record identity, and concurrent ownership. Never solve duplicates by checkpointing before persistence.

## Batches overlap
Check `>` versus `>=`, composite ordering, worker ownership, and checkpoint updates.

---

# 23. Common Mistakes

1. Using deep `OFFSET` pagination without understanding its cost.
2. Omitting deterministic ordering.
3. Checkpointing before persistence.
4. Assuming row count alone controls memory.
5. Ignoring fetch size.
6. Making every batch an unnecessarily huge transaction.
7. Using timestamps without a tie-breaker.
8. Assuming batching provides source consistency.
9. Ignoring deletes.
10. Increasing workers without measuring database capacity.
11. Using row count as the only workload metric.
12. Keeping no batch audit trail.

---

# 24. Production Tools You Should Know

### 1. PostgreSQL
Learn indexes, `EXPLAIN`, query execution, transactions, cursors, and database resource behavior.

### 2. Psycopg
Learn parameterized queries, cursor behavior, streaming, transactions, and PostgreSQL connection management from Python.

### 3. SQLAlchemy
Recognize its engine/pool abstractions and database execution model.

These tools implement production mechanisms, but the batching mechanism should remain understandable without them.

---

# 25. Definition of Done

You can independently:
- explain why database extraction needs bounded batches
- distinguish batch size from fetch size
- implement offset and keyset batching
- design deterministic and composite boundaries
- define finite upper bounds
- connect batching to checkpointing
- make repeated batches idempotent
- reason about batch atomicity
- size batches using rows, bytes, and duration
- investigate query plans
- parallelize ranges safely
- identify batch skew
- test batch failure and recovery
- operate a large-table extraction safely

---

# 26. What You Learned
> **Database extraction batching turns an unbounded source read into bounded, observable, recoverable units of work. A good batch has a deterministic boundary, controlled resource usage, a persistence boundary, a checkpoint strategy, and a recovery story.**
````text
DEFINE BOUNDARY
      ↓
READ BOUNDED BATCH
      ↓
VALIDATE
      ↓
PERSIST
      ↓
VERIFY
      ↓
AUDIT
      ↓
CHECKPOINT
      ↓
NEXT BATCH
      ↓
RECONCILE
````

The key question is: **Can I process each batch within a predictable resource budget, identify exactly what it contains, safely replay it, and prove that no source progress is lost?**