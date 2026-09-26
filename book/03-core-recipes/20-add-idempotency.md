# Recipe 20 — Add Idempotency

A pipeline can fail without losing data.

That sounds good until the pipeline tries the same work again.

A network request times out after the server already accepted it. A worker crashes after inserting a record but before marking the message as complete. A scheduled job runs twice. An operator replays a failed batch.

The pipeline now sees the same input more than once.

If the pipeline simply processes every input again, duplicate data or duplicate side effects can appear.

This is where **idempotency** becomes important.

Idempotency means that processing the same input multiple times produces the same intended result as processing it once.

This chapter turns that idea into a practical pipeline recipe.

---

## 1. Goal

The goal is to add idempotency to a pipeline so that repeated processing of the same logical event does not create unintended duplicate results.

By the end of this recipe, you should understand how to:

- identify the unit of work
- choose a stable idempotency key
- enforce uniqueness in the database
- use an idempotent write
- handle duplicate processing safely
- combine idempotency with transactions
- test repeated processing
- test concurrent processing
- handle retries and replay
- distinguish idempotency from deduplication
- verify that the final state is correct

The central idea is simple:

    Same logical input
           |
           v
    Processed once or many times
           |
           v
    Same intended final result

---

## 2. Problem

Imagine an ingestion pipeline receives this event:

~~~json
{
  "event_id": "evt-1001",
  "customer_id": "cust-42",
  "amount": 250,
  "currency": "EUR"
}
~~~

The pipeline inserts the event into a database.

The database confirms the insert.

But the response is lost because of a network problem.

The worker does not know whether the insert succeeded.

It retries.

The same event arrives again.

Without protection:

    Attempt 1
        |
        v
    INSERT event
        |
        v
    Database saves it
        |
        X
    Response lost

    Attempt 2
        |
        v
    INSERT event
        |
        v
    Duplicate row

The pipeline has processed one logical event twice.

This can happen even when the application code is correct.

The failure is caused by uncertainty between systems.

---

## 3. Why This Matters

Retries are normal.

Crashes are normal.

Workers restart.

Messages can be delivered again.

Scheduled jobs can overlap.

Operators replay data.

A production pipeline must assume that repeated execution can happen.

A useful rule is:

> If the pipeline can retry or replay a unit of work, think about what happens when that unit of work is processed twice.

This is especially important when processing has side effects.

Examples include:

- inserting database rows
- updating account balances
- creating orders
- sending notifications
- creating invoices
- calling external APIs
- publishing another event
- writing files
- updating warehouse records

Some duplicate operations are harmless.

Others are serious.

For example:

    Duplicate log
        -> usually annoying

    Duplicate telemetry event
        -> incorrect analytics

    Duplicate order
        -> incorrect business data

    Duplicate payment operation
        -> potentially serious financial problem

The pipeline design must therefore make the intended behavior explicit.

---

## 4. When to Use This Recipe

Add idempotency when a pipeline can process the same logical input more than once.

Common situations include:

- message queues
- API ingestion
- scheduled batch jobs
- event consumers
- retryable workers
- file processing
- database synchronization
- replay systems
- backfills
- ETL jobs
- webhook processing

Idempotency is particularly important when the system uses **at-least-once delivery**.

At-least-once delivery means that the system tries to make sure a message is processed, but the same message may be delivered more than once.

That is often a practical reliability choice.

It means the consumer must be prepared for duplicates.

---

## 5. Architecture

A simple idempotent pipeline can look like this:

    Source
      |
      v
    Receive
      |
      v
    Validate
      |
      v
    Find idempotency key
      |
      v
    +----------------+
    | Already exists?|
    +----------------+
      |          |
     Yes         No
      |          |
      v          v
    Skip or     Process
    return      normally
                  |
                  v
              Database
                  |
                  v
          Unique constraint

The database is an important part of this design.

Application code can check whether a record exists, but the database should enforce the final uniqueness rule.

Why?

Because multiple workers can execute at the same time.

---

## 6. Before You Start

Before changing code, answer these questions:

1. What is the unit of work?
2. What makes two inputs represent the same logical event?
3. Is there already a stable identifier?
4. Where is that identifier stored?
5. Which database table represents the processed result?
6. Can multiple workers process the same input?
7. What happens when processing is retried?
8. What happens when processing is replayed?
9. Does processing have external side effects?
10. What should happen when the same event arrives again?

Do not start by adding a random unique column.

First understand the data.

---

## 7. Repository Investigation

This is a core part of the recipe.

Before implementing idempotency in an existing repository, investigate how the pipeline currently works.

Start with the execution path:

    Entry point
        |
        v
    Input acquisition
        |
        v
    Validation
        |
        v
    Processing
        |
        v
    Database write

Then find the exact database write.

Look for:

- INSERT statements
- ORM create operations
- repository methods
- database models
- unique constraints
- primary keys
- conflict handling
- transaction boundaries

Also search for:

- event IDs
- source IDs
- external IDs
- request IDs
- message IDs
- webhook IDs
- job IDs
- batch IDs

These identifiers may already provide an idempotency key.

Do not create another identifier unless the existing identifiers cannot represent the required uniqueness.

---

## 8. Files to Inspect

For an existing repository, inspect files in roughly this order:

    README
      |
      v
    Entry point
      |
      v
    Pipeline/service code
      |
      v
    Database access code
      |
      v
    Models/schema
      |
      v
    Migrations
      |
      v
    Tests
      |
      v
    Configuration

Look for the complete path from input to database write.

You want to know:

    Where does the event enter?

    Where is its identity created?

    Where is it validated?

    Where is it transformed?

    Where is it written?

    Where is the transaction committed?

    What happens when the write fails?

    What happens when the same event is received again?

---

## 9. Files to Change

The exact files depend on the repository.

Typical changes may include:

    database migration
    database model
    repository/data-access layer
    processing service
    idempotency helper
    tests

Do not assume all of these are required.

The actual repository structure should determine the implementation.

---

## 10. Choose the Idempotency Key

The most important design decision is the idempotency key.

An idempotency key identifies one logical operation.

For example:

    event_id = evt-1001

If the same event is delivered five times:

    evt-1001
    evt-1001
    evt-1001
    evt-1001
    evt-1001

all five deliveries represent the same logical event.

The pipeline should not create five independent results.

---

### 10.1 Good Idempotency Keys

A good key is:

- stable
- unique for the intended operation
- available during processing
- stored with the result
- generated consistently
- independent of retry count

Examples:

    event_id
    webhook_id
    external_transaction_id
    source_record_id
    request_id

The correct choice depends on what the pipeline considers one logical operation.

---

### 10.2 Bad Idempotency Keys

Avoid values that change between attempts.

For example:

    current_timestamp
    random_uuid_generated_during_retry
    worker_id
    retry_number

Suppose the first attempt generates:

    retry_id = abc

and the second attempt generates:

    retry_id = xyz

The database sees two different operations.

Idempotency has failed.

The key must identify the logical operation, not the individual attempt.

---

## 11. Idempotency Key vs Primary Key

These are related but not always the same thing.

A database row may have:

    id
    event_id
    customer_id
    amount
    created_at

The id may be an internal database identifier.

The event_id may identify the external logical event.

For example:

    id       = 501
    event_id = evt-1001

If the event is retried, the application should still recognize:

    evt-1001

as the same logical input.

The internal row ID does not necessarily provide that protection.

---

## 12. Add a Database Uniqueness Rule

The application should not be the only place that knows the rule.

The database should enforce it.

For example:

~~~sql
CREATE UNIQUE INDEX
    idx_events_event_id_unique
ON events (event_id);
~~~

This is a **generic example**.

The actual table and column names must come from the repository being changed.

The important idea is:

    event_id must be unique

Now the database itself prevents two rows from representing the same event.

---

## 13. Why Application-Only Checks Are Not Enough

A common implementation is:

    1. SELECT whether event exists
    2. If it does not exist:
           INSERT event

This looks correct.

But it has a race condition.

Two workers can execute it at the same time.

    Worker A                  Worker B

    SELECT event              SELECT event
    not found                 not found

    INSERT event              INSERT event
         |                         |
         +-----------+-------------+
                     |
                 duplicate

Both workers checked before either insert became visible to the other.

This is why the database constraint matters.

The database is responsible for enforcing uniqueness.

---

## 14. Use an Idempotent Database Write

For PostgreSQL, a common pattern is an upsert or conflict-aware insert.

For example:

~~~sql
INSERT INTO events (
    event_id,
    customer_id,
    amount,
    currency
)
VALUES (
    'evt-1001',
    'cust-42',
    250,
    'EUR'
)
ON CONFLICT (event_id) DO NOTHING;
~~~

This is a **generic example**.

If the event already exists, PostgreSQL does not create another row.

The result is effectively:

    First attempt
        |
        v
    Insert succeeds
        |
        v
    One row exists


    Second attempt
        |
        v
    Conflict detected
        |
        v
    No second row

This is one of the simplest forms of idempotent database processing.

---

## 15. Should a Duplicate Be Ignored?

Not always.

There are several valid behaviors.

### Option 1 — Ignore the duplicate

    Duplicate detected
          |
          v
    No new write
          |
          v
    Continue

This is useful when duplicate delivery is expected and no further action is required.

### Option 2 — Return the existing result

    Duplicate detected
          |
          v
    Find existing record
          |
          v
    Return existing result

This is useful for APIs where the caller expects the same result from repeated requests.

### Option 3 — Record duplicate information

    Duplicate detected
          |
          +----> do not process again
          |
          +----> record duplicate metric

This can help operational monitoring.

The correct behavior depends on the pipeline.

Do not automatically treat every duplicate as an error.

---

## 16. Idempotency and Transactions

Idempotency becomes more important when several database operations belong to one logical operation.

Imagine:

    Receive event
        |
        v
    Insert event record
        |
        v
    Update account summary
        |
        v
    Write processing status

If these operations are not coordinated, a retry can produce an inconsistent result.

A transaction can group the database changes:

    BEGIN
       |
       +--> check/write idempotency record
       |
       +--> apply business change
       |
       +--> update processing state
       |
    COMMIT

If something fails:

    ROLLBACK

The exact transaction boundary should match the business operation.

Do not assume that putting everything into one large transaction is always correct.

---

## 17. The Important Failure Window

Consider this sequence:

    1. Insert event
    2. Update another table
    3. Commit
    4. Worker crashes
    5. Worker retries

If step 2 did not commit with step 1, the system can become inconsistent.

If both belong to one logical database operation, they may need to be in the same transaction.

For example:

    BEGIN

    insert event
    update target state

    COMMIT

Then either both changes become visible or neither does.

This does not automatically make external side effects idempotent, but it gives the database part of the operation a clear atomic boundary.

---

## 18. Idempotency and External Side Effects

Database idempotency does not automatically protect external systems.

Suppose processing does this:

    1. Save event in database
    2. Call external API

The API call succeeds.

Then the worker crashes before recording the result.

The worker retries.

It may call the external API again.

The database's unique constraint cannot undo the external API call.

The problem is now:

    Database idempotency
            !=
    External side-effect idempotency

For external operations, investigate whether the external system supports:

- idempotency keys
- request IDs
- duplicate detection
- safe retries
- status lookup

If the external API accepts an idempotency key, use the same stable logical key.

For example:

    event_id = evt-1001

    API request:
    Idempotency-Key: evt-1001

This is a **generic example**.

The actual external API must be checked for its supported behavior.

---

## 19. Idempotency vs Deduplication

These terms are related but different.

### Deduplication

Deduplication tries to identify duplicate inputs.

    A
    A
    B
    C
    C

          |
          v

    A
    B
    C

### Idempotency

Idempotency makes repeated processing safe.

    A
    A
    A

          |
          v

    same intended result

A pipeline can use both.

For example:

    Receive
       |
       v
    Deduplicate
       |
       v
    Validate
       |
       v
    Process idempotently
       |
       v
    Store

But deduplication is not a substitute for idempotency.

Duplicates can still appear after the deduplication step.

Retries can happen later.

Replay can happen later.

The final write boundary should still be safe.

---

## 20. Idempotency in a Batch Pipeline

Suppose a batch contains:

    evt-1
    evt-2
    evt-3
    evt-4

The pipeline processes the first three records and crashes.

The batch is restarted.

The second run receives:

    evt-1
    evt-2
    evt-3
    evt-4

Without idempotency:

    evt-1 -> processed twice
    evt-2 -> processed twice
    evt-3 -> processed twice
    evt-4 -> processed once

With idempotency:

    evt-1 -> already processed
    evt-2 -> already processed
    evt-3 -> already processed
    evt-4 -> process now

The pipeline can safely retry the whole batch.

This is one reason idempotency makes recovery much easier.

---

## 21. Idempotency in a Streaming Pipeline

Streaming systems often provide at-least-once delivery.

A simplified flow is:

    Producer
       |
       v
    Message broker
       |
       v
    Consumer
       |
       v
    Database

The consumer may receive:

    message-100
    message-101
    message-100

The database must be able to recognize that message-100 was already processed.

A common pattern is:

    message ID
         |
         v
    database uniqueness
         |
         v
    idempotent write

This allows the consumer to safely retry work.

---

## 22. Idempotency and Replay

Replay means processing historical data again.

For example:

    Raw data
       |
       v
    Replay
       |
       v
    Processing

Replay can intentionally send records through the pipeline again.

That means idempotency becomes extremely important.

Without it:

    Original processing
           |
           v
    10,000 records

    Replay
           |
           v
    another 10,000 records

    Result
           |
           v
    20,000 rows

If replay is supposed to rebuild or refresh the same logical result, this may be wrong.

A well-designed replay process needs an explicit policy.

Possible policies include:

- ignore already processed records
- update existing records
- replace a target partition
- write to a new version
- create a separate replay dataset

Do not assume that every replay should simply insert again.

---

## 23. Idempotency and Backfills

Backfills are another form of repeated processing.

Imagine processing January data again after fixing a bug.

    January data
         |
         v
    Corrected pipeline
         |
         v
    Backfill

The backfill may encounter records that already exist.

The pipeline needs to know whether the correct behavior is:

    insert
    update
    replace
    skip

This decision depends on the target data model.

Idempotency provides safety, but it does not decide the business meaning of the backfill.

---

## 24. Processing Status Is Not the Same as Idempotency

A status column might look like:

    status = pending
    status = processing
    status = completed
    status = failed

This helps track processing state.

But it does not automatically prevent duplicates.

For example:

    Worker A -> sees pending
    Worker B -> sees pending

    Worker A -> processes
    Worker B -> processes

Both workers may still perform the same work.

Processing status and idempotency solve different problems.

They can be used together:

    Idempotency
        +
    Processing status
        +
    Transaction

---

## 25. A Practical Generic Implementation

The following example uses Python and PostgreSQL.

It is a **generic learning example**, not a claim about any specific repository.

Suppose the database contains:

~~~sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    event_id TEXT NOT NULL UNIQUE,
    customer_id TEXT NOT NULL,
    amount NUMERIC NOT NULL,
    currency TEXT NOT NULL
);
~~~

The Python processing function could use a conflict-aware insert:

~~~python
def process_event(conn, event):
    with conn.cursor() as cur:
        cur.execute(
            """
            INSERT INTO events (
                event_id,
                customer_id,
                amount,
                currency
            )
            VALUES (%s, %s, %s, %s)
            ON CONFLICT (event_id) DO NOTHING
            """,
            (
                event["event_id"],
                event["customer_id"],
                event["amount"],
                event["currency"],
            ),
        )

    conn.commit()
~~~

The important part is not the Python syntax.

The important design is:

    Stable event ID
           +
    Unique database constraint
           +
    Conflict-safe write
           =
    Idempotent processing boundary

---

## 26. Returning Whether the Event Was New

Sometimes the application needs to know whether it inserted a new record.

A PostgreSQL query can use RETURNING.

For example:

~~~sql
INSERT INTO events (
    event_id,
    customer_id,
    amount,
    currency
)
VALUES (
    'evt-1001',
    'cust-42',
    250,
    'EUR'
)
ON CONFLICT (event_id) DO NOTHING
RETURNING id;
~~~

Possible outcomes:

    First processing
        |
        v
    id returned
        |
        v
    New event


    Repeated processing
        |
        v
    No row returned
        |
        v
    Existing event

This can be useful for metrics or downstream decisions.

Again, the actual behavior should match the repository's requirements.

---

## 27. Do Not Build a Check-Then-Insert Race

Avoid this pattern when concurrency is possible:

~~~python
if not event_exists(event_id):
    insert_event(event)
~~~

The problem is not that checking existence is always useless.

The problem is assuming the check itself guarantees uniqueness.

The safe boundary is:

    Application
        |
        v
    Database unique constraint
        |
        v
    Conflict-safe write

The database should enforce the invariant.

---

## 28. Testing Idempotency

Idempotency must be tested directly.

Do not assume that a unique constraint proves the whole pipeline is idempotent.

At minimum, test these cases.

---

### Test 1 — Process the Same Event Twice

Input:

    evt-1001
    evt-1001

Expected:

    one logical result

Not:

    two rows

---

### Test 2 — Process the Same Event Many Times

Input:

    evt-1001
    evt-1001
    evt-1001
    evt-1001
    evt-1001

Expected:

    one result

This checks that idempotency is stable across repeated retries.

---

### Test 3 — Process Different Events

Input:

    evt-1001
    evt-1002

Expected:

    two results

Idempotency must not accidentally reject legitimate new events.

---

### Test 4 — Concurrent Processing

Start two workers with the same event:

    Worker A
        |
        +---- evt-1001
                                     +--> database

    Worker B
        |
        +---- evt-1001

Expected:

    one logical result

This test is important because sequential tests may not expose race conditions.

---

### Test 5 — Retry After a Database Error

Simulate a failure during processing.

Then retry the same event.

Verify:

    no duplicate result

---

### Test 6 — Replay

Process an existing historical event again.

Verify that the replay follows the intended policy.

For example:

    existing event
          |
          v
    replay
          |
          v
    same intended result

---

## 29. Test the Failure Window

A stronger test simulates a failure after part of the operation has happened.

For example:

    BEGIN
       |
       v
    write event
       |
       v
    simulate failure
       |
       v
    ROLLBACK

Then retry.

The test should verify that the retry can complete correctly.

Another case is:

    database operation succeeds
           |
           v
    worker fails before acknowledgement
           |
           v
    same message delivered again

The second processing attempt should be safe.

This is a realistic failure scenario for message-driven systems.

---

## 30. Verify the Database, Not Only the Logs

A test that says:

    "duplicate detected"

is not enough.

Verify the actual database state.

For example:

~~~sql
SELECT COUNT(*)
FROM events
WHERE event_id = 'evt-1001';
~~~

Expected:

    1

Also verify the important business result.

For example:

    event count
    target record
    processing status
    related rows
    aggregate values

Idempotency is about final state, not only application messages.

---

## 31. Useful Idempotency Metrics

A production pipeline can expose metrics such as:

    events_processed_total
    events_duplicate_total
    events_failed_total
    events_retried_total

A duplicate metric can be useful.

For example:

    duplicate_events_total

If duplicate deliveries suddenly increase, the problem may be upstream.

Possible causes include:

- consumer acknowledgement failures
- worker crashes
- broker redelivery
- network instability
- retry configuration
- producer bugs

The idempotency mechanism protects the data while the metric helps reveal the underlying problem.

---

## 32. Logging Idempotency Decisions

Useful structured logging might include:

    event_id
    processing_result
    duplicate
    pipeline_stage

For example:

~~~json
{
  "event_id": "evt-1001",
  "processing_result": "duplicate",
  "duplicate": true,
  "pipeline_stage": "database_write"
}
~~~

This is a **generic example**.

Do not log sensitive payload fields just because they make debugging easier.

The event identifier itself may also be sensitive depending on the system.

Apply the project's privacy and logging rules.

---

## 33. Failure Scenarios

A production implementation should consider at least these failures.

### Failure 1 — Same event delivered twice

Expected:

    one logical result

### Failure 2 — Two workers process the same event

Expected:

    database constraint prevents duplicate result

### Failure 3 — Worker crashes after database commit

Expected:

    retry is safe

### Failure 4 — Worker crashes before database commit

Expected:

    transaction rolls back
    retry can process normally

### Failure 5 — Database becomes temporarily unavailable

Expected:

    retry according to retry policy

Idempotency makes the retry safe, but it does not replace retry handling.

### Failure 6 — Historical replay

Expected:

    replay follows an explicit data policy

---

## 34. Idempotency Does Not Fix Everything

It is important to understand what this recipe does not solve.

Idempotency does not automatically solve:

- incorrect input
- bad business rules
- missing data
- schema incompatibility
- ordering problems
- transaction design problems
- external side effects
- data corruption
- bad deduplication keys
- incorrect replay policy

It solves one important problem:

> repeated execution of the same logical operation should not create unintended additional effects.

The rest of the pipeline still needs its own controls.

---

## 35. Common Mistakes

### Mistake 1 — Using a random key for every attempt

Bad:

    retry 1 -> random key A
    retry 2 -> random key B

The retries look like different operations.

Use a stable logical key.

### Mistake 2 — Relying only on SELECT

Bad:

    SELECT
    then
    INSERT

Concurrent workers can still race.

Use a database constraint.

### Mistake 3 — Assuming primary keys provide idempotency

A generated primary key can be different on every insert.

The logical event identifier needs its own uniqueness rule when appropriate.

### Mistake 4 — Confusing idempotency with deduplication

Deduplication can remove some duplicates.

Idempotency protects the processing boundary against repeated execution.

They are related but different.

### Mistake 5 — Ignoring external side effects

A database constraint cannot prevent a duplicate email or external API call.

External systems need their own safe retry strategy.

### Mistake 6 — Not testing concurrency

A sequential test can pass while two workers still create a race.

Test concurrent processing when concurrency is possible.

### Mistake 7 — Not defining replay behavior

Replay is not automatically the same as normal processing.

Decide whether replay should:

    skip
    update
    replace
    rebuild
    version

before running a large replay.

### Mistake 8 — Treating every duplicate as an error

Duplicate delivery may be expected.

The correct response may be:

    detect
    record metric
    skip
    continue

rather than failing the whole pipeline.

---

## 36. Production Considerations

Before calling an idempotent pipeline production-ready, verify:

### Identity

- Is the idempotency key stable?
- Is it unique for the intended logical operation?
- Is it available on retries?
- Is it persisted where necessary?

### Database

- Is uniqueness enforced by the database?
- Is the constraint indexed?
- Is the write conflict-safe?
- Is the transaction boundary correct?

### Concurrency

- Can multiple workers process the same input?
- Has concurrent processing been tested?
- Can race conditions create side effects?

### Retries

- Can a failed operation be retried safely?
- Is the retry policy bounded?
- Are temporary and permanent failures separated?

### Replay

- Is replay safe?
- Is the replay policy explicit?
- Can historical processing be repeated?

### External systems

- Do external APIs support idempotency keys?
- Can external operations be repeated safely?
- Is the external result recorded correctly?

### Observability

- Are duplicate events counted?
- Can operators distinguish new processing from duplicate processing?
- Are idempotency failures visible?

---

## 37. Practical Investigation Example

Suppose an existing pipeline reports:

    Duplicate transaction rows

Do not immediately change the INSERT query.

Investigate the complete flow.

    Source
      |
      v
    Producer
      |
      v
    Message broker
      |
      v
    Consumer
      |
      v
    Retry
      |
      v
    Database

Ask:

1. Is the source sending duplicates?
2. Is the broker redelivering messages?
3. Is the consumer acknowledging too late?
4. Does the worker retry after a timeout?
5. Is the event identifier stable?
6. Does the database have a unique constraint?
7. Is the application doing check-then-insert?
8. Can two workers process the same event?
9. Is the duplicate created before or after a retry?
10. Is the database transaction correct?

This turns:

    "we have duplicate data"

into an engineering investigation.

---

## 38. Practical Implementation Sequence

For an existing repository, a useful implementation order is:

    1. Understand the current pipeline
            |
            v
    2. Find the logical unit of work
            |
            v
    3. Find an existing stable identifier
            |
            v
    4. Decide the idempotency boundary
            |
            v
    5. Inspect the target database schema
            |
            v
    6. Add the required uniqueness constraint
            |
            v
    7. Change the write operation
            |
            v
    8. Define duplicate behavior
            |
            v
    9. Review transaction boundaries
            |
            v
    10. Review external side effects
            |
            v
    11. Add unit tests
            |
            v
    12. Add integration tests
            |
            v
    13. Add concurrency testing where needed
            |
            v
    14. Test retry behavior
            |
            v
    15. Test replay behavior
            |
            v
    16. Verify final database state
            |
            v
    17. Add observability
            |
            v
    18. Document the idempotency rule

This order matters.

Do not start with code.

Start with the identity and the invariant.

---

## 39. Definition of Done

The idempotency implementation is complete when:

- [ ] The logical unit of work is clearly identified.
- [ ] A stable idempotency key has been selected.
- [ ] The key represents the correct uniqueness boundary.
- [ ] The database enforces the required uniqueness rule.
- [ ] The write operation handles conflicts safely.
- [ ] Duplicate behavior is explicitly defined.
- [ ] Transaction boundaries have been reviewed.
- [ ] External side effects have been reviewed.
- [ ] Repeated processing has been tested.
- [ ] Concurrent processing has been tested where relevant.
- [ ] Retry behavior has been tested.
- [ ] Replay behavior has been tested or explicitly defined.
- [ ] Final database state has been verified.
- [ ] Duplicate processing is observable.
- [ ] The implementation does not depend on a check-then-insert race-prone pattern.
- [ ] The idempotency rule is documented.

---

## 40. What You Learned

Idempotency is one of the most useful reliability patterns in Data Engineering.

The basic idea is simple:

    Same logical input
            |
            v
    Repeated processing
            |
            v
    Same intended result

The practical implementation requires more than an application-level check.

A strong design usually combines:

    Stable identifier
           +
    Database uniqueness
           +
    Conflict-safe write
           +
    Correct transaction boundary
           +
    Safe retry behavior
           +
    Tests
           +
    Observability

The most important lesson is this:

> Design the pipeline so that normal failure recovery does not create abnormal data.

Once a pipeline can safely process the same event more than once, retries and replay become much easier to reason about.

But idempotency is only one part of reliability.

The next problem is related but different:

**What happens when the same logical data appears multiple times with different records or identifiers?**

That is the problem of **deduplication**.
