# E60 — Compressed File Extraction

## 1. Problem Recognition

Compression is common in ETL file deliveries because it reduces storage and network transfer costs.

Typical sources include:

- `file.csv.gz`;
- `file.jsonl.gz`;
- `payments.zip`;
- `export.tar.gz`;
- compressed object-storage deliveries;
- compressed files transferred through SFTP.

Compression changes the extraction boundary.

A pipeline must answer:

1. What compression format is this?
2. Is it a single compressed stream or an archive?
3. Can it be streamed?
4. How large can the decompressed output become?
5. Are multiple files contained inside?
6. Can archive members escape the intended extraction directory?
7. How do we validate completeness?
8. How do we preserve source identity?
9. What happens when decompression fails halfway through?

The production flow is:

```
DISCOVER COMPRESSED SOURCE
        |
        v
IDENTIFY FORMAT
        |
        v
VALIDATE SOURCE
        |
        v
OPEN SAFE DECOMPRESSION BOUNDARY
        |
        v
STREAM / ENUMERATE MEMBERS
        |
        v
EXTRACT DATA
        |
        v
VALIDATE
        |
        v
STAGE
        |
        v
RECONCILE + OBSERVE
```

Core rule:

> Compression is a transport/storage concern. It does not replace the validation and extraction contract of the underlying data format.

---

## 2. Compression vs Archive

These concepts are related but not identical.

### Compression

Compression transforms one byte stream into a smaller byte stream.

Example:

```
payments.csv
     |
     v
gzip
     |
     v
payments.csv.gz
```

### Archive

An archive contains one or more files.

Example:

```
payments.csv
customers.csv
refunds.csv
      |
      v
     ZIP
      |
      v
delivery.zip
```

A ZIP file can contain multiple members. gzip normally represents one compressed stream.

This distinction determines the extractor design.

---

## 3. Common Formats

| Format | Typical Python module | Typical use |
|---|---|---|
| gzip | `gzip` | One compressed file |
| bzip2 | `bz2` | One compressed file |
| xz | `lzma` | One compressed file |
| ZIP | `zipfile` | Multi-file archive |
| tar | `tarfile` | File archive |
| tar.gz | `tarfile` + gzip | Compressed archive |

Do not choose the decoder only from the filename.

Where possible, combine:

- filename extension;
- source contract;
- magic/signature bytes;
- actual decoder behavior.

---

## 4. Compression Is Not the Data Format

A file named:

```
payments.jsonl.gz
```

has two layers:

```
transport/storage:
    gzip

payload:
    JSONL
```

The extractor therefore has two boundaries:

```
compressed bytes
      |
      v
decompression
      |
      v
JSONL bytes
      |
      v
JSONL parser
      |
      v
records
```

Likewise:

```
payments.csv.gz
```

means:

```
gzip -> CSV -> records
```

Do not build a separate business parser for every compression format. Keep decompression and payload extraction separate.

---

## 5. Why Streaming Matters

A compressed file can expand dramatically.

For example:

```
compressed size   = 100 MB
decompressed size = 5 GB
```

Reading the complete decompressed output into memory is unsafe.

Bad:

```
data = gzip.open("payments.csv.gz", "rb").read()
```

Better:

```
with gzip.open("payments.csv.gz", "rb") as file:
    for line in file:
        process(line)
```

The important principle is:

> Measure resource usage against the decompressed stream, not only the compressed file size.

---

## 6. gzip Streaming

Python's standard library provides `gzip`.

Example:

```
import gzip
from pathlib import Path

def read_gzip_lines(path: Path):
    with gzip.open(path, "rt", encoding="utf-8", newline="") as file:
        for line in file:
            yield line
```

The payload can then be passed to the normal format parser.

For CSV:

```
import csv

with gzip.open(
    "payments.csv.gz",
    "rt",
    encoding="utf-8",
    newline="",
) as file:
    for row in csv.DictReader(file):
        process(row)
```

Compression does not change CSV parsing rules.

---

## 7. JSONL Inside gzip

A common production delivery is JSONL compressed with gzip.

```
import gzip
import json

def extract_jsonl_gzip(path):
    with gzip.open(path, "rt", encoding="utf-8") as file:
        for line_number, line in enumerate(file, start=1):
            if not line.strip():
                continue

            yield line_number, json.loads(line)
```

The pipeline still needs JSON validation, record identity, error handling, quarantine, and reconciliation.

gzip only changed how bytes reach the JSON parser.

---

## 8. Detect the Compression Layer

Filename detection is useful but insufficient.

For example:

```
payments.csv.gz
```

strongly suggests gzip.

But a producer can incorrectly name a file.

A robust boundary can inspect magic bytes.

Common signatures include:

```
gzip:
1F 8B

ZIP:
50 4B 03 04

BZIP2:
42 5A 68

XZ:
FD 37 7A 58 5A 00
```

Use signatures as evidence, while still respecting the producer contract.

---

## 9. Do Not Trust Extensions Alone

This is unsafe:

```
if path.suffix == ".gz":
    use_gzip()
```

A file could be:

```
payments.csv.gz
```

but actually contain invalid or unrelated bytes.

A stronger process is:

```
filename
   +
magic bytes
   +
source contract
   |
   v
compression classification
```

If these disagree, fail or quarantine according to the source contract rather than guessing.

---

## 10. gzip Integrity

gzip can fail during reading.

Possible causes include:

- truncated compressed stream;
- corrupted checksum;
- incomplete download;
- invalid header;
- damaged compressed data.

Do not mark the source successful until the complete stream has been consumed.

This is important because the beginning of a gzip stream may decode correctly while corruption occurs near the end.

---

## 11. ZIP Archives

ZIP is different because it is an archive.

Inspect members before extracting:

```
from zipfile import ZipFile

with ZipFile("delivery.zip") as archive:
    for info in archive.infolist():
        print(
            info.filename,
            info.file_size,
            info.compress_size,
        )
```

This lets the pipeline understand what it received before processing every member.

---

## 12. ZIP Member Inventory

Create an inventory before extraction.

Useful fields include:

- member name;
- compressed size;
- uncompressed size;
- modification time;
- directory/file status;
- CRC where available.

Example:

```
def inventory_zip(path):
    with ZipFile(path) as archive:
        return [
            {
                "name": info.filename,
                "compressed_size": info.compress_size,
                "uncompressed_size": info.file_size,
                "is_directory": info.is_dir(),
            }
            for info in archive.infolist()
        ]
```

The inventory becomes part of source-level observability.

---

## 13. Archive Safety

Archives are untrusted input boundaries.

A dangerous archive may contain paths such as:

```
../../outside.txt
```

or absolute paths.

Never blindly extract an archive into a production filesystem.

Use a safe extraction directory and validate every member path before writing it.

---

## 14. ZIP Path Validation

A simple safety boundary:

```
from pathlib import Path

def safe_member_path(root: Path, member_name: str) -> Path:
    root = root.resolve()
    target = (root / member_name).resolve()

    if target != root and root not in target.parents:
        raise ValueError(
            f"Unsafe archive member: {member_name}"
        )

    return target
```

This prevents an archive member from escaping the intended directory.

Prefer library-supported safe extraction mechanisms where available, but understand the underlying path-traversal problem.

---

## 15. Do Not Extract Everything Automatically

A ZIP archive may contain:

```
README.txt
payments.csv
customers.csv
backup.sql
secrets.txt
```

The pipeline should process only expected members.

Define an archive contract:

```
expected members
allowed extensions
required members
optional members
maximum member size
maximum member count
```

Unexpected members should be reported rather than silently ignored.

---

## 16. Archive Completeness

For multi-file deliveries, completeness may mean:

```
payments.csv
customers.csv
refunds.csv
```

must all arrive.

An archive itself can be complete while its business delivery is incomplete.

Therefore distinguish:

```
archive integrity
       !=
delivery completeness
```

A valid ZIP containing only two of three required members is still an incomplete delivery.

---

## 17. Zip Bomb Protection

Compression can produce extreme expansion ratios.

For example:

```
compressed = 10 MB
expanded   = 20 GB
```

The file may be technically valid but operationally unsafe.

Track:

```
total compressed bytes
total uncompressed bytes
expansion ratio
member count
largest member
```

Apply source-specific limits.

Never allow an archive to consume unlimited disk or memory.

---

## 18. Streaming ZIP Members

Do not always extract ZIP members to disk.

A member can often be streamed directly:

```
from zipfile import ZipFile

with ZipFile("delivery.zip") as archive:
    with archive.open("payments.csv") as member:
        for line in member:
            process(line)
```

This is useful when the payload parser can consume a binary or text stream.

For text:

```
import io

with ZipFile("delivery.zip") as archive:
    with archive.open("payments.csv") as member:
        text = io.TextIOWrapper(
            member,
            encoding="utf-8",
            newline="",
        )

        for line in text:
            process(line)
```

---

## 19. gzip Over Object Storage

For remote sources, avoid downloading the entire decompressed payload when the architecture supports streaming.

Conceptually:

```
object storage
      |
      v
compressed response stream
      |
      v
gzip decompressor
      |
      v
payload parser
      |
      v
records
```

The exact implementation depends on the object-storage client.

The important design is:

> Keep decompression as close to the source byte stream as practical.

---

## 20. Source Identity

Compression creates another identity question.

These can be different:

```
delivery.zip
```

with one set of bytes and:

```
delivery.zip
```

with different bytes.

Use immutable source identity such as:

- object version;
- content hash;
- checksum;
- source URI plus version;
- acquisition metadata.

For archives, also track member identity:

```
source_identity
+
member_name
+
member checksum/size where useful
```

This makes replay and diagnosis much safer.

---

## 21. Compressed File Lineage

A useful lineage chain is:

```
delivery.zip
    |
    +--> payments.csv
    |       |
    |       +--> payment record 1
    |       +--> payment record 2
    |
    +--> customers.csv
            |
            +--> customer record 1
```

For gzip:

```
payments.csv.gz
      |
      v
payments.csv stream
      |
      v
records
```

Store enough lineage to answer:

> Which exact compressed artifact and archive member produced this record?

---

## 22. Payload Format Detection

After decompression, identify the payload format.

Examples:

```
.gz
 |
 +--> CSV
 +--> JSONL
 +--> XML
 +--> Avro
 +--> Parquet
```

The decompressor should not assume the payload type.

A clean architecture is:

```
Compression Reader
       |
       v
Byte/Text Stream
       |
       v
Payload Extractor
       |
       v
Record Validator
       |
       v
Staging
```

This separation makes the pipeline reusable.

---

## 23. Generic Compression Boundary

A simple interface:

```
from typing import BinaryIO

def open_compressed(path) -> BinaryIO:
    ...
```

The caller should not care whether the source is gzip or another supported single-stream codec.

Then:

```
stream = open_compressed(path)
records = extract_payload(stream)
```

The compression layer handles bytes. The payload extractor handles records.

---

## 24. Detect and Handle Unsupported Compression

Do not silently treat unknown data as uncompressed.

Classify:

```
SUPPORTED_GZIP
SUPPORTED_BZIP2
SUPPORTED_XZ
SUPPORTED_ZIP
UNCOMPRESSED
UNSUPPORTED
INVALID
```

For unsupported compression:

1. record the source;
2. record the detected signature;
3. quarantine it;
4. notify the owner or route to the appropriate pipeline.

---

## 25. Decompression Errors

Examples:

- invalid header;
- checksum failure;
- unexpected EOF;
- corrupted block;
- unsupported compression feature.

Treat decompression errors as source/extraction failures.

Do not retry forever if the artifact itself is deterministic and corrupted.

A useful state flow is:

```
DISCOVERED
   |
   v
VALIDATING
   |
   v
DECOMPRESSING
   |
   +---- failure ---> QUARANTINED
   |
   v
EXTRACTING
   |
   v
STAGED
   |
   v
RECONCILED
   |
   v
SUCCESS
```

---

## 26. Partial Output Must Not Become Success

Suppose a 5 GB gzip file produces 4.8 GB successfully and then fails.

The pipeline must not say:

```
4.8 GB processed
SUCCESS
```

unless the source contract explicitly supports partial delivery.

Default behavior should be:

```
decompression failure
       |
       v
source FAILED
       |
       v
partial output isolated
       |
       v
source repaired/reacquired
       |
       v
full replay
```

---

## 27. Quarantine

Quarantine should preserve enough information to diagnose the source.

Record:

```
source_identity
compression_type
payload_type
member_name
failure_class
error_message
observed_size
expected_size if available
timestamp
```

Keep the original artifact when retention policy allows it.

Do not quarantine only a filename. The artifact identity matters.

---

## 28. Database Staging

Compression should not change the idempotent staging boundary.

For example:

```
compressed source
      |
      v
decompression
      |
      v
CSV/JSONL/etc.
      |
      v
record identity
      |
      v
ON CONFLICT DO NOTHING
      |
      v
staging
```

The same record should remain idempotent regardless of whether the source arrived compressed or uncompressed.

---

## 29. Bounded Processing

A safe pipeline keeps these bounded:

- compressed input;
- decompressed bytes;
- parser buffers;
- records in memory;
- database batch size;
- archive members processed concurrently.

Do not assume that compression makes memory usage smaller.

Compression usually reduces storage/network bytes, not the logical dataset size.

---

## 30. Checkpointing

For a single compressed stream, checkpoints can be difficult because compressed byte offsets do not necessarily correspond directly to payload record positions.

Prefer logical checkpoints such as:

```
source identity
record count
last durable record identity
payload-specific checkpoint
```

For archives, checkpoint by member:

```
archive identity
member name
member processing status
last durable record identity
```

Do not resume from an arbitrary compressed byte offset unless the codec and implementation explicitly support safe seeking.

---

## 31. ZIP Member Processing State

For a multi-file archive, use member-level state:

```
archive
  |
  +-- payments.csv   SUCCESS
  +-- customers.csv  SUCCESS
  +-- refunds.csv    FAILED
```

This lets the pipeline identify the exact failed member.

If the archive is immutable and staging is idempotent, replaying the complete archive can still be safe.

---

## 32. Reconciliation for Archives

Suppose the contract requires three members.

Reconcile:

```
expected members
=
observed required members
```

Then reconcile records:

```
member decoded
=
member staged
+
member quarantined
+
explicitly skipped
```

Finally reconcile the delivery:

```
all required members successful
+
all record counts accounted for
=
delivery successful
```

---

## 33. Resource Monitoring

Monitor both compressed and decompressed dimensions.

Useful metrics:

### Source

- compressed bytes;
- decompressed bytes;
- expansion ratio;
- member count.

### Processing

- records processed;
- bytes processed;
- records/sec;
- decompression throughput.

### Failures

- decompression errors;
- archive safety violations;
- unexpected members;
- oversized members;
- quota/resource violations.

---

## 34. Example gzip ETL

```
import csv
import gzip
from pathlib import Path

def extract_csv_gzip(path: Path):
    with gzip.open(
        path,
        "rt",
        encoding="utf-8",
        newline="",
    ) as file:
        reader = csv.DictReader(file)

        for row_number, row in enumerate(
            reader,
            start=1,
        ):
            yield row_number, row
```

The CSV parser remains responsible for CSV rules. gzip remains responsible for decompression.

---

## 35. Example ZIP ETL

```
import csv
import io
from zipfile import ZipFile

def extract_csv_from_zip(path, member_name):
    with ZipFile(path) as archive:
        with archive.open(member_name) as member:
            text = io.TextIOWrapper(
                member,
                encoding="utf-8",
                newline="",
            )

            reader = csv.DictReader(text)

            for row_number, row in enumerate(
                reader,
                start=1,
            ):
                yield row_number, row
```

Validate the member name against the delivery contract before calling this function.

---

## 36. Testing Strategy

Test compression separately from payload extraction.

### Compression tests

Test:

- valid gzip;
- truncated gzip;
- corrupted gzip;
- valid ZIP;
- corrupted ZIP;
- oversized member;
- unsafe member path;
- unexpected member;
- missing required member.

### Payload tests

Test:

- valid CSV;
- malformed JSONL;
- invalid XML;
- invalid Avro;
- invalid Parquet.

### Integration tests

Test:

```
compressed source
      |
      v
decompression
      |
      v
payload extraction
      |
      v
validation
      |
      v
staging
```

---

## 37. Example Failure Test

A gzip corruption test:

```
def test_corrupt_gzip_fails(tmp_path):
    path = tmp_path / "broken.csv.gz"
    path.write_bytes(b"not-a-valid-gzip-stream")

    try:
        list(extract_csv_gzip(path))
    except Exception:
        pass
    else:
        raise AssertionError(
            "Expected decompression failure"
        )
```

In a production codebase, assert the specific exception class and pipeline failure state rather than catching every exception.

---

## 38. Intentional Failure Drill

### Drill A — Truncated gzip

1. Take a valid gzip file.
2. Remove bytes from the end.
3. Run extraction.
4. Confirm the decompressor fails.
5. Confirm the source is not marked successful.
6. Confirm partial records are not published as complete.
7. Quarantine the artifact.
8. Restore the valid source.
9. Reprocess.
10. Reconcile.

### Drill B — ZIP path traversal

1. Create an archive containing an unsafe member path.
2. Run archive validation.
3. Confirm extraction is blocked.
4. Confirm the source is quarantined.
5. Confirm no file escapes the extraction directory.

### Drill C — ZIP bomb/resource limit

1. Use a highly compressed test archive.
2. Set a low expansion limit.
3. Process the archive.
4. Confirm the resource guard stops extraction.
5. Confirm the source is classified as a resource-limit failure.

---

## 39. Recovery Recipe

When compressed extraction fails:

1. Identify the exact source artifact.
2. Determine compression type.
3. Determine whether the failure is source corruption, archive safety, resource exhaustion, or payload parsing.
4. Check durable staging state.
5. Quarantine unsafe or corrupt artifacts.
6. Reacquire the source when necessary.
7. Replay using idempotent record identity.
8. Reconcile compressed source, archive members, and records.
9. Record the root cause.

Never solve a corrupted source by repeatedly retrying the same bytes forever.

---

## 40. Production Tools You Should Know

### 1. Python standard compression libraries

Know:

- `gzip`;
- `bz2`;
- `lzma`;
- `zipfile`;
- `tarfile`.

These are sufficient for many ETL ingestion boundaries.

### 2. libarchive / archive tooling

Useful when production environments need broad archive-format support and controlled extraction behavior.

### 3. Object-storage streaming clients

Cloud storage SDKs can provide streaming response bodies so compressed objects can be decompressed without first materializing the entire payload locally.

The transferable skill is separating:

```
storage
+
compression
+
archive
+
payload format
+
record extraction
```

---

## 41. Common Mistakes

1. Treating gzip and ZIP as the same thing.
2. Trusting only the filename extension.
3. Reading the entire decompressed output into memory.
4. Extracting ZIP archives without path validation.
5. Processing unexpected archive members.
6. Ignoring decompression expansion.
7. Marking success before EOF is reached.
8. Retrying corrupted artifacts forever.
9. Losing source/member lineage.
10. Checkpointing arbitrary compressed byte offsets.
11. Assuming compression makes the logical dataset smaller.
12. Mixing decompression and business parsing into one function.

---

## 42. Production Implementation Sequence

```
1. Identify the producer's compression contract.
2. Determine whether the source is a stream or archive.
3. Identify supported compression formats.
4. Define magic-byte and extension validation.
5. Define maximum compressed and decompressed sizes.
6. Define archive member expectations.
7. Implement safe decompression.
8. Implement streaming payload extraction.
9. Preserve source and member identity.
10. Validate payload records.
11. Stage idempotently.
12. Add member-level state for archives.
13. Add checkpointing at logical boundaries.
14. Add quarantine.
15. Add reconciliation.
16. Add expansion/resource metrics.
17. Test truncation and corruption.
18. Test archive traversal protection.
19. Test oversized expansion.
20. Run the recovery drill.
21. Enable production scheduling.
```

---

## 43. Production Checklist

### Compression

- [ ] Compression types identified.
- [ ] Archive vs single-stream distinction documented.
- [ ] Magic-byte validation defined.
- [ ] Maximum expansion defined.
- [ ] Unsupported formats fail safely.

### Archives

- [ ] Expected members defined.
- [ ] Required and optional members defined.
- [ ] Member paths validated.
- [ ] Unexpected members reported.
- [ ] Member-level state exists where required.

### Extraction

- [ ] Decompression streams data.
- [ ] Full decompressed dataset is not loaded into memory.
- [ ] Payload parser is separate from decompressor.
- [ ] EOF is required for success.
- [ ] Partial output cannot masquerade as complete success.

### Lineage

- [ ] Source identity recorded.
- [ ] Member identity recorded.
- [ ] Payload format recorded.
- [ ] Compression format recorded.

### Safety

- [ ] Resource limits exist.
- [ ] Archive traversal is blocked.
- [ ] Oversized members are handled.
- [ ] Sensitive paths/data are not logged.

### Operations

- [ ] Metrics exist.
- [ ] Structured errors exist.
- [ ] Quarantine exists.
- [ ] Reconciliation exists.
- [ ] Recovery procedure is tested.

---

## 44. Definition of Done

E60 is complete when you can independently:

- distinguish compression from archive formats;
- identify gzip, ZIP, bzip2, and xz boundaries;
- stream decompressed data;
- separate decompression from payload parsing;
- safely inspect ZIP members;
- prevent archive path traversal;
- detect incomplete compressed sources;
- protect against excessive expansion;
- preserve source and member lineage;
- stage records idempotently;
- design logical checkpoints;
- reconcile archive and record completeness;
- observe compression and extraction performance;
- intentionally break compressed deliveries;
- recover failed deliveries safely.

---

## 45. What You Learned

Compressed-file extraction is not a separate data format problem. It is a **transport boundary around another data format**.

The mental model is:

```
COMPRESSED SOURCE
       |
       v
IDENTIFY COMPRESSION
       |
       v
SAFE DECOMPRESSION
       |
       v
PAYLOAD FORMAT
       |
       v
RECORD EXTRACTION
       |
       v
VALIDATION
       |
       v
IDEMPOTENT STAGING
       |
       v
RECONCILIATION
       |
       v
OBSERVE + RECOVER
```

The most important production lesson is:

> A compressed file is successful only when the compressed source is valid, the complete decompressed payload has been consumed, the expected delivery contract is satisfied, and every extracted record is accounted for.

---

## 46. Next Recipe

**E61 — Large File Streaming**

The next recipe focuses specifically on processing files larger than available memory, including bounded buffers, chunking, streaming parsers, checkpoints, backpressure, throughput measurement, and restart-safe processing.
