## 1. Problem Recognition

Production file pipelines often receive a logical **delivery** containing many files, not one isolated file.

Example:

    customers_2026-09-26.csv
    accounts_2026-09-26.csv
    transactions_2026-09-26.csv
    manifest_2026-09-26.json

These files may arrive at different times. Processing each independently can publish a partial business day.

The problem is:

> How do I discover, group, validate, process, retry, and reconcile multiple files as one logical delivery while preserving independent file-level recovery?

This recipe applies:

- Core Recipe 05 — Processing Status
- Core Recipe 06 — Idempotency
- Core Recipe 07 — Deduplication
- Core Recipe 09 — Quarantine
- Core Recipe 12 — Checkpointing
- Core Recipe 16 — Reconciliation
- Core Recipe 24 — Partial Failure
- Core Recipe 32 — Concurrency
- E49 — File Discovery
- E50 — File Naming Conventions
- E51 — File Arrival Detection
- E52 — File Completeness Detection
- E53 — File Validation
- E61 — Large File Streaming

The goal is not to repeat those mechanisms. The goal is to apply them at delivery level.

## 2. What You Are Building

Target architecture:

    Source
      |
      v
    File Discovery
      |
      v
    Delivery Grouping
      |
      v
    Delivery Contract
      |
      +-----------------------+
      |                       |
      v                       v
    File Registry          Manifest
      |                       |
      +-----------+-----------+
                  |
                  v
            Completeness
                  |
                  v
          File Extraction
        +---------+---------+
        |         |         |
        v         v         v
      File A    File B    File C
        |         |         |
        +---------+---------+
                  |
                  v
            File Staging
                  |
                  v
          Delivery Reconcile
                  |
                  v
          Delivery Complete

The critical rule is:

    file state != delivery state

A file can succeed while the delivery is still incomplete.

## 3. Learning Objectives

By the end you should be able to:

1. Define a multi-file delivery contract.
2. Generate stable delivery and file identities.
3. Group discovered files correctly.
4. Distinguish required, optional, and unexpected files.
5. Detect delivery completeness.
6. Process files independently where safe.
7. Bound extraction concurrency.
8. Retry only failed files when safe.
9. Quarantine failed files.
10. Prevent duplicate processing.
11. Handle file replacements explicitly.
12. Track delivery-level state.
13. Reconcile file and record evidence.
14. Recover from partial delivery failure.
15. Prove that a delivery was completely processed.

## 4. Delivery Is Not the Same as a File

A **file** is a physical source artifact.

A **delivery** is a logical business package containing one or more files.

Example:

    Delivery: bank_a / daily / 2026-09-26

    customers.csv
    accounts.csv
    transactions.csv

The identities are separate:

    delivery identity
          |
          +---- file identity
                    |
                    +---- record identity

Do not collapse these levels into one identifier.

## 5. Delivery Identity

A delivery identity answers:

> Which logical source delivery is this?

A practical identity can be:

    source_system
    delivery_type
    business_date

Example:

    bank_a|daily|2026-09-26

The identity must be deterministic. Rediscovering the same delivery must not create a new delivery.

If the producer provides a delivery ID, prefer that stable producer identifier when its semantics are trustworthy.

## 6. File Identity

A file identity answers:

> Which logical file inside this delivery is this?

Example:

    bank_a|daily|2026-09-26|transactions

If sequence numbers matter:

    bank_a|daily|2026-09-26|transactions|003

Do not automatically use the physical path as the business identity because files may move from incoming storage to archive storage.

## 7. Record Identity

A record identity answers:

> Which logical record is represented by this extracted item?

Example:

    transaction_id = TX123

The three identities should remain separate:

    delivery
      |
      +-- file
            |
            +-- record

This separation makes replay, correction, and reconciliation manageable.

## 8. Delivery Contract

Before implementing extraction, define what a valid delivery contains.

Example:

| File type | Required | Cardinality | Dependency |
|---|---:|---:|---|
| customers | yes | 1 | none |
| accounts | yes | 1 | customers |
| transactions | yes | 1 | accounts |
| fees | no | 0 or 1 | transactions |
| manifest | yes | 1 | all files |

The contract should answer:

- Which files are expected?
- Which files are required?
- Which files are optional?
- How many instances are allowed?
- How are files identified?
- Is a manifest required?
- Can files arrive out of order?
- Can files be replaced?
- What makes the delivery complete?

Without a contract, completeness becomes guesswork.

## 9. Required, Optional, and Unexpected Files

Suppose:

    required:
        customers
        accounts
        transactions

    optional:
        fees

Observed:

    customers
    accounts
    transactions

The delivery may be complete.

Observed:

    customers
    accounts

The delivery is incomplete.

Observed:

    customers
    accounts
    transactions
    marketing

The pipeline has an unexpected file.

Unexpected does not automatically mean failure. It means an explicit policy is required:

- reject delivery;
- quarantine file;
- ignore with audit;
- accept as an extension.

Never silently ignore it.

## 10. Contract as Code

Use an executable contract.

```
from dataclasses import dataclass
from typing import FrozenSet


@dataclass(frozen=True)
class DeliveryContract:
    source_system: str
    delivery_type: str
    required_file_types: FrozenSet[str]
    optional_file_types: FrozenSet[str]
    max_files_per_type: int = 1

    @property
    def allowed_file_types(self) -> FrozenSet[str]:
        return (
            self.required_file_types
            | self.optional_file_types
        )
```

Example:

```
contract = DeliveryContract(
    source_system="bank_a",
    delivery_type="daily",
    required_file_types=frozenset({
        "customers",
        "accounts",
        "transactions",
    }),
    optional_file_types=frozenset({
        "fees",
    }),
)
```

The contract is now testable rather than tribal knowledge.

## 11. Deterministic Delivery Key

```
from datetime import date


def delivery_key(
    source_system: str,
    delivery_type: str,
    business_date: date,
) -> str:
    return (
        f"{source_system}|"
        f"{delivery_type}|"
        f"{business_date.isoformat()}"
    )
```

Example:

```
key = delivery_key(
    "bank_a",
    "daily",
    date(2026, 9, 26),
)

assert key == "bank_a|daily|2026-09-26"
```

Running the function repeatedly with the same inputs must return the same key.

## 12. Parse File Names

E50 teaches the general naming mechanism. E62 consumes it to identify delivery membership.

Example filename grammar:

    bank_a_daily_customers_2026-09-26.csv

Parser:

```
from dataclasses import dataclass
from datetime import date


@dataclass(frozen=True)
class DiscoveredFile:
    path: str
    source_system: str
    delivery_type: str
    file_type: str
    business_date: date


def parse_file_name(path: str) -> DiscoveredFile:
    name = path.rsplit("/", 1)[-1]
    stem = name.removesuffix(".csv")

    source, delivery_type, file_type, date_text = stem.split("_")

    return DiscoveredFile(
        path=path,
        source_system=source,
        delivery_type=delivery_type,
        file_type=file_type,
        business_date=date.fromisoformat(date_text),
    )
```

Use the producer's actual naming grammar. Do not invent parsing rules inside the worker.

## 13. Group Files into Deliveries

```
from collections import defaultdict


def group_by_delivery(files: list[DiscoveredFile]):
    groups = defaultdict(list)

    for file in files:
        key = delivery_key(
            file.source_system,
            file.delivery_type,
            file.business_date,
        )
        groups[key].append(file)

    return dict(groups)
```

Input:

    bank_a_daily_customers_2026-09-26.csv
    bank_a_daily_accounts_2026-09-26.csv
    bank_a_daily_transactions_2026-09-26.csv
    bank_a_daily_customers_2026-09-27.csv

Produces two delivery groups.

The grouping function must be deterministic.

## 14. Never Group by Polling Cycle

This is unsafe:

    files discovered during the same polling cycle
        -> same delivery

Files may arrive:

- minutes apart;
- hours apart;
- out of order;
- through different transfer sessions;
- after retries.

Use a producer delivery ID or deterministic business key.

Arrival time is evidence about timing, not necessarily business identity.

## 15. Manifest-Based Delivery

A manifest is often the strongest delivery contract.

Example:

```
{
  "delivery_id": "bank-a-2026-09-26-001",
  "business_date": "2026-09-26",
  "files": [
    {
      "file_type": "customers",
      "name": "customers.csv",
      "size_bytes": 18231,
      "sha256": "..."
    },
    {
      "file_type": "accounts",
      "name": "accounts.csv",
      "size_bytes": 71231,
      "sha256": "..."
    }
  ]
}
```

A manifest can provide:

- exact expected file set;
- file names;
- sizes;
- checksums;
- record counts;
- producer delivery ID.

The manifest itself must be validated.

## 16. Expected Set vs Observed Set

Let:

    E = expected file identities
    O = observed file identities

Then:

    missing = E - O
    unexpected = O - E
    present = E intersect O

Python:

```
def compare_file_sets(expected, observed):
    expected = set(expected)
    observed = set(observed)

    return {
        "missing": expected - observed,
        "unexpected": observed - expected,
        "present": expected & observed,
    }
```

This turns completeness into a deterministic calculation.

## 17. Completeness Is a Delivery Decision

A useful model is:

    delivery_complete =
        all_required_files_present
        AND all_required_files_valid
        AND reconciliation_passed
        AND no_unresolved_required_failures

Optional files follow explicit contract semantics.

A file existing on disk is not equivalent to a complete delivery.

## 18. Delivery State Machine

Use durable delivery states:

    DISCOVERED
        |
        v
    WAITING_FOR_FILES
        |
        v
    READY
        |
        v
    PROCESSING
        |
        +------------------+
        |                  |
        v                  v
    PARTIAL_FAILURE      COMPLETE
        |
        v
    RETRYING
        |
        +--------+
        |        |
        v        v
    PROCESSING  QUARANTINED

The exact state names can vary. The transitions must be explicit.

## 19. File State Machine

Each file has its own lifecycle:

    DISCOVERED
        |
        v
    VALIDATING
        |
        +----------+
        |          |
        v          v
    EXTRACTING  QUARANTINED
        |
        v
    STAGED
        |
        v
    COMPLETE

A retryable failure can move back to EXTRACTING.

Do not make every file failure automatically equal delivery failure.

## 20. PostgreSQL Delivery Registry

```
CREATE TABLE etl_delivery (
    delivery_id TEXT PRIMARY KEY,
    source_system TEXT NOT NULL,
    delivery_type TEXT NOT NULL,
    business_date DATE NOT NULL,
    state TEXT NOT NULL,
    expected_file_count INTEGER NOT NULL,
    observed_file_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (
        source_system,
        delivery_type,
        business_date
    )
);
```

This creates durable delivery state and prevents accidental duplicate business deliveries.

## 21. PostgreSQL File Registry

```
CREATE TABLE etl_delivery_file (
    delivery_id TEXT NOT NULL,
    file_id TEXT NOT NULL,
    file_type TEXT NOT NULL,
    path TEXT NOT NULL,
    state TEXT NOT NULL,
    size_bytes BIGINT,
    sha256 TEXT,
    attempt_count INTEGER NOT NULL DEFAULT 0,
    records_extracted BIGINT NOT NULL DEFAULT 0,
    error_code TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (delivery_id, file_id),

    FOREIGN KEY (delivery_id)
        REFERENCES etl_delivery(delivery_id)
);
```

Every file remains traceable to its delivery.

## 22. Idempotent Delivery Registration

```
INSERT INTO etl_delivery (
    delivery_id,
    source_system,
    delivery_type,
    business_date,
    state,
    expected_file_count
)
VALUES (
    %s,
    %s,
    %s,
    %s,
    'DISCOVERED',
    %s
)
ON CONFLICT (delivery_id)
DO NOTHING;
```

A repeated discovery cannot create a second delivery.

## 23. Idempotent File Registration

```
INSERT INTO etl_delivery_file (
    delivery_id,
    file_id,
    file_type,
    path,
    state
)
VALUES (%s, %s, %s, %s, 'DISCOVERED')
ON CONFLICT (delivery_id, file_id)
DO NOTHING;
```

Polling can safely rediscover the same file.

## 24. File Hashing

A content hash is useful evidence.

```
import hashlib


def sha256_file(path: str, chunk_size: int = 1024 * 1024) -> str:
    digest = hashlib.sha256()

    with open(path, "rb") as source:
        while chunk := source.read(chunk_size):
            digest.update(chunk)

    return digest.hexdigest()
```

Hash answers:

> Is this the same content as the artifact previously observed?

But:

    business identity != content identity

Two valid deliveries can contain identical content.

## 25. Duplicate vs Replacement

Same identity and same hash:

    duplicate observation

Same logical identity and different hash:

    possible replacement or correction

Do not silently overwrite the first version.

If replacements are allowed, model versions explicitly.

## 26. File Versioning

```
CREATE TABLE etl_file_version (
    delivery_id TEXT NOT NULL,
    file_id TEXT NOT NULL,
    version INTEGER NOT NULL,
    sha256 TEXT NOT NULL,
    path TEXT NOT NULL,
    state TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (delivery_id, file_id, version)
);
```

Then:

    version 1 = original
    version 2 = corrected

Downstream behavior must explicitly identify which version is authoritative.

## 27. Independent Extraction

Prefer:

    delivery
      |
      +--> customers
      +--> accounts
      +--> transactions

over a chain where unrelated files block one another.

Dependencies should exist only when they are real.

If transactions require reference data, encode that dependency explicitly.

## 28. Bounded Parallel Extraction

```
from concurrent.futures import ThreadPoolExecutor, as_completed


def process_file(file):
    return {
        "file_id": file.file_id,
        "state": "COMPLETE",
    }


def process_files(files, max_workers=4):
    results = []

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {
            executor.submit(process_file, file): file
            for file in files
        }

        for future in as_completed(futures):
            results.append(future.result())

    return results
```

The important control is the worker limit.

Never create unbounded workers simply because a delivery contains many files.

## 29. Why Concurrency Must Be Bounded

Suppose:

    500 files
    200 MB working memory per worker

Unbounded processing can demand enormous memory.

A pool of:

    8 workers

creates a much more controllable resource budget.

Concurrency must also respect:

- source bandwidth;
- CPU;
- memory;
- database connections;
- object-storage limits;
- disk I/O;
- downstream API limits.

The bottleneck resource should influence the worker limit.

## 30. Per-File Result Contract

```
from dataclasses import dataclass
from typing import Optional


@dataclass(frozen=True)
class FileResult:
    file_id: str
    state: str
    records_extracted: int = 0
    error_code: Optional[str] = None
    retryable: bool = False
```

Example success:

```
FileResult(
    file_id="transactions",
    state="COMPLETE",
    records_extracted=120_000,
)
```

Example retryable failure:

```
FileResult(
    file_id="accounts",
    state="RETRYABLE_FAILURE",
    error_code="OBJECT_STORE_TIMEOUT",
    retryable=True,
)
```

A stable result contract makes aggregation deterministic.

## 31. Isolate Worker Exceptions

```
def safe_process_file(file):
    try:
        return process_file(file)
    except TimeoutError:
        return FileResult(
            file_id=file.file_id,
            state="RETRYABLE_FAILURE",
            error_code="TIMEOUT",
            retryable=True,
        )
    except Exception:
        return FileResult(
            file_id=file.file_id,
            state="NON_RETRYABLE_FAILURE",
            error_code="UNEXPECTED_ERROR",
            retryable=False,
        )
```

In production, use narrower exception classes.

Do not turn every exception into a retry.

## 32. Retry Only the Failed File

Suppose:

    customers = COMPLETE
    accounts = COMPLETE
    transactions = TIMEOUT

If extraction is independently idempotent:

    retry transactions only

This reduces source load, destination load, processing time, and duplicate work.

Coordinated reprocessing is appropriate only when the source or business semantics require it.

## 33. Retry Classification

| Failure | Usually retry? | Reason |
|---|---:|---|
| object-storage timeout | yes | transient |
| connection reset | yes | transient |
| temporary database outage | yes | transient |
| malformed CSV | no | deterministic |
| invalid encoding | no | deterministic |
| unknown schema | no | contract issue |
| permission denied | usually no | configuration |
| process killed | yes | interrupted work |

Retry policy belongs at file scope, but delivery state must reflect unresolved required files.

## 34. Per-File Quarantine

A failed file can be quarantined without destroying evidence.

Store:

- delivery ID;
- file ID;
- original path;
- checksum;
- error code;
- error details;
- attempt count;
- timestamps;
- quarantine location.

Example:

    customers = COMPLETE
    accounts = COMPLETE
    transactions = QUARANTINED

The delivery remains unresolved unless the contract explicitly allows the missing/failed member.

## 35. Delivery Quarantine

Quarantine the entire delivery when the delivery contract itself is invalid.

Examples:

- invalid manifest;
- conflicting delivery identity;
- incompatible file versions;
- security failure;
- unreconcilable expected set.

Keep delivery quarantine separate from file quarantine.

## 36. Multi-Format Deliveries

A single delivery may contain:

    customers.csv
    accounts.json
    transactions.parquet

The coordinator should not implement every parser.

Use format-specific extractors:

```
EXTRACTORS = {
    "csv": extract_csv,
    "json": extract_json,
    "parquet": extract_parquet,
}


def extract_file(file):
    extractor = EXTRACTORS[file.format]
    return extractor(file)
```

The coordinator owns orchestration. The extractor owns format mechanics.

## 37. File Dependencies

If transactions depend on reference data, model the dependency:

```
DEPENDENCIES = {
    "transactions": {"reference"},
}
```

Do not use alphabetical filename order as a dependency model.

If no dependency exists, allow independent processing.

## 38. Two-Phase Delivery Processing

A useful architecture is:

### Phase 1 — Acquire and validate

    discover
    register
    checksum
    validate
    stage

### Phase 2 — Reconcile and publish

    verify required files
    verify file states
    reconcile counts
    publish delivery
    mark complete

This prevents an individual successful extraction from being mistaken for a complete business delivery.

## 39. Delivery Coordinator

```
def process_delivery(delivery, contract):
    observed = {file.file_type for file in delivery.files}

    missing = contract.required_file_types - observed
    unexpected = observed - contract.allowed_file_types

    if missing:
        return "WAITING_FOR_FILES"

    if unexpected:
        return "CONTRACT_VIOLATION"

    results = process_files(
        delivery.files,
        max_workers=4,
    )

    if any(r.state == "RETRYABLE_FAILURE" for r in results):
        return "PARTIAL_FAILURE"

    if any(
        r.state in {"NON_RETRYABLE_FAILURE", "QUARANTINED"}
        for r in results
    ):
        return "QUARANTINED"

    if all(r.state == "COMPLETE" for r in results):
        return "COMPLETE"

    return "WAITING_FOR_RECONCILIATION"
```

Real implementations should persist every transition.

## 40. Never Mark Complete from Worker Count

Unsafe:

```
if completed_workers == expected_workers:
    delivery.state = "COMPLETE"
```

Workers may finish before:

- staging commits;
- validation commits;
- quarantine state persists;
- reconciliation runs.

Better:

    derive delivery state
    from durable file state
    and delivery contract

A coordinator restart must not erase completion evidence.

## 41. Durable Delivery Reconciliation

Query persisted state:

```
SELECT
    file_type,
    state,
    records_extracted,
    error_code
FROM etl_delivery_file
WHERE delivery_id = %s;
```

Then evaluate:

- expected files;
- observed files;
- file states;
- optional-file rules;
- record reconciliation;
- unresolved failures.

This is safer than trusting in-memory worker results.

## 42. Reconciliation Equation

For a successful delivery:

    required expected files
    =
    required complete files

For an unresolved delivery:

    expected
    =
    complete
    + failed
    + retrying
    + missing

Quarantined required files remain unresolved unless the business contract explicitly permits them.

## 43. Record Reconciliation

File existence is not enough.

Example:

    expected records = 1,000,000
    extracted records = 800,000

If the source provides control totals, the file is not complete from a data perspective.

Use:

- manifest record counts;
- trailer counts;
- control totals;
- source counts;
- checksums.

## 44. Control Totals

Example manifest:

```
{
  "transactions": {
    "records": 1000000,
    "amount_total": "98234512.11"
  }
}
```

Validate:

    decoded_count == expected_count

and, where meaningful:

    decoded_amount == expected_amount

This is much stronger evidence than file existence.

## 45. Temporary Working Directory

A multi-file extraction may need local temporary storage:

    /work/
      delivery-2026-09-26/
        customers.csv
        accounts.csv
        transactions.csv

Rules:

1. isolate each delivery;
2. enforce disk limits;
3. clean only after durable processing;
4. preserve failed artifacts when required;
5. record original source URI;
6. never mix files from different deliveries.

Do not use one shared temporary filename for concurrent deliveries.

## 46. Object Storage Layout

A useful layout:

    raw/
      bank_a/
        daily/
          2026-09-26/
            customers.csv
            accounts.csv
            transactions.csv
            manifest.json

An immutable delivery ID can be included as another directory level.

The important property is that raw artifacts remain recoverable and traceable.

## 47. SFTP Delivery

A typical flow:

1. list remote directory;
2. discover candidate files;
3. parse delivery identity;
4. group files;
5. verify stable size;
6. verify manifest or completion marker;
7. copy to immutable raw storage;
8. calculate or verify checksum;
9. register delivery;
10. extract.

A visible filename does not prove that the producer has finished writing the file.

## 48. Completion Marker

Some producers publish:

    _SUCCESS

or:

    delivery.complete

after all members are uploaded.

When this is part of the source contract, it can define the delivery freeze point.

It still does not prove file validity.

## 49. Delivery Freeze

Without a freeze point:

    expected set keeps changing
    while processing is already running

Possible freeze mechanisms:

- manifest;
- completion marker;
- close event;
- contractual deadline;
- quiet period.

After freezing, a later file must follow an explicit late/correction policy.

## 50. Quiet Period

A quiet period means:

> No new files have arrived for a defined interval.

Example:

    no new files for 10 minutes

This can be useful for legacy producers, but is weaker than an explicit completion signal.

Record which completeness method was used.

## 51. Work Claiming

When multiple workers share the same database queue, use row locking.

```
SELECT delivery_id, file_id, path
FROM etl_delivery_file
WHERE state IN ('DISCOVERED', 'RETRYABLE_FAILURE')
ORDER BY delivery_id, file_id
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

Then:

    claim
    commit claim
    process outside transaction

Do not hold the database transaction open during network I/O.

## 52. Atomic State Transition

A simple alternative is an atomic update:

```
UPDATE etl_delivery_file
SET
    state = 'EXTRACTING',
    attempt_count = attempt_count + 1,
    updated_at = now()
WHERE delivery_id = %s
  AND file_id = %s
  AND state IN ('DISCOVERED', 'RETRYABLE_FAILURE');
```

The affected-row count tells the worker whether it successfully claimed the file.

## 53. Staging Identity

Staging should preserve delivery and file lineage.

```
CREATE TABLE etl_staging_record (
    delivery_id TEXT NOT NULL,
    file_id TEXT NOT NULL,
    record_id TEXT NOT NULL,
    payload JSONB NOT NULL,
    ingested_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (delivery_id, file_id, record_id)
);
```

The exact payload representation depends on the source.

The important property is traceability:

    delivery
        |
        file
            |
            record

## 54. Record-Level Idempotency

Without idempotency:

    first attempt  -> 100,000 rows
    retry          -> 100,000 rows
    result         -> 200,000 rows

With deterministic identity:

```
INSERT INTO etl_staging_record (
    delivery_id,
    file_id,
    record_id,
    payload
)
VALUES (%s, %s, %s, %s)
ON CONFLICT (delivery_id, file_id, record_id)
DO NOTHING;
```

Replay becomes safe.

If corrections are allowed, an explicit update/version strategy is required instead.

## 55. Batch Transactions

Do not necessarily commit every record.

Use bounded batches:

```
BATCH_SIZE = 5_000

for batch in batched(records, BATCH_SIZE):
    write_batch(batch)
    commit()
```

This provides:

- bounded memory;
- manageable transaction duration;
- predictable database load;
- useful restart points.

## 56. Checkpointing

A per-file checkpoint may contain:

    delivery_id
    file_id
    source_position
    records_extracted
    last_record_id
    updated_at

For line-oriented data, a byte offset may work.

For format-specific data, a block, object, row group, or other parser-safe position may be better.

E61 teaches the general checkpointing mechanism.

E62 applies it independently to each file.

## 57. Delivery-Level Checkpoint

Do not confuse:

    file checkpoint

with:

    delivery checkpoint

A delivery checkpoint can show:

    customers = COMPLETE
    accounts = COMPLETE
    transactions = IN_PROGRESS

On restart, completed files can be skipped while the incomplete file resumes or replays.

This is a major benefit of separating delivery and file state.

## 58. Failure Scenario — One File Fails

Delivery:

    customers       COMPLETE
    accounts        COMPLETE
    transactions    TIMEOUT
    fees            COMPLETE

Result:

    delivery = PARTIAL_FAILURE

Recovery:

1. preserve completed file state;
2. classify timeout as retryable;
3. retry transactions;
4. reconcile;
5. mark delivery complete only after all required evidence passes.

Do not reprocess successful files unnecessarily.

## 59. Failure Scenario — Missing Required File

Expected:

    customers
    accounts
    transactions

Observed:

    customers
    accounts

State:

    WAITING_FOR_FILES

Do not immediately mark failure if the delivery is still within its expected arrival window.

E51/E52 provide the arrival and completeness mechanisms.

## 60. Failure Scenario — Unexpected File

Observed:

    customers
    accounts
    transactions
    transactions_backup

If not allowed by the contract:

    CONTRACT_VIOLATION

Do not silently choose one transactions file.

Inspect the source semantics and apply the correction/duplicate policy.

## 61. Failure Scenario — Same Identity, New Hash

Original:

    file_id = transactions
    hash = ABC

Later:

    file_id = transactions
    hash = XYZ

This is a possible correction.

Do not silently overwrite.

Record versions or explicitly reject according to the contract.

## 62. Failure Scenario — Worker Crash

Suppose a worker stages 50,000 records and dies before marking the file complete.

On restart:

1. detect stale processing state;
2. inspect staging;
3. use checkpoint if safe;
4. replay or resume;
5. rely on idempotent record identity;
6. reconcile;
7. mark complete.

This is why durable state and idempotency must work together.

## 63. Observability

Track delivery-level metrics:

    deliveries_discovered_total
    deliveries_ready_total
    deliveries_complete_total
    deliveries_failed_total
    deliveries_waiting_total
    deliveries_duration_seconds

Track file-level metrics:

    files_discovered_total
    files_complete_total
    files_failed_total
    files_quarantined_total
    file_processing_duration_seconds
    file_records_extracted_total

Do not expose only aggregate delivery metrics.

## 64. Useful Dimensions

Useful dimensions include:

    source_system
    delivery_type
    file_type
    environment
    state
    failure_class

Avoid putting high-cardinality values such as customer IDs or unique file paths into ordinary metrics labels.

Use structured logs for high-cardinality identifiers.

## 65. Structured Logs

Useful events:

    delivery_discovered
    delivery_ready
    file_claimed
    file_validation_failed
    file_extraction_started
    file_extraction_completed
    file_quarantined
    delivery_reconciled
    delivery_completed

Include:

    delivery_id
    file_id
    file_type
    attempt
    state
    duration
    error_code

## 66. Audit Record

For important deliveries, preserve:

- delivery ID;
- source;
- business date;
- expected file set;
- observed file set;
- completeness method;
- file hashes;
- record counts;
- state transitions;
- retry attempts;
- quarantine events;
- reconciliation result.

This makes production incidents diagnosable after the pipeline has restarted.

## 67. Testing Strategy

Test at three levels.

### Unit

Test:

- delivery identity;
- filename parsing;
- grouping;
- expected/observed comparison;
- contract classification;
- state aggregation.

### Integration

Test:

- PostgreSQL registration;
- idempotent inserts;
- concurrent claiming;
- staging;
- reconciliation.

### End-to-End

Test:

    discovery
      ->
    grouping
      ->
    completeness
      ->
    extraction
      ->
    staging
      ->
    reconciliation
      ->
    delivery completion

## 68. Unit Test — Grouping

```
def test_files_group_into_delivery():
    files = [
        DiscoveredFile(
            path="customers.csv",
            source_system="bank_a",
            delivery_type="daily",
            file_type="customers",
            business_date=date(2026, 9, 26),
        ),
        DiscoveredFile(
            path="accounts.csv",
            source_system="bank_a",
            delivery_type="daily",
            file_type="accounts",
            business_date=date(2026, 9, 26),
        ),
    ]

    groups = group_by_delivery(files)

    assert list(groups) == [
        "bank_a|daily|2026-09-26"
    ]
    assert len(groups["bank_a|daily|2026-09-26"]) == 2
```

## 69. Unit Test — Missing File

```
def test_missing_required_file():
    expected = {
        "customers",
        "accounts",
        "transactions",
    }

    observed = {
        "customers",
        "accounts",
    }

    result = compare_file_sets(expected, observed)

    assert result["missing"] == {"transactions"}
```

## 70. Unit Test — Unexpected File

```
def test_unexpected_file():
    expected = {
        "customers",
        "accounts",
    }

    observed = {
        "customers",
        "accounts",
        "debug",
    }

    result = compare_file_sets(expected, observed)

    assert result["unexpected"] == {"debug"}
```

## 71. Unit Test — Completion

```
def test_delivery_complete_when_required_files_complete():
    results = [
        FileResult("customers", "COMPLETE"),
        FileResult("accounts", "COMPLETE"),
        FileResult("transactions", "COMPLETE"),
    ]

    state = aggregate_delivery_state(
        {"customers", "accounts", "transactions"},
        results,
    )

    assert state == "COMPLETE"
```

## 72. Unit Test — Partial Failure

```
def test_delivery_partial_failure():
    results = [
        FileResult("customers", "COMPLETE"),
        FileResult(
            "accounts",
            "RETRYABLE_FAILURE",
            error_code="TIMEOUT",
            retryable=True,
        ),
        FileResult("transactions", "COMPLETE"),
    ]

    state = aggregate_delivery_state(
        {"customers", "accounts", "transactions"},
        results,
    )

    assert state == "PARTIAL_FAILURE"
```

## 73. Integration Test — Idempotent Registration

Test:

1. register a delivery;
2. register the same delivery again;
3. assert one delivery row;
4. register the same file twice;
5. assert one file row.

Expected:

    delivery rows = 1
    file rows = 1

This proves rediscovery is safe.

## 74. Integration Test — Concurrent Claim

Start two workers against the same file.

Expected:

    worker A -> claims file
    worker B -> cannot claim file

There must not be two concurrent extraction jobs for the same logical file.

## 75. Integration Test — Replay

Process a complete delivery.

Then replay it.

Expected:

- no duplicate delivery;
- no duplicate file rows;
- no duplicate staging records;
- deterministic final state.

Replay safety is a core production property.

## 76. Integration Test — One File Fails

Create five files.

Force file 3 to fail.

Expected:

    files 1,2,4,5 = COMPLETE
    file 3 = RETRYABLE_FAILURE
    delivery = PARTIAL_FAILURE

Retry file 3.

Expected:

    file 3 = COMPLETE
    delivery = COMPLETE

## 77. Integration Test — Missing File

Expected:

    A
    B
    C

Observed:

    A
    B

Expected:

    delivery = WAITING_FOR_FILES

Add C.

Then process and reconcile.

Expected:

    delivery = COMPLETE

## 78. Integration Test — Replacement

Process:

    transactions v1

Then submit:

    transactions v2

Expected:

- v1 remains auditable;
- v2 is explicitly identified;
- the system does not silently treat v2 as v1.

## 79. Performance Test

Generate:

    1 delivery
    100 files
    10,000 records per file

Measure:

- discovery time;
- grouping time;
- validation time;
- extraction throughput;
- staging throughput;
- delivery duration;
- peak memory;
- database connections.

Compare worker counts such as:

    1
    2
    4
    8

Do not assume more workers means more throughput.

## 80. Intentional Failure Drill — One File

Delivery:

    A
    B
    C
    D
    E

Force:

    C -> timeout

Expected:

    A = COMPLETE
    B = COMPLETE
    C = RETRYABLE_FAILURE
    D = COMPLETE
    E = COMPLETE

Delivery:

    PARTIAL_FAILURE

Restore the dependency and retry C.

Expected:

    C = COMPLETE
    delivery = COMPLETE

Replay the entire delivery.

Expected:

    no duplicate records
    no duplicate business delivery

## 81. Intentional Failure Drill — Missing File

Remove D.

Run the pipeline.

Expected:

    D = MISSING
    delivery = unresolved

Add D.

Run again.

Expected:

    D = COMPLETE
    delivery = COMPLETE

## 82. Intentional Failure Drill — Duplicate Discovery

Run discovery twice.

Expected:

    same delivery ID
    same file IDs
    no duplicate registry rows

Run extraction twice.

Expected:

    idempotent staging
    no duplicate records

## 83. Intentional Failure Drill — Unexpected File

Add:

    unexpected.csv

Run discovery.

Expected:

    unexpected file is explicitly classified

Then follow the configured policy:

    quarantine
    ignore with audit
    or reject delivery

## 84. Recovery Runbook

When a delivery is PARTIAL_FAILURE:

1. identify delivery ID;
2. list all files;
3. inspect file states;
4. identify failures;
5. classify failures;
6. check retry eligibility;
7. retry only safe failures;
8. inspect quarantined files;
9. correct source artifacts if necessary;
10. revalidate;
11. re-extract;
12. reconcile staging;
13. reconcile delivery;
14. mark complete only after evidence passes.

## 85. Recovery — Missing File

When a required file is missing:

1. verify the contract;
2. verify the arrival window;
3. inspect source storage;
4. inspect manifest or completion marker;
5. confirm that the file is genuinely missing;
6. recover or request the source artifact;
7. register it when available;
8. validate;
9. extract;
10. reconcile.

Never manually mark a required file complete without source evidence.

## 86. Recovery — Quarantined File

When a file is quarantined:

1. inspect the reason;
2. preserve the original artifact;
3. classify the failure;
4. correct source/configuration;
5. create a new version if replacement is allowed;
6. revalidate;
7. re-extract;
8. reconcile;
9. update the audit trail.

## 87. Production Tools You Should Know

### Python concurrent.futures

Use for bounded file-level parallelism.

Know:

- ThreadPoolExecutor;
- ProcessPoolExecutor;
- Future;
- as_completed;
- worker limits;
- exception propagation.

### PostgreSQL

Use for durable:

- delivery state;
- file state;
- idempotency;
- work claiming;
- reconciliation;
- audit metadata.

Know:

- unique constraints;
- transactions;
- ON CONFLICT;
- FOR UPDATE SKIP LOCKED;
- indexes;
- row locking.

### Object Storage SDK

Examples:

- boto3;
- Azure Blob SDK;
- Google Cloud Storage SDK.

Know:

- listing;
- metadata;
- checksums;
- streaming reads;
- retries;
- object semantics.

Use the SDK matching the actual source platform.

## 88. Package Structure

A practical separation:

    etl/
      deliveries/
        contract.py
        identity.py
        grouping.py
        completeness.py
        coordinator.py
        reconciliation.py

      files/
        registry.py
        claiming.py
        validation.py
        extraction.py
        quarantine.py

      extractors/
        csv.py
        json.py
        parquet.py
        xml.py

      staging/
        writer.py

      tests/
        test_delivery_identity.py
        test_grouping.py
        test_completeness.py
        test_concurrency.py
        test_reconciliation.py

The exact layout can vary. Responsibility separation matters more than filenames.

## 89. What Not to Put in the Coordinator

Avoid putting these inside the delivery coordinator:

- CSV parser internals;
- JSON parser internals;
- Parquet reader internals;
- XML namespace handling;
- decompression logic;
- database bulk insert implementation.

Prefer:

    coordinator
        |
        +--> validator
        +--> extractor
        +--> stager
        +--> reconciler

This keeps the system testable.

## 90. Production Implementation Sequence

### Step 1 — Define the delivery contract

Document:

- source;
- delivery type;
- business key;
- required files;
- optional files;
- cardinality;
- manifest;
- completion signal;
- correction policy.

### Step 2 — Define identities

Create delivery ID, file ID, and record ID.

### Step 3 — Parse source naming

Extract delivery key, file type, date, sequence, and version.

### Step 4 — Group discovered files

Group by stable delivery identity.

### Step 5 — Persist delivery state

Create the delivery registry.

### Step 6 — Persist file state

Create the file registry.

### Step 7 — Implement completeness

Compare expected and observed file sets.

### Step 8 — Implement file claiming

Make processing mutually exclusive.

### Step 9 — Implement extraction

Keep format-specific logic outside the coordinator.

### Step 10 — Add bounded concurrency

Set and test a worker limit.

### Step 11 — Add staging idempotency

Use delivery, file, and record identity.

### Step 12 — Add failure classification

Separate retryable and non-retryable failures.

### Step 13 — Add quarantine

Preserve artifacts and metadata.

### Step 14 — Add delivery reconciliation

Derive delivery state from durable file state.

### Step 15 — Add observability

Track both delivery and file metrics/logs.

### Step 16 — Add recovery

Retry only safe failed files.

### Step 17 — Run failure drills

Break one file, remove one required file, duplicate discovery, and introduce an unexpected file.

## 91. Production Checklist

### Contract

- [ ] Delivery identity is deterministic.
- [ ] Required files are explicit.
- [ ] Optional files are explicit.
- [ ] Cardinality is explicit.
- [ ] Unexpected-file policy exists.
- [ ] Correction policy exists.
- [ ] Completion signal exists.

### State

- [ ] Delivery state is persisted.
- [ ] File state is persisted.
- [ ] State transitions are explicit.
- [ ] Stale processing can be detected.
- [ ] Restart does not lose state.

### Identity

- [ ] Delivery ID is stable.
- [ ] File ID is stable.
- [ ] Record ID is stable.
- [ ] Physical path is not incorrectly used as business identity.
- [ ] Hashes are recorded where useful.

### Processing

- [ ] Independent files can run independently.
- [ ] Concurrency is bounded.
- [ ] Source load is controlled.
- [ ] Destination load is controlled.
- [ ] Dependencies are explicit.

### Reliability

- [ ] Duplicate discovery is safe.
- [ ] Duplicate extraction is safe.
- [ ] Retry classification exists.
- [ ] Quarantine exists.
- [ ] Partial failure is supported.
- [ ] Recovery is documented.

### Reconciliation

- [ ] Expected files are compared with observed files.
- [ ] File states are reconciled.
- [ ] Record counts are reconciled where possible.
- [ ] Completion requires evidence.
- [ ] Corrections remain auditable.

### Observability

- [ ] Delivery metrics exist.
- [ ] File metrics exist.
- [ ] Structured logs exist.
- [ ] Delivery ID appears in logs.
- [ ] File ID appears in logs.
- [ ] Failure class is recorded.

### Testing

- [ ] Unit tests exist.
- [ ] Integration tests exist.
- [ ] Replay test exists.
- [ ] Duplicate test exists.
- [ ] Missing-file test exists.
- [ ] Partial-failure test exists.
- [ ] Concurrent-claim test exists.
- [ ] Failure drill has been executed.

## 92. Common Mistakes

### Mistake 1 — Treating each file as a delivery

**Fix:** Define delivery identity separately from file identity.

### Mistake 2 — Grouping by polling cycle

**Fix:** Use producer delivery IDs or deterministic business keys.

### Mistake 3 — Marking complete from worker count

**Fix:** Derive completion from persisted state and reconciliation.

### Mistake 4 — Retrying every file

**Fix:** Retry only safe failed files.

### Mistake 5 — No unexpected-file policy

**Fix:** Explicitly classify unexpected artifacts.

### Mistake 6 — No replacement policy

**Fix:** Version or explicitly supersede corrected files.

### Mistake 7 — Unlimited concurrency

**Fix:** Bound workers and measure resource usage.

### Mistake 8 — Using physical paths as identity

**Fix:** Define logical file identity separately.

### Mistake 9 — Holding database transactions during extraction

**Fix:** Claim in a short transaction, then perform I/O outside it.

### Mistake 10 — No record-level idempotency

**Fix:** Use deterministic record identity.

### Mistake 11 — No delivery freeze

**Fix:** Use a manifest, completion marker, close event, deadline, or explicit completeness boundary.

### Mistake 12 — Treating optional files as required

**Fix:** Encode optional semantics in the contract.

### Mistake 13 — Treating required files as optional

**Fix:** Make required completion a hard prerequisite.

## 93. Debugging Questions

When a delivery is stuck, ask:

1. What is the delivery ID?
2. What files should exist?
3. What files were observed?
4. Which files are missing?
5. Which files failed?
6. Are failures retryable?
7. Is a worker still processing?
8. Did staging commit?
9. Are control totals reconciled?
10. Can the failed file be replayed safely?

If these questions cannot be answered from the registry and logs, the state model is incomplete.

## 94. End-to-End Example

Source sends:

    manifest.json
    customers.csv
    accounts.csv
    transactions.csv

Pipeline:

    1. discover files
    2. derive delivery ID
    3. register delivery
    4. register files
    5. verify expected set
    6. validate artifacts
    7. claim files
    8. extract with bounded concurrency
    9. stage records
    10. reconcile file results
    11. reconcile control totals
    12. mark files complete
    13. mark delivery complete

If transactions fails:

    customers      COMPLETE
    accounts       COMPLETE
    transactions   RETRYABLE_FAILURE

Delivery remains:

    PARTIAL_FAILURE

After retry and successful reconciliation:

    customers      COMPLETE
    accounts       COMPLETE
    transactions   COMPLETE

Delivery becomes:

    COMPLETE

## 95. Design Principle — Coordinate, Do Not Couple

The delivery coordinator should coordinate files without making unrelated files dependent on each other.

Prefer:

    delivery
       |
       +--> A
       +--> B
       +--> C

Only introduce:

    A -> C

when the dependency is real.

This keeps the pipeline parallelizable and recoverable.

## 96. Design Principle — Persist Evidence

A production delivery should leave evidence for:

    what was expected
    what arrived
    what was processed
    what failed
    what was retried
    what was quarantined
    what was reconciled
    why it became complete

If a restart destroys that knowledge, the pipeline state model is incomplete.

## 97. Design Principle — Completion Requires Evidence

Treat COMPLETE as a claim, not a convenience flag.

Evidence should include:

- required file set satisfied;
- file processing succeeded;
- staging committed;
- reconciliation passed;
- no unresolved required-file failures.

This prevents premature publication.

## 98. Definition of Done

You are done with E62 when you can independently implement a pipeline that:

1. receives multiple files for one logical delivery;
2. assigns a deterministic delivery ID;
3. assigns deterministic file IDs;
4. groups files correctly;
5. identifies required and optional files;
6. detects missing and unexpected files;
7. waits for a valid completeness boundary;
8. validates every file;
9. processes files independently where safe;
10. uses bounded concurrency;
11. persists file state;
12. persists delivery state;
13. prevents duplicate registration;
14. prevents duplicate staging;
15. retries safe failures;
16. quarantines unsafe failures;
17. supports partial delivery failure;
18. survives worker restart;
19. reconciles file and record evidence;
20. marks delivery complete only after reconciliation;
21. exposes delivery and file observability;
22. passes normal and failure tests;
23. can be operated from a documented runbook.

If you can implement those without copying this recipe line by line, you understand multi-file extraction.

## 99. What You Learned

The central model is:

    delivery identity
        |
        +---- file identity
        |         |
        |         +---- validation
        |         +---- extraction
        |         +---- staging
        |         +---- retry
        |         +---- quarantine
        |
        +---- completeness
        +---- reconciliation
        +---- final state

The key rules are:

1. Separate delivery identity from file identity.
2. Define the expected file set explicitly.
3. Treat completeness as a delivery-level decision.
4. Keep file processing independently recoverable.
5. Bound concurrency.
6. Make registration and staging idempotent.
7. Persist durable state.
8. Retry only safe failures.
9. Preserve failed artifacts.
10. Reconcile before declaring success.
11. Make corrections explicit.
12. Keep coordination separate from format-specific extraction.

Next:

    E63 — Duplicate File Detection

E63 will focus specifically on proving that a file has already been seen or processed and preventing duplicate delivery from becoming duplicate business data.