# E54 — CSV Extraction — ETL Application

## 1. Problem Recognition

CSV is simple to read and surprisingly easy to ingest incorrectly.

A production CSV extractor must handle:

- large files;
- headers;
- delimiters;
- quoting;
- escaped characters;
- encodings;
- empty fields;
- malformed rows;
- duplicate headers;
- unexpected columns;
- missing columns;
- embedded newlines;
- compressed CSV;
- partial extraction;
- deterministic retries.

The goal is not merely to open a CSV.

```
VALIDATED FILE
      ↓
CSV READER
      ↓
NORMALIZED ROWS
      ↓
STAGING
      ↓
TRANSFORMATION
```

Extraction should preserve source information rather than silently changing its meaning.

---

## 2. Extraction Boundary

E53 established that the file passed file validation.

E54 interprets the CSV structure.

```
FILE VALIDATION
      ↓
CSV EXTRACTION
      ↓
RAW / STAGING ROWS
      ↓
TRANSFORMATION
```

Do not mix extraction with business transformations unnecessarily.

Extraction asks:

> What records and fields are physically present?

Transformation asks:

> How should those records be represented downstream?

---

## 3. CSV Is a Contract

Define these properties before implementation:

- delimiter;
- quote character;
- escape behavior;
- encoding;
- header presence;
- expected columns;
- line-ending behavior;
- null representation;
- whitespace rules;
- malformed-row policy.

Example:

| Property | Example |
|---|---|
| delimiter | comma |
| quote | double quote |
| encoding | UTF-8 |
| header | yes |
| null token | empty field |
| malformed row | quarantine |
| extra columns | reject |

Do not allow parser defaults to become an undocumented production contract.

---

## 4. Never Parse CSV With split

This is unsafe:

```
line.split(",")
```

It breaks when commas occur inside quoted fields.

Example:

```
1,"Smith, John",100.00
```

A correct CSV parser understands that the comma inside the quoted field is data.

Use Python's csv module or an equivalent standards-aware parser.

---

## 5. Basic Streaming Extraction

```
import csv
from pathlib import Path


def extract_csv(path: Path):
    with path.open(
        "r",
        encoding="utf-8",
        newline="",
    ) as handle:
        reader = csv.DictReader(handle)

        for row in reader:
            yield row
```

Important details:

- explicit encoding;
- newline set to an empty string;
- iterator-based processing;
- no full-file list.

---

## 6. Why newline Matters

Python's CSV parser should normally receive a file opened with newline set to an empty string.

This allows the CSV parser to handle different newline conventions and quoted fields containing embedded newlines.

A physical newline is not necessarily a logical CSV record boundary.

---

## 7. Header Validation

The header defines the mapping between source columns and logical fields.

Example:

```
transaction_id,account_id,amount,currency
tx-001,acc-100,50.25,EUR
```

Validate required columns before processing thousands of rows.

```
def missing_columns(
    observed: list[str],
    required: set[str],
) -> set[str]:
    return required - set(observed)
```

A missing required column should normally block extraction.

---

## 8. Extra Columns

Suppose the source adds:

```
transaction_id,account_id,amount,currency,source_region
```

Possible policies:

- reject;
- ignore;
- preserve;
- route through schema evolution.

Choose explicitly.

For raw/staging pipelines, preserving source fields can be useful because it protects source fidelity.

---

## 9. Headerless CSV

Some sources intentionally omit headers.

If headers are required, reject the file.

If headerless CSV is supported, define the columns explicitly:

```
fieldnames = [
    "transaction_id",
    "account_id",
    "amount",
    "currency",
]
```

Never guess column meanings from observed values.

---

## 10. Delimiters

CSV-like files may use:

```
,
;
|
TAB
```

Configure the delimiter:

```
reader = csv.DictReader(
    handle,
    delimiter=";",
)
```

For critical pipelines, explicit configuration is safer than automatic delimiter detection.

---

## 11. Quoting

Valid CSV can contain quoted delimiters:

```
tx-001,"Smith, John",100.00
```

It can also contain quoted newlines:

```
tx-001,"Customer note
continued on next line",100.00
```

Use a real CSV parser. Do not process physical lines independently.

---

## 12. Escaped Quotes

A standard CSV representation can contain:

```
"Customer ""VIP"" account"
```

The parser should return:

```
Customer "VIP" account
```

Nonstandard escape rules must be part of the source contract.

---

## 13. Encoding

Use an explicit encoding.

```
with path.open(
    "r",
    encoding="utf-8",
    newline="",
) as handle:
    ...
```

Do not rely on the machine's default encoding.

If the producer uses another encoding, configure it explicitly and validate it in E53.

---

## 14. UTF-8 BOM

Some files contain a UTF-8 byte-order mark.

If the contract permits it, Python can consume it with:

```
encoding="utf-8-sig"
```

Do not silently switch encoding because one file looks unusual. Make the behavior part of the contract.

---

## 15. Null Representation

CSV has no universal null type.

These values can have different meanings:

```
""
NULL
null
N/A
-
0
```

Extraction should normally preserve source text.

Convert values to null only when the source contract defines the representation.

---

## 16. Whitespace

Consider:

```
" EUR "
```

Should it become EUR or remain exactly as received?

Extraction should generally preserve source values.

Whitespace normalization belongs in a documented transformation rule unless the source contract requires trimming.

---

## 17. Type Conversion

CSV values begin as text.

For example:

```
"100.25"
```

may later become a decimal value.

A clean boundary is:

```
CSV TEXT
   ↓
EXTRACT
   ↓
RAW/STAGING
   ↓
TYPE NORMALIZATION
```

If immediate conversion is required, make it explicit and observable.

---

## 18. Financial Numbers

Do not use binary floating-point for financial amounts merely because CSV contains numeric text.

Prefer Decimal when converting:

```
from decimal import Decimal

amount = Decimal("100.25")
```

Keep extraction and financial transformation separate where practical.

---

## 19. Date and Timestamp Fields

Values such as:

```
2026-09-26T14:30:00Z
```

must not be interpreted using ambiguous local-time assumptions.

Preserve raw values at the extraction boundary when possible and parse according to the source contract.

Timezone handling must be explicit.

---

## 20. Source Row Numbers

Every extracted row should remain traceable.

```
def extract_csv_with_row_numbers(path: Path):
    with path.open(
        "r",
        encoding="utf-8",
        newline="",
    ) as handle:
        reader = csv.DictReader(handle)

        for row_number, row in enumerate(
            reader,
            start=2,
        ):
            yield row_number, row
```

Starting at 2 assumes row 1 is the header.

Row numbers are essential for diagnosing malformed records.

---

## 21. Preserve File Lineage

Attach source metadata to extracted rows.

Useful fields:

```
file_id
delivery_id
source_system
dataset
business_date
source_row_number
extracted_at
```

Do not lose lineage when moving from files into staging.

---

## 22. Streaming Large CSV Files

Never do this for a large source:

```
rows = list(csv.DictReader(handle))
```

That makes memory proportional to total rows.

Prefer:

```
for row in reader:
    process(row)
```

Memory then remains approximately proportional to the active batch.

---

## 23. Batch Processing

Database writes should normally be batched.

```
batch = []

for row in reader:
    batch.append(row)

    if len(batch) == 5000:
        write_batch(batch)
        batch.clear()

if batch:
    write_batch(batch)
```

Tune batch size according to:

- row width;
- database capacity;
- transaction size;
- memory;
- throughput;
- recovery requirements.

---

## 24. Staging Table

A useful staging boundary:

```
CREATE TABLE csv_staging (
    file_id UUID NOT NULL,
    source_row_number BIGINT NOT NULL,
    raw_row JSONB NOT NULL,
    extracted_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (
        file_id,
        source_row_number
    )
);
```

This preserves raw row representation while providing a database boundary for downstream processing.

---

## 25. Idempotent Extraction

Retries must not create duplicate staging rows.

A useful logical key is:

```
(file_id, source_row_number)
```

Example:

```
INSERT INTO csv_staging (
    file_id,
    source_row_number,
    raw_row
)
VALUES (
    :file_id,
    :row_number,
    :raw_row
)
ON CONFLICT (
    file_id,
    source_row_number
)
DO NOTHING;
```

This makes replay safer.

---

## 26. Do Not Use Business IDs Alone

A transaction ID may not be globally unique across files.

For example:

```
transaction_id = tx-001
```

could occur in:

- a correction file;
- a replay;
- another business date;
- another source.

Source lineage should remain part of the extraction identity.

---

## 27. Malformed Rows

CSV can contain:

- too few fields;
- too many fields;
- invalid quoting;
- invalid encoding;
- malformed structure.

Define a policy.

### Reject entire file

Use when one malformed row makes the delivery untrustworthy.

### Reject individual rows

Use only when partial acceptance is allowed.

### Quarantine malformed rows

Use when investigation and replay are required.

Never silently discard malformed records.

---

## 28. Strict Parsing

Python's CSV reader supports strict parsing.

```
reader = csv.DictReader(
    handle,
    strict=True,
)
```

Use strict parsing when malformed structure should stop extraction.

The correct strictness level belongs to the producer contract.

---

## 29. Row Width

For a four-column contract:

```
expected = 4
observed = 5
```

Possible result:

```
ROW_WIDTH_MISMATCH
```

Do not silently discard the fifth field.

Similarly, do not silently fill missing fields unless the contract allows it.

---

## 30. Extraction Versus Data Quality

A value can be syntactically valid CSV but fail a business rule.

Example:

```
amount = -500
```

The CSV parser can successfully extract it.

Whether the amount is valid belongs to data-quality or business validation.

Keep the boundaries clear:

```
CSV PARSER
    ↓
STRUCTURALLY VALID ROW
    ↓
DATA QUALITY
    ↓
CURATED DATA
```

---

## 31. Error Classification

Classify extraction failures.

### File-level

- cannot open;
- invalid encoding;
- malformed CSV;
- invalid compression.

### Row-level

- wrong width;
- invalid row structure.

### Infrastructure-level

- database unavailable;
- object-store timeout;
- disk full.

Different failure classes require different recovery policies.

---

## 32. Checkpointing

Very large files may need progress checkpoints.

Possible state:

```
file_id
last_successful_row
rows_extracted
updated_at
```

Checkpointing can support restart after a failure.

Only resume against the same immutable file version and the same deterministic parser configuration.

---

## 33. Deterministic Extraction

For the same immutable file and parser contract:

```
same input
+
same configuration
=
same extracted rows
```

Avoid:

- random values in extracted records;
- implicit locale behavior;
- current-time-dependent parsing;
- changing parser rules during a retry.

Determinism makes reconciliation and replay much easier.

---

## 34. CSV Dialect

A source-specific dialect can make the contract explicit.

```
class PaymentsDialect(csv.Dialect):
    delimiter = ","
    quotechar = '"'
    doublequote = True
    skipinitialspace = False
    lineterminator = "\n"
    quoting = csv.QUOTE_MINIMAL
```

Treat the dialect as configuration, not an incidental parser preference.

---

## 35. Object Storage Extraction

CSV extraction does not require a full local copy.

```
OBJECT STORAGE
      ↓
STREAM
      ↓
CSV READER
      ↓
BATCH
      ↓
STAGING
```

The object-storage client should expose a stream-like interface.

Object identity and transport retries belong to the storage layer.

---

## 36. Compressed CSV

A common source is:

```
payments.csv.gz
```

The streaming flow is:

```
OBJECT
  ↓
GZIP STREAM
  ↓
TEXT DECODER
  ↓
CSV READER
  ↓
ROWS
```

Avoid creating a full uncompressed copy unless there is a specific operational reason.

E60 covers compressed-file extraction in detail.

---

## 37. Gzip CSV Example

```
import csv
import gzip


def extract_gzip_csv(path: Path):
    with gzip.open(
        path,
        "rt",
        encoding="utf-8",
        newline="",
    ) as handle:
        reader = csv.DictReader(handle)

        for row in reader:
            yield row
```

The same streaming principle applies.

---

## 38. Extracted Row Model

Make lineage explicit.

```
from dataclasses import dataclass


@dataclass(frozen=True)
class ExtractedRow:
    file_id: str
    source_row_number: int
    values: dict[str, str | None]
```

Then:

```
def extract_rows(path: Path, file_id: str):
    with path.open(
        "r",
        encoding="utf-8",
        newline="",
    ) as handle:
        reader = csv.DictReader(handle)

        for row_number, row in enumerate(
            reader,
            start=2,
        ):
            yield ExtractedRow(
                file_id=file_id,
                source_row_number=row_number,
                values=row,
            )
```

This makes downstream lineage explicit.

---

## 39. Batch Database Writer

A simple staging writer:

```
def write_batch(cursor, rows):
    for row in rows:
        cursor.execute(
            """
            INSERT INTO csv_staging (
                file_id,
                source_row_number,
                raw_row
            )
            VALUES (%s, %s, %s)
            ON CONFLICT (
                file_id,
                source_row_number
            )
            DO NOTHING
            """,
            (
                row.file_id,
                row.source_row_number,
                row.values,
            ),
        )
```

For higher throughput, use PostgreSQL bulk-loading facilities where appropriate.

The logical requirement remains idempotent staging.

---

## 40. Transaction Boundaries

Common strategies:

### One transaction per file

Simple and atomic, but potentially huge.

### One transaction per batch

More scalable and easier to recover, but partial progress can be committed.

### Hybrid

Commit batches while maintaining durable extraction state.

Choose according to:

- file size;
- database capacity;
- recovery requirements;
- replay policy.

---

## 41. Extraction State

A useful state model:

```
VALIDATED
    ↓
EXTRACTING
    ├── EXTRACTED
    └── EXTRACTION_FAILED
```

For checkpointed extraction:

```
EXTRACTING
    ↓
BATCH COMMITTED
    ↓
CHECKPOINT UPDATED
    ↓
NEXT BATCH
```

Make restart behavior explicit.

---

## 42. Observability

Track:

```
csv_extraction_started_total
csv_extraction_completed_total
csv_extraction_failed_total
csv_rows_extracted_total
csv_rows_rejected_total
csv_extraction_duration_seconds
csv_extraction_bytes_total
csv_extraction_batch_total
```

Useful structured fields:

```
file_id
delivery_id
source_system
dataset
business_date
row_number
parser_version
delimiter
encoding
status
```

Do not use row numbers or file IDs as metric labels.

---

## 43. Logging

Useful events:

```
csv_extraction_started
csv_header_validated
csv_batch_committed
csv_extraction_completed
csv_extraction_failed
csv_row_rejected
```

Log metadata rather than sensitive row contents.

For rejected rows, preserve:

- file ID;
- row number;
- error code;
- parser version.

Store rejected payloads only through an approved controlled mechanism.

---

## 44. Testing

Test at least:

- normal CSV;
- quoted commas;
- embedded newlines;
- escaped quotes;
- semicolon delimiter;
- tab delimiter;
- UTF-8;
- UTF-8 BOM;
- invalid encoding;
- missing header;
- extra columns;
- missing columns;
- malformed quoting;
- empty file;
- header-only file;
- duplicate rows;
- large file;
- compressed CSV;
- retry after partial extraction.

---

## 45. Unit Test

```
def test_csv_preserves_quoted_comma(tmp_path):
    path = tmp_path / "input.csv"

    path.write_text(
        'id,name\n'
        '1,"Smith, John"\n',
        encoding="utf-8",
    )

    rows = list(extract_csv(path))

    assert rows[0]["name"] == "Smith, John"
```

This catches the common split-based parsing mistake.

---

## 46. Malformed CSV Test

Create malformed input according to the parser's strictness policy.

Verify:

```
malformed input
      ↓
controlled exception
      ↓
EXTRACTION_FAILED
      ↓
recovery / quarantine
```

A parser exception must not leave the file permanently stuck in EXTRACTING.

---

## 47. Failure Recovery

If extraction fails:

1. preserve the source artifact;
2. record the error code;
3. determine retryability;
4. roll back or preserve the current batch according to transaction policy;
5. update extraction state;
6. retry transient failures;
7. quarantine permanent format failures;
8. reconcile staging after restart.

The source file should remain immutable.

---

## 48. Intentional Failure Drills

### Drill 1 — Quoted comma

Verify the parser returns one field.

### Drill 2 — Embedded newline

Verify one logical record can span multiple physical lines.

### Drill 3 — Invalid encoding

Verify extraction fails with a stable encoding error.

### Drill 4 — Missing required column

Verify extraction stops before processing rows.

### Drill 5 — Extra column

Verify the configured schema policy is applied.

### Drill 6 — Malformed quoting

Verify the configured reject/quarantine policy.

### Drill 7 — Large CSV

Verify memory remains bounded.

### Drill 8 — Duplicate extraction

Run extraction twice and verify staging remains idempotent.

### Drill 9 — Mid-file database failure

Verify restart behavior after several committed batches.

### Drill 10 — Compressed CSV

Verify streaming decompression without an unnecessary full uncompressed copy.

---

## 49. Recovery Runbook

### CSV parser fails

1. Identify the file and row location.
2. Inspect the parser error.
3. Confirm the source artifact is unchanged.
4. Determine whether the failure is format-level or transient.
5. Quarantine permanent format failures.
6. Retry transient infrastructure failures.

### Missing required column

1. Compare observed and expected headers.
2. Check source schema version.
3. Determine whether the producer changed its contract.
4. Do not silently map a different column.
5. Update parser support through a controlled change.
6. Reprocess the original immutable file.

### Partial staging exists

1. Identify the last committed batch.
2. Check the idempotency key.
3. Resume or replay according to checkpoint policy.
4. Verify row counts.
5. Confirm no duplicate logical rows exist.

### Malformed row

1. Apply the configured row-error policy.
2. Preserve row number and error code.
3. Quarantine the row or entire file as required.
4. Never silently discard it.

---

## 50. Production Tools You Should Know

### 1. Python csv

Know DictReader, dialects, quoting, strict parsing, newline handling, and streaming iteration.

### 2. pandas

Useful for analytical and moderate-sized CSV processing. Understand memory behavior before using it for large production ingestion.

### 3. PostgreSQL COPY

Learn bulk CSV loading, transaction behavior, staging tables, and high-throughput ingestion.

The underlying mechanism remains:

```
VALIDATED CSV
      ↓
STREAM PARSER
      ↓
ROW LINEAGE
      ↓
BATCH
      ↓
IDEMPOTENT STAGING
```

---

## 51. Common Mistakes

1. Using split(",").
2. Loading the entire CSV into memory.
3. Relying on platform-default encoding.
4. Ignoring newline handling.
5. Guessing delimiters in critical pipelines.
6. Treating physical lines as logical records.
7. Silently dropping extra columns.
8. Silently filling missing columns.
9. Converting all values during extraction without a contract.
10. Losing source row numbers.
11. Losing file identity.
12. Using only business IDs as extraction keys.
13. Retrying permanent parser errors forever.
14. Leaving files stuck in EXTRACTING.
15. Creating duplicate staging rows on replay.
16. Logging sensitive row contents.
17. Using unbounded row-level metric labels.
18. Decompressing huge files fully to disk unnecessarily.
19. Treating CSV extraction as data-quality validation.
20. Changing parser behavior without versioning the extraction contract.

---

## 52. Definition of Done

You can independently:

- define a CSV extraction contract;
- configure delimiters and quoting;
- handle headers;
- validate required and extra columns;
- correctly parse quoted commas;
- correctly parse embedded newlines;
- handle encoding explicitly;
- handle BOMs intentionally;
- preserve raw source values;
- track source row numbers;
- attach file and delivery lineage;
- stream large CSV files;
- batch extracted rows;
- write idempotently to staging;
- classify row and file failures;
- checkpoint large-file extraction when required;
- handle compressed CSV;
- test parser edge cases;
- observe extraction health;
- recover failed extraction safely.

---

## 53. What You Learned

> **CSV extraction is a parsing contract, not a string-splitting operation.**

The production pattern is:

```
VALIDATED CSV
      ↓
READ WITH EXPLICIT CONTRACT
      ↓
VALIDATE HEADER
      ↓
STREAM LOGICAL RECORDS
      ↓
ATTACH SOURCE LINEAGE
      ↓
BATCH
      ↓
IDEMPOTENT STAGING
      ↓
TRANSFORMATION / DATA QUALITY
```

The key question is:

> **Can I extract every logical CSV record correctly, preserve where it came from, keep memory bounded, and safely replay the extraction when something fails?**

That is the foundation of reliable CSV ingestion.
