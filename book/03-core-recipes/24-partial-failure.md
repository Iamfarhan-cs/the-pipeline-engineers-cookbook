# Recipe 24 — Partial Failure

Production pipelines rarely fail completely.

More often:

- 99,000 records succeed
- 1,000 records fail
- one chunk succeeds
- another chunk fails
- one API request times out
- one database operation violates a constraint

This is **partial failure**.

The engineering problem is to preserve successful work, isolate failed work, record exactly what happened, and recover without duplicating or losing data.

---

## 1. Problem Recognition

### The production problem

Consider:

```text
100,000 records
      ↓
10 chunks
      ↓
Chunk 1 ✓
Chunk 2 ✓
Chunk 3 ✓
Chunk 4 ✗
Chunk 5 ✓
Chunk 6 ✓
...
```

The pipeline has not completely failed.

It has produced a **mixed outcome**.

A dangerous implementation may respond by:

```text
one failure
   ↓
restart everything
   ↓
duplicate successful work
   ↓
longer recovery
```

Another dangerous implementation may ignore the failure:

```text
99,000 succeeded
1,000 failed
   ↓
pipeline reports SUCCESS
```

Both are incorrect.

### How do you recognize partial failure?

Look for:

- success and failure counts that differ
- individual records failing validation
- individual API calls timing out
- database constraints affecting only some records
- failed chunks among successful chunks
- downstream systems accepting only part of a request
- retries affecting only a subset of work.

The pipeline needs to distinguish:

```text
SUCCESS
PARTIAL_SUCCESS
FAILURE
```

---

## 2. Concept and Reasoning

### All-or-nothing vs partial success

Some operations require atomicity.

For example:

```text
debit account
credit account
```

These belong in one transaction when the business operation requires atomicity.

But a batch containing unrelated payments may legitimately produce:

```text
Payment A ✓
Payment B ✓
Payment C ✗
Payment D ✓
```

The correct behavior depends on the logical unit of work.

### The first question

Before implementing partial failure handling, ask:

> What is the smallest unit that must succeed or fail atomically?

It could be:

- one record
- one payment
- one file
- one chunk
- one database transaction
- one business operation.

Do not use partial success where atomicity is required.

### Failure isolation

A good design isolates failures:

```text
INPUT
  ↓
CHUNK
  ↓
PROCESS
  ├── SUCCESS → TARGET
  │
  └── FAILURE → FAILURE STORE
```

The failed work should remain identifiable and recoverable.

---

## 3. Define an Explicit Outcome Model

Do not represent everything with a single boolean.

Use explicit states:

```text
PENDING
PROCESSING
SUCCEEDED
FAILED
RETRYABLE
QUARANTINED
```

For a batch:

```text
batch_status =
    PENDING
    PROCESSING
    SUCCEEDED
    PARTIAL_SUCCESS
    FAILED
```

A batch containing:

```text
950 successful
50 failed
```

should not be represented as simply:

```text
SUCCESS
```

A useful summary is:

```text
input = 1000
success = 950
failed = 50
```

And:

```text
success_rate = 950 / 1000 = 95%
```

---

## 4. Implementation

### 4.1 Result model

Create an explicit processing result:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class RecordResult:
    record_id: str
    status: str
    error_type: str | None = None
    error_message: str | None = None
```

A batch can then aggregate these results.

### 4.2 Batch summary

```python
@dataclass(frozen=True)
class BatchResult:
    batch_id: str
    total: int
    succeeded: int
    failed: int

    @property
    def status(self) -> str:
        if self.failed == 0:
            return "SUCCEEDED"

        if self.succeeded == 0:
            return "FAILED"

        return "PARTIAL_SUCCESS"
```

This makes the state explicit.

### 4.3 Process records independently

For workloads where records are independent:

```python
def process_records(records, process_one):
    results = []

    for record in records:
        try:
            process_one(record)

            results.append(
                RecordResult(
                    record_id=record.id,
                    status="SUCCEEDED",
                )
            )

        except Exception as exc:
            results.append(
                RecordResult(
                    record_id=record.id,
                    status="FAILED",
                    error_type=type(exc).__name__,
                    error_message=str(exc),
                )
            )

    return results
```

The important property is that one failed record does not automatically destroy successful independent records.

### 4.4 Aggregate results

```python
def summarize(batch_id, results):
    succeeded = sum(
        result.status == "SUCCEEDED"
        for result in results
    )

    failed = sum(
        result.status == "FAILED"
        for result in results
    )

    return BatchResult(
        batch_id=batch_id,
        total=len(results),
        succeeded=succeeded,
        failed=failed,
    )
```

Example:

```text
Input: 10 records

Success: 8
Failure: 2

Batch status:
PARTIAL_SUCCESS
```

---

## 5. Failure Classification

Not every failure should be retried.

Classify failures.

### Retryable

Examples:

- temporary network failure
- connection timeout
- database temporarily unavailable
- HTTP 503
- rate-limit response

Potential action:

```text
retry
```

### Non-retryable

Examples:

- invalid schema
- malformed identifier
- impossible business value
- invalid currency
- violated permanent business rule

Potential action:

```text
quarantine
```

### Unknown

If the pipeline cannot classify the failure safely:

```text
UNKNOWN
   ↓
investigate
   ↓
do not blindly retry forever
```

---

## 6. Testing

Test both successful and failed records.

### Test 1 — All success

```text
10 input
10 success
0 failure

→ SUCCEEDED
```

### Test 2 — All failure

```text
10 input
0 success
10 failure

→ FAILED
```

### Test 3 — Partial failure

```text
10 input
8 success
2 failure

→ PARTIAL_SUCCESS
```

### Test 4 — One record fails

Verify that successful independent records remain committed.

### Test 5 — Retryable failure

Verify the failed record is retried.

### Test 6 — Permanent failure

Verify the record is not retried indefinitely and is moved to the appropriate failure state.

### Example pytest

```python
def test_partial_success():
    results = [
        RecordResult("1", "SUCCEEDED"),
        RecordResult("2", "SUCCEEDED"),
        RecordResult("3", "FAILED", "ValidationError"),
    ]

    result = summarize("batch-1", results)

    assert result.total == 3
    assert result.succeeded == 2
    assert result.failed == 1
    assert result.status == "PARTIAL_SUCCESS"
```

---

## 7. Observability

Partial failure requires more than a single pipeline status.

Track:

```text
records_total
records_succeeded
records_failed
records_retryable
records_quarantined
batches_total
batches_succeeded
batches_partial
batches_failed
```

Useful rates:

```text
success_rate
failure_rate
retry_rate
quarantine_rate
```

### Structured failure event

```json
{
  "event": "record_failed",
  "record_id": "evt-123",
  "batch_id": "batch-17",
  "status": "RETRYABLE",
  "error_type": "TimeoutError"
}
```

Do not put sensitive payloads into logs.

### Operational alert

A pipeline can be technically running while data quality is deteriorating.

For example:

```text
pipeline_status = RUNNING
failure_rate = 18%
```

The first signal says the process is alive.

The second says the data flow may be unhealthy.

Both matter.

---

## 8. Intentional Failure

Create a batch containing 10 independent records.

Force:

```text
Record 3 → failure
Record 7 → failure
```

Expected:

```text
1 ✓
2 ✓
3 ✗
4 ✓
5 ✓
6 ✓
7 ✗
8 ✓
9 ✓
10 ✓
```

The final result must be:

```text
total = 10
success = 8
failed = 2
status = PARTIAL_SUCCESS
```

### Second drill

Make one failure retryable.

Then:

```text
initial:
8 success
2 failure

retry:
1 succeeds
1 remains failed

final:
9 success
1 failed
```

Verify that the successful records are not processed again unnecessarily.

---

## 9. Recovery

Suppose:

```text
1000 records

950 succeeded
50 failed
```

Recovery should not automatically restart all 1000 records.

### Step 1 — Persist the outcome

Record:

```text
batch_id
record_id
status
error_type
attempt_count
```

### Step 2 — Separate failures

```text
50 failed
   ↓
classify
   ├── 35 retryable
   └── 15 permanent
```

### Step 3 — Retry retryable records

```text
35 retryable
   ↓
retry
   ↓
30 success
5 still failing
```

### Step 4 — Quarantine permanent failures

The five remaining retryable records may eventually become:

```text
QUARANTINED
```

depending on retry policy.

### Step 5 — Reconcile

Final accounting must explain every input record:

```text
input
=
success
+
retry_pending
+
quarantined
+
known_failed
```

There must be no unexplained remainder.

---

## 10. Partial Failure and Transactions

Recipe 21 established transaction boundaries.

The key question remains:

> What must be atomic?

Suppose a single payment requires:

```text
create payment
update balance
write audit record
```

These may belong in one transaction.

But 1,000 unrelated payments do not necessarily need one transaction.

A useful design is:

```text
Batch
 ├── Payment A → transaction → success
 ├── Payment B → transaction → success
 ├── Payment C → transaction → failure
 └── Payment D → transaction → success
```

Only use this pattern when the records are genuinely independent.

---

## 11. Partial Failure and Idempotency

Partial failure often creates retries.

Example:

```text
Record 500
   ↓
destination write succeeds
   ↓
response is lost
   ↓
worker marks it failed
   ↓
record is retried
```

Without idempotency:

```text
duplicate
```

With a stable idempotency key:

```text
same event_id
   ↓
existing result detected
   ↓
no duplicate logical effect
```

Partial failure and idempotency therefore belong together.

---

## 12. Partial Failure and Chunking

Recipe 23 introduced chunks.

A chunk can itself have partial failure:

```text
Chunk 10
   ├── 990 success
   └── 10 failure
```

The chunk should expose that outcome rather than hiding it.

Possible lifecycle:

```text
PROCESSING
    ↓
PARTIAL_SUCCESS
    ↓
retry failed records
    ↓
SUCCEEDED
```

Or:

```text
PARTIAL_SUCCESS
    ↓
permanent failures remain
    ↓
QUARANTINED
```

---

## 13. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Apache Kafka** | Consumer and producer failures can be handled at message/partition boundaries with offsets and retry patterns. |
| **Apache Flink** | Supports stateful stream processing and failure recovery at distributed processing boundaries. |
| **PostgreSQL** | Provides transaction boundaries that can isolate independent units of database work. |

> These tools provide production mechanisms for handling failures at different processing boundaries. The underlying principle is still failure isolation plus explicit outcome accounting.

---

## 14. Production Runbook

### When a pipeline reports PARTIAL_SUCCESS

Check:

1. total input
2. successful count
3. failed count
4. retryable count
5. quarantined count
6. failure categories
7. affected batches
8. retry attempts.

### When failures increase suddenly

Check:

- dependency availability
- schema changes
- database constraints
- API response codes
- source-data changes
- deployment changes
- rate limits.

### When retry volume grows

Check whether:

```text
retryable failure
        ↓
retry
        ↓
same failure
        ↓
retry again
        ↓
same failure
```

is becoming an infinite loop.

Set retry limits.

### What not to do

Do not:

- mark a partial result as complete
- restart successful records unnecessarily
- retry permanent failures forever
- discard failed records without recording them
- hide failure counts inside a generic error
- use partial success when the business operation requires atomicity.

---

## 15. Common Mistakes

### Mistake 1 — Treating every failure as total failure

This causes unnecessary reprocessing.

### Mistake 2 — Treating every failure as retryable

Permanent data errors will never recover through retries.

### Mistake 3 — Losing failed records

A failure without an audit trail becomes data loss.

### Mistake 4 — No explicit PARTIAL_SUCCESS state

Operators cannot understand what actually happened.

### Mistake 5 — No failure classification

Retry and quarantine decisions become arbitrary.

### Mistake 6 — Ignoring business atomicity

Some operations must succeed or fail together.

### Mistake 7 — No reconciliation

The pipeline cannot prove that every input has a known final state.

---

## 16. Definition of Done

You are done when you can:

- recognize partial failure in a real pipeline
- distinguish total failure from partial failure
- identify the correct atomic unit
- model explicit processing outcomes
- implement per-record failure isolation
- classify retryable and permanent failures
- test all-success, all-failure, and mixed outcomes
- intentionally create partial failure
- observe failure rates and categories
- recover only the failed work
- prevent unnecessary reprocessing
- combine partial failure handling with idempotency
- combine partial failure handling with chunking
- reconcile every input record
- explain when partial success is unsafe because atomicity is required
- explain how PostgreSQL, Kafka, and Flink relate to the concept.

---

## 17. What You Learned

The central principle is:

> **A production pipeline must know exactly which work succeeded, which work failed, why it failed, and what should happen next.**

The pattern is:

```text
INPUT
  ↓
PROCESS INDEPENDENT UNITS
  ↓
┌───────────────┐
│ SUCCESS       │ → TARGET
│ RETRYABLE     │ → RETRY
│ PERMANENT     │ → QUARANTINE
└───────────────┘
  ↓
RECONCILE
```

Partial failure is not an exception to production pipeline design.

It is one of the normal states that a reliable pipeline must be designed to handle.
