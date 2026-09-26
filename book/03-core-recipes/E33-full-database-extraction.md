# E33 — Full Database Extraction

## 1. Problem Recognition

A full database extraction means reading an entire source table or dataset and producing a durable copy or downstream representation.

It sounds simple:

```text
SELECT * FROM source_table
        ↓
write destination
```

In production, that approach can fail because the table may contain millions or billions of rows, the source may be under transactional load, rows may change while extraction is running, the query may consume excessive memory, the destination may fail halfway through, or a restart may duplicate work.

### Recognize the problem

Use this recipe when:

- the source is a relational database;
- the target requires a complete snapshot;
- no reliable incremental/change feed is available or a deliberate full refresh is required;
- the table is too large for naive in-memory extraction;
- source consistency matters;
- the extraction must be restartable;
- operators need evidence of exactly what snapshot was extracted.

### Core distinction

A full extraction is not necessarily one giant query followed by one giant write.

A production full extraction is usually:

```text
DISCOVER SOURCE
      ↓
DEFINE SNAPSHOT BOUNDARY
      ↓
READ IN CONTROLLED BATCHES
      ↓
VALIDATE
      ↓
PERSIST DURABLY
      ↓
VERIFY
      ↓
RECONCILE
      ↓
PUBLISH COMPLETE SNAPSHOT
```

---

## 2. Concept and Reasoning

### What does "full" mean?

A full extraction should have a precise scope.

Examples:

```text
all rows currently visible in table customers
all rows for tenant X
all rows as of snapshot S
all rows from partition 2026-09
```

Never leave the extraction boundary implicit.

### Full extraction vs incremental extraction

| Full extraction | Incremental extraction |
|---|---|
| Reads complete defined scope | Reads changes since a position |
| Often used for initial load | Used for recurring synchronization |
| Can be expensive | Usually smaller |
| Needs snapshot consistency | Needs change-position correctness |
| Often produces a complete replacement | Often produces append/upsert changes |

### Why snapshot consistency matters

Suppose a table contains 10 million rows.

The extractor reads rows 1–5 million.

During extraction, transactions insert, update, and delete rows.

The extractor then reads rows 5–10 million.

Without a consistent extraction boundary, the result can represent multiple logical moments in time.

You may get:

- rows that were inserted after the extraction began;
- rows that changed between batches;
- duplicate or skipped rows under unstable pagination;
- a mixture of old and new versions.

The desired property is usually:

> The extracted dataset represents one defined source state or one explicitly defined extraction window.

---

## 3. Source Investigation

Before implementing, inspect the source database.

Determine:

- database engine and version;
- table name;
- schema name;
- row count estimate;
- primary key;
- unique constraints;
- indexes;
- nullable columns;
- data types;
- large columns;
- generated columns;
- foreign keys;
- partitioning;
- update/delete behavior;
- transaction isolation behavior;
- replication/read-replica options.

Example PostgreSQL inspection:

```sql
SELECT
    table_schema,
    table_name
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_name = 'customers';
```

Inspect columns:

```sql
SELECT
    column_name,
    data_type,
    is_nullable
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name = 'customers'
ORDER BY ordinal_position;
```

Find primary-key columns:

```sql
SELECT
    kcu.column_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
  ON tc.constraint_name = kcu.constraint_name
 AND tc.table_schema = kcu.table_schema
WHERE tc.table_schema = 'public'
  AND tc.table_name = 'customers'
  AND tc.constraint_type = 'PRIMARY KEY'
ORDER BY kcu.ordinal_position;
```

Do not begin a large extraction before understanding the source access path.

---

## 4. Choose the Extraction Boundary

There are several ways to define a full extraction.

### Strategy A — Database snapshot

Read from a transactionally consistent snapshot.

Conceptually:

```text
BEGIN
 ↓
consistent snapshot
 ↓
read batches
 ↓
COMMIT
```

This gives the strongest consistency when the database and transaction duration can support it.

### Strategy B — MVCC transaction

PostgreSQL can provide a consistent view under appropriate transaction isolation.

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

All reads within the transaction can observe one logical snapshot.

The trade-off is that a very long transaction can increase storage/version pressure and may interact badly with a heavily modified source.

### Strategy C — Source-defined snapshot identifier

Some systems provide a snapshot or export mechanism outside the extractor's normal SQL query path.

The extractor records the provider's snapshot identity and reads from that fixed state.

### Strategy D — Explicit extraction window

If true snapshot consistency is not practical, define a bounded window and document the semantics.

For example:

```text
source state observed between T1 and T2
```

This is weaker than a true consistent snapshot and must not be presented as one.

---

## 5. The Simplest Safe Implementation

For a small table, a complete extraction can be straightforward.

```python
import psycopg


def extract_small_table(connection):
    with connection.cursor() as cur:
        cur.execute(
            """
            SELECT id, customer_name, email, created_at
            FROM public.customers
            ORDER BY id
            """
        )
        return cur.fetchall()
```

This is acceptable only when the result comfortably fits the application's memory and the consistency model is understood.

Do not use this pattern blindly for large production tables.

---

## 6. Controlled Batch Extraction

A scalable extractor should avoid loading the complete table into memory.

Use deterministic ordering and a bounded batch size.

```python
import psycopg


def extract_batches(connection, batch_size=10_000):
    last_id = 0

    while True:
        with connection.cursor() as cur:
            cur.execute(
                """
                SELECT id, customer_name, email, created_at
                FROM public.customers
                WHERE id > %s
                ORDER BY id
                LIMIT %s
                """,
                (last_id, batch_size),
            )
            rows = cur.fetchall()

        if not rows:
            break

        yield rows
        last_id = rows[-1][0]
```

This uses keyset-style traversal.

However, this code alone does **not** guarantee a consistent full snapshot if rows can change during extraction.

That distinction is important.

---

## 7. Keyset Pagination for Full Extraction

For large tables, prefer a stable indexed key where possible.

Avoid:

```sql
SELECT *
FROM customers
ORDER BY id
OFFSET 5000000
LIMIT 10000;
```

Deep offsets can require the database to scan or discard large numbers of rows.

Prefer:

```sql
SELECT id, customer_name, email, created_at
FROM customers
WHERE id > %s
ORDER BY id
LIMIT %s;
```

The source should have a suitable index:

```sql
CREATE INDEX IF NOT EXISTS idx_customers_id
ON customers (id);
```

If `id` is already the primary key, the primary-key index normally provides this access path.

---

## 8. Composite Key Extraction

A single-column key is not always available.

Suppose the stable ordering is:

```text
created_at, id
```

Use a composite boundary:

```sql
SELECT id, created_at, customer_name
FROM customers
WHERE (created_at, id) > (%s, %s)
ORDER BY created_at, id
LIMIT %s;
```

The boundary is the pair:

```text
(last_created_at, last_id)
```

This avoids ambiguous ordering when multiple records share the same timestamp.

---

## 9. Deterministic Ordering

Never assume that a database returns rows in a stable order without `ORDER BY`.

Bad:

```sql
SELECT * FROM customers;
```

Better:

```sql
SELECT *
FROM customers
ORDER BY id;
```

For a composite boundary:

```sql
ORDER BY created_at, id;
```

The ordering columns should provide a deterministic position.

---

## 10. Snapshot-Consistent PostgreSQL Extraction

For a controlled PostgreSQL extraction, one approach is a repeatable-read transaction.

```python
import psycopg


connection = psycopg.connect("postgresql://extractor:secret@source/db")

connection.execute(
    "BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ"
)

try:
    with connection.cursor() as cur:
        cur.execute(
            """
            SELECT id, customer_name, email, created_at
            FROM public.customers
            ORDER BY id
            """
        )

        for row in cur:
            process_row(row)

    connection.commit()
except Exception:
    connection.rollback()
    raise
```

For a truly large extraction, do not automatically assume one long transaction is the correct architecture. Long-running snapshots can have source-side costs.

Investigate the database workload before selecting this strategy.

---

## 11. Server-Side Cursors and Streaming

For PostgreSQL, server-side cursors can avoid materializing the entire result set in client memory.

```python
import psycopg


with psycopg.connect("postgresql://extractor:secret@source/db") as conn:
    with conn.cursor(name="full_extract") as cur:
        cur.execute(
            """
            SELECT id, customer_name, email, created_at
            FROM public.customers
            ORDER BY id
            """
        )

        while True:
            rows = cur.fetchmany(10_000)
            if not rows:
                break

            process_batch(rows)
```

The exact transaction behavior and cursor lifetime depend on the database driver and transaction configuration. Test the implementation against the actual database version and driver.

---

## 12. Separate Read and Write Connections

Do not accidentally use the same transaction for source extraction and a remote destination unless the architecture explicitly requires distributed transactional behavior.

Prefer:

```text
SOURCE CONNECTION
       ↓
read batch
       ↓
validate
       ↓
DESTINATION CONNECTION
       ↓
persist
```

The source read and destination write have different failure boundaries.

For a complete snapshot, publish only after the extraction has been verified.

---

## 13. Destination Design

A full extraction should normally land into an isolated destination structure first.

For example:

```text
source.customers
       ↓
extract
       ↓
staging.customers_load_20260926
       ↓
validate
       ↓
publish
       ↓
warehouse.customers
```

This avoids replacing a known-good destination with an incomplete extraction.

### Why not overwrite the target immediately?

Suppose the extractor writes directly into the production table and fails at 60%.

The target now contains a partial snapshot.

A safer pattern is:

```text
build complete snapshot
        ↓
validate snapshot
        ↓
atomically publish/swap where supported
```

---

## 14. Full Extraction Run State

Create a run record before processing.

Example:

```sql
CREATE TABLE database_extraction_run (
    run_id UUID PRIMARY KEY,
    source_database TEXT NOT NULL,
    source_schema TEXT NOT NULL,
    source_table TEXT NOT NULL,
    extraction_started_at TIMESTAMPTZ NOT NULL,
    extraction_completed_at TIMESTAMPTZ,
    snapshot_id TEXT,
    expected_rows BIGINT,
    extracted_rows BIGINT NOT NULL DEFAULT 0,
    persisted_rows BIGINT NOT NULL DEFAULT 0,
    failed_rows BIGINT NOT NULL DEFAULT 0,
    status TEXT NOT NULL,
    error_message TEXT
);
```

Typical state flow:

```text
STARTED
  ↓
EXTRACTING
  ↓
VALIDATING
  ↓
PUBLISHING
  ↓
SUCCEEDED
```

Failure can move the run to:

```text
FAILED
```

or:

```text
PARTIAL
```

according to the contract.

---

## 15. Capture Source Snapshot Evidence

Record enough metadata to identify what source state was extracted.

Useful fields:

```text
run_id
source database
source schema/table
source database version
snapshot/isolation identifier where available
source row-count estimate
source exact count where practical
start time
end time
query version
application version
```

For PostgreSQL, you can record transaction-related evidence according to the selected snapshot strategy.

Do not pretend that a timestamp is a snapshot ID if the database does not provide that guarantee.

---

## 16. Count Before Extraction

For a manageable table, obtain an expected count:

```sql
SELECT COUNT(*)
FROM public.customers;
```

For very large tables, understand that an exact count can itself be expensive.

PostgreSQL statistics can provide estimates:

```sql
SELECT reltuples::bigint AS estimated_rows
FROM pg_class
WHERE oid = 'public.customers'::regclass;
```

Use estimates as estimates. Do not label them exact counts.

A practical run can record both:

```text
expected_rows_exact
expected_rows_estimate
```

when available.

---

## 17. Batch-Level Counts

Every batch should have explicit accounting.

Example:

```text
batch 1
  read      = 10,000
  accepted  = 9,990
  rejected  = 10
  persisted = 9,990
```

Persist batch metadata when recovery and reconciliation require it.

A simple batch table:

```sql
CREATE TABLE database_extraction_batch (
    run_id UUID NOT NULL,
    batch_number BIGINT NOT NULL,
    first_key TEXT,
    last_key TEXT,
    rows_read BIGINT NOT NULL,
    rows_persisted BIGINT NOT NULL,
    rows_failed BIGINT NOT NULL DEFAULT 0,
    completed_at TIMESTAMPTZ,
    status TEXT NOT NULL,
    PRIMARY KEY (run_id, batch_number)
);
```

---

## 18. Checkpointing a Full Extraction

A full extraction can checkpoint its position.

Example:

```text
last_safe_id = 2500000
```

After a successful batch:

```text
read IDs 2500001–2510000
 ↓
persist successfully
 ↓
verify
 ↓
checkpoint = 2510000
```

Never do this:

```text
read batch
 ↓
checkpoint
 ↓
persist
```

If persistence fails after the checkpoint moves, a restart can skip data.

### Important limitation

A key checkpoint alone does not solve snapshot consistency.

If source rows are changing, the checkpoint says where you were, not necessarily what source state existed there.

---

## 19. Handling Source Changes

A full extraction can encounter:

- inserts;
- updates;
- deletes.

Consider a row:

```text
id = 500
balance = 100
```

The extractor reads it.

Then the source updates:

```text
balance = 200
```

If the extraction is not snapshot-consistent, the destination may contain the old value while later rows represent a newer state.

The solution is not always "read faster".

Choose an explicit consistency strategy:

- database snapshot;
- repeatable-read transaction;
- provider export/snapshot;
- bounded extraction semantics;
- downstream reconciliation.

---

## 20. Large Column Handling

Large tables may contain:

- JSON documents;
- text blobs;
- binary data;
- large arrays.

Do not automatically select every column.

Prefer explicit projection:

```sql
SELECT
    id,
    customer_name,
    status,
    created_at,
    updated_at
FROM customers
ORDER BY id;
```

Benefits:

- lower network traffic;
- lower memory use;
- lower source I/O;
- clearer extraction contract.

If a large column is required, stream or chunk it appropriately.

---

## 21. Query Plan Investigation

Before running a production-scale extraction, inspect the query plan.

```sql
EXPLAIN
SELECT id, customer_name, email
FROM customers
WHERE id > 5000000
ORDER BY id
LIMIT 10000;
```

For deeper investigation:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, customer_name, email
FROM customers
WHERE id > 5000000
ORDER BY id
LIMIT 10000;
```

Be careful with `ANALYZE` on production systems because it executes the query.

Look for:

- sequential scans where an index should be usable;
- excessive rows removed by filters;
- high buffer reads;
- sorts that spill to disk;
- unexpectedly expensive joins.

---

## 22. Source Load Protection

A full extraction can compete with application traffic.

Protect the source by considering:

- read replicas;
- off-peak scheduling;
- bounded concurrency;
- controlled batch sizes;
- explicit column projection;
- query timeouts;
- connection limits;
- resource monitoring.

Never assume a read-only query is free.

A full table scan can consume substantial I/O and cache capacity.

---

## 23. Read Replica Extraction

A replica can isolate extraction load from the primary.

But replication introduces a new question:

> What source state does the replica represent?

If the replica is behind, it may not contain the newest primary state.

Record:

```text
replica identity
replication position/lag evidence where available
snapshot timestamp
```

Do not call a replica extraction a current primary snapshot unless the consistency guarantee supports that statement.

---

## 24. Parallel Full Extraction

Large tables may benefit from parallel extraction.

For example, partition by key ranges:

```text
worker 1 → id 1–1,000,000
worker 2 → id 1,000,001–2,000,000
worker 3 → id 2,000,001–3,000,000
```

But parallelism increases source pressure.

It also creates more failure boundaries.

A production design should control:

```text
worker count
batch size
connection count
range ownership
retry behavior
checkpoint per range
```

Never let workers overlap ranges accidentally.

### Range assignment

Store explicit range state:

```sql
CREATE TABLE database_extraction_range (
    run_id UUID NOT NULL,
    range_id BIGINT NOT NULL,
    lower_key BIGINT,
    upper_key BIGINT,
    status TEXT NOT NULL,
    rows_extracted BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (run_id, range_id)
);
```

This makes parallel work auditable and recoverable.

---

## 25. Parallelism vs Consistency

Parallel extraction is not automatically compatible with one global transaction snapshot.

Before parallelizing, determine whether the database supports the required snapshot-sharing mechanism.

Otherwise you may have:

```text
worker 1 → snapshot A
worker 2 → snapshot B
worker 3 → snapshot C
```

The combined result may not represent one database state.

This can be acceptable only if the extraction contract explicitly permits it.

---

## 26. Destination Persistence

A simple batch persistence interface might be:

```python
def persist_batch(destination, rows):
    with destination.cursor() as cur:
        cur.executemany(
            """
            INSERT INTO staging.customers_load (
                id, customer_name, email, created_at
            ) VALUES (%s, %s, %s, %s)
            """,
            rows,
        )
    destination.commit()
```

For large loads, the exact loading mechanism should be chosen deliberately. Batch inserts may be slower than database bulk-loading mechanisms.

The important full-extraction rule is:

```text
batch read
 ↓
persist
 ↓
commit
 ↓
record successful progress
```

---

## 27. Publish Only a Complete Snapshot

A safe architecture is:

```text
source
 ↓
extract
 ↓
staging snapshot
 ↓
validate
 ↓
reconcile
 ↓
publish
```

Possible publication strategies include:

```text
partition replacement
atomic rename/swap
view switch
staging-to-target transaction
MERGE where appropriate
```

The correct method depends on the destination technology.

Do not expose an incomplete staging load as the production dataset.

---

## 28. Validation

Before publishing, validate at least:

### Count

```sql
SELECT COUNT(*) FROM staging.customers_load;
```

### Key uniqueness

```sql
SELECT id, COUNT(*)
FROM staging.customers_load
GROUP BY id
HAVING COUNT(*) > 1;
```

### Null expectations

```sql
SELECT COUNT(*)
FROM staging.customers_load
WHERE id IS NULL;
```

### Range evidence

```sql
SELECT MIN(id), MAX(id)
FROM staging.customers_load;
```

### Sample comparison

Compare deterministic samples between source and destination.

For large datasets, also consider checksums or aggregate reconciliation where appropriate.

---

## 29. Reconciliation

A full extraction should finish with explicit reconciliation.

At minimum compare:

```text
source expected rows
source extracted rows
staging persisted rows
destination published rows
```

Example:

```sql
SELECT
    expected_rows,
    extracted_rows,
    persisted_rows
FROM database_extraction_run
WHERE run_id = %s;
```

Then define the expected relationship.

For a strict full snapshot:

```text
expected source rows = persisted staging rows
```

If rejected rows are allowed, the contract must explain them.

---

## 30. Recovery

### Failure during batch 37

Do not immediately restart from row zero.

First determine:

1. Which batches committed?
2. What was the last safe checkpoint?
3. Which destination rows exist?
4. Is the load idempotent?
5. Is the source snapshot still valid?
6. Can the failed batch be retried safely?

Then either:

```text
resume from last safe boundary
```

or:

```text
discard incomplete staging snapshot
restart complete snapshot
```

The correct choice depends on snapshot validity and destination semantics.

### If the source snapshot expires

A checkpoint from an invalid snapshot may no longer be safe.

Restart from a new valid snapshot rather than assuming the old checkpoint remains meaningful.

---

## 31. Testing

### Unit tests

Test:

- deterministic ordering;
- batch boundaries;
- empty table;
- one-row table;
- exact batch-size table;
- batch-size-plus-one table;
- duplicate key detection;
- checkpoint advancement only after persistence;
- count accounting;
- explicit query projection.

### Integration tests

Use a real database container or test database to verify:

- extraction from a populated table;
- transaction isolation behavior;
- server-side cursor behavior;
- keyset traversal;
- destination staging;
- validation;
- publication;
- rollback after failure.

### Edge cases

Test:

```text
0 rows
1 row
batch_size - 1 rows
batch_size rows
batch_size + 1 rows
large IDs
NULL values
very long text
concurrent source updates
source connection loss
source query timeout
destination failure
```

---

## 32. Intentional Failure Drills

### Drill 1 — Kill extraction after batch 5

Expected:

```text
batches 1–5 durable
later batches absent
checkpoint = last safe batch
```

### Drill 2 — Fail destination during batch 6

Expected:

```text
batch 6 not marked complete
checkpoint does not move past batch 6
```

### Drill 3 — Update source while extraction is running

Observe whether the selected snapshot strategy produces the documented consistency behavior.

### Drill 4 — Delete rows during extraction

Verify that the resulting dataset matches the documented snapshot/window semantics.

### Drill 5 — Kill one parallel worker

Expected:

```text
completed ranges remain complete
failed range remains recoverable
```

### Drill 6 — Make the extraction query slow

Observe:

- source load;
- query latency;
- timeout behavior;
- extraction throughput.

---

## 33. Observability

Track:

```text
database_extraction_runs_started_total
database_extraction_runs_succeeded_total
database_extraction_runs_failed_total
rows_extracted_total
rows_persisted_total
rows_failed_total
extraction_batch_duration
extraction_throughput_rows_per_second
source_query_latency
source_connection_errors
destination_write_latency
```

For each run, record:

```text
run_id
source table
snapshot strategy
expected rows
rows extracted
rows persisted
start/end time
duration
batch count
status
```

Monitor source impact separately:

```text
CPU
I/O
connections
replication lag
query latency
cache pressure
```

---

## 34. Production Runbook

### Before extraction

- [ ] Confirm source table and schema.
- [ ] Confirm primary/stable extraction key.
- [ ] Inspect indexes and query plan.
- [ ] Define snapshot consistency strategy.
- [ ] Estimate table size.
- [ ] Choose batch size.
- [ ] Confirm source load limits.
- [ ] Confirm destination staging location.
- [ ] Confirm rollback/publication strategy.
- [ ] Confirm extraction credentials have least privilege.

### During extraction

Check:

1. Source query latency.
2. Source CPU/I/O.
3. Rows processed per second.
4. Batch duration.
5. Connection count.
6. Destination write latency.
7. Checkpoint movement.
8. Error rate.

### If extraction slows down

Investigate:

- source load;
- missing/ineffective index;
- large batch size;
- excessive parallelism;
- network throughput;
- destination bottleneck;
- replication lag.

Do not immediately increase workers.

### If extraction fails

1. Preserve the failed run record.
2. Identify the last safe batch/checkpoint.
3. Determine whether the source snapshot remains valid.
4. Verify destination state.
5. Resume only if the boundary is safe.
6. Otherwise discard the incomplete snapshot and restart.
7. Reconcile before publication.

### What not to do

- Do not use `SELECT *` blindly on huge tables.
- Do not load the entire result into memory without evidence it is safe.
- Do not rely on unordered database output.
- Do not confuse a timestamp with a snapshot guarantee.
- Do not expose partial staging data as the final dataset.
- Do not increase parallelism without checking source load.
- Do not advance checkpoints before durable persistence.

---

## 35. Common Mistakes

### Mistake 1 — One giant `fetchall()`

Memory usage grows with table size.

### Mistake 2 — No deterministic ordering

Rows can be traversed inconsistently.

### Mistake 3 — Deep OFFSET pagination

Performance can degrade severely on large tables.

### Mistake 4 — Ignoring source changes

The result may represent no coherent source state.

### Mistake 5 — Long transaction without source analysis

A consistent snapshot can create source-side version/storage pressure.

### Mistake 6 — Reading from a replica without checking lag

The extraction may be stale.

### Mistake 7 — Directly replacing the production target

A failed extraction can leave the destination incomplete.

### Mistake 8 — No batch accounting

Operators cannot determine where extraction stopped.

### Mistake 9 — Checkpoint before persistence

A failure can permanently skip data.

### Mistake 10 — Parallel workers with overlapping ranges

Rows can be duplicated or inconsistently processed.

### Mistake 11 — Treating estimated counts as exact

Database statistics are estimates.

### Mistake 12 — Assuming read-only means free

Large reads consume CPU, I/O, connections, cache, and network capacity.

---

## 36. Definition of Done

You are done when you can:

- [ ] Define exactly what "full extraction" means for a source.
- [ ] Inspect the source schema and access path.
- [ ] Choose a stable extraction key.
- [ ] Extract a small table safely.
- [ ] Extract a large table in controlled batches.
- [ ] Explain keyset extraction.
- [ ] Explain why deep offsets are problematic.
- [ ] Explain snapshot consistency.
- [ ] Implement a PostgreSQL consistent-read strategy where appropriate.
- [ ] Stream results without loading the entire table into memory.
- [ ] Track extraction run and batch state.
- [ ] Checkpoint only after durable persistence.
- [ ] Protect the source from excessive load.
- [ ] Use a staging snapshot before publication.
- [ ] Validate and reconcile the extracted dataset.
- [ ] Recover from a failed batch.
- [ ] Test concurrent source changes.
- [ ] Test destination failures.
- [ ] Operate the extraction using a production runbook.

---

## 37. What You Learned

A full database extraction is not simply a `SELECT *` statement. It is a controlled snapshot-and-delivery problem.

You learned to:

1. Define the extraction scope explicitly.
2. Investigate the source before choosing an extraction method.
3. Use deterministic ordering and stable keys.
4. Read large tables in controlled batches.
5. Understand snapshot consistency and its trade-offs.
6. Protect the source database from extraction load.
7. Track progress at batch boundaries.
8. Persist before advancing progress.
9. Build the complete dataset in staging before publication.
10. Reconcile the source, extraction, and destination counts.
11. Recover safely from partial failures.
12. Understand the difference between a checkpoint and a snapshot guarantee.

The key mental model is:

```text
SOURCE DATABASE
      ↓
DEFINE SNAPSHOT BOUNDARY
      ↓
DETERMINISTIC READ
      ↓
CONTROLLED BATCHES
      ↓
VALIDATE
      ↓
PERSIST TO STAGING
      ↓
CHECKPOINT SAFE PROGRESS
      ↓
RECONCILE
      ↓
PUBLISH COMPLETE SNAPSHOT
```

A production Data Engineer should be able to explain not only how to read every row, but how to prove that the rows represent the intended source state, survive failures, protect the source, and become a complete downstream snapshot.