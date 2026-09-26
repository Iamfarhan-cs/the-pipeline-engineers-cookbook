# Recipe 6 — Retries and Failures

Failures are normal in data pipelines.

A network request can time out. A database can become unavailable. An API can return an error. A worker can crash. A record can contain invalid data. A service can restart while a pipeline is processing data.

The important question is not:

> How do I build a pipeline that never fails?

The better question is:

> What should the pipeline do when something fails?

A production pipeline needs a clear failure strategy.

That strategy normally includes:

- Detecting failures
- Classifying failures
- Deciding whether to retry
- Waiting before retrying when appropriate
- Limiting retry attempts
- Recording failure information
- Preventing duplicate side effects
- Moving permanently failed data somewhere safe
- Recovering from partial processing
- Making the failure visible to operators

This chapter explains these ideas step by step.

---

# What Is a Failure?

A failure happens when a pipeline cannot complete an operation as expected.

For example:

~~~text
Source
  |
  v
Extract
  |
  v
Validate
  |
  X
Failure
~~~

The failure may happen at any stage.

A pipeline can fail while:

- Reading data
- Calling an API
- Parsing a response
- Validating a record
- Connecting to a database
- Writing data
- Publishing an event
- Committing a transaction
- Running a transformation
- Loading a warehouse
- Updating processing state

Different failures require different responses.

That is why failure classification matters.

---

# Temporary vs Permanent Failures

One of the first questions should be:

**Can the same operation succeed later without changing the input?**

If the answer is yes, the failure may be temporary.

If the answer is no, the failure may be permanent.

---

## Temporary Failures

A temporary failure may disappear if the operation is attempted again.

Examples include:

- Network timeout
- Temporary DNS problem
- Database unavailable
- Connection reset
- Service temporarily overloaded
- HTTP 429 rate limit response
- Temporary upstream outage
- Worker restart

For these failures, retrying can be useful.

For example:

~~~text
Request
  |
  X
Temporary failure
  |
  v
Wait
  |
  v
Retry
  |
  v
Success
~~~

---

## Permanent Failures

A permanent failure usually cannot be fixed by repeating the same operation.

Examples include:

- Invalid JSON
- Missing required field
- Invalid data type
- Unsupported schema
- Invalid business rule
- Corrupt input
- Invalid identifier
- Record that violates a permanent constraint

Consider:

~~~text
amount = "not-a-number"
~~~

Retrying the same value ten times does not make it numeric.

Repeated retries waste resources and delay other work.

Permanent failures usually need to be recorded, rejected, corrected, or quarantined.

---

# Why Retrying Everything Is Dangerous

A simple strategy might be:

> If anything fails, retry it.

This sounds reasonable.

It can cause serious problems.

Imagine a database is unavailable.

One worker retries every second.

Now imagine 100 workers doing the same thing.

Instead of helping the database recover, the pipeline creates even more traffic.

This can make the original outage worse.

This behavior is sometimes called a **retry storm**.

A retry strategy therefore needs:

- A retry limit
- A delay
- Appropriate backoff
- Failure classification
- Monitoring

---

# Basic Retry Flow

A simple retry process looks like:

~~~text
Start
  |
  v
Attempt operation
  |
  +---- success ----> Done
  |
  +---- failure ----> Is it retryable?
                         |
                    No   |   Yes
                    |    |
                    v    v
                  Failed Wait
                         |
                         v
                       Retry
~~~

The pipeline should not blindly retry every error.

---

# Retry Count

A retry policy normally limits how many times an operation can be attempted.

For example:

~~~text
Attempt 1 -> failure
Attempt 2 -> failure
Attempt 3 -> failure
Attempt 4 -> failure
Stop
~~~

The number is a policy decision.

There is no universal correct value.

It depends on:

- How long the failure is expected to last
- Cost of each attempt
- Importance of the operation
- Source rate limits
- Downstream capacity
- Maximum acceptable delay

A retry policy should have a clear stopping point.

---

# Backoff

Backoff means increasing the waiting time between retry attempts.

Instead of:

~~~text
Retry immediately
Retry immediately
Retry immediately
~~~

the pipeline can use:

~~~text
Attempt 1 -> fail
     |
     v
    wait
     |
Attempt 2 -> fail
     |
     v
    wait longer
     |
Attempt 3 -> fail
     |
     v
    wait even longer
~~~

The exact delay depends on the retry policy.

---

# Exponential Backoff

A common strategy is exponential backoff.

A simplified example could be:

~~~text
Retry 1 -> wait 1 second
Retry 2 -> wait 2 seconds
Retry 3 -> wait 4 seconds
Retry 4 -> wait 8 seconds
~~~

The delay grows after each failure.

This gives the downstream system time to recover.

The actual implementation may use a different base, maximum delay, or formula.

---

# Jitter

If many workers fail at the same time and all retry using exactly the same schedule, they may all retry together.

For example:

~~~text
Worker A -> retry at 10:01:00
Worker B -> retry at 10:01:00
Worker C -> retry at 10:01:00
Worker D -> retry at 10:01:00
~~~

This can create another traffic spike.

**Jitter** adds some randomness to retry delays.

For example:

~~~text
Worker A -> 10:01:01
Worker B -> 10:01:03
Worker C -> 10:01:00
Worker D -> 10:01:04
~~~

The exact jitter algorithm depends on the system.

The goal is to avoid many workers retrying at exactly the same time.

---

# Retryable Errors

A pipeline should define which failures are retryable.

For example, a generic HTTP-based ingestion pipeline might treat these differently:

| Failure | Possible response |
|---|---|
| Timeout | Retry |
| Connection reset | Retry |
| HTTP 429 | Retry after appropriate delay |
| HTTP 500 | Often retry |
| HTTP 400 | Usually do not retry unchanged request |
| Invalid JSON | Do not retry unchanged data |
| Missing required field | Do not retry unchanged data |

This is a generic example.

The correct behavior depends on the source API and its documented contract.

Do not classify an error only by its number.

An HTTP 400 may sometimes indicate a request that can be corrected and retried after changing the request.

An HTTP 500 may sometimes represent a permanent application error.

The pipeline should understand the actual failure.

---

# HTTP 429 and Rate Limits

APIs may limit how frequently a client can send requests.

A service may respond with a rate-limit error.

The correct response is usually not to immediately send another request.

The API may provide information about when to retry.

A good client should respect the API's documented rate-limit behavior.

Conceptually:

~~~text
Request
  |
  v
429 Rate Limited
  |
  v
Wait
  |
  v
Retry
~~~

Retrying too aggressively can extend the problem.

---

# Database Failures

Database failures also need classification.

Examples include:

- Connection timeout
- Database unavailable
- Connection reset
- Deadlock
- Serialization failure
- Constraint violation
- Invalid SQL

These failures are not equivalent.

A temporary connection problem may be retryable.

A constraint violation caused by invalid data may not be.

A deadlock may be retryable depending on the database and transaction.

An invalid SQL statement is normally a code problem, not a data retry problem.

The pipeline should classify database errors deliberately.

---

# Deadlocks

A deadlock can happen when transactions wait for each other.

A simplified example:

~~~text
Transaction A locks Row 1
Transaction B locks Row 2

Transaction A waits for Row 2
Transaction B waits for Row 1

Deadlock
~~~

Some databases detect the deadlock and abort one transaction.

The application may be able to retry the transaction.

However, the transaction should be designed so that retrying it is safe.

This is another place where idempotency matters.

---

# Retry the Right Unit

An important design question is:

**What exactly should be retried?**

Suppose a batch contains 10,000 records and one record fails.

Should the pipeline retry:

- The entire batch?
- The failed record?
- The current database transaction?
- The API request?
- The current processing stage?

There is no universal answer.

Retrying too much wastes work.

Retrying too little may leave the pipeline incomplete.

The retry boundary should match the operation's failure boundary.

---

# Record-Level Retry

Suppose the pipeline processes events individually:

~~~text
Event A -> success
Event B -> failure
Event C -> success
~~~

If Event B fails because of a temporary problem, the pipeline may retry only Event B.

This avoids reprocessing A and C unnecessarily.

However, record-level retry requires careful tracking of processing state.

---

# Batch-Level Retry

Some operations are naturally batch-oriented.

For example:

~~~text
Read file
  |
  v
Process complete file
  |
  X
Failure
~~~

Retrying the whole file may be reasonable.

But if the file is large, restarting from the beginning may be expensive.

In that case, checkpointing or smaller processing units may be useful.

The design depends on the workload.

---

# Retry and Idempotency

Retries and idempotency are closely connected.

Suppose a database write succeeds but the response is lost.

The application retries.

Without idempotency:

~~~text
Attempt 1 -> write
Attempt 2 -> write again
             |
             v
          duplicate
~~~

With idempotency:

~~~text
Attempt 1 -> write
Attempt 2 -> same operation
             |
             v
          safe result
~~~

This is why retry logic should not be designed separately from idempotency.

---

# Retry and Transactions

Suppose a pipeline does this:

~~~text
Begin transaction
    |
    v
Write data
    |
    v
Commit
~~~

If the transaction fails before commit, the application may retry the transaction.

If the transaction commits successfully but the client does not receive the response, the retry may run against already-committed data.

The database constraint and idempotent operation must handle that situation safely.

---

# Retry and External APIs

External APIs make retries more difficult.

Suppose:

~~~text
Pipeline
   |
   v
External API
   |
   v
Operation succeeds
   |
   X
Response lost
~~~

The pipeline does not know whether the operation succeeded.

If it retries, the external operation may happen twice.

If the external API supports an idempotency key, use it when appropriate.

If it does not, the pipeline may need another reconciliation strategy.

Never assume an external API call is safe to repeat without understanding its semantics.

---

# Retry and Message Processing

Streaming consumers often need to retry failed messages.

A simplified flow is:

~~~text
Message
  |
  v
Process
  |
  X
Failure
  |
  v
Retry
~~~

But the consumer must also decide what happens when retries are exhausted.

Possible outcomes include:

- Send the message to a dead-letter destination
- Quarantine it
- Record the failure
- Pause processing
- Skip the message
- Stop the consumer

The correct choice depends on whether the message is independent or whether later messages depend on it.

---

# Poison Messages

A **poison message** is a message that repeatedly fails processing.

For example:

~~~text
Message A
   |
   v
Process -> failure
   |
   v
Retry -> failure
   |
   v
Retry -> failure
   |
   v
Retry -> failure
~~~

If the consumer keeps retrying forever, it may never make progress.

A poison message can therefore block processing or consume large amounts of resources.

A production system needs a strategy for these messages.

Possible strategies include:

- Maximum retry attempts
- Dead-letter storage
- Quarantine
- Manual investigation
- Fixing the underlying data or code

---

# Dead-Letter Destinations

A dead-letter destination stores messages that could not be processed successfully after the allowed retry behavior.

A conceptual flow is:

~~~text
Event
  |
  v
Process
  |
  X
Retry
  |
  X
Retry limit reached
  |
  v
Dead-letter destination
~~~

The purpose is not to hide the failure.

The purpose is to separate permanently failed work from the normal processing path while preserving it for investigation or later recovery.

A dead-letter record should normally contain enough information to investigate the failure.

Depending on the system, useful information may include:

- Original payload or reference to it
- Event or record identifier
- Failure reason
- First failure time
- Last failure time
- Retry count
- Processing component
- Relevant error details

Do not expose sensitive information unnecessarily.

---

# Quarantine vs Dead Letter

These terms are sometimes used interchangeably, but a system may give them different responsibilities.

Quarantine often emphasizes keeping invalid or suspicious data separate until it can be investigated.

Dead-letter handling often emphasizes messages that exhausted normal processing attempts.

Either model can work.

What matters is that the team clearly defines:

- Why data enters the area
- What information is stored
- Who investigates it
- How it can be recovered
- When it can be deleted

---

# Retry Storms

A retry storm happens when a large number of failed operations retry at the same time.

Consider 1,000 workers.

All of them experience a database outage.

All of them immediately retry.

The database receives another 1,000 requests.

If the database is still unhealthy, the cycle repeats.

A safer design uses:

- Backoff
- Jitter
- Retry limits
- Concurrency control
- Circuit breaking where appropriate
- Monitoring

The exact combination depends on the architecture.

---

# Circuit Breakers

A circuit breaker is a pattern that can temporarily stop calls to a failing dependency.

A simplified model is:

~~~text
Normal
  |
  v
Requests allowed
  |
  v
Repeated failures
  |
  v
Circuit opens
  |
  v
Requests temporarily blocked
  |
  v
Recovery check
  |
  v
Requests allowed again
~~~

The goal is to prevent an unhealthy dependency from receiving a continuous stream of requests.

Circuit breakers are more common in service-to-service systems, but the same principle can be useful when designing robust pipeline dependencies.

---

# Retry Does Not Mean Hide the Error

A common mistake is to catch an error and keep retrying without recording it.

That makes the system difficult to operate.

A useful failure record should answer:

- What failed?
- When did it fail?
- Which input was being processed?
- Which attempt failed?
- Why did it fail?
- Will it retry?
- When will it retry?
- What happens if the retry limit is reached?

Retries should improve reliability without hiding operational information.

---

# Failure State

A pipeline often benefits from explicit processing states.

For example:

~~~text
RECEIVED
   |
   v
PROCESSING
   |
   +---- success ----> COMPLETED
   |
   +---- retryable --> RETRY_PENDING
   |                       |
   |                       v
   |                    PROCESSING
   |
   +---- permanent --> FAILED
                           |
                           v
                       QUARANTINED
~~~

The exact state names are system-specific.

The important point is that the pipeline should make progress and failure visible.

---

# Partial Processing

One of the hardest problems is partial processing.

Suppose a batch contains five records:

~~~text
A B C D E
~~~

The pipeline processes A, B, and C.

Then it fails.

The destination now contains:

~~~text
A B C
~~~

When the pipeline restarts, it must know what to do with A, B, and C.

Possible strategies include:

- Rerun everything with idempotent writes
- Resume from a checkpoint
- Retry only failed work
- Replace the affected batch
- Roll back the entire transaction

The correct strategy depends on the processing model.

This is why failure behavior should be designed before production deployment.

---

# A Practical Failure Example

Consider a generic API ingestion pipeline.

~~~text
API
 |
 v
Fetch page
 |
 v
Parse JSON
 |
 v
Validate records
 |
 v
Write PostgreSQL
 |
 v
Mark batch complete
~~~

Now imagine the API returns a timeout.

That is likely a temporary failure.

The pipeline may retry the request using backoff.

Now imagine the API returns malformed JSON.

Retrying the exact same response will probably not fix the problem.

The pipeline should record the failure and investigate the source or payload.

Now imagine PostgreSQL is temporarily unavailable.

The database operation may be retried if the failure is known to be transient and the write is safe to repeat.

Three failures occurred.

They require three different responses.

That is the main idea of failure classification.

---

# Designing a Retry Policy

A practical retry policy should define:

### 1. Retryable failures

Which failures can be retried?

### 2. Maximum attempts

How many times can the operation be attempted?

### 3. Backoff

How long should the pipeline wait between attempts?

### 4. Jitter

Should retry timing include variation?

### 5. Timeout

How long can one attempt run before it is considered failed?

### 6. Retry boundary

What exact operation is repeated?

### 7. Final failure behavior

What happens after retries are exhausted?

### 8. Observability

How will operators know that retries are happening?

These should be explicit design decisions.

---

# Retry Budget

A retry can consume resources.

Those resources may include:

- CPU
- Memory
- Network connections
- API quota
- Database connections
- Worker capacity
- Queue capacity
- Time

A retry policy should therefore have a practical budget.

For example:

~~~text
Maximum attempts = 5
Maximum delay = 60 seconds
Maximum total retry time = 5 minutes
~~~

These values are only examples.

The correct values depend on the system.

---

# Retry Observability

Retries should be measurable.

Useful metrics can include:

- Retry count
- Retry rate
- Failures by type
- Failures by dependency
- Exhausted retries
- Dead-letter count
- Time spent waiting
- Processing latency

Logs can include:

- Input identifier
- Attempt number
- Error category
- Dependency
- Timestamp
- Next retry time

Metrics show the overall problem.

Logs help investigate individual failures.

---

# Testing Failure Handling

Failure behavior should be tested deliberately.

Do not test only the successful path.

Useful tests include:

## Test 1 — Temporary failure

Force a temporary dependency failure.

Verify that the operation retries.

## Test 2 — Permanent failure

Provide invalid input.

Verify that the pipeline does not retry forever.

## Test 3 — Retry exhaustion

Make every attempt fail.

Verify that the final failure state is correct.

## Test 4 — Backoff

Verify that retry attempts do not happen immediately when backoff is required.

## Test 5 — Duplicate safety

Make the first attempt succeed but simulate a lost response.

Retry the operation.

Verify that the final result is not duplicated.

## Test 6 — Restart

Stop the worker during processing.

Restart it.

Verify that the pipeline recovers correctly.

## Test 7 — Poison input

Send an input that always fails.

Verify that it cannot block the entire pipeline indefinitely.

---

# Common Mistakes

## Mistake 1: Retrying every error

Some failures are permanent.

Retry only when retrying has a reasonable chance of success.

## Mistake 2: Retrying immediately

Immediate retries can overload an already unhealthy dependency.

Use appropriate delays.

## Mistake 3: No retry limit

A permanently failing operation can consume resources forever.

Set a clear limit.

## Mistake 4: No idempotency

A retry can create duplicate data or duplicate side effects.

Retries and idempotency must be designed together.

## Mistake 5: Retrying the wrong unit

Retrying an entire large batch because one record failed may waste significant work.

Choose the retry boundary deliberately.

## Mistake 6: Hiding retry failures

If the system retries silently, operators may not know the pipeline is unhealthy.

Record and measure retries.

## Mistake 7: Treating dead-letter storage as a trash bin

Failed records still need investigation and recovery procedures.

A dead-letter destination should be part of the recovery design.

---

# Production Considerations

A production pipeline should document:

- Which failures are retryable
- Which failures are permanent
- Maximum retry attempts
- Backoff policy
- Jitter policy
- Timeouts
- Retry boundary
- Idempotency strategy
- Failure states
- Dead-letter or quarantine behavior
- Alert thresholds
- Recovery procedure
- Manual intervention procedure
- Retention of failure records

Operators should be able to answer:

> What happened?

> Is the pipeline still making progress?

> Will it retry automatically?

> What happens if retries are exhausted?

> How do I recover the failed data?

These are operational requirements, not optional documentation.

---

# Practical Checklist

Before adding retries to a pipeline, ask:

- [ ] What can fail?
- [ ] Which failures are temporary?
- [ ] Which failures are permanent?
- [ ] Which failures should be retried?
- [ ] What is the maximum retry count?
- [ ] What backoff strategy is used?
- [ ] Is jitter needed?
- [ ] What is the timeout for one attempt?
- [ ] What exact operation is retried?
- [ ] Is the operation idempotent?
- [ ] What happens when retries are exhausted?
- [ ] Where are failed records stored?
- [ ] Can failed records be replayed?
- [ ] How are retries monitored?
- [ ] How are retry storms prevented?
- [ ] What happens after a worker restart?
- [ ] How is partial processing recovered?
- [ ] Has failure behavior been tested?

If these questions have clear answers, retry behavior becomes much easier to operate.

---

# What You Learned

In this recipe, you learned:

- Failures are normal in production pipelines.
- Temporary and permanent failures should be treated differently.
- Retrying everything can make an outage worse.
- Retry limits prevent endless work.
- Backoff gives dependencies time to recover.
- Jitter can prevent many workers from retrying together.
- The retry boundary should match the operation's failure boundary.
- Idempotency is required for safe retries.
- Poison messages need a separate handling strategy.
- Dead-letter or quarantine areas preserve failed work for investigation.
- Partial processing requires an explicit recovery strategy.
- Retry behavior should be observable and tested.

The main lesson is:

**A retry is not a failure strategy by itself. A reliable pipeline knows what failed, why it failed, whether it should retry, when it should stop, and how the failed work can be recovered.**

---
