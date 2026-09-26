# Recipe 22 — Bulk Loading

A pipeline can be logically correct and still be too slow for production. Loading one row at a time creates unnecessary network round trips, transaction overhead, and database work.

## 1. Problem Recognition

### The production problem

A naive loader may perform one database interaction and one commit for every record. For 1,000,000 records, this can create approximately 1,000,000 application/database interactions.

Recognize the problem when you see:
- one INSERT per record
- one COMMIT per record
- low rows/second
- high database round-trip counts
- long-running loads despite low worker CPU
- large files taking unexpectedly long
- database connections busy mostly waiting on individual operations.

Measure the baseline:

rows_per_second = rows_processed / elapsed_seconds

### The key question

> Can I move many records through the database as a bounded unit without creating an unsafe transaction?

That is the core problem this recipe solves.

## 2. Concept and Reasoning

Bulk loading means sending many records as a controlled batch rather than performing an independent database operation for every row.

```text
source
  ↓
bounded batch
  ↓
bulk write
  ↓
transaction
  ↓
commit
  ↓
next batch
```

### Batch size

A batch limits both memory usage and failure scope.

Too small:
- more round trips
- more transaction overhead
- lower throughput.

Too large:
- higher memory usage
- longer transactions
- larger rollback scope
- more expensive recovery.

There is no universal batch size. Measure it.

### Bulk loading is not one giant transaction

Prefer:

```text
batch 1 → transaction → commit
batch 2 → transaction → commit
batch 3 → transaction → commit
```

rather than keeping an enormous dataset inside one transaction.

### Bulk loading and correctness

Bulk loading does not replace:
- validation
- idempotency
- retry handling
- checkpointing
- reconciliation.

It combines with those mechanisms.

## 3. Implementation

Create:

```text
bulk-loading/
├── loader.py
└── test_loader.py
```

### 3.1 Memory-safe batching

```python
from collections.abc import Iterable, Iterator
from typing import TypeVar

T = TypeVar('T')

def batches(items: Iterable[T], batch_size: int) -> Iterator[list[T]]:
    if batch_size <= 0:
        raise ValueError('batch_size must be greater than zero')

    batch: list[T] = []

    for item in items:
        batch.append(item)
        if len(batch) == batch_size:
            yield batch
            batch = []

    if batch:
        yield batch
```

This keeps only the current batch in memory.

### 3.2 Transactional batch loader

```python
def load_batches(connection, rows, batch_size=1000):
    total_loaded = 0

    for batch in batches(rows, batch_size):
        with connection:
            with connection.cursor() as cursor:
                for row in batch:
                    cursor.execute(
                        '''
                        INSERT INTO payments (event_id, amount, currency)
                        VALUES (%s, %s, %s)
                        ON CONFLICT (event_id) DO NOTHING
                        ''',
                        (row.event_id, row.amount, row.currency),
                    )

        total_loaded += len(batch)

    return total_loaded
```

This gives bounded memory and bounded transaction scope. For very large loads, use the database's native bulk-loading mechanism rather than executing one statement per row.

### 3.3 PostgreSQL COPY

PostgreSQL COPY is designed for high-throughput loading.

Conceptually:

```text
source
  ↓
prepare rows
  ↓
COPY
  ↓
PostgreSQL
```

With psycopg, a COPY operation can be used to stream rows into PostgreSQL without issuing an individual INSERT for every record.

Understand the important distinction:

> Bulk loading improves throughput; it does not automatically solve invalid data, duplicates, partial failures, or reconciliation.

## 4. Testing

Test:
- correct batch sizes
- empty input
- invalid batch size
- final partial batch
- duplicate records
- batch rollback
- recovery after a failed batch.

Example:

```python
def test_batches():
    result = list(batches(range(10), 3))
    assert [len(batch) for batch in result] == [3, 3, 3, 1]

def test_empty_input():
    assert list(batches([], 100)) == []

def test_partial_batch():
    result = list(batches(range(7), 3))
    assert result == [[0, 1, 2], [3, 4, 5], [6]]
```

Run:

```bash
pytest -q
```

Also integration-test a real database and verify that a failed batch leaves no partial committed state.

## 5. Observability

Track:

```text
records_read_total
records_loaded_total
records_failed_total
batches_processed_total
batch_duration_seconds
load_duration_seconds
rows_per_second
batch_size
```

Calculate throughput as:

```text
rows_per_second = loaded_rows / elapsed_seconds
```

Useful batch-level events should include batch number, row count, duration, status, and safe failure classification.

Do not log entire batches or sensitive payloads.

Watch for:
- falling throughput
- increasing batch duration
- increasing rollback rate
- database lock waits
- connection pool pressure
- growing source backlog.

## 6. Intentional Failure

Deliberately fail Batch 3 after Batch 1 and Batch 2 have committed.

Expected:

```text
Batch 1 → committed
Batch 2 → committed
Batch 3 → rolled back
Batch 4 → not processed yet
```

Verify the database contains no partial records from Batch 3.

Then deliberately use an excessively large batch and observe memory usage, transaction duration, throughput, and rollback cost.

## 7. Recovery

When a batch fails:

1. Identify the failed batch using a batch ID, source range, or checkpoint.
2. Confirm that its transaction rolled back.
3. Classify the failure.
4. Fix the root cause.
5. Retry only the affected batch.
6. Continue from the correct source position.
7. Reconcile source and target.

Safe progression:

```text
Batch 1 ✓
Batch 2 ✓
Batch 3 ✗
Batch 4 not started

       recovery

Batch 3 ✓
Batch 4 ✓
Batch 5 ✓
```

Do not restart the entire dataset unless there is a deliberate reason.

## 8. Batch Size Tuning

Test several sizes such as:

```text
1,000
5,000
10,000
25,000
50,000
```

Measure throughput, memory, transaction duration, rollback cost, database load, and lock contention.

> The objective is the highest operationally safe throughput, not the largest possible batch.

## 9. Bulk Loading and Idempotency

A worker can crash after a batch commits but before its checkpoint advances.

```text
batch commits
    ↓
worker crashes
    ↓
batch runs again
```

Without idempotency, duplicates can appear.

Use the stable event identity and the idempotency mechanism from Recipe 6.

Bulk loading also connects directly to Recipes 10, 12, and 16: replay, checkpointing, and reconciliation.

## 10. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **PostgreSQL COPY** | Native high-throughput PostgreSQL loading mechanism. |
| **Apache Spark** | Distributed processing and writing for datasets that exceed a single worker's practical capacity. |
| **dbt** | Set-based warehouse transformations and incremental model loading. |

> These tools provide production implementations of high-volume loading. Understand batching, transaction scope, throughput, and recovery first.

## 11. Production Runbook

### Slow loading

Check:
1. rows/second
2. batch size
3. transaction duration
4. database CPU and I/O
5. lock waits
6. indexes
7. network latency
8. connection pool usage.

### Failed batch

Check:
1. batch identifier
2. source range
3. error type
4. rollback status
5. partial-write evidence
6. retryability
7. quarantine requirements.

### High memory

Check batch size, source buffering, serialization, and worker concurrency. Reduce batch size before simply adding more memory.

Do not:
- commit every row
- create one transaction for an enormous dataset
- increase batch size without measuring
- retry without understanding the failure
- checkpoint before successful commit.

## 12. Common Mistakes

### One commit per row
Creates excessive transaction overhead.

### One giant transaction
Makes failures expensive and increases lock duration.

### Guessing batch size
Measure instead.

### No idempotency
Retries can duplicate data.

### Checkpoint before commit
A crash can permanently skip data.

### Measuring only total runtime
Measure throughput and database impact together.

## 13. Definition of Done

You are done when you can:
- recognize inefficient row-by-row loading
- explain why batching improves throughput
- implement memory-safe batching
- choose a bounded transaction scope
- test complete and partial batches
- test rollback
- measure rows/second
- observe batch failures and duration
- intentionally fail a batch
- prove it rolled back
- recover from the failed batch
- resume from the correct position
- combine bulk loading with idempotency
- explain PostgreSQL COPY, Spark, and dbt
- tune batch size using evidence
- reconcile source and target.

## 14. What You Learned

The central principle is:

> **Move data in bounded, measurable units rather than treating every row or the entire dataset as one operation.**

Reliable bulk loading looks like:

```text
SOURCE
  ↓
READ BOUNDED BATCH
  ↓
VALIDATE
  ↓
BULK WRITE
  ↓
COMMIT
  ↓
CHECKPOINT
  ↓
NEXT BATCH
```

When a batch fails:

```text
FAILED BATCH
     ↓
ROLLBACK
     ↓
DIAGNOSE
     ↓
FIX
     ↓
RETRY
     ↓
RECONCILE
```

A fast pipeline that cannot recover safely is not production-grade.