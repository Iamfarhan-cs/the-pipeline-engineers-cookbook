# E04 — Extract Data from Relational Databases

Relational databases are one of the most common sources for data pipelines.

The extraction problem looks simple:

    SELECT * FROM payments;

In production, that approach can overload the source database, consume excessive memory, read inconsistent data, or repeatedly extract the same rows.

> Extract the required source data while protecting source-system correctness, performance, consistency, and recoverability.

## 1. Problem Recognition

Common production problems:

- The table is too large for one query.
- Extraction consumes excessive memory.
- Extraction overloads the source database.
- Rows change while extraction is running.
- Records are skipped between batches.
- Records are extracted more than once.
- Long-running transactions create pressure.
- Network connections fail during extraction.
- The source schema changes.
- The extractor restarts halfway through.
- Multiple workers read overlapping ranges.
- The extraction query ignores useful indexes.
- A full extraction is repeated unnecessarily.

## 2. Extraction Architecture

    SOURCE DATABASE
          ↓
    CONNECT
          ↓
    DISCOVER TABLE / SCHEMA
          ↓
    DEFINE EXTRACTION BOUNDARY
          ↓
    READ BATCH
          ↓
    VALIDATE
          ↓
    PERSIST RAW/STAGING DATA
          ↓
    CHECKPOINT
          ↓
    NEXT BATCH
          ↓
    COMPLETE

The extraction boundary determines which rows belong to the extraction.

Typical boundaries:

- full table
- primary-key range
- timestamp range
- high-watermark
- snapshot
- change stream

## 3. Full Extraction

A full extraction reads the complete source dataset.

    SELECT
        id,
        amount,
        currency,
        status,
        updated_at
    FROM payments;

Use a full extraction when the table is small, a complete refresh is required, no reliable incremental boundary exists, or a periodic snapshot is required.

For large active tables, a full extraction can create unnecessary source load.

## 4. Explicit Columns

Prefer explicit columns instead of SELECT *.

Benefits:

- avoids accidental ingestion of new columns
- reduces data transfer
- reduces memory use
- makes the extraction contract explicit
- avoids unintentionally extracting sensitive columns

## 5. Inspect the Source First

Identify:

- table name
- primary key
- approximate row count
- useful indexes
- creation and update timestamps
- nullable columns
- data types
- foreign keys
- sensitive columns
- expected change rate

Example:

    SELECT
        column_name,
        data_type,
        is_nullable
    FROM information_schema.columns
    WHERE table_name = 'payments'
    ORDER BY ordinal_position;

The extractor should understand the source before reading large volumes.

## 6. Deterministic Ordering

Batch extraction needs deterministic ordering.

Prefer a unique key:

    ORDER BY id

If ordering by a non-unique timestamp:

    ORDER BY updated_at, id

The second column breaks timestamp ties.

Without deterministic ordering, rows can move between batches and cause omissions or duplicates.

## 7. Batching

Instead of reading millions of rows into memory, read bounded batches.

    READ BATCH
       ↓
    PROCESS
       ↓
    PERSIST
       ↓
    CHECKPOINT
       ↓
    NEXT BATCH

Batch size is a tuning parameter.

Too small:

- excessive round trips
- excessive transaction overhead
- lower throughput

Too large:

- higher memory usage
- longer transactions
- larger failure scope
- greater source load

## 8. Keyset Pagination

For large tables, keyset extraction is often preferable to large OFFSET values.

Instead of:

    LIMIT 10000 OFFSET 1000000

use:

    WHERE id > :last_id
    ORDER BY id
    LIMIT :batch_size

Example:

    SELECT
        id,
        amount,
        currency,
        status,
        updated_at
    FROM payments
    WHERE id > :last_id
    ORDER BY id
    LIMIT :batch_size;

After successful persistence, record the maximum ID from the batch.

Large OFFSET values can require the database to locate and discard many preceding rows. Verify actual behavior with the query plan.

## 9. Composite Keyset Boundaries

A single key is not always enough.

Suppose extraction uses:

    ORDER BY updated_at, id

The checkpoint contains:

    last_updated_at
    last_id

The next query can use:

    WHERE
        updated_at > :last_updated_at
        OR (
            updated_at = :last_updated_at
            AND id > :last_id
        )
    ORDER BY updated_at, id
    LIMIT :batch_size;

This prevents rows sharing a timestamp from being skipped.

## 10. Timestamp Extraction

A timestamp can be used as a boundary:

    WHERE updated_at > :watermark

But timestamps can be unsafe by themselves because:

- several rows can share a timestamp
- timestamp precision may differ
- timezone handling can be wrong
- updates may arrive late
- some application updates may not modify the timestamp

A timestamp plus a unique key is often safer.

## 11. Watermarks

A watermark represents the furthest successfully extracted source position.

Example:

    last_updated_at = 2026-09-26 12:00:00
    last_id = 58291

Critical rule:

> Advance the watermark only after the corresponding data is durably persisted.

Correct:

    READ → PERSIST → VERIFY → ADVANCE WATERMARK

Incorrect:

    READ → ADVANCE WATERMARK → PERSIST

The second sequence can permanently skip data after a failure.

## 12. Source Consistency

Rows can change while a long extraction is running.

Decide whether the pipeline requires:

- best-effort current data
- a transactionally consistent snapshot
- CDC/change capture
- a bounded extraction window

The correct choice depends on the business requirement.

## 13. Consistent Snapshots

When the dataset must represent one logical point in time, a database snapshot or suitable transaction isolation mechanism may be required.

    SNAPSHOT
       ↓
    BATCH 1
       ↓
    BATCH 2
       ↓
    BATCH 3
       ↓
    COMPLETE

Understand the operational cost before using a long-lived transaction. Depending on the database, it can affect MVCC/version storage, locks, replication, and source workload.

## 14. Read Replicas

A read replica can protect the primary workload:

    PRIMARY → REPLICATION → READ REPLICA → EXTRACTOR

But replication lag means the replica may not contain the newest source rows.

Understand:

- replica lag
- freshness requirements
- failover behavior
- whether the watermark is safe to interpret on the replica

## 15. Connection Management

Control:

- connection timeout
- statement timeout
- idle timeout where appropriate
- connection pool size
- transaction scope

Do not create a new connection for every batch.

Prefer a controlled connection lifecycle or pool.

## 16. Python Example

Using PostgreSQL with psycopg:

    import psycopg

    BATCH_SIZE = 10_000

    with psycopg.connect(
        'postgresql://user:password@host/db',
        connect_timeout=10,
    ) as conn:

        last_id = 0

        while True:
            with conn.cursor() as cur:
                cur.execute(
                    '''
                    SELECT id, amount, currency, status, updated_at
                    FROM payments
                    WHERE id > %s
                    ORDER BY id
                    LIMIT %s
                    ''',
                    (last_id, BATCH_SIZE),
                )
                rows = cur.fetchall()

            if not rows:
                break

            persist(rows)
            last_id = rows[-1][0]

This is a learning implementation. Production code also needs durable checkpoints, validation, retries, metrics, structured logging, safe credentials, and restart handling.

## 17. Large Result Sets

Using fetchall() for a very large table can consume excessive memory.

Prefer bounded techniques such as:

- server-side cursors
- chunked fetches
- iterator-based reads
- bounded batches

The target is:

    SOURCE → BOUNDED MEMORY → PERSIST → NEXT CHUNK

Memory should not grow linearly with total table size.

## 18. Source Load Protection

The extractor competes with normal application workloads.

Control source impact with:

- appropriate indexes
- bounded batch sizes
- controlled concurrency
- statement timeouts
- extraction scheduling
- read replicas
- query filtering
- off-peak execution where appropriate

Measure source impact instead of guessing.

Important signals include CPU, I/O, active connections, locks, query latency, replication lag, and application latency.

## 19. Index Awareness

Extraction predicates and ordering should have suitable indexes where appropriate.

For:

    WHERE id > :last_id
    ORDER BY id

a primary-key index is normally useful.

For:

    WHERE updated_at > :watermark
    ORDER BY updated_at, id

a composite index on updated_at and id may be useful.

Do not add indexes blindly. Indexes improve reads but add storage, write, and maintenance cost.

## 20. Query Plans

Inspect large extraction queries.

PostgreSQL:

    EXPLAIN
    SELECT ...

For deeper analysis:

    EXPLAIN ANALYZE
    SELECT ...

EXPLAIN ANALYZE executes the statement, so use care against expensive production queries.

Look for:

- sequential scans
- index scans
- large row estimates
- large actual row counts
- unexpected joins
- expensive sorts
- high execution time

## 21. Parallel Extraction

Large tables can sometimes be divided into independent ranges:

    Worker 1 → IDs 1–1,000,000
    Worker 2 → IDs 1,000,001–2,000,000
    Worker 3 → IDs 2,000,001–3,000,000

Parallelism increases source pressure.

Before adding workers establish:

- source capacity
- safe concurrency
- independent ranges
- checkpoint ownership
- duplicate prevention
- failure recovery

Parallel extraction is an optimization, not the default.

## 22. Schema Changes

Source tables can change.

Possible changes:

- new column
- removed column
- renamed column
- type change
- nullability change
- index change

Explicit extraction columns protect against accidental new-column ingestion, but compatibility testing is still required.

Record an extraction schema/version where useful.

## 23. Source Deletions

Incremental extraction based only on updated_at may not detect deletions.

Possible approaches:

- CDC
- deletion markers
- tombstones
- periodic reconciliation
- full snapshots
- source-specific deletion tracking

Do not assume an incremental update stream represents deletions automatically.

## 24. Testing

Test at least:

1. Empty table.
2. Small table.
3. Exactly one batch.
4. Multiple batches.
5. Batch boundary.
6. Duplicate timestamp values.
7. Restart after a successful batch.
8. Failure before checkpoint.
9. Failure after persistence.
10. Repeated extraction.
11. Large row volume.
12. Slow source query.
13. Connection failure.
14. Statement timeout.
15. Source schema change.
16. Missing expected column.
17. New source column.
18. Null values.
19. Duplicate source identifiers.
20. Concurrent source updates.

For timestamp extraction, explicitly test several rows with the same timestamp but different IDs.

## 25. Observability

Useful metrics:

    db_extraction_runs_total
    db_extraction_failures_total
    db_extraction_batches_total
    db_extraction_rows_total
    db_extraction_duration_seconds
    db_extraction_batch_duration_seconds
    db_extraction_rows_per_second
    db_extraction_retries_total
    db_extraction_checkpoint_updates_total
    db_extraction_source_errors_total

Useful log fields:

    extraction_run_id
    source_database
    source_table
    batch_number
    batch_size
    first_key
    last_key
    watermark
    rows_extracted
    duration
    attempt

Monitor the source as well as the pipeline.

## 26. Intentional Failure

### Failure drill 1 — Stop between batches

Stop the extractor after a successful batch. Restart it and verify that it resumes from the last durable checkpoint.

### Failure drill 2 — Fail persistence

Allow database reading to succeed but make downstream persistence fail. Verify that the source checkpoint does not advance.

### Failure drill 3 — Repeated timestamps

Create several rows with the same timestamp. Verify timestamp-plus-ID extraction does not skip rows.

### Failure drill 4 — Slow query

Run an intentionally inefficient query in a test database. Inspect its plan and observe extraction latency.

### Failure drill 5 — Connection interruption

Terminate the extraction connection. Verify the extractor reports the failure and recovers without silently skipping data.

### Failure drill 6 — Source schema change

Change a test source column. Verify that the extractor follows its compatibility policy.

## 27. Recovery

When extraction fails:

1. Identify the extraction run.
2. Identify the source table.
3. Read the last durable checkpoint.
4. Determine whether the failed batch was persisted.
5. Check source database health.
6. Check query latency and connection errors.
7. Verify whether source data changed during the failed period.
8. Resume from the last safe boundary.
9. Reconcile row counts and key ranges.
10. Confirm the final watermark.

If persistence succeeded but checkpointing failed, do not blindly replay without considering downstream idempotency.

## 28. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **PostgreSQL** | Understand indexes, query plans, transactions, snapshots, replicas, and server-side cursors. |
| **psycopg** | Python PostgreSQL driver for controlled database extraction. |
| **SQLAlchemy** | Python database toolkit for connection management, SQL construction, and database abstraction. |

> These are reference tools for production vocabulary. The underlying extraction mechanics should still be understood independently.

## 29. Production Runbook

### Extraction is slow

Check:

1. Query plan.
2. Index availability.
3. Batch size.
4. Source CPU and I/O.
5. Concurrent extraction workers.
6. Locks.
7. Network latency.

### Source database is overloaded

Check extraction concurrency, batch size, query frequency, indexes, read-replica availability, and scheduling. Reduce extraction pressure before increasing throughput.

### Rows appear to be missing

Check ordering, watermark, timestamp precision, composite boundary logic, concurrent source updates, and checkpoint position.

### Duplicate rows appear

Check restart behavior, checkpoint timing, retries, overlapping worker ranges, and source key uniqueness.

### What not to do

Do not:

- use SELECT * blindly
- load an entire production table into memory
- use huge OFFSET values without understanding the query plan
- advance checkpoints before persistence
- add parallel workers without measuring source capacity
- assume timestamps are unique
- assume incremental extraction captures deletes
- hold long transactions without understanding source impact

## 30. Common Mistakes

### Mistake 1 — Treating extraction as a simple SELECT

At production scale, extraction is a workload-management problem.

### Mistake 2 — Using OFFSET for everything

Keyset extraction is often safer for large ordered datasets.

### Mistake 3 — Using timestamp alone

Ties can cause missed records.

### Mistake 4 — No deterministic ordering

Rows can move unpredictably between batches.

### Mistake 5 — Advancing the watermark too early

A failure can permanently skip data.

### Mistake 6 — Ignoring source load

The pipeline can be correct while damaging the application database.

### Mistake 7 — Assuming a replica is current

Replication lag changes freshness.

## 31. Definition of Done

You are done when you can:

- inspect a relational source before extracting
- define an explicit extraction contract
- choose full versus incremental extraction
- extract large tables in bounded batches
- use deterministic ordering
- implement keyset extraction
- implement composite keyset boundaries
- reason about timestamp watermarks
- checkpoint safely
- explain snapshot consistency
- reason about replica lag
- control connection and statement timeouts
- stream large result sets without unbounded memory
- inspect query plans
- understand extraction indexes
- protect the source from excessive load
- decide when parallel extraction is appropriate
- test source failures and restarts
- detect extraction gaps and duplicates
- recover from failed extraction safely

## 32. What You Learned

The central principle is:

> Database extraction is a controlled workload against a live system, not just a SELECT statement.

A production extractor must control:

    SOURCE LOAD
        +
    EXTRACTION BOUNDARY
        +
    BATCH SIZE
        +
    CONSISTENCY
        +
    CHECKPOINT
        +
    RECOVERY

The key question is:

> What proves that I extracted every required source row according to the extraction contract, without putting unacceptable load on the source?

That question drives batching, ordering, indexes, watermarks, snapshots, checkpointing, testing, and recovery.