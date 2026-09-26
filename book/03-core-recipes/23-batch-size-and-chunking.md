# Recipe 23 — Batch Size & Chunking

Bulk loading introduced bounded batches. This recipe goes deeper: how to divide a large workload into safe chunks, how to choose chunk boundaries, how chunk size affects memory and throughput, and how to recover when one chunk fails.

## 1. Problem Recognition

Large datasets cannot always be processed as one unit. You need a way to divide work into predictable pieces.

Recognize a chunking problem when:
- the full dataset does not fit comfortably in memory
- one operation takes too long
- failures force expensive full-job restarts
- workers need bounded workloads
- parallel processing is required
- source APIs impose page or record limits
- database queries become expensive when the requested range is too large

The goal is not simply to make chunks. Chunks should be bounded, deterministic, recoverable, observable, and safe to retry.

## 2. Concept and Reasoning

### Chunk vs batch

- **Chunk:** a bounded portion of a larger dataset or workload.
- **Batch:** a chunk treated as one processing unit.

Example:

```text
1,000,000 records
       ↓
chunks of 10,000
       ↓
100 processing units
```

Chunk size affects:

```text
chunk size
   ├── memory usage
   ├── transaction duration
   ├── failure scope
   ├── retry cost
   ├── throughput
   └── parallelism
```

Larger chunks usually reduce per-chunk overhead but increase failure and resource cost. Smaller chunks improve recovery granularity but can create excessive scheduling and I/O overhead.

### Deterministic chunking

A production pipeline should be able to answer:

> Which exact records belong to chunk 417?

For example:

```text
source_id = payments
partition = 2026-09-26
offset_start = 4,170,000
offset_end = 4,179,999
```

A chunk should not silently change membership between retries.

## 3. Implementation

### 3.1 Fixed-size chunks

```python
from collections.abc import Iterable, Iterator
from typing import TypeVar

T = TypeVar("T")


def chunked(items: Iterable[T], size: int) -> Iterator[list[T]]:
    if size <= 0:
        raise ValueError("size must be greater than zero")

    chunk: list[T] = []

    for item in items:
        chunk.append(item)

        if len(chunk) == size:
            yield chunk
            chunk = []

    if chunk:
        yield chunk
```

Example:

```python
list(chunked(range(10), 4))
```

Produces:

```text
[0, 1, 2, 3]
[4, 5, 6, 7]
[8, 9]
```

### 3.2 Give every chunk an identity

Do not treat a production chunk as only a list of records.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Chunk:
    chunk_id: str
    start: int
    end: int
```

Chunk identity allows logs, checkpoints, retries, and reconciliation to refer to the same processing unit.

### 3.3 Range-based chunking

```python
def ranges(start: int, end: int, size: int):
    if size <= 0:
        raise ValueError("size must be greater than zero")

    current = start

    while current <= end:
        chunk_end = min(current + size - 1, end)
        yield current, chunk_end
        current = chunk_end + 1
```

Example:

```text
1–100
101–200
201–300
...
```

Range-based chunking is useful because a failed chunk can be named precisely.

## 4. Choosing Chunk Boundaries

Not every source should be chunked by row count.

| Strategy | Example |
|---|---|
| Row count | 10,000 records |
| ID range | IDs 1,000,000–1,009,999 |
| Time range | 10:00–10:05 |
| File | one input file |
| Partition | one Kafka/database partition |
| Page | API page of 500 records |

Choose a boundary that is stable and meaningful for the source.

### Time-based chunks

```text
2026-09-26 00:00–01:00
2026-09-26 01:00–02:00
2026-09-26 02:00–03:00
```

These are useful for event and warehouse workloads, but the pipeline must account for late-arriving data.

### ID-based chunks

ID ranges are useful when IDs are indexed and reasonably distributed.

Do not assume ID ranges contain equal numbers of rows. Gaps are normal.

## 5. Testing

Test the chunking mechanism independently from the processing logic.

### Test 1 — Exact division

```text
100 records / 10
→ 10 chunks
```

### Test 2 — Partial final chunk

```text
103 records / 10
→ 10 full chunks + 1 chunk of 3
```

### Test 3 — Empty input

Expected: zero chunks.

### Test 4 — Invalid size

```text
size = 0
size = -1
```

Expected: `ValueError`.

### Test 5 — No overlap

Every record must belong to at most one chunk.

### Test 6 — No gaps

Every source record intended for processing must belong to exactly one chunk.

### Test 7 — Deterministic retry

Running chunk generation twice against the same fixed source range must produce the same boundaries.

Example:

```python
def test_no_overlap():
    result = list(chunked(range(10), 3))
    flattened = [x for chunk in result for x in chunk]

    assert len(flattened) == len(set(flattened))
    assert flattened == list(range(10))
```

## 6. Observability

Track each chunk as a first-class processing unit.

Useful fields:

```text
chunk_id
chunk_size
source_start
source_end
started_at
completed_at
duration
status
records_read
records_written
records_failed
worker_id
retry_count
```

Useful metrics:

```text
chunks_started_total
chunks_completed_total
chunks_failed_total
chunk_duration_seconds
chunk_records_total
chunk_retries_total
```

If one chunk takes 20 seconds and another takes 5 minutes, the problem may be data skew rather than chunk size.

Investigate:
- uneven ID distribution
- expensive records
- hot partitions
- database contention

## 7. Intentional Failure

Create 10 chunks and deliberately fail chunk 6.

Expected:

```text
1 ✓
2 ✓
3 ✓
4 ✓
5 ✓
6 ✗
7 not processed
8 not processed
9 not processed
10 not processed
```

Then retry only chunk 6.

Verify:

```text
1 ✓
2 ✓
3 ✓
4 ✓
5 ✓
6 ✓
```

### Duplicate execution drill

Run the same chunk twice.

Verify that the destination remains correct when the downstream operation is idempotent.

## 8. Recovery

Suppose:

```text
Chunk 41 ✓
Chunk 42 ✓
Chunk 43 ✗
Chunk 44 not started
```

Recovery:

1. identify chunk 43
2. inspect its source boundary
3. determine why it failed
4. confirm previous chunks are committed
5. fix the failure
6. retry chunk 43
7. continue with chunk 44
8. reconcile the final result

Do not regenerate arbitrary chunks after a failure if that can change membership.

## 9. Chunking and Parallelism

Once work is divided into independent chunks, chunks can potentially run concurrently.

```text
             ┌─ Chunk 1 → Worker A
SOURCE ──────┼─ Chunk 2 → Worker B
             ├─ Chunk 3 → Worker C
             └─ Chunk 4 → Worker D
```

But chunking does not automatically make parallelism safe.

Consider:
- database connection limits
- destination contention
- ordering requirements
- rate limits
- shared resources
- partition ownership

Start sequentially. Add concurrency only when measurements justify it.

## 10. Chunk Size Tuning

Run controlled experiments:

```text
1,000
5,000
10,000
25,000
50,000
```

Measure:

```text
throughput
memory
CPU
database load
duration
failure/retry cost
```

Do not tune solely for maximum throughput.

A chunk size that increases throughput by 5% but makes failures ten times more expensive may be operationally worse.

## 11. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Apache Spark** | Splits distributed workloads into partitions and tasks. |
| **Apache Kafka** | Uses partitions and offsets as durable units of parallel event consumption. |
| **Apache Airflow** | Orchestrates bounded task units and coordinates large workflows. |

> These tools implement production-scale forms of workload partitioning and orchestration. First understand deterministic boundaries, failure scope, and resource limits.

## 12. Production Runbook

### When chunks are too slow

Check:
1. chunk size
2. source distribution
3. database latency
4. lock contention
5. worker utilization
6. downstream limits

### When memory is too high

Check:
- records retained in memory
- serialization buffers
- chunk size
- concurrency

### When one chunk is much slower

Investigate data skew before simply increasing every chunk size.

### When a chunk fails

Check:
1. exact chunk identity
2. source boundaries
3. retry count
4. transaction state
5. destination state

### What not to do

Do not:
- create overlapping chunk boundaries
- leave gaps between chunks
- generate different boundaries on retry
- increase concurrency without checking downstream capacity
- assume equal row counts mean equal processing cost

## 13. Common Mistakes

### Mistake 1 — Arbitrary boundaries

Chunks cannot be reliably resumed.

### Mistake 2 — Overlapping chunks

The same records can be processed multiple times.

### Mistake 3 — Gaps

Records silently disappear from the workload.

### Mistake 4 — Huge chunks

Failures become expensive and memory usage increases.

### Mistake 5 — Tiny chunks

Scheduling and I/O overhead dominate useful work.

### Mistake 6 — Ignoring skew

Some chunks can become much more expensive than others.

### Mistake 7 — Adding parallelism too early

Concurrency can move the bottleneck to the database or another shared dependency.

## 14. Definition of Done

You are done when you can:
- recognize when a workload needs chunking
- distinguish chunks from transactions
- implement fixed-size chunking from scratch
- implement deterministic range-based chunking
- choose meaningful boundaries
- prove chunks have no gaps
- prove chunks have no overlaps
- test partial final chunks
- identify a failed chunk precisely
- retry one failed chunk
- observe chunk-level performance
- detect data skew
- tune chunk size using measurements
- explain how chunking enables controlled parallelism
- explain how Spark, Kafka, and Airflow relate to the concept

## 15. What You Learned

The central principle is:

> **A good chunk is a deterministic, bounded, observable unit of work that can be retried independently.**

The production pattern is:

```text
SOURCE
  ↓
DEFINE STABLE BOUNDARY
  ↓
CREATE CHUNK
  ↓
PROCESS
  ↓
COMMIT
  ↓
RECORD RESULT
  ↓
NEXT CHUNK
```

When a chunk fails:

```text
IDENTIFY EXACT CHUNK
        ↓
DIAGNOSE
        ↓
FIX
        ↓
RETRY SAME BOUNDARY
        ↓
VERIFY
        ↓
CONTINUE
```

Chunking is not merely a performance trick. It is a fundamental mechanism for controlling memory, failure scope, recovery cost, and scalable execution.
