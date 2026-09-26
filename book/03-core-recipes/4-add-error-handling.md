# Recipe 4 — Add Error Handling

A pipeline will fail.

An API can return an error.

A database can become unavailable.

A record can contain invalid data.

A network connection can time out.

A worker can crash.

An external service can return an unexpected response.

The important question is not whether failures happen.

The important question is:

> What does the pipeline do when something fails?

Good error handling makes failures understandable and recoverable.

Poor error handling can turn one bad record into a failed batch, hide the real cause, create duplicate work, or leave records in an unclear state.

This chapter explains how to design error handling as part of the pipeline instead of treating it as an afterthought.

---

## 1. Goal

The goal is to make pipeline failures explicit, useful, and recoverable.

By the end of this recipe, you should understand how to:

- classify errors
- distinguish expected and unexpected errors
- distinguish retryable and non-retryable errors
- handle record-level failures
- handle batch-level failures
- store useful error information
- log errors safely
- connect errors with processing status
- connect errors with retries
- quarantine records when required
- preserve transaction correctness
- test failure paths
- monitor error rates
- troubleshoot failed records
- avoid hiding important failures

The central idea is:

> An error should tell the pipeline what happened and help it decide what to do next.

---

## 2. Problem

Consider a batch containing 1,000 records.

999 records are valid.

One record contains an invalid value.

A simple implementation might do this:

~~~text
read batch
   |
   v
process record 1
   |
   v
process record 2
   |
   v
...
   |
   v
record 500 fails
   |
   X
worker crashes
~~~

The result may be:

- 499 records processed
- 1 record failed
- 500 records never reached
- no clear record-level error
- unclear recovery state

That is not necessarily the behavior the pipeline needs.

A better design asks:

~~~text
Did one record fail?

Can the other records continue?

Should the failed record be retried?

Should it be quarantined?

Should the entire batch stop?
~~~

The answer depends on the type of error and the processing model.

---

## 3. Why This Matters

Error handling affects almost every reliability feature in a pipeline.

It interacts with:

- processing status
- retries
- idempotency
- transactions
- validation
- quarantine
- observability
- replay
- recovery

For example:

~~~text
Record fails
    |
    v
Classify error
    |
    +------> retryable
    |           |
    |           v
    |         retry
    |
    +------> non-retryable
                |
                v
            quarantine
~~~

Without classification, the pipeline may retry errors that will never succeed.

Or it may permanently reject temporary failures.

---

## 4. When to Use This Recipe

Error handling should be considered whenever a pipeline can fail at any stage.

Common examples include:

- API requests
- database operations
- file processing
- validation
- transformations
- message consumption
- serialization
- deserialization
- schema validation
- external service calls
- warehouse writes
- batch jobs
- streaming consumers

It becomes especially important when processing must continue after individual records fail.

---

## 5. Architecture

A simple error-handling flow looks like this:

~~~text
                 Process record
                       |
                       v
                    Success?
                   /       \
                 Yes        No
                 |           |
                 v           v
             completed    classify
                             |
                    +--------+--------+
                    |                 |
                retryable        non-retryable
                    |                 |
                    v                 v
                  retry           quarantine
                    |
                    v
                processing
~~~

Unexpected system-level failures may follow another path:

~~~text
Database unavailable
       |
       v
stop or retry operation
       |
       v
protect transaction
       |
       v
recover
~~~

The key idea is that different failures can require different actions.

---

## 6. Before You Start

Before adding error handling, answer these questions:

1. What can fail?
2. Which failures are expected?
3. Which failures are temporary?
4. Which failures are permanent?
5. Which failures should be retried?
6. Which failures should be quarantined?
7. Which failures should stop the batch?
8. Where should errors be stored?
9. What should be logged?
10. Which information must not be logged?
11. How does an error affect processing status?
12. How is the failed record recovered?
13. How is the error tested?

Do not begin by wrapping the entire worker in one broad exception handler.

First understand the failure categories.

---

## 7. Expected vs Unexpected Errors

An expected error is a failure the pipeline knows can occur.

Examples:

- invalid input
- missing required field
- unsupported value
- known API response
- duplicate record
- business-rule violation

The application can usually handle these explicitly.

An unexpected error is a failure the application did not anticipate.

Examples:

- programming bug
- unexpected response structure
- corrupted application state
- unknown database failure
- unexpected exception

Unexpected errors should still be captured and observed.

But they should not be silently converted into normal business failures.

---

## 8. Retryable vs Non-Retryable Errors

This is one of the most important classifications.

### Retryable error

The same operation may succeed later.

Examples:

- temporary network timeout
- service unavailable
- temporary database connection failure
- rate limit
- temporary infrastructure problem

### Non-retryable error

Repeating the same operation is unlikely to fix the problem.

Examples:

- required field missing
- invalid identifier
- malformed data
- unsupported value
- known business-rule violation

Conceptually:

~~~text
Error
 |
 +--> retryable ------> retry
 |
 +--> non-retryable --> quarantine / reject
~~~

Do not retry everything.

---

## 9. Why Retrying Everything Is Dangerous

Suppose an API request fails because the payload contains an invalid identifier.

The pipeline retries:

~~~text
attempt 1 -> invalid identifier
attempt 2 -> invalid identifier
attempt 3 -> invalid identifier
attempt 4 -> invalid identifier
~~~

Nothing changed.

The retries only create:

- unnecessary load
- extra latency
- noisy logs
- wasted worker capacity

They may also hide the real data-quality problem.

Retry should be based on the error category.

---

## 10. Error Categories

A practical pipeline can classify errors into categories such as:

~~~text
ValidationError
DatabaseError
NetworkError
TimeoutError
RateLimitError
AuthenticationError
BusinessRuleError
SerializationError
UnknownError
~~~

These names are **generic examples**.

The actual error model should match the application.

The purpose is to make error handling explicit.

---

## 11. Validation Errors

Validation errors usually indicate that the input does not meet the required contract.

For example:

~~~text
missing customer_id
invalid amount
unsupported currency
invalid timestamp
~~~

A typical flow is:

~~~text
record
  |
  v
validate
  |
  X
invalid
  |
  v
record-level error
  |
  v
quarantine / reject
~~~

Retrying the same invalid record without changing the input usually does not help.

This connects directly to Recipe 3.

---

## 12. Database Errors

Database failures can have different meanings.

For example:

### Temporary connection failure

Potentially retryable.

### Deadlock

Often retryable.

### Unique constraint violation

May indicate:

- duplicate input
- incorrect business logic
- unexpected data

It should not automatically be treated like a network timeout.

### Syntax error

Usually a programming problem rather than a retryable operational error.

Therefore:

> "Database error" is not one single retry category.

The specific database error matters.

---

## 13. API Errors

API errors also need classification.

For example:

~~~text
HTTP 400
    |
    v
bad request
    |
    v
usually not retryable
~~~

Whereas:

~~~text
HTTP 503
    |
    v
service unavailable
    |
    v
often retryable
~~~

And:

~~~text
HTTP 429
    |
    v
rate limited
    |
    v
retry according to rate-limit policy
~~~

These are general patterns.

The actual retry behavior should follow the API contract.

---

## 14. Record-Level Errors

A record-level error affects one processing unit.

For example:

~~~text
Record 1 -> success
Record 2 -> success
Record 3 -> invalid
Record 4 -> success
Record 5 -> success
~~~

If the processing model supports isolation, the pipeline may continue:

~~~text
Record 1 -> completed
Record 2 -> completed
Record 3 -> failed/quarantined
Record 4 -> completed
Record 5 -> completed
~~~

This is often preferable for independent records.

But it is not always safe.

If records depend on each other, one failure may require the larger operation to stop.

---

## 15. Batch-Level Errors

A batch-level error affects the processing unit as a whole.

For example:

~~~text
Database unavailable
~~~

If every record requires that database, continuing may cause every record to fail.

Another example:

~~~text
Input file cannot be parsed
~~~

There may be no safe record-level processing at all.

In such cases:

~~~text
batch
  |
  v
system failure
  |
  v
stop
  |
  v
retry / investigate
~~~

The important question is:

> Can the failure be isolated to one record?

If yes, record-level handling may be appropriate.

If no, batch-level handling may be required.

---

## 16. Error Isolation

A reliable pipeline tries to isolate failures where it is safe to do so.

For example:

~~~text
Batch
 |
 +--> Record A -> success
 |
 +--> Record B -> failure
 |
 +--> Record C -> success
 |
 +--> Record D -> success
~~~

The failure of B does not necessarily need to stop A, C, and D.

But consider:

~~~text
Batch
 |
 +--> shared database unavailable
 |
 +--> every record depends on database
~~~

Now the failure is shared.

Continuing may only create more failures.

This is why error isolation should be based on dependency boundaries.

---

## 17. Structured Error Information

An error should contain enough information to investigate the failure.

Useful fields may include:

~~~text
record_id
error_type
error_message
occurred_at
attempt_count
pipeline_stage
source
correlation_id
~~~

A generic error object might look like:

~~~json
{
  "record_id": "1001",
  "error_type": "ValidationError",
  "error_message": "currency is required",
  "pipeline_stage": "validation",
  "attempt_count": 1
}
~~~

This is a **generic example**.

The exact fields depend on the system.

---

## 18. Do Not Put Sensitive Data Into Errors

Error messages can accidentally expose sensitive information.

For example, avoid logging:

~~~text
password
authentication token
full payment details
identity documents
private credentials
~~~

A safer message might be:

~~~text
record 1001 failed validation
reason = required field missing
~~~

rather than logging the entire input record.

This matters because logs and error tables can have different access controls from the original data.

---

## 19. Error Persistence

For important pipeline failures, storing error information can be useful.

A generic failure table might look like:

~~~sql
CREATE TABLE pipeline_errors (
    id BIGSERIAL PRIMARY KEY,
    record_id BIGINT,
    error_type TEXT NOT NULL,
    error_message TEXT NOT NULL,
    pipeline_stage TEXT,
    attempt_count INTEGER,
    occurred_at TIMESTAMP NOT NULL
);
~~~

This is a **generic example**.

A failure table can support:

- investigation
- dashboards
- retry decisions
- audit trails
- operational reporting

But storing every error forever may create unnecessary storage.

Retention should be considered.

---

## 20. Current Error vs Error History

There is a difference between storing the latest error and storing every error.

A record might currently have:

~~~text
status = failed
last_error = timeout
~~~

But the actual history could be:

~~~text
attempt 1 -> timeout
attempt 2 -> rate limited
attempt 3 -> timeout
attempt 4 -> completed
~~~

If only the latest error is stored, the previous failures disappear.

A separate error-history table can preserve the sequence when detailed troubleshooting is required.

---

## 21. Error Handling and Processing Status

The processing-status model from Recipe 6 should work with error handling.

For example:

~~~text
pending
   |
   v
processing
   |
   X
error
   |
   v
failed
~~~

Then:

~~~text
failed
   |
   v
retry allowed?
   |
   v
processing
~~~

Or:

~~~text
failed
   |
   v
non-retryable
   |
   v
quarantined
~~~

The status tells the pipeline where the record is.

The error information explains why it is there.

---

## 22. Error Handling and Retry

Error handling should classify the failure before deciding whether to retry.

For example:

~~~text
process
  |
  v
error
  |
  v
classify
  |
  +---- retryable ------> retry policy
  |
  +---- non-retryable --> quarantine
  |
  +---- unknown --------> alert / investigate
~~~

This prevents the retry system from having to understand every possible exception itself.

A useful separation is:

~~~text
Error classification
        |
        v
Retry policy
        |
        v
Retry execution
~~~

Each part has a different responsibility.

---

## 23. Retry Limits

Even retryable errors should have limits.

For example:

~~~text
attempt 1 -> fail
attempt 2 -> fail
attempt 3 -> fail
attempt 4 -> stop
~~~

After the maximum number of attempts:

~~~text
failed
   |
   v
quarantine / manual review
~~~

Without a retry limit, one bad dependency can keep a record in a retry loop indefinitely.

Retry behavior was discussed in Chapter 6.

Here the focus is how error classification feeds that retry system.

---

## 24. Error Handling and Transactions

Transaction boundaries matter when an error occurs.

Suppose a database transaction does:

~~~text
BEGIN
    write result A
    write result B
    update status
COMMIT
~~~

If an error occurs before commit:

~~~text
ROLLBACK
~~~

The partial transaction should not remain.

But consider:

~~~text
database transaction
       |
       v
external API call
       |
       v
database commit
~~~

The external API may already have changed state before the database transaction fails.

A database rollback cannot undo that external operation automatically.

This is another reason idempotency and external-side-effect design are important.

---

## 25. Do Not Swallow Exceptions

A dangerous pattern is:

~~~python
try:
    process_record(record)
except Exception:
    pass
~~~

The error disappears.

The pipeline may continue as if nothing happened.

This can produce silent data loss.

A better approach is to:

1. capture the error
2. classify it
3. update processing state
4. persist useful information when required
5. log safely
6. retry or quarantine according to policy
7. propagate the error when the surrounding system needs to know

The exact sequence depends on the architecture.

---

## 26. Avoid One Giant Exception Handler

Another common pattern is:

~~~python
try:
    entire_pipeline()
except Exception:
    handle_error()
~~~

This can be useful as a final safety boundary.

But it should not replace specific error handling.

If all failures become one generic error, the pipeline loses important information.

For example:

~~~text
ValidationError
DatabaseTimeout
RateLimit
ProgrammingBug
~~~

should not necessarily have identical behavior.

A top-level handler can catch unexpected failures.

Lower-level code should handle known failure categories appropriately.

---

## 27. Custom Exceptions

A codebase can define specific exception types.

For example:

~~~python
class ValidationError(Exception):
    pass


class RetryableError(Exception):
    pass


class NonRetryableError(Exception):
    pass
~~~

This is a **generic example**.

The goal is not to create dozens of exception classes.

The goal is to make meaningful error categories explicit when they improve control flow.

---

## 28. Error Context

When an error is raised, useful context should travel with it.

For example:

~~~text
error_type
record_id
pipeline_stage
source
attempt
correlation_id
~~~

Suppose the same error appears in hundreds of records.

Without context:

~~~text
timeout
~~~

With context:

~~~text
timeout
record_id = 1001
stage = external_api
attempt = 2
~~~

The second message is much easier to investigate.

---

## 29. Correlation IDs

A correlation ID can connect logs from different components.

For example:

~~~text
ingestion
    |
    v
validation
    |
    v
processing
    |
    v
database
~~~

All related operations can carry the same correlation ID.

Then an engineer can search:

~~~text
correlation_id = abc-123
~~~

and follow the operation across the pipeline.

This is especially useful when a single record passes through multiple services.

---

## 30. Error Logging

A useful error log should answer:

- what failed?
- where did it fail?
- which record was involved?
- what type of error occurred?
- can it be retried?
- which attempt was this?
- when did it happen?
- how can the operation be correlated with other logs?

A generic structured log might contain:

~~~json
{
  "level": "ERROR",
  "event": "record_processing_failed",
  "record_id": "1001",
  "error_type": "TimeoutError",
  "stage": "external_api",
  "attempt": 2
}
~~~

This is a **generic example**.

Do not include sensitive payload data just because it is available.

---

## 31. Error Metrics

Logs explain individual failures.

Metrics show the larger pattern.

Useful metrics can include:

~~~text
errors_total
errors_by_type
errors_by_stage
retryable_errors_total
non_retryable_errors_total
quarantined_records_total
failed_records_total
~~~

You can also measure error rates:

~~~text
error rate =
failed records / processed records
~~~

The exact metric definition should be consistent.

A sudden increase can indicate:

- upstream changes
- API degradation
- database problems
- bad deployments
- data-quality problems

---

## 32. Error Rate vs Error Count

These two metrics answer different questions.

Suppose:

~~~text
10 errors out of 100 records
~~~

Error rate:

~~~text
10%
~~~

Now:

~~~text
100 errors out of 100,000 records
~~~

Error rate:

~~~text
0.1%
~~~

The second case has more errors in absolute terms but a lower failure rate.

Monitor both when useful.

Context matters.

---

## 33. Quarantine

Some errors should move records to a quarantine path.

For example:

~~~text
record
  |
  v
validation
  |
  X
invalid
  |
  v
quarantine
~~~

Quarantine is useful when:

- the record needs investigation
- automatic retry will not help
- the original data should be preserved
- processing should continue for other records

A quarantine record may include:

~~~text
original data reference
error type
error message
failure stage
created_at
attempt_count
~~~

The exact structure depends on the pipeline.

---

## 34. Quarantine Is Not a Garbage Bin

Do not use quarantine as a place where failed records disappear.

A useful quarantine design should support:

- investigation
- classification
- correction
- replay
- retention
- monitoring

A quarantined record should have a clear reason.

For example:

~~~text
reason = invalid_currency
~~~

is more useful than:

~~~text
reason = failed
~~~

The goal is to preserve information about why the normal path could not process the record.

---

## 35. Record-Level Recovery

A useful recovery flow can look like:

~~~text
failed record
      |
      v
inspect error
      |
      +---- temporary ----> retry
      |
      +---- bad data ------> correct / quarantine
      |
      +---- code bug ------> deploy fix
      |
      +---- unknown -------> investigate
~~~

This is more useful than simply rerunning the whole pipeline.

It allows recovery to be targeted.

---

## 36. Error Handling During Replay

Replay introduces another important question.

Suppose a failed record is replayed after a bug fix.

The pipeline should determine:

- should the original error remain?
- should attempt count increase?
- should the record return to pending?
- should a new processing run be created?
- should previous error history be preserved?
- is the processing idempotent?

A useful history might look like:

~~~text
Run 1
  attempt 1 -> validation failure

Fix applied

Run 2
  attempt 1 -> completed
~~~

Do not erase the original failure if the history is operationally important.

---

## 37. Error Handling During Backfill

Backfills can produce large numbers of failures.

For example:

~~~text
1,000,000 historical records
        |
        v
20,000 failures
~~~

Stopping the entire backfill may be too expensive.

But ignoring the failures is also dangerous.

A controlled backfill can separate:

~~~text
successful records
failed records
quarantined records
~~~

Then the failed population can be investigated separately.

This makes large historical operations easier to recover.

---

## 38. Testing Error Handling

Error handling needs failure tests.

At minimum, test:

### Test 1 — Validation failure

Expected:

~~~text
record rejected or quarantined
~~~

### Test 2 — Temporary API failure

Expected:

~~~text
retry path
~~~

### Test 3 — Permanent API error

Expected:

~~~text
no unnecessary retries
~~~

### Test 4 — Database timeout

Expected:

~~~text
appropriate retry or failure handling
~~~

### Test 5 — Unexpected exception

Expected:

~~~text
error captured and observable
~~~

### Test 6 — Error during transaction

Expected:

~~~text
transaction rolled back
~~~

### Test 7 — Error after external side effect

Expected:

~~~text
idempotency or compensation policy is applied
~~~

### Test 8 — Multiple failing records

Expected:

~~~text
failure isolation follows the pipeline design
~~~

---

## 39. Test the Error Classification

Do not only test that an exception occurs.

Test the classification.

For example:

~~~text
HTTP 400
    |
    v
non-retryable
~~~

and:

~~~text
HTTP 503
    |
    v
retryable
~~~

The classification itself is part of the business and operational logic.

A wrong classification can create either:

- unnecessary retries
- unnecessary permanent failures

---

## 40. Practical Investigation Example

Suppose a pipeline dashboard shows:

~~~text
processed: 100,000
failed:      5,000
~~~

Start by grouping failures.

For example:

~~~text
ValidationError      3,800
TimeoutError            700
DatabaseError           300
UnknownError            200
~~~

Now the problem is easier to understand.

The majority may be data quality.

The timeout errors may indicate an external dependency problem.

The database errors may require infrastructure investigation.

The unknown errors may indicate a software bug.

This is much more useful than one number called failed.

---

## 41. Error Investigation Workflow

When a production failure occurs, use a structured sequence.

### Step 1 — Find the failed records

Identify affected record IDs or processing units.

### Step 2 — Group by error type

Look for patterns.

### Step 3 — Group by pipeline stage

Determine where failures occur.

### Step 4 — Check timing

Look for a common start time.

### Step 5 — Check dependencies

Inspect APIs, databases, queues, storage, and other external systems.

### Step 6 — Check recent changes

Look for deployments, migrations, configuration changes, or source changes.

### Step 7 — Decide whether failures are retryable

Do not retry blindly.

### Step 8 — Recover

Retry, replay, correct, quarantine, or redeploy as appropriate.

### Step 9 — Verify

Confirm that the affected records reach the expected final state.

---

## 42. Common Mistakes

### Mistake 1 — Catching every exception and ignoring it

This creates silent failures.

### Mistake 2 — Retrying every exception

Some failures will never succeed without changing the input or code.

### Mistake 3 — Logging the full input record

This can expose sensitive data.

### Mistake 4 — Using one generic error type

This makes automated handling and investigation difficult.

### Mistake 5 — No error persistence

Important failures disappear when logs expire.

### Mistake 6 — No retry limit

A permanent failure can retry forever.

### Mistake 7 — Treating batch failures as record failures

Some system failures affect the whole processing unit.

### Mistake 8 — Treating record failures as batch failures

One bad record should not always stop thousands of independent records.

### Mistake 9 — Losing transaction boundaries

A failed operation can leave partial results.

### Mistake 10 — No recovery procedure

Knowing why something failed is not enough.

The pipeline must also have a path to recover.

---

## 43. Practical Implementation Sequence

For an existing repository, use this order:

~~~text
1. Understand the processing flow
        |
        v
2. Identify failure points
        |
        v
3. Inspect existing exception handling
        |
        v
4. Classify expected errors
        |
        v
5. Classify retryable errors
        |
        v
6. Classify non-retryable errors
        |
        v
7. Define record-level vs batch-level behavior
        |
        v
8. Define processing-status transitions
        |
        v
9. Define error persistence requirements
        |
        v
10. Define safe logging fields
        |
        v
11. Implement specific error handling
        |
        v
12. Connect errors to retry logic
        |
        v
13. Connect errors to quarantine
        |
        v
14. Review transaction boundaries
        |
        v
15. Add error metrics
        |
        v
16. Add failure tests
        |
        v
17. Test retryable and permanent failures
        |
        v
18. Test recovery
        |
        v
19. Verify final record states
        |
        v
20. Document the recovery procedure
~~~

The important part is to design the error categories before writing the exception handlers.

---

## 44. Definition of Done

The error-handling implementation is complete when:

- [ ] Failure points have been identified.
- [ ] Expected errors are classified.
- [ ] Retryable errors are identified.
- [ ] Non-retryable errors are identified.
- [ ] Unknown errors are still observable.
- [ ] Record-level and batch-level failures are distinguished.
- [ ] Error information contains useful context.
- [ ] Sensitive information is excluded from logs.
- [ ] Processing status is updated correctly after failure.
- [ ] Retry behavior is connected to error classification.
- [ ] Retry limits are defined where required.
- [ ] Quarantine behavior is defined for appropriate failures.
- [ ] Transaction behavior has been reviewed.
- [ ] External side effects have an idempotency strategy.
- [ ] Error metrics are available.
- [ ] Failure paths are tested.
- [ ] Retryable failures are tested.
- [ ] Non-retryable failures are tested.
- [ ] Worker and batch failures are tested where relevant.
- [ ] Recovery procedures are documented.
- [ ] Final record states can be verified.

---

## 45. What You Learned

Error handling is not simply adding a try/except block.

It is a decision system.

A reliable pipeline asks:

~~~text
Something failed
      |
      v
What failed?
      |
      v
Where did it fail?
      |
      v
Can it be retried?
      |
   +--+--+
   |     |
  Yes    No
   |     |
   v     v
Retry  Quarantine
   |
   v
Verify
~~~

The most important lesson is:

> An error should lead to a deliberate next action.

That action may be:

- retry
- reject
- quarantine
- stop
- alert
- investigate
- replay
- recover

The correct action depends on the failure.

Error handling also connects the reliability features built throughout Part III:

~~~text
Validation
    |
    v
Processing Status
    |
    v
Error Classification
    |
    +------> Retry
    |
    +------> Quarantine
    |
    +------> Recovery
    |
    v
Observability
~~~

Once errors have clear categories and recovery paths, the pipeline becomes much easier to operate.
