
# E63 — Duplicate File Detection

## 1. Problem Recognition

Production file pipelines rarely see a perfect sequence:

    one file -> one discovery -> one extraction -> one load

The same logical file may be observed multiple times.

Examples:

- a polling process lists the same object on every scan;
- an SFTP producer retries an upload;
- a network client downloads the same artifact twice;
- a producer sends the same filename again;
- a delivery is replayed after an incident;
- two workers discover the same file concurrently;
- a corrected file keeps the same name but has different content.

These cases are related, but they are **not all the same kind of duplicate**.

The production problem is:

> **How do I detect duplicate file observations, distinguish them from replacements and legitimate repeated deliveries, and prevent duplicate business data without accidentally discarding valid corrections?**

This recipe applies:

- Core Recipe 06 — Add Idempotency
- Core Recipe 07 — Add Deduplication
- Core Recipe 10 — Replay / Reprocessing
- Core Recipe 16 — Data Reconciliation
- Core Recipe 24 — Partial Failure
- Core Recipe 32 — Concurrency & Parallel Processing
- E49 — File Discovery
- E50 — File Naming Conventions
- E52 — File Completeness Detection
- E53 — File Validation — ETL Application
- E60 — Compressed File Extraction
- E61 — Large File Streaming
- E62 — Multi-File Extraction

The core distinction is:

    duplicate observation
          !=
    duplicate content
          !=
    replacement
          !=
    legitimate new delivery

---

## 2. What You Are Building

Target architecture:

    Source
      |
      v
    File Discovery
      |
      v
    Identity Extraction
      |
      +-------------------+
      |                   |
      v                   v
    Logical Identity    Content Hash
      |                   |
      +---------+---------+
                |
                v
          Duplicate Classifier
                |
        +-------+-------+---------+
        |       |       |         |
        v       v       v         v
      NEW   DUPLICATE REPLACEMENT REPLAY
        |       |       |         |
        +-------+-------+---------+
                |
                v
          Processing Policy
                |
                v
          Idempotent Staging
                |
                v
             Audit

The classifier should produce an explicit decision.

Do not simply write:

    if filename exists:
        skip

That rule is too weak for production.

---

## 3. Learning Objectives

By the end of this recipe you should be able to:

1. Explain what makes two file observations duplicates.
2. Separate logical identity from content identity.
3. Detect repeated discovery safely.
4. Detect duplicate content.
5. Detect same-identity/different-content replacements.
6. Distinguish replay from a new delivery.
7. Use hashes correctly.
8. Build database uniqueness constraints.
9. Prevent concurrent duplicate processing.
10. Design idempotent file registration.
11. Detect duplicate records caused by file replay.
12. Handle duplicate archives and downloads.
13. Preserve evidence of duplicates.
14. Define duplicate policies explicitly.
15. Test duplicate scenarios.
16. Recover safely from ambiguous duplicate cases.

---

## 4. The First Question: Duplicate What?

Before implementing detection, define the identity being duplicated.

Possible levels:

    delivery
    file
    file version
    record
    physical artifact

These are different.

Example:

    Delivery D100
        transactions.csv
            version 1

The same artifact can be downloaded twice:

    /tmp/a.csv
    /tmp/b.csv

That is a duplicate **physical observation**.

The same logical file can be represented by two physical paths:

    incoming/transactions.csv
    archive/transactions.csv

That does not necessarily mean two business files.

---

## 5. Physical Duplicate

A physical duplicate is the same artifact observed or copied more than once.

Example:

    source URI:
        s3://bucket/daily/transactions.csv

Observed twice:

    observation 1
    observation 2

Both refer to the same source object.

This is usually an observation-level duplicate.

---

## 6. Logical File Duplicate

A logical duplicate means the same business file identity appears more than once.

Example:

    delivery_id = D20260926
    file_id = transactions

Observed:

    transactions.csv
    transactions.csv

The logical identity is the same.

The pipeline should normally process the logical file once.

---

## 7. Content Duplicate

Two artifacts can have different names but identical content.

Example:

    transactions.csv
    transactions_backup.csv

Both:

    SHA-256 = ABC123

This is evidence of duplicate content.

It is **not automatically proof** that the files represent the same business file.

Two different valid deliveries can legitimately contain identical content.

Therefore:

    content equality
        !=
    business identity equality

---

## 8. Replacement

A replacement occurs when the logical identity is the same but content differs.

Example:

    delivery = D20260926
    file_id = transactions

First:

    SHA-256 = ABC

Later:

    SHA-256 = XYZ

This may be:

- a corrected file;
- a producer retry that produced different bytes;
- a partial-upload artifact;
- a new version;
- an accidental overwrite.

Never silently classify it as an ordinary duplicate.

---

## 9. Replay

A replay occurs when a previously processed delivery or file is intentionally submitted again.

Example:

    D20260926
        transactions.csv

was processed yesterday.

The producer or operator asks the pipeline to process it again.

Replay may be legitimate.

The correct behavior is not necessarily:

    reject

Instead:

    identify replay
    apply replay policy
    rely on idempotent downstream processing

Replay is an operational event, not automatically a data-quality error.

---

## 10. Duplicate Classification Matrix

| Logical identity | Content | Previous state | Classification |
|---|---|---|---|
| new | new | none | NEW |
| same | same | processed | DUPLICATE |
| same | different | processed | REPLACEMENT |
| same | same | failed | RETRY / REPLAY |
| different | same | processed | DUPLICATE_CONTENT |
| different | different | processed | NEW |

This table is only a starting point.

The source contract can override these defaults.

---

## 11. File Identity Must Be Deterministic

Example:

    delivery_id + file_type + sequence

can form a file ID.

Python:

```
def file_id(
    delivery_id: str,
    file_type: str,
    sequence: int | None = None,
) -> str:
    parts = [
        delivery_id,
        file_type,
    ]

    if sequence is not None:
        parts.append(str(sequence))

    return "|".join(parts)
```

Example:

```
assert file_id(
    "D20260926",
    "transactions",
    1,
) == "D20260926|transactions|1"
```

If the same business file is discovered twice, the ID must be identical.

---

## 12. Do Not Use Random UUIDs as File Identity

This is wrong for business identity:

```
import uuid

file_id = str(uuid.uuid4())
```

Every discovery creates a new ID.

Therefore:

    discovery 1 -> ID A
    discovery 2 -> ID B

The pipeline cannot recognize that they are the same logical file.

Random UUIDs can still be useful as:

- observation IDs;
- attempt IDs;
- trace IDs.

They should not replace deterministic business identity.

---

## 13. Observation Identity

Sometimes you need to record every time a file was observed.

Use a separate observation ID.

Example:

    file_id = D20260926|transactions

    observation_id = random UUID

Database:

```
CREATE TABLE etl_file_observation (
    observation_id UUID PRIMARY KEY,
    delivery_id TEXT NOT NULL,
    file_id TEXT NOT NULL,
    source_uri TEXT NOT NULL,
    observed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    size_bytes BIGINT,
    sha256 TEXT,
    classification TEXT
);
```

Now the system can say:

    one logical file
    observed 14 times

without creating 14 logical files.

---

## 14. Why Observation History Matters

Suppose a polling process runs every minute.

For one file:

    minute 1 -> observed
    minute 2 -> observed
    minute 3 -> observed
    ...
    minute 14 -> observed

If the system stores only:

    file exists

it loses useful operational information.

Observation history can reveal:

- noisy source behavior;
- polling inefficiency;
- unexpected retransmission;
- source instability;
- duplicate transfer patterns.

Duplicate detection should not mean deletion of evidence.

---

## 15. Content Hash

A cryptographic hash provides deterministic content identity.

Example:

```
import hashlib


def sha256_bytes(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()
```

For large files, stream the file:

```
import hashlib


def sha256_file(path: str) -> str:
    digest = hashlib.sha256()

    with open(path, "rb") as source:
        while chunk := source.read(1024 * 1024):
            digest.update(chunk)

    return digest.hexdigest()
```

Do not load a multi-gigabyte file into memory just to hash it.

---

## 16. Hash the Exact Artifact

A hash must represent the exact bytes whose identity you care about.

These can produce different hashes:

    LF line endings
    CRLF line endings

or:

    UTF-8
    UTF-8 with BOM

or:

    compressed bytes
    decompressed payload

Therefore decide whether identity is based on:

    raw artifact bytes

or:

    logical decompressed payload

For auditability, preserve the raw artifact hash even if a logical payload hash is also calculated.

---

## 17. Compressed Files

For:

    transactions.csv.gz

there are at least two useful hashes:

    compressed_hash
    decompressed_payload_hash

They answer different questions.

Compressed hash:

> Is this exact archive/object identical?

Payload hash:

> Does this contain the same logical uncompressed content?

Do not substitute one for the other without documenting the semantic meaning.

---

## 18. Hash Does Not Define Business Identity

Consider:

    D20260925/transactions.csv
    D20260926/transactions.csv

Both happen to contain zero transactions.

Their content may be identical.

Their business identities are different:

    delivery 2026-09-25
    delivery 2026-09-26

Therefore:

    same hash
        !=
    same business file

Hash is evidence for duplicate-content detection, not a universal business key.

---

## 19. Database Uniqueness Is the First Protection

If logical file identity is:

    delivery_id + file_id

enforce it in PostgreSQL.

```
CREATE TABLE etl_file (
    delivery_id TEXT NOT NULL,
    file_id TEXT NOT NULL,
    file_type TEXT NOT NULL,
    state TEXT NOT NULL,
    sha256 TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (delivery_id, file_id)
);
```

This prevents two concurrent workers from creating two logical file rows.

Application checks alone are insufficient because two workers can race.

---

## 20. Idempotent Registration

Use:

```
INSERT INTO etl_file (
    delivery_id,
    file_id,
    file_type,
    state,
    sha256
)
VALUES (
    %s,
    %s,
    %s,
    'DISCOVERED',
    %s
)
ON CONFLICT (delivery_id, file_id)
DO NOTHING;
```

This handles repeated discovery.

But DO NOTHING is not enough for replacement detection.

You must inspect the existing row when the identity already exists.

---

## 21. Detect Existing Identity

Use:

```
SELECT
    delivery_id,
    file_id,
    state,
    sha256
FROM etl_file
WHERE delivery_id = %s
  AND file_id = %s;
```

Then compare:

    existing hash
    vs
    observed hash

Possible outcomes:

    no row
        -> NEW

    same row + same hash
        -> DUPLICATE

    same row + different hash
        -> REPLACEMENT

---

## 22. Atomic Registration Pattern

A production implementation should avoid a race between:

    SELECT
    then
    INSERT

Prefer an atomic database operation or a transaction with the appropriate uniqueness constraint.

A useful pattern is:

```
INSERT INTO etl_file (
    delivery_id,
    file_id,
    file_type,
    state,
    sha256
)
VALUES (%s, %s, %s, 'DISCOVERED', %s)
ON CONFLICT (delivery_id, file_id)
DO UPDATE SET
    updated_at = now()
RETURNING
    delivery_id,
    file_id,
    state,
    sha256;
```

Then compare the returned existing hash with the observed hash.

The exact SQL can vary depending on whether you want to mutate the row during classification.

---

## 23. Do Not Overwrite the Existing Hash

This is dangerous:

```
UPDATE etl_file
SET sha256 = %s
WHERE delivery_id = %s
  AND file_id = %s;
```

If the existing file had:

    ABC

and the new artifact has:

    XYZ

you have destroyed the evidence needed to detect a replacement.

Instead:

    existing hash = ABC
    observed hash = XYZ
    classification = REPLACEMENT

Preserve both.

---

## 24. File Version Table

A version table provides a durable history.

```
CREATE TABLE etl_file_version (
    delivery_id TEXT NOT NULL,
    file_id TEXT NOT NULL,
    version INTEGER NOT NULL,
    sha256 TEXT NOT NULL,
    source_uri TEXT NOT NULL,
    size_bytes BIGINT,
    state TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (
        delivery_id,
        file_id,
        version
    ),

    UNIQUE (
        delivery_id,
        file_id,
        sha256
    )
);
```

This prevents the same content from being recorded as two versions for the same logical file.

---

## 25. Why Versioning Matters

Without versioning:

    transactions.csv
        hash ABC

is later replaced with:

    transactions.csv
        hash XYZ

You only know:

    current hash = XYZ

With versioning:

    v1 -> ABC
    v2 -> XYZ

Now you can answer:

- what arrived first?
- what changed?
- when did it change?
- which version was processed?
- which version is authoritative?

---

## 26. Duplicate Content Across Different Identities

Suppose:

    D100|customers -> ABC
    D101|customers -> ABC

This may be legitimate.

For example, the same customer snapshot may be repeated across daily deliveries.

Therefore do not globally reject:

    hash ABC already exists

unless the source contract says content must be unique globally.

Instead, maintain a content index as evidence:

```
CREATE INDEX idx_etl_file_sha256
ON etl_file (sha256);
```

Use it to find possible duplicate content, not automatically to reject it.

---

## 27. Duplicate Content Classification

A useful query:

```
SELECT
    sha256,
    COUNT(*) AS occurrences
FROM etl_file
WHERE sha256 IS NOT NULL
GROUP BY sha256
HAVING COUNT(*) > 1;
```

This identifies repeated content.

Then investigate whether repeated identities are:

- expected snapshots;
- duplicate deliveries;
- producer retries;
- legitimate repeated files.

Detection and policy should remain separate.

---

## 28. Source URI as Evidence

Store the original source location.

Example:

    s3://bank-a/raw/2026-09-26/transactions.csv

or:

    sftp://partner/incoming/transactions.csv

Source URI helps answer:

> Where did this artifact come from?

But it is not necessarily a stable business identity.

A file can move:

    incoming/transactions.csv
        ->
    archive/transactions.csv

The logical file remains the same.

---

## 29. Object Version IDs

Object storage can provide stronger source identity.

For example, an object store may expose:

    object key
    version ID
    ETag
    last modified
    size

When versioning is enabled, store the source version ID.

Do not assume an ETag is always a SHA-256 checksum. Its semantics vary by storage system and upload method.

Treat provider metadata according to that provider's documented meaning.

---

## 30. SFTP Duplicate Detection

SFTP often gives weaker object identity than object storage.

Possible evidence:

- normalized filename;
- size;
- modification time;
- checksum;
- source directory;
- transfer completion marker.

Do not rely on modification time alone.

Two different files can share timestamps.

Two transfers of the same file can have different timestamps.

Use the strongest available combination.

---

## 31. Duplicate Detection Evidence Hierarchy

A practical hierarchy is:

    producer delivery/file ID
            |
            v
    logical filename + delivery identity
            |
            v
    source object version ID
            |
            v
    content hash
            |
            v
    size + metadata

Higher-level business identity is generally stronger for business deduplication.

Lower-level evidence is useful for artifact comparison.

---

## 32. Duplicate Detection Decision Function

A simplified classifier:

```
from dataclasses import dataclass
from typing import Optional


@dataclass(frozen=True)
class ExistingFile:
    file_id: str
    sha256: Optional[str]
    state: str


def classify_file(
    existing: ExistingFile | None,
    observed_hash: str,
) -> str:
    if existing is None:
        return "NEW"

    if existing.sha256 == observed_hash:
        if existing.state == "COMPLETE":
            return "DUPLICATE"
        return "REPLAY"

    return "REPLACEMENT"
```

Real implementations should include delivery state, version, source URI, and explicit replay/correction signals.

---

## 33. Why State Changes the Classification

Suppose the same file arrives again.

Existing state:

    COMPLETE

Same hash:

    DUPLICATE

Existing state:

    RETRYABLE_FAILURE

Same hash:

    REPLAY

The second observation may actually be the source retrying an incomplete processing attempt.

The business result can be:

    process again

because downstream idempotency should make it safe.

---

## 34. Duplicate After Quarantine

Existing state:

    QUARANTINED

Same content arrives again.

Do not automatically discard it.

Ask:

- Was the original failure corrected?
- Was the quarantine caused by transient infrastructure?
- Was the source artifact malformed?
- Has the producer resent the same bad file?

The answer determines whether to retry, reject, or preserve as another observation.

---

## 35. Duplicate During Processing

Existing state:

    EXTRACTING

Same file is discovered again.

The second worker should not process it concurrently.

Use an atomic claim:

```
UPDATE etl_file
SET
    state = 'EXTRACTING',
    attempt_count = attempt_count + 1,
    updated_at = now()
WHERE delivery_id = %s
  AND file_id = %s
  AND state IN ('DISCOVERED', 'RETRYABLE_FAILURE');
```

If zero rows are updated, the worker did not acquire the file.

This is concurrency control, not merely duplicate detection.

---

## 36. Duplicate After Successful Staging

Suppose:

    file = COMPLETE

and it is discovered again.

The pipeline should:

1. record the observation;
2. classify as duplicate;
3. avoid unnecessary extraction;
4. keep original processing evidence.

Do not delete the second observation.

It may be useful for source-quality monitoring.

---

## 37. Duplicate Record vs Duplicate File

These are different problems.

A duplicate file may cause duplicate records.

But duplicate records can also occur inside one valid file.

Example:

    transactions.csv
        TX001
        TX002
        TX001

The file itself is not necessarily duplicated.

Record-level deduplication belongs to the record identity layer.

Do not solve every duplicate record problem by rejecting files.

---

## 38. Duplicate File vs Duplicate Delivery

Example:

    D20260926
        transactions.csv

and:

    D20260926_RETRY
        transactions.csv

The files may have identical content.

If the producer intentionally created a new delivery ID, the second delivery may be legitimate.

The source contract determines whether:

    delivery ID is authoritative

or:

    business date + type defines uniqueness

Do not invent a global rule.

---

## 39. Replay Policy

A production pipeline should document replay semantics.

Example:

    replay same file
        ->
    allow processing
        ->
    idempotent staging
        ->
    audit replay

Alternative:

    replay same file
        ->
    skip extraction
        ->
    report existing result

Both can be correct depending on operational needs.

The important property is that the behavior is intentional.

---

## 40. Explicit Replay Flag

An operator can provide:

```
{
  "delivery_id": "D20260926",
  "file_id": "transactions",
  "reason": "reprocess_after_bug_fix"
}
```

The pipeline can record:

    replay_requested = true
    replay_reason = "reprocess_after_bug_fix"

This is stronger than trying to infer operator intent from timestamps.

---

## 41. Duplicate Processing Policy Table

| Classification | Default action |
|---|---|
| NEW | process |
| DUPLICATE / COMPLETE | record observation, skip |
| REPLAY | process if replay policy allows |
| REPLACEMENT | require correction/version policy |
| DUPLICATE_CONTENT / different identity | record and investigate if unexpected |
| IN_PROGRESS duplicate | do not process concurrently |
| QUARANTINED same content | inspect failure and retry policy |

The source contract can change these defaults.

---

## 42. Database Model

A practical model uses four concepts:

    etl_delivery
        |
        +-- etl_file
              |
              +-- etl_file_version
              |
              +-- etl_file_observation

Example relationships:

    delivery
      |
      +--> logical file
              |
              +--> versions
              |
              +--> observations

This model preserves both business identity and artifact history.

---

## 43. Observation Registration

```
INSERT INTO etl_file_observation (
    observation_id,
    delivery_id,
    file_id,
    source_uri,
    size_bytes,
    sha256,
    classification
)
VALUES (
    %s,
    %s,
    %s,
    %s,
    %s,
    %s,
    %s
);
```

Every observation can be retained.

The logical file table remains stable.

---

## 44. Idempotent Observation Handling

If the same discovery event itself may be duplicated, give the observation a deterministic event ID when possible.

For example:

    source_uri
    source_version_id
    observed_at_bucket

or preferably a producer event ID.

Then:

```
CREATE UNIQUE INDEX uq_file_observation_source_version
ON etl_file_observation (
    source_uri,
    sha256
);
```

Only use a uniqueness rule that matches the actual source semantics.

Do not invent a constraint that prevents legitimate repeated observations.

---

## 45. Database Constraint Strategy

Use constraints for invariants that must never be violated.

Examples:

    one logical file identity
        ->
    PRIMARY KEY (delivery_id, file_id)

    one version per file/version
        ->
    PRIMARY KEY (delivery_id, file_id, version)

    no duplicate hash inside one file version
        ->
    UNIQUE (delivery_id, file_id, sha256)

Do not encode business assumptions as constraints unless the source contract guarantees them.

---

## 46. Hashing at Ingestion Boundary

A useful sequence:

    discover
        |
        v
    acquire
        |
        v
    calculate hash
        |
        v
    register observation
        |
        v
    classify
        |
        v
    validate
        |
        v
    extract

Hashing too late can cause duplicate downloads or duplicate processing before classification.

Hashing too early may be impossible when the source does not expose content without downloading.

Choose the earliest reliable point where content identity is available.

---

## 47. Hashing Cost

Hashing requires reading the entire artifact.

For a:

    500 GB

file, SHA-256 requires substantial I/O.

Possible strategies:

- use provider checksums when their semantics are sufficient;
- use producer-provided hashes;
- calculate hash while streaming the extraction;
- hash once and persist it;
- avoid repeated hashing of immutable artifacts.

Do not calculate the same expensive hash repeatedly during every poll.

---

## 48. Hash During Streaming Extraction

For large immutable files, calculate the hash while reading.

```
import hashlib


def stream_records(source):
    digest = hashlib.sha256()

    for chunk in source:
        digest.update(chunk)
        yield chunk

    return digest.hexdigest()
```

In actual implementations, return the digest through a result object rather than relying on a generator return value.

The important principle is:

    one read
        ->
    extraction + hashing

rather than:

    read once to hash
        ->
    read again to extract

---

## 49. Duplicate Detection and Large Files

E61 taught bounded streaming.

E63 adds:

    streaming content identity

The pipeline should avoid:

    read entire file
    hash
    keep file in memory
    parse

Prefer:

    stream
      |
      +--> hash
      |
      +--> parse
      |
      +--> batch
      |
      +--> stage

This is both memory-efficient and operationally useful.

---

## 50. Duplicate Detection Before Extraction

If the source exposes a trustworthy content identity before download, duplicate detection can happen early.

Example:

    object version ID
    checksum
    immutable source key

Then:

    discover
        ->
    compare identity
        ->
    skip duplicate
        ->
    no download

This saves network and compute.

Only use provider metadata when its semantics are trustworthy.

---

## 51. Duplicate Detection After Acquisition

If the source gives no reliable identity, acquisition may be required first:

    discover
        ->
    download
        ->
    hash
        ->
    classify
        ->
    process

In this case, store the raw artifact immutably so future duplicate classification does not require another source download.

---

## 52. Immutable Raw Storage

A robust architecture:

    source
      |
      v
    raw immutable artifact
      |
      +--> hash
      +--> validation
      +--> extraction
      +--> staging

Raw storage gives you:

- replay;
- audit;
- duplicate comparison;
- recovery;
- forensic analysis.

Do not delete the only copy simply because the file was classified as duplicate.

---

## 53. Duplicate Archive Policy

For an exact duplicate:

    original artifact
    duplicate artifact

Possible policies:

1. retain both;
2. retain original and record duplicate observation;
3. retain one physical copy with multiple references.

For high-audit environments, preserving evidence may be important.

Balance audit requirements against storage cost.

---

## 54. Duplicate File Alerting

Not every duplicate should page an operator.

Separate:

    expected duplicate observation

from:

    unexpected duplicate behavior

Expected:

    polling sees same immutable object every minute

Unexpected:

    producer sends same business file 20 times

Possible metrics:

    duplicate_file_observations_total
    duplicate_content_total
    file_replacements_total
    file_replays_total

Alert on abnormal rates rather than every occurrence.

---

## 55. Structured Duplicate Logs

Example:

```
logger.info(
    "file_duplicate_detected",
    extra={
        "delivery_id": delivery_id,
        "file_id": file_id,
        "observed_sha256": observed_hash,
        "existing_sha256": existing_hash,
        "classification": classification,
        "existing_state": existing_state,
    },
)
```

Useful fields:

- delivery ID;
- file ID;
- source URI;
- hash;
- classification;
- existing state;
- attempt;
- producer ID.

Avoid logging sensitive payload contents.

---

## 56. Metrics

Useful metrics:

    file_observations_total
    duplicate_file_observations_total
    duplicate_content_total
    file_replacements_total
    file_replays_total
    duplicate_processing_prevented_total

Useful dimensions:

    source_system
    delivery_type
    file_type
    classification

Avoid high-cardinality file IDs as ordinary metric labels.

---

## 57. Test Matrix

At minimum test:

1. first observation;
2. repeated identical observation;
3. same identity/different hash;
4. different identity/same hash;
5. duplicate while processing;
6. duplicate after completion;
7. duplicate after failure;
8. replay;
9. concurrent registration;
10. concurrent processing;
11. compressed artifact;
12. large artifact;
13. duplicate record inside a valid file.

Each case should have an explicit expected classification.

---

## 58. Unit Test — New File

```
def test_new_file():
    existing = None

    assert classify_file(
        existing,
        "ABC",
    ) == "NEW"
```

The first observation must not be classified as duplicate.

---

## 59. Unit Test — Completed Duplicate

```
def test_completed_duplicate():
    existing = ExistingFile(
        file_id="transactions",
        sha256="ABC",
        state="COMPLETE",
    )

    assert classify_file(
        existing,
        "ABC",
    ) == "DUPLICATE"
```

This is the basic idempotent discovery case.

---

## 60. Unit Test — Replacement

```
def test_replacement():
    existing = ExistingFile(
        file_id="transactions",
        sha256="ABC",
        state="COMPLETE",
    )

    assert classify_file(
        existing,
        "XYZ",
    ) == "REPLACEMENT"
```

A different hash must not silently become a duplicate.

---

## 61. Unit Test — Replay

```
def test_replay_after_retryable_failure():
    existing = ExistingFile(
        file_id="transactions",
        sha256="ABC",
        state="RETRYABLE_FAILURE",
    )

    assert classify_file(
        existing,
        "ABC",
    ) == "REPLAY"
```

This supports safe recovery.

---

## 62. Integration Test — Concurrent Registration

Start two workers with:

    same delivery_id
    same file_id

Both attempt registration.

Expected:

    one logical file row

Possible result:

    worker A -> inserts
    worker B -> conflict

No duplicate logical file should exist.

---

## 63. Integration Test — Concurrent Processing

Register one file.

Start two workers.

Both try:

    DISCOVERED -> EXTRACTING

Expected:

    exactly one worker claims the file

The other worker must not extract concurrently.

This is essential because duplicate detection without concurrency control still permits duplicate work.

---

## 64. Integration Test — Replay

Process:

    D20260926|transactions

to COMPLETE.

Submit the same file again.

Expected:

    classification = DUPLICATE
    no duplicate staging records
    original result remains auditable

If replay was explicitly requested:

    classification = REPLAY
    processing follows replay policy
    staging remains idempotent

---

## 65. Integration Test — Replacement

Process:

    file version 1
    hash ABC

Then submit:

    same logical identity
    hash XYZ

Expected:

    classification = REPLACEMENT

And:

    version 1 remains auditable
    version 2 is explicitly recorded

The pipeline must not silently overwrite version 1.

---

## 66. Integration Test — Same Content, Different Delivery

Create:

    D100|customers -> hash ABC
    D101|customers -> hash ABC

Expected:

    both can exist

unless the source contract says repeated content is invalid.

This test prevents an over-aggressive global hash uniqueness rule.

---

## 67. Integration Test — Duplicate Discovery Storm

Simulate:

    100 discovery events
    same file
    same content

Expected:

    one logical file
    no duplicate staging
    observations retained according to policy
    duplicate metric increases

This models a noisy polling source.

---

## 68. Integration Test — Duplicate During Processing

Start extraction.

Before it finishes, issue another discovery.

Expected:

    second observation -> DUPLICATE or IN_PROGRESS
    second worker -> cannot claim file

The result must not produce two concurrent extractions.

---

## 69. Intentional Failure Drill — Duplicate File

1. Process a file to COMPLETE.
2. Place the same artifact back into the incoming location.
3. Run discovery.
4. Verify duplicate classification.
5. Verify no second business load.
6. Verify observation/audit evidence exists.

Expected:

    original processing preserved
    duplicate processing prevented

---

## 70. Intentional Failure Drill — Replacement

1. Process file version 1.
2. Change file contents.
3. Keep the same logical file identity.
4. Rediscover.
5. Verify REPLACEMENT classification.
6. Verify original hash remains.
7. Verify new artifact is not silently processed as a duplicate.

Then follow the explicit correction policy.

---

## 71. Intentional Failure Drill — Replay

1. Process file successfully.
2. Trigger an explicit replay.
3. Run extraction again.
4. Verify record-level idempotency.
5. Verify no duplicate target rows.
6. Verify replay audit metadata.

This proves that replay and deduplication can coexist.

---

## 72. Intentional Failure Drill — Concurrent Workers

1. Register one file.
2. Start worker A.
3. Start worker B immediately.
4. Both attempt to claim.
5. Inspect database state.

Expected:

    one claimant
    one non-claimant
    one extraction

This proves that the database invariant protects the pipeline.

---

## 73. Recovery — Duplicate

If a file is classified DUPLICATE:

1. inspect existing logical file;
2. verify existing hash;
3. verify existing processing state;
4. confirm no unresolved prior failure;
5. record observation;
6. skip unnecessary processing;
7. monitor duplicate rate.

Do not delete the duplicate evidence automatically.

---

## 74. Recovery — Replacement

If a replacement is detected:

1. preserve original artifact;
2. preserve original hash;
3. create a new file version;
4. identify correction reason;
5. determine authoritative version;
6. decide whether downstream reprocessing is required;
7. reprocess safely;
8. reconcile;
9. preserve both versions in audit history.

Replacement is a business decision, not just a technical duplicate.

---

## 75. Recovery — Ambiguous Duplicate

If identity evidence conflicts:

    same filename
    different delivery ID
    same content

do not automatically reject or accept.

Investigate:

- producer contract;
- delivery identity;
- business date;
- sequence number;
- manifest;
- source metadata;
- previous processing state.

Then apply the documented policy.

---

## 76. Production Runbook

When duplicate files are reported:

### Step 1 — Identify the logical file

Find:

    delivery_id
    file_id

### Step 2 — Inspect observations

Compare:

    source URI
    hash
    size
    timestamps
    source version ID

### Step 3 — Inspect existing state

Determine:

    COMPLETE
    PROCESSING
    FAILED
    QUARANTINED

### Step 4 — Classify

Determine:

    DUPLICATE
    REPLAY
    REPLACEMENT
    DUPLICATE_CONTENT
    NEW

### Step 5 — Apply policy

Do not manually delete records to make the pipeline look clean.

### Step 6 — Reconcile

Verify:

    source artifact
    file state
    staging
    target

### Step 7 — Record outcome

Preserve the incident and classification.

---

## 77. Common Mistakes

### Mistake 1 — Using filename existence as deduplication

Two different deliveries can use the same filename.

**Fix:** Include delivery/business identity.

### Mistake 2 — Using hash as the business key

Two legitimate deliveries can contain identical content.

**Fix:** Treat hash as content evidence.

### Mistake 3 — Overwriting the stored hash

This destroys replacement evidence.

**Fix:** Preserve versions or observations.

### Mistake 4 — Using random UUID as file identity

Every discovery becomes a new file.

**Fix:** Generate deterministic business identity.

### Mistake 5 — Only checking duplicates in application code

Concurrent workers can race.

**Fix:** Enforce uniqueness in the database.

### Mistake 6 — Treating every repeat as an error

Polling and retries naturally produce repeated observations.

**Fix:** Distinguish expected duplicates from abnormal duplicate rates.

### Mistake 7 — Rejecting all same-content files

Identical snapshots can be legitimate.

**Fix:** Evaluate content equality together with business identity.

### Mistake 8 — Ignoring replacement files

Same identity with new content may be a correction.

**Fix:** Explicitly classify replacement.

### Mistake 9 — Deleting duplicate artifacts

This removes forensic evidence.

**Fix:** Preserve according to retention policy.

### Mistake 10 — Confusing duplicate file and duplicate record

A valid file can contain repeated business keys.

**Fix:** Handle record identity separately.

### Mistake 11 — Reprocessing duplicates without idempotency

This can double-load the destination.

**Fix:** Make staging and loading idempotent.

### Mistake 12 — Assuming ETag means SHA-256

Provider metadata can have different semantics.

**Fix:** Use documented provider semantics.

---

## 78. Debugging Questions

When a duplicate alert appears, ask:

1. What is the delivery ID?
2. What is the file ID?
3. What is the source URI?
4. What is the source version ID?
5. What is the observed hash?
6. What hash was previously recorded?
7. What is the previous processing state?
8. Is this a new observation or a new artifact?
9. Was replay explicitly requested?
10. Could this be a replacement?
11. Is identical content legitimate for this source?
12. Did any records reach the destination twice?

These questions should be answerable from durable metadata.

---

## 79. Production Implementation Sequence

### Step 1 — Define identity

Document:

- delivery identity;
- file identity;
- record identity;
- source artifact identity.

### Step 2 — Define duplicate classes

At minimum:

    NEW
    DUPLICATE
    REPLAY
    REPLACEMENT
    DUPLICATE_CONTENT

### Step 3 — Persist logical file identity

Create a unique constraint on the logical file key.

### Step 4 — Persist observations

Record meaningful source observations according to retention policy.

### Step 5 — Calculate content identity

Use producer/provider hashes where trustworthy or stream a local hash.

### Step 6 — Compare identity and content

Determine whether the artifact is new, duplicate, replay, or replacement.

### Step 7 — Prevent concurrent processing

Use atomic state transitions or row locking.

### Step 8 — Add idempotent staging

Make repeated processing safe.

### Step 9 — Add replacement/version handling

Never silently overwrite a processed artifact.

### Step 10 — Add observability

Measure duplicate, replay, and replacement rates.

### Step 11 — Run failure drills

Test duplicate discovery, replay, replacement, and concurrency.

---

## 80. Production Checklist

### Identity

- [ ] Delivery identity is deterministic.
- [ ] File identity is deterministic.
- [ ] Record identity is deterministic.
- [ ] Physical observation identity is separate.
- [ ] Source version IDs are stored where available.

### Duplicate Detection

- [ ] Repeated discovery is detected.
- [ ] Duplicate content can be identified.
- [ ] Same-identity/different-content is classified as replacement.
- [ ] Replay has an explicit policy.
- [ ] Different deliveries with identical content are not automatically rejected.

### Database

- [ ] Logical file uniqueness is enforced.
- [ ] Registration is idempotent.
- [ ] Concurrent claims are protected.
- [ ] Hashes are persisted.
- [ ] Replacement history is preserved.

### Processing

- [ ] Duplicate COMPLETE files are not unnecessarily reprocessed.
- [ ] Replay is safe.
- [ ] Staging is idempotent.
- [ ] Duplicate workers cannot process one logical file concurrently.
- [ ] Large files can be hashed without unbounded memory.

### Audit

- [ ] Duplicate observations are retained according to policy.
- [ ] Original artifacts remain recoverable.
- [ ] Replacements are versioned or otherwise auditable.
- [ ] Replay reasons can be recorded.

### Observability

- [ ] Duplicate rate is measurable.
- [ ] Replacement rate is measurable.
- [ ] Replay rate is measurable.
- [ ] Logs contain delivery and file identity.
- [ ] High-cardinality IDs are kept out of unsuitable metric labels.

### Testing

- [ ] New-file test exists.
- [ ] Duplicate test exists.
- [ ] Replacement test exists.
- [ ] Replay test exists.
- [ ] Concurrent-registration test exists.
- [ ] Concurrent-processing test exists.
- [ ] Same-content/different-delivery test exists.
- [ ] Duplicate storm test exists.

---

## 81. Production Tools You Should Know

### PostgreSQL

Use for:

- logical identity constraints;
- atomic registration;
- state transitions;
- version history;
- observation history;
- duplicate queries.

Important features:

- PRIMARY KEY;
- UNIQUE;
- ON CONFLICT;
- transactions;
- FOR UPDATE SKIP LOCKED;
- indexes.

### Python hashlib

Use for content hashing.

Know:

- SHA-256;
- streaming updates;
- incremental hashing;
- hashing while reading.

### Object Storage SDKs

Use provider metadata to reduce unnecessary downloads when supported.

Know:

- object version IDs;
- checksums;
- ETags and their actual semantics;
- object metadata;
- conditional reads.

Use the provider's documented semantics rather than assuming all checksum fields mean SHA-256.

---

## 82. Package Structure

A practical structure:

    etl/
      files/
        identity.py
        hashing.py
        classifier.py
        registry.py
        observations.py
        versions.py
        claiming.py

      deliveries/
        grouping.py
        replay.py
        reconciliation.py

      staging/
        writer.py

      tests/
        test_identity.py
        test_classifier.py
        test_registration.py
        test_replay.py
        test_replacement.py
        test_concurrency.py

The exact names can vary.

The separation of concerns is the important part.

---

## 83. Design Principle — Identity Before Deduplication

You cannot reliably deduplicate something whose identity you have not defined.

Start with:

    What does "same file" mean?

Then implement:

    identity
        ->
    evidence
        ->
    classification
        ->
    policy

Do not start with:

    if hash exists, skip

That shortcut creates false positives.

---

## 84. Design Principle — Detect, Then Decide

Duplicate detection should produce evidence and classification.

It should not automatically make every business decision.

For example:

    same hash
        ->
    DUPLICATE_CONTENT

does not necessarily mean:

    reject

The source contract may allow identical snapshots.

Separate:

    detection

from:

    processing policy

from:

    alerting policy

---

## 85. Design Principle — Preserve History

A production duplicate system should be able to answer:

    What arrived?
    When did it arrive?
    What was its identity?
    What content did it contain?
    Was it processed?
    Was it skipped?
    Was it replayed?
    Was it replaced?

If the system only stores final file state, these questions become difficult or impossible.

---

## 86. Design Principle — Idempotency Is the Safety Net

Duplicate detection reduces unnecessary work.

Idempotency protects the business result when duplicate work still occurs.

Use both:

    duplicate detection
            +
    idempotent processing

Do not assume duplicate detection alone is enough.

A race, crash, manual replay, or orchestration bug can still cause repeated processing.

---

## 87. Definition of Done

You are done with E63 when you can independently implement a system that:

1. defines logical file identity;
2. separates physical observation from business identity;
3. records content hashes safely;
4. detects repeated observations;
5. detects same-identity replacements;
6. detects duplicate content;
7. distinguishes replay from ordinary duplicate discovery;
8. prevents concurrent duplicate processing;
9. enforces uniqueness at the database layer;
10. preserves replacement history;
11. makes replay safe with idempotent staging;
12. handles large files without loading them fully into memory;
13. records useful duplicate evidence;
14. exposes duplicate/replay/replacement metrics;
15. passes concurrency and failure tests;
16. provides an operational recovery procedure.

If you can implement those mechanisms without copying this recipe line by line, you understand production duplicate file detection.

---

## 88. What You Learned

The central model is:

    logical identity
          |
          +---- content identity
          |
          +---- observation history
          |
          v
    duplicate classifier
          |
      +---+---+---+---+
      |   |   |   |   |
      v   v   v   v   v
     NEW DUP REPLAY REPL CONTENT-DUP
      |   |   |   |   |
      +---+---+---+---+
              |
              v
        explicit policy
              |
              v
        idempotent processing

The key rules are:

1. Define identity before defining deduplication.
2. Keep logical identity separate from content hash.
3. Keep observations separate from logical files.
4. Enforce uniqueness in the database.
5. Treat same identity/different content as a possible replacement.
6. Treat replay as an explicit operational case.
7. Do not reject different deliveries merely because their content is identical.
8. Preserve evidence instead of silently deleting duplicates.
9. Use duplicate detection together with idempotent processing.
10. Protect concurrent processing with atomic state transitions.

Next:

    E64 — Missing File Detection

E64 will focus specifically on proving that an expected file has not arrived, distinguishing genuinely missing files from files that are merely late, and producing actionable missing-file state.
