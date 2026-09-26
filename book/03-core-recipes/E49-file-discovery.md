# E49 — File Discovery

## 1. Problem Recognition

### The production problem

A file-based pipeline cannot process a file it does not reliably know exists.

In simple scripts, file discovery is often treated as:

```
files = list(input_dir.glob("*.csv"))
```

Production pipelines may receive files through local directories, shared network storage, SFTP, object storage, cloud storage prefixes, mounted volumes, vendor drop zones, or generated export directories.

The real problem is:

> **How do I discover the correct files, prove that they belong to this pipeline, avoid processing the wrong or duplicate files, and create durable work items for downstream processing?**

### Why file discovery is an ETL problem

Discovery happens before parsing. If discovery is wrong, every downstream stage can be technically correct while the pipeline still produces an incorrect result.

```
FILES ARRIVE
     ↓
DISCOVER CANDIDATES
     ↓
FILTER
     ↓
IDENTIFY
     ↓
CLASSIFY
     ↓
REGISTER
     ↓
PROCESS
```

A production discovery layer should answer:

- What files are present?
- Where did they come from?
- Which files belong to this pipeline?
- Which files are new?
- Which files have already been processed?
- Which files are still being written?
- Which files are suspicious?
- Which files should be ignored?
- Which files require manual review?

---

## 2. Concept and Reasoning

### Discovery is not ingestion

Separate these responsibilities:

```
DISCOVERY
Find candidate files
     ↓
REGISTRATION
Create durable file identity/state
     ↓
VALIDATION
Check whether the file is acceptable
     ↓
INGESTION
Read file contents
     ↓
TRANSFORMATION
Process records
     ↓
LOAD
Persist business state
```

A scanner should not silently parse every file it finds.

### The discovery contract

For each discovered candidate, capture enough metadata to answer:

> What exactly did the pipeline discover?

| Field | Example | Purpose |
|---|---|---|
| file_id | UUID | Internal identity |
| source_system | `bank_partner_a` | Source identification |
| location | S3/SFTP path | Physical location |
| filename | `payments_20260926.csv` | Human-readable identity |
| size_bytes | 1839201 | Completeness/change signal |
| modified_at | timestamp | Source metadata |
| discovered_at | timestamp | Pipeline evidence |
| content_hash | SHA-256 | Content identity |
| status | `DISCOVERED` | Processing state |
| discovery_run_id | UUID | Operational correlation |

Do not rely on filename alone as the permanent identity of a file.

---

## 3. Define the Discovery Scope

Before scanning anything, define exactly where the pipeline is allowed to look.

Example:

```
s3://partner-a/inbound/payments/
```

or:

```
/srv/etl/inbound/payments/
```

The scope should define:

- allowed source;
- allowed directory/prefix;
- allowed file extensions;
- expected naming pattern;
- excluded directories;
- maximum file size;
- optional minimum file size;
- whether hidden files are ignored;
- whether recursive discovery is allowed;
- whether symlinks are allowed;
- retention/archive rules.

A discovery process with an unrestricted root such as `/` is an operational hazard.

---

## 4. Discover Candidates, Do Not Process Them Immediately

A basic local discovery implementation:

```
from pathlib import Path

INPUT_DIR = Path("/srv/etl/inbound")

for path in INPUT_DIR.glob("*.csv"):
    if path.is_file():
        print(path)
```

This is only candidate discovery.

A production implementation should separate:

```
PATH FOUND
   ↓
IS REGULAR FILE?
   ↓
EXPECTED EXTENSION?
   ↓
EXPECTED NAME?
   ↓
EXPECTED LOCATION?
   ↓
WITHIN SIZE LIMIT?
   ↓
READY TO DISCOVER?
   ↓
REGISTER
```

This makes each decision testable.

---

## 5. Use an Explicit File Naming Contract

Suppose a vendor sends:

```
payments_20260926_120000.csv
```

Define what each component means.

Example contract:

```
payments_<business_date>_<sequence>.csv
```

Do not merely check `*.csv`.

That may accept unrelated files such as `test.csv`, `notes.csv`, or temporary artifacts.

Naming conventions are covered in E50. In this recipe, discovery should consume a known naming contract and reject candidates that do not satisfy it.

---

## 6. Candidate Filtering

A useful filtering sequence is:

1. location/prefix;
2. regular-file check;
3. extension;
4. filename pattern;
5. source identifier;
6. expected date/window;
7. size constraints;
8. readiness condition;
9. duplicate identity check.

Example:

```
import re
from pathlib import Path

PATTERN = re.compile(
    r"^payments_(?P<date>\d{8})_(?P<sequence>\d+)\.csv$"
)

def is_candidate(path: Path) -> bool:
    if not path.is_file():
        return False

    if path.suffix.lower() != ".csv":
        return False

    return PATTERN.fullmatch(path.name) is not None
```

Keep discovery rules deterministic.

---

## 7. File Identity

Filename is a useful business identifier, but it is not always a unique content identity.

Consider:

```
payments_20260926.csv
```

The same filename could be uploaded twice with different contents.

Conversely, the same content could be copied under two different filenames.

Therefore distinguish:

- **location identity** — where the file was found;
- **name identity** — what the producer called it;
- **content identity** — what bytes it contains;
- **business identity** — what logical delivery it represents.

These identities solve different problems.

### Content hash

A cryptographic hash can provide content identity:

```
import hashlib

def sha256_file(path, chunk_size=1024 * 1024):
    digest = hashlib.sha256()

    with path.open("rb") as handle:
        while chunk := handle.read(chunk_size):
            digest.update(chunk)

    return digest.hexdigest()
```

Read the file in chunks rather than loading the entire file into memory.

For very large files, hash calculation itself is an I/O operation and should be accounted for in discovery throughput.

---

## 8. Do Not Hash Everything Blindly

Hashing every candidate can be expensive.

For high-volume object storage, first use cheap metadata to narrow candidates:

```
LIST
 ↓
PREFIX FILTER
 ↓
NAME FILTER
 ↓
SIZE / METADATA FILTER
 ↓
READINESS CHECK
 ↓
HASH ONLY WHEN NEEDED
```

Whether a content hash is required should be an explicit pipeline policy.

Useful situations include duplicate detection, immutable-file verification, audit requirements, and vendor replay detection.

---

## 9. Register Discovery Results

Do not keep discovery state only in process memory.

Use a durable registry.

Example PostgreSQL table:

```
CREATE TABLE discovered_file (
    file_id UUID PRIMARY KEY,
    source_system TEXT NOT NULL,
    location TEXT NOT NULL,
    filename TEXT NOT NULL,
    size_bytes BIGINT NOT NULL,
    modified_at TIMESTAMPTZ,
    content_hash TEXT,
    discovered_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status TEXT NOT NULL,
    discovery_run_id UUID NOT NULL,
    UNIQUE (source_system, location)
);
```

The registry becomes the durable boundary between discovery and processing.

---

## 10. Discovery Must Be Idempotent

Suppose the scanner runs at:

```
10:00 → finds file A
10:05 → finds file A again
10:10 → finds file A again
```

That should not create three independent processing jobs.

Use a stable uniqueness rule.

```
INSERT INTO discovered_file (
    file_id,
    source_system,
    location,
    filename,
    size_bytes,
    modified_at,
    status,
    discovery_run_id
)
VALUES (
    gen_random_uuid(),
    %(source_system)s,
    %(location)s,
    %(filename)s,
    %(size_bytes)s,
    %(modified_at)s,
    'DISCOVERED',
    %(discovery_run_id)s
)
ON CONFLICT (source_system, location)
DO NOTHING;
```

The exact uniqueness key depends on the source.

If a source overwrites the same path with new content, location alone is insufficient. In that case, combine location/version metadata or content identity according to the source contract.

---

## 11. Mutable Files Are a Different Problem

Consider:

```
10:00  report.csv = 10 MB
10:01  report.csv = 20 MB
10:02  report.csv = 35 MB
10:03  report.csv = complete
```

A scanner that discovers the file at 10:00 may process incomplete data.

This is why discovery and readiness must be separate concepts.

```
DISCOVERED
    ≠
READY
```

Possible readiness mechanisms include:

- producer writes to a temporary name and renames when complete;
- companion `.done` or manifest file;
- object-storage completion metadata;
- expected size from a manifest;
- stable size/metadata over repeated observations;
- producer-provided delivery status.

Do not assume that “file exists” means “file is complete.”

---

## 12. Stability Checks

For a filesystem source without a producer-controlled completion marker, a stability check can reduce the risk of reading an actively written file.

Example:

```
from dataclasses import dataclass
from pathlib import Path
import time

@dataclass(frozen=True)
class FileObservation:
    size: int
    modified_at: float

def observe(path: Path) -> FileObservation:
    stat = path.stat()
    return FileObservation(
        size=stat.st_size,
        modified_at=stat.st_mtime,
    )

def is_stable(path: Path, wait_seconds: int = 30) -> bool:
    first = observe(path)
    time.sleep(wait_seconds)
    second = observe(path)

    return first == second
```

This is evidence of stability during the observation window, not proof that the producer will never modify the file again.

A producer-controlled completion signal is generally stronger.

---

## 13. Temporary and Hidden Files

Many producers create temporary artifacts such as:

```
payments_20260926.csv.tmp
payments_20260926.csv.part
.~lock.payments
```

Discovery should explicitly reject known temporary patterns.

```
TEMP_SUFFIXES = {".tmp", ".part", ".partial"}

def is_temporary(path: Path) -> bool:
    return path.suffix.lower() in TEMP_SUFFIXES
```

Do not rely on one naming convention across all vendors. Make the source-specific rule configurable.

---

## 14. Recursive Discovery

Recursive scanning can be useful:

```
inbound/
├── partner_a/
│   ├── payments/
│   └── refunds/
└── partner_b/
    └── payments/
```

But recursive discovery can accidentally include archives, quarantine files, temporary directories, previous pipeline output, or unrelated vendor files.

Define allowed prefixes explicitly.

For object storage, prefix-based listing is usually preferable to pretending the bucket is a normal filesystem.

---

## 15. Object Storage Discovery

For object storage, discovery normally begins with a list operation over a controlled prefix.

Conceptually:

```
BUCKET
  ↓
PREFIX
  ↓
OBJECT METADATA
  ↓
FILTER
  ↓
REGISTER
```

Typical metadata includes:

- object key;
- size;
- last-modified timestamp;
- ETag or provider-specific identifier;
- version identifier where versioning is enabled;
- storage class;
- metadata/tags.

Do not assume that an ETag is always a simple MD5 content hash. Its meaning can depend on the storage provider and upload mechanism.

For versioned object stores, object version identity can be stronger than path identity.

---

## 16. SFTP Discovery

SFTP discovery adds remote-system concerns:

```
CONNECT
  ↓
LIST REMOTE DIRECTORY
  ↓
FILTER
  ↓
READ REMOTE METADATA
  ↓
REGISTER
  ↓
DOWNLOAD LATER
```

Do not download every discovered file merely to determine what exists.

Register candidates first when the workflow permits it.

For large remote files, separate discovery connection, transfer, and processing.

---

## 17. Discovery Run Identity

Every scan should have a run identity.

Example:

```
discovery_run_id = 8a7...
source = partner_a
started_at = 10:00:00
finished_at = 10:00:14
candidates = 183
accepted = 170
rejected = 13
new = 22
already_known = 148
errors = 0
```

This provides operational evidence.

A discovery run can then be investigated without reconstructing logs from multiple processes.

---

## 18. Discovery Run Table

Example:

```
CREATE TABLE discovery_run (
    discovery_run_id UUID PRIMARY KEY,
    source_system TEXT NOT NULL,
    started_at TIMESTAMPTZ NOT NULL,
    finished_at TIMESTAMPTZ,
    candidates_found BIGINT NOT NULL DEFAULT 0,
    candidates_accepted BIGINT NOT NULL DEFAULT 0,
    new_files BIGINT NOT NULL DEFAULT 0,
    existing_files BIGINT NOT NULL DEFAULT 0,
    rejected_files BIGINT NOT NULL DEFAULT 0,
    error_count BIGINT NOT NULL DEFAULT 0,
    status TEXT NOT NULL
);
```

Now discovery itself has lifecycle state.

---

## 19. Discovery Status Model

A simple state model:

```
DISCOVERED
    ↓
READY
    ↓
PROCESSING
    ↓
PROCESSED
```

Failure paths:

```
DISCOVERED → REJECTED
DISCOVERED → QUARANTINED
READY      → FAILED
PROCESSING → FAILED
```

Do not allow arbitrary state transitions.

For example, `PROCESSED → DISCOVERED` should require an explicit replay or reset operation rather than an accidental scanner update.

---

## 20. Discovery Does Not Mean Processing

This distinction is critical.

A scanner should be allowed to say:

```
"I found this file."
```

without saying:

```
"I processed this file successfully."
```

That separation allows retries, delayed processing, manual approval, quarantine, backpressure, controlled replay, and multiple consumers.

---

## 21. Complete Python Discovery Skeleton

A small production-oriented local discovery component can look like this:

```
from dataclasses import dataclass
from pathlib import Path
import re
import uuid


@dataclass(frozen=True)
class FileCandidate:
    source_system: str
    location: str
    filename: str
    size_bytes: int
    modified_at: float


class FileDiscovery:
    def __init__(self, root: Path, source_system: str):
        self.root = root
        self.source_system = source_system
        self.pattern = re.compile(
            r"^payments_(?P<date>\d{8})_(?P<sequence>\d+)\.csv$"
        )

    def discover(self) -> list[FileCandidate]:
        candidates = []

        for path in self.root.glob("*.csv"):
            if not path.is_file():
                continue

            match = self.pattern.fullmatch(path.name)
            if not match:
                continue

            stat = path.stat()

            candidates.append(
                FileCandidate(
                    source_system=self.source_system,
                    location=str(path.resolve()),
                    filename=path.name,
                    size_bytes=stat.st_size,
                    modified_at=stat.st_mtime,
                )
            )

        return sorted(
            candidates,
            key=lambda item: (item.filename, item.location),
        )


if __name__ == "__main__":
    discovery = FileDiscovery(
        root=Path("/srv/etl/inbound"),
        source_system="partner_a",
    )

    discovery_run_id = uuid.uuid4()

    for candidate in discovery.discover():
        print(discovery_run_id, candidate)
```

This is intentionally not the complete ingestion system. Its responsibility is discovery.

---

## 22. Register Candidates Transactionally

Discovery should register a candidate atomically.

Example:

```
def register_candidate(conn, candidate, discovery_run_id):
    with conn.cursor() as cur:
        cur.execute(
            """
            INSERT INTO discovered_file (
                file_id,
                source_system,
                location,
                filename,
                size_bytes,
                modified_at,
                discovered_at,
                status,
                discovery_run_id
            )
            VALUES (
                gen_random_uuid(),
                %(source_system)s,
                %(location)s,
                %(filename)s,
                %(size_bytes)s,
                to_timestamp(%(modified_at)s),
                now(),
                'DISCOVERED',
                %(discovery_run_id)s
            )
            ON CONFLICT (source_system, location)
            DO NOTHING
            """,
            {
                "source_system": candidate.source_system,
                "location": candidate.location,
                "filename": candidate.filename,
                "size_bytes": candidate.size_bytes,
                "modified_at": candidate.modified_at,
                "discovery_run_id": discovery_run_id,
            },
        )

    conn.commit()
```

If registration succeeds but downstream processing fails, the file remains discoverable through durable state.

---

## 23. Security Boundaries

File discovery is also a security boundary.

Do not blindly process arbitrary paths, symlinks escaping the allowed directory, executable files, unexpected archives, files from untrusted directories, filenames containing unexpected control characters, or objects outside the approved prefix.

For local files, resolve paths and verify they remain inside the allowed root.

```
def is_inside_root(path: Path, root: Path) -> bool:
    try:
        path.resolve().relative_to(root.resolve())
        return True
    except ValueError:
        return False
```

Discovery should reject paths outside the configured boundary.

---

## 24. Symlink Handling

A file may appear inside the directory while actually pointing somewhere else.

Example:

```
inbound/payments.csv
        ↓
/secret/other-file.csv
```

Unless symlinks are explicitly part of the source contract, reject them.

```
if path.is_symlink():
    continue
```

This is especially important when the discovery root is writable by another process.

---

## 25. Ordering

Filesystem listing order is not a reliable business ordering.

Do not assume:

```
readdir order = arrival order
```

If processing order matters, derive it from explicit metadata such as filename sequence, business date, source timestamp, manifest order, or object metadata.

Discovery should produce deterministic ordering for downstream work claiming, but that ordering must come from the contract rather than filesystem behavior.

---

## 26. Duplicate Discovery Versus Duplicate Content

These are different cases.

### Same location discovered twice

```
/path/payments_01.csv
/path/payments_01.csv
```

This is duplicate discovery.

### Two locations containing identical content

```
/path/a.csv
/path/b.csv
```

This is duplicate content.

### Same business delivery under different filenames

```
payments_20260926.csv
payments_20260926_retry.csv
```

This may be a business-level duplicate.

Use different controls for each case.

---

## 27. Backfill and Historical Discovery

Historical discovery should not reuse the normal "new files only" assumptions blindly.

For a backfill:

```
HISTORICAL PREFIX
       ↓
DISCOVER
       ↓
REGISTER
       ↓
CLASSIFY AS BACKFILL
       ↓
PROCESS WITH BACKFILL POLICY
```

Record the reason for discovery and the logical processing mode.

A historical file should not accidentally enter the current-day operational flow simply because it matches the same filename pattern.

---

## 28. Testing

### Unit tests

Test:

- valid filename;
- invalid filename;
- wrong extension;
- directory instead of file;
- temporary file;
- hidden file;
- symlink;
- file outside allowed root;
- zero-byte file;
- oversized file;
- duplicate location;
- duplicate content;
- malformed metadata.

Example:

```
def test_discovery_rejects_wrong_filename(tmp_path):
    path = tmp_path / "notes.csv"
    path.write_text("hello")

    discovery = FileDiscovery(tmp_path, "partner_a")

    assert discovery.discover() == []
```

### Integration tests

Test:

- discovery registry insertion;
- repeated discovery;
- database uniqueness;
- discovery run accounting;
- status transitions;
- concurrent scanners;
- object-storage listing;
- SFTP listing where applicable.

---

## 29. Concurrency

Two scanners may run simultaneously:

```
SCANNER A ──→ finds file X
                  ↘
                   REGISTRY
                  ↗
SCANNER B ──→ finds file X
```

Do not rely on application-level "check then insert":

```
SELECT → not found
INSERT
```

Two workers can both observe "not found."

Use a database uniqueness constraint and an atomic insert.

The database should enforce the identity rule.

---

## 30. Intentional Failure Drills

### Drill 1 — Scan the wrong directory

Verify the pipeline rejects or reports the unexpected scope rather than processing unrelated files.

### Drill 2 — Create a temporary file

Verify it is not registered as ready.

### Drill 3 — Modify a file during discovery

Verify the pipeline does not treat an actively changing file as complete.

### Drill 4 — Run two scanners simultaneously

Verify only one durable file identity is registered.

### Drill 5 — Re-run the scanner

Verify the same file does not create duplicate work.

### Drill 6 — Replace file contents at the same path

Verify the pipeline follows the configured version/content identity policy.

### Drill 7 — Create a symlink outside the root

Verify the discovery boundary rejects it.

### Drill 8 — Introduce an unexpected filename

Verify the file is rejected with an observable reason.

---

## 31. Observability

Track discovery metrics such as:

```
file_discovery_runs_total
file_discovery_candidates_total
file_discovery_accepted_total
file_discovery_rejected_total
file_discovery_new_files_total
file_discovery_duplicate_total
file_discovery_error_total
file_discovery_duration_seconds
file_discovery_bytes_discovered_total
```

Useful dimensions include source system, pipeline, discovery location, and rejection reason.

Avoid high-cardinality labels such as full filenames on every metric. Put detailed file identity in structured logs or database state.

A useful discovery log should contain:

```
discovery_run_id
source_system
location
filename
size_bytes
modified_at
status
reason
```

---

## 32. Recovery Runbook

### Files are not being discovered

1. Verify the source location.
2. Verify credentials or mount availability.
3. Verify the configured prefix/path.
4. Check filename filters.
5. Check source arrival timing.
6. Check discovery-run errors.
7. Test listing manually in a controlled environment.

### Files are discovered but never processed

1. Inspect the durable file registry.
2. Check file status.
3. Check readiness state.
4. Check downstream worker health.
5. Check whether backpressure is blocking processing.

### Duplicate files are appearing

1. Determine whether the duplication is by location, content, or business delivery.
2. Inspect the uniqueness key.
3. Compare hashes and source metadata.
4. Check concurrent scanners.
5. Review producer behavior.

### A file was discovered while still being written

1. Stop downstream processing for the file.
2. Determine whether the source provides a completion signal.
3. Wait for a valid readiness condition.
4. Revalidate size/metadata.
5. Process only after the file satisfies the contract.

### Wrong files entered the pipeline

1. Quarantine the affected candidates.
2. Stop further discovery from the incorrect scope if necessary.
3. Tighten the discovery contract.
4. Identify already-created downstream work.
5. Reconcile and replay only approved files.

---

## 33. Production Tools You Should Know

### 1. Python pathlib

Learn filesystem traversal, path validation, file metadata, and safe local-file handling.

### 2. S3-compatible object storage APIs

Learn prefix listing, object metadata, versioning, pagination, and object identity.

### 3. SFTP

Learn remote directory listing, metadata inspection, connection handling, and controlled file transfer.

These tools change the transport mechanism, but the discovery principles remain the same:

```
DISCOVER
  ↓
FILTER
  ↓
IDENTIFY
  ↓
REGISTER
  ↓
READY
  ↓
PROCESS
```

---

## 34. Common Mistakes

1. Treating `*.csv` as a complete discovery policy.
2. Processing files immediately during directory scanning.
3. Assuming file existence means file completeness.
4. Using filename as the only identity.
5. Keeping discovery state only in memory.
6. Using check-then-insert instead of database uniqueness.
7. Hashing every large file unnecessarily.
8. Assuming filesystem order equals arrival order.
9. Recursively scanning archives and output directories.
10. Accepting symlinks without a security policy.
11. Ignoring mutable files.
12. Mixing discovery state with processing state.
13. Having no discovery-run identity.
14. Having no rejection reason.
15. Treating duplicate discovery and duplicate business delivery as the same problem.
16. Assuming object-store ETags always represent a simple MD5 hash.
17. Allowing two scanners to create duplicate processing work.
18. Treating historical backfill files as normal current arrivals.

---

## 35. Definition of Done

You can independently:

- define a controlled file-discovery scope;
- identify candidate files;
- apply deterministic filters;
- consume a naming contract;
- distinguish location, filename, content, and business identity;
- detect or reason about incomplete files;
- register discovered files durably;
- make discovery idempotent;
- protect against concurrent scanners;
- handle local files, object storage, and SFTP conceptually;
- reject unsafe paths and unexpected files;
- separate discovery from readiness and processing;
- create discovery-run evidence;
- test duplicate and failure scenarios;
- observe discovery health;
- recover discovery failures safely;
- distinguish duplicate discovery from duplicate business deliveries.

---

## 36. What You Learned

> **File discovery is the control plane for file-based ETL. The pipeline should first establish what files exist and whether they belong to the workload before it attempts to process their contents.**

The production pattern is:

```
DEFINE SOURCE SCOPE
       ↓
DISCOVER CANDIDATES
       ↓
FILTER
       ↓
VERIFY IDENTITY
       ↓
CHECK READINESS
       ↓
REGISTER DURABLY
       ↓
CLAIM FOR PROCESSING
       ↓
INGEST
```

The key question is:

> **Can I prove which files entered the pipeline, why they were accepted or rejected, whether they were complete, and whether repeated discovery can safely occur?**

That is the foundation required before reliable file ingestion, validation, and processing can begin.
