# E47 — Handling Source Database Load

## 1. Problem Recognition

An extraction pipeline is a database workload. Every query consumes CPU, memory, I/O, connections, cache, network bandwidth, and sometimes replication capacity.

```text
MEASURE SOURCE
     ↓
DEFINE SOURCE BUDGET
     ↓
BOUND EXTRACTION
     ↓
OBSERVE
     ↓
ADJUST
     ↓
VERIFY SOURCE HEALTH
```

Recognize the problem when application latency increases during extraction, database CPU/I/O rises, connections are exhausted, lock waits increase, replica lag grows, or extraction throughput becomes unstable.

> **How can I extract the data I need without consuming so much source capacity that the source application's workload becomes unsafe?**

---

## 2. Concept and Reasoning

Source-load management is not about making extraction slow. It is about controlling extraction as one workload among many.

Important controls:
- query shape and indexes
- selected columns
- batch size and fetch size
- worker concurrency
- connection limits
- scheduling
- replica selection
- query timeout
- backpressure
- retry behavior

Concurrency and batch size must be treated as source-capacity controls, not merely performance settings.

---

## 3. Understand the Source Before Extracting

Determine whether the database is a latency-sensitive OLTP primary, reporting source, replica, or shared platform. Identify peak traffic, connection budgets, maintenance windows, backups, CDC consumers, and existing reporting workloads.

Do not optimize the extractor in isolation.

---

## 4. Define a Source Capacity Budget

Create an explicit operating envelope.

```text
application connections = reserved
other services         = reserved
CDC/maintenance        = reserved
extraction connections = bounded
worker concurrency      = bounded
query duration          = bounded
replica lag             = bounded
```

The actual values must come from the environment and database owners. The important mechanism is that the extractor has a defined budget.

---

## 5. Baseline Before Changing Anything

Record normal:
- application latency
- database CPU and I/O
- active connections
- connection-pool wait
- query latency
- lock waits
- transaction age
- replication lag
- cache behavior

Run a controlled extraction and compare the same signals. Without a baseline, source-load claims are difficult to verify.

---

## 6. Query Shape Is the First Control

A bad query can overload a database with one worker.

Prefer explicit, bounded, indexed queries:

```sql
SELECT transaction_id, account_id, amount, updated_at
FROM transactions
WHERE transaction_id > :last_id
  AND transaction_id <= :upper_id
ORDER BY transaction_id
LIMIT :batch_size;
```

Good extraction queries normally have explicit columns, deterministic predicates, indexed boundaries, predictable ordering, and bounded result size.

---

## 7. Index the Extraction Path

Example:

```sql
CREATE INDEX idx_transactions_id
ON transactions (transaction_id);
```

For a composite boundary:

```sql
CREATE INDEX idx_transactions_updated_id
ON transactions (updated_at, transaction_id);
```

Verify with `EXPLAIN`. Do not assume an index is being used simply because it exists.

---

## 8. Column Pruning

Every unnecessary column can increase database I/O, network traffic, serialization cost, memory usage, and destination writes.

Large JSON, text, binary, and document columns deserve particular attention.

Column pruning is often safer than increasing worker count.

---

## 9. Batch Size Controls Per-Query Cost

Small batches create more round trips. Large batches create heavier queries and larger retry scope.

```text
small batch → more queries, smaller per-query cost
large batch → fewer queries, larger per-query cost
```

Choose batch size using measured query duration, rows, bytes, memory, and source load.

---

## 10. Concurrency Controls Aggregate Load

Four acceptable queries can become unacceptable when sixteen run simultaneously.

```text
workers ↑
   ↓
active queries ↑
   ↓
connections / CPU / I/O ↑
```

Bound concurrency explicitly. Do not equate available application threads with safe database concurrency.

---

## 11. Connection Budget

Example capacity model:

```text
database capacity       = 200 connections
application reservation = 140
other services           = 30
maintenance/CDC         = 10
extraction budget        = 20
```

Remember that process count multiplied by pool maximum can create much more connection demand than expected.

---

## 12. Scheduling

Running extraction during lower business traffic can reduce contention, but off-peak hours are not automatically safe. Backups, reporting, maintenance, and other jobs may also run then.

Use measured workload patterns rather than assumptions.

---

## 13. Read Replicas

If supported, a read replica can isolate extraction reads.

```text
PRIMARY
   ↓
REPLICATION
   ↓
READ REPLICA ← extraction
```

Monitor replica CPU, I/O, connection capacity, and replication lag. A replica introduces freshness and consistency considerations.

---

## 14. Replica Lag Is a Load Signal

Heavy extraction can compete with replication replay on a replica.

```text
replication input
       ↓
    replica
       ↑
large extraction reads
```

If lag grows beyond the agreed threshold, reduce or pause extraction according to the runbook.

---

## 15. Query Timeouts

Long-running extraction queries should have explicit limits where supported.

```sql
SET statement_timeout = '60s';
```

A timeout protects the source from an unexpectedly long statement, but retry behavior must account for ambiguous outcomes. Use idempotency and checkpoint rules so retries are safe.

---

## 16. Transaction Duration

Long transactions can retain old row versions under MVCC, delay vacuum, increase bloat, hold locks, and affect replicas.

Monitor transaction age, source write rate, vacuum behavior, bloat, replica lag, and extraction duration.

Do not hold one enormous transaction open merely because the extraction is large. Conversely, do not break a transaction boundary when doing so would violate the required consistency contract.

---

## 17. Snapshot Consistency vs Source Pressure

A consistent snapshot may require a long-lived transaction or snapshot reference. That can create source-side retention pressure.

The design must balance:

```text
required consistency
       +
acceptable source impact
       +
appropriate extraction architecture
```

If the required snapshot is too expensive to hold, consider an approved replica, source export, partitioned snapshot strategy, or another architecture.

---

## 18. Backpressure

Extraction should slow when the source or destination cannot safely accept more work.

```text
SOURCE HEALTH DEGRADES
        ↓
REDUCE CONCURRENCY
        ↓
REDUCE IN-FLIGHT WORK
        ↓
SOURCE RECOVERS
        ↓
RESUME CONTROLLED EXTRACTION
```

Backpressure prevents temporary capacity problems from becoming sustained incidents.

---

## 19. Adaptive Concurrency

A controlled extractor can adjust worker count using measured signals.

```text
source healthy → bounded increase → observe
source pressure → reduce concurrency → recover
```

Keep minimum and maximum limits. Use stable observation windows so the controller does not oscillate rapidly.

---

## 20. Rate-Limit Database Work

Database extraction can be limited by concurrent queries, batches per second, rows per second, bytes per second, or active connections.

Concurrency is usually the simplest first control because query cost varies widely.

---

## 21. Source-Aware Retries

Workers should not immediately retry against an overloaded database.

```text
QUERY SLOWS / TIMEOUTS
       ↓
CLASSIFY FAILURE
       ↓
BACKOFF
       ↓
REDUCE OR HOLD CONCURRENCY
       ↓
RETRY
```

Otherwise the extractor can create a retry storm against an already overloaded source.

---

## 22. Locks and Contention

Monitor lock waits, blocked queries, transaction age, hot tables, and index contention.

A query waiting behind other work is not necessarily a reason to increase concurrency.

---

## 23. Query Plan Regression

A previously fast extraction query can become expensive after statistics changes, data growth, index changes, schema changes, distribution changes, or database upgrades.

Monitor execution plans and duration over time. Do not treat a previously fast query as permanently safe.

---

## 24. Maintenance Interaction

Extraction can compete with vacuum, analyze, backups, index maintenance, replication, CDC capture, and reporting queries.

Coordinate large extraction jobs with these workloads. A database has one finite resource pool even when different teams own the jobs.

---

## 25. Source-Protection Tuning Sequence

Use this as a diagnostic sequence:

```text
QUERY SHAPE
    ↓
INDEX / ACCESS PATH
    ↓
COLUMN PRUNING
    ↓
BATCH SIZE
    ↓
CONCURRENCY
    ↓
SCHEDULING / REPLICA
    ↓
ARCHITECTURAL CHANGE
```

This is not a mandatory universal order. It is a practical way to avoid jumping directly to more workers or more database capacity before fixing an inefficient query.

---

## 26. Testing

### Unit tests
Test concurrency limits, batch limits, timeout classification, backoff decisions, source-health thresholds, pause/resume behavior, and checkpoint safety.

### Integration tests
Use a realistic database workload and verify that application queries remain acceptable, connection limits are respected, extraction completes, failures retry safely, and source-pressure thresholds trigger controls.

---

## 27. Intentional Failure Drills

### Drill 1 — Increase concurrency
Gradually increase workers until source latency rises. Record the useful operating range.

### Drill 2 — Increase batch size
Observe query duration, CPU, I/O, memory, and throughput.

### Drill 3 — Saturate the pool
Verify extraction waits or backs off instead of creating uncontrolled connections.

### Drill 4 — Introduce query latency
Force slow queries in a controlled environment and verify timeout and retry classification.

### Drill 5 — Simulate replica lag
Verify extraction slows or stops according to policy.

### Drill 6 — Slow the destination
Verify downstream backpressure eventually reduces source extraction pressure.

---

## 28. Observability

### Extraction metrics
```text
extraction_active_workers
extraction_batches_total
extraction_batch_duration_seconds
extraction_rows_per_second
extraction_bytes_per_second
extraction_retry_total
extraction_timeout_total
extraction_backpressure_events_total
```

### Database metrics
```text
database_cpu_usage
database_io_usage
database_active_connections
database_connection_pool_wait
database_query_latency
database_lock_wait
database_transaction_age
database_replication_lag
```

Interpret them together. A fast extractor is not successful if source health is deteriorating.

---

## 29. Production Recovery

### Source database is overloaded
1. Reduce extraction concurrency.
2. Stop launching new ranges if necessary.
3. Reduce batch size when individual queries are heavy.
4. Move to an approved replica if available.
5. Wait for source health to recover.
6. Resume gradually.

### Connection exhaustion
1. Identify extraction connections.
2. Check process count × pool maximum.
3. Reduce worker count.
4. Release idle or stale connections.
5. Verify application capacity before resuming.

### Replica lag
1. Pause or reduce extraction.
2. Confirm replication recovers.
3. Resume below the observed safe concurrency.

### Query timeouts
1. Inspect the query plan.
2. Determine whether source load or query inefficiency caused the timeout.
3. Check whether durable destination writes occurred.
4. Retry only when the operation is safe to repeat.

---

## 30. Production Runbook

### Application latency increases during extraction
Check CPU, I/O, active connections, locks, query latency, and extraction concurrency. Reduce workers, reduce heavy batch size, move reads to an approved replica, reschedule, or optimize the query.

### Extraction is fast but source is unhealthy
Treat source health as a failure signal. Reduce extraction pressure.

### Replica lag keeps growing
Pause or throttle extraction and investigate replica capacity.

### Connection-pool wait is high
Check worker count, pool size, transaction duration, and connection leaks.

### Queries suddenly become slow
Compare plans, source load, statistics, indexes, and data distribution.

### What not to do
Do not increase workers during saturation. Do not increase database max connections as the first response. Do not disable timeouts blindly. Do not retry immediately in a tight loop. Do not run heavy extraction against a production primary without an explicit capacity decision.

---

## 31. Common Mistakes

1. Measuring only extraction throughput.
2. Treating CPU as the only capacity signal.
3. Using too many workers.
4. Ignoring application connection capacity.
5. Ignoring replica lag.
6. Running inefficient queries with high concurrency.
7. Holding long transactions unnecessarily.
8. Retrying overloaded queries immediately.
9. Assuming off-peak hours are always safe.
10. Ignoring backups and maintenance workloads.
11. Increasing connection limits without capacity analysis.
12. Failing to define a source-load budget.
13. Ignoring destination backpressure.
14. Treating source protection as an afterthought.

---

## 32. Production Tools You Should Know

### 1. PostgreSQL
Learn EXPLAIN, pg_stat_activity, pg_stat_statements, locks, transactions, connection behavior, and replication monitoring.

### 2. PgBouncer
Recognize external connection pooling and session/transaction pooling when many application and extraction clients share PostgreSQL.

### 3. Prometheus
Recognize time-series monitoring and alerting for database CPU, connections, query latency, replication lag, and extraction pressure.

These tools provide production visibility and control, but the source-load principles should remain understandable without them.

---

## 33. Definition of Done

You can independently:
- identify how extraction consumes database resources
- establish a source capacity budget
- baseline source workload
- design efficient extraction queries
- use indexes appropriately
- prune unnecessary columns
- choose safe batch sizes
- bound extraction concurrency
- protect application connections
- use replicas with explicit consistency limitations
- reason about long transactions
- implement source-aware backpressure
- classify and safely retry source failures
- detect query-plan regression
- coordinate extraction with maintenance workloads
- monitor source and extraction health together
- recover from source overload without losing extraction progress

---

## 34. What You Learned

> **A data extractor is a database workload. Production-grade extraction therefore means controlling not only whether data is correct, but also how much source capacity the pipeline consumes. The source database must remain healthy while the pipeline makes progress.**

```text
MEASURE SOURCE
      ↓
DEFINE CAPACITY BUDGET
      ↓
OPTIMIZE QUERY
      ↓
BOUND BATCHES
      ↓
BOUND CONCURRENCY
      ↓
OBSERVE SOURCE HEALTH
      ↓
APPLY BACKPRESSURE
      ↓
RECOVER SAFELY
```

The key question is:
> **Can this extractor make reliable progress while proving that it is not consuming more database capacity than the source workload can safely provide?**