# Task 18 — Structured Logging & Error Taxonomy

## 1. Task Overview

Task 18 adds a production-safe structured logging and error taxonomy layer to the payment telemetry pipeline.

The implementation makes pipeline failures and operational problems consistently classified, machine-readable, searchable, and traceable to the pipeline run and processing mode when those identifiers are available.

The implementation builds on the existing Python pipeline and database lifecycle. It does not introduce a separate logging architecture.

## 2. Technical Objective

The objectives are:

- Define a small stable taxonomy for pipeline errors.
- Give each taxonomy entry a deterministic machine-readable error code.
- Represent classified pipeline errors without destroying their original cause.
- Emit JSON structured logs through the standard Python logging package.
- Propagate run and processing-mode context through operational logs.
- Log meaningful pipeline boundaries without logging every telemetry event.
- Keep logs free from telemetry payloads, secrets, credentials, and PII.
- Preserve the existing pipeline transaction and failure behavior.
- Provide a foundation for future operational monitoring and retry decisions.

Retry and recovery policy are explicitly outside Task 18.

## 3. Why Structured Logging Is Needed

Before Task 18, failures were mainly represented by Python exceptions and, for run failures, the existing `error_code` and `error_message` fields in `telemetry_pipeline_run`.

Those records remained sufficient for persisted state but did not provide a common machine-readable operational log format.

Task 18 separates two concerns:

1. Existing database records remain authoritative for pipeline, run, batch, and quarantine state.
2. Structured logs provide operational context for diagnosing what happened and where it happened.

The implementation therefore does not create a second accounting or event-history system.

## 4. Implementation Sequence

1. Checked repository status, branch, and recent commit history.
2. Reviewed the Task 10–17 pipeline architecture and relevant documentation.
3. Inspected extraction, validation, staging, quarantine, accounting, checkpoint, run, runner, and metrics code.
4. Confirmed no established structured logging framework existed.
5. Reviewed the existing database migrations and run failure fields.
6. Defined the stable Task 18 error taxonomy and codes.
7. Added the application-level classified error representation.
8. Added JSON structured logging and scoped context propagation.
9. Integrated classification at meaningful normal-pipeline failure boundaries.
10. Added run-level logging for normal, replay, and backfill execution.
11. Added configuration and database connection logging.
12. Added focused tests.
13. Ran focused tests and the full regression suite.
14. Ran Python compilation and `git diff --check`.
15. Created the Task 18 documentation.

## 5. Error Taxonomy

| Category | Operational meaning |
|---|---|
| EXTRACTION | Source read or extraction failure |
| VALIDATION | Validation-layer execution failure |
| STAGING | Valid-event staging persistence failure |
| QUARANTINE | Invalid-event quarantine persistence/handling failure |
| DATABASE | Database connection or persistence failure |
| CONFIGURATION | Invalid or incomplete runtime configuration |
| PIPELINE | Pipeline/run execution failure |
| INTERNAL | Unexpected failure without a more specific category |

The taxonomy is intentionally small and classifies operational meaning rather than exception-message text.

## 6. Error Codes

| Code | Category |
|---|---|
| TEL-EXTRACT-001 | EXTRACTION |
| TEL-VALID-001 | VALIDATION |
| TEL-STAGE-001 | STAGING |
| TEL-QUAR-001 | QUARANTINE |
| TEL-DB-001 | DATABASE |
| TEL-CONFIG-001 | CONFIGURATION |
| TEL-PIPE-001 | PIPELINE |
| TEL-INTERNAL-001 | INTERNAL |

Codes are centrally defined in `src/payment_telemetry/errors.py` and are the stable machine-readable identifiers.

## 7. Structured Log Fields

Logs are emitted as JSON through Python's standard logging package.

Common fields are:

- `timestamp`
- `level`
- `logger`
- `message`
- `error_code`
- `error_category`
- `pipeline_name`
- `run_id`
- `batch_id`
- `event_id`
- `processing_mode`
- `source_position`
- `exception_type`

Batch-level operational context may also include `extracted_count`, `valid_count`, `invalid_count`, `staged_count`, and `quarantined_count`.

Only explicitly allow-listed context fields are serialized.

## 8. Context Propagation

Logging context uses Python context variables.

The scoped `log_context` helper propagates `pipeline_name`, `run_id`, `processing_mode`, and other permitted identifiers through nested pipeline operations.

The context is restored when the scope exits, preventing leakage between unrelated operations.

Task 18 never creates a new run, batch, or event identifier. It only records identifiers already available at the logging point.

## 9. Pipeline Integration

### Extraction

Normal incremental extraction failures are logged as `TEL-EXTRACT-001`.

### Validation

Validation execution failures are logged as `TEL-VALID-001`. Invalid events are summarized at batch level rather than logged individually.

### Staging

Staging persistence failures are logged as `TEL-STAGE-001`.

### Quarantine

Quarantine persistence failures are logged as `TEL-QUAR-001`.

### Accounting

Batch accounting persistence failures are logged as `TEL-DB-001`.

### Checkpoint

Checkpoint persistence failures are logged as `TEL-DB-001`.

### Pipeline execution

Normal, replay, and backfill run failures are logged at the runner boundary as `TEL-PIPE-001`.

Successful run completion is logged at the low-volume run boundary.

### Configuration

Incomplete database configuration is logged as `TEL-CONFIG-001` without logging configuration values.

### Database connection

PostgreSQL connection failures are logged as `TEL-DB-001`.

## 10. Error Handling Boundaries

The existing failure behavior is preserved.

1. A lower-level operation logs the relevant operational classification.
2. The original exception is preserved and re-raised.
3. Existing transaction behavior determines rollback or commit.
4. The runner persists the existing run failure state.
5. The runner emits a run-level failure record.
6. Validation outcomes remain validation/quarantine outcomes rather than being converted into run failures.
7. Unexpected exceptions are classified as INTERNAL when no more specific classification is supplied.

## 11. Database / Migration Changes

No Task 18 database migration was required.

The existing schema already contains the relevant authoritative state:

- `telemetry_pipeline_run` stores run status and existing failure code/message.
- `telemetry_pipeline_batch` stores batch accounting and processing mode.
- `telemetry_event_quarantine` stores quarantine lifecycle state.
- `telemetry_pipeline_checkpoint` stores the normal incremental cursor.

Persisting every log entry in PostgreSQL would duplicate operational information and add unnecessary database write volume.

Task 18 therefore keeps structured logs in the application/observability layer.

No new table, column, index, or migration was added.

## 12. Security and Privacy

The logging layer deliberately avoids telemetry payload logging.

The implementation does not log:

- PII
- document contents
- payment payloads
- secrets
- credentials
- authentication tokens
- free-form sensitive user data
- entire telemetry event objects

Operational identifiers such as `run_id` or `event_id` may be logged when already available.

Structured context is allow-listed. Arbitrary keys such as `secret`, `token`, or `payment_payload` are not serialized.

Task 18 also avoids per-event success logging, keeping the design suitable for high-volume processing.

## 13. Tests

New focused tests:

- `tests/test_errors.py`
- `tests/test_logging.py`

Coverage includes:

1. Stable and unique error codes.
2. Expected taxonomy categories.
3. `PipelineError` representation.
4. Exception cause preservation.
5. Deterministic classification.
6. Structured JSON fields.
7. Run/event/processing-mode context propagation.
8. Context restoration.
9. Sensitive context exclusion.
10. Existing pipeline and runner regression behavior.

The existing Task 10–17 tests remain in the suite.

## 14. Validation Results

Focused Task 18 plus pipeline/runner tests:

**29 passed**

Full regression suite:

**101 passed in 0.44s**

Python compilation:

**Passed**

Command:

~~~bash
python -m compileall -q src tests
~~~

Git whitespace validation:

**Passed**

Command:

~~~bash
git diff --check
~~~

No live PostgreSQL result is claimed by this validation pass.

## 15. Failure Behavior

Task 18 does not change normal failure semantics.

If extraction, staging, quarantine, accounting, checkpoint persistence, or another pipeline operation raises an exception:

- the exception remains an exception;
- the original exception is preserved;
- the relevant structured classification is emitted;
- the runner can persist the existing run failure state;
- the exception is re-raised to the caller.

`PipelineError` supports explicit classification and normal Python exception chaining.

Unexpected exceptions are classified as `TEL-INTERNAL-001` by the generic classifier when no specific classification is supplied.

## 16. Design Decisions

### Standard Python logging

No established structured logging framework existed, so the implementation uses Python's built-in logging package.

### JSON records

JSON makes records machine-readable and suitable for log aggregation and searching.

### Application/observability layer

Existing PostgreSQL tables are authoritative for pipeline state. Log persistence would duplicate that state and increase database write volume.

### Small stable taxonomy

A small taxonomy is easier to operate than a large hierarchy of highly specific error types.

### Preserve original exceptions

Task 18 does not replace an original failure with an opaque logging exception. Chaining remains available.

### Boundary logging

The pipeline may process very high event volumes. The implementation therefore logs at run, batch, and failure boundaries instead of logging every event.

### No retry policy

Classification is useful input for future recovery decisions, but Task 18 does not decide whether an error is retryable.

## 17. What Task 18 Does NOT Implement

- retry policy
- automatic recovery
- retry backoff
- retry scheduling
- dead-letter lifecycle changes
- new DQ accounting
- new pipeline-run persistence
- new batch or event identifiers
- PostgreSQL log persistence
- a separate logging service
- Grafana dashboards
- OpenTelemetry log export
- alert rules
- frontend telemetry changes
- frontend/backend product discovery
- changes to existing business processing semantics

The existing Task 10–17 processing architecture remains the foundation.

## 18. Relationship to Task 19 — Retry & Recovery

Task 18 provides stable error categories and codes that future recovery logic can consume.

Task 19 is responsible for retry and recovery behavior.

Task 19 may use Task 18 classifications to distinguish recoverable operational failures from validation or terminal failures.

Task 18 itself does not decide which errors are retryable, retry an operation, change retry counts, schedule recovery, introduce backoff, or automatically reprocess failed work.

This separation keeps classification and observability independent from recovery policy.

## 19. Implementation Files

- `src/payment_telemetry/errors.py`
- `src/payment_telemetry/logging.py`
- `src/payment_telemetry/config.py`
- `src/payment_telemetry/db.py`
- `src/payment_telemetry/pipeline.py`
- `src/payment_telemetry/runner.py`
- `tests/test_errors.py`
- `tests/test_logging.py`
- `docs/task_18_structured_logging_error_taxonomy.md`

No Task 18 database migration was required.

The pre-existing `../docker-compose.dev.yml` modification was not reverted, modified, staged, committed, or included in the Task 18 work.

## Final Result

Task 18 adds structured logging and a stable error taxonomy without creating a second persistence or accounting system.