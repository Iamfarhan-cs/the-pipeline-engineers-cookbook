# Recipe 31 — Bottleneck Detection

A pipeline can be reliable, observable, and still be too slow.

A bottleneck is the part of a pipeline that limits overall throughput or causes work to accumulate.

This recipe teaches how to find the limiting stage using measurements rather than guessing.

## 1. Problem Recognition

You have a pipeline:

    Extract → Validate → Transform → Load

The pipeline takes 40 minutes.

Do not immediately optimize everything.

First determine where time is being spent.

Example:

| Stage | Duration |
|---|---:|
| Extract | 4 min |
| Validate | 3 min |
| Transform | 27 min |
| Load | 6 min |

The transformation stage is consuming most of the execution time.

Other bottleneck symptoms include:

- queue depth continuously increases
- one task has much higher duration than others
- CPU is saturated while other resources are idle
- database connections are exhausted
- API rate limits are constantly reached
- workers wait on a dependency
- throughput stops increasing when more workers are added
- downstream stages are frequently idle waiting for one upstream stage

The first skill is recognizing that **slow does not identify the bottleneck**.

## 2. Concept and Reasoning

A bottleneck is the constraint that limits the system.

For a sequential pipeline:

    total_time ≈ stage_1 + stage_2 + stage_3 + ...

For parallel work, the relationship changes because stages can overlap.

The important question is:

> Which constraint prevents the pipeline from processing more work?

Possible constraints:

| Constraint | Example |
|---|---|
| CPU | expensive transformation |
| Memory | large joins or aggregations |
| Disk I/O | slow local storage |
| Database | slow query or lock contention |
| Network | slow API/object storage |
| External API | rate limit |
| Concurrency | too few workers |
| Serialization | expensive encoding/decoding |
| Partition imbalance | one worker receives much more data |
| Dependency | downstream service is slow |

Do not optimize based only on CPU usage.

A pipeline can be CPU-idle while waiting on a database or API.

## 3. Bottleneck Investigation Model

Use this sequence:

    MEASURE
       ↓
    BREAK PIPELINE INTO STAGES
       ↓
    MEASURE EACH STAGE
       ↓
    FIND THE LIMITING CONSTRAINT
       ↓
    FORM A HYPOTHESIS
       ↓
    CHANGE ONE THING
       ↓
    MEASURE AGAIN
       ↓
    KEEP OR REVERT

This prevents random optimization.

## 4. Establish a Baseline

Before changing code, record:

    total duration
    records read
    records written
    throughput
    stage durations
    error rate
    retry count
    queue depth
    resource usage

Example:

    Input:       1,000,000 records
    Duration:    40 minutes
    Throughput:  416 records/sec

Without a baseline, you cannot prove that an optimization helped.

## 5. Instrument Pipeline Stages

Every meaningful stage should expose duration.

Example:

    def measure_stage(name, function):
        start = time.monotonic()

        try:
            return function()
        finally:
            duration = time.monotonic() - start
            metrics.stage_duration(name, duration)

Then:

    extract()
    validate()
    transform()
    load()

becomes measurable.

The goal is not to measure every line of code.

Measure meaningful execution boundaries.

## 6. Build a Stage Profiler

Create a small profiler from scratch.

    class StageProfiler:
        def __init__(self):
            self.records = []

        def start(self, name):
            return {
                "name": name,
                "started_at": time.monotonic(),
            }

        def finish(self, context):
            duration = (
                time.monotonic() -
                context["started_at"]
            )

            self.records.append({
                "name": context["name"],
                "duration_seconds": duration,
            })

            return duration

Use it like:

    profiler = StageProfiler()

    context = profiler.start("transform")

    try:
        transform(records)
    finally:
        profiler.finish(context)

This deliberately simple implementation teaches the mechanism behind stage timing.

## 7. Measure Throughput

Duration alone is insufficient.

Suppose:

    Pipeline A:
    10 minutes
    1,000 records

    Pipeline B:
    20 minutes
    100,000 records

Pipeline B takes longer but processes dramatically more data.

Measure:

    throughput =
        records_processed / duration_seconds

For example:

    100,000 / 1,200
    = 83.3 records/sec

Use the same workload when comparing optimizations.

## 8. Find the Slowest Stage

Given:

    extract      5 sec
    validate     8 sec
    transform   90 sec
    load        12 sec

The transform stage deserves investigation.

But do not conclude that it is definitely the root cause yet.

Ask:

- Is the stage CPU-bound?
- Is it waiting on I/O?
- Is it doing unnecessary work?
- Is its input much larger than expected?
- Is there lock contention?
- Is it calling an external service?
- Is it processing data serially?
- Is there skew?

The slowest stage is a starting point, not a diagnosis.

## 9. Bottleneck Types

### CPU-bound

Symptoms:

    CPU utilization high
    throughput increases with more CPU
    little waiting on I/O

Typical causes:

- expensive parsing
- inefficient algorithms
- repeated serialization
- Python loops over large datasets

### I/O-bound

Symptoms:

    CPU relatively low
    long waits
    slow network/disk/database operations

Typical causes:

- database queries
- APIs
- object storage
- filesystem operations

### Concurrency-bound

Symptoms:

    workers are busy
    work queue remains large
    additional work waits

The system may need more concurrency, but increasing workers blindly can overload dependencies.

### Dependency-bound

Example:

    Pipeline → External API

The API permits only 100 requests/sec.

Adding 1,000 workers will not create 1,000 requests/sec of useful throughput.

The dependency is the bottleneck.

### Data-skew bound

Suppose:

    Worker 1 → 100k records
    Worker 2 → 100k records
    Worker 3 → 100k records
    Worker 4 → 2M records

Worker 4 can determine the total completion time.

This is partition/data skew.

## 10. Amdahl's Law

If a fraction of execution cannot be improved, optimization has a limit.

Suppose:

    80% of runtime = transform
    20% = everything else

If transform becomes infinitely fast, the theoretical maximum speedup is:

    1 / 0.20 = 5x

This teaches an important lesson:

> Optimize the dominant constraint, not a convenient piece of code.

## 11. Controlled Optimization

Change one variable at a time.

Example experiment:

    Baseline:
    40 min

    Change:
    Add database index

    Result:
    18 min

Then test another change separately.

Do not simultaneously:

- rewrite SQL
- increase workers
- change batch size
- add caching
- change database indexes

Otherwise you cannot determine which change mattered.

## 12. Testing

### Test 1 — Stage timing

Verify every required stage produces a duration.

### Test 2 — Failure timing

A failed stage must still record its duration.

### Test 3 — Throughput

Verify:

    throughput = records / duration

### Test 4 — Ordering

Verify stage records retain enough information to reconstruct execution order.

### Test 5 — Slow-stage detection

Create a synthetic slow stage and verify it is identified.

### Test 6 — Empty workload

Do not divide by zero when zero records are processed.

### Test 7 — Parallel stages

Verify concurrent stage executions can be measured independently.

### Test 8 — Regression comparison

Run the same workload before and after an optimization and compare measurements.

## 13. Observability

Useful bottleneck signals include:

    stage_duration_seconds
    records_processed_total
    throughput
    queue_depth
    active_workers
    CPU utilization
    memory utilization
    database latency
    network latency
    retry count

Correlate them.

Example:

    Transform duration ↑
    CPU utilization ↑
    throughput ↓

This suggests a CPU-related constraint.

Another example:

    Transform duration ↑
    CPU utilization ↓
    Database latency ↑

This suggests the transform is waiting on the database.

Metrics become much more useful when interpreted together.

## 14. Intentional Failure

### Failure drill 1 — Artificial CPU bottleneck

Add expensive computation to one stage.

Observe:

    CPU ↑
    stage duration ↑
    throughput ↓

### Failure drill 2 — Artificial I/O bottleneck

Add a controlled sleep or slow dependency.

Observe:

    CPU remains relatively low
    stage duration ↑
    throughput ↓

### Failure drill 3 — Worker bottleneck

Limit worker concurrency.

Observe:

    queue depth ↑
    active work stays limited
    total duration ↑

### Failure drill 4 — Dependency bottleneck

Introduce a strict rate limit.

Observe:

    requests wait
    throughput reaches a ceiling
    increasing workers does not improve throughput

### Failure drill 5 — Data skew

Give one worker significantly more records.

Observe:

    most workers finish early
    one worker remains active
    pipeline completion waits for it

## 15. Recovery

When a bottleneck is detected:

1. Confirm the baseline.
2. Identify the affected stage.
3. Determine whether it is CPU, I/O, concurrency, dependency, or skew related.
4. Check recent changes.
5. Form one optimization hypothesis.
6. Test the change with a controlled workload.
7. Compare against the baseline.
8. Verify correctness did not change.
9. Keep the change only if measurements improve.
10. Continue monitoring after deployment.

Never treat faster execution as sufficient.

A pipeline that is faster but produces incorrect data is not an optimization.

## 16. Bottleneck vs Backpressure

These concepts are related but different.

### Bottleneck

A stage cannot process work quickly enough.

### Backpressure

The system prevents upstream work from overwhelming downstream capacity.

Example:

    Producer → Queue → Slow Consumer

The slow consumer is the bottleneck.

The queue growing is a symptom of the capacity mismatch.

Recipe 18 teaches the flow-control mechanism. This recipe teaches how to identify the limiting constraint.

## 17. Bottleneck vs Rate Limiting

Rate limiting intentionally caps traffic.

Example:

    API limit = 100 requests/sec

The limit may be the effective throughput ceiling.

Do not remove a rate limit simply because it appears to be the bottleneck.

It may protect the dependency.

Instead determine whether:

- the limit is expected
- the limit is configured correctly
- more permitted capacity is available
- batching can reduce request count
- the workload can be scheduled differently

## 18. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Prometheus** | Measure stage latency, throughput, queue depth and resource-related metrics over time. |
| **OpenTelemetry** | Correlate pipeline stages and downstream operations using traces and spans. |
| **Grafana** | Build dashboards that compare stage duration, throughput and resource signals. |

> These tools provide production observability capabilities. The bottleneck-detection reasoning must still be understood independently.

## 19. Production Runbook

### Pipeline becomes slower

Check:

1. Total duration.
2. Stage durations.
3. Input volume.
4. Throughput.
5. Recent deployments.
6. Dependency latency.
7. Resource utilization.
8. Retry count.

### One stage dominates runtime

Check:

1. What operation it performs.
2. CPU usage.
3. I/O latency.
4. Database queries.
5. External API calls.
6. Data volume.
7. Data skew.

### Queue keeps growing

Check:

1. Producer rate.
2. Consumer rate.
3. Worker count.
4. Consumer failures.
5. Dependency limits.
6. Backpressure behavior.

### More workers do not improve throughput

Check:

1. Dependency rate limits.
2. Database capacity.
3. Lock contention.
4. Network capacity.
5. CPU saturation.
6. Partition skew.

The key question is:

> What resource or dependency is currently limiting useful throughput?

## 20. Common Mistakes

### Mistake 1 — Optimizing the slowest-looking code

The slowest stage may be waiting on another dependency.

### Mistake 2 — Changing everything at once

You lose the ability to measure causality.

### Mistake 3 — Measuring only total duration

You cannot locate the constraint.

### Mistake 4 — Increasing concurrency blindly

More workers can overload databases and APIs.

### Mistake 5 — Ignoring input volume

A longer run with ten times more data may actually have better throughput.

### Mistake 6 — Ignoring correctness

Performance improvements must preserve data correctness.

### Mistake 7 — Optimizing without a baseline

You cannot prove improvement without comparable measurements.

## 21. Definition of Done

You are done when you can:

- explain what a bottleneck is
- distinguish bottlenecks from general slowness
- establish a performance baseline
- instrument meaningful pipeline stages
- measure stage duration
- calculate throughput
- identify the dominant constraint
- distinguish CPU-bound and I/O-bound work
- recognize dependency bottlenecks
- recognize concurrency limits
- recognize data skew
- understand the practical implication of Amdahl's Law
- run controlled optimization experiments
- intentionally create CPU, I/O, concurrency, dependency and skew bottlenecks
- diagnose the bottleneck from measurements
- recover and verify the pipeline
- explain Prometheus, OpenTelemetry and Grafana
- operate bottleneck investigation using a production runbook

## 22. What You Learned

The central principle is:

> **Do not optimize what looks slow. Measure the system, identify the limiting constraint, change one thing, and measure again.**

The investigation model is:

    BASELINE
       ↓
    STAGE MEASUREMENTS
       ↓
    BOTTLENECK HYPOTHESIS
       ↓
    CONTROLLED CHANGE
       ↓
    MEASURE
       ↓
    VERIFY CORRECTNESS
       ↓
    KEEP OR REVERT

A production Data Engineer should be able to answer:

> **What is limiting this pipeline right now, and what evidence proves it?**
