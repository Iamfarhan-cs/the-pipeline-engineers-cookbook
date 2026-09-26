# Chapter 1 — How Data Pipelines Work

When you first hear the words **Data Engineering**, it can sound complicated.

You may hear about Kafka, Airflow, Spark, data warehouses, streaming, orchestration, ETL, ELT, and many other tools.

But before we talk about those tools, we need to understand one simple idea:

> A data pipeline moves data from one place to another and does some work with that data along the way.

That is the basic idea.

Everything else comes from this.

## What Is a Data Pipeline?

A data pipeline is a system that receives data from a source, processes it, and delivers it to a destination.

A simple pipeline might look like this:

~~~text
Source
  |
  v
Ingestion
  |
  v
Processing
  |
  v
Destination
~~~

The source could be an API, database, file, application, event stream, or another service.

The destination could be PostgreSQL, a data warehouse, object storage, an analytics system, or another application.

The pipeline connects the two.

At first, this sounds like a simple programming task.

For example:

~~~python
data = fetch_data()
save_data(data)
~~~

This code might work perfectly on a good day.

But production systems do not always give us good days.

The API can fail. The database can be unavailable. The data can be invalid. The same event can arrive twice. The process can stop halfway through. The source can change its format. A transformation can contain a bug. Yesterday's data may need to be processed again.

A real pipeline needs to account for these situations.

## Why Do We Need Pipelines?

Software systems generate data all the time.

An application might generate:

- customer events
- payment events
- account changes
- API responses
- application logs
- telemetry events
- database records
- files
- operational metrics

That data often needs to move somewhere else.

For example, an application may use one database for its normal operations while another system is used for reporting and analysis.

~~~text
Application
    |
    v
Operational Database
    |
    v
Data Pipeline
    |
    v
Analytics System
    |
    v
Reports
~~~

The operational database and the analytics system have different jobs.

The operational database may be designed for application transactions.

The analytics system may be designed for reporting, aggregation, analysis, and historical queries.

The pipeline connects these systems.

It may need to:

1. obtain the data
2. validate it
3. transform it
4. store it
5. detect failures
6. prevent incorrect duplicate processing
7. record processing state
8. support recovery

This is why Data Engineering is more than copying records from one database to another.

## The Pipeline Has a Lifecycle

Let's follow one piece of data through a simple pipeline.

~~~text
Source
  |
  v
Acquisition
  |
  v
Validation
  |
  v
Processing
  |
  v
Storage
  |
  v
Verification
  |
  v
Monitoring
~~~

Each stage has a different responsibility.

### Source

The source is where the data comes from.

For example, imagine that a pipeline receives payment events from an external API.

The source might return an event like this:

~~~json
{
  "event_id": "evt_1001",
  "payment_id": "pay_1001",
  "amount": 250.00,
  "currency": "EUR"
}
~~~

The pipeline does not necessarily control the source.

That matters.

An external system can become unavailable, respond slowly, return invalid data, or change its behavior.

A reliable pipeline treats external dependencies as possible failure points.

### Acquisition

Acquisition is the process of getting the data.

For an API, that could mean making an HTTP request.

For a database, it could mean running a query.

For a file, it could mean reading records from the file.

For example:

~~~text
Pipeline
   |
   v
HTTP Request
   |
   v
API Response
~~~

The first important question is not only:

> Did we receive the data?

It is also:

> What happens if we cannot receive the data?

An API timeout is not an unusual event. It is something the pipeline should be designed to handle.

### Validation

After the data arrives, we need to decide whether it is acceptable.

Suppose our pipeline expects:

~~~json
{
  "payment_id": "pay_1001",
  "amount": 250.00,
  "currency": "EUR"
}
~~~

We might require:

- **payment_id** to exist
- **amount** to exist
- **amount** to be numeric
- **currency** to exist

If the data does not meet the expected rules, it needs a defined path.

~~~text
Invalid Data
     |
     +----> Reject
     |
     +----> Quarantine
~~~

The exact choice depends on the system.

The important point is that invalid data should not simply disappear.

### Processing

Processing changes the incoming data into the form required by the destination.

Processing can include:

- transformation
- normalization
- filtering
- enrichment
- deduplication
- aggregation
- business rules

For example, incoming data might contain:

~~~text
customer = "1001"
amount   = "250.00"
~~~

The pipeline could transform it into:

~~~text
customer_id = 1001
amount      = 250.00
~~~

The exact transformation depends on the requirements.

The important thing is that the behavior should be understandable and testable.

### Storage

After processing, the result is stored or delivered.

For example:

~~~text
Processing
    |
    v
PostgreSQL
~~~

Or:

~~~text
Processing
    |
    v
Data Warehouse
~~~

Or:

~~~text
Processing
    |
    v
Object Storage
~~~

The destination affects the design of the pipeline.

A PostgreSQL destination may require tables, columns, constraints, indexes, transactions, and unique keys.

A warehouse may have different loading and storage requirements.

## The Happy Path

The simplest pipeline follows what engineers often call the **happy path**.

Everything works:

~~~text
Data Received
     |
     v
Validation Passes
     |
     v
Processing Succeeds
     |
     v
Data Stored
     |
     v
Success
~~~

For our payment example:

~~~text
Payment Event
     |
     v
Event Received
     |
     v
Validation Passes
     |
     v
Transformation Succeeds
     |
     v
Database Write Succeeds
     |
     v
Pipeline Completes
~~~

This path is important.

But it is not enough.

A production pipeline cannot be designed only around the case where everything works.

The more useful question is:

> What happens when something does not work?

## The Failure Path

Imagine that the pipeline successfully receives and validates an event.

Processing also succeeds.

Then the database becomes unavailable.

~~~text
Payment Event
     |
     v
Validation
     |
     v
Processing
     |
     v
Database
     |
     X
   ERROR
~~~

Now we have to decide what the pipeline should do.

Possible approaches include:

- retry the database operation
- record the failure
- stop the current processing
- keep the input available for later processing
- move the failed item to a failure or quarantine mechanism

There is no single answer that works for every pipeline.

The correct behavior depends on the requirements.

But one principle is important:

> Failure behavior should be designed, not discovered accidentally in production.

This idea will appear throughout this book.

## Pipelines Have More Than One Path

A real pipeline often has several possible outcomes.

~~~text
                    +--> Success
                    |
Input --> Validation
                    |
                    +--> Invalid
~~~

Processing can have even more outcomes:

~~~text
                    +--> Processed
                    |
Input --> Processing +--> Retry
                    |
                    +--> Quarantine
                    |
                    +--> Failed
~~~

The exact states depend on the system.

The important thing is to think about the different paths before writing the implementation.

When designing a pipeline, ask:

> What happens to this data if this step succeeds?

Then ask:

> What happens if this step fails?

That simple habit prevents many problems later.

## What Happens When Data Arrives Twice?

Duplicate data is one of the basic problems in pipeline engineering.

Imagine that the pipeline receives:

~~~text
event_id = evt_1001
~~~

The pipeline processes it successfully.

Later, the same event arrives again:

~~~text
event_id = evt_1001
~~~

If the pipeline simply inserts it again, the destination could contain two copies.

That may be harmless in some systems.

In other systems, it can create serious problems.

For example, if an event represents a payment and the processing logic applies the payment twice, the result may be incorrect.

This is where **idempotency** becomes important.

### Idempotency

An operation is idempotent when repeating the same operation does not incorrectly create additional effects.

A simple way to think about it is:

~~~text
Event
  |
  v
Check event_id
  |
  +---- Already processed? ----> Do not process again
  |
  +---- New event -------------> Process
~~~

The exact implementation can vary.

A database might use a unique constraint.

A pipeline might keep a processing record.

A message system might use an identifier together with consumer state.

We will study these patterns in detail later.

For now, remember this:

> If the same input can arrive more than once, the pipeline needs a deliberate duplicate-handling strategy.

## What Happens When Processing Stops Halfway?

Consider this pipeline:

~~~text
Read Data
   |
   v
Validate
   |
   v
Transform
   |
   v
Write to Database
   |
   v
Mark Complete
~~~

Now imagine the database write succeeds, but the process crashes before the pipeline records that the work is complete.

The system may now be in an intermediate state.

When the pipeline starts again, it needs to answer an important question:

> What has already happened?

This is where several concepts start working together:

- transactions
- processing status
- idempotency
- checkpoints
- retries
- recovery

These are not isolated features.

They are different ways of making pipeline processing safer when something interrupts the normal flow.

## Raw Data and Staging

Some pipeline architectures preserve the original input before transforming it.

A simplified design is:

~~~text
Source
  |
  v
Raw Storage
  |
  v
Validation
  |
  v
Processing
~~~

Why might this be useful?

Imagine that a transformation contains a bug.

If the original input was preserved, the pipeline may be able to run the corrected transformation against that original data.

That can support:

- debugging
- replay
- reprocessing
- auditing
- historical processing

Raw storage is not automatically required for every pipeline.

The right architecture depends on the requirements.

### Staging

A staging layer provides an intermediate place for incoming data.

For example:

~~~text
Source
  |
  v
Ingestion
  |
  v
Staging
  |
  v
Validation
  |
  v
Transformation
  |
  v
Final Table
~~~

A staging layer can make it easier to:

- inspect incoming data
- validate data
- separate ingestion from transformation
- retry processing
- reprocess data

Again, this is a design choice.

Not every pipeline needs a separate staging layer.

Later in the book, we will look at raw, staging, curated, and warehouse layers in much more detail.

## Batch Processing

A batch pipeline processes data in groups.

For example, a pipeline might run every hour:

~~~text
Every hour
    |
    v
Read records
    |
    v
Process records
    |
    v
Write results
~~~

Or it might run once per day:

~~~text
Every day
    |
    v
Read yesterday's data
    |
    v
Process it
    |
    v
Load results
~~~

Batch processing is useful when data does not need to be processed immediately.

Batch jobs can be small scripts or large distributed systems.

## Streaming Processing

A streaming pipeline processes events continuously or in small increments.

A simplified streaming architecture is:

~~~text
Producer
   |
   v
Message Broker
   |
   v
Consumer
   |
   v
Processing
   |
   v
Destination
~~~

Instead of waiting for a scheduled batch, the system processes events as they become available.

Kafka is one technology commonly used for this type of architecture.

Streaming introduces concepts such as:

- producers
- consumers
- topics
- partitions
- consumer groups
- offsets
- delivery semantics
- replay

We will introduce these concepts later in the book.

### Batch and Streaming Solve Different Requirements

| Batch | Streaming |
|---|---|
| Processes groups of data | Processes events continuously or incrementally |
| Often schedule-driven | Often event-driven |
| Naturally supports historical ranges | Often designed around continuously arriving events |
| Common for periodic workloads | Common when lower processing latency is required |

This does not mean that one approach is always better.

The requirements should decide the architecture.

Consider:

- latency
- data volume
- processing frequency
- complexity
- cost
- recovery requirements
- operational constraints

## Moving Data Is Not the Same as Producing Correct Data

A pipeline can process data successfully and still produce a bad result.

Imagine a pipeline reports:

~~~text
1,000,000 records processed
~~~

That sounds good.

But suppose:

- 50,000 records were invalid
- 20,000 records were duplicated
- 10,000 records were missing required fields

The pipeline processed data.

The result may still be unusable.

This is why **data quality** needs to be considered separately from pipeline execution.

Important data-quality areas include:

- completeness
- uniqueness
- validity
- referential integrity
- freshness
- anomaly detection

Later in the book, these become their own set of practical recipes.

## Observability

A pipeline that runs without visibility is difficult to operate.

Imagine someone tells you:

> The pipeline is not processing data.

What would you want to know?

You might need to know:

- when the pipeline last succeeded
- how many records were processed
- how many records failed
- how long processing took
- whether the source is available
- whether the destination is available
- whether errors are increasing

This is the role of **observability**.

A simple model is:

~~~text
              Pipeline
             /   |   \
            v    v    v
         Logs Metrics Traces
             \   |   /
              v  v  v
           Observability
~~~

Observability helps engineers understand what the system is doing.

Later we will look at logging, metrics, tracing, alerting, and monitoring separately.

## A Script Is Not Necessarily a Pipeline

Consider this code:

~~~python
data = fetch_data()
save_data(data)
~~~

It may be perfectly valid code.

But it does not answer many production questions.

What happens if fetch_data fails?

What happens if save_data fails?

What happens if the process stops halfway through?

What happens if the same data arrives twice?

What happens if the source changes its schema?

What happens if the destination becomes unavailable?

What happens if the transformation contains a bug?

What happens if yesterday's data needs to be processed again?

How do we know whether the pipeline is healthy?

These questions turn simple data-processing code into an engineering problem.

That is one of the main ideas of this book.

## How Pipeline Engineers Think

A useful production-oriented model is:

~~~text
Source
  |
  v
Ingestion
  |
  v
Validation
  |
  v
Processing
  |
  v
Storage
  |
  v
Verification
  |
  v
Observability
~~~

At every important step, ask three questions:

1. What can fail here?
2. How will we detect it?
3. How will we recover?

These questions are simple, but they are useful.

They can be applied to a small Python pipeline, a PostgreSQL-based system, a Kafka pipeline, an Airflow workflow, or a larger production platform.

## The Pipeline Engineering Cycle

Throughout this book, we will use the following engineering cycle:

~~~text
Understand
    |
    v
Investigate
    |
    v
Design
    |
    v
Implement
    |
    v
Test
    |
    v
Verify
    |
    v
Observe
    |
    v
Recover
    |
    v
Improve
~~~

### Understand

Understand the problem before changing the implementation.

Identify:

- the source
- the destination
- the data
- the requirements
- the expected behavior
- the failure conditions

### Investigate

Inspect the existing system.

Find:

- entry points
- configuration
- database code
- migrations
- tests
- infrastructure
- CI/CD

This becomes especially important when working in an existing repository.

### Design

Decide how the pipeline should behave.

Think about:

- data flow
- validation
- storage
- duplicates
- failures
- retries
- recovery
- observability

### Implement

Make the required changes.

Keep the implementation understandable and focused.

### Test

Test both the expected path and important failure paths.

A test that only checks success is not enough for many pipeline problems.

### Verify

Check the actual result.

Do not assume that a successful command means the system is correct.

Verification may involve checking:

- database state
- processed records
- logs
- metrics
- output files
- processing status

### Observe

Once the pipeline is running, observe it.

A pipeline can pass its tests and still behave differently in a real environment.

### Recover

Think about what happens after failure.

Can the pipeline retry?

Can it resume?

Can failed data be replayed?

Can historical data be processed again?

### Improve

Use what you learn from testing and operation to improve the system.

A production system teaches you things that are difficult to discover from code alone.

## Failure Is Part of the Architecture

One of the biggest differences between a simple data-processing script and a production-oriented pipeline is how failure is treated.

A simple mental model is:

~~~text
Input
  |
  v
Process
  |
  v
Output
~~~

A production-oriented engineer asks more questions:

- What if input acquisition fails?
- What if validation fails?
- What if processing fails?
- What if the database fails?
- What if the same event arrives twice?
- What if the process stops halfway through?
- What if the schema changes?
- What if old data needs to be processed again?

These questions should influence the architecture.

Failure handling should not always be something we add after the pipeline is finished.

For important failure scenarios, it should be part of the initial design.

## A Practical Example: Payment Events

Let's bring the ideas together.

Suppose an application produces a payment event:

~~~json
{
  "event_id": "evt_1001",
  "payment_id": "pay_1001",
  "amount": 250.00,
  "currency": "EUR"
}
~~~

A simplified pipeline could be:

~~~text
Payment Event
     |
     v
Ingestion
     |
     v
Validation
     |
     v
Staging
     |
     v
Processing
     |
     v
PostgreSQL
~~~

Now consider several situations.

### Scenario 1 — Valid Event

Everything succeeds.

~~~text
Event
  |
  v
Validation passes
  |
  v
Processing succeeds
  |
  v
Database write succeeds
  |
  v
Success
~~~

### Scenario 2 — Invalid Event

Validation fails.

~~~text
Event
  |
  v
Validation fails
  |
  v
Defined failure path
~~~

The event should follow whatever invalid-data path the system has defined.

### Scenario 3 — Database Failure

The event passes validation and processing, but the database operation fails.

~~~text
Event
  |
  v
Validation
  |
  v
Processing
  |
  v
Database
  |
  X
Failure
~~~

The pipeline needs a defined retry or recovery strategy.

### Scenario 4 — Duplicate Event

The same event arrives again.

~~~text
evt_1001
   |
   v
Already processed?
   |
   +---- Yes ----> Do not incorrectly process again
~~~

This is where idempotency becomes important.

### Scenario 5 — Reprocessing

Suppose the transformation logic contains a bug and is later corrected.

Previously stored input may need to be processed again:

~~~text
Stored Input
    |
    v
Corrected Processing
    |
    v
New Result
~~~

This is replay or reprocessing.

We will build these ideas properly in later chapters.

## Production Considerations

The architecture of a pipeline changes as its requirements change.

Before choosing a design, think about the following.

### Volume

How much data must be processed?

A pipeline processing 100 records per day can have very different requirements from one processing millions of records.

### Frequency

How often does processing need to happen?

These are different requirements:

- once per day
- every hour
- every minute
- continuously

### Latency

How quickly must new data become available?

A reporting pipeline may tolerate some delay.

A real-time operational system may have much stricter requirements.

### Reliability

What happens when a dependency fails?

How much data can be lost?

Can the pipeline recover?

### Correctness

What happens if data is processed twice?

What happens if data is missing?

What happens if data is invalid?

### Recovery

Can previously processed data be replayed?

Can a failed job resume?

Can historical data be backfilled?

These questions often influence architecture more than the choice of a particular technology.

## Common Mistakes

### Mistake 1 — Designing Only the Happy Path

A pipeline that works when everything succeeds is not necessarily production-ready.

Always ask:

> What happens when this step fails?

### Mistake 2 — Ignoring Duplicate Data

Events can sometimes be delivered more than once.

Duplicate handling should be considered during design.

### Mistake 3 — Treating Validation as Optional

Invalid data can create problems downstream.

Define what makes input valid and what happens when validation fails.

### Mistake 4 — Assuming Tests Are Enough

Tests are essential.

But tests are not the only verification mechanism.

Database state, logs, metrics, and actual pipeline behavior may also need to be inspected.

### Mistake 5 — Changing an Existing Repository Without Investigation

Before modifying an existing pipeline, understand its structure.

Find the relevant code first.

A useful starting point is to check the current repository state and branch, then inspect the project structure before making changes.

We will cover repository investigation in Part II.

### Mistake 6 — Adding Technology Before Understanding the Problem

A common mistake is to start with a technology:

> “I need Kafka.”

Or:

> “I need Airflow.”

A better starting point is the requirement.

Ask:

> What problem am I solving?

Then determine which architecture and technologies fit that problem.

## Practical Checklist

Before designing or modifying a pipeline, try to answer these questions.

### Source

- [ ] Where does the data originate?
- [ ] What format does the source provide?
- [ ] Is the source controlled by us or by another system?
- [ ] Can the source become unavailable?

### Processing

- [ ] What validation is required?
- [ ] What transformations are required?
- [ ] What business rules apply?
- [ ] Can processing fail?

### Storage

- [ ] Where is the data stored?
- [ ] Is raw data preserved?
- [ ] Is a staging layer required?
- [ ] What constraints exist in the destination?

### Duplicates

- [ ] Can the same input arrive more than once?
- [ ] Is there an identifier?
- [ ] Is idempotency required?
- [ ] How are duplicates detected?

### Failure Handling

- [ ] What happens when the source fails?
- [ ] What happens when the destination fails?
- [ ] What happens when processing fails?
- [ ] Should the operation be retried?
- [ ] Where are failures recorded?

### Recovery

- [ ] Can failed data be replayed?
- [ ] Can historical data be reprocessed?
- [ ] Can processing resume after interruption?
- [ ] Is checkpointing required?

### Observability

- [ ] Can we see whether the pipeline is running?
- [ ] Can we count processed records?
- [ ] Can we identify failures?
- [ ] Can we investigate slow processing?
- [ ] Can we verify the final state?

## What You Learned

A Data Engineering pipeline is not simply a mechanism for moving data.

It is a system that needs to define how data is:

1. received
2. validated
3. processed
4. stored
5. monitored
6. recovered

A simple pipeline might look like:

~~~text
Source
  |
  v
Ingestion
  |
  v
Processing
  |
  v
Destination
~~~

A production-oriented pipeline requires more thinking around:

- validation
- duplicates
- idempotency
- failures
- retries
- recovery
- replay
- data quality
- observability

The most important mental model from this chapter is:

~~~text
Understand
    |
    v
Investigate
    |
    v
Design
    |
    v
Implement
    |
    v
Test
    |
    v
Verify
    |
    v
Observe
    |
    v
Recover
    |
    v
Improve
~~~

You will see this cycle again and again throughout the book.

## Recipe Preview

The concepts from this chapter will become practical implementation recipes later in the book.

We will learn how to:

- create an ingestion pipeline
- create a staging layer
- validate incoming data
- add idempotency
- prevent duplicate processing
- track processing status
- handle failures
- implement retries
- replay previously stored data
- quarantine failed records
- backfill historical data
- process data incrementally
- use checkpoints for recovery

Before we implement those recipes, we need to understand the thing moving through many of these systems:

**the event.**

That is the subject of the next chapter.

## Chapter Summary

A pipeline moves data, but reliable pipeline engineering goes much further.

A good pipeline considers both the successful path and the failure path.

It asks:

1. Where does the data come from?
2. Where does it go?
3. What happens while it is being processed?
4. What can go wrong?
5. How do we detect failure?
6. How do we prevent incorrect duplicate processing?
7. How do we recover?
8. How do we verify the result?

These questions form the foundation for everything that follows.
