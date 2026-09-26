# E61 — Large File Streaming

## 1. Problem Recognition

A file can be perfectly valid and still be too large for the available memory.

Examples:

- 20 GB CSV;
- 8 GB JSONL;
- 15 GB XML;
- 50 GB compressed delivery that expands to hundreds of GB;
- large database exports;
- multi-gigabyte Avro files.

The dangerous assumption is:

```
file size <= available memory
```

A production extractor should instead use:

```
SOURCE
  |
  v
STREAM
  |
  v
BOUNDED BUFFER
  |
  v
PARSE
  |
  v
VALIDATE
  |
  v
BATCH
  |
  v
COMMIT
  |
  v
CHECKPOINT
```

Core rule:

> Process data in bounded memory regardless of total source size.

---

## 2. Why Large Files Break Naive ETL

A naive implementation may do:

```
data = file.read()
```

or:

```
records = list(reader(file))
```

If the file contains 100 million records, memory consumption can become enormous.

The correct question is not:

> How large is the file?

It is:

> How much data must the process hold at one time?

A streaming design makes the working set bounded.

---

## 3. Memory Model

Suppose:

```
file size       = 100 GB
available RAM   = 4 GB
batch size      = 5,000 records
```

The pipeline can still process the file because it does not need to hold the complete source in memory.

Conceptually:

```
100 GB source
    |
    v
small read buffer
    |
    v
5,000-record batch
    |
    v
database commit
    |
    v
clear batch
    |
    v
next records
```

The source can be much larger than RAM.

---

## 4. Streaming vs Chunking

These concepts are related but not identical.

### Streaming

Read records progressively:

```
for record in reader:
    process(record)
```

### Chunking

Read a bounded group of records or bytes:

```
records = next_batch(reader, 5000)
process(records)
```

A production pipeline commonly combines them:

```
stream source
    |
    v
bounded batch
    |
    v
process
    |
    v
commit
```

---

## 5. Choose the Correct Streaming Boundary

The streaming boundary depends on the format.

### Line-oriented

CSV and JSONL can usually stream by line or record.

### Binary record formats

Avro can stream records using an Avro reader.

### Columnar formats

Parquet is normally processed through row groups, record batches, and column projection rather than arbitrary byte chunks.

### XML

Large XML documents require event-based parsing such as `iterparse`.

The principle is:

> Stream at the semantic record boundary whenever possible.

---

## 6. Byte Streaming Is Not Record Streaming

This distinction matters.

Suppose a CSV record contains:

```
"customer","large
multi-line
description"
```

Reading arbitrary 1 MB byte chunks can split a logical record.

Likewise, JSON objects may contain nested structures spanning many bytes.

Therefore:

```
byte chunks
    |
    v
format-aware parser
    |
    v
records
```

Do not assume a byte chunk is a complete record.

---

## 7. Python File Streaming

Python file objects already support bounded iteration.

```
from pathlib import Path

with Path("large.csv").open(
    "rt",
    encoding="utf-8",
    newline="",
) as file:
    for line in file:
        process(line)
```

The program does not need to read the entire file into memory.

For simple line-oriented data, this may be sufficient.

---

## 8. Streaming CSV

Use the CSV parser rather than manually splitting lines.

```
import csv

with open(
    "large.csv",
    "r",
    encoding="utf-8",
    newline="",
) as file:
    reader = csv.DictReader(file)

    for row in reader:
        process(row)
```

The CSV reader handles quoting, delimiters, and embedded newlines.

A manual `line.split(",")` implementation is not production CSV parsing.

---

## 9. Streaming JSONL

JSONL naturally supports record-by-record processing.

```
import json

with open(
    "large.jsonl",
    "r",
    encoding="utf-8",
) as file:
    for line_number, line in enumerate(file, start=1):
        if not line.strip():
            continue

        record = json.loads(line)
        process(record)
```

Memory remains approximately proportional to:

```
largest record
+
parser overhead
+
current batch
```

rather than total file size.

---

## 10. Large JSON Is Different

A normal JSON document may contain one huge array:

```
[
  {...},
  {...},
  {...}
]
```

Using:

```
json.load(file)
```

can load the entire document.

For very large JSON documents, use a streaming JSON parser or change the source contract to JSONL when possible.

A useful design is:

```
large JSON document
       |
       v
streaming JSON parser
       |
       v
individual records
```

---

## 11. Streaming XML

XML can contain deeply nested structures.

A practical Python approach is event-based parsing:

```
import xml.etree.ElementTree as ET

for event, element in ET.iterparse(
    "large.xml",
    events=("end",),
):
    if element.tag == "payment":
        process(element)
        element.clear()
```

`element.clear()` releases processed element content so memory does not grow with the document.

The exact clearing strategy depends on parent-child relationships.

---

## 12. Large Avro Files

Avro Object Container Files are designed around blocks and records.

Use a streaming reader:

```
from fastavro import reader

with open("large.avro", "rb") as file:
    for record in reader(file):
        process(record)
```

Avoid:

```
records = list(reader(file))
```

The second approach defeats the memory benefits of the format.

---

## 13. Large Parquet Files

Parquet should normally be processed using its columnar execution model.

Use:

- column projection;
- row-group processing;
- predicate pushdown;
- dataset scanning;
- record batches.

Example:

```
import pyarrow.dataset as ds

dataset = ds.dataset("payments.parquet")

scanner = dataset.scanner(
    columns=["payment_id", "amount"],
)

for batch in scanner.to_batches():
    process_batch(batch)
```

This is different from treating Parquet as a raw byte stream.

---

## 14. Bounded Batch Processing

Streaming records one at a time is memory-safe, but database writes may become inefficient.

Use bounded batches:

```
stream
  |
  v
5000 records
  |
  v
validate
  |
  v
database transaction
  |
  v
commit
  |
  v
clear
  |
  v
next 5000
```

Batch size is an operational parameter, not a universal constant.

Measure:

- memory;
- transaction duration;
- throughput;
- database load;
- recovery cost.

---

## 15. Generic Batch Iterator

A reusable pattern:

```
from collections.abc import Iterable, Iterator
from typing import TypeVar

T = TypeVar("T")

def batches(
    items: Iterable[T],
    size: int,
) -> Iterator[list[T]]:
    batch: list[T] = []

    for item in items:
        batch.append(item)

        if len(batch) == size:
            yield batch
            batch = []

    if batch:
        yield batch
```

Then:

```
for batch in batches(records, 5000):
    write_batch(batch)
```

This keeps memory bounded by the selected batch size plus parser overhead.

---

## 16. Memory Budgeting

Do not choose batch size blindly.

Suppose:

```
available memory       = 2 GB
application overhead  = 500 MB
safe working memory   = 1 GB
average record size   = 20 KB
```

A theoretical upper bound is:

```
1 GB / 20 KB
```

But Python objects have overhead, database drivers buffer data, parsers allocate memory, and the operating system needs memory.

Therefore leave a safety margin.

Measure actual resident memory during realistic loads.

---

## 17. Backpressure

Backpressure occurs when downstream processing is slower than upstream reading.

Example:

```
disk/network
    |
    v
fast reader
    |
    v
parser
    |
    v
slow database
```

If the reader continues without limits, memory can grow because unread records accumulate.

A bounded queue provides flow control:

```
reader
  |
  v
bounded queue
  |
  v
processor
  |
  v
database
```

When the queue is full, the producer must wait.

---

## 18. Bounded Queues

A simple threaded architecture:

```
producer
   |
   v
Queue(maxsize=N)
   |
   v
consumer
   |
   v
database
```

The important property is `maxsize`.

An unbounded queue simply moves the memory problem from a list to another data structure.

Backpressure is a memory-safety mechanism.

---

## 19. Concurrency Can Increase Memory

More workers do not automatically mean better throughput.

If each worker holds:

```
batch_size records
+
parser buffers
+
database buffers
```

then:

```
memory ~= workers * worker_memory
```

Increasing concurrency can therefore exhaust memory.

Tune together:

- worker count;
- batch size;
- queue size;
- connection pool;
- database capacity.

---

## 20. Read Rate vs Process Rate

Measure both.

Example:

```
read rate     = 300 MB/s
process rate  = 80 MB/s
```

If buffering is unbounded, backlog grows.

A stable system should eventually satisfy:

```
average input rate
≈
average sustainable processing rate
```

or deliberately throttle the reader.

---

## 21. Do Not Read Faster Than Necessary

If the database can safely process only 50,000 records/sec, reading 500,000 records/sec into memory provides little benefit.

A controlled extractor can use:

- bounded queues;
- batch limits;
- worker pools;
- rate limiting;
- transaction boundaries.

The goal is stable throughput, not maximum disk-read speed.

---

## 22. Checkpointing Large Files

Large files make restartability important.

A checkpoint should represent **durably committed work**.

Example:

```
source_identity
last_committed_record_id
records_committed
checkpoint_time
status
```

The sequence should be:

```
read
  |
  v
validate
  |
  v
write batch
  |
  v
COMMIT
  |
  v
CHECKPOINT
```

Do not checkpoint before the database commit.

---

## 23. Record-Based Restart

If the source has a stable record identity:

```
payment_id
```

the pipeline can resume using that identity plus idempotent staging.

This is safer than assuming:

```
line 5,000,001
```

will always represent the same record.

Source regeneration can change record ordering.

---

## 24. Byte Offset Checkpoints

Byte offsets can be useful for simple line-oriented immutable files.

Example:

```
source hash
byte offset
record count
```

But do not use byte offsets blindly.

They can become unsafe when:

- the source changes;
- encoding is variable-width;
- records span chunks;
- compression is involved;
- the parser maintains internal state;
- the format is not independently seekable.

For compressed streams, logical record checkpoints are often safer.

---

## 25. Atomic Batch Commit

A good large-file pipeline uses bounded transactions:

```
records 1-5000
    |
    v
transaction
    |
    v
COMMIT
    |
    v
checkpoint

records 5001-10000
    |
    v
transaction
    |
    v
COMMIT
    |
    v
checkpoint
```

If the process dies after batch 1, batch 1 is durable and batch 2 can be retried.

Idempotency protects against duplicate replay.

---

## 26. Idempotent Large-File Processing

The ideal relationship is:

```
retry same batch
      |
      v
same record identities
      |
      v
unique constraint
      |
      v
no duplicate publication
```

Example:

```
INSERT ...
ON CONFLICT (record_id) DO NOTHING;
```

Exactly-once behavior should not be claimed merely because the application has a checkpoint. The durable storage boundary and recovery semantics matter.

---

## 27. Progress Reporting

Large files can run for hours.

Report progress such as:

```
source_size
bytes_read
records_read
records_committed
elapsed_seconds
throughput
estimated_progress
```

For a file with known size:

```
progress =
bytes_consumed / source_size
```

For formats where byte progress is unavailable or unreliable, report records processed and throughput instead.

---

## 28. Throughput Measurement

Measure:

```
bytes_per_second
records_per_second
batch_duration
commit_duration
decode_duration
```

For example:

```
decode = 180 MB/s
database = 65 MB/s
```

The database is the bottleneck.

Optimizing the parser may not improve end-to-end throughput.

---

## 29. Bottleneck Diagnosis

Use a pipeline model:

```
READ
  |
  v
DECODE
  |
  v
VALIDATE
  |
  v
TRANSFORM
  |
  v
WRITE
```

Measure each stage.

If:

```
READ      500 MB/s
DECODE    300 MB/s
VALIDATE  250 MB/s
WRITE      70 MB/s
```

then optimizing the reader from 500 to 700 MB/s is unlikely to improve the overall pipeline.

Optimize the slowest sustained stage first.

---

## 30. Large File Processing With PostgreSQL

A common pattern:

```
stream records
     |
     v
validate
     |
     v
bounded batch
     |
     v
PostgreSQL transaction
     |
     v
commit
     |
     v
checkpoint
```

For high volumes, consider database-native bulk loading where the source and validation model permit it.

Do not sacrifice validation, lineage, or idempotency merely to maximize insert throughput.

---

## 31. COPY and Bulk Loading

PostgreSQL `COPY` can provide high-throughput loading.

The architecture may become:

```
large file
    |
    v
stream parser
    |
    v
validated rows
    |
    v
COPY-compatible stream
    |
    v
PostgreSQL
```

But bulk loading still needs:

- source identity;
- batch/source state;
- validation;
- failure handling;
- reconciliation.

Fast loading without correctness is not a production pipeline.

---

## 32. Temporary Storage

Sometimes the parser or downstream tool requires seekable input.

Do not automatically load the whole file into memory.

Use bounded local storage:

```
remote source
     |
     v
stream download
     |
     v
temporary file
     |
     v
stream parser
```

Temporary storage must have:

- capacity limits;
- cleanup;
- unique paths;
- permissions;
- lifecycle handling.

---

## 33. Disk Backpressure

Streaming does not eliminate disk pressure.

A pipeline can fail if:

```
download speed
>
parser speed
```

and the temporary staging area grows without limit.

Set a disk budget:

```
maximum temporary storage
maximum active files
cleanup policy
```

Delete temporary data only after downstream processing no longer requires it.

---

## 34. Failure Classification

Large-file failures should be classified.

### Source failures

- incomplete file;
- corrupt file;
- changed file.

### Parser failures

- malformed record;
- invalid encoding;
- schema mismatch.

### Resource failures

- memory;
- disk;
- CPU;
- database capacity.

### Infrastructure failures

- network;
- database connection;
- process termination.

Each class has a different recovery strategy.

---

## 35. Quarantine and Partial Work

When a large source fails, do not automatically delete all previous durable batches.

If batches were committed idempotently:

```
batch 1 -> committed
batch 2 -> committed
batch 3 -> committed
batch 4 -> failed
```

the pipeline can repair the source and replay it safely.

The source-processing state should distinguish:

```
PARTIALLY_PROCESSED
FAILED
RETRYABLE
QUARANTINED
SUCCESS
```

---

## 36. Large File Integrity

For immutable files, record a content hash before processing when practical.

Then:

```
source hash
      |
      v
processing
      |
      v
final source verification
```

If the source can change while being read, the pipeline may process an inconsistent file.

Prefer immutable/versioned source objects.

---

## 37. Encoding Considerations

Text streaming must use the correct encoding.

UTF-8 is common:

```
open(
    path,
    "rt",
    encoding="utf-8",
)
```

But do not assume every producer uses UTF-8.

An incorrect encoding can cause failures or silent character corruption.

For large files, avoid decoding arbitrary byte chunks yourself unless you understand incremental decoder state.

Use the text stream or format parser's encoding support.

---

## 38. Maximum Record Size

Streaming a file does not guarantee bounded memory if one record can itself be enormous.

Example:

```
10 GB file
+
one 8 GB JSON object
```

The parser may still need enormous memory for that one record.

Define a maximum record size where the format and parser allow it.

A useful resource model is:

```
memory
=
parser state
+
largest record
+
batch
+
application overhead
```

---

## 39. Streaming and Data Quality

Streaming should not mean weaker validation.

Validate each record or bounded batch before publication.

Example:

```
stream
  |
  v
decode
  |
  v
schema validation
  |
  v
business validation
  |
  +---- invalid ---> quarantine
  |
  v
batch
  |
  v
commit
```

Data quality is part of the streaming pipeline, not a later excuse for bad ingestion.

---

## 40. Example Production Generator

```
from pathlib import Path
from collections.abc import Iterator
import csv

def stream_csv(path: Path) -> Iterator[dict]:
    with path.open(
        "r",
        encoding="utf-8",
        newline="",
    ) as file:
        reader = csv.DictReader(file)

        for row in reader:
            yield row

def process_large_file(
    path: Path,
    batch_size: int = 5000,
) -> None:
    batch = []

    for row in stream_csv(path):
        validate(row)
        batch.append(row)

        if len(batch) >= batch_size:
            write_batch(batch)
            batch.clear()

    if batch:
        write_batch(batch)
```

The working set is bounded by the parser, current record, batch, and runtime overhead.

---

## 41. Example Progress Tracker

```
from time import monotonic

def progress_tracker():
    started = monotonic()
    records = 0

    def record_processed():
        nonlocal records
        records += 1

        elapsed = monotonic() - started

        if elapsed > 0:
            rate = records / elapsed
            return {
                "records": records,
                "records_per_second": rate,
                "elapsed_seconds": elapsed,
            }

        return {
            "records": records,
            "records_per_second": 0,
            "elapsed_seconds": elapsed,
        }

    return record_processed
```

In production, emit periodic metrics rather than logging every record.

---

## 42. Testing Strategy

Test with files that are:

- smaller than memory;
- approximately equal to memory;
- larger than memory;
- much larger than memory;
- malformed near the beginning;
- malformed near the end;
- composed of unusually large records.

Measure:

- peak memory;
- throughput;
- batch latency;
- recovery behavior.

A streaming implementation is not proven by a 10 MB fixture.

---

## 43. Failure Test — Kill During Processing

A valuable drill:

1. Start processing a large immutable source.
2. Allow several batches to commit.
3. Terminate the process.
4. Restart the pipeline.
5. Confirm already committed records are not duplicated.
6. Confirm remaining records are processed.
7. Reconcile the final count.

Expected behavior:

```
batch 1 -> committed
batch 2 -> committed
batch 3 -> committed
PROCESS DIES
batch 4 -> retry
batch 5 -> retry
...
      |
      v
RECONCILE
```

This proves restartability rather than merely testing normal execution.

---

## 44. Failure Test — Slow Database

1. Artificially slow database writes.
2. Start the extractor.
3. Observe queue/batch growth.
4. Confirm memory remains bounded.
5. Confirm reader throughput falls or backpressure activates.
6. Confirm the process does not continue buffering indefinitely.

This validates the backpressure design.

---

## 45. Intentional Resource Failure

Test:

- too-small memory limit;
- full temporary disk;
- oversized record;
- oversized batch;
- slow database;
- database outage.

The expected outcome is controlled failure, not process-wide uncontrolled memory growth.

---

## 46. Recovery Recipe

When a large-file pipeline fails:

1. Identify the immutable source.
2. Determine the last durable checkpoint.
3. Check committed staging records.
4. Verify the source has not changed.
5. Restart from the logical checkpoint or replay safely.
6. Keep the batch boundary bounded.
7. Preserve idempotent record identity.
8. Reconcile the entire source.
9. Mark success only after reconciliation.

The safest restart strategy is the one that remains correct even if the previous checkpoint is slightly behind.

---

## 47. Production Tools You Should Know

### 1. Python streaming iterators

Generators and file iterators are foundational for memory-safe ETL.

### 2. PyArrow

Useful for large Parquet datasets through datasets, scanners, row groups, and record batches.

### 3. PostgreSQL COPY

Useful when validated records need high-throughput database loading.

The transferable skill is not memorizing one API. It is understanding:

```
bounded memory
+
bounded concurrency
+
bounded batches
+
durable progress
```

---

## 48. Common Mistakes

1. Calling `read()` on a huge source.
2. Calling `list(reader)` on a large dataset.
3. Treating byte chunks as records.
4. Using an unbounded queue.
5. Increasing workers without measuring memory.
6. Choosing batch size arbitrarily.
7. Checkpointing before commit.
8. Using unstable record positions as identity.
9. Ignoring oversized individual records.
10. Measuring only compressed bytes.
11. Optimizing the reader when the database is the bottleneck.
12. Assuming streaming automatically means restartable.

---

## 49. Production Implementation Sequence

```
1. Measure the source size and record distribution.
2. Identify the semantic record boundary.
3. Choose a streaming parser.
4. Define the memory budget.
5. Define maximum record size.
6. Define batch size.
7. Define concurrency limits.
8. Implement bounded streaming.
9. Add validation.
10. Add bounded database writes.
11. Add backpressure.
12. Add durable processing state.
13. Add logical checkpoints.
14. Add idempotent staging.
15. Add progress metrics.
16. Add throughput metrics.
17. Test files larger than available memory.
18. Kill the process during processing.
19. Test slow downstream systems.
20. Test resource exhaustion.
21. Reconcile after recovery.
22. Enable production scheduling.
```

---

## 50. Production Checklist

### Memory

- [ ] Full file is never loaded into memory.
- [ ] Batch size is bounded.
- [ ] Queue size is bounded.
- [ ] Concurrency is bounded.
- [ ] Maximum record size is considered.

### Streaming

- [ ] Parser operates at the correct semantic boundary.
- [ ] Large JSON uses a streaming parser where necessary.
- [ ] XML uses event-based parsing where necessary.
- [ ] Avro streams records.
- [ ] Parquet uses dataset/row-batch semantics.

### Storage

- [ ] Database writes are bounded.
- [ ] Transactions commit before checkpoints.
- [ ] Temporary disk usage is bounded.
- [ ] Cleanup is reliable.

### Restartability

- [ ] Source identity is immutable.
- [ ] Record identity is deterministic.
- [ ] Checkpoints represent durable work.
- [ ] Replay is idempotent.
- [ ] Recovery has been tested.

### Operations

- [ ] Memory metrics exist.
- [ ] Throughput metrics exist.
- [ ] Batch latency is observable.
- [ ] Backpressure is observable.
- [ ] Failure states are classified.
- [ ] Reconciliation exists.

---

## 51. Definition of Done

E61 is complete when you can independently:

- explain why large files break naive ETL;
- design bounded-memory extraction;
- distinguish byte streaming from record streaming;
- choose the correct parser boundary;
- batch records safely;
- calculate and measure memory pressure;
- implement backpressure;
- control concurrency;
- checkpoint only durable work;
- resume safely after process termination;
- use idempotent staging;
- measure throughput and bottlenecks;
- handle oversized records;
- test resource exhaustion;
- recover a failed large-file job without duplicating data.

---

## 52. What You Learned

Large-file processing is fundamentally a **bounded-resource and restartability problem**.

The mental model is:

```
LARGE SOURCE
    |
    v
STREAM
    |
    v
BOUNDED BUFFER
    |
    v
PARSE
    |
    v
VALIDATE
    |
    v
BOUNDED BATCH
    |
    v
COMMIT
    |
    v
CHECKPOINT
    |
    v
NEXT BATCH
```

The key production lesson is:

> A pipeline is genuinely streaming only when memory, buffering, concurrency, and downstream pressure remain bounded as source size increases.

---

## 53. Next Recipe

**E62 — Multi-File Extraction**

The next recipe covers extracting coordinated deliveries containing many files, including file grouping, parallelism, completeness, per-file state, ordering, idempotency, and delivery-level reconciliation.
