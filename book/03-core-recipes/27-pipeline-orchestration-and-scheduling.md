# Recipe 27 — Pipeline Orchestration & Scheduling

A production pipeline is not just code that processes data. It must know when to run, what must run first, what can run in parallel, what succeeded, what failed, and what should happen next.

This recipe teaches the underlying mechanism of pipeline orchestration and scheduling by building a small workflow engine from scratch.

## 1. Problem Recognition

A simple script can work like this:

~~~text
python pipeline.py
~~~

Production pipelines usually contain dependent steps:

~~~text
Extract
  ↓
Validate
  ↓
Transform
  ↓
Load
  ↓
Reconcile
~~~

The problem appears when you need to answer:

- When should the pipeline run?
- Which task must run first?
- Which tasks can run in parallel?
- What happens when one task fails?
- Should downstream tasks run after a failure?
- How do we know which task is running?
- How do we prevent overlapping runs?
- How do we retry a failed task without rerunning everything?
- How do we recover an interrupted workflow?

If these decisions are hidden inside one large script, the pipeline becomes difficult to operate.

## 2. Concept and Reasoning

### What is orchestration?

Orchestration coordinates independent pieces of work.

A workflow can be represented as a directed acyclic graph (DAG):

~~~text
              ┌→ Transform A ─┐
Extract ──────┤               ├→ Load
              └→ Transform B ─┘
~~~

The arrows represent dependencies.

### Scheduler vs orchestrator

| Component | Responsibility |
|---|---|
| Scheduler | Decides when a workflow should start |
| Orchestrator | Decides how tasks execute and depend on one another |
| Task | Performs one unit of work |
| Worker | Executes the task |
| State store | Records workflow/task state |

Mental model:

~~~text
SCHEDULE
   ↓
CREATE RUN
   ↓
RESOLVE DEPENDENCIES
   ↓
RUN READY TASKS
   ↓
RECORD STATE
   ↓
RUN NEXT TASKS
   ↓
COMPLETE WORKFLOW
~~~

## 3. Task States

Start with:

~~~text
PENDING
RUNNING
SUCCESS
FAILED
SKIPPED
~~~

A workflow can expose:

~~~text
workflow: RUNNING

extract:     SUCCESS
validate:    SUCCESS
transform:   RUNNING
load:        PENDING
reconcile:   PENDING
~~~

Do not infer operational state from log messages alone. Persist state explicitly.

## 4. Dependency Resolution

A task is ready when all required upstream tasks succeeded.

~~~text
A ─→ C
B ─→ C
~~~

C is ready only when A and B are SUCCESS.

If A fails, C must not execute.

This is the basic dependency rule behind DAG execution.

## 5. Implementation

Build a small orchestration engine using Python standard-library mechanisms.

### 5.1 Task definition

~~~python
from dataclasses import dataclass, field
from enum import Enum
from typing import Callable

class TaskState(str, Enum):
    PENDING = "PENDING"
    RUNNING = "RUNNING"
    SUCCESS = "SUCCESS"
    FAILED = "FAILED"
    SKIPPED = "SKIPPED"

@dataclass
class Task:
    name: str
    function: Callable[[], None]
    dependencies: list[str] = field(default_factory=list)
    state: TaskState = TaskState.PENDING
~~~

The task contains identity, executable function, dependencies, and current state.

### 5.2 Workflow definition

~~~python
class Workflow:
    def __init__(self, tasks: list[Task]):
        self.tasks = {task.name: task for task in tasks}
        self._validate_graph()

    def _validate_graph(self):
        for task in self.tasks.values():
            for dependency in task.dependencies:
                if dependency not in self.tasks:
                    raise ValueError(f"Unknown dependency: {dependency}")
~~~

The workflow validates that dependencies actually exist.

### 5.3 Find ready tasks

~~~python
def ready_tasks(self):
    ready = []

    for task in self.tasks.values():
        if task.state != TaskState.PENDING:
            continue

        dependencies_succeeded = all(
            self.tasks[dependency].state == TaskState.SUCCESS
            for dependency in task.dependencies
        )

        if dependencies_succeeded:
            ready.append(task)

    return ready
~~~

This implements the core dependency rule.

### 5.4 Execute a task

~~~python
def execute_task(self, task):
    task.state = TaskState.RUNNING

    try:
        task.function()
    except Exception:
        task.state = TaskState.FAILED
        raise
    else:
        task.state = TaskState.SUCCESS
~~~

The state transition is:

~~~text
PENDING
   ↓
RUNNING
   ↓
SUCCESS / FAILED
~~~

### 5.5 Run the workflow

~~~python
def run(self):
    while True:
        ready = self.ready_tasks()

        if not ready:
            break

        for task in ready:
            try:
                self.execute_task(task)
            except Exception:
                self.skip_downstream_tasks(task.name)

        if all(
            task.state in {
                TaskState.SUCCESS,
                TaskState.FAILED,
                TaskState.SKIPPED,
            }
            for task in self.tasks.values()
        ):
            break
~~~

This is intentionally simple. The goal is to understand the mechanism an orchestrator provides, not to rebuild Airflow.

## 6. Dependency Failure and Skipping

A downstream task should not run if an upstream dependency failed.

~~~python
def skip_downstream_tasks(self, failed_task_name):
    for task in self.tasks.values():
        if failed_task_name in task.dependencies:
            if task.state == TaskState.PENDING:
                task.state = TaskState.SKIPPED
~~~

For a larger DAG, downstream skipping must propagate through the graph.

## 7. DAG Validation

A workflow should reject invalid dependency graphs.

### Unknown dependency

~~~text
transform → missing_task
~~~

This should fail before execution.

### Cyclic dependency

~~~text
A → B
B → C
C → A
~~~

This is not a valid DAG.

A simple cycle detector can use depth-first search:

~~~python
def has_cycle(graph):
    visiting = set()
    visited = set()

    def visit(node):
        if node in visiting:
            return True
        if node in visited:
            return False

        visiting.add(node)

        for child in graph.get(node, []):
            if visit(child):
                return True

        visiting.remove(node)
        visited.add(node)
        return False

    return any(visit(node) for node in graph)
~~~

Validate the graph before production execution.

## 8. Scheduling

Orchestration answers how tasks execute. Scheduling answers when the workflow starts.

Schedules can be time-based:

~~~text
Every day at 02:00
Every 15 minutes
~~~

or event-driven:

~~~text
New source file arrives
       ↓
Start workflow
~~~

Keep WHEN TO RUN separate from WHAT TO RUN.

Do not put scheduling logic inside business transformation code.

## 9. Workflow Runs

A workflow definition is not the same as a workflow execution.

~~~text
Workflow:
daily_payments

Runs:
2026-09-25 02:00 → SUCCESS
2026-09-26 02:00 → FAILED
~~~

Every execution should have a unique run ID.

Store at least:

~~~text
workflow_name
run_id
started_at
finished_at
state
~~~

For each task:

~~~text
run_id
task_name
attempt
started_at
finished_at
state
error
~~~

This makes execution observable and recoverable.

## 10. Preventing Overlapping Runs

Consider a workflow scheduled every 15 minutes:

~~~text
02:00 → Run A starts
02:15 → Run B starts
02:30 → Run C starts
~~~

If Run A takes 40 minutes, all three may overlap.

Define an explicit policy:

| Policy | Meaning |
|---|---|
| Allow overlap | Multiple runs may execute concurrently |
| Prevent overlap | Only one run may execute |
| Queue | New runs wait |
| Coalesce | Multiple pending triggers become one run |

Do not accidentally choose a policy through implementation details.

## 11. Retries at the Orchestration Level

Task retries and workflow retries are different.

### Task retry

~~~text
extract
  ↓
failure
  ↓
retry extract
  ↓
success
  ↓
continue
~~~

### Workflow rerun

~~~text
entire workflow
       ↓
new run
       ↓
re-execute selected work
~~~

Prefer retrying the smallest safe unit.

Combine this recipe with:

- Recipe 6 — Idempotency
- Recipe 8 — Retry Logic
- Recipe 12 — Checkpointing
- Recipe 24 — Partial Failure
- Recipe 26 — Network Failure

## 12. Testing

Test the orchestrator as an execution engine.

### Test 1 — Linear dependencies

~~~text
A → B → C
~~~

Expected order: A, then B, then C.

### Test 2 — Independent tasks

~~~text
A     B
 \\   /
   C
~~~

A and B are eligible before C.

### Test 3 — Failed dependency

If A fails:

~~~text
A = FAILED
C = SKIPPED
~~~

### Test 4 — Unknown dependency

Workflow creation should fail.

### Test 5 — Cyclic dependency

Workflow creation should fail.

### Test 6 — Task retry

A task that fails transiently should be retried according to policy.

### Test 7 — Retry exhaustion

Verify maximum attempts and no infinite retry loop.

### Test 8 — Workflow state

Verify that final workflow state accurately reflects task outcomes.

### Test 9 — Duplicate trigger

Trigger the same scheduled run twice and verify the configured overlap/idempotency policy.

## 13. Observability

At minimum, record:

~~~text
workflow_runs_total
workflow_success_total
workflow_failure_total
task_runs_total
task_success_total
task_failure_total
task_duration_seconds
workflow_duration_seconds
task_retries_total
scheduled_runs_total
skipped_tasks_total
~~~

Useful dimensions:

~~~text
workflow
task
state
attempt
failure_type
~~~

Example:

~~~json
{
  "event": "task_finished",
  "workflow": "daily_payments",
  "run_id": "run-2026-09-26-0200",
  "task": "transform",
  "state": "SUCCESS",
  "duration_ms": 1832
}
~~~

Avoid logging credentials, tokens, PII, or complete source records.

## 14. Intentional Failure

Do not stop after the happy path works.

### Failure drill 1 — Extract fails

Force extract to FAILED.

Observe downstream tasks, workflow state, error information, and retry behavior.

### Failure drill 2 — Transform fails

Force:

~~~text
extract → SUCCESS
transform → FAILED
load → SKIPPED
reconcile → SKIPPED
~~~

Verify that successful work is not unnecessarily repeated.

### Failure drill 3 — Worker interruption

Terminate the process while a task is RUNNING.

Ask:

- What state remains?
- How is the interrupted run detected?
- Can it be resumed safely?

This exposes why durable task state matters.

### Failure drill 4 — Duplicate scheduler trigger

Trigger the same workflow twice. Verify that the configured overlap policy is enforced.

## 15. Recovery

When a workflow fails:

1. Identify the workflow run.
2. Identify the failed task.
3. Inspect task state and logs.
4. Determine whether the failure is transient or permanent.
5. Retry the smallest safe unit.
6. Verify idempotency before retrying state-changing work.
7. Resume downstream execution only after prerequisites succeed.
8. Reconcile the final state.

Example:

~~~text
extract       SUCCESS
validate      SUCCESS
transform     FAILED
load          PENDING
reconcile     PENDING
~~~

Recovery should normally focus on transform, then load, then reconcile — not rerun everything blindly.

## 16. Scheduling Semantics

A beginner must understand:

### Scheduled time

When the workflow was supposed to run.

### Actual execution time

When the workflow actually started.

### Data interval

The period of data the run is responsible for.

Example:

~~~text
Run: 2026-09-26 02:00
Data interval:
2026-09-25 02:00 → 2026-09-26 02:00
~~~

This distinction becomes critical for backfills, late data, retries, and missed schedules.

Do not assume:

~~~text
run time = data time
~~~

## 17. Catchup and Missed Runs

Suppose a daily pipeline should run on Sep 23, Sep 24, Sep 25 and Sep 26, but the scheduler was offline from Sep 23–25.

Should it execute three historical runs when it returns?

This is a catchup policy.

| Policy | Behavior |
|---|---|
| Catch up | Execute missed intervals |
| No catchup | Start from current interval |
| Selective catchup | Recover only required intervals |

The correct choice depends on the pipeline's data semantics.

## 18. Production Tools You Should Know

These tools provide production implementations of orchestration concepts covered in this recipe.

| Tool | What to know |
|---|---|
| **Apache Airflow** | DAG-based workflow orchestration, scheduling, task state, retries and dependencies. |
| **Dagster** | Data-oriented orchestration with assets, dependencies, runs and observability. |
| **Prefect** | Python-oriented workflow orchestration with tasks, flows, scheduling and execution state. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the orchestration mechanism.

## 19. Production Runbook

### Workflow did not start

Check:

1. Scheduler health.
2. Workflow enabled/disabled state.
3. Schedule configuration.
4. Time zone.
5. Previous active run.
6. Concurrency limits.
7. Scheduler logs.

### Task failed

Check:

1. Task state.
2. Error type.
3. Attempt number.
4. Task logs.
5. Dependency state.
6. External dependency health.
7. Whether retry is safe.

### Workflow is stuck

Check:

1. RUNNING tasks.
2. Worker availability.
3. Scheduler health.
4. State-store connectivity.
5. Lock/concurrency limits.
6. Heartbeat/timeout state.

### Workflow is running twice

Check:

1. Scheduler duplication.
2. Overlap policy.
3. Run identifiers.
4. Locking/concurrency controls.
5. Whether both runs operate on the same data interval.

### What not to do

Do not:

- rerun an entire pipeline blindly
- manually change task state without understanding consequences
- hide task failures with unconditional downstream execution
- allow infinite retries
- mix scheduling logic with transformation logic
- assume every failed task is safe to retry
- ignore data intervals
- depend only on logs to determine execution state

## 20. Common Mistakes

### Mistake 1 — Treating the pipeline as one script

Large scripts hide task boundaries and failure state.

### Mistake 2 — No explicit dependencies

Tasks execute in accidental rather than defined order.

### Mistake 3 — No persisted state

After a worker dies, nobody knows what happened.

### Mistake 4 — Retrying the entire workflow

This can duplicate successful work.

### Mistake 5 — Ignoring overlapping runs

Two executions may modify the same data simultaneously.

### Mistake 6 — Confusing execution time with data time

This causes incorrect incremental processing and backfills.

### Mistake 7 — No DAG validation

Unknown dependencies and cycles reach runtime.

### Mistake 8 — Using orchestration as a substitute for idempotency

An orchestrator can retry work, but the underlying operation still needs to be safe to repeat.

## 21. Definition of Done

You are done when you can:

- explain why orchestration is needed
- distinguish a scheduler from an orchestrator
- represent a pipeline as a dependency graph
- define task states
- resolve task dependencies
- reject unknown dependencies
- detect cyclic dependencies
- execute tasks according to dependency order
- prevent downstream execution after failed prerequisites
- distinguish workflow runs from workflow definitions
- assign unique run IDs
- understand task-level vs workflow-level retries
- define an overlap policy
- explain scheduling and data intervals
- explain catchup behavior
- persist meaningful execution state
- test failure and recovery paths
- intentionally interrupt a running workflow
- diagnose a stuck workflow
- recover the smallest safe unit of work
- explain how Airflow, Dagster and Prefect relate to orchestration
- operate the workflow using a runbook

## 22. What You Learned

The central principle is:

> **A production pipeline needs an explicit execution system that knows when work should run, what depends on what, what state each task is in, and how failed work can be recovered safely.**

The core mechanism is:

~~~text
SCHEDULE
   ↓
CREATE RUN
   ↓
RESOLVE DEPENDENCIES
   ↓
RUN READY TASKS
   ↓
RECORD TASK STATE
   ↓
HANDLE FAILURE / RETRY
   ↓
RUN NEXT TASKS
   ↓
COMPLETE WORKFLOW
~~~

Orchestration does not make unreliable code reliable by itself.

It provides the execution framework in which reliable pipeline mechanisms — idempotency, retries, checkpointing, transactions, validation, reconciliation and observability — can operate together.
