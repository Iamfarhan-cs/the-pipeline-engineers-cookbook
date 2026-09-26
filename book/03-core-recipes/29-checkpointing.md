# Chapter 29 — Checkpointing

A pipeline needs to know where it stopped.

If a worker processes one million records and crashes after 700,000, the next worker needs a reliable answer to:

> Which records have definitely been processed?

That is the purpose of checkpointing.

A checkpoint is saved progress.

It tells the pipeline where it can safely continue.

The basic idea is:

~~~text
source
  |
  v
process records
  |
  v
save progress
  |
  v
continue
~~~

Without checkpoints, recovery often becomes guesswork.

With checkpoints, recovery can become a controlled operation.

But a checkpoint is more than a number in a database.

It is part of the pipeline's correctness model.

If the checkpoint moves too early, records can be lost.

If it moves too late, records may be processed again.

If it becomes corrupted, the pipeline may resume from the wrong place.

The central idea is:

> A checkpoint should represent work that is safely completed, not work that merely started.


---

# 1. Goal

The goal of this recipe is to understand how checkpointing allows a pipeline to stop and resume safely.

By the end of this recipe, you should understand how to:

- define a checkpoint
- distinguish checkpoints from cursors
- distinguish checkpoints from watermarks
- choose checkpoint storage
- checkpoint batches
- checkpoint partitions
- update checkpoints safely
- handle worker crashes
- handle partial processing
- recover from checkpoint corruption
- verify checkpoint state
- combine checkpoints with idempotency
- replay after checkpoint problems
- monitor checkpoint progress

The central principle is:

> The checkpoint is part of the pipeline's state, so it must be designed and protected like any other important production data.


---

# 2. Problem

Suppose a worker receives:

~~~text
1,000,000 records
~~~

It processes:

~~~text
700,000 records
~~~

Then the worker crashes.

The system now needs to determine:

~~~text
What was completed?
What was not completed?
What can safely be repeated?
Where should processing resume?
~~~

Without checkpointing:

~~~text
worker crashes
     |
     v
unknown position
     |
     v
manual investigation
~~~

With checkpointing:

~~~text
worker crashes
     |
     v
read checkpoint
     |
     v
resume from known position
~~~

This does not automatically guarantee correctness.

The checkpoint itself must be correct.


---

# 3. What Is a Checkpoint?

A checkpoint is persistent state representing processing progress.

A simple checkpoint could be:

~~~text
last_processed_id = 500000
~~~

Another could be:

~~~text
last_processed_timestamp = 2026-09-26T10:00:00
~~~

A streaming system may use:

~~~text
partition = 3
offset = 982341
~~~

A batch pipeline may use:

~~~text
batch_number = 70
~~~

The exact representation depends on the processing model.

The common idea is:

~~~text
processing state
       |
       v
checkpoint
       |
       v
resume position
~~~


---

# 4. Checkpoint vs Cursor

These concepts are closely related but should not be treated as identical.

### Cursor

A cursor identifies where to read from.

For example:

~~~text
id > 500000
~~~

### Checkpoint

A checkpoint records the processing progress that has been safely completed.

For example:

~~~text
last_successful_id = 500000
~~~

A pipeline may use the same value for both.

But the meanings are different.

The cursor is about selection.

The checkpoint is about committed progress.


---

# 5. Checkpoint vs Watermark

A watermark usually represents a boundary up to which data is considered safe or complete enough to process.

For example:

~~~text
current time = 10:30
watermark    = 10:20
~~~

A checkpoint may then record:

~~~text
processed through = 10:15
~~~

The two values can be different.

Conceptually:

~~~text
source timeline
---------------------------------------------------->

       checkpoint        watermark       current time
           |                 |                 |
           v                 v                 v
          10:15             10:20             10:30
~~~

The exact relationship depends on the pipeline.

Do not use the terms interchangeably without defining their meaning.


---

# 6. Why Checkpointing Matters

Checkpointing supports:

- worker recovery
- restart after deployment
- retry after failure
- batch processing
- streaming processing
- partitioned processing
- incremental processing
- replay
- backfill recovery

It also makes pipeline progress observable.

For example:

~~~text
processed = 650,000
checkpoint = 650,000
~~~

An operator can see how far the pipeline has progressed.


---

# 7. Before You Start

Before adding checkpointing, investigate the existing pipeline.

Find:

1. Where processing starts.
2. How records are selected.
3. How records are ordered.
4. How batches are created.
5. Where successful processing is committed.
6. Where processing state is stored.
7. Whether the target is idempotent.
8. How retries work.
9. How workers restart.
10. Whether multiple workers can process the same data.
11. Whether the source can change during processing.
12. Whether partitions exist.
13. How failures are currently recorded.

Checkpointing should fit the existing processing model.


---

# 8. Choosing a Checkpoint Value

The checkpoint should correspond to a stable processing boundary.

Possible values include:

~~~text
last_processed_id
last_processed_timestamp
last_processed_event_id
batch_number
partition + offset
sequence_number
version
~~~

The correct value depends on the source.

A good checkpoint should be:

- deterministic
- persistent
- recoverable
- meaningful
- associated with a processing scope
- safe to compare
- sufficient for resuming


---

# 9. ID-Based Checkpoint

An increasing ID is one of the simplest checkpoint models.

Suppose records are:

~~~text
1
2
3
...
100
~~~

After processing through 100:

~~~text
checkpoint = 100
~~~

Next selection:

~~~text
id > 100
~~~

This works well when:

- IDs are stable
- IDs increase consistently
- records are not inserted with old IDs
- the ordering is meaningful

Verify these assumptions before using this approach.


---

# 10. Timestamp Checkpoint

A timestamp can also represent progress.

For example:

~~~text
checkpoint = 2026-09-26T10:00:00
~~~

The next run may select:

~~~text
updated_at > checkpoint
~~~

But timestamps can have:

- duplicate values
- precision limitations
- clock differences
- late-arriving records
- timezone issues

A timestamp checkpoint often works better with a secondary ordering key.


---

# 11. Composite Checkpoint

A composite checkpoint can contain:

~~~text
(updated_at, id)
~~~

For example:

~~~text
updated_at = 10:00:00
id = 105
~~~

This allows the pipeline to resume precisely after the last processed record.

Conceptually:

~~~text
10:00:00, 103
10:00:00, 104
10:00:00, 105  <- checkpoint
10:00:00, 106
10:00:01, 107
~~~

The next position starts after:

~~~text
(10:00:00, 105)
~~~

This can be safer than timestamp-only checkpointing.


---

# 12. Batch Checkpoint

A batch pipeline can checkpoint after each successful batch.

For example:

~~~text
batch 1 -> success -> checkpoint 1
batch 2 -> success -> checkpoint 2
batch 3 -> success -> checkpoint 3
batch 4 -> failure
~~~

After restart:

~~~text
checkpoint = 3
~~~

The pipeline resumes from batch 4.

The exact resume behavior still depends on whether a failed batch may have partially committed work.


---

# 13. Record Checkpoint vs Batch Checkpoint

### Record checkpoint

Progress is saved at a fine-grained level.

~~~text
record 1001
record 1002
record 1003
...
~~~

This can provide precise recovery.

But frequent checkpoint writes may be expensive.

### Batch checkpoint

Progress is saved after a batch.

~~~text
batch 1
batch 2
batch 3
~~~

This reduces checkpoint overhead.

But a crash inside a batch may require the batch to be processed again.

Idempotency makes this safer.


---

# 14. The Most Important Rule

Never advance the checkpoint before the work it represents is safely committed.

Unsafe flow:

~~~text
read batch
    |
    v
advance checkpoint
    |
    v
write target
    |
    X
failure
~~~

Now the checkpoint says the batch is complete even though the target write failed.

This can permanently skip data.

Safer flow:

~~~text
read batch
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
~~~

The exact atomicity depends on the architecture, but the ordering must be deliberate.


---

# 15. Checkpoint and Target Must Agree

Consider:

~~~text
target says:
record processed

checkpoint says:
record not processed
~~~

The record may be processed again.

That is not necessarily incorrect if the target is idempotent.

Now consider:

~~~text
checkpoint says:
record processed

target says:
record not processed
~~~

This is dangerous.

The next run may skip the record permanently.

This is why checkpoint design must be connected to target commit behavior.


---

# 16. Checkpoint and Idempotency

Checkpointing and idempotency solve different problems.

Checkpointing answers:

> Where should I resume?

Idempotency answers:

> What happens if I process something again?

Together:

~~~text
checkpoint
     +
idempotency
     |
     v
safe recovery
~~~

If a worker crashes after the target commit but before the checkpoint update:

~~~text
target = processed
checkpoint = old
~~~

The batch may run again.

Idempotency prevents the second execution from corrupting the target.


---

# 17. Checkpoint Storage

Checkpoint state can be stored in different places.

Possible options include:

- PostgreSQL table
- workflow metadata
- object storage
- local file
- message-system offset
- distributed state store

The choice depends on the architecture.

For a database-backed pipeline, a database table may be practical because processing state and data state can sometimes be coordinated.


---

# 18. Generic Checkpoint Table

A generic PostgreSQL design might look like:

~~~sql
CREATE TABLE pipeline_checkpoints (
    pipeline_name TEXT PRIMARY KEY,
    checkpoint_value TEXT,
    status TEXT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
~~~

This is a **generic example**.

A real design may need:

- partition information
- run ID
- cursor type
- version
- timestamps
- error information
- ownership
- concurrency control

Do not copy this schema without understanding the actual pipeline.


---

# 19. One Checkpoint Per Pipeline

A simple pipeline may have:

~~~text
pipeline A
   |
   v
checkpoint A
~~~

This is easy to reason about.

But it may not work when multiple independent processing streams exist.


---

# 20. One Checkpoint Per Partition

A partitioned system may need:

~~~text
pipeline
   |
   +--> partition 1 -> checkpoint 1000
   +--> partition 2 -> checkpoint 850
   +--> partition 3 -> checkpoint 920
~~~

This allows partitions to progress independently.

The checkpoint identity must include the partition.

For example:

~~~text
pipeline_name + partition_id
~~~

The exact model depends on the processing system.


---

# 21. Checkpoint Ownership

With multiple workers, consider who owns a checkpoint.

For example:

~~~text
worker A -> partition 1
worker B -> partition 2
~~~

The system needs to prevent two workers from incorrectly updating the same checkpoint.

Possible mechanisms include:

- row locks
- leases
- partition ownership
- compare-and-set updates
- coordinator state

The correct mechanism depends on the architecture.


---

# 22. Concurrent Checkpoint Updates

Consider:

~~~text
worker A:
checkpoint 100 -> 200

worker B:
checkpoint 100 -> 150
~~~

If worker B writes after worker A, the checkpoint may move backward:

~~~text
200 -> 150
~~~

This can cause repeated processing or inconsistent progress.

Checkpoint updates therefore need concurrency protection.


---

# 23. Monotonic Progress

For many pipelines, checkpoint progress should move forward.

For example:

~~~text
100
 |
 v
200
 |
 v
300
 |
 v
400
~~~

Unexpected backward movement should be treated as suspicious.

A checkpoint should not normally move:

~~~text
400 -> 250
~~~

unless the system explicitly supports rollback or controlled recovery.


---

# 24. Checkpoint Versioning

Checkpoint state can sometimes require a version.

For example:

~~~text
checkpoint_version = 2
checkpoint_value = ...
~~~

This can help when the checkpoint format changes.

For example:

~~~text
version 1:
last_id

version 2:
(updated_at, id)
~~~

Changing checkpoint semantics requires careful migration.

Do not reinterpret an old checkpoint using a new cursor format without a defined transition.


---

# 25. Atomic Checkpoint Updates

The strongest designs make progress updates atomic with the work they represent when possible.

For example:

~~~text
transaction
    |
    +--> write target
    |
    +--> update checkpoint
    |
    v
commit
~~~

Then either both changes commit or neither does.

This is easier when the target and checkpoint are in the same transactional system.

If they are separate systems:

~~~text
source
  |
  v
processing
  |
  +--> target system
  |
  +--> checkpoint system
~~~

atomicity becomes harder.

The design may require idempotency, transactional messaging, a durable workflow state, or reconciliation.

Do not assume cross-system writes are atomic.


---

# 26. Worker Crash Scenarios

Checkpointing should be tested against different crash points.

### Crash before processing

~~~text
read batch
   |
   X
crash
~~~

No progress should be recorded.

### Crash during processing

~~~text
process batch
   |
   X
crash
~~~

The batch may need to be retried.

### Crash after target commit

~~~text
write target
   |
   v
commit
   |
   X
crash
~~~

The checkpoint may still point to the previous position.

The batch may run again.

Idempotency becomes important.

### Crash after checkpoint commit

~~~text
write target
   |
   v
checkpoint
   |
   v
crash
~~~

The next run should continue after the checkpoint.


---

# 27. The Dangerous Crash Window

The most dangerous scenario is:

~~~text
checkpoint updated
      |
      X
target write fails
~~~

This creates:

~~~text
checkpoint = advanced
target = incomplete
~~~

The pipeline may skip the missing work.

This is why checkpoint ordering matters.


---

# 28. Checkpoint and Transactions

For a database-backed pipeline, one possible pattern is:

~~~text
BEGIN

process batch

write target

update checkpoint

COMMIT
~~~

If anything fails:

~~~text
ROLLBACK
~~~

This can provide strong consistency when the target and checkpoint are in the same database transaction.

But this is a design option, not a universal requirement.

For external systems, different coordination strategies are needed.


---

# 29. Checkpoint and External Systems

Suppose the pipeline writes to:

~~~text
PostgreSQL
~~~

and:

~~~text
external API
~~~

A single database transaction cannot automatically roll back the API call.

The flow:

~~~text
API call
   |
   v
database write
   |
   v
checkpoint
~~~

can fail at different points.

This is why external side effects need their own idempotency and recovery design.

Do not treat the checkpoint as proof that every external side effect was successfully completed unless the system can actually guarantee that.


---

# 30. Checkpoint Recovery

When a worker starts, it should read checkpoint state.

Conceptually:

~~~text
worker starts
     |
     v
read checkpoint
     |
     v
validate checkpoint
     |
     v
determine resume position
     |
     v
process
~~~

The validation step matters.

A corrupt checkpoint should not automatically become the new processing boundary.


---

# 31. Checkpoint Validation

Before trusting a checkpoint, verify:

- value has valid format
- referenced partition exists
- cursor is within expected range
- checkpoint version is supported
- checkpoint is not unexpectedly in the future
- ownership is valid
- state is internally consistent

The exact checks depend on the system.


---

# 32. Checkpoint Corruption

Checkpoint corruption can happen because of:

- manual edits
- migration mistakes
- software bugs
- partial writes
- incorrect worker coordination
- storage failures
- bad deployment

Suppose the checkpoint says:

~~~text
last_id = 9000000
~~~

but the source currently contains only:

~~~text
5000000 records
~~~

That checkpoint should be investigated.

Do not blindly continue.


---

# 33. Recovery From Checkpoint Corruption

A controlled recovery process is:

~~~text
detect invalid checkpoint
       |
       v
stop processing
       |
       v
inspect source and target
       |
       v
identify last safe position
       |
       v
restore checkpoint
       |
       v
replay/backfill affected range
       |
       v
verify
~~~

The correct recovery depends on how much state can be trusted.


---

# 34. Checkpoint Backups

For important pipelines, checkpoint state should be recoverable.

Depending on the storage system, this may involve:

- database backups
- versioned state
- audit history
- checkpoint history
- replicated storage

The goal is not necessarily to back up every checkpoint value forever.

The goal is to have enough history to recover from state corruption.


---

# 35. Checkpoint History

Instead of storing only:

~~~text
current checkpoint = 500000
~~~

a system may retain history:

~~~text
run 101 -> 450000
run 102 -> 470000
run 103 -> 500000
~~~

This can help investigate:

- unexpected jumps
- backward movement
- stalled progress
- deployment-related changes
- worker coordination problems

A history table can be useful for operational debugging.


---

# 36. Checkpoint Progress Monitoring

Useful metrics include:

~~~text
current checkpoint
records processed
records remaining
checkpoint age
processing lag
checkpoint advancement rate
~~~

A healthy pipeline should normally show progress.

For example:

~~~text
10:00 -> checkpoint 100000
10:05 -> checkpoint 120000
10:10 -> checkpoint 140000
~~~

If the checkpoint stops moving, investigation may be required.


---

# 37. Checkpoint Lag

Checkpoint lag represents how far processing is behind the source.

For example:

~~~text
source position:
1,000,000

checkpoint:
900,000

lag:
100,000
~~~

For timestamp-based processing:

~~~text
current source time:
10:30

checkpoint:
10:10

lag:
20 minutes
~~~

The exact lag calculation depends on the source.


---

# 38. Stalled Checkpoint

A checkpoint may stop advancing because:

- worker is down
- records are failing
- database is unavailable
- external API is slow
- lock contention exists
- checkpoint update is failing
- processing is stuck on one record
- concurrency is misconfigured

Checkpoint monitoring helps identify these conditions.


---

# 39. Checkpoint and Empty Runs

Sometimes there is nothing new to process.

For example:

~~~text
checkpoint = 1000
source maximum = 1000
~~~

The pipeline should safely perform a no-op.

It should not:

- reset the checkpoint
- create duplicate work
- report a false failure
- corrupt progress state

An empty run is a valid pipeline outcome.


---

# 40. Checkpoint and Backfill

A backfill may need its own checkpoint.

For example:

~~~text
normal pipeline checkpoint
       |
       v
live processing

backfill checkpoint
       |
       v
historical processing
~~~

Do not automatically use the same checkpoint for both unless the architecture explicitly defines that behavior.

The two processes may represent different processing scopes.


---

# 41. Checkpoint and Replay

Replay can also require separate progress state.

For example:

~~~text
replay job
   |
   v
replay checkpoint
~~~

This allows a replay operation to stop and resume independently from the normal pipeline.

A useful design separates:

~~~text
live progress
historical progress
replay progress
~~~

when those operations have independent lifecycles.


---

# 42. Partition Checkpoints and Rebalancing

In partitioned systems, ownership can change.

For example:

~~~text
worker A owns partition 1
worker B owns partition 2
~~~

After rebalancing:

~~~text
worker C owns partition 1
worker B owns partition 2
~~~

The checkpoint should belong to the partition, not permanently to worker A.

This allows another worker to resume from the same progress point.


---

# 43. Checkpoint and Ordering

Checkpointing depends on the processing order.

Suppose records are processed in this order:

~~~text
1
2
4
3
5
~~~

A checkpoint of:

~~~text
4
~~~

does not necessarily mean record 3 was completed.

Therefore a simple numeric checkpoint is safe only when the processing order and checkpoint semantics guarantee that all earlier records have been handled.

This is why deterministic ordering matters.


---

# 44. Out-of-Order Processing

Some systems intentionally process records out of order.

For example:

~~~text
partition A:
1, 2, 3

partition B:
100, 101, 102
~~~

Each partition can have its own checkpoint.

But a single global checkpoint may not represent the true state.

Use the state model that matches the processing topology.


---

# 45. Checkpoint Granularity

Checkpoint frequency is a tradeoff.

Frequent checkpoints:

- reduce recovery work
- increase checkpoint overhead
- create more writes

Infrequent checkpoints:

- reduce overhead
- increase replay after failure
- may increase recovery time

For example:

~~~text
checkpoint every record
~~~

versus:

~~~text
checkpoint every 10,000 records
~~~

There is no universal correct value.

Measure the workload and choose intentionally.


---

# 46. Checkpoint and Performance

Checkpoint writes themselves consume resources.

If a pipeline processes:

~~~text
1,000,000 records
~~~

and writes a checkpoint after every record, it may create unnecessary database traffic.

Batch checkpoints can reduce overhead:

~~~text
10,000 records
     |
     v
one checkpoint
~~~

The correct frequency depends on:

- processing speed
- failure cost
- storage cost
- transaction behavior
- recovery requirements


---

# 47. Testing Checkpoint Behavior

Checkpointing needs dedicated tests.

At minimum, test:

### Test 1 — Successful batch

Expected:

~~~text
target committed
checkpoint advanced
~~~

### Test 2 — Processing failure

Expected:

~~~text
target changes rolled back where applicable
checkpoint not incorrectly advanced
~~~

### Test 3 — Crash after target commit

Expected:

~~~text
reprocessing is safe
~~~

### Test 4 — Crash before target commit

Expected:

~~~text
batch can be retried
~~~

### Test 5 — Invalid checkpoint

Expected:

~~~text
pipeline stops safely
~~~

### Test 6 — Empty source

Expected:

~~~text
safe no-op
~~~

### Test 7 — Concurrent workers

Expected:

~~~text
checkpoint cannot move backward incorrectly
~~~

### Test 8 — Partition recovery

Expected:

~~~text
new worker resumes from partition checkpoint
~~~


---

# 48. Testing Checkpoint Boundaries

For a checkpoint:

~~~text
100
~~~

test:

~~~text
99
100
101
~~~

Verify exactly which records are selected next.

For a timestamp:

~~~text
10:00:00
~~~

test:

~~~text
09:59:59
10:00:00
10:00:01
~~~

Boundary tests should also include duplicate timestamps when timestamps are part of the cursor.


---

# 49. Testing Checkpoint Corruption

Simulate:

~~~text
checkpoint:
invalid value
~~~

Then verify the pipeline:

1. detects the problem
2. stops safely
3. produces a useful error
4. does not silently skip data
5. provides enough information for recovery


---

# 50. Testing Worker Restart

Simulate:

~~~text
worker starts
   |
   v
processes batch
   |
   X
worker stops
~~~

Restart it.

Verify:

~~~text
checkpoint loaded
resume position correct
target remains correct
no data is permanently skipped
~~~

This should be part of the normal integration test suite for important pipelines.


---

# 51. Testing Backward Checkpoint Movement

If checkpoints are expected to be monotonic, test:

~~~text
checkpoint = 200
update to 150
~~~

The system should reject the invalid movement unless rollback is explicitly supported.

This protects against race conditions and incorrect workers.


---

# 52. Testing Duplicate Processing

Force the same batch to execute twice.

Expected:

~~~text
first execution -> target correct
second execution -> target still correct
~~~

This test connects checkpoint behavior to idempotency.


---

# 53. Troubleshooting

## Problem: checkpoint does not advance

Check:

1. target commit
2. checkpoint update
3. transaction behavior
4. worker errors
5. database permissions
6. concurrency conflicts
7. stuck records

---

## Problem: checkpoint advances but data is missing

This is a high-priority issue.

Check:

1. checkpoint update order
2. transaction boundary
3. target commit
4. worker crash timing
5. manual checkpoint changes
6. concurrent workers

The likely problem is that progress was recorded before the corresponding work was safely completed.

---

## Problem: checkpoint moves backward

Check:

1. multiple workers
2. partition ownership
3. concurrent updates
4. stale worker state
5. retry behavior
6. manual changes

---

## Problem: worker repeatedly processes the same batch

Check:

1. checkpoint persistence
2. checkpoint transaction
3. target commit
4. worker restart behavior
5. checkpoint write failures

Repeated processing may be safe if the target is idempotent, but it can still indicate inefficient recovery.

---

## Problem: checkpoint is invalid

Do not guess.

First inspect:

1. checkpoint history
2. source position
3. target state
4. deployment changes
5. recent worker behavior

Then determine the last known safe checkpoint.

---

## Problem: checkpoint is healthy but data is still missing

A correct checkpoint does not guarantee a correct source-selection strategy.

Check:

1. cursor logic
2. late data
3. source ordering
4. timestamp precision
5. timezone behavior
6. deletes
7. filtering conditions

The checkpoint may be accurately recording progress through an incomplete selection.


---

# 54. Production Considerations

Before deploying checkpointing, consider:

### Meaning

Define exactly what the checkpoint represents.

### Storage

Use durable storage.

### Atomicity

Coordinate checkpoint updates with processing where possible.

### Idempotency

Assume repeated work can occur.

### Ordering

Make source ordering deterministic.

### Concurrency

Prevent workers from corrupting shared progress.

### Partitions

Track independent progress when required.

### Recovery

Define how invalid or stale checkpoints are repaired.

### Monitoring

Track checkpoint movement and lag.

### History

Retain enough state to investigate unexpected changes.

### Performance

Balance checkpoint frequency against recovery cost.

### Security

Protect checkpoint state if it contains sensitive information.


---

# 55. Definition of Done

Checkpointing is complete when:

- [ ] The meaning of the checkpoint is documented.
- [ ] Cursor and checkpoint semantics are clearly separated.
- [ ] Watermark semantics are defined where applicable.
- [ ] Checkpoint storage is durable.
- [ ] Checkpoint identity is defined.
- [ ] Ordering is deterministic.
- [ ] Checkpoint advancement occurs only after safe processing.
- [ ] Transaction boundaries are understood.
- [ ] Idempotent recovery is supported.
- [ ] Concurrent checkpoint updates are controlled.
- [ ] Partition checkpoints are supported where required.
- [ ] Checkpoint validation exists.
- [ ] Invalid checkpoint recovery is documented.
- [ ] Checkpoint history is available where operationally necessary.
- [ ] Worker crash behavior is tested.
- [ ] Duplicate processing is tested.
- [ ] Boundary behavior is tested.
- [ ] Empty runs are tested.
- [ ] Checkpoint lag is observable.
- [ ] Stalled progress is detectable.
- [ ] Backfill and replay checkpoints are separated where appropriate.
- [ ] Recovery procedures are documented.


---

# 56. What You Learned

A checkpoint is not just a number.

It is a statement about completed work.

The basic model is:

~~~text
read source
    |
    v
process records
    |
    v
write target
    |
    v
commit successful work
    |
    v
advance checkpoint
~~~

The most important lessons are:

1. A checkpoint represents safely completed progress.
2. A cursor identifies where to read.
3. A watermark represents a completeness boundary.
4. Checkpoints must be durable.
5. Checkpoint updates must be carefully ordered.
6. Idempotency is essential because recovery may repeat work.
7. Concurrent workers need checkpoint coordination.
8. Partitioned systems often need partition-level checkpoints.
9. Checkpoint movement should normally be monotonic.
10. Checkpoint corruption requires controlled recovery.
11. Checkpoint history can make production incidents easier to investigate.
12. Checkpoint frequency is a tradeoff between recovery cost and overhead.
13. A checkpoint can be correct while the selection logic is still wrong.
14. Checkpoint behavior must be tested under failure, restart, concurrency, and boundary conditions.

The key question is not:

> Where did the worker stop?

The better question is:

> What work can the system prove was safely completed, and what is the safest position from which to continue?

That is what a production checkpoint should represent.


---

# Chapter 30 Preview

The next chapter begins **Part IV — Data Storage** with **PostgreSQL Pipeline**.

We will move from pipeline control and recovery into the database layer.

We will cover:

- PostgreSQL's role in Data Engineering
- source and target database responsibilities
- connections and connection pools
- transactions
- schemas
- tables
- indexes
- constraints
- inserts and upserts
- reading and writing pipeline data
- migrations
- database failures
- connection failures
- transaction failures
- performance considerations
- testing PostgreSQL pipelines
- production database practices

The key question will be:

> How do we use PostgreSQL as a reliable part of a Data Engineering pipeline rather than treating it as just a place to store rows?
