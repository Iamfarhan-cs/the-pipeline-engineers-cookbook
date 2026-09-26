# Recipe 8 — Pipeline Observability

A pipeline can be running and still be broken.

A worker may be alive while no new data is being processed. A database connection may work while records are failing validation. A job may finish successfully while the output is incomplete.

Without observability, these problems are difficult to see.

Observability gives us information about what the pipeline is doing and helps us understand why it is doing it.

A practical pipeline usually needs several types of signals:

- Logs
- Metrics
- Traces
- Data quality signals
- Processing state
- Alerts

This chapter explains how these signals work together.

---

# What Is Observability?

Observability is the ability to understand the internal behavior of a system from the information it produces.

For a data pipeline, that means being able to answer questions such as:

- Is the pipeline running?
- Is data arriving?
- How much data was processed?
- How long did processing take?
- How many records failed?
- Where did the failure happen?
- Which dependency caused the problem?
- Are retries increasing?
- Is the pipeline falling behind?
- Is the output fresh?

Monitoring usually focuses on known signals and known conditions.

Observability goes further by helping engineers investigate unexpected behavior.

In practice, the two ideas work together.

---

# Why Observability Matters

Imagine a pipeline that normally processes 100,000 events every hour.

At 10:00 it processes 99,000.

At 11:00 it processes 98,000.

At 12:00 it processes 12,000.

The application may still be running.

No container crashed.

No process reported an exception.

But something is clearly wrong.

Without data-related signals, the problem may remain hidden.

Observability helps turn this:

~~~text
Something seems wrong
~~~

into this:

~~~text
Input volume dropped
Database latency increased
Retry count increased
Processing lag increased
Output freshness exceeded threshold
~~~

Now the investigation has a starting point.

---

# The Four Main Questions

A useful observability model for pipelines is:

1. What happened?
2. Where did it happen?
3. Why did it happen?
4. What is the current state?

Logs help answer what happened.

Metrics help show how often and how much something happened.

Traces help connect work across components.

Processing state and data quality signals show the condition of the pipeline and its data.

---

# Logs

A log is a record of something that happened.

For example:

~~~text
Started processing batch 123
~~~

or:

~~~text
Failed to connect to database
~~~

Logs are useful when investigating individual events or failures.

---

# What Should a Pipeline Log?

Useful pipeline logs may include:

- Pipeline name
- Job or run identifier
- Processing stage
- Event or batch identifier
- Operation
- Status
- Error category
- Timestamp
- Duration
- Retry attempt
- Dependency name

The exact fields depend on the system.

The goal is to provide enough context to investigate a problem.

---

# Structured Logging

Plain text logs can be difficult to search.

For example:

~~~text
Failed to process event 123 because database connection timed out
~~~

A structured log can represent the same information as fields:

~~~json
{
  "event_id": "123",
  "operation": "process_event",
  "status": "failed",
  "error_type": "database_timeout"
}
~~~

**GENERIC EXAMPLE**

Structured logs are easier for machines to filter, aggregate, and search.

---

# Log Levels

Applications often use different log levels.

Common examples are:

- DEBUG
- INFO
- WARNING
- ERROR
- CRITICAL

An informational message might say:

~~~text
INFO: batch processing started
~~~

A warning might say:

~~~text
WARNING: retry attempt 3
~~~

An error might say:

~~~text
ERROR: batch processing failed
~~~

The exact logging framework and level names depend on the application.

Do not log everything at the highest severity.

Otherwise important failures become difficult to distinguish from normal messages.

---

# Logs and Sensitive Data

Pipeline logs can accidentally expose sensitive information.

Do not automatically log:

- Passwords
- Authentication tokens
- Full payment information
- Personal documents
- Secrets
- Unnecessary personal data
- Full request bodies when they contain sensitive fields

Observability must respect the same privacy boundaries as the rest of the pipeline.

If an identifier is enough for debugging, do not log the entire payload.

---

# Metrics

A metric is a numerical measurement collected over time.

Examples include:

- Records processed
- Records failed
- Processing duration
- Queue depth
- Retry count
- API request count
- Database latency
- Consumer lag

Metrics are especially useful for seeing trends.

---

# Counter Metrics

A counter represents a value that increases as events occur.

For example:

~~~text
records_processed_total
~~~

Conceptually:

~~~text
10:00 -> 10,000
11:00 -> 20,000
12:00 -> 30,000
~~~

The counter tells us that processing continued.

Other examples:

- records_failed_total
- retry_attempts_total
- messages_received_total

These are generic metric names.

---

# Gauge Metrics

A gauge represents a value that can increase or decrease.

Examples include:

- Queue depth
- Active workers
- Current lag
- Memory usage
- Number of pending records

For example:

~~~text
Queue depth
10:00 -> 500
10:10 -> 700
10:20 -> 1,200
~~~

A growing queue can indicate that processing is falling behind.

---

# Histogram Metrics

A histogram can help measure distributions such as processing duration.

Suppose one operation usually takes between 50 and 200 milliseconds.

Suddenly many operations take several seconds.

A simple average may hide useful information.

Latency distributions can provide more detail.

Histograms are useful for measurements such as:

- API latency
- Database query duration
- Event processing time
- Batch duration

---

# Rates

A raw counter is not always enough.

Engineers often care about the rate at which something is happening.

For example:

~~~text
10,000 records processed
~~~

does not tell us whether that happened in:

- One second
- One minute
- One hour

Rate measurements provide more context.

---

# Metrics for Pipeline Health

A useful pipeline may measure:

| Signal | What it tells us |
|---|---|
| Input records | How much data arrived |
| Output records | How much data was produced |
| Failed records | How much processing failed |
| Processing duration | How long work takes |
| Retry count | How often retries happen |
| Queue depth | How much work is waiting |
| Lag | How far processing is behind |
| Freshness | How old the latest output is |

These signals should be selected based on the actual pipeline.

Do not collect hundreds of metrics just because the monitoring system supports them.

---

# Traces

A trace follows a unit of work across multiple components.

Consider:

~~~text
API
 |
 v
Queue
 |
 v
Worker
 |
 v
PostgreSQL
~~~

A trace can connect these operations as part of the same request or processing flow.

This helps answer:

> Where did the time go?

> Which component failed?

> Which downstream operation was involved?

---

# Spans

A trace is usually made up of smaller operations called spans.

For example:

~~~text
Trace: process_event
  |
  +-- fetch_event
  |
  +-- validate_event
  |
  +-- write_database
  |
  +-- publish_result
~~~

Each span can contain information such as:

- Start time
- End time
- Duration
- Operation name
- Status
- Related attributes

This makes it easier to locate slow or failing stages.

---

# Trace Correlation

Logs, metrics, and traces become much more useful when they can be connected.

For example:

~~~text
Trace ID
   |
   +---- API log
   +---- worker log
   +---- database span
   +---- processing metric
~~~

A correlation identifier can help an engineer move from one signal to another.

This is especially useful in systems with multiple services or workers.

---

# Processing State

Not every useful signal needs to be a log or metric.

Processing state is also important.

For example:

~~~text
RECEIVED
   |
   v
PROCESSING
   |
   +----> COMPLETED
   |
   +----> RETRY_PENDING
   |
   +----> FAILED
~~~

An operator can inspect the state of work and understand whether records are progressing.

This is particularly useful for recovery.

---

# Data Observability

System observability tells us about the software.

Data observability tells us about the data moving through it.

Useful data signals include:

- Record counts
- Null counts
- Duplicate counts
- Freshness
- Schema changes
- Distribution changes
- Reconciliation differences
- Failed validation counts

Consider this situation:

~~~text
Application health: OK
Database health: OK
Worker health: OK

Data quality: FAILED
~~~

The system is running.

The data is not healthy.

Both perspectives are necessary.

---

# Pipeline Lag

Lag measures how far processing is behind the source or expected processing point.

For example:

~~~text
Latest source event: 12:00
Latest processed event: 11:45

Lag: approximately 15 minutes
~~~

Lag is especially important for streaming systems.

It can also be useful in scheduled or batch pipelines.

---

# Freshness vs Lag

Freshness and lag are related but not always identical.

Freshness often asks:

> How old is the latest available data?

Lag often asks:

> How far behind is processing compared with the source or expected position?

A pipeline can have good freshness for one dataset while having significant lag in another processing path.

Define the exact measurement before building an alert.

---

# Heartbeats

A heartbeat is a signal that a component is alive or making progress.

For example:

~~~text
Worker started
Worker still processing
Worker still processing
Worker still processing
~~~

A heartbeat alone does not prove that useful work is happening.

A worker can be alive but stuck.

That is why heartbeat signals should be combined with progress metrics.

---

# Liveness vs Progress

These are different questions.

**Liveness:** Is the process alive?

**Progress:** Is the process actually moving work forward?

Consider:

~~~text
Worker: alive
Processed records: unchanged for 30 minutes
Queue depth: increasing
~~~

The worker is alive.

The pipeline is unhealthy.

Progress signals are therefore important.

---

# Alerts

An alert is a notification triggered when a defined condition requires attention.

Examples:

- Pipeline has failed
- Retry count is unusually high
- Queue depth is growing
- Data is stale
- Required quality check failed
- Consumer lag exceeds a defined threshold

An alert should lead to an action.

If nobody knows what to do after receiving an alert, the alert is probably not well designed.

---

# Good Alerts

A useful alert should answer:

- What is wrong?
- Which pipeline is affected?
- When did it start?
- How serious is it?
- What signal triggered the alert?
- What should the operator investigate?

For example:

~~~text
BAD:
Pipeline problem

BETTER:
Payment event pipeline freshness exceeded the configured threshold.
~~~

The second message provides a much better starting point.

---

# Alert Fatigue

Too many alerts create another problem.

If engineers receive alerts for every small variation, they may start ignoring them.

This is called alert fatigue.

Alerts should therefore be based on meaningful conditions.

Possible approaches include:

- Thresholds
- Time windows
- Consecutive failures
- Rate changes
- Severity levels

The exact rule depends on the system.

---

# Symptoms vs Causes

An alert often identifies a symptom rather than the root cause.

For example:

~~~text
Alert:
Queue depth is increasing

Possible cause:
Database writes are slow

Possible deeper cause:
Database connection pool is exhausted

Possible root cause:
Unexpected query behavior after a deployment
~~~

Observability should help engineers move from symptom to cause.

It does not automatically determine the root cause.

---

# A Simple Observability Flow

Consider a pipeline:

~~~text
API
 |
 v
Ingestion
 |
 v
Validation
 |
 v
PostgreSQL
 |
 v
Warehouse
~~~

Useful signals could be:

~~~text
API
  -> request count
  -> error rate
  -> latency

Ingestion
  -> records received
  -> retries
  -> failures

Validation
  -> valid records
  -> invalid records

PostgreSQL
  -> query latency
  -> connection failures

Warehouse
  -> load count
  -> freshness
  -> quality status
~~~

This gives engineers several ways to understand what is happening.

---

# Repository Investigation for Observability

When investigating an existing repository, do not assume observability is implemented in one place.

Search for:

- Logger setup
- Logging calls
- Metrics definitions
- Metrics exporters
- Trace configuration
- Middleware
- Instrumentation
- Health endpoints
- Processing status tables
- Error handling
- Alert configuration
- Dashboard configuration
- Docker Compose monitoring services
- CI checks related to observability

The exact file names and tools depend on the repository.

This is why repository investigation comes before implementation.

---

# Files to Inspect

**GENERIC INVESTIGATION CHECKLIST**

Depending on the repository, inspect:

~~~text
src/
  logging/
  metrics/
  tracing/
  pipeline/
  workers/

tests/
  test_logging/
  test_metrics/
  test_pipeline/

config/
docker-compose.yml
CI configuration
monitoring configuration
~~~

These are investigation examples, not claims about a specific repository.

---

# Instrumentation Boundaries

Do not instrument every line of code.

Useful boundaries are often:

- API request
- Batch execution
- Pipeline stage
- External API call
- Database operation
- Queue publish
- Queue consume
- Validation stage
- Warehouse load

These boundaries provide meaningful information without creating unnecessary noise.

---

# High-Cardinality Data

Metrics can become expensive or difficult to use when labels have too many unique values.

For example, a metric label containing a unique event ID could create a very large number of time series.

That is usually a poor metric design.

Use metrics for dimensions that have controlled values.

Put detailed identifiers in logs or traces when appropriate.

---

# Logs, Metrics, and Traces Together

Each signal answers different questions.

| Signal | Best for |
|---|---|
| Logs | Detailed events and failures |
| Metrics | Trends, counts, rates, thresholds |
| Traces | Request and processing paths |
| Data quality | Health of the data |
| Processing state | Current work status |

The strongest systems combine them.

For example:

~~~text
Alert
  |
  v
Metric shows failure rate increased
  |
  v
Trace shows database stage is slow
  |
  v
Logs show connection timeout
  |
  v
Processing state shows records are retrying
~~~

This is the kind of investigation path observability should support.

---

# Testing Observability

Observability code should also be tested.

## Test 1 — Successful processing

Verify that expected success signals are produced.

## Test 2 — Processing failure

Verify that the failure is logged and the appropriate metric or state is updated.

## Test 3 — Retry

Verify that retry attempts are visible.

## Test 4 — Final failure

Verify that exhausted retries produce the expected failure signal.

## Test 5 — Trace propagation

Where tracing is implemented, verify that correlation information survives across the expected processing boundaries.

## Test 6 — Sensitive data

Verify that logs and telemetry do not expose prohibited sensitive fields.

Observability should be reliable without becoming a data-leak source.

---

# Common Mistakes

## Mistake 1: Logging everything

Large volumes of useless logs make important information harder to find.

Log useful events.

## Mistake 2: Logging sensitive data

Debugging is not a reason to expose secrets or personal information.

## Mistake 3: Only monitoring infrastructure

CPU and memory can look healthy while the data pipeline is broken.

Monitor data signals too.

## Mistake 4: Only using logs

Logs are useful, but trends and distributions are easier to understand with metrics.

## Mistake 5: No correlation

If logs, traces, and processing state cannot be connected, investigations become slower.

## Mistake 6: Alerting on everything

Too many alerts create alert fatigue.

## Mistake 7: Measuring activity instead of progress

A worker can produce logs while making no useful progress.

Measure completed work, lag, and output.

## Mistake 8: No recovery information

Observability should help explain what happened and what can be recovered.

---

# Production Considerations

A production pipeline should define its observability requirements.

At minimum, consider:

- Pipeline execution status
- Input volume
- Output volume
- Failure count
- Retry count
- Processing duration
- Processing lag
- Data freshness
- Data quality failures
- Dependency failures
- Current processing state
- Error logs
- Correlation identifiers
- Alerting

The exact list depends on the pipeline.

Do not build observability as a separate activity after the pipeline is finished.

Observability is part of the pipeline design.

---

# Practical Investigation Example

Imagine an operator receives this alert:

~~~text
Pipeline freshness exceeded threshold
~~~

The investigation can follow a structured path.

### Step 1 — Check pipeline state

Is the pipeline running?

### Step 2 — Check input volume

Did data arrive from the source?

### Step 3 — Check processing progress

Are records being processed?

### Step 4 — Check failures

Did error or retry counts increase?

### Step 5 — Check dependencies

Is the database, API, queue, or other dependency slow or unavailable?

### Step 6 — Check data quality

Are records being rejected?

### Step 7 — Check recent changes

Was there a deployment, configuration change, schema change, or infrastructure event?

This is a generic troubleshooting flow.

It turns observability data into an investigation process.

---

# Observability and Recovery

Observability becomes especially valuable during recovery.

Suppose a pipeline failed halfway through a batch.

The operator needs to know:

- Which batch failed?
- How many records were processed?
- Which records failed?
- Was the transaction committed?
- Are retries pending?
- Was data partially written?
- Can the batch be safely replayed?

Good observability provides evidence for these decisions.

Without that evidence, recovery becomes guesswork.

---

# Observability and Idempotency

Idempotency also improves observability.

If a record is retried, the system should be able to show:

~~~text
event_id = 123
attempt = 3
status = retrying
~~~

This helps operators understand whether the same logical work is being repeated safely.

Again, the exact fields depend on the implementation.

---

# Observability and Data Quality

Chapter 7 showed that a pipeline can be technically healthy while producing bad data.

Observability gives us a way to expose those problems.

For example:

~~~text
Pipeline status: RUNNING
Records processed: 100,000
Records invalid: 18,000
Freshness: 45 minutes behind
~~~

The application is running.

The data pipeline is not healthy.

This is why data quality signals should be part of pipeline observability.

---

# Practical Checklist

Before calling a pipeline observable, ask:

- [ ] Can I tell whether the pipeline is running?
- [ ] Can I tell whether it is making progress?
- [ ] Can I see how much data arrived?
- [ ] Can I see how much data was processed?
- [ ] Can I see how much data failed?
- [ ] Can I see retry behavior?
- [ ] Can I measure processing duration?
- [ ] Can I measure lag?
- [ ] Can I measure freshness?
- [ ] Can I detect important data quality failures?
- [ ] Can I identify the failing stage?
- [ ] Can I identify failing dependencies?
- [ ] Can I connect related logs and traces?
- [ ] Are sensitive fields protected?
- [ ] Are alerts actionable?
- [ ] Are observability signals tested?
- [ ] Can the signals help with recovery?

---

# What You Learned

In this recipe, you learned:

- Observability helps engineers understand pipeline behavior.
- Logs provide detailed event information.
- Metrics provide numerical signals and trends.
- Traces connect work across system boundaries.
- Processing state shows the current condition of work.
- Data quality signals show the health of the data itself.
- Liveness does not necessarily mean progress.
- Lag and freshness are important pipeline signals.
- Alerts should be actionable.
- Too many alerts create alert fatigue.
- Observability should protect sensitive information.
- Logs, metrics, traces, and data signals work best together.
- Observability should help investigation and recovery.

The main lesson is:

**A production pipeline should not only process data. It should also tell you what it is doing, whether it is making progress, what is failing, and where to look when something goes wrong.**

---
