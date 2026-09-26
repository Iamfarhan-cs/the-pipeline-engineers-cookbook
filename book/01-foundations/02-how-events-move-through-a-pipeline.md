# Chapter 2 — How Events Move Through a Pipeline

In the previous chapter, we looked at the basic idea of a data pipeline.

A pipeline receives data, processes it, and sends the result somewhere else.

Now we need to look more closely at the data itself.

One common form of data that moves through modern systems is an **event**.

You will see events everywhere:

- a customer creates an account
- a payment is submitted
- a payment succeeds
- a user logs in
- a file is uploaded
- an order is created
- a shipment is updated
- a service reports an error
- a user opens a page

An event tells us that something happened.

That sounds simple.

But once events start moving between systems, several engineering questions appear:

- Where was the event created?
- What information does it contain?
- How does another system receive it?
- Can the same event arrive twice?
- What happens if the consumer is unavailable?
- How do we know whether the event was processed?
- Can we process it again later?

This chapter builds the mental model we will use for event-based pipelines throughout the book.

## What Is an Event?

An event is a record that describes something that happened.

For example:

~~~json
{
  "event_id": "evt_1001",
  "event_name": "payment_created",
  "occurred_at": "2026-09-26T10:15:00Z",
  "payment_id": "pay_1001",
  "amount": 250.00,
  "currency": "EUR"
}
~~~

This event tells us several things:

- the event has an identifier
- the event has a name
- the event happened at a particular time
- it is related to a payment
- the payment amount is 250 EUR

The exact structure depends on the system.

The important idea is:

> An event describes something that happened at a particular point in time.

For example:

~~~text
Customer creates account
        |
        v
account_created event
~~~

Or:

~~~text
Payment succeeds
        |
        v
payment_succeeded event
~~~

The application creates the event.

Another part of the system can then receive and process it.

## Event vs Data

It is useful to understand the difference between general data and an event.

A database row might represent the **current state** of something.

For example:

~~~text
payment_id = pay_1001
status     = completed
amount     = 250.00
~~~

An event usually describes a **change or occurrence**.

For example:

~~~json
{
  "event_name": "payment_completed",
  "payment_id": "pay_1001"
}
~~~

The database row tells us something about the current state.

The event tells us that something happened.

This distinction becomes important when building event-driven systems.

A system may use both.

For example:

~~~text
Application
   |
   +------------------> Database
   |
   +------------------> Event
~~~

The database can hold the current state while the event can communicate a change to another system.

## A Simple Event Lifecycle

Let's follow an event from creation to processing.

~~~text
Event Created
     |
     v
Event Sent
     |
     v
Event Received
     |
     v
Event Validated
     |
     v
Event Processed
     |
     v
Result Stored
~~~

There can be more steps in a real system, but this gives us a useful starting point.

Each step creates its own questions.

### Event Created

Some application or service decides that an event should exist.

For example:

~~~text
Payment Completed
       |
       v
Create payment_completed event
~~~

The event should contain enough information for the receiving system to understand what happened.

### Event Sent

The event needs to travel somewhere.

It might be:

- sent through an API
- written to a message broker
- placed in a file
- stored in a database
- passed directly to another service

The transport mechanism can change.

The basic problem stays the same:

> Move information about something that happened from one system to another.

### Event Received

Another system receives the event.

For example:

~~~text
Producer
   |
   v
Event
   |
   v
Consumer
~~~

The consumer now needs to decide whether the event is valid and what should happen to it.

### Event Validated

Before processing the event, the consumer may check:

- Is the event present?
- Does it have the required fields?
- Is the event name known?
- Is the event version supported?
- Are the values in the expected format?
- Is the event identifier valid?

If validation fails, the event needs a defined failure path.

### Event Processed

If the event is valid, the consumer performs the required work.

For example:

~~~text
payment_completed
       |
       v
Update payment record
       |
       v
Record processing result
~~~

The processing depends on the system.

### Result Stored

The result may be written to:

- PostgreSQL
- another database
- a warehouse
- object storage
- another service

The event has now moved through the pipeline.

## Producer and Consumer

Two words appear frequently in event-driven systems:

**producer** and **consumer**.

A producer creates or sends events.

A consumer receives and processes events.

A simple model is:

~~~text
Producer
   |
   v
Event
   |
   v
Consumer
~~~

For example:

~~~text
Payment Service
      |
      v
payment_created
      |
      v
Transaction Service
~~~

The payment service is the producer.

The transaction service is the consumer.

The producer and consumer do not necessarily need to be part of the same application.

They can be separate services.

They can even be owned by different teams.

This separation can make systems more flexible, but it also introduces new reliability problems.

## Why Event IDs Matter

An event usually needs an identifier.

For example:

~~~json
{
  "event_id": "evt_1001",
  "event_name": "payment_created"
}
~~~

Why?

Because the receiving system may need to distinguish one event from another.

Imagine receiving:

~~~text
evt_1001
evt_1002
evt_1003
~~~

The consumer can track them separately.

The identifier can also help with duplicate detection.

For example, suppose event **evt_1001** is processed successfully.

Then the same event arrives again.

The consumer can use the identifier to determine whether it has already processed the event.

This is one of the foundations of idempotent event processing.

We will go much deeper into this later.

## Events Can Arrive More Than Once

One of the most important things to understand about event processing is that receiving an event does not always mean receiving it exactly once.

Consider:

~~~text
Producer
   |
   v
evt_1001
   |
   v
Consumer
~~~

The consumer processes it.

Then, because of a retry or another delivery condition, the same event arrives again.

~~~text
Producer
   |
   +----> evt_1001 ----> Consumer
   |
   +----> evt_1001 ----> Consumer
~~~

The consumer now sees the same event twice.

This is not necessarily a bug in the producer.

It can be a normal part of reliable delivery.

The important question is:

> What will the consumer do when the same event arrives again?

A well-designed system should answer this before production.

## Event Processing Should Be Safe to Repeat

Suppose an event means:

~~~text
payment_created
amount = 250 EUR
~~~

If the consumer processes it once, the result should be correct.

If the same event arrives again, the consumer should not incorrectly create another payment effect.

A simplified approach is:

~~~text
Event
  |
  v
Check event_id
  |
  +---- Already processed? ----> Ignore / safely handle
  |
  +---- New ------------------> Process
~~~

The exact implementation depends on the system.

Possible mechanisms include:

- unique identifiers
- database constraints
- processing-state records
- transactions
- message-system state

This is why event IDs are more than just metadata.

They can become part of the reliability design.

## Event Time and Processing Time

Another useful distinction is between **when an event happened** and **when a system processed it**.

Imagine an event says:

~~~text
occurred_at = 10:00
~~~

But the consumer does not receive it until:

~~~text
received_at = 10:03
~~~

And processing finishes at:

~~~text
processed_at = 10:04
~~~

These are different times.

A simplified timeline is:

~~~text
10:00             10:03             10:04
  |                 |                 |
  v                 v                 v
Occurred          Received          Processed
~~~

This distinction becomes important when dealing with:

- delayed events
- late data
- event ordering
- analytics
- monitoring
- replay
- historical processing

An event can be old while still being newly received by a consumer.

We will return to this idea later when we discuss late data and replay.

## Event Order Is Not Always Guaranteed

Imagine three events:

~~~text
evt_1001
evt_1002
evt_1003
~~~

You might expect them to arrive in that order.

But a distributed system can sometimes produce:

~~~text
evt_1002
evt_1001
evt_1003
~~~

Now the consumer has to decide whether order matters.

For some systems, it may not matter.

For others, it can be very important.

For example, imagine:

~~~text
account_created
      |
      v
account_verified
~~~

If account_verified is processed before account_created, the consumer may not have the state it expects.

This creates another design question:

> Does this pipeline require ordered processing?

The answer depends on the business and technical requirements.

Do not assume that events will always arrive in the order in which they were created.

## What Does an Event Contain?

There is no single event structure that works for every system.

But many useful event models contain some common information.

For example:

~~~json
{
  "event_id": "evt_1001",
  "event_name": "payment_created",
  "event_version": 1,
  "occurred_at": "2026-09-26T10:15:00Z",
  "properties": {
    "payment_id": "pay_1001",
    "amount": 250.00,
    "currency": "EUR"
  }
}
~~~

Here we have:

### event_id

Identifies the individual event.

### event_name

Describes what happened.

Examples might be:

- account_created
- payment_created
- payment_completed
- file_uploaded

### event_version

Identifies the version of the event structure or contract.

This can help when the structure changes over time.

### occurred_at

Records when the event happened.

### properties

Contains information related to the event.

The exact fields depend on the event.

This is only a **generic example**.

A real project may use a different structure.

## Event Contracts

When one system sends events to another system, both sides need to agree on what the event means.

That agreement is an **event contract**.

For example, a producer might promise that every payment_created event contains:

~~~text
event_id
payment_id
amount
currency
occurred_at
~~~

The consumer can then build its processing around that contract.

The contract answers questions such as:

- What is the event called?
- Which fields are required?
- What type should each field have?
- What does each field mean?
- Which values are allowed?
- What happens when the event changes?

Without an agreed contract, producers and consumers can easily become incompatible.

## Schema Changes

Suppose the original event looks like:

~~~json
{
  "payment_id": "pay_1001",
  "amount": 250.00,
  "currency": "EUR"
}
~~~

Later, the producer changes it:

~~~json
{
  "payment_id": "pay_1001",
  "amount": 250.00,
  "currency": "EUR",
  "payment_method": "card"
}
~~~

Adding a field may be harmless if the consumer does not require it.

But other changes can be dangerous.

For example, changing amount from a number into a string can break processing if the consumer expects a number.

Removing a required field can also break the consumer.

This is why schema evolution needs to be handled deliberately.

We will study schema evolution and data contracts in the production section of the book.

## What Happens When the Consumer Is Down?

Imagine:

~~~text
Producer
   |
   v
Event
   |
   X
Consumer unavailable
~~~

The producer has generated the event, but the consumer cannot process it.

What happens now?

The answer depends on the architecture.

The system might:

- keep the event in a durable queue
- retry delivery
- store the event temporarily
- record the failure
- send the event to another failure path

The important question is:

> Is the event still available after the consumer fails?

If the event disappears permanently, recovery becomes much harder.

This is why durable event storage and replay are important concepts in many event-driven systems.

## What Happens When Processing Fails?

The consumer may receive an event successfully but fail while processing it.

For example:

~~~text
Event
  |
  v
Validation
  |
  v
Processing
  |
  X
Error
~~~

The failure could happen because:

- the database is unavailable
- the data is invalid
- a dependency fails
- a programming error occurs
- a business rule rejects the event

The consumer needs a defined response.

Possible responses include:

- retry
- record the failure
- quarantine the event
- stop processing
- continue with other events

The correct response depends on the type of failure.

This distinction matters.

A temporary database outage may be worth retrying.

A permanently invalid event may not become valid simply because we retry it.

We will study retry and failure handling in later recipes.

## Temporary Failure vs Permanent Failure

This is one of the most useful distinctions in pipeline engineering.

### Temporary failure

A temporary failure may succeed later.

Examples:

- database temporarily unavailable
- network timeout
- service temporarily overloaded

A retry may make sense.

### Permanent failure

A permanent failure is unlikely to succeed without changing something.

Examples:

- required field is missing
- event format is invalid
- unsupported event type
- invalid value

Retrying the same invalid event repeatedly may not help.

A better approach may be to record or quarantine it.

The general idea is:

~~~text
Failure
   |
   +---- Temporary ----> Retry
   |
   +---- Permanent ----> Record / Quarantine
~~~

The actual implementation depends on the system.

## Event Acknowledgement

In some event-processing systems, the consumer needs to tell the delivery system that an event was successfully processed.

This is often called an **acknowledgement** or **ack**.

A simplified flow is:

~~~text
Event
  |
  v
Consumer
  |
  v
Process
  |
  v
Success
  |
  v
Acknowledge
~~~

The important question is:

> When should the consumer acknowledge the event?

If it acknowledges too early, the event might be considered complete even though processing failed.

For example:

~~~text
Receive Event
     |
     v
Acknowledge
     |
     v
Process
     |
     X
   ERROR
~~~

Now the system may believe the event was handled even though it was not.

On the other hand, acknowledging only after successful processing can make recovery safer, depending on the delivery system.

The exact acknowledgement behavior depends on the technology.

The principle is more general:

> Do not mark work as complete before the work is actually complete.

## At-Least-Once, At-Most-Once, and Exactly-Once

Event systems often discuss delivery semantics.

The names can sound complicated, but the basic ideas are simple.

### At-most-once

An event is delivered zero or one time.

The system tries to avoid duplicates, but an event may be lost.

### At-least-once

The system tries to make sure the event is delivered.

But the same event may be delivered more than once.

This means consumers often need duplicate-safe processing.

### Exactly-once

The goal is for the effect of processing to occur exactly once.

This is much harder than simply saying "deliver the event once."

It depends on the entire processing and storage design.

For now, the important lesson is:

> Delivery semantics affect how you design the consumer.

We will explore these concepts in more detail when we reach streaming systems.

## Events and Database Transactions

Suppose a consumer receives an event and needs to update PostgreSQL.

A simplified flow might be:

~~~text
Event
  |
  v
Validate
  |
  v
Update Database
  |
  v
Record Processing
~~~

What happens if the database update succeeds but recording the processing state fails?

The system may have completed part of the work without recording the complete result.

This is another reason transactions and idempotency matter.

A carefully designed transaction can sometimes make multiple related database operations succeed or fail together.

For example:

~~~text
Begin Transaction
       |
       v
Update Data
       |
       v
Record Processing State
       |
       v
Commit
~~~

If something fails before the commit, the transaction may be rolled back.

The exact behavior depends on the database operations and system design.

The important idea is that event processing often crosses the boundary between message handling and database state.

That boundary needs careful design.

## Events and Replay

One of the most useful properties of event-based systems is the ability to process an event again when the architecture supports it.

Imagine:

~~~text
Stored Event
     |
     v
Processing Logic
     |
     v
Result
~~~

Later, the processing logic is corrected.

The same event may be processed again:

~~~text
Stored Event
     |
     v
Corrected Processing Logic
     |
     v
New Result
~~~

This is **replay** or **reprocessing**.

Replay can be useful when:

- processing logic had a bug
- a downstream system was unavailable
- a new consumer needs historical events
- data needs to be rebuilt
- a transformation changed

Replay is not simply "run the same code again."

The processing must be designed so that replay does not create incorrect duplicate effects.

That brings us back to idempotency.

## A Complete Event Flow

Let's combine the ideas from this chapter.

~~~text
                Event Created
                     |
                     v
                  Producer
                     |
                     v
              Event Transport
                     |
                     v
                  Consumer
                     |
                     v
                Validation
                     |
              +------+------+
              |             |
           Invalid         Valid
              |             |
              v             v
          Failure Path   Processing
                            |
                            v
                         Storage
                            |
                            v
                         Success
~~~

A production system may add:

- retries
- processing state
- deduplication
- quarantine
- monitoring
- tracing
- replay
- recovery

The important thing is not the number of boxes.

It is understanding what happens at each boundary.

## A Practical Example

Let's imagine a simple order system.

A customer places an order.

The application creates:

~~~json
{
  "event_id": "evt_2001",
  "event_name": "order_created",
  "order_id": "ord_5001",
  "occurred_at": "2026-09-26T10:30:00Z"
}
~~~

The event is sent to another service.

The flow is:

~~~text
Customer
   |
   v
Order Service
   |
   v
order_created
   |
   v
Event Transport
   |
   v
Order Consumer
   |
   v
PostgreSQL
~~~

Now consider different situations.

### Normal case

The event arrives and is processed successfully.

~~~text
order_created
     |
     v
Validate
     |
     v
Process
     |
     v
Store
     |
     v
Success
~~~

### Duplicate case

The same event arrives twice.

~~~text
evt_2001
   |
   +----> Process
   |
   +----> Already processed
~~~

The second delivery should not incorrectly create another order effect.

### Database failure

The event is valid, but PostgreSQL is unavailable.

~~~text
evt_2001
   |
   v
Validate
   |
   v
Process
   |
   v
PostgreSQL
   |
   X
Failure
~~~

The system needs a recovery strategy.

### Invalid event

The event is missing a required field.

~~~text
Event
  |
  v
Validation
  |
  X
Invalid
  |
  v
Failure / Quarantine
~~~

### Replay

Later, the processing logic changes.

~~~text
Stored Event
     |
     v
New Processing Logic
     |
     v
New Result
~~~

The same event can become useful again if the system has preserved the information required for replay.

## Common Mistakes

### Mistake 1 — Treating Every Event as Unique

An event can be delivered more than once.

Design the consumer with duplicate delivery in mind.

### Mistake 2 — Assuming Arrival Order

Events may not always arrive in the order you expect.

If ordering matters, make that requirement explicit.

### Mistake 3 — Acknowledging Too Early

Do not mark an event as successfully processed before the important work is complete.

### Mistake 4 — Retrying Every Failure

Not every failure is temporary.

Retrying permanently invalid data can create unnecessary load and repeated failures.

### Mistake 5 — Ignoring Event Schema Changes

A producer and consumer need a compatible contract.

A small schema change can break downstream processing.

### Mistake 6 — Designing Events Without Replay in Mind

If historical processing may be required, think about how the original event data will be preserved.

### Mistake 7 — Putting Too Much Data Into an Event

An event should contain the information needed by its consumers.

Do not automatically copy every field from the source system into every event.

Data minimization can also reduce privacy and security risk.

## Practical Checklist

Before implementing an event-based pipeline, ask:

### Event

- [ ] What happened?
- [ ] What is the event name?
- [ ] Does the event have a unique identifier?
- [ ] When did the event occur?
- [ ] What information does the consumer need?

### Producer

- [ ] Where is the event created?
- [ ] When is it created?
- [ ] How is it sent?
- [ ] What happens if delivery fails?

### Consumer

- [ ] How is the event received?
- [ ] How is it validated?
- [ ] What processing is performed?
- [ ] When is processing considered complete?

### Duplicates

- [ ] Can an event arrive more than once?
- [ ] How are duplicates detected?
- [ ] Is processing idempotent?

### Ordering

- [ ] Does event order matter?
- [ ] Can events arrive late?
- [ ] Can events arrive out of order?

### Failure

- [ ] What happens when validation fails?
- [ ] What happens when processing fails?
- [ ] Which failures are temporary?
- [ ] Which failures are permanent?
- [ ] Should failed events be retried or quarantined?

### Recovery

- [ ] Can events be replayed?
- [ ] Is the original event preserved?
- [ ] Can processing resume after failure?

### Schema

- [ ] What is the event contract?
- [ ] Which fields are required?
- [ ] How are schema changes handled?
- [ ] Is the event versioned?

## What You Learned

An event describes something that happened.

An event can move through several stages:

~~~text
Created
  |
  v
Sent
  |
  v
Received
  |
  v
Validated
  |
  v
Processed
  |
  v
Stored
~~~

Reliable event processing requires more than receiving the message.

You need to think about:

- event identifiers
- duplicate delivery
- event time
- processing time
- ordering
- event contracts
- schema changes
- retries
- acknowledgements
- delivery semantics
- transactions
- replay
- recovery

The most important idea from this chapter is:

> Receiving an event is only the beginning. The pipeline must also define what it means to process that event successfully.

In the next chapter, we will compare two common ways of processing data:

**batch processing and streaming processing.**
