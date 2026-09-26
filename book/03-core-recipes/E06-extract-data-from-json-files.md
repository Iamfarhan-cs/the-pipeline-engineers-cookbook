# E06 — Extract Data from JSON Files

JSON is widely used for API exports, application dumps, event files, configuration-driven feeds, and object-storage data.

Unlike CSV, JSON can represent nested objects and arrays. That flexibility creates extraction problems that are easy to miss:

- the root structure may change
- fields may be nested at different paths
- arrays can contain many records
- null and missing fields are different
- one malformed document can invalidate an entire file
- large JSON documents can exhaust memory
- schema drift can happen without a parser error

The goal is:

> Extract JSON only after proving that its structure matches the expected contract, then process it with bounded memory and explicit recovery behavior.

## 1. Problem Recognition

Common production failures include:

- assuming every JSON file has the same root type
- loading a multi-gigabyte JSON document into memory
- confusing missing fields with null fields
- treating an object as a record when it is actually a container
- failing because one nested field changed type
- accepting unexpected arrays or objects
- silently ignoring unknown fields
- processing the same file twice
- reading a partially written file
- failing the entire dataset because one record is malformed
- checkpointing without binding progress to the source file identity

Do not equate:

    valid JSON syntax

with:

    valid pipeline data

## 2. JSON Extraction Architecture

    FILE ARRIVES
        ↓
    DISCOVER
        ↓
    VERIFY COMPLETENESS
        ↓
    IDENTIFY FILE
        ↓
    PARSE JSON
        ↓
    VALIDATE ROOT STRUCTURE
        ↓
    EXTRACT RECORDS
        ↓
    VALIDATE RECORDS
        ↓
    PERSIST
        ↓
    CHECKPOINT
        ↓
    MARK COMPLETE

Each stage answers a different question.

## 3. JSON Is a Data Format, Not a Schema

JSON syntax defines how data is represented.

It does not automatically define:

- which fields are required
- which fields are integers
- which fields are dates
- which values are allowed
- whether a field may be null
- whether an array contains records

Therefore the extractor needs a separate data contract.

## 4. Common JSON Root Structures

A JSON file can have an object root:

    {
        "payments": [...]
    }

or an array root:

    [
        {"id": 1},
        {"id": 2}
    ]

These structures require different extraction logic.

Do not assume the root is always an array.

## 5. Root Validation

Before extracting records, validate the expected root.

For example:

    expected root: object
    expected collection: payments

Then validate:

    data
      ↓
    payments
      ↓
    array
      ↓
    records

If the contract says the root must contain payments as an array, a scalar or object at that location should fail validation rather than being silently accepted.

## 6. Basic JSON Extraction

A small JSON file can be loaded directly:

    import json

    with open(
        'payments.json',
        'r',
        encoding='utf-8'
    ) as file:
        document = json.load(file)

    records = document['payments']

This is appropriate only when the document is small enough for bounded memory usage.

## 7. Large JSON Documents

A large JSON document can make this dangerous:

    document = json.load(file)

The complete document is materialized in memory.

For large data, prefer formats or layouts that support streaming or chunked processing.

Common approaches include:

- JSON Lines
- streaming JSON parsers
- producer-side partitioning
- smaller JSON files

The preferred solution depends on the source contract.

## 8. JSON Lines

JSON Lines, often called JSONL or NDJSON, stores one JSON value per line.

Example:

    {"id":1,"amount":100}
    {"id":2,"amount":200}
    {"id":3,"amount":300}

This structure is naturally streamable.

Conceptually:

    READ LINE
       ↓
    PARSE ONE JSON VALUE
       ↓
    VALIDATE
       ↓
    PERSIST
       ↓
    NEXT LINE

JSONL is often easier to process incrementally than one enormous JSON array.

## 9. JSONL Is Not Automatically Valid

Each line should contain one valid JSON value according to the contract.

Problems can include:

- blank lines
- malformed JSON
- arrays instead of objects
- unexpected scalar values
- truncated final records

Define whether blank lines are allowed and how malformed records are handled.

## 10. Python JSONL Reader

A simple streaming implementation:

    import json

    with open(
        'payments.jsonl',
        'r',
        encoding='utf-8'
    ) as file:

        for line_number, line in enumerate(file, start=1):
            if not line.strip():
                continue

            record = json.loads(line)
            validate(record)
            persist(record)

This keeps memory bounded by the current record plus downstream buffers.

## 11. Nested Objects

JSON commonly contains nested objects:

    {
        "id": 1001,
        "customer": {
            "id": 50,
            "country": "PK"
        }
    }

The extractor must know which nested paths are part of the contract.

Example path:

    customer.id

Do not assume missing nested objects are equivalent to missing leaf values.

## 12. Missing vs Null

These are different:

    {}

and:

    {"description": null}

The first means the field is absent.
The second means the field is present with a null value.

The pipeline contract should define whether each is acceptable.

## 13. Arrays Inside Records

A JSON record may contain arrays:

    {
        "id": 1001,
        "items": [
            {"sku": "A", "quantity": 2},
            {"sku": "B", "quantity": 1}
        ]
    }

Decide whether the target model should:

- preserve the nested array
- flatten it into child records
- aggregate it
- extract it into a separate dataset

Do not flatten automatically without understanding the relationship.

## 14. Cardinality Problems

Flattening nested arrays can multiply rows.

Example:

    1 payment
       ×
    100 line items
       =
    100 output records

With multiple nested arrays, accidental Cartesian multiplication is possible.

Model the relationship before flattening.

## 15. JSON Schema Contract

A JSON extraction contract should define:

    path
    type
    required?
    nullable?
    array/object?
    allowed values

Example:

    id              integer       required
    amount          number        required
    currency        string        required
    customer.id     integer       required
    description     string        optional

The parser handles syntax. The schema contract handles meaning.

## 16. Type Validation

JSON supports basic value types:

- string
- number
- boolean
- null
- object
- array

Do not assume a field has the expected type because the key exists.

Bad source:

    {"amount": "100"}

Expected:

    {"amount": 100}

Whether coercion is allowed must be explicit.

## 17. Type Drift

A field can change between files:

    amount: 100

then:

    amount: "100"

then:

    amount: null

A permissive parser may accept all three while downstream logic behaves differently.

Detect type drift at extraction boundaries.

## 18. Unknown Fields

New fields are not automatically errors.

For example:

    {
        "id": 1,
        "amount": 100,
        "new_source_field": "x"
    }

Possible policies:

- allow unknown fields
- warn on unknown fields
- reject unknown fields

The correct policy depends on the source contract and evolution strategy.

## 19. Schema Evolution

JSON schema changes may include:

- new field
- removed field
- renamed field
- type change
- nested structure change
- array-to-object change
- object-to-array change

Protect the extractor with:

- schema validation
- compatibility tests
- versioned contracts
- controlled deployments
- observability for drift

## 20. File Completeness

A JSON file can be syntactically invalid because it was read before the producer finished writing it.

Prefer an explicit publication contract:

    WRITE TEMP
       ↓
    CLOSE / VERIFY
       ↓
    ATOMIC RENAME
       ↓
    PUBLISHED JSON

If no completion marker exists, a stable-size check is only a heuristic.

## 21. File Identity

Track file identity using information such as:

    source
    filename
    file_size
    checksum
    discovered_at

A checksum can be calculated with SHA-256.

File identity protects against:

- duplicate delivery
- accidental replacement
- incorrect checkpoint reuse

## 22. Duplicate Files

The same JSON file may arrive twice.

Examples:

- transfer retry
- scheduler retry
- manual resend
- object-storage copy

Maintain a processed-file registry or equivalent durable state.

Do not rely only on the filename.

## 23. Duplicate Records

Different files can contain the same logical record.

Example:

    export_001.json → id 1001
    export_002.json → id 1001

File-level deduplication does not solve record-level duplication.

Use stable source or business identifiers where available.

## 24. Checkpointing

Checkpointing is easier with JSONL because records have a natural sequence.

Possible state:

    file_id
    checksum
    line_number
    batch_number
    last_source_id

For a streaming parser over a large JSON document, checkpointing may require parser-specific offsets or producer-level partitioning.

Always bind progress to immutable file identity.

## 25. Batch Persistence

Do not necessarily persist every record independently.

For larger files:

    PARSE RECORDS
         ↓
    BUILD BATCH
         ↓
    VALIDATE BATCH
         ↓
    PERSIST
         ↓
    CHECKPOINT

Batching reduces persistence overhead while keeping memory bounded.

## 26. Malformed Records

A malformed JSON document can be handled at different levels.

For a single JSON document:

    parse failure
       ↓
    file failure

For JSONL:

    malformed line
       ↓
    reject/quarantine record
       ↓
    continue or fail file

The policy must be explicit.

Never silently discard malformed records.

## 27. Error Thresholds

An ingestion contract can define:

    max_bad_records = 0

or:

    max_bad_record_ratio = 0.01

When the threshold is exceeded:

    STOP
      ↓
    QUARANTINE
      ↓
    ALERT

The threshold should be agreed as part of the source contract.

## 28. JSON Number Precision

JSON numbers can create precision concerns when they represent financial or very large numeric values.

Do not blindly convert every number to a binary floating-point type.

For monetary values, a decimal representation may be more appropriate.

The extraction contract should define numeric handling before implementation.

## 29. Date and Time Values

JSON has no native date/time type.

Dates commonly arrive as strings:

    "2026-09-26T12:30:00Z"

The extractor should define:

- accepted format
- timezone expectations
- nullability
- invalid-value behavior

Do not silently interpret ambiguous local timestamps.

## 30. Testing

Test at least:

1. Valid object-root JSON.
2. Valid array-root JSON.
3. Empty object.
4. Empty array.
5. Missing expected collection.
6. Wrong root type.
7. Missing required field.
8. Explicit null.
9. Wrong field type.
10. Unknown field.
11. Nested object.
12. Nested array.
13. Duplicate record.
14. Duplicate file.
15. Malformed JSON.
16. Malformed JSONL record.
17. Blank JSONL line.
18. Large JSONL file.
19. File mutation after discovery.
20. Persistence failure.
21. Checkpoint restart.
22. Schema change.
23. Numeric precision edge case.
24. Invalid timestamp.

## 31. Observability

Useful metrics:

    json_files_discovered_total
    json_files_processed_total
    json_files_failed_total
    json_files_quarantined_total
    json_files_duplicated_total
    json_records_read_total
    json_records_persisted_total
    json_records_rejected_total
    json_parse_errors_total
    json_schema_errors_total
    json_type_errors_total
    json_schema_drift_total
    json_processing_duration_seconds
    json_bytes_processed_total

Useful log fields:

    extraction_run_id
    file_id
    filename
    checksum
    line_number
    record_number
    batch_number
    record_count
    error_type
    schema_version

Do not log entire JSON records when they contain sensitive data.

## 32. Intentional Failure

### Failure drill 1 — Incomplete JSON file

Stop writing a JSON document before its final structure is complete.

Verify that publication controls prevent extraction of the incomplete file.

### Failure drill 2 — Wrong root type

Replace an expected array with an object.

Verify root validation fails.

### Failure drill 3 — Type drift

Change amount from a number to a string.

Verify the schema contract detects the change or applies the explicitly defined coercion policy.

### Failure drill 4 — Malformed JSONL record

Corrupt one JSONL line.

Verify the configured reject/quarantine policy is applied.

### Failure drill 5 — Duplicate file

Deliver the same file twice.

Verify file-level idempotency.

### Failure drill 6 — Persistence failure

Fail persistence halfway through a large JSONL file.

Restart and verify checkpoint and idempotency behavior.

### Failure drill 7 — File mutation

Change the source file after checkpointing.

Verify the old checkpoint cannot be applied blindly to the changed artifact.

## 33. Recovery

When JSON extraction fails:

1. Identify the file and extraction run.
2. Verify filename, size, and checksum.
3. Confirm publication completeness.
4. Identify whether the failure is syntax, schema, type, or persistence related.
5. Inspect rejected records if applicable.
6. Verify the checkpoint.
7. Confirm the source artifact has not changed.
8. Resume only from a valid checkpoint.
9. Reconcile expected and persisted records.
10. Mark the file complete only after verification.

Do not resume a changed artifact using state from the old artifact.

## 34. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Python json** | Standard-library JSON parser; understand load, loads, encoding, and object/array handling. |
| **ijson** | Streaming JSON parser useful when a large JSON document cannot safely be materialized in memory. |
| **Pydantic** | Useful for explicit JSON data contracts, type validation, and structured validation. |

> These are reference tools for production vocabulary. The underlying JSON extraction mechanics should still be understood independently.

## 35. Production Runbook

### JSON file cannot be parsed

Check:

1. File completeness.
2. Encoding.
3. JSON syntax.
4. Truncation.
5. Producer version.

### JSON structure changed

Check:

1. Root type.
2. Required paths.
3. Field types.
4. Schema version.
5. Recent source deployment.

### Memory usage is too high

Check:

1. json.load usage.
2. File size.
3. JSONL availability.
4. Streaming parser options.
5. Batch size.

### Duplicate records appear

Check:

1. File identity.
2. Processed-file state.
3. Source identifiers.
4. Checkpoint restart behavior.
5. Upstream duplicate delivery.

### What not to do

Do not:

- assume valid JSON means valid business data
- load enormous documents blindly into memory
- confuse missing fields with null fields
- silently coerce type changes
- silently discard malformed JSONL records
- reuse checkpoints against changed files
- rely only on filenames for identity
- flatten nested arrays without understanding cardinality

## 36. Common Mistakes

### Mistake 1 — Treating JSON as schema-free

JSON syntax does not provide your pipeline's business contract.

### Mistake 2 — Loading everything into memory

Large JSON documents can exhaust workers.

### Mistake 3 — Ignoring nested cardinality

Flattening arrays can multiply records unexpectedly.

### Mistake 4 — Treating null and missing as identical

They can have different business meanings.

### Mistake 5 — Silent type coercion

A field changing from number to string can hide upstream schema drift.

### Mistake 6 — No file identity

Duplicate or replaced files can invalidate checkpointing.

### Mistake 7 — No malformed-record policy

Undefined failure behavior creates silent data loss or unnecessary pipeline failures.

## 37. Definition of Done

You are done when you can:

- distinguish JSON syntax from a data contract
- handle object and array roots
- validate JSON root structure
- extract nested objects
- reason about nested arrays and cardinality
- distinguish missing from null
- validate JSON field types
- detect type drift
- define unknown-field policy
- process JSONL incrementally
- stream large JSON safely
- batch JSON records
- handle malformed records explicitly
- identify duplicate files
- identify duplicate records
- use checksums for file identity
- checkpoint JSON extraction safely
- reason about numeric precision
- handle timestamp strings correctly
- test schema changes
- intentionally break JSON extraction
- recover from failed extraction
- operate JSON extraction with a production runbook

## 38. What You Learned

The central principle is:

> Valid JSON syntax is only the beginning; production extraction requires a validated data contract and controlled processing.

A production JSON pipeline follows:

    DISCOVER
       ↓
    VERIFY COMPLETENESS
       ↓
    IDENTIFY FILE
       ↓
    PARSE
       ↓
    VALIDATE STRUCTURE
       ↓
    VALIDATE RECORDS
       ↓
    PERSIST
       ↓
    CHECKPOINT
       ↓
    MARK COMPLETE

The key question is:

> What proves that this JSON artifact contains records in the exact structure, types, and completeness required by my pipeline?

That question drives schema contracts, streaming, validation, file identity, checkpointing, observability, testing, and recovery.