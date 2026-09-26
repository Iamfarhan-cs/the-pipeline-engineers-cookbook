# Recipe 29 — Pipeline Logging

A production pipeline needs more than application output. When something fails, an engineer must be able to reconstruct what happened from structured, useful, and correlated logs.

This recipe teaches how to design and implement pipeline logging so that execution can be investigated without exposing sensitive data or drowning operators in noise.

## 1. Problem Recognition

A beginner pipeline often uses:

    print("starting pipeline")
    print("loading data")
    print("done")

This breaks down in production when:

- several runs execute simultaneously
- multiple workers process tasks
- failures happen intermittently
- retries occur
- a task processes thousands of records
- operators need to correlate events with a specific run
- sensitive data must not appear in logs

A production log should answer:

- What happened?
- When did it happen?
- Which pipeline?
- Which run?
- Which task?
- Which attempt?
- What operation was occurring?
- Did it succeed or fail?
- How long did it take?
- What error class occurred?

## 2. Concept and Reasoning

Logging is an **event record**, not the authoritative pipeline state.

Recipe 28 gives you durable execution state.

Recipe 29 gives you the detailed evidence around that execution.

    RUN TRACKING
         ↓
    authoritative lifecycle
         +
    LOGGING
         ↓
    detailed execution evidence

### Structured vs unstructured logs

Unstructured:

    transform failed after reading 100 records

Structured:

    {
      "event": "task_failed",
      "pipeline": "payments",
      "run_id": "run-123",
      "task": "transform",
      "attempt": 2,
      "error_type": "ValidationError"
    }

Structured logs are easier to search, aggregate, and analyze.

## 3. Log Levels

Start with:

| Level | Use |
|---|---|
| DEBUG | Detailed troubleshooting information |
| INFO | Normal important execution events |
| WARNING | Unexpected but recoverable condition |
| ERROR | Operation failed and needs attention |
| CRITICAL | Severe failure affecting the system |

Do not log every record at INFO level.

Prefer meaningful lifecycle events.

## 4. What to Log

A useful pipeline log normally includes:

    timestamp
    log level
    event
    pipeline
    run_id
    task
    task_run_id
    attempt
    message
    duration
    rows_read
    rows_written
    error_type

Example:

    {
      "event": "task_finished",
      "pipeline": "daily_payments",
      "run_id": "run-123",
      "task": "load",
      "attempt": 1,
      "status": "SUCCESS",
      "rows_written": 9821,
      "duration_ms": 4210
    }

Do not put complete records into normal operational logs.

## 5. Correlation

The most important logging concept for a pipeline is correlation.

A single run should carry the same run ID through all tasks:

    pipeline
       ↓
    run_id
       ↓
    task
       ↓
    worker
       ↓
    external request

Example:

    run_id = run-123

Every related log event includes run-123.

This allows an operator to retrieve the complete execution trail.

For distributed work, also use:

    task_run_id
    trace_id

when appropriate.

## 6. Implementation

Use Python's standard logging library.

### 6.1 Basic logger

    import logging

    logger = logging.getLogger("pipeline")

    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s %(levelname)s %(name)s %(message)s"
    )

### 6.2 Structured context

Keep pipeline context explicit.

    def log_task_started(logger, pipeline, run_id,
                         task, attempt):
        logger.info(
            "task_started",
            extra={
                "pipeline": pipeline,
                "run_id": run_id,
                "task": task,
                "attempt": attempt,
            },
        )

In production, a structured JSON formatter is preferable so fields remain queryable.

## 7. Build a Small Structured Logger

For the learning implementation, create a simple structured logger:

    import json
    import logging
    from datetime import datetime, timezone

    class PipelineLogger:
        def __init__(self, logger, pipeline,
                     run_id, task=None, task_run_id=None):
            self.logger = logger
            self.pipeline = pipeline
            self.run_id = run_id
            self.task = task
            self.task_run_id = task_run_id

        def event(self, name, level=logging.INFO, **fields):
            record = {
                "timestamp": datetime.now(timezone.utc).isoformat(),
                "event": name,
                "pipeline": self.pipeline,
                "run_id": self.run_id,
                "task": self.task,
                "task_run_id": self.task_run_id,
                **fields,
            }

            self.logger.log(level, json.dumps(record))
    
This gives every event consistent context.

## 8. Lifecycle Logging

A useful task lifecycle is:

    task_started
         ↓
    processing
         ↓
    task_finished

or:

    task_started
         ↓
    task_failed

Example:

    logger.event(
        "task_started",
        attempt=1
    )

    logger.event(
        "task_finished",
        status="SUCCESS",
        rows_read=10000,
        rows_written=9950,
        duration_ms=1842
    )

Failure:

    logger.event(
        "task_failed",
        level=logging.ERROR,
        status="FAILED",
        error_type="ValidationError"
    )

Do not log secrets or raw request payloads.

## 9. Exception Logging

Always preserve useful exception context.

    try:
        transform()
    except Exception:
        logger.exception("transform failed")
        raise

For structured logging, add stable fields such as:

    error_type
    operation
    retryable
    attempt

Avoid putting sensitive exception payloads directly into logs.

## 10. Logging Data Quality and Pipeline Events

Useful events include:

    pipeline_started
    pipeline_finished
    task_started
    task_finished
    task_failed
    retry_started
    record_quarantined
    checkpoint_saved
    backfill_started
    backfill_finished
    reconciliation_started
    reconciliation_failed

Log state-changing operational events.

Do not log every successful record.

## 11. Sensitive Data

Logs are operational data and often have broad access.

Never casually log:

- passwords
- API keys
- access tokens
- authentication headers
- full payment details
- identity documents
- raw personal data
- database credentials
- secret connection strings

Instead log safe identifiers or metadata:

    record_id
    entity_type
    row_count
    error_type

Even identifiers should be reviewed for sensitivity.

## 12. Log Redaction

Implement a basic redaction mechanism for the learning exercise.

    SENSITIVE_KEYS = {
        "password",
        "token",
        "access_token",
        "api_key",
        "authorization",
        "secret",
    }

    def redact(fields):
        result = {}

        for key, value in fields.items():
            if key.lower() in SENSITIVE_KEYS:
                result[key] = "[REDACTED]"
            else:
                result[key] = value

        return result

Then:

    safe_fields = redact({
        "operation": "extract",
        "token": "secret-value",
    })

The resulting log must not contain the token.

Redaction should be defense-in-depth, not an excuse to log sensitive data.

## 13. Logging and Run Tracking

Recipe 28 records durable state.

Recipe 29 records evidence.

Example:

    pipeline_runs
        RUN-123 → FAILED

    logs
        task_started
        API request timed out
        retry_started
        retry_exhausted
        task_failed

The database tells you **what state the run ended in**.

Logs help explain **why**.

## 14. Testing

### Test 1 — Required fields

Every pipeline event should contain:

    timestamp
    event
    pipeline
    run_id

Task events should additionally contain task context.

### Test 2 — Correlation

Create multiple runs and verify their logs remain distinguishable.

### Test 3 — Failure logging

Force a task failure and verify:

    event = task_failed
    error_type is present
    run_id is present

### Test 4 — Redaction

Log a fake secret and verify the actual secret does not appear in the output.

### Test 5 — Exception context

Raise an exception and verify the error event contains useful failure context without sensitive payloads.

### Test 6 — Log level

DEBUG events should not appear when the configured level is INFO.

### Test 7 — No record-level noise

Verify normal processing does not emit one INFO event for every record.

## 15. Observability

Logging is itself observable.

Track:

    logs_generated_total
    log_errors_total
    log_redactions_total

Operationally monitor:

- logging failures
- ingestion delays
- excessive log volume
- repeated error messages
- missing correlation fields
- sensitive-data detection

High log volume can become a production problem.

## 16. Intentional Failure

### Failure drill 1 — Remove run ID

Generate an event without run_id.

Try to investigate a specific execution.

The exercise should demonstrate how quickly debugging becomes difficult.

### Failure drill 2 — Log a secret

Intentionally pass a fake API key.

Verify the redaction mechanism removes it before output.

### Failure drill 3 — Logging failure

Make the log destination unavailable.

Ask:

> Should pipeline processing stop because logging failed?

For most pipelines, ordinary logging should not become a single point of failure for the data path.

Define the desired behavior explicitly.

### Failure drill 4 — Log storm

Generate thousands of unnecessary INFO events.

Observe:

- increased log volume
- storage growth
- slower investigation
- potential logging cost

Then reduce logging to meaningful lifecycle events.

## 17. Recovery

When logs are missing:

1. Check whether the pipeline state is still available from run tracking.
2. Check worker/service health.
3. Check log collector or destination health.
4. Check filtering and log-level configuration.
5. Check whether the application actually emitted the event.
6. Restore logging without changing pipeline correctness behavior.
7. Use run metadata and metrics to reconstruct missing information where possible.

Never invent historical events just to make the logs look complete.

## 18. Production Logging Design

A practical production pipeline should separate:

    APPLICATION
       ↓
    STRUCTURED LOG EVENT
       ↓
    LOG COLLECTION
       ↓
    CENTRAL LOG STORE
       ↓
    SEARCH / ALERTING

The application should not need to know the internals of the final log storage system.

Important properties:

- structured fields
- consistent timestamps
- correlation IDs
- controlled verbosity
- redaction
- retention policy
- searchable errors

## 19. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Python logging** | Standard library foundation for application logging, handlers, levels and formatters. |
| **OpenTelemetry** | Correlates logs, traces and other telemetry through shared context. |
| **Grafana Loki** | Log aggregation and querying system commonly used with Grafana. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding structured logging and correlation.

## 20. Production Runbook

### A pipeline failed and logs are needed

Check:

1. run_id
2. pipeline name
3. task name
4. task_run_id
5. attempt
6. error type
7. surrounding events
8. run-tracking state

### Logs suddenly disappear

Check:

1. application logging configuration
2. log level
3. collector health
4. destination health
5. network connectivity
6. filtering rules
7. disk/storage limits

### Log volume suddenly increases

Check:

1. repeated retry loops
2. exception loops
3. record-level INFO logging
4. deployment changes
5. log level changes
6. duplicate workers

### Sensitive data appears in logs

Immediately:

1. stop further emission if necessary
2. identify affected log stream
3. determine retention/exposure
4. remove the logging source
5. rotate exposed credentials if applicable
6. follow the organization's incident procedure

Do not assume that deleting the application log line removes already stored copies.

### What not to do

Do not:

- use print statements as the production logging strategy
- log secrets
- log entire records for convenience
- omit run IDs
- use inconsistent field names
- emit huge record-level INFO streams
- make the data pipeline depend completely on log delivery
- treat logs as the authoritative pipeline state

## 21. Common Mistakes

### Mistake 1 — Logging without correlation

Messages cannot be tied to a particular execution.

### Mistake 2 — Logging everything

Noise hides important failures.

### Mistake 3 — Logging sensitive data

Logs can have broad access and long retention.

### Mistake 4 — No structured fields

Operators are forced to parse message strings.

### Mistake 5 — Inconsistent event names

Search and aggregation become difficult.

### Mistake 6 — Logs as state

A log saying "load completed" is not the same as durable execution state.

### Mistake 7 — Logging failure stops the pipeline

Operational logging should normally degrade independently from business processing.

## 22. Definition of Done

You are done when you can:

- explain why production pipelines need structured logs
- distinguish logs from durable run state
- define useful log levels
- design structured pipeline events
- include run and task correlation identifiers
- implement structured logging in Python
- log task lifecycle events
- capture useful exception context
- redact sensitive fields
- test log correlation
- test secret redaction
- intentionally create a log storm
- diagnose missing logs
- understand logging failure as a separate operational problem
- design a centralized logging flow
- explain Python logging, OpenTelemetry, and Loki
- operate pipeline logging using a runbook

## 23. What You Learned

The central principle is:

> **Logs should provide enough structured evidence to reconstruct what happened in a pipeline without exposing sensitive data or overwhelming operators with noise.**

The operational model is:

    PIPELINE RUN
         ↓
    STRUCTURED EVENTS
         ↓
    CORRELATION
         ↓
    LOG COLLECTION
         ↓
    CENTRAL SEARCH
         ↓
    DIAGNOSIS

Run tracking tells you the authoritative execution state.

Logging gives you the detailed evidence needed to understand that state.
