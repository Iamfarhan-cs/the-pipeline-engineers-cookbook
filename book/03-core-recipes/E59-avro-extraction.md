# E59 — Avro Extraction

## 1. Problem Recognition

Avro is a compact, row-oriented binary serialization format built around an explicit schema. It is common in event pipelines, Kafka systems, service-to-service data exchange, object storage, and batch ETL.

Production extraction must understand:

- physical Avro encoding;
- writer and reader schemas;
- schema resolution;
- unions and defaults;
- logical types;
- Object Container File blocks;
- compression;
- schema evolution;
- source and record identity;
- streaming;
- idempotent staging;
- reconciliation and recovery.

The production flow is:

```
DISCOVER SOURCE
      |
      v
IDENTIFY AVRO ENCODING
      |
      v
READ / RESOLVE SCHEMA
      |
      v
STREAM RECORDS
      |
      v
VALIDATE
      |
      v
IDENTIFY + STAGE
      |
      v
CHECKPOINT
      |
      v
RECONCILE + OBSERVE
```

Core rule:

> Treat the schema as part of the data contract, not as optional metadata.

---

## 2. Avro vs Other Formats

| Format | Model | Typical strength |
|---|---|---|
| CSV | Text rows | Simple interchange |
| JSONL | Text records | Flexible event data |
| Avro | Binary rows + schema | Streaming and schema-governed records |
| Parquet | Binary columns | Analytical scans |

Avro is not binary JSON. Its schema participates in decoding and evolution.

```
Avro    -> row-oriented binary serialization
Parquet -> columnar analytical storage
JSONL   -> text record stream
```

---

## 3. Identify the Physical Encoding

“Avro data” can refer to different representations.

### Object Container File

Usually contains:

- magic header;
- metadata;
- writer schema;
- sync marker;
- data blocks;
- optional compression.

### Schemaless Avro

Raw binary data is decoded with a schema supplied separately.

### Registry-Framed Messages

A broker message may carry a schema identifier or another framing mechanism. The consumer retrieves the schema from an external registry.

These are different decoding boundaries. Do not use an Object Container reader for a raw Avro message.

---

## 4. Avro Schema Fundamentals

A schema describes the record structure:

```
{
  "type": "record",
  "name": "Payment",
  "namespace": "com.example.payments",
  "fields": [
    {"name": "payment_id", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string"}
  ]
}
```

Important concepts:

- record name and namespace;
- fields;
- primitive and complex types;
- unions;
- defaults;
- aliases;
- logical types.

The decoder needs this contract to interpret the binary values correctly.

---

## 5. Primitive and Complex Types

Primitive types include null, boolean, int, long, float, double, bytes, and string.

Complex types include:

- record;
- array;
- map;
- enum;
- fixed;
- union.

Do not automatically map every numeric type to the same database type.

```
Avro int    -> often INTEGER
Avro long   -> often BIGINT
Avro double -> often DOUBLE PRECISION
```

The target mapping must match the business and database contract.

---

## 6. Nullable Fields and Unions

A nullable field is commonly represented as:

```
{
  "name": "reference",
  "type": ["null", "string"],
  "default": null
}
```

Do not silently collapse:

```
null
empty string
missing according to schema resolution
actual string
```

These states can have different downstream meanings.

---

## 7. Logical Types

Logical types attach semantic meaning to primitive representations.

Examples include decimal, date, time-millis, time-micros, timestamp-millis, timestamp-micros, and duration.

Timestamp example:

```
{
  "type": "long",
  "logicalType": "timestamp-millis"
}
```

Decimal example:

```
{
  "type": "bytes",
  "logicalType": "decimal",
  "precision": 18,
  "scale": 2
}
```

Preserve logical-type meaning. A timestamp is not merely an arbitrary long, and a decimal is not merely arbitrary bytes.

---

## 8. Writer Schema vs Reader Schema

**Writer schema** is the schema used when the producer serialized the data.

**Reader schema** is the schema the consumer wants to use when decoding it.

```
PRODUCER
   |
   | writer schema
   v
encoded bytes
   |
   | schema resolution
   v
CONSUMER
   |
   | reader schema
   v
decoded record
```

For an Object Container File, the writer schema is normally stored in the file header.

---

## 9. Schema Resolution

Suppose version 1 contains payment_id and amount. Version 2 adds currency.

A reader can often read old records using the new field when an appropriate default exists:

```
{
  "name": "currency",
  "type": "string",
  "default": "EUR"
}
```

Schema compatibility is not the same as entire pipeline compatibility. Also check staging tables, transformation code, database mappings, quality rules, and downstream consumers.

---

## 10. Aliases and Schema Fingerprints

A rename can be represented with an alias:

```
{
  "name": "payment_reference",
  "type": "string",
  "aliases": ["reference"]
}
```

A schema fingerprint provides a compact identity for a schema.

Conceptually:

```
schema
  |
  v
canonical representation
  |
  v
fingerprint
```

Use fingerprints to detect schema changes, group files by schema, and audit which schema interpretation was used. Do not use filenames as schema identity.

---

## 11. Object Container Files

Conceptually:

```
+-------------------------+
| magic + metadata        |
| writer schema           |
| sync marker             |
+-------------------------+
| data block              |
| records                 |
| optional compression    |
+-------------------------+
| sync marker             |
+-------------------------+
| next data block         |
+-------------------------+
```

Blocks make large files processable without loading every record into memory. Sync markers separate blocks. Compression changes CPU and I/O trade-offs.

Use a tested Avro library rather than manually decoding container bytes.

---

## 12. Python Implementation

For Python ETL, fastavro is a practical choice:

```
pip install fastavro
```

Basic Object Container extraction:

```
from pathlib import Path
from fastavro import reader

def extract_avro(path: Path):
    with path.open("rb") as file:
        for record in reader(file):
            yield record

for record in extract_avro(Path("payments.avro")):
    print(record)
```

The generator is important. Avoid converting the complete reader to a list for large files.

---

## 13. Inspect the Writer Schema

Capture schema information at the extraction boundary:

```
from pathlib import Path
from fastavro import reader

def inspect_avro(path: Path) -> dict:
    with path.open("rb") as file:
        avro_reader = reader(file)

        return {
            "metadata": dict(avro_reader.metadata),
            "writer_schema": avro_reader.writer_schema,
        }
```

Record the actual metadata exposed by the source. Do not invent metadata fields.

---

## 14. Source Lineage and Identity

Every decoded record should remain traceable to its source.

A useful source identity can contain:

```
source URI or object key
source version where available
file size
content hash
acquisition timestamp
```

A record should preferably use a stable business identifier such as payment_id.

Record number is useful for diagnostics but is not necessarily stable business identity.

A deterministic record identity can be based on:

```
source identity + business identifier
```

This makes replay and idempotency safer.

---

## 15. Validate After Decoding

Successful Avro decoding does not mean the business record is valid.

Example:

```
def validate_payment(record: dict) -> None:
    required = {"payment_id", "amount", "currency"}
    missing = required - record.keys()

    if missing:
        raise ValueError(
            f"Missing required fields: {sorted(missing)}"
        )

    if not record["payment_id"]:
        raise ValueError("payment_id must not be empty")

    if record["amount"] < 0:
        raise ValueError("amount must not be negative")

    if len(record["currency"]) != 3:
        raise ValueError("currency must be a 3-character code")
```

Keep these failure classes separate:

```
Avro decoding failure
Business validation failure
Database failure
```

---

## 16. Staging in PostgreSQL

A simple staging boundary can preserve decoded records:

```
CREATE TABLE avro_record_staging (
    record_id TEXT PRIMARY KEY,
    source_file TEXT NOT NULL,
    source_record_number BIGINT NOT NULL,
    schema_fingerprint TEXT,
    payload JSONB NOT NULL,
    extracted_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Useful fields include deterministic record identity, source lineage, schema identity, payload, and extraction time.

---

## 17. Idempotent Staging

If the same file is processed twice, the pipeline must not blindly duplicate records.

Use a durable uniqueness boundary:

```
INSERT INTO avro_record_staging (
    record_id,
    source_file,
    source_record_number,
    schema_fingerprint,
    payload
)
VALUES (
    %(record_id)s,
    %(source_file)s,
    %(source_record_number)s,
    %(schema_fingerprint)s,
    %(payload)s
)
ON CONFLICT (record_id) DO NOTHING;
```

Idempotency should not depend only on an in-memory Python set.

---

## 18. Batch Writes

Avoid one database transaction per record.

Prefer bounded batches:

```
decode
  |
  v
validate
  |
  v
collect bounded batch
  |
  v
database transaction
  |
  v
commit
  |
  v
checkpoint
```

Tune batch size using measured memory, transaction duration, database throughput, and recovery cost.

---

## 19. Checkpointing

For large files, durable progress may be required.

A checkpoint can contain:

```
source_identity
records_processed
last_committed_position
status
updated_at
```

Checkpoint only after the corresponding records are durably committed.

Do not checkpoint only by filename and record number when filenames can be reused.

---

## 20. Raw Artifact Preservation

Where the architecture permits it, preserve the original Avro artifact in immutable raw storage.

```
source Avro
    |
    +---- immutable raw artifact
    |
    +---- decoded staging
              |
              v
          transformations
```

Raw retention allows replay after an extractor or transformation bug.

---

## 21. Object Storage Extraction

For object storage:

```
object storage
      |
      v
download / stream
      |
      v
Avro decoder
      |
      v
validation
      |
      v
staging
```

Capture bucket/container, object key, version where available, content length, checksum or ETag where appropriate, acquisition time, schema identity, and processing state.

---

## 22. File Completeness and Corruption

A valid header does not prove a complete delivery.

Before processing, verify the source contract:

- expected object exists;
- source is stable;
- content length is plausible;
- checksum matches when supplied;
- producer completion signal is satisfied when one exists.

Possible extraction failures include:

- truncated final block;
- corrupted bytes;
- invalid schema;
- unsupported codec;
- incomplete download.

Do not mark success because most records decoded.

---

## 23. File-Level vs Record-Level Failure

File-level failures can include invalid headers, unreadable schemas, corrupted containers, and unsupported codecs. They may invalidate the whole source.

Record-level failures can include invalid business values. These may be quarantined individually when the contract permits partial success.

Define the scope explicitly.

Never silently do this:

```
try:
    process(record)
except Exception:
    pass
```

Instead classify the failure and route it to the correct recovery path.

---

## 24. Schema Evolution Testing

Maintain serialized fixtures for important versions:

```
writer v1 -> reader v1
writer v1 -> reader v2
writer v2 -> reader v1
writer v2 -> reader v2
```

Classify each case as:

- compatible;
- incompatible;
- accepted with default;
- accepted with alias;
- rejected.

This catches schema changes before production data exposes them.

---

## 25. Schemaless Avro

Raw Avro binary without an Object Container File requires an explicit schema.

```
raw bytes
   +
writer schema
   |
   v
schemaless decoder
   |
   v
record
```

Example:

```
from io import BytesIO
from fastavro import schemaless_reader

def decode_message(payload: bytes, schema: dict):
    return schemaless_reader(
        BytesIO(payload),
        schema,
    )
```

Do not confuse this with Object Container File extraction.

---

## 26. Schema Registry

A schema registry manages schemas externally.

A common event flow is:

```
producer
   |
   +---- schema ---> registry
   |
   +---- message ---> broker
                         |
                         v
                      consumer
                         |
                         +---- schema lookup
                         |
                         v
                       decode
```

A registry can provide schema versions, compatibility checks, identifiers, and governance.

For a file whose writer schema is embedded in the Object Container header, a registry may not be necessary.

---

## 27. Observability

Track at least:

### Source metrics

- files discovered;
- accepted;
- rejected;
- bytes processed.

### Record metrics

- records decoded;
- staged;
- quarantined;
- duplicates.

### Performance metrics

- bytes per second;
- records per second;
- decode duration;
- database write duration;
- batch duration.

### Schema metrics

- writer schema fingerprint;
- reader schema/version;
- compatibility failures.

### Error metrics

- decode errors;
- validation failures;
- database errors;
- retries.

Structured completion logs should contain safe counts and identifiers, not complete sensitive payloads.

---

## 28. Reconciliation

Use an explicit accounting invariant:

```
decoded_records
=
staged_records
+
quarantined_records
+
explicitly_skipped_records
```

If the numbers do not reconcile, the pipeline has an accounting gap.

Reconciliation turns silent data loss into an observable failure.

---

## 29. Complete Extraction Skeleton

```
from hashlib import sha256
import json
from pathlib import Path
from typing import Iterator

from fastavro import reader

def schema_fingerprint(schema: dict) -> str:
    canonical = json.dumps(
        schema,
        sort_keys=True,
        separators=(",", ":"),
    ).encode("utf-8")

    return sha256(canonical).hexdigest()

def extract(path: Path) -> Iterator[dict]:
    with path.open("rb") as file:
        avro_reader = reader(file)

        fingerprint = schema_fingerprint(
            avro_reader.writer_schema
        )

        for number, record in enumerate(
            avro_reader,
            start=1,
        ):
            yield {
                "record_number": number,
                "schema_fingerprint": fingerprint,
                "record": record,
            }

def validate(record: dict) -> None:
    required = {"payment_id", "amount", "currency"}
    missing = required - record.keys()

    if missing:
        raise ValueError(
            f"Missing required fields: {sorted(missing)}"
        )

if __name__ == "__main__":
    for item in extract(Path("payments.avro")):
        validate(item["record"])
        print(item)
```

This is an extraction boundary, not a complete orchestration system. Add state, quarantine, database writes, and orchestration according to the repository architecture.

---

## 30. Security and Resource Limits

Binary serialization does not remove operational risk.

Protect the boundary against:

- unexpectedly large files;
- excessive record counts;
- deeply nested data;
- decompression resource exhaustion;
- malformed schemas;
- untrusted source locations;
- sensitive logging.

Useful operational limits include:

```
maximum file size
maximum processing duration
maximum retry count
maximum batch size
maximum records per source
```

Tune these using production measurements.

---

## 31. Testing Strategy

### Unit tests

Test:

- schema fingerprinting;
- required-field validation;
- nullable fields;
- logical types;
- deterministic identity.

### Fixture tests

Maintain Avro fixtures for:

- normal records;
- nested records;
- arrays/maps;
- nullable fields;
- logical types;
- compressed files;
- schema evolution.

### Integration tests

Test:

```
Avro
  |
  v
decoder
  |
  v
validation
  |
  v
PostgreSQL
```

### Failure tests

Break the header, schema, block, required field, database connection, duplicate identity, and checkpoint state.

---

## 32. Example Pytest

```
def test_extractor_reads_all_records(sample_avro):
    records = list(extract(sample_avro))

    assert len(records) == 3
    assert records[0]["record"]["payment_id"] == "p-001"

def test_missing_required_field():
    record = {
        "payment_id": "p-001",
        "amount": 10.0,
    }

    try:
        validate(record)
    except ValueError as exc:
        assert "currency" in str(exc)
    else:
        raise AssertionError("Expected validation failure")
```

Use actual serialized Avro fixtures to test decoding. A Python dictionary alone does not test Avro decoding.

---

## 33. Intentional Failure Drill

1. Copy a valid Avro file.
2. Truncate the copy.
3. Run extraction.
4. Confirm decoding fails.
5. Confirm the source becomes failed.
6. Confirm it is quarantined.
7. Confirm partial success is not published.
8. Restore the valid artifact.
9. Reprocess it.
10. Reconcile counts.

Expected lifecycle:

```
CORRUPT
  |
  v
FAILED
  |
  v
QUARANTINED
  |
  v
REPAIRED
  |
  v
REPROCESSED
  |
  v
RECONCILED
  |
  v
SUCCESS
```

---

## 34. Recovery Recipe

When extraction fails:

1. Identify the exact source using immutable source identity.
2. Classify acquisition, completeness, corruption, schema, decoding, validation, database, or infrastructure failure.
3. Check what was durably committed.
4. Do not manually delete random rows.
5. Reacquire the artifact if necessary.
6. Reprocess idempotently.
7. Reconcile all source records.
8. Record root cause and recovery result.

The same source should be safe to replay.

---

## 35. Production Tools You Should Know

### 1. fastavro

Use for Python Avro serialization/deserialization, streaming Object Container Files, schemaless decoding, and high-throughput ETL.

### 2. Apache Avro Python

Useful when you want the Apache Avro implementation and specification-aligned behavior.

### 3. Schema Registry

Use when the event architecture manages schemas externally.

The transferable skill is understanding:

```
schema
+
version
+
compatibility
+
producer
+
consumer
```

---

## 36. Common Mistakes

1. Treating Avro like JSON.
2. Ignoring the writer schema.
3. Treating reader schema as cosmetic.
4. Loading the entire file into memory.
5. Using record number as identity.
6. Assuming schema changes are automatically safe.
7. Ignoring logical types.
8. Confusing container files with raw messages.
9. Silently dropping bad records.
10. Marking success before complete processing.

---

## 37. Production Implementation Sequence

```
1. Identify the physical Avro encoding.
2. Obtain a representative fixture.
3. Inspect the writer schema.
4. Identify unions and logical types.
5. Determine whether a reader schema is required.
6. Define compatibility expectations.
7. Choose the Avro library.
8. Implement streaming extraction.
9. Preserve source lineage.
10. Validate decoded records.
11. Generate deterministic record identity.
12. Stage idempotently.
13. Add bounded database batches.
14. Add processing state.
15. Add checkpointing when required.
16. Add quarantine.
17. Add reconciliation.
18. Add metrics and structured logs.
19. Test schema evolution.
20. Test corruption and truncation.
21. Run the failure drill.
22. Document recovery.
23. Enable production scheduling.
```

---

## 38. Production Checklist

### Source

- [ ] Encoding identified.
- [ ] Object Container vs schemaless understood.
- [ ] Source identity defined.
- [ ] Completeness contract defined.
- [ ] Raw artifact retention decided.

### Schema

- [ ] Writer schema captured.
- [ ] Reader schema defined when required.
- [ ] Fingerprint/version recorded.
- [ ] Logical types understood.
- [ ] Union behavior tested.
- [ ] Evolution policy documented.

### Extraction

- [ ] Records are streamed.
- [ ] Large files do not require full memory.
- [ ] Required codecs are supported.
- [ ] Decode failures are classified.
- [ ] Complete-source success is enforced.

### Data quality

- [ ] Required fields validated.
- [ ] Business validation is separate from decoding.
- [ ] Invalid records are quarantined when appropriate.
- [ ] Counts are reconciled.

### Storage

- [ ] Raw source preserved when required.
- [ ] Staging is idempotent.
- [ ] Record identity is deterministic.
- [ ] Database writes are batched.
- [ ] Transactions are bounded.

### Operations

- [ ] Metrics exist.
- [ ] Structured logs exist.
- [ ] Schema changes are observable.
- [ ] Checkpoints are durable when needed.
- [ ] Recovery is documented.
- [ ] Failure drill passes.

---

## 39. Definition of Done

E59 is complete when you can independently:

- identify the Avro encoding used by a source;
- explain writer and reader schemas;
- explain schema resolution;
- handle nullable unions;
- understand logical types;
- inspect an Object Container File;
- stream records from a large file;
- preserve source and schema lineage;
- validate decoded records;
- generate deterministic record identity;
- stage records idempotently;
- handle schema evolution deliberately;
- quarantine invalid or corrupt sources;
- reconcile extraction counts;
- observe extraction performance and failures;
- test compatibility and corruption;
- recover a failed extraction safely.

---

## 40. What You Learned

Avro extraction is primarily a **schema, serialization, and record-streaming problem**.

The mental model is:

```
SOURCE
  |
  v
ENCODING
  |
  v
WRITER SCHEMA
  |
  v
SCHEMA RESOLUTION
  |
  v
STREAM RECORDS
  |
  v
VALIDATE
  |
  v
IDENTIFY
  |
  v
STAGE IDEMPOTENTLY
  |
  v
RECONCILE
  |
  v
OBSERVE + RECOVER
```

Production lesson:

> A decoded Avro record is trustworthy only when you know which source bytes produced it, which schema interpreted those bytes, and whether the complete source was processed successfully.

---

## 41. Next Recipe

**E60 — Compressed File Extraction**

The next recipe covers gzip, ZIP, and other compressed deliveries, including streaming decompression, archive safety, nested files, compression detection, and failure handling.
