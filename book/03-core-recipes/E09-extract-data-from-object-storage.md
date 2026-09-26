# E09 — Extract Data from Object Storage

Object storage is a common source for production data pipelines.

Examples include:

- S3-compatible storage
- cloud object storage
- data exports
- partner-delivered files
- application event archives
- batch landing zones

Object storage looks like a filesystem, but it is not a normal filesystem. Objects are identified by keys and metadata rather than directories and mutable files.

The goal is:

> Discover the correct objects, prove they are complete and immutable enough to process, extract them safely, and record durable progress.

## 1. Problem Recognition

Common production problems include:

- discovering the same object repeatedly
- treating prefixes as real directories
- processing an object before it is complete
- incomplete or failed uploads
- duplicate deliveries
- object replacement after discovery
- listing millions of objects inefficiently
- missing objects because listing was incomplete
- downloading entire objects unnecessarily
- credentials expiring during extraction
- transient storage errors
- versioned objects being confused
- checkpointing only a filename or key
- processing objects in the wrong order

Do not equate:

    object exists

with:

    object is ready and safe to process

## 2. Object-Storage Extraction Architecture

    STORAGE BUCKET
          ↓
    DISCOVER OBJECTS
          ↓
    FILTER EXPECTED OBJECTS
          ↓
    VERIFY OBJECT IDENTITY / COMPLETENESS
          ↓
    READ OBJECT
          ↓
    PARSE
          ↓
    VALIDATE
          ↓
    PERSIST
          ↓
    CHECKPOINT
          ↓
    MARK OBJECT COMPLETE

Each stage has a different responsibility.

## 3. Implementation — Build the Mechanism

Before using a cloud SDK, build the extraction contract in plain Python.

The important state is not only the object key. It is the object identity plus metadata and durable processing state.

### 3.1 Object identity

```python
from dataclasses import dataclass
from typing import Optional

@dataclass(frozen=True)
class ObjectRef:
    bucket: str
    key: str
    version_id: Optional[str]
    etag: Optional[str]
    size: int
```

Do not use the key alone when an object can be replaced.

### 3.2 Idempotent object processing

```python
def object_identity(ref: ObjectRef) -> tuple:
    return (ref.bucket, ref.key, ref.version_id, ref.etag)

def should_process(ref: ObjectRef, processed: set[tuple]) -> bool:
    return object_identity(ref) not in processed

def mark_complete(ref: ObjectRef, processed: set[tuple]) -> None:
    processed.add(object_identity(ref))
```

The production version should store this state durably rather than in an in-memory set.

### 3.3 Paginated discovery

```python
def discover_objects(list_page, prefix: str):
    continuation = None

    while True:
        objects, continuation = list_page(
            prefix=prefix,
            continuation=continuation,
        )

        for obj in objects:
            yield obj

        if continuation is None:
            break
```

The extractor processes one page at a time instead of assuming that a bucket listing fits in memory.

### 3.4 Streaming object content

```python
def stream_lines(byte_stream):
    buffer = b''

    for chunk in byte_stream:
        buffer += chunk

        while b'\n' in buffer:
            line, buffer = buffer.split(b'\n', 1)
            yield line

    if buffer:
        yield buffer
```

For very large objects, use a provider streaming body or buffered reader so memory remains bounded.

### 3.5 End-to-end extraction contract

```python
def extract_objects(objects, processed, read_object, persist_record):
    for ref in objects:
        if not should_process(ref, processed):
            continue

        body = read_object(ref)
        try:
            for line in stream_lines(body):
                record = parse_record(line)
                persist_record(record)

            # Advance the checkpoint only after persistence succeeds.
            mark_complete(ref, processed)
        finally:
            body.close()

def parse_record(line: bytes) -> dict:
    import json
    return json.loads(line)
```

The critical ordering is:

```text
READ → PARSE → PERSIST → MARK COMPLETE
```

### 3.6 Production S3 adapter

```python
import boto3

s3 = boto3.client('s3')

def list_s3_objects(bucket: str, prefix: str):
    paginator = s3.get_paginator('list_objects_v2')

    for page in paginator.paginate(Bucket=bucket, Prefix=prefix):
        for item in page.get('Contents', []):
            yield item

def get_s3_metadata(bucket: str, key: str):
    response = s3.head_object(Bucket=bucket, Key=key)
    return {
        'size': response['ContentLength'],
        'etag': response.get('ETag'),
        'version_id': response.get('VersionId'),
        'content_type': response.get('ContentType'),
        'last_modified': response.get('LastModified'),
    }

def stream_s3_object(bucket: str, key: str):
    response = s3.get_object(Bucket=bucket, Key=key)
    return response['Body']
```

The SDK is only the source adapter. The pipeline still owns identity, readiness, parsing, persistence, checkpointing, and recovery.

### 3.7 Safe S3 extraction skeleton

```python
def extract_s3_prefix(bucket, prefix, processed):
    for item in list_s3_objects(bucket, prefix):
        key = item['Key']
        metadata = get_s3_metadata(bucket, key)

        ref = ObjectRef(
            bucket=bucket,
            key=key,
            version_id=metadata['version_id'],
            etag=metadata['etag'],
            size=metadata['size'],
        )

        if not should_process(ref, processed):
            continue

        body = stream_s3_object(bucket, key)
        try:
            for line in stream_lines(body):
                persist_record(parse_record(line))

            mark_complete(ref, processed)
        finally:
            body.close()
```

### 3.8 Test the mechanism without cloud credentials

```python
def test_duplicate_object_is_skipped():
    processed = set()
    ref = ObjectRef('raw', 'payments/001.jsonl', 'v1', 'etag-1', 100)

    assert should_process(ref, processed)
    mark_complete(ref, processed)
    assert not should_process(ref, processed)

def test_replaced_object_is_not_the_same_identity():
    processed = set()
    old = ObjectRef('raw', 'payments/001.jsonl', 'v1', 'etag-1', 100)
    new = ObjectRef('raw', 'payments/001.jsonl', 'v2', 'etag-2', 120)

    mark_complete(old, processed)
    assert should_process(new, processed)
```

These tests prove that duplicate processing is suppressed while replacement under the same key is detected.

### 3.9 Implementation rule

Keep this boundary:

```text
OBJECT STORAGE SDK
        ↓
SOURCE ADAPTER
        ↓
OBJECT EXTRACTION CONTRACT
        ↓
PARSER
        ↓
PERSISTENCE
        ↓
CHECKPOINT
```

This makes the mechanism testable locally and portable across object-storage providers.

## 4. Bucket, Prefix, Key

Object storage commonly uses:

    bucket
      ↓
    key

Example:

    bucket: raw-payments
    key: 2026/09/26/payments_001.json

A key may look like a filesystem path, but object storage does not necessarily provide filesystem semantics.

A prefix such as:

    2026/09/26/

is generally a naming filter rather than a physical directory.

## 4. Object Discovery

An extractor may discover objects using:

- prefix listing
- manifest files
- event notifications
- inventory systems
- database metadata
- explicit object keys

For small controlled feeds, a prefix listing may be sufficient.

For very large feeds, a manifest or event-driven approach can reduce repeated listing work.

## 5. Listing Is Pagination

Object listings can themselves be paginated.

Conceptually:

    LIST OBJECTS
        ↓
    RECEIVE PAGE
        ↓
    PROCESS KEYS
        ↓
    NEXT PAGE
        ↓
    COMPLETE

Never assume one list operation returns every object.

The storage API's continuation token must be handled according to its contract.

## 6. Prefix Filtering

Use precise prefixes where possible.

Instead of:

    list entire bucket

prefer:

    list prefix 2026/09/26/payments/

This reduces discovery work and accidental ingestion.

Filtering by filename pattern can provide another layer of control.

## 7. Manifest-Based Extraction

A manifest can explicitly define the expected objects.

Example:

    manifest
      ↓
    object A
    object B
    object C

This can provide stronger completeness semantics than discovering whatever happens to exist in a prefix.

A manifest can contain:

- object key
- size
- checksum
- version identifier
- record count
- partition

## 8. Object Identity

A key alone may not uniquely identify content if an object can be replaced.

Useful identity information can include:

    bucket
    key
    version_id
    etag
    checksum
    size
    last_modified

Do not assume an ETag is always a simple MD5 checksum. Multipart uploads and provider-specific behavior can change its meaning.

Use a checksum designed for the storage contract when exact content identity is required.

## 9. Object Immutability

The safest extraction model is:

    OBJECT PUBLISHED
          ↓
    OBJECT IMMUTABLE
          ↓
    EXTRACTOR READS OBJECT

If producers overwrite objects after publication, checkpointing becomes more difficult.

Prefer immutable object keys or object versioning when possible.

## 10. Incomplete Uploads

Some object-storage systems support multipart uploads.

An incomplete multipart upload should not be treated as a published dataset object merely because an upload operation has started.

Use the storage system's completed-object semantics.

Producer contract should define:

    upload
      ↓
    complete
      ↓
    publish

Do not invent readiness rules from object size alone when the storage system provides a stronger signal.

## 11. Object Metadata

Metadata can help establish readiness and identity.

Useful fields may include:

- content length
- content type
- checksum
- last modified time
- version ID
- custom metadata

Do not treat metadata supplied by an untrusted producer as authoritative business data without validation.

## 12. Downloading Objects

A basic extraction reads the object body:

    object
       ↓
    byte stream
       ↓
    parser
       ↓
    records

For small objects, a complete download may be acceptable.

For large objects, stream the body.

## 13. Streaming Object Reads

Prefer:

    OBJECT
      ↓
    STREAM
      ↓
    PARSER
      ↓
    BOUNDED BUFFER
      ↓
    PERSIST

rather than:

    OBJECT
      ↓
    LOAD ENTIRE OBJECT INTO MEMORY

Streaming limits worker memory usage.

## 14. Range Reads

Some object stores support byte-range reads.

Conceptually:

    object bytes 0–99 MB
    object bytes 100–199 MB
    object bytes 200–299 MB

Range reads can be useful for formats or recovery workflows that support independent byte ranges.

Do not assume arbitrary byte ranges are safe for every file format.

CSV records, compressed files, and JSON documents may cross range boundaries.

## 15. Object Format Matters

The extraction strategy depends on the object format.

Examples:

    CSV      → streaming parser
    JSONL    → line-oriented streaming
    JSON     → parser/streaming strategy
    Parquet  → columnar reads
    Avro     → schema-aware reader
    ZIP      → archive-aware processing

Object storage is the transport layer; the file format determines how bytes become records.

## 16. Content-Type Validation

Metadata may identify:

    application/json

but the content may not actually be valid JSON.

Do not trust content type as the only validation layer.

Use:

    metadata
      +
    file extension where useful
      +
    content/parser validation

## 17. Object Size Validation

Unexpected size can indicate:

- incomplete upload
- empty file
- wrong dataset
- producer failure
- compressed/uncompressed mismatch

Define expected size rules carefully.

Do not reject legitimate small files merely because a historical average was larger.

Size is evidence, not proof of correctness.

## 18. Empty Objects

Distinguish:

    object missing

from:

    object exists with zero bytes

and:

    object exists with a valid zero-record dataset

These can represent different business states.

## 19. Missing Objects

If a manifest expects:

    payments_001.json
    payments_002.json
    payments_003.json

but only two objects arrive, the extractor should detect the missing object.

Do not interpret:

    object not discovered

as:

    object contains zero records

## 20. Object Ordering

Object listings are not automatically a business sequence.

Do not assume lexicographic key order represents event order or processing order.

If order matters, define it explicitly using:

- manifest sequence
- partition
- source timestamp
- object metadata
- source-specific ordering

## 21. Event-Driven Discovery

Object creation notifications can trigger extraction.

Conceptually:

    OBJECT CREATED
         ↓
    EVENT
         ↓
    EXTRACTOR
         ↓
    VERIFY OBJECT
         ↓
    PROCESS

Events can be duplicated or delayed.

Therefore event-driven discovery still needs idempotency and object verification.

## 22. Polling Discovery

Polling periodically lists a prefix:

    LIST
      ↓
    FIND NEW OBJECTS
      ↓
    PROCESS
      ↓
    NEXT POLL

Polling must maintain durable state so that repeated listings do not cause repeated processing.

## 23. Checkpointing

Object-level checkpoint state can contain:

    bucket
    key
    version_id
    checksum
    status
    rows_processed
    completed_at

For a large object, record parser-specific progress where safe.

Always bind progress to the exact object identity.

## 24. Object Replacement

Suppose:

    key = payments/2026-09-26.json

was processed once.

Later the producer replaces its content under the same key.

A key-only checkpoint can incorrectly report the new object as already processed.

Use version IDs, checksums, or an immutable publication contract.

## 25. Idempotent Object Processing

Safe processing often follows:

    DISCOVER
       ↓
    IDENTIFY OBJECT
       ↓
    CHECK PROCESSED STATE
       ↓
    PROCESS
       ↓
    PERSIST
       ↓
    MARK COMPLETE

Do not mark an object complete before its data is durably persisted.

## 26. Partial Failure

Suppose an object contains 10 million records and persistence fails after 6 million.

Possible recovery approaches:

- restart the object with idempotent downstream writes
- resume from a safe parser checkpoint
- process immutable chunks
- use object-format-specific partitioning

The safest choice depends on the format and downstream idempotency.

## 27. Large Object Strategy

For very large objects, consider:

- streaming
- object partitioning
- producer-side file splitting
- format-specific parallel reads
- compression
- columnar formats

Do not introduce parallel range reads merely because an object is large.

Measure the actual bottleneck first.

## 28. Credentials

Object storage access requires credentials or workload identity.

Production rules:

- do not hard-code credentials
- use managed identity/workload identity where available
- use short-lived credentials where practical
- grant minimum required permissions
- separate read and write permissions

The extractor normally needs only the permissions required to list/read its source objects.

## 29. Permissions

Typical permissions include:

- list bucket/prefix
- read object
- read object metadata

Do not grant write/delete permissions to a read-only extraction worker unless the workflow genuinely requires them.

Least privilege limits blast radius.

## 30. Transient Storage Failures

Object storage requests can fail because of:

- timeouts
- temporary service errors
- connection resets
- throttling
- expired credentials
- network interruptions

Classify failures before retrying.

Permanent authorization failures should not be retried indefinitely.

## 31. Retry Strategy

Retry transient failures using controlled backoff.

Conceptually:

    REQUEST
       ↓
    TRANSIENT FAILURE?
       ↓ yes
    BACKOFF + JITTER
       ↓
    RETRY

Combine retries with a maximum attempt count and clear failure classification.

## 32. Testing

Test at least:

1. One valid object.
2. Multiple objects.
3. Paginated object listing.
4. Duplicate object notification.
5. Duplicate object delivery.
6. Missing expected object.
7. Empty object.
8. Unexpected object size.
9. Checksum mismatch.
10. Object replacement.
11. Incomplete upload scenario.
12. Large object.
13. Streaming read.
14. Read timeout.
15. Transient storage error.
16. Authorization failure.
17. Expired credentials.
18. Parser failure.
19. Persistence failure.
20. Checkpoint restart.
21. Manifest mismatch.
22. Out-of-order object arrival.

## 33. Observability

Useful metrics:

    object_list_requests_total
    object_list_failures_total
    objects_discovered_total
    objects_processed_total
    objects_failed_total
    objects_skipped_total
    objects_duplicated_total
    object_bytes_read_total
    object_read_duration_seconds
    object_read_failures_total
    object_retry_total
    object_records_extracted_total
    object_checkpoint_updates_total

Useful log fields:

    extraction_run_id
    bucket
    object_key
    version_id
    checksum
    object_size
    content_type
    attempt
    rows_processed
    duration
    error_type

Do not log access tokens, secret credentials, or sensitive object contents.

## 34. Intentional Failure

### Failure drill 1 — Duplicate notification

Deliver the same object-created event twice.

Verify the object is processed only according to the idempotency policy.

### Failure drill 2 — Missing manifest object

Remove one expected object.

Verify the pipeline reports an incomplete dataset rather than silently succeeding.

### Failure drill 3 — Object replacement

Replace an object's content under the same key.

Verify identity checking detects the changed artifact.

### Failure drill 4 — Transient storage error

Inject a temporary read failure.

Verify controlled retry and backoff.

### Failure drill 5 — Persistent authorization failure

Remove read permission.

Verify the extractor stops instead of retrying indefinitely.

### Failure drill 6 — Persistence failure

Fail persistence after part of an object is processed.

Restart and verify checkpoint/idempotency behavior.

### Failure drill 7 — Incomplete publication

Expose an object before the producer's completion signal.

Verify the extraction contract prevents premature processing.

## 35. Recovery

When object-storage extraction fails:

1. Identify the extraction run.
2. Identify bucket, key, and object version.
3. Verify object identity and checksum.
4. Check publication/completion state.
5. Determine whether the failure is discovery, authorization, network, parser, or persistence related.
6. Inspect the last durable checkpoint.
7. Confirm the object has not changed.
8. Resume or restart according to the object format's checkpoint capability.
9. Reconcile extracted records.
10. Mark the object complete only after verification.

Never use a key-only checkpoint when object replacement is possible.

## 36. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Amazon S3** | Understand buckets, object keys, prefixes, metadata, versioning, multipart uploads, range reads, and consistency semantics. |
| **MinIO** | S3-compatible object storage useful for local development and understanding object-storage behavior. |
| **boto3** | Python SDK commonly used to list, inspect, stream, and retrieve S3 objects. |

> These are reference tools for production vocabulary. The underlying object-storage extraction mechanics should still be understood independently.

## 37. Production Runbook

### Expected object is missing

Check:

1. Prefix.
2. Listing pagination.
3. Manifest.
4. Producer publication.
5. Arrival window.
6. Permissions.

### Object exists but cannot be read

Check:

1. Object version.
2. Permissions.
3. Credentials.
4. Storage service health.
5. Network connectivity.

### Object was processed twice

Check:

1. Object identity.
2. Processed-object registry.
3. Duplicate notifications.
4. Checkpoint timing.
5. Object replacement.

### Object read is too slow

Check:

1. Object size.
2. Network throughput.
3. Compression.
4. Parser speed.
5. Batch size.
6. Whether range/parallel reads are actually supported safely.

### What not to do

Do not:

- treat prefixes as real directories
- assume one listing returns every object
- trust object existence as readiness
- use filename/key alone as immutable identity
- assume ETag is always MD5
- download huge objects entirely into memory
- grant unnecessary delete/write permissions
- retry permanent authorization failures forever
- assume object listing order represents business order
- mark objects complete before persistence

## 38. Common Mistakes

### Mistake 1 — Treating object storage like a filesystem

Keys and prefixes do not automatically provide filesystem semantics.

### Mistake 2 — No object identity

Objects can be replaced under the same key.

### Mistake 3 — Ignoring listing pagination

Large prefixes may require multiple listing requests.

### Mistake 4 — Assuming object existence means completeness

Publication and upload completion are separate concerns.

### Mistake 5 — Unbounded downloads

Large objects can exhaust memory.

### Mistake 6 — Event-only processing

Notifications can be duplicated or delayed.

### Mistake 7 — No source-load or credential discipline

Storage access still needs controlled retries and least privilege.

## 39. Definition of Done

You are done when you can:

- explain bucket, key, and prefix semantics
- discover objects safely
- handle paginated listings
- use manifests when stronger completeness is required
- define object identity
- reason about object immutability
- detect incomplete publication
- stream large objects
- understand range-read limitations
- choose format-appropriate extraction methods
- distinguish missing, empty, and valid zero-record objects
- handle event-driven and polling discovery
- checkpoint object processing safely
- detect object replacement
- implement idempotent object processing
- classify storage failures
- use controlled retries
- apply least-privilege access
- test duplicate and missing objects
- intentionally break object extraction
- recover safely
- operate object-storage extraction with a production runbook

## 40. What You Learned

The central principle is:

> Object storage is a durable object system, not a normal filesystem; extraction must establish object identity, completeness, and safe progress explicitly.

A production object-storage pipeline follows:

    DISCOVER
       ↓
    IDENTIFY OBJECT
       ↓
    VERIFY COMPLETENESS
       ↓
    READ / STREAM
       ↓
    VALIDATE
       ↓
    PERSIST
       ↓
    CHECKPOINT
       ↓
    MARK COMPLETE

The key question is:

> What proves that this is the exact, complete object I intended to process, and what durable evidence proves that I processed it safely?

That question drives object identity, listing, manifests, streaming, permissions, retries, checkpointing, idempotency, testing, observability, and recovery.