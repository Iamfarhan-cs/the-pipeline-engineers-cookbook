# E05 — Extract Data from CSV Files

CSV files look simple because they are human-readable. In production pipelines, CSV extraction is a file-contract and data-quality problem.

A file can exist while still being:

- incomplete
- partially uploaded
- incorrectly encoded
- structurally malformed
- duplicated
- truncated
- schema-incompatible
- incorrectly delimited
- too large to load into memory

The goal is:

> Extract CSV data only after proving that the file is complete, readable, structurally valid, and safe for downstream processing.

## 1. Problem Recognition

Common production failures include:

- reading a file while it is still being uploaded
- processing the same file twice
- missing an expected file
- incorrect delimiter
- inconsistent column counts
- broken quoting
- unexpected encoding
- malformed rows
- embedded newlines
- duplicate records
- missing required columns
- unexpected columns
- huge files exhausting memory
- file replacement after discovery
- partially written files appearing valid

Do not equate:

    file exists

with:

    file is ready to process

## 2. CSV Extraction Architecture

    FILE ARRIVES
        ↓
    DISCOVER
        ↓
    DETERMINE COMPLETENESS
        ↓
    VALIDATE FILE
        ↓
    VALIDATE HEADER / SCHEMA
        ↓
    STREAM ROWS
        ↓
    VALIDATE RECORDS
        ↓
    PERSIST
        ↓
    CHECKPOINT
        ↓
    MARK FILE PROCESSED

Each stage answers a different question.

## 3. File Discovery

Never assume the filename is the only identity.

A useful file identity can include:

    source
    filename
    path
    file_size
    modification_time
    checksum
    discovered_at

Example:

    payments_2026-09-26.csv

File discovery should record what was found before processing begins.

## 4. File Completeness

A file may appear before the producer has finished writing it.

Unsafe:

    upload starts
       ↓
    file appears
       ↓
    extractor immediately reads it

Safer approaches include:

- producer writes to a temporary name and renames when complete
- completion marker files
- manifest files
- stable file size checks
- source-specific completion signals

The strongest solution is normally a producer-defined completion contract.

## 5. Temporary Filename Pattern

A producer can write:

    payments.csv.tmp

and after successful completion:

    payments.csv

The extractor watches only for the final name.

This creates a simple contract:

    .tmp → being written
    final name → ready

Do not invent this contract if the producer does not support it. Agree on the file-arrival protocol first.

## 6. File Stability Checks

If no completion marker exists, a temporary heuristic is checking that:

    size(t0) == size(t1)

after a suitable interval.

This reduces the risk of reading an actively growing file but does not prove correctness.

A producer could pause during upload and then continue.

Therefore:

> File stability is a heuristic; an explicit completion contract is stronger.

## 7. Filename Conventions

Predictable names make file discovery safer.

Example:

    payments_YYYY-MM-DD.csv

Useful filename components can include:

- source system
- dataset
- extraction date
- partition
- sequence number
- version

Do not rely on filenames alone for uniqueness. A checksum or source identifier can provide stronger identity.

## 8. Encoding

CSV files may use different encodings.

Common examples include:

- UTF-8
- UTF-8 with BOM
- UTF-16
- legacy regional encodings

Prefer a documented encoding contract.

Example:

    UTF-8

Do not silently guess an encoding and continue. A successful decode does not necessarily mean the characters were interpreted correctly.

## 9. Delimiter

CSV does not always mean comma.

Possible delimiters include:

    ,
    ;
    \t
    |

Prefer an explicit source contract.

Automatic delimiter detection can misinterpret unusual data.

## 10. Header Validation

Before processing records, validate the header.

Expected:

    id,amount,currency,status

Validate:

- required columns exist
- duplicate column names are rejected
- column names are normalized according to policy
- unexpected columns are handled explicitly
- column order is understood if positional mapping matters

Do not blindly map columns by position.

## 11. Schema Contract

A CSV extraction contract should define:

    column
    type
    required?
    nullable?
    allowed values
    transformation rule

Example:

    id          integer       required
    amount      decimal       required
    currency    string        required
    status      enum          required
    note        string        optional

The CSV parser should not be responsible for silently inventing business meaning.

## 12. Parsing CSV Correctly

Do not split lines manually:

    line.split(',')

This fails when fields contain:

- commas
- quotes
- embedded newlines

Example valid CSV:

    id,description
    1,"Payment, international"

Use a real CSV parser.

## 13. Quoting and Embedded Newlines

A valid CSV field can contain a newline:

    id,description
    1,"First line
    second line"

Therefore:

> A physical line is not necessarily a CSV record.

Record boundaries must be determined by the CSV parser.

## 14. Python CSV Reader

A basic streaming reader:

    import csv

    with open(
        'payments.csv',
        'r',
        encoding='utf-8',
        newline=''
    ) as file:

        reader = csv.DictReader(file)

        for row in reader:
            process(row)

Using DictReader makes column names explicit and supports streaming rather than requiring the entire file in memory.

## 15. Large CSV Files

Do not automatically use:

    rows = list(reader)

for a multi-gigabyte file.

Prefer:

    READ ROW
       ↓
    VALIDATE
       ↓
    BUFFER BATCH
       ↓
    PERSIST
       ↓
    NEXT ROW

Memory should remain bounded.

## 16. Batch Processing

For large files, group records into bounded batches.

Example concept:

    batch_size = 10_000

    for row in reader:
        batch.append(row)

        if len(batch) == batch_size:
            persist(batch)
            batch.clear()

Persist the final partial batch after the loop.

## 17. Row Validation

Validate required fields before persistence.

Example:

    if not row['id']:
        reject(row, 'missing id')

    if not row['currency']:
        reject(row, 'missing currency')

Validation can include:

- required fields
- type conversion
- numeric ranges
- allowed values
- date format
- identifier format

Keep structural parsing separate from business validation.

## 18. Malformed Rows

A malformed row should not necessarily destroy an otherwise valid file.

Define a policy:

    malformed row
         ↓
    classify
         ↓
    quarantine / reject
         ↓
    continue or fail file

The correct choice depends on the source contract.

For critical financial extracts, a small number of malformed rows may justify rejecting the entire file.

Do not silently skip malformed records.

## 19. Error Thresholds

Some pipelines use an explicit tolerance:

    max_bad_rows = 0

or:

    max_bad_row_ratio = 0.01

If the threshold is exceeded:

    STOP FILE
       ↓
    QUARANTINE
       ↓
    ALERT

The threshold must be part of the contract, not an undocumented developer preference.

## 20. Duplicate Files

The same physical file may arrive more than once.

Examples:

- producer retry
- SFTP re-upload
- manual resend
- storage copy
- scheduler retry

Possible file identity:

    source + filename + checksum

Record processed-file state before accepting a duplicate as new work.

## 21. Duplicate Records

Different files can contain the same logical records.

Example:

    file_001.csv → payment 1001
    file_002.csv → payment 1001

File-level deduplication does not solve record-level duplication.

Use a stable business or source identifier where available.

## 22. Checkpointing

For large files, checkpointing can record progress.

Possible checkpoint data:

    file_id
    checksum
    byte_position
    rows_processed
    last_source_id
    batch_number

However, byte offsets are only safe under the correct file and parser assumptions.

If the file changes, the checkpoint must not blindly resume.

Always bind checkpoint state to file identity, preferably including a checksum or immutable source identifier.

## 23. File Immutability

The safest model is:

    DISCOVER FILE
         ↓
    VERIFY IDENTITY
         ↓
    PROCESS IMMUTABLE FILE

If a producer can modify a file after publication, extraction becomes harder to reason about.

Prefer immutable published files.

## 24. Checksums

A checksum can identify file content.

Example concept:

    SHA-256(file bytes)

Store:

    filename
    size
    checksum
    discovered_at

If the same filename appears with a different checksum, treat it as a different file version or an operational anomaly according to the source contract.

## 25. Compressed CSV

CSV files are often compressed:

    payments.csv.gz

Extraction can stream decompression rather than creating a full uncompressed copy.

Conceptually:

    COMPRESSED FILE
          ↓
    STREAM DECOMPRESSOR
          ↓
    CSV PARSER
          ↓
    VALIDATION
          ↓
    PERSIST

Validate both the archive/file and the CSV content.

## 26. Missing Files

A pipeline may expect:

    payments_2026-09-26.csv

but receive nothing.

Do not automatically interpret missing as empty data.

Distinguish:

    NO FILE ARRIVED
          ≠
    FILE ARRIVED WITH ZERO ROWS

A missing file can represent an operational failure.

## 27. Empty Files

An empty file can be valid or invalid depending on the contract.

Distinguish:

- zero-byte file
- header-only file
- valid file containing zero records

Define the expected behavior explicitly.

## 28. Atomic Publication

A strong producer pattern is:

    WRITE TEMP FILE
         ↓
    FLUSH / CLOSE
         ↓
    VERIFY
         ↓
    ATOMIC RENAME
         ↓
    PUBLISHED FILE

The extractor sees only published files.

This dramatically reduces partial-file ingestion risk.

## 29. Testing

Test at least:

1. Valid CSV.
2. Empty file.
3. Header-only file.
4. Missing required column.
5. Duplicate column.
6. Unexpected column.
7. Wrong delimiter.
8. Wrong encoding.
9. Quoted comma.
10. Embedded newline.
11. Malformed quote.
12. Malformed row.
13. Duplicate file.
14. Duplicate record.
15. Large file.
16. Compressed CSV.
17. File changes after discovery.
18. Missing file.
19. Checkpoint restart.
20. Persistence failure.

## 30. Observability

Useful metrics:

    csv_files_discovered_total
    csv_files_processed_total
    csv_files_failed_total
    csv_files_quarantined_total
    csv_files_duplicated_total
    csv_rows_read_total
    csv_rows_rejected_total
    csv_rows_persisted_total
    csv_parse_errors_total
    csv_validation_errors_total
    csv_processing_duration_seconds
    csv_bytes_processed_total

Useful log fields:

    file_id
    filename
    source
    checksum
    file_size
    row_number
    batch_number
    rows_read
    rows_rejected
    error_type
    extraction_run_id

Do not log complete sensitive rows unnecessarily.

## 31. Intentional Failure

### Failure drill 1 — Process an incomplete file

Start reading a file while another process is still writing it.

Verify that the ingestion contract prevents processing or detects the incomplete state.

### Failure drill 2 — Wrong delimiter

Change comma-separated input to semicolon-separated input.

Verify schema/header validation detects the problem.

### Failure drill 3 — Malformed quoting

Break a quoted field.

Verify the parser fails or quarantines according to policy.

### Failure drill 4 — Duplicate file

Submit the same file twice.

Verify file-level idempotency prevents duplicate processing.

### Failure drill 5 — Persistence failure

Fail downstream persistence halfway through a file.

Restart extraction and verify checkpoint/idempotency behavior.

### Failure drill 6 — File mutation

Modify the file after the initial checkpoint.

Verify the extractor refuses to resume against a changed file.

## 32. Recovery

When CSV extraction fails:

1. Identify the file.
2. Verify filename, size, and checksum.
3. Determine whether publication was complete.
4. Inspect parser and schema errors.
5. Check rejected-row counts.
6. Check persistence state.
7. Read the last valid checkpoint.
8. Verify the source file has not changed.
9. Resume or restart according to the checkpoint policy.
10. Reconcile expected and persisted row counts.
11. Mark the file complete only after verification.

Never resume a changed file using an old checkpoint.

## 33. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Python csv** | Standard-library CSV parser; understand DictReader, quoting, delimiters, and streaming reads. |
| **Pandas** | Useful for tabular CSV processing, especially when datasets fit controlled memory limits. |
| **Polars** | High-performance dataframe engine useful for larger CSV workloads and streaming-oriented processing. |

> These are reference tools for production vocabulary. The underlying file-ingestion mechanics should still be understood independently.

## 34. Production Runbook

### File arrived but cannot be parsed

Check:

1. Encoding.
2. Delimiter.
3. Quoting.
4. Header.
5. File completeness.
6. Source version.

### Expected file did not arrive

Check:

1. Producer job.
2. Arrival window.
3. Filename convention.
4. Source transfer system.
5. Completion marker or manifest.

Do not convert a missing file into an empty dataset without an explicit contract.

### Duplicate file detected

Check:

1. Filename.
2. Checksum.
3. Source delivery history.
4. Processed-file registry.

### File is too large

Check:

1. Streaming parser.
2. Batch size.
3. Memory usage.
4. Compression.
5. Whether parallelism is actually necessary.

### What not to do

Do not:

- read an unverified file immediately after discovery
- split CSV lines manually
- silently guess encoding
- silently skip malformed rows
- treat missing files as zero records
- use an old checkpoint against a changed file
- load multi-gigabyte CSV files entirely into memory
- rely only on filenames for file identity

## 35. Common Mistakes

### Mistake 1 — File existence equals completeness

A file can exist while still being written.

### Mistake 2 — Treating CSV as simple text

Quotes and embedded newlines make manual parsing unsafe.

### Mistake 3 — No schema contract

Columns can change without obvious parser errors.

### Mistake 4 — Silent row rejection

Skipped records create hidden data loss.

### Mistake 5 — File-level deduplication only

Different files can contain the same logical record.

### Mistake 6 — Unbounded memory

Large files can exhaust the extraction worker.

### Mistake 7 — Mutable source files

Changing a file after checkpointing can invalidate recovery.

## 36. Definition of Done

You are done when you can:

- recognize the risks of CSV extraction
- design a file-arrival contract
- distinguish discovery from completeness
- validate file identity
- validate encoding and delimiter
- validate headers and schemas
- correctly parse quoted fields
- handle embedded newlines
- stream large CSV files
- batch records safely
- classify malformed rows
- implement file-level deduplication
- implement record-level deduplication
- detect missing files
- distinguish empty files from missing files
- use checksums
- reason about immutable files
- checkpoint safely
- recover from persistence failure
- test malformed and incomplete files
- observe file and row-level pipeline behavior
- operate CSV extraction with a production runbook

## 37. What You Learned

The central principle is:

> A CSV extractor must prove that the file is complete and structurally valid before treating its rows as data.

A production CSV pipeline follows:

    DISCOVER
       ↓
    VERIFY COMPLETENESS
       ↓
    IDENTIFY FILE
       ↓
    VALIDATE FORMAT
       ↓
    VALIDATE SCHEMA
       ↓
    STREAM + VALIDATE ROWS
       ↓
    PERSIST
       ↓
    CHECKPOINT
       ↓
    MARK COMPLETE

The key question is:

> What evidence proves that this file is the exact, complete source artifact I intended to process?

That question drives file-arrival contracts, checksums, schema validation, streaming, checkpointing, idempotency, testing, and recovery.