# Chapter 3 — Batch vs Streaming

A data pipeline usually processes data in one of two broad ways:

- **Batch processing**
- **Streaming processing**

The difference is mainly about **when data is processed and how continuously the pipeline works**.

Batch processing collects data and processes it as a group.

Streaming processing handles data continuously as it arrives.

Neither approach is automatically better. The correct choice depends on the problem, the required freshness, the amount of data, the failure model, and the operational complexity the team can support.

This chapter explains both approaches, shows where they fit, and explains how to choose between them.

---

## What Is Batch Processing?

Batch processing means collecting a group of records and processing them together.

For example, imagine a system that receives payment events throughout the day.

Instead of processing every event immediately, the system could collect the events and process them every hour.

The flow might look like this:

```text
Payment events
     |
     v
Raw storage
     |
     | every hour
     v
Batch job
     |
     v
Validation
     |
     v
Transformation
     |
     v
PostgreSQL
```

The important point is that the pipeline waits for a batch.

A batch could contain:

- 100 records
- 10,000 records
- 1 million records
- all records from one hour
- all records from one day
- all records from a specific historical period

The batch size depends on the system.

---

## A Simple Batch Example

Suppose an API provides transaction data.

The API may return 10,000 transactions.

A batch pipeline could:

1. Request the data.
2. Store the response.
3. Validate the records.
4. Transform the records.
5. Insert them into PostgreSQL.
6. Run data quality checks.
7. Mark the batch as completed.

The next run processes another batch.

A simple schedule could look like:

```text
00:00  -> Batch 1
01:00  -> Batch 2
02:00  -> Batch 3
03:00  -> Batch 4
...
```

The system does not need to process every record at the exact moment it arrives.

---

## Why Use Batch Processing?

Batch processing is useful when immediate processing is not required.

For example:

- Daily financial reporting
- Daily analytics loads
- Historical data processing
- Periodic API extraction
- Large file processing
- Data warehouse loading
- Scheduled data quality checks
- Backfills

Suppose a company needs a report every morning showing yesterday's transactions.

There may be little value in processing every transaction within milliseconds.

A nightly batch may be enough.

This can make the system simpler.

---

## Batch Processing Has a Natural Delay

One important characteristic of batch processing is **latency**.

Latency means the time between data becoming available and the data becoming available to the next system.

For example:

```text
10:00  Event arrives
10:05  Event is stored
11:00  Batch starts
11:05  Event is processed
```

The event existed at 10:00, but the downstream system did not receive the processed result until 11:05.

That delay may be completely acceptable.

The requirement determines whether it is acceptable.

---

# What Is Streaming Processing?

Streaming processing means processing data continuously as events become available.

Instead of waiting for a large batch, the system can process events one by one or in very small groups.

A simple streaming flow looks like this:

```text
Producer
   |
   v
Event Stream
   |
   v
Consumer
   |
   v
Validation
   |
   v
Processing
   |
   v
Database / Service
```

For example:

```text
10:00:01  Event A arrives -> processed
10:00:02  Event B arrives -> processed
10:00:04  Event C arrives -> processed
10:00:05  Event D arrives -> processed
```

The exact processing delay depends on the architecture.

Streaming does not always mean zero latency.

It means the system is designed to process data continuously rather than waiting for a large scheduled batch.

---

## A Simple Streaming Example

Imagine a payment platform that needs to detect suspicious transaction patterns quickly.

A transaction arrives.

The system may:

1. Receive the event.
2. Validate it.
3. Check required fields.
4. Send it to a processing service.
5. Evaluate rules.
6. Store the result.
7. Produce another event if required.

This can happen continuously.

The system does not need to wait until midnight to process the transaction.

---

# Batch vs Streaming

The simplest comparison is this:

| Batch | Streaming |
|---|---|
| Processes data in groups | Processes data continuously |
| Usually scheduled or triggered | Usually event-driven or continuously running |
| Higher latency is often acceptable | Lower latency is usually important |
| Easier to reason about in many cases | Can be more operationally complex |
| Good for historical processing | Good for continuously arriving events |
| Useful for reports and warehouse loads | Useful for real-time or near-real-time workflows |
| Often easier to replay as a complete batch | Replay requires careful event/offset handling |

This table is only a starting point.

Real systems can combine both approaches.

---

# Batch Windows

A batch pipeline often uses a **batch window**.

A batch window defines the period of data that should be processed.

For example:

```text
Batch 1 -> 00:00 to 01:00
Batch 2 -> 01:00 to 02:00
Batch 3 -> 02:00 to 03:00
```

The pipeline can use timestamps to decide which records belong to each batch.

For example:

```sql
WHERE occurred_at >= '10:00'
  AND occurred_at <  '11:00'
```

This is a generic example.

The exact query and timestamp column depend on the actual system.

---

## Why Batch Windows Matter

A clear batch boundary makes processing easier to reason about.

Suppose a job fails while processing the 10:00–11:00 window.

You can identify the affected data.

You may then rerun that window.

This is one reason batch processing works well for historical workloads.

However, the pipeline still needs to handle duplicates correctly.

If the first run inserted some records before failing, rerunning the same window must not create incorrect duplicate results.

That brings us back to idempotency.

---

# Streaming Has a Different Boundary

Streaming does not normally have a fixed start and end for every processing run.

Instead, the consumer keeps processing available events.

For example:

```text
Event 101
   |
   v
Consumer
   |
   v
Process

Event 102
   |
   v
Consumer
   |
   v
Process

Event 103
   |
   v
Consumer
   |
   v
Process
```

The consumer may continue running for hours, days, or longer.

This creates a different set of operational questions.

For example:

- Where did processing stop?
- Which events were successfully processed?
- Which event failed?
- Can the failed event be retried?
- Has the event already been stored?
- What happens after the consumer restarts?

Streaming systems therefore need strong handling for state, offsets, retries, and recovery.

---

# Event Time and Processing Time

Streaming systems often need to distinguish between two different times.

## Event Time

Event time is when the event actually happened.

For example:

```text
occurred_at = 10:00:00
```

## Processing Time

Processing time is when the pipeline processed the event.

For example:

```text
occurred_at  = 10:00:00
processed_at = 10:03:12
```

The difference matters.

An event may arrive late.

For example:

```text
10:00  Event happens
10:01  Network problem
10:05  Event arrives
10:05  Pipeline processes it
```

If the pipeline only looks at processing time, it may incorrectly treat the event as a 10:05 event.

For analytics and time-based processing, the system may need to preserve the original event time.

---

# Late Data

Late data is data that arrives after the time when the pipeline expected it.

This can happen because of:

- Network delays
- Producer failures
- Temporary service outages
- Mobile devices reconnecting later
- Retries
- Clock differences
- Slow upstream systems

Consider this sequence:

```text
Event A -> occurred at 10:00 -> arrives at 10:01
Event B -> occurred at 10:02 -> arrives at 10:02
Event C -> occurred at 10:01 -> arrives at 10:06
```

Event C arrived after Event B even though it happened earlier.

A pipeline that assumes arrival order always equals event order can produce incorrect results.

Late data becomes especially important when calculating:

- Time windows
- Counts
- Aggregations
- Metrics
- Reports
- Event sequences

---

# Ordering

Another important question is whether events must be processed in order.

Suppose these events describe an account:

```text
1. Account Created
2. Account Verified
3. Account Activated
```

If the pipeline processes them as:

```text
1. Account Created
3. Account Activated
2. Account Verified
```

the result may be incorrect.

Not every system requires strict ordering.

But when ordering matters, the pipeline needs an explicit strategy.

Possible approaches include:

- Partitioning
- Sequence numbers
- Ordering keys
- Timestamps
- State management
- Consumer coordination

The exact solution depends on the technology and business requirement.

---

# Micro-Batching

There is another approach between traditional batch and record-by-record streaming.

It is often called **micro-batching**.

Instead of processing one event at a time, the system collects a small group of events and processes them frequently.

For example:

```text
Events arrive continuously

A B C D E F G H I J
|     |     |
v     v     v
Batch Batch Batch
1     2     3
```

Each batch may contain only a small number of records or cover a short time interval.

Micro-batching can provide lower latency than large scheduled batches while keeping some of the operational simplicity of batch processing.

The exact behavior depends on the implementation.

---

# Batch and Streaming Can Exist Together

A common mistake is to think that a system must choose only one.

Real platforms often use both.

For example:

```text
                  +------------------+
                  | Event Producers  |
                  +--------+---------+
                           |
                           v
                     Event Stream
                           |
             +-------------+-------------+
             |                           |
             v                           v
      Streaming Consumer            Raw Storage
             |                           |
             v                           |
      Operational DB                     |
                                         v
                                  Batch Processing
                                         |
                                         v
                                   Data Warehouse
```

The streaming path can support operational use cases.

The batch path can support historical processing, analytics, backfills, or warehouse loading.

The same source data may therefore participate in more than one processing path.

---

# When Batch Processing Is a Good Fit

Batch is often a good fit when:

### 1. Freshness does not need to be immediate

If users only need updated results every hour or every day, streaming may add unnecessary complexity.

### 2. Data arrives in files

For example:

```text
CSV file
   |
   v
Batch ingestion
   |
   v
Validation
   |
   v
Database
```

### 3. Historical data needs processing

A large historical dataset is naturally suited to batch processing.

### 4. The workload is predictable

A scheduled job can be easier to operate when the workload has clear boundaries.

### 5. Simplicity matters

A simple scheduled pipeline can be easier to test, debug, and recover.

---

# When Streaming Is a Good Fit

Streaming is often a good fit when:

### 1. Data arrives continuously

For example, application events, device events, or transaction events.

### 2. Low latency matters

If downstream systems need new information quickly, waiting for a large batch may not work.

### 3. The system reacts to events

Examples include:

- Triggering workflows
- Updating operational state
- Detecting events
- Sending notifications
- Processing continuous telemetry

### 4. The workload does not have useful batch boundaries

Some event streams are continuous by nature.

---

# Do Not Choose Streaming Just Because It Sounds Faster

Streaming introduces additional engineering problems.

A streaming system may need to handle:

- Consumer restarts
- Offsets
- Duplicate delivery
- Ordering
- Late events
- Retries
- Backpressure
- Failed messages
- Replay
- Consumer lag
- State management
- Monitoring

This does not mean streaming is bad.

It means streaming should solve a real requirement.

If a daily report is enough, a daily batch may be easier to operate than a continuously running streaming system.

The design should follow the requirement.

---

# Backpressure

Streaming systems can receive data faster than they can process it.

Suppose events arrive at this rate:

```text
1,000 events/second
```

but the consumer can only process:

```text
700 events/second
```

The difference creates a growing backlog.

This is commonly called **backpressure** or consumer lag, depending on the architecture and technology.

Conceptually:

```text
Incoming rate:  1,000/sec
Processing rate: 700/sec

Backlog:
300/sec
600/sec
900/sec
...
```

If the situation continues, the backlog can become very large.

A production streaming system therefore needs to observe processing capacity and backlog.

Possible responses include:

- Scaling consumers
- Increasing processing capacity
- Reducing expensive work
- Controlling producers
- Buffering
- Temporarily accepting higher latency

The correct response depends on the architecture.

---

# Failure Handling in Batch Pipelines

Batch failures are often easier to see because the job has a clear execution boundary.

For example:

```text
Batch starts
    |
    v
Extract
    |
    v
Validate
    |
    X
Processing fails
```

The job can record that the batch failed.

But there are still important questions:

- Did extraction succeed?
- Did validation succeed?
- Were some records already written?
- Can the batch be safely rerun?
- Which records were processed?
- Which records failed?
- Did the failure happen before or after the database transaction?
- Can the same input be replayed?

A batch job is not reliable just because it runs on a schedule.

It still needs failure handling.

---

# Failure Handling in Streaming Pipelines

Streaming failures have a slightly different shape.

For example:

```text
Event 101 -> success
Event 102 -> success
Event 103 -> failure
Event 104 -> not processed yet
```

The consumer must decide what happens to Event 103.

Possible strategies include:

- Retry the event
- Move it to a failure destination
- Log the error and continue
- Stop processing
- Quarantine the event
- Retry later

The correct strategy depends on the failure.

A temporary database outage should not necessarily be treated the same way as invalid input.

---

# Temporary Failure vs Permanent Failure

This distinction is important in both batch and streaming systems.

## Temporary failure

A temporary failure may succeed later.

Examples:

- Database unavailable
- Network timeout
- Temporary API failure
- Service restart
- Connection pool exhaustion

Retrying may be useful.

## Permanent failure

A permanent failure is unlikely to succeed without changing the input or code.

Examples:

- Invalid required field
- Unsupported schema
- Malformed data
- Invalid business rule
- Corrupt payload

Repeatedly retrying the same bad record may waste resources.

This topic becomes important later when the book covers retries, error handling, and quarantine.

---

# Batch Replay

Batch systems often have a useful property: a batch can be identified by its input range.

For example:

```text
Batch: 2026-09-26 10:00 -> 11:00
```

If processing fails, the same range can potentially be processed again.

But replay is only safe when the pipeline handles duplicates correctly.

Suppose the first attempt inserted:

```text
Record A
Record B
Record C
```

and then failed.

The second attempt reads:

```text
Record A
Record B
Record C
Record D
```

If the pipeline simply inserts everything again, A, B, and C may be duplicated.

Therefore:

**Batch replay still requires idempotency.**

---

# Streaming Replay

Streaming systems can also replay events.

A simplified flow is:

```text
Event Stream
     |
     v
Consumer
     |
     X
Failure
     |
     v
Restart
     |
     v
Replay previous events
```

Replay can be useful for:

- Recovering from failures
- Fixing processing bugs
- Rebuilding derived data
- Testing new processing logic
- Recovering from downstream outages

But replay creates the same basic problem:

**Can the consumer safely process an event more than once?**

This is why event IDs, offsets, idempotency, and correct storage design matter.

---

# Checkpointing

Long-running processing often needs a way to remember progress.

A checkpoint represents a known processing position.

For example:

```text
Event 100
Event 101
Event 102
Event 103
Event 104
       ^
       |
   checkpoint
```

If the consumer crashes after processing Event 104, it needs some way to know where to continue.

The exact checkpoint mechanism depends on the technology.

In some systems it may be an offset.

In others it may be:

- A database record
- A timestamp
- A sequence number
- A file position
- A workflow state

Checkpointing becomes more important as pipelines become longer-running and more stateful.

---

# A Practical Example

Consider a payment telemetry system.

Events may look conceptually like:

```text
page_view
login
onboarding_started
api_request_failed
onboarding_completed
```

Suppose the business wants dashboards showing events from the previous day.

A batch design could be:

```text
Frontend / services
        |
        v
Raw event storage
        |
        | nightly
        v
Batch processing
        |
        v
PostgreSQL
        |
        v
Dashboard
```

If the business later requires near-real-time operational monitoring, a streaming path could be introduced:

```text
Frontend / services
        |
        v
Event stream
        |
        v
Streaming consumer
        |
        v
Operational storage
        |
        v
Monitoring
```

The important part is not choosing a technology first.

The important part is identifying the requirement.

---

# How to Decide

When designing a pipeline, ask these questions.

## Question 1: How fresh must the data be?

Possible requirements:

- Once per day
- Every few hours
- Every 15 minutes
- Every minute
- Within seconds

The required freshness strongly affects the architecture.

---

## Question 2: How does the data arrive?

Does it arrive as:

- Files?
- API responses?
- Database records?
- Messages?
- Events?
- Continuous telemetry?

The source often suggests a natural processing model.

---

## Question 3: How much data is involved?

A small periodic dataset may be easy to process in a batch.

A continuously growing event stream may require a different design.

But volume alone should not determine the architecture.

The business requirement matters too.

---

## Question 4: What happens when processing fails?

Ask:

- Can the input be retried?
- Can it be replayed?
- Can partial progress be detected?
- Can the same input be processed again safely?
- Where are failed records stored?
- How does the operator recover the pipeline?

These questions should be answered before calling the pipeline production-ready.

---

## Question 5: Does ordering matter?

If event order affects correctness, the design must explicitly account for it.

Do not assume arrival order is always business order.

---

## Question 6: Is the team ready to operate it?

A streaming system may require more operational knowledge than a simple batch job.

Consider:

- Monitoring
- Alerts
- Scaling
- Consumer health
- Lag
- Replay
- Recovery
- Deployment
- Schema changes

The architecture should match the team's ability to operate it.

---

# A Simple Decision Guide

This is a practical starting point, not a strict rule.

```text
Does the system need continuously available results?
             |
          No | Yes
             |  \
             v   v
          Batch  Streaming

Does it need low latency?
             |
          No | Yes
             |  \
             v   v
          Batch  Streaming

Does data arrive naturally as files or scheduled API results?
             |
             v
           Batch

Does data arrive continuously as events?
             |
             v
         Consider Streaming
```

Sometimes the answer is both.

For example, streaming may handle operational events while batch processing handles historical warehouse workloads.

---

# Common Mistakes

## Mistake 1: Treating streaming as automatically better

Streaming can reduce latency, but it also introduces operational complexity.

Use it when the requirement needs it.

---

## Mistake 2: Ignoring late data

An event can happen at one time and arrive much later.

If time-based processing matters, design for late events.

---

## Mistake 3: Assuming events always arrive in order

Network and system behavior can change arrival order.

If ordering matters, make it explicit.

---

## Mistake 4: Forgetting replay

Failures happen.

A good pipeline should have a clear recovery story.

Ask:

**How would I process this data again?**

---

## Mistake 5: Building streaming without monitoring

A consumer can continue running while falling behind.

A green process does not always mean healthy processing.

Monitor the actual work being done.

---

## Mistake 6: Ignoring duplicates

Retries and replay can cause the same data to be processed more than once.

Idempotency must be part of the design.

---

## Mistake 7: Choosing technology before understanding the requirement

Starting with:

> "We need Kafka."

is not a pipeline design.

Start with:

> "What does the system need to do, and how fresh must the result be?"

Then choose the appropriate architecture.

---

# Production Considerations

A production pipeline should make its processing model clear.

For batch pipelines, document:

- Schedule
- Batch boundaries
- Input selection
- Expected runtime
- Failure handling
- Retry behavior
- Replay procedure
- Backfill procedure
- Data quality checks
- Monitoring
- Alerts

For streaming pipelines, document:

- Event source
- Consumer behavior
- Processing guarantees
- Ordering requirements
- Offset/checkpoint behavior
- Retry behavior
- Failed-event handling
- Replay procedure
- Consumer scaling
- Lag monitoring
- Schema handling
- Deployment and recovery

The exact list can vary by system.

The important point is that operational behavior should not be left as tribal knowledge.

---

# Practical Checklist

Before choosing batch or streaming, ask:

- [ ] How fresh does the data need to be?
- [ ] How does the data arrive?
- [ ] Is the workload continuous?
- [ ] Is ordering important?
- [ ] Can data arrive late?
- [ ] Can records be duplicated?
- [ ] Can processing fail halfway through?
- [ ] Can the input be replayed?
- [ ] How is progress tracked?
- [ ] How are temporary failures handled?
- [ ] How are permanent failures handled?
- [ ] What happens when the consumer or batch job restarts?
- [ ] What monitoring is required?
- [ ] Can the team operate the chosen architecture?
- [ ] Is a hybrid design more appropriate?

If these questions do not have clear answers, the design probably needs more investigation.

---

# What You Learned

In this recipe, you learned:

- Batch processing handles data in groups.
- Streaming processing handles data continuously.
- Batch processing is useful when immediate results are not required.
- Streaming is useful when continuously arriving data needs low-latency processing.
- Batch pipelines often have clear processing windows.
- Streaming pipelines need to deal with ordering, late data, offsets, retries, and recovery.
- Batch and streaming can exist together in the same system.
- Replay and idempotency matter in both models.
- Technology should follow the requirement rather than the other way around.

The main lesson is simple:

**Choose the processing model based on the data and the requirement, not because one architecture sounds more advanced.**

---
