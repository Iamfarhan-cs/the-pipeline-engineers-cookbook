# E55 — JSON Extraction — ETL Application

## 1. Problem Recognition

JSON is one of the most common interchange formats in modern data platforms. APIs, application exports, payment systems, configuration services, event producers, and object-storage feeds frequently deliver JSON.

A JSON extraction step has to answer more than “can Python parse this file?”

It must establish:

- where the records are inside the document;
- what the root structure is;
- which fields are required;
- how nested objects and arrays are handled;
- how missing and null values behave;
- how malformed documents are rejected;
- how large documents are processed without uncontrolled memory growth;
- how source identity and lineage are preserved;
- how duplicate extraction is prevented;
- how schema changes are detected;
- how failures are observable and recoverable.

The production lifecycle is:

```
VALIDATED JSON
      |
      v
PARSE
      |
      v
LOCATE RECORDS
      |
      v
NORMALIZE
      |
      v
STAGE
      |
      v
DOWNSTREAM TRANSFORM
```

The extraction layer should not silently invent business meaning. Its primary job is to safely turn a valid source document into deterministic records while preserving lineage.

---

# 2. Concept and Reasoning

## 2.1 JSON is a document format

JSON represents one complete document.

A document may be:

```
{
  "customer_id": "C001",
  "name": "Alice"
}
```

or:

```
[
  {
    "customer_id": "C001"
  },
  {
    "customer_id": "C002"
  }
]
```

or a nested envelope:

```
{
  "metadata": {
    "source": "payments"
  },
  "data": {
    "records": [
      {
        "payment_id": "P001"
      },
      {
        "payment_id": "P002"
      }
    ]
  }
}
```

The extractor must know the expected document shape.

Do not assume that every JSON file contains an array at the root.

---

# 3. JSON vs JSONL

JSON and JSONL solve different ingestion problems.

| Format | Typical structure | Natural processing model |
|---|---|---|
| JSON | One document | Document parsing |
| JSON array | One document containing many records | Parse document, iterate records |
| Nested JSON | Envelope containing records | Parse document, locate record path |
| JSONL | One JSON value per line | Record streaming |

For E55, the focus is JSON documents.

E56 covers JSONL extraction separately.

A JSON array containing 5 million objects is still one JSON document. Calling it JSONL does not make it streamable.

---

# 4. Extraction Contract

Before writing extraction code, define the contract.

Example:

```
{
  "source": "payments",
  "format": "json",
  "root_path": ["data", "records"],
  "record_key": "payment_id",
  "encoding": "utf-8",
  "required_fields": [
    "payment_id",
    "amount",
    "currency"
  ]
}
```

The contract should answer:

1. What file or object is expected?
2. What encoding is expected?
3. What root type is expected?
4. Where are records located?
5. What identifies one record?
6. Which fields are required?
7. What happens when a field is missing?
8. What happens when a field is null?
9. What happens when the document is malformed?
10. How is the source version identified?

The contract turns an ambiguous parser into a deterministic extraction component.

---

# 5. Problem: Loading the Entire Document

Python's standard JSON parser is straightforward.

```
import json

with open("payments.json", "r", encoding="utf-8") as f:
    document = json.load(f)
```

This is appropriate for bounded files.

It is dangerous when the source document can become very large.

If a 2 GB JSON document is loaded into memory, the in-memory representation can be substantially larger than the file itself.

The production question is therefore:

> Is the document small enough to parse as one object?

If yes, standard json is usually sufficient.

If no, use an incremental parser and process the relevant objects as they arrive.

---

# 6. Basic JSON Extraction

## 6.1 Parse a JSON object

```
import json

with open("customer.json", "r", encoding="utf-8") as f:
    document = json.load(f)

if not isinstance(document, dict):
    raise ValueError("Expected JSON object")

customer_id = document["customer_id"]
name = document.get("name")
```

The root type should be validated before accessing fields.

---

## 6.2 Parse a JSON array

```
import json

with open("customers.json", "r", encoding="utf-8") as f:
    document = json.load(f)

if not isinstance(document, list):
    raise ValueError("Expected JSON array")

for record in document:
    if not isinstance(record, dict):
        raise ValueError("Expected each array element to be an object")

    customer_id = record.get("customer_id")
    print(customer_id)
```

Do not blindly iterate over a value just because Python allows iteration.

A string is iterable.

A dictionary is iterable over its keys.

Validate the expected type first.

---

# 7. Nested Record Extraction

Many production feeds use an envelope.

Example:

```
{
  "batch_id": "B20260926",
  "generated_at": "2026-09-26T08:00:00Z",
  "data": {
    "records": [
      {
        "payment_id": "P001",
        "amount": 125.50,
        "currency": "EUR"
      },
      {
        "payment_id": "P002",
        "amount": 80.00,
        "currency": "USD"
      }
    ]
  }
}
```

The extraction path is:

```
data -> records
```

Implementation:

```
import json

with open("payments.json", "r", encoding="utf-8") as f:
    document = json.load(f)

records = document["data"]["records"]

if not isinstance(records, list):
    raise ValueError("data.records must be an array")

for record in records:
    if not isinstance(record, dict):
        raise ValueError("Each record must be an object")

    print(record["payment_id"])
```

The extractor should validate every structural assumption.

---

# 8. Safe Nested Access

Direct indexing is appropriate when a field is mandatory.

```
payment_id = record["payment_id"]
```

Use get when absence is allowed.

```
description = record.get("description")
```

Do not use get for mandatory fields and then silently continue.

Bad:

```
payment_id = record.get("payment_id")
```

This converts a structural error into a potentially corrupted record.

Better:

```
if "payment_id" not in record:
    raise ValueError("Missing required field: payment_id")

payment_id = record["payment_id"]
```

---

# 9. Missing vs Null

These are different states.

Missing:

```
{}
```

Null:

```
{
  "description": null
}
```

A production extractor should preserve that distinction when the downstream contract cares about it.

Example:

```
if "description" not in record:
    description_status = "missing"
elif record["description"] is None:
    description_status = "null"
else:
    description_status = "present"
```

Do not automatically convert all missing values and explicit nulls into the same business state.

---

# 10. Nested Objects

Example:

```
{
  "payment_id": "P001",
  "customer": {
    "id": "C001",
    "country": "DE"
  }
}
```

Extraction:

```
customer = record.get("customer")

if customer is not None and not isinstance(customer, dict):
    raise ValueError("customer must be an object")

customer_id = customer.get("id") if customer else None
country = customer.get("country") if customer else None
```

If customer is mandatory, enforce it explicitly instead.

---

# 11. Nested Arrays

Example:

```
{
  "payment_id": "P001",
  "items": [
    {
      "item_id": "I001",
      "amount": 50
    },
    {
      "item_id": "I002",
      "amount": 75
    }
  ]
}
```

Extraction:

```
items = record.get("items", [])

if not isinstance(items, list):
    raise ValueError("items must be an array")

for item in items:
    if not isinstance(item, dict):
        raise ValueError("Each item must be an object")

    item_id = item.get("item_id")
    amount = item.get("amount")
```

Do not flatten nested arrays accidentally.

First decide whether the downstream model requires:

- one payment record with an embedded array;
- one payment row per item;
- a parent table and child table;
- or raw JSON preservation.

That is a data-model decision, not merely a parsing decision.

---

# 12. JSON Path Extraction

When many feeds use nested structures, centralize path traversal.

```
def get_path(document, path):
    current = document

    for key in path:
        if not isinstance(current, dict):
            raise ValueError(
                f"Expected object while reading path: {path}"
            )

        if key not in current:
            raise KeyError(f"Missing JSON path: {path}")

        current = current[key]

    return current
```

Usage:

```
records = get_path(
    document,
    ["data", "records"]
)

if not isinstance(records, list):
    raise ValueError("Record path must point to an array")
```

This prevents every extractor from implementing its own nested traversal rules.

---

# 13. Schema Validation

Parsing proves that the document is syntactically valid JSON.

It does not prove that the document follows your application contract.

Example:

```
{
  "payment_id": "P001",
  "amount": "125.50",
  "currency": "EUR"
}
```

The JSON is valid.

But perhaps amount must be numeric.

A separate schema-validation layer can enforce that.

For controlled contracts, validation should happen after parsing and before staging.

The conceptual sequence is:

```
bytes
  |
  v
JSON syntax validation
  |
  v
document structure validation
  |
  v
record validation
  |
  v
staging
```

---

# 14. Type Validation

Do not assume JSON types match business types.

Example:

```
def require_string(record, field):
    value = record.get(field)

    if not isinstance(value, str):
        raise ValueError(
            f"{field} must be a string"
        )

    if not value.strip():
        raise ValueError(
            f"{field} must not be blank"
        )

    return value
```

Numeric fields need similar validation.

Be especially careful with financial amounts.

Do not casually convert financial values through binary floating-point if exact decimal semantics are required.

For example, use Decimal where appropriate:

```
from decimal import Decimal

amount = Decimal(str(record["amount"]))
```

The extraction layer should preserve source meaning rather than introduce rounding errors.

---

# 15. Dates and Timestamps

JSON does not have a native date or timestamp type.

These values are strings.

Example:

```
{
  "occurred_at": "2026-09-26T08:30:00Z"
}
```

Parse them deliberately.

```
from datetime import datetime

occurred_at = datetime.fromisoformat(
    record["occurred_at"].replace("Z", "+00:00")
)
```

Do not silently accept multiple timestamp formats unless the source contract explicitly allows them.

Normalize timezone handling consistently.

---

# 16. Encoding

UTF-8 should normally be the default contract.

```
with open(
    "customers.json",
    "r",
    encoding="utf-8"
) as f:
    document = json.load(f)
```

If a source can deliver another encoding, make it an explicit contract.

Do not solve encoding problems by blindly ignoring decoding errors.

This can turn corrupted source data into apparently valid but incorrect text.

---

# 17. BOM Handling

Some producers emit a UTF-8 BOM.

When this is a known source behavior, use an explicit encoding strategy.

```
with open(
    "customers.json",
    "r",
    encoding="utf-8-sig"
) as f:
    document = json.load(f)
```

Do not automatically use utf-8-sig for every source.

The encoding behavior belongs in the source contract.

---

# 18. Malformed JSON

Malformed JSON should fail extraction.

Example:

```
{
  "payment_id": "P001",
  "amount": 100,
}
```

The trailing comma makes this invalid JSON.

Catch parsing errors at the extraction boundary.

```
import json

try:
    with open("payments.json", "r", encoding="utf-8") as f:
        document = json.load(f)
except json.JSONDecodeError as exc:
    raise ValueError(
        f"Malformed JSON at line {exc.lineno}, "
        f"column {exc.colno}"
    ) from exc
```

Record the source identity and failure details.

Do not publish partially parsed records as successful extraction.

---

# 19. Empty Documents

An empty file is not a valid JSON document.

```
open("empty.json", "w").close()
```

It should fail parsing.

Do not classify an empty file as “zero records” unless the source contract explicitly defines that behavior.

A valid empty array is different:

```
[]
```

That means the document is valid JSON and contains zero records.

---

# 20. Root Type Validation

Always distinguish these cases:

```
{}
[]
"hello"
42
null
```

All are valid JSON.

Only some may satisfy your source contract.

Example:

```
document = json.load(f)

if not isinstance(document, dict):
    raise ValueError(
        "Expected root JSON object"
    )
```

Root validation prevents downstream assumptions from becoming runtime surprises.

---

# 21. Duplicate JSON Object Keys

JSON object member names are expected to be unique by normal data contracts.

However, parsers may handle duplicate keys by retaining only one value.

Example:

```
{
  "payment_id": "P001",
  "payment_id": "P999"
}
```

Do not rely on the parser to preserve both values.

If duplicate-key detection is a contractual requirement, implement explicit parsing or upstream validation that detects duplicate keys before normal object construction.

This is an edge case worth testing for high-integrity feeds.

---

# 22. Preserve Raw Source

Extraction should not destroy evidence.

For every extracted record, preserve enough lineage to answer:

> Which source document produced this record?

Typical metadata:

```
source_file
source_uri
source_system
source_batch_id
source_object_version
source_hash
extracted_at
record_number
```

Example staged record:

```
{
  "payment_id": "P001",
  "amount": 125.50,
  "_source_file": "payments_20260926.json",
  "_source_hash": "sha256:...",
  "_record_number": 1
}
```

Keep business fields separate from technical lineage fields.

---

# 23. Deterministic Record Identity

A record needs a stable identity.

Prefer a source-provided business identifier:

```
record_id = record["payment_id"]
```

If no business key exists, use a deterministic identity based on source identity and record position or a stable content hash, according to the source contract.

Do not generate a random UUID during extraction and then use it as the only deduplication key.

Random identifiers make replay behavior harder to control.

---

# 24. Idempotent Extraction

A rerun of the same source document should not silently create duplicates.

A practical staging identity can be:

```
(source_hash, record_number)
```

or, where appropriate:

```
(source_system, source_document_id, record_id)
```

Database example:

```
CREATE TABLE json_staging (
    source_hash TEXT NOT NULL,
    record_number INTEGER NOT NULL,
    record_id TEXT,
    payload JSONB NOT NULL,
    extracted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (source_hash, record_number)
);
```

Then insert idempotently:

```
INSERT INTO json_staging (
    source_hash,
    record_number,
    record_id,
    payload
)
VALUES (%s, %s, %s, %s)
ON CONFLICT (source_hash, record_number)
DO NOTHING;
```

The identity must reflect the source contract.

---

# 25. Batch Inserts

Do not issue one database transaction per record for large files.

Collect records into bounded batches.

```
BATCH_SIZE = 1000

batch = []

for record_number, record in enumerate(records, start=1):
    batch.append(
        (
            source_hash,
            record_number,
            record.get("payment_id"),
            json.dumps(record),
        )
    )

    if len(batch) >= BATCH_SIZE:
        insert_batch(batch)
        batch.clear()

if batch:
    insert_batch(batch)
```

Bounded batches provide a compromise between:

- database round trips;
- transaction size;
- memory usage;
- failure recovery.

---

# 26. Transaction Boundaries

Decide what should happen if record 7,501 fails.

Two common policies are:

### Whole-document transaction

All records commit or none commit.

Useful when the document is a logical atomic unit.

### Batch-level transaction

Successful batches remain committed while the failed batch is retried or quarantined.

Useful for large feeds where rerunning the entire document is expensive.

The correct policy comes from business requirements.

Do not accidentally create partial success semantics.

---

# 27. A Practical JSON Extractor

The following implementation provides a small but production-oriented foundation.

```
import hashlib
import json
from pathlib import Path
from typing import Any


class JsonExtractionError(Exception):
    pass


def sha256_file(path: Path) -> str:
    digest = hashlib.sha256()

    with path.open("rb") as source:
        for chunk in iter(lambda: source.read(1024 * 1024), b""):
            digest.update(chunk)

    return digest.hexdigest()


def get_path(document: Any, path: list[str]) -> Any:
    current = document

    for key in path:
        if not isinstance(current, dict):
            raise JsonExtractionError(
                f"Expected object while reading path {path}"
            )

        if key not in current:
            raise JsonExtractionError(
                f"Missing JSON path: {path}"
            )

        current = current[key]

    return current


def extract_json(
    path: Path,
    record_path: list[str],
    required_fields: list[str],
) -> list[dict[str, Any]]:
    source_hash = sha256_file(path)

    try:
        with path.open(
            "r",
            encoding="utf-8",
        ) as source:
            document = json.load(source)
    except json.JSONDecodeError as exc:
        raise JsonExtractionError(
            f"Invalid JSON in {path}: "
            f"line={exc.lineno}, column={exc.colno}"
        ) from exc

    records = get_path(document, record_path)

    if not isinstance(records, list):
        raise JsonExtractionError(
            f"Record path {record_path} must point to an array"
        )

    output = []

    for record_number, record in enumerate(
        records,
        start=1,
    ):
        if not isinstance(record, dict):
            raise JsonExtractionError(
                f"Record {record_number} is not an object"
            )

        for field in required_fields:
            if field not in record:
                raise JsonExtractionError(
                    f"Record {record_number} missing "
                    f"required field {field}"
                )

        output.append(
            {
                "source_hash": source_hash,
                "record_number": record_number,
                "record": record,
            }
        )

    return output
```

Example:

```
records = extract_json(
    Path("payments.json"),
    record_path=["data", "records"],
    required_fields=[
        "payment_id",
        "amount",
        "currency",
    ],
)

for item in records:
    print(
        item["record_number"],
        item["record"]["payment_id"],
    )
```

This is intentionally explicit.

The extraction contract remains visible rather than hidden inside a generic framework.

---

# 28. Avoid Returning Huge Lists

The previous function is easy to understand but has an important limitation.

It returns every extracted record in a Python list.

That means memory grows with the document.

For production-scale files, prefer an iterator when the parser supports incremental extraction.

Conceptually:

```
for record in extract_records(...):
    stage(record)
```

rather than:

```
records = extract_all(...)
stage(records)
```

The second approach creates unnecessary memory pressure.

---

# 29. Streaming Large JSON Documents

For large JSON arrays or deeply nested documents, an incremental parser such as ijson can process matching objects without constructing the complete document in memory.

Example:

```
import ijson

with open(
    "large-payments.json",
    "rb",
) as source:
    for record in ijson.items(
        source,
        "data.records.item",
    ):
        process_record(record)
```

The important idea is the path:

```
data.records.item
```

It identifies each array element.

This is different from JSONL.

The source remains one JSON document.

---

# 30. Streaming with Object Storage

Production JSON commonly arrives in object storage.

The extraction layer should not assume that the source is a local file.

The same logical flow can be:

```
object storage
      |
      v
stream object
      |
      v
incremental JSON parser
      |
      v
record validation
      |
      v
staging
```

Keep object retrieval separate from parsing.

That separation makes local testing easier and allows the same parser to work with different storage systems.

---

# 31. Reader Interface

A useful design is to give extraction code a file-like object.

```
def extract_records(source):
    for record in ijson.items(
        source,
        "data.records.item",
    ):
        yield record
```

Then the caller can provide:

- a local file;
- an object-storage stream;
- a test fixture;
- a decompressed stream.

This reduces coupling between storage and parsing.

---

# 32. Compression

JSON is frequently compressed.

Examples:

```
payments.json.gz
payments.json.zst
```

The correct sequence is:

```
compressed bytes
      |
      v
decompression stream
      |
      v
JSON parser
      |
      v
records
```

Do not decompress a multi-gigabyte file into another equally large temporary file unless the operational design requires it.

Prefer streaming decompression where practical.

---

# 33. JSON Schema Evolution

A source may add fields:

```
{
  "payment_id": "P001",
  "amount": 100,
  "currency": "EUR",
  "channel": "mobile"
}
```

Adding optional fields is often backward-compatible.

Removing a required field is not.

Changing a field type can also be breaking:

```
"amount": 100
```

becoming:

```
"amount": "100"
```

The extractor should detect contract-breaking changes rather than silently adapting to them.

---

# 34. Unknown Fields

Do not automatically reject every unknown field.

That can make additive source evolution unnecessarily fragile.

A better policy may be:

- required fields must exist;
- known fields must have expected types;
- unknown fields are preserved in raw payload;
- contract-breaking changes fail;
- schema changes are observable.

The exact policy depends on downstream consumers.

---

# 35. Record-Level vs Document-Level Failure

Suppose a document contains 100,000 records and one record is malformed.

There are two different policies.

### Strict document policy

One invalid record invalidates the entire document.

Use when the document is an atomic financial or regulatory delivery.

### Quarantine policy

Valid records continue and invalid records are quarantined.

Use only when the business contract explicitly permits partial processing.

Do not invent partial-success behavior in the extractor.

---

# 36. Quarantine

A useful quarantine record contains:

```
source_hash
source_uri
record_number
error_code
error_message
raw_record
detected_at
extractor_version
```

Example:

```
{
  "source_hash": "abc123",
  "record_number": 501,
  "error_code": "MISSING_REQUIRED_FIELD",
  "error_message": "currency is required"
}
```

Never discard a rejected record without preserving enough context for investigation.

---

# 37. Resource Limits

JSON extraction is an input boundary.

Treat source data as untrusted input.

Define limits where appropriate:

- maximum document size;
- maximum nesting depth;
- maximum record count;
- maximum string length;
- maximum array length;
- maximum processing time.

The exact limits should reflect the source contract.

Resource limits protect the pipeline from accidental or malicious pathological input.

---

# 38. Security

Do not execute JSON values as code.

Never do this:

```
eval(record["expression"])
```

JSON is data.

Also avoid logging entire records when they may contain:

- credentials;
- account numbers;
- identity data;
- tokens;
- personal information;
- financial information.

Log structural metadata and safe identifiers instead.

---

# 39. ETL Staging Model

A practical JSON staging table can retain both normalized metadata and the original payload.

```
CREATE TABLE json_staging (
    source_hash TEXT NOT NULL,
    source_uri TEXT NOT NULL,
    record_number INTEGER NOT NULL,
    record_id TEXT,
    payload JSONB NOT NULL,
    extracted_at TIMESTAMPTZ NOT NULL,
    extractor_version TEXT NOT NULL,
    PRIMARY KEY (source_hash, record_number)
);
```

This gives downstream transformations access to the original parsed structure.

Do not force every JSON field into columns during extraction.

That belongs to a later modeling decision unless the staging contract explicitly requires a normalized shape.

---

# 40. Example PostgreSQL Insert

Using psycopg:

```
import json
import psycopg


INSERT_SQL = """
INSERT INTO json_staging (
    source_hash,
    source_uri,
    record_number,
    record_id,
    payload,
    extracted_at,
    extractor_version
)
VALUES (
    %s, %s, %s, %s, %s, now(), %s
)
ON CONFLICT (source_hash, record_number)
DO NOTHING
"""


def insert_record(
    connection,
    source_hash,
    source_uri,
    record_number,
    record,
    extractor_version,
):
    with connection.cursor() as cursor:
        cursor.execute(
            INSERT_SQL,
            (
                source_hash,
                source_uri,
                record_number,
                record.get("payment_id"),
                json.dumps(record),
                extractor_version,
            ),
        )
```

For large workloads, replace one-row-at-a-time insertion with bounded batch operations.

---

# 41. Extraction State

A production pipeline should track extraction state separately from the data.

Example:

```
CREATE TABLE json_extraction_run (
    run_id BIGSERIAL PRIMARY KEY,
    source_uri TEXT NOT NULL,
    source_hash TEXT NOT NULL,
    extractor_version TEXT NOT NULL,
    status TEXT NOT NULL,
    records_seen BIGINT NOT NULL DEFAULT 0,
    records_staged BIGINT NOT NULL DEFAULT 0,
    records_rejected BIGINT NOT NULL DEFAULT 0,
    started_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ
);
```

Useful states:

```
DISCOVERED
VALIDATING
EXTRACTING
COMPLETED
FAILED
QUARANTINED
```

The state should reflect actual processing progress.

---

# 42. Checkpointing

Large JSON extraction can be expensive.

If the parser supports deterministic record positions, checkpoint progress at controlled boundaries.

Example:

```
source_hash = ...
last_committed_record = 500000
```

On retry, the extractor can determine whether records before that point are already staged.

Idempotent database writes are often simpler and safer than complex parser-level checkpoint state.

Use checkpoints when restart cost justifies the additional complexity.

---

# 43. Exactly-Once Expectations

Do not casually promise exactly-once processing.

A more realistic architecture is:

```
at-least-once extraction
        +
idempotent staging
        +
deterministic record identity
        =
effectively-once downstream result
```

The extractor may execute more than once.

The important requirement is that retries do not create incorrect duplicate business state.

---

# 44. Testing Strategy

A JSON extractor needs more than a happy-path test.

Test at least:

1. valid object;
2. valid array;
3. nested record path;
4. empty array;
5. malformed JSON;
6. empty file;
7. wrong root type;
8. missing required field;
9. explicit null;
10. wrong field type;
11. nested object;
12. nested array;
13. duplicate extraction;
14. Unicode content;
15. BOM when supported;
16. large document behavior;
17. schema evolution;
18. quarantine behavior if supported.

---

# 45. Example Tests

```
import json
import pytest


def write_json(path, value):
    path.write_text(
        json.dumps(value),
        encoding="utf-8",
    )


def test_extracts_nested_records(tmp_path):
    path = tmp_path / "payments.json"

    write_json(
        path,
        {
            "data": {
                "records": [
                    {
                        "payment_id": "P001",
                        "amount": 100,
                        "currency": "EUR",
                    }
                ]
            }
        },
    )

    result = extract_json(
        path,
        ["data", "records"],
        ["payment_id", "amount", "currency"],
    )

    assert len(result) == 1
    assert result[0]["record"]["payment_id"] == "P001"


def test_rejects_malformed_json(tmp_path):
    path = tmp_path / "broken.json"
    path.write_text(
        '{"payment_id": "P001",',
        encoding="utf-8",
    )

    with pytest.raises(JsonExtractionError):
        extract_json(
            path,
            [],
            [],
        )


def test_rejects_missing_required_field(tmp_path):
    path = tmp_path / "payments.json"

    write_json(
        path,
        {
            "data": {
                "records": [
                    {
                        "payment_id": "P001",
                    }
                ]
            }
        },
    )

    with pytest.raises(JsonExtractionError):
        extract_json(
            path,
            ["data", "records"],
            ["payment_id", "currency"],
        )
```

---

# 46. Property-Based Testing

For mature extractors, generate many valid and invalid JSON structures.

Useful properties include:

- valid documents never produce non-dictionary records when the contract requires objects;
- record numbers are deterministic;
- rerunning the same source does not change record identity;
- malformed JSON never produces successful extraction;
- missing required fields are never silently converted to null.

Property-based testing is especially useful for nested structures because the number of possible shapes grows quickly.

---

# 47. Intentional Failure Drill 1 — Malformed JSON

Break the source:

```
{
  "data": [
    {"payment_id": "P001"}
  ]
```

Expected behavior:

1. parser raises JSON decoding error;
2. extraction run becomes FAILED;
3. zero partial-success records are reported unless explicitly supported;
4. source identity remains available;
5. alert/metric is emitted;
6. operator can replace or correct the source and retry.

Failure should be visible.

---

# 48. Intentional Failure Drill 2 — Wrong Record Path

Change:

```
data.records
```

to:

```
data.payments
```

Expected behavior:

- extractor detects missing path;
- extraction fails;
- error identifies the configured path;
- no silent empty dataset is produced.

An empty result caused by a wrong path is one of the most dangerous extraction bugs because it can look like a successful zero-record delivery.

---

# 49. Intentional Failure Drill 3 — Wrong Type

Change:

```
"amount": 100
```

to:

```
"amount": {
  "value": 100
}
```

Expected behavior:

- record validation fails;
- error identifies amount type;
- source and record identity remain available;
- record is quarantined or document fails according to policy.

---

# 50. Intentional Failure Drill 4 — Duplicate Replay

Run the same source twice.

Expected behavior:

- source hash is identical;
- record identities are identical;
- staging row count does not double;
- second run is reported as a replay or idempotent no-op.

This test validates the most important property of retry-safe extraction.

---

# 51. Observability

Measure extraction behavior.

Useful metrics:

```
json_documents_discovered_total
json_documents_succeeded_total
json_documents_failed_total
json_records_seen_total
json_records_staged_total
json_records_rejected_total
json_extraction_duration_seconds
json_document_bytes_total
json_parse_failures_total
json_schema_failures_total
json_duplicate_records_total
```

Useful dimensions:

- source system;
- dataset;
- environment;
- extractor version;
- failure category.

Avoid high-cardinality dimensions such as arbitrary payload values.

---

# 52. Structured Logs

A useful success event:

```
{
  "event": "json_extraction_completed",
  "source": "payments",
  "source_hash": "abc123",
  "records_seen": 10000,
  "records_staged": 10000,
  "records_rejected": 0,
  "duration_ms": 4200,
  "extractor_version": "1.3.0"
}
```

A useful failure event:

```
{
  "event": "json_extraction_failed",
  "source": "payments",
  "source_hash": "abc123",
  "error_code": "MALFORMED_JSON",
  "records_staged": 0,
  "extractor_version": "1.3.0"
}
```

Do not put the complete payload into logs.

---

# 53. Data Quality Checks

After extraction, validate basic invariants.

Examples:

```
records_seen >= records_staged + records_rejected
```

and:

```
record_number starts at 1
record_number has no unexpected duplicates
required identifiers are present
source_hash is consistent for the run
```

For atomic document processing:

```
records_rejected = 0
```

may be required for successful completion.

---

# 54. Recovery Runbook

When JSON extraction fails:

### Step 1 — Identify the source

Find:

- source URI;
- source hash;
- source system;
- batch or delivery ID.

### Step 2 — Classify the failure

Determine whether it is:

- malformed JSON;
- missing path;
- wrong root type;
- missing required field;
- type mismatch;
- resource limit;
- storage failure;
- database failure.

### Step 3 — Preserve the source

Do not overwrite the original failed artifact.

### Step 4 — Decide whether the producer must correct it

A source-contract violation normally belongs with the producer.

### Step 5 — Retry safely

Use the same source identity and idempotent staging.

### Step 6 — Verify counts

Compare:

- records discovered;
- records extracted;
- records staged;
- records rejected.

### Step 7 — Close the run

Only mark the extraction completed after all required checks pass.

---

# 55. Production Tools You Should Know

## 55.1 Python json

Use for:

- normal JSON parsing;
- bounded documents;
- simple object and array extraction.

Core functions:

```
json.load()
json.loads()
json.dump()
json.dumps()
```

Know its memory implications before using it on large documents.

---

## 55.2 ijson

Use when JSON documents are too large for whole-document parsing.

Key idea:

```
for record in ijson.items(source, "data.records.item"):
    process_record(record)
```

It allows incremental extraction from large JSON structures.

---

## 55.3 PostgreSQL JSONB

Use JSONB when the staging layer needs to preserve structured JSON while still allowing database-side inspection and indexing.

Example:

```
CREATE TABLE json_staging (
    id BIGSERIAL PRIMARY KEY,
    payload JSONB NOT NULL
);
```

Do not treat JSONB as a replacement for good data modeling. Use it deliberately at the raw/staging boundary.

---

# 56. Common Mistakes

## Mistake 1 — Assuming every JSON file is an array

JSON can have any valid root type.

**Fix:** validate the root type.

## Mistake 2 — Using get for required fields

Missing data becomes indistinguishable from optional data.

**Fix:** enforce required fields explicitly.

## Mistake 3 — Loading multi-gigabyte JSON with json.load

Memory usage can become excessive.

**Fix:** use incremental parsing for large documents.

## Mistake 4 — Treating JSON and JSONL as the same

Their processing models differ.

**Fix:** design E55 and E56 separately.

## Mistake 5 — Returning a giant list

Memory grows with the entire document.

**Fix:** yield records or stage them incrementally.

## Mistake 6 — Flattening nested arrays without a model

The extractor can change the meaning of the source.

**Fix:** define the downstream relational model first.

## Mistake 7 — Losing the raw source

Without lineage, replay and debugging become difficult.

**Fix:** retain source identity and raw payload where appropriate.

## Mistake 8 — Generating random IDs for every extraction

Retries create different identities.

**Fix:** use deterministic source and record identity.

## Mistake 9 — Silently accepting wrong paths

A wrong path can produce a false zero-record success.

**Fix:** missing required paths must fail.

## Mistake 10 — Logging complete JSON records

This can expose sensitive information.

**Fix:** log metadata and safe identifiers only.

---

# 57. Production Implementation Sequence

Use this sequence when implementing a new JSON feed.

### Step 1

Investigate the producer contract.

### Step 2

Capture a representative valid document.

### Step 3

Capture malformed and edge-case documents.

### Step 4

Define the expected root type.

### Step 5

Define the record path.

### Step 6

Define required fields and types.

### Step 7

Define missing/null semantics.

### Step 8

Define source identity and hashing.

### Step 9

Implement parsing.

### Step 10

Implement structural validation.

### Step 11

Implement record validation.

### Step 12

Add lineage fields.

### Step 13

Add idempotent staging.

### Step 14

Add bounded batching.

### Step 15

Add extraction-state tracking.

### Step 16

Add metrics and structured logs.

### Step 17

Add malformed-input tests.

### Step 18

Add replay tests.

### Step 19

Test large-document behavior.

### Step 20

Run an intentional failure drill.

### Step 21

Document the recovery procedure.

### Step 22

Promote only after the full contract is verified.

---

# 58. Production Checklist

Before production:

- [ ] JSON contract is documented.
- [ ] Root type is validated.
- [ ] Record path is explicit.
- [ ] Required fields are enforced.
- [ ] Missing and null semantics are defined.
- [ ] Field types are validated.
- [ ] Encoding is explicit.
- [ ] Malformed JSON is rejected.
- [ ] Source identity is deterministic.
- [ ] Record identity is deterministic.
- [ ] Raw lineage is preserved.
- [ ] Idempotent staging exists.
- [ ] Batch size is bounded.
- [ ] Transaction policy is documented.
- [ ] Large-file strategy is defined.
- [ ] Compression behavior is defined.
- [ ] Schema evolution policy is defined.
- [ ] Resource limits are defined.
- [ ] Sensitive payloads are not logged.
- [ ] Tests cover malformed input.
- [ ] Tests cover replay.
- [ ] Observability exists.
- [ ] Recovery runbook exists.

---

# 59. Definition of Done

E55 is complete when you can independently:

1. identify a JSON document's structure;
2. distinguish JSON from JSONL;
3. validate the root type;
4. locate nested records;
5. safely handle missing and null fields;
6. validate field types;
7. parse malformed-input failures correctly;
8. preserve source lineage;
9. create deterministic record identity;
10. stage records idempotently;
11. process bounded documents with the standard library;
12. stream large JSON documents with an incremental parser;
13. handle compressed input;
14. detect contract-breaking schema changes;
15. design document-level or record-level failure policy;
16. test replay behavior;
17. instrument extraction metrics;
18. recover a failed extraction safely.

---

# 60. What You Learned

JSON extraction is not simply:

```
json.load(file)
```

A production extractor is a controlled boundary between an external document and the internal data platform.

The core pattern is:

```
SOURCE
  |
  v
IDENTIFY
  |
  v
PARSE
  |
  v
VALIDATE
  |
  v
LOCATE RECORDS
  |
  v
NORMALIZE
  |
  v
PRESERVE LINEAGE
  |
  v
IDEMPOTENT STAGE
  |
  v
OBSERVE
  |
  v
RECOVER
```

The most important distinction is between syntactic validity and pipeline correctness.

A document can be valid JSON and still be unusable because:

- the structure changed;
- the record path disappeared;
- a required field is missing;
- a field changed type;
- the document is too large;
- duplicate replay is not controlled;
- lineage is missing.

Once you understand those failure modes, JSON extraction becomes an engineering discipline rather than a parsing exercise.

---

# 61. Next Recipe

**E56 — JSONL Extraction**

The next recipe focuses on one-JSON-value-per-line processing, record streaming, malformed-line isolation, line-level checkpoints, and high-volume ingestion.
