# Recipe 15 — Handle Missing Data

Data pipelines do not only receive bad data. Sometimes they receive less data than they should.

A source may contain 100,000 records while the pipeline receives 99,700. A daily file may never arrive. A partition may be absent. An API may stop after page 4. A consumer may stop before the end of a stream. A transformation may accidentally filter valid records.

The pipeline can still report SUCCESS.

This is the missing-data problem.

A production data engineer must be able to detect missing data, determine where it disappeared, decide whether the absence is expected, recover the missing scope, and prove that recovery worked.

---

## 1. Goal

By the end of this recipe, you should be able to:

- recognize missing data in a real pipeline
- distinguish missing data from legitimate zero-volume periods
- define what complete means for a dataset
- find the first pipeline boundary where data disappeared
- compare expected and observed records
- reconcile stable IDs and control totals
- detect missing files, partitions, windows, and API pages
- account for rejected, quarantined, and intentionally filtered records
- choose fail, alert, retry, quarantine, or recovery
- recover from an authoritative source
- make recovery bounded and idempotent
- test missing-data behavior
- observe completeness metrics
- intentionally break the pipeline
- recover it without creating duplicates
- prove the final state is correct

Core model:

    Expected scope
          |
          v
    Observe actual scope
          |
          v
    Compare and account
       /          \
   complete     incomplete
      |              |
      v              v
   continue       investigate
                     |
                     v
              identify scope
                     |
                     v
                 recover
                     |
                     v
                reconcile
                     |
                     v
                 validate

The key rule:

> A successful pipeline execution does not prove complete data.

---

## 2. The Problem

Suppose a payment source reports:

    Expected: 100,000

The pipeline contains:

    Observed: 99,700

The job says:

    SUCCESS

There are 300 records requiring explanation.

Possible causes:

- source itself produced fewer records
- extraction stopped early
- API pagination was incomplete
- a file was missing
- a file was truncated
- a partition never arrived
- a consumer stopped
- validation rejected records
- records entered quarantine
- a transformation filtered valid data
- loading failed for part of the batch
- deduplication removed valid records
- the reconciliation query used the wrong scope

The number 99,700 is only a symptom.

The engineering task is to identify the cause and the exact missing scope.

---

## 3. Missing Data Is a Completeness Problem

The simple model is:

    expected - observed = difference

But expected must come from a real contract.

Expected scope may come from:

- source count
- source manifest
- expected file list
- expected partitions
- sequence range
- event IDs
- time windows
- source control totals
- upstream reconciliation
- business-day rules

Do not use yesterday's record count as an automatic definition of today's expected count.

---

## 4. Missing Data vs Valid Zero

Zero records is not automatically an incident.

A zero-volume period may be valid because:

- there was genuinely no activity
- the business was closed
- the source intentionally produced an empty partition
- a customer had no events
- a reporting period legitimately contains no records

Zero can also indicate:

- source outage
- missing file
- broken API extraction
- stopped consumer
- incorrect filter
- missing partition

Therefore define the difference between:

    expected empty

and:

    unexpectedly missing

---

## 5. Recognize the Problem

Look for:

- source count greater than target count
- missing files
- missing partitions
- missing time windows
- gaps in meaningful sequences
- incomplete API pages
- consumer offsets that stopped
- source and warehouse totals that do not reconcile
- sudden unexplained volume drops
- records disappearing between stages
- reports with unexpectedly low totals
- a pipeline marked successful while completeness is unknown

Example:

    Source: 1,000,000
    Raw:      999,800
    Staging:  999,500
    Curated:  999,500

The first loss is between source and raw.

That is where the investigation begins.

---

## 6. Repository Investigation

Before implementing anything, trace:

    Source
      |
      v
    Extract
      |
      v
    Raw
      |
      v
    Parse
      |
      v
    Staging
      |
      v
    Transform
      |
      v
    Curated
      |
      v
    Warehouse
      |
      v
    Reporting

Find:

    event_id
    source_id
    batch_id
    file_id
    partition
    sequence_number
    occurred_at
    ingested_at
    processed_at
    processing_status

Search for:

    COUNT
    LIMIT
    OFFSET
    cursor
    next_page
    watermark
    checkpoint
    partition
    manifest
    rejected
    quarantine

You are looking for every place where data can disappear.

---

## 7. Find the First Point of Loss

Compare every boundary.

Example:

    Source  = 100,000
    Raw     = 100,000
    Staging =  99,950
    Curated =  99,950

The first unexplained loss is Raw to Staging.

Another example:

    Source  = 100,000
    Raw     =  99,500
    Staging =  99,500

The first loss is Source to Raw.

This is much more useful than starting from the final warehouse and guessing.

---

## 8. Data Accounting

For each pipeline run, record enough information to explain movement.

A useful control record can contain:

    pipeline_run_id
    source
    dataset
    batch_id
    input_count
    output_count
    rejected_count
    quarantined_count
    filtered_count
    missing_count
    completeness_status
    recovery_status
    created_at
    completed_at

The accounting principle is:

    Input
      =
    Output
      +
    Rejected
      +
    Quarantined
      +
    Intentional exclusions
      +
    Unexplained

The unexplained value must be zero for a complete run.

---

## 9. Define the Completeness Contract

Examples:

### File completeness

    Expected: 24 files
    Received: 23 files

### Partition completeness

    Expected: one partition per hour
    Observed: hour 02 absent

### Count completeness

    Expected: 100,000
    Observed: 99,700

### Sequence completeness

    Expected: 1001 through 2000
    Observed: 1001 through 1999

### Event completeness

    Every event_id in source scope must exist downstream.

### Financial completeness

    Source amount = target amount
    Source count = target count
    Source IDs = target IDs

Use only controls that make sense for the dataset.

---

## 10. Count Reconciliation

A first check is:

    expected_count - observed_count

Example:

    expected = 100,000
    observed = 99,700
    difference = 300

Useful, but not sufficient.

Equal counts can hide:

    missing records
    +
    unexpected records
    +
    duplicates

Therefore combine counts with identity or other controls.

---

## 11. Compare Stable IDs

If the dataset has a stable event identifier, compare identity.

Source:

    A B C D E

Target:

    A B D E

Missing:

    C

PostgreSQL example:

```sql
SELECT s.event_id
FROM source_events AS s
LEFT JOIN target_events AS t
    ON t.event_id = s.event_id
WHERE t.event_id IS NULL;
```

Count the missing records:

```sql
SELECT COUNT(*) AS missing_count
FROM source_events AS s
LEFT JOIN target_events AS t
    ON t.event_id = s.event_id
WHERE t.event_id IS NULL;
```

This turns a difference into a recovery set.

---

## 12. Check the Reverse Direction

Also identify target records that do not exist in the source scope:

```sql
SELECT t.event_id
FROM target_events AS t
LEFT JOIN source_events AS s
    ON s.event_id = t.event_id
WHERE s.event_id IS NULL;
```

Possible explanations include:

- wrong reconciliation scope
- stale records
- incorrect recovery
- unexpected source mismatch
- bad joins
- target corruption

A good reconciliation considers both directions.

---

## 13. Sequence Gaps

A meaningful sequence can reveal missing events:

    1001
    1002
    1003
    1005
    1006

Potential gap:

    1004

But not every sequence is guaranteed to be gapless.

Gaps may be valid because of:

- rolled-back transactions
- parallel producers
- sharding
- reserved IDs
- deleted events
- independent sequence allocation

Use sequence gaps only when the source contract says they are meaningful.

---

## 14. Missing Files

Suppose a source manifest expects:

    payments_01.csv
    payments_02.csv
    payments_03.csv
    payments_04.csv

The pipeline receives:

    payments_01.csv
    payments_02.csv
    payments_04.csv

Then:

    payments_03.csv

is missing.

A manifest may contain:

    batch_id
    expected_file
    expected_size
    expected_checksum
    expected_record_count
    received_at
    processing_status

This makes the missing scope explicit.

---

## 15. Missing Partitions

Expected:

    date=2026-09-26/hour=00
    date=2026-09-26/hour=01
    date=2026-09-26/hour=02
    date=2026-09-26/hour=03

Observed:

    00
    01
    03

Potentially missing:

    02

But distinguish:

    partition exists with zero records

from:

    partition never arrived

The source contract decides which condition is an incident.

---

## 16. Missing Time Windows

A pipeline may process:

    10:00–11:00
    11:00–12:00
    12:00–13:00
    13:00–14:00

If 12:00–13:00 was never processed, the gap must be visible.

A window registry can contain:

    window_start
    window_end
    extraction_status
    record_count
    completed_at

Then a missing window can be recovered specifically.

---

## 17. Incomplete API Pagination

A common extraction failure is stopping before the last API page.

Suppose:

    API total = 50,000
    extracted = 40,000

Every request may still return HTTP 200.

Check:

- total count
- page number
- page size
- next-page cursor
- continuation token
- response metadata
- source watermark
- final-page condition

Important rule:

> HTTP 200 means the request succeeded. It does not necessarily mean the full dataset was extracted.

---

## 18. Cursor-Based Pagination

A cursor API might return:

```json
{
  "data": [...],
  "next_cursor": "abc123"
}
```

Conceptual flow:

    request
       |
       v
    receive page
       |
       v
    process page
       |
       v
    next_cursor?
      /       \
    yes        no
    |           |
    v           v
  continue     complete

A common bug is accidentally stopping after the first page.

Test the final-page behavior explicitly.

---

## 19. Watermarks Can Hide Gaps

Incremental processing may use:

    last_processed_at = 10:00

Then process:

    10:00–11:00

But a gap can occur:

    processed: 10:00–10:30
    missing:   10:30–10:45
    processed: 10:45–11:00

If the watermark jumps directly to 11:00, the gap may disappear from the pipeline's control logic.

Therefore:

> Progress is not the same thing as completeness.

Gap detection must be part of the incremental design.

---

## 20. Stage-Level Loss

Suppose:

    Raw       = 1,000
    Staging   =   990
    Curated   =   980

There may be:

    10 lost between Raw and Staging
    10 lost between Staging and Curated

Do not only compare:

    Source vs Warehouse

Also compare:

    Source -> Raw
    Raw -> Staging
    Staging -> Curated
    Curated -> Warehouse

The first unexplained loss is usually the best investigation boundary.

---

## 21. Account for Rejections and Quarantine

Example:

    Input      = 100,000
    Accepted   =  99,700
    Rejected   =     200
    Quarantine =     100

Then:

    99,700 + 200 + 100 = 100,000

All input is accounted for.

But:

    Accepted   = 99,700
    Rejected   =    100
    Quarantine =    100

gives:

    99,900

So:

    100 unexplained

That is missing data.

---

## 22. Account for Intentional Filtering

Suppose:

    source = 100,000
    test_records = 500

The curated layer intentionally excludes test records.

Then:

    curated = 99,500

This is not missing data if the contract says test records must be excluded.

Always separate:

    intentionally excluded

from:

    unexplained loss

---

## 23. Completeness Status

Execution status and completeness status can be separate.

Example:

    execution_status = SUCCESS
    completeness_status = INCOMPLETE

Useful states include:

    UNKNOWN
    COMPLETE
    COMPLETE_WITH_WARNING
    INCOMPLETE
    PARTIALLY_COMPLETE
    RECOVERING

This prevents a technically successful job from being interpreted as a complete dataset.

---

## 24. Choosing the Response

### Retry

Use when the source or transport failure is transient.

### Alert

Use when the missing scope requires investigation.

### Fail

Use when downstream publication must not happen with incomplete data.

### Quarantine

Use when data exists but cannot safely enter the normal path.

### Recover

Use when the missing scope is known and authoritative data is available.

The correct response depends on business impact and source guarantees.

---

## 25. Recovery Source

Use the highest trustworthy layer.

For example:

    Source
       |
       v
    Raw
       |
       v
    Staging
       |
       v
    Curated
       |
       v
    Warehouse

If curated is incomplete but raw is complete:

    recover from raw

If raw is incomplete:

    recover from source

Do not recover from a downstream layer already known to be incomplete.

---

## 26. Recovery by Missing IDs

If missing IDs are known:

    missing_ids
         |
         v
    authoritative source
         |
         v
    recovered records
         |
         v
    normal validation
         |
         v
    normal processing
         |
         v
    reconciliation

Example:

```sql
SELECT *
FROM source_events
WHERE event_id IN (
    'evt-1001',
    'evt-1007',
    'evt-1020'
);
```

The recovered records should normally use the same validation and idempotency rules as normal data.

---

## 27. Recovery by Time Range

If exact IDs are unavailable but the missing window is known:

    missing:
    2026-09-25 10:00–11:00

Re-extract only that interval.

Then:

    validate
       |
       v
    ingest
       |
       v
    transform
       |
       v
    reconcile

This is a bounded backfill.

---

## 28. Recovery by Partition

For an incomplete partition:

    1. identify authoritative source
    2. build corrected data separately
    3. validate counts
    4. validate IDs
    5. validate control totals
    6. publish/merge corrected partition
    7. reconcile downstream data
    8. record recovery metadata

Do not blindly overwrite production data.

---

## 29. Recovery Must Be Idempotent

Recovery means processing data again.

Suppose target contains:

    A B C D

Recovery input contains:

    C D E F

Correct final result:

    A B C D E F

Incorrect result:

    A B C C D D E F

Therefore recovery depends on:

    idempotency
        +
    deduplication
        +
    bounded scope

These earlier recipes are directly reused here.

---

## 30. Recovery Must Be Bounded

Avoid:

    reprocess everything

when only:

    300 records

are missing.

Prefer:

    event IDs
    batch ID
    file ID
    partition
    time range

Record the exact recovery scope.

Large uncontrolled recovery can create:

- duplicates
- database load
- downstream overload
- unexpected historical changes
- difficult rollback
- long recovery times

---

## 31. Dry-Run Recovery

Where practical, recovery should support dry run.

Example:

    scope = 2026-09-25 10:00–11:00

Dry run reports:

    source records = 3,204
    already present = 3,100
    new records = 104
    expected final = 3,204

Review this before changing production data.

---

## 32. Recovery Validation

Before recovery:

    source = 100,000
    target =  99,700
    missing =     300

After recovery:

    source = 100,000
    target = 100,000
    missing =       0

Then validate:

- uniqueness
- control totals
- financial totals where relevant
- partition completeness
- downstream counts
- business invariants
- processing status

A recovery job returning SUCCESS is not enough.

---

## 33. Control Totals

Useful controls include:

    COUNT(*)

    COUNT(DISTINCT event_id)

    SUM(amount)

    MIN(sequence_number)

    MAX(sequence_number)

    deterministic checksum

Use several controls when correctness requires it.

---

## 34. Financial Reconciliation

For financial data, equal counts do not prove equal business value.

Example:

    Source:
    1,000 payments
    EUR 1,000,000

    Target:
    1,000 payments
    EUR 990,000

Counts match.

Amount does not.

Reconcile financial datasets using appropriate controls such as:

- transaction count
- amount totals
- currency-level totals
- unique transaction IDs
- settlement totals

The exact controls depend on the accounting model.

---

## 35. Currency-Aware Reconciliation

Do not add different currencies as though they were the same unit.

Example:

    EUR 100
    USD 100
    GBP 100

A total of 300 is not a meaningful financial control.

Reconcile per currency or use a documented normalized amount.

---

## 36. Missing-Data Observability

Useful metrics:

    expected_records
    observed_records
    missing_records
    rejected_records
    quarantined_records
    unexplained_records

Also:

    missing_files_total
    missing_partitions_total
    missing_windows_total
    incomplete_runs_total
    recovery_runs_total
    recovered_records_total
    recovery_failures_total

Useful dimensions:

    pipeline
    source
    dataset
    environment

Avoid unnecessarily high-cardinality labels.

---

## 37. Alert Design

Bad:

    Pipeline failed.

Better:

    Payment ingestion incomplete:
    expected=100000
    observed=99700
    missing=300
    run_id=abc123

Best operational form:

    Payment ingestion incomplete.
    Missing records: 300.
    Affected partition: 2026-09-25.
    Recovery scope: event_id.
    Run: abc123.

An alert should tell the operator what is missing and what recovery scope is available.

---

## 38. Production Runbook

Use:

    1. identify affected run
    2. determine expected scope
    3. determine observed scope
    4. calculate difference
    5. locate first loss boundary
    6. identify missing scope
    7. verify authoritative source
    8. dry-run recovery
    9. execute bounded recovery
    10. reconcile
    11. validate downstream state
    12. record recovery and root cause

This should be documented where operators can use it during an incident.

---

## 39. Test — Complete Input

Start with:

    source = 1,000
    target = 1,000

Expected:

    COMPLETE

No recovery should run.

---

## 40. Test — Missing Records

Start with:

    source = 1,000
    target = 990

Expected:

    missing = 10
    status = INCOMPLETE

Verify the affected records can be identified.

---

## 41. Test — Missing IDs

Source:

    A B C D E

Target:

    A B D E

Expected:

    C is missing

Verify C can be recovered.

---

## 42. Test — Missing File

Expected:

    file_01
    file_02
    file_03
    file_04

Received:

    file_01
    file_02
    file_04

Expected:

    file_03 = MISSING

Verify the run becomes incomplete and recovery can resume when the file arrives.

---

## 43. Test — Missing Partition

Expected:

    00
    01
    02
    03

Observed:

    00
    01
    03

Expected:

    02 = MISSING

Verify the pipeline does not silently advance past the gap.

---

## 44. Test — Valid Empty Partition

Create a partition that the source contract says may be empty.

Expected:

    partition exists
    count = 0
    completeness = COMPLETE

This protects against false alerts.

---

## 45. Test — Incomplete Pagination

Create an API fixture:

    total = 5,000

Make the extractor process only:

    4,000

Expected:

    INCOMPLETE

This test is important because every individual request can still return HTTP 200.

---

## 46. Test — Loss Between Stages

Create:

    Raw = 1,000
    Staging = 990

Expected:

    unexplained = 10

Then introduce a valid rejection reason for five records.

The five should become accounted-for.

The other five must remain unexplained until fixed.

---

## 47. Test — Recovery Overlap

Original target:

    A B C D

Recovery source:

    C D E F

Expected final state:

    A B C D E F

Verify each ID exists exactly once.

---

## 48. Test — Recovery Failure

Start with missing records.

Force recovery to fail halfway.

Verify:

- recovery is visibly incomplete
- retry is safe
- processed records are not duplicated
- final reconciliation can be repeated
- dataset is not falsely marked complete

---

## 49. Test — Wrong Recovery Scope

Missing data belongs to:

    2026-09-25

Attempt recovery for:

    2026-09-24

The system should not silently modify unrelated data.

This is a production safety test.

---

## 50. Test — Source Also Missing

Suppose:

    source = 99,700
    target = 99,700

but an upstream business control says:

    expected = 100,000

Source-to-target reconciliation passes.

The source itself may be incomplete.

This is why reconciliation proves agreement between systems, not necessarily completeness of the first system.

Additional controls may be required:

- upstream reconciliation
- source manifests
- expected file counts
- sequence controls
- business control totals

---

## 51. Intentionally Break the Pipeline

Perform these failure drills in a safe environment.

### Drill 1 — Delete target records

Remove a known set.

Expected:

    missing records detected

### Drill 2 — Remove a source file

Prevent one expected file from arriving.

Expected:

    missing file detected

### Drill 3 — Break pagination

Stop extraction before the last page.

Expected:

    incomplete extraction detected

### Drill 4 — Drop records during transformation

Introduce an incorrect filter.

Expected:

    stage reconciliation detects unexplained loss

### Drill 5 — Break recovery

Force recovery to fail halfway.

Expected:

    incomplete recovery visible
    retry safe

### Drill 6 — Run recovery twice

Expected:

    no duplicates

### Drill 7 — Create a legitimate empty period

Expected:

    no false missing-data alert

---

## 52. Full Recovery Exercise

After breaking the pipeline:

    Detect
       |
       v
    Measure
       |
       v
    Identify
       |
       v
    Scope
       |
       v
    Dry run
       |
       v
    Recover
       |
       v
    Reconcile
       |
       v
    Validate
       |
       v
    Record

Do not manually insert arbitrary rows simply to make the count match.

Use authoritative data and the normal correctness controls.

---

## 53. Incident Investigation Example

Suppose monitoring reports:

    expected = 1,200,000
    observed = 1,197,500
    missing = 2,500

### Step 1 — Check raw

    raw = 1,200,000

Extraction is complete.

### Step 2 — Check staging

    staging = 1,197,500

The first loss is Raw to Staging.

### Step 3 — Account for outcomes

    rejected = 2,000
    quarantined = 400

Then:

    1,197,500 + 2,000 + 400
    =
    1,199,900

There are still 100 unexplained records.

### Step 4 — Identify the 100 IDs

Compare raw to staging.

### Step 5 — Inspect transformation logic

Suppose an incorrect filter dropped records containing a particular optional field.

### Step 6 — Fix the defect

### Step 7 — Reprocess the affected raw scope

### Step 8 — Reconcile

Example final accounting:

    accepted = 1,197,600
    rejected =     2,000
    quarantine =     400

Total:

    1,200,000

The data is now fully accounted for.

This is the reasoning you should be able to perform independently.

---

## 54. Five Questions to Ask First

When someone says:

> Some data is missing.

Ask:

### 1. What exactly is missing?

Records?

Files?

Partitions?

Windows?

Amounts?

Events?

### 2. What was expected?

Which contract, manifest, control total, or source defines expected scope?

### 3. Where did it disappear?

Source?

Raw?

Staging?

Transformation?

Warehouse?

### 4. Can we identify the missing scope?

IDs?

Batch?

File?

Partition?

Time range?

### 5. What is the safest recovery source?

Source?

Raw?

Quarantine?

Replay?

Backfill?

These questions turn a vague incident into a bounded engineering problem.

---

## 55. Common Mistakes

### Mistake 1 — Treating SUCCESS as completeness

A successful process can still lose data.

### Mistake 2 — Comparing counts only

Equal counts can hide missing and unexpected records.

### Mistake 3 — Treating zero as failure

Zero can be valid.

### Mistake 4 — Treating zero as success

Zero can indicate an outage.

### Mistake 5 — Ignoring pagination

A partial API extraction can look successful.

### Mistake 6 — Advancing a watermark across a gap

This can permanently skip a time range.

### Mistake 7 — Reprocessing everything

Recovery should normally be bounded.

### Mistake 8 — Recovering without idempotency

This can create duplicates.

### Mistake 9 — Recovering from an incomplete downstream layer

Use an authoritative source.

### Mistake 10 — No audit trail

You should be able to explain what was recovered and why.

### Mistake 11 — One completeness rule for every dataset

Different datasets have different contracts.

### Mistake 12 — Alerts without missing scope

An operator should know what needs recovery.

---

## 56. Practical Implementation Sequence

For an existing repository:

    1. Trace the complete pipeline
    2. Identify the authoritative source
    3. Define expected scope
    4. Identify stable IDs
    5. Identify batch/file/partition/window identifiers
    6. Measure current counts
    7. Identify rejection and quarantine paths
    8. Define completeness rules
    9. Add stage-level accounting
    10. Add completeness checks
    11. Add missing-scope identification
    12. Add bounded recovery
    13. Verify idempotency
    14. Add reconciliation
    15. Add metrics and structured logs
    16. Add alerts
    17. Add unit tests
    18. Add integration tests
    19. Break the pipeline intentionally
    20. Execute recovery
    21. Reconcile again
    22. Validate downstream state
    23. Document the runbook

The key is to define the semantics before implementing the recovery path.

---

## 57. Example PostgreSQL Stage Accounting

Suppose staging records contain processing status:

```sql
SELECT
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE status = 'PROCESSED') AS processed,
    COUNT(*) FILTER (WHERE status = 'REJECTED') AS rejected,
    COUNT(*) FILTER (WHERE status = 'QUARANTINED') AS quarantined
FROM staging_events
WHERE batch_id = 'batch-2026-09-26';
```

Then verify that:

    processed
    + rejected
    + quarantined
    + intentional exclusions

accounts for the input.

Any remainder is unexplained.

Use the actual statuses in the repository.

---

## 58. Example Completeness Record

A control table may contain:

```text
pipeline_run_id
source
dataset
scope_start
scope_end
expected_count
observed_count
rejected_count
quarantined_count
filtered_count
missing_count
completeness_status
recovery_status
created_at
completed_at
```

Before recovery:

```text
expected = 100000
observed = 99700
missing = 300
completeness = INCOMPLETE
recovery = PENDING
```

After recovery:

```text
expected = 100000
observed = 100000
missing = 0
completeness = COMPLETE
recovery = COMPLETED
```

This provides an auditable history.

---

## 59. Observability Dashboard

Useful dashboard sections:

### Volume

    Expected records
    Observed records
    Missing records

### Completeness

    Complete runs
    Incomplete runs
    Recovery-pending runs

### Source health

    Missing files
    Missing partitions
    Missing windows
    API extraction gaps

### Recovery

    Records recovered
    Recovery failures
    Recovery duration

### Reconciliation

    Count difference
    Amount difference
    Duplicate difference

The objective is to make unexplained data loss visible.

---

## 60. Production Checklist

### Expected scope

- [ ] Expected files defined where applicable
- [ ] Expected partitions defined where applicable
- [ ] Expected windows defined where applicable
- [ ] Expected record scope defined where applicable
- [ ] Legitimate empty periods documented

### Detection

- [ ] Count reconciliation exists
- [ ] Stable IDs compared where possible
- [ ] Stage accounting exists
- [ ] Missing files detectable
- [ ] Missing partitions detectable
- [ ] Pagination completeness verified
- [ ] Watermark gaps considered
- [ ] Unexplained differences measurable

### Recovery

- [ ] Authoritative recovery source defined
- [ ] Recovery scope can be bounded
- [ ] Recovery is idempotent
- [ ] Duplicate recovery is safe
- [ ] Dry run available where appropriate
- [ ] Recovery failures visible
- [ ] Recovery retryable

### Validation

- [ ] Reconciliation runs after recovery
- [ ] Unique IDs checked
- [ ] Control totals checked
- [ ] Financial totals checked where relevant
- [ ] Downstream state validated

### Observability

- [ ] Missing count measurable
- [ ] Incomplete runs visible
- [ ] Recovery metrics exist
- [ ] Alerts identify missing scope
- [ ] Recovery actions auditable

### Failure drills

- [ ] Records can be intentionally removed in a safe environment
- [ ] Missing files can be simulated
- [ ] Pagination failure can be simulated
- [ ] Transformation loss can be simulated
- [ ] Recovery failure can be simulated
- [ ] Recovery can be repeated safely

---


## Production Tools You Should Know

These tools solve or provide production implementations of concepts covered in this recipe. Learn what the tool provides, but understand the underlying problem first.

| Tool | What to know |
|---|---|
| **Great Expectations** | Completeness and expectation checks. |
| **Soda** | Missing-data and freshness checks. |
| **dbt** | Source/model tests and completeness-oriented assertions. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the engineering mechanism.

---

## Implementation Lab — Runnable Missing-Data Detection and Recovery

This implementation compares expected IDs with observed IDs, identifies missing records, distinguishes a genuinely empty period from an incomplete period, and performs bounded recovery.

### 1. Completeness functions

```python
# src/missing_data.py
from dataclasses import dataclass
from datetime import datetime, timezone


@dataclass(frozen=True)
class CompletenessResult:
    expected: int
    observed: int
    missing_ids: set[str]
    extra_ids: set[str]
    complete: bool


def reconcile_ids(expected_ids: set[str], observed_ids: set[str]) -> CompletenessResult:
    missing = expected_ids - observed_ids
    extra = observed_ids - expected_ids

    return CompletenessResult(
        expected=len(expected_ids),
        observed=len(observed_ids),
        missing_ids=missing,
        extra_ids=extra,
        complete=not missing and not extra,
    )


def classify_partition(
    expected_count: int,
    observed_count: int,
    source_declared_empty: bool,
) -> str:
    if expected_count == 0 and source_declared_empty:
        return "VALID_EMPTY"

    if observed_count == expected_count:
        return "COMPLETE"

    return "INCOMPLETE"


def missing_windows(expected_hours: list[int], observed_hours: set[int]) -> list[int]:
    return [hour for hour in expected_hours if hour not in observed_hours]
```

### 2. Tests

```python
# tests/test_missing_data.py
from src.missing_data import (
    reconcile_ids,
    classify_partition,
    missing_windows,
)


def test_complete_dataset():
    result = reconcile_ids({"a", "b", "c"}, {"a", "b", "c"})

    assert result.complete is True
    assert result.missing_ids == set()
    assert result.extra_ids == set()


def test_missing_ids_are_explicit():
    result = reconcile_ids({"a", "b", "c"}, {"a", "c"})

    assert result.complete is False
    assert result.missing_ids == {"b"}


def test_extra_ids_are_detected():
    result = reconcile_ids({"a", "b"}, {"a", "b", "x"})

    assert result.complete is False
    assert result.extra_ids == {"x"}


def test_empty_partition_is_not_missing_data():
    assert classify_partition(0, 0, True) == "VALID_EMPTY"


def test_missing_partition_is_incomplete():
    assert classify_partition(100, 0, False) == "INCOMPLETE"


def test_missing_time_window():
    assert missing_windows([9, 10, 11, 12], {9, 11, 12}) == [10]
```

### 3. PostgreSQL completeness accounting

Do not rely only on a final row count. Store the accounting result.

```sql
CREATE TABLE pipeline_completeness (
    run_id              UUID PRIMARY KEY,
    expected_count      BIGINT NOT NULL,
    observed_count      BIGINT NOT NULL,
    missing_count       BIGINT NOT NULL,
    extra_count         BIGINT NOT NULL,
    status              TEXT NOT NULL,
    checked_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Example calculation:

```sql
SELECT
    COUNT(*) AS observed_count,
    COUNT(DISTINCT event_id) AS distinct_event_count
FROM staged_events
WHERE run_id = $1;
```

### 4. Recovery by missing ID

Recovery should be bounded to the missing scope.

```python
def recover_missing_ids(
    missing_ids: set[str],
    source: dict[str, dict],
    destination: dict[str, dict],
) -> int:
    recovered = 0

    for event_id in missing_ids:
        if event_id not in source:
            continue

        # Idempotent insert/update.
        if event_id not in destination:
            destination[event_id] = source[event_id]
            recovered += 1

    return recovered
```

Test it:

```python
def test_recovery_is_idempotent():
    source = {"a": {"amount": 10}, "b": {"amount": 20}}
    destination = {"a": {"amount": 10}}

    missing = {"b"}

    assert recover_missing_ids(missing, source, destination) == 1
    assert recover_missing_ids(missing, source, destination) == 0
    assert set(destination) == {"a", "b"}
```

### 5. Break the pipeline intentionally

Delete one destination record.

Run completeness reconciliation.

Expected:

```text
expected = 3
observed = 2
missing = 1
status = INCOMPLETE
```

Recover only the missing ID, rerun reconciliation, and require:

```text
missing = 0
status = COMPLETE
```

This is the core production loop:

```text
measure
  ↓
identify missing scope
  ↓
recover bounded scope
  ↓
reconcile
  ↓
close
```


## 61. Definition of Done

The recipe is complete when:

- [ ] The authoritative source is identified.
- [ ] Expected scope is explicitly defined.
- [ ] Legitimate zero-volume periods are distinguishable from missing data.
- [ ] Stable record identifiers are used where available.
- [ ] File, partition, and window expectations are defined where applicable.
- [ ] Stage-level accounting exists.
- [ ] Rejected records are accounted for.
- [ ] Quarantined records are accounted for.
- [ ] Intentional filters are accounted for.
- [ ] Unexplained differences are measurable.
- [ ] Completeness status is exposed.
- [ ] Missing scope can be identified.
- [ ] API pagination completeness is verified where applicable.
- [ ] Watermark gaps cannot silently pass as complete.
- [ ] Recovery is bounded.
- [ ] Recovery uses an authoritative layer.
- [ ] Recovery is idempotent.
- [ ] Recovery overlap is safe.
- [ ] Dry-run recovery is available where appropriate.
- [ ] Recovery failures are retryable.
- [ ] Reconciliation runs after recovery.
- [ ] Control totals are validated.
- [ ] Missing-data metrics are available.
- [ ] Incomplete runs are observable.
- [ ] Recovery actions are auditable.
- [ ] Failure drills have been performed.
- [ ] Missing data can be detected automatically.
- [ ] Missing scope can be recovered without rerunning unrelated data.
- [ ] Final state can be independently verified.

---

## 62. What You Learned

Missing data is not simply:

    COUNT(source) > COUNT(target)

It is a question of data completeness.

The full reasoning model is:

    Define expected scope
            |
            v
    Observe actual scope
            |
            v
    Account for valid exclusions
            |
            v
    Identify unexplained difference
            |
            v
    Locate first stage where data disappeared
            |
            v
    Identify missing scope
            |
            v
    Recover from authoritative layer
            |
            v
    Reprocess idempotently
            |
            v
    Reconcile again
            |
            v
    Validate final state

The most important lessons:

1. A successful pipeline can still lose data.
2. Zero records is not automatically an error.
3. Equal counts do not prove completeness.
4. Every pipeline boundary should be accountable.
5. Stable IDs make recovery much easier.
6. Recovery should be bounded.
7. Recovery requires idempotency.
8. Reconciliation must happen after recovery.
9. Control totals can reveal problems counts miss.
10. Completeness must come from the source contract.
11. A watermark does not automatically prove completeness.
12. Observability must expose unexplained differences.

The goal is not merely to detect:

    300 records are missing.

The goal is to answer:

    Which 300?
    From where?
    Why?
    Since when?
    Can they be recovered?
    How do we know recovery worked?
    How do we prevent the same loss again?

When you can answer all of those questions independently, you are engineering data correctness rather than merely running jobs.

---
