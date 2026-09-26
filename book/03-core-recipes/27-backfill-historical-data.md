# Chapter 27 — Backfill Historical Data

A production pipeline usually processes new data as it arrives.

But sometimes the normal pipeline is not enough.

You may need to process data from the past because:

- a new transformation was introduced
- historical records were missing
- a bug produced incorrect results
- a new column was added
- a source system was unavailable
- a downstream table needs to be rebuilt
- a new business rule must be applied to historical data
- a new data product requires older records

This is called a backfill.

A backfill sounds simple:

~~~text
read old data
     |
     v
process it
     |
     v
write the result
~~~

In production, it is much more complicated.

You must decide:

- which records should be processed
- which time range is included
- how to avoid duplicates
- how to protect normal workloads
- how to handle failures
- how to resume after interruption
- how to verify the result
- how to stop safely

The central idea is:

> A backfill is a controlled historical processing operation, not simply running yesterday's pipeline again.

---

# 1. Goal

The goal of this recipe is to understand how to design and execute a historical backfill safely.

By the end of this recipe, you should understand how to:

- identify when a backfill is required
- define a precise historical range
- select the correct source records
- separate backfill execution from normal processing
- process historical data in batches
- protect production workloads
- make backfills idempotent
- checkpoint progress
- handle failures
- stop and resume a backfill
- handle late or missing data
- verify backfill results
- reconcile source and target data
- monitor backfill progress
- avoid common backfill mistakes

The central principle is:

> Never start a large historical operation without knowing exactly what data it will touch and how you will recover if it stops halfway.

---

# 2. Problem

Suppose a pipeline processed January data incorrectly.

The normal pipeline is now working correctly.

You could run the pipeline again over January:

~~~text
January data
     |
     v
normal pipeline
     |
     v
target
~~~

But this can create problems.

The target may already contain January records.

Running the same processing again may create:

- duplicate rows
- duplicate events
- conflicting updates
- incorrect aggregates
- repeated external side effects

The historical operation therefore needs its own controlled design.

A better approach is:

~~~text
historical source
       |
       v
define backfill range
       |
       v
select records
       |
       v
process in batches
       |
       v
write safely
       |
       v
verify
~~~

---

# 3. Why Backfills Matter

Backfills are a normal part of Data Engineering.

They are not necessarily signs that a pipeline is broken.

Systems change over time.

A pipeline may originally calculate:

~~~text
amount
~~~

Later, a new requirement adds:

~~~text
amount
currency
exchange_rate
base_amount
~~~

Historical records may not contain the new calculated value.

A backfill can process older records using the new logic.

Another example:

~~~text
Bug introduced
      |
      v
Historical records incorrect
      |
      v
Bug fixed
      |
      v
Backfill affected period
      |
      v
Verify corrected results
~~~

---

# 4. When to Use a Backfill

A backfill may be appropriate when:

- historical data was missing
- historical data was incorrect
- a transformation changed
- a new derived field was introduced
- a new table needs historical population
- a new business rule needs historical application
- a source system recovered old data
- a failed historical processing window must be completed
- a downstream system needs older records

The exact decision depends on the system and the business requirement.

---

# 5. Backfill vs Replay

Backfill and replay are related but not identical.

### Replay

Replay usually means processing existing records again through a pipeline.

~~~text
existing records
      |
      v
process again
~~~

### Backfill

Backfill usually means intentionally processing a defined historical range or population to populate or correct data.

~~~text
historical range
      |
      v
backfill processing
      |
      v
target
~~~

There can be overlap.

For example:

~~~text
historical records
      |
      v
backfill job
      |
      v
replay processing
~~~

The operational difference is the purpose and scope.

---

# 6. Backfill vs Retry

A retry deals with a failed operation.

~~~text
record
  |
  v
processing
  |
  X
temporary failure
  |
  v
retry
~~~

A backfill is an intentional historical operation.

~~~text
January 1 -> January 31
          |
          v
       backfill
~~~

Do not use retries as a substitute for backfill planning.

---

# 7. Backfill Architecture

A simple architecture looks like this:

~~~text
                 Historical Source
                       |
                       v
                Define Scope
                       |
                       v
                 Select Records
                       |
                       v
                 Create Backfill
                       |
                       v
                  Process Batch
                       |
              +--------+--------+
              |                 |
           success            failure
              |                 |
              v                 v
         checkpoint          handle failure
              |                 |
              v                 |
         next batch <-----------+
              |
              v
          reconciliation
              |
              v
           completed
~~~

The backfill should have a clear lifecycle.

---

# 8. Before You Start

Before running a backfill, answer these questions:

1. Why is the backfill required?
2. What historical period is affected?
3. Which records are included?
4. Which records are excluded?
5. What source contains the authoritative data?
6. What target will be changed?
7. Will existing target data be replaced or updated?
8. Can the operation be safely rerun?
9. How large is the dataset?
10. How will the operation be batched?
11. How will progress be recorded?
12. What happens if the worker stops?
13. What happens if the database fails?
14. How will results be verified?
15. How will normal production traffic be protected?
16. How will the backfill be stopped?
17. How will it be resumed?
18. How long will the backfill remain operationally active?

Do not start execution until these questions have reasonable answers.

---

# 9. Define the Backfill Scope

The first important step is defining exactly what the backfill will process.

For example:

~~~text
start = 2026-01-01
end   = 2026-02-01
~~~

This should normally be treated as a bounded range.

A common pattern is:

~~~text
timestamp >= start
AND
timestamp < end
~~~

This is often safer than using an inclusive end timestamp because adjacent ranges can then be processed without overlap.

For example:

~~~text
Batch A:
2026-01-01 <= timestamp < 2026-02-01

Batch B:
2026-02-01 <= timestamp < 2026-03-01
~~~

The ranges meet cleanly.

---

# 10. Use a Precise Selection Rule

Do not define a backfill as:

~~~text
process old records
~~~

That is too vague.

Define:

- source
- date/time range
- record type
- status
- tenant or partition where applicable
- inclusion rules
- exclusion rules

For example:

~~~text
source = transaction_events
date >= 2026-01-01
date < 2026-02-01
status = completed
~~~

This is a generic example.

The real selection rule must come from the actual system.

---

# 11. Verify the Selection Before Processing

Before modifying anything, run the selection as a read-only query.

First determine:

~~~text
number of records
~~~

Then inspect:

~~~text
minimum timestamp
maximum timestamp
sample records
distinct sources
distinct statuses
~~~

For example:

~~~text
Expected range:
2026-01-01 -> 2026-02-01

Selected:
1,250,000 records

Minimum:
2026-01-01 00:00

Maximum:
2026-01-31 23:59
~~~

These numbers are examples only.

Do not proceed if the selection does not match the intended scope.

---

# 12. Estimate the Workload

Before execution, estimate:

- number of records
- average record size
- total data volume
- expected processing time
- database load
- API usage
- storage requirements

This helps determine batch size and scheduling.

A backfill that takes five minutes in development may take many hours in production.

---

# 13. Create a Backfill Run

For serious production systems, it can be useful to represent the backfill itself as an operational entity.

A generic backfill run may contain:

~~~text
backfill_id
name
source
start_range
end_range
status
created_at
started_at
completed_at
total_records
processed_records
failed_records
checkpoint
~~~

This is a **generic example**.

The exact design depends on the system.

The important idea is:

> A large historical operation should have an identity and observable state.

---

# 14. Backfill States

A backfill can have states such as:

~~~text
planned
   |
   v
running
   |
   +----> paused
   |         |
   |         v
   |       running
   |
   +----> failed
   |         |
   |         v
   |       running
   |
   +----> completed
   |
   +----> cancelled
~~~

These are generic example states.

The actual state machine should match operational requirements.

---

# 15. Plan the Batch Size

Do not process millions of records in one transaction unless the system is specifically designed for it.

Instead:

~~~text
1,000,000 records

batch 1 -> 10,000
batch 2 -> 10,000
batch 3 -> 10,000
...
batch 100 -> 10,000
~~~

Batch size should balance:

- throughput
- memory
- database load
- transaction size
- lock duration
- recovery time

There is no universal perfect batch size.

Measure and adjust.

---

# 16. Why Small Batches Help

Suppose a backfill processes 500,000 records in one transaction.

If it fails near the end:

~~~text
500,000 records
      |
      X
failure
~~~

You may lose a large amount of work.

With smaller batches:

~~~text
10,000 -> complete
10,000 -> complete
10,000 -> complete
10,000 -> failure
~~~

Only the affected batch needs recovery.

Smaller transactions can also reduce lock duration and resource pressure.

---

# 17. Backfill Checkpoints

A checkpoint records how far the backfill has progressed.

For example:

~~~text
last_processed_id = 500000
~~~

Or:

~~~text
last_processed_timestamp = 2026-01-15T12:00:00
~~~

Or another stable cursor.

The checkpoint should be based on a deterministic ordering.

For example:

~~~text
ORDER BY id
~~~

or:

~~~text
ORDER BY occurred_at, id
~~~

The second pattern is useful when timestamps can be identical.

The exact cursor depends on the source.

---

# 18. Checkpoint Requirements

A useful checkpoint should allow the process to answer:

> Where can I safely resume?

A checkpoint should be:

- deterministic
- persisted
- recoverable
- associated with the backfill run
- updated only after successful processing

Do not advance the checkpoint before the corresponding batch is safely completed.

---

# 19. Checkpoint Example

Suppose records are ordered by ID:

~~~text
1
2
3
...
10000
~~~

Batch 1:

~~~text
IDs 1-10000
~~~

After successful processing:

~~~text
checkpoint = 10000
~~~

Next run:

~~~text
IDs > 10000
~~~

If the worker crashes while processing IDs 10001-20000, the checkpoint remains:

~~~text
10000
~~~

The next execution can safely resume from there, assuming the processing operation is idempotent.

---

# 20. Checkpointing Is Not Enough

A checkpoint alone does not guarantee correctness.

Suppose:

~~~text
process batch
     |
     v
write target
     |
     X
worker crashes
     |
     v
checkpoint not updated
~~~

The batch may be processed again.

Therefore backfills also need idempotency.

The relationship is:

~~~text
checkpoint
    +
idempotent processing
    =
safe resume
~~~

---

# 21. Backfill Idempotency

A backfill should ideally be safe to run more than once.

For example:

~~~text
backfill batch
     |
     v
upsert target
~~~

or:

~~~text
backfill batch
     |
     v
unique key
     |
     v
duplicate prevented
~~~

The correct approach depends on the target model.

Possible strategies include:

- unique constraints
- upserts
- deterministic replacement
- partition replacement
- delete-and-rebuild for controlled ranges

Do not choose a strategy without understanding target semantics.

---

# 22. Replace vs Update vs Insert

A backfill may need to:

### Insert missing records

~~~text
source
  |
  v
target
~~~

### Update existing records

~~~text
old target
    |
    v
recalculated result
~~~

### Replace a historical partition

~~~text
old partition
     |
     v
rebuild
     |
     v
replace
~~~

The correct strategy depends on the target's meaning.

A backfill should never accidentally overwrite unrelated data.

---

# 23. Backfill and Transactions

Transactions should usually be scoped to manageable units.

For example:

~~~text
begin
  process batch
  write target
commit
~~~

If the batch fails:

~~~text
rollback
~~~

Then the batch can be retried or investigated.

Large historical operations should avoid unnecessarily huge transactions.

---

# 24. Backfill and Normal Production Processing

A backfill can compete with the normal pipeline.

For example:

~~~text
normal workers
       |
       v
     database
       ^
       |
backfill workers
~~~

Both may consume:

- CPU
- memory
- database connections
- disk I/O
- locks
- network bandwidth

A backfill should therefore be treated as production workload.

---

# 25. Protect Normal Workloads

Possible controls include:

- lower backfill concurrency
- smaller batches
- scheduling during lower-load periods
- separate worker pools
- rate limits
- database resource controls
- separate queues
- explicit pause controls

The exact approach depends on the architecture.

The normal pipeline should remain healthy while the backfill runs.

---

# 26. Backfill Read Load

Backfills can create large read workloads.

For example:

~~~text
SELECT millions of historical rows
~~~

This can affect normal queries.

Consider:

- indexes
- partition pruning
- bounded ranges
- cursor-based reads
- read replicas where appropriate
- controlled concurrency

Do not assume historical queries are free simply because the data already exists.

---

# 27. Backfill Write Load

Backfills can also create large write workloads.

Potential effects include:

- transaction log growth
- index maintenance
- lock contention
- storage growth
- replication lag
- downstream load

Monitor the database while the backfill runs.

---

# 28. Backfill and External APIs

Be careful if historical processing calls external APIs.

Suppose the backfill processes:

~~~text
1,000,000 historical records
~~~

and each record calls an API.

This can create:

~~~text
1,000,000 API requests
~~~

The external system may have rate limits.

It may also interpret the requests as new activity.

Before running such a backfill, determine whether external side effects should happen at all.

---

# 29. Avoid Repeating External Side Effects

A backfill should not accidentally send historical operations as new real-world actions.

Examples of dangerous side effects include:

- sending emails
- sending notifications
- creating payments
- creating external records
- triggering webhooks
- changing customer state

A common pattern is to separate:

~~~text
historical calculation
~~~

from:

~~~text
real-time side effects
~~~

If external effects are required, they should be explicitly designed and controlled.

---

# 30. Backfill Mode

Some systems introduce an explicit backfill mode.

For example:

~~~text
normal mode
backfill mode
replay mode
~~~

Backfill mode can change behavior such as:

- disabling notifications
- reducing concurrency
- using a different output
- writing audit metadata
- skipping external side effects

The exact implementation is system-specific.

Do not assume a backfill flag exists unless the repository actually implements one.

---

# 31. Backfill and Schema Changes

Backfills often happen after schema changes.

For example:

~~~text
new column added
      |
      v
historical rows do not have derived value
      |
      v
backfill
~~~

Before running the backfill, verify:

- migration is complete
- target schema exists
- application code supports the new schema
- historical records can be transformed
- old and new records remain compatible

Schema changes and backfills should be planned together.

---

# 32. Backfill and Versioned Logic

Suppose version 1 of a transformation produced:

~~~text
result_v1
~~~

Version 2 produces:

~~~text
result_v2
~~~

A backfill should make the processing version clear.

For example:

~~~text
backfill_run
processing_version = v2
~~~

This helps explain why historical results differ from older results.

The exact versioning strategy depends on the project.

---

# 33. Backfill and Data Contracts

Historical records may not match today's schema.

For example:

~~~text
2023 record:
field_a
field_b

2026 record:
field_a
field_b
field_c
~~~

The backfill must decide how older records map to the current model.

Possible approaches include:

- default values
- explicit transformations
- version-specific parsers
- migration logic
- excluding unsupported records

Do not silently assume missing historical fields have today's meaning.

---

# 34. Missing Historical Data

A backfill may discover that some source data is missing.

For example:

~~~text
expected:
January 1 -> January 31

available:
January 1 -> January 25
~~~

Do not report the backfill as complete simply because all available records were processed.

Track:

~~~text
expected range
available range
processed range
missing range
~~~

Missing data is a data-quality problem that should be visible.

---

# 35. Late Historical Data

Sometimes data arrives after the original historical window.

For example:

~~~text
January data
    |
    v
backfill runs
    |
    v
late January record arrives
~~~

The system needs a policy for this.

Possible approaches:

- rerun the affected range
- process late records separately
- maintain watermarks
- use incremental correction

The correct approach depends on the pipeline.

---

# 36. Backfill in Small Historical Windows

Instead of:

~~~text
January -> December
~~~

consider:

~~~text
January
February
March
...
December
~~~

This provides smaller units of work.

Each period can be:

- processed
- verified
- reconciled
- marked complete

This can make recovery easier.

---

# 37. Backfill Control Plane

For large production backfills, it can be useful to have a control record.

A generic structure might track:

~~~text
backfill_id
range_start
range_end
current_cursor
status
batch_size
processed_count
failed_count
started_at
updated_at
completed_at
~~~

This is a **generic example**.

The exact implementation depends on the pipeline.

The purpose is operational visibility.

---

# 38. Backfill Progress

A useful progress view might show:

~~~text
Backfill: January 2026

Total:
1,000,000

Processed:
650,000

Failed:
120

Remaining:
349,880

Progress:
65%
~~~

These values are examples only.

Progress should be based on meaningful counts.

Do not report 90% completion merely because 90% of time has passed.

---

# 39. Backfill Failure Handling

A backfill can fail at several levels.

### Record-level failure

~~~text
one record fails
~~~

The batch may continue if the design supports isolation.

### Batch-level failure

~~~text
batch fails
~~~

The batch can be retried or investigated.

### Infrastructure failure

~~~text
database unavailable
worker crash
network outage
~~~

The backfill should stop or pause safely.

### Logic failure

~~~text
new transformation is incorrect
~~~

The backfill may need to be stopped immediately.

---

# 40. Stop Conditions

A production backfill should have explicit stop conditions.

For example:

- error rate exceeds threshold
- database load becomes unsafe
- replication lag becomes too large
- unexpected schema appears
- data-quality checks fail
- target counts do not reconcile
- external API rate limit is reached

Stopping early can prevent a small problem from becoming a large data incident.

---

# 41. Dry Run

A dry run can be useful before modifying data.

A dry run may:

- calculate record counts
- validate selection criteria
- inspect samples
- estimate workload
- validate transformations
- identify expected failures

For example:

~~~text
DRY RUN

Range:
2026-01-01 -> 2026-02-01

Records:
1,250,000

Expected target changes:
1,250,000

Validation failures:
120
~~~

These values are examples only.

A dry run should not modify production data.

---

# 42. Test on a Small Sample

Before a large backfill:

~~~text
1,000,000 records
        |
        v
first test
        |
        v
100 records
~~~

Verify:

- transformation results
- target writes
- duplicates
- constraints
- performance
- logs
- metrics

Then increase gradually.

For example:

~~~text
100
  ->
1,000
  ->
10,000
  ->
100,000
~~~

The actual progression depends on the system.

---

# 43. Reconciliation

After processing, compare source and target.

Useful checks include:

### Count reconciliation

~~~text
source count
vs
target count
~~~

### Key reconciliation

~~~text
source IDs
vs
target IDs
~~~

### Aggregate reconciliation

~~~text
source total amount
vs
target total amount
~~~

### Time-range reconciliation

~~~text
source min/max timestamp
vs
target min/max timestamp
~~~

Counts alone may not prove correctness.

---

# 44. Sample-Based Verification

In addition to aggregate checks, inspect individual records.

For example:

~~~text
source record
      |
      v
expected transformation
      |
      v
target record
~~~

Check:

- identifiers
- timestamps
- transformed values
- derived fields
- relationships

A backfill can have the correct row count and still contain incorrect values.

---

# 45. Reconciliation Failures

Suppose:

~~~text
source:
1,000,000

target:
999,700
~~~

Do not simply mark the backfill completed.

Investigate the difference.

Possible causes include:

- filtered records
- validation failures
- duplicate handling
- missing source data
- transaction failures
- target constraints
- incorrect selection
- late-arriving data

The difference needs an explanation.

---

# 46. Backfill and Quarantine

Backfill processing can also produce quarantine records.

For example:

~~~text
historical records
      |
      v
backfill
      |
      +---- valid ----> target
      |
      +---- invalid --> quarantine
~~~

The backfill should report both:

~~~text
processed successfully
processed to quarantine
failed unexpectedly
~~~

Do not hide quarantined records inside a generic failure count.

---

# 47. Backfill and Replay

Backfill and replay can work together.

For example:

~~~text
historical source
      |
      v
backfill
      |
      v
replay processing
      |
      +---- success
      |
      +---- quarantine
~~~

Chapter 25 covered replay.

Chapter 26 covered quarantine.

This chapter combines those ideas into a larger historical operation.

---

# 48. Backfill Recovery

Suppose the worker crashes at:

~~~text
650,000 / 1,000,000
~~~

A safe recovery flow is:

~~~text
read backfill state
      |
      v
read checkpoint
      |
      v
verify target state
      |
      v
resume next safe batch
~~~

Do not simply restart from zero unless the operation is known to be safe and efficient.

---

# 49. Resume After Partial Completion

A backfill can be interrupted because of:

- deployment
- worker crash
- database outage
- network failure
- operator cancellation
- infrastructure restart

The recovery process should answer:

> What work has definitely completed?

and:

> What work may have completed but was not checkpointed?

This is where idempotency becomes critical.

---

# 50. Backfill Cancellation

Operators should be able to stop a backfill safely.

A cancellation flow might be:

~~~text
running
   |
   v
cancel requested
   |
   v
finish current safe unit
   |
   v
cancelled
~~~

Do not abruptly terminate a process in the middle of a critical transaction if a graceful stop is possible.

---

# 51. Backfill Scheduling

Large backfills may be scheduled during lower-load periods.

Possible strategies include:

- overnight execution
- controlled maintenance windows
- limited concurrency during business hours
- automatic pause during high load

Scheduling should not replace resource controls.

A backfill can still overload a system at night.

---

# 52. Backfill Observability

Monitor at least:

~~~text
records processed
records remaining
records failed
records quarantined
processing rate
batch duration
error rate
database load
queue depth
replication lag
API rate limits
~~~

The exact metrics depend on the architecture.

Logs should include enough context to identify:

- backfill run
- batch
- range
- processing version
- failure reason

Avoid logging sensitive payloads.

---

# 53. Backfill Alerts

Useful alerts may include:

- processing stops unexpectedly
- error rate increases
- progress stops
- batch duration increases sharply
- database load becomes unsafe
- replication lag exceeds threshold
- quarantine volume increases unexpectedly
- reconciliation fails

The goal is to detect when the backfill is no longer behaving as expected.

---

# 54. Production Backfill Workflow

A practical workflow is:

~~~text
1. Define the reason
        |
        v
2. Define historical scope
        |
        v
3. Identify authoritative source
        |
        v
4. Inspect target state
        |
        v
5. Estimate workload
        |
        v
6. Run dry validation
        |
        v
7. Test small sample
        |
        v
8. Start controlled backfill
        |
        v
9. Monitor progress
        |
        v
10. Handle failures
        |
        v
11. Reconcile results
        |
        v
12. Mark completed
        |
        v
13. Document outcome
~~~

This should be treated as an operational change, not a casual script execution.

---

# 55. Common Mistakes

## Mistake 1 — Running the entire history at once

This can create:

- huge transactions
- memory pressure
- database load
- difficult recovery

Use controlled batches.

---

## Mistake 2 — No defined range

A query such as:

~~~text
process all old records
~~~

is dangerous.

Define explicit boundaries.

---

## Mistake 3 — No idempotency

If the worker restarts, duplicate processing can occur.

Backfills should be safe to resume.

---

## Mistake 4 — No checkpoint

Without progress state, operators may not know where the operation stopped.

---

## Mistake 5 — No reconciliation

A completed process is not necessarily a correct process.

Verify the results.

---

## Mistake 6 — Ignoring normal production traffic

A backfill can compete with live workloads.

Protect the normal pipeline.

---

## Mistake 7 — Triggering external side effects

Historical processing should not accidentally create new real-world actions.

---

## Mistake 8 — No stop mechanism

Every large backfill should have a safe way to pause or stop.

---

## Mistake 9 — Treating all failures equally

Separate:

- retryable failures
- quarantine failures
- unexpected failures

---

## Mistake 10 — Changing the logic during execution

Changing transformation code halfway through a backfill can create inconsistent results.

Control the processing version.

---

# 56. Troubleshooting

## Problem: backfill is too slow

Check:

1. batch size
2. query plan
3. indexes
4. database load
5. worker concurrency
6. network latency
7. external API calls
8. transaction size

Do not increase concurrency blindly.

---

## Problem: duplicate target records appear

Check:

1. target uniqueness constraints
2. idempotency logic
3. checkpoint handling
4. batch boundaries
5. source ordering
6. retry behavior

---

## Problem: backfill keeps restarting the same batch

Check:

1. checkpoint update
2. transaction boundary
3. worker crash behavior
4. target commit
5. checkpoint persistence

The target write and checkpoint behavior need to be understood together.

---

## Problem: source and target counts do not match

Check:

1. selection criteria
2. excluded records
3. validation failures
4. quarantine records
5. duplicate handling
6. late data
7. missing source records

Do not assume the target count must always equal the source count without defining what the backfill is expected to produce.

---

## Problem: normal pipeline becomes slow

Check:

1. database CPU
2. database I/O
3. connection usage
4. locks
5. replication lag
6. backfill concurrency
7. batch size

Reduce or pause the backfill if production health is being affected.

---

## Problem: transformation results change between batches

Check:

1. code version
2. configuration
3. reference data
4. schema changes
5. external dependencies
6. time-dependent logic

A backfill should use controlled processing conditions.

---

# 57. Production Considerations

Before running a production backfill, consider:

### Scope

Know exactly which records will be touched.

### Source

Know which system is authoritative.

### Target

Know whether records are inserted, updated, or replaced.

### Idempotency

Make reruns safe.

### Checkpointing

Persist progress.

### Batching

Keep work units manageable.

### Resource protection

Protect normal workloads.

### External side effects

Prevent unintended historical actions.

### Versioning

Know which processing logic is being used.

### Reconciliation

Verify the output.

### Observability

Monitor progress and system health.

### Stop controls

Be able to pause or cancel safely.

### Recovery

Know how to resume after failure.

### Documentation

Record what was processed and why.

---

# 58. Definition of Done

A backfill is complete when:

- [ ] The reason for the backfill is documented.
- [ ] The historical scope is explicitly defined.
- [ ] Source selection criteria are verified.
- [ ] Target behavior is defined.
- [ ] Expected record counts are known.
- [ ] Batch strategy is defined.
- [ ] Checkpoint strategy is defined.
- [ ] Idempotency behavior is defined.
- [ ] Retry behavior is defined.
- [ ] Quarantine behavior is defined.
- [ ] External side effects are controlled.
- [ ] Processing version is known.
- [ ] Schema compatibility is verified.
- [ ] Dry-run or sample validation is completed where appropriate.
- [ ] Normal production workload protection is defined.
- [ ] Progress is observable.
- [ ] Failure handling is tested.
- [ ] Resume behavior is tested.
- [ ] Cancellation behavior is defined.
- [ ] Reconciliation checks are defined.
- [ ] Results are verified.
- [ ] Missing or late data is accounted for.
- [ ] Quarantined records are accounted for.
- [ ] The final outcome is documented.

---

# 59. What You Learned

A backfill is controlled historical processing.

The basic model is:

~~~text
define scope
     |
     v
select historical data
     |
     v
validate scope
     |
     v
process in batches
     |
     v
checkpoint progress
     |
     v
handle failures
     |
     v
reconcile results
     |
     v
verify
~~~

The most important lessons are:

1. Define the historical range precisely.
2. Verify the records before modifying data.
3. Process large datasets in manageable batches.
4. Persist checkpoints.
5. Make processing idempotent.
6. Protect normal production workloads.
7. Avoid unintended external side effects.
8. Control the processing version.
9. Expect historical data to have schema and quality differences.
10. Treat missing and late data explicitly.
11. Reconcile source and target results.
12. Make failure recovery and resume behavior part of the design.
13. Give operators a safe way to stop the operation.
14. Monitor the backfill like any other production workload.

The key question is not:

> Can I run this script over the old data?

The better question is:

> Can I process this historical range safely, resume it after failure, prove what changed, and recover without damaging the normal pipeline?

That is the difference between a historical script and a production backfill.

---

# Chapter 28 Preview

The next recipe will cover **Incremental Processing**.

We will move from one-time historical processing to continuously processing only the data that has changed or arrived since the previous run.

We will cover:

- what incremental processing means
- full load vs incremental load
- high-water marks
- timestamps and cursors
- change detection
- inserts vs updates
- late-arriving data
- overlap windows
- idempotency
- checkpointing
- incremental failures
- recovery
- reconciliation
- incremental processing with PostgreSQL
- production monitoring

The key question will be:

> How do we process only the new or changed data without missing records or processing the same data unnecessarily?## Production Tools You Should Know

These tools solve or provide production implementations of concepts covered in this recipe. Learn what the tool provides, but understand the underlying problem first.

| Tool | What to know |
|---|---|
| **Apache Airflow** | Scheduling and controlling historical backfills. |
| **Apache Spark** | Large-scale historical recomputation. |
| **dbt** | Targeted model backfills and incremental rebuilds. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the engineering mechanism.

---


