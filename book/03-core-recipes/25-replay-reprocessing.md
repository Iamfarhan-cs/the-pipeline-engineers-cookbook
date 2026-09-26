# Chapter 25 — Replay / Reprocessing

Sometimes retry is not enough.

A record may have failed because of a temporary problem.

A transformation may have contained a bug.

A processing rule may have changed.

A schema may have changed.

Historical data may need to pass through a corrected pipeline.

In these situations, the pipeline may need to process data again.

This is called replay or reprocessing.

Replay sounds simple:

~~~text
take old data
   |
   v
process it again
~~~

But production replay is more complicated.

You need to know:

- Which records should be replayed?
- Why are they being replayed?
- Which version of the code should process them?
- Can replay create duplicates?
- Should the original result be replaced?
- Should a new result be created?
- How should replay history be recorded?
- What happens if replay fails?
- How can the replay be verified?

The goal is not simply to run old data again.

The goal is to reprocess it safely and intentionally.

---

## 1. Goal

The goal is to build a controlled replay and reprocessing process.

By the end of this recipe, you should understand how to:

- distinguish retry from replay
- identify when replay is needed
- select records for replay
- define replay boundaries
- preserve original history
- reset or update processing state safely
- prevent duplicate results
- use idempotency during replay
- replay failed records
- replay after code changes
- replay after schema changes
- perform partial replay
- handle failed replay
- track replay runs
- test replay
- verify replay results
- recover safely when replay goes wrong

The central idea is:

> Replay should be an explicit operation with a clear scope, reason, and verification process.

---

## 2. Problem

Suppose a pipeline processed 100,000 records.

A transformation bug was discovered after the processing completed.

The pipeline now contains:

~~~text
100,000 processed records
       |
       v
transformation bug discovered
~~~

The question becomes:

> How do we correct the affected data?

One option is to delete everything and run the entire pipeline again.

That may be dangerous and unnecessary.

Another option is to identify the affected records:

~~~text
100,000 records
       |
       v
identify affected population
       |
       v
5,000 records
       |
       v
reprocess only those records
~~~

This is controlled replay.

---

## 3. Why This Matters

Replay is one of the most important recovery capabilities in a Data Engineering system.

Without replay, a pipeline may depend on:

- manual database changes
- deleting data
- rerunning entire jobs
- restoring backups
- writing one-off scripts
- manually correcting records

These approaches can be risky.

A well-designed replay mechanism gives the team a controlled way to process historical data again.

Replay also supports:

- bug fixes
- corrected transformations
- schema migrations
- business-rule changes
- failed processing recovery
- historical reconstruction
- downstream recovery

---

## 4. Replay vs Retry

Retry and replay are related, but they are not the same thing.

### Retry

Retry means:

> Try the same operation again because the failure may be temporary.

Example:

~~~text
API request
   |
   X
timeout
   |
   v
retry
~~~

The operation usually remains within the same processing attempt or workflow.

### Replay

Replay means:

> Process previously received or stored data again.

Example:

~~~text
stored event
    |
    v
new processing run
    |
    v
process event again
~~~

Replay may happen:

- minutes later
- hours later
- days later
- months later

It may also use a newer version of the processing code.

---

## 5. Replay vs Backfill

Replay and backfill are also related.

### Replay

Usually focuses on reprocessing existing records or events through the pipeline again.

### Backfill

Usually focuses on intentionally processing historical data for a time period or missing population.

For example:

~~~text
Replay
failed records from yesterday
~~~

versus:

~~~text
Backfill
all records from January 1 to January 31
~~~

The boundary can overlap.

A backfill may internally use replay-like processing.

The important part is to define the operation clearly before starting it.

---

## 6. When Replay Is Needed

Replay can be useful when:

- processing code contained a bug
- a transformation was incorrect
- a business rule changed
- a downstream write failed
- a schema migration changed processing requirements
- a temporary dependency problem affected many records
- historical data needs reconstruction
- a failed processing population needs recovery
- a new derived field must be calculated from existing raw data

These are general use cases.

The correct replay strategy depends on the pipeline architecture.

---

## 7. Replay Architecture

A simple replay architecture looks like this:

~~~text
                 Stored source data
                        |
                        v
                 Select replay set
                        |
                        v
                  Create replay run
                        |
                        v
                  Process records
                        |
             +----------+----------+
             |                     |
          success                 failure
             |                     |
             v                     v
          verify              retry/quarantine
             |
             v
        replay complete
~~~

The important addition is the replay run.

Replay should not be an invisible modification to normal processing.

---

## 8. Before You Start

Before starting a replay, answer:

1. Why are we replaying?
2. Which records are affected?
3. What is the exact replay scope?
4. What source data will be used?
5. Which code version will process it?
6. Which schema version is expected?
7. What destination will receive the result?
8. Can the processing be repeated safely?
9. What happens to the existing result?
10. How will duplicates be prevented?
11. How will replay progress be tracked?
12. What happens if replay partially fails?
13. How will the result be verified?
14. How will the replay be stopped if something goes wrong?

Do not start a large replay without answering these questions.

---

## 9. Replay Scope

Replay scope defines exactly what will be processed again.

Possible scopes include:

### Record IDs

~~~text
record 1001
record 1002
record 1003
~~~

### Time range

~~~text
2026-01-01 to 2026-01-31
~~~

### Processing status

~~~text
status = failed
~~~

### Error type

~~~text
error_type = TransformationError
~~~

### Pipeline version

~~~text
processed_with_version = 1.4
~~~

### Source

~~~text
source = partner_a
~~~

The scope should be deterministic.

You should be able to explain exactly why a record was selected.

---

## 10. Why Replay Scope Matters

Imagine a bug affected only records processed between:

~~~text
10:00 and 10:30
~~~

If the replay selects every record from the entire day, the pipeline may perform unnecessary work.

Worse, it may modify records that were already correct.

A better selection is:

~~~text
affected records
      |
      v
exact replay scope
      |
      v
reprocess
~~~

Narrow replay scope reduces unnecessary risk.

But the scope must not be so narrow that affected records are missed.

---

## 11. Replay Selection Query

A generic selection might look like:

~~~sql
SELECT id
FROM processing_records
WHERE status = 'failed'
  AND created_at >= '2026-01-01'
  AND created_at < '2026-01-02';
~~~

This is a **generic example**.

Before using a query like this in production, verify:

- correct table
- correct status values
- correct timestamps
- timezone behavior
- whether the table contains the required history
- whether the selection is stable

Never assume a generic query matches an actual repository.

---

## 12. Freeze the Replay Set

For important replay operations, the selected population should be stable.

Suppose you run:

~~~text
SELECT failed records
~~~

Then new failures appear while replay is running.

If the selection is evaluated continuously, the replay may keep changing scope.

A safer design can create a replay set:

~~~text
source records
      |
      v
select population
      |
      v
freeze replay set
      |
      v
process selected records
~~~

The exact implementation can use:

- a replay-run table
- a temporary selection
- a materialized list
- a batch identifier

The choice depends on the system.

---

## 13. Replay Run

A replay run can provide a durable record of the operation.

A generic replay-run model might contain:

~~~text
replay_run_id
reason
scope
created_at
started_at
completed_at
status
code_version
schema_version
requested_by
~~~

This is a **generic example**.

The purpose is to answer:

> What replay happened, why did it happen, and what did it process?

---

## 14. Replay Status

A replay run can have states such as:

~~~text
created
   |
   v
running
   |
   +----> completed
   |
   +----> partially_failed
   |
   +----> failed
   |
   +----> cancelled
~~~

These are generic example states.

The exact state model depends on the implementation.

The important thing is that a replay has a lifecycle.

---

## 15. Replay History

Do not silently overwrite the history of the original processing.

Suppose:

~~~text
original run
    |
    v
record failed
~~~

Then:

~~~text
replay run
    |
    v
record completed
~~~

The system should ideally preserve both facts:

~~~text
original processing -> failed
replay processing   -> completed
~~~

This provides operational history.

It also helps answer questions later:

- Why was this record processed twice?
- Which replay corrected it?
- Which code version produced the final result?
- Did the original processing fail?

---

## 16. Replay and Processing Status

A replay must interact carefully with processing status.

Suppose a record is:

~~~text
failed
~~~

A replay may change its state to:

~~~text
replay_pending
~~~

Then:

~~~text
replay_pending
      |
      v
processing
      |
      v
completed
~~~

Another design may create a separate processing-run record instead of changing the original status.

Both approaches can work.

The choice depends on whether the system needs to preserve each processing attempt independently.

---

## 17. Do Not Simply Delete the Failure

A dangerous replay approach is:

~~~text
failed record
   |
   v
delete record
   |
   v
insert again
~~~

This can destroy useful history.

It may also break:

- audit trails
- foreign keys
- references
- debugging
- reconciliation
- operational history

Prefer a design where replay is explicit and historical information is preserved.

---

## 18. Replay and Idempotency

Replay makes idempotency even more important.

Suppose the original processing created:

~~~text
result for event 1001
~~~

Now replay processes event 1001 again.

Without protection:

~~~text
event 1001
   |
   +----> result A
   |
   +----> result B
~~~

Now the system has duplicates.

With idempotency:

~~~text
event 1001
   |
   +----> existing result recognized
   |
   v
safe reprocessing
~~~

The exact strategy depends on the destination.

Possible controls include:

- stable event IDs
- unique constraints
- idempotency keys
- deterministic record keys
- upserts
- replay-specific run identifiers

---

## 19. Replay Does Not Automatically Mean Ignore Existing Data

There are different replay goals.

### Goal A — Rebuild missing data

The destination should contain the missing result.

### Goal B — Correct existing data

The old result should be replaced or updated.

### Goal C — Create a new version

The old result should remain, and the replay should create a new version.

### Goal D — Recalculate derived data

The source remains unchanged, but downstream derived values are recalculated.

These goals require different write strategies.

Define the desired result before replay begins.

---

## 20. Replace vs Version

Suppose a transformation originally produced:

~~~text
amount = 100
~~~

A corrected transformation produces:

~~~text
amount = 110
~~~

Should the database contain:

~~~text
amount = 110
~~~

or:

~~~text
version 1 -> 100
version 2 -> 110
~~~

There is no universal answer.

The correct choice depends on:

- audit requirements
- business meaning
- data model
- downstream consumers
- correction policy

Do not overwrite historical data simply because replay is technically possible.

---

## 21. Replay After a Code Bug

A common replay scenario is a processing bug.

For example:

~~~text
records
   |
   v
version 1.2
   |
   v
incorrect transformation
~~~

The team fixes the code:

~~~text
version 1.3
~~~

Now the affected records can be replayed:

~~~text
raw data
   |
   v
version 1.3
   |
   v
correct result
~~~

Before doing this, confirm that the original source data is still available.

This is one reason raw data preservation is important.

---

## 22. Replay After Schema Changes

Schema changes can also require reprocessing.

For example:

~~~text
old schema
    |
    v
records stored
~~~

A new schema introduces:

~~~text
new field
~~~

The pipeline may need to process historical records to populate the new field.

The replay should account for:

- schema compatibility
- migration state
- old records
- new records
- versioned transformations
- downstream consumers

Do not assume that old records can automatically pass through the newest pipeline.

---

## 23. Replay After Business-Rule Changes

Suppose a business rule changes.

For example:

~~~text
old rule
customer classified as type A
~~~

New rule:

~~~text
customer classified as type B
~~~

Historical records may need reprocessing.

The replay scope should identify which records were affected by the old rule.

This can be more complicated than selecting records by failure status because the original processing may have succeeded.

This is an important distinction:

> Replay is not only for failed records.

---

## 24. Replay Successful Records

A record can be successfully processed and still need replay.

For example:

~~~text
processing status = completed
~~~

But later:

~~~text
business rule changed
~~~

The record may need to be processed again.

Therefore, replay selection should not be based only on:

~~~text
status = failed
~~~

Possible selection criteria include:

- processing version
- transformation version
- date range
- source
- business rule version
- affected population
- schema version

---

## 25. Partial Replay

Large datasets should often be replayed in smaller chunks.

For example:

~~~text
1,000,000 records
       |
       v
100,000 records
       |
       v
100,000 records
       |
       v
...
~~~

This provides control.

If something goes wrong, the entire replay does not need to be stopped after processing everything.

A chunk can be:

- selected
- processed
- verified
- marked complete

Then the next chunk can begin.

---

## 26. Replay Batches

A replay can use explicit batches:

~~~text
Replay Run 100
   |
   +--> Batch 1
   +--> Batch 2
   +--> Batch 3
   +--> Batch 4
~~~

Each batch can have its own state.

For example:

~~~text
batch 1 -> completed
batch 2 -> completed
batch 3 -> failed
batch 4 -> pending
~~~

This makes recovery easier.

Only the failed batch may need additional investigation.

---

## 27. Replay and Concurrency

Replay can compete with normal processing.

For example:

~~~text
normal workers
      |
      v
production database
      ^
      |
replay workers
~~~

Both can consume:

- CPU
- memory
- database connections
- API capacity
- queue capacity
- storage I/O

Replay should therefore be treated as production workload.

Possible controls include:

- separate workers
- limited concurrency
- batch sizes
- scheduling
- rate limits
- maintenance windows

The correct approach depends on the system.

---

## 28. Replay and Downstream Systems

Reprocessing data can affect downstream systems.

For example:

~~~text
source
  |
  v
pipeline
  |
  v
database
  |
  v
warehouse
  |
  v
analytics
~~~

If replay changes database values, downstream systems may also need to update.

Before replay, understand the complete dependency chain.

Ask:

> What downstream systems observe the result of this processing?

---

## 29. Replay and External Side Effects

Replay is especially dangerous when processing causes external side effects.

Examples include:

- sending an email
- creating a payment
- calling another service
- issuing a notification
- creating an external record

Replaying the same event could repeat the side effect.

For example:

~~~text
original processing
      |
      v
send email
      |
      v
replay
      |
      v
send email again
~~~

The replay design must decide whether the side effect should happen again.

Possible strategies include:

- idempotency keys
- side-effect history
- replay mode
- disabling external side effects during replay
- separate replay handlers

The correct strategy depends on the business operation.

---

## 30. Replay Mode

Some systems use an explicit replay mode.

For example:

~~~text
normal mode
    |
    v
process
    |
    v
external side effects enabled
~~~

Replay mode:

~~~text
replay mode
    |
    v
process
    |
    v
external side effects controlled
~~~

This can prevent unintended actions.

But replay mode must be designed carefully.

Simply adding a boolean flag is not enough if many downstream systems can create side effects.

---

## 31. Replay Failure

Replay can fail too.

For example:

~~~text
original record
     |
     v
replay
     |
     X
new failure
~~~

The new failure should be tracked separately from the original failure.

A useful history might look like:

~~~text
Original run
  attempt -> failed: old transformation bug

Replay run
  attempt -> failed: database timeout
~~~

This distinction matters during investigation.

---

## 32. Replay Retry

A failed replay batch may itself be retried.

For example:

~~~text
replay batch
    |
    X
timeout
    |
    v
retry
~~~

This is where retry and replay meet.

The system should distinguish:

~~~text
replay run
    |
    +--> processing attempt 1
    +--> processing attempt 2
    +--> processing attempt 3
~~~

Otherwise replay history can become confusing.

---

## 33. Stop Conditions

A large replay should have clear stop conditions.

For example, stop if:

- error rate exceeds a threshold
- duplicate rate increases
- database load becomes unsafe
- downstream errors increase
- unexpected data appears
- processing correctness cannot be verified

The exact thresholds are system-specific.

The important idea is:

> A replay should be stoppable.

Do not design a process that must run to completion once started.

---

## 34. Dry Run

A dry run can help verify replay selection before processing.

For example:

~~~text
select affected records
        |
        v
count records
        |
        v
inspect sample
        |
        v
verify scope
        |
        v
start replay
~~~

A dry run can answer:

- How many records will be replayed?
- Which dates are included?
- Which sources are included?
- Which processing versions are affected?
- Are unexpected records included?

This is especially useful for large replay operations.

---

## 35. Replay Verification

Do not finish a replay just because the worker completed.

Verify:

### Population

Did the expected number of records get selected?

### Processing

Did all selected records reach a final state?

### Errors

How many failed?

### Duplicates

Did duplicate results appear?

### Counts

Do source and destination counts reconcile?

### Data quality

Did the replayed data pass validation?

### Downstream effects

Did dependent systems receive the expected updates?

Verification should be part of the replay process.

---

## 36. Replay Reconciliation

Suppose the replay set contains:

~~~text
10,000 records
~~~

After replay:

~~~text
completed: 9,950
failed:       50
~~~

The replay is not fully complete.

A reconciliation report can make this clear:

~~~text
selected:    10,000
completed:    9,950
failed:          50
missing:          0
duplicates:       0
~~~

This gives a much clearer result than:

~~~text
replay finished
~~~

---

## 37. Replay Audit Information

A production replay should leave an audit trail.

Useful information may include:

~~~text
replay_run_id
reason
scope
started_at
completed_at
requested_by
code_version
schema_version
selected_count
success_count
failure_count
status
~~~

This is a **generic example**.

The exact audit model depends on the system.

The goal is reproducibility and investigation.

---

## 38. Code Version Tracking

When replaying after a code change, record which version performed the replay.

For example:

~~~text
original:
processing_version = 1.4

replay:
processing_version = 1.5
~~~

This makes later investigation easier.

Without version information, it can be difficult to understand why two processing runs produced different results.

---

## 39. Schema Version Tracking

The same principle applies to schema.

For example:

~~~text
schema version 10
~~~

versus:

~~~text
schema version 11
~~~

A replay may use a newer schema.

Record the relevant version where the system needs that information.

This is especially important when historical results must remain explainable.

---

## 40. Practical Replay Workflow

A controlled replay can follow this sequence:

~~~text
1. Identify the reason
        |
        v
2. Identify affected records
        |
        v
3. Verify the source data
        |
        v
4. Freeze the replay scope
        |
        v
5. Create replay run
        |
        v
6. Record code/schema versions
        |
        v
7. Run dry check
        |
        v
8. Start small batch
        |
        v
9. Process records
        |
        v
10. Verify results
        |
        v
11. Continue remaining batches
        |
        v
12. Investigate failures
        |
        v
13. Reconcile counts
        |
        v
14. Mark replay complete
~~~

For high-risk replays, stop after the first small batch and verify the output before continuing.

---

## 41. Testing Replay

Replay needs dedicated tests.

At minimum, test:

### Test 1 — Replay one failed record

Expected:

~~~text
record can be processed again
~~~

### Test 2 — Replay completed record

Expected:

~~~text
behavior follows the defined replay policy
~~~

### Test 3 — Replay creates no duplicate

Expected:

~~~text
one logical result
~~~

### Test 4 — Replay after code change

Expected:

~~~text
new processing behavior is applied
~~~

### Test 5 — Replay failure

Expected:

~~~text
new failure is recorded separately
~~~

### Test 6 — Partial replay

Expected:

~~~text
one failed batch does not corrupt completed batches
~~~

### Test 7 — Worker restart

Expected:

~~~text
replay can continue safely
~~~

### Test 8 — External side effect

Expected:

~~~text
side effect is not unintentionally duplicated
~~~

---

## 42. Common Mistakes

### Mistake 1 — Replaying everything

This creates unnecessary load and risk.

### Mistake 2 — Replaying without a clear scope

You may process the wrong records.

### Mistake 3 — Deleting history before replay

This removes valuable operational information.

### Mistake 4 — Ignoring idempotency

Replay can create duplicates.

### Mistake 5 — Replaying external side effects blindly

Emails, payments, notifications, and other actions may happen twice.

### Mistake 6 — No dry run

A large selection error can affect millions of records.

### Mistake 7 — No stop condition

A bad replay can continue damaging data.

### Mistake 8 — No version tracking

Later investigation becomes difficult.

### Mistake 9 — No reconciliation

You may not know whether the replay actually completed correctly.

### Mistake 10 — Treating replay as a normal retry

Replay often has a different scope, purpose, and lifecycle.

---

## 43. Troubleshooting

### Problem: replay selected too many records

Check:

1. selection query
2. time boundaries
3. timezone handling
4. status filters
5. processing-version filters
6. source filters
7. business-rule selection

Use a dry run before processing.

### Problem: replay created duplicates

Investigate:

1. idempotency keys
2. unique constraints
3. destination write behavior
4. replay identifiers
5. external side effects
6. whether the original operation already succeeded

Do not assume that a previous failure means no data was written.

### Problem: replay produces different results unexpectedly

Check:

1. code version
2. schema version
3. configuration
4. source data
5. external dependencies
6. current business rules
7. time-dependent logic

Historical processing can depend on more than the input record itself.

### Problem: replay keeps failing

Group the failures.

For example:

~~~text
validation failures
database failures
API failures
unknown failures
~~~

Then determine whether the problem is:

- data-related
- code-related
- dependency-related
- configuration-related

Do not repeatedly replay the same population without understanding the failure.

### Problem: replay affects normal processing

Check:

1. worker concurrency
2. database connection usage
3. API request volume
4. queue capacity
5. storage throughput
6. scheduler configuration

Replay is production workload and should be controlled accordingly.

---

## 44. Production Considerations

Replay can be one of the highest-risk operations in a pipeline.

Before production replay, consider:

### Scope

Know exactly which records are included.

### Source preservation

Confirm the original data is still available.

### Idempotency

Confirm repeated processing is safe.

### Side effects

Identify all external actions.

### Capacity

Estimate database, API, CPU, memory, and storage load.

### Concurrency

Do not let replay overwhelm normal workloads.

### Observability

Monitor progress, errors, latency, and duplicates.

### Recovery

Define what happens if replay fails halfway through.

### Auditability

Record why and how the replay was performed.

### Verification

Define success criteria before starting.

---

## 45. Definition of Done

Replay/reprocessing is complete when:

- [ ] The reason for replay is documented.
- [ ] The affected population is clearly defined.
- [ ] Replay scope can be reproduced.
- [ ] Source data is preserved.
- [ ] A replay run can be identified.
- [ ] Code version is recorded where required.
- [ ] Schema version is recorded where required.
- [ ] Existing processing history is preserved.
- [ ] Idempotency has been verified.
- [ ] Destination behavior is clearly defined.
- [ ] External side effects have been reviewed.
- [ ] A dry run or equivalent scope check has been performed where appropriate.
- [ ] Replay can be stopped safely.
- [ ] Replay progress is observable.
- [ ] Partial failures are represented.
- [ ] Retry behavior is defined for replay failures.
- [ ] Duplicate results are checked.
- [ ] Source and destination counts are reconciled.
- [ ] Data quality is verified after replay.
- [ ] Downstream effects are verified where required.
- [ ] Replay audit information is preserved.
- [ ] Recovery procedures are documented.

---

## 46. What You Learned

Replay is not simply:

~~~text
run the pipeline again
~~~

A production replay is a controlled operation.

The basic model is:

~~~text
Why replay?
     |
     v
Which records?
     |
     v
Is the source available?
     |
     v
Is replay safe?
     |
     v
Create replay run
     |
     v
Process in controlled batches
     |
     v
Verify
     |
     v
Reconcile
     |
     v
Complete or recover
~~~

The most important lessons are:

1. Define the replay reason.
2. Define the exact replay population.
3. Preserve the original processing history.
4. Make replay idempotent.
5. Treat successful records as possible replay candidates when business rules or code change.
6. Control external side effects.
7. Use small batches for high-risk or large replays.
8. Track code and schema versions when they matter.
9. Make replay observable and stoppable.
10. Verify the result instead of assuming completion means success.

The final goal is not to process old data again.

The goal is to make historical processing safe, controlled, explainable, and recoverable.

---

# Chapter 26 Preview

The next recipe will cover **Quarantine Failed Data**.

We will turn failed records into a controlled recovery path.

We will cover:

- what quarantine means
- when to quarantine
- quarantine vs retry
- quarantine vs rejection
- preserving failed records
- quarantine metadata
- quarantine storage
- failure reasons
- reviewing quarantined records
- correcting bad data
- replaying quarantined records
- quarantine retention
- quarantine monitoring
- security considerations
- testing quarantine
- production recovery

The key question will be:

> When a record cannot safely continue through the normal pipeline, where should it go and how can we recover it later?## Production Tools You Should Know

These tools solve or provide production implementations of concepts covered in this recipe. Learn what the tool provides, but understand the underlying problem first.

| Tool | What to know |
|---|---|
| **Apache Kafka** | Replayable event retention and offset-based reprocessing. |
| **Apache Spark** | Batch reprocessing over historical data. |
| **Apache Flink** | Stateful stream replay and recovery patterns. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the engineering mechanism.

---


