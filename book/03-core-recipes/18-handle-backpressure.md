# Recipe 18 — Handle Backpressure

A production data pipeline is a flow system.

Data enters at one rate, moves through processing stages, and leaves at another rate.

    source -> ingestion -> queue -> processing -> database -> downstream

If an upstream stage produces data faster than a downstream stage can safely process it, pressure accumulates.

That condition is called **backpressure**.

Core lifecycle:

    Measure incoming rate
           |
           v
    Measure processing capacity
           |
           v
    Detect pressure
           |
           v
    Identify bottleneck
           |
           v
    Control intake
       /    |     \
    slow   buffer  scale
       \    |     /
           v
    protect downstream
           |
           v
    drain backlog
           |
           v
    reconcile

> Backpressure is not automatically a failure. Uncontrolled backpressure is a production failure risk.

---

## 1. Goal

By the end of this recipe, you should be able to:

- recognize backpressure in a real pipeline
- explain why backlog grows
- distinguish backpressure from ordinary latency
- measure producer rate and consumer rate
- calculate whether a stage can keep up
- identify the stage creating pressure
- measure queue depth and backlog age
- understand bounded and unbounded buffering
- implement bounded queues
- slow or pause upstream production
- use consumer-side flow control
- use rate limiting intentionally
- use concurrency without creating overload
- scale consumers safely
- protect databases and external dependencies
- preserve data while applying pressure
- distinguish buffering from durable storage
- handle overload without silently dropping records
- detect when the system is approaching capacity
- test recovery after pressure is removed
- intentionally create overload
- observe and diagnose a pressure incident
- recover a growing backlog safely
- reconcile after recovery
- define operational thresholds and runbooks.

---

## 2. What Is Backpressure?

Suppose:

    producer = 10,000 events/sec
    consumer = 4,000 events/sec

The system receives:

    10,000

but processes:

     4,000

The difference is:

     6,000 events/sec

If the excess data is buffered, backlog grows.

After one minute:

    backlog growth =
    6,000 * 60
    = 360,000 events

The important relationship is:

    backlog change = arrival rate - processing rate

If:

    arrival rate > processing rate

backlog grows.

If:

    arrival rate < processing rate

backlog drains.

If:

    arrival rate = processing rate

the system can remain approximately stable, assuming the rates are sustainable.

---

## 3. Backpressure Is a Capacity Problem

A pipeline has finite capacity.

For a stage:

    capacity =
        workers
        * processing rate per worker

Example:

    workers = 8
    worker capacity = 500 events/sec

Then approximately:

    total capacity = 4,000 events/sec

If the source produces:

    5,000 events/sec

the stage cannot keep up indefinitely.

Do not solve this by simply increasing the queue.

A larger queue delays the failure.

---

## 4. Backpressure vs Latency

These are related but different.

### Latency

How long one record takes to move through the system.

### Backpressure

The downstream system cannot accept or process incoming work at the rate being offered.

You can have:

    high latency
    but no growing backlog

or:

    growing backlog
    with apparently low processing latency for individual records.

Monitor both.

---

## 5. Recognize the Problem

Common symptoms:

- queue depth continuously increases
- oldest message age increases
- consumer lag increases
- processing latency increases
- database connections remain saturated
- CPU remains near capacity
- memory increases
- worker pools remain fully occupied
- downstream API rate limits increase
- request timeouts increase
- retries increase
- ingestion begins timing out
- batch completion time grows
- pipeline SLA starts being missed.

The most important signal is usually not:

    queue is non-zero

It is:

    queue is growing faster than the system can drain it.

---

## 6. The First Question: Where Is the Pressure?

Do not immediately add workers.

Trace the pipeline:

    source
      |
      v
    ingestion
      |
      v
    queue
      |
      v
    parser
      |
      v
    transformer
      |
      v
    database
      |
      v
    downstream

Measure each boundary.

Example:

    source             8,000/s
    ingestion          8,000/s
    queue              8,000/s
    parser             7,500/s
    transformer        7,400/s
    database           3,000/s

The database is the bottleneck.

Adding parser workers will not solve the database capacity problem.

> Backpressure must be traced to the limiting stage.

---

## 7. Measure Arrival Rate

Measure:

    incoming_events_total

over a time window.

For example:

    arrivals_last_minute = 480,000

Then:

    arrival_rate =
        480,000 / 60
        = 8,000 events/sec

Use a consistent measurement window.

Also measure:

- records/sec
- bytes/sec
- batches/sec
- requests/sec
- partitions/sec.

Count alone may be misleading.

A record may be 1 KB or 10 MB.

---

## 8. Measure Processing Rate

Measure:

    processed_events_total

over the same window.

Example:

    processed_last_minute = 300,000

Then:

    processing_rate =
        300,000 / 60
        = 5,000 events/sec

If:

    arrival_rate = 8,000/s
    processing_rate = 5,000/s

then:

    backlog growth = 3,000/s

This tells you that the pressure is structural, not random noise.

---

## 9. Measure Queue Depth

Track:

    queue_depth

Example:

    09:00  = 10,000
    09:05  = 40,000
    09:10  = 75,000
    09:15  = 110,000

The absolute value matters.

The slope matters more.

Calculate:

    backlog_growth_rate

A queue that remains at 100,000 for an hour is different from one that grows from 10,000 to 100,000 in five minutes.

---

## 10. Measure Oldest Message Age

Queue depth does not tell you how long data has been waiting.

Track:

    oldest_event_age

Example:

    queue depth = 50,000
    oldest event = 8 seconds

versus:

    queue depth = 50,000
    oldest event = 45 minutes

The second condition is much more serious for latency-sensitive pipelines.

This metric is often called:

    backlog age
    consumer lag age
    oldest message age.

---

## 11. Estimate Drain Time

Suppose:

    backlog = 600,000 events
    processing rate = 10,000/s
    arrival rate = 4,000/s

Net drain rate:

    10,000 - 4,000
    = 6,000/s

Estimated drain time:

    600,000 / 6,000
    = 100 seconds

Without subtracting the continuing arrival rate, the estimate would be wrong.

Use:

    drain_time =
        backlog /
        (processing_rate - arrival_rate)

Only valid when:

    processing_rate > arrival_rate

---

## 12. Capacity Headroom

Do not operate every stage at 100% theoretical capacity.

Example:

    sustainable capacity = 10,000/s
    normal traffic = 7,000/s

Headroom:

    3,000/s

Utilization:

    7,000 / 10,000
    = 70%

The remaining capacity absorbs:

- traffic spikes
- retries
- slower downstream dependencies
- deployment effects
- garbage collection
- temporary failures.

A pipeline with no headroom is easier to overload.

---

## 13. Bounded vs Unbounded Buffers

### Unbounded buffer

Conceptually:

    producer -> infinite queue -> consumer

This is dangerous.

If consumers remain slower:

    queue -> keeps growing

Eventually:

- memory is exhausted
- disk is exhausted
- storage cost increases
- latency becomes extreme
- recovery becomes harder.

### Bounded buffer

Conceptually:

    producer -> [capacity-limited queue] -> consumer

When capacity is reached, the producer must:

- wait
- slow down
- pause
- reject
- spill to durable storage
- or apply another explicit policy.

> Every buffer needs an explicit capacity and overload policy.

---

## 14. Buffering Is Not a Recovery Strategy

A queue can absorb temporary bursts.

It cannot permanently compensate for insufficient processing capacity.

Good use:

    arrival = 10,000/s
    processing = 12,000/s

A temporary spike to 15,000/s creates backlog that later drains.

Bad use:

    arrival = 10,000/s
    processing = 6,000/s

The backlog grows forever.

Do not call an indefinitely growing queue “backpressure handling.”

---

## 15. Backpressure Policies

When downstream capacity is exhausted, choose deliberately.

### Policy 1 — Block

Stop accepting more work.

Useful when data must not be lost.

### Policy 2 — Slow producer

Reduce upstream production rate.

Useful when the producer supports flow control.

### Policy 3 — Pause consumption

Temporarily stop pulling from the upstream queue.

Useful when the queue is durable.

### Policy 4 — Buffer

Temporarily absorb a burst.

Useful for short-lived spikes.

### Policy 5 — Spill to durable storage

Move pressure from memory to durable storage.

Useful for larger but recoverable bursts.

### Policy 6 — Scale consumers

Increase processing capacity.

Useful when the bottleneck is horizontally scalable.

### Policy 7 — Reject

Refuse work explicitly.

Useful when the producer can retry later and rejection semantics are safe.

### Policy 8 — Shed non-critical work

Stop optional work while protecting critical processing.

This must be explicit and observable.

---

## 16. Never Silently Drop Data

A dangerous overload implementation is:

    queue full
       |
       v
    drop event

without recording anything.

This creates silent data loss.

If data is rejected, record:

    event_id
    rejection_reason
    timestamp
    source
    pipeline_stage
    retryability
    recovery location

Then make recovery possible.

Backpressure handling must preserve the data contract.

---

## 17. Producer-Side Flow Control

If the producer can observe downstream pressure:

    producer
       |
       v
    capacity check
       |
       +---- capacity available -> send
       |
       +---- capacity unavailable -> wait

Example pseudocode:

~~~python
while events:
    if downstream_has_capacity():
        send(events.pop())
    else:
        sleep(backoff_interval)
~~~

The important property is:

    producer rate follows downstream capacity.

Do not use unlimited retries without delay.

---

## 18. Consumer-Side Flow Control

A consumer can control how much work it pulls.

Instead of:

~~~python
while True:
    batch = read_all_available()
    process(batch)
~~~

Use bounded consumption:

~~~python
while True:
    batch = read_batch(max_records=500)
    process(batch)
~~~

This prevents the consumer from loading an unbounded amount of work into memory.

Bound:

- records
- bytes
- processing time
- concurrent tasks.

---

## 19. Batch Size Is a Backpressure Control

Large batches can improve throughput.

But excessive batch size can increase:

- memory usage
- transaction duration
- lock duration
- failure blast radius
- retry cost
- latency.

Example:

    batch = 10,000

If one record causes the entire transaction to fail, recovery may become expensive.

Use controlled batch sizes and measure the effect.

---

## 20. Concurrency Is Not Free Capacity

Suppose:

    workers = 4
    each worker = 500/s

Increasing to:

    workers = 40

does not guarantee:

    20,000/s

The downstream database may only support:

    5,000 writes/s

The result can be:

    more workers
       |
       v
    more connections
       |
       v
    database saturation
       |
       v
    slower queries
       |
       v
    timeouts
       |
       v
    retries
       |
       v
    even more load

This is an overload feedback loop.

---

## 21. The Retry-Amplification Problem

Suppose:

    normal traffic = 5,000/s
    capacity = 5,000/s

A dependency slows down.

Requests fail.

Workers retry.

Now effective offered load becomes:

    original traffic
    +
    retries

Example:

    original = 5,000/s
    retries = 3,000/s

Total:

    8,000/s

If the dependency is already overloaded, retries make the condition worse.

> Backpressure and retry policy must be designed together.

Use:

- bounded retries
- exponential backoff
- jitter
- retry classification
- concurrency limits
- circuit breaking where appropriate.

---

## 22. Rate Limiting as Backpressure

Suppose an external API allows:

    100 requests/sec

but your workers can generate:

    500 requests/sec

Do not allow the pipeline to continuously exceed the dependency limit.

Apply a rate limiter:

    producer
       |
       v
    rate limiter
       |
       v
    external API

Measure:

    allowed requests
    delayed requests
    rejected requests
    rate-limit responses
    wait time.

Rate limiting is a deliberate pressure-control mechanism.

---

## 23. Database Backpressure

Databases are common bottlenecks.

Symptoms:

- connection pool exhaustion
- lock contention
- slow inserts
- slow updates
- high transaction duration
- CPU saturation
- I/O saturation
- replication lag
- increasing query latency.

Do not respond only by increasing worker count.

First determine:

    Is the database the limiting resource?

Then consider:

- batch writes
- bulk loading
- connection limits
- indexing
- query optimization
- partitioning
- transaction size
- concurrency limits
- write scheduling.

---

## 24. External API Backpressure

External dependencies may impose:

    rate limits
    quotas
    concurrency limits
    payload limits
    timeouts.

If the dependency returns:

    HTTP 429

do not immediately launch more requests.

Interpret it as a pressure signal.

Possible response:

    reduce concurrency
       |
       v
    wait
       |
       v
    retry with backoff
       |
       v
    gradually recover

---

## 25. Memory Backpressure

An in-memory queue is especially dangerous.

Example:

~~~python
events = []

while True:
    events.extend(read_events())
    process(events)
~~~

If processing falls behind, memory grows.

A safer design uses a bounded queue.

Conceptually:

~~~python
queue = BoundedQueue(max_items=10_000)
~~~

When full:

    producer waits

or follows the explicit overflow policy.

Never assume RAM is an infinite buffer.

---

## 26. Disk Backpressure

Disk can also become the limiting resource.

Symptoms:

- temporary files grow
- local spool directory grows
- disk utilization approaches capacity
- write latency increases.

Track:

    disk_used_bytes
    disk_free_bytes
    spool_size
    spool_oldest_age

A durable spill mechanism must itself have capacity limits and cleanup/recovery rules.

---

## 27. Queue Partition Hotspots

A distributed queue may appear healthy globally while one partition is overloaded.

Example:

    partition 0 = 5,000/s
    partition 1 = 800/s
    partition 2 = 700/s
    partition 3 = 500/s

Total traffic may look acceptable.

Partition 0 is not.

Monitor:

- lag by partition
- throughput by partition
- oldest message by partition
- consumer assignment
- partition skew.

Backpressure can be local rather than global.

---

## 28. Detect the Bottleneck With a Boundary Table

Create a table like:

| Stage | Input/s | Output/s | Queue | Latency | Saturated |
|---|---:|---:|---:|---:|---|
| Ingestion | 8,000 | 8,000 | 0 | 20 ms | No |
| Parser | 8,000 | 7,900 | 100 | 30 ms | No |
| Transform | 7,900 | 7,700 | 200 | 50 ms | No |
| Database | 7,700 | 5,000 | 2,700 | 400 ms | Yes |
| Export | 5,000 | 5,000 | 0 | 60 ms | No |

This makes the bottleneck visible.

---

## 29. Backlog Age Is Often More Important Than Backlog Size

Two pipelines:

### Pipeline A

    backlog = 1,000,000
    oldest age = 2 minutes

### Pipeline B

    backlog = 20,000
    oldest age = 3 hours

Pipeline B may violate a freshness requirement much more severely.

Track both:

    backlog size
    backlog age

For partitioned systems, track them by partition.

---

## 30. High-Water Marks

Define operational thresholds.

Example:

    normal       < 10,000
    warning      >= 10,000
    critical     >= 100,000

Also define age thresholds:

    normal       < 30 sec
    warning      >= 30 sec
    critical     >= 5 min

Thresholds should be based on:

- normal workload
- recovery capacity
- SLA/SLO
- storage limits
- downstream capacity.

Do not copy thresholds from another system without measuring your own pipeline.

---

## 31. Pressure States

Use explicit states:

    NORMAL
    ELEVATED
    PRESSURED
    CRITICAL
    RECOVERING

Example:

### NORMAL

Backlog stable and age within target.

### ELEVATED

Backlog increasing but recoverable.

### PRESSURED

Capacity margin is low.

### CRITICAL

Backlog or age threatens SLA or storage.

### RECOVERING

Processing capacity exceeds arrival rate and backlog is draining.

Explicit states make runbooks and alerts easier to operate.

---

## 32. Observability Metrics

At minimum monitor:

### Flow

    records_in_total
    records_out_total

### Rate

    records_in_per_second
    records_out_per_second

### Backlog

    queue_depth
    queue_bytes
    consumer_lag

### Age

    oldest_event_age
    oldest_partition_event_age

### Capacity

    worker_utilization
    concurrency
    available_capacity

### Dependencies

    database_latency
    API_latency
    rate_limit_count
    timeout_count

### Failures

    retry_count
    rejected_count
    dead_letter_count
    processing_error_count

### Recovery

    backlog_drain_rate
    estimated_drain_time
    recovery_duration.

---

## 33. Alert Design

Bad alert:

    Queue depth > 0

This will alert constantly.

Better:

    queue depth increasing continuously for 10 minutes

Better still:

    backlog age > 5 minutes
    AND processing rate < arrival rate

Also alert when:

    disk usage approaches limit
    database saturation persists
    external API rate limits increase
    retry volume spikes
    drain time exceeds SLA.

Alert on conditions that require action.

---

## 34. Capacity Test

Before relying on a pipeline, determine sustainable capacity.

Test:

    1,000/s
    2,000/s
    4,000/s
    6,000/s
    8,000/s
    10,000/s

Measure:

- throughput
- latency
- CPU
- memory
- database load
- queue depth
- error rate.

Find where throughput stops scaling.

That point identifies a practical capacity boundary.

---

## 35. Burst Test

A pipeline may handle steady traffic but fail during spikes.

Test:

    baseline = 2,000/s

then:

    burst = 10,000/s

Observe:

    queue depth
    memory
    processing rate
    oldest event age

Then return to:

    2,000/s

Expected:

    backlog increases during burst
    backlog drains afterward
    no data is lost
    system returns to normal state.

---

## 36. Sustained Overload Test

Now intentionally exceed capacity.

Example:

    arrival = 10,000/s
    capacity = 6,000/s

Expected:

    backlog grows

The important test is what happens next.

The system should:

- detect pressure
- prevent uncontrolled memory growth
- apply flow control
- preserve data
- expose the backlog
- protect downstream systems
- remain recoverable.

Do not expect the pipeline to magically process more than its measured capacity.

---

## 37. Recovery Test

Remove the overload.

Change:

    arrival = 4,000/s
    capacity = 8,000/s

Expected:

    backlog drains

Measure:

    drain rate
    oldest age
    errors
    resource utilization

Do not declare recovery merely because:

    queue depth decreased once.

Recovery is complete when:

    backlog = within target
    oldest age = within target
    errors = normal
    downstream state = reconciled.

---

## 38. Test — Producer Faster Than Consumer

Configure:

    producer = 10,000/s
    consumer = 5,000/s

Run for five minutes.

Expected:

    backlog grows predictably

Verify:

    no records silently disappear
    queue capacity is respected
    pressure is visible
    alert fires.

---

## 39. Test — Consumer Faster Than Producer

Configure:

    producer = 5,000/s
    consumer = 10,000/s

Expected:

    backlog remains low
    consumer waits for work
    resources remain within expected limits.

---

## 40. Test — Bounded Queue

Configure:

    max_queue = 10,000

Fill it beyond capacity.

Expected:

    producer blocks
    or explicit overflow policy executes

Never:

    queue grows indefinitely.

---

## 41. Test — Retry Amplification

Inject a downstream timeout.

Observe:

    failures
    retries
    concurrency
    request rate

Expected:

    retry rate is bounded
    backoff increases delay
    concurrency does not grow without limit
    downstream receives controlled load.

---

## 42. Test — Database Saturation

Artificially slow database writes.

Expected:

    processing rate decreases
    queue grows
    consumer concurrency remains bounded
    database is protected from uncontrolled connection growth.

---

## 43. Test — External Rate Limit

Configure a test dependency to return:

    HTTP 429

Expected:

    requests slow down
    retries use backoff
    backlog is visible
    no retry storm occurs.

---

## 44. Test — Partition Hotspot

Create uneven traffic:

    partition A = 10,000/s
    partition B = 500/s
    partition C = 500/s

Expected:

    hotspot is visible
    global metrics do not hide the overloaded partition
    consumer assignment and partition capacity can be investigated.

---

## 45. Intentionally Break the Pipeline

Perform these drills only in a safe environment.

### Drill 1 — Slow the consumer

Expected: backlog grows.

### Drill 2 — Stop the consumer

Expected: durable backlog grows while source data remains recoverable.

### Drill 3 — Reduce database capacity

Expected: downstream becomes the bottleneck and pressure propagates upstream.

### Drill 4 — Fill the in-memory queue

Expected: bounded queue applies its configured policy.

### Drill 5 — Remove rate limiting

Expected: external dependency begins returning throttling responses.

Then restore rate limiting.

### Drill 6 — Remove retry backoff

Expected: retry amplification becomes visible.

Restore bounded exponential backoff.

### Drill 7 — Increase worker count beyond database capacity

Expected: database saturation worsens rather than throughput scaling linearly.

### Drill 8 — Create a partition hotspot

Expected: one partition shows disproportionate lag.

### Drill 9 — Fill local disk spool

Expected: capacity alert fires before disk exhaustion.

### Drill 10 — Create a sustained overload

Expected: pressure state becomes CRITICAL without silent data loss.

### Drill 11 — Restore capacity

Expected: state changes to RECOVERING and backlog drains.

### Drill 12 — Run recovery twice

Expected: processing remains idempotent and no duplicate output is created.

---

## 46. Incident Investigation

Suppose:

    queue_depth = 2,000,000
    oldest_event_age = 18 minutes

Start with:

### Step 1 — Compare rates

    arrival rate
    processing rate

If:

    arrival > processing

the backlog is still growing.

### Step 2 — Find the first bottleneck

Inspect every boundary:

    source
    ingestion
    queue
    parser
    transformer
    database
    downstream

### Step 3 — Inspect resource saturation

Check:

    CPU
    memory
    disk
    network
    database
    external API quotas.

### Step 4 — Check recent changes

Look for:

    deployment
    schema migration
    configuration change
    traffic spike
    dependency degradation.

### Step 5 — Check retries

Determine whether failures created additional load.

### Step 6 — Check partition skew

A single hot partition can create local pressure.

### Step 7 — Protect the bottleneck

Do not keep feeding an already saturated dependency.

### Step 8 — Choose the pressure-control action

Possible actions:

    slow producer
    pause consumers
    reduce concurrency
    rate-limit
    scale consumers
    increase batch efficiency
    spill to durable storage.

### Step 9 — Estimate recovery

Calculate:

    drain_rate =
        processing_rate - arrival_rate

Then:

    drain_time =
        backlog / drain_rate

### Step 10 — Reconcile after recovery

Confirm:

    expected IDs
    processed IDs
    duplicates
    rejected records
    failed records
    downstream totals.

---

## 47. Recovery Strategies

### Temporary traffic spike

Use bounded buffering and allow the backlog to drain.

### Consumer slowdown

Optimize or scale the bottleneck.

### Database bottleneck

Reduce concurrency and improve write efficiency before adding more workers.

### External API throttling

Respect the dependency's rate limit and use controlled retry/backoff.

### Memory pressure

Reduce queue size and batch size; move buffering to durable storage when appropriate.

### Disk pressure

Drain or process the spool, enforce retention, and restore capacity before disk exhaustion.

### Partition hotspot

Investigate partitioning and consumer assignment rather than scaling unrelated stages.

### Sustained overload

Reduce intake, scale the actual bottleneck, or redesign the capacity boundary.

Do not hide sustained overload by increasing an unbounded buffer.

---

## 48. Backpressure and Idempotency

Pressure recovery often involves retries or reprocessing.

Therefore:

    backpressure
         |
         v
    retry/replay
         |
         v
    duplicate delivery possible

The processing path must remain idempotent.

Use stable:

    event_id
    operation_id
    batch_id

and enforce appropriate uniqueness.

A pressure-control mechanism that creates duplicates is not a complete solution.

---

## 49. Backpressure and Checkpointing

If a consumer pauses because downstream capacity is exhausted, it must not lose its position.

Use checkpoints or durable queue offsets.

Conceptually:

    read
      |
      v
    process
      |
      v
    commit durable result
      |
      v
    advance checkpoint

Do not acknowledge work before the required durable processing succeeds.

Otherwise pressure can turn into data loss.

---

## 50. Backpressure and Dead Letters

Backpressure does not mean:

    send everything to DLQ.

A slow dependency is usually not a data-quality failure.

Classify correctly:

    temporary capacity problem
        -> control flow / retry / wait

    malformed data
        -> DQ / quarantine / DLQ

    permanent business rejection
        -> reject / record outcome

Mixing these categories creates operational confusion.

---

## 51. Backpressure and Data Freshness

A pipeline can be processing successfully while becoming increasingly stale.

Example:

    processing = 5,000/s
    arrival = 6,000/s

The consumer is healthy.

But backlog increases:

    1,000/s

Therefore monitor:

    processing health
    AND freshness.

Useful signals:

    oldest event age
    end-to-end latency
    watermark lag
    queue age.

---

## 52. Backpressure State Machine

A practical state machine:

    NORMAL
       |
       v
    ELEVATED
       |
       v
    PRESSURED
       |
       v
    CRITICAL
       |
       v
    RECOVERING
       |
       v
    NORMAL

Example transitions:

    backlog slope > threshold
        -> ELEVATED

    capacity headroom < threshold
        -> PRESSURED

    oldest age > SLA
        -> CRITICAL

    processing rate > arrival rate
        -> RECOVERING

    backlog and age return to target
        -> NORMAL

State transitions should be observable.

---

## 53. Practical Implementation Sequence

For an existing pipeline:

    1. Map every pipeline stage
    2. Identify every queue and buffer
    3. Measure arrival rate
    4. Measure processing rate
    5. Measure queue depth
    6. Measure backlog age
    7. Measure resource utilization
    8. Calculate sustainable capacity
    9. Identify bottleneck stages
    10. Define capacity headroom
    11. Bound in-memory queues
    12. Define overflow behavior
    13. Add producer flow control where possible
    14. Bound consumer concurrency
    15. Bound batch sizes
    16. Add rate limits for external dependencies
    17. Add retry backoff and jitter
    18. Add pressure-state tracking
    19. Add backlog and age metrics
    20. Add capacity alerts
    21. Test burst overload
    22. Test sustained overload
    23. Test dependency throttling
    24. Test recovery and backlog drain
    25. Reconcile after recovery
    26. Document the production runbook.

---

## 54. Production Runbook

When backpressure is detected:

    1. Confirm backlog is actually growing
    2. Compare arrival and processing rates
    3. Identify the first bottleneck
    4. Check queue depth and oldest age
    5. Check CPU, memory, disk, network, and database saturation
    6. Check external dependency limits
    7. Check retry amplification
    8. Check partition-level skew
    9. Protect the bottleneck
    10. Reduce intake or concurrency if required
    11. Scale the actual bottleneck if safe
    12. Verify no silent data loss
    13. Estimate backlog drain time
    14. Monitor recovery
    15. Verify backlog age returns to target
    16. Reconcile processed records
    17. Verify duplicates and rejected records
    18. Record root cause
    19. Record capacity findings
    20. Update limits, alerts, or architecture if required.

---

## 55. Common Mistakes

### Mistake 1 — Increasing the queue forever

A larger queue does not fix insufficient processing capacity.

### Mistake 2 — Adding workers without finding the bottleneck

More workers can overload the real limiting dependency.

### Mistake 3 — Using RAM as an unlimited buffer

Memory exhaustion turns pressure into a crash.

### Mistake 4 — Ignoring backlog age

Queue size alone does not show freshness.

### Mistake 5 — Retrying immediately

Immediate retries can create a retry storm.

### Mistake 6 — Ignoring external rate limits

The dependency's capacity is part of your pipeline's capacity.

### Mistake 7 — Dropping records silently

This converts overload into data loss.

### Mistake 8 — Acknowledging work too early

The record may disappear before durable processing succeeds.

### Mistake 9 — Measuring only global queue depth

One hot partition can be hidden by healthy partitions.

### Mistake 10 — Treating every failure as a DQ failure

Temporary dependency pressure should not automatically enter the dead-letter lifecycle.

### Mistake 11 — Running at 100% capacity

No headroom means small spikes can create backlog.

### Mistake 12 — Scaling the wrong stage

Scaling a parser does not fix a saturated database.

### Mistake 13 — Declaring recovery too early

A single decreasing queue measurement does not prove recovery.

### Mistake 14 — Ignoring reconciliation

A drained queue does not prove that all expected data reached the destination.

### Mistake 15 — Testing only steady traffic

Production traffic contains bursts and dependency degradation.

---


## Production Tools You Should Know

These tools solve or provide production implementations of concepts covered in this recipe. Learn what the tool provides, but understand the underlying problem first.

| Tool | What to know |
|---|---|
| **Apache Kafka** | Durable buffering, partitions, consumer groups, and lag. |
| **Apache Flink** | Streaming backpressure and flow coordination. |
| **RabbitMQ** | Queue-based flow control and consumer prefetch. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the engineering mechanism.

---

## Implementation Lab — Runnable Backpressure Controller

This implementation demonstrates a bounded queue, producer blocking, consumer throughput, pressure metrics, and recovery. It is deliberately small enough to run locally and understand before applying the same ideas to Kafka, RabbitMQ, APIs, or database workers.

### 1. Bounded queue

```python
# src/backpressure.py
from dataclasses import dataclass
from queue import Queue, Full, Empty
from threading import Thread
import time


@dataclass
class Metrics:
    accepted: int = 0
    completed: int = 0
    blocked: int = 0


class BackpressurePipeline:
    def __init__(self, max_queue: int = 100):
        self.queue = Queue(maxsize=max_queue)
        self.metrics = Metrics()
        self.running = True

    def submit(self, event: dict, timeout: float = 0.1) -> bool:
        try:
            self.queue.put(event, timeout=timeout)
            self.metrics.accepted += 1
            return True
        except Full:
            self.metrics.blocked += 1
            return False

    def consume(self, processing_seconds: float = 0.01):
        while self.running:
            try:
                event = self.queue.get(timeout=0.1)
            except Empty:
                continue

            try:
                time.sleep(processing_seconds)
                self.metrics.completed += 1
            finally:
                self.queue.task_done()

    def stop(self):
        self.running = False

    def backlog(self) -> int:
        return self.queue.qsize()
```

### 2. Tests

```python
# tests/test_backpressure.py
from src.backpressure import BackpressurePipeline


def test_queue_is_bounded():
    pipeline = BackpressurePipeline(max_queue=2)

    assert pipeline.submit({"id": 1})
    assert pipeline.submit({"id": 2})
    assert pipeline.submit({"id": 3}, timeout=0) is False

    assert pipeline.backlog() == 2
    assert pipeline.metrics.blocked == 1


def test_consumer_releases_queue_capacity():
    pipeline = BackpressurePipeline(max_queue=1)

    assert pipeline.submit({"id": 1})
    assert pipeline.backlog() == 1

    pipeline.queue.get_nowait()
    pipeline.queue.task_done()

    assert pipeline.submit({"id": 2}, timeout=0)
```

### 3. Producer/consumer rate experiment

Run this small experiment:

```python
# examples/pressure_demo.py
from src.backpressure import BackpressurePipeline
from threading import Thread
import time


pipeline = BackpressurePipeline(max_queue=100)

worker = Thread(
    target=pipeline.consume,
    kwargs={"processing_seconds": 0.01},
    daemon=True,
)
worker.start()

for i in range(500):
    accepted = pipeline.submit({"id": i}, timeout=0)
    if not accepted:
        print("PRODUCER BLOCKED", i)

    time.sleep(0.001)

time.sleep(2)

print("accepted =", pipeline.metrics.accepted)
print("completed =", pipeline.metrics.completed)
print("blocked =", pipeline.metrics.blocked)
print("backlog =", pipeline.backlog())

pipeline.stop()
```

The producer attempts roughly 1,000 events/sec while the consumer can process roughly 100 events/sec.

The queue should fill and producer blocking should become visible.

### 4. Calculate drain time

```python
def estimate_drain_time(
    backlog: int,
    arrival_rate: float,
    processing_rate: float,
) -> float | None:
    net_drain_rate = processing_rate - arrival_rate

    if net_drain_rate <= 0:
        return None

    return backlog / net_drain_rate
```

Test:

```python
assert estimate_drain_time(600_000, 4_000, 10_000) == 100.0
assert estimate_drain_time(600_000, 8_000, 10_000) == 300.0
assert estimate_drain_time(600_000, 10_000, 10_000) is None
```

### 5. Pressure state

```python
def pressure_state(
    backlog: int,
    oldest_age_seconds: float,
    warning_backlog: int = 10_000,
    critical_backlog: int = 100_000,
) -> str:
    if backlog >= critical_backlog or oldest_age_seconds >= 300:
        return "CRITICAL"

    if backlog >= warning_backlog or oldest_age_seconds >= 30:
        return "PRESSURED"

    if backlog > 0:
        return "ELEVATED"

    return "NORMAL"
```

### 6. PostgreSQL queue accounting

For a database-backed worker, persist durable work state rather than relying on RAM:

```sql
CREATE TABLE work_queue (
    event_id      TEXT PRIMARY KEY,
    payload       JSONB NOT NULL,
    status        TEXT NOT NULL
        CHECK (status IN ('READY', 'PROCESSING', 'DONE', 'FAILED')),
    available_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    claimed_at    TIMESTAMPTZ,
    completed_at  TIMESTAMPTZ
);

CREATE INDEX idx_work_queue_ready
    ON work_queue (available_at)
    WHERE status = 'READY';
```

Claim a bounded batch:

```sql
WITH claimed AS (
    SELECT event_id
    FROM work_queue
    WHERE status = 'READY'
      AND available_at <= now()
    ORDER BY available_at
    FOR UPDATE SKIP LOCKED
    LIMIT 100
)
UPDATE work_queue w
SET
    status = 'PROCESSING',
    claimed_at = now()
FROM claimed
WHERE w.event_id = claimed.event_id
RETURNING w.*;
```

This prevents one worker from claiming unlimited work.

### 7. Intentional overload drill

Configure:

```text
producer = 1,000 events/sec
consumer  = 100 events/sec
queue     = 100 events
```

Expected:

```text
queue fills
    ↓
producer blocks
    ↓
backpressure becomes visible
    ↓
no unlimited memory growth
```

Now increase consumer capacity to 2,000 events/sec.

Expected:

```text
backlog drains
    ↓
pressure decreases
    ↓
state returns to NORMAL
```

### 8. Retry-amplification drill

Add a simulated dependency failure:

```python
def call_dependency(event: dict) -> None:
    raise TimeoutError("dependency timeout")
```

Do not write:

```python
while True:
    try:
        call_dependency(event)
        break
    except TimeoutError:
        continue
```

That creates an uncontrolled retry loop.

Use bounded retries:

```python
import random
import time


def call_with_backoff(operation, max_attempts: int = 5):
    for attempt in range(max_attempts):
        try:
            return operation()
        except TimeoutError:
            if attempt == max_attempts - 1:
                raise

            delay = min(30, 2 ** attempt) + random.uniform(0, 0.5)
            time.sleep(delay)
```

The combination of bounded queues, bounded concurrency, and bounded retries prevents pressure from becoming a feedback loop.


## 56. Definition of Done

- [ ] Every major pipeline stage is mapped.
- [ ] Every queue and buffer is identified.
- [ ] Arrival rate is measured.
- [ ] Processing rate is measured.
- [ ] Queue depth is measured.
- [ ] Queue age is measured.
- [ ] Partition-level pressure is visible where applicable.
- [ ] Sustainable processing capacity is known.
- [ ] Capacity headroom is defined.
- [ ] In-memory buffers are bounded.
- [ ] Overflow behavior is explicit.
- [ ] Producer flow control exists where supported.
- [ ] Consumer concurrency is bounded.
- [ ] Batch sizes are bounded and measured.
- [ ] External dependencies have rate limits where required.
- [ ] Retries use bounded backoff and jitter.
- [ ] Retry amplification is observable.
- [ ] Database capacity is treated as a pipeline constraint.
- [ ] Pressure states are defined.
- [ ] Backlog growth alerts exist.
- [ ] Backlog age alerts exist.
- [ ] Resource saturation is observable.
- [ ] Silent record dropping is prevented.
- [ ] Durable acknowledgment/checkpoint semantics are correct.
- [ ] Burst testing has been performed.
- [ ] Sustained overload testing has been performed.
- [ ] Dependency throttling has been tested.
- [ ] Recovery and backlog draining have been tested.
- [ ] Recovery is idempotent.
- [ ] Reconciliation runs after recovery.
- [ ] A production backpressure runbook exists.
- [ ] Failure drills have been performed.
- [ ] Final system behavior has been independently verified.

---

## 57. What You Learned

Backpressure is not simply:

    queue is getting large

It is:

    incoming work
          |
          v
    available capacity
          |
          v
    pressure detection
          |
          v
    controlled intake
          |
          v
    protected downstream
          |
          v
    backlog recovery
          |
          v
    reconciliation

The most important lessons are:

1. Backpressure occurs when offered work exceeds sustainable downstream capacity.
2. Backlog growth is determined by arrival rate minus processing rate.
3. The first task is to find the actual bottleneck.
4. Queue depth and backlog age are both important.
5. Buffers absorb bursts; they do not solve permanent capacity shortages.
6. Every buffer needs an explicit capacity and overflow policy.
7. Concurrency is not unlimited capacity.
8. More workers can make a saturated dependency worse.
9. Retries can amplify overload.
10. Rate limiting is a valid form of flow control.
11. Database and external API capacity are pipeline capacity constraints.
12. Partition-level hotspots can be hidden by global metrics.
13. Pressure must not silently become data loss.
14. Durable acknowledgment and checkpointing protect records during pressure.
15. Backpressure must work together with idempotency, retries, DQ, and replay.
16. Recovery means the backlog drains and freshness returns to target.
17. Reconciliation proves that recovery restored the expected data.
18. Sustained overload requires capacity or architecture changes, not an infinitely larger queue.

The objective is not:

    Keep the queue empty at all costs.

The objective is:

    Control the flow,
    protect the bottleneck,
    preserve every required record,
    recover from pressure,
    and prove the final data state is correct.

When you can intentionally overload a pipeline, observe exactly where pressure forms, control the flow without silent loss, restore capacity, drain the backlog, and reconcile the final state, you can operate backpressure as an engineering mechanism rather than discovering it during an outage.
