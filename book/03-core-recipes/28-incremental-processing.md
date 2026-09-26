# Recipe 28 — Incremental Processing

A full load processes everything.

An incremental pipeline processes only what is new or changed.

That difference becomes important as data grows.

Suppose a table contains:

~~~text
100 million records
~~~

A daily job does not normally need to process all 100 million records.

If only 50,000 records changed today, processing the entire table is wasteful.

A better approach is:

~~~text
existing data
      |
      +----------------------+
                             |
new or changed data          |
      |                      |
      v                      |
incremental processing       |
      |                      |
      v                      |
target <---------------------+
~~~

Incremental processing reduces unnecessary work.

But it introduces new problems:

- How do we know what is new?
- How do we know what changed?
- Where did the previous run stop?
- What happens when data arrives late?
- What happens when timestamps are duplicated?
- What happens when records are updated?
- How do we avoid missing records?
- How do we avoid processing the same records repeatedly?

The central idea is:

> Incremental processing is not simply filtering by today's date. It is maintaining a reliable boundary between processed and unprocessed data.

---

# 1. Goal

The goal of this recipe is to build a clear mental model for incremental processing and understand how to implement it safely.

By the end of this recipe, you should understand how to:

- distinguish full loads from incremental loads
- identify new and changed records
- use high-water marks
- use stable cursors
- handle timestamps safely
- handle late-arriving data
- use overlap windows
- combine incremental processing with idempotency
- checkpoint progress
- handle failures
- process inserts and updates
- reconcile incremental results
- recover from missed data
- monitor incremental pipelines

The central principle is:

> Process the smallest safe set of data while maintaining enough overlap and state to avoid missing changes.

---

# 2. Problem

Imagine a source table with:

~~~text
100,000,000 records
~~~

A daily full load might do:

~~~text
read 100,000,000
      |
      v
transform 100,000,000
      |
      v
write 100,000,000
~~~

If only 100,000 records changed, most of that work is unnecessary.

An incremental load tries to do:

~~~text
find changed records
      |
      v
process 100,000
      |
      v
update target
~~~

This is much more efficient.

But the difficult part is finding the correct 100,000 records.

---

# 3. Full Load vs Incremental Load

## Full Load

A full load processes the complete source population.

~~~text
source
  |
  v
all records
  |
  v
target
~~~

It is simple conceptually.

It can also become expensive as data grows.

## Incremental Load

An incremental load processes only new or changed records.

~~~text
source
  |
  v
new/changed records
  |
  v
target
~~~

The challenge is detecting those records correctly.

---

# 4. Why Incremental Processing Matters

Incremental processing can reduce:

- database reads
- database writes
- CPU usage
- network traffic
- storage operations
- API requests
- processing time

It can also make frequent processing practical.

For example:

~~~text
every 5 minutes
every hour
every day
~~~

Instead of repeatedly rebuilding the entire dataset.

However:

> Efficiency is useful only if correctness is preserved.

A fast pipeline that silently misses records is not a reliable pipeline.

---

# 5. When to Use Incremental Processing

Incremental processing is useful when:

- source data grows continuously
- only a small portion changes between runs
- the source provides change timestamps
- the source provides monotonically increasing IDs
- the source provides CDC events
- the target is too expensive to rebuild completely
- processing needs to run frequently

A full load may still be appropriate when:

- the dataset is small
- the target is rebuilt intentionally
- the source does not provide reliable change information
- correctness is easier to guarantee with a full rebuild
- historical correction is required

Do not use incremental processing simply because it sounds more scalable.

Choose it when the source and target semantics support it.

---

# 6. Incremental Processing Signals

An incremental pipeline needs a way to identify changes.

Common signals include:

- increasing IDs
- creation timestamps
- update timestamps
- version numbers
- sequence numbers
- change-data-capture events
- source-provided offsets

These signals are not equally reliable.

The best choice depends on the source.

---

# 7. High-Water Mark

A high-water mark records the furthest point that the pipeline has safely processed.

For example:

~~~text
last_processed_id = 500000
~~~

The next run can select:

~~~text
id > 500000
~~~

The model is:

~~~text
source
  |
  v
id = 1 ... 500000
          |
          v
       processed
          |
          v
high-water mark = 500000
~~~

Next run:

~~~text
id > 500000
~~~

This works well when IDs are stable and monotonically increasing.

---

# 8. Timestamp-Based Incremental Processing

Another common approach uses an update timestamp.

For example:

~~~text
updated_at
~~~

The previous run stores:

~~~text
last_processed_time = 2026-09-25 10:00:00
~~~

The next run selects records after that point.

Conceptually:

~~~text
updated_at > last_processed_time
~~~

This can work well.

But timestamps have important edge cases.

---

# 9. The Timestamp Problem

Suppose two records have:

~~~text
updated_at = 10:00:00
~~~

If the first run processes one record and stores:

~~~text
last_processed_time = 10:00:00
~~~

Then the next query uses:

~~~text
updated_at > 10:00:00
~~~

The second record may be skipped.

The timestamp alone is not always a sufficient cursor.

A safer ordering can be:

~~~text
ORDER BY updated_at, id
~~~

with a cursor containing both values.

For example:

~~~text
last_updated_at = 10:00:00
last_id = 105
~~~

The next selection can continue after that exact position.

The exact implementation depends on the database and source behavior.

---

# 10. Composite Cursor

A composite cursor can look like:

~~~text
(updated_at, id)
~~~

Suppose processed records end at:

~~~text
updated_at = 10:00:00
id = 105
~~~

The next records are:

~~~text
updated_at > 10:00:00
OR
(updated_at = 10:00:00 AND id > 105)
~~~

This creates a deterministic ordering.

The important principle is:

> The cursor must identify a precise position in the source ordering.

---

# 11. High-Water Mark Requirements

A good incremental cursor should be:

- stable
- deterministic
- persisted
- ordered
- available from the source
- suitable for resuming
- resistant to ambiguity

Before choosing a cursor, investigate the source.

Ask:

1. Can the value change?
2. Is it unique?
3. Is it monotonically increasing?
4. Can multiple records share the same value?
5. Can records arrive late?
6. Can records be updated after creation?
7. Can records be deleted?
8. Does the source clock have sufficient precision?

These questions determine whether the cursor is safe.

---

# 12. Created Time vs Updated Time

There is an important difference between:

~~~text
created_at
~~~

and:

~~~text
updated_at
~~~

If you only need new records:

~~~text
created_at
~~~

may be enough.

If existing records can change:

~~~text
updated_at
~~~

may be required.

For example:

~~~text
Record A created:
09:00

Record A updated:
11:00
~~~

A pipeline using only created_at may never see the update.

---

# 13. Inserts vs Updates

Incremental processing often needs to handle both:

~~~text
new records
+
changed records
~~~

The source may contain:

~~~text
record 1 -> new
record 2 -> unchanged
record 3 -> updated
~~~

The target may need:

~~~text
insert record 1
skip record 2
update record 3
~~~

This usually requires a stable business or technical key.

---

# 14. Upsert Pattern

A common target strategy is an upsert:

~~~text
source record
      |
      v
find target key
      |
      +---- not found ----> insert
      |
      +---- found --------> update
~~~

For PostgreSQL, an upsert can be implemented with a unique constraint and conflict handling.

The exact SQL depends on the target schema.

The important design requirement is:

> The target must have a reliable identity for the record being incrementally processed.

---

# 15. Incremental Processing and Idempotency

Incremental processing should still be idempotent.

Suppose the same record is selected twice:

~~~text
run 1
  |
  v
record A

run 2
  |
  v
record A
~~~

The second processing should not corrupt the target.

This is especially important when using:

- overlap windows
- retries
- worker restarts
- checkpoint recovery
- late data handling

Incremental processing and idempotency work together.

---

# 16. Overlap Windows

A common technique is to intentionally process a small amount of already-seen data again.

Suppose the previous cursor is:

~~~text
10:00:00
~~~

Instead of starting exactly there, the next run starts slightly earlier:

~~~text
09:55:00
~~~

This creates an overlap.

The flow becomes:

~~~text
previous boundary
      |
      v
move backward slightly
      |
      v
read overlap
      |
      v
deduplicate / upsert
~~~

Why?

Because timestamps and distributed systems can have delays.

The overlap reduces the chance of missing late-arriving records.

---

# 17. Why Overlap Helps

Suppose a record was actually created at:

~~~text
09:59:59
~~~

but the source did not expose it until:

~~~text
10:01:00
~~~

A strict boundary may miss it.

An overlap window gives the pipeline another opportunity to see it.

This creates a tradeoff:

~~~text
larger overlap
    |
    +--> fewer missed records
    |
    +--> more repeated processing
~~~

Therefore overlap must be combined with idempotency.

---

# 18. Choosing the Overlap Window

There is no universal overlap value.

It depends on:

- source latency
- clock behavior
- replication delay
- network delay
- scheduling frequency
- data arrival patterns

For example:

~~~text
5 minutes
15 minutes
1 hour
~~~

may be reasonable in different systems.

The value should be based on observed behavior rather than guesswork.

---

# 19. High-Water Mark vs Overlap Window

Without overlap:

~~~text
processed through 10:00
      |
      v
next starts at 10:00
~~~

With overlap:

~~~text
processed through 10:00
      |
      v
next starts at 09:55
~~~

The overlap creates repeated reads.

Idempotency protects the target from repeated writes.

This gives:

~~~text
overlap
   +
idempotency
   =
safer incremental processing
~~~

---

# 20. Checkpointing Incremental Runs

Incremental processing needs persistent progress.

A checkpoint may contain:

~~~text
last_processed_id
last_processed_timestamp
processed_at
run_id
~~~

Or a composite cursor:

~~~text
last_updated_at
last_id
~~~

The checkpoint should only advance after the corresponding work is safely completed.

---

# 21. Do Not Advance the Cursor Too Early

Consider:

~~~text
read records
     |
     v
update cursor
     |
     X
write target fails
~~~

The cursor now says the records were processed even though they were not.

The next run may skip them.

This is a classic data-loss scenario.

Safer:

~~~text
read records
     |
     v
process records
     |
     v
write target
     |
     v
commit
     |
     v
advance checkpoint
~~~

The exact transaction boundary depends on the architecture.

---

# 22. Incremental Processing With Batches

Even incremental runs may contain large amounts of data.

Suppose:

~~~text
one hour of changes
=
500,000 records
~~~

Do not necessarily process all 500,000 in one transaction.

Use batches:

~~~text
500,000
   |
   +--> 10,000
   +--> 10,000
   +--> 10,000
   +--> ...
~~~

The same batching principles from Chapter 27 apply.

---

# 23. Incremental Processing and Late Data

Late data is one of the hardest incremental-processing problems.

Suppose the pipeline processes:

~~~text
10:00
10:05
10:10
~~~

Then a record belonging to 10:03 arrives at 10:15.

If the pipeline only processes:

~~~text
updated_at > 10:10
~~~

the 10:03 record may be missed.

This is why incremental systems often need:

- overlap windows
- watermarks
- late-data handling
- source-side change tracking
- periodic reconciliation

---

# 24. Event Time vs Processing Time

Suppose:

~~~text
event occurred:
10:03

event received:
10:15
~~~

The two times are different.

An incremental pipeline must know which timestamp represents the change it cares about.

Using the wrong timestamp can cause:

- missed records
- duplicate processing
- incorrect windows
- delayed results

The correct timestamp depends on the source and business meaning.

---

# 25. Deletes

Incremental processing is easy to misunderstand when records can be deleted.

Suppose:

~~~text
source:
record A
record B
record C
~~~

Then record B is deleted.

A simple query for new records may never tell the target that B disappeared.

The pipeline may need a deletion signal such as:

- tombstone
- CDC event
- deleted flag
- audit table
- source change log

If the source does not expose deletion information, incremental synchronization becomes more difficult.

---

# 26. Soft Deletes

Some systems use:

~~~text
deleted = true
~~~

instead of physically deleting records.

This can make incremental processing easier because the update is visible.

For example:

~~~text
record B
deleted = true
updated_at = 10:30
~~~

The incremental pipeline can detect the update and apply the deletion to the target.

The exact target behavior depends on the data model.

---

# 27. Source Updates That Move Backward

Suppose a source record has:

~~~text
updated_at = 10:00
~~~

Then an unexpected correction sets it to:

~~~text
updated_at = 09:00
~~~

A timestamp-based cursor may not detect it.

This is why source semantics matter.

Before using an update timestamp as the only incremental signal, verify whether the source guarantees that it moves forward.

---

# 28. Clock and Timezone Issues

Timestamp-based incremental processing can fail when systems use inconsistent time handling.

Potential problems include:

- local time vs UTC
- daylight-saving changes
- inconsistent precision
- database timezone settings
- application timezone conversion
- clock drift

Prefer a clear timestamp standard.

For example:

~~~text
UTC timestamps
~~~

The exact project standard should be verified.

---

# 29. Incremental Processing and Reference Data

A transformation may depend on reference data.

For example:

~~~text
transaction
    +
exchange rates
    |
    v
converted amount
~~~

If the reference data changes, reprocessing only new transactions may not update historical results.

This creates an important distinction:

~~~text
incremental source change
~~~

versus:

~~~text
derived result affected by reference-data change
~~~

The second case may require a backfill.

---

# 30. Incremental Processing and Backfills

Incremental processing and backfills complement each other.

A typical lifecycle is:

~~~text
normal incremental processing
          |
          v
problem discovered
          |
          v
historical correction
          |
          v
backfill
          |
          v
return to incremental processing
~~~

Backfill handles the historical correction.

Incremental processing handles new changes going forward.

---

# 31. Incremental Processing and Replay

Replay can also be used when the incremental pipeline needs to process an existing event again.

For example:

~~~text
event already stored
      |
      v
replay
      |
      v
incremental processing logic
~~~

The same idempotency rules should still apply.

---

# 32. Incremental Processing With PostgreSQL

For a PostgreSQL source, common incremental signals include:

- increasing primary keys
- created timestamps
- updated timestamps
- version columns
- change tables
- CDC mechanisms

A simple ID-based query might conceptually be:

~~~sql
SELECT *
FROM source_table
WHERE id > :last_processed_id
ORDER BY id
LIMIT :batch_size;
~~~

This is a **generic example**.

It is safe only if the ID has the required ordering and the source semantics support this approach.

---

# 33. Timestamp-Based PostgreSQL Example

A generic timestamp-based query could be:

~~~sql
SELECT *
FROM source_table
WHERE updated_at >= :window_start
  AND updated_at < :window_end
ORDER BY updated_at, id
LIMIT :batch_size;
~~~

This is a **generic example**.

A real implementation must account for:

- cursor persistence
- overlapping windows
- identical timestamps
- pagination
- late data
- updates
- transaction boundaries

---

# 34. Cursor Pagination

Offset pagination can become expensive for large datasets.

For example:

~~~text
OFFSET 500000
~~~

may require the database to scan or skip many rows.

A cursor-based approach can be more efficient:

~~~text
WHERE id > last_id
ORDER BY id
LIMIT batch_size
~~~

The correct approach depends on indexes and query patterns.

---

# 35. Indexing for Incremental Queries

Incremental processing depends heavily on efficient source selection.

If filtering by:

~~~text
updated_at
~~~

the source may need an appropriate index.

If ordering by:

~~~text
updated_at, id
~~~

the index strategy should support the actual query.

Do not assume an index exists.

Inspect the real schema and query plan before optimizing.

---

# 36. Incremental State Storage

The checkpoint can be stored in several ways.

For example:

~~~text
database table
configuration store
workflow metadata
checkpoint file
message offset
~~~

The choice depends on the architecture.

For a database-backed pipeline, a database control table may be practical.

A generic record could contain:

~~~text
pipeline_name
last_cursor
last_run_at
status
updated_at
~~~

This is a **generic example**.

---

# 37. One Cursor Per Pipeline or Partition

Some systems need multiple cursors.

For example:

~~~text
pipeline
   |
   +--> tenant A cursor
   +--> tenant B cursor
   +--> tenant C cursor
~~~

Or:

~~~text
partition 1 cursor
partition 2 cursor
partition 3 cursor
~~~

This can improve parallelism.

But it also increases operational complexity.

The state model must clearly identify which cursor belongs to which processing unit.

---

# 38. Incremental Failure Recovery

Suppose a run processes:

~~~text
batch 1 -> success
batch 2 -> success
batch 3 -> failure
batch 4 -> not started
~~~

The checkpoint should represent the last safely completed position.

Recovery:

~~~text
checkpoint
    |
    v
resume batch 3
~~~

If batch 3 partially completed before failure, idempotency protects against duplicate effects.

---

# 39. Incremental Processing and Retries

A temporary error should normally be retried.

For example:

~~~text
batch
  |
  X
database timeout
  |
  v
retry
~~~

If the retry succeeds:

~~~text
batch
  |
  v
commit
  |
  v
checkpoint
~~~

If retries are exhausted, the batch may require:

- quarantine
- failure state
- manual investigation
- replay
- backfill

The correct behavior depends on the pipeline.

---

# 40. Incremental Processing and Quarantine

Record-level invalid data can be quarantined without stopping the entire incremental run.

For example:

~~~text
batch
 |
 +--> record A -> success
 |
 +--> record B -> quarantine
 |
 +--> record C -> success
~~~

This is useful when failures are isolated.

If the failure affects the entire batch, the batch may need to stop instead.

---

# 41. Watermarks

A watermark represents a boundary up to which the system considers data sufficiently complete.

For example:

~~~text
current time = 10:30
watermark    = 10:20
~~~

The system may intentionally process data only up to 10:20 because later data may still arrive.

This trades latency for completeness.

The exact watermark strategy depends on the source and business requirements.

---

# 42. High-Water Mark vs Watermark

These concepts are related but different.

### High-water mark

Usually describes the furthest source position already processed.

~~~text
last processed ID = 500000
~~~

### Watermark

Often describes a boundary beyond which data is not yet considered complete.

~~~text
safe event time = 10:20
~~~

A pipeline may use both.

For example:

~~~text
watermark
   |
   v
select safe records
   |
   v
high-water mark
   |
   v
checkpoint progress
~~~

---

# 43. Overlap and Watermarks

Overlap can be used with a watermark.

For example:

~~~text
watermark = 10:20
overlap   = 5 minutes
~~~

The next run may reconsider a range around the previous boundary.

The exact semantics depend on whether the source is append-only, updateable, or event-based.

---

# 44. Incremental Reconciliation

Incremental pipelines should not be trusted forever without periodic verification.

Possible reconciliation methods include:

- row counts by time window
- sums by day
- distinct key counts
- source/target comparison
- freshness checks
- missing-key detection
- periodic full comparison

For example:

~~~text
source count for hour
vs
target count for hour
~~~

Periodic reconciliation can catch problems that normal checkpointing does not.

---

# 45. Recovery From a Missed Window

Suppose an incremental pipeline failed from:

~~~text
10:00 -> 12:00
~~~

A recovery operation can process that bounded interval:

~~~text
10:00 <= timestamp < 12:00
~~~

This becomes a small backfill.

This is an important relationship:

> Incremental processing failures are often repaired with controlled backfills.

---

# 46. Recovery From a Bad Cursor

Suppose the checkpoint is accidentally advanced too far.

For example:

~~~text
actual processed:
1 -> 5000

checkpoint:
10000
~~~

Records 5001-10000 may be skipped.

Recovery requires determining the missing range and rebuilding it.

Possible approaches include:

- restore a previous checkpoint
- replay an overlap window
- run a bounded backfill
- reconcile source and target

Never blindly move the cursor backward without understanding the target's current state.

---

# 47. Incremental Processing and Deletes

If deletes are important, include them in the design from the beginning.

A complete incremental synchronization model may be:

~~~text
source change
     |
     +---- insert
     |
     +---- update
     |
     +---- delete
~~~

Ignoring deletes can leave stale records in the target indefinitely.

---

# 48. Production Workflow

A practical incremental workflow is:

~~~text
1. Read checkpoint
        |
        v
2. Determine safe processing window
        |
        v
3. Select new/changed records
        |
        v
4. Process a batch
        |
        v
5. Validate results
        |
        v
6. Write target
        |
        v
7. Commit
        |
        v
8. Advance checkpoint
        |
        v
9. Continue next batch
        |
        v
10. Reconcile and monitor
~~~

The exact transaction and checkpoint boundary depends on the architecture.

---

# 49. Testing Incremental Processing

At minimum, test:

### Test 1 — New record

Expected:

~~~text
new record is processed
~~~

### Test 2 — Unchanged record

Expected:

~~~text
unchanged record is not unnecessarily processed
~~~

### Test 3 — Updated record

Expected:

~~~text
updated record reaches target
~~~

### Test 4 — Duplicate selection

Expected:

~~~text
target remains correct
~~~

### Test 5 — Same timestamp

Expected:

~~~text
records sharing a timestamp are not skipped
~~~

### Test 6 — Late-arriving record

Expected:

~~~text
late record is eventually processed
~~~

### Test 7 — Worker failure

Expected:

~~~text
processing resumes safely
~~~

### Test 8 — Checkpoint failure

Expected:

~~~text
repeated processing does not corrupt target
~~~

### Test 9 — Delete

Expected:

~~~text
deleted source record is handled correctly
~~~

### Test 10 — Empty run

Expected:

~~~text
no new data
    |
    v
safe no-op
~~~

---

# 50. Testing Overlap Windows

If an overlap window is used, test:

~~~text
record processed in run 1
      |
      v
same record selected in run 2
      |
      v
target remains correct
~~~

This verifies that overlap and idempotency work together.

Also test a record that arrives inside the overlap but was not visible during the first run.

---

# 51. Testing Cursor Boundaries

Boundary tests are important.

For a cursor at:

~~~text
id = 100
~~~

Test:

~~~text
id = 99
id = 100
id = 101
~~~

The expected selection should be explicit.

For timestamps, test:

~~~text
before boundary
exactly at boundary
after boundary
~~~

Many incremental bugs occur at boundaries.

---

# 52. Testing Late Data

Create a test where:

~~~text
event time = 10:03
arrival time = 10:15
~~~

Then verify the pipeline eventually processes the record.

This test should match the actual late-data strategy.

---

# 53. Testing Recovery

Simulate:

~~~text
batch 1 -> success
batch 2 -> success
batch 3 -> crash
~~~

Restart the worker.

Verify:

~~~text
batch 3 is safely recovered
batch 1 and 2 are not corrupted
checkpoint is correct
~~~

This is one of the most important incremental-processing tests.

---

# 54. Common Mistakes

## Mistake 1 — Using only current date

For example:

~~~text
updated_at >= today
~~~

This can miss late records and make reruns difficult.

Use explicit state and boundaries.

---

## Mistake 2 — Using a non-unique timestamp

Multiple records can share the same timestamp.

A composite cursor may be required.

---

## Mistake 3 — Advancing the checkpoint before writing the target

This can permanently skip records.

---

## Mistake 4 — No overlap

Strict boundaries can miss late-arriving data.

---

## Mistake 5 — Overlap without idempotency

This can create duplicate target writes.

---

## Mistake 6 — Ignoring updates

A pipeline that only looks for new records may never process changes to existing records.

---

## Mistake 7 — Ignoring deletes

The target can become stale.

---

## Mistake 8 — Ignoring timezone behavior

Timestamp boundaries can become inconsistent.

---

## Mistake 9 — No reconciliation

A pipeline can keep running while silently missing data.

---

## Mistake 10 — Assuming the cursor is trustworthy

A corrupted or incorrectly advanced cursor can create a data gap.

---

# 55. Troubleshooting

## Problem: records are missing

Check:

1. checkpoint value
2. cursor ordering
3. timestamp precision
4. late-arriving data
5. timezone conversion
6. source updates
7. deleted or changed records
8. boundary conditions

---

## Problem: records are processed repeatedly

Check:

1. overlap window
2. checkpoint updates
3. worker retries
4. idempotency
5. target uniqueness
6. cursor persistence

Repeated processing is not automatically a bug if the target remains correct.

---

## Problem: updated records are not reaching the target

Check:

1. source update timestamp
2. query filter
3. cursor state
4. update detection
5. target upsert logic
6. source transaction visibility

---

## Problem: late records are missed

Check:

1. overlap window
2. watermark
3. event time vs processing time
4. source replication delay
5. source clock
6. reconciliation process

---

## Problem: incremental query is slow

Check:

1. indexes
2. query plan
3. cursor design
4. batch size
5. unnecessary joins
6. partition pruning
7. source table growth

Do not solve a query problem by blindly increasing worker concurrency.

---

## Problem: cursor moved too far

First determine:

~~~text
which records were skipped?
~~~

Then compare:

~~~text
source
vs
target
~~~

Use a controlled backfill or replay to repair the gap.

Do not simply reset the cursor and hope the target becomes correct.

---

# 56. Production Considerations

Before putting incremental processing into production, consider:

### Change signal

Know exactly how new or changed records are identified.

### Cursor

Use a stable and deterministic cursor.

### Checkpoint

Persist progress safely.

### Boundaries

Define inclusive and exclusive behavior clearly.

### Overlap

Consider whether late data requires an overlap window.

### Idempotency

Make repeated processing safe.

### Updates

Handle changes to existing records.

### Deletes

Define how deletions are propagated.

### Late data

Define how delayed records are recovered.

### Batching

Keep processing manageable.

### Resource usage

Protect normal workloads.

### Reconciliation

Periodically verify source and target.

### Recovery

Know how to repair missed windows.

### Observability

Monitor lag, throughput, errors, and checkpoint progress.

---

# 57. Definition of Done

Incremental processing is complete when:

- [ ] The source change signal is identified.
- [ ] Full vs incremental behavior is documented.
- [ ] The cursor strategy is defined.
- [ ] Cursor ordering is deterministic.
- [ ] Checkpoint storage is defined.
- [ ] Checkpoint advancement is safe.
- [ ] Boundary behavior is tested.
- [ ] Batch processing is defined.
- [ ] Idempotency is implemented or otherwise guaranteed.
- [ ] Overlap behavior is defined where needed.
- [ ] Late-arriving data behavior is defined.
- [ ] Updates are handled.
- [ ] Deletes are handled where required.
- [ ] Timezone behavior is defined.
- [ ] Failure recovery is tested.
- [ ] Retry behavior is defined.
- [ ] Quarantine behavior is defined.
- [ ] Reconciliation checks exist.
- [ ] Production resource usage is understood.
- [ ] Monitoring is available.
- [ ] Cursor corruption recovery is understood.
- [ ] Missed-window recovery is documented.
- [ ] Empty runs are handled safely.

---

# 58. What You Learned

Incremental processing is about maintaining a reliable boundary between what has been processed and what still needs to be processed.

The basic model is:

~~~text
read checkpoint
      |
      v
define safe window
      |
      v
select new/changed records
      |
      v
process batch
      |
      v
write target
      |
      v
commit
      |
      v
advance checkpoint
      |
      v
repeat
~~~

The most important lessons are:

1. Incremental processing is not simply filtering by today's date.
2. A stable change signal is required.
3. High-water marks help track progress.
4. Timestamps may require composite cursors.
5. Checkpoints must advance only after successful processing.
6. Overlap windows can protect against late-arriving data.
7. Overlap requires idempotency.
8. Updates and deletes need explicit handling.
9. Watermarks and high-water marks solve different problems.
10. Incremental failures can often be repaired with bounded backfills.
11. Periodic reconciliation is important.
12. Boundary conditions must be tested.
13. The cursor is part of the pipeline's correctness model.
14. An incremental pipeline must be observable and recoverable.

The key question is not:

> What records arrived since the last run?

The better question is:

> What is the safest, deterministic set of new or changed records I can process now without missing data or corrupting the target?

That is the foundation of reliable incremental processing.

---

# Chapter 29 Preview

The next recipe will cover **Checkpointing**.

We will go deeper into the state that allows a pipeline to resume safely after interruption.

We will cover:

- what a checkpoint is
- checkpoint vs cursor
- checkpoint vs watermark
- where checkpoint state should live
- atomic checkpoint updates
- batch checkpoints
- partition checkpoints
- worker crashes
- partial processing
- checkpoint corruption
- recovery
- replay after checkpoint failure
- checkpoint verification
- production monitoring

The key question will be:

> How do we record pipeline progress so that a worker can stop at any point and resume without losing or duplicating data?## Production Tools You Should Know

These tools solve or provide production implementations of concepts covered in this recipe. Learn what the tool provides, but understand the underlying problem first.

| Tool | What to know |
|---|---|
| **dbt** | Incremental analytical transformations. |
| **Apache Spark** | Incremental batch processing at scale. |
| **Apache Kafka** | Offset-based incremental event consumption. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the engineering mechanism.

---




## Implementation Lab — Incremental Cursor

### 1. Use a stable cursor

~~~python
from dataclasses import dataclass

@dataclass
class Cursor:
    occurred_at: str
    event_id: str

def is_after(event, cursor):
    return (event["occurred_at"], event["event_id"]) > (cursor.occurred_at, cursor.event_id)
~~~

The second field makes equal timestamps deterministic.

### 2. Use overlap safely

Read a small overlap window and rely on idempotency at the target boundary.

### 3. Intentional failure drill

Advance the cursor before writing the target. Crash between those operations. The cursor now says processed while the target is missing data.

Restore: read → write durable result → advance cursor.

### 4. Recovery

Move the cursor back to a safe checkpoint and replay the affected range idempotently.

---

