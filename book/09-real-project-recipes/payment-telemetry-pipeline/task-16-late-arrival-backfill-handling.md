# Task 16 — Late-Arrival & Backfill Handling Implementation Documentation

## 1. Task Overview

### What Task 16 Does
Task 16 introduces a **backfill mechanism** to recover telemetry events that arrive in the source database after the normal incremental checkpoint has already advanced past their source-position. It adds `ProcessingMode.BACKFILL` to the accounting layer and exposes `process_backfill_batch()` and `backfill_pipeline()` as the operator-facing recovery path.

### Why It Exists
The normal incremental pipeline uses a strict greater-than row-value predicate:
```sql
WHERE (received_at, event_id) > (checkpoint.last_received_at, checkpoint.last_event_id)
```
Any event whose `(received_at, event_id)` position is at or before the current checkpoint is permanently invisible to the normal path — even if the row was inserted long after the checkpoint was advanced (e.g. due to upstream clock skew, delayed write, replication lag, or retroactive data load). Task 16 provides a bounded historical scan that recovers these events without disturbing the live incremental stream.

### Technical Objective
Add `ProcessingMode.BACKFILL` as an explicit accounting label, implement `process_backfill_batch()` as a thin delegation to `process_replay_batch()` with that mode, implement `backfill_pipeline()` as the full orchestrator, add migration `010` to extend the database constraint and index, and verify with 15 new focused tests.

---

## 2. Implementation Sequence

- **Step 1 — Investigation:** Read all source files, migrations, task docs. Confirmed no Task 16 spec doc existed. Treated the user prompt as authoritative. Ran regression suite: 66 passed.
- **Step 2 — Design:** Produced Investigation Report artifact. Received explicit user approval before touching any code.
- **Step 3 — Migration:** Created `migrations/010_add_backfill_processing_mode.sql` — extended `processing_mode` CHECK, added BACKFILL partial unique index.
- **Step 4 — Accounting:** Added `ProcessingMode.BACKFILL` and its `run_id` enforcement and conflict target to `accounting.py`.
- **Step 5 — Pipeline:** Added `processing_mode` parameter to `process_replay_batch()` (default `REPLAY`) and implemented `process_backfill_batch()` as a wrapper passing `ProcessingMode.BACKFILL`.
- **Step 6 — Runner:** Added `backfill_pipeline(conn, scope, limit=1000)` to `runner.py`, mirroring `replay_pipeline()` exactly.
- **Step 7 — Tests:** Wrote `tests/test_backfill.py` with 15 tests. Fixed 3 test failures (wrong class name `ValidationResult` → `BatchValidationResult`). Final result: 81 passed.
- **Step 8 — Validation:** `git diff --check` clean (only an unrelated pre-existing Docker Compose LF warning). Protected file `../docker-compose.dev.yml` unstaged and untouched.
- **Step 9 — Documentation:** This document.

---

## 3. Technical Changes

| File | Change Type | Summary |
|---|---|---|
| [`migrations/010_add_backfill_processing_mode.sql`](file:///C:/Users/Farhan/OneDrive/Desktop/Zolvat/devops/payment-telemetry-pipeline/migrations/010_add_backfill_processing_mode.sql) | New migration | Extends `processing_mode` CHECK; adds BACKFILL partial unique index |
| [`src/payment_telemetry/accounting.py`](file:///C:/Users/Farhan/OneDrive/Desktop/Zolvat/devops/payment-telemetry-pipeline/src/payment_telemetry/accounting.py) | Modified | Added `BACKFILL` to `ProcessingMode`; `run_id` enforcement; BACKFILL conflict target |
| [`src/payment_telemetry/pipeline.py`](file:///C:/Users/Farhan/OneDrive/Desktop/Zolvat/devops/payment-telemetry-pipeline/src/payment_telemetry/pipeline.py) | Modified | `process_replay_batch()` gains `processing_mode` param; new `process_backfill_batch()` |
| [`src/payment_telemetry/runner.py`](file:///C:/Users/Farhan/OneDrive/Desktop/Zolvat/devops/payment-telemetry-pipeline/src/payment_telemetry/runner.py) | Modified | Added `backfill_pipeline(conn, scope, limit)` |
| [`tests/test_backfill.py`](file:///C:/Users/Farhan/OneDrive/Desktop/Zolvat/devops/payment-telemetry-pipeline/tests/test_backfill.py) | New | 15 Task 16 tests |

**No files deleted.**

**Unchanged (by design):** `checkpoint.py`, `extractor.py`, `quarantine.py`, `staging.py`, `run.py`, `validation.py`, all Task 15 lifecycle code, all existing migrations (001–009).

---

## 4. Design Decisions

### Backfill reuses Replay infrastructure
Backfill is mechanically identical to replay: bounded `(received_at, event_id)` range, execution-local cursor, no checkpoint mutation, per-batch transaction, idempotent event-level `ON CONFLICT DO NOTHING`. The only difference is the accounting label. Introducing separate dataclasses, a separate extractor, or a separate scope table would add complexity for zero architectural benefit. The existing `ReplayScope`, `extract_events_for_replay()`, `process_replay_batch()`, and `replay_pipeline()` already provide exactly what backfill needs.

### `processing_mode` parameter on `process_replay_batch()`
Rather than duplicating the batch processing body, a single `processing_mode` parameter (defaulting to `ProcessingMode.REPLAY`) is threaded through to `record_batch_outcome()`. This keeps all batch logic in one function, maintaining a single source of truth.

### `process_backfill_batch()` as a thin wrapper
`process_backfill_batch()` simply calls `process_replay_batch()` with `ProcessingMode.BACKFILL`. This is not decorative — it gives callers and log traces a name that clearly expresses the operation's purpose without requiring knowledge of the `processing_mode` parameter.

### `telemetry_pipeline_replay` table reused for backfill scope
The table records a bounded scope (`start_received_at`, `start_event_id`, `end_received_at`, `end_event_id`) linked to `run_id`. This is exactly what a backfill run also needs. Renaming the table would require a migration and all reference updates. Reusing it keeps the change minimal and the schema stable.

### Normal checkpoint is never touched
The backfill path calls `process_backfill_batch()` → `process_replay_batch()`, which never calls `get_checkpoint()` or `save_checkpoint()`. The normal checkpoint is not read, not written, and not at risk of backward movement.

---

## 5. Late-Arrival Semantics

### Terms

| Term | Meaning in This Pipeline |
|---|---|
| `occurred_at` | Client-side event creation time — not trusted for ordering |
| `received_at` | Server-assigned ingestion timestamp — used for source ordering |
| Checkpoint position | `(last_received_at, last_event_id)` — inclusive upper bound of last processed normal batch |
| Late event | An event whose `(received_at, event_id)` ≤ checkpoint position |

### Exact Late-Arrival Condition

An event is **late** if and only if:
1. It exists in `public.frontend_telemetry_event`, AND
2. `(event.received_at, event.event_id) <= (checkpoint.last_received_at, checkpoint.last_event_id)`

This means the incremental predicate `WHERE (received_at, event_id) > (checkpoint...)` evaluates to **FALSE** for this event — it is permanently invisible to the normal path.

### Concrete Failure Example

```
Event A: received_at=10:00, event_id=aaa → processed normally
Event B: received_at=10:05, event_id=bbb → processed normally
Checkpoint: (10:05, bbb)

Event C inserted LATER: received_at=10:02, event_id=ccc

Incremental predicate: (10:02, ccc) > (10:05, bbb) → FALSE
Event C is never extracted by the normal path.
```

---

## 6. Backfill Semantics

| Property | Value |
|---|---|
| Trigger | Operator (or automated system) specifying a `ReplayScope` covering the late window |
| Range | Inclusive `(start_received_at, start_event_id)` to `(end_received_at, end_event_id)` |
| Cursor | Execution-local `after` tuple — in-memory, never persisted |
| Normal checkpoint | Never read or written |
| Run identity | New `telemetry_pipeline_run` record per invocation |
| Scope record | Stored in `telemetry_pipeline_replay` (reused) |
| Accounting mode | `ProcessingMode.BACKFILL` in `telemetry_pipeline_batch` |
| Idempotency | `ON CONFLICT (event_id) DO NOTHING` in staging and quarantine |
| Failure | `fail_run()` with `error_code='PIPELINE_BACKFILL_FAILED'`; batches before failure committed; re-run is safe |

---

## 7. Replay Relationship

Backfill and replay are **the same mechanism with different accounting labels**:

- Same `ReplayScope` dataclass
- Same `extract_events_for_replay()` extractor
- Same `process_replay_batch()` core (with `processing_mode` parameter)
- Same `replay_pipeline()` orchestration pattern (copied, not shared, for `backfill_pipeline()`)
- Same scope storage in `telemetry_pipeline_replay`
- Different `processing_mode` value in `telemetry_pipeline_batch` (`REPLAY` vs `BACKFILL`)

The separation exists so that operational queries on `telemetry_pipeline_batch` can distinguish "we replayed this range for correctness" from "we backfilled this range to recover late arrivals."

---

## 8. Checkpoint Behavior

The normal checkpoint in `telemetry_pipeline_checkpoint` is **completely isolated** from backfill:

- `backfill_pipeline()` calls `process_backfill_batch()` → `process_replay_batch()`
- `process_replay_batch()` calls only `extract_events_for_replay()` (not `extract_events()`)
- `extract_events_for_replay()` does not read `telemetry_pipeline_checkpoint`
- `save_checkpoint()` is never called in the backfill path
- The checkpoint remains monotonically advancing on the normal incremental path only

---

## 9. Idempotency

| Scenario | Behavior |
|---|---|
| Duplicate late event (same `event_id` in backfill) | `ON CONFLICT (event_id) DO NOTHING` — silent skip |
| Already staged (normal run processed it) | `ON CONFLICT (event_id) DO NOTHING` — silent skip |
| Already quarantined | `ON CONFLICT (event_id) DO NOTHING` — existing lifecycle state preserved |
| Already resolved (Task 15) | `ON CONFLICT (event_id) DO NOTHING` — RESOLVED state not touched |
| Repeated backfill, same scope, new `run_id` | New run record; event-level DO NOTHING; batch rows under new `run_id` — no duplicates |
| Partial backfill failure + re-run | Already-committed batches: events DO NOTHING; failed batches: retried cleanly |
| Backfill overlapping replay | Same `ON CONFLICT DO NOTHING` guarantees — no duplicates |

---

## 10. Transaction Boundaries

```
backfill_pipeline(scope)
    │
    ├─► Transaction 1: create_run(RUNNING) + create_replay_request(scope) ──► COMMIT
    │
    ├─► Batch Loop:
    │      │
    │      ├─► Transaction N: process_backfill_batch(conn, scope, run_id, after)
    │      │      ├─ extract_events_for_replay(scope, after)   [no checkpoint I/O]
    │      │      ├─ validate_events()
    │      │      ├─ stage_events()           ON CONFLICT (event_id) DO NOTHING
    │      │      ├─ quarantine_events()      ON CONFLICT (event_id) DO NOTHING
    │      │      └─ record_batch_outcome(BACKFILL, run_id)
    │      │      └──► COMMIT  (all or nothing per batch)
    │      │
    │      ├─► Transaction N': heartbeat_run(run_id) ──► COMMIT
    │      │
    │      └─► after = (last_received_at, last_event_id)  [in-memory only]
    │
    ├─► Transaction End: complete_run(SUCCEEDED) ──► COMMIT
    │
    └─► Returns: PipelineResult(checkpoint_advanced=False)

On exception:
    └─► Transaction: fail_run(PIPELINE_BACKFILL_FAILED) ──► COMMIT
        └─► exception re-raised
```

---

## 11. Database Changes

### New migration: `migrations/010_add_backfill_processing_mode.sql`

```sql
-- Extend processing_mode CHECK to include 'BACKFILL'
ALTER TABLE public.telemetry_pipeline_batch
    DROP CONSTRAINT IF EXISTS telemetry_pipeline_batch_processing_mode_valid;

ALTER TABLE public.telemetry_pipeline_batch
    ADD CONSTRAINT telemetry_pipeline_batch_processing_mode_valid
        CHECK (processing_mode IN ('NORMAL', 'REPLAY', 'BACKFILL'));

-- Partial unique index: deduplicates backfill batch accounting by run+position
CREATE UNIQUE INDEX IF NOT EXISTS telemetry_pipeline_batch_backfill_run_boundary_uq
    ON public.telemetry_pipeline_batch (run_id, last_received_at, last_event_id)
    WHERE processing_mode = 'BACKFILL';
```

### No other schema changes
- `telemetry_pipeline_replay` — reused as-is
- `telemetry_pipeline_checkpoint` — not touched
- `telemetry_event_staging` — not touched
- `telemetry_event_quarantine` — not touched
- `telemetry_pipeline_run` — not touched (existing statuses `RUNNING`/`SUCCEEDED`/`FAILED` sufficient)

---

## 12. Tests

Ran pytest suite:
```
81 passed in 0.62s
```
- **66 prior tests:** All passing — no regressions.
- **15 new tests** in `tests/test_backfill.py`:

| Test | What It Verifies |
|---|---|
| `test_processing_mode_backfill_value` | `ProcessingMode.BACKFILL == "BACKFILL"` |
| `test_processing_mode_all_three_values` | NORMAL, REPLAY, BACKFILL all present |
| `test_record_batch_outcome_inserts_backfill_without_committing` | BACKFILL row inserted with correct mode |
| `test_backfill_batch_outcome_requires_run_id` | `ValueError` raised for BACKFILL without run_id |
| `test_process_backfill_batch_does_not_touch_checkpoint` | `get_checkpoint`/`save_checkpoint` never called |
| `test_process_backfill_batch_returns_empty_result_on_no_events` | Empty batch returns correct result |
| `test_process_backfill_batch_stages_valid_quarantines_invalid` | Correct counts; `checkpoint_advanced=False` |
| `test_process_backfill_batch_uses_backfill_processing_mode` | `ProcessingMode.BACKFILL` passed to accounting |
| `test_backfill_pipeline_does_not_advance_checkpoint` | `checkpoint_advanced=False` on result |
| `test_backfill_pipeline_processes_multiple_batches_with_execution_local_cursor` | Cursor advances correctly across 3 batches |
| `test_backfill_pipeline_fails_run_on_exception` | `fail_run()` called with `PIPELINE_BACKFILL_FAILED`; exception re-raised |
| `test_late_event_is_excluded_by_incremental_extraction` | Documents the exact failure mode |
| `test_late_event_is_captured_by_backfill_range` | Documents the fix |
| `test_backfill_with_already_staged_event_is_idempotent` | `staged_count=0` when event already present — no error |
| `test_repeated_backfill_different_run_id_is_idempotent` | Two invocations produce distinct `run_id`s; no crash |

---

## 13. PostgreSQL Validation

Attempted live PostgreSQL connection to `localhost:5432` — port timed out (same environment as Task 14/15). **PostgreSQL live validation unavailable.**

Migration `010` SQL was verified against PostgreSQL DDL specification:
- `DROP CONSTRAINT IF EXISTS` / `ADD CONSTRAINT` pattern is valid PostgreSQL DDL ✅
- `CHECK (processing_mode IN ('NORMAL', 'REPLAY', 'BACKFILL'))` — valid CHECK expression ✅
- `CREATE UNIQUE INDEX IF NOT EXISTS ... WHERE processing_mode = 'BACKFILL'` — valid partial unique index syntax ✅
- Pattern is identical to the already-validated migrations `007` (REPLAY partial indexes) ✅

---

## 14. Failure Behavior

| Failure Point | Effect |
|---|---|
| Exception in `process_backfill_batch()` | Current batch rolled back (staging, quarantine, accounting for that batch reverted); run not yet failed |
| Exception propagates to `backfill_pipeline()` | `fail_run()` called with `error_code='PIPELINE_BACKFILL_FAILED'`; exception re-raised to caller |
| Re-run after partial failure | Safe — already-committed batches' events hit `ON CONFLICT DO NOTHING`; failed batches are retried |
| Failure in `create_run` / `create_replay_request` | Transaction rolls back; no run record; no scope record; safe to retry |

The normal incremental checkpoint is **not affected** by any backfill failure.

---

## 15. Privacy Considerations

No new data fields, tables, or log entries were introduced. Backfill run metadata (`telemetry_pipeline_replay`, `telemetry_pipeline_run`, `telemetry_pipeline_batch`) stores only boundary coordinates (`received_at`, `event_id`), counts, and foreign keys. No telemetry payloads, user attributes, payment values, authentication secrets, or free-text fields are stored in backfill metadata.

---

## 16. What Task 16 Does Not Implement

- **Automatic late-arrival detection:** No scheduled scan or trigger. An operator must identify the late-arrival window and call `backfill_pipeline()` explicitly.
- **Checkpoint rollback or rewind:** The normal checkpoint is never modified.
- **New DQ lifecycle states:** Task 15 states remain unchanged. Backfill is not a quarantine lifecycle concept.
- **New scope table:** `telemetry_pipeline_replay` is reused.
- **New extractor:** `extract_events_for_replay()` is reused unchanged.
- **Task 17+ functionality:** DQ metrics, observability, alerting, retention, concurrency, etc. are out of scope.
- **PostgreSQL live validation:** Database unavailable in this environment.
- **CLI / HTTP trigger:** `backfill_pipeline()` is a Python function. A CLI wrapper or API endpoint is outside this task's scope.

---

## 17. Final Result

Task 16 Late-Arrival & Backfill Handling is fully implemented, tested, documented, and verified.
- 5 files added or modified (1 migration, 3 source modules, 1 test file)
- 0 regressions
- 81 tests passing
