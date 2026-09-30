# Task 19 — Retry & Recovery Engine

## 1. Task Overview

Task 19 adds bounded retry and recovery for telemetry events already present in the existing quarantine lifecycle.

The implementation reuses the existing quarantine table and lifecycle statuses:

- `QUARANTINED`
- `RETRY_PENDING`
- `REPROCESSING`
- `RESOLVED`
- `UNRESOLVABLE`

No second quarantine or dead-letter system was introduced.

## 2. Problem Being Solved

Before Task 19, the project had quarantine lifecycle states but no production-oriented retry worker with explicit classification, bounded attempts, persistent backoff, concurrency protection, and retry outcome logging.

Task 19 provides those controls while preserving the existing staging idempotency constraint and DQ metrics model.

## 3. Existing Architecture Used

The implementation builds on:

- `quarantine.py` for the existing quarantine record and lifecycle.
- `staging.py` for idempotent telemetry staging.
- `validation.py` for deterministic event validation.
- `errors.py` for Task 18 error taxonomy.
- `logging.py` for structured JSON operational logs.
- `metrics.py` for existing retry and quarantine metrics.
- PostgreSQL transactions and row-level locking for concurrency control.

The retry engine is implemented in `src/payment_telemetry/retry.py`.

## 4. Implementation Sequence

1. Added persistent retry scheduling/error metadata.
2. Defined a bounded retry policy.
3. Added explicit retry exception classification.
4. Added `QUARANTINED → RETRY_PENDING` scheduling.
5. Added due-event claiming with PostgreSQL row locking.
6. Reused existing event validation.
7. Reused existing idempotent staging.
8. Added retry outcome transitions.
9. Added structured retry logging.
10. Added focused retry tests and regression coverage.

## 5. Retry Classification

### Retryable

- PostgreSQL operational failures.
- PostgreSQL interface/connection failures.
- `ConnectionError`.
- `TimeoutError`.
- `OSError`.
- Existing `PipelineError` values classified as DATABASE or STAGING.

### Non-retryable

- Deterministic validation failures.
- Configuration errors.
- Validation `PipelineError`.
- Unknown/internal exceptions.
- Other existing taxonomy categories not explicitly designated retryable.

The implementation does not treat every exception as retryable.

The existing Task 18 `ValueError → TEL-CONFIG-001` behavior is not changed. Retry classification uses the operational context and does not make ValueError a generic retryable failure.

## 6. Retry Policy

Configuration is environment-based:

- `TELEMETRY_RETRY_MAX_ATTEMPTS`
- `TELEMETRY_RETRY_BASE_DELAY_SECONDS`
- `TELEMETRY_RETRY_MAX_DELAY_SECONDS`

Defaults:

- Maximum attempts: 3
- Base delay: 1 second
- Maximum delay: 60 seconds

Backoff is exponential and capped:

- attempt 1: base delay
- attempt 2: 2 × base delay
- attempt 3+: capped at maximum delay

The retry worker persists `next_retry_at` instead of sleeping inside the database transaction.

No unbounded retry loop exists.

## 7. State Transitions

Retry path:

`QUARANTINED → RETRY_PENDING → REPROCESSING`

Successful processing:

`REPROCESSING → RESOLVED`

Retryable failure before exhaustion:

`REPROCESSING → RETRY_PENDING`

Non-retryable failure:

`REPROCESSING → UNRESOLVABLE`

Retry exhaustion:

`REPROCESSING → UNRESOLVABLE`

Terminal states are not eligible for retry processing.

The existing lifecycle transition helper no longer increments `retry_count` merely because a status changed.

## 8. Transaction Safety

A retry attempt is processed inside an explicit transaction.

The retry engine:

1. Claims the quarantine record.
2. Locks it with PostgreSQL row-level locking.
3. Moves it to REPROCESSING.
4. Validates the event.
5. Performs staging inside a nested transaction/savepoint.
6. Records the retry outcome.
7. Commits the complete lifecycle result together.

If staging raises a PostgreSQL error, the nested transaction rolls back the staging operation while leaving the outer transaction usable for recording the retry outcome.

If the outer transaction itself fails, the lifecycle change and retry count are rolled back together.

No explicit commit is hidden inside the retry implementation.

## 9. Concurrency Handling

Retry workers claim records using:

`SELECT ... FOR UPDATE SKIP LOCKED`

The row is selected only when:

- status is `RETRY_PENDING`;
- `next_retry_at` is null or due.

The selected row is immediately moved to `REPROCESSING` inside the same transaction.

This avoids introducing a separate distributed queue.

## 10. Idempotency

The retry engine reuses the existing staging operation:

~~~sql
ON CONFLICT (event_id) DO NOTHING
~~~

Therefore:

- a successful retry cannot create a duplicate staging event;
- retrying an already-staged event remains safe;
- `RESOLVED` events are not selected by the retry worker;
- `UNRESOLVABLE` events are not selected by the retry worker.

The engine does not create a second idempotency mechanism.

## 11. Retry Exhaustion

`retry_count` remains the single retry attempt counter.

Its Task 19 semantic meaning is the number of completed retry executions.

The counter is incremented exactly once when a retry attempt reaches an outcome.

It is not incremented merely for:

- `QUARANTINED → RETRY_PENDING`;
- `RETRY_PENDING → REPROCESSING`.

When the maximum number of attempts is reached, the event is transitioned to `UNRESOLVABLE` and the reason is recorded as `RETRY_EXHAUSTED`.

The event is never silently discarded.

## 12. Metrics / Accounting Integration

No second accounting system was introduced.

Task 17's existing metrics continue to derive retry information from `telemetry_event_quarantine.retry_count` and lifecycle status.

Current metric semantics include:

- `retry_attempts`: completed retry executions represented by retry_count.
- `successful_retries`: quarantined events with retry attempts that ended RESOLVED.
- `failed_retries`: quarantined events with retry attempts that ended UNRESOLVABLE.

Per-attempt historical metrics are not introduced because the existing architecture does not contain a retry-attempt fact table.

## 13. Structured Logging

Task 18 structured JSON logging is reused.

Retry log context can include:

- `pipeline_name`
- `run_id`
- `event_id`
- `quarantine_id`
- `processing_mode`
- `retry_attempt`
- `max_attempts`
- `retry_classification`
- `error_code`
- `outcome`

The retry implementation does not log event payloads, documents, user free text, credentials, tokens, or secrets.

Database exception messages are not emitted as the structured message. The existing error taxonomy supplies the stable operational error code/category.

## 14. Database Changes

Migration added:

`migrations/011_add_retry_recovery_fields.sql`

New quarantine fields:

- `next_retry_at TIMESTAMPTZ NULL`
- `last_retry_error_code TEXT NULL`
- `last_retry_error_category TEXT NULL`

New partial index:

`idx_telemetry_quarantine_retry_due`

The existing schema did not persist when a retry should next run or the stable classification of the most recent retry failure. Application-only state would be lost when the worker restarted, so these fields are persistent.

No new retry counter was added.

Migration 009's corrected foreign key remains unchanged and continues to reference `public.telemetry_pipeline_run(id)`.

## 15. Tests

Added:

`tests/test_retry.py`

Coverage includes:

- exponential backoff;
- maximum backoff;
- invalid retry policy configuration;
- PostgreSQL operational failures;
- interface failures;
- connection failures;
- timeout failures;
- infrastructure failures;
- retryable DATABASE/STAGING taxonomy;
- deterministic validation/configuration/internal failures;
- scheduling;
- row-locking SQL;
- successful retry;
- retryable failure and backoff;
- non-retryable failure;
- retry exhaustion;
- terminal-event protection;
- batch limits;
- nested transaction boundary;
- structured logging safety.

Existing quarantine lifecycle tests were updated for the Task 19 retry-count semantics and new persistent quarantine fields.

## 16. Verification Results

Focused Task 19/lifecycle/config/logging suite:

**38 passed**

Complete Python regression suite:

**126 passed**

Compilation:

**compileall passed**

Diff validation:

**git diff --check passed**

A read-only attempt was made to connect to the configured local PostgreSQL instance (`localhost:5432/emi_db`) to inspect the live quarantine schema. The connection did not return during the verification window and was terminated.

Therefore, live PostgreSQL integration execution for Task 19 cannot be confirmed from this validation pass.

## 17. Operational Considerations

Retry workers should call `process_retry_batch()` with a bounded batch size.

The worker should run periodically or be invoked by the existing operational scheduler. Task 19 does not add a new scheduler, queue, or service.

Operators can use `schedule_retry()` to move a `QUARANTINED` event into the existing `RETRY_PENDING` lifecycle.

Backoff is persisted through `next_retry_at`, so worker restarts do not lose the scheduled retry time.

A future operational deployment should apply migration 011 before enabling retry workers against a database that does not already contain the new fields.

## 18. Known Limitations

1. Live PostgreSQL integration could not be completed because the configured local database connection did not return during verification.
2. There is no separate retry-attempt history table. Task 17 metrics therefore remain event/lifecycle based rather than providing a complete per-attempt audit trail.
3. Stale `REPROCESSING` recovery after a hard process termination is not implemented as a separate watchdog.
4. No jitter was added. The deterministic exponential backoff is sufficient for this task and keeps policy tests deterministic.
5. No new external queue or retry service was introduced.

## 19. Task 20 Boundary

Task 19 stops at the retry and recovery engine.

Task 20 and later functionality were not implemented.

The final production audit was not performed. It remains reserved for the project stage after all pipeline tasks are implemented.

## Final Result

Task 19 adds bounded, persistent retry and recovery on top of the existing quarantine lifecycle, using explicit classification, exponential backoff, row-level concurrency protection, idempotent staging, and structured operational logging.