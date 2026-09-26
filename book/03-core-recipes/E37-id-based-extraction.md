# E37 — ID-Based Extraction

## 1. Problem Recognition

A source table is large and changes continuously, but it exposes a reliable identifier that increases monotonically for new records. The pipeline can use that identifier as an extraction position instead of repeatedly scanning the entire table.

```text
LOAD LAST SAFE ID
       ↓
CAPTURE FINITE UPPER ID
       ↓
READ id > LAST AND id <= UPPER
       ↓
VALIDATE
       ↓
PERSIST
       ↓
ADVANCE ID
       ↓
RECONCILE
```

ID-based extraction is simple, but only when the ID's semantics actually support the required change-detection guarantee.

## 2. Concept and Reasoning

### What is an ID watermark?

A durable ID watermark records the highest source identifier the pipeline has safely processed.

Example:

```text
last_safe_id = 10000
```

The next extraction window may be:

```sql
WHERE id > 10000
  AND id <= 15000
```

The critical distinction is:

```text
highest observed ID
        ≠
highest safely processed ID
```

Never advance the ID merely because the source query returned it.

### When ID-based extraction works well

It is appropriate for append-oriented sources where:

- IDs are monotonically increasing for relevant new records;
- IDs are never reused;
- newly relevant records receive IDs greater than previously processed records;
- the source does not require update detection through the ID;
- hard-delete requirements are handled separately.

### When it is not enough

An ID watermark normally does not detect:

- updates to an existing row;
- hard deletes;
- backfilled rows with old IDs;
- records inserted with arbitrary IDs;
- changes that occur without a new ID.

This is the central distinction from timestamp-based extraction.

## 3. ID Contract

Before implementation, document:

| Property | Question |
|---|---|
| Type | BIGINT, UUID, string, etc.? |
| Ordering | Does numeric/string order represent insertion progress? |
| Monotonicity | Do new records always move forward? |
| Reuse | Can IDs be reused? |
| Gaps | Are gaps normal? |
| Assignment | When is the ID assigned? |
| Visibility | When is the row queryable? |
| Updates | Can existing rows change? |
| Deletes | How are deletes represented? |
| Backfills | Can old IDs appear later? |
| Scope | Is ordering global or partition-specific? |

A primary key is not automatically an extraction sequence.

## 4. Gaps Are Normal

IDs can contain gaps:

```text
100
101
104
108
```

That is usually safe.

The extractor should ask:

> Have I processed every relevant ID greater than the previous safe position?

It should not require:

```text
next_id = previous_id + 1
```

Do not confuse sequential allocation with monotonic progress.

## 5. Durable ID State

```sql
CREATE TABLE extraction_id_state (
    pipeline_name TEXT NOT NULL,
    source_system TEXT NOT NULL,
    source_object TEXT NOT NULL,
    partition_key TEXT NOT NULL DEFAULT '',
    last_safe_id BIGINT NOT NULL,
    run_id UUID,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (pipeline_name, source_system, source_object, partition_key)
);
```

Read it with:

```sql
SELECT last_safe_id, run_id
FROM extraction_id_state
WHERE pipeline_name = %s
  AND source_system = %s
  AND source_object = %s
  AND partition_key = %s;
```

The state must survive process restarts.

## 6. Capture a Finite Upper ID

Do not allow the extraction window to move indefinitely while new rows are inserted.

A source-derived upper bound can be:

```sql
SELECT MAX(id)
FROM customers;
```

Then:

```sql
SELECT id, customer_name, status, created_at
FROM customers
WHERE id > %s
  AND id <= %s
ORDER BY id
LIMIT %s;
```

The run now has a finite ID domain:

```text
(last_safe_id, upper_id]
```

Rows inserted after the upper bound belong to a later run.

## 7. Why `MAX(id)` Is Not Always a Perfect Snapshot

Consider:

```text
worker A starts transaction
worker B inserts id=100
extractor reads MAX(id)=100
worker B transaction later commits
```

The meaning of `MAX(id)` depends on transaction isolation and visibility.

More importantly, an ID can be allocated before the corresponding transaction becomes visible.

For stronger guarantees, define the source transaction/visibility contract and consider:

- overlap;
- snapshot isolation;
- source sequence semantics;
- CDC;
- a source-side change log.

Do not assume that numeric order alone proves visibility order.

## 8. Safe Advancement

Correct:

```text
READ ID RANGE
    ↓
VALIDATE
    ↓
PERSIST
    ↓
COMMIT
    ↓
ADVANCE SAFE ID
```

Incorrect:

```text
READ ID RANGE
    ↓
ADVANCE ID
    ↓
PERSIST
```

If persistence fails after advancement, future runs can permanently skip records.

## 9. Batch Processing

For large ranges, process in bounded batches.

```python
while True:
    rows = read_batch(
        connection,
        last_safe_id,
        upper_id,
        batch_size=5_000,
    )

    if not rows:
        break

    persist(rows)

    last_safe_id = rows[-1].id
```

The final ID becomes progress only after the corresponding batch is durably persisted.

## 10. Keyset Pagination

ID extraction naturally supports keyset pagination.

```sql
SELECT id, customer_name, status
FROM customers
WHERE id > %s
  AND id <= %s
ORDER BY id
LIMIT %s;
```

After processing a batch:

```text
last_safe_id = last_row.id
```

The next query starts after that value.

This avoids the growing cost of deep offsets:

```sql
OFFSET 5000000
```

The database can seek using the ID index instead.

## 11. Indexing

A primary key or suitable B-tree index usually supports this pattern:

```sql
CREATE INDEX idx_customers_id
ON customers (id);
```

If `id` is already the primary key, an additional duplicate index is unnecessary.

Verify with:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, customer_name
FROM customers
WHERE id > 10000
  AND id <= 15000
ORDER BY id
LIMIT 5000;
```

## 12. Gaps and Missing IDs

Suppose:

```text
100
101
105
106
```

Processing through 106 is safe if the source contract says IDs are monotonically assigned and missing IDs do not represent future records.

Do not wait forever for IDs 102–104.

But if the source can later insert a record with ID 103, a simple high-ID watermark is unsafe.

The source contract determines whether gaps are harmless.

## 13. Late Visibility

Consider:

```text
ID 200 allocated
 ↓
transaction remains open
 ↓
extractor reaches 200
 ↓
transaction not visible yet
 ↓
extractor advances beyond 200
 ↓
transaction commits
```

The row can be skipped permanently.

Possible protections:

- extract from a committed sequence rather than an allocation sequence;
- use a source transaction boundary;
- delay the upper bound;
- use an overlap strategy with a durable lower safety margin;
- use CDC.

An auto-increment value is not automatically a commit-order watermark.

## 14. Overlap for ID Extraction

If source visibility is uncertain, you can intentionally re-read a safe ID range.

Example:

```text
last_safe_id = 10000
overlap = 100
query_start = 9900
```

The query may reprocess IDs 9901–10000.

The durable watermark remains:

```text
10000
```

not 9900.

The destination must be idempotent.

Do not use overlap as a substitute for understanding the source transaction model.

## 15. Idempotent Destination Writes

```sql
INSERT INTO customer_target (
    id,
    customer_name,
    status
)
VALUES (%s, %s, %s)
ON CONFLICT (id)
DO UPDATE SET
    customer_name = EXCLUDED.customer_name,
    status = EXCLUDED.status;
```

If the source row is immutable after creation, an insert-only destination may be enough:

```sql
INSERT INTO customer_target (id, customer_name, status)
VALUES (%s, %s, %s)
ON CONFLICT (id) DO NOTHING;
```

Choose the write semantics according to the source mutation model.

## 16. Updates Are the Major Limitation

Suppose:

```text
ID 42 created
ID 42 processed
watermark = 100
```

Later:

```text
ID 42 status changes
```

The ID is still 42.

This query will not find it:

```sql
WHERE id > 100
```

If updates matter, combine the ID strategy with another change signal or use CDC/change tables.

Possible designs:

```text
ID for inserts
+
updated_at for updates
```

or:

```text
CDC sequence for all mutations
```

## 17. Deletes

Hard deletes have the same limitation.

```text
ID 42 exists
 ↓
ID 42 deleted
 ↓
no row with ID 42 remains
```

A current-table ID scan cannot discover the deletion.

Use soft deletes, deletion logs, CDC, or reconciliation when deletes matter.

## 18. ID Reuse

If IDs can be reused:

```text
ID 42 exists
 ↓
row deleted
 ↓
ID 42 reused
```

A high-water ID can no longer represent unique progress safely.

Production extraction requires non-reused identifiers or another durable change position.

## 19. UUIDs and Non-Numeric IDs

Not every primary key is suitable as a high-watermark.

Random UUIDs do not provide insertion order:

```text
UUID A
UUID B
UUID C
```

Lexicographic ordering is not the same as creation order.

If the table uses UUID primary keys, use another ordered change signal such as:

- creation sequence;
- timestamp plus tie-breaker;
- database sequence;
- CDC position.

Do not turn an arbitrary primary key into an extraction watermark merely because it is unique.

## 20. Partitioned ID Extraction

If IDs are independently generated per tenant:

```text
tenant A → IDs 1..N
tenant B → IDs 1..N
```

A global watermark is meaningless.

Store:

```text
partition A → watermark A
partition B → watermark B
```

The partition must be part of the extraction-state key.

## 21. Concurrent Workers

For one shared ID stream, two workers must coordinate their ranges.

Unsafe:

```text
worker A → 10000..15000
worker B → 10000..15000
```

This creates duplicate work and competing progress.

Better:

```text
worker A → 10000..15000
worker B → 15001..20000
```

Range ownership must be durable.

Alternatively, use a work queue or a single ordered consumer.

## 22. Range Allocation

A range-allocation table can coordinate workers:

```sql
CREATE TABLE extraction_id_ranges (
    range_id BIGSERIAL PRIMARY KEY,
    start_id BIGINT NOT NULL,
    end_id BIGINT NOT NULL,
    status TEXT NOT NULL,
    worker_id TEXT,
    claimed_at TIMESTAMPTZ
);
```

Workers claim ranges before processing.

The range lifecycle can be:

```text
AVAILABLE
   ↓
CLAIMED
   ↓
PROCESSING
   ↓
COMPLETED
```

A failed range must be safely retryable.

## 23. First Run

For a historical extraction:

```text
last_safe_id = 0
```

only if zero is outside the valid source domain.

Otherwise use the documented minimum ID or an explicit configured starting point.

Do not assume all databases start IDs at 1.

## 24. Crash Recovery

Suppose:

```text
last_safe_id = 10000
batch 10001..10500 persisted
process crashes before checkpoint
```

Restart from 10000.

Records 10001..10500 may be processed again.

That is correct if the destination is idempotent.

If the checkpoint had advanced before persistence, the batch could be skipped.

## 25. Reconciliation

For append-only sources, compare counts by ID range where possible.

Example:

```sql
SELECT COUNT(*)
FROM customers
WHERE id > %s
  AND id <= %s;
```

Compare with persisted records for the same range.

Also inspect:

```sql
SELECT MAX(id)
FROM customers;
```

A source maximum far ahead of the committed watermark may represent normal backlog or an extraction problem. Interpret it using expected source activity.

## 26. Testing

### Unit tests

Test:

- ID comparison;
- boundary inclusion/exclusion;
- range calculations;
- gap handling;
- overlap calculation;
- partition-specific state.

### Integration tests

Test:

- IDs with gaps;
- exact upper-bound ID;
- ID just above upper bound;
- duplicate replay;
- crash after persistence;
- crash before persistence;
- concurrent workers;
- late visibility;
- source connection failure;
- destination failure.

### Source-contract tests

Verify:

- new IDs increase as expected;
- IDs are never reused;
- IDs are not assigned independently of visibility in a way that can create permanent gaps;
- updates have a separate change mechanism when required.

## 27. Intentional Failure Drills

### Drill 1 — Crash after persistence

Persist a range, prevent checkpoint advancement, restart.

Expected:

```text
duplicate work
    ↓
idempotent destination
    ↓
correct final state
```

### Drill 2 — Gap creation

Delete or skip IDs inside a range.

Expected: the extractor continues if the source contract allows permanent gaps.

### Drill 3 — Concurrent range claim

Have two workers attempt to claim the same range.

Expected: only one worker owns the range.

### Drill 4 — Late visibility

Delay a transaction after its ID is allocated.

Expected: the test exposes whether the ID is a safe commit-order position.

### Drill 5 — Existing-row update

Update an old ID after the watermark passes it.

Expected: demonstrate that ID-only extraction does not detect the update.

This failure is an important limitation, not a bug in the query.

## 28. Observability

Track:

```text
starting_id
upper_id
ending_id
id_lag
records_read
records_persisted
records_duplicated
records_rejected
range_duration_seconds
checkpoint_write_failures
source_query_duration
```

For parallel extraction also track:

```text
range_id
worker_id
range_start
range_end
range_status
range_age
```

Alert on:

- watermark not advancing;
- growing ID backlog;
- stuck ranges;
- repeated range failures;
- checkpoint regressions;
- unexpected duplicate rates.

## 29. Production Runbook

### Before deployment

- [ ] Verify ID ordering semantics.
- [ ] Verify IDs are not reused.
- [ ] Understand gaps.
- [ ] Understand transaction visibility.
- [ ] Determine update strategy.
- [ ] Determine delete strategy.
- [ ] Create/verify ID index.
- [ ] Define initial ID.
- [ ] Define upper-bound strategy.
- [ ] Define partition ownership.
- [ ] Make destination writes idempotent.
- [ ] Test crash recovery.

### Before each run

Check:

1. Last safe ID.
2. Current source maximum ID.
3. ID backlog.
4. Previous run status.
5. Expected source growth.

### If the watermark does not advance

Trace:

```text
source query
    ↓
rows returned?
    ↓
validation
    ↓
persistence
    ↓
transaction commit
    ↓
watermark update
```

### If IDs are unexpectedly missing

Determine whether they are:

- permanent allocation gaps;
- uncommitted transactions;
- filtered records;
- invalid source data;
- evidence that the ID is not a safe progress signal.

Do not wait indefinitely for missing IDs unless the source contract requires them.

### If an old record changes

ID-only extraction will not see it. Use the source's update/change mechanism.

### What not to do

- Do not assume primary key means insertion order.
- Do not require gap-free IDs unless documented.
- Do not advance before persistence.
- Do not use random UUIDs as high-watermarks.
- Do not ignore ID reuse.
- Do not assume IDs represent commit order.
- Do not assume ID extraction captures updates or deletes.
- Do not allow workers to claim the same range without coordination.

## 30. Common Mistakes

1. Treating every primary key as a watermark.
2. Assuming IDs are gap-free.
3. Assuming allocation order equals commit order.
4. Advancing before persistence.
5. Forgetting that updates do not change IDs.
6. Forgetting that hard deletes disappear from the current table.
7. Ignoring ID reuse.
8. Using random UUID ordering as insertion progress.
9. Using one watermark for independently generated partition IDs.
10. Letting concurrent workers own overlapping ranges.
11. Using `MAX(id)` without understanding visibility.
12. Waiting forever for missing IDs.
13. Loading large ranges into memory.
14. Resetting a damaged watermark to the current maximum.
15. Treating duplicate replay as unsafe instead of making the destination idempotent.

## 31. Definition of Done

- [ ] Explain what an ID watermark represents.
- [ ] Verify the source ID contract.
- [ ] Distinguish unique identity from ordered progress.
- [ ] Handle gaps correctly.
- [ ] Capture a finite upper ID.
- [ ] Implement keyset pagination.
- [ ] Store ID progress durably.
- [ ] Advance only after persistence.
- [ ] Make destination writes idempotent.
- [ ] Handle overlap safely when required.
- [ ] Understand late visibility.
- [ ] Explain why updates are invisible to ID-only extraction.
- [ ] Explain why deletes are invisible to current-table ID scans.
- [ ] Handle partitions independently.
- [ ] Coordinate parallel range ownership.
- [ ] Test crash, gap, concurrency, and visibility failures.
- [ ] Monitor ID backlog and range health.
- [ ] Operate the extractor using the production runbook.

## 32. What You Learned

ID-based extraction is powerful because ordered IDs make keyset pagination simple and efficient. It is dangerous when the engineer confuses uniqueness with ordering or assumes ID progress captures every kind of source mutation.

You learned to:

1. Treat the ID as a source contract.
2. Distinguish identity from progress.
3. Accept legitimate gaps.
4. Capture finite ID windows.
5. Use keyset pagination instead of deep offsets.
6. Advance state only after durable persistence.
7. Make replay idempotent.
8. Understand allocation versus commit order.
9. Recognize that IDs normally do not detect updates or deletes.
10. Handle independently generated partition IDs.
11. Coordinate parallel range ownership.
12. Test the source contract instead of assuming it.

Core mental model:

```text
LAST SAFE ID
      ↓
CAPTURE FINITE UPPER ID
      ↓
READ ID RANGE
      ↓
VALIDATE
      ↓
PERSIST
      ↓
VERIFY
      ↓
ADVANCE ID
      ↓
RECONCILE
```

The key question is not merely "What is the highest ID?" It is:

> Does this ID provide a trustworthy ordered position for the exact source changes this pipeline is responsible for?
