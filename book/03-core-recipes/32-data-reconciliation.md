# Recipe 32 — Data Reconciliation

Data reconciliation answers a fundamental pipeline question:

> Do two data states agree according to a defined technical and business contract?

A pipeline can execute successfully and still produce incorrect data. It can have missing records, extra records, duplicates, changed values, different aggregates, stale records, or different business scopes.

Data reconciliation is the systematic process of detecting, explaining, and resolving those differences.

By the end of this recipe, you should be able to recognize reconciliation problems, choose the right reconciliation method, implement it, test it, observe it, intentionally break it, investigate the mismatch, recover the affected data, and prove that the final state reconciles.

---

## 1. Goal

You should be able to:
- define a reconciliation contract
- define identical business and time scope
- choose count, identity, duplicate, field, aggregate, or business reconciliation
- find missing and unexpected records
- detect duplicate keys
- detect changed values
- reconcile financial amounts safely
- reconcile pipeline stages
- distinguish expected differences from defects
- automate reconciliation
- persist and observe reconciliation results
- intentionally create mismatches
- investigate their root cause
- perform bounded recovery
- reconcile again and prove recovery worked.

Core model:

    Define scope
         |
         v
    Define expected relationship
         |
         v
    Compare controls
         |
         v
    Classify differences
         |
         v
    Investigate
         |
         v
    Recover or correct
         |
         v
    Reconcile again
         |
         v
       Close

> Reconciliation is not simply checking whether two counts match.

---

## 2. Why Reconciliation Exists

Suppose:

    Source:    1,000,000 payments
    Warehouse: 1,000,000 payments

Counts match. The data can still be wrong.

For example, 5,000 source records may be absent while 5,000 unrelated records are present. The total remains 1,000,000.

Another example:

    Source amount:    EUR 10,000,000
    Target amount:    EUR  9,950,000

Even with equal record counts, the business result differs.

Therefore:

    execution success != data correctness

---

## 3. Reconciliation vs Validation

Validation asks:

    Is this record acceptable?

Examples:
- required field is present
- amount is valid
- timestamp is parseable
- currency is supported
- identifier is present.

Reconciliation asks:

    Does this dataset agree with another dataset or control?

Examples:
- source count equals target count
- source IDs equal target IDs
- important fields match
- amount totals reconcile
- status distributions reconcile
- source partitions exist downstream.

Use both. Validation protects individual records. Reconciliation protects relationships between datasets or states.

---

## 4. Define the Reconciliation Contract First

Before writing SQL, document:

    source dataset
    target dataset
    business scope
    time scope
    timezone
    join/business key
    fields that must match
    valid transformations
    allowed differences
    tolerance
    severity
    action on failure

Example:

    source: payment_service.transactions
    target: warehouse.fact_payment
    scope: settlement_date = 2026-09-26
    key: transaction_id
    required relationship: same transaction IDs
    amount: same amount by transaction and currency
    allowed difference: none
    failure action: block publication and alert

Without this contract, a reconciliation can report false failures simply because the two datasets have different intended semantics.

---

## 5. Scope Comes Before Comparison

Always make sure both sides represent the same population.

Scope may include:
- date or timestamp window
- timezone
- tenant
- customer
- country
- currency
- account type
- transaction type
- status
- source system
- batch
- partition.

Example:

    source: settlement_date = 2026-09-26
    target: settlement_date = 2026-09-26

Do not compare:

    all source payments

against:

    successful target payments only

and call the difference missing data.

---

## 6. Reconciliation Pyramid

Use progressively stronger controls:

    1. Dataset existence
    2. Partition presence
    3. Record count
    4. Identity
    5. Uniqueness
    6. Important field values
    7. Aggregates
    8. Business invariants

Start with the cheapest useful control. Increase depth when the dataset or risk requires it.

---

## 7. Dataset and Partition Reconciliation

First verify that the expected dataset, batch, file, or partition exists.

Example expected partitions:

    date=2026-09-26/hour=00
    date=2026-09-26/hour=01
    date=2026-09-26/hour=02
    date=2026-09-26/hour=03

If hour 02 is absent, investigate before running an expensive row comparison.

Distinguish:

    partition exists with zero rows

from:

    partition never arrived

The source contract determines whether an empty partition is valid.

---

## 8. Count Reconciliation

Basic control:

    source_count = target_count

PostgreSQL example:

~~~sql
WITH source AS (
    SELECT COUNT(*) AS count
    FROM source_events
    WHERE event_date = DATE '2026-09-26'
), target AS (
    SELECT COUNT(*) AS count
    FROM target_events
    WHERE event_date = DATE '2026-09-26'
)
SELECT
    source.count AS source_count,
    target.count AS target_count,
    source.count - target.count AS difference
FROM source
CROSS JOIN target;
~~~

Count reconciliation is a useful first signal, not proof of correctness.

---

## 9. Count Matching Does Not Prove Correctness

Source:

    A B C D E

Target:

    A B C X Y

Both contain five records.

Identity reconciliation fails because:

    source-only: D E
    target-only: X Y

Therefore count controls should be followed by identity controls when stable business keys exist.

---

## 10. Identity Reconciliation

Use a stable business identifier such as:

    transaction_id
    payment_id
    event_id
    order_id

Find source records missing from target:

~~~sql
SELECT s.transaction_id
FROM source_transactions AS s
LEFT JOIN target_transactions AS t
    ON t.transaction_id = s.transaction_id
WHERE s.settlement_date = DATE '2026-09-26'
  AND t.transaction_id IS NULL;
~~~

Find target records absent from source:

~~~sql
SELECT t.transaction_id
FROM target_transactions AS t
LEFT JOIN source_transactions AS s
    ON s.transaction_id = t.transaction_id
WHERE t.settlement_date = DATE '2026-09-26'
  AND s.transaction_id IS NULL;
~~~

These two result sets classify the difference as source-only and target-only.

---

## 11. FULL OUTER JOIN Classification

A full outer join can classify both sides in one result:

~~~sql
SELECT
    COALESCE(s.transaction_id, t.transaction_id) AS transaction_id,
    CASE
        WHEN s.transaction_id IS NULL THEN 'TARGET_ONLY'
        WHEN t.transaction_id IS NULL THEN 'SOURCE_ONLY'
        ELSE 'BOTH'
    END AS reconciliation_status
FROM source_transactions AS s
FULL OUTER JOIN target_transactions AS t
    ON t.transaction_id = s.transaction_id;
~~~

For large tables, restrict the comparison to the exact reconciliation scope and ensure the join keys are indexed appropriately.

---

## 12. Duplicate Reconciliation

Every expected ID can exist and the target can still be wrong because an ID occurs multiple times.

~~~sql
SELECT
    transaction_id,
    COUNT(*) AS occurrences
FROM target_transactions
GROUP BY transaction_id
HAVING COUNT(*) > 1;
~~~

If the business key must be unique:

    total_count = unique_key_count

should hold.

If:

    total = 100,000
    unique = 99,950

then 50 rows participate in duplicate keys.

---

## 13. Field-Level Reconciliation

After identity matches, important values can still differ.

Compare fields such as:

    amount
    currency
    status
    account_id
    transaction_type
    event_time

Example:

~~~sql
SELECT
    s.transaction_id,
    s.amount AS source_amount,
    t.amount AS target_amount,
    s.currency AS source_currency,
    t.currency AS target_currency
FROM source_transactions AS s
JOIN target_transactions AS t
    ON t.transaction_id = s.transaction_id
WHERE s.settlement_date = DATE '2026-09-26'
  AND (
      s.amount IS DISTINCT FROM t.amount
      OR s.currency IS DISTINCT FROM t.currency
  );
~~~

PostgreSQL's `IS DISTINCT FROM` is important because it compares NULL values safely.

---

## 14. NULL-Safe Comparison

Do not assume:

    source.value <> target.value

will detect every difference.

If both values involve NULL, normal SQL comparison can produce UNKNOWN rather than TRUE.

Use:

    source.value IS DISTINCT FROM target.value

to detect differences safely in PostgreSQL.

Use:

    source.value IS NOT DISTINCT FROM target.value

when testing equality including NULL semantics.

---

## 15. Aggregate Reconciliation

Useful controls include:

    COUNT(*)
    COUNT(DISTINCT key)
    SUM(amount)
    MIN(value)
    MAX(value)

Example:

~~~sql
SELECT
    COUNT(*) AS record_count,
    SUM(amount) AS total_amount,
    MIN(amount) AS min_amount,
    MAX(amount) AS max_amount
FROM source_transactions
WHERE settlement_date = DATE '2026-09-26';
~~~

Run equivalent controls on the target.

Aggregates are efficient controls, but they are not automatically proof of row-level equality.

---

## 16. Distribution Reconciliation

Compare distributions when transformations can change categories.

Example:

    Source:
    SUCCESS = 90,000
    FAILED  =  7,000
    PENDING =  3,000

    Target:
    SUCCESS = 88,000
    FAILED  =  9,000
    PENDING =  3,000

Total count matches, but status mapping differs.

Example query:

~~~sql
SELECT
    status,
    COUNT(*) AS record_count
FROM source_transactions
WHERE settlement_date = DATE '2026-09-26'
GROUP BY status
ORDER BY status;
~~~

Compare the grouped result with the target.

---

## 17. Dimensional Reconciliation

Global reconciliation can hide where the problem is.

Break controls down by:

    currency
    country
    tenant
    transaction_type
    status
    day
    hour
    account_type

Example:

    EUR difference = 0
    USD difference = 300
    GBP difference = 0

The investigation can now focus on USD.

A useful strategy is:

    Global
      -> Day
      -> Hour
      -> Dimension
      -> Individual records

Start broad and narrow only when a control fails.

---

## 18. Reconcile Pipeline Stages

Reconciliation is not only source versus warehouse.

Use it between:

    Source -> Raw
    Raw -> Staging
    Staging -> Curated
    Curated -> Warehouse

For every boundary ask:

    What must be preserved?
    What is intentionally changed?
    What is allowed to disappear?
    What relationship should hold?

Example:

    raw = 1,000,000
    staging = 999,950
    documented rejects = 30
    unexplained = 20

The correct reconciliation result is not simply '50 missing'. It is '20 unexplained after accounting for 30 expected rejects'.

---

## 19. Reconciliation Is Not Always Equality

Sometimes the correct relationship is:

    source_count
      =
    processed_count + rejected_count

Or:

    source_events
      =
    production_events + test_events

Or:

    source_amount
      =
    settled_amount + pending_amount + failed_amount

First define the equation. Then implement the check.

This is one of the most important reconciliation skills.

---

## 20. Tolerances

Some systems allow documented numerical differences because of:

- rounding
- currency conversion
- floating-point calculations
- delayed updates
- approximate metrics.

Example absolute tolerance:

    abs(source - target) <= 0.01

Example relative tolerance:

    abs(source - target) / abs(source) <= 0.001

Do not invent a tolerance just to make a failing check pass.

Define what happens when the source value is zero or near zero.

---

## 21. Financial Reconciliation

Financial data normally needs stronger controls.

Useful controls include:

    transaction count
    unique transaction count
    total amount
    amount by currency
    amount by transaction type
    status totals
    settlement totals

Example:

    Source EUR:
    count = 50,000
    amount = 12,500,000

    Target EUR:
    count = 50,000
    amount = 12,500,000

Reconcile each currency separately unless a documented normalization is applied.

Never treat EUR 100 + USD 100 as a meaningful financial total of 200 without a defined conversion.

---

## 22. Time and Timezone Reconciliation

Many apparent data mismatches are really scope mismatches.

Define:

    timezone
    start boundary
    end boundary
    timestamp precision
    event time vs processing time

Prefer half-open intervals:

    start <= timestamp < end

This avoids overlapping adjacent windows.

Example problem:

    event_time = 23:59 UTC
    processed_at = 00:03 UTC

If one system groups by event time and another by processing time, daily totals can legitimately differ.

---

## 23. Snapshot Reconciliation

When reconciling snapshots, record:

    snapshot_id
    source
    created_at
    scope
    record_count
    control totals

Compare snapshots from equivalent points in time.

Do not compare a live source against a historical target snapshot and call the difference a pipeline defect.

---

## 24. Batch Reconciliation

Batch pipelines should reconcile using the actual batch identity.

Example:

    source batch = batch-1001
    target batch = batch-1001

Controls:

    record count
    unique IDs
    amount totals
    status distribution

Do not compare source batch-1001 with target batch-1002 simply because both ran on the same day.

---

## 25. Checksum and Row Hash Reconciliation

A deterministic hash can summarize selected fields.

Conceptually:

    row_hash = hash(
        stable key + important canonical fields
    )

Compare row hashes for matching business keys.

This is useful for detecting changed rows without manually comparing dozens of columns.

Canonicalization must be consistent. Define:

    field order
    NULL representation
    timestamp format
    numeric format
    encoding
    row ordering or an order-independent aggregation

A hash only covers the fields included in it.

---

## 26. Reconciliation Result Model

Persist structured results such as:

    reconciliation_run_id
    source_dataset
    target_dataset
    scope
    source_count
    target_count
    missing_count
    extra_count
    duplicate_count
    changed_count
    source_amount
    target_amount
    amount_difference
    status
    started_at
    completed_at

Possible statuses:

    PASS
    WARN
    FAIL
    UNKNOWN
    RUNNING

Classify differences explicitly:

    MISSING
    EXTRA
    DUPLICATE
    CHANGED
    SCOPE_MISMATCH
    TIMING_DIFFERENCE
    TOLERANCE_EXCEEDED
    EXPECTED_DIFFERENCE

---

## 27. Automate Reconciliation

Production flow:

    Extract controls
         |
         v
       Compare
         |
         v
       Classify
         |
         v
    Persist result
         |
         v
      Emit metrics
         |
         v
    Alert / block if required

Run reconciliation at useful points:

- after ingestion
- after transformation
- before publication
- after recovery
- after important backfills.

---

## 28. Do Not Reconcile Only at the End

Suppose:

    Source -> Raw -> Staging -> Curated -> Warehouse -> Dashboard

If you only reconcile the dashboard, locating the original loss can be difficult.

Earlier checks make the first divergence visible:

    Source -> Raw
    Raw -> Staging
    Staging -> Curated
    Curated -> Warehouse

The earlier the divergence is detected, the smaller the investigation scope.

---

## 29. Reconciliation, Idempotency, and Recovery

Suppose:

    source = A B C D E
    target = A B D E

Reconciliation identifies:

    missing = C

Recover C.

Final:

    A B C D E

Run recovery again.

Final must remain:

    A B C D E

Therefore reconciliation depends on the idempotency and deduplication controls established in earlier recipes.

---

## 30. Performance Strategy

Large-scale reconciliation can be expensive.

Use a staged approach:

    1. partition existence
    2. count
    3. aggregate controls
    4. dimensional controls
    5. identity comparison
    6. field-level comparison for affected records

Useful techniques:

- partition pruning
- indexed business keys
- incremental reconciliation
- snapshot controls
- precomputed aggregates
- row hashes
- checksums.

For critical financial or regulatory datasets, sampling may not be sufficient.

Control strength should match data risk.

---

## 31. Reconciliation as a Publication Gate

For important datasets:

    Transform
       |
       v
    Reconcile
       |
      / \
   PASS FAIL
    |     |
    v     v
 Publish  Block

A failed reconciliation can block publication when publishing incorrect data is more harmful than delaying it.

Do not make every reconciliation a hard gate automatically. Define the policy per dataset.

---

## 32. Test — Perfect Match

Source:

    A 100
    B 200
    C 300

Target:

    A 100
    B 200
    C 300

Expected:

    PASS

---

## 33. Test — Missing Record

Source:

    A B C

Target:

    A B

Expected:

    source_only = C
    status = FAIL

---

## 34. Test — Extra Record

Source:

    A B

Target:

    A B C

Expected:

    target_only = C
    status = FAIL

---

## 35. Test — Duplicate Record

Source:

    A B C

Target:

    A B B C

Expected:

    duplicate_count > 0

The result should identify duplication instead of reporting only a generic count mismatch.

---

## 36. Test — Changed Value

Source:

    A amount=100

Target:

    A amount=110

Expected:

    changed_count = 1
    amount_difference = 10

---

## 37. Test — Count Match but Wrong IDs

Source:

    A B C

Target:

    A B D

Counts match.

Expected:

    source_only = C
    target_only = D
    status = FAIL

This proves why count-only reconciliation is insufficient.

---

## 38. Test — NULL Difference

Source:

    A amount=NULL

Target:

    A amount=100

Expected:

    changed_count = 1

Use NULL-safe comparison.

---

## 39. Test — Tolerance

Source:

    amount = 100.00

Target:

    amount = 100.005

If tolerance is 0.01:

    PASS

If tolerance is 0.001:

    FAIL

The tolerance must be part of the contract.

---

## 40. Test — Scope Mismatch

Source:

    all transactions

Target:

    successful transactions only

Expected:

    identify scope mismatch

Do not label the difference as missing data until the scope contract is corrected.

---

## 41. Test — Timezone Boundary

Create records around midnight UTC.

Example:

    23:59:59 UTC
    00:00:01 UTC

Run one reconciliation with UTC and another with a different timezone.

Expected:

    scope difference is visible

Then normalize the reconciliation window and rerun.

---

## 42. Test — Aggregate Mismatch

Source:

    count = 100
    amount = 10,000

Target:

    count = 100
    amount = 9,900

Expected:

    count passes
    amount fails

This proves that multiple controls may be necessary.

---

## 43. Test — Distribution Mismatch

Source:

    SUCCESS = 90
    FAILED  = 10

Target:

    SUCCESS = 80
    FAILED  = 20

Total count is equal.

Expected:

    status distribution fails

---

## 44. Test — Recovery

Start:

    source = A B C D
    target = A B D

Reconciliation:

    missing = C

Recover C and reconcile again.

Expected:

    PASS

Run recovery a second time.

Expected:

    still PASS
    no duplicate C

---

## 45. Intentionally Break the Pipeline

Perform these drills in a safe environment.

### Drill 1 — Delete a target row

Expected:

    source_only > 0

### Drill 2 — Insert an unexpected target row

Expected:

    target_only > 0

### Drill 3 — Duplicate a target row

Expected:

    duplicate_count > 0

### Drill 4 — Change an important field

Expected:

    changed_count > 0

### Drill 5 — Change an amount

Expected:

    amount difference detected

### Drill 6 — Change the timezone

Expected:

    scope mismatch detected

### Drill 7 — Change a transformation filter

Expected:

    identity or distribution mismatch

### Drill 8 — Run recovery twice

Expected:

    no new duplicates

---

## 46. Incident Investigation

Suppose reconciliation reports:

    source_count = 1,000,000
    target_count = 999,700

Do not immediately rerun everything.

### Step 1 — Confirm scope

Check:

    date
    timezone
    tenant
    status rules
    partition
    source version
    target snapshot

### Step 2 — Break down by dimension

Example:

    EUR difference = 0
    USD difference = 300
    GBP difference = 0

Now the investigation focuses on USD.

### Step 3 — Compare IDs

Find the 300 source-only records.

### Step 4 — Locate the first pipeline stage

Check whether those IDs exist in:

    raw
    staging
    curated

### Step 5 — Determine the cause

Possible causes:

- validation rejection
- transformation filter
- loading failure
- wrong scope
- late data
- missing source records
- incorrect deduplication.

### Step 6 — Recover

Use the smallest authoritative scope that safely repairs the data.

### Step 7 — Reconcile again

Expected:

    source_only = 0
    target_only = 0
    unexpected duplicates = 0
    changed values = 0

### Step 8 — Record root cause

Do not close the incident with only a corrected number. Record why the mismatch happened and what evidence proved the fix.

---

## 47. Practical Implementation Sequence

For an existing repository:

    1. Identify datasets to compare
    2. Define business scope
    3. Define time boundaries and timezone
    4. Identify stable business key
    5. Identify fields that must match
    6. Identify valid transformations
    7. Define allowed differences
    8. Define tolerances
    9. Implement count checks
    10. Implement identity checks
    11. Implement duplicate checks
    12. Implement field-level checks
    13. Implement aggregate controls
    14. Implement dimensional controls
    15. Classify differences
    16. Persist reconciliation results
    17. Add metrics
    18. Add alerts
    19. Add tests
    20. Break the data intentionally
    21. Investigate the mismatch
    22. Recover affected scope
    23. Reconcile again
    24. Document the runbook

Start with the cheapest useful control and add depth according to risk.

---

## 48. Example Reconciliation Result

A useful result record:

~~~text
reconciliation_run_id
source_dataset
target_dataset
scope_start
scope_end
source_count
target_count
missing_count
extra_count
duplicate_count
changed_count
source_amount
target_amount
amount_difference
status
started_at
completed_at
~~~

Failure example:

~~~text
source_count = 100000
target_count = 99700
missing = 300
extra = 0
duplicates = 0
changed = 0
amount_difference = -12500
status = FAIL
~~~

After recovery:

~~~text
source_count = 100000
target_count = 100000
missing = 0
extra = 0
duplicates = 0
changed = 0
amount_difference = 0
status = PASS
~~~

---

## 49. Observability

Useful metrics:

    reconciliation_runs_total
    reconciliation_pass_total
    reconciliation_fail_total
    reconciliation_missing_records
    reconciliation_extra_records
    reconciliation_duplicate_records
    reconciliation_changed_records
    reconciliation_amount_difference
    reconciliation_duration_seconds

Useful dimensions:

    pipeline
    source
    target
    dataset
    environment

Do not use individual transaction IDs as metric labels. Store detailed differences in a reconciliation result table or diagnostic store.

---

## 50. Alert Design

A useful alert contains:

    reconciliation name
    source
    target
    scope
    difference category
    difference count
    amount difference if relevant
    run ID
    severity

Example:

    Payment reconciliation FAILED

    scope: 2026-09-26
    source: payment_service
    target: warehouse.fact_payment
    source_count: 100000
    target_count: 99700
    source_only: 300
    amount_difference: EUR -12500
    action: investigate missing source IDs

This is much more useful than 'Data mismatch'.

---

## 51. Reconciliation Runbook

When reconciliation fails:

    1. Identify the failed reconciliation
    2. Confirm source and target versions
    3. Confirm exact scope
    4. Confirm timezone and boundaries
    5. Check count difference
    6. Check dimensional differences
    7. Check missing IDs
    8. Check extra IDs
    9. Check duplicates
    10. Check changed fields
    11. Check aggregate controls
    12. Locate first stage with the divergence
    13. Determine root cause
    14. Define bounded recovery
    15. Execute recovery
    16. Reconcile again
    17. Validate downstream outputs
    18. Record root cause and evidence

Do not close the incident because a number looks correct. Close it when the reconciliation contract passes and the cause is understood.

---


## Implementation Lab — Runnable Data Reconciliation

The implementation below compares source and destination datasets by count, IDs, aggregates, and field-level differences. It produces a machine-readable reconciliation result instead of a manual comparison.

### 1. Reconciliation model

```python
# src/reconcile.py
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True)
class ReconciliationResult:
    source_count: int
    target_count: int
    missing_ids: set[str]
    extra_ids: set[str]
    source_total: Decimal
    target_total: Decimal
    count_match: bool
    id_match: bool
    amount_match: bool

    @property
    def passed(self) -> bool:
        return self.count_match and self.id_match and self.amount_match


def reconcile(
    source: list[dict],
    target: list[dict],
) -> ReconciliationResult:
    source_map = {row["id"]: row for row in source}
    target_map = {row["id"]: row for row in target}

    source_total = sum((Decimal(str(r["amount"])) for r in source), Decimal("0"))
    target_total = sum((Decimal(str(r["amount"])) for r in target), Decimal("0"))

    return ReconciliationResult(
        source_count=len(source),
        target_count=len(target),
        missing_ids=set(source_map) - set(target_map),
        extra_ids=set(target_map) - set(source_map),
        source_total=source_total,
        target_total=target_total,
        count_match=len(source) == len(target),
        id_match=set(source_map) == set(target_map),
        amount_match=source_total == target_total,
    )
```

### 2. Tests

```python
# tests/test_reconcile.py
from decimal import Decimal
from src.reconcile import reconcile


def test_reconciliation_passes():
    source = [
        {"id": "a", "amount": "10.00"},
        {"id": "b", "amount": "20.00"},
    ]
    target = [
        {"id": "a", "amount": "10.00"},
        {"id": "b", "amount": "20.00"},
    ]

    result = reconcile(source, target)

    assert result.passed
    assert result.missing_ids == set()
    assert result.extra_ids == set()


def test_missing_record_fails():
    source = [
        {"id": "a", "amount": "10.00"},
        {"id": "b", "amount": "20.00"},
    ]
    target = [{"id": "a", "amount": "10.00"}]

    result = reconcile(source, target)

    assert result.passed is False
    assert result.missing_ids == {"b"}


def test_amount_difference_fails_even_when_counts_match():
    source = [{"id": "a", "amount": "10.00"}]
    target = [{"id": "a", "amount": "11.00"}]

    result = reconcile(source, target)

    assert result.count_match
    assert result.id_match
    assert result.amount_match is False
    assert result.passed is False
```

### 3. PostgreSQL control totals

For large datasets, reconcile in the database rather than loading everything into Python.

```sql
SELECT
    COUNT(*) AS row_count,
    COUNT(DISTINCT event_id) AS distinct_ids,
    COALESCE(SUM(amount), 0) AS amount_total
FROM source_snapshot
WHERE run_id = $1;
```

Compare against the destination:

```sql
SELECT
    COUNT(*) AS row_count,
    COUNT(DISTINCT event_id) AS distinct_ids,
    COALESCE(SUM(amount), 0) AS amount_total
FROM target_snapshot
WHERE run_id = $1;
```

For ID-level differences:

```sql
SELECT s.event_id
FROM source_snapshot s
LEFT JOIN target_snapshot t
    ON t.event_id = s.event_id
WHERE t.event_id IS NULL
  AND s.run_id = $1;
```

### 4. Persist the reconciliation result

```sql
CREATE TABLE reconciliation_runs (
    reconciliation_id UUID PRIMARY KEY,
    run_id            UUID NOT NULL,
    source_count      BIGINT NOT NULL,
    target_count      BIGINT NOT NULL,
    missing_count     BIGINT NOT NULL,
    extra_count       BIGINT NOT NULL,
    source_total      NUMERIC NOT NULL,
    target_total      NUMERIC NOT NULL,
    status            TEXT NOT NULL,
    checked_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5. Intentional failure

Change one destination amount:

```sql
UPDATE target_snapshot
SET amount = amount + 1
WHERE event_id = 'event-100';
```

Run reconciliation.

Expected:

```text
count_match  = true
id_match     = true
amount_match = false
status       = FAILED
```

Repair the value and rerun reconciliation.

The final state must be:

```text
PASSED
```

### 6. Production rule

Never define reconciliation as only:

```text
source_count == target_count
```

A robust reconciliation compares several independent controls:

```text
count
  +
unique IDs
  +
control totals
  +
important fields
  +
business invariants
```

One matching metric does not prove correctness.


## 52. Definition of Done

- [ ] Datasets being compared are explicitly defined.
- [ ] Business scope is explicit.
- [ ] Time boundaries are explicit.
- [ ] Timezone semantics are explicit.
- [ ] Stable business key is identified where available.
- [ ] Expected transformations are documented.
- [ ] Allowed differences are documented.
- [ ] Tolerances are documented where required.
- [ ] Count reconciliation exists.
- [ ] Identity reconciliation exists where possible.
- [ ] Missing records can be identified.
- [ ] Extra records can be identified.
- [ ] Duplicate records can be identified.
- [ ] Important field differences can be identified.
- [ ] NULL-safe comparisons are used where required.
- [ ] Aggregate controls exist where appropriate.
- [ ] Dimensional reconciliation exists where useful.
- [ ] Financial controls exist for financial datasets where required.
- [ ] Reconciliation results are auditable.
- [ ] Reconciliation status is explicit.
- [ ] Metrics are emitted.
- [ ] Alerts contain actionable difference information.
- [ ] Large datasets are reconciled efficiently.
- [ ] Recovery is bounded.
- [ ] Recovery is idempotent.
- [ ] Reconciliation runs after recovery.
- [ ] Failure scenarios are intentionally tested.
- [ ] Scope mismatch is tested.
- [ ] Count-only limitations are tested.
- [ ] Duplicate detection is tested.
- [ ] Field-change detection is tested.
- [ ] Tolerance behavior is tested.
- [ ] Final state can be independently verified.

---

## 53. What You Learned

Data reconciliation is not:

    compare two counts

It is:

    define the relationship
          |
          v
    define the scope
          |
          v
    compare appropriate controls
          |
          v
    classify the difference
          |
          v
    locate the first divergence
          |
          v
    recover or correct
          |
          v
    reconcile again
          |
          v
       prove closure

Most important lessons:

1. Pipeline success does not prove data correctness.
2. Reconciliation starts with a contract.
3. Scope mismatches can look like data defects.
4. Equal counts do not prove equal records.
5. Identity checks expose missing and extra records.
6. Duplicate checks expose a separate failure class.
7. Field checks detect changed values.
8. Aggregate controls provide efficient additional evidence.
9. Financial datasets require stronger controls.
10. Timezone and boundary errors can create false mismatches.
11. Reconcile at useful pipeline boundaries.
12. Tolerances must be documented.
13. Recovery is incomplete until reconciliation passes again.
14. A useful reconciliation explains where, how, and why data diverged.

The goal is not merely:

    PASS / FAIL

The goal is to answer:

    What should agree?
    What actually differs?
    Which records differ?
    Where did the divergence begin?
    Is the difference expected?
    What is the safest correction?
    How do we prove the correction worked?

When you can answer those questions independently, reconciliation becomes an engineering control rather than a manual comparison exercise.

---
