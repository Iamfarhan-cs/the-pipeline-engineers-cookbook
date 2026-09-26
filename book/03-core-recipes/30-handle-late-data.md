# Recipe 30 — Handle Late Data

Data does not always arrive in the order in which it happened.

A payment can happen at 10:00 and arrive at 10:07.

A mobile device can go offline and upload events hours later.

A source system can retry yesterday's records today.

A batch file can arrive after the warehouse job has already run.

A streaming consumer can receive an older event after newer events have already been processed.

This is **late data**.

A production pipeline must decide what to do when data arrives after the time window in which it was expected.

This chapter turns late-data handling into a practical pipeline recipe.

---

## 1. Goal

The goal is to design a pipeline that can recognize, process, observe, test, and recover from data that arrives later than expected.

By the end of this recipe, you should be able to:

- identify late data in a real pipeline
- distinguish event time from processing time
- measure data lateness
- define an acceptable lateness policy
- choose between immediate processing, correction, reprocessing, or rejection
- implement late-data handling in batch pipelines
- implement late-data handling in streaming pipelines
- use watermarks and allowed-lateness concepts
- update previously calculated aggregates
- preserve auditability
- test out-of-order events
- intentionally inject late records
- observe late-data rates
- recover from a bad lateness policy
- explain the trade-offs of each approach

The central idea is:

    Event happened
          |
          v
    Event timestamp
          |
          v
    Data arrives
          |
          v
    Is it late?
       /       \
     No         Yes
     |           |
     v           v
  Process     Apply late-data policy
                 |
                 v
        Correct / update / replay / quarantine

---

## 2. Problem

Consider a payment event:

~~~json
{
  "event_id": "evt-1001",
  "occurred_at": "2026-09-26T10:00:00Z",
  "amount": 250,
  "currency": "EUR"
}
~~~

The event does not reach the pipeline until:

    2026-09-26T10:08:00Z

The event happened at 10:00.

The pipeline received it at 10:08.

Those are different times.

If the pipeline groups events into one-minute windows, the event belongs to:

    10:00–10:01

even though it arrived during:

    10:08–10:09

If the pipeline only considers arrival time, it may put the payment into the wrong reporting window.

This can produce incorrect:

- transaction counts
- revenue totals
- payment metrics
- fraud metrics
- SLA calculations
- operational dashboards
- customer reports

Late data is therefore not simply a performance problem.

It is a **data correctness problem**.

---

## 3. Event Time vs Processing Time

This distinction is foundational.

### Event time

Event time is when the business event actually happened.

Example:

    occurred_at = 10:00

### Processing time

Processing time is when the pipeline processes the event.

Example:

    processed_at = 10:08

### Ingestion time

Ingestion time is when the pipeline receives or stores the event.

Example:

    ingested_at = 10:08

A useful model is:

    Event happens
         |
         | occurred_at
         v
    Source
         |
         | network / queue / batch delay
         v
    Ingestion
         |
         | ingested_at
         v
    Processing
         |
         | processed_at
         v
    Output

Do not assume these timestamps are interchangeable.

---

## 4. Calculate Lateness

A simple definition is:

    lateness = arrival_time - event_time

For example:

    event_time   = 10:00
    arrival_time = 10:08

Therefore:

    lateness = 8 minutes

In a real system, the exact calculation depends on which timestamps represent the business semantics.

For example:

    lateness = ingested_at - occurred_at

may be appropriate when measuring source-to-pipeline delay.

But:

    lateness = processed_at - occurred_at

measures end-to-end processing delay.

Choose the definition deliberately.

---

## 5. Why Late Data Happens

Late data can have many causes.

Common causes include:

- network delays
- offline mobile devices
- source-system outages
- message broker delays
- consumer downtime
- retry queues
- overloaded workers
- batch files arriving late
- upstream scheduling problems
- clock differences
- manual corrections
- replay
- backfill
- source-system buffering

Do not automatically treat every late event as a pipeline failure.

Some late data is normal.

The important question is:

> Is the amount and distribution of lateness within the system's expected operating range?

---

## 6. Recognize the Problem in a Real Pipeline

Look for symptoms such as:

- yesterday's totals changing today
- dashboards changing after a reporting window closed
- events appearing in the wrong time bucket
- aggregates that are temporarily incorrect
- source records arriving with old timestamps
- growing "late events" counts
- reconciliation differences between source and warehouse
- records being rejected because their timestamp is old
- repeated backfills
- analysts manually correcting historical reports

A particularly strong signal is:

    The pipeline processed newer events,
    then an older event arrived,
    and historical output changed.

That is a late-data problem.

---

## 7. Repository Investigation

Before implementing a solution, trace the data path.

Start with:

    Source
      |
      v
    Acquisition
      |
      v
    Raw storage
      |
      v
    Parsing
      |
      v
    Staging
      |
      v
    Transformation
      |
      v
    Aggregation
      |
      v
    Warehouse / reporting

Find every timestamp involved.

Search for fields such as:

    occurred_at
    event_time
    event_timestamp
    created_at
    source_timestamp
    received_at
    ingested_at
    processed_at
    updated_at

Then answer:

1. Which timestamp represents when the event actually happened?
2. Which timestamp represents when the pipeline received it?
3. Which timestamp is used for partitioning?
4. Which timestamp is used for windowing?
5. Which timestamp is used for reporting?
6. Can events arrive out of order?
7. Are historical partitions mutable?
8. How long can data reasonably be delayed?
9. What happens when a record arrives after the reporting window?
10. Can the pipeline recompute affected results?

Do not change the pipeline until these questions are understood.

---

## 8. Establish the Data Contract

A late-data strategy starts with a timestamp contract.

For an event, define fields such as:

~~~json
{
  "event_id": "evt-1001",
  "occurred_at": "2026-09-26T10:00:00Z",
  "ingested_at": "2026-09-26T10:08:00Z"
}
~~~

The contract should make clear:

- which timestamp is authoritative
- what timezone is used
- whether timestamps are UTC
- expected precision
- whether timestamps can be missing
- whether timestamps can be corrected
- acceptable clock skew
- expected maximum delay

A timestamp without defined semantics is not enough for reliable late-data handling.

---

## 9. Define the Lateness Policy

Before writing code, decide what the pipeline should do.

A useful policy might be:

    0–5 minutes
        -> normal processing

    5–60 minutes
        -> process normally but mark as late

    1–24 hours
        -> process and correct affected aggregates

    >24 hours
        -> route to controlled historical-reprocessing path

These values are only an example.

Do not copy them blindly.

The correct threshold depends on the source, business requirement, SLA, and reporting behavior.

---

## 10. Common Late-Data Strategies

There is no single correct strategy.

Common approaches are:

### Strategy A — Process immediately

Late records are accepted and processed as soon as they arrive.

Useful when:

- historical correction is easy
- downstream systems tolerate updates
- correctness matters more than fixed reporting windows

### Strategy B — Process and update the affected window

The event is assigned to its event-time window and the previous result is corrected.

Useful for:

- time-windowed aggregates
- analytics
- reporting

### Strategy C — Allow a bounded lateness period

The pipeline keeps a window open for a defined period.

Example:

    Window: 10:00–10:05
    Allowed lateness: 10 minutes

The result is considered final only after the lateness period expires.

### Strategy D — Quarantine very late records

Records beyond the acceptable historical boundary are stored separately for controlled processing.

Useful when:

- automatic historical mutation is risky
- business approval is required
- very old records are unusual

### Strategy E — Reprocess affected partitions

A late record causes the relevant historical partition or date range to be recomputed.

Useful for batch data warehouses.

---

## 11. Bounded Lateness

A common streaming concept is **allowed lateness**.

Suppose:

    Window:
    10:00–10:05

and:

    Allowed lateness:
    10 minutes

The pipeline does not immediately assume that 10:00–10:05 is final.

It waits long enough to accept reasonably late events.

Conceptually:

    10:00
      |
      +------------------+
      | event-time window|
      +------------------+
                         |
                       10:05
                         |
                         | allowed lateness
                         v
                       10:15
                         |
                         v
                    finalization

An event arriving at 10:08 may still update the 10:00–10:05 window.

An event arriving at 10:20 may be handled by a different late-data path.

The actual implementation depends on the processing engine.

---

## 12. Watermarks

A **watermark** is a progress signal used by event-time processing systems.

Conceptually:

    "The system believes events older than this point
     should no longer normally arrive."

For example:

    watermark = 10:15

The system can use this to reason about event-time windows.

A watermark is not proof that no older event will ever arrive.

It is an operational assumption about event-time progress.

This distinction matters.

A late event can still arrive after the watermark.

The pipeline must have a policy for that case.

---

## 13. Watermark vs Allowed Lateness

These concepts are related but different.

### Watermark

Represents event-time progress.

Example:

    watermark = 10:15

### Allowed lateness

Defines how long a completed or closing window can continue accepting late data.

Example:

    allowed lateness = 10 minutes

A simplified model is:

    Event time
       |
       v
    Window
       |
       v
    Watermark advances
       |
       v
    Late-data period
       |
       v
    Window finalized

Do not confuse the two.

---

## 14. Batch Pipelines Have the Same Problem

Late data is not only a streaming problem.

Consider a daily batch:

    01:00
      |
      v
    Process 2026-09-25
      |
      v
    Warehouse partition finalized

At 09:00, a source file containing September 25 records arrives.

Those records are late.

The batch pipeline needs to decide:

    ignore
    append
    merge
    replace partition
    rerun transformation
    quarantine

A robust batch pipeline treats late arrivals as a normal operational case rather than assuming that every partition is permanently immutable.

---

## 15. Partition-Based Correction

Suppose a warehouse is partitioned by event date:

    events/
      date=2026-09-24/
      date=2026-09-25/
      date=2026-09-26/

A late event arrives:

    occurred_at = 2026-09-25
    arrived_at  = 2026-09-26

The event belongs to:

    date=2026-09-25

A simple correction strategy is:

    1. Store the late event
    2. Identify affected partition
    3. Recompute that partition
    4. Replace or merge the affected output
    5. Validate the result
    6. Record the correction

This is often easier to reason about than trying to patch every aggregate individually.

---

## 16. Incremental Aggregation Problem

Suppose the pipeline calculates:

    daily_payment_count
    daily_payment_amount

At 10:00:

    100 payments
    total = EUR 25,000

At 11:00, a late payment arrives:

    amount = EUR 500
    occurred_at = 09:00

The correct 09:00 day's total may now be:

    count = 101
    total = EUR 25,500

If the aggregate was treated as permanently final, it becomes wrong.

This is the central late-data problem for incremental analytics:

> A new event can change a result that was already calculated.

---

## 17. Correcting Aggregates

There are several approaches.

### Approach 1 — Increment the aggregate

If the late event can safely be added:

    count = count + 1
    amount = amount + 500

This is simple for additive metrics.

### Approach 2 — Recompute the affected window

Read all events for the affected window and calculate the aggregate again.

This is safer for complex metrics.

### Approach 3 — Maintain corrections separately

Store:

    original aggregate
    correction events

and derive the current result.

This can provide strong auditability.

### Approach 4 — Rebuild a partition

For batch warehouses, recompute the affected date or partition.

Choose based on data volume, metric complexity, performance, and correctness requirements.

---

## 18. Additive vs Non-Additive Metrics

Late-data correction is easier for some metrics than others.

### Additive

Examples:

    count
    sum

A late record can often be incorporated directly.

### Semi-additive

Examples:

    account balance across time dimensions

Correction depends on the dimension and business semantics.

### Non-additive

Examples:

    median
    percentile
    distinct count in some implementations

A simple increment may not be possible.

The pipeline may need to recompute the affected window.

Do not assume every metric can be corrected with:

    aggregate + late_value

---

## 19. Late Data and Slowly Changing State

Consider a state transition:

    payment_created
    payment_authorized
    payment_completed

Suppose the completion event arrives before the authorization event.

The pipeline may temporarily see:

    completed

and later receive:

    authorized

This is both a late-data and ordering problem.

The pipeline needs a state model that can handle events arriving out of order.

Possible techniques include:

- event-time ordering
- version numbers
- sequence numbers
- state transitions with validation
- replay of the affected entity
- event-sourced reconstruction

The correct approach depends on the source contract.

---

## 20. Late Data and Deduplication

Late records are not automatically duplicates.

For example:

    event A
    occurred_at = 10:00
    arrived_at  = 10:05

and:

    event B
    occurred_at = 10:00
    arrived_at  = 10:06

These may be two legitimate events.

Do not deduplicate solely because timestamps are equal.

Use the event's logical identity and business rules.

Late-data handling answers:

    "When did this data arrive relative to when it happened?"

Deduplication answers:

    "Are these records representing the same logical event?"

Both concerns must be handled independently.

---

## 21. Store Both Event Time and Ingestion Time

A robust event model commonly keeps both timestamps.

For example:

~~~sql
event_id
occurred_at
ingested_at
processed_at
~~~

This makes it possible to answer:

- When did the event happen?
- When did we receive it?
- When did we process it?
- How late was it?
- How long did processing take?

Without these timestamps, diagnosing late-data behavior becomes much harder.

---

## 22. Add a Lateness Measurement

A useful derived field is:

    lateness_seconds

For example:

    lateness_seconds =
        ingested_at - occurred_at

Store the derived value only if it is useful for the system; otherwise calculate it in queries or metrics.

The important point is that the pipeline should be able to measure lateness.

Useful metrics include:

    late_events_total
    late_events_ratio
    lateness_seconds
    lateness_p50
    lateness_p95
    lateness_p99
    events_beyond_allowed_lateness_total

Percentiles are often more useful than averages because a small number of extremely late records can distort the average.

---

## 23. Observe the Lateness Distribution

Suppose the pipeline receives:

    95% within 1 minute
     4% within 10 minutes
     1% within 6 hours

An average lateness number may hide the operational problem.

Instead, inspect the distribution.

Useful questions:

- What percentage is late?
- What is p50 lateness?
- What is p95?
- What is p99?
- How many records exceed the allowed lateness?
- Which source produces the most late data?
- Which event type is most affected?
- Is lateness increasing over time?

This helps determine whether the current lateness policy is appropriate.

---

## 24. Source-Specific Lateness

Different sources can have different delivery behavior.

For example:

    Web API
        -> usually near real time

    Mobile application
        -> may be delayed when offline

    Bank file
        -> scheduled batch delivery

    External partner
        -> unpredictable delivery delay

Do not necessarily use one lateness threshold for every source.

A source-aware policy may be more accurate.

For example:

    source = mobile
        allowed lateness = larger

    source = API
        allowed lateness = smaller

The thresholds must come from actual system requirements and observed behavior.

---

## 25. A Generic SQL Pattern

Suppose events contain:

~~~sql
event_id
occurred_at
ingested_at
payload
~~~

A simple lateness query could be:

~~~sql
SELECT
    event_id,
    occurred_at,
    ingested_at,
    EXTRACT(EPOCH FROM (ingested_at - occurred_at))
        AS lateness_seconds
FROM events;
~~~

This is a generic PostgreSQL example.

For a threshold:

~~~sql
SELECT *
FROM events
WHERE ingested_at - occurred_at > INTERVAL '10 minutes';
~~~

This identifies events later than ten minutes.

The threshold is only an example. Use the repository's actual policy.

---

## 26. Generic Batch Recovery Pattern

A practical batch pattern is:

    1. Ingest all records
            |
            v
    2. Record event and ingestion timestamps
            |
            v
    3. Identify late records
            |
            v
    4. Determine affected partitions/windows
            |
            v
    5. Recompute affected outputs
            |
            v
    6. Validate corrected results
            |
            v
    7. Publish corrected output
            |
            v
    8. Record the correction

This creates a controlled correction path instead of silently modifying historical data.

---

## 27. Generic Streaming Recovery Pattern

A simplified event-time streaming pattern is:

    Event
      |
      v
    Validate
      |
      v
    Determine event-time window
      |
      v
    Compare with watermark
      |
      +------------------+
      |                  |
   within policy      too late
      |                  |
      v                  v
    update             late-data path
    window             / quarantine /
                       correction /
                       replay
                              |
                              v
                           observe

The actual implementation depends on the streaming framework.

The concepts remain the same.

---

## 28. Testing Late Data

Late-data behavior must be tested deliberately.

At minimum, test:

### Test 1 — On-time event

    occurred_at = 10:00
    arrived_at  = 10:00

Expected:

    normal processing

### Test 2 — Slightly late event

    occurred_at = 10:00
    arrived_at  = 10:02

Expected:

    processed according to normal late-data policy

### Test 3 — Boundary event

Test exactly at the configured threshold.

For example:

    allowed lateness = 10 minutes

Test:

    10:10

Do not assume the boundary behavior.

Define it explicitly.

### Test 4 — Very late event

    occurred_at = 10:00
    arrived_at  = next day

Expected:

    correction, replay, quarantine, or another explicitly defined policy

### Test 5 — Out-of-order events

Send:

    event A -> 10:05
    event B -> 10:01
    event C -> 10:03

Expected:

    correct event-time result

### Test 6 — Late event changes an aggregate

Calculate an aggregate, then inject an older event.

Verify that the final aggregate is corrected.

### Test 7 — Duplicate late event

Send the same late event twice.

Verify both:

    late-data handling
    +
    idempotency

work correctly.

---

## 29. Intentionally Break the Pipeline

A production-grade recipe should include failure drills.

### Drill 1 — Delay a source

Make test data arrive several minutes late.

Verify:

- lateness is detected
- event is not lost
- affected output is corrected

### Drill 2 — Send events out of order

Send:

    10:05
    10:01
    10:03

Verify the event-time result.

### Drill 3 — Send data after the allowed-lateness boundary

Verify:

- the record is classified correctly
- the normal path does not silently discard it
- the late-data path handles it

### Drill 4 — Send a duplicate late event

Verify:

- no duplicate business result
- late-data metrics remain correct

### Drill 5 — Fail during correction

Force a failure while recomputing an affected window.

Verify:

- partial output is not published as final
- the correction can be retried
- the final state is correct

---

## 30. Recovery

When late data exposes an incorrect historical result, use a controlled recovery sequence.

    1. Identify affected event(s)
            |
            v
    2. Identify affected window/partition
            |
            v
    3. Determine current incorrect output
            |
            v
    4. Recompute or apply correction
            |
            v
    5. Validate against source data
            |
            v
    6. Publish corrected result
            |
            v
    7. Record what was corrected
            |
            v
    8. Verify downstream consumers

Do not simply rerun an entire pipeline unless that is the intended recovery mechanism.

A targeted replay is often easier to control.

---

## 31. Quarantine Very Late Data

For records that are too late for automatic correction, a quarantine path can be useful.

For example:

    normal event
         |
         v
    normal processing

    late event
         |
         v
    late threshold check
         |
         v
    too late
         |
         v
    quarantine
         |
         v
    controlled review / replay

A quarantine record should preserve enough information to recover the event later.

Useful metadata can include:

    event_id
    source
    occurred_at
    ingested_at
    lateness
    reason
    detected_at
    processing_attempt
    replay_status

Apply the project's privacy rules to stored payloads.

---

## 32. Do Not Silently Drop Late Data

This is one of the most dangerous mistakes.

Bad:

    event is too old
          |
          v
        discard

If the record affects financial, operational, compliance, or analytical correctness, silently dropping it can create permanent data loss.

If the business explicitly allows dropping a class of late records, that rule should be documented, measurable, and auditable.

Otherwise use a correction or quarantine path.

---

## 33. Historical Mutability

A late-data policy requires an answer to:

> Can historical data change?

Some systems intentionally treat historical partitions as mutable.

Others require finalized periods to remain immutable.

Neither is universally correct.

If historical data is mutable:

    late event
       |
       v
    correction
       |
       v
    historical output changes

If historical data is immutable:

    late event
       |
       v
    separate correction path
       |
       v
    adjustment / exception dataset

The business and reporting contract must define which model is valid.

---

## 34. Auditability

When historical data changes because of late data, the change should be explainable.

Useful audit information includes:

    original result
    correction reason
    affected window
    event IDs
    correction timestamp
    pipeline version
    operator or job identity
    replay identifier

A useful audit chain is:

    Late event
        |
        v
    Detection
        |
        v
    Correction
        |
        v
    New result
        |
        v
    Validation

This is especially important for financial and regulated data.

---

## 35. Late Data and Versioning

Sometimes the best approach is to version outputs.

For example:

    aggregate version 1
          |
          v
    late event arrives
          |
          v
    aggregate version 2

Instead of silently overwriting the old result, the system can preserve the versions.

This can help with:

- auditability
- debugging
- reproducibility
- historical analysis

But versioning adds storage and query complexity.

Use it when the business requires historical reconstruction.

---

## 36. Late Data and Data Contracts

A source contract should define expected delivery behavior.

For example:

    Event timestamp:
        UTC

    Expected delivery:
        near real time

    Normal lateness:
        defined by source SLA

    Maximum automatic correction:
        defined by pipeline policy

    Historical replay:
        supported

These are examples of contract concepts, not universal values.

If a source consistently violates the expected delivery contract, the pipeline should make that visible rather than silently absorbing the problem.

---

## 37. Common Mistakes

### Mistake 1 — Using ingestion time as event time

This places delayed events into the wrong business window.

### Mistake 2 — Treating every late event as an error

Some late data is expected.

The pipeline should distinguish normal lateness from exceptional lateness.

### Mistake 3 — Dropping old records

This can create silent data loss.

### Mistake 4 — Assuming a watermark means no older events can arrive

A watermark represents event-time progress, not a physical guarantee.

### Mistake 5 — Applying one threshold to every source

Different sources can have different delivery characteristics.

### Mistake 6 — Correcting additive metrics only

Some metrics require full recomputation.

### Mistake 7 — Ignoring duplicates

A late event can also be delivered more than once.

Late-data handling must work with idempotency.

### Mistake 8 — Not testing boundary conditions

The exact lateness threshold needs explicit tests.

### Mistake 9 — Updating historical data without audit information

Operators should be able to explain why a historical result changed.

### Mistake 10 — Making historical data immutable without a correction path

If late records can legitimately arrive, an immutable model still needs a defined correction mechanism.

---

## 38. Production Checklist

Before calling the implementation production-ready, verify:

### Time semantics

- [ ] Event time is clearly defined.
- [ ] Ingestion time is captured.
- [ ] Processing time is available where useful.
- [ ] Timezone and timestamp precision are defined.
- [ ] Clock-skew behavior is understood.

### Lateness policy

- [ ] Normal lateness is defined.
- [ ] Allowed lateness is defined where applicable.
- [ ] Very late data has an explicit path.
- [ ] Boundary behavior is tested.
- [ ] Source-specific differences are considered.

### Processing

- [ ] Event-time windows use the correct timestamp.
- [ ] Late events can correct affected results.
- [ ] Historical mutation policy is explicit.
- [ ] Batch and streaming behavior are defined.
- [ ] Replay behavior is documented.

### Reliability

- [ ] Late data works with idempotency.
- [ ] Duplicate late events are safe.
- [ ] Correction operations are retryable.
- [ ] Partial correction failures are recoverable.

### Observability

- [ ] Late event count is measured.
- [ ] Lateness distribution is measurable.
- [ ] Very late events are visible.
- [ ] Source-level lateness can be identified.
- [ ] Corrections are auditable.

### Recovery

- [ ] Affected windows can be identified.
- [ ] Historical outputs can be recomputed or corrected.
- [ ] Corrections are validated.
- [ ] Downstream consumers are considered.
- [ ] Operators have a documented recovery procedure.

---

## 39. Practical Implementation Sequence

For an existing repository, implement late-data handling in this order:

    1. Trace the current pipeline
            |
            v
    2. Identify event time
            |
            v
    3. Identify ingestion time
            |
            v
    4. Measure current lateness
            |
            v
    5. Inspect current window/partition logic
            |
            v
    6. Define normal and exceptional lateness
            |
            v
    7. Choose the late-data strategy
            |
            v
    8. Implement event-time processing
            |
            v
    9. Implement correction/reprocessing path
            |
            v
    10. Add quarantine if required
            |
            v
    11. Ensure idempotency
            |
            v
    12. Add unit tests
            |
            v
    13. Add integration tests
            |
            v
    14. Test out-of-order events
            |
            v
    15. Test lateness boundaries
            |
            v
    16. Break the pipeline intentionally
            |
            v
    17. Recover affected data
            |
            v
    18. Add metrics and audit logging
            |
            v
    19. Validate final historical state
            |
            v
    20. Document the policy

The key is to define the semantics before changing the implementation.

---

## 40. Definition of Done

The late-data implementation is complete when:

- [ ] Event time and ingestion time are clearly distinguished.
- [ ] Lateness can be measured.
- [ ] The expected lateness distribution is understood.
- [ ] A lateness policy is explicitly defined.
- [ ] Normal late data is handled without data loss.
- [ ] Very late data has a controlled path.
- [ ] Event-time windows use the correct timestamp.
- [ ] Historical corrections are possible where required.
- [ ] Replay behavior is defined.
- [ ] Duplicate late events are safe.
- [ ] Boundary conditions are tested.
- [ ] Out-of-order events are tested.
- [ ] Failure during correction is tested.
- [ ] Late-data metrics are available.
- [ ] Corrections are auditable.
- [ ] The pipeline can be intentionally broken and recovered.
- [ ] The final data state can be independently validated.
- [ ] The policy is documented for operators and developers.

---

## 41. What You Learned

Late data is not simply "old data."

It is data whose **event time** is earlier than the point at which the pipeline expected to receive or finalize it.

The essential model is:

    Event time
         |
         v
    Data arrives
         |
         v
    Measure lateness
         |
         v
    Apply policy
       /   |    \
     accept correct quarantine
       \   |    /
          v
       final data

A strong late-data design usually combines:

    Correct event-time semantics
            +
    Ingestion-time tracking
            +
    Explicit lateness policy
            +
    Event-time windows/partitions
            +
    Correction or replay capability
            +
    Idempotency
            +
    Observability
            +
    Recovery testing

The most important lesson is:

> A pipeline is not correct merely because it processes data quickly. It must also remain correct when data arrives late.

Late data naturally leads to another reliability problem.

Even when events arrive late, they may also arrive **out of order**.

That requires careful reasoning about event ordering, state transitions, sequence numbers, and reconstruction.

---

# Recipe 31 Preview

The next recipe will focus on **Out-of-Order Events**.

We will cover:

- ordered vs unordered event streams
- event time vs processing order
- sequence numbers
- entity-level ordering
- state transitions
- buffering
- reordering windows
- late and out-of-order events together
- incorrect state caused by event reordering
- recovery and replay
- testing intentionally scrambled events
- observing ordering violations

The distinction will be:

    Late data
        |
        v
    Data arrived later than expected

    Out-of-order data
        |
        v
    Events arrived in a different order
    from their logical/event-time order

A real production pipeline may need to handle both at the same time.
