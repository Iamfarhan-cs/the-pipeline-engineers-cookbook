# Recipe 28 — Pipeline Run Tracking

A production pipeline needs a durable record of every execution. Logs alone are not enough to answer which run executed, what data it owned, which tasks ran, what failed, how long work took, and whether recovery is safe.

## 1. Problem Recognition

Without run tracking, multiple executions can become indistinguishable:

    02:00 → pipeline starts
    02:15 → pipeline starts
    02:30 → pipeline starts

The production problem is:

> Every pipeline execution needs a durable identity and lifecycle state separate from ordinary application logs.

Run tracking matters for scheduled runs, retries, backfills, partial failures, multiple workers, incident investigation, reconciliation, and SLA monitoring.

## 2. Concept and Reasoning

A pipeline definition describes what should happen. A pipeline run describes one actual execution.

    Pipeline: daily_payments
        ├── Run A
        ├── Run B
        └── Run C

Every run needs a stable unique ID.

Typical states:

    PENDING → RUNNING → SUCCESS
                     ├→ FAILED
                     └→ CANCELLED

Do not infer authoritative state only from process exit codes or log messages.

## 3. Run Metadata

A useful run record contains:

| Field | Purpose |
|---|---|
| run_id | Unique execution identity |
| pipeline_name | Pipeline being executed |
| status | Current/final state |
| scheduled_at | Intended execution time |
| started_at | Actual start |
| finished_at | Actual completion |
| data_interval_start | Data period start |
| data_interval_end | Data period end |
| attempt | Execution attempt |
| trigger_type | scheduled/manual/backfill/retry |
| error_message | Failure summary |
| created_at | Record creation time |

Keep execution metadata separate from large business data.

## 4. Task Run Tracking

Pipeline-level tracking is not enough.

    Pipeline Run
       ├── extract
       ├── validate
       ├── transform
       ├── load
       └── reconcile

Each task execution should record:

    run_id
    task_run_id
    task_name
    status
    attempt
    started_at
    finished_at
    rows_read
    rows_written
    error_type
    error_message

This lets an operator see exactly where a run failed.

## 5. Implementation

Use PostgreSQL for the learning implementation.

### Pipeline runs

    CREATE TABLE pipeline_runs (
        run_id UUID PRIMARY KEY,
        pipeline_name TEXT NOT NULL,
        status TEXT NOT NULL,
        scheduled_at TIMESTAMPTZ,
        started_at TIMESTAMPTZ,
        finished_at TIMESTAMPTZ,
        data_interval_start TIMESTAMPTZ,
        data_interval_end TIMESTAMPTZ,
        attempt INTEGER NOT NULL DEFAULT 1,
        trigger_type TEXT NOT NULL,
        error_message TEXT,
        created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

        CHECK (status IN (
            'PENDING', 'RUNNING', 'SUCCESS',
            'FAILED', 'CANCELLED'
        )),
        CHECK (attempt > 0)
    );

### Task runs

    CREATE TABLE task_runs (
        task_run_id UUID PRIMARY KEY,
        run_id UUID NOT NULL REFERENCES pipeline_runs(run_id),
        task_name TEXT NOT NULL,
        status TEXT NOT NULL,
        attempt INTEGER NOT NULL DEFAULT 1,
        started_at TIMESTAMPTZ,
        finished_at TIMESTAMPTZ,
        rows_read BIGINT,
        rows_written BIGINT,
        error_type TEXT,
        error_message TEXT,
        created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

        CHECK (status IN (
            'PENDING', 'RUNNING', 'SUCCESS',
            'FAILED', 'SKIPPED', 'CANCELLED'
        )),
        CHECK (attempt > 0)
    );

Add indexes for operational queries:

    CREATE INDEX idx_pipeline_runs_name_started
    ON pipeline_runs (pipeline_name, started_at DESC);

    CREATE INDEX idx_pipeline_runs_status
    ON pipeline_runs (status);

    CREATE INDEX idx_task_runs_run
    ON task_runs (run_id);

## 6. Starting a Run

A run should be created before task execution begins.

    import uuid
    from datetime import datetime, timezone

    def start_run(connection, pipeline_name,
                  scheduled_at=None,
                  data_interval_start=None,
                  data_interval_end=None,
                  trigger_type="scheduled"):
        run_id = uuid.uuid4()

        with connection:
            with connection.cursor() as cursor:
                cursor.execute(
                    """
                    INSERT INTO pipeline_runs (
                        run_id, pipeline_name, status,
                        scheduled_at, started_at,
                        data_interval_start,
                        data_interval_end,
                        trigger_type
                    )
                    VALUES (
                        %s, %s, 'RUNNING', %s, %s,
                        %s, %s, %s
                    )
                    """,
                    (
                        run_id, pipeline_name,
                        scheduled_at,
                        datetime.now(timezone.utc),
                        data_interval_start,
                        data_interval_end,
                        trigger_type,
                    ),
                )

        return run_id

Pass the run ID to every task.

## 7. Starting and Completing Tasks

Create a task-run record when a task begins:

    task_run_id = UUID
    run_id = pipeline_run_id
    task_name = "transform"
    status = "RUNNING"
    attempt = 1

When it finishes, persist:

    status
    finished_at
    rows_read
    rows_written
    error_type
    error_message

A task should move only through valid lifecycle transitions:

    PENDING → RUNNING
    RUNNING → SUCCESS
    RUNNING → FAILED
    RUNNING → SKIPPED
    RUNNING → CANCELLED

Reject contradictory transitions such as SUCCESS → RUNNING.

## 8. Pipeline Final State

The pipeline state should be derived from task outcomes.

Example:

    extract       SUCCESS
    validate      SUCCESS
    transform     SUCCESS
    load          SUCCESS
    reconcile     SUCCESS

    → pipeline SUCCESS

If a required task fails:

    transform     FAILED
    load          SKIPPED

    → pipeline FAILED

Do not allow arbitrary task code to declare the entire pipeline successful.

## 9. Trigger Types

Record why a run started:

    scheduled
    manual
    retry
    backfill
    replay
    recovery

This becomes important during incident investigation.

## 10. Run vs Attempt

Do not automatically treat every retry as a new logical run.

Example:

    Logical Run: RUN-123

    Attempt 1 → FAILED
    Attempt 2 → SUCCESS

Define explicitly whether your orchestration system keeps one logical run with multiple attempts or creates a new run for each execution.

The important requirement is that operators can reconstruct the relationship.

## 11. Data Intervals

A timestamp does not tell you which data a run owns.

Record:

    scheduled_at
    started_at
    data_interval_start
    data_interval_end

For example:

    Run: 2026-09-26 02:00
    Data:
    2026-09-25 02:00 → 2026-09-26 02:00

Do not assume:

    run time = data time

This distinction is essential for backfills, late data, retries, and missed schedules.

## 12. Testing

Test at least:

### Test 1 — Start run
Verify unique ID, state, timestamps, trigger type, and data interval.

### Test 2 — Start task
Verify that the task run belongs to the correct pipeline run.

### Test 3 — Successful task
Verify RUNNING → SUCCESS and completion timestamp.

### Test 4 — Failed task
Verify error type and message are persisted.

### Test 5 — Invalid transition
Attempt SUCCESS → RUNNING and verify rejection.

### Test 6 — Pipeline success
All required tasks succeed; pipeline becomes SUCCESS.

### Test 7 — Pipeline failure
A required task fails; pipeline becomes FAILED.

### Test 8 — Retry
Verify attempt tracking and relationship to the logical run.

### Test 9 — Concurrent update
Two workers attempt to update the same execution. Verify the final state cannot become contradictory.

## 13. Observability

Run tracking is itself an operational data source.

Useful metrics:

    pipeline_runs_total
    pipeline_success_total
    pipeline_failure_total
    pipeline_duration_seconds
    task_runs_total
    task_success_total
    task_failure_total
    task_duration_seconds
    rows_read_total
    rows_written_total

Useful operational queries include:

### Failed runs

    SELECT run_id, pipeline_name, started_at,
           finished_at, error_message
    FROM pipeline_runs
    WHERE status = 'FAILED'
    ORDER BY started_at DESC;

### Active runs

    SELECT run_id, pipeline_name, started_at
    FROM pipeline_runs
    WHERE status = 'RUNNING'
    ORDER BY started_at;

### Slow tasks

    SELECT task_name,
           AVG(finished_at - started_at) AS avg_duration
    FROM task_runs
    WHERE status = 'SUCCESS'
    GROUP BY task_name
    ORDER BY avg_duration DESC;

## 14. Intentional Failure

### Failure drill 1 — Worker crash

Start a task and terminate the worker while it is RUNNING.

Observe the stale RUNNING state.

Then determine how the system detects stale execution through heartbeat, timeout, or worker-state checks.

### Failure drill 2 — Invalid transition

Attempt:

    SUCCESS → RUNNING

The operation must be rejected.

### Failure drill 3 — Duplicate completion

Complete an already completed task.

Verify historical execution state is not silently overwritten.

### Failure drill 4 — Wrong run ID

Intentionally associate a task with the wrong run.

Your constraints and tests should prevent silent corruption.

## 15. Recovery

When a run appears stuck:

1. Identify the run ID.
2. Identify the running task.
3. Check worker/process health.
4. Check heartbeat or timeout information.
5. Determine whether the external operation actually completed.
6. Determine whether retry is safe.
7. Mark stale execution according to the orchestration policy.
8. Retry or recover the task.
9. Reconcile the final result.

Do not mark a RUNNING task failed merely because it looks old.

## 16. Backfills and Run Tracking

A backfill may cover multiple data intervals:

    2026-09-01 → 2026-09-07

Each independently processed interval should be identifiable through its run metadata.

Store the interval and trigger type so operators can distinguish:

    scheduled execution
    backfill
    retry
    manual execution

This makes recovery and reconciliation much easier.

## 17. Run Tracking and Idempotency

Run tracking answers:

> What happened?

Idempotency answers:

> Can this work safely happen again?

You need both.

    RUN TRACKING
         ↓
    identify execution
         +
    IDEMPOTENCY
         ↓
    safe repeated execution

Run tracking does not make a non-idempotent operation safe to retry.

## 18. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Apache Airflow** | Tracks DAG/task execution metadata and exposes task and run state. |
| **Dagster** | Tracks runs, steps, events, and execution metadata. |
| **OpenTelemetry** | Provides traces and correlation context that can complement pipeline run tracking. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding durable run identity and state.

## 19. Production Runbook

### Pipeline appears stuck

Check:

1. Run ID.
2. Pipeline status.
3. Running tasks.
4. Task start times.
5. Worker health.
6. Heartbeats/timeouts.
7. External dependencies.

### Pipeline failed

Check:

1. Run ID.
2. Failed task.
3. Attempt number.
4. Error type.
5. Error message.
6. Data interval.
7. Trigger type.
8. Whether recovery is safe.

### Multiple runs are active

Check:

1. Schedule.
2. Overlap policy.
3. Data intervals.
4. Run IDs.
5. Whether both runs touch the same target.

### What not to do

Do not:

- use logs as the only run history
- reuse run IDs
- overwrite completed execution history
- allow arbitrary task code to declare pipeline success
- confuse run identity with task identity
- confuse retry attempts with unrelated scheduled runs
- mark stale runs failed without checking worker state

## 20. Common Mistakes

### Mistake 1 — One ID for everything

Pipeline run, task run, and record identity serve different purposes.

### Mistake 2 — No data interval

A timestamp alone does not identify the data owned by a run.

### Mistake 3 — Mutable historical state

Changing old execution records destroys incident evidence.

### Mistake 4 — No explicit state transitions

Impossible states can enter the database.

### Mistake 5 — No task-level records

A pipeline can say FAILED without identifying the failing task.

### Mistake 6 — Run tracking without recovery semantics

Knowing that a run failed is not enough. You must know what can safely be retried.

### Mistake 7 — Treating logs as execution state

Logs describe events. Durable run state describes the authoritative execution lifecycle.

## 21. Definition of Done

You are done when you can:

- explain why pipeline run tracking is necessary
- distinguish a pipeline definition from a pipeline run
- generate unique run IDs
- define explicit pipeline states
- define valid state transitions
- persist pipeline run metadata
- persist task run metadata
- associate task runs with pipeline runs
- record scheduled time and actual execution time
- record data intervals
- record trigger type
- distinguish logical runs from attempts
- query failed and active runs
- identify slow tasks
- test concurrent state updates
- intentionally create a stuck task
- diagnose stale execution
- recover a failed or interrupted run safely
- explain why run tracking does not replace idempotency
- explain how Airflow, Dagster, and OpenTelemetry relate to execution tracking
- operate run tracking using a production runbook

## 22. What You Learned

The central principle is:

> **Every production pipeline execution needs a durable identity, lifecycle state, task-level history, and enough metadata to explain what happened and recover safely.**

The operational model is:

    TRIGGER
       ↓
    CREATE RUN ID
       ↓
    PERSIST RUN
       ↓
    START TASK
       ↓
    PERSIST TASK STATE
       ↓
    EXECUTE
       ↓
    UPDATE TASK STATE
       ↓
    DERIVE RUN STATE
       ↓
    PERSIST FINAL RESULT

Logs help investigate execution.

Metrics help measure execution.

Durable run tracking provides the authoritative execution history needed to operate and recover a production pipeline.
