# E45 — Database Extraction Parallelism

## 1. Problem Recognition

A large database extraction may be correct but too slow when performed by one worker. Parallel extraction divides independent source work across multiple workers.
````text
ONE WORKER
   ↓
SOURCE DATABASE

becomes

RANGE A → WORKER A ─┐
RANGE B → WORKER B ─┼→ SOURCE DATABASE
RANGE C → WORKER C ─┘
````

Parallelism is useful when the source can safely support concurrent reads and the extraction workload can be partitioned without overlap or omission.

Recognize the problem when:
- one worker cannot meet the extraction window
- independent source ranges can be processed concurrently
- source CPU/I/O has available capacity
- database connection capacity allows more readers
- batches have predictable ownership

The key question is:
> **Can the source workload be divided into independent units that can run concurrently without exceeding the database's safe resource budget or corrupting extraction correctness?**

---

# 2. Concept and Reasoning

Parallelism means multiple extraction operations are active at the same time. It does not mean simply creating more threads.

A production design must coordinate four things:
1. **Partitioning** — what each worker owns.
2. **Concurrency limit** — how many workers may run.
3. **Resource budget** — how much database capacity may be consumed.
4. **Correctness boundary** — how overlaps, gaps, retries, and checkpoints are prevented.

Parallelism increases throughput potential, but also increases source pressure.

---

# 3. Parallelism vs Batching

E44 introduced batching. E45 introduces concurrent execution of those batches.
````text
BATCHING:
source → batch 1 → batch 2 → batch 3

PARALLEL BATCHING:
source → batch 1 ─→ worker A
       → batch 2 ─→ worker B
       → batch 3 ─→ worker C
````

Batching controls work-unit size. Parallelism controls how many work units execute simultaneously.

You need both concepts to reason about large database extraction.

---

# 4. Partition the Source

The safest parallelism starts with deterministic partitions.

For a numeric primary key:
````text
0–1,000,000      → worker A
1,000,001–2M     → worker B
2,000,001–3M     → worker C
````

For timestamp ranges:
````text
10:00–11:00 → worker A
11:00–12:00 → worker B
12:00–13:00 → worker C
````

For database partitions, use the database's physical/logical partition boundaries where appropriate.

Every record should have deterministic ownership.

---

# 5. Range Ownership

Do not let workers independently guess which range they should process.

Create explicit work state:
````sql
CREATE TABLE extraction_ranges (
    run_id UUID NOT NULL,
    range_id BIGINT NOT NULL,
    lower_boundary TEXT,
    upper_boundary TEXT,
    status TEXT NOT NULL,
    worker_id TEXT,
    attempts INTEGER NOT NULL DEFAULT 0,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    PRIMARY KEY (run_id, range_id)
);
````

Typical states:
````text
AVAILABLE → CLAIMED → SUCCEEDED
                  ↘ FAILED → RETRYING
````

This makes ownership observable and recoverable.

---

# 6. Claim Work Atomically

Multiple workers must not claim the same range accidentally.

PostgreSQL can coordinate work using row locking.
````sql
SELECT range_id, lower_boundary, upper_boundary
FROM extraction_ranges
WHERE run_id = :run_id
  AND status = 'AVAILABLE'
ORDER BY range_id
FOR UPDATE SKIP LOCKED
LIMIT 1;
````

After claiming the row, update it to `CLAIMED` in the same transaction.

The important mechanism is atomic ownership, not the specific SQL syntax.

---

# 7. Worker Lifecycle
````text
START WORKER
     ↓
CLAIM RANGE
     ↓
READ RANGE IN BATCHES
     ↓
VALIDATE
     ↓
PERSIST
     ↓
VERIFY
     ↓
MARK RANGE COMPLETE
     ↓
CLAIM NEXT RANGE
````

If a worker fails, another worker can recover the range according to its state and retry policy.

---

# 8. Do Not Confuse Workers with Database Connections

Suppose:
````text
workers = 20
pool max = 5
````

Only five workers may hold database connections simultaneously if they share that pool.

If every worker process has its own pool, total capacity changes. Always calculate:
````text
total possible connections
≈ process count × pool maximum
````

This connects E45 directly to E43.

---

# 9. Concurrency Is a Source-Side Budget

Suppose the database has spare CPU but very limited I/O. Increasing extraction workers can still make the source slower.

Possible effects:
- CPU saturation
- I/O saturation
- buffer-cache contention
- lock contention
- replication lag
- longer query latency
- connection exhaustion

Therefore:
> **Concurrency must be bounded by observed source capacity, not by the number of workers available.**

---

# 10. Parallelism Does Not Automatically Improve Throughput

Consider:
````text
1 worker  → 100 seconds
2 workers → 55 seconds
4 workers → 32 seconds
8 workers → 30 seconds
16 workers → 45 seconds
````

The database reached a saturation point.

The goal is not maximum concurrency. The goal is useful throughput within the source's safe operating envelope.

---

# 11. Query Independence

Parallel extraction works best when workers perform independent reads.

Good:
````text
worker A → id 1–1M
worker B → id 1M–2M
worker C → id 2M–3M
````

Risky:
````text
worker A → same range
worker B → same range
worker C → same range
````

Overlapping work increases duplicate processing and database load.

---

# 12. Boundary Correctness

Use explicit inclusive/exclusive rules.

For adjacent ranges, prefer a convention such as:
````text
range A: id >= 1 AND id < 1,000,001
range B: id >= 1,000,001 AND id < 2,000,001
````

This avoids ambiguous ownership at the boundary.

Never mix `>`/`>=` or `<`/`<=` casually between workers.

---

# 13. Gaps Are Not Automatically Errors

Suppose IDs are:
````text
1, 2, 3, 10, 11, 20
````

The ranges may contain gaps because IDs are not necessarily contiguous.

A range should represent a value interval, not an assumption that every integer exists.

Completeness means every source record belongs to exactly one extraction rule, not that every possible ID value exists.

---

# 14. Snapshot Consistency

E42 established that multiple reads may need one source visibility boundary.

Parallel workers make this harder.

If workers independently start transactions:
````text
worker A → snapshot S1
worker B → snapshot S2
worker C → snapshot S3
````

the combined result may not represent one source state.

For snapshot-consistent extraction, use an appropriate database snapshot/export mechanism or another explicit consistency strategy.

Do not claim a globally consistent snapshot merely because every worker uses `REPEATABLE READ` independently.

---

# 15. Parallelism with Exported Snapshots
````text
COORDINATOR
     ↓
ESTABLISH SNAPSHOT
     ↓
SNAPSHOT IDENTIFIER
     ↓
 ┌──────┼──────┐
 ↓      ↓      ↓
A      B      C
 ↓      ↓      ↓
SAME LOGICAL SOURCE STATE
````

When the database supports snapshot export/import, workers can read independent ranges while preserving one logical snapshot boundary.

Snapshot lifecycle and privileges are database-specific. Follow the source database's documented semantics.

---

# 16. Parallelism and Connection Pooling

Each active database worker normally needs a connection while executing its query or transaction.

Therefore:
````text
worker concurrency ↑
        ↓
connection demand ↑
        ↓
pool demand ↑
        ↓
database session count ↑
````

E43's connection budget must be part of E45's design.

---

# 17. Parallelism and Batch Size

Concurrency and batch size multiply resource usage.

For example:
````text
8 workers × 10,000 rows/batch
≈ up to 80,000 rows actively in flight
````

Actual memory depends on row width, fetch behavior, driver buffering, and destination processing.

Do not tune batch size and concurrency independently.

---

# 18. Source Protection

Use a maximum concurrency limit.

Example:
````python
from concurrent.futures import ThreadPoolExecutor

MAX_WORKERS = 4

with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
    futures = [executor.submit(process_range, r) for r in ranges]
    for future in futures:
        future.result()
````

The code is intentionally simple. The important mechanism is the bounded worker count.

In production, combine this with connection-pool limits and database monitoring.

---

# 19. Work Stealing

Static assignment can create skew:
````text
worker A → easy range → finishes early
worker B → huge range → runs much longer
````

A shared work queue lets workers claim another range after finishing.

This improves utilization when ranges have uneven cost.

The trade-off is additional coordination and database state.

---

# 20. Partition Skew

Equal row counts do not guarantee equal runtime.

A range containing large JSON payloads may be much slower than another range containing narrow rows.

Monitor:
- rows
- bytes
- query duration
- processing duration
- database time

If skew is severe, split large ranges further or use dynamic work claiming.

---

# 21. Parallelism and Database Indexes

Each worker's query should use an efficient access path.

For range extraction:
````sql
SELECT id, amount
FROM transactions
WHERE id >= :lower
  AND id < :upper
ORDER BY id;
````

Ensure the boundary column is appropriately indexed.

Multiple workers performing inefficient sequential scans can turn parallelism into a source outage.

---

# 22. Parallelism and Read Replicas

A read replica can isolate extraction load from a primary database.

But it introduces:
- replication lag
- replica capacity limits
- consistency differences
- failover considerations

Record which source instance produced the extraction.

Do not assume that distributing work to a replica eliminates all database load concerns.

---

# 23. Parallelism and CDC

CDC streams already contain partitioning and ordering semantics.

Do not independently parallelize CDC records without respecting:
- partition ownership
- source ordering
- transaction boundaries
- offsets
- event identity

Parallelism for snapshot extraction and parallelism for CDC consumption are related but not identical problems.

---

# 24. Parallel Failure Handling

Suppose:
````text
A → SUCCESS
B → SUCCESS
C → FAILED
D → SUCCESS
````

The run is not complete.

The failed range should remain identifiable and retryable.

Never mark the whole run successful because most workers completed.

A safe lifecycle is:
````text
range failure
   ↓
FAILED
   ↓
retry / recover
   ↓
SUCCEEDED
   ↓
run completion check
````

---

# 25. Worker Crash

A worker can disappear after claiming work.

Use lease/heartbeat or stale-claim detection where necessary.

Example state:
````text
CLAIMED
worker_id = W7
last_heartbeat = T
````

If the worker is silent beyond the configured recovery threshold, the range can be made eligible for recovery according to the run's operational policy.

Do not immediately reclaim work while a slow but healthy worker is still processing it.

---

# 26. Duplicate Processing

A range may be processed twice after a timeout or worker crash.

That is acceptable only if destination writes are idempotent or duplicates are otherwise controlled.

Use:
- stable record identity
- destination uniqueness
- upsert/merge semantics where appropriate
- batch audit
- deterministic range ownership

Parallelism should assume retries can happen.

---

# 27. Global Run Completion

The run should become complete only when every required range reaches a terminal successful state.
````text
ranges:
A SUCCESS
B SUCCESS
C SUCCESS
D SUCCESS

       ↓

RUN = SUCCEEDED
````

If any required range is unresolved:
````text
RUN ≠ SUCCEEDED
````

This prevents silent partial snapshots.

---

# 28. Testing

Test:
- one worker
- multiple workers
- more workers than ranges
- overlapping ranges
- adjacent boundaries
- empty ranges
- missing ranges
- skewed ranges
- worker crashes
- database connection failures
- pool exhaustion
- source slowdown
- duplicate retries
- snapshot consistency
- run completion logic

Integration tests should verify both correctness and source resource behavior.

---

# 29. Intentional Failure Drills

### Drill 1 — Double-claim a range
Remove the atomic claim protection in a test environment. Verify that two workers can process the same range, then restore atomic ownership.

### Drill 2 — Kill a worker
Terminate a worker after claiming a range. Verify stale-work recovery without losing the range.

### Drill 3 — Saturate the pool
Set worker concurrency above pool capacity. Observe bounded database connections and pool wait.

### Drill 4 — Overload the source
Increase concurrency gradually until query latency worsens. Identify the useful operating point.

### Drill 5 — Create skew
Make one range much heavier than the others. Compare static assignment with dynamic work claiming.

### Drill 6 — Break one range
Force one worker to fail repeatedly. Verify the overall run remains incomplete until the range succeeds or the run is explicitly failed.

---

# 30. Observability

Track:
````text
extraction_active_workers
extraction_ranges_total
extraction_ranges_succeeded_total
extraction_ranges_failed_total
extraction_range_duration_seconds
database_connection_pool_wait_seconds
database_active_connections
database_query_duration_seconds
database_cpu_usage
database_io_usage
replica_lag_seconds
````

Every worker log should include run ID, range ID, worker ID, boundaries, attempt number, status, row count, and duration.

Useful relationships:
````text
workers ↑ + query latency ↑
→ source saturation likely

workers ↑ + pool wait ↑
→ connection budget likely limiting

workers ↑ + throughput flat
→ parallelism has reached diminishing returns
````

---

# 31. Recovery

## Failed range
1. Identify range and attempt.
2. Check whether persistence occurred.
3. Verify destination idempotency.
4. Check source snapshot/boundary validity.
5. Retry the range.
6. Reconcile the range.
7. Mark it successful only after verification.

## Worker disappeared
1. Check heartbeat/lease.
2. Confirm the worker is actually dead.
3. Reclaim only after the recovery threshold.
4. Retry the range.
5. Watch for duplicate processing.

## Source overloaded
1. Reduce worker concurrency.
2. Reduce batch size if queries are too heavy.
3. Move extraction to an appropriate replica if available.
4. Optimize indexes/query plans.
5. Resume gradually.

---

# 32. Production Runbook

## Symptom: More workers make extraction slower
Check source CPU, I/O, query latency, locks, connection count, and replica lag. Reduce concurrency and identify the saturation point.

## Symptom: Database connection exhaustion
Check process count × pool maximum, other services, worker count, and external poolers. Reduce concurrency or pool capacity before changing database-wide limits.

## Symptom: Partial run reported as successful
Check range state and run-completion logic. Every required range must reach the required terminal state.

## Symptom: Same range processed twice
Check atomic claiming, worker leases, retries, and destination idempotency.

## Symptom: One worker takes much longer
Check range skew, record sizes, query plans, and source hotspots. Split the range or use dynamic work claiming if appropriate.

## What not to do
- Do not equate worker count with safe database concurrency.
- Do not let every worker create an unrestricted connection.
- Do not use overlapping ranges accidentally.
- Do not publish a partial run.
- Do not claim snapshot consistency when workers used unrelated snapshots.
- Do not increase concurrency while ignoring source saturation.

---

# 33. Common Mistakes

1. Starting one database connection per worker without a global budget.
2. Using overlapping range boundaries.
3. Missing a range because ownership was implicit.
4. Marking the run complete when most workers succeeded.
5. Ignoring worker crashes.
6. Assuming equal row counts mean equal workloads.
7. Ignoring database indexes.
8. Parallelizing an already saturated source.
9. Forgetting snapshot consistency.
10. Treating duplicate processing as impossible.
11. Using static assignment despite severe range skew.
12. Measuring only pipeline throughput and not source health.

---

# 34. Production Tools You Should Know

### 1. PostgreSQL
Learn `FOR UPDATE SKIP LOCKED`, query plans, connection behavior, MVCC, and database resource monitoring.

### 2. PgBouncer
Recognize external connection pooling when many workers/processes need controlled database sessions.

### 3. Apache Airflow
Recognize task-level parallelism, pools, concurrency limits, retries, and dependency management in orchestrated data pipelines.

These tools provide production mechanisms, but the underlying concepts should remain understandable without them.

---

# 35. Definition of Done

You can independently:
- partition a large database extraction into deterministic ranges
- assign ranges safely to concurrent workers
- enforce bounded concurrency
- connect worker count to database connection budgets
- distinguish batching from parallelism
- reason about source CPU/I/O limits
- preserve snapshot consistency when required
- recover abandoned work
- handle duplicate processing
- detect range skew
- determine when parallelism has stopped improving throughput
- prove run completion
- test worker and source failures
- operate parallel extraction without treating the database as an unlimited resource

---

# 36. What You Learned
> **Database extraction parallelism is controlled concurrent work against a finite source resource. The difficult part is not starting many workers; it is partitioning ownership correctly, bounding concurrency, preserving consistency, handling failure, and proving that increased concurrency actually improves useful throughput.**
````text
PARTITION SOURCE
      ↓
ASSIGN OWNERSHIP
      ↓
BOUND CONCURRENCY
      ↓
READ IN BATCHES
      ↓
PERSIST
      ↓
VERIFY
      ↓
RECOVER FAILED RANGES
      ↓
PROVE ALL RANGES COMPLETE
      ↓
RECONCILE
````

The key question is:
> **Can I increase extraction concurrency while proving that every source record has one deterministic owner, every failure is recoverable, and the database remains inside its safe operating budget?**