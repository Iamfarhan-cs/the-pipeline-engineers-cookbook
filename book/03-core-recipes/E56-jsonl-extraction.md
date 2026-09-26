# E56 — JSONL Extraction

## 1. Problem Recognition

JSONL, also called newline-delimited JSON or NDJSON, stores one complete JSON value per line.

Example:

@@@
{"payment_id":"P001","amount":100,"currency":"EUR"}
{"payment_id":"P002","amount":250,"currency":"USD"}
{"payment_id":"P003","amount":75,"currency":"GBP"}
@@@

This format is common in:

- event exports;
- application logs;
- API dumps;
- data platform feeds;
- object-storage deliveries;
- batch event files;
- machine-learning datasets.

JSONL is attractive for ETL because records can be processed incrementally.

You do not normally need to load the entire file into memory.

The production lifecycle is:

@@@
DISCOVER FILE
     |
     v
VALIDATE FILE
     |
     v
READ ONE LINE
     |
     v
PARSE JSON
     |
     v
VALIDATE RECORD
     |
     v
STAGE
     |
     +----> QUARANTINE INVALID RECORD
     |
     v
CHECKPOINT
     |
     v
COMPLETE
@@@

The important property is that the file is a sequence of independently parseable JSON values.

---

# 2. Concept and Reasoning

## 2.1 JSONL is not ordinary JSON

This is valid JSON:

@@@
[
  {"id": 1},
  {"id": 2},
  {"id": 3}
]
@@@

This is JSONL:

@@@
{"id":1}
{"id":2}
{"id":3}
@@@

The first is one JSON document.

The second is three separate JSON values.

That difference determines the extraction strategy.

With JSONL, the natural processing unit is a line.

---

# 3. Why JSONL Works Well for ETL

A large JSON document may require a streaming parser to avoid loading the whole structure.

JSONL has a simpler property:

@@@
for line in file:
    record = json.loads(line)
    process(record)
@@@

The operating system and Python can read the file incrementally.

Memory therefore depends primarily on the current record and staging batch rather than the entire file.

This makes JSONL useful for high-volume feeds.

It does not, however, solve every problem.

A single JSONL line can still be extremely large.

A file can still be malformed.

A producer can still change schema.

Duplicate files can still be replayed.

---

# 4. JSONL Extraction Contract

Before implementation, define:

- expected file extension;
- encoding;
- whether blank lines are allowed;
- whether whitespace-only lines are allowed;
- whether each line must contain a JSON object;
- required fields;
- field types;
- maximum line size;
- duplicate policy;
- malformed-line policy;
- ordering requirements;
- source identity;
- checkpoint policy;
- compression behavior.

Example:

@@@
{
  "format": "jsonl",
  "encoding": "utf-8",
  "blank_lines": "reject",
  "root_type": "object",
  "required_fields": [
    "payment_id",
    "amount",
    "currency"
  ],
  "max_line_bytes": 1048576
}
@@@

The contract should be explicit before production deployment.

---

# 5. Line Boundaries Are Data Boundaries

In JSONL, the newline is part of the transport structure.

Example:

@@@
{"id":1}
{"id":2}
@@@

Line 1 and line 2 are independent records.

This enables:

- line-level error reporting;
- line-level metrics;
- deterministic record positions;
- partial retries;
- bounded memory;
- checkpointing.

The line number is therefore useful technical lineage.

---

# 6. Basic JSONL Reader

The simplest implementation is:

@@@
import json

with open(
    "payments.jsonl",
    "r",
    encoding="utf-8",
) as source:
    for line_number, line in enumerate(
        source,
        start=1,
    ):
        record = json.loads(line)
        print(record)
@@@

This is the foundation.

Production code needs additional handling around it.

---

# 7. Blank Lines

A blank line is not a JSON value.

Example:

@@@
{"id":1}

{"id":2}
@@@

Decide whether blank lines are:

- invalid;
- ignored;
- counted;
- reported.

For strict feeds, reject them.

For tolerant feeds, explicitly skip them.

Example:

@@@
if not line.strip():
    continue
@@@

Do not silently ignore blank lines when the delivery contract expects exactly one record per line.

---

# 8. Whitespace

A line can contain surrounding whitespace:

@@@
  {"id":1}
@@@

JSON parsers can normally handle this.

Avoid unnecessary transformations that change the payload.

A safe pattern is:

@@@
raw_line = line.rstrip("\r\n")
record = json.loads(raw_line)
@@@

Preserve the original line separately if exact source reconstruction matters.

---

# 9. Windows and Unix Line Endings

JSONL may arrive with:

- LF;
- CRLF.

Python text iteration handles normal newline translation.

If exact byte-level lineage matters, process binary input and decode deliberately.

Do not build record identity from a platform-dependent representation if byte-for-byte reproducibility is required.

---

# 10. Malformed Line Handling

Example:

@@@
{"id":1}
{"id":2,
{"id":3}
@@@

Line 2 is malformed.

A production extractor should identify:

- source file;
- line number;
- parse error;
- raw record where policy permits;
- extractor version.

Example:

@@@
import json

try:
    record = json.loads(line)
except json.JSONDecodeError as exc:
    raise ValueError(
        f"Malformed JSONL record at line {line_number}: "
        f"line={exc.lineno}, column={exc.colno}"
    ) from exc
@@@

Whether processing stops or continues depends on the delivery contract.

---

# 11. Strict vs Tolerant Processing

There are two common models.

## Strict

One invalid line fails the entire file.

Use when:

- the file is an atomic financial delivery;
- row counts must reconcile exactly;
- partial processing is not acceptable.

## Tolerant

Invalid lines are quarantined and valid lines continue.

Use when:

- records are independently meaningful;
- partial success is explicitly supported;
- the producer contract defines quarantine behavior.

Do not choose tolerant processing simply because it is easier operationally.

It changes the meaning of successful delivery.

---

# 12. Record Validation

Valid JSON does not mean valid business data.

Example:

@@@
{"payment_id": "P001", "amount": "unknown"}
@@@

The JSON is syntactically valid.

The record may still violate the ETL contract.

Validation should occur after parsing:

@@@
def validate_record(record):
    if not isinstance(record, dict):
        raise ValueError("Record must be a JSON object")

    required = ["payment_id", "amount", "currency"]

    for field in required:
        if field not in record:
            raise ValueError(
                f"Missing required field: {field}"
            )

    if not isinstance(record["payment_id"], str):
        raise ValueError(
            "payment_id must be a string"
        )
@@@

Keep syntax parsing and business-contract validation as separate stages.

---

# 13. Root Type Validation

JSONL does not necessarily mean every line is an object.

These are valid JSON values:

@@@
123
@@@

@@@
"hello"
@@@

@@@
null
@@@

@@@
[1, 2, 3]
@@@

If your contract requires objects, enforce it:

@@@
if not isinstance(record, dict):
    raise ValueError(
        "JSONL record must be an object"
    )
@@@

Do not assume the format guarantees the semantic type.

---

# 14. Missing vs Null

Consider:

@@@
{"payment_id":"P001"}
@@@

and:

@@@
{"payment_id":"P001","description":null}
@@@

The first has a missing field.

The second explicitly provides null.

Preserve that distinction when downstream behavior depends on it.

Use explicit validation rather than relying on get for required fields.

---

# 15. Line Number as Technical Identity

A line number is useful lineage:

@@@
source_file = "payments_20260926.jsonl"
line_number = 15342
@@@

However, line number alone is not a globally stable business identifier.

If a producer inserts a line near the beginning, every later line number changes.

Use line number as:

- source position;
- debugging reference;
- checkpoint position;

not automatically as the business key.

---

# 16. Source Hash

Calculate a stable source hash.

@@@
import hashlib

def sha256_file(path):
    digest = hashlib.sha256()

    with path.open("rb") as source:
        for chunk in iter(
            lambda: source.read(1024 * 1024),
            b"",
        ):
            digest.update(chunk)

    return digest.hexdigest()
@@@

A source hash allows a pipeline to recognize the same physical file during replay.

For large files, calculate the hash while reading when the architecture permits it, rather than reading the file twice.

---

# 17. Deterministic Record Identity

A practical technical identity is:

@@@
(source_hash, line_number)
@@@

Example:

@@@
source_hash = "abc123..."
line_number = 15342
@@@

Together they identify a specific record position in a specific source artifact.

If the producer supplies a stable business ID, preserve it separately:

@@@
technical_identity = (source_hash, line_number)
business_identity = payment_id
@@@

Do not confuse the two.

---

# 18. Idempotent Staging

Example table:

@@@
CREATE TABLE jsonl_staging (
    source_hash TEXT NOT NULL,
    line_number BIGINT NOT NULL,
    record_id TEXT,
    payload JSONB NOT NULL,
    extracted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (source_hash, line_number)
);
@@@

Insert:

@@@
INSERT INTO jsonl_staging (
    source_hash,
    line_number,
    record_id,
    payload
)
VALUES (%s, %s, %s, %s)
ON CONFLICT (source_hash, line_number)
DO NOTHING;
@@@

Now a replay does not automatically duplicate staged records.

---

# 19. Business-Key Uniqueness

Technical identity and business identity are different.

You may additionally require:

@@@
CREATE UNIQUE INDEX uq_jsonl_payment
ON jsonl_staging (record_id);
@@@

Only add this if the source contract says record_id must be globally unique in the relevant scope.

Do not add arbitrary uniqueness constraints that can reject legitimate repeated events.

For event streams, the same business entity may legitimately appear many times.

---

# 20. Streaming Batches

A common pattern is:

@@@
BATCH_SIZE = 1000
batch = []

for line_number, line in enumerate(source, start=1):
    record = parse_and_validate(line)

    batch.append(
        (source_hash, line_number, record)
    )

    if len(batch) >= BATCH_SIZE:
        stage_batch(batch)
        batch.clear()

if batch:
    stage_batch(batch)
@@@

Memory remains bounded by the batch size plus the largest record.

Tune the batch size according to database performance and record size.

---

# 21. Transaction Boundaries

If a batch contains 1,000 records and record 997 is invalid, decide what happens.

Possible policies:

### Batch atomicity

The entire batch rolls back.

### Record quarantine

The invalid record is quarantined and the remaining valid records commit.

### File atomicity

Nothing commits until the entire file passes.

The policy must be explicit.

Do not accidentally implement batch-level partial success when the business requires file-level atomicity.

---

# 22. A Production-Oriented Parser

@@@
import json
from typing import Iterator


class JsonlExtractionError(Exception):
    pass


def parse_jsonl(
    source,
    required_fields,
) -> Iterator[tuple[int, dict]]:
    for line_number, line in enumerate(
        source,
        start=1,
    ):
        raw = line.rstrip("\r\n")

        if not raw.strip():
            raise JsonlExtractionError(
                f"Blank line at {line_number}"
            )

        try:
            record = json.loads(raw)
        except json.JSONDecodeError as exc:
            raise JsonlExtractionError(
                f"Invalid JSON at line {line_number}: "
                f"column={exc.colno}"
            ) from exc

        if not isinstance(record, dict):
            raise JsonlExtractionError(
                f"Line {line_number} must contain "
                f"a JSON object"
            )

        for field in required_fields:
            if field not in record:
                raise JsonlExtractionError(
                    f"Line {line_number} missing "
                    f"required field {field}"
                )

        yield line_number, record
@@@

Usage:

@@@
with open(
    "payments.jsonl",
    "r",
    encoding="utf-8",
) as source:
    for line_number, record in parse_jsonl(
        source,
        ["payment_id", "amount", "currency"],
    ):
        process_record(
            line_number,
            record,
        )
@@@

The generator keeps extraction incremental.

---

# 23. Tolerant Parser with Quarantine

If the contract permits partial processing:

@@@
def parse_jsonl_tolerant(
    source,
    required_fields,
):
    for line_number, line in enumerate(
        source,
        start=1,
    ):
        raw = line.rstrip("\r\n")

        if not raw.strip():
            yield {
                "status": "rejected",
                "line_number": line_number,
                "error_code": "BLANK_LINE",
                "raw": raw,
            }
            continue

        try:
            record = json.loads(raw)

            if not isinstance(record, dict):
                raise ValueError(
                    "record is not an object"
                )

            for field in required_fields:
                if field not in record:
                    raise ValueError(
                        f"missing field: {field}"
                    )

            yield {
                "status": "accepted",
                "line_number": line_number,
                "record": record,
            }

        except (
            json.JSONDecodeError,
            ValueError,
        ) as exc:
            yield {
                "status": "rejected",
                "line_number": line_number,
                "error_code": "INVALID_RECORD",
                "error_message": str(exc),
                "raw": raw,
            }
@@@

This makes the partial-processing policy explicit.

---

# 24. Do Not Hide Parse Errors

Bad:

@@@
try:
    record = json.loads(line)
except Exception:
    continue
@@@

This silently loses data.

Better:

@@@
except json.JSONDecodeError as exc:
    quarantine(
        line_number=line_number,
        error_code="MALFORMED_JSON",
        error_message=str(exc),
        raw=line,
    )
@@@

Every rejected record must be explainable.

---

# 25. Large Record Protection

JSONL provides line-level streaming, but a single line can still be huge.

Example:

@@@
{"id":"P001","description":"<many megabytes>"}
@@@

Define a maximum line size.

For byte-level enforcement, read binary input.

Conceptually:

@@@
MAX_LINE_BYTES = 1024 * 1024
@@@

Reject or quarantine lines that exceed the configured limit.

Do not allow one malformed record to consume unlimited memory.

---

# 26. Binary Reading for Strict Limits

Example:

@@@
with open(
    "payments.jsonl",
    "rb",
) as source:
    for line_number, raw_line in enumerate(
        source,
        start=1,
    ):
        if len(raw_line) > MAX_LINE_BYTES:
            raise ValueError(
                f"Line {line_number} exceeds "
                f"maximum size"
            )

        line = raw_line.decode("utf-8")
        record = json.loads(line)
@@@

This makes byte limits explicit.

The tradeoff is that encoding validation now becomes your responsibility.

---

# 27. Encoding

UTF-8 should normally be the source contract.

Do not use:

@@@
line.decode("utf-8", errors="ignore")
@@@

This can silently remove invalid bytes.

Prefer strict decoding:

@@@
line.decode("utf-8")
@@@

If decoding fails, classify the record or file according to the contract.

---

# 28. Compression

JSONL is frequently compressed:

@@@
payments.jsonl.gz
@@@

Use streaming decompression:

@@@
import gzip

with gzip.open(
    "payments.jsonl.gz",
    "rt",
    encoding="utf-8",
) as source:
    for line_number, line in enumerate(
        source,
        start=1,
    ):
        record = json.loads(line)
        process_record(record)
@@@

Do not create a huge uncompressed temporary copy unless required.

---

# 29. Object Storage

A production JSONL file may live in object storage.

The logical flow remains:

@@@
object storage
      |
      v
stream bytes
      |
      v
decode lines
      |
      v
parse JSON
      |
      v
validate
      |
      v
stage
@@@

Keep object retrieval separate from record parsing.

That makes the parser testable with ordinary local file handles and in-memory streams.

---

# 30. Source Immutability

JSONL extraction assumes that the source does not change during processing.

If a producer overwrites the same object while the extractor is reading it, the result may be inconsistent.

Use the file-arrival and completeness controls from earlier recipes:

- stable object size;
- stable object version;
- checksum;
- producer completion marker;
- immutable object naming.

Do not solve source mutability inside the JSON parser.

---

# 31. Checkpointing

Line-oriented data is naturally checkpointable.

Example:

@@@
source_hash = "abc123"
last_committed_line = 500000
@@@

If the process fails after line 500,000, the next attempt can determine what has already been staged.

A robust checkpoint should include enough source identity:

@@@
source_hash
line_number
extractor_version
@@@

Never checkpoint only a line number without identifying the source file.

---

# 32. Checkpoint vs Idempotency

These are related but different.

### Checkpoint

Answers:

> Where should I resume?

### Idempotency

Answers:

> What happens if I process something again?

A strong pipeline uses both when appropriate.

For example:

@@@
checkpoint -> reduce restart cost
idempotency -> prevent duplicate state
@@@

Do not rely on checkpoint state alone to guarantee correctness.

---

# 33. Resume Example

Conceptually:

@@@
last_committed_line = get_checkpoint(
    source_hash
)

for line_number, line in enumerate(
    source,
    start=1,
):
    if line_number <= last_committed_line:
        continue

    process(line)
@@@

This only works safely if:

- source bytes are immutable;
- line numbering is stable;
- the checkpoint is durable;
- staging is idempotent.

---

# 34. Exactly-Once Expectations

Avoid claiming that the file reader itself provides exactly-once processing.

A practical architecture is:

@@@
at-least-once reading
        +
durable checkpoint
        +
idempotent staging
        +
deterministic identity
        =
safe replay
@@@

The extractor may read the same line more than once.

The resulting database state should remain correct.

---

# 35. Ordering

JSONL preserves source order.

That does not automatically mean downstream processing must preserve it.

Ask:

- Does order have business meaning?
- Is there an event sequence number?
- Is the file a snapshot or event stream?
- Can records be processed independently?

If order matters, preserve:

@@@
source_hash
line_number
event_sequence
@@@

Do not infer business ordering from ingestion time.

---

# 36. Event Time vs File Position

A JSONL event may contain:

@@@
{
  "event_id": "E001",
  "occurred_at": "2026-09-26T08:00:00Z"
}
@@@

The line number tells you where the record appeared in the file.

The timestamp tells you when the business event occurred.

They are not interchangeable.

Keep both when useful.

---

# 37. Deduplication

There are several possible duplicate concepts:

1. same physical file replayed;
2. same line replayed;
3. same event ID appearing twice;
4. two distinct events representing the same business object.

Use separate controls for each.

For example:

@@@
(source_hash, line_number)
@@@

for technical source identity, and:

@@@
event_id
@@@

for event identity if the producer guarantees it.

Do not collapse distinct events simply because they share a business entity ID.

---

# 38. Raw Payload Preservation

Store the parsed JSON payload in staging when practical.

Example:

@@@
{
  "source_hash": "abc123",
  "line_number": 42,
  "record_id": "P042",
  "payload": {
    "payment_id": "P042",
    "amount": 100,
    "currency": "EUR"
  }
}
@@@

This makes downstream transformations reproducible.

If raw retention is restricted by policy, retain the minimum permitted evidence and a source reference.

---

# 39. Staging Schema

Example PostgreSQL schema:

@@@
CREATE TABLE jsonl_staging (
    source_hash TEXT NOT NULL,
    source_uri TEXT NOT NULL,
    line_number BIGINT NOT NULL,
    record_id TEXT,
    payload JSONB NOT NULL,
    extracted_at TIMESTAMPTZ NOT NULL,
    extractor_version TEXT NOT NULL,
    PRIMARY KEY (source_hash, line_number)
);
@@@

Technical metadata stays separate from business payload.

---

# 40. Extraction Run Table

Track the file-level operation separately.

@@@
CREATE TABLE jsonl_extraction_run (
    run_id BIGSERIAL PRIMARY KEY,
    source_uri TEXT NOT NULL,
    source_hash TEXT NOT NULL,
    extractor_version TEXT NOT NULL,
    status TEXT NOT NULL,
    lines_seen BIGINT NOT NULL DEFAULT 0,
    records_staged BIGINT NOT NULL DEFAULT 0,
    records_rejected BIGINT NOT NULL DEFAULT 0,
    bytes_read BIGINT NOT NULL DEFAULT 0,
    last_committed_line BIGINT,
    started_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ
);
@@@

This gives operators a durable processing record.

---

# 41. Batch Insert Example

Using psycopg:

@@@
from psycopg.types.json import Jsonb


INSERT_SQL = """
INSERT INTO jsonl_staging (
    source_hash,
    source_uri,
    line_number,
    record_id,
    payload,
    extracted_at,
    extractor_version
)
VALUES (
    %s, %s, %s, %s, %s, now(), %s
)
ON CONFLICT (source_hash, line_number)
DO NOTHING
"""


def insert_batch(
    connection,
    rows,
):
    with connection.cursor() as cursor:
        cursor.executemany(
            INSERT_SQL,
            [
                (
                    source_hash,
                    source_uri,
                    line_number,
                    record.get("payment_id"),
                    Jsonb(record),
                    extractor_version,
                )
                for (
                    source_hash,
                    source_uri,
                    line_number,
                    record,
                    extractor_version,
                ) in rows
            ],
        )
    connection.commit()
@@@

For very high volumes, use the database driver's bulk-loading capabilities where the data model permits them.

---

# 42. End-to-End Extraction Skeleton

@@@
import json
from pathlib import Path


BATCH_SIZE = 1000


def extract_jsonl(
    path: Path,
    source_hash: str,
    source_uri: str,
    connection,
    extractor_version: str,
):
    batch = []
    lines_seen = 0

    with path.open(
        "r",
        encoding="utf-8",
    ) as source:

        for line_number, line in enumerate(
            source,
            start=1,
        ):
            lines_seen += 1

            if not line.strip():
                raise ValueError(
                    f"Blank line at {line_number}"
                )

            try:
                record = json.loads(line)
            except json.JSONDecodeError as exc:
                raise ValueError(
                    f"Invalid JSON at line "
                    f"{line_number}"
                ) from exc

            if not isinstance(record, dict):
                raise ValueError(
                    f"Line {line_number} "
                    f"must contain an object"
                )

            for field in (
                "payment_id",
                "amount",
                "currency",
            ):
                if field not in record:
                    raise ValueError(
                        f"Line {line_number} "
                        f"missing {field}"
                    )

            batch.append(
                (
                    source_hash,
                    source_uri,
                    line_number,
                    record,
                    extractor_version,
                )
            )

            if len(batch) >= BATCH_SIZE:
                insert_batch(
                    connection,
                    batch,
                )
                batch.clear()

    if batch:
        insert_batch(
            connection,
            batch,
        )

    return lines_seen
@@@

This is a foundation, not a universal framework.

Keep source-specific rules in the source contract.

---

# 43. Avoid One Transaction Per Line

Bad:

@@@
for line in source:
    insert_record(line)
    connection.commit()
@@@

This creates excessive transaction overhead.

Prefer bounded batches.

The right batch size depends on:

- row size;
- database capacity;
- network latency;
- transaction duration;
- recovery requirements.

Benchmark rather than choosing an arbitrary number forever.

---

# 44. Handling Database Failure

Suppose 400,000 records have been processed and the database becomes unavailable.

A production design should:

1. preserve source identity;
2. preserve the last durable checkpoint;
3. roll back the failed transaction;
4. retry according to policy;
5. use idempotent inserts;
6. resume without creating duplicate state.

Do not advance the checkpoint before the corresponding database transaction is durable.

---

# 45. Checkpoint Ordering Rule

The safe order is:

@@@
READ RECORDS
    |
    v
WRITE DATA
    |
    v
COMMIT TRANSACTION
    |
    v
ADVANCE CHECKPOINT
@@@

Not:

@@@
READ RECORDS
    |
    v
ADVANCE CHECKPOINT
    |
    v
WRITE DATA
@@@

The second sequence can permanently skip data after a crash.

---

# 46. Failure Drill — Malformed Line

Inject:

@@@
{"payment_id":"P001","amount":100}
{"payment_id":"P002","amount":
{"payment_id":"P003","amount":300}
@@@

Expected strict behavior:

- line 2 is identified;
- extraction fails according to file policy;
- the error contains source and line number;
- no false success is recorded.

Expected tolerant behavior:

- line 2 is quarantined;
- lines 1 and 3 are processed;
- rejected count becomes 1;
- completion status reflects the partial-processing policy.

---

# 47. Failure Drill — Missing Field

Inject:

@@@
{"payment_id":"P002","amount":250}
@@@

Expected:

- JSON parsing succeeds;
- schema validation fails;
- error is classified as missing required field;
- line number is preserved;
- quarantine or file failure follows the contract.

This demonstrates why syntax validation and record validation must be separate.

---

# 48. Failure Drill — Duplicate Replay

Run the same file twice.

Expected:

@@@
first run:
1000 staged

second run:
1000 attempted
0 duplicate rows created
@@@

The second run should not silently corrupt counts.

Track attempted, inserted, and duplicate records separately where useful.

---

# 49. Failure Drill — Database Failure

Stop the database during a batch.

Verify:

1. transaction rolls back;
2. checkpoint does not advance past the failed batch;
3. retry is safe;
4. already committed batches remain;
5. no duplicate rows are created.

This validates the actual recovery model rather than merely testing parser correctness.

---

# 50. Failure Drill — Oversized Record

Create one line larger than the configured limit.

Expected:

- record is rejected;
- error category is explicit;
- memory usage remains bounded;
- subsequent records follow the configured strict/tolerant policy.

A parser without resource limits is incomplete for untrusted feeds.

---

# 51. Failure Drill — Source Mutation

Start processing a file and modify it during extraction.

Expected production behavior:

- source immutability check detects the problem;
- extraction does not silently combine different versions;
- the run is failed or quarantined;
- the source must be replaced with a stable artifact.

This is a source-control problem, not something to hide inside the parser.

---

# 52. Observability

Track at least:

@@@
jsonl_files_discovered_total
jsonl_files_completed_total
jsonl_files_failed_total
jsonl_lines_seen_total
jsonl_records_staged_total
jsonl_records_rejected_total
jsonl_parse_errors_total
jsonl_validation_errors_total
jsonl_duplicate_records_total
jsonl_bytes_read_total
jsonl_extraction_duration_seconds
@@@

Useful dimensions:

- source;
- dataset;
- environment;
- extractor version;
- outcome;
- error category.

Avoid putting raw record IDs or arbitrary payload values into metric labels.

---

# 53. Structured Logging

Success:

@@@
{
  "event": "jsonl_extraction_completed",
  "source": "payments",
  "source_hash": "abc123",
  "lines_seen": 1000000,
  "records_staged": 1000000,
  "records_rejected": 0,
  "duration_ms": 62000
}
@@@

Failure:

@@@
{
  "event": "jsonl_extraction_failed",
  "source": "payments",
  "source_hash": "abc123",
  "line_number": 532901,
  "error_code": "MALFORMED_JSON"
}
@@@

Log metadata, not sensitive payloads.

---

# 54. Data Quality Invariants

At completion, verify relevant invariants.

For strict processing:

@@@
lines_seen = records_staged
records_rejected = 0
@@@

For tolerant processing:

@@@
lines_seen =
    records_staged
    + records_rejected
    + explicitly_skipped_lines
@@@

Also verify:

- source hash is stable;
- line numbers are unique;
- required identifiers exist;
- checkpoint does not exceed durable data;
- staged count matches the run policy.

---

# 55. File-Level Reconciliation

Some producers provide expected counts.

Example manifest:

@@@
{
  "file": "payments.jsonl",
  "expected_records": 1000000
}
@@@

After extraction:

@@@
expected_records = 1000000
lines_seen = 1000000
records_staged = 1000000
@@@

If the counts do not reconcile, the delivery should not be marked complete unless the contract explicitly permits the difference.

JSONL parsing alone cannot prove delivery completeness.

---

# 56. Schema Evolution

JSONL makes additive fields easy:

@@@
{"id":"P001","amount":100}
{"id":"P002","amount":200,"channel":"mobile"}
@@@

Your extractor can preserve unknown fields while enforcing required fields.

But these changes require explicit treatment:

- required field removed;
- field type changed;
- identifier semantics changed;
- nested structure changed;
- encoding changed.

Do not silently coerce breaking changes just to keep the pipeline green.

---

# 57. Versioned Contracts

For important producers, store:

@@@
producer
dataset
format_version
extractor_version
schema_version
@@@

This allows operators to determine which parser contract processed a record.

Do not confuse application version with data-contract version.

They answer different questions.

---

# 58. Object Storage and Atomic Delivery

For object-storage JSONL feeds, prefer immutable object keys or versioned objects.

A robust sequence is:

@@@
producer writes object
       |
       v
producer completes delivery
       |
       v
arrival detected
       |
       v
file completeness verified
       |
       v
JSONL extraction
       |
       v
staging
@@@

This builds on the earlier file-arrival and file-completeness recipes.

Do not parse a file merely because its name has appeared.

---

# 59. Compression and Splitting

JSONL can be split conceptually by record boundaries.

That makes parallel processing easier than arbitrary splitting of a normal JSON document.

However, parallelism requires care with:

- duplicate records;
- source positions;
- ordering;
- compressed formats;
- checkpoint identity;
- output commit semantics.

Do not add parallelism until the single-worker pipeline is deterministic.

---

# 60. Parallel Extraction

A large uncompressed JSONL file can sometimes be partitioned into byte ranges.

Each worker must start and end at valid line boundaries.

Conceptually:

@@@
worker 1: lines 1 - 1,000,000
worker 2: lines 1,000,001 - 2,000,000
worker 3: lines 2,000,001 - 3,000,000
@@@

The implementation should coordinate actual byte offsets and boundary discovery.

Never split blindly in the middle of a JSON record.

---

# 61. When Not to Parallelize

Do not automatically parallelize because a file is large.

Single-threaded streaming may be sufficient when:

- parsing is cheap;
- database writes dominate;
- the source is small enough;
- ordering matters;
- operational simplicity is more valuable.

Measure the bottleneck first.

---

# 62. Security

Treat every JSONL line as untrusted input.

Do not:

@@@
eval(record["code"])
exec(record["code"])
@@@

Do not log entire payloads when they contain sensitive information.

Protect against:

- oversized lines;
- deeply nested objects;
- unexpectedly large strings;
- malformed encoding;
- maliciously large arrays inside a line.

Resource limits are part of extraction security.

---

# 63. Testing Strategy

Test the format and the operational behavior.

Minimum cases:

1. one valid record;
2. many valid records;
3. blank line;
4. whitespace-only line;
5. malformed JSON;
6. wrong root type;
7. missing required field;
8. wrong field type;
9. Unicode;
10. CRLF;
11. oversized record;
12. compressed file;
13. duplicate replay;
14. database rollback;
15. checkpoint recovery;
16. source mutation;
17. expected-count mismatch.

---

# 64. Example Unit Tests

@@@
import json
import pytest


def test_jsonl_extracts_records(tmp_path):
    path = tmp_path / "payments.jsonl"

    path.write_text(
        "\n".join(
            [
                json.dumps({
                    "payment_id": "P001",
                    "amount": 100,
                    "currency": "EUR",
                }),
                json.dumps({
                    "payment_id": "P002",
                    "amount": 200,
                    "currency": "USD",
                }),
            ]
        ),
        encoding="utf-8",
    )

    with path.open(
        "r",
        encoding="utf-8",
    ) as source:
        rows = list(
            parse_jsonl(
                source,
                ["payment_id", "amount", "currency"],
            )
        )

    assert [row[0] for row in rows] == [1, 2]
    assert rows[0][1]["payment_id"] == "P001"


def test_malformed_line_fails(tmp_path):
    path = tmp_path / "broken.jsonl"

    path.write_text(
        '{"payment_id":"P001"}\n'
        '{"payment_id":\n',
        encoding="utf-8",
    )

    with path.open(
        "r",
        encoding="utf-8",
    ) as source:
        with pytest.raises(JsonlExtractionError):
            list(
                parse_jsonl(
                    source,
                    ["payment_id"],
                )
            )


def test_missing_field_fails(tmp_path):
    path = tmp_path / "payments.jsonl"

    path.write_text(
        '{"payment_id":"P001"}\n',
        encoding="utf-8",
    )

    with path.open(
        "r",
        encoding="utf-8",
    ) as source:
        with pytest.raises(JsonlExtractionError):
            list(
                parse_jsonl(
                    source,
                    ["payment_id", "currency"],
                )
            )
@@@

---

# 65. Integration Testing

Unit tests validate parsing.

Integration tests validate the pipeline.

A useful integration test is:

@@@
JSONL file
   |
   v
extractor
   |
   v
PostgreSQL
   |
   v
staging assertions
@@@

Verify:

- row count;
- source hash;
- line number;
- payload;
- idempotent replay;
- transaction rollback;
- checkpoint state.

A parser can be correct while the ingestion transaction is wrong.

---

# 66. Property-Based Testing

Useful properties include:

- accepted lines always produce valid records;
- malformed JSON never becomes an accepted record;
- line numbers are monotonic;
- source hash remains stable;
- replay does not create duplicate technical identities;
- checkpoint never advances before durable commit;
- rejected records remain traceable.

Generate random valid and invalid records to test the parser against unexpected structures.

---

# 67. Performance Testing

Measure:

- records per second;
- bytes per second;
- CPU usage;
- memory usage;
- database throughput;
- batch latency;
- checkpoint frequency.

Test realistic distributions.

A million tiny records behaves differently from a million records containing large nested payloads.

Do not benchmark only an ideal fixture.

---

# 68. Production Tools You Should Know

## 68.1 Python json

Use for:

- parsing individual JSONL records;
- validation;
- serialization.

The core operation is:

@@@
json.loads(line)
@@@

The key advantage is that parsing is limited to one record at a time.

---

## 68.2 gzip

Use for compressed JSONL files.

Example:

@@@
with gzip.open(
    "payments.jsonl.gz",
    "rt",
    encoding="utf-8",
) as source:
    for line in source:
        record = json.loads(line)
@@@

Use streaming decompression instead of creating unnecessary temporary files.

---

## 68.3 PostgreSQL JSONB

Useful for staging the original parsed record while retaining structured query capability.

Example:

@@@
payload JSONB NOT NULL
@@@

Keep the raw structure available until downstream modeling has safely extracted the required business fields.

---

# 69. Common Mistakes

## Mistake 1 — Treating JSONL as one JSON array

JSONL is a sequence of independent JSON values.

**Fix:** process one line at a time.

## Mistake 2 — Loading the entire file

This defeats the primary benefit of JSONL.

**Fix:** stream records.

## Mistake 3 — Silently skipping malformed lines

Data disappears without explanation.

**Fix:** quarantine or fail according to an explicit policy.

## Mistake 4 — Using line number as the business key

Line position can change when the producer regenerates the file.

**Fix:** separate technical identity from business identity.

## Mistake 5 — Advancing checkpoints before commits

A crash can cause permanent data loss.

**Fix:** commit data first, checkpoint second.

## Mistake 6 — No maximum line size

One pathological record can consume excessive memory.

**Fix:** enforce a source-appropriate limit.

## Mistake 7 — Ignoring source mutation

The file can change while it is being processed.

**Fix:** verify source stability before extraction.

## Mistake 8 — Assuming valid JSON means valid data

Syntax correctness is not schema correctness.

**Fix:** validate the record contract.

## Mistake 9 — One transaction per line

This can create severe database overhead.

**Fix:** use bounded batches.

## Mistake 10 — Claiming exactly-once because of checkpoints

Checkpoints do not eliminate retries.

**Fix:** combine durable checkpoints with idempotent staging.

---

# 70. Production Implementation Sequence

### Step 1

Investigate the producer contract.

### Step 2

Confirm that the format is actually JSONL.

### Step 3

Define encoding and line rules.

### Step 4

Define blank-line behavior.

### Step 5

Define required fields and types.

### Step 6

Define maximum line size.

### Step 7

Define strict or tolerant failure semantics.

### Step 8

Define source identity.

### Step 9

Define technical record identity.

### Step 10

Define business record identity.

### Step 11

Implement streaming parsing.

### Step 12

Implement record validation.

### Step 13

Implement lineage capture.

### Step 14

Implement idempotent staging.

### Step 15

Implement bounded database batches.

### Step 16

Implement durable checkpoints.

### Step 17

Add metrics and structured logs.

### Step 18

Test malformed lines.

### Step 19

Test duplicate replay.

### Step 20

Test database rollback and recovery.

### Step 21

Test large records and realistic volume.

### Step 22

Test source immutability.

### Step 23

Run a production failure drill.

### Step 24

Document recovery.

---

# 71. Production Checklist

- [ ] Format confirmed as JSONL/NDJSON.
- [ ] One-record-per-line contract documented.
- [ ] Encoding defined.
- [ ] Blank-line policy defined.
- [ ] Root type defined.
- [ ] Required fields defined.
- [ ] Field types validated.
- [ ] Maximum line size defined.
- [ ] Source identity deterministic.
- [ ] Technical record identity deterministic.
- [ ] Business identity defined where applicable.
- [ ] Streaming parser used.
- [ ] Raw lineage preserved.
- [ ] Idempotent staging implemented.
- [ ] Batch size bounded.
- [ ] Transaction policy documented.
- [ ] Checkpoint policy implemented where needed.
- [ ] Checkpoint advances only after durable commit.
- [ ] Compression handled safely.
- [ ] Source immutability verified.
- [ ] Schema evolution policy documented.
- [ ] Malformed records are visible.
- [ ] Sensitive payloads are not logged.
- [ ] Metrics exist.
- [ ] Integration tests exist.
- [ ] Replay test passes.
- [ ] Recovery drill passes.

---

# 72. Definition of Done

E56 is complete when you can independently:

1. explain why JSONL differs from ordinary JSON;
2. stream a JSONL file without loading it into memory;
3. parse one JSON value per line;
4. validate the expected root type;
5. distinguish missing fields from null;
6. detect malformed records;
7. choose strict versus tolerant processing;
8. quarantine invalid lines when permitted;
9. preserve line-level lineage;
10. create deterministic technical identity;
11. separate technical and business identity;
12. stage records idempotently;
13. process bounded database batches;
14. implement durable checkpoints;
15. resume safely after database failure;
16. protect against oversized records;
17. process compressed JSONL;
18. validate source immutability;
19. reconcile expected and observed counts;
20. test replay and failure recovery.

---

# 73. What You Learned

JSONL changes the extraction unit from a whole document to an individual line.

That gives the pipeline a natural streaming model:

@@@
SOURCE
  |
  v
READ LINE
  |
  v
PARSE
  |
  v
VALIDATE
  |
  +----> QUARANTINE
  |
  v
STAGE
  |
  v
COMMIT
  |
  v
CHECKPOINT
@@@

The critical production rules are:

- never silently discard malformed lines;
- never advance a checkpoint before durable data commit;
- never assume line number is a business identifier;
- never confuse valid JSON with valid business data;
- never remove idempotency just because the format is stream-friendly;
- never assume a source file is immutable without verification.

JSONL is simple at the syntax level, but production ingestion still requires identity, validation, transaction control, observability, and recovery.

---

# 74. Next Recipe

**E57 — XML Extraction — ETL Application**

The next recipe focuses on XML documents, namespaces, attributes, repeated elements, XPath-style extraction, streaming XML parsing, and safe ETL staging.
