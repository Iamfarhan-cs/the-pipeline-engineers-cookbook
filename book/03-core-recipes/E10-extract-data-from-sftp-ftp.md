# E10 — Extract Data from SFTP/FTP

SFTP and FTP are common integration sources for partner-delivered files, bank exports, settlement files, compliance reports, and legacy systems.

- **SFTP** runs over SSH and provides encrypted file transfer.
- **FTP** is an older file-transfer protocol and does not provide encryption by itself.

The extraction problem is not simply downloading a file. A production extractor must discover the right files, determine whether they are ready, transfer them safely, preserve file identity, avoid duplicates, and checkpoint only after successful persistence.

## 1. Problem Recognition

Typical production problems:

- partner uploads a file while the extractor is reading it
- files arrive late
- the same filename is uploaded again
- a producer replaces a file
- directory listings are incomplete or large
- connection drops during download
- credentials expire
- FTP transfers expose credentials or data without encryption
- files are moved or deleted before processing
- a zero-byte file is mistaken for a valid empty dataset
- large files are loaded entirely into memory
- the extractor marks a file complete before downstream persistence

Do not equate:

```text
file exists on remote server
```

with:

```text
file is complete and safe to process
```

## 2. Core Extraction Architecture

```text
REMOTE SERVER
     ↓
DISCOVER FILES
     ↓
FILTER EXPECTED FILES
     ↓
VERIFY READINESS / IDENTITY
     ↓
STREAM DOWNLOAD
     ↓
PARSE
     ↓
VALIDATE
     ↓
PERSIST
     ↓
CHECKPOINT
     ↓
MARK FILE COMPLETE
```

The most important ordering rule is:

```text
DOWNLOAD → PERSIST → CHECKPOINT
```

## 3. SFTP vs FTP

| Property | SFTP | FTP |
|---|---|---|
| Transport | SSH | FTP protocol |
| Encryption | Yes when configured normally | No by default |
| Authentication | SSH key/password depending on server | Username/password and server-specific mechanisms |
| Typical use | Modern secure partner exchange | Legacy integrations |

Prefer SFTP when the partner supports it.

Do not treat plain FTP as equivalent to encrypted transport.

## 4. Implementation — Build the Mechanism

The first implementation should separate the extraction contract from the transfer library.

### 4.1 Remote file identity

```python
from dataclasses import dataclass
from datetime import datetime

@dataclass(frozen=True)
class RemoteFile:
    path: str
    size: int
    modified_at: datetime | None
    fingerprint: str | None = None
```

A filename alone is often not enough to identify a delivery.

Depending on the partner contract, identity can include:

- remote path
- size
- modification time
- checksum
- server-provided file ID
- content hash

### 4.2 File readiness

A simple readiness contract can require the producer to publish a completed file under its final name.

```python
def is_ready(file: RemoteFile) -> bool:
    return file.size > 0
```

This is only a demonstration. Size alone is not a universal completeness guarantee.

A stronger producer contract is:

```text
UPLOAD temporary-name
        ↓
UPLOAD COMPLETE
        ↓
RENAME / PUBLISH final-name
        ↓
EXTRACTOR DISCOVERS final-name
```

The extractor should process only files that satisfy the agreed publication contract.

### 4.3 File stability check

When no atomic publication contract exists, a stability check can compare metadata across two observations.

```python
import time

def wait_until_stable(stat_file, path, interval_seconds=5):
    first = stat_file(path)
    time.sleep(interval_seconds)
    second = stat_file(path)

    return (
        first.size == second.size
        and first.modified_at == second.modified_at
    )
```

A stability check reduces the chance of reading an actively changing file, but it is weaker than an explicit producer publication contract.

### 4.4 Durable processed-file identity

```python
def file_identity(file: RemoteFile) -> tuple:
    return (
        file.path,
        file.size,
        file.modified_at,
        file.fingerprint,
    )

def should_process(file: RemoteFile, processed: set[tuple]) -> bool:
    return file_identity(file) not in processed

def mark_complete(file: RemoteFile, processed: set[tuple]) -> None:
    processed.add(file_identity(file))
```

In production, `processed` must be durable state in a database or reliable checkpoint store.

### 4.5 Stream the remote file

Do not require the entire file to fit in memory.

```python
def stream_remote_file(client, remote_path, chunk_size=1024 * 1024):
    with client.open(remote_path, "rb") as remote:
        while True:
            chunk = remote.read(chunk_size)
            if not chunk:
                break
            yield chunk
```

### 4.6 Stream into a local staging file

Local staging is useful when network transfer should be separated from parsing.

```python
from pathlib import Path

def download_to_staging(client, remote_path, destination: Path):
    temporary = destination.with_suffix(destination.suffix + ".part")

    with client.open(remote_path, "rb") as source:
        with temporary.open("wb") as target:
            while True:
                chunk = source.read(1024 * 1024)
                if not chunk:
                    break
                target.write(chunk)

    temporary.replace(destination)
```

The `.part` file prevents a partial download from looking like a completed local artifact.

### 4.7 Parse after transfer

Once the remote transfer is complete, hand the staged file to the appropriate parser.

```python
import csv

def read_csv_records(path: Path):
    with path.open("r", encoding="utf-8", newline="") as handle:
        yield from csv.DictReader(handle)
```

Detailed CSV parsing belongs to E05. This recipe focuses on the remote-transfer boundary.

### 4.8 End-to-end SFTP extraction skeleton

```python
def extract_sftp_files(client, files, processed, staging_dir):
    for remote_file in files:
        if not is_ready(remote_file):
            continue

        if not should_process(remote_file, processed):
            continue

        destination = staging_dir / remote_file.path.rsplit("/", 1)[-1]

        download_to_staging(
            client,
            remote_file.path,
            destination,
        )

        try:
            for record in read_csv_records(destination):
                persist_record(record)

            mark_complete(remote_file, processed)
        finally:
            destination.unlink(missing_ok=True)
```

The checkpoint is advanced only after all records have been persisted successfully.

## 5. SFTP Implementation with Paramiko

`paramiko` is a common Python SSH/SFTP library.

```python
import paramiko

def connect_sftp(host, username, password=None, key_filename=None):
    transport = paramiko.Transport((host, 22))

    if key_filename:
        private_key = paramiko.RSAKey.from_private_key_file(key_filename)
        transport.connect(username=username, pkey=private_key)
    else:
        transport.connect(username=username, password=password)

    return paramiko.SFTPClient.from_transport(transport)
```

Production code should use secure host-key verification and a proper secret-management mechanism rather than accepting an unknown server key blindly.

### Discover remote files

```python
from datetime import datetime, timezone

def discover_sftp_files(sftp, directory):
    for item in sftp.listdir_attr(directory):
        modified = datetime.fromtimestamp(
            item.st_mtime,
            tz=timezone.utc,
        )

        yield RemoteFile(
            path=f"{directory.rstrip('/')}/{item.filename}",
            size=item.st_size,
            modified_at=modified,
        )
```

Filter this result using the source contract instead of processing every remote file.

```python
def expected_csv_files(files):
    for file in files:
        if file.path.endswith(".csv") and file.size > 0:
            yield file
```

## 6. FTP Implementation

Python's standard library provides `ftplib` for FTP.

```python
from ftplib import FTP

def connect_ftp(host, username, password):
    ftp = FTP(host, timeout=30)
    ftp.login(username, password)
    return ftp
```

For encrypted FTP variants, use the appropriate secure protocol and library configuration. Do not send sensitive production data over plain FTP merely because the connection works.

### FTP download

```python
def download_ftp_file(ftp, remote_path, destination):
    temporary = destination.with_suffix(destination.suffix + ".part")

    with temporary.open("wb") as output:
        ftp.retrbinary(
            f"RETR {remote_path}",
            output.write,
            blocksize=1024 * 1024,
        )

    temporary.replace(destination)
```

Completion of the local download does not mean downstream processing succeeded.

## 7. Remote File Discovery

Do not blindly process every file in the remote directory.

Example:

```text
/incoming/payments/
    payments_20260926.csv
    payments_20260927.csv
    README.txt
    temporary.part
```

An extractor might accept only:

```text
payments_*.csv
```

and reject temporary files.

## 8. Temporary and Partial Files

Common producer patterns include:

```text
payments_20260926.csv.part
payments_20260926.csv.tmp
payments_20260926.uploading
```

These should normally be excluded.

Do not invent suffix rules without confirming the partner contract.

## 9. Atomic Publication

The strongest file-delivery contract is:

```text
CREATE temporary file
        ↓
WRITE COMPLETE CONTENT
        ↓
OPTIONALLY VERIFY CHECKSUM
        ↓
RENAME TO FINAL NAME
        ↓
EXTRACTOR DISCOVERS FINAL NAME
```

This separates writing from publication.

When possible, make the partner integration use an explicit completion marker or atomic rename convention.

## 10. Completion Marker Pattern

Some partners provide:

```text
payments.csv
payments.csv.done
```

The extractor can require the `.done` marker before processing the data file.

```python
def has_completion_marker(files, data_name):
    return f"{data_name}.done" in files
```

The marker is part of the producer contract and should not be assumed to prove content correctness without further validation.

## 11. Checksum Verification

If the partner provides a checksum, verify it after download.

```python
import hashlib

def sha256_file(path: Path) -> str:
    digest = hashlib.sha256()

    with path.open("rb") as handle:
        while chunk := handle.read(1024 * 1024):
            digest.update(chunk)

    return digest.hexdigest()

def verify_checksum(path: Path, expected: str) -> None:
    actual = sha256_file(path)

    if actual != expected:
        raise ValueError("checksum mismatch")
```

Checksum verification is stronger than trusting filename, size, or modification time alone.

## 12. File Replacement

Suppose:

```text
/incoming/payments.csv
```

is processed once and later replaced with different content.

A filename-only checkpoint will incorrectly treat the replacement as already processed.

Use:

- checksum
- source version
- size + modification time where appropriate
- immutable naming
- another source-defined identity

## 13. Duplicate Deliveries

A partner may send identical content under different filenames.

```text
payments_001.csv
payments_001_retry.csv
```

Filename-based deduplication will not detect identical content.

If the contract requires content-level deduplication, compute a checksum and maintain a durable fingerprint registry.

```python
def file_fingerprint(path: Path) -> str:
    return sha256_file(path)
```

Do not hash every enormous file unnecessarily when the producer already supplies a trusted checksum.

## 14. Large File Handling

Use streaming transfer and staged processing.

```text
REMOTE FILE
     ↓
STREAM
     ↓
LOCAL .part FILE
     ↓
ATOMIC LOCAL RENAME
     ↓
PARSER
     ↓
BATCH PERSISTENCE
```

This limits memory use and makes partial downloads distinguishable from completed local artifacts.

## 15. Connection Lifecycle

Prefer:

```text
CONNECT
   ↓
AUTHENTICATE
   ↓
DISCOVER
   ↓
DOWNLOAD REQUIRED FILES
   ↓
CLOSE
```

Reconnect when the connection is no longer usable.

## 16. Authentication and Secrets

Production rules:

- never hard-code passwords
- prefer SSH keys for SFTP when supported
- store secrets in a secret manager
- rotate credentials
- use least-privilege accounts
- restrict the account to the required directory where supported
- never log passwords or private keys

## 17. SFTP Host-Key Verification

SFTP authentication has two separate concerns:

1. **Who are we authenticating as?**
2. **Which server are we connecting to?**

Host-key verification helps prevent connecting to an unexpected server.

Do not disable host-key verification simply to make a connection work.

## 18. Testing

Test at least:

1. One valid file.
2. Multiple files.
3. Temporary files.
4. Empty file.
5. Missing expected file.
6. Duplicate filename.
7. Duplicate content under another filename.
8. File replacement.
9. File stability failure.
10. Completion marker missing.
11. Checksum mismatch.
12. Connection timeout.
13. Connection reset during download.
14. Authentication failure.
15. Host-key mismatch.
16. Remote permission failure.
17. Large file.
18. Partial local download.
19. Parser failure.
20. Persistence failure.
21. Restart after failure.
22. Late-arriving file.

## 19. Example Unit Tests

```python
def test_duplicate_file_is_skipped():
    processed = set()
    file = RemoteFile(
        "/incoming/payments.csv",
        100,
        None,
        "abc123",
    )

    assert should_process(file, processed)
    mark_complete(file, processed)
    assert not should_process(file, processed)


def test_replacement_has_new_identity():
    processed = set()

    old = RemoteFile(
        "/incoming/payments.csv",
        100,
        None,
        "old-hash",
    )

    new = RemoteFile(
        "/incoming/payments.csv",
        120,
        None,
        "new-hash",
    )

    mark_complete(old, processed)

    assert should_process(new, processed)


def test_partial_suffix_is_not_selected():
    files = [
        RemoteFile("/incoming/payments.csv.part", 100, None),
        RemoteFile("/incoming/payments.csv", 1000, None),
    ]

    selected = [
        f for f in files
        if f.path.endswith(".csv")
        and not f.path.endswith(".part")
    ]

    assert [f.path for f in selected] == [
        "/incoming/payments.csv"
    ]
```

## 20. Observability

Useful metrics:

```text
sftp_connection_attempts_total
sftp_connection_failures_total
remote_files_discovered_total
remote_files_selected_total
remote_files_skipped_total
remote_files_downloaded_total
remote_files_failed_total
remote_bytes_downloaded_total
remote_download_duration_seconds
remote_download_retries_total
remote_checksum_failures_total
remote_files_processed_total
remote_files_late_total
```

Useful log fields:

```text
run_id
protocol
host_alias
remote_path
file_size
modified_at
fingerprint
attempt
download_duration
records_processed
error_type
```

Do not log passwords, private keys, session tokens, or sensitive file contents.

## 21. Intentional Failure

### Failure 1 — Process a temporary file

Place a `.part` file in the incoming directory.

Verify discovery excludes it.

### Failure 2 — File changes during extraction

Modify the source file between metadata checks.

Verify the extractor detects instability or refuses unsafe processing according to the publication contract.

### Failure 3 — Connection reset

Interrupt the connection during a large download.

Verify the partial local file is not mistaken for a completed artifact.

### Failure 4 — Checksum mismatch

Alter a downloaded file before verification.

Verify extraction fails before persistence.

### Failure 5 — Duplicate delivery

Deliver identical content under two filenames.

Verify content-level deduplication when the source contract requires it.

### Failure 6 — Persistence failure

Fail downstream persistence after download.

Verify the file is not marked complete and can be safely retried.

### Failure 7 — Authentication failure

Use invalid credentials.

Verify the pipeline fails clearly without infinite retries.

## 22. Recovery

1. Identify the pipeline run.
2. Identify protocol and remote server.
3. Identify the remote file path.
4. Inspect file size, modification time, and fingerprint.
5. Determine whether the producer has published the file completely.
6. Check connection/authentication status.
7. Check the local staging artifact.
8. Verify checksum if available.
9. Determine whether downstream persistence completed.
10. Resume or restart according to the checkpoint contract.
11. Reconcile persisted records.
12. Mark the remote file complete only after successful persistence.

Never delete a source file as the first response to a failed extraction unless the source contract explicitly defines the extractor as responsible for deletion and the downstream result is already verified.

## 23. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Paramiko** | Python SSH/SFTP library for secure remote file transfer and SFTP operations. |
| **Python ftplib** | Standard-library FTP client for legacy FTP integrations. |
| **SFTP** | Secure file-transfer protocol over SSH; understand host keys, authentication, remote paths, and streaming transfers. |

## 24. Production Runbook

### Files are missing

Check:

1. Remote directory.
2. Filename pattern.
3. Expected arrival time.
4. Completion marker.
5. Producer status.
6. Remote permissions.

### File is present but extraction rejects it

Check:

1. File identity.
2. Size.
3. Modification time.
4. Stability.
5. Checksum.
6. Parser validation.

### Download repeatedly fails

Check:

1. Network connectivity.
2. Server availability.
3. Credentials.
4. Host-key configuration.
5. Remote permissions.
6. File size.
7. Retry classification.

### Duplicate records appear

Check:

1. Remote file identity.
2. Durable processed-file registry.
3. Content fingerprints.
4. Checkpoint timing.
5. Whether the producer replaced or duplicated the file.

### What not to do

Do not:

- process temporary files
- assume a filename proves content identity
- read huge files fully into memory
- disable SFTP host-key verification
- hard-code credentials
- use plain FTP for sensitive data without an explicitly secured architecture
- mark files complete before persistence
- delete source files before successful reconciliation
- retry permanent authentication failures forever

## 25. Common Mistakes

### Mistake 1 — Treating directory listing as readiness

A file can exist while still being uploaded.

### Mistake 2 — Filename-only checkpoints

The same path can contain different content over time.

### Mistake 3 — Ignoring temporary files

Temporary artifacts may be incomplete.

### Mistake 4 — No local partial-download protection

A failed download can look like a valid staging file unless `.part` semantics are used.

### Mistake 5 — Disabling host-key verification

This removes an important SFTP server-authentication control.

### Mistake 6 — Treating FTP and SFTP as equivalent

Plain FTP does not provide the same transport security.

## 26. Definition of Done

You are done when you can:

- explain SFTP and FTP differences
- discover remote files safely
- distinguish published files from temporary files
- design a completion contract
- use stability checks when necessary
- define remote-file identity
- stream large files
- stage downloads atomically
- verify checksums
- detect replacement and duplicate delivery
- maintain durable processed-file state
- handle connection failures
- classify authentication failures
- use SFTP host-key verification
- protect credentials
- test partial downloads
- intentionally break remote extraction
- recover without corrupting downstream state
- operate the integration using a runbook

## 27. What You Learned

The central principle is:

> A remote file is not safe to process merely because it exists; the extractor needs a publication contract, a stable identity, a safe transfer, and durable processing state.

The production pattern is:

```text
DISCOVER
   ↓
VERIFY PUBLICATION
   ↓
IDENTIFY FILE
   ↓
STREAM DOWNLOAD
   ↓
STAGE SAFELY
   ↓
PARSE / VALIDATE
   ↓
PERSIST
   ↓
CHECKPOINT
   ↓
MARK COMPLETE
```

The key question is:

> What proves that this remote file is complete, exactly which artifact I processed, and that the downstream result was persisted before I consider the file complete?
