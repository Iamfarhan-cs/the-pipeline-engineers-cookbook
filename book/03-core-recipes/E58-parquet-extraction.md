# E58 — Parquet Extraction

## 1. Problem Recognition

Parquet is a columnar storage format designed for analytical workloads. It is common in data lakes, object storage, warehouse staging, batch ETL, and distributed data processing.

Parquet extraction is different from CSV or JSON extraction because the main performance question is not only how to parse records. It is how to read the minimum amount of data required.

Production extraction must understand:

- columns;
- physical and logical types;
- row groups;
- compression;
- partitioned datasets;
- predicate pushdown;
- column projection;
- schema evolution;
- statistics;
- null semantics;
- source identity;
- idempotent staging.

The core principle is:

@@@
DO NOT READ WHAT YOU DO NOT NEED
@@@

Production flow:

@@@
DISCOVER PARQUET
      |
      v
INSPECT SCHEMA
      |
      v
SELECT REQUIRED COLUMNS
      |
      v
APPLY FILTERS
      |
      v
READ RELEVANT ROW GROUPS / FILES
      |
      v
VALIDATE
      |
      v
STAGE
      |
      v
OBSERVE + RECONCILE
@@@

## 2. Why Parquet Is Different

A row-oriented format stores records together. Parquet stores data by columns inside row groups.

Conceptually:

@@@
Row format:
record 1 -> A B C
record 2 -> A B C
record 3 -> A B C

Parquet:
column A -> A A A
column B -> B B B
column C -> C C C
@@@

If an ETL job needs only A and C, a columnar engine can avoid reading B.

That is the basis of column projection.

## 3. Column Projection

Bad:

@@@
table = pq.read_table("payments.parquet")
@@@

when the pipeline needs only:

@@@
payment_id
amount
currency
@@@

Better:

@@@
import pyarrow.parquet as pq

table = pq.read_table(
    "payments.parquet",
    columns=[
        "payment_id",
        "amount",
        "currency",
    ],
)
@@@

Reading only required columns reduces I/O and memory.

## 4. Predicate Pushdown

Suppose the source contains millions of records but the pipeline needs only one business date.

Instead of reading everything and filtering afterward, push the filter toward the Parquet reader.

Conceptually:

@@@
source
  |
  v
predicate
  |
  v
read matching data
@@@

Predicate pushdown can allow the reader to skip irrelevant row groups or files.

## 5. Why Pushdown Works

Parquet files contain metadata and statistics at the row-group level.

For example, a row group may have:

@@@
min(event_date) = 2026-09-01
max(event_date) = 2026-09-01
@@@

If the query asks for 2026-09-26, that row group can be skipped.

This is data skipping, not magic.

It depends on:

- available statistics;
- file layout;
- row-group boundaries;
- predicate shape;
- reader capabilities.

## 6. Row Groups

A Parquet file is divided into row groups.

Conceptually:

@@@
file
 |
 +-- row group 1
 +-- row group 2
 +-- row group 3
 +-- row group 4
@@@

Each row group contains column chunks.

Row-group size affects:

- parallelism;
- metadata overhead;
- compression efficiency;
- data skipping;
- read performance.

Do not treat row groups as application records. They are storage-level units.

## 7. Inspecting Metadata

Use PyArrow to inspect the file:

@@@
import pyarrow.parquet as pq

metadata = pq.ParquetFile(
    "payments.parquet"
)

print(metadata.metadata)
print(metadata.schema)
@@@

Useful metadata includes:

- number of rows;
- number of row groups;
- schema;
- file metadata.

## 8. Inspecting Row Groups

@@@
parquet_file = pq.ParquetFile(
    "payments.parquet"
)

print(
    parquet_file.num_row_groups
)

for index in range(
    parquet_file.num_row_groups
):
    row_group = parquet_file.metadata.row_group(index)
    print(
        index,
        row_group.num_rows,
        row_group.total_byte_size,
    )
@@@

This is useful when diagnosing unexpectedly slow extraction.

## 9. Schema Inspection

Inspect the schema before extraction:

@@@
schema = pq.read_schema(
    "payments.parquet"
)

print(schema)
@@@

Verify required columns before reading millions of rows.

Fail early if a required column is missing.

## 10. Required vs Optional Columns

Define the extraction contract.

Required:

@@@
required_columns = {
    "payment_id",
    "amount",
    "currency",
}
@@@

Optional:

@@@
optional_columns = {
    "channel",
    "merchant_country",
}
@@@

Check the source schema:

@@@
available = set(schema.names)
missing = required_columns - available

if missing:
    raise ValueError(
        f"Missing required columns: {sorted(missing)}"
    )
@@@

Do not let a missing required column silently become an all-null output.

## 11. Type Validation

Parquet schemas contain types, but the application contract may be stricter.

Example:

@@@
schema = pq.read_schema(
    "payments.parquet"
)

print(schema.field("amount").type)
@@@

Validate important types explicitly.

Be particularly careful with:

- decimal vs floating point;
- timestamp units;
- timezone metadata;
- integer width;
- nullable fields;
- dictionary encoding.

## 12. Decimal Fields

Financial amounts should retain exact decimal semantics when required.

Prefer a Parquet decimal type over a binary floating-point representation when the source contract requires exact values.

Validate the expected precision and scale before loading into a financial model.

Do not convert every monetary value to float merely because Python can represent it.

## 13. Timestamp Semantics

Parquet can store timestamp values with different units and timezone metadata.

Inspect them rather than assuming:

@@@
field = schema.field("occurred_at")
print(field.type)
@@@

Your extraction contract should define the expected timezone semantics.

## 14. Null Semantics

Parquet supports nullable columns.

Do not confuse:

- missing column;
- present column with null values;
- empty string;
- zero;
- false.

These can represent different business states.

Preserve the source semantics until the transformation layer intentionally changes them.

## 15. Partitioned Parquet Datasets

Parquet is frequently stored as a directory of files rather than one file.

Example:

@@@
payments/
  year=2026/
    month=09/
      day=25/
        part-0001.parquet
        part-0002.parquet
      day=26/
        part-0003.parquet
@@@

Partition columns are often encoded in the path.

This can allow entire directories or files to be skipped before opening them.

## 16. Partition Pruning

Suppose the query requires:

@@@
year = 2026
month = 9
day = 26
@@@

A partition-aware reader can avoid reading day 25 entirely.

This is often more significant than row-level filtering because entire files are eliminated before scanning.

## 17. Reading a Dataset

Use PyArrow Dataset for partitioned collections:

@@@
import pyarrow.dataset as ds

dataset = ds.dataset(
    "payments/",
    format="parquet",
    partitioning="hive",
)

table = dataset.to_table(
    columns=[
        "payment_id",
        "amount",
        "currency",
    ]
)
@@@

Dataset is generally a better abstraction than treating hundreds of files as unrelated individual reads.

## 18. Filtered Dataset Read

Example:

@@@
import pyarrow.dataset as ds

dataset = ds.dataset(
    "payments/",
    format="parquet",
    partitioning="hive",
)

table = dataset.to_table(
    columns=[
        "payment_id",
        "amount",
        "currency",
    ],
    filter=(
        (ds.field("year") == 2026)
        & (ds.field("month") == 9)
        & (ds.field("day") == 26)
    ),
)
@@@

The filter can enable partition pruning and predicate pushdown.

## 19. Filter Correctness

Performance optimizations must not change semantics.

Verify that:

- filter columns exist;
- date types match the predicate;
- timezone semantics are correct;
- partition values are interpreted correctly;
- null behavior is understood.

A fast wrong query is still wrong.

## 20. Row Group Statistics

Parquet readers can use statistics such as min and max values to skip row groups.

Conceptually:

@@@
requested date = 2026-09-26

row group 1: 2026-09-20 -> 2026-09-20  SKIP
row group 2: 2026-09-26 -> 2026-09-26  READ
row group 3: 2026-09-30 -> 2026-09-30  SKIP
@@@

Statistics are most useful when data layout aligns with common predicates.

## 21. File Layout Matters

Two Parquet datasets can contain the same data but have very different extraction performance.

Good layout for a date-heavy workload:

@@@
year/month/day partitions
@@@

Potentially poor layout:

@@@
one enormous unpartitioned file with random dates
@@@

Extraction performance is partly determined upstream by how files are written.

## 22. Small Files Problem

A dataset containing millions of tiny Parquet files can be slow even though each file is efficient.

Problems include:

- excessive metadata operations;
- object-storage request overhead;
- scheduler overhead;
- too many file opens;
- inefficient parallelism.

Do not assume more files always means more parallelism.

## 23. Large Files Problem

One enormous file can also be problematic.

Potential issues:

- slower recovery;
- limited parallelism;
- large metadata units;
- expensive rewrites.

Choose file and row-group sizes based on workload rather than a universal number.

## 24. Reading Batches

For large extracts, process record batches rather than materializing a giant table.

@@@
scanner = dataset.scanner(
    columns=[
        "payment_id",
        "amount",
        "currency",
    ],
)

for batch in scanner.to_batches():
    process_batch(batch)
@@@

This keeps application memory bounded by the processing batch rather than the entire dataset.

## 25. Batch Size

Batch size affects:

- memory;
- CPU efficiency;
- database write size;
- transaction duration.

Use measured values.

Do not create a batch so small that database overhead dominates, or so large that failure recovery becomes expensive.

## 26. Schema Evolution

Parquet datasets can evolve.

Examples:

- new optional column;
- renamed column;
- changed type;
- changed nullability;
- incompatible logical type.

Additive changes may be compatible. Type changes may not be.

Define a schema compatibility policy.

## 27. Missing Column in One Partition

Suppose day 25 has:

@@@
payment_id
amount
currency
@@@

but day 26 adds:

@@@
channel
@@@

A dataset reader may need to unify schemas across files.

Do not assume every file has identical physical schemas without checking.

## 28. Schema Unification

For evolving datasets, inspect how the reader resolves schemas.

Possible outcomes include:

- union of fields;
- null for missing fields;
- failure on incompatible types.

The chosen behavior must match the ETL contract.

Do not allow automatic schema unification to hide a breaking type change.

## 29. Partition Columns vs Physical Columns

A partitioned dataset may expose:

@@@
year
month
day
@@@

even when those values are not stored as normal Parquet columns in every file.

Treat partition metadata and physical file schema as related but distinct concepts.

## 30. File Identity

For every source file, preserve:

@@@
source_uri
source_file
source_hash
dataset_partition
file_size
row_count
extracted_at
@@@

Dataset-level extraction should not destroy file-level lineage.

## 31. Record Identity

Parquet does not automatically provide a business record identity.

Prefer a source business key:

@@@
payment_id
@@@

If none exists, combine source file identity and deterministic row position where appropriate.

Do not use a random UUID as the only identity for replay-sensitive ingestion.

## 32. Idempotent Staging

Example PostgreSQL staging table:

@@@
CREATE TABLE parquet_staging (
    source_file TEXT NOT NULL,
    source_hash TEXT NOT NULL,
    source_row_number BIGINT NOT NULL,
    record_id TEXT,
    payload JSONB NOT NULL,
    extracted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (source_hash, source_row_number)
);
@@@

Insert idempotently:

@@@
INSERT INTO parquet_staging (
    source_file,
    source_hash,
    source_row_number,
    record_id,
    payload
)
VALUES (%s, %s, %s, %s, %s)
ON CONFLICT (source_hash, source_row_number)
DO NOTHING;
@@@

Choose the identity based on the source contract. A dataset may have stable business keys that are better than physical row positions.

## 33. Do Not Assume Row Order

Parquet is not a database table with guaranteed application ordering.

Do not assume that:

@@@
row 1 < row 2 < row 3
@@@

represents event order.

If business ordering matters, use an explicit event timestamp or sequence field.

## 34. Reading Only What Is Needed

A common production pattern is:

@@@
required_columns = [
    "payment_id",
    "amount",
    "currency",
]

scanner = dataset.scanner(
    columns=required_columns,
)
@@@

Keep the extraction projection close to the business requirement.

Do not read hundreds of columns because they might be useful later.

## 35. Predicate Pushdown vs Application Filtering

Less efficient:

@@@
table = dataset.to_table()
filtered = table.filter(
    pc.equal(table["currency"], "EUR")
)
@@@

Preferred when supported:

@@@
table = dataset.to_table(
    filter=(ds.field("currency") == "EUR")
)
@@@

The second approach gives the storage reader an opportunity to skip data.

## 36. Projection and Pushdown Together

Combine both:

@@@
table = dataset.to_table(
    columns=[
        "payment_id",
        "amount",
    ],
    filter=(
        ds.field("currency") == "EUR"
    ),
)
@@@

Read fewer columns and fewer rows.

That is one of the most important Parquet extraction patterns.

## 37. Source Validation

Before processing a file, validate:

- extension or object metadata;
- readable Parquet footer;
- expected schema;
- expected partition;
- file size;
- optional checksum;
- row count when known.

Do not treat a file with a valid filename as a valid dataset artifact.

## 38. Corrupt Parquet

A damaged Parquet file may fail when reading the footer or a specific column chunk.

Classify this separately from a schema mismatch.

Useful categories:

@@@
PARQUET_CORRUPT
PARQUET_SCHEMA_MISMATCH
PARQUET_READ_FAILURE
PARTITION_MISMATCH
@@@

Quarantine the artifact and preserve its source identity.

## 39. Footer and Metadata Failures

Parquet readers often depend on the footer for schema and row-group metadata.

If the footer cannot be read, fail before pretending that zero rows were found.

Zero rows and unreadable metadata are different outcomes.

## 40. Compression

Parquet commonly uses compression internally.

The extractor generally should not manually decompress the file before reading it.

Let the Parquet reader decode the column chunks.

Manual whole-file decompression can destroy the advantages of the format.

## 41. Object Storage

Parquet is frequently stored in object storage.

Logical flow:

@@@
object storage
      |
      v
Parquet metadata
      |
      v
partition pruning
      |
      v
column projection
      |
      v
predicate pushdown
      |
      v
record batches
      |
      v
staging
@@@

Keep storage access separate from transformation logic.

## 42. Multi-File Extraction

A partitioned dataset can contain many files.

Do not treat the dataset as one opaque source.

Track each file:

@@@
source_file
source_hash
partition
rows_seen
rows_staged
status
@@@

This makes partial failure recovery much easier.

## 43. File-Level State

Example:

@@@
DISCOVERED
  -> VALIDATING
  -> EXTRACTING
  -> COMPLETED

or

DISCOVERED
  -> FAILED
  -> QUARANTINED
@@@

A dataset run can be completed only after its required files are successfully processed.

## 44. Dataset Completeness

Parquet partition directories do not automatically prove completeness.

If a daily dataset should contain 24 hourly files, define that contract explicitly.

Extraction should consume the completeness state from earlier pipeline controls rather than guessing that the dataset is complete because some files exist.

## 45. Partition Discovery

Use explicit partition metadata when possible.

Example:

@@@
year=2026/month=09/day=26/hour=08
@@@

Validate that the partition encoded in the path agrees with the data when this invariant matters.

A file named day=26 containing day=25 data is a data-quality problem even if Parquet itself is valid.

## 46. Predicate Correctness

Filtering on timestamps requires careful timezone handling.

Example problem:

@@@
partition day = UTC day
application day = local timezone day
@@@

Those may not represent the same records.

Define the business timezone explicitly.

## 47. Statistics and Nulls

Min/max statistics do not behave like ordinary application comparisons when nulls are involved.

Understand the reader's null semantics before relying on row-group skipping for correctness-sensitive logic.

Optimization must never be confused with business validation.

## 48. Snapshot vs Incremental Extraction

Parquet datasets may represent:

- full snapshots;
- daily partitions;
- append-only events;
- corrections;
- late-arriving records.

The extraction strategy depends on the dataset semantics.

Do not infer incremental behavior merely because new Parquet files appear.

## 49. Late and Corrected Files

A partition can be rewritten or corrected.

Track:

@@@
source_hash
file_version
partition
arrival_time
@@@

Use source versioning or deterministic replacement logic when the producer supports it.

Never blindly append a corrected snapshot as though it were a new event stream.

## 50. End-to-End Extraction Skeleton

@@@
import pyarrow.dataset as ds

def extract_payments(dataset_path, start_day, end_day):
    dataset = ds.dataset(
        dataset_path,
        format="parquet",
        partitioning="hive",
    )

    required_columns = [
        "payment_id",
        "amount",
        "currency",
        "occurred_at",
    ]

    scanner = dataset.scanner(
        columns=required_columns,
        filter=(
            (ds.field("occurred_at") >= start_day)
            & (ds.field("occurred_at") < end_day)
        ),
    )

    for batch in scanner.to_batches():
        validate_batch(batch)
        yield batch
@@@

This keeps memory bounded and allows the dataset reader to apply storage-level optimizations.

## 51. Batch Validation

Validate each batch for:

- required columns;
- null constraints;
- type assumptions;
- business-key presence;
- allowed ranges;
- source partition consistency.

Do not assume that a valid Parquet footer means every business record is correct.

## 52. Batch to Database

Convert only the required fields and stage in bounded transactions.

Conceptually:

@@@
for batch in scanner.to_batches():
    rows = normalize(batch)
    insert_batch(rows)
    commit()
    checkpoint()
@@@

Checkpoint only after the corresponding transaction is durable.

## 53. Failure Drill — Missing Column

Remove a required column from one source file.

Expected:

- schema validation fails before large extraction;
- the file is identified;
- the failure is categorized;
- no false zero-column result is treated as success;
- the file is quarantined or the dataset run fails.

## 54. Failure Drill — Corrupt File

Damage a Parquet footer or column chunk.

Expected:

- reader failure is detected;
- source file is identified;
- the file is not marked successfully processed;
- previously committed files remain safe;
- retry occurs only after the source artifact is corrected or replaced.

## 55. Failure Drill — Partition Mismatch

Place a file containing day 25 data under a day=26 partition.

Expected:

- path metadata and data are compared where required;
- mismatch is detected;
- downstream state is not silently corrupted.

## 56. Failure Drill — Replay

Process the same file twice.

Expected:

- same source identity;
- same technical record identity;
- no duplicate staging rows;
- second run is safe.

## 57. Failure Drill — Database Failure

Stop the database during a batch.

Verify:

1. the transaction rolls back;
2. the checkpoint does not advance;
3. retry is safe;
4. committed data remains;
5. duplicate rows are not created.

## 58. Failure Drill — Schema Evolution

Add an optional column.

Expected:

- extraction behavior follows the compatibility contract;
- optional data can be preserved;
- existing required columns continue to validate.

Then change a required column from numeric to string.

Expected:

- the breaking change is detected;
- the file or dataset is not silently coerced into an unsafe representation.

## 59. Observability

Track:

@@@
parquet_files_discovered_total
parquet_files_completed_total
parquet_files_failed_total
parquet_rows_seen_total
parquet_rows_staged_total
parquet_bytes_read_total
parquet_extraction_duration_seconds
parquet_schema_failures_total
parquet_read_failures_total
parquet_duplicate_rows_total
@@@

Useful dimensions:

- source;
- dataset;
- partition;
- schema version;
- extractor version;
- failure category.

Avoid high-cardinality payload values in metrics.

## 60. Structured Logging

Example:

@@@
{
  "event": "parquet_file_completed",
  "source_file": "part-0003.parquet",
  "source_hash": "abc123",
  "partition": "2026-09-26",
  "rows_seen": 500000,
  "rows_staged": 500000,
  "extractor_version": "1.2.0"
}
@@@

Record enough metadata to reproduce the extraction decision without logging entire datasets.

## 61. Performance Diagnostics

When Parquet extraction is slow, inspect in this order:

1. Are unnecessary columns being read?
2. Are filters pushed into the dataset reader?
3. Are partitions being pruned?
4. Are row groups being skipped?
5. Are there too many small files?
6. Is the database the actual bottleneck?
7. Is the object store the bottleneck?
8. Is the file layout aligned with query patterns?

Do not optimize Python loops before checking storage and database I/O.

## 62. Production Tools You Should Know

### 62.1 PyArrow Parquet

Use for:

- reading Parquet files;
- inspecting schema and metadata;
- row-group inspection;
- column projection.

### 62.2 PyArrow Dataset

Use for:

- partitioned datasets;
- predicate pushdown;
- partition pruning;
- batch scanning.

### 62.3 DuckDB

Use for local and analytical Parquet inspection when SQL is useful.

Example:

@@@
SELECT payment_id, amount
FROM read_parquet('payments/*.parquet')
WHERE currency = 'EUR';
@@@

DuckDB is particularly useful for validating a dataset before wiring it into a larger ETL pipeline.

## 63. Common Mistakes

### Mistake 1 — Reading every column

**Fix:** project only required columns.

### Mistake 2 — Filtering after materializing the whole dataset

**Fix:** push predicates into the dataset reader.

### Mistake 3 — Ignoring partition pruning

**Fix:** use partition-aware dataset reads.

### Mistake 4 — Treating row groups as business records

**Fix:** understand row groups as storage units.

### Mistake 5 — Assuming all files have identical schemas

**Fix:** inspect and enforce schema compatibility.

### Mistake 6 — Ignoring small files

**Fix:** inspect file counts and layout before blaming the parser.

### Mistake 7 — Loading the entire dataset into memory

**Fix:** scan record batches.

### Mistake 8 — Assuming Parquet guarantees business correctness

**Fix:** apply business validation after reading.

### Mistake 9 — Assuming path partitions are correct

**Fix:** validate partition metadata when required.

### Mistake 10 — No idempotency

**Fix:** preserve file and record identity.

## 64. Production Implementation Sequence

### Step 1
Inspect the dataset layout.

### Step 2
Identify partition columns.

### Step 3
Inspect the physical schema.

### Step 4
Define required and optional columns.

### Step 5
Define type and null contracts.

### Step 6
Define business filters.

### Step 7
Define source and record identity.

### Step 8
Validate file readability and metadata.

### Step 9
Implement column projection.

### Step 10
Implement predicate pushdown.

### Step 11
Implement partition pruning.

### Step 12
Process batches instead of one giant table.

### Step 13
Implement idempotent staging.

### Step 14
Implement file-level state.

### Step 15
Implement checkpoints where needed.

### Step 16
Add reconciliation and observability.

### Step 17
Test schema evolution and corrupted files.

### Step 18
Benchmark realistic workloads.

### Step 19
Document the recovery runbook.

## 65. Production Checklist

- [ ] Dataset layout documented.
- [ ] Partition strategy documented.
- [ ] Required columns documented.
- [ ] Optional columns documented.
- [ ] Physical and logical types validated.
- [ ] Null semantics defined.
- [ ] Column projection implemented.
- [ ] Predicate pushdown implemented where useful.
- [ ] Partition pruning implemented.
- [ ] Large datasets scanned in batches.
- [ ] Row-group behavior understood.
- [ ] Small-file risk assessed.
- [ ] Source file identity preserved.
- [ ] Record identity defined.
- [ ] Idempotent staging implemented.
- [ ] Schema evolution policy documented.
- [ ] Corrupt-file handling implemented.
- [ ] Partition correctness validated where needed.
- [ ] Transaction policy documented.
- [ ] Checkpoint policy documented.
- [ ] Reconciliation implemented.
- [ ] Metrics and logs exist.
- [ ] Performance tests exist.
- [ ] Recovery runbook exists.

## 66. Definition of Done

E58 is complete when you can independently:

1. explain why Parquet is columnar;
2. inspect Parquet metadata and schema;
3. select only required columns;
4. explain predicate pushdown;
5. explain row-group statistics;
6. use partition pruning;
7. read partitioned datasets;
8. process large datasets in batches;
9. handle schema evolution;
10. validate decimal and timestamp semantics;
11. distinguish missing columns from null values;
12. detect corrupt files;
13. preserve file-level lineage;
14. create deterministic record identity;
15. stage Parquet data idempotently;
16. checkpoint safely after durable commits;
17. diagnose small-file and layout problems;
18. test replay and database recovery;
19. reconcile dataset completeness;
20. optimize reads without changing business semantics.

## 67. What You Learned

Parquet extraction is primarily an exercise in minimizing unnecessary I/O while preserving correctness.

@@@
PARQUET DATASET
      |
      v
PARTITION PRUNING
      |
      v
FILE SELECTION
      |
      v
ROW-GROUP SKIPPING
      |
      v
COLUMN PROJECTION
      |
      v
PREDICATE PUSHDOWN
      |
      v
RECORD BATCHES
      |
      v
VALIDATE + STAGE
      |
      v
OBSERVE + RECOVER
@@@

The central rule is simple: read the minimum data required, but never sacrifice source identity, schema validation, deterministic replay, or business correctness for performance.

## 68. Next Recipe

**E59 — Avro Extraction**

The next recipe focuses on Avro schemas, schema fingerprints, binary decoding, schema evolution, unions, logical types, and ETL staging.