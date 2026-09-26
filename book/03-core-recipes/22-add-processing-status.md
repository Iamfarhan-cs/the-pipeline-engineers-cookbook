# Chapter 22 — Add Processing Status

A pipeline needs to know more than whether a record exists.

It also needs to know what happened to that record.

Consider a record that has entered a pipeline.

Was it:

- received but not processed?
- currently being processed?
- completed successfully?
- failed?
- waiting for retry?
- stuck because a worker crashed?

Without processing status, these questions become difficult to answer.

Processing status gives the pipeline a simple state model.

```text
pending
   |
   v
processing
   |
   +------> failed
   |
   v
completed
```

This chapter turns that idea into a practical pipeline recipe.

---

## 1. Goal

The goal is to add a clear processing-status model so that the pipeline can track the lifecycle of each record.

By the end of this recipe, you should understand how to:

- define processing states
- define valid state transitions
- store processing status
- claim work safely
- handle worker crashes
- handle failed processing
- support retries
- detect stale processing records
- coordinate multiple workers
- test state transitions
- recover stuck records
- expose useful status metrics
- distinguish processing state from business state

The central idea is:

> Every record should have a clear answer to the question: what happened to this work?

---

## 2. Problem

Imagine a staging table containing 10,000 records.

A worker starts processing them.

After several minutes, someone asks:

> How many records are still waiting?

Without processing status, the answer may require inspecting logs or reconstructing what happened from several systems.

Now imagine the worker crashes.

Some records may have been:

- processed successfully
- partially processed
- never started
- currently being processed when the worker disappeared

If the database does not store processing state, recovery becomes difficult.

A status field can make the state visible.

For example:

```text
record 1001 -> completed
record 1002 -> completed
record 1003 -> failed
record 1004 -> processing
record 1005 -> pending
```

Now the pipeline has a basic operational picture.

---

## 3. Why This Matters

Processing status supports several important pipeline operations.

### Recovery

The system can find records that failed.

### Retry

The system can identify records that are eligible for another attempt.

### Monitoring

Operators can see how many records are pending, processing, completed, or failed.

### Debugging

An engineer can inspect the current state of one record.

### Concurrency

Workers can coordinate around records that are being processed.

### Reprocessing

A completed or failed record can be intentionally moved into another processing cycle when the system supports that behavior.

Processing status therefore connects the data pipeline with its operational state.

---

## 4. When to Use This Recipe

Processing status is useful when records are processed asynchronously or in multiple stages.

Common situations include:

- ingestion workers
- queue consumers
- batch processors
- API ingestion
- staging pipelines
- ETL jobs
- file processing
- background workers
- retryable processing
- replay systems
- long-running transformations

It is especially useful when processing can take time or fail independently for individual records.

---

## 5. Architecture

A simple status-aware pipeline can look like this:

```text
                +-------------+
                |   pending   |
                +-------------+
                       |
                       v
                +-------------+
                | processing  |
                +-------------+
                  |         |
             success       failure
                  |         |
                  v         v
            +-----------+ +---------+
            | completed | | failed  |
            +-----------+ +---------+
                              |
                              v
                           retry
                              |
                              v
                         processing
```

The important part is that the pipeline does not randomly change status.

There should be defined transitions.

---

## 6. Before You Start

Before adding processing status, answer these questions:

1. What does one processing unit represent?
2. Which table or object stores that unit?
3. Which states are required?
4. What does each state mean?
5. Which transitions are valid?
6. Who is allowed to change the state?
7. What happens when processing fails?
8. What happens when a worker crashes?
9. How is a stale processing record detected?
10. Can two workers claim the same record?
11. Should status history be retained?
12. How are retries represented?

Do not start by adding a status column without defining the state machine.

---

## 7. Define the States

A simple pipeline may need these states:

```text
pending
processing
completed
failed
```

Each state should have one clear meaning.

### pending

The record is available for processing but has not been claimed by a worker.

### processing

A worker has claimed the record and is currently working on it.

### completed

Processing finished successfully.

### failed

Processing failed and the record requires retry, investigation, or another explicit action.

These are generic states.

A real pipeline may need additional states.

For example:

```text
pending
processing
completed
failed
retry_pending
quarantined
cancelled
```

Do not add states just because they sound useful.

Every state should represent a meaningful operational condition.

---

## 8. Status Is a State Machine

A status column becomes much more useful when treated as a state machine.

For example:

```text
pending
   |
   v
processing
   |
   +---------> failed
   |             |
   |             v
   |          retry
   |             |
   |             v
   +--------- processing
   |
   v
completed
```

This means:

- pending can become processing
- processing can become completed
- processing can become failed
- failed can become processing again when retry is allowed

But this may not be valid:

```text
completed
    |
    v
pending
```

unless the system explicitly supports resetting completed work.

This is why transitions should be defined before implementation.

---

## 9. State Transition Table

A simple transition table can make the rules explicit.

| Current State | Next State | Meaning |
|---|---|---|
| pending | processing | Worker claimed the record |
| processing | completed | Processing succeeded |
| processing | failed | Processing failed |
| failed | processing | Retry started |
| pending | failed | Usually avoid unless failure occurs before processing begins |
| completed | pending | Only if explicit reprocessing supports it |

The exact transitions depend on the pipeline.

The table should be treated as part of the design.

---

## 10. Database Schema

A generic processing table might contain:

```sql
CREATE TABLE pipeline_records (
    id BIGSERIAL PRIMARY KEY,
    status TEXT NOT NULL,
    attempt_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

This is a **generic example**.

The actual table and columns depend on the repository.

The important fields are:

- status
- attempt count
- timestamps

Additional fields may be useful:

```text
last_error
processed_at
started_at
worker_id
retry_at
```

Do not add every possible field automatically.

Add fields that support actual operational requirements.

---

## 11. Status Constraints

If the set of states is small and controlled, the database can enforce valid values.

For example:

```sql
CHECK (
    status IN (
        'pending',
        'processing',
        'completed',
        'failed'
    )
)
```

This prevents accidental values such as:

```text
done
complete
finished
processing_now
```

when the application expects a different vocabulary.

Consistent status values make querying and monitoring much easier.

---

## 12. Status vs Processing Result

Status tells you the current state.

It does not necessarily explain the full history.

For example:

```text
status = failed
```

does not tell you:

- how many attempts happened
- when the first attempt happened
- when the last attempt happened
- what error occurred
- which worker processed it

This is why status often works together with other metadata.

For example:

```text
status
attempt_count
started_at
updated_at
last_error
```

The exact fields depend on the pipeline.

---

## 13. Attempt Count

Retryable pipelines often need an attempt count.

For example:

```text
attempt_count = 0
```

Record has not been processed yet.

After the first attempt:

```text
attempt_count = 1
```

After a second attempt:

```text
attempt_count = 2
```

This can support:

- retry limits
- monitoring
- debugging
- retry policies

For example:

```text
if attempt_count >= max_attempts:
    stop retrying
```

The exact retry policy should be defined separately from the status model.

---

## 14. Processing Status vs Business Status

Do not confuse pipeline processing status with business state.

For example:

```text
Processing status:
completed

Business status:
pending_payment
```

The record can be successfully processed while its business lifecycle is still pending.

Another example:

```text
Processing status:
failed

Business status:
pending_review
```

These are different dimensions.

A useful rule is:

> Processing status answers whether the pipeline successfully handled the record. Business status answers what the record means in the business domain.

Do not combine unrelated concepts into one status field.

---

## 15. Claiming Work

A common pipeline flow is:

```text
Find pending record
        |
        v
Claim record
        |
        v
Set status = processing
        |
        v
Do work
        |
        v
Set completed or failed
```

The important part is the claim.

If multiple workers can run at the same time, they must not all claim the same record.

---

## 16. The Check-Then-Update Problem

A naive worker might do:

```text
1. SELECT pending record
2. Receive record
3. UPDATE status = processing
4. Process record
```

Two workers can race.

For example:

```text
Worker A                  Worker B

SELECT pending            SELECT pending
       |                         |
       v                         v
same record               same record
       |                         |
       v                         v
UPDATE processing         UPDATE processing
       |                         |
       +----------+--------------+
                  |
              both process
```

The status field alone did not prevent duplicate work.

Work claiming must be atomic.

---

## 17. Atomic Work Claiming

A database can often be used to claim work safely.

One PostgreSQL pattern is to use row locking.

For example:

```sql
SELECT id
FROM pipeline_records
WHERE status = 'pending'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

This is a **generic example**.

The important idea is:

- lock a candidate row
- skip rows already locked by another worker
- claim work inside a transaction

A complete implementation must also update the status within the correct transaction boundary.

---

## 18. Why SKIP LOCKED Is Useful

Suppose three workers are running:

```text
Worker A
Worker B
Worker C
```

There are many pending records.

Without coordination, workers may select the same record.

With row-level locking and SKIP LOCKED:

```text
Worker A -> record 1
Worker B -> record 2
Worker C -> record 3
```

A worker skips rows already locked by another worker.

This can make a database-backed work queue practical for certain workloads.

It is not automatically the best design for every pipeline.

---

## 19. Claim and Process Are Different

A worker can successfully claim a record and then fail during processing.

For example:

```text
pending
   |
   v
processing
   |
   X
worker crashes
```

The record remains:

```text
processing
```

But no worker is actually processing it anymore.

This is a **stale processing record**.

The pipeline needs a recovery strategy.

---

## 20. Stale Processing Records

A stale record is a record that says:

```text
status = processing
```

but has not shown evidence of active processing for an expected amount of time.

For example:

```text
started_at = 10:00
current time = 12:00
```

If normal processing takes seconds, two hours may indicate a problem.

But the timeout must be based on actual workload behavior.

Do not choose an arbitrary value without understanding the processing duration.

---

## 21. Heartbeats

For long-running work, a worker can update a heartbeat timestamp.

For example:

```text
status = processing
heartbeat_at = current time
```

The worker updates it periodically:

```text
10:00 heartbeat
10:01 heartbeat
10:02 heartbeat
10:03 heartbeat
```

If heartbeats stop:

```text
last heartbeat = 10:03
current time = 10:20
```

the record may be stale.

A heartbeat is especially useful when processing can legitimately take a long time.

---

## 22. Recovering Stale Records

A recovery process can find stale records:

```text
processing
    |
    v
heartbeat expired
    |
    v
mark retryable
    |
    v
pending or retry_pending
    |
    v
worker claims again
```

The exact transition depends on the retry design.

Do not simply change every old processing record back to pending.

Some jobs may legitimately run for a long time.

Recovery needs enough information to distinguish:

```text
slow but alive
```

from:

```text
worker crashed
```

---

## 23. Failure Handling

Suppose processing fails.

A simple transition is:

```text
processing
     |
     v
failed
```

But failed does not always mean permanent failure.

The system may decide:

```text
failed
   |
   v
retry allowed?
   /       \
 yes        no
  |          |
  v          v
retry      quarantine
```

This is why status design should work with the retry policy from Chapter 6.

Processing status tells you the current state.

Retry policy decides what happens next.

---

## 24. Retry-Pending State

Some systems benefit from an explicit retry state.

For example:

```text
processing
    |
    v
failed
    |
    v
retry_pending
    |
    v
processing
```

This can be useful when retries are scheduled for the future.

For example:

```text
failed at 10:00
retry_at = 10:05
```

The worker should not immediately claim it again.

A retry scheduler can wait until the retry time.

This is one reason status models should reflect actual operational behavior.

---

## 25. Status History

A single status column only tells you the current state.

Sometimes you need the history.

For example:

```text
10:00 pending
10:01 processing
10:01 failed
10:02 processing
10:02 failed
10:05 processing
10:05 completed
```

This history can answer:

- how many attempts happened?
- how long did processing take?
- why did it fail?
- how long did it remain pending?
- how often does this record fail?

There are several ways to represent history.

One option is a separate table.

For example:

```sql
CREATE TABLE pipeline_status_history (
    id BIGSERIAL PRIMARY KEY,
    record_id BIGINT NOT NULL,
    old_status TEXT,
    new_status TEXT NOT NULL,
    changed_at TIMESTAMP NOT NULL,
    reason TEXT
);
```

This is a **generic example**.

A history table is useful when auditability or detailed troubleshooting matters.

It also increases storage and implementation complexity.

Use it when the operational need justifies it.

---

## 26. Current State vs History

A useful design can have both:

```text
pipeline_records
    |
    +--> current status
    |
    +--> attempt count
    |
    +--> timestamps

pipeline_status_history
    |
    +--> status transitions
    +--> reasons
    +--> timestamps
```

The current table supports fast operational queries.

The history table supports investigation.

For example:

```text
Current:
status = failed

History:
pending -> processing
processing -> failed
failed -> processing
processing -> failed
```

This is much easier to troubleshoot.

---

## 27. Status Transition Rules

A robust implementation should prevent invalid transitions.

For example:

Valid:

```text
pending -> processing
processing -> completed
processing -> failed
failed -> processing
```

Potentially invalid:

```text
completed -> processing
```

unless an explicit replay operation allows it.

Another potentially invalid transition:

```text
pending -> completed
```

if all work must pass through processing.

The application should make these rules explicit.

---

## 28. Implement a Transition Function

Instead of changing status everywhere in the code, centralize the transition logic.

A generic example:

```python
VALID_TRANSITIONS = {
    "pending": {"processing"},
    "processing": {"completed", "failed"},
    "failed": {"processing"},
    "completed": set(),
}


def can_transition(current, new):
    return new in VALID_TRANSITIONS.get(current, set())
```

This is a **generic example**.

The actual implementation may use a database operation, service method, domain object, or another design.

The important idea is to avoid scattered, inconsistent state changes.

---

## 29. Atomic Status Updates

Status changes should be tied to the work they represent.

For example:

```text
Process succeeds
      |
      v
Update result
      |
      v
Set completed
```

If the result is committed but the status update fails, the record may appear incomplete even though the work succeeded.

If the status is committed but the result is not, the record may appear successful when it is not.

For database-only processing, related changes should often be placed in the same transaction.

For example:

```text
BEGIN
    write result
    update status = completed
COMMIT
```

If the transaction fails:

```text
ROLLBACK
```

The exact transaction boundary depends on the pipeline.

---

## 30. External Side Effects Again

Transactions cannot automatically cover external systems.

Suppose processing does:

```text
1. Update database
2. Call external API
3. Mark record completed
```

If the API succeeds but step 3 fails, the record may be retried.

The external API may receive the operation again.

This is the same reliability problem discussed in the idempotency chapter.

Processing status does not replace idempotency.

A strong design may require:

```text
Processing status
       +
Idempotency
       +
Transaction
       +
External operation policy
```

---

## 31. Status and Idempotency

These concepts should work together.

Suppose:

```text
record 1001
status = failed
attempt_count = 2
```

A retry worker moves it to:

```text
status = processing
attempt_count = 3
```

If the worker crashes after the business operation succeeds, the record may become stuck in processing.

When recovery retries it, idempotency should prevent the second execution from creating an unintended duplicate result.

This is why:

> Processing status tells the system what state the work is in. Idempotency makes repeating the work safe.

Both are needed for reliable recovery.

---

## 32. Generic Processing Example

A simplified worker might look like this:

```python
def process_record(record):
    claim_record(record)

    try:
        process_business_logic(record)
        mark_completed(record)
    except Exception as exc:
        mark_failed(record, str(exc))
        raise
```

This is a **generic example**.

A production implementation must consider:

- atomic claiming
- transaction boundaries
- retries
- stale records
- idempotency
- error classification
- external side effects
- concurrent workers

The simple example is useful for understanding the lifecycle.

It is not a complete production worker.

---

## 33. Testing State Transitions

Status logic should be tested directly.

At minimum, test:

### Test 1 — Pending to processing

Input:

```text
pending
```

Action:

```text
worker claims record
```

Expected:

```text
processing
```

### Test 2 — Processing to completed

Expected:

```text
completed
```

### Test 3 — Processing to failed

Expected:

```text
failed
```

### Test 4 — Failed to processing

Expected:

```text
processing
```

when retry is allowed.

### Test 5 — Invalid transition

For example:

```text
completed -> processing
```

Expected:

```text
transition rejected
```

unless replay explicitly allows it.

---

## 34. Test Worker Crashes

Simulate:

```text
pending
   |
   v
processing
   |
   X
worker crashes
```

Then verify:

- the record can be detected as stale
- recovery can identify it
- retry behavior is correct
- idempotency prevents duplicate effects
- the final state becomes correct

This test is much more useful than testing only the happy path.

---

## 35. Test Concurrent Workers

Start multiple workers against the same pending records.

For example:

```text
Worker A
Worker B
Worker C
```

Verify:

- one worker claims each record
- the same record is not processed concurrently unless explicitly allowed
- status changes remain consistent
- completed records are not claimed again
- failed records follow retry rules

Concurrency bugs often appear only under real parallel execution.

---

## 36. Test Stale Records

Create a record such as:

```text
status = processing
started_at = old timestamp
heartbeat_at = old timestamp
```

Run the recovery process.

Verify that the record follows the expected recovery transition.

For example:

```text
processing
    |
    v
stale
    |
    v
pending
```

The exact transition may differ.

The test should follow the repository's defined recovery policy.

---

## 37. Verify Status Counts

Operational verification should include status counts.

For example:

```sql
SELECT
    status,
    COUNT(*) AS record_count
FROM pipeline_records
GROUP BY status
ORDER BY status;
```

A result might look like:

```text
completed   9,500
pending       300
processing    100
failed        100
```

These numbers give an operational snapshot.

But the numbers need context.

A growing failed count may indicate a problem.

A growing pending count may indicate insufficient workers.

A large processing count may indicate slow work or stuck workers.

A growing completed count is usually expected during healthy processing.

---

## 38. Useful Processing Metrics

Status data can support metrics such as:

```text
records_pending
records_processing
records_completed
records_failed
records_retried
processing_duration
queue_age
stale_processing_records
```

These metrics can help operators understand pipeline health.

For example:

```text
pending count increasing
        |
        v
processing capacity may be insufficient
```

Or:

```text
processing count stays high
        |
        v
workers may be slow or stuck
```

Status is therefore not only a database feature.

It is an observability signal.

---

## 39. Common Mistakes

### Mistake 1 — Using one vague status

Bad:

```text
status = done
```

What does "done" mean?

Does it mean:

- processed?
- validated?
- stored?
- published?
- acknowledged?

Use precise states.

---

### Mistake 2 — Mixing processing and business state

Bad:

```text
status = pending
```

when the field actually means either "pipeline pending" or "business pending."

Keep the concepts separate.

---

### Mistake 3 — Allowing arbitrary transitions

If every part of the code can change status freely, invalid states eventually appear.

Centralize or clearly enforce transition rules.

---

### Mistake 4 — No stale-record recovery

A worker can crash.

A record can remain in processing forever.

The pipeline needs a recovery strategy.

---

### Mistake 5 — Using status instead of idempotency

Status alone does not guarantee that repeated work is safe.

Use both when the processing model requires both.

---

### Mistake 6 — Treating every failed record as permanently failed

Some failures are temporary.

Retryable failures should have a retry path.

---

### Mistake 7 — No attempt count

Without attempt information, it becomes harder to understand repeated failures and enforce retry limits.

---

### Mistake 8 — Updating status outside the correct transaction

A record can become completed while its actual result is not safely committed.

Review transaction boundaries carefully.

---

## 40. Practical Investigation Example

Suppose a pipeline dashboard shows:

```text
pending:     20,000
processing:      50
completed:   80,000
failed:       5,000
```

The first question is not:

> Why are there 5,000 failed records?

Investigate the state transitions.

Ask:

1. When did the failed records enter the failed state?
2. How many attempts have they had?
3. Are failures temporary or permanent?
4. Are retries happening?
5. Are records becoming stale in processing?
6. Are workers successfully claiming work?
7. Is the pending queue growing?
8. Are processing times increasing?
9. Are all workers using the same status rules?
10. Are status updates committed with the processing result?

The state counts are symptoms.

The transition history can reveal the cause.

---

## 41. Practical Implementation Sequence

For an existing repository, use this order:

```text
1. Understand the processing unit
        |
        v
2. Define the state model
        |
        v
3. Define valid transitions
        |
        v
4. Inspect the existing database schema
        |
        v
5. Decide whether current status fields can be reused
        |
        v
6. Add required status fields or migration
        |
        v
7. Implement atomic work claiming
        |
        v
8. Implement success transition
        |
        v
9. Implement failure transition
        |
        v
10. Add retry behavior
        |
        v
11. Add stale-record recovery
        |
        v
12. Review transaction boundaries
        |
        v
13. Review idempotency
        |
        v
14. Add transition tests
        |
        v
15. Add concurrency tests
        |
        v
16. Test worker crash recovery
        |
        v
17. Verify status counts
        |
        v
18. Add operational metrics
        |
        v
19. Document the state machine
```

The state model should be designed before the database column is added.

---

## 42. Definition of Done

The processing-status implementation is complete when:

- [ ] Processing states are clearly defined.
- [ ] Every state has a precise meaning.
- [ ] Valid state transitions are documented.
- [ ] Invalid transitions are prevented where appropriate.
- [ ] Processing status is stored durably.
- [ ] Work claiming is safe for concurrent workers.
- [ ] Successful processing moves records to completed.
- [ ] Failed processing moves records to the appropriate failure state.
- [ ] Retry behavior is defined.
- [ ] Attempt counts are tracked when required.
- [ ] Stale processing records can be detected.
- [ ] Stale records have a recovery path.
- [ ] Transaction boundaries have been reviewed.
- [ ] Idempotency has been considered.
- [ ] State transitions are unit tested.
- [ ] Integration tests cover database behavior.
- [ ] Concurrent workers have been tested where relevant.
- [ ] Worker crash recovery has been tested.
- [ ] Status counts can be queried.
- [ ] Useful processing metrics are available.
- [ ] The state machine is documented.

---

## 43. What You Learned

Processing status gives a pipeline a durable view of what happened to its work.

The basic lifecycle is:

```text
pending
   |
   v
processing
   |
   +------> failed
   |          |
   |          v
   |        retry
   |          |
   |          v
   +------ processing
   |
   v
completed
```

The important lesson is:

> A status field is not just a label. It is part of the pipeline's state machine.

A reliable implementation needs:

```text
Clear states
    +
Valid transitions
    +
Safe work claiming
    +
Transaction boundaries
    +
Retry behavior
    +
Stale-record recovery
    +
Idempotency
    +
Tests
    +
Observability
```

Once processing status is available, the pipeline can answer operational questions much more clearly:

- What is waiting?
- What is running?
- What succeeded?
- What failed?
- What needs retry?
- What may be stuck?
- How much work remains?

That makes the next reliability features much easier to build.

---

# Chapter 23 Preview

The next recipe will add **Error Handling**.

We will look at how to make failures useful instead of simply crashing a worker.

We will cover:

- expected vs unexpected errors
- retryable vs non-retryable errors
- record-level errors
- batch-level errors
- structured error information
- database errors
- API errors
- validation errors
- transaction failures
- error persistence
- failed-record handling
- quarantine
- error logging
- error metrics
- error propagation
- testing failure paths
- production troubleshooting

The key question will be:

> When something fails, how does the pipeline decide what to do next?## Production Tools You Should Know

These tools solve or provide production implementations of concepts covered in this recipe. Learn what the tool provides, but understand the underlying problem first.

| Tool | What to know |
|---|---|
| **Apache Airflow** | Task and run state management. |
| **Dagster** | Asset and run status with observable orchestration. |
| **Prefect** | Flow/task state and failure tracking. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the engineering mechanism.

---


