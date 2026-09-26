# Recipe 30 — Pipeline Metrics

Logs explain individual events. Run tracking records execution state. Metrics answer a different question:

> **How is the pipeline behaving over time?**

This recipe teaches how to define, implement, test, and operate pipeline metrics so an engineer can detect failures, degradation, bottlenecks, and abnormal behavior before relying on manual log investigation.

## 1. Problem Recognition

A pipeline can appear healthy while slowly becoming worse.

For example:

    Monday      10 minutes
    Tuesday     12 minutes
    Wednesday   18 minutes
    Thursday    31 minutes
    Friday      47 minutes

Nothing may have crashed, but the pipeline is degrading.

Other examples:

- failure rate increases
- throughput decreases
- retries increase
- API latency increases
- rows processed suddenly drop
- quarantine volume increases
- pipeline runs miss their expected schedule
- task duration becomes highly variable

Logs can show individual events, but metrics make these patterns measurable.

## 2. Concept and Reasoning

A metric is a numerical measurement collected over time.

Common pipeline metric types:

| Type | Purpose |
|---|---|
| Counter | Counts occurrences |
| Gauge | Represents a current value |
| Histogram | Measures distributions such as duration |
| Rate | Measures change per unit of time |

For pipeline engineering, the important signals are usually:

    throughput
    latency
    errors
    retries
    data volume
    freshness
    resource usage

### Metrics vs logs

| Logs | Metrics |
|---|---|
| Detailed event evidence | Numerical behavior |
| Good for investigation | Good for trends |
| High-cardinality details possible | Must control dimensions |
| Individual events | Aggregated measurements |

Use both.

## 3. What to Measure

A basic pipeline should measure:

### Execution

    pipeline_runs_total
    pipeline_success_total
    pipeline_failure_total
    pipeline_duration_seconds

### Tasks

    task_runs_total
    task_success_total
    task_failure_total
    task_duration_seconds

### Data volume

    rows_read_total
    rows_written_total
    rows_failed_total
    rows_quarantined_total

### Reliability

    retries_total
    retry_exhausted_total
    reconciliation_failures_total

### Performance

    records_per_second
    bytes_per_second

Do not create metrics merely because a number exists. A metric should answer an operational question.

## 4. Metric Dimensions

A metric may have labels/dimensions:

    pipeline="payments"
    task="transform"
    environment="production"

Good dimensions help comparison.

Bad dimensions create enormous cardinality.

Avoid labels such as:

    user_id
    transaction_id
    run_id
    email
    request_body

A unique value for every event can create a metric-series explosion.

Use run IDs in logs and traces, not as metric labels.

## 5. Implementation

For the learning implementation, build a small in-memory metrics registry.

### 5.1 Counter

    class Counter:
        def __init__(self):
            self.value = 0

        def inc(self, amount=1):
            self.value += amount

        def get(self):
            return self.value

Example:

    pipeline_runs = Counter()
    pipeline_runs.inc()

### 5.2 Gauge

    class Gauge:
        def __init__(self, value=0):
            self.value = value

        def set(self, value):
            self.value = value

        def inc(self, amount=1):
            self.value += amount

        def dec(self, amount=1):
            self.value -= amount

        def get(self):
            return self.value

Useful for:

    active_runs
    queue_depth
    current_workers

### 5.3 Histogram

A simple learning histogram can collect observations:

    class Histogram:
        def __init__(self):
            self.values = []

        def observe(self, value):
            self.values.append(value)

        def count(self):
            return len(self.values)

        def average(self):
            if not self.values:
                return 0
            return sum(self.values) / len(self.values)

A production histogram should use bounded buckets or a proper metrics library rather than retaining every observation in memory.

## 6. Build a Metrics Registry

A basic registry can group metrics:

    class Metrics:
        def __init__(self):
            self.pipeline_runs = Counter()
            self.pipeline_success = Counter()
            self.pipeline_failure = Counter()

            self.task_runs = Counter()
            self.task_success = Counter()
            self.task_failure = Counter()

            self.rows_read = Counter()
            self.rows_written = Counter()
            self.rows_failed = Counter()

            self.task_duration = Histogram()
            self.pipeline_duration = Histogram()

Then the pipeline updates metrics at lifecycle points.

## 7. Instrument a Pipeline

Example:

    metrics.pipeline_runs.inc()

    start = time.monotonic()

    try:
        run_pipeline()
    except Exception:
        metrics.pipeline_failure.inc()
        raise
    else:
        metrics.pipeline_success.inc()
    finally:
        metrics.pipeline_duration.observe(
            time.monotonic() - start
        )

This gives:

    runs
      ↓
    success / failure
      ↓
    duration

## 8. Calculate Useful Rates

Counters are usually converted into rates.

For example:

    success_rate =
        successful_runs / total_runs

Failure rate:

    failure_rate =
        failed_runs / total_runs

Throughput:

    throughput =
        rows_processed / elapsed_seconds

Retry rate:

    retry_rate =
        retries / task_attempts

Be careful with small sample sizes. A single failure in one run is 100% failure rate for that tiny sample but does not necessarily represent a long-term problem.

## 9. Data Volume Metrics

Suppose yesterday:

    rows_read = 1,000,000

Today:

    rows_read = 12,000

The pipeline may technically be SUCCESS, but the volume change is suspicious.

Track:

    rows_read
    rows_written
    rows_rejected
    rows_quarantined

Then compare against historical behavior.

Metrics can therefore detect **silent degradation**, not just explicit failures.

## 10. Freshness Metrics

A pipeline can successfully process old data while still violating a freshness requirement.

Define:

    freshness =
        current_time - newest_successfully_processed_data_time

Example:

    newest source event = 10:55
    current time         = 11:05

    freshness = 10 minutes

This becomes important later when learning data SLAs and SLOs.

## 11. Testing

### Test 1 — Counter

Increment the counter and verify its value.

### Test 2 — Gauge

Set, increment, and decrement values.

### Test 3 — Histogram

Record observations and verify count and summary statistics.

### Test 4 — Successful pipeline

Verify:

    runs += 1
    success += 1
    failure unchanged

### Test 5 — Failed pipeline

Verify:

    runs += 1
    failure += 1

### Test 6 — Duration

Verify a duration observation is recorded even when the pipeline fails.

### Test 7 — Volume

Verify rows read/written are recorded correctly.

### Test 8 — Labels

Verify allowed metric dimensions are stable and sensitive/high-cardinality identifiers are rejected.

## 12. Observability

Metrics are an observability layer themselves.

Build a small operational view such as:

    Pipeline: payments

    Runs:             1,240
    Success rate:       99.1%
    Failure rate:        0.9%
    Avg duration:       8.2 min
    Rows processed:   18.4M
    Retry rate:          2.1%
    Current active:         2

The exact dashboard technology is less important at this stage than learning which signals matter.

## 13. Intentional Failure

### Failure drill 1 — Increase failure rate

Force several tasks to fail.

Observe:

    pipeline_failure_total ↑
    task_failure_total ↑

### Failure drill 2 — Slow the pipeline

Add an artificial delay.

Observe:

    task_duration ↑
    pipeline_duration ↑
    throughput ↓

### Failure drill 3 — Reduce source volume

Provide only a small fraction of the expected input.

The pipeline may remain SUCCESS.

Check whether:

    rows_read

reveals the abnormal behavior.

### Failure drill 4 — Retry storm

Force a transient dependency failure.

Observe:

    retries_total ↑

Then determine whether the retries are creating additional load.

## 14. Recovery

When a metric crosses an operational threshold:

1. Identify which metric changed.
2. Identify the affected pipeline/task.
3. Compare against historical behavior.
4. Use run tracking to find affected runs.
5. Use logs to identify the underlying event.
6. Determine whether the problem is transient or persistent.
7. Recover the pipeline.
8. Verify that the metric returns to the expected range.

The investigation path is:

    METRIC
       ↓
    RUN
       ↓
    LOG
       ↓
    ROOT CAUSE
       ↓
    RECOVERY

## 15. Metric Naming

Use consistent names.

Prefer:

    pipeline_runs_total
    task_failures_total
    task_duration_seconds
    rows_processed_total

Avoid inconsistent names such as:

    pipelineRuns
    task_fail
    duration
    processedRows

Units should be clear.

For example:

    _seconds
    _bytes
    _total

Consistent naming becomes increasingly important as the number of pipelines grows.

## 16. Metric Cardinality

Suppose you create:

    task_duration_seconds{run_id="..."}

for millions of unique runs.

You may create millions of metric series.

Instead:

    task_duration_seconds{
        pipeline="payments",
        task="transform"
    }

Keep high-cardinality information in logs or traces.

This is one of the most important production metrics lessons.

## 17. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Prometheus** | Pull-based metrics collection, labels, counters, gauges, histograms and PromQL. |
| **OpenTelemetry** | Standard telemetry model and APIs for metrics, traces and logs. |
| **Grafana** | Visualization and dashboarding for operational metrics and telemetry. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding metric design and instrumentation.

## 18. Production Runbook

### Failure rate increases

Check:

1. Failure metric.
2. Affected pipeline/task.
3. Recent deployments.
4. Error logs.
5. Dependency health.
6. Retry behavior.
7. Data changes.

### Duration increases

Check:

1. Task-level duration.
2. Input volume.
3. Database/query latency.
4. Network latency.
5. Worker/resource utilization.
6. Recent code changes.

### Throughput decreases

Check:

1. Input volume.
2. Worker count.
3. CPU/memory.
4. Database performance.
5. Network performance.
6. Backpressure.
7. Rate limiting.

### Data volume drops

Check:

1. Source availability.
2. Source filters.
3. Incremental watermark.
4. Validation/rejection counts.
5. Upstream changes.
6. Missing partitions/files.

### What not to do

Do not:

- create a metric for every record
- put run IDs or user IDs into labels
- rely on one metric to explain root cause
- alert on arbitrary values without understanding normal behavior
- discard logs because metrics exist
- create dozens of nearly identical metrics

## 19. Common Mistakes

### Mistake 1 — Measuring only failures

A pipeline can degrade without failing.

### Mistake 2 — No duration metrics

You cannot detect performance degradation reliably.

### Mistake 3 — No volume metrics

Silent data loss can look like successful execution.

### Mistake 4 — High-cardinality labels

Metric storage becomes expensive and difficult to query.

### Mistake 5 — Metrics without units

A duration of 500 is meaningless if you do not know whether it is milliseconds or seconds.

### Mistake 6 — Alerting on everything

Too many alerts become noise.

### Mistake 7 — Metrics replacing logs

Metrics tell you that something changed. Logs often help explain why.

## 20. Definition of Done

You are done when you can:

- explain why pipelines need metrics
- distinguish metrics from logs and run tracking
- implement counters
- implement gauges
- understand histograms
- instrument pipeline and task execution
- measure duration
- measure throughput
- measure data volume
- calculate failure and retry rates
- understand freshness measurements
- design useful metric dimensions
- identify high-cardinality problems
- intentionally increase failures
- intentionally slow a pipeline
- detect abnormal input volume
- investigate a metric anomaly using runs and logs
- recover from a metric-detected problem
- explain Prometheus, OpenTelemetry, and Grafana
- operate pipeline metrics using a production runbook

## 21. What You Learned

The central principle is:

> **Metrics turn pipeline behavior into measurable signals that reveal failures, degradation, performance changes, and abnormal data volumes over time.**

The operational investigation model is:

    METRIC
       ↓
    FIND AFFECTED RUN
       ↓
    SEARCH CORRELATED LOGS
       ↓
    IDENTIFY CAUSE
       ↓
    RECOVER
       ↓
    VERIFY METRICS RETURN TO NORMAL

A reliable pipeline needs all three:

    RUN TRACKING → What happened?
    LOGGING      → Why did it happen?
    METRICS      → How is it behaving over time?
