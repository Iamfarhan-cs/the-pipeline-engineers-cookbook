# Recipe 34 — Handle Schema Changes

A production pipeline does not process data in a vacuum. It processes data according to a contract between producers, consumers, storage, and downstream systems.

    producer -> schema -> consumer -> downstream

If the structure or meaning of data changes, the pipeline can fail, silently corrupt data, or produce incomplete results.

This recipe teaches how to detect, classify, safely introduce, test, observe, and recover from schema changes.

Core lifecycle:

    Discover current schema
           |
           v
    Detect schema change
           |
           v
    Classify compatibility
       /         |         \
    safe      migration   breaking
      |           |           |
      v           v           v
   continue    deploy    block/quarantine
                  |           |
                  +------> recover
                              |
                              v
                         validate again

> A schema change is a production change, even when the application still starts successfully.

---

## 1. Goal

By the end of this recipe, you should be able to:

- recognize schema-change failures in real pipelines
- inspect producer, consumer, and database schemas
- detect added, removed, and renamed fields
- detect type, nullability, constraint, enum, and nested-structure changes
- detect semantic changes hidden behind unchanged physical types
- distinguish backward-compatible from breaking changes
- define compatibility rules
- snapshot and diff schemas
- validate incoming data against supported schemas
- implement safe additive changes
- implement expand-and-contract migrations
- use schema versions and contracts
- test mixed-version deployments
- intentionally break schemas
- recover from incompatible deployments
- reconcile data after recovery.

---

## 2. Why Schema Changes Are Dangerous

Original payload:

~~~json
{
  "event_id": "E1",
  "amount": 100,
  "currency": "EUR"
}
~~~

Adding an optional field may be safe:

~~~json
{
  "event_id": "E1",
  "amount": 100,
  "currency": "EUR",
  "country": "PK"
}
~~~

But changing the type can break consumers:

~~~json
{
  "event_id": "E1",
  "amount": "100 EUR",
  "currency": "EUR"
}
~~~

Removing a required field can also break the pipeline.

Possible consequences:

- parser failures
- NULL insertion
- transformation failures
- incorrect joins
- broken reports
- incorrect aggregates
- quarantined records
- silent field loss.

---

## 3. Schema vs Data Quality

Data quality asks:

    Does this value satisfy the current contract?

Schema management asks:

    Is this still the contract the consumer expects?

Example:

    currency = XYZ

The schema can be valid while the value is invalid.

Different problem:

    currency field removed

That is a schema change.

The controls work together:

    Schema validation
          |
          v
    Is the structure compatible?
          |
          v
    Data validation
          |
          v
    Are the values valid?

---

## 4. Schema Change Types

Recognize at least:

    ADD FIELD
    REMOVE FIELD
    RENAME FIELD
    CHANGE TYPE
    CHANGE NULLABILITY
    CHANGE DEFAULT
    CHANGE CONSTRAINT
    CHANGE ENUM / DOMAIN
    CHANGE NESTED STRUCTURE
    CHANGE FIELD FORMAT
    CHANGE FIELD MEANING
    CHANGE KEY SEMANTICS

Not every change has the same risk.

Compatibility classification determines the deployment strategy.

---

## 5. Added Field

Before:

    id
    amount
    currency

After:

    id
    amount
    currency
    country

This can be backward compatible when:

- the field is optional
- old consumers ignore unknown fields
- storage accepts the change
- existing business logic does not depend on the new field.

Do not assume every additive change is safe. A required field or a field that changes business behavior can still be breaking.

---

## 6. Removed Field

Before:

    id
    amount
    currency

After:

    id
    amount

If an active consumer requires currency, the change is breaking.

Before removing a field, identify:

- all consumers
- reports
- transformations
- database columns
- contracts
- historical-data requirements.

---

## 7. Renamed Field

Example:

    customer_id -> client_id

A consumer cannot safely infer that these names are equivalent.

Safer migration:

    old field + new field
             |
             v
       compatibility layer
             |
             v
       migrate consumers
             |
             v
       remove old field later

---

## 8. Type Change

Examples:

    INTEGER -> BIGINT
    INTEGER -> DECIMAL
    DECIMAL -> STRING
    STRING -> OBJECT

Check actual behavior in:

- serializers
- parsers
- application code
- database loaders
- SQL expressions
- indexes
- downstream consumers.

Do not classify type changes by intuition alone.

---

## 9. Nullability, Defaults, and Constraints

These are part of the contract too.

Examples:

    nullable -> NOT NULL
    NOT NULL -> nullable
    DEFAULT 'PENDING' -> DEFAULT 'NEW'
    amount >= 0 -> amount > 0
    UNIQUE constraint added
    foreign key added

A change may preserve column names and types while changing what data is accepted or what values mean.

---

## 10. Enum and Domain Changes

Suppose:

    SUCCESS | FAILED | PENDING

becomes:

    SUCCESS | FAILED | PENDING | CANCELLED

An old consumer may reject CANCELLED.

Adding an enum value is not automatically safe. Test the behavior of every relevant consumer.

---

## 11. Nested Schema Changes

Nested structures can change while keeping the outer field name unchanged.

Before:

~~~json
{
  "customer": {
    "id": "C1",
    "name": "Farhan"
  }
}
~~~

After:

~~~json
{
  "customer": {
    "id": "C1",
    "name": {
      "first": "Farhan",
      "last": "Khan"
    }
  }
}
~~~

The field still exists, but its type and meaning changed.

Schema comparison must therefore handle nested structures recursively when the format requires it.

---

## 12. Semantic Schema Changes

Some of the most dangerous changes are invisible to structural schema comparison.

Example:

    amount = cents

becomes:

    amount = major currency units

Both may still be INTEGER.

Other examples:

- UTC timestamp becomes local time
- internal customer ID becomes external customer ID
- gross amount becomes net amount
- percentage changes from 0-100 to 0-1
- status meaning changes.

> Schema management must protect semantics, not only column names and data types.

---

## 13. Compatibility Models

### Backward compatibility

Can the new consumer read old data?

### Forward compatibility

Can the old consumer read new data?

### Full compatibility

Can old and new producers and consumers safely interact during rollout?

Compatibility depends on deployment order and the serialization/database technology.

---

## 14. Deployment Order Matters

Producer first:

    producer v2 -> old consumer

The old consumer must survive the new schema.

Consumer first:

    new consumer <- producer v1

The new consumer must survive old data.

Therefore a schema change cannot be called safe without considering rollout order.

---

## 15. Expand-and-Contract

A robust migration pattern is:

### Expand

Add the new structure while keeping the old structure.

### Migrate

Update consumers and backfill data where required.

### Switch

Move reads/writes to the new structure.

### Contract

Remove the old structure only after all consumers have migrated.

Example:

    customer_name
          +
    customer_first_name
    customer_last_name

After verification:

    customer_name removed

Do not contract before proving that active consumers no longer depend on the old contract.

---

## 16. Schema Versioning

For event-based systems, explicit schema versions can make compatibility easier to manage.

~~~json
{
  "event_name": "payment.created",
  "event_version": 2,
  "event_id": "E1",
  "amount": 100,
  "currency": "EUR"
}
~~~

Versioning identifies the contract. It does not automatically make an unsafe change safe.

---

## 17. Schema Registry Concept

A registry can maintain:

    subject
    schema
    version
    compatibility policy
    created_at

Conceptually:

    payment.created
        |
        +-- v1
        +-- v2
        +-- v3

The implementation may be a dedicated registry or a version-controlled schema directory.

The important capability is controlled schema history and compatibility policy.

---

## 18. Schema Snapshots

Capture a known-good schema before accepting a new version.

Example:

~~~text
schema_id = payment.created
version = 3
captured_at = 2026-09-26T10:00:00Z

fields:
  event_id     TEXT      required
  amount       DECIMAL   required
  currency     TEXT      required
  country      TEXT      optional
~~~

Historical snapshots provide the baseline for drift detection and incident investigation.

---

## 19. Schema Diff

A useful schema diff should say exactly what changed.

~~~text
ADDED:
    country TEXT NULL

REMOVED:
    customer_type

CHANGED:
    amount INTEGER -> DECIMAL

NULLABILITY:
    currency NOT NULL -> NULL

CONSTRAINT:
    amount >= 0 -> amount > 0
~~~

Do not reduce the output to:

    schema changed

The engineer needs the actual difference.

---

## 20. PostgreSQL Schema Inspection

For database-backed pipelines:

~~~sql
SELECT
    column_name,
    data_type,
    is_nullable,
    column_default
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name = 'payments'
ORDER BY ordinal_position;
~~~

Inspect constraints separately.

This metadata can be converted into a canonical schema snapshot.

---

## 21. Schema Fingerprint

A schema fingerprint can detect change quickly.

Conceptually:

    fingerprint = hash(canonical schema definition)

Include relevant properties such as:

- column name
- data type
- nullability
- default
- relevant constraints.

If:

    current fingerprint != expected fingerprint

trigger investigation.

A fingerprint detects change; it does not explain the change. Always produce the actual diff.

---

## 22. Validate Schema at the Boundary

A robust ingestion boundary can perform:

    receive
      |
      v
    identify schema version
      |
      v
    check supported versions
      |
      +----------------+
      |                |
   supported      unsupported
      |                |
      v                v
   validate        quarantine
      |
      v
    process

This prevents incompatible data from silently entering downstream systems.

---

## 23. Unknown-Field Policy

Choose explicitly between:

### Strict

Reject unknown fields.

### Permissive

Ignore unknown fields.

### Capture

Preserve unknown fields for later processing.

None is universally correct.

The policy must match the contract and operational risk.

---

## 24. Missing Required Fields

Expected:

    id
    amount
    currency

Actual:

    id
    amount

Do not silently invent currency unless the contract defines a safe default.

Possible lifecycle:

    schema validation failure
         -> quarantine
         -> alert
         -> investigate producer
         -> fix contract
         -> replay affected data
         -> reconcile

---

## 25. Schema Change and Dead Letter

Schema violations connect directly to Recipe 33.

    schema mismatch
         |
         v
      classify
         |
         v
      quarantine
         |
         v
    identify version
         |
         v
      fix contract
         |
         v
       validate
         |
         v
        replay
         |
         v
      reconcile

Treat a schema failure as a contract incident, not merely a parser exception.

---

## 26. Database Migration Safety

Dangerous migration:

    DROP COLUMN customer_id

while active application code still queries customer_id.

Safer sequence:

    add new structure
         |
         v
    deploy compatible code
         |
         v
    backfill
         |
         v
    switch reads/writes
         |
         v
    validate
         |
         v
    remove old structure later

This is the database form of expand-and-contract.

---

## 27. Intermediate Migration States

Never assume a migration is either completely applied or completely absent.

Example:

    add column       PASS
    backfill         PASS
    deploy consumer  FAIL
    remove old field NOT RUN

The database may be in an intermediate state.

Design intermediate states so that active code can continue safely.

This is especially important for zero-downtime deployment.

---

## 28. Migration Metadata

Track migration state:

    migration_id
    version
    description
    applied_at
    applied_by
    checksum
    status

For long migrations also track:

    backfill progress
    affected row count
    validation result

The objective is to know exactly which schema state each environment has.

---

## 29. Contract Testing

Contract tests verify producer and consumer compatibility before deployment.

Example:

    Producer v2
         |
         v
    Consumer validates v2
         |
         v
       PASS

Also test old/new combinations when rolling deployments require them.

Run contract tests in CI before production whenever practical.

---

## 30. Compatibility Matrix

| Change | Old consumer reads new data | New consumer reads old data | Typical risk |
|---|---|---|---|
| Optional field added | Often possible | Usually possible | Lower |
| Required field added | Often breaking | Often possible | High |
| Optional field removed | May break if referenced | Usually possible | Medium/High |
| Required field removed | Breaking | Usually possible | High |
| Type widened | Depends | Depends | Medium |
| Type narrowed | Often breaking | Often breaking | High |
| Rename | Usually breaking | Usually breaking | High |
| Enum value added | Depends | Usually possible | Medium/High |
| Meaning changed | Structurally invisible | Structurally invisible | Very high |

This table is a decision aid, not a universal compatibility guarantee. Test actual producer, serializer, parser, database, and consumer behavior.

---

## 31. Observability During Migration

Monitor:

    old schema volume
    new schema volume
    schema validation failures
    parse failures
    DQ failures
    quarantine backlog
    replay backlog
    reconciliation differences
    downstream failures

A migration is not complete when deployment succeeds. It is complete when the new contract works and the old contract can be retired safely.

---

## 32. Test — Add Optional Field

Start:

~~~json
{
  "id": "E1",
  "amount": 100
}
~~~

Change:

~~~json
{
  "id": "E1",
  "amount": 100,
  "country": "PK"
}
~~~

Expected when unknown optional fields are permitted:

    compatibility = PASS
    existing consumer behavior remains correct

Verify downstream data.

---

## 33. Test — Remove Required Field

Expected:

    id
    amount
    currency

Input:

~~~json
{
  "id": "E1",
  "amount": 100
}
~~~

Expected:

    schema validation fails
    record is quarantined
    failure code is stable
    original payload is preserved

---

## 34. Test — Rename Field

Change:

    customer_id -> client_id

Expected:

    compatibility mapping or migration is required

Test both:

    old producer -> new consumer
    new producer -> old consumer

according to the rollout strategy.

---

## 35. Test — Type Change

Change:

    amount: integer
        ->
    amount: string

Input:

~~~json
{
  "id": "E1",
  "amount": "100",
  "currency": "EUR"
}
~~~

Expected:

    compatibility behavior is explicitly tested
    no silent numeric corruption occurs

---

## 36. Test — Nullability

Change:

    currency NOT NULL
        ->
    currency NULL

and test the reverse.

Test:

    currency present
    currency NULL

Expected behavior must be explicit.

---

## 37. Test — Enum Expansion

Existing:

    SUCCESS
    FAILED
    PENDING

New:

    CANCELLED

Send CANCELLED to an old consumer.

Expected:

    explicit compatibility behavior
    or controlled rejection

Never assume unknown enum values are safe.

---

## 38. Test — Semantic Change

Keep:

    amount INTEGER

Change meaning:

    old = cents
    new = major currency units

Structural schema comparison may pass.

Expected:

    semantic contract test detects the incompatible change

This proves why schema management cannot depend only on names and types.

---

## 39. Test — Breaking Database Migration

Start with active code using customer_id.

Attempt to remove it.

Expected:

    unsafe migration is blocked or prevented

Then implement expand-and-contract.

Expected:

    active consumers continue working throughout the migration

---

## 40. Test — Mixed Versions

Run:

    producer v1
    producer v2
    consumer supporting v1 + v2

Expected:

    both versions process correctly

Retire v1 only after:

    v1 traffic = 0
    historical requirements are satisfied
    reconciliation passes

---

## 41. Intentionally Break the Pipeline

Perform these drills in a safe environment.

### Drill 1 — Add an unexpected required field

Expected: compatibility failure.

### Drill 2 — Remove a required field

Expected: schema validation failure.

### Drill 3 — Rename a field without migration

Expected: contract failure.

### Drill 4 — Change a numeric type

Expected: type mismatch detected.

### Drill 5 — Add an unknown enum value

Expected: compatibility policy is triggered.

### Drill 6 — Change field meaning without changing type

Expected: semantic contract test detects it.

### Drill 7 — Deploy consumer before producer

Expected: old data remains processable.

### Drill 8 — Deploy producer before consumer

Expected: compatibility protection prevents unsafe processing.

### Drill 9 — Fail a migration halfway

Expected: intermediate state remains recoverable.

### Drill 10 — Replay after schema repair

Expected: repaired records process once and reconcile successfully.

---

## 42. Incident Investigation

Suppose:

    schema_validation_failures = 12,000

Investigate systematically.

### Step 1 — Identify schema

    payment.created

### Step 2 — Identify producer/version

    producer v4

### Step 3 — Compare actual and expected schema

Example:

    currency
    expected = STRING
    actual   = OBJECT

### Step 4 — Determine rollout timing

Ask:

    When did v4 deploy?
    Which environments are affected?
    Which consumers received it?

### Step 5 — Determine compatibility

If old consumers cannot process v4:

    quarantine
    block unsafe publication
    fix rollout

### Step 6 — Determine affected scope

Identify:

    first failure time
    affected partitions
    affected IDs
    schema version

### Step 7 — Fix contract

Possible actions:

    rollback producer
    deploy compatible consumer
    add mapping
    repair affected data

### Step 8 — Replay safely

Only after validation succeeds.

### Step 9 — Reconcile

Check:

    counts
    IDs
    important fields
    aggregates

### Step 10 — Record root cause

Document the change, cause, affected scope, mitigation, recovery, and prevention.

---

## 43. Recovery Strategies

### Additive field

Continue when compatibility permits.

### Removed required field

Restore the field or deploy compatible consumers.

### Rename

Use explicit mapping or expand-and-contract.

### Type change

Use versioned parsing or an explicit transformation.

### Semantic change

Treat it as a contract migration.

### Broken database migration

Restore a compatible application/schema state, then migrate safely.

Never blindly replay data against an incompatible schema.

---

## 44. Schema Rollback

Rollback is not always simply:

    deploy old application

If the database has already changed, old code may fail.

Consider together:

    application version
    schema version
    data version

Design migrations for safe rollback or forward-compatible recovery.

---

## 45. Historical Data

Historical data may remain on an old schema.

Decide explicitly:

    Can the new pipeline read old records?

If yes:

    support v1 + v2

If no:

    migrate historical data first

Never assume historical data automatically follows the newest schema.

---

## 46. Schema Version vs Application Version

They can evolve independently.

Example:

    application v5
    event schema v3
    database schema v12

Record them separately when they have independent lifecycle semantics.

During incidents this answers:

    Which application processed the data?
    Which schema did it expect?
    Which database structure did it write?

---

## 47. Reconciliation After Migration

After migration, reconcile:

    record counts
    unique IDs
    NULL counts
    field distributions
    aggregate totals
    status distributions
    important business values

Example:

    Before:
    count = 1,000,000
    amount = 50,000,000

    After:
    count = 1,000,000
    amount = 50,000,000

Also verify semantics. Preserving row count does not prove that the meaning of the data was preserved.

---

## 48. Practical Implementation Sequence

For an existing pipeline:

    1. Inventory producer schemas
    2. Inventory consumer expectations
    3. Identify database schemas
    4. Identify schema ownership
    5. Snapshot current schemas
    6. Define compatibility policy
    7. Define supported schema versions
    8. Add schema comparison
    9. Add boundary validation
    10. Add stable schema failure codes
    11. Add contract tests
    12. Implement additive changes safely
    13. Implement expand-and-contract for breaking changes
    14. Add migration metadata
    15. Add metrics and alerts
    16. Test mixed-version deployment
    17. Break schemas intentionally
    18. Exercise rollback/recovery
    19. Replay affected records where required
    20. Reconcile after migration
    21. Retire old schema only after verification
    22. Document migration and rollback runbook

---

## 49. Common Mistakes

### Mistake 1 — Treating every added field as breaking

Optional additive fields can be safe.

### Mistake 2 — Treating every added field as safe

Required fields can break old consumers.

### Mistake 3 — Removing fields immediately

Active consumers may still depend on them.

### Mistake 4 — Renaming without migration

A rename is usually a breaking contract change.

### Mistake 5 — Comparing only names and types

Semantic changes can be invisible.

### Mistake 6 — Migrating database and application simultaneously

Intermediate states may be unsafe.

### Mistake 7 — No schema snapshots

Drift becomes difficult to detect or explain.

### Mistake 8 — No compatibility tests

Production becomes the integration test.

### Mistake 9 — Ignoring old consumers

Rolling deployments create mixed versions.

### Mistake 10 — Replaying before schema repair

The same failure occurs again.

### Mistake 11 — Dropping old schema immediately

Rollback becomes harder.

### Mistake 12 — Assuming schema version equals application version

They can evolve independently.

---

## 50. Production Runbook

When a schema change is detected:

    1. Identify the affected schema
    2. Identify producer and consumer versions
    3. Capture actual schema
    4. Compare with expected schema
    5. Classify every change
    6. Determine backward compatibility
    7. Determine forward compatibility
    8. Determine affected scope
    9. Stop unsafe publication if required
    10. Quarantine incompatible records
    11. Decide rollback vs forward migration
    12. Deploy compatible code/schema
    13. Validate repaired or newly accepted data
    14. Replay affected records
    15. Reconcile counts and business controls
    16. Monitor old/new schema traffic
    17. Verify unresolved failures are handled
    18. Retire old schema only after evidence supports it
    19. Record root cause and prevention

---


## Production Tools You Should Know

These tools solve or provide production implementations of concepts covered in this recipe. Learn what the tool provides, but understand the underlying problem first.

| Tool | What to know |
|---|---|
| **Confluent Schema Registry** | Central schema storage, compatibility checks, and versioning. |
| **AWS Glue Schema Registry** | Managed schema registry and compatibility controls. |
| **Apicurio Registry** | Open-source schema and artifact registry. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the engineering mechanism.

---

## Implementation Lab — Runnable Schema-Change Detection and Migration

The implementation below creates a canonical schema representation, fingerprints it, diffs versions, validates payloads, and demonstrates an expand-and-contract database migration.

### 1. Canonical schema representation

```python
# src/schema_contract.py
import hashlib
import json


def canonical_schema(fields: dict[str, dict]) -> str:
    return json.dumps(fields, sort_keys=True, separators=(",", ":"))


def fingerprint(fields: dict[str, dict]) -> str:
    canonical = canonical_schema(fields)
    return hashlib.sha256(canonical.encode()).hexdigest()


def diff_schema(old: dict[str, dict], new: dict[str, dict]) -> dict:
    old_keys = set(old)
    new_keys = set(new)

    added = sorted(new_keys - old_keys)
    removed = sorted(old_keys - new_keys)
    changed = {
        key: {"old": old[key], "new": new[key]}
        for key in old_keys & new_keys
        if old[key] != new[key]
    }

    return {
        "added": added,
        "removed": removed,
        "changed": changed,
    }
```

### 2. Boundary validation

```python
SUPPORTED = {
    1: {
        "event_id": {"type": "string", "required": True},
        "amount": {"type": "number", "required": True},
        "currency": {"type": "string", "required": True},
    },
    2: {
        "event_id": {"type": "string", "required": True},
        "amount": {"type": "number", "required": True},
        "currency": {"type": "string", "required": True},
        "country": {"type": "string", "required": False},
    },
}


def validate_payload(payload: dict, version: int) -> list[str]:
    schema = SUPPORTED.get(version)
    if schema is None:
        return ["UNSUPPORTED_SCHEMA_VERSION"]

    errors = []

    for field, definition in schema.items():
        if definition["required"] and field not in payload:
            errors.append(f"REQUIRED_FIELD_MISSING:{field}")

    if "amount" in payload and not isinstance(payload["amount"], (int, float)):
        errors.append("TYPE_ERROR:amount")

    return errors
```

### 3. Tests

```python
# tests/test_schema_contract.py
from src.schema_contract import (
    diff_schema,
    fingerprint,
    validate_payload,
)


def test_added_optional_field_is_detected():
    old = {
        "id": {"type": "string"},
        "amount": {"type": "number"},
    }
    new = {
        **old,
        "country": {"type": "string", "required": False},
    }

    diff = diff_schema(old, new)

    assert diff["added"] == ["country"]
    assert diff["removed"] == []


def test_type_change_is_detected():
    old = {"amount": {"type": "number"}}
    new = {"amount": {"type": "string"}}

    diff = diff_schema(old, new)

    assert "amount" in diff["changed"]


def test_schema_fingerprint_changes():
    old = {"id": {"type": "string"}}
    new = {"id": {"type": "integer"}}

    assert fingerprint(old) != fingerprint(new)


def test_missing_required_field_fails():
    errors = validate_payload(
        {"event_id": "e1", "amount": 10},
        version=1,
    )

    assert "REQUIRED_FIELD_MISSING:currency" in errors


def test_unsupported_version_fails():
    assert validate_payload({}, version=99) == ["UNSUPPORTED_SCHEMA_VERSION"]
```

### 4. PostgreSQL expand-and-contract example

Unsafe:

```sql
ALTER TABLE customers DROP COLUMN customer_name;
```

Safer migration:

```sql
-- Expand
ALTER TABLE customers
    ADD COLUMN customer_first_name TEXT,
    ADD COLUMN customer_last_name TEXT;
```

Backfill:

```sql
UPDATE customers
SET
    customer_first_name = split_part(customer_name, ' ', 1),
    customer_last_name = NULLIF(
        substring(customer_name from position(' ' in customer_name) + 1),
        ''
    )
WHERE customer_first_name IS NULL;
```

Deploy application code that can read both representations.

After validation and consumer migration:

```sql
-- Contract only after all consumers have migrated.
ALTER TABLE customers
    DROP COLUMN customer_name;
```

### 5. Schema-change incident drill

Run:

```python
old = {
    "amount": {"type": "number"},
    "currency": {"type": "string"},
}

new = {
    "amount": {"type": "string"},
    "currency": {"type": "string"},
}

print(diff_schema(old, new))
```

Expected result:

```text
amount appears under changed
```

Now send:

```json
{"event_id": "e1", "amount": "100", "currency": "EUR"}
```

Version 1 must reject the payload.

Repair the producer or add an explicit versioned transformation, then rerun the contract tests.

### 6. Intentionally break the detector

Change:

```python
if old[key] != new[key]
```

to:

```python
if False
```

The type-change test must fail.

Restore the comparison.

The critical production property is:

```text
schema changes
    ↓
detected
    ↓
classified
    ↓
compatibility decision
    ↓
safe migration or quarantine
    ↓
validation
    ↓
reconciliation
```


## 51. Definition of Done

- [ ] Producer schemas are known.
- [ ] Consumer expectations are known.
- [ ] Database schemas are documented where relevant.
- [ ] Schema ownership is defined.
- [ ] Schema snapshots exist.
- [ ] Schema changes can be detected.
- [ ] Schema diffs explain actual changes.
- [ ] Added fields are classified.
- [ ] Removed fields are classified.
- [ ] Renames are explicitly handled.
- [ ] Type changes are classified.
- [ ] Nullability changes are classified.
- [ ] Constraint changes are considered.
- [ ] Enum/domain changes are tested.
- [ ] Semantic changes are considered.
- [ ] Compatibility policy is explicit.
- [ ] Supported schema versions are explicit.
- [ ] Boundary schema validation exists.
- [ ] Stable schema failure codes exist.
- [ ] Incompatible records enter the DQ/dead-letter lifecycle.
- [ ] Contract tests exist.
- [ ] Database migrations use safe rollout patterns.
- [ ] Expand-and-contract is used where appropriate.
- [ ] Migration state is auditable.
- [ ] Mixed-version deployments are tested.
- [ ] Rollback/recovery is tested.
- [ ] Replayed data is idempotent.
- [ ] Reconciliation runs after migration/recovery.
- [ ] Schema metrics and alerts exist.
- [ ] Old schema retirement is controlled.
- [ ] Failure drills have been performed.
- [ ] Final state is independently verified.

---

## 52. What You Learned

Schema management is not:

    add a column

It is:

    understand the contract
           |
           v
    detect the change
           |
           v
    classify compatibility
           |
           v
    choose deployment strategy
           |
           v
    migrate safely
           |
           v
    validate
           |
           v
    reconcile
           |
           v
    retire old contract

The most important lessons are:

1. Schema changes are production changes.
2. Added, removed, renamed, and type-changed fields have different risks.
3. Nullability, constraints, and defaults are part of the contract.
4. Semantic changes can be breaking even when physical structure is unchanged.
5. Compatibility depends on deployment order.
6. Expand-and-contract is a powerful safe-evolution pattern.
7. Schema versions and application versions are different concepts.
8. Historical data may use older schemas.
9. Schema snapshots make drift detectable.
10. Schema diffs must explain what changed.
11. Contract tests should catch incompatible changes before production.
12. Incompatible records should enter a controlled DQ/dead-letter lifecycle.
13. Database migrations must account for intermediate states.
14. Rollback must consider application and schema state together.
15. Replaying before schema repair repeats the failure.
16. Reconciliation is required after migration and recovery.
17. Old schemas should be retired only after active consumers migrate.
18. Schema correctness protects data meaning, not just pipeline execution.

The objective is not:

    The new schema deployed.

The objective is:

    The new contract was introduced safely,
    old consumers were protected,
    new consumers were validated,
    affected data was recovered,
    and downstream results still reconcile.

When you can independently execute that lifecycle, schema evolution becomes a controlled engineering process instead of a production surprise.

---

