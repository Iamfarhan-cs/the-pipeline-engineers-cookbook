# T01 — Raw Data to Staging

## 1. Problem Recognition

Raw data is evidence of what the source delivered.

Staging data is a controlled, queryable representation of that evidence that downstream transformation can safely consume.

Do not treat raw and staging as the same layer.

Typical flow:

    source
      |
      v
    raw artifact
      |
      v
    extraction
      |
      v
    staging
      |
      v
    curated transformation

The staging layer answers:

> What records did we successfully extract from the raw source, with enough metadata to trace them back and process them safely?

This recipe covers the transformation boundary between immutable raw evidence and structured staging data.

It builds on:

- Core Recipe 02 — Staging Layer
- Core Recipe 06 — Idempotency
- Core Recipe 07 — Deduplication
- Core Recipe 16 — Data Reconciliation
- Core Recipe 17 — Schema Changes
- E49–E65 — ETL file discovery, extraction, completeness, duplicate, missing, and late-file mechanisms

---

## 2. What You Are Building

Target architecture:

    Raw Artifact
        |
        v
    Extraction
        |
        v
    Raw Record
        |
        +-------------------+
        |                   |
        v                   v
    Validation          Lineage
        |                   |
        +---------+---------+
                  |
                  v
             Staging Table
                  |
                  v
          Transformation Jobs

The staging layer should be:

- structured;
- traceable;
- restartable;
- idempotent;
- queryable;
- isolated from curated business logic.

---

## 3. Learning Objectives

By the end of this recipe you should be able to:

1. Explain why staging exists.
2. Separate raw evidence from staging records.
3. Design a staging table.
4. Preserve source lineage.
5. Define a staging record identity.
6. Load records idempotently.
7. Handle malformed records.
8. Handle schema drift.
9. Preserve raw values when appropriate.
10. Track processing status.
11. Batch inserts safely.
12. Reprocess a raw artifact.
13. Reconcile raw records to staging records.
14. Test staging behavior.
15. Observe and operate staging in production.

---

## 4. Raw vs Staging

### Raw

Raw should preserve what the source actually delivered.

Examples:

    original JSON
    original CSV
    original XML
    original Parquet
    original Avro

Raw should normally be immutable.

### Staging

Staging converts that source evidence into records that the pipeline can query and process.

Example:

    raw JSON:
        {"customerId":"C1","amount":"42.50"}

    staging:
        customer_id = C1
        amount_text = 42.50
        source_file = file.json
        record_number = 17

Staging can normalize technical representation while preserving enough information to trace the record back to raw.

---

## 5. Staging Is Not the Curated Layer

Do not turn staging into the final business model.

Staging should usually represent:

    what was received

Curated models represent:

    what the business means

Example:

    staging:
        customer_id
        amount_text
        transaction_time_text
        source_file

    curated:
        customer_key
        amount_numeric
        transaction_timestamp
        reporting_currency

Business joins and derived metrics generally belong downstream.

---

## 6. Why Not Load Directly Into Curated Tables?

Direct loading creates several problems.

If extraction fails halfway through:

    curated table may be partially changed

If business transformation changes:

    raw source may need to be downloaded again

If a source field is malformed:

    the original evidence may be lost

If a producer sends a corrected file:

    it becomes difficult to determine what was originally received

Staging creates a controlled boundary.

---

## 7. The Staging Contract

A staging contract should define:

    source identity
    record identity
    raw lineage
    technical fields
    source fields
    validation state
    processing state
    ingestion timestamps

Example:

```
CREATE TABLE staging_transactions (
    staging_id BIGSERIAL PRIMARY KEY,
    source_file_id TEXT NOT NULL,
    source_record_id TEXT NOT NULL,
    customer_id TEXT,
    amount_text TEXT,
    transaction_time_text TEXT,
    record_payload JSONB,
    validation_status TEXT NOT NULL,
    processing_status TEXT NOT NULL,
    extracted_at TIMESTAMPTZ NOT NULL,
    staged_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (source_file_id, source_record_id)
);
```

The exact columns depend on the source.

---

## 8. Record Identity

Every staging record needs deterministic identity.

Possible identity:

    source_file_id + record_number

or:

    source_file_id + source_record_id

or:

    hash(raw_record)

Choose according to source semantics.

Do not use:

    random UUID

as the only business identity.

A random UUID changes every replay and cannot identify the same source record deterministically.

---

## 9. File Identity vs Record Identity

These are different.

File identity:

    transactions_20260926.csv

Record identity:

    row 481

A useful composite identity may be:

    file_id + record_number

For a source-provided transaction ID:

    file_id + transaction_id

may be more meaningful.

Do not assume a business ID is globally unique unless the source contract guarantees it.

---

## 10. Preserve Source Lineage

A staging record should answer:

> Which raw source produced this record?

Useful fields:

    source_file_id
    source_uri
    source_object_version
    source_sha256
    source_record_number
    source_record_id
    extracted_at

Lineage is not optional operational metadata.

It is how you debug bad records.

---

## 11. Record Number

For line-oriented formats, preserve the source position.

Example:

    record_number = 481

This helps operators locate the original record.

For JSONL:

    line_number

For CSV:

    physical row number

For XML:

    source element sequence

For Parquet:

    dataset/file + row-group/row position where practical

For formats without stable physical positions, use a deterministic record identity.

---

## 12. Raw Payload Preservation

Some staging designs preserve:

    raw_record JSONB

Others preserve only normalized columns.

A useful compromise is:

    typed staging columns
    +
    raw_record

The raw payload can support debugging and schema evolution.

But do not blindly duplicate massive payloads if storage cost is material.

Use the source retention policy.

---

## 13. Raw Payload vs Raw Artifact

These are not interchangeable.

Raw artifact:

    complete original file/object

Raw record:

    one extracted record representation

The artifact is authoritative evidence.

The record payload is a convenient debugging representation.

Do not delete the raw artifact merely because staging contains a payload copy.

---

## 14. Technical Metadata

Useful technical columns:

    source_file_id
    source_record_id
    source_record_number
    source_uri
    source_hash
    extraction_run_id
    staged_at
    extracted_at

Avoid mixing business columns with technical metadata without a clear naming convention.

---

## 15. Processing Metadata

Useful fields:

    validation_status
    processing_status
    rejection_reason
    first_seen_at
    last_seen_at
    processed_at

Example states:

    validation_status:
        VALID
        INVALID
        UNKNOWN

    processing_status:
        NEW
        READY
        PROCESSED
        QUARANTINED
        FAILED

Keep validation state and processing state separate.

---

## 16. Staging State Machine

A practical lifecycle:

    EXTRACTED
       |
       v
    VALIDATING
       |
       +-------> INVALID
       |
       v
     READY
       |
       v
    PROCESSED

Failure path:

    READY
       |
       v
    FAILED
       |
       v
    RETRY
       |
       v
    PROCESSED

The exact states depend on the pipeline.

The important principle is that staging records have observable lifecycle state.

---

## 17. Batch Loading

Do not insert one row and commit for every record.

Bad pattern:

```
for record in records:
    insert_record(record)
    connection.commit()
```

This creates excessive transaction overhead.

Prefer bounded batches:

```
BATCH_SIZE = 1000

for batch in batches(records, BATCH_SIZE):
    insert_batch(batch)
    connection.commit()
```

Choose batch size based on:

- row size;
- database capacity;
- transaction duration;
- network latency;
- recovery requirements.

---

## 18. Idempotent Staging

Reprocessing the same raw artifact must not create duplicate staging records.

Use a deterministic uniqueness constraint:

```
CREATE UNIQUE INDEX uq_staging_transaction
ON staging_transactions (
    source_file_id,
    source_record_id
);
```

Then use an idempotent insert strategy.

---

## 19. PostgreSQL Upsert

Example:

```
INSERT INTO staging_transactions (
    source_file_id,
    source_record_id,
    customer_id,
    amount_text,
    transaction_time_text,
    record_payload,
    validation_status,
    processing_status,
    extracted_at
)
VALUES (
    %s, %s, %s, %s, %s, %s, %s, %s, %s
)
ON CONFLICT (
    source_file_id,
    source_record_id
)
DO NOTHING;
```

This makes replay safe when identical source records are encountered again.

---

## 20. When DO NOTHING Is Wrong

Do not always use:

    ON CONFLICT DO NOTHING

A corrected source record may require an update.

Example:

    source record C1
    amount = 100

Later:

    source record C1
    amount = 125

If the source contract allows replacement, you may need:

```
ON CONFLICT (source_file_id, source_record_id)
DO UPDATE SET
    amount_text = EXCLUDED.amount_text,
    record_payload = EXCLUDED.record_payload,
    last_seen_at = now();
```

The replay policy must define whether a record is immutable or replaceable.

---

## 21. Immutable vs Mutable Staging

### Immutable

Every source version creates a new record version.

Useful for:

- auditability;
- event history;
- corrections;
- replay.

### Mutable

The staging row represents the current source version.

Useful when:

- source files are authoritative snapshots;
- corrections should replace previous staging state.

Do not choose randomly.

The source contract determines the correct model.

---

## 22. Versioned Staging

For corrections, add:

    source_version

or:

    observation_id

Example:

```
CREATE TABLE staging_transaction_version (
    staging_version_id BIGSERIAL PRIMARY KEY,
    source_file_id TEXT NOT NULL,
    source_record_id TEXT NOT NULL,
    source_hash TEXT NOT NULL,
    record_payload JSONB NOT NULL,
    observed_at TIMESTAMPTZ NOT NULL,

    UNIQUE (
        source_file_id,
        source_record_id,
        source_hash
    )
);
```

This preserves multiple source observations.

---

## 23. Validation Before Staging

Do not assume extraction success means data is valid.

Example:

```
def validate_record(record):
    if not record.get("customer_id"):
        return False, "MISSING_CUSTOMER_ID"

    if not record.get("amount"):
        return False, "MISSING_AMOUNT"

    return True, None
```

The validation policy should be explicit.

Technical extraction validity and business validity are different concerns.

---

## 24. Structural vs Business Validation

Structural validation:

    is the JSON valid?
    are required fields present?
    is the value type correct?

Business validation:

    is amount positive?
    does currency exist?
    does customer exist?

T01 focuses on establishing staging.

Complex business validation belongs in later transformation/data-quality recipes.

---

## 25. What If a Record Is Invalid?

Do not necessarily abort the entire file.

Possible strategies:

    strict:
        one invalid record fails the batch/file

    tolerant:
        valid records continue
        invalid records go to quarantine

The correct choice depends on the source contract.

For large operational pipelines, record-level quarantine is often useful when partial processing is acceptable.

---

## 26. Quarantine

A quarantine table can contain:

```
CREATE TABLE staging_quarantine (
    quarantine_id BIGSERIAL PRIMARY KEY,
    source_file_id TEXT NOT NULL,
    source_record_id TEXT,
    source_record_number BIGINT,
    raw_record JSONB,
    reason_code TEXT NOT NULL,
    error_message TEXT,
    quarantined_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Never silently discard invalid records.

---

## 27. Error Classification

Separate:

    malformed_source
    schema_error
    validation_error
    duplicate_record
    database_error
    unexpected_exception

This allows the pipeline to choose:

    retry
    quarantine
    fail file
    fail run

instead of applying one response to every error.

---

## 28. Schema Mapping

Suppose source fields are:

    customerId
    transactionAmount
    transactionDate

Staging may use:

    customer_id
    amount_text
    transaction_time_text

Keep the mapping explicit.

Example:

```
FIELD_MAP = {
    "customerId": "customer_id",
    "transactionAmount": "amount_text",
    "transactionDate": "transaction_time_text",
}
```

Do not depend on accidental dictionary ordering or implicit field matching.

---

## 29. Preserve Original Field Names When Useful

A source may change:

    transactionAmount

to:

    amount

If raw payload is preserved, the original field remains available.

If raw payload is not retained, schema debugging becomes harder.

Lineage should therefore connect staging to the immutable raw artifact.

---

## 30. Type Conversion Boundary

Staging can preserve source representation:

    amount_text = "00125.50"

Curated transformation can later produce:

    amount_numeric = 125.50

This can be useful when source formatting itself is operationally significant.

However, do not keep every field as TEXT merely to avoid thinking about types.

Choose types based on the staging contract.

---

## 31. When to Use JSONB

JSONB is useful for:

- flexible source payloads;
- debugging;
- source fields not yet modeled;
- evolving schemas.

But avoid making the entire staging layer:

    payload JSONB

when downstream processing requires frequent typed queries.

A common pattern is:

    important typed columns
    +
    raw_record JSONB

---

## 32. CSV Example

Suppose a CSV contains:

```
customer_id,amount,currency
C1,100.50,EUR
C2,75.20,USD
```

Extraction produces:

```
{
    "source_record_number": 2,
    "customer_id": "C1",
    "amount_text": "100.50",
    "currency": "EUR"
}
```

Staging stores the structured record plus lineage.

The curated layer can later convert amount to a numeric type.

---

## 33. JSON Example

Source:

```
{
  "customerId": "C1",
  "amount": "100.50",
  "currency": "EUR"
}
```

Staging:

```
{
    "source_record_id": "C1",
    "customer_id": "C1",
    "amount_text": "100.50",
    "currency": "EUR",
    "raw_record": {
        "customerId": "C1",
        "amount": "100.50",
        "currency": "EUR"
    }
}
```

The source representation remains traceable.

---

## 34. Record Hash

A deterministic hash can support identity and change detection.

Example:

```
import hashlib
import json

def record_hash(record):
    payload = json.dumps(
        record,
        sort_keys=True,
        separators=(",", ":"),
    ).encode("utf-8")

    return hashlib.sha256(payload).hexdigest()
```

Hash the canonical representation, not an arbitrary Python string representation.

---

## 35. Hash Is Not Automatically Identity

Two identical records can have the same hash.

That may be useful for:

    duplicate detection

but not necessarily for:

    business identity

Use:

    source_record_id

when the source provides a stable identity.

Use hashes for content identity where appropriate.

---

## 36. Raw Artifact Identity

The staging record should be traceable to the raw artifact using an identifier such as:

    source_file_id
    source_object_version
    source_sha256

A useful lineage chain is:

    staging_record
        |
        v
    source_file
        |
        v
    raw_artifact

This makes replay and investigation possible.

---

## 37. Extraction Run Identity

A record may be processed multiple times.

Create:

    extraction_run_id

Example:

```
CREATE TABLE etl_extraction_run (
    run_id UUID PRIMARY KEY,
    source_file_id TEXT NOT NULL,
    started_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    status TEXT NOT NULL
);
```

The run identifies an execution.

The source file identity identifies the business input.

Do not confuse the two.

---

## 38. Why Run ID Is Not Record Identity

A replay creates:

    run_001
    run_002

for the same source file.

If record identity uses run ID:

    same record
    ->
    different identity on every replay

That breaks idempotency.

Use:

    source identity

for the record key.

Use:

    run ID

for execution lineage.

---

## 39. Staging Load Function

A practical Python shape:

```
def stage_records(
    connection,
    source_file_id,
    records,
    extraction_run_id,
):
    inserted = 0
    rejected = 0

    for record in records:
        valid, reason = validate_record(record)

        if not valid:
            quarantine_record(
                connection,
                source_file_id,
                record,
                reason,
            )
            rejected += 1
            continue

        if insert_staging_record(
            connection,
            source_file_id,
            extraction_run_id,
            record,
        ):
            inserted += 1

    return inserted, rejected
```

Production implementations should batch database operations.

The structure matters more than this exact function signature.

---

## 40. Batch Insert With psycopg

Example:

```
from psycopg import sql
from psycopg.rows import dict_row

INSERT_SQL = """
INSERT INTO staging_transactions (
    source_file_id,
    source_record_id,
    customer_id,
    amount_text,
    currency
)
VALUES (%s, %s, %s, %s, %s)
ON CONFLICT (source_file_id, source_record_id)
DO NOTHING
"""
```

Batch execution can be implemented with the database driver's batch facilities.

Keep transaction boundaries explicit.

---

## 41. Transaction Boundary

A useful pattern:

    extract records
        |
        v
    validate batch
        |
        v
    insert batch
        |
        v
    commit

If a batch fails:

    rollback
    retry or quarantine according to error class

Do not leave half-applied transaction state.

---

## 42. Batch Size

There is no universal perfect batch size.

Start with something such as:

    500
    1,000
    5,000

Then measure.

Trade-offs:

    smaller batch
        + lower rollback cost
        + less memory
        - more database round trips

    larger batch
        + higher throughput
        - larger transaction
        - larger rollback
        - more memory

Benchmark with realistic records.

---

## 43. Backpressure

If extraction is faster than PostgreSQL:

    extractor
        >>>>
    database

memory can grow.

Use bounded batches or queues.

Example:

    read batch
       |
       v
    validate
       |
       v
    write
       |
       v
    next batch

Do not load the entire raw file into memory merely because staging is slower.

---

## 44. Large Files

For large sources:

    raw file
       |
       v
    streaming parser
       |
       v
    bounded batch
       |
       v
    staging
       |
       v
    next batch

Combine this recipe with E61 Large File Streaming.

The staging layer should not force whole-file memory usage.

---

## 45. Checkpointing

For very large files, a staging process may need checkpoints.

Possible checkpoint:

    source_file_id
    source_record_number
    last_committed_at

Example:

```
CREATE TABLE etl_staging_checkpoint (
    source_file_id TEXT PRIMARY KEY,
    last_record_number BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Only advance the checkpoint after the corresponding staging transaction commits.

---

## 46. Checkpoint Correctness

Bad sequence:

    insert records
    checkpoint
    transaction fails

Now the checkpoint says the records were staged when they were not.

Correct sequence:

    insert batch
    commit
    advance checkpoint

Or make checkpoint and staging update part of one database transaction when they share the same database.

---

## 47. Replay

A replay should be safe.

Example:

    run_001 -> stages 10,000 records
    run_002 -> replays same file

Expected:

    no duplicate records

The uniqueness constraint and deterministic source identity provide the safety boundary.

---

## 48. Partial Replay

You may replay:

    record 5000 onward

This is useful after a failure.

But do not make replay correctness depend only on a checkpoint.

The database uniqueness constraint should remain the final protection against duplicates.

---

## 49. Reconciliation

After staging:

    extracted_records
        =
    staged_valid_records
        +
    quarantined_records
        +
    explicitly_skipped_records

Example:

    extracted = 1000
    staged = 980
    quarantined = 20

Then:

    1000 = 980 + 20

If the equation does not balance, investigate.

---

## 50. Control Totals

If the source provides:

    record_count = 1000

compare it to staging outcomes.

Possible checks:

    source_count
    extracted_count
    staged_count
    quarantined_count

The reconciliation rule should be explicit.

---

## 51. Hash Reconciliation

If the source provides a checksum or manifest:

    source_hash

compare it with the raw artifact.

Staging lineage should point to the verified raw artifact.

Do not recompute a different representation and call it the source hash.

---

## 52. Schema Drift

Source adds:

    customer_segment

A robust staging layer should not immediately destroy the entire load.

Possible responses:

1. ignore unknown field;
2. preserve in raw_record;
3. add staging column;
4. fail according to contract.

The correct response depends on schema policy.

Do not silently discard fields if the contract requires strict schema control.

---

## 53. Missing Required Field

Suppose:

    customer_id

is required but absent.

Possible result:

    validation_status = INVALID
    processing_status = QUARANTINED
    rejection_reason = MISSING_CUSTOMER_ID

Do not insert an invalid record as READY merely because the database permits NULL.

---

## 54. Type Mismatch

Source:

    amount = "ABC"

If staging expects numeric:

    validation fails

If staging intentionally preserves source text:

    amount_text = "ABC"
    validation_status = INVALID

Choose one contract.

Do not let implicit database casts determine the pipeline's validation behavior.

---

## 55. Null vs Missing

These are different:

    field absent

and:

    field present with null

Example:

```
{}
```

versus:

```
{"customer_id": null}
```

A source contract may assign different meanings.

Preserve that distinction when it matters.

---

## 56. Empty String

Also distinguish:

    null
    ""
    "   "

Normalization may happen later.

Do not silently collapse them during staging unless the staging contract explicitly requires it.

---

## 57. Encoding

The raw artifact may use:

    UTF-8
    UTF-8 with BOM
    Latin-1
    another declared encoding

Staging should record encoding metadata when useful.

Do not silently replace undecodable bytes.

Extraction should either:

    decode correctly

or:

    fail/quarantine with evidence.

---

## 58. Source Order

Do not assume staging table order equals source order.

PostgreSQL tables do not guarantee row order without ORDER BY.

If source position matters, store:

    source_record_number

and query:

```
SELECT *
FROM staging_transactions
ORDER BY source_record_number;
```

---

## 59. Staging Table Indexes

Useful indexes often include:

```
CREATE INDEX idx_staging_source_file
ON staging_transactions (source_file_id);

CREATE INDEX idx_staging_processing_status
ON staging_transactions (processing_status);

CREATE INDEX idx_staging_staged_at
ON staging_transactions (staged_at);
```

Do not add every imaginable index.

Indexes increase:

- storage;
- write cost;
- maintenance.

Create indexes based on actual access patterns.

---

## 60. Partitioning

For very large staging tables, partitioning may be useful.

Possible key:

    staged_date

or:

    source_business_date

Partition only when volume and query/retention requirements justify it.

Do not partition merely because a table is called staging.

---

## 61. Retention

Staging retention depends on:

- replay requirements;
- audit requirements;
- storage cost;
- downstream rebuild capability;
- regulatory requirements.

A common model is:

    raw -> long retention
    staging -> shorter retention
    curated -> business retention

But the exact policy is system-specific.

---

## 62. Cleanup Safety

Never delete staging data merely because:

    processed_status = PROCESSED

unless the retention policy says it is safe.

Before cleanup, verify:

- raw artifact is retained;
- replay mechanism works;
- downstream audit requirements are satisfied;
- no active incident requires the staging rows.

---

## 63. Concurrency

Two workers may process the same file.

Without protection:

    worker A -> insert record
    worker B -> insert record

Use:

    UNIQUE(source_file_id, source_record_id)

and transactional conflict handling.

Do not depend on the orchestrator guaranteeing one worker forever.

---

## 64. Worker Claiming

If records are processed after staging, use a claim pattern.

Example:

```
UPDATE staging_transactions
SET processing_status = 'PROCESSING'
WHERE staging_id IN (
    SELECT staging_id
    FROM staging_transactions
    WHERE processing_status = 'READY'
    ORDER BY staging_id
    FOR UPDATE SKIP LOCKED
    LIMIT 100
)
RETURNING *;
```

This supports multiple workers without processing the same row simultaneously.

---

## 65. Failed Processing

A downstream transformation failure should not necessarily delete the staging record.

Instead:

    processing_status = FAILED

and store:

    failure_code
    failure_message
    failed_at
    retry_count

Staging is valuable precisely because it provides a durable processing boundary.

---

## 66. Retry

Retry transient failures:

    database unavailable
    network timeout
    temporary dependency failure

Do not endlessly retry permanent data errors:

    invalid type
    missing required field
    malformed business value

Classify errors before deciding to retry.

---

## 67. Quarantine vs Retry

Use quarantine for:

    invalid record
    schema violation
    unrecoverable data error

Use retry for:

    transient infrastructure failure
    temporary database failure
    temporary source dependency failure

A retry loop should not be used as a substitute for validation.

---

## 68. Audit Trail

For production operation, record:

    who/what loaded it
    when
    from which source
    which run
    which contract version
    how many records
    how many rejected
    how many inserted
    how many duplicates

This allows an operator to reconstruct the staging event.

---

## 69. Staging Load Audit Table

Example:

```
CREATE TABLE etl_staging_load (
    load_id BIGSERIAL PRIMARY KEY,
    source_file_id TEXT NOT NULL,
    extraction_run_id UUID NOT NULL,
    started_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    extracted_count BIGINT NOT NULL DEFAULT 0,
    staged_count BIGINT NOT NULL DEFAULT 0,
    quarantined_count BIGINT NOT NULL DEFAULT 0,
    duplicate_count BIGINT NOT NULL DEFAULT 0,
    status TEXT NOT NULL
);
```

This is useful for reconciliation and operations.

---

## 70. Example End-to-End Flow

Input:

```
{
  "customerId": "C1",
  "amount": "100.50",
  "currency": "EUR"
}
```

Flow:

    raw artifact
        |
        v
    JSON extraction
        |
        v
    source record identity
        |
        v
    validation
        |
        +---- invalid ---> quarantine
        |
        v
    staging insert
        |
        v
    READY
        |
        v
    downstream transformation

Lineage remains available throughout.

---

## 71. Example Staging Row

Conceptually:

```
{
  "source_file_id": "file-20260926-001",
  "source_record_id": "C1",
  "customer_id": "C1",
  "amount_text": "100.50",
  "currency": "EUR",
  "validation_status": "VALID",
  "processing_status": "READY",
  "extraction_run_id": "run-123",
  "source_record_number": 1
}
```

The exact representation depends on the database schema.

---

## 72. Production Test — Happy Path

Input:

    3 valid records

Expected:

    extracted = 3
    staged = 3
    quarantined = 0
    duplicates = 0

All records should be READY.

---

## 73. Production Test — Invalid Record

Input:

    2 valid
    1 invalid

Expected:

    extracted = 3
    staged = 2
    quarantined = 1

Reconciliation must balance.

---

## 74. Production Test — Replay

Run the same source file twice.

Expected:

    first run -> inserts records
    second run -> duplicates ignored or handled according to policy

Final logical record count remains unchanged.

---

## 75. Production Test — Concurrent Replay

Run two workers against the same source simultaneously.

Expected:

    no duplicate logical records

The database uniqueness constraint is the final guard.

---

## 76. Production Test — Batch Failure

Inject a database error during a batch.

Verify:

    failed transaction rolls back
    checkpoint does not advance incorrectly
    retry can safely replay the batch

---

## 77. Production Test — Schema Addition

Add an unknown source field.

Verify behavior matches the contract:

    preserve
    ignore
    or fail

Do not leave behavior undefined.

---

## 78. Production Test — Source Replacement

Load:

    source version A

then:

    source version B

Verify that the staging model follows the configured replacement/versioning policy.

Do not accidentally overwrite historical evidence.

---

## 79. Intentional Failure Drill — Duplicate Replay

1. Stage a source file.
2. Run the same file again.
3. Verify uniqueness protection.
4. Verify no duplicate records.
5. Verify duplicate count is observable.
6. Verify the load remains restartable.

---

## 80. Intentional Failure Drill — Database Failure

1. Begin a staging batch.
2. Interrupt the database connection.
3. Verify transaction rollback.
4. Verify checkpoint did not advance incorrectly.
5. Restart the load.
6. Verify idempotent recovery.

---

## 81. Intentional Failure Drill — Quarantine

1. Include malformed records.
2. Run staging.
3. Verify valid records continue if tolerant mode is configured.
4. Verify invalid records are quarantined.
5. Verify rejection reasons are preserved.
6. Reconcile counts.

---

## 82. Intentional Failure Drill — Concurrent Workers

1. Start two staging workers for the same file.
2. Process overlapping records.
3. Verify unique constraint behavior.
4. Verify no duplicate logical records.
5. Verify both workers terminate safely.

---

## 83. Observability

Track:

    staging_records_extracted_total
    staging_records_inserted_total
    staging_records_quarantined_total
    staging_records_duplicate_total
    staging_load_duration_seconds
    staging_batch_duration_seconds
    staging_failed_loads_total
    staging_current_ready_records

Useful dimensions:

    source_system
    file_type
    environment
    status

Avoid unique record IDs as metric labels.

---

## 84. Structured Logs

Example:

```
logger.info(
    "staging_load_completed",
    extra={
        "source_file_id": source_file_id,
        "extraction_run_id": str(extraction_run_id),
        "extracted_count": extracted_count,
        "staged_count": staged_count,
        "quarantined_count": quarantined_count,
        "duplicate_count": duplicate_count,
        "duration_seconds": duration_seconds,
    },
)
```

Log technical metadata, not raw sensitive payloads.

---

## 85. Production Runbook

When a staging load fails:

### Step 1 — Identify the source

Find:

    source_file_id
    source artifact
    extraction run

### Step 2 — Inspect the load audit

Check:

    extracted count
    staged count
    quarantine count
    duplicate count
    failure state

### Step 3 — Check database transaction

Determine whether the last batch committed or rolled back.

### Step 4 — Check checkpoint

Verify it only advanced after a successful commit.

### Step 5 — Classify the failure

Determine:

    data error
    schema error
    database error
    infrastructure error

### Step 6 — Recover

Choose:

    retry
    replay
    quarantine
    contract change

### Step 7 — Reconcile

Verify:

    extracted
    staged
    quarantined
    duplicates

### Step 8 — Continue downstream processing

Only after staging is known to be consistent.

---

## 86. Common Mistakes

### Mistake 1 — Treating raw and staging as the same layer

**Fix:** Keep raw evidence immutable and staging structured.

### Mistake 2 — Using random UUIDs as record identity

**Fix:** Derive deterministic source identity.

### Mistake 3 — Committing every record

**Fix:** Use bounded transactions.

### Mistake 4 — No uniqueness constraint

**Fix:** Put idempotency enforcement in the database.

### Mistake 5 — Using run ID as record identity

**Fix:** Run identity describes execution, not source identity.

### Mistake 6 — Dropping invalid records silently

**Fix:** Quarantine or fail explicitly.

### Mistake 7 — Treating every error as retryable

**Fix:** Classify transient vs permanent errors.

### Mistake 8 — Advancing checkpoint before commit

**Fix:** Commit data before advancing the checkpoint.

### Mistake 9 — Losing source lineage

**Fix:** Preserve source file and record identity.

### Mistake 10 — Turning staging into the business model

**Fix:** Keep business transformations downstream.

### Mistake 11 — Assuming table order equals source order

**Fix:** Persist source position when order matters.

### Mistake 12 — Adding every possible index

**Fix:** Index based on actual query and processing patterns.

### Mistake 13 — Deleting processed staging immediately

**Fix:** Follow an explicit retention and replay policy.

### Mistake 14 — Storing only JSONB

**Fix:** Model frequently queried fields with appropriate types.

### Mistake 15 — Storing every field as TEXT

**Fix:** Choose types deliberately while preserving source representation where useful.

---

## 87. Debugging Questions

When a staging load looks wrong, ask:

1. Which raw artifact produced these rows?
2. Which extraction run created them?
3. What is the deterministic record identity?
4. How many records were extracted?
5. How many were staged?
6. How many were quarantined?
7. How many were duplicates?
8. Did the batch transaction commit?
9. Did the checkpoint advance correctly?
10. Was the source schema changed?
11. Was a source record replaced?
12. Did two workers process the same file?
13. Is the raw artifact still available?
14. Can the load be replayed safely?
15. Does reconciliation balance?

These questions should be answerable from metadata.

---

## 88. Production Implementation Sequence

### Step 1 — Define the staging contract

Specify:

- record identity;
- source lineage;
- typed columns;
- raw payload policy;
- validation state;
- processing state.

### Step 2 — Create the staging schema

Add:

- primary key;
- deterministic uniqueness;
- operational indexes.

### Step 3 — Define source-to-staging mapping

Map source fields explicitly.

### Step 4 — Define validation behavior

Separate structural validation from downstream business validation.

### Step 5 — Define error policy

Choose:

    retry
    quarantine
    fail

for each error class.

### Step 6 — Implement batch loading

Use bounded transactions.

### Step 7 — Add idempotency

Enforce deterministic uniqueness in PostgreSQL.

### Step 8 — Add lineage

Persist source file, record, and extraction run identity.

### Step 9 — Add checkpointing when required

Advance checkpoints only after successful commits.

### Step 10 — Add reconciliation

Compare extracted, staged, quarantined, duplicate, and source counts.

### Step 11 — Add observability

Metrics, structured logs, and load audits.

### Step 12 — Run failure drills

Test replay, concurrency, database failures, invalid records, and schema changes.

---

## 89. Production Checklist

### Architecture

- [ ] Raw is preserved separately.
- [ ] Staging has a clear contract.
- [ ] Curated business logic remains downstream.

### Identity

- [ ] Record identity is deterministic.
- [ ] File identity is preserved.
- [ ] Run identity is separate from record identity.
- [ ] Unique constraints enforce idempotency.

### Lineage

- [ ] Source artifact is identifiable.
- [ ] Source record is identifiable.
- [ ] Source position is preserved where useful.
- [ ] Extraction run is recorded.

### Validation

- [ ] Structural validation is defined.
- [ ] Invalid records are quarantined or explicitly fail.
- [ ] Error reasons are preserved.
- [ ] Retryable and permanent errors are distinguished.

### Loading

- [ ] Inserts use bounded batches.
- [ ] Transactions are explicit.
- [ ] Checkpoints advance only after commit.
- [ ] Concurrent replay is safe.

### Reconciliation

- [ ] Extracted count is recorded.
- [ ] Staged count is recorded.
- [ ] Quarantined count is recorded.
- [ ] Duplicate count is recorded.
- [ ] Counts reconcile.

### Operations

- [ ] Load audit exists.
- [ ] Structured logs exist.
- [ ] Metrics exist.
- [ ] Replay procedure exists.
- [ ] Retention policy exists.

### Testing

- [ ] Happy path exists.
- [ ] Invalid record test exists.
- [ ] Replay test exists.
- [ ] Concurrent replay test exists.
- [ ] Batch rollback test exists.
- [ ] Schema drift test exists.
- [ ] Replacement test exists.
- [ ] Failure recovery test exists.

---

## 90. Production Tools You Should Know

### PostgreSQL

Use for:

- staging tables;
- uniqueness constraints;
- transactional batches;
- checkpoints;
- reconciliation;
- load audit.

Know:

- UNIQUE constraints;
- ON CONFLICT;
- transactions;
- conditional UPDATE;
- FOR UPDATE SKIP LOCKED;
- JSONB;
- indexes.

### Python

Use for:

- source mapping;
- validation;
- deterministic identity;
- extraction orchestration;
- batch construction.

Know:

- streaming iterators;
- dataclasses;
- hashing;
- timezone-aware datetimes;
- explicit exception classification.

### Object Storage

Use for:

- immutable raw artifacts;
- source versioning;
- replay;
- lineage;
- retention.

Know:

- object version IDs;
- checksums;
- metadata;
- immutable/raw retention policies.

---

## 91. Package Structure

A practical implementation:

    etl/
      staging/
        models.py
        mapper.py
        validator.py
        loader.py
        checkpoint.py
        reconciliation.py
        quarantine.py

      lineage/
        source.py
        records.py

      tests/
        test_mapping.py
        test_validation.py
        test_loader.py
        test_idempotency.py
        test_replay.py
        test_reconciliation.py

Keep:

    extraction
    mapping
    validation
    loading
    reconciliation

as separate responsibilities.

---

## 92. Design Principle — Raw Is Evidence

Raw should answer:

> What exactly did the source deliver?

Do not mutate raw to make staging easier.

If staging is wrong, fix staging.

If a source sends bad data, preserve the source evidence and handle the problem downstream.

---

## 93. Design Principle — Staging Is a Boundary

Staging should answer:

> What extracted records are ready for controlled downstream processing?

It is the boundary between:

    source representation

and:

    transformation logic

Do not let every downstream job parse raw files independently.

---

## 94. Design Principle — Idempotency Belongs in the Database

Application logic can accidentally fail.

A database uniqueness constraint remains an enforceable invariant.

Use:

    deterministic identity
        +
    UNIQUE constraint
        +
    idempotent write

as the production replay boundary.

---

## 95. Design Principle — Preserve Lineage

Every staging record should be traceable:

    staging record
        |
        v
    source record
        |
        v
    source file
        |
        v
    raw artifact

If an operator cannot follow that chain, production debugging becomes unnecessarily difficult.

---

## 96. Design Principle — Reconciliation Is Part of Loading

A successful database transaction does not prove the staging load is complete.

Always reconcile:

    source/extracted
        =
    staged
      +
    quarantined
      +
    explicitly skipped

The exact equation depends on the source contract.

---

## 97. Definition of Done

You are done with T01 when you can independently implement a staging layer that:

1. preserves raw evidence separately;
2. defines deterministic record identity;
3. preserves source lineage;
4. maps source fields explicitly;
5. separates structural validation from business transformation;
6. handles invalid records according to policy;
7. loads records in bounded transactions;
8. enforces idempotency in PostgreSQL;
9. supports safe replay;
10. supports concurrent processing;
11. uses checkpoints correctly when required;
12. reconciles extracted and staged outcomes;
13. preserves audit metadata;
14. handles schema drift according to contract;
15. supports operational observability;
16. survives database and worker failures;
17. provides a production recovery runbook.

If you can implement those mechanisms independently, you understand production raw-to-staging transformation.

---

## 98. What You Learned

The central model is:

    immutable raw evidence
             |
             v
         extraction
             |
             v
       source identity
             |
             v
         validation
          /       \
         /         \
        v           v
     valid        invalid
       |             |
       v             v
    staging      quarantine
       |
       v
   downstream
 transformation

The key rules are:

1. Raw and staging are different layers.
2. Staging is a controlled transformation boundary.
3. Record identity must be deterministic.
4. Run identity must remain separate from record identity.
5. Source lineage must survive the transformation.
6. Idempotency should be enforced by database constraints.
7. Batch transactions should be bounded.
8. Invalid records must be explicitly handled.
9. Checkpoints advance only after successful commits.
10. Reconciliation proves that extracted outcomes are accounted for.
11. Raw evidence should remain available for replay.
12. Staging should remain technical rather than becoming the final business model.

Next:

    T02 — Data Type Conversion
