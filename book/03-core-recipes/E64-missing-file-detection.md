# E64 — Missing File Detection

## 1. Problem Recognition

A production file pipeline must know what should arrive before it can determine that something is missing.

Example:

    transactions.csv
    customers.csv
    balances.csv

If transactions and customers arrive but balances does not, the pipeline must distinguish:

    not observed yet
        !=
    late
        !=
    missing

A file should become MISSING only after a defined expectation, delivery calendar, deadline, and matching rule establish that it should have arrived.

This recipe applies:

- E49 — File Discovery
- E50 — File Naming Conventions
- E51 — File Arrival Detection
- E52 — File Completeness Detection
- E62 — Multi-File Extraction
- E63 — Duplicate File Detection
- Core Recipe 16 — Data Reconciliation
- Core Recipe 20 — SLA Monitoring
- Core Recipe 24 — Partial Failure

---

## 2. What You Are Building

Architecture:

    Delivery Contract
          |
          v
    Expected File Registry
          |
          v
    Arrival Registration
          |
          v
    Deadline Evaluation
          |
       +--+--+
       |     |
       v     v
    ARRIVED  NOT ARRIVED
                |
          +-----+-----+
          |           |
          v           v
        LATE        DEADLINE
                        |
                        v
                     MISSING
                        |
                        v
                  Alert / Recovery
                        |
                        v
                   Reconcile

The detector must be durable, repeatable, timezone-aware, and idempotent.

---

## 3. Learning Objectives

By the end of this recipe you should be able to:

1. Define an expected file contract.
2. Generate expectations from a delivery calendar.
3. Distinguish expected, late, and missing states.
4. Calculate deadlines using timezone-aware timestamps.
5. Match arrivals to expectations.
6. Handle required and optional files.
7. Handle manifests and dynamic file sets.
8. Handle conditional dependencies.
9. Persist historical expectations.
10. Prevent duplicate missing incidents.
11. Handle arrival/deadline races.
12. Recover when a missing file arrives later.
13. Handle explicit producer waivers.
14. Test calendar, deadline, and concurrency behavior.
15. Operate the detector in production.

---

## 4. Missing Requires an Expectation

This is unsafe:

```
files = list_incoming()
if expected_name not in files:
    mark_missing(expected_name)
```

The code does not know:

- whether the file was required;
- when it was expected;
- whether today is a delivery day;
- whether the producer changed its schedule;
- whether the file is optional;
- whether an upload is still in progress.

A correct detector starts with a contract.

---

## 5. Expected File Contract

A useful contract contains:

    source_system
    delivery_type
    business_date
    file_type
    sequence
    required
    expected_at
    deadline_at
    timezone
    calendar_id
    naming_pattern
    source_location
    dependency

Example:

```
{
  "source_system": "bank_a",
  "delivery_type": "daily",
  "business_date": "2026-09-26",
  "file_type": "transactions",
  "sequence": null,
  "required": true,
  "expected_at": "2026-09-26T06:00:00+05:00",
  "deadline_at": "2026-09-26T08:00:00+05:00",
  "timezone": "Asia/Karachi",
  "calendar_id": "PK_BANKING"
}
```

The expectation becomes the durable evidence used by the detector.

---

## 6. Expected Is Not Missing

Suppose:

    expected_at = 06:00
    deadline_at = 08:00

At 05:59:

    EXPECTED

At 06:30, depending on the contract:

    DUE or LATE

At 08:01 with no matching complete file:

    MISSING

Do not classify absence as missing merely because one polling cycle did not see the file.

---

## 7. State Machine

Use explicit states:

    EXPECTED
       |
       v
      DUE
       |
       +---------> ARRIVED
       |
       v
      LATE
       |
       +---------> ARRIVED
       |
       v
    MISSING
       |
       +---------> ARRIVED_LATE
       |
       +---------> WAIVED
       |
       +---------> CANCELLED

A state describes the current operational fact.

A history table or audit event records how the file reached that state.

---

## 8. Late vs Missing

Define the terms.

### Late

The expected arrival point has passed, but the missing evaluation threshold has not.

### Missing

The contractual missing threshold has passed and no matching file has been observed as complete.

Example:

    expected_at = 06:00
    late_after = 06:15
    missing_at = 08:00

Then:

    06:05 -> not missing
    06:30 -> LATE
    07:59 -> LATE
    08:01 -> MISSING

E65 will go deeper into late-file measurement and SLA handling.

---

## 9. Business Date Is Not Arrival Date

A file may contain:

    business_date = 2026-09-25

and arrive:

    2026-09-26 06:15

Generate the expectation from the business date and source contract.

Do not use the machine's current date as the business date unless the source contract explicitly defines that behavior.

---

## 10. Time Zones

Store timezone-aware timestamps.

Example:

```
from datetime import datetime
from zoneinfo import ZoneInfo

expected = datetime(
    2026,
    9,
    26,
    6,
    0,
    tzinfo=ZoneInfo("Europe/London"),
)

print(expected.astimezone(ZoneInfo("Asia/Karachi")))
```

Do not compare naive timestamps from systems operating in different time zones.

Do not hard-code offsets for regions that observe daylight saving time.

Use IANA timezone names.

---

## 11. Delivery Calendars

Some sources deliver:

- every day;
- weekdays;
- banking days;
- month-end;
- first business day;
- selected weekdays.

A Saturday file should not be called missing if Saturday is not a delivery day.

The expectation generator must apply the source calendar before creating an expected-file row.

---

## 12. Holiday Calendars

Monday-Friday is not always a business calendar.

For financial sources, local holidays may matter.

Persist a calendar identifier such as:

    PK_BANKING
    UK_BANKING

The calendar implementation should be versioned and testable.

Do not silently change historical expectations when the calendar changes.

---

## 13. Optional Files

Example:

    transactions.csv -> required
    chargebacks.csv -> optional

If chargebacks does not arrive, it is not MISSING.

Requiredness belongs in the expectation:

    required = true
    required = false

Do not infer requiredness from filename.

---

## 14. Conditional Files

A file can become required only when a condition is true.

Examples:

    if manifest declares chargebacks:
        chargeback_detail is required

or:

    if record_count > 0:
        transaction_detail is required

Conditional expectations need an explicit dependency.

Possible states:

    EXPECTED
    BLOCKED
    NOT_REQUIRED

Do not manufacture missing incidents for files whose expectation has not been established.

---

## 15. Manifest-Driven Expectations

A manifest can declare the expected set:

```
{
  "delivery_id": "D20260926",
  "files": [
    {
      "file_id": "transactions",
      "required": true
    },
    {
      "file_id": "customers",
      "required": true
    },
    {
      "file_id": "chargebacks",
      "required": false
    }
  ]
}
```

The manifest can be the authoritative contract for that delivery.

If the manifest itself is missing, dependent expectations may be BLOCKED rather than falsely marked missing.

---

## 16. Static Expected-File Registry

For predictable sources, use configuration:

```
EXPECTED_FILES = {
    "bank_a": [
        {
            "file_type": "transactions",
            "required": True,
            "expected_hour": 6,
            "deadline_hour": 8,
        },
        {
            "file_type": "balances",
            "required": True,
            "expected_hour": 6,
            "deadline_hour": 8,
        },
    ]
}
```

Keep schedules in versioned configuration or database records.

Do not hide source contracts deep inside procedural code.

---

## 17. Persist Expectations

A historical pipeline must be able to answer:

    What did we expect yesterday?
    When was it due?
    Which contract generated that expectation?

Do not reconstruct yesterday's expectation using today's configuration.

Persist generated expectations.

---

## 18. PostgreSQL Expected File Table

Example:

```
CREATE TABLE etl_expected_file (
    expectation_id BIGSERIAL PRIMARY KEY,
    source_system TEXT NOT NULL,
    delivery_type TEXT NOT NULL,
    business_date DATE NOT NULL,
    file_type TEXT NOT NULL,
    sequence_no INTEGER,
    required BOOLEAN NOT NULL,
    expected_at TIMESTAMPTZ NOT NULL,
    deadline_at TIMESTAMPTZ NOT NULL,
    calendar_id TEXT,
    state TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (
        source_system,
        delivery_type,
        business_date,
        file_type,
        sequence_no
    )
);
```

The unique constraint prevents duplicate expectations for the same logical delivery member.

---

## 19. Historical Contract Version

Add contract metadata when required:

```
ALTER TABLE etl_expected_file
ADD COLUMN contract_version TEXT;
```

Then an expectation can say:

    contract_version = v4

even after the source moves to v5.

Historical incident analysis remains reproducible.

---

## 20. Arrival Registry

The arrival mechanism should record complete-file evidence.

Example:

```
CREATE TABLE etl_file_arrival (
    arrival_id BIGSERIAL PRIMARY KEY,
    expectation_id BIGINT,
    source_uri TEXT NOT NULL,
    file_name TEXT NOT NULL,
    size_bytes BIGINT,
    sha256 TEXT,
    observed_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

The arrival can be discovered through polling, object events, SFTP, or another transport.

Missing detection should consume the durable arrival registry rather than caring how the file was discovered.

---

## 21. Match Arrivals Deterministically

Extract:

    source_system
    delivery_type
    business_date
    file_type
    sequence

Then match the expectation:

```
SELECT
    expectation_id,
    required,
    state
FROM etl_expected_file
WHERE source_system = %s
  AND delivery_type = %s
  AND business_date = %s
  AND file_type = %s
  AND sequence_no IS NOT DISTINCT FROM %s;
```

If no match exists, classify the file as UNEXPECTED.

An unexpected file must not satisfy another expectation simply because its name looks similar.

---

## 22. Incomplete Uploads Must Not Satisfy Expectations

An object can exist while still being incomplete.

Examples:

    upload in progress
    temporary filename
    missing completion marker
    incomplete multipart transfer

Use E51 and E52 mechanisms to establish trustworthy arrival/completeness.

The missing detector should consume:

    complete arrival

not merely:

    object exists

---

## 23. Completion Marker

Some sources use:

    transactions.csv
    transactions.csv.done

The expectation is satisfied only after the producer's completion protocol is satisfied.

Otherwise a partial file can incorrectly prevent a missing incident.

---

## 24. Missing Evaluation

At evaluation time, find required expectations whose deadline has passed:

```
SELECT
    expectation_id,
    source_system,
    delivery_type,
    business_date,
    file_type,
    deadline_at
FROM etl_expected_file
WHERE required = TRUE
  AND state IN ('EXPECTED', 'DUE', 'LATE')
  AND deadline_at < now();
```

Before changing state, check the authoritative arrival registry.

---

## 25. Deadline Race

Consider:

    07:59:59 -> source uploads
    08:00:00 -> missing evaluator runs

If arrival registration has not committed yet, the evaluator may incorrectly classify the file as missing.

Possible protections:

- transactional arrival registration;
- a short evaluation grace interval;
- source-side completion markers;
- repeat evaluation;
- conditional state transitions;
- authoritative source metadata.

The exact mechanism depends on the source.

---

## 26. Missing Detection Function

A simple service:

```
def evaluate_missing_files(db, now):
    expectations = find_due_required_expectations(db, now)

    for expectation in expectations:
        if matching_complete_file_exists(db, expectation):
            mark_arrived(db, expectation)
            continue

        marked = mark_missing_if_still_open(
            db,
            expectation.expectation_id,
            now,
        )

        if marked:
            create_missing_event(db, expectation)
```

The evaluator should be safe to run repeatedly.

---

## 27. Atomic Missing Transition

Use a conditional update:

```
UPDATE etl_expected_file
SET
    state = 'MISSING',
    missing_detected_at = now()
WHERE expectation_id = %s
  AND state IN ('EXPECTED', 'DUE', 'LATE');
```

If the update affects one row, this worker won the transition.

If it affects zero rows, another worker already changed the state.

This prevents duplicate missing incidents.

---

## 28. Missing Detection Event

Keep incident history separate from expectation state:

```
CREATE TABLE etl_missing_file_event (
    event_id BIGSERIAL PRIMARY KEY,
    expectation_id BIGINT NOT NULL,
    detected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    reason TEXT NOT NULL,
    alert_state TEXT NOT NULL,
    resolved_at TIMESTAMPTZ
);
```

The expectation is the business requirement.

The event is the operational incident.

---

## 29. Missing Reason Codes

Useful reasons:

    NO_MATCHING_FILE
    SOURCE_UNAVAILABLE
    SOURCE_WINDOW_CLOSED
    MANIFEST_INCOMPLETE
    PRODUCER_CONFIRMED_SKIP
    CONTRACT_VIOLATION

A reason code makes incident routing and analysis more useful than a generic MISSING value.

---

## 30. Alert Deduplication

Repeated evaluation must not produce:

    alert
    alert
    alert
    alert

Use expectation identity as the incident identity.

Possible incident states:

    OPEN
    ACKNOWLEDGED
    RESOLVED
    WAIVED

Repeated evaluation should update the existing incident rather than create another active incident.

---

## 31. Arrival After Missing

A missing file can arrive later.

Use:

    MISSING -> ARRIVED_LATE

Record:

    expected_at
    deadline_at
    missing_detected_at
    arrived_at
    resolved_at

Do not erase the missing event.

The file missed its deadline even though it eventually arrived.

---

## 32. Explicit Waiver

A producer may explicitly confirm:

    today's balances file will not be delivered

Transition to:

    WAIVED

Record:

- reason;
- timestamp;
- source confirmation;
- operator;
- ticket/reference if applicable.

A waiver is not the same as success.

It means the missing requirement was intentionally excused.

---

## 33. Multi-File Delivery

For a delivery:

    D100
      A -> ARRIVED
      B -> ARRIVED
      C -> MISSING

the delivery should not normally become COMPLETE.

Possible delivery state:

    PARTIAL
    MISSING_FILES
    FAILED

The delivery policy can define the exact state.

The file-level missing state should remain visible.

---

## 34. Set Reconciliation

For:

    expected = {A, B, C, D}
    observed = {A, B, D}

the missing set is:

    {C}

Python:

```
expected = {"A", "B", "C", "D"}
observed = {"A", "B", "D"}

missing = expected - observed

assert missing == {"C"}
```

This is the core mathematical model for variable-size deliveries.

---

## 35. Required and Optional Sets

Example:

    required = {A, B, C}
    optional = {D, E}

Observed:

    {A, B, D}

Then:

    missing_required = {C}
    missing_optional = {E}

Only C creates a missing-file condition.

---

## 36. Sequence-Based Expectations

Suppose a source contract requires:

    001
    002
    003
    004

Observed:

    001
    002
    004

Then:

    003

is missing.

Do not infer completeness from the highest observed sequence number.

---

## 37. Zero-File Deliveries

A source may legitimately omit a file when there is no data.

Possible contracts:

1. always produce an empty file;
2. omit the file when empty;
3. send an explicit NO_DATA marker.

The detector must follow the contract.

Never assume:

    no file = zero records

---

## 38. No-Data Marker

Example:

```
{
  "delivery_id": "D20260926",
  "file_type": "chargebacks",
  "record_count": 0,
  "status": "NO_DATA"
}
```

The expectation can become:

    SATISFIED_NO_DATA

or:

    WAIVED

depending on the business model.

Preserve the distinction from physical file arrival.

---

## 39. Dependency-Aware Detection

Suppose:

    manifest
        |
        +--> detail files

If the manifest is missing, the expected detail set may be unknowable.

Model:

    manifest = MISSING
    details = BLOCKED

rather than:

    manifest = MISSING
    detail_001 = MISSING
    detail_002 = MISSING
    detail_003 = MISSING

This prevents cascaded false alarms.

---

## 40. Severity

Severity should come from the source contract.

Possible values:

    INFO
    WARNING
    HIGH
    CRITICAL

For example:

    settlement file -> CRITICAL
    optional marketing report -> INFO

Do not infer severity from the filename.

---

## 41. Metrics

Useful metrics:

    expected_files_total
    arrived_files_total
    late_files_total
    missing_files_total
    arrived_after_missing_total
    missing_incidents_open
    missing_incidents_resolved

Useful dimensions:

    source_system
    delivery_type
    file_type
    severity

Avoid high-cardinality file IDs as metric labels.

---

## 42. Structured Logging

Example:

```
logger.error(
    "required_file_missing",
    extra={
        "expectation_id": expectation_id,
        "source_system": source_system,
        "delivery_type": delivery_type,
        "business_date": str(business_date),
        "file_type": file_type,
        "deadline_at": deadline_at.isoformat(),
        "detected_at": detected_at.isoformat(),
        "reason": reason,
    },
)
```

Do not log sensitive file contents.

---

## 43. Unit Test — Before Deadline

Given:

    expected_at = 06:00
    deadline = 08:00
    now = 07:00
    no arrival

Expected:

    not MISSING

The exact state may be DUE or LATE depending on the contract.

---

## 44. Unit Test — After Deadline

Given:

    deadline = 08:00
    now = 08:01
    no arrival

Expected:

    MISSING

if any configured evaluation grace period has also elapsed.

---

## 45. Unit Test — Arrival Before Deadline

Given:

    deadline = 08:00
    file arrived at 07:59

Expected:

    ARRIVED

The evaluator must not mark it missing.

---

## 46. Unit Test — Optional File

Given:

    required = false
    deadline passed
    no arrival

Expected:

    not MISSING

---

## 47. Unit Test — Non-Delivery Day

Given:

    Saturday
    source delivers Monday-Friday

Expected:

    no required expectation

Therefore:

    no missing incident

---

## 48. Unit Test — Holiday

Given:

    source calendar excludes a holiday

Expected:

    no required expectation for that date.

The test should use the actual calendar implementation.

---

## 49. Unit Test — Unexpected File

Observed:

    balances_extra.csv

No matching expectation exists.

Expected:

    UNEXPECTED

It must not satisfy balances.csv.

---

## 50. Unit Test — Missing Manifest

Given:

    manifest is required
    dependent detail expectations require manifest
    manifest absent

Expected:

    manifest = MISSING
    dependent details = BLOCKED or UNKNOWN

Not:

    every detail = MISSING

---

## 51. Integration Test — Multi-File Delivery

Expected:

    A
    B
    C

Observed:

    A
    C

At deadline:

    B = MISSING

Delivery:

    PARTIAL / MISSING_FILES

according to the contract.

---

## 52. Integration Test — Late Arrival

1. Generate expectation.
2. Advance time beyond expected time.
3. Verify late state.
4. Advance time beyond missing deadline.
5. Verify MISSING.
6. Deliver the file.
7. Verify ARRIVED_LATE.
8. Resolve the incident.

This validates the complete lifecycle.

---

## 53. Integration Test — Evaluator Restart

1. Generate expectation.
2. Leave the file absent.
3. Run evaluator.
4. Stop the process after the state transition.
5. Restart evaluator.
6. Run again.

Expected:

    one missing state
    one active incident

No duplicate missing alerts.

---

## 54. Integration Test — Deadline Race

Run concurrently:

    arrival registration
    missing evaluation

at the deadline boundary.

Expected:

    no contradictory permanent state

Repeat the test several times.

This catches timing bugs that ordinary unit tests miss.

---

## 55. Intentional Failure Drill — Missing Required File

1. Generate an expectation.
2. Never deliver one required file.
3. Allow expected time to pass.
4. Allow the missing deadline to pass.
5. Run evaluator.
6. Verify MISSING.
7. Verify one incident.
8. Verify metrics.
9. Verify structured logs.

This is the fundamental production drill.

---

## 56. Intentional Failure Drill — False Missing Race

1. Prepare a file to arrive just before the deadline.
2. Run arrival registration and missing evaluation concurrently.
3. Repeat the race.
4. Inspect final state.

Expected:

    no incorrect permanent MISSING state

---

## 57. Intentional Failure Drill — Missing Manifest

1. Keep the manifest absent.
2. Run evaluation.
3. Verify manifest = MISSING.
4. Verify dependent expectations = BLOCKED/UNKNOWN.
5. Confirm no cascade of false missing alerts.

---

## 58. Intentional Failure Drill — Holiday

1. Configure a non-delivery holiday.
2. Generate expectations.
3. Verify no expectation exists for that date.
4. Run evaluator.
5. Verify no missing incident.

---

## 59. Recovery — File Arrives After Missing

When the file eventually arrives:

1. register the arrival;
2. match the expectation;
3. classify ARRIVED_LATE;
4. resolve the missing incident;
5. preserve missing timestamps;
6. process the file if safe;
7. reconcile downstream state;
8. record the SLA breach.

Do not simply overwrite MISSING with ARRIVED and lose history.

---

## 60. Recovery — Producer Confirms Omission

When a producer confirms that the file will not be delivered:

1. verify the source identity;
2. record the confirmation;
3. transition to WAIVED;
4. record the reason;
5. record the operator or source reference;
6. keep the original expectation.

This makes the waiver auditable.

---

## 61. Recovery — Wrong Contract

If the detector is correct but the contract is wrong:

1. verify the producer's actual schedule;
2. correct the contract for future deliveries;
3. preserve historical expectations;
4. reconcile affected incidents;
5. document the contract change.

Do not rewrite history to hide the previous configuration.

---

## 62. Recovery — Source Outage

A source outage may cause many files to become missing.

Keep file-level state, but correlate incidents under the source outage.

Example:

    SOURCE_OUTAGE
       |
       +--> transactions missing
       +--> balances missing
       +--> customers missing

This prevents operators from treating every symptom as an independent root cause.

---

## 63. Production Runbook

When a required file is reported missing:

### Step 1 — Identify the expectation

Find:

    source_system
    delivery_type
    business_date
    file_type
    sequence

### Step 2 — Verify the contract

Check:

    requiredness
    calendar
    timezone
    expected_at
    deadline_at
    grace policy
    dependency

### Step 3 — Check arrivals

Search by deterministic file identity.

### Step 4 — Check source evidence

Inspect:

    source location
    completion marker
    manifest
    provider metadata
    source outage status

### Step 5 — Classify

Determine:

    LATE
    MISSING
    BLOCKED
    WAIVED
    UNEXPECTED

### Step 6 — Act

Retry retrieval, contact the producer, or wait according to policy.

### Step 7 — Reconcile

After arrival, verify:

    file
    extraction
    staging
    target

### Step 8 — Close

Resolve the incident without deleting its history.

---

## 64. Common Mistakes

### Mistake 1 — Treating one failed poll as missing

A file may simply be late.

**Fix:** Use contractual deadlines.

### Mistake 2 — Ignoring time zones

The detector can evaluate the deadline at the wrong instant.

**Fix:** Use timezone-aware datetimes.

### Mistake 3 — Ignoring holidays

A non-delivery day becomes a false incident.

**Fix:** Use explicit delivery calendars.

### Mistake 4 — Treating optional files as missing

Optional absence is not a contract violation.

**Fix:** Persist requiredness.

### Mistake 5 — Using a similarly named file to satisfy an expectation

An unexpected file can hide a missing required file.

**Fix:** Match deterministic identity.

### Mistake 6 — Evaluating dependencies without prerequisites

This creates cascaded false alarms.

**Fix:** Use BLOCKED/UNKNOWN states.

### Mistake 7 — Running the evaluator once

A scheduler failure can hide the incident.

**Fix:** Re-run against durable state.

### Mistake 8 — Creating an alert every cycle

This creates an alert storm.

**Fix:** Make the missing transition idempotent.

### Mistake 9 — Erasing history when the file arrives

The SLA breach still happened.

**Fix:** Use ARRIVED_LATE and retain timestamps.

### Mistake 10 — Reconstructing historical expectations from current configuration

Contracts change.

**Fix:** Persist expectations and contract versions.

### Mistake 11 — Assuming no file means zero data

Some producers use explicit no-data markers.

**Fix:** Follow the source contract.

### Mistake 12 — Treating a source outage as unrelated file failures

Many missing files can share one root cause.

**Fix:** Correlate source-level incidents.

---

## 65. Debugging Questions

When a missing alert appears, ask:

1. Was the file actually expected?
2. What is the business date?
3. Which delivery calendar applies?
4. Which timezone applies?
5. What were expected_at and deadline_at?
6. Which contract version generated the expectation?
7. Was the file observed under another identity?
8. Did the manifest declare it?
9. Was the source unavailable?
10. Was a completion marker required?
11. Is the file optional or conditional?
12. Is a dependency blocking evaluation?
13. Did the file arrive after missing was detected?
14. Was the incident already resolved or waived?

These questions should be answerable from durable metadata.

---

## 66. Production Implementation Sequence

### Step 1 — Define the contract

Document:

- file identity;
- requiredness;
- expected time;
- deadline;
- timezone;
- calendar;
- grace period;
- dependencies.

### Step 2 — Generate expectations

Create durable expected-file rows.

### Step 3 — Register arrivals

Match discovered complete files to expectations.

### Step 4 — Implement states

Use:

    EXPECTED
    DUE
    LATE
    ARRIVED
    MISSING
    ARRIVED_LATE
    WAIVED
    CANCELLED

### Step 5 — Build the evaluator

Repeatedly find expectations eligible for missing evaluation.

### Step 6 — Make transitions atomic

Prevent duplicate missing events.

### Step 7 — Add dependency handling

Block dependent expectations when prerequisites are absent.

### Step 8 — Add incident handling

Create one active incident per expectation.

### Step 9 — Add observability

Measure expected, late, missing, and recovered files separately.

### Step 10 — Run failure drills

Test races, holidays, missing manifests, outages, and late arrivals.

---

## 67. Production Checklist

### Contract

- [ ] File identity is deterministic.
- [ ] Requiredness is explicit.
- [ ] Expected time is defined.
- [ ] Missing deadline is defined.
- [ ] Timezone is explicit.
- [ ] Delivery calendar is explicit.
- [ ] Dependencies are documented.

### Expectations

- [ ] Expectations are persisted.
- [ ] Contract versions are retained.
- [ ] Duplicate expectations are prevented.
- [ ] Non-delivery days do not create expectations.

### Detection

- [ ] Arrival matching is deterministic.
- [ ] Incomplete uploads do not satisfy expectations.
- [ ] Unexpected files are separate.
- [ ] Missing evaluation runs repeatedly.
- [ ] Missing transition is idempotent.
- [ ] Deadline races are handled.

### Operations

- [ ] Missing incidents are deduplicated.
- [ ] Missing reasons are stored.
- [ ] Missing detection time is stored.
- [ ] Late arrivals resolve incidents without deleting history.
- [ ] Waivers are auditable.
- [ ] Source outages can be correlated.

### Testing

- [ ] Before-deadline test exists.
- [ ] After-deadline test exists.
- [ ] Arrival-race test exists.
- [ ] Optional-file test exists.
- [ ] Holiday test exists.
- [ ] Unexpected-file test exists.
- [ ] Missing-manifest test exists.
- [ ] Multi-file test exists.
- [ ] Evaluator-restart test exists.
- [ ] Late-arrival recovery test exists.

---

## 68. Production Tools You Should Know

### PostgreSQL

Use for:

- expected-file registry;
- state transitions;
- incident deduplication;
- historical contract evidence;
- reconciliation.

Know:

- UNIQUE constraints;
- transactions;
- conditional UPDATE;
- indexes;
- row locking.

### Python zoneinfo

Use for timezone-aware deadline calculation.

Know:

- IANA timezone names;
- aware datetimes;
- DST behavior;
- local-to-UTC conversion.

### Orchestration / Scheduling Platform

Use an orchestration system to run the evaluator repeatedly.

The evaluator itself should be:

    restartable
    idempotent
    state-driven

A missed scheduler run should delay detection, not permanently lose the incident.

---

## 69. Package Structure

A practical structure:

    etl/
      expectations/
        contracts.py
        calendar.py
        generator.py
        matcher.py

      missing/
        evaluator.py
        classifier.py
        incidents.py
        recovery.py

      arrivals/
        registry.py
        completion.py

      tests/
        test_expectations.py
        test_deadlines.py
        test_calendar.py
        test_missing.py
        test_races.py
        test_dependencies.py
        test_recovery.py

Keep contract generation, arrival matching, missing evaluation, and incident handling separate.

---

## 70. Design Principle — You Need a Contract

The pipeline needs to know:

    what
    when
    where
    required?
    according to which calendar?

Only then can it say:

    missing

Without those facts, the correct state may be:

    UNKNOWN

Do not manufacture certainty from absence.

---

## 71. Design Principle — Deadline Is a Business Rule

A deadline can depend on:

- source timezone;
- business date;
- holiday;
- delivery calendar;
- grace period;
- dependency;
- manifest;
- source-specific rules.

Treat deadline calculation as an explicit, testable rule.

---

## 72. Design Principle — Preserve the Timeline

A mature detector preserves:

    expected_at
    late_at
    deadline_at
    missing_detected_at
    arrived_at
    resolved_at

This allows operators and analysts to reconstruct exactly what happened.

Do not replace these facts with one final status.

---

## 73. Design Principle — Missing Does Not Mean Never

A missing state means:

    the required file was absent at the contractual evaluation point

It does not mean:

    the file will never arrive

Therefore:

    MISSING -> ARRIVED_LATE

is a valid production transition.

---

## 74. Definition of Done

You are done with E64 when you can independently implement a system that:

1. defines durable expected-file contracts;
2. generates expectations from business calendars;
3. handles time zones and DST correctly;
4. distinguishes required and optional files;
5. matches arrivals deterministically;
6. distinguishes unexpected files from expected files;
7. distinguishes late from missing;
8. handles manifests and dynamic file sets;
9. handles dependency-aware expectations;
10. evaluates missing files repeatedly and safely;
11. prevents duplicate missing incidents;
12. preserves the full timing history;
13. handles late arrivals after missing classification;
14. supports explicit waivers;
15. survives evaluator restarts;
16. passes deadline, calendar, race, and recovery tests;
17. provides an operational runbook.

If you can implement those mechanisms independently, you understand production missing-file detection.

---

## 75. What You Learned

The central model is:

    contract
       |
       v
    expectation
       |
       v
    arrival matching
       |
       +----------+
       |          |
       v          v
    arrived    not arrived
                  |
          +-------+-------+
          |               |
          v               v
        late            deadline
                          |
                          v
                       missing
                          |
                          v
                    recovery/alert
                          |
                          v
                    arrived_late

Key rules:

1. Absence is not missing until an expectation and deadline exist.
2. Persist expectations instead of reconstructing them later.
3. Use business calendars and time zones explicitly.
4. Separate late from missing.
5. Do not let unexpected files satisfy required expectations.
6. Model optional and conditional files explicitly.
7. Make missing-state transitions idempotent.
8. Preserve the timeline after late arrival.
9. Treat missing incidents as durable operational state.
10. Make the evaluator restartable and repeatable.

Next:

    E65 — Late File Detection
