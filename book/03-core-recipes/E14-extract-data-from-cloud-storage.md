# E14 — Extract Data from Cloud Storage

Cloud object storage is a common extraction boundary for data pipelines. Examples include Amazon S3, Google Cloud Storage, and Azure Blob Storage.

The production problem is not simply downloading a file. The extractor must discover the right objects, determine readiness, preserve object identity, stream large objects safely, tolerate failures, prevent duplicate processing, and checkpoint only after durable work succeeds.

## 1. Problem Recognition

Common problems:
- objects arrive continuously
- listing APIs contain many objects
- the same object is discovered repeatedly
- an object is overwritten after discovery
- a producer publishes an object before it is complete
- large objects exhaust worker memory
- downloads fail halfway through
- provider APIs throttle requests
- objects arrive late
- workers process the same object concurrently
- the pipeline restarts after partial progress

The extraction boundary is:

```text
CLOUD STORAGE
      ↓
DISCOVER OBJECTS
      ↓
IDENTIFY OBJECT
      ↓
VERIFY READINESS
      ↓
READ / STREAM OBJECT
      ↓
VALIDATE
      ↓
PERSIST
      ↓
CHECKPOINT OBJECT
```

## 2. Object Storage Mental Model

An object is commonly identified by:

```text
BUCKET / CONTAINER
        +
OBJECT KEY / BLOB NAME
        +
VERSION OR GENERATION
```

Useful metadata can include size, content type, checksum, ETag, last-modified time, version ID, generation number, and custom metadata.

Do not assume an object key alone identifies one immutable object. A producer can replace the object while keeping the same key.

## 3. Object Identity

Suppose:

```text
bucket = payments
key = daily/2026-09-26.json
```

Version A may exist at one point and version B may later replace it.

Therefore:

```text
KEY
 ≠
IMMUTABLE OBJECT IDENTITY
```

Where the provider exposes versions or generations, use them when replacement detection matters. If versioning is unavailable, use a documented combination of key, size, checksum, and modification metadata as appropriate.

## 4. Discovery

A basic discovery flow is:

```text
LIST PREFIX
    ↓
OBJECT METADATA
    ↓
FILTER ELIGIBLE OBJECTS
    ↓
IDENTIFY OBJECT
    ↓
PROCESS
```

Use paginated listing for large prefixes. Do not load millions of object records into memory.

Example:

```python
def discover_objects(client, bucket, prefix):
    continuation_token = None

    while True:
        response = client.list_objects(
            bucket=bucket,
            prefix=prefix,
            continuation_token=continuation_token,
        )

        for obj in response["objects"]:
            yield obj

        continuation_token = response.get("next_token")

        if not continuation_token:
            break
```

The exact fields differ by provider. The principle is that discovery must be bounded and restartable.

## 5. Prefix Design

A useful object layout can look like:

```text
payments/
  year=2026/
    month=09/
      day=26/
        payments-001.json
        payments-002.json
```

Good prefixes help discovery, targeted extraction, backfills, retention, troubleshooting, and parallel processing.

## 6. Readiness

An object being visible does not necessarily mean it is ready.

Common producer contracts include:

### Temporary publication
```text
upload.part
    ↓
upload.json
```

### Completion marker
```text
data.json
done.marker
```

### Manifest
```text
manifest.json
    ↓
references complete objects
```

### Immutable publication
The producer guarantees that once published, the object will not be modified.

Use the actual producer contract. Do not invent arbitrary sleep periods as a readiness mechanism.

## 7. Metadata Inspection

Before downloading a large object, inspect metadata when useful:

```python
def inspect_object(client, bucket, key):
    return client.head_object(
        bucket=bucket,
        key=key,
    )
```

Useful checks include existence, size, content type, checksum, ETag, version/generation, and modification time.

## 8. Extraction Contract

Define what the source is allowed to publish.

```python
OBJECT_CONTRACT = {
    "prefix": "payments/",
    "allowed_content_types": {
        "application/json",
        "application/jsonl",
    },
    "max_size_bytes": 5_000_000_000,
}
```

Validate each object against the contract before expensive processing.

## 9. Streaming Large Objects

Do not assume an object fits in memory.

Unsafe:

```python
payload = client.download_object(bucket, key)
data = parse(payload)
```

Prefer streaming:

```python
response = client.get_object(bucket=bucket, key=key)

for chunk in response.iter_chunks():
    process_chunk(chunk)
```

The exact API depends on the provider.

## 10. Local Staging

When a complete local artifact is required, write to a temporary file and publish atomically:

```python
from pathlib import Path


def download_atomically(client, bucket, key, destination):
    destination = Path(destination)
    temporary = destination.with_suffix(destination.suffix + ".part")

    with client.open_object(bucket, key) as source:
        with temporary.open("wb") as target:
            while chunk := source.read(1024 * 1024):
                target.write(chunk)

    temporary.replace(destination)
```

The .part file prevents downstream code from treating an incomplete download as complete.

## 11. Checksums

Where the provider exposes a reliable checksum, verify downloaded content when practical:

```text
EXPECTED CHECKSUM
        ↓
DOWNLOAD
        ↓
CALCULATE CHECKSUM
        ↓
COMPARE
        ↓
ACCEPT / REJECT
```

Do not assume an ETag is always a cryptographic checksum. Its semantics depend on the provider and upload method.

## 12. Duplicate Object Detection

The same object can be discovered repeatedly because of scheduler reruns, retries, multiple workers, or repeated listings.

Persist processing identity:

```sql
CREATE TABLE processed_object (
    bucket_name TEXT NOT NULL,
    object_key TEXT NOT NULL,
    object_version TEXT,
    checksum TEXT,
    processed_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (
        bucket_name,
        object_key,
        object_version
    )
);
```

The identity columns must match the provider and source contract.

## 13. Replacement Detection

An object key may point to version A today and version B later.

Do not automatically discard version B merely because the key was processed before. Determine whether the replacement is expected, a correction, or an invalid overwrite.

## 14. Checkpoint Ordering

Safe:

```text
OBJECT ID
   ↓
EXTRACT
   ↓
PERSIST
   ↓
VERIFY
   ↓
CHECKPOINT
```

Unsafe:

```text
DISCOVER
   ↓
CHECKPOINT
   ↓
DOWNLOAD
```

If the worker crashes after an unsafe checkpoint, the object may be skipped permanently.

## 15. Durable Processing State

Example state table:

```sql
CREATE TABLE object_extraction_run (
    object_id TEXT PRIMARY KEY,
    status TEXT NOT NULL,
    discovered_at TIMESTAMPTZ NOT NULL,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    attempt_count INTEGER NOT NULL DEFAULT 0,
    error_message TEXT
);
```

Typical states:

```text
DISCOVERED
   ↓
PROCESSING
   ↓
SUCCEEDED

PROCESSING
   ↓
FAILED
   ↓
RETRY

FAILED
   ↓
QUARANTINED
```

## 16. Network Failure and Retry

Cloud downloads can fail because of timeouts, connection resets, provider errors, throttling, DNS failures, or worker interruption.

Transient failures should use bounded retries and backoff. Permanent validation failures should not be retried forever.

```python
import time


def retry_download(download, attempts=3):
    for attempt in range(1, attempts + 1):
        try:
            return download()
        except TemporaryDownloadError:
            if attempt == attempts:
                raise
            time.sleep(2 ** (attempt - 1))
```

Production implementations should classify provider-specific errors and add jitter where appropriate.

## 17. Provider Throttling

If listing or download requests are throttled:

```text
THROTTLE
   ↓
BACKOFF
   ↓
REDUCE REQUEST RATE
   ↓
RETRY
```

Do not respond to throttling by immediately increasing concurrency.

## 18. Parallel Extraction

Objects can often be processed concurrently:

```text
OBJECT A ── Worker 1
OBJECT B ── Worker 2
OBJECT C ── Worker 3
OBJECT D ── Worker 4
```

Bound concurrency according to cloud request limits, bandwidth, CPU, memory, and destination capacity.

## 19. Ordering

Object listing order should not normally be treated as business ordering.

If business ordering matters, use an explicit source field such as event time, sequence number, or batch ID.

Object names and modification times are not automatically equivalent to event order.

## 20. Late and Missing Objects

An expected object that is absent now is not necessarily permanently missing.

```text
EXPECTED OBJECT
      ↓
NOT PRESENT
      ↓
MISSING NOW
      ≠
PERMANENTLY MISSING
```

Use the source's expected-arrival contract and freshness policy.

Possible states include received, late, missing, not expected, and superseded.

## 21. Compression

Common formats include gzip, zip, bzip2, and zstandard.

Prefer streaming decompression for large objects:

```text
CLOUD OBJECT
     ↓
DOWNLOAD STREAM
     ↓
DECOMPRESS
     ↓
PARSE
     ↓
PERSIST
```

Avoid fully decompressing very large objects into memory.

## 22. Format Validation

Cloud storage identifies where an object lives, not whether its contents are valid.

Validate content type, format, schema, and record structure before accepting the object.

Format-specific extraction is covered by the dedicated CSV, JSON, XML, Excel, and future Parquet/Avro recipes.

## 23. Object Immutability and Versioning

The safest contract is:

```text
OBJECT PUBLISHED
      ↓
OBJECT IMMUTABLE
      ↓
CONSUMER READS
```

If objects can be overwritten, use version IDs, generations, checksums, manifests, or another source-defined mechanism.

Version-aware extraction is useful when corrections, auditability, and replay matter.

## 24. Example S3-Style Extraction

Using an S3-compatible Python client:

```python
import boto3


s3 = boto3.client("s3")


def extract_object(bucket, key):
    response = s3.get_object(
        Bucket=bucket,
        Key=key,
    )

    body = response["Body"]

    while True:
        chunk = body.read(1024 * 1024)

        if not chunk:
            break

        yield chunk
```

The important behavior is streaming rather than loading the entire object into memory.

## 25. Testing

Test at least:
1. Object discovery.
2. Paginated discovery.
3. Empty prefix.
4. Missing object.
5. Object replacement.
6. Duplicate discovery.
7. Incomplete object.
8. Completion marker.
9. Manifest validation.
10. Large object.
11. Download timeout.
12. Network interruption.
13. Provider throttling.
14. Checksum mismatch.
15. Invalid content type.
16. Invalid schema.
17. Compressed object.
18. Concurrent extraction.
19. Worker crash.
20. Checkpoint recovery.
21. Late object.
22. Missing expected object.
23. Object version change.
24. Graceful shutdown.

## 26. Example Unit Tests

```python
def test_object_identity_includes_version():
    first = ("payments", "daily/data.json", "version-a")
    second = ("payments", "daily/data.json", "version-b")
    assert first != second


def test_same_object_identity_is_duplicate():
    processed = {
        ("payments", "daily/data.json", "version-a")
    }
    identity = ("payments", "daily/data.json", "version-a")
    assert identity in processed


def test_replacement_is_new_object_identity():
    processed = {
        ("payments", "daily/data.json", "version-a")
    }
    replacement = ("payments", "daily/data.json", "version-b")
    assert replacement not in processed
```

## 27. Observability

Useful metrics:

```text
cloud_objects_discovered_total
cloud_objects_processed_total
cloud_objects_failed_total
cloud_objects_retried_total
cloud_objects_skipped_total
cloud_duplicate_objects_total
cloud_bytes_downloaded_total
cloud_download_duration_seconds
cloud_object_processing_duration_seconds
cloud_checksum_failures_total
cloud_throttling_errors_total
cloud_network_errors_total
cloud_late_objects_total
cloud_missing_objects_total
```

Useful log fields:

```text
provider
bucket
object_key
object_version
object_size
checksum
content_type
attempt
processing_status
processing_duration
error_type
```

Do not log sensitive object contents by default.

## 28. Intentional Failure

### Failure 1 — Process before readiness
Publish an incomplete object and verify that the extractor rejects or defers it.

### Failure 2 — Crash during download
Interrupt a large download and verify that an incomplete local artifact is not published as complete.

### Failure 3 — Checkpoint too early
Record an object as complete before durable persistence. Simulate a crash. Observe the potential loss and fix checkpoint ordering.

### Failure 4 — Duplicate discovery
Run two workers against the same prefix and verify only one logical processing result is created.

### Failure 5 — Object replacement
Process version A, replace it with version B, and verify the new identity is detected when required.

### Failure 6 — Network interruption
Introduce a temporary network failure and verify bounded retry and recovery.

### Failure 7 — Throttling
Generate controlled request pressure in a test environment and verify backoff.

### Failure 8 — Checksum mismatch
Corrupt downloaded content and verify validation rejects the artifact.

## 29. Recovery

1. Identify the bucket or container.
2. Identify the object key.
3. Determine version or generation where available.
4. Inspect object metadata.
5. Check readiness.
6. Check processing state.
7. Inspect the last extraction attempt.
8. Determine whether the object was durably persisted.
9. Retry transient failures.
10. Quarantine invalid objects.
11. Reprocess failed objects safely.
12. Reconcile object counts and destination state.
13. Confirm the checkpoint reflects successful processing only.

## 30. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Amazon S3** | Object storage using buckets, keys, metadata, versioning, multipart uploads, and lifecycle controls. |
| **Google Cloud Storage** | Object storage using buckets, objects, generations, metadata, and resumable transfers. |
| **Azure Blob Storage** | Object storage using containers, blobs, metadata, versions, and access controls. |

The goal is to recognize common object-storage mechanics while understanding that provider APIs and semantics differ.

## 31. Production Runbook

### Objects are being processed twice
Check object identity, version/generation, processing-state uniqueness, worker concurrency, job restarts, and checkpoint timing.

### Objects are missing
Check prefix and discovery window, expected-arrival contract, listing pagination, readiness markers, producer failures, late-arrival state, and checkpoints.

### Downloads are failing
Check provider status, network connectivity, credentials, permissions, throttling, object size, and retry behavior.

### Pipeline memory is increasing
Check object size, full-object buffering, decompression behavior, batch size, concurrent downloads, and parser buffering.

### What not to do
Do not assume object-key uniqueness means immutable identity. Do not process objects before the source contract says they are complete. Do not load multi-gigabyte objects entirely into memory. Do not checkpoint before durable persistence. Do not treat ETag as universally equivalent to a checksum. Do not retry permanent validation failures forever.

## 32. Definition of Done

You are done when you can:
- discover cloud objects safely
- paginate large listings
- define object identity
- detect replacements
- understand readiness contracts
- inspect metadata
- stream large objects
- use temporary local files safely
- validate checksums where appropriate
- prevent duplicate processing
- checkpoint after durable persistence
- handle network failures
- handle throttling
- bound concurrency
- handle compressed objects
- distinguish late and missing objects
- understand versioning and generations
- recover after worker crashes
- intentionally break extraction
- diagnose the failure
- recover without silently losing objects

## 33. What You Learned

Cloud-storage extraction is fundamentally about **object identity, readiness, durable processing, and safe discovery**.

The core pattern is:

```text
DISCOVER
   ↓
IDENTIFY
   ↓
VERIFY READINESS
   ↓
STREAM
   ↓
VALIDATE
   ↓
PERSIST
   ↓
VERIFY
   ↓
CHECKPOINT
   ↓
MONITOR
```

The most important rule is:

> Never treat discovery as proof that an object is ready, immutable, or safely processed.

Once this principle is understood, pagination, versioning, checksums, streaming, retries, duplicate detection, late objects, and recovery become extensions of the same extraction boundary.