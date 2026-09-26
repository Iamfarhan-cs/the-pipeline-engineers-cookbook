# E65 — Late File Detection

## 1. Problem Recognition

A production file pipeline must detect more than whether a file arrived.

It must also answer:

> Did the file arrive later than the delivery contract allowed?

Example:

    expected_at = 06:00
    warning_at  = 06:15
    sla_at      = 07:00
    missing_at  = 08:00

A file arriving at 06:20 is late.

A file arriving at 07:30 is late and has breached its SLA.

A file that is still absent after 08:00 is missing.

The key distinction is:

    arrived
        !=
    arrived on time
        !=
    arrived late
        !=
    missing

This recipe builds on E49, E50, E51, E52, E62, E63, and E64.

---

## 2. What You Are Building

Target architecture:

    Delivery Contract
          |
          v
    Expected File
          |
          v
    Arrival Observation
          |
          v
    Timing Classification
          |
       +--+----------------+
       |                   |
       v                   v
    ON_TIME              LATE
                           |
                           v
                      SLA_BREACH
                           |
                           v
                        MISSING
                           |
                           v
                       Recovery

E65 measures delivery timing.

E64 determines when an absent required file becomes a missing condition.

---

## 3. Learning Objectives

By the end of this recipe you should be able to:

1. Define a file delivery SLA.
2. Calculate lateness deterministically.
3. Distinguish on-time, late, SLA-breached, and missing files.
4. Use warning and escalation thresholds.
5. handle time zones and business calendars.
6. Measure lateness in seconds.
7. Track late arrivals after missing classification.
8. Prevent duplicate timing alerts.
9. Calculate SLA compliance and late rates.
10. Handle multi-file delivery timing.
11. Test timing boundaries and races.
12. Operate late-file detection in production.

---

## 4. The Core Formula

The basic calculation is:

    lateness = actual_arrival - expected_arrival

Example:

    expected = 06:00
    actual   = 06:17

Therefore:

    lateness = 17 minutes

If:

    actual <= expected + tolerance

the file is considered on time or within tolerance.

If:

    actual > expected + tolerance

the file is late.

The contract determines the exact boundary.

---

## 5. Expected Time, SLA, and Missing Deadline

These are different business thresholds.

Example:

    expected_at      = 06:00
    late_after       = 06:15
    sla_deadline     = 07:00
    missing_deadline = 08:00

Possible interpretation:

    06:00 -> expected delivery
    06:15 -> late warning becomes eligible
    07:00 -> SLA breach
    08:00 -> missing evaluation

Do not collapse all four timestamps into one field.

---

## 6. Delivery Timing Contract

A useful contract contains:

    expected_at
    late_after
    sla_deadline
    missing_deadline
    timezone
    calendar_id
    tolerance
    severity
    contract_version

Example:

```
{
  "file_type": "settlement",
  "expected_at": "06:00",
  "warning_after_minutes": 15,
  "sla_after_minutes": 60,
  "missing_after_minutes": 120,
  "timezone": "Europe/London",
  "severity": "CRITICAL",
  "contract_version": "v4"
}
```

Store the contract outside alert code so it can be reviewed and versioned.

---

## 7. Business Date Is Not Arrival Date

A file may contain:

    business_date = 2026-09-25

while it arrives:

    2026-09-26 06:20

The expectation is generated from the business date and delivery calendar.

Do not use the machine's current date as a substitute for the source business date.

---

## 8. Time Zones

Use timezone-aware timestamps.

Python:

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

actual = datetime(
    2026,
    9,
    26,
    6,
    25,
    tzinfo=ZoneInfo("Europe/London"),
)

lateness_seconds = (actual - expected).total_seconds()
print(lateness_seconds)
```

Never compare naive timestamps from different systems.

Use IANA timezone names and let the timezone database handle daylight-saving transitions.

---

## 9. Delivery Calendars

Expected delivery may depend on:

- weekdays;
- banking days;
- local holidays;
- month-end;
- first business day;
- source-specific calendars.

If Saturday is not a delivery day, there should be no late incident for Saturday.

Calendar logic belongs in expectation generation, not inside alerting code.

---

## 10. Tolerance

Some contracts allow a small tolerance.

Example:

    expected_at = 06:00
    tolerance   = 5 minutes

Then:

    06:04 -> within tolerance
    06:05 -> boundary
    06:06 -> late

The contract must define whether tolerance affects:

- operational alerting;
- SLA reporting;
- business reporting;
- all of the above.

Do not silently apply tolerance everywhere.

---

## 11. Classification Model

A practical timing classification is:

    NOT_DUE
    ON_TIME
    WITHIN_TOLERANCE
    LATE
    SLA_BREACH
    ARRIVED_AFTER_MISSING

Delivery state is separate:

    EXPECTED
    ARRIVED
    MISSING
    WAIVED

This separation is important.

A file can be:

    delivery_state = ARRIVED
    timing_status  = SLA_BREACH

That means the file eventually arrived, but it breached the SLA.

---

## 12. PostgreSQL Timing Columns

A practical expectation table can contain:

```
ALTER TABLE etl_expected_file
ADD COLUMN late_after TIMESTAMPTZ,
ADD COLUMN sla_deadline TIMESTAMPTZ,
ADD COLUMN missing_deadline TIMESTAMPTZ,
ADD COLUMN arrived_at TIMESTAMPTZ,
ADD COLUMN late_detected_at TIMESTAMPTZ,
ADD COLUMN sla_breached_at TIMESTAMPTZ,
ADD COLUMN lateness_seconds BIGINT,
ADD COLUMN timing_status TEXT;
```

The exact schema can be normalized differently, but the timing facts must remain durable.

---

## 13. Why Persist Raw Timestamps

Store:

    expected_at
    available_at
    discovered_at
    processed_at

Then derive:

    lateness
    discovery_delay
    processing_delay

Raw timestamps are more useful than storing only one derived duration.

They allow historical calculations when reporting rules change.

---

## 14. What Does Arrival Mean?

Possible timestamps include:

    producer_created_at
    upload_completed_at
    object_available_at
    manifest_published_at
    discovered_at
    registered_at

Choose the timestamp defined by the SLA contract.

For a file-delivery SLA, it is usually the point at which the complete file became available.

Do not silently use pipeline discovery time if the business SLA measures source availability.

---

## 15. Availability vs Discovery Delay

Example:

    source upload completed = 06:10
    pipeline discovered     = 06:25

If the SLA measures source availability:

    source lateness = 10 minutes

If it measures discovery:

    discovery lateness = 25 minutes

These are different operational facts.

When useful, store both.

---

## 16. File Timing Lifecycle

Example:

    EXPECTED
       |
       v
    NOT_DUE
       |
       v
    LATE
       |
       v
    SLA_BREACH
       |
       +----------------------+
       |                      |
       v                      v
    ARRIVED               MISSING
       |
       v
    ARRIVED_LATE

The exact state names can vary, but the timing transitions must be explicit.

---

## 17. Current Delay vs Final Lateness

Before a file arrives:

    current_delay = now - expected_at

After it arrives:

    final_lateness = arrived_at - expected_at

Example:

    expected = 06:00
    now      = 06:45
    file absent

Current delay:

    45 minutes

If the file arrives at 06:50:

    final lateness = 50 minutes

Do not store current delay as final lateness.

---

## 18. Late Detection Query

A repeated evaluator can find expectations eligible for timing evaluation:

```
SELECT
    expectation_id,
    file_type,
    expected_at,
    late_after,
    sla_deadline,
    missing_deadline,
    state
FROM etl_expected_file
WHERE state IN ('EXPECTED', 'DUE', 'LATE')
  AND late_after <= now()
  AND missing_deadline > now();
```

The evaluator can run every few minutes.

The database state makes the operation restartable.

---

## 19. Marking a File Late

Use a conditional state transition:

```
UPDATE etl_expected_file
SET
    state = 'LATE',
    late_detected_at = COALESCE(late_detected_at, now()),
    timing_status = 'LATE'
WHERE expectation_id = %s
  AND state IN ('EXPECTED', 'DUE')
  AND late_after <= %s;
```

Only one concurrent worker should win the transition.

This prevents duplicate late incidents.

---

## 20. SLA Breach Without Arrival

At:

    07:01

with:

    sla_deadline = 07:00
    file absent

the file can be:

    delivery_state = LATE
    timing_status  = SLA_BREACH

It does not have to be MISSING yet.

MISSING may have a later deadline.

This is an important boundary between E65 and E64.

---

## 21. SLA Breach Transition

Example:

```
UPDATE etl_expected_file
SET
    timing_status = 'SLA_BREACH',
    sla_breached_at = COALESCE(sla_breached_at, now())
WHERE expectation_id = %s
  AND timing_status = 'LATE'
  AND sla_deadline <= %s;
```

Make the transition conditional and idempotent.

---

## 22. Late Arrival

When the file arrives:

```
UPDATE etl_expected_file
SET
    arrived_at = %s,
    lateness_seconds = GREATEST(
        0,
        EXTRACT(EPOCH FROM (%s - expected_at))
    )::BIGINT,
    state = 'ARRIVED',
    timing_status = CASE
        WHEN %s > missing_deadline
            THEN 'ARRIVED_AFTER_MISSING'
        WHEN %s > sla_deadline
            THEN 'SLA_BREACH'
        WHEN %s > expected_at
            THEN 'LATE'
        ELSE
            'ON_TIME'
    END
WHERE expectation_id = %s;
```

The actual transition must use the same concurrency protections established in E64.

---

## 23. On-Time Arrival

An on-time file can be represented as:

    state = ARRIVED
    timing_status = ON_TIME

This is clearer than creating a combined state such as:

    ARRIVED_ON_TIME

Keeping dimensions separate makes reporting easier.

---

## 24. Late Warning

A useful alert progression is:

    ON_TIME
       |
       v
    LATE_WARNING
       |
       v
    SLA_BREACH
       |
       v
    MISSING

The evaluator can run frequently without sending repeated notifications.

Alert on transitions, not polling cycles.

---

## 25. Alert Deduplication

Use an incident table:

```
CREATE TABLE etl_file_timing_incident (
    incident_id BIGSERIAL PRIMARY KEY,
    expectation_id BIGINT NOT NULL,
    incident_type TEXT NOT NULL,
    severity TEXT NOT NULL,
    opened_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at TIMESTAMPTZ,
    UNIQUE (expectation_id, incident_type)
);
```

Possible incident types:

    LATE_WARNING
    SLA_BREACH
    MISSING

The unique constraint prevents duplicate active incidents for the same expectation and type.

---

## 26. Escalation Should Be State-Based

Example:

    LATE_WARNING
        -> operations notification

    SLA_BREACH
        -> on-call escalation

    MISSING
        -> critical incident

The exact recipients and channels belong to operational policy.

The detector should publish the state transition rather than embed arbitrary notification behavior.

---

## 27. Multi-File Delivery

Suppose:

    A -> 06:02
    B -> 06:47
    C -> 06:05

Then:

    A -> ON_TIME
    B -> LATE
    C -> ON_TIME

Do not reduce every file to one timing value unless the delivery contract defines a delivery-level SLA.

---

## 28. Delivery-Level SLA

A delivery can have a contract such as:

> All required files must be available by 07:00.

Then:

    A -> 06:02
    B -> 06:47
    C -> 06:05

The delivery completed at:

    06:47

Therefore:

    delivery_lateness = 47 minutes

while individual file lateness remains:

    A = 2 minutes
    B = 47 minutes
    C = 5 minutes

Both views can be useful.

---

## 29. Partial Delivery

A delivery can be:

    A -> ARRIVED
    B -> ARRIVED
    C -> LATE

while the delivery itself remains:

    PARTIAL

This allows file-level timing and delivery-level completeness to coexist.

---

## 30. Business Cutoffs

One file may have several thresholds:

    expected = 06:00
    downstream_cutoff = 06:30
    SLA = 07:00
    missing = 08:00

A 06:40 arrival may:

    be late;
    miss a downstream business cutoff;
    still be within SLA.

Represent these thresholds separately when they drive different business actions.

---

## 31. Timing Classification Function

Python:

```
def classify_timing(
    expected_at,
    actual_at,
    tolerance_seconds,
    sla_deadline,
):
    if actual_at <= expected_at:
        return "ON_TIME"

    delay = (actual_at - expected_at).total_seconds()

    if delay <= tolerance_seconds:
        return "WITHIN_TOLERANCE"

    if actual_at <= sla_deadline:
        return "LATE"

    return "SLA_BREACH"
```

Keep this function deterministic and independently testable.

---

## 32. Lateness Calculation

Python:

```
def final_lateness_seconds(
    expected_at,
    actual_at,
    tolerance_seconds=0,
):
    delay = (actual_at - expected_at).total_seconds()
    return max(0, int(delay - tolerance_seconds))
```

If:

    expected = 06:00
    actual = 06:05
    tolerance = 5 minutes

the reported lateness is zero under this tolerance policy.

If actual is 06:06, reported lateness is one minute.

The policy must define whether reported lateness is raw or tolerance-adjusted.

---

## 33. No Arrival Yet

If:

    actual_at = None

do not calculate final lateness.

Instead calculate:

    current_delay = now - expected_at

Current delay is an open operational measurement.

Final lateness exists only after arrival.

---

## 34. Late Duration Buckets

For reporting, useful buckets include:

    0-5m
    5-15m
    15-30m
    30-60m
    60-120m
    120m+

Keep raw timestamps as the source of truth.

Buckets should be reporting logic, not the primary state model.

---

## 35. Percentiles

Average lateness can hide severe outliers.

Useful measurements:

    p50
    p90
    p95
    p99

Example:

    p50 = 4 minutes
    p95 = 38 minutes
    p99 = 91 minutes

Calculate these from raw arrival and expected timestamps.

---

## 36. SLA Compliance

A basic metric is:

    SLA compliance =
        eligible files delivered within SLA
        --------------------------------
        eligible files

Define the denominator carefully.

Normally exclude:

- waived expectations;
- cancelled expectations;
- non-delivery days.

The reporting contract should define exactly what is eligible.

---

## 37. Late Rate

Another metric:

    late rate =
        late eligible files
        ------------------
        eligible arrived files

Keep this separate from:

    missing rate

A source with high lateness but no missing files has a different operational profile from a source with frequent missing files.

---

## 38. Repeated Late Deliveries

One late file is an incident.

Repeated lateness is a trend.

Track by:

    source_system
    file_type
    delivery_type
    business calendar
    business date

Example:

```
SELECT
    source_system,
    file_type,
    COUNT(*) AS deliveries,
    COUNT(*) FILTER (
        WHERE timing_status IN ('LATE', 'SLA_BREACH')
    ) AS late_deliveries
FROM etl_expected_file
WHERE business_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY source_system, file_type;
```

This supports producer-performance analysis.

---

## 39. Month-End and Peak Periods

A source may be punctual on ordinary days but slower during:

    month-end
    quarter-end
    year-end

Analyze lateness by calendar characteristics when operationally relevant.

Do not assume a normal-day SLA distribution represents every business period.

---

## 40. Contract Versioning

Suppose the source changes:

    old expected = 06:00
    new expected = 06:30

Future expectations can use:

    contract_version = v4

Historical expectations retain:

    contract_version = v3

Do not rewrite old expected_at values.

Historical SLA reporting must remain reproducible.

---

## 41. Late File and Duplicate File

A late file may be observed more than once.

Example:

    logical file arrives at 06:20
    same file is replayed at 06:30

Use E63 deterministic identity.

The replay is an observation, not a second delivery.

Do not double-count it in late-rate metrics.

---

## 42. Late File and Replacement

Suppose:

    v1 arrives at 06:30
    v2 replacement arrives at 07:00

The original delivery was already late.

Do not reset the SLA clock because the replacement arrived later.

Keep:

    original arrival
    replacement observation
    replacement reason

as separate evidence.

---

## 43. Late File and Completeness

A file can be discovered at:

    06:10

but become complete at:

    06:20

If the SLA measures complete availability:

    actual_at = 06:20

not 06:10.

Use E52 completeness semantics to select the authoritative availability timestamp.

---

## 44. Separate Source, Discovery, and Processing Delay

Example:

    source available = 06:20
    pipeline discovered = 06:22
    processing complete = 06:35

Report:

    source lateness = 20 minutes
    discovery delay = 2 minutes
    processing delay = 13 minutes

Do not report 35 minutes of source lateness.

This separation makes ownership and remediation clearer.

---

## 45. Metrics

Recommended metrics:

    expected_files_total
    files_arrived_total
    files_on_time_total
    files_late_total
    files_sla_breached_total
    files_missing_total
    current_late_files
    current_sla_breaches
    late_duration_seconds
    discovery_delay_seconds

Bounded dimensions:

    source_system
    delivery_type
    file_type
    environment
    severity

Avoid file IDs and object URIs as ordinary metric labels.

---

## 46. Structured Logging

Example:

```
logger.warning(
    "file_delivery_late",
    extra={
        "expectation_id": expectation_id,
        "source_system": source_system,
        "file_type": file_type,
        "business_date": str(business_date),
        "expected_at": expected_at.isoformat(),
        "detected_at": detected_at.isoformat(),
        "current_delay_seconds": current_delay_seconds,
        "sla_deadline": sla_deadline.isoformat(),
    },
)
```

SLA breach:

```
logger.error(
    "file_delivery_sla_breach",
    extra={
        "expectation_id": expectation_id,
        "source_system": source_system,
        "file_type": file_type,
        "expected_at": expected_at.isoformat(),
        "sla_deadline": sla_deadline.isoformat(),
    },
)
```

Do not log file contents or sensitive payloads.

---

## 47. Intentional Failure Drill — Late File

1. Generate an expectation.
2. Set expected time to 06:00.
3. Do not deliver the file by the warning threshold.
4. Run the evaluator.
5. Verify LATE.
6. Verify one warning incident.
7. Continue past the SLA.
8. Verify SLA_BREACH.
9. Deliver the file.
10. Verify ARRIVED with the correct timing status.
11. Resolve the incident.
12. Verify historical timestamps remain intact.

---

## 48. Intentional Failure Drill — Severe Late File

1. Generate a critical expectation.
2. Keep the file unavailable beyond SLA.
3. Verify escalation.
4. Continue until the missing threshold.
5. Verify E64 classifies the expectation as MISSING.
6. Deliver the file.
7. Verify ARRIVED_AFTER_MISSING.
8. Reconcile downstream processing.

This validates the boundary between E65 and E64.

---

## 49. Intentional Failure Drill — Alert Storm

1. Run the evaluator every minute.
2. Keep one file late for 30 minutes.
3. Observe incident creation.
4. Verify only one active warning incident.
5. Cross the SLA threshold.
6. Verify exactly one SLA escalation.

---

## 50. Intentional Failure Drill — Timing Boundaries

Test arrivals at:

    expected - 1 second
    expected
    expected + 1 second
    tolerance boundary
    tolerance + 1 second
    SLA boundary
    SLA + 1 second

This catches off-by-one timing defects.

---

## 51. Intentional Failure Drill — Time Zone

Test a source around:

    UTC offset transition
    DST transition
    midnight local time

Verify that the expected local business time remains correct.

Do not replace the timezone with a fixed offset.

---

## 52. Unit Test — On Time

Given:

    expected = 06:00
    actual = 05:59

Expected:

    ON_TIME

---

## 53. Unit Test — Exact Boundary

Given:

    expected = 06:00
    actual = 06:00

Expected:

    ON_TIME

---

## 54. Unit Test — Tolerance

Given:

    expected = 06:00
    tolerance = 5 minutes
    actual = 06:05

Expected:

    WITHIN_TOLERANCE

The contract must define whether the boundary is inclusive.

---

## 55. Unit Test — Late

Given:

    expected = 06:00
    tolerance = 5 minutes
    actual = 06:06

Expected:

    LATE

---

## 56. Unit Test — SLA Breach

Given:

    expected = 06:00
    SLA = 07:00
    actual = 07:01

Expected:

    SLA_BREACH

---

## 57. Unit Test — Missing With No Arrival

Given:

    expected = 06:00
    missing deadline = 08:00
    now = 08:01
    actual = None

Expected:

    delivery_state = MISSING
    timing_status = SLA_BREACH

E64 owns the missing transition.

---

## 58. Unit Test — Arrival After Missing

Given:

    missing was already detected
    actual arrival = 08:10

Expected:

    delivery_state = ARRIVED
    timing_status = ARRIVED_AFTER_MISSING

The missing incident remains in history.

---

## 59. Unit Test — Current Delay

Given:

    expected = 06:00
    now = 06:30
    actual = None

Expected:

    current_delay = 30 minutes

Do not call this final lateness.

---

## 60. Unit Test — Duplicate Observation

Given:

    same logical file observed twice

Expected:

    one logical delivery
    multiple observations only if required for audit

Do not count the replay as another late delivery.

---

## 61. Unit Test — Replacement

Given:

    original file arrived late
    replacement arrives later

Expected:

    original timing remains unchanged
    replacement is recorded separately

---

## 62. Unit Test — Non-Delivery Day

Given:

    holiday
    no expected file

Expected:

    no late incident

---

## 63. Integration Test — Multi-File Timing

Expected:

    A
    B
    C

Arrivals:

    A -> 06:02
    B -> 06:31
    C -> 06:04

Expected:

    A = ON_TIME
    B = LATE
    C = ON_TIME

Verify delivery-level timing according to its contract.

---

## 64. Integration Test — Repeated Evaluator

1. Generate an expectation.
2. Keep the file late.
3. Run the evaluator repeatedly.
4. Verify timing state remains correct.
5. Verify one active incident.
6. Cross the SLA threshold.
7. Verify exactly one escalation.

---

## 65. Integration Test — Evaluator Restart

1. Detect lateness.
2. Stop the evaluator.
3. Restart it.
4. Re-evaluate.
5. Verify no duplicate incident.
6. Verify late_detected_at did not reset.

---

## 66. Recovery — Late but Within SLA

When a file arrives after expected time but before SLA:

1. register complete arrival;
2. calculate final lateness;
3. set delivery state ARRIVED;
4. set timing status LATE;
5. resolve the warning;
6. preserve lateness metrics;
7. continue processing.

The file is late but did not breach SLA.

---

## 67. Recovery — SLA Breach

When a file arrives after SLA:

1. register the complete file;
2. calculate final lateness;
3. set timing status SLA_BREACH;
4. resolve the escalation;
5. preserve the breach timestamp;
6. process according to downstream policy;
7. record the SLA violation.

Eventual arrival does not turn an SLA breach into success.

---

## 68. Recovery — Arrived After Missing

If E64 already classified the file missing:

1. register arrival;
2. match the expectation;
3. set ARRIVED_AFTER_MISSING;
4. resolve the missing incident;
5. preserve missing and arrival timestamps;
6. reconcile downstream state;
7. retain the complete incident timeline.

---

## 69. Recovery — Source Schedule Change

When a producer changes its schedule:

1. verify the new contract;
2. version the contract;
3. apply it only to future expectations;
4. preserve historical expectations;
5. document the change.

Do not rewrite past SLA performance.

---

## 70. Production Runbook

When a file is reported late:

### Step 1 — Identify expectation

Find:

    source_system
    delivery_type
    business_date
    file_type
    sequence

### Step 2 — Check contract

Verify:

    expected_at
    tolerance
    SLA deadline
    missing deadline
    timezone
    calendar
    severity

### Step 3 — Determine state

Is it:

    ON_TIME
    LATE
    SLA_BREACH
    MISSING

### Step 4 — Check source evidence

Inspect:

    upload status
    completion marker
    source metadata
    manifest
    provider outage

### Step 5 — Separate delays

Determine:

    source availability delay
    discovery delay
    processing delay

### Step 6 — Escalate

Apply contract-defined severity.

### Step 7 — When it arrives

Record:

    available_at
    discovered_at
    arrived_at
    lateness_seconds

### Step 8 — Reconcile

Verify downstream processing and business impact.

### Step 9 — Close

Resolve the incident while preserving history.

---

## 71. Common Mistakes

### Mistake 1 — Using one boolean called late

This loses duration and SLA information.

**Fix:** Store timestamps and timing status.

### Mistake 2 — Treating current delay as final lateness

The file may arrive later.

**Fix:** Separate current delay from final lateness.

### Mistake 3 — Using discovery time as delivery time without defining it

Pipeline outages can look like producer delays.

**Fix:** Define the authoritative availability timestamp.

### Mistake 4 — Ignoring time zones

The deadline can be evaluated at the wrong instant.

**Fix:** Use timezone-aware datetimes.

### Mistake 5 — Ignoring tolerance boundaries

Off-by-one rules create inconsistent reports.

**Fix:** Test exact boundaries.

### Mistake 6 — Alerting every evaluator cycle

One late file creates hundreds of alerts.

**Fix:** Alert on state transitions.

### Mistake 7 — Resetting lateness after replacement

The original delivery was still late.

**Fix:** Preserve observation and replacement history.

### Mistake 8 — Treating every late file as missing

A late file can still arrive before the missing threshold.

**Fix:** Keep E65 and E64 semantics separate.

### Mistake 9 — Treating eventual arrival as SLA success

Arrival after SLA is still a breach.

**Fix:** Preserve SLA_BREACH.

### Mistake 10 — Reporting only averages

Severe outliers disappear inside averages.

**Fix:** Track percentiles and raw timestamps.

### Mistake 11 — Including waived files in SLA denominators

This distorts performance.

**Fix:** Define the eligible population explicitly.

### Mistake 12 — Using unique file IDs as metric labels

This creates high-cardinality telemetry.

**Fix:** Use bounded dimensions.

---

## 72. Debugging Questions

When a late-file alert appears, ask:

1. What was expected_at?
2. What timezone was used?
3. What calendar was used?
4. What tolerance applies?
5. What is the SLA deadline?
6. What timestamp defines actual availability?
7. Was the file complete when observed?
8. Was discovery delayed?
9. Was the source unavailable?
10. Did the producer change its schedule?
11. Is this part of a multi-file delivery?
12. Was there a duplicate or replacement?
13. Has the file crossed the missing threshold?
14. Is an incident already open?
15. Is the lateness represented correctly in historical metrics?

These questions should be answerable from durable metadata.

---

## 73. Production Implementation Sequence

### Step 1 — Define timing contract

Document:

- expected time;
- tolerance;
- warning threshold;
- SLA deadline;
- missing deadline;
- timezone;
- calendar;
- severity.

### Step 2 — Persist expectations

Store expected timestamps and contract version.

### Step 3 — Define authoritative availability

Choose whether SLA uses:

    upload complete
    manifest publication
    object availability
    pipeline discovery

### Step 4 — Register complete arrivals

Use E51 and E52 mechanisms.

### Step 5 — Calculate timing

Implement deterministic classification.

### Step 6 — Separate state dimensions

Keep delivery state separate from timing status.

### Step 7 — Add threshold transitions

Implement:

    LATE
    SLA_BREACH
    MISSING

### Step 8 — Add incidents

Deduplicate alerts by expectation and incident type.

### Step 9 — Preserve history

Record expected, availability, discovery, processing, and resolution timestamps.

### Step 10 — Add metrics

Track on-time, late, SLA-breach, and missing rates.

### Step 11 — Run failure drills

Test boundaries, races, restarts, time zones, duplicates, and recovery.

---

## 74. Production Checklist

### Contract

- [ ] Expected time is defined.
- [ ] Tolerance is defined.
- [ ] SLA deadline is defined.
- [ ] Missing deadline is defined.
- [ ] Timezone is explicit.
- [ ] Calendar is explicit.
- [ ] Severity is explicit.
- [ ] Contract version is retained.

### Timing

- [ ] Availability timestamp is defined.
- [ ] Discovery timestamp is separate.
- [ ] Processing timestamp is separate.
- [ ] Current delay is separate from final lateness.
- [ ] Timing classification is deterministic.
- [ ] Boundary behavior is tested.

### State

- [ ] Delivery state is separate from timing status.
- [ ] Late transition is idempotent.
- [ ] SLA transition is idempotent.
- [ ] Missing is handled by E64.
- [ ] Late arrival after missing is preserved.

### Alerting

- [ ] Alerts are threshold-driven.
- [ ] Alerts are deduplicated.
- [ ] Escalation is state-based.
- [ ] One late file cannot create an alert storm.

### Metrics

- [ ] On-time rate is measured.
- [ ] Late rate is measured.
- [ ] SLA compliance is measured.
- [ ] Missing rate is separate.
- [ ] Percentiles are available.
- [ ] Metric labels remain bounded.

### Testing

- [ ] On-time test exists.
- [ ] Tolerance boundary test exists.
- [ ] Late test exists.
- [ ] SLA breach test exists.
- [ ] Missing test exists.
- [ ] Late-after-missing test exists.
- [ ] Duplicate observation test exists.
- [ ] Replacement test exists.
- [ ] Holiday test exists.
- [ ] Timezone/DST test exists.
- [ ] Evaluator restart test exists.

---

## 75. Production Tools You Should Know

### PostgreSQL

Use for:

- expected-file contracts;
- timing state;
- incident deduplication;
- historical timestamps;
- SLA reporting.

Important features:

- TIMESTAMPTZ;
- conditional UPDATE;
- transactions;
- indexes;
- aggregate functions.

### Python zoneinfo

Use for:

- local business times;
- timezone conversion;
- DST-aware deadlines;
- deterministic timing tests.

### Prometheus

Use for:

- late-file counters;
- SLA-breach counters;
- current late-file gauges;
- duration histograms.

Prefer bounded labels:

    source_system
    file_type
    environment
    severity

Do not use unique file identifiers as labels.

---

## 76. Package Structure

A practical structure:

    etl/
      expectations/
        contracts.py
        calendar.py
        generator.py

      timing/
        classifier.py
        evaluator.py
        sla.py
        incidents.py

      arrivals/
        registry.py
        completion.py

      metrics/
        file_delivery.py

      tests/
        test_timing.py
        test_sla.py
        test_boundaries.py
        test_timezone.py
        test_races.py
        test_recovery.py

Keep timing classification independent from notification delivery.

---

## 77. Design Principle — Preserve Raw Time

Always preserve:

    expected_at
    available_at
    discovered_at
    processed_at

Then derive:

    lateness
    discovery delay
    processing delay

Do not store only derived durations.

Raw timestamps are the source of truth.

---

## 78. Design Principle — Separate Business State From Timing

A file can be:

    delivery_state = ARRIVED
    timing_status = SLA_BREACH

This is clearer than creating one huge enum containing every combination.

Separate dimensions produce simpler queries and cleaner operational logic.

---

## 79. Design Principle — Missing and Late Are Different

A file can be:

    late but present

or:

    missing and never observed

or:

    missing at deadline, then arrived later

E64 answers:

> Was the file absent at the missing evaluation point?

E65 answers:

> How late was the delivery and did it breach its SLA?

Do not collapse both mechanisms into one status.

---

## 80. Design Principle — Alert on Transitions

The evaluator may run:

    every minute

The alert should not.

Use:

    state transition
        ->
    incident

not:

    scheduler iteration
        ->
    incident

This makes continuous evaluation safe.

---

## 81. Definition of Done

You are done with E65 when you can independently implement a system that:

1. defines expected delivery times;
2. defines lateness tolerance;
3. defines SLA and missing thresholds;
4. handles source time zones and DST;
5. selects an authoritative availability timestamp;
6. separates current delay from final lateness;
7. calculates lateness deterministically;
8. separates delivery state from timing status;
9. detects late files repeatedly and idempotently;
10. detects SLA breaches before missing classification;
11. integrates correctly with E64;
12. handles arrivals after missing classification;
13. handles duplicates and replacement files;
14. calculates file-level and delivery-level timing when appropriate;
15. produces bounded operational metrics;
16. prevents alert storms;
17. preserves historical contract versions;
18. passes timing boundary and recovery tests;
19. provides a production runbook.

If you can implement these mechanisms independently, you understand production late-file detection.

---

## 82. What You Learned

The central model is:

    contract
       |
       v
    expected_at
       |
       v
    availability
       |
       v
    timing classification
       |
       +------------+-------------+
       |            |             |
       v            v             v
    ON_TIME       LATE       SLA_BREACH
                                  |
                                  v
                               MISSING
                                  |
                                  v
                         ARRIVED_AFTER_MISSING

Key rules:

1. Define expected delivery time before measuring lateness.
2. Use timezone-aware timestamps.
3. Separate current delay from final lateness.
4. Define the authoritative availability timestamp.
5. Keep delivery state separate from timing status.
6. Treat tolerance as an explicit contract rule.
7. Treat SLA breach separately from missing.
8. Preserve raw timestamps.
9. Alert on state transitions, not scheduler iterations.
10. Keep late rate and missing rate separate.
11. Preserve historical contract versions.
12. Never erase late or SLA-breach history when the file eventually arrives.

E65 completes the ETL file-detection section of the cookbook.
