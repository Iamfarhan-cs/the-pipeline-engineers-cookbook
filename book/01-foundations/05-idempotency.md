# Chapter 5 — Idempotency

A reliable pipeline must be safe to run more than once.

This sounds simple. In practice, it is one of the most important ideas in Data Engineering.

Consider a pipeline that receives this event:

~~~text
event_id = 1001
amount   = 250
~~~

The pipeline processes it successfully.

Later, the same event is received again.

If the pipeline inserts it again, the result may become:

~~~text
event_id = 1001
amount   = 250

event_id = 1001
amount   = 250
~~~

The pipeline processed the same input twice. This is a duplicate.

Now imagine the same problem with payments, orders, account events, inventory changes, financial transactions, or analytics events.

Duplicates can create incorrect results.

Idempotency is the design principle that helps prevent this.

The simple idea is:

**Running the same operation multiple times should produce the same intended result as running it once.**

This chapter explains what idempotency means, why duplicates happen, how to implement it, and how to test it.

---

# What Does Idempotent Mean?

An operation is idempotent when repeating the same operation does not keep changing the final result.

A simple mathematical idea is:

~~~text
f(f(x)) = f(x)
~~~

In pipeline engineering, the idea is more practical.

Suppose an event should create one database record.

First execution:

~~~text
Event A
   |
   v
Database
   |
   v
One record
~~~

Second execution of the same event:

~~~text
Event A
   |
   v
Database
   |
   v
Still one record
~~~

The second execution does not create another copy.

That is the behavior we want.

---

# Why Idempotency Matters

Data pipelines fail.

A network request can time out. A database connection can disappear. A worker can crash. A job can be restarted. A message can be delivered again. An operator can manually rerun a job. A historical batch can be replayed.

All of these situations can cause the same input to reach the pipeline more than once.

Without idempotency, a retry can turn a temporary failure into a data correctness problem.

For example:

~~~text
Source
  |
  v
Pipeline
  |
  v
Database insert
  |
  X
Response lost
~~~

The database may have successfully inserted the record.

But the application did not receive the success response.

The application may retry.

Now the same database operation is attempted again.

If the database has no protection against duplicates, the same record may be inserted twice.

The pipeline cannot assume that a failed request means the database did not change.

This is one of the most important reasons idempotency matters.

---

# A Simple Real-World Analogy

Imagine a person asks you:

> Add this customer to the list.

You add the customer.

Then the person asks again:

> Add this customer to the list.

If the list should contain one entry per customer, you should not create another identical entry.

You should recognize that the requested operation has already been completed.

That is the basic idea of idempotency.

---

# Idempotency Is Not the Same as Deduplication

These concepts are related but not identical.

## Deduplication

Deduplication identifies duplicate records or events.

For example:

~~~text
Event A
Event A
Event B
Event C
Event C
~~~

A deduplication process may reduce this to:

~~~text
Event A
Event B
Event C
~~~

## Idempotency

Idempotency makes repeated processing safe.

For example:

~~~text
Process Event A
Process Event A again
~~~

The final state should still represent one intended processing result.

A system can use deduplication to help achieve idempotent behavior.

But the two concepts should not be treated as exactly the same thing.

---

# Where Duplicates Come From

Duplicates can happen for many reasons.

## Retry

A request fails or times out and is retried.

~~~text
Attempt 1
   |
   X
Retry
   |
   v
Attempt 2
~~~

The first attempt may have already changed the destination.

## Message Redelivery

A message may be delivered again.

~~~text
Message A
   |
   v
Consumer
   |
   X
Processing acknowledgement lost
   |
   v
Message A delivered again
~~~

The consumer now sees the same message again.

## Job Restart

A batch job may fail after processing part of its input.

~~~text
Records 1 2 3 4 5
          |
          v
Process 1 2 3
          |
          X
        failure
          |
          v
Restart
          |
          v
Process 1 2 3 4 5
~~~

Records 1, 2, and 3 may be processed twice.

## Manual Rerun

An engineer may rerun a failed job.

The pipeline must not assume that the second execution starts from an empty destination.

## Replay

A historical replay intentionally sends old events through the pipeline again.

Replay is useful, but it makes idempotency even more important.

## Network Uncertainty

Distributed systems often have an uncomfortable situation: the client does not know whether the server completed the operation.

For example:

~~~text
Client
  |
  | request
  v
Server
  |
  | write succeeds
  v
Database
  |
  X
Network response lost
  |
  v
Client thinks request failed
~~~

The client retries.

The server receives the same request again.

This is a normal distributed-systems problem.

Idempotency provides a way to make the retry safe.

---

# The Most Important Question

When designing a pipeline, ask:

**What uniquely identifies this piece of data or operation?**

That identifier is often the foundation of idempotency.

For an event, it may be:

~~~text
event_id
~~~

For a transaction, it may be:

~~~text
transaction_id
~~~

For a source record, it may be:

~~~text
source_record_id
~~~

For a file, it may be:

~~~text
file_id
~~~

For a batch, it may be:

~~~text
batch_id
~~~

The correct key depends on the system.

Do not invent a key that does not represent the actual identity of the data.

---

# Idempotency Keys

An **idempotency key** is an identifier used to recognize that a request or operation has already been processed.

A generic request might contain:

~~~text
idempotency_key = abc-123
~~~

The server can use that key to determine whether the operation has already been completed.

For example:

~~~text
Request 1
key = abc-123
   |
   v
Process
   |
   v
Store result

Request 2
key = abc-123
   |
   v
Already processed
   |
   v
Return existing result
~~~

The second request does not create another operation.

---

# Event IDs as Idempotency Keys

Event-driven pipelines often already have a useful identifier.

~~~json
{
  "event_id": "evt-1001",
  "event_name": "payment_created",
  "amount": 250
}
~~~

The event ID can be used to protect the destination from duplicate processing.

A generic database design might use:

~~~sql
CREATE TABLE processed_events (
    event_id TEXT PRIMARY KEY,
    processed_at TIMESTAMP NOT NULL
);
~~~

This is a generic example.

The exact table design depends on the system.

The important part is the uniqueness constraint.

---

# Database Constraints Are Powerful

A database can enforce uniqueness.

For example:

~~~sql
CREATE TABLE payments (
    payment_id TEXT PRIMARY KEY,
    amount NUMERIC NOT NULL
);
~~~

If the pipeline attempts to insert the same payment twice, the database can reject the second insert because the primary key already exists.

This is much safer than relying only on application code.

Why?

Because multiple workers may process data at the same time.

Application-level checks can have race conditions.

For example:

~~~text
Worker A: Does ID exist?
Worker B: Does ID exist?

Worker A: No
Worker B: No

Worker A: Insert
Worker B: Insert
~~~

If there is no database uniqueness constraint, both workers may insert the record.

A database constraint provides a stronger final protection.

---

# Check-Then-Insert Is Not Always Safe

A common implementation is:

1. Check whether the record exists.
2. If it does not exist, insert it.

Conceptually:

~~~sql
SELECT 1
FROM payments
WHERE payment_id = 'pay-1001';
~~~

Then:

~~~sql
INSERT INTO payments (
    payment_id,
    amount
)
VALUES (
    'pay-1001',
    250
);
~~~

The problem is concurrency.

Two workers can perform the check at nearly the same time.

Both may see no record.

Both may attempt the insert.

This is a race condition.

A unique constraint is safer.

Depending on the database, the application can then use an atomic insert or upsert pattern.

---

# Generic Upsert Pattern

A database may support an operation conceptually like:

~~~sql
INSERT INTO payments (
    payment_id,
    amount
)
VALUES (
    'pay-1001',
    250
)
ON CONFLICT (payment_id)
DO NOTHING;
~~~

This is a PostgreSQL example.

If the record does not exist, it is inserted.

If the record already exists, the second operation does nothing.

The result is:

~~~text
First attempt  -> inserted
Second attempt -> no duplicate
Third attempt  -> no duplicate
~~~

This is a common way to make inserts idempotent.

The exact SQL depends on the database.

---

# Idempotent Does Not Always Mean Do Nothing

Sometimes the desired behavior is:

~~~text
First request:
Create record

Repeated request:
Return the existing record
~~~

Sometimes the desired behavior is:

~~~text
First request:
Create record

Repeated request:
Update the existing record
~~~

Both can be valid.

Idempotency is about the final intended result, not about one specific SQL technique.

---

# Example: Upserting Current State

Suppose a source sends customer status:

~~~text
customer_id = 100
status = ACTIVE
~~~

The pipeline receives the same event again.

The desired final state may simply remain:

~~~text
customer_id = 100
status = ACTIVE
~~~

A later event may change it:

~~~text
customer_id = 100
status = SUSPENDED
~~~

Now the system needs to distinguish between:

- Duplicate delivery of the same event
- A legitimate new state change

This is why identity and event semantics matter.

---

# Idempotency Depends on the Operation

Not every operation behaves the same way.

Consider:

~~~text
Set balance to 100
~~~

Repeating it can be idempotent:

~~~text
Set balance to 100
Set balance to 100
~~~

Now consider:

~~~text
Increase balance by 100
~~~

Repeating it:

~~~text
Increase balance by 100
Increase balance by 100
~~~

produces a different result.

The first operation sets a state.
The second operation applies an additional change.

Therefore, making a pipeline idempotent may require changing the processing design.

---

# Idempotency and Side Effects

Database writes are not the only side effects.

A pipeline may:

- Send an email
- Send a notification
- Charge a payment method
- Call another API
- Publish another event
- Create a file
- Update an external system

Suppose an event causes an email to be sent.

If the event is processed twice:

~~~text
Event
  |
  +--> Email
  |
  +--> Email
~~~

The user may receive two emails.

The database may be idempotent while the external side effect is not.

Therefore, idempotency must be considered across the entire operation.

---

# External API Calls

Suppose a pipeline calls another service:

~~~text
Pipeline
   |
   v
External API
~~~

The external API may support an idempotency key.

For example:

~~~text
idempotency_key = evt-1001
~~~

The pipeline sends the same key on retries.

The external service can recognize the duplicate request.

This is a useful pattern for operations that create external side effects.

If the external service does not support idempotency, the pipeline may need another strategy.

Possible strategies include:

- Store request state
- Store the external response
- Use a unique business key
- Use an outbox pattern
- Reconcile results
- Design the operation as a safe upsert

The correct solution depends on the external system.

---

# Idempotency and Transactions

Database transactions can help maintain correctness, but a transaction does not automatically make the entire pipeline idempotent.

Suppose the pipeline performs:

~~~text
1. Insert database record
2. Call external API
3. Commit
~~~

If the external API succeeds but the transaction fails, a retry may call the external API again.

The database transaction protects the database operation.

It does not automatically protect the external side effect.

This is why distributed workflows require careful design.

---

# Idempotency and Exactly-Once Processing

You may hear the phrase **exactly once**.

It is important to understand what this means.

At a high level:

- At-most-once means data may be processed zero or one time.
- At-least-once means data may be processed one or more times.
- Exactly-once aims to produce one intended result.

In real systems, exactly-once behavior can be difficult across multiple independent systems.

A practical approach is often:

**Allow retries and duplicate delivery, then make processing idempotent.**

This can provide correct final results even when an operation executes more than once.

---

# Idempotency in Batch Processing

Batch jobs need idempotency too.

Consider a batch:

~~~text
Batch 2026-09-26
Records:
A
B
C
D
E
~~~

The job processes A, B, and C, then fails.

When the job restarts, it may process A, B, C, D, and E.

If the pipeline is not idempotent, A, B, and C may be duplicated.

A safer design uses a stable identity for each source record.

For example:

~~~text
source_record_id
~~~

Then the destination can enforce uniqueness.

---

# Idempotency in Streaming

Streaming systems face the same problem continuously.

Suppose a consumer receives:

~~~text
Event 1
Event 2
Event 3
~~~

It processes Event 3 but crashes before acknowledging it.

After restart, Event 3 may be delivered again.

The consumer should be able to process Event 3 safely.

This often means:

- The event has a stable ID.
- The destination has a uniqueness rule.
- The write is atomic where possible.
- The processing state is consistent.
- Retries are expected.

Idempotency is therefore not a batch-only concept.

---

# Idempotency and Replay

Replay is one of the strongest reasons to design for idempotency.

Imagine that a transformation bug affected one month of data.

The pipeline needs to replay historical events.

Without idempotency:

~~~text
Existing data
     +
Replayed data
     =
Potential duplicates
~~~

With a suitable idempotent design:

~~~text
Existing data
     +
Replayed data
     =
Correct intended state
~~~

This does not mean every replay is automatically safe.

The replay logic must still understand:

- What identifies a record?
- What should happen to existing records?
- Which version of the transformation should be used?
- Should records be inserted, updated, or replaced?
- How should downstream data be refreshed?

Idempotency is a foundation, not a complete replay strategy.

---

# Idempotency and Backfills

Backfills are another common source of repeated processing.

Suppose a new business rule must be applied to historical data.

The pipeline processes January, February, and March.

Those months may already exist in the destination.

The backfill must define whether it should:

- Insert missing records
- Update existing records
- Replace a partition
- Rebuild a dataset
- Create a new version

There is no single correct answer.

The important point is that rerunning the backfill should have a predictable result.

---

# Designing an Idempotent Pipeline

A practical process is:

## Step 1 — Identify the input

Ask:

**What exactly is being processed?**

It may be:

- An event
- A transaction
- A file
- A batch
- A source row
- An API request

## Step 2 — Identify the stable identity

Ask:

**What uniquely identifies this input?**

For example:

~~~text
event_id
transaction_id
source_record_id
file_id
batch_id
~~~

Do not assume timestamps are always unique.

Do not use a random ID generated during processing as the identity of the source record.

If every retry creates a new random ID, the system cannot recognize the retry as the same input.

## Step 3 — Define the desired final result

Ask:

**What should happen if this input arrives twice?**

Possible answers:

- Ignore the duplicate
- Return the existing result
- Update the existing record
- Replace the previous state
- Merge the changes
- Record the event only once

The answer depends on the business meaning.

## Step 4 — Enforce the rule

Use appropriate mechanisms.

Possible mechanisms include:

- Primary keys
- Unique constraints
- Unique indexes
- Upserts
- Idempotency-key storage
- Processing-state tables
- Conditional writes
- Atomic transactions

Use the strongest mechanism available at the correct boundary.

## Step 5 — Test repeated execution

Do not stop after implementing the logic.

Run the same input again.

~~~text
Run 1
  |
  v
Expected result

Run 2 with same input
  |
  v
Same intended result
~~~

Then test concurrency and failure cases where relevant.

---

# A Generic Database Example

Suppose a pipeline receives:

~~~text
event_id = evt-1001
customer_id = 42
amount = 500
~~~

A destination table might be:

~~~sql
CREATE TABLE payment_events (
    event_id TEXT PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    amount NUMERIC NOT NULL,
    processed_at TIMESTAMP NOT NULL
);
~~~

The processing operation might use:

~~~sql
INSERT INTO payment_events (
    event_id,
    customer_id,
    amount,
    processed_at
)
VALUES (
    'evt-1001',
    42,
    500,
    CURRENT_TIMESTAMP
)
ON CONFLICT (event_id)
DO NOTHING;
~~~

This is a generic PostgreSQL example.

The primary key prevents multiple rows with the same event ID.

The conflict behavior makes repeated insertion safe.

However, this example only protects the database write.

If the same event also sends an email or calls another service, those side effects need their own idempotency strategy.

---

# Idempotency Keys vs Business Keys

These are sometimes confused.

A **business key** identifies a business entity or event.

For example:

~~~text
transaction_id = TX-1001
~~~

An **idempotency key** identifies a particular operation or request.

For example:

~~~text
request_id = REQ-5001
~~~

Sometimes the same identifier can serve both purposes.

Sometimes it should not.

Consider a retry:

~~~text
Request 1
request_id = REQ-5001
transaction_id = TX-1001

Request 2
request_id = REQ-5001
transaction_id = TX-1001
~~~

The same request key tells the service that this is a repeated operation.

The business key tells the service which business object is involved.

The correct distinction depends on the system.

---

# Idempotency Windows

Some systems only need to prevent duplicates within a limited period.

For example:

~~~text
Keep idempotency keys for 24 hours
~~~

Other systems need long-term uniqueness.

For example:

~~~text
One transaction ID must never be processed twice.
~~~

The retention period should be based on how long duplicates can realistically reappear.

This becomes important for:

- Storage cost
- Cleanup
- Replay
- Historical processing
- Late events

Do not choose an arbitrary retention period without understanding the data lifecycle.

---

# What If the Input Has No ID?

Sometimes a source does not provide a stable identifier.

This creates a difficult problem.

The pipeline may need to derive identity from available fields.

For example:

~~~text
source
timestamp
sequence
~~~

could potentially form a composite key.

Another possibility is a deterministic fingerprint or checksum of the source payload.

For example:

~~~text
hash(source_payload)
~~~

But this approach has limitations.

Two legitimate records with identical payloads may produce the same hash.

Two records that should be considered the same may differ in harmless fields.

Therefore, deriving identity requires understanding the source semantics.

Do not assume a hash automatically solves identity.

---

# Idempotency and Data Quality

Idempotency protects against repeated processing.

It does not guarantee that the data is correct.

For example:

~~~text
Event A
event_id = 1001
amount = -999999
~~~

The event may be unique.

But it may still be invalid.

Therefore, a reliable pipeline usually needs both:

~~~text
Idempotency
+
Data Quality
~~~

One protects processing behavior.
The other protects data correctness.

---

# Testing Idempotency

Idempotency should be tested deliberately.

## Test 1 — First execution

Input:

~~~text
Event A
~~~

Expected:

~~~text
One destination record
~~~

## Test 2 — Same input again

Input:

~~~text
Event A
~~~

Expected:

~~~text
Still one destination record
~~~

## Test 3 — Different input

Input:

~~~text
Event B
~~~

Expected:

~~~text
Two destination records
~~~

## Test 4 — Same input after restart

Stop the worker. Start it again. Process Event A again.

Expected:

~~~text
No unintended duplicate
~~~

## Test 5 — Concurrent processing

Two workers receive the same event.

Expected:

~~~text
One intended result
~~~

This test is especially useful because simple check-then-insert logic can fail under concurrency.

## Test 6 — Replay

Process historical input again.

Expected:

~~~text
Predictable final state
~~~

The exact expected behavior depends on whether the pipeline inserts, updates, replaces, or merges data.

---

# Common Mistakes

## Mistake 1: Generating a new ID during every retry

For example:

~~~text
Retry 1 -> random ID A
Retry 2 -> random ID B
Retry 3 -> random ID C
~~~

The pipeline now sees three different records.

A retry needs a stable identity.

## Mistake 2: Checking existence without a database constraint

Application code can have race conditions.

Use database constraints where appropriate.

## Mistake 3: Assuming transactions solve everything

A transaction can protect one database operation.

It does not automatically protect external APIs, messages, emails, or other side effects.

## Mistake 4: Treating duplicate delivery as an exceptional situation

In distributed systems, duplicate delivery can be normal.

Design for it.

## Mistake 5: Ignoring manual reruns

Operators will eventually rerun jobs.

The pipeline should behave predictably when they do.

## Mistake 6: Confusing unique data with idempotent processing

A unique constraint can prevent duplicate rows.

But the complete operation may still have duplicate side effects elsewhere.

## Mistake 7: Making every operation idempotent in the same way

Different operations have different semantics.

Some should ignore duplicates. Some should update. Some should return an existing result. Some need reconciliation.

Understand the operation before choosing the implementation.

---

# Production Considerations

Before calling a pipeline idempotent, document:

- What identifies the input?
- What identifies the business record?
- What identifies the operation?
- What happens when the same input arrives twice?
- Where is uniqueness enforced?
- What happens during concurrent processing?
- What happens after a worker restart?
- What happens during replay?
- What happens during backfill?
- How long are idempotency records retained?
- Which external side effects need protection?
- How are failed attempts distinguished from completed attempts?
- How is idempotency tested?

A useful design document should make these answers explicit.

---

# Practical Checklist

Before implementing idempotency, ask:

- [ ] What is the input being processed?
- [ ] Does the source provide a stable identifier?
- [ ] If not, can a reliable identity be derived?
- [ ] What should happen if the same input arrives twice?
- [ ] Is the destination protected by a unique constraint?
- [ ] Is an upsert required?
- [ ] Can two workers process the same input concurrently?
- [ ] What happens if the response is lost after a successful write?
- [ ] What happens when a job is rerun?
- [ ] What happens during replay?
- [ ] What happens during backfill?
- [ ] Are external side effects protected?
- [ ] How long must idempotency information be retained?
- [ ] Have repeated executions been tested?
- [ ] Have failure and concurrency cases been tested?

If these questions have clear answers, the pipeline is much easier to reason about.

---

# What You Learned

In this chapter, you learned:

- Idempotency makes repeated processing safe.
- Retries and restarts can cause the same input to be processed more than once.
- Duplicate delivery is a normal problem in distributed systems.
- Stable identity is the foundation of idempotent processing.
- Database uniqueness constraints are an important protection.
- Check-then-insert logic can have race conditions.
- Idempotency is different from deduplication.
- Idempotency must consider external side effects, not only database writes.
- Batch jobs, streaming consumers, replays, and backfills all need idempotency.
- Exactly-once behavior is difficult across independent systems.
- A practical design often accepts retries and duplicate delivery while making the final result safe.

The main lesson is:

**Assume the same input can arrive more than once, and design the pipeline so that this does not corrupt the result.**

---

# Recipe Preview

Later in the book, this concept will become an implementation recipe.

You will learn how to:

- Choose an idempotency key.
- Add database uniqueness.
- Implement safe inserts.
- Implement upserts.
- Track processing state.
- Protect external side effects.
- Handle concurrent duplicate processing.
- Test repeated execution.
- Make replay safe.
- Make backfills predictable.
- Investigate duplicate records in production.

The next chapter moves from duplicate processing to another unavoidable part of pipeline engineering:

**Retries and Failures.**

We will look at why failures happen, which failures should be retried, how retry policies work, and why retrying everything can make a system worse.