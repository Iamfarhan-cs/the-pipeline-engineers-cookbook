# E53 — File Validation — ETL Application

## 1. Problem Recognition

E51 established that a delivery arrived.

E52 established that the expected delivery set is complete.

The next question is:

> **Are the delivered files actually safe and valid enough to enter extraction?**

A file can:

- arrive on time;
- belong to a complete delivery;
- have the correct filename;

and still be unusable.

Examples:

```
0-byte file
wrong compression format
corrupted archive
unexpected encoding
wrong delimiter
missing header
duplicate columns
truncated final record
invalid checksum
unsupported schema version
```

A production ETL pipeline therefore needs a validation boundary before parsing data into downstream staging tables.

---

## 2. Validation Is a Gate

The file lifecycle should look like:

```
DISCOVER
   ↓
ARRIVAL DETECTION
   ↓
COMPLETENESS DETECTION
   ↓
FILE VALIDATION
   ├── VALID
   │      ↓
   │   EXTRACT
   │
   └── INVALID
          ↓
       QUARANTINE
```

Validation is not merely logging.

It decides whether the file is allowed to cross the extraction boundary.

---

## 3. What File Validation Should Prove

Before extraction, the pipeline should be able to answer:

1. Does the file exist?
2. Is it the expected object?
3. Is it readable?
4. Is it non-empty when required?
5. Is the size plausible?
6. Does the extension match the expected format?
7. Does the actual format match the declared format?
8. Is compression valid?
9. Is the checksum correct when one is supplied?
10. Is the encoding supported?
11. Is the structural envelope valid?
12. Is the header valid?
13. Are required columns present?
14. Are duplicate columns absent?
15. Is the schema/version supported?
16. Is the file stable and complete enough to read?
17. Has this exact artifact already been accepted?

Not every source needs every check.

The source contract determines which validations are mandatory.

---

## 4. Validation Layers

Separate validation into layers.

### Layer 1 — Physical validation

Checks the artifact itself.

```
exists
readable
non-zero size
stable
not truncated at transport layer
```

### Layer 2 — Identity validation

Checks whether the file is the artifact the pipeline expected.

```
source
dataset
business date
sequence
filename contract
delivery membership
```

### Layer 3 — Integrity validation

Checks whether bytes are intact.

```
checksum
compression integrity
content hash
archive integrity
```

### Layer 4 — Format validation

Checks whether the content has the expected structural format.

```
CSV
JSON
JSONL
XML
Parquet
Avro
```

### Layer 5 — Schema-envelope validation

Checks lightweight structural expectations before full transformation.

```
header
required fields
column names
schema version
record envelope
```

Keep business-rule validation separate.

---

## 5. File Validation Versus Data Quality

Do not confuse file validation with downstream data quality.

### File validation

```
Can this artifact safely enter extraction?
```

Examples:

- readable file;
- correct format;
- valid archive;
- required header exists.

### Data quality

```
Are the extracted records acceptable for the business?
```

Examples:

- transaction amount must be positive;
- country code must be recognized;
- account ID must exist;
- event timestamp must be within a valid range.

The first protects the extraction boundary.

The second protects data correctness.

---

## 6. Validation Contract

Represent validation rules as configuration where practical.

Example:

```
source: bank_a
dataset: payments
format: csv
compression: gzip
encoding: utf-8
allow_empty: false
min_size_bytes: 100
max_size_bytes: 5000000000
required_columns:
  - transaction_id
  - account_id
  - amount
  - currency
  - occurred_at
checksum: sha256
```

This makes source-specific assumptions explicit.

Avoid scattering these rules across extraction code.

---

## 7. File Metadata Model

Create a typed representation.

```
from dataclasses import dataclass
from pathlib import Path


@dataclass(frozen=True)
class FileMetadata:
    path: Path
    size_bytes: int
    suffix: str
    content_hash: str | None
```

A richer implementation may also contain:

- source;
- dataset;
- delivery ID;
- business date;
- sequence;
- modification time;
- object version;
- storage URI.

---

## 8. Validation Result Model

Do not return only True or False.

Operators need to know why validation failed.

```
from dataclasses import dataclass


@dataclass(frozen=True)
class ValidationIssue:
    code: str
    message: str


@dataclass(frozen=True)
class ValidationResult:
    valid: bool
    issues: tuple[ValidationIssue, ...]
```

Example result:

```
ValidationResult(
    valid=False,
    issues=(
        ValidationIssue(
            code="EMPTY_FILE",
            message="File size is zero bytes",
        ),
    ),
)
```

Machine-readable error codes are valuable for metrics and runbooks.

---

## 9. Check File Existence

For local files:

```
from pathlib import Path


def validate_exists(path: Path) -> list[ValidationIssue]:
    if not path.exists():
        return [
            ValidationIssue(
                code="FILE_NOT_FOUND",
                message=f"File does not exist: {path}",
            )
        ]

    return []
```

For object storage or SFTP, existence checks use the transport API instead.

Do not assume local filesystem semantics apply to every source.

---

## 10. Check File Type

A directory should not accidentally pass as an input file.

```
def validate_regular_file(
    path: Path,
) -> list[ValidationIssue]:
    if not path.is_file():
        return [
            ValidationIssue(
                code="NOT_REGULAR_FILE",
                message=f"Expected a regular file: {path}",
            )
        ]

    return []
```

For security-sensitive local ingestion, also define a symlink policy.

---

## 11. Empty File Detection

A zero-byte file may mean:

- valid empty delivery;
- failed producer export;
- interrupted transfer;
- placeholder object;
- upstream bug.

The contract decides whether empty files are valid.

```
def validate_non_empty(
    size_bytes: int,
    allow_empty: bool,
) -> list[ValidationIssue]:
    if size_bytes == 0 and not allow_empty:
        return [
            ValidationIssue(
                code="EMPTY_FILE",
                message="File is empty",
            )
        ]

    return []
```

Never assume zero records and zero bytes mean the same thing.

---

## 12. File Size Bounds

Unexpected file size can detect obvious delivery failures.

Example contract:

```
minimum = 100 bytes
maximum = 5 GB
```

Implementation:

```
def validate_size(
    size_bytes: int,
    minimum: int,
    maximum: int,
) -> list[ValidationIssue]:
    issues = []

    if size_bytes < minimum:
        issues.append(
            ValidationIssue(
                code="FILE_TOO_SMALL",
                message=f"Size {size_bytes} < {minimum}",
            )
        )

    if size_bytes > maximum:
        issues.append(
            ValidationIssue(
                code="FILE_TOO_LARGE",
                message=f"Size {size_bytes} > {maximum}",
            )
        )

    return issues
```

Size thresholds should come from actual source expectations.

---

## 13. Extension Validation

If a source promises CSV:

```
payments_20260926.csv
```

then:

```
payments_20260926.exe
```

should not enter the CSV extractor.

Example:

```
def validate_suffix(
    path: Path,
    allowed_suffixes: set[str],
) -> list[ValidationIssue]:
    if path.suffix.lower() not in allowed_suffixes:
        return [
            ValidationIssue(
                code="UNSUPPORTED_EXTENSION",
                message=f"Unexpected extension: {path.suffix}",
            )
        ]

    return []
```

But extension alone does not prove actual format.

---

## 14. Extension Versus Actual Content

A file named:

```
payments.csv
```

could contain:

```
<html>error page</html>
```

This occurs in real pipelines when downloads return an error document but the pipeline saves it under the expected filename.

Therefore validate both:

```
DECLARED FORMAT
+
OBSERVED STRUCTURE
```

Do not trust extensions blindly.

---

## 15. Magic Bytes

Some formats contain recognizable signatures.

Examples include compressed files and binary formats.

Conceptually:

```
extension = .gz
magic bytes = gzip signature
```

If they disagree:

```
FORMAT_MISMATCH
```

A format-specific validator should inspect the file using the appropriate parser or library rather than maintaining fragile custom signatures for every format.

---

## 16. Compression Integrity

A compressed file can exist and have a plausible size while being corrupted.

For gzip:

```
import gzip
from pathlib import Path


def validate_gzip(path: Path) -> list[ValidationIssue]:
    try:
        with gzip.open(path, "rb") as stream:
            while stream.read(1024 * 1024):
                pass
    except (OSError, EOFError) as exc:
        return [
            ValidationIssue(
                code="INVALID_GZIP",
                message=str(exc),
            )
        ]

    return []
```

For large files, choose validation strategies that do not unnecessarily duplicate full processing.

---

## 17. Archive Safety

Archives introduce additional risks.

Before extracting ZIP/TAR content, validate:

- allowed member count;
- allowed total expanded size;
- allowed extensions;
- path traversal;
- absolute paths;
- symbolic links when relevant;
- nested archive policy.

Never blindly extract an untrusted archive into a production filesystem.

---

## 18. Checksum Validation

If the producer supplies a checksum, verify it before extraction.

Example SHA-256:

```
import hashlib
from pathlib import Path


def sha256_file(
    path: Path,
    chunk_size: int = 1024 * 1024,
) -> str:
    digest = hashlib.sha256()

    with path.open("rb") as stream:
        while chunk := stream.read(chunk_size):
            digest.update(chunk)

    return digest.hexdigest()
```

Then compare:

```
actual_hash == expected_hash
```

A mismatch means the bytes are not the expected artifact.

---

## 19. Hash Files as Streams

Do not load a multi-gigabyte file entirely into memory.

Avoid:

```
data = path.read_bytes()
hashlib.sha256(data)
```

Prefer:

```
READ CHUNK
   ↓
UPDATE HASH
   ↓
READ NEXT CHUNK
```

This keeps memory usage bounded.

---

## 20. Encoding Validation

Text files require an encoding contract.

Common example:

```
UTF-8
```

Validation:

```
def validate_utf8(
    path: Path,
    chunk_size: int = 1024 * 1024,
) -> list[ValidationIssue]:
    import codecs

    decoder = codecs.getincrementaldecoder("utf-8")()

    try:
        with path.open("rb") as stream:
            while chunk := stream.read(chunk_size):
                decoder.decode(chunk)

            decoder.decode(b"", final=True)

    except UnicodeDecodeError as exc:
        return [
            ValidationIssue(
                code="INVALID_ENCODING",
                message=str(exc),
            )
        ]

    return []
```

Incremental decoding avoids loading the entire file into memory.

---

## 21. BOM Handling

Some text files include a byte-order mark.

For UTF-8:

```
UTF-8 BOM
```

Your source contract should decide whether to:

- accept it;
- normalize it;
- reject it.

Do not let BOM handling vary accidentally between environments.

---

## 22. CSV Structural Validation

Before full extraction, validate lightweight structure.

Possible checks:

- file can be opened;
- delimiter is expected;
- header exists;
- required columns exist;
- duplicate headers do not exist;
- header count is plausible.

Example:

```
import csv


def read_csv_header(path):
    with path.open(
        "r",
        encoding="utf-8",
        newline="",
    ) as stream:
        reader = csv.reader(stream)
        return next(reader)
```

Handle empty files explicitly because next(reader) can raise StopIteration.

---

## 23. Required Columns

Example:

```
REQUIRED = {
    "transaction_id",
    "amount",
    "currency",
}
```

Validation:

```
def missing_columns(
    header: list[str],
    required: set[str],
) -> set[str]:
    return required - set(header)
```

If the result is non-empty, the extractor should not silently continue unless schema evolution policy explicitly allows it.

---

## 24. Duplicate Columns

A header like:

```
transaction_id,amount,amount,currency
```

is ambiguous.

Detect it:

```
from collections import Counter


def duplicate_columns(
    columns: list[str],
) -> set[str]:
    counts = Counter(columns)

    return {
        column
        for column, count in counts.items()
        if count > 1
    }
```

Do not allow downstream libraries to silently rename or overwrite duplicate fields without an explicit policy.

---

## 25. JSON Structural Validation

For ordinary JSON:

```
import json


def validate_json(path):
    try:
        with path.open(
            "r",
            encoding="utf-8",
        ) as stream:
            json.load(stream)
    except (
        json.JSONDecodeError,
        UnicodeDecodeError,
    ) as exc:
        return [
            ValidationIssue(
                code="INVALID_JSON",
                message=str(exc),
            )
        ]

    return []
```

For very large JSON documents, use a streaming parser instead of loading the whole document.

---

## 26. JSONL Structural Validation

JSONL should contain one JSON value per line according to the source contract.

A validation approach:

```
import json


def validate_jsonl(path):
    issues = []

    with path.open(
        "r",
        encoding="utf-8",
    ) as stream:
        for line_number, line in enumerate(
            stream,
            start=1,
        ):
            try:
                json.loads(line)
            except json.JSONDecodeError as exc:
                issues.append(
                    ValidationIssue(
                        code="INVALID_JSONL_RECORD",
                        message=(
                            f"line={line_number}: {exc}"
                        ),
                    )
                )

    return issues
```

For production, cap stored error details to avoid enormous validation reports.

---

## 27. XML Structural Validation

XML validation can include:

- well-formed XML;
- expected root element;
- optional XSD validation;
- namespace expectations.

Security matters.

Use XML libraries/configuration appropriate for untrusted input and avoid unsafe external entity processing.

---

## 28. Parquet Validation

For Parquet, validate using a Parquet-aware library.

Typical checks:

- footer readable;
- schema readable;
- required columns exist;
- metadata accessible;
- row-group metadata valid.

Do not attempt to validate Parquet as text.

Format-specific libraries should own format-specific validation.

---

## 29. Avro Validation

For Avro container files, inspect:

- container header;
- embedded schema;
- codec;
- readable blocks;
- required fields/schema compatibility.

Again:

```
FORMAT-SPECIFIC FILE
        ↓
FORMAT-SPECIFIC VALIDATOR
```

Avoid one generic parser that pretends all file formats behave alike.

---

## 30. Schema Version Validation

A producer may declare a version:

```
schema_version = 3
```

The pipeline should know which versions it supports.

Example:

```
SUPPORTED = {2, 3}
```

If:

```
received = 4
```

then:

```
UNSUPPORTED_SCHEMA_VERSION
```

Do not guess how to interpret a new producer schema.

---

## 31. Truncation Detection

Truncation is format-specific.

Possible signals:

- invalid compression footer;
- malformed final JSON structure;
- incomplete XML closing structure;
- unreadable Parquet footer;
- checksum mismatch;
- record count mismatch;
- producer control total mismatch.

Do not invent a universal truncation detector.

Use the strongest evidence available for that source and format.

---

## 32. Stable File Before Validation

Do not validate a file while the producer is still writing it.

A simple local-file stability check may compare:

```
size at T1
mtime at T1

wait

size at T2
mtime at T2
```

If unchanged for the configured stability interval, the file is a candidate for validation.

However:

```
UNCHANGED ≠ GUARANTEED COMPLETE
```

Completeness should still come from E52's contract.

---

## 33. Validation Pipeline

A deterministic validator can run checks in a defined order.

Example:

```
EXISTENCE
   ↓
FILE TYPE
   ↓
SIZE
   ↓
IDENTITY
   ↓
CHECKSUM
   ↓
COMPRESSION
   ↓
ENCODING
   ↓
FORMAT
   ↓
HEADER / SCHEMA ENVELOPE
   ↓
VALID
```

Run cheap checks before expensive checks where possible.

---

## 34. Fail Fast Versus Collect Errors

Two strategies exist.

### Fail fast

Stop on the first blocking validation failure.

Advantages:

- less unnecessary work;
- simple control flow.

### Collect errors

Run all safe independent checks and report multiple issues.

Advantages:

- better diagnosis;
- fewer repair cycles.

A practical design is:

- stop when later checks are unsafe or meaningless;
- otherwise collect independent validation failures.

---

## 35. Blocking Versus Warning Rules

Not every issue must block extraction.

Example:

```
BLOCKING:
checksum mismatch
invalid gzip
missing required column

WARNING:
file larger than historical average
optional column absent
```

Represent severity explicitly.

Do not bury operational policy inside log wording.

---

## 36. Validation Rule Model

Example:

```
from enum import Enum


class Severity(str, Enum):
    WARNING = "WARNING"
    ERROR = "ERROR"
```

Then:

```
@dataclass(frozen=True)
class ValidationIssue:
    code: str
    message: str
    severity: Severity
```

The final decision can be:

```
valid = no ERROR issues
```

---

## 37. Validation Orchestrator

Example:

```
def validate_file(
    path,
    contract,
) -> ValidationResult:
    issues = []

    issues.extend(
        validate_exists(path)
    )

    if issues:
        return ValidationResult(
            valid=False,
            issues=tuple(issues),
        )

    metadata = path.stat()

    issues.extend(
        validate_non_empty(
            metadata.st_size,
            contract.allow_empty,
        )
    )

    issues.extend(
        validate_size(
            metadata.st_size,
            contract.min_size_bytes,
            contract.max_size_bytes,
        )
    )

    issues.extend(
        validate_suffix(
            path,
            contract.allowed_suffixes,
        )
    )

    return ValidationResult(
        valid=not any(
            issue.severity == Severity.ERROR
            for issue in issues
        ),
        issues=tuple(issues),
    )
```

A production implementation should dispatch format-specific validators after the common checks.

---

## 38. Validator Registry

Avoid a giant conditional chain.

Conceptually:

```
VALIDATORS = {
    "csv": validate_csv,
    "json": validate_json,
    "jsonl": validate_jsonl,
    "xml": validate_xml,
    "parquet": validate_parquet,
    "avro": validate_avro,
}
```

Then:

```
validator = VALIDATORS[contract.format]
validator(path, contract)
```

This keeps format behavior isolated and testable.

---

## 39. Persist Validation Results

Do not rely only on logs.

Example table:

```
CREATE TABLE file_validation_run (
    validation_id UUID PRIMARY KEY,
    file_id UUID NOT NULL,
    validator_version TEXT NOT NULL,
    started_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    status TEXT NOT NULL,
    issue_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

And issue table:

```
CREATE TABLE file_validation_issue (
    validation_id UUID NOT NULL,
    issue_code TEXT NOT NULL,
    severity TEXT NOT NULL,
    message TEXT NOT NULL
);
```

This creates an auditable validation history.

---

## 40. Validator Version

Validation logic changes over time.

Store:

```
validator_version
```

Why?

Because the question:

> Why did this file pass last month but fail today?

may depend on which rules were active.

Validation behavior should be reproducible.

---

## 41. File State Transition

A file can move through:

```
DISCOVERED
   ↓
ARRIVED
   ↓
COMPLETE
   ↓
VALIDATING
   ├── VALIDATED
   └── QUARANTINED
```

Use explicit state transitions.

Do not mark a file processed merely because validation succeeded.

Extraction has not happened yet.

---

## 42. Transactional State Update

Example:

```
UPDATE discovered_file
SET
    status = 'VALIDATED',
    validated_at = now(),
    updated_at = now()
WHERE file_id = :file_id
  AND status = 'VALIDATING';
```

The current-state condition protects against invalid transitions.

---

## 43. Quarantine Invalid Files

An invalid file should not disappear.

A quarantine flow:

```
INVALID FILE
    ↓
PRESERVE ORIGINAL
    ↓
RECORD FAILURE
    ↓
QUARANTINE
    ↓
ALERT / INVESTIGATE
```

Preserve:

- file identity;
- source;
- delivery;
- original path;
- quarantine path;
- hash;
- validation issues;
- validation version;
- timestamps.

---

## 44. Quarantine Is Not Deletion

Never implement:

```
if invalid:
    delete(file)
```

unless an explicit retention/security policy requires deletion.

Invalid artifacts are often critical evidence during incident investigation.

Quarantine should isolate them from normal processing while preserving traceability.

---

## 45. Quarantine Path

A deterministic quarantine path can look like:

```
quarantine/
  bank_a/
    payments/
      2026-09-26/
        <file-id>/
          payments_20260926_001.csv
```

Avoid relying only on the original filename because collisions can occur.

---

## 46. Validation Idempotency

Validation should be safe to rerun.

The same:

```
file identity
+
content hash
+
validator version
```

should produce the same deterministic result when external dependencies are unchanged.

Do not mutate the source file during validation.

---

## 47. Revalidation

Revalidation is appropriate when:

- validator logic changed;
- producer corrected metadata;
- an operator explicitly retries;
- a previously unsupported schema is now supported;
- quarantine repair workflow requires it.

Store each validation run rather than silently replacing historical evidence.

---

## 48. Do Not Repair Silently

Suppose a CSV header contains:

```
TransctionId
```

instead of:

```
TransactionId
```

Do not silently rename it unless the source contract contains an explicit normalization rule.

Silent repair can hide upstream contract violations.

Prefer:

```
REJECT
or
NORMALIZE BY DOCUMENTED RULE
```

---

## 49. Security Boundary

Files are external input.

Treat them as untrusted.

Relevant controls include:

- path normalization;
- archive traversal protection;
- resource limits;
- parser hardening;
- decompression limits;
- XML security;
- extension allowlists;
- size limits;
- isolated quarantine.

Validation is part of the pipeline's security boundary.

---

## 50. Decompression Bomb Protection

A tiny compressed file can expand dramatically.

Track or limit:

```
compressed_size
expanded_size
member_count
compression_ratio
```

Define safe thresholds for the environment.

Do not fully expand arbitrary archives without resource controls.

---

## 51. Validation Timeouts

Malformed files can cause expensive parser behavior.

Production validators should have operational limits such as:

- execution timeout;
- memory limit;
- maximum archive members;
- maximum nested depth;
- maximum error count.

A validator that can run forever is itself a pipeline failure mode.

---

## 52. Error Detail Limits

A corrupt JSONL file could contain millions of invalid records.

Do not store millions of validation messages.

Example policy:

```
total_errors = 2,430,901
stored_examples = first 100
```

Persist:

- total count;
- representative samples;
- issue type;
- first/last positions when useful.

---

## 53. Validation Metrics

Track:

```
file_validation_started_total
file_validation_success_total
file_validation_failure_total
file_validation_warning_total
file_validation_quarantined_total
file_validation_duration_seconds
file_validation_bytes_total
```

Break down failures using bounded dimensions such as:

```
source
dataset
format
issue_code
```

Avoid high-cardinality metric labels such as full file paths.

---

## 54. Structured Logs

Example log fields:

```
event = file_validation_failed
file_id = ...
delivery_id = ...
source = bank_a
dataset = payments
format = csv
validator_version = 3
issue_code = CHECKSUM_MISMATCH
```

Logs should allow an operator to move from an alert to the exact validation record.

---

## 55. Alerts

Alert on actionable validation failures.

Examples:

### Critical

Repeated required-source validation failures.

### Warning

Unexpected increase in validation warnings.

### Security

Archive/path behavior violates safety policy.

Avoid paging on every individual malformed optional file if the source contract does not require immediate intervention.

---

## 56. Testing Strategy

### Unit tests

Test each validator independently.

Examples:

- empty file;
- file below minimum size;
- file above maximum size;
- unsupported suffix;
- checksum mismatch;
- invalid UTF-8;
- duplicate CSV header;
- missing required column;
- malformed JSON;
- malformed JSONL line;
- corrupt gzip.

### Integration tests

Test the entire gate:

```
REGISTER FILE
   ↓
VALIDATE
   ↓
PERSIST RESULT
   ↓
STATE TRANSITION
   ↓
VALIDATED or QUARANTINED
```

---

## 57. Test Fixtures

Keep small deterministic fixtures.

Example:

```
tests/fixtures/files/
  valid.csv
  empty.csv
  missing_header.csv
  duplicate_header.csv
  malformed.json
  invalid_utf8.csv
  corrupt.gz
```

Do not rely on production files for basic automated tests.

---

## 58. Property-Based Testing

Validation logic can benefit from generated inputs.

Examples:

- random filenames;
- random header permutations;
- malformed delimiters;
- random Unicode;
- boundary file sizes.

Property-based testing is particularly useful for parsers and contract validators.

---

## 59. Intentional Failure Drills

### Drill 1 — Empty file

Verify it is rejected when empty deliveries are forbidden.

### Drill 2 — Wrong extension

Rename a JSON file as CSV and verify the pipeline does not blindly trust the extension.

### Drill 3 — Corrupt gzip

Truncate a gzip file and verify integrity validation fails.

### Drill 4 — Checksum mismatch

Change one byte after the checksum is generated.

Verify the file is quarantined.

### Drill 5 — Invalid encoding

Inject invalid UTF-8 bytes.

Verify the failure is deterministic.

### Drill 6 — Missing required column

Remove a mandatory CSV header.

Verify extraction never starts.

### Drill 7 — Duplicate header

Create two amount columns.

Verify the ambiguity is rejected.

### Drill 8 — Duplicate validation run

Validate the same immutable file twice.

Verify the outcome is consistent and audit history remains correct.

---

## 60. Recovery Runbook

### File fails checksum

1. Confirm expected checksum source.
2. Recalculate the artifact hash.
3. Verify transport did not alter the file.
4. Compare with producer metadata.
5. Keep the file quarantined.
6. Request retransmission when appropriate.
7. Register the replacement as a new artifact/version according to policy.

### File is structurally invalid

1. Identify the validation issue code.
2. Confirm the active source contract.
3. Confirm validator version.
4. Inspect a bounded sample.
5. Determine whether the producer changed format.
6. Update the contract only when the change is intentional and approved.
7. Revalidate explicitly.

### File uses unsupported schema version

1. Preserve the file.
2. Quarantine it.
3. Confirm the producer version.
4. Add compatibility intentionally.
5. Test the new version.
6. deploy the validator/extractor update.
7. Revalidate the quarantined artifact.

### File is unexpectedly empty

1. Confirm whether zero-record delivery is valid.
2. Check producer export status.
3. Check transport status.
4. Check expected control totals.
5. Keep the artifact isolated until resolved.

---

## 61. Production Tools You Should Know

### 1. Great Expectations

Useful for declarative expectations and data-quality validation after or around extraction boundaries. Understand where file-level checks end and record-level expectations begin.

### 2. Apache Tika

Useful for content-type detection and metadata extraction across many document/file formats when such broad detection is appropriate.

### 3. Prometheus/Grafana

Useful for validation failure rates, quarantine counts, duration, throughput, and source-specific operational health.

Tools help implement validation, but the core design remains:

```
CONTRACT
   ↓
VALIDATE ARTIFACT
   ↓
PERSIST EVIDENCE
   ↓
VALIDATED or QUARANTINED
```

---

## 62. Common Mistakes

1. Trusting file extensions.
2. Treating arrival as validation.
3. Treating completeness as validation.
4. Loading huge files entirely into memory.
5. Ignoring compression integrity.
6. Ignoring checksums supplied by producers.
7. Silently repairing malformed source files.
8. Mixing file validation with all business data-quality rules.
9. Deleting invalid files immediately.
10. Having no quarantine process.
11. Storing only a boolean validation result.
12. Having no machine-readable issue codes.
13. Not versioning validation logic.
14. Validating files while they are still being written.
15. Blindly extracting archives.
16. Ignoring decompression-bomb risks.
17. Allowing unbounded validation errors.
18. Having no parser/resource timeout policy.
19. Using full file paths as metric labels.
20. Marking a file processed when it has only been validated.

---

## 63. Definition of Done

You can independently:

- define a file-validation contract;
- separate file validation from data quality;
- validate file existence and type;
- validate size constraints;
- handle empty-file policy;
- validate declared versus observed format;
- verify checksums;
- validate compression integrity;
- validate text encoding;
- validate CSV headers;
- detect duplicate columns;
- validate JSON/JSONL structure;
- understand XML safety concerns;
- use format-specific validation for Parquet and Avro;
- validate schema versions;
- stream large validation operations;
- classify blocking errors versus warnings;
- persist validation history;
- version validators;
- quarantine invalid artifacts;
- revalidate safely;
- protect archive extraction;
- monitor validation health;
- recover from validation failures.

---

## 64. What You Learned

> **A file should not enter extraction merely because it exists and arrived on time.**

The production validation pattern is:

```
DELIVERY ARRIVED
      ↓
DELIVERY COMPLETE
      ↓
LOAD VALIDATION CONTRACT
      ↓
PHYSICAL CHECKS
      ↓
IDENTITY CHECKS
      ↓
INTEGRITY CHECKS
      ↓
FORMAT CHECKS
      ↓
SCHEMA-ENVELOPE CHECKS
      ↓
VALIDATED
   OR
QUARANTINED
```

The key question is:

> **Can I prove that this exact artifact is structurally safe, contract-compliant, auditable, and ready for extraction?**

Once the answer is yes, format-specific extraction can begin.
