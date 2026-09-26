# E29 — API Partial Failure

## 1. Problem Recognition

An API extraction request can succeed only partially.

Examples:
- a page contains valid and invalid records
- an API returns data plus item-level errors
- a batch request succeeds for some IDs and fails for others
- one nested resource cannot be retrieved
- a provider returns partial data with an error indicator
- a multi-resource extraction completes some resources but fails others

Recognize the problem when an extraction run can no longer be described simply as success or failure.

The production objective is:

> Make partial success explicit, durable, observable, and recoverable without silently losing records or corrupting progress.

## 2. Concept and Reasoning

Partial failure creates an important distinction:

```text
FULL SUCCESS
    ↓
ALL REQUIRED WORK COMPLETED

PARTIAL FAILURE
    ↓
SOME WORK COMPLETED
    ↓
SOME WORK FAILED
    ↓
PROGRESS MUST BE RECOVERABLE
```

Never represent partial success as ordinary success.

## 3. Failure Boundaries

First identify the smallest unit that can fail independently.

Possible boundaries: entire HTTP response, page, record, nested child, requested resource, tenant, partition, or batch item.

| Failure boundary | Typical recovery |
|---|---|
| response | retry/re-extract page |
| record | quarantine/retry record |
| child resource | retry child |
| tenant | resume tenant |
| batch item | retry failed item |

The recovery strategy depends on this boundary.

## 4. Response-Level Partial Failure

Some APIs return data plus errors in the same response.

```json
{
  "data": [{"id": "1"}, {"id": "2"}],
  "errors": [{"id": "3", "code": "INVALID"}]
}
```

The response is not equivalent to a normal successful response. The extraction contract must define whether valid data may be persisted while errors are isolated.

## 5. Item-Level Partial Failure

Suppose a batch contains A success, B success, C failure, D success, and E failure.

Track those outcomes independently when the API contract provides item-level results.

```text
BATCH
 ├── A SUCCESS
 ├── B SUCCESS
 ├── C FAILED
 ├── D SUCCESS
 └── E FAILED
```

## 6. Partial Failure State Model

```text
PENDING
   ↓
PROCESSING
   ↓
 ┌───────────────┬───────────────┐
 ↓               ↓               ↓
SUCCESS       RETRYABLE       PERMANENT
                ↓               ↓
             RETRY          QUARANTINE
                ↓
             SUCCESS
```

Do not use a single boolean to represent this lifecycle.

## 7. Durable Item Status

For independent work units, persist status.

```sql
CREATE TABLE extraction_items (
    run_id BIGINT NOT NULL,
    item_id TEXT NOT NULL,
    status TEXT NOT NULL,
    attempt_count INTEGER NOT NULL DEFAULT 0,
    last_error TEXT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (run_id, item_id)
 );
```

This makes partial progress visible after process crashes.

## 8. Success Must Be Durable

Persist successful work before marking it successful.

```text
PROCESS ITEM
    ↓
PERSIST RESULT
    ↓
COMMIT
    ↓
MARK SUCCESS
```

Never mark success before durable persistence.

## 9. Failed Items Must Remain Addressable

A failed item needs enough identity to retry it later.

Store appropriate metadata such as run_id, item_id, source, endpoint, page, cursor, attempt_count, error_class, last_error, and failed_at.

Do not depend on logs alone for recovery.

## 10. Partial Failure and Pagination

Consider a page containing 100 records where 3 records fail.

Two valid designs are:

### Page-atomic

```text
100 records
 ↓
3 fail
 ↓
whole page remains incomplete
```

Retry the entire page idempotently.

### Record-independent

```text
100 records
 ↓
97 success + 3 failed
 ↓
persist success state
 ↓
retry only 3
```

The second model requires durable per-record progress and safe deduplication.

Choose deliberately.

## 11. Partial Failure and Checkpointing

The checkpoint must represent the semantics of the chosen recovery model.

Page-atomic:

```text
PAGE N
 ↓
PROCESS ALL
 ↓
ANY REQUIRED FAILURE?
 /              \
YES              NO
 ↓                ↓
NO CHECKPOINT   CHECKPOINT N+1
```

Record-independent processing may checkpoint page progress while separately tracking failed records.

Never mix these models accidentally.

## 12. Partial Failure and Idempotency

Retrying partially successful work can produce duplicates unless writes are idempotent.

```text
Batch A
 ├── record 1 persisted
 ├── record 2 persisted
 └── record 3 failed
        ↓
retry batch
        ↓
record 1 must not duplicate
record 2 must not duplicate
record 3 must succeed or remain failed
```

Use stable source identity and destination uniqueness where possible.

## 13. Partial Failure and Transactions

Transactions can make a unit atomic.

```python
with connection.transaction():
    persist_records(records)
    update_checkpoint(checkpoint)
```

This is useful when destination and checkpoint share the same transactional system.

When systems are separate, use explicit recovery semantics rather than pretending the operation is globally atomic.

## 14. Batch-Level Transactions

Choose the transaction boundary deliberately.

```text
ONE LARGE TRANSACTION
    ↓
strong atomicity
    ↓
larger rollback / lock / resource cost
```

versus smaller transactions with smaller recovery units but more partial states.

## 15. Retryable vs Permanent Partial Failures

Potentially retryable: timeout, connection failure, HTTP 429, HTTP 503, temporary provider dependency failure.

Potentially permanent: invalid source identifier, unsupported schema, malformed record, authorization failure requiring configuration, documented business-rule rejection.

Classify the actual error rather than retrying everything.

## 16. Quarantine

Use quarantine when a failed item cannot safely continue through the normal path.

```text
FAILED ITEM
    ↓
CLASSIFY
    ↓
PERMANENT / EXHAUSTED
    ↓
QUARANTINE
```

Quarantine should preserve enough context for diagnosis and replay.

## 17. Partial Failure vs Dead-Letter Handling

Partial failure describes the processing outcome. Dead-letter or quarantine mechanisms describe where unrecoverable work is placed.

```text
PARTIAL FAILURE
      ↓
RETRY?
 /          \
YES          NO
 ↓            ↓
RETRY      DEAD-LETTER / QUARANTINE
```

## 18. Batch Requests

Batch APIs create special partial-failure behavior. If 50 IDs produce 45 successes, 3 not-found results, and 2 temporary failures, store those outcomes independently when the API provides item-level results.

Do not turn all 50 into one undifferentiated failure.

## 19. Nested Resource Extraction

A parent can succeed while a child request fails.

```text
Customer
   ↓
Orders
   ├── success
   ├── success
   └── timeout
```

Define whether the parent is complete. If child completeness is required, the parent remains incomplete until recovery.

## 20. Multi-Tenant Extraction

One tenant can fail while others succeed.

```text
RUN
 ├── tenant A → SUCCESS
 ├── tenant B → SUCCESS
 ├── tenant C → FAILED
 └── tenant D → SUCCESS
```

Do not restart every tenant because one tenant failed. Persist tenant-level progress where independent recovery is required.

## 21. Partial Failure and Concurrency

Concurrent workers can produce mixed outcomes. Protect state updates with appropriate database constraints and transactional boundaries.

## 22. Avoiding False Run Success

A run should not be marked successful merely because the process exited normally.

Example: expected 100, success 97, failed 3.

Possible explicit final states include SUCCESS, PARTIAL_SUCCESS, and FAILED. The exact policy is operational, but the states must be explicit.

## 23. Run-Level Status

```sql
CREATE TABLE pipeline_runs (
    run_id BIGSERIAL PRIMARY KEY,
    status TEXT NOT NULL,
    expected_items INTEGER,
    succeeded_items INTEGER NOT NULL DEFAULT 0,
    failed_items INTEGER NOT NULL DEFAULT 0,
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    finished_at TIMESTAMPTZ
 );
```

Do not derive important recovery state from log text.

## 24. Completion Rules

Define completion mathematically when useful:

```text
expected = success + failed + pending
```

A run is complete when pending equals zero. Then classify the result based on failed count and policy.

## 25. Partial Failure and Reprocessing

Reprocessing should target incomplete work where possible.

```text
RUN 1000 ITEMS
   ↓
990 SUCCESS
10 FAILED
   ↓
REPROCESS FAILED SET
   ↓
10 SUCCESS
   ↓
RECONCILE
```

This is safer and cheaper than blindly replaying all 1000 items.

## 26. Testing

### Unit tests

Test all-success batches, one or multiple failed items, retryable and permanent failures, retry exhaustion, duplicate retry, worker crashes, partial transactions, checkpoint behavior, and final run status.

### Integration tests

Verify that successful items remain durable, failed items remain retryable, retries do not duplicate successful records, checkpoints match the recovery model, and final run status is accurate.

## 27. Intentional Failure

### Failure Drill A — Fail One Item

Force one record to fail while others succeed. Expected: successful records remain durable, the failed record is tracked, and the run is not falsely reported as full success.

### Failure Drill B — Crash After Partial Persistence

Persist some records and terminate the worker. Expected: restart identifies incomplete work and does not duplicate already persisted records.

### Failure Drill C — Exhaust Retries

Make one item fail every attempt. Expected: bounded retries and final quarantine/dead-letter state without unnecessarily blocking remaining work.

### Failure Drill D — Fail One Tenant

Force one tenant to fail. Expected: other tenants complete and the failed tenant remains recoverable.

### Failure Drill E — Break Checkpoint Ordering

Advance progress before persistence. Expected: the test exposes data-loss risk and the implementation is corrected so durable persistence precedes progress advancement.

## 28. Observability

| Metric | Purpose |
|---|---|
| items processed | total work |
| items succeeded | completed work |
| items failed | incomplete work |
| retry count | transient instability |
| exhausted retries | unresolved failures |
| quarantined items | permanent failures |
| partial runs | operational health |
| recovery duration | recovery efficiency |
| failure rate | source/system quality |
| oldest pending item | recovery backlog |

Useful logs:

```text
run_id
item_id
attempt
status
error_class
source_position
worker_id
retryable
```

## 29. Recovery

When a partial failure occurs:

1. Identify the failure boundary.
2. Determine which work is durably successful.
3. Identify pending and failed work.
4. Classify failures.
5. Retry only retryable work.
6. Quarantine exhausted or permanent failures.
7. Reconcile successful output.
8. Reprocess incomplete work.
9. Verify final completion state.

Never restart blindly if durable item-level progress already exists.

## 30. Production Tools You Should Know

### PostgreSQL
Useful for durable item status, uniqueness, transactions, and recovery state.

### Celery
A task-processing framework that provides useful concepts for independent task retries and failure states.

### Apache Airflow
Useful for orchestration-level task states, retries, dependencies, and run management.

These tools provide production implementations, but the underlying problem is durable partial-progress management.

## 31. Production Runbook

### When a run becomes partial

1. Inspect run status.
2. Determine expected, successful, failed, and pending counts.
3. Identify failed item identities.
4. Classify failure causes.
5. Retry only eligible items.
6. Quarantine permanent failures.
7. Reconcile destination state.
8. Confirm no checkpoint has skipped incomplete work.

### When the worker crashes

- inspect durable item state
- identify processing items that may have ambiguous outcomes
- verify destination uniqueness
- safely retry ambiguous work where idempotency permits
- recalculate run status

### When failures are widespread

- stop aggressive retries
- inspect provider health and rate limits
- inspect schema and contract changes
- protect downstream systems
- preserve the last safe recovery point

### What not to do

- Do not mark a partially completed run as full success.
- Do not discard failed item identities.
- Do not retry permanent failures indefinitely.
- Do not duplicate successful work during recovery.
- Do not advance a checkpoint beyond unproven progress.
- Do not depend only on logs for recovery state.

## 32. Common Mistakes

1. Treating a batch as all-success or all-failure when item-level results exist.
2. Losing the identity of failed items.
3. Advancing checkpoints after only partial persistence.
4. Retrying permanent failures forever.
5. Reprocessing successful records without idempotency.
6. Marking a run successful because the process exited with code zero.
7. Restarting all tenants because one tenant failed.
8. Storing recovery state only in logs.
9. Ignoring ambiguous outcomes after crashes.
10. Failing to reconcile after recovery.
11. Using transactions without understanding their actual boundary.
12. Calling partial success ordinary success.

## 33. Definition of Done

- [ ] Failure boundaries are explicitly defined.
- [ ] Successful work is durably recorded.
- [ ] Failed work remains addressable.
- [ ] Retryable and permanent failures are classified.
- [ ] Partial progress is durable.
- [ ] Checkpoint semantics match the recovery model.
- [ ] Writes are idempotent where retries can repeat work.
- [ ] Run status represents partial completion accurately.
- [ ] Quarantine/dead-letter behavior exists where required.
- [ ] Worker-crash recovery is tested.
- [ ] Retry exhaustion is tested.
- [ ] Partial pagination behavior is defined.
- [ ] Tenant/resource-level failure behavior is defined where applicable.
- [ ] Metrics expose partial failures.
- [ ] Recovery and reconciliation procedures are documented.

## 34. What You Learned

After this recipe, you should be able to independently:
- identify partial failures at the correct boundary
- track independent work-unit state
- distinguish success, retryable failure, and permanent failure
- design page-atomic and record-independent recovery
- protect checkpoints during partial processing
- combine partial failure handling with idempotency
- use quarantine for exhausted work
- recover from worker crashes
- manage partial tenant and batch failures
- avoid false run success
- reprocess only incomplete work
- reconcile the final state
- operate partial-failure recovery in production

### Core Mental Model

```text
IDENTIFY WORK UNIT
       ↓
PROCESS
       ↓
SUCCESS OR FAILURE?
    /           \
 SUCCESS       FAILURE
    ↓              ↓
 PERSIST       CLASSIFY
    ↓          /       \
COMMIT      RETRY    QUARANTINE
    ↓          ↓
STATUS      RECOVER
    \          /
     FINAL RUN STATE
          ↓
      RECONCILE
```

> Partial failure is not an exception to the pipeline model. It is a normal production state that must have explicit progress, retry, recovery, and completion semantics.