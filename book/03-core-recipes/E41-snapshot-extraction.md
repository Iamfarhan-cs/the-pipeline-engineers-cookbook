# E41 — Snapshot Extraction

## 1. Problem Recognition

### The production problem

A source system may contain millions or billions of existing records. Before an incremental pipeline can process future changes, the destination often needs a **baseline snapshot** of the source state.

The naive approach is:

```text
SELECT * FROM source_table
       ↓
load everything
```

That becomes dangerous when the table is large, the source is busy, rows change while the extraction runs, or the pipeline needs to restart after failure.

Snapshot extraction is the controlled process of creating a consistent baseline from a source at a defined point in time.

### How to recognize the problem

You need snapshot extraction when:

- building a destination for the first time
- initializing a CDC pipeline
- rebuilding a lost downstream database
- creating a historical baseline
- migrating a large source table
- recovering after CDC retention was exceeded
- validating a downstream copy against the source

The central question is:

> **What exact source state does this snapshot represent?**

---

# 2. Concept and Reasoning

## 2.1 What is a snapshot?

A snapshot is a logical view of source data corresponding to a defined consistency boundary.

Conceptually:

```text
SOURCE
  ↓
SNAPSHOT BOUNDARY
  ↓
READ DATA AS OF THAT BOUNDARY
  ↓
PERSIST BASELINE
```

A snapshot is not simply "whatever rows happened to be returned while the query was running."

For a large table, rows can change while extraction is in progress. Without a consistency model, the resulting dataset may combine values from different moments.

---

# 3. Snapshot vs Full Extraction

These terms are related but not identical.

| Concept | Meaning |
|---|---|
| Full extraction | Read all currently selected source rows |
| Snapshot extraction | Read a defined consistent source state |
| Incremental extraction | Read changes after a prior boundary |
| CDC | Capture source changes from a change stream |

A full extraction can be inconsistent.

A snapshot extraction explicitly defines what consistency means.

Example:

```text
10:00:00 → snapshot boundary established
10:00:01 → row A changes
10:00:02 → row B changes
10:00:03 → extraction reads both
```

The pipeline needs to know whether A and B should reflect their state at 10:00:00, their later state, or another source-defined snapshot boundary.

---

# 4. Snapshot Consistency Models

There are several possible models.

## 4.1 Best-effort snapshot

The extractor reads rows over time without a shared consistency boundary.

Simple, but weaker correctness.

## 4.2 Transaction-consistent snapshot

The database provides a transactionally consistent view.

This is stronger and usually preferable when supported safely.

## 4.3 Exported snapshot

Some databases can create a snapshot identifier that another transaction can use to read the same logical state.

## 4.4 Snapshot + change stream

For CDC initialization:

```text
capture change position
        ↓
create/read consistent snapshot
        ↓
load snapshot
        ↓
consume changes from captured position
```

The snapshot boundary and CDC position must be related correctly or changes can be missed.

---

# 5. The Snapshot Correctness Boundary

The most important rule is:

> **Every snapshot must have an explicit consistency boundary.**

Without one, the pipeline cannot clearly answer which source state it captured.

A useful mental model is:

```text
SOURCE STATE
     ↓
CONSISTENCY BOUNDARY
     ↓
EXTRACTION
     ↓
VALIDATION
     ↓
PERSISTENCE
     ↓
SNAPSHOT COMPLETE
```

Do not confuse extraction completion with snapshot consistency.

---

# 6. Large Snapshot Architecture

For a large table, use bounded extraction rather than loading the entire result into application memory.

```text
SOURCE DATABASE
      ↓
CONSISTENCY BOUNDARY
      ↓
DETERMINISTIC ORDER
      ↓
CONTROLLED BATCHES
      ↓
STAGING
      ↓
VALIDATION
      ↓
PUBLISH SNAPSHOT
```

A common implementation is:

1. establish snapshot semantics
2. determine the extraction boundary
3. read deterministic batches
4. persist batches into staging
5. validate completeness
6. publish the completed snapshot

---

# 7. Deterministic Ordering

A snapshot extractor needs a deterministic traversal strategy.

Avoid relying on:

```sql
SELECT * FROM source_table;
```

with no ordering when batch boundaries matter.

For an indexed ID:

```sql
SELECT id, name, status
FROM source_table
WHERE id > :last_id
ORDER BY id
LIMIT :batch_size;
```

For a composite key:

```sql
SELECT id, tenant_id, status
FROM source_table
WHERE (tenant_id, id) > (:tenant_id, :id)
ORDER BY tenant_id, id
LIMIT :batch_size;
```

The ordering must be compatible with the checkpoint strategy.

---

# 8. Snapshot Upper Boundary

A useful safety mechanism is to establish a finite upper boundary before extraction.

For example:

```sql
SELECT MAX(id)
FROM source_table;
```

Then process:

```text
last_id < id <= upper_id
```

This prevents a continuously growing table from extending the extraction indefinitely.

However, `MAX(id)` alone is **not** a consistency boundary. A row with ID 500 may still be updated while the snapshot is being extracted.

Upper bounds control traversal. They do not automatically provide transaction consistency.

---

# 9. PostgreSQL Snapshot Example

PostgreSQL provides transaction isolation semantics that can be used to obtain a consistent read.

A conceptual approach is:

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT ... FROM source_table;

COMMIT;
```

Within a suitable transaction, the reads observe a consistent database snapshot according to PostgreSQL's MVCC semantics.

For large extractions, keeping one transaction open for the entire operation has operational costs. It can increase transaction lifetime, retain old row versions, and interact with vacuum behavior.

Therefore the snapshot strategy must be designed around the source workload rather than blindly using one enormous transaction.

---

# 10. Snapshot Export Pattern

Some database systems support exporting a snapshot identifier.

Conceptually:

```text
Transaction A
    ↓
CREATE CONSISTENT SNAPSHOT
    ↓
EXPORT SNAPSHOT ID
    ↓
Worker B / C / D
    ↓
READ USING SAME SNAPSHOT
```

This can allow parallel extraction while preserving a common logical source state, when the database supports the required semantics.

The exact implementation is database-specific.

---

# 11. Build a Snapshot Extractor from Scratch

The following example demonstrates the core mechanics using PostgreSQL and Python.

## 11.1 Snapshot state

```sql
CREATE TABLE IF NOT EXISTS snapshot_run (
    snapshot_id UUID PRIMARY KEY,
    pipeline_name TEXT NOT NULL,
    source_table TEXT NOT NULL,
    status TEXT NOT NULL,
    upper_bound BIGINT,
    rows_extracted BIGINT NOT NULL DEFAULT 0,
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
```

The run record lets operators determine whether a snapshot is:

- running
- completed
- failed
- abandoned

---

# 12. Staging the Snapshot

Never make a partially extracted snapshot look like a complete destination table.

Use staging:

```sql
CREATE TABLE IF NOT EXISTS customer_snapshot_stage (
    snapshot_id UUID NOT NULL,
    customer_id BIGINT NOT NULL,
    name TEXT NOT NULL,
    status TEXT NOT NULL,
    PRIMARY KEY (snapshot_id, customer_id)
);
```

The `snapshot_id` isolates one extraction attempt from another.

---

# 13. Batch Extraction

A simple keyset-based reader is:

```python
def read_batch(conn, last_id: int, upper_id: int, batch_size: int):
    with conn.cursor() as cur:
        cur.execute(
            """
            SELECT id, name, status
            FROM customers
            WHERE id > %s
              AND id <= %s
            ORDER BY id
            LIMIT %s
            """,
            (last_id, upper_id, batch_size),
        )
        return cur.fetchall()
```

Then persist each batch:

```python
def write_batch(conn, snapshot_id, rows):
    with conn.cursor() as cur:
        for row in rows:
            cur.execute(
                """
                INSERT INTO customer_snapshot_stage (
                    snapshot_id,
                    customer_id,
                    name,
                    status
                )
                VALUES (%s, %s, %s, %s)
                ON CONFLICT (snapshot_id, customer_id)
                DO UPDATE SET
                    name = EXCLUDED.name,
                    status = EXCLUDED.status
                """,
                (snapshot_id, row[0], row[1], row[2]),
            )
```

The staging table makes retries safe and keeps incomplete snapshots isolated.

---

# 14. Snapshot Progress

Track progress explicitly.

For an ID traversal:

```text
last processed ID = 1000
upper bound        = 10000
```

The next batch begins after 1000.

The progress state should only move after the corresponding batch has been durably persisted.

```text
READ BATCH
   ↓
VALIDATE
   ↓
WRITE STAGING
   ↓
COMMIT
   ↓
ADVANCE PROGRESS
```

This follows the same safe-progress principle used by watermarks and CDC.

---

# 15. Snapshot Completion

A snapshot is complete only when the extractor has proven that its defined boundary has been processed.

For an ID-bounded snapshot:

```text
last_processed_id >= upper_bound
```

is one useful completion condition.

But also verify:

- expected batch accounting
- row counts where meaningful
- duplicate constraints
- validation failures
- source visibility assumptions
- staging integrity

Do not mark the snapshot complete merely because the last query returned an empty page.

---

# 16. Publishing the Snapshot

The destination should not expose a partial snapshot as the current truth.

A common pattern is:

```text
BUILD STAGING SNAPSHOT
        ↓
VALIDATE
        ↓
MARK SNAPSHOT COMPLETE
        ↓
PUBLISH / SWAP
        ↓
CURRENT SNAPSHOT
```

For a table replacement strategy, the exact publication mechanism depends on database capabilities and foreign-key dependencies.

The important invariant is:

> Consumers should see either the previous complete snapshot or the new complete snapshot, not an accidentally exposed half-built one.

---

# 17. Snapshot Identity

Every snapshot run should have a durable identity.

Useful metadata includes:

- snapshot ID
- pipeline name
- source
- source table
- consistency model
- source position or snapshot identifier
- upper bound
- row count
- start time
- completion time
- status
- failure reason

This makes snapshot recovery and auditing much easier.

---

# 18. Snapshot Failure Recovery

Suppose a snapshot processes 4 million rows and crashes at 3 million.

Do not automatically publish it.

The safe state is:

```text
snapshot = FAILED
current destination = previous complete snapshot
```

Recovery options:

### Restart from the beginning

Simple and safest when the snapshot is manageable.

### Resume from a durable batch boundary

Useful when the source consistency model supports it.

### Restart with a new snapshot

Often required if the original consistency boundary can no longer be guaranteed.

Do not resume a snapshot against a changed source state while pretending it still represents the original snapshot boundary.

---

# 19. Snapshot vs Concurrent Source Writes

The source may continue accepting writes during extraction.

That is normal.

The question is whether the snapshot has a defined view.

For example:

```text
snapshot boundary = S

write after S
   ↓
not part of snapshot
```

The subsequent change should be handled by:

- the next snapshot
- CDC
- incremental extraction
- another explicitly defined mechanism

Do not silently mix post-boundary changes into a snapshot that claims to represent S.

---

# 20. Snapshot + CDC Initialization

This is one of the most important production applications.

The desired flow is:

```text
CAPTURE CDC POSITION P
        ↓
CREATE CONSISTENT SNAPSHOT S
        ↓
LOAD SNAPSHOT S
        ↓
READ CDC CHANGES FROM P
        ↓
APPLY CHANGES
        ↓
RECONCILE
        ↓
CONTINUE CDC
```

The implementation must ensure that every change after the snapshot boundary and before the CDC consumer catches up is represented exactly once from the perspective of the final state.

A common safe technique is to allow the CDC stream to contain changes that overlap the snapshot and make the snapshot/CDC application idempotent.

---

# 21. Large Table Memory Management

Never assume the full snapshot fits in memory.

Use:

- server-side cursors where appropriate
- bounded batches
- keyset pagination
- streaming result consumption
- staged writes
- controlled transaction sizes

Avoid:

```python
rows = cursor.fetchall()
```

for a billion-row table.

Memory usage should be bounded independently of total table size.

---

# 22. Source Load Protection

Snapshot extraction can put significant load on the source.

Control:

- batch size
- query frequency
- worker count
- transaction lifetime
- index usage
- replica selection
- extraction schedule

Where appropriate, run snapshots against a read replica.

But remember:

> A replica can have replication lag, so its snapshot represents the replica's state, not necessarily the primary's current state.

Record which source endpoint produced the snapshot and its consistency characteristics.

---

# 23. Index Awareness

A snapshot query should use an access path that scales with the traversal strategy.

For keyset extraction:

```sql
EXPLAIN
SELECT id, name, status
FROM customers
WHERE id > 100000
  AND id <= 200000
ORDER BY id
LIMIT 1000;
```

Verify that the database can efficiently locate the requested range.

Do not assume an index exists merely because the column is frequently queried.

---

# 24. Parallel Snapshot Extraction

Large snapshots can sometimes be partitioned:

```text
Worker A → IDs 1–1,000,000
Worker B → IDs 1,000,001–2,000,000
Worker C → IDs 2,000,001–3,000,000
```

But parallelism introduces correctness questions:

- Do workers share the same snapshot boundary?
- Are ranges non-overlapping?
- Can source rows move between ranges?
- Are all partitions covered?
- Can a worker retry safely?
- How is global completion determined?

Parallelism without a common consistency model can create a fragmented snapshot rather than a consistent one.

---

# 25. Snapshot Completeness

Completeness should be proven using more than one signal where practical.

Possible checks:

```text
source row count
vs
staging row count
```

Also consider:

- min/max keys
- partition counts
- checksums/control totals
- null counts
- duplicate keys
- business totals
- source-specific reconciliation queries

A matching row count alone does not prove the rows are correct.

---

# 26. Snapshot Validation

Validate before publication.

Examples:

```text
required columns present
IDs unique
expected types valid
no impossible statuses
record count within expected range
control totals reconcile
```

A snapshot that loads successfully can still be semantically wrong.

---

# 27. Snapshot Testing

## Essential tests

| Test | Expected result |
|---|---|
| Empty source | Valid empty snapshot |
| Small source | All rows captured |
| Large source | Memory remains bounded |
| Duplicate key | Rejected or handled according to contract |
| Batch failure | Snapshot not published |
| Worker crash | Safe retry/recovery |
| Source grows during run | Post-boundary data handled separately |
| Source updates during run | Snapshot remains consistent under chosen model |
| Partial staging | Not visible as current snapshot |
| Validation failure | Publication blocked |
| Upper-bound completion | Correct completion decision |
| Replica source | Lag documented and controlled |
| Snapshot + CDC | No change gap |

---

# 28. Intentional Failure Drills

### Drill 1 — Kill the extractor halfway

Expected:

```text
snapshot = incomplete/failed
previous published snapshot remains active
```

### Drill 2 — Fail staging write

Expected:

```text
batch transaction rolls back
progress does not advance
```

### Drill 3 — Inject a duplicate key

Expected:

```text
validation/constraint failure
snapshot not published
```

### Drill 4 — Add source writes during extraction

Verify that the snapshot follows its defined consistency boundary rather than accidentally mixing states.

### Drill 5 — Break snapshot/CDC handoff

Verify that the test detects a missing change between the snapshot and CDC stream.

---

# 29. Observability

## Logs

Record:

- snapshot ID
- source endpoint
- source table
- consistency model
- snapshot position/identifier
- upper bound
- batch number
- rows extracted
- duration
- status
- validation result
- publication result

## Metrics

Useful metrics include:

```text
snapshot_rows_extracted_total
snapshot_batches_completed_total
snapshot_duration_seconds
snapshot_rows_per_second
snapshot_failures_total
snapshot_validation_failures_total
snapshot_publish_failures_total
snapshot_source_lag
snapshot_memory_usage
```

## Alerts

Alert on:

- snapshot duration exceeding expected range
- extraction throughput collapse
- repeated batch failures
- source load becoming unsafe
- incomplete snapshots remaining active too long
- snapshot/CDC handoff failures

---

# 30. Reconciliation

After completion, reconcile the snapshot against the source boundary.

Possible checks:

```text
source count at snapshot boundary
        vs
snapshot count
```

and:

```text
source control totals
        vs
snapshot control totals
```

For large data sets, use partition-level reconciliation instead of one enormous comparison query.

If the source is changing continuously, reconciliation must use a meaningful consistency boundary. Comparing a completed snapshot to the source's current state can produce false differences.

---

# 31. Common Mistakes

### 1. Treating full extraction as a snapshot

A full scan without a consistency model may produce a mixed-time dataset.

### 2. Keeping one transaction open indefinitely

This can create source-side operational pressure.

### 3. Using `MAX(id)` as the entire consistency model

An upper ID controls traversal; it does not freeze row contents.

### 4. Publishing partial staging data

Consumers can observe an incomplete baseline.

### 5. No snapshot identity

Operators cannot distinguish one attempt from another.

### 6. Ignoring source growth

The extraction may never finish or may mix post-boundary records.

### 7. No reconciliation

A successful query does not prove a correct snapshot.

### 8. Unsafe snapshot + CDC handoff

Changes can be missed between the two processes.

### 9. Snapshotting from a lagging replica without recording it

The snapshot's actual consistency boundary becomes unclear.

### 10. Loading the entire table into memory

Large sources can exhaust the extractor.

---

# 32. Production Tools You Should Know

### 1. PostgreSQL

Useful for MVCC, transaction isolation, consistent reads, snapshot semantics, keyset extraction, and staging/publishing.

### 2. Debezium

Useful for the production snapshot + CDC initialization pattern and source-position management.

### 3. Apache Spark

Useful when snapshot extraction must be distributed across very large datasets and the source can tolerate the resulting parallel read workload.

These are recognition tools. The snapshot mechanism should remain understandable without them.

---

# 33. Production Runbook

## Symptom: Snapshot is running too slowly

Check:

1. query plan
2. index usage
3. batch size
4. transaction duration
5. source load
6. worker count
7. replica lag
8. destination write throughput

Do not immediately increase parallelism. First identify the bottleneck and source impact.

## Symptom: Snapshot failed halfway

1. Mark the snapshot failed.
2. Keep the previous complete snapshot active.
3. Inspect the failed batch.
4. Decide whether the consistency boundary can still be honored.
5. Resume only if the design explicitly supports it.
6. Otherwise start a new snapshot.
7. Validate before publication.

## Symptom: Snapshot and source counts differ

1. Confirm the source consistency boundary.
2. Confirm the same boundary is used for reconciliation.
3. Check missing partitions/batches.
4. Check duplicate handling.
5. Check validation failures.
6. Check source replica lag if applicable.
7. Reconcile using control totals or key ranges.

Do not compare a historical snapshot directly with a continuously changing current source and assume every difference is an error.

## Symptom: CDC after snapshot shows duplicate changes

This can be expected when snapshot and CDC overlap.

Check:

1. event identity
2. destination idempotency
3. snapshot/CDC boundary
4. final reconciled state

Do not blindly discard the overlapping CDC stream.

## What not to do

- Do not publish an incomplete snapshot.
- Do not use `MAX(id)` as a substitute for transaction consistency.
- Do not leave source transactions open indefinitely without understanding the impact.
- Do not resume an old snapshot after its consistency boundary has become invalid.
- Do not ignore replica lag.
- Do not compare a historical snapshot to an unrelated current source state.
- Do not run unbounded `fetchall()` operations on huge tables.

---

# 34. Definition of Done

You understand this recipe when you can independently:

- explain what a database snapshot represents
- distinguish snapshot extraction from a simple full scan
- choose and document a consistency model
- define a snapshot boundary
- extract a large table in bounded batches
- use deterministic ordering
- persist into isolated staging
- track snapshot identity and progress
- validate completeness
- publish only a complete snapshot
- recover from partial failure
- protect the source from excessive load
- reason about replicas and lag
- design parallel extraction safely
- design snapshot + CDC initialization
- test crash and consistency failures
- reconcile the snapshot against the correct source boundary

---

# 35. What You Learned

> **Snapshot extraction is the controlled creation of a baseline representing a defined source state. The difficult part is not reading every row; it is defining the consistency boundary, preserving it during large-scale extraction, proving completeness, and publishing the result without exposing partial state.**

The practical mental model is:

```text
DEFINE CONSISTENCY BOUNDARY
          ↓
CAPTURE SNAPSHOT POSITION
          ↓
DETERMINISTIC BOUNDED READS
          ↓
STAGE
          ↓
VALIDATE
          ↓
PUBLISH COMPLETE SNAPSHOT
          ↓
RECONCILE
          ↓
CONTINUE INCREMENTAL / CDC PROCESSING
```

The key question is:

> **What exact source state does this snapshot prove it represents?**
