# E46 — Extracting Large Tables Safely

## 1. Problem Recognition

A table containing millions or billions of rows is not simply a larger small-table problem. At scale, memory, source load, network transfer, transaction duration, failure scope, and recovery cost become first-class design constraints.

```text
SOURCE TABLE
     ↓
CONSISTENCY BOUNDARY
     ↓
FINITE RANGE
     ↓
BOUNDED BATCHES
     ↓
CONTROLLED CONCURRENCY
     ↓
STAGING
     ↓
VALIDATION
     ↓
PUBLICATION
```

Recognize the problem when the table cannot fit in application memory, one query runs for a long time, extraction impacts the source, replicas fall behind, network transfer is significant, or the full extraction exceeds the available window.

> **How can I extract a very large table while keeping memory, source load, transaction duration, network usage, failure scope, and recovery cost bounded?**

---

## 2. Concept and Reasoning

Large-table extraction combines several mechanisms from earlier recipes:

- E41/E42 — snapshot consistency
- E43 — connection pooling
- E44 — batching
- E45 — parallelism
- checkpointing
- reconciliation
- staging and publication

The objective is not maximum extraction speed at any cost. It is predictable extraction inside the source system's safe operating envelope.

```text
DEFINE CONSISTENCY
       ↓
CAPTURE FINITE BOUNDARY
       ↓
PARTITION WORK
       ↓
READ BOUNDED BATCHES
       ↓
PERSIST TO STAGING
       ↓
VALIDATE
       ↓
RECONCILE
       ↓
PUBLISH
```

---

## 3. Why Large Tables Are Different

Large tables amplify poor design decisions:

- inefficient indexes
- deep OFFSET pagination
- unbounded result sets
- excessive sorting
- long transactions
- connection leaks
- inefficient serialization
- network bottlenecks
- retry cost
- destination write pressure
- replication lag

A large-table pipeline must make each resource explicit.

---

## 4. Establish the Consistency Contract

Before extracting, decide what source state the result represents.

Possible contracts include:

- one database snapshot
- finite ID range
- finite timestamp window
- CDC position
- source-provided export

For a multi-table baseline, use the snapshot mechanisms covered in E42.

Do not start extracting first and decide consistency later.

---

## 5. Capture a Finite Upper Boundary

A large extraction should normally have a defined completion boundary.

```sql
SELECT MAX(id) AS upper_id
FROM transactions;
```

Then:

```sql
WHERE id > :last_id
  AND id <= :upper_id
```

This prevents new rows from continuously extending a supposedly finite run.

---

## 6. Choose the Extraction Key

Prefer a stable, indexed ordering key.

Candidates can include:

- monotonically increasing ID
- composite `(updated_at, id)` key
- database-native change position
- partition key

A primary key provides identity, but it does not automatically provide ordered extraction progress.

A good extraction key provides deterministic ordering, efficient filtering, stable boundary semantics, and an index-supported access path.

---

## 7. Avoid SELECT *

Large tables often contain columns the extraction does not need.

Prefer explicit columns:

```sql
SELECT
    transaction_id,
    account_id,
    amount,
    currency,
    updated_at
FROM transactions
WHERE transaction_id > :last_id
  AND transaction_id <= :upper_id
ORDER BY transaction_id
LIMIT :batch_size;
```

Column pruning reduces database I/O, network transfer, serialization cost, application memory, and destination write volume.

---

## 8. Index the Extraction Path

For a keyset query, the source needs an efficient access path.

```sql
CREATE INDEX idx_transactions_id
ON transactions (transaction_id);
```

For a composite boundary:

```sql
CREATE INDEX idx_transactions_updated_id
ON transactions (updated_at, transaction_id);
```

Verify the actual plan with EXPLAIN. At large scale, an inefficient plan can dominate the entire extraction window.

---

## 9. Use Bounded Batches

Never materialize an entire large table in application memory unless the volume is known to be safely bounded.

```text
FINITE BOUNDARY
      ↓
10,000 rows
      ↓
10,000 rows
      ↓
10,000 rows
      ↓
...
```

Choose batch size using observed row width, memory, query duration, network transfer, source load, and retry cost.

---

## 10. Bound Client Memory

A logical batch is not enough if the driver or application buffers excessive data.

Use:

- bounded fetch sizes
- streaming/server-side cursors where appropriate
- bounded transformation buffers
- bounded destination batches

Every layer should have a bounded amount of in-flight data.

---

## 11. Server-Side Cursors

For very large result sets, server-side cursors can reduce client-side buffering.

```text
DATABASE
   ↓
CURSOR
   ↓
FETCH 1,000
   ↓
PROCESS
   ↓
FETCH 1,000
   ↓
PROCESS
   ↓
...
```

The connection and transaction may remain active for the cursor lifetime. Therefore combine cursor use with the transaction, connection-pool, timeout, and source-pressure rules from E42 and E43.

---

## 12. Transaction Duration

A large extraction transaction can remain open for a long time.

Under MVCC databases, long-lived snapshots can retain old row versions and affect vacuum or storage behavior.

Monitor:

- transaction age
- source write rate
- vacuum behavior
- table bloat
- replica lag
- extraction duration

Consistency and source health must be designed together.

---

## 13. Use a Replica When Appropriate

A read replica can isolate extraction workload from the primary.

```text
PRIMARY
   ↓
REPLICATION
   ↓
READ REPLICA
   ↓
LARGE EXTRACTION
```

Verify replication lag, replica capacity, snapshot semantics, failover behavior, and whether the replica is an approved source.

Do not call a lagging replica a current representation of the primary without qualification.

---

## 14. Parallelize Only After One Range Is Safe

First prove that one range works correctly:

```text
ONE RANGE
  ↓
ONE BATCH
  ↓
VALIDATE
  ↓
PERSIST
  ↓
CHECKPOINT
```

Then introduce bounded parallelism.

This establishes correctness before concurrency adds another variable.

---

## 15. Partition a Large Table

Possible strategies:

- ID ranges
- timestamp ranges
- native database partitions
- hash partitions where appropriate
- source-provided partitions

Example:

```text
0–10M       → worker A
10M–20M     → worker B
20M–30M     → worker C
30M–40M     → worker D
```

Use explicit inclusive/exclusive rules to prevent overlap.

---

## 16. Partition Skew

Equal numeric ranges do not necessarily contain equal work.

```text
0–10M      → 1 GB
10M–20M    → 20 GB
20M–30M    → 2 GB
```

The second worker becomes the bottleneck.

Possible solutions:

- smaller ranges
- dynamic work claiming
- range splitting
- byte-aware partitioning
- adaptive scheduling

Measure before optimizing.

---

## 17. Network Bandwidth

A large table may be limited by network capacity rather than database execution.

Track:

- bytes extracted
- bytes per second
- compression ratio
- query duration
- serialization time
- destination transfer time

Column pruning and compression can reduce network pressure when appropriate. Do not compress blindly if CPU becomes the new bottleneck.

---

## 18. Destination Throughput and Backpressure

The source may produce data faster than the destination can persist it.

```text
SOURCE
  ↓
FAST EXTRACTION
  ↓
DESTINATION BUFFER
  ↓
SLOW DESTINATION
  ↓
BUFFER GROWTH
```

Bound the destination buffer and reduce extraction concurrency when the destination cannot keep up.

Large-table extraction must balance both source and destination capacity.

---

## 19. Stage Before Publication

For high-risk large extractions, stage the data before publishing it as current.

```text
SOURCE
  ↓
EXTRACTION STAGING
  ↓
VALIDATE
  ↓
RECONCILE
  ↓
PUBLISH
```

This prevents consumers from seeing a partially extracted large table and makes failed runs easier to inspect.

---

## 20. Checkpoint Design

Checkpoint at a durable, deterministic boundary.

```text
last_safe_id = 50,000,000
```

Advance only after corresponding data is durably persisted and verified.

For parallel extraction, checkpoints are normally partition/range-specific rather than one uncontrolled global number.

---

## 21. Large-Table Recovery

A failed run should not automatically require a full restart.

Recover using:

- completed range state
- batch audit
- durable checkpoints
- idempotent destination writes
- source boundary
- reconciliation

Example:

```text
40 ranges
39 complete
1 failed
      ↓
retry only failed range
```

Explicit partition and batch state is what makes this possible.

---

## 22. Idempotent Reprocessing

A worker may persist a batch and crash before its checkpoint is recorded.

The batch may then run again.

The destination must make this safe using:

- primary-key uniqueness
- upsert/merge
- deterministic batch identity
- version-aware writes where required

Do not assume exactly-once execution. Design for safe repeated processing.

---

## 23. Deletes and Historical State

A full table extraction represents the rows visible under its selected source boundary.

If downstream state must remain synchronized afterward, use a change mechanism such as CDC, deletion tracking, or reconciliation.

Do not assume a new full-table extraction is the only way to discover deletes.

---

## 24. Snapshot + CDC Initialization

A common architecture is:

```text
CAPTURE CDC POSITION
        ↓
ESTABLISH CONSISTENT SNAPSHOT
        ↓
EXTRACT LARGE TABLE
        ↓
VALIDATE / PUBLISH
        ↓
APPLY CDC FROM CAPTURED POSITION
        ↓
CATCH UP
        ↓
CONTINUE CDC
```

The exact handoff is database/CDC-system specific. The critical requirement is that no change interval is silently lost.

---

## 25. Reconciliation

Compare control totals against the correct source boundary.

Useful checks:

- row counts
- min/max extraction keys
- control sums
- distinct key counts
- partition counts
- critical-field null counts
- foreign-key relationships
- batch/range completeness

A current source count taken hours later may not be a valid comparison for a historical snapshot.

---

## 26. Large-Table Testing

Test at several scales:

- tiny dataset
- one-batch dataset
- multi-batch dataset
- multi-range dataset
- skewed dataset
- large-record dataset
- concurrent-write dataset
- failure/replay dataset

Measure correctness and resource behavior. Correct rows are not sufficient if the pipeline exhausts source memory, connections, or storage.

---

## 27. Intentional Failure Drills

### Drill 1 — Remove the extraction index

Compare the execution plan and runtime in a controlled environment.

### Drill 2 — Increase batch size aggressively

Observe memory, query duration, source CPU/I/O, and retry cost.

### Drill 3 — Increase parallel workers

Find the point where throughput stops improving and source latency increases.

### Drill 4 — Kill one worker

Verify only the affected range is recovered.

### Drill 5 — Crash after destination persistence

Verify safe replay without duplicate business state.

### Drill 6 — Introduce source writes

Verify that the result follows the selected consistency contract.

### Drill 7 — Slow the destination

Verify bounded buffering and backpressure.

---

## 28. Observability

Pipeline metrics:

```text
large_table_rows_processed_total
large_table_bytes_processed_total
large_table_batches_total
large_table_ranges_total
large_table_range_duration_seconds
large_table_batch_duration_seconds
large_table_retry_total
large_table_checkpoint_lag
large_table_reconciliation_failures_total
```

Source health:

```text
database_active_connections
database_query_duration_seconds
database_cpu
database_io
replication_lag_seconds
transaction_age_seconds
```

Destination health:

```text
destination_write_rate
destination_queue_depth
destination_write_latency
```

---

## 29. Operational Signals

### Source CPU rises with worker count

Concurrency may be too high.

### Query latency rises while throughput stays flat

The database may be saturated.

### Replica lag rises

Extraction is consuming replica capacity faster than replication can keep up.

### Memory rises continuously

Buffers or downstream backpressure may be unbounded.

### One range dominates duration

Partition skew is likely.

### Checkpoint stops moving

A worker may be stuck, a batch may be failing, or persistence may be blocked.

---

## 30. Recovery Runbook

### Source is overloaded

1. Reduce extraction concurrency.
2. Reduce batch size if individual queries are too heavy.
3. Move extraction to an approved replica if appropriate.
4. Optimize the access path.
5. Resume gradually.

### One range is stuck

1. Inspect worker and query state.
2. Check transaction age.
3. Check locks and source load.
4. Determine whether the worker is healthy.
5. Recover the range only when safe.

### Memory is growing

1. Check batch size.
2. Check driver fetch size.
3. Check transformation buffers.
4. Check destination queue depth.
5. Apply backpressure.

### Run is partially completed

Do not publish it as complete. Resume or restart only unresolved ranges according to the snapshot and checkpoint contract.

### Reconciliation fails

Stop publication. Determine whether the problem is missing ranges, duplicate processing, source consistency, destination writes, or an incorrect comparison boundary.

---

## 31. What Not to Do

- Do not run an unbounded SELECT against a billion-row table.
- Do not remove indexes to make extraction simpler.
- Do not use maximum worker count as the performance strategy.
- Do not hold a huge transaction open without monitoring source effects.
- Do not publish partial large-table results as complete.
- Do not advance a checkpoint based only on observed rows.
- Do not ignore replica lag.
- Do not assume row count alone represents workload.
- Do not retry ambiguous writes without idempotency.
- Do not compare a historical snapshot with an unrelated current source state.

---

## 32. Common Mistakes

1. Treating a large table like a small table.
2. Reading the entire table into application memory.
3. Using deep OFFSET pagination blindly.
4. Missing deterministic ordering.
5. Ignoring source indexes.
6. Running too many workers.
7. Ignoring connection-pool limits.
8. Holding snapshots or transactions too long.
9. Ignoring network bandwidth.
10. Ignoring destination throughput.
11. Publishing partial results.
12. Having no durable range/batch state.
13. Assuming a primary key automatically gives ordered progress.
14. Forgetting deletes after the baseline.
15. Having no reconciliation.

---

## 33. Production Tools You Should Know

### 1. PostgreSQL

Learn indexes, EXPLAIN, MVCC, server-side cursors, partitioning, replication, and source resource monitoring.

### 2. Psycopg

Learn controlled result fetching, transactions, cursors, connection pooling, and PostgreSQL-specific extraction behavior from Python.

### 3. Apache Spark

Recognize distributed large-table extraction, partitioned reads, parallel execution, and additional source-load considerations at cluster scale.

These tools implement production mechanisms. You should still be able to explain the underlying large-table extraction design without them.

---

## 34. Definition of Done

You can independently:

- define a large-table consistency contract
- select an extraction key and finite boundary
- design index-supported bounded queries
- control client memory
- choose batch and fetch sizes
- reason about transaction duration
- use replicas safely when appropriate
- partition large tables
- control parallelism
- detect partition skew
- protect the destination with backpressure
- stage and publish safely
- checkpoint durable progress
- recover individual failed ranges
- handle duplicate replay
- reconcile the completed snapshot
- monitor source and destination health
- operate a large-table extraction without treating resources as unlimited

---

## 35. What You Learned

> **Extracting a large table safely is not a single SQL optimization. It is a coordinated design for consistency, bounded reads, memory control, source protection, concurrency, staging, checkpointing, recovery, and reconciliation.**

```text
DEFINE CONSISTENCY
        ↓
DEFINE FINITE BOUNDARY
        ↓
CHOOSE INDEXED ACCESS PATH
        ↓
PARTITION WORK
        ↓
READ BOUNDED BATCHES
        ↓
CONTROL CONCURRENCY
        ↓
STAGE
        ↓
VALIDATE
        ↓
RECONCILE
        ↓
PUBLISH
```

The key question is:

> **Can I extract this table at scale while proving that source load, memory, network usage, progress, failure scope, and final completeness all remain controlled?**