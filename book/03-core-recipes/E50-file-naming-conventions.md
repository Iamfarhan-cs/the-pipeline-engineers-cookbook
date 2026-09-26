# E50 — File Naming Conventions

## 1. Problem Recognition

File names are often treated as presentation details.

In production ETL, they can be part of the data contract.

A name such as:

```
payments_20260926_001.csv
```

may encode:

- source system;
- dataset;
- business date;
- delivery sequence;
- environment;
- region;
- version;
- file format.

The production question is:

> **How do I define a deterministic file naming contract that producers can follow, consumers can validate, and operators can use to identify, reject, replay, and reconcile deliveries?**

A naming convention should make invalid or ambiguous files obvious before their contents enter the transformation pipeline.

---

## 2. Concept and Reasoning

### Naming is a contract

A production naming convention should answer:

- What dataset is this?
- Which source produced it?
- What logical period does it represent?
- Is it a full or incremental delivery?
- Which sequence or partition is it?
- Which format/version is being used?
- Can two valid files have the same name?
- How should the consumer detect invalid names?

Example:

```
<source>_<dataset>_<business_date>_<sequence>.<extension>
```

Concrete example:

```
bank_a_payments_20260926_001.csv
```

The filename is not the data itself. It is metadata about the delivery.

### Naming versus identity

Do not confuse:

```
filename
   ≠
file identity
   ≠
business delivery identity
   ≠
content identity
```

E49 introduced these distinctions. E50 focuses on making the filename itself deterministic and machine-validated.

---

## 3. Define the Naming Grammar

Start with a formal grammar.

```
<source>_<dataset>_<date>_<sequence>.csv
```

Contract:

| Component | Rule | Example |
|---|---|---|
| source | lowercase identifier | `bank_a` |
| dataset | approved dataset name | `payments` |
| date | YYYYMMDD | `20260926` |
| sequence | three-digit positive number | `001` |
| extension | `.csv` | `.csv` |

Valid:

```
bank_a_payments_20260926_001.csv
bank_a_payments_20260926_002.csv
```

Invalid:

```
BankA_Payments_26-09-2026_1.csv
payments.csv
bank_a_payments_2026_001.txt
bank_a_payments_20260926.csv
```

The important part is not the exact format. The important part is that the format is explicit.

---

## 4. Choose Delimiters Carefully

Common delimiters include:

- underscore: `_`
- hyphen: `-`
- dot: `.`

For machine-generated names, choose a delimiter that cannot appear inside components or escape it consistently.

Prefer:

```
partner_a_payments_20260926_001.csv
```

over a format whose parser cannot distinguish where one component ends.

If source names can contain underscores, either normalize them or use a grammar that removes ambiguity.

---

## 5. Define Allowed Character Sets

A production contract should define which characters are legal.

A simple contract might allow:

```
[a-z0-9_]
```

and reserve path/platform special characters.

Example:

```
^[a-z0-9]+(?:_[a-z0-9]+)*$
```

for a component containing lowercase words separated by underscores.

Do not create a regex before deciding the business rules.

The regex should implement the contract, not become the contract.

---

## 6. Define Case Sensitivity

Avoid relying on filesystem-specific case behavior.

These may be different strings:

```
payments.csv
Payments.csv
PAYMENTS.CSV
```

Some filesystems treat them as equivalent; others do not.

Choose one rule.

For example:

> All generated filenames are lowercase.

Then validate it.

```
def is_lowercase_name(name: str) -> bool:
    return name == name.lower()
```

This removes platform-dependent ambiguity.

---

## 7. Separate Business Date from File Timestamp

Do not automatically interpret file modification time as business date.

These are different concepts:

```
filename business date
file modified timestamp
file discovery timestamp
file transfer timestamp
file processing timestamp
```

Example:

```
payments_20260925_001.csv
```

may arrive on September 26.

The filename can legitimately represent September 25 while the operating system reports September 26 as the modification date.

Use the naming contract to identify business semantics and filesystem/object metadata to describe physical delivery.

---

## 8. Date Format

Choose a single machine-friendly date format.

A strong default for dates is:

```
YYYYMMDD
```

Example:

```
20260926
```

Advantages:

- fixed width;
- lexicographically sortable;
- unambiguous;
- easy to validate;
- easy to extract with regex.

Avoid ambiguous formats such as:

```
09-26-26
26-09-26
9_26_26
```

unless the producer contract explicitly requires them.

---

## 9. Time Components

If the filename includes time, define the exact representation.

Example:

```
YYYYMMDDTHHMMSSZ
```

produces:

```
20260926T143000Z
```

This explicitly represents UTC.

If local time is used, define the timezone.

Do not create names such as:

```
20260926_1430
```

and leave the timezone implicit.

A filename should not require an operator to guess which clock produced it.

---

## 10. Sequence Numbers

Sequence numbers are useful when one business period produces multiple files.

Example:

```
payments_20260926_001.csv
payments_20260926_002.csv
payments_20260926_003.csv
```

Define:

- starting value;
- width;
- whether gaps are allowed;
- whether duplicates are allowed;
- whether sequence resets daily;
- whether sequence is global;
- whether sequence represents partition or delivery order.

For example:

> Sequence starts at 001 and resets for each business date.

Then:

```
20260926_001
20260926_002
20260926_003
```

is valid, while:

```
20260926_000
```

may be invalid.

---

## 11. Sequence Gaps Are a Separate Problem

A naming contract can validate:

```
001
002
004
```

without knowing whether `003` is missing.

This is important.

Filename validation asks:

> Is this filename structurally valid?

Arrival/completeness logic asks:

> Have all expected files arrived?

Therefore:

```
NAMING VALIDATION
       ≠
DELIVERY COMPLETENESS
```

E51/E52 handle the latter problems.

Do not make a filename parser responsible for detecting missing deliveries.

---

## 12. Full Versus Incremental Files

If the same dataset has different delivery modes, encode the mode explicitly.

Example:

```
bank_a_payments_full_20260926_001.csv
bank_a_payments_delta_20260926_001.csv
```

This is safer than requiring downstream logic to infer whether a file is full or incremental from its contents.

The contract should define the allowed values:

```
full
delta
snapshot
correction
replay
```

Do not allow arbitrary mode strings if the pipeline has finite supported semantics.

---

## 13. Environment Names

Be careful with environment in filenames.

Possible names:

```
prod
stage
dev
test
```

If the source has separate physical environments, including environment can prevent accidental cross-environment processing.

Example:

```
bank_a_prod_payments_20260926_001.csv
```

However, do not duplicate environment metadata unnecessarily if the delivery location already uniquely determines the environment.

The important rule is:

> A file should not be ambiguous about which processing environment it belongs to.

---

## 14. Versioning

A file format may evolve.

Example:

```
bank_a_payments_v1_20260926_001.csv
bank_a_payments_v2_20260926_001.csv
```

Versioning can be useful when the same filename otherwise hides incompatible layouts.

But version numbers must have defined meaning.

For example:

```
v1 = payment_id, amount, currency
v2 = payment_id, amount, currency, merchant_id
```

Do not add `v2` simply because the producer changed something without documenting the compatibility impact.

The filename version identifies a contract. The schema contract must define what that version means.

---

## 15. Extension Is Part of the Contract

Define whether the pipeline accepts:

```
.csv
.csv.gz
.parquet
.json
.json.gz
```

Do not assume that extension alone proves the format.

For example:

```
payments.csv
```

may contain invalid or unexpected bytes.

Filename validation determines the expected format. File validation later proves that the content actually matches it.

---

## 16. Compression Naming

If compression is used, make the layers explicit.

Example:

```
payments_20260926_001.csv.gz
```

Interpretation:

```
.csv  = logical data format
.gz   = transport/storage compression
```

This is clearer than opaque names such as:

```
payments_20260926_001.dat
```

The downstream pipeline can then select the correct decompression and parser stages.

---

## 17. Filename Length

Define a practical maximum filename length.

Avoid contracts that permit arbitrary lengths.

Long names can create problems in:

- filesystem paths;
- object-storage tooling;
- logs;
- dashboards;
- downstream systems;
- manual operations.

Keep machine-generated components compact and meaningful.

Do not put entire metadata payloads into filenames.

---

## 18. Paths and Filenames Are Different Contracts

Do not encode the entire path into the filename grammar.

For example:

```
s3://company/prod/eu/bank_a/payments/2026/09/26/file.csv
```

contains multiple metadata dimensions:

- environment;
- region;
- source;
- dataset;
- date.

The filename might only need:

```
payments_20260926_001.csv
```

Use the path contract for routing and the filename contract for file metadata.

This prevents excessive filename complexity.

---

## 19. Parse, Then Validate Semantics

A regex can identify components:

```
import re

PATTERN = re.compile(
    r"^(?P<source>[a-z0-9]+)"
    r"_(?P<dataset>[a-z0-9]+)"
    r"_(?P<date>\d{8})"
    r"_(?P<sequence>\d{3})"
    r"\.csv$"
)
```

Parsing:

```
match = PATTERN.fullmatch("bank_a_payments_20260926_001.csv")
```

But parsing is not validation.

After parsing, validate:

- source is registered;
- dataset is allowed;
- date is a real calendar date;
- sequence is allowed;
- extension is permitted;
- date falls within an acceptable window;
- version/mode is supported.

---

## 20. Validate Calendar Dates

A string matching `\d{8}` can still be invalid.

For example:

```
20261399
```

matches the shape but is not a valid date.

Use actual date parsing.

```
from datetime import datetime

def parse_business_date(value: str):
    return datetime.strptime(value, "%Y%m%d").date()
```

A naming contract should validate both syntax and meaning.

---

## 21. Parse Into a Structured Object

Do not pass raw filenames throughout the pipeline.

Create a typed representation.

```
from dataclasses import dataclass
from datetime import date


@dataclass(frozen=True)
class FileNameMetadata:
    source: str
    dataset: str
    business_date: date
    sequence: int
    extension: str
```

Then the rest of the pipeline can work with structured metadata:

```
FileNameMetadata
       ↓
Discovery
       ↓
Registration
       ↓
Readiness
       ↓
Validation
       ↓
Processing
```

This keeps string parsing at the boundary.

---

## 22. Complete Filename Parser

```
import re
from dataclasses import dataclass
from datetime import date, datetime


PATTERN = re.compile(
    r"^(?P<source>[a-z0-9]+)"
    r"_(?P<dataset>[a-z0-9]+)"
    r"_(?P<date>\d{8})"
    r"_(?P<sequence>\d{3})"
    r"\.csv$"
)


@dataclass(frozen=True)
class FileNameMetadata:
    source: str
    dataset: str
    business_date: date
    sequence: int


def parse_filename(filename: str) -> FileNameMetadata:
    match = PATTERN.fullmatch(filename)

    if not match:
        raise ValueError(
            f"Filename does not match naming contract: {filename}"
        )

    try:
        business_date = datetime.strptime(
            match.group("date"),
            "%Y%m%d",
        ).date()
    except ValueError as exc:
        raise ValueError(
            f"Invalid business date in filename: {filename}"
        ) from exc

    return FileNameMetadata(
        source=match.group("source"),
        dataset=match.group("dataset"),
        business_date=business_date,
        sequence=int(match.group("sequence")),
    )
```

This provides a deterministic parsing boundary.

---

## 23. Validate Against Configuration

Do not hard-code every producer into the parser.

Use configuration where appropriate.

Example:

```
DATASET_RULES = {
    "payments": {
        "allowed_sources": {"bank_a", "bank_b"},
        "extension": ".csv",
        "sequence_width": 3,
    },
    "refunds": {
        "allowed_sources": {"bank_a"},
        "extension": ".csv",
        "sequence_width": 3,
    },
}
```

The parser extracts structure.

Configuration determines whether that structure is allowed.

---

## 24. Producer-Specific Naming Contracts

Different producers may have different conventions.

Producer A:

```
bank_a_payments_20260926_001.csv
```

Producer B:

```
PAYMENTS-BANK-B-2026-09-26-001.csv
```

Do not force one universal regex onto all sources.

Instead:

```
SOURCE
  ↓
SOURCE-SPECIFIC PARSER
  ↓
COMMON FileNameMetadata
  ↓
COMMON DISCOVERY PIPELINE
```

The internal representation should be standardized even when external contracts differ.

---

## 25. Canonical Internal Representation

Different names can map to the same internal structure.

For example, both producer formats can become:

```
source        = bank_a
dataset       = payments
business_date = 2026-09-26
sequence      = 1
```

This is an important integration pattern:

> Normalize external naming differences at the boundary instead of spreading them through the pipeline.

---

## 26. Collision Detection

A naming convention should reduce collisions, but the pipeline should still detect them.

Suppose two files have the same filename in different source locations.

They may be:

- the same delivery copied twice;
- different producer environments;
- different versions;
- conflicting deliveries.

The registry should retain source/location information and, where required, content identity.

Never assume that a globally unique-looking filename is actually globally unique.

---

## 27. Business Identity

Sometimes the filename identifies a logical delivery.

Example:

```
source + dataset + business_date + sequence
```

can form:

```
bank_a/payments/2026-09-26/001
```

Store this business identity separately from the physical location.

This lets the pipeline distinguish the same logical delivery from different physical copies.

Business identity should be defined by the source contract, not guessed from filename similarity.

---

## 28. Filename Validation Before File Reading

The correct sequence is:

```
FILE DISCOVERED
      ↓
FILENAME PARSED
      ↓
FILENAME VALIDATED
      ↓
FILE REGISTERED
      ↓
READINESS CHECK
      ↓
CONTENT VALIDATION
      ↓
PROCESS
```

Rejecting an invalid filename before opening a huge file saves network transfer, disk I/O, CPU, parser work, staging, and operator time.

This is boundary validation.

---

## 29. Invalid Filename Handling

Do not simply ignore invalid files.

Record:

- filename;
- source;
- discovery run;
- rejection reason;
- discovered timestamp;
- physical location.

Example:

```
status = REJECTED
reason = INVALID_FILENAME
```

This creates operational evidence.

---

## 30. Naming Contract Versioning

Naming contracts themselves evolve.

Example:

```
v1:
payments_20260926_001.csv

v2:
bank_a_payments_20260926_001.csv
```

Define:

```
contract_version
supported_from
deprecated_after
migration_policy
```

Example:

| Version | Status | Policy |
|---|---|---|
| v1 | deprecated | accept temporarily |
| v2 | active | preferred |
| v3 | planned | reject until enabled |

Do not silently accept every historical format forever.

---

## 31. Producer Migration

A controlled migration can look like:

```
v1 accepted
     ↓
v2 introduced
     ↓
both accepted temporarily
     ↓
monitor v1
     ↓
producer completes migration
     ↓
v1 rejected
```

Do not remove v1 immediately if files may still be in flight.

Also do not keep deprecated formats indefinitely without ownership.

---

## 32. Testing

### Unit tests

Test:

- valid filename;
- invalid filename;
- wrong case;
- wrong extension;
- missing component;
- extra component;
- invalid date;
- invalid sequence;
- unsupported source;
- unsupported dataset;
- unsupported contract version;
- boundary dates;
- sequence width;
- special characters.

Example:

```
def test_valid_filename():
    result = parse_filename(
        "bank_a_payments_20260926_001.csv"
    )

    assert result.source == "bank_a"
    assert result.dataset == "payments"
    assert result.sequence == 1
```

### Property-style tests

Useful invariants include:

- valid generated names always parse;
- parsed metadata can be reconstructed into the canonical name;
- invalid characters are rejected;
- unsupported datasets are rejected.

---

## 33. Intentional Failure Drills

### Drill 1 — Change the date format

Send:

```
bank_a_payments_26-09-2026_001.csv
```

Verify rejection.

### Drill 2 — Change case

Send:

```
BANK_A_PAYMENTS_20260926_001.csv
```

Verify the configured case policy is enforced.

### Drill 3 — Remove sequence

Send:

```
bank_a_payments_20260926.csv
```

Verify rejection.

### Drill 4 — Create an impossible date

Send:

```
bank_a_payments_20261399_001.csv
```

Verify semantic validation rejects it.

### Drill 5 — Add an unsupported dataset

Send:

```
bank_a_unknown_20260926_001.csv
```

Verify rejection.

### Drill 6 — Introduce a new contract version

Verify old and new naming contracts follow the configured migration policy.

### Drill 7 — Create a filename collision

Place logically conflicting files in different locations and verify identity handling.

---

## 34. Observability

Track:

```
filename_validation_total
filename_validation_success_total
filename_validation_rejected_total
filename_contract_version_total
filename_parse_error_total
filename_business_date_error_total
filename_sequence_error_total
filename_unsupported_dataset_total
```

Useful structured log fields:

```
source_system
filename
contract_version
parsed_dataset
parsed_business_date
parsed_sequence
validation_status
rejection_reason
discovery_run_id
```

Avoid putting every filename into metric labels. Filenames can create high-cardinality metrics.

---

## 35. Recovery Runbook

### Producer sends invalid filenames

1. Capture the rejected filename.
2. Identify the violated rule.
3. Confirm the producer contract.
4. Do not manually rename production files without an approved procedure.
5. Request correction or use a controlled mapping if supported.
6. Re-run discovery after correction.
7. Record the resolution.

### Producer changes naming format unexpectedly

1. Stop automatic acceptance of the unknown format.
2. Identify the new contract.
3. Compare old and new semantics.
4. Create or update the parser.
5. Add tests.
6. Deploy the new contract version.
7. Reprocess rejected files only after validation.

### Duplicate business deliveries appear

1. Compare parsed business identity.
2. Compare source location.
3. Compare content hashes where available.
4. Determine whether the producer replayed a delivery.
5. Keep one canonical processing record.
6. Quarantine ambiguous duplicates.

### Valid files are being rejected

1. Inspect parser logs.
2. Inspect contract version.
3. Compare actual filename with the expected grammar.
4. Test the exact filename.
5. Fix the contract implementation rather than manually bypassing validation.

---

## 36. Production Tools You Should Know

### 1. Python `re`

Learn regular expressions for controlled filename parsing, while keeping business validation outside the regex where possible.

### 2. Python `pathlib`

Learn safe path handling, filename extraction, suffix handling, and filesystem boundary checks.

### 3. Object-storage/SFTP listing APIs

Learn how remote object keys and filenames are discovered, paginated, filtered, and mapped into a common internal naming contract.

The tool is secondary.

The production skill is:

```
EXTERNAL NAME
      ↓
PARSE
      ↓
VALIDATE
      ↓
NORMALIZE
      ↓
REGISTER
      ↓
PROCESS
```

---

## 37. Common Mistakes

1. Treating filenames as arbitrary strings.
2. Using `.*` as the filename parser.
3. Making regex responsible for every business rule.
4. Accepting multiple undocumented naming formats.
5. Ignoring case sensitivity.
6. Using file modification time as business date.
7. Allowing ambiguous date formats.
8. Not defining sequence semantics.
9. Treating sequence gaps as filename-validation failures.
10. Using filenames as the only identity.
11. Putting too much metadata into filenames.
12. Mixing path and filename contracts.
13. Ignoring contract versioning.
14. Changing producer rules without a migration period.
15. Silently renaming invalid production files.
16. Processing content before filename validation.
17. Logging rejection without durable evidence.
18. Using high-cardinality filenames as metric labels.
19. Assuming a filename that looks unique is globally unique.
20. Allowing source-specific naming differences to leak throughout the pipeline.

---

## 38. Definition of Done

You can independently:

- define a machine-readable filename contract;
- choose delimiters and allowed characters;
- define date and time semantics;
- define sequence semantics;
- distinguish filename metadata from file identity;
- parse filenames deterministically;
- validate syntax and business semantics separately;
- normalize different producer conventions;
- represent parsed metadata as structured data;
- version naming contracts;
- migrate producers safely;
- reject invalid files with durable reasons;
- detect naming collisions;
- connect naming validation with discovery;
- write unit and integration tests;
- run intentional failure drills;
- observe naming failures;
- recover producer naming problems safely.

---

## 39. What You Learned

> **A production filename is an interface contract, not decoration.**

The reliable pattern is:

```
DEFINE NAMING CONTRACT
        ↓
DISCOVER FILE
        ↓
PARSE NAME
        ↓
VALIDATE SYNTAX
        ↓
VALIDATE SEMANTICS
        ↓
NORMALIZE METADATA
        ↓
REGISTER DELIVERY
        ↓
CHECK READINESS
        ↓
VALIDATE CONTENT
        ↓
PROCESS
```

The key question is:

> **Can I look at a filename and deterministically determine what delivery it represents, whether it is valid, which contract version produced it, and what the pipeline should do with it?**
