# Task 17 — Data Quality Metrics & Aggregation

## 1. Task Overview

Task 17 adds a read-only data-quality measurement layer over the accounting, pipeline-run, and quarantine data already created by earlier tasks. It does not create a second accounting system.

The implementation answers event-volume, DQ outcome, retry/recovery, run, batch, and rate questions from existing database records.

## 2. Implementation Sequence

1. Verified the repository branch and existing Task 16 implementation.
2. Confirmed `telemetry_pipeline_batch` already stores extracted, valid, invalid, staged, quarantined counts and `processing_mode`.
3. Confirmed `telemetry_pipeline_run` stores run status and timestamps.
4. Confirmed `telemetry_event_quarantine` stores lifecycle status and retry count.
5. Added `src/payment_telemetry/metrics.py` as the aggregation/read model.
6. Added focused tests in `tests/test_metrics.py`.
7. Ran the full pytest suite and verified no regression.
8. Live PostgreSQL validation must remain explicitly reported as unavailable when the local endpoint cannot be reached.

## 3. Technical Changes

| File | Change |
|---|---|
| `src/payment_telemetry/metrics.py` | New read-only DQ aggregation module |
| `tests/test_metrics.py` | New focused aggregation tests |
| `docs/task_17_data_quality_metrics_aggregation.md` | Task implementation documentation |

No Task 17 database migration is required by the current design because the required measurements can be derived from existing tables.

## 4. Authoritative Sources

| Metric family | Source |
|---|---|
| Event received/validated outcomes | `telemetry_pipeline_batch` |
| Pipeline run outcomes | `telemetry_pipeline_run` |
| Quarantine resolution/unresolvable/retry state | `telemetry_event_quarantine` |

Batch accounting is the authoritative source for extracted, valid, invalid, and quarantined batch outcomes. Quarantine lifecycle is the authoritative source for current resolution and retry state. Run status is the authoritative source for run outcomes.

## 5. Metric Model

`DQMetricScope` supports `pipeline_name`, optional `processing_mode`, and optional start/end timestamps.

`DQMetrics` exposes:

- `events_received`
- `events_processed`
- `events_succeeded`
- `events_failed`
- `events_quarantined`
- `events_resolved`
- `events_unresolvable`
- `retry_attempts`
- `successful_retries`
- `failed_retries`
- `pipeline_runs`
- `successful_runs`
- `failed_runs`
- `batches`
- `successful_batches`
- `failed_batches`
- `success_rate`
- `failure_rate`
- `quarantine_rate`
- `resolution_rate`

Rates use decimal fractions rounded to four decimal places. A rate returns `None` when its denominator is zero so the implementation does not manufacture a percentage from empty data.

## 6. Aggregation Logic

The batch query sums the existing immutable batch outcome counters. Because the schema enforces `extracted_count = valid_count + invalid_count`, `events_received` is the sum of extracted counts and `events_processed` is the sum of valid plus invalid counts.

The quarantine query aggregates current lifecycle state from `telemetry_event_quarantine`. `retry_attempts` is the sum of the existing `retry_count` values. Resolution and unresolvable counts use the persisted terminal statuses.

The run query counts `RUNNING`, `SUCCEEDED`, and `FAILED` records for the selected pipeline/time window. Processing mode filtering applies to batch metrics because the mode is stored on the batch record.

## 7. Normal / Replay / Backfill

The scope can explicitly filter batch metrics by `NORMAL`, `REPLAY`, or `BACKFILL`. The aggregation layer does not change processing behavior, checkpoints, replay isolation, or backfill isolation.

## 8. Idempotency

The metrics layer performs SELECT queries only. Re-running the aggregation does not insert, update, or otherwise mutate persisted accounting. It therefore cannot double-write metric observations.

Duplicate event handling remains owned by the existing staging/quarantine `ON CONFLICT (event_id) DO NOTHING` behavior, not by Task 17.

## 9. Transaction / Query Boundaries

`get_dq_metrics()` opens a database cursor and issues three SELECT statements. It does not call `commit()` or `rollback()` and does not own or alter the caller transaction boundary.

## 10. Tests

`tests/test_metrics.py` verifies aggregation of mixed outcomes, empty data and zero-denominator rates, processing-mode/time-window parameterization, read-only behavior, and dataclass immutability.

Full regression result: **86 passed**.

## 11. PostgreSQL Validation

The implementation was syntax-checked and the full Python test suite passed. The local PostgreSQL endpoint was not used for live result validation in this implementation pass; therefore no live PostgreSQL validation result is claimed.

## 12. Failure Behavior

The aggregation function propagates database query errors to its caller. It does not catch and hide source-data or connection failures. Empty source sets are handled explicitly with SQL `COALESCE` and `None` rates for zero denominators.

## 13. Privacy Considerations

Task 17 adds aggregate measurements only. It does not collect form values, documents, free text, authentication tokens, raw payment information, unredacted PII, or session recordings.

## 14. Intentional Non-Scope

- No `telemetry_pipeline_metrics` persistence table.
- No new telemetry payload fields.
- No modification to the quarantine lifecycle.
- No checkpoint changes.
- No automatic alerting or OTEL integration.
- No Grafana dashboard implementation in this task.
- No commit or push.

## 15. Final Result

Task 17 now has a focused, read-only DQ aggregation layer over the existing pipeline accounting model, with explicit processing-mode and time-window scoping and focused regression coverage.
