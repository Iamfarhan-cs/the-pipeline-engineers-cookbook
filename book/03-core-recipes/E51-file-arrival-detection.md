# E51 — File Arrival Detection

## 1. Problem Recognition

File discovery answers:

> What files can I currently see?

Arrival detection answers:

> **Did the file or delivery we expected actually arrive within the expected time window?**

A scanner may run successfully and find nothing. That does not automatically mean the producer failed, the file is late, the file was never expected, the source is unavailable, or the pipeline configuration is wrong.

Production ETL needs an explicit arrival contract.

```
EXPECTED
partner_a / payments / 2026-09-26 / sequence 001
        ↓
WAIT
        ↓
DISCOVER
        ↓
MATCH
        ↓
ARRIVED
```

Arrival detection is therefore a control-plane problem.

---

## 2. Arrival Versus Discovery

Do not treat these as the same mechanism.

### Discovery

Find what currently exists.

```
LIST SOURCE
    ↓
FIND CANDIDATES
```

### Arrival detection

Compare what should exist against what has been observed.

```
EXPECTED DELIVERIES
       ↓
OBSERVED DELIVERIES
       ↓
COMPARE
       ↓
ARRIVED / PENDING / LATE / UNKNOWN
```

A source can be successfully scanned while an expected delivery is still missing.

---

## 3. Define an Arrival Contract

Before implementing detection, define:

- source system;
- dataset;
- business date;
- expected delivery frequency;
- expected sequence;
- expected arrival window;
- timezone;
- grace period;
- delivery identifier;
- whether zero-file days are valid;
- whether holidays/weekends change expectations;
- escalation policy.

Example:

| Field | Example |
|---|---|
| source | bank_a |
| dataset | payments |
| frequency | daily |
| expected time | 02:00 |
| timezone | UTC |
| grace period | 30 minutes |
| sequence | 001 |
| business date | 2026-09-26 |

Without these rules, "late" has no deterministic meaning.

---

## 4. Expected Delivery Is the Source of Truth

Create an expected-delivery record before waiting for the file.

Example:

```
business_date = 2026-09-26
source = bank_a
dataset = payments
sequence = 001
expected_at = 02:00 UTC
deadline_at = 02:30 UTC
status = EXPECTED
```

Then discovery updates the record when the matching file is found.

This creates a durable lifecycle:

```
EXPECTED
   ↓
ARRIVED
   ↓
READY
   ↓
PROCESSED
```

Failure path:

```
EXPECTED
   ↓
DEADLINE PASSED
   ↓
LATE
```

---

## 5. Expected Delivery Table

A PostgreSQL implementation can start with:

```
CREATE TABLE expected_file_delivery (
    delivery_id UUID PRIMARY KEY,
    source_system TEXT NOT NULL,
    dataset TEXT NOT NULL,
    business_date DATE NOT NULL,
    sequence_number INTEGER,
    expected_at TIMESTAMPTZ NOT NULL,
    deadline_at TIMESTAMPTZ NOT NULL,
    arrived_at TIMESTAMPTZ,
    status TEXT NOT NULL,
    discovered_file_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (
        source_system,
        dataset,
        business_date,
        sequence_number
    )
);
```

The uniqueness rule represents the logical delivery identity. Adjust it when the source contract differs.

---

## 6. Delivery Identity

A file path is not necessarily the delivery identity.

For example:

```
s3://partner-a/inbound/payments_20260926_001.csv
```

may identify the physical object.

The logical delivery could be:

```
partner_a
payments
2026-09-26
001
```

Store the logical identity separately.

This allows the pipeline to detect:

- the same delivery under a different path;
- a replay;
- a replacement;
- a duplicate;
- an unexpected file.

---

## 7. Generate Expectations

For a daily source:

```
2026-09-26 → expected
2026-09-27 → expected
2026-09-28 → expected
```

A scheduler can create these expectations before the deadline.

Example Python:

```
from datetime import date, datetime, time
from zoneinfo import ZoneInfo


def expected_delivery(
    business_date: date,
    hour: int,
    minute: int,
    timezone: str,
):
    tz = ZoneInfo(timezone)

    return datetime.combine(
        business_date,
        time(hour, minute),
        tzinfo=tz
    )
```

The important point is that expected delivery time should be timezone-aware.

---

## 8. Do Not Assume Every Day Requires a File

Some sources do not deliver on weekends, public holidays, month-end exceptions, or maintenance windows.

Therefore:

```
NO FILE
   ≠
MISSING FILE
```

The expectation generator must know the source calendar.

The final expectation policy should come from the source contract.

---

## 9. Arrival Windows

Define a window instead of using one exact instant.

Example:

```
expected_at = 02:00
deadline_at = 02:30
```

States:

```
before 02:00 → WAITING
02:00–02:30  → EXPECTED / WAITING
after 02:30  → LATE
```

Do not mark a file late merely because a scanner ran before the expected time.

---

## 10. Grace Periods

Source systems can have normal variation.

A grace period prevents every small delay from becoming an incident.

Example:

```
expected_at = 02:00
grace = 30 minutes
deadline = 02:30
```

The grace period should be part of the contract rather than hidden inside code.

Do not make it so large that genuine source failures become invisible.

---

## 11. Timezones

A common production failure is comparing local source schedules with UTC incorrectly.

Example:

```
Source contract:
02:00 Europe/Helsinki
```

Convert the schedule using an actual timezone database.

Do not manually add a fixed offset.

Daylight-saving changes can alter the UTC representation of local time.

Use timezone-aware timestamps throughout the control plane.

---

## 12. Match Arrivals to Expectations

Once E49 discovers a file, match its parsed metadata against expected deliveries.

Example:

```
DISCOVERED FILE
source = bank_a
dataset = payments
business_date = 2026-09-26
sequence = 001
        ↓
EXPECTED DELIVERY
same logical identity
        ↓
MATCH
        ↓
ARRIVED
```

The match should use the source-defined business identity rather than filename similarity.

---

## 13. Arrival Registration

Example SQL:

```
UPDATE expected_file_delivery
SET
    arrived_at = COALESCE(arrived_at, now()),
    discovered_file_id = :file_id,
    status = CASE
        WHEN status = 'LATE' THEN 'ARRIVED_LATE'
        ELSE 'ARRIVED'
    END,
    updated_at = now()
WHERE source_system = :source_system
  AND dataset = :dataset
  AND business_date = :business_date
  AND sequence_number = :sequence_number
  AND status IN ('EXPECTED', 'LATE');
```

Use an explicit state policy rather than blindly overwriting status.

---

## 14. Idempotent Arrival Registration

Discovery may observe the same file repeatedly.

Arrival registration must therefore be idempotent.

The operation:

```
EXPECTED → ARRIVED
```

should be safe to execute again.

Prefer preserving the first accepted arrival timestamp rather than replacing it on every discovery.

---

## 15. Unexpected Arrivals

A discovered file may have no matching expectation.

Example:

```
EXPECTED:
payments / 2026-09-26 / 001

OBSERVED:
payments / 2026-09-26 / 002
```

Possible explanations:

- producer sent an extra partition;
- sequence configuration is wrong;
- expectation generation is incomplete;
- producer replayed data;
- wrong source file arrived;
- business contract changed.

Do not automatically classify the file as valid.

Use a state such as UNEXPECTED and route it through the appropriate investigation policy.

---

## 16. Multiple Files Per Delivery

Some sources produce multiple files per business date.

Example:

```
payments_20260926_001.csv
payments_20260926_002.csv
payments_20260926_003.csv
```

The expectation must define whether:

- all sequences are known in advance;
- the final sequence is known;
- a manifest defines the complete set;
- files arrive dynamically.

Do not assume sequence 001 is the entire daily delivery when the source contract defines multiple partitions.

---

## 17. Arrival Detection With a Manifest

A manifest can define expected files explicitly.

Example:

```
manifest_20260926.json
```

containing:

```
payments_20260926_001.csv
payments_20260926_002.csv
payments_20260926_003.csv
```

Then:

```
MANIFEST
   ↓
EXPECTED FILE SET
   ↓
DISCOVERY
   ↓
COMPARE
   ↓
ARRIVAL STATUS
```

A manifest can be stronger than guessing the expected sequence count.

Manifest validation itself becomes part of the source contract.

---

## 18. Arrival Detection Query

Find deliveries that have passed their deadline without arrival:

```
SELECT
    delivery_id,
    source_system,
    dataset,
    business_date,
    sequence_number,
    expected_at,
    deadline_at
FROM expected_file_delivery
WHERE status = 'EXPECTED'
  AND deadline_at <= now()
ORDER BY deadline_at;
```

These records are candidates for late classification.

---

## 19. Mark Late Atomically

Use a controlled state transition.

```
UPDATE expected_file_delivery
SET
    status = 'LATE',
    updated_at = now()
WHERE status = 'EXPECTED'
  AND deadline_at <= now();
```

The condition on the current state prevents an already-arrived delivery from being incorrectly marked late.

For more complex workflows, use row locking or a state-transition service.

---

## 20. Late Arrival

A file can arrive after being marked late.

That is not the same as never arriving.

Lifecycle:

```
EXPECTED
   ↓
LATE
   ↓
ARRIVED_LATE
   ↓
READY
   ↓
PROCESSED
```

Preserve the fact that the delivery missed its expected deadline.

Do not overwrite the historical state with simply ARRIVED.

---

## 21. Arrival Time Versus Processing Time

These timestamps answer different questions:

```
arrived_at
processed_at
completed_at
```

Example:

```
02:18 → file arrived
02:30 → file registered
02:45 → processing started
02:52 → processing completed
```

Do not use processing time as evidence of source arrival.

---

## 22. Source Arrival Evidence

Depending on the transport, arrival evidence can include:

### Local filesystem

- first observed timestamp;
- file metadata;
- file identity where relevant;
- size;
- modification timestamp.

### Object storage

- object key;
- object version;
- last-modified metadata;
- provider event;
- object metadata.

### SFTP

- remote path;
- remote metadata;
- observed timestamp.

The exact timestamp semantics differ by transport.

Record what the source actually provides instead of pretending all timestamps mean the same thing.

---

## 23. Polling Versus Event-Driven Arrival

Two common architectures exist.

### Polling

```
EVERY 5 MINUTES
      ↓
LIST SOURCE
      ↓
MATCH EXPECTATIONS
      ↓
REGISTER ARRIVALS
```

Advantages:

- simple;
- works with many legacy systems;
- easy to reason about.

Costs:

- repeated listing;
- detection latency;
- source polling load.

### Event-driven

```
FILE CREATED
     ↓
SOURCE EVENT
     ↓
ARRIVAL HANDLER
     ↓
REGISTER
```

Advantages:

- lower detection latency;
- less repeated listing.

Costs:

- event delivery failures;
- duplicate events;
- ordering concerns;
- more infrastructure.

The control-plane state should remain durable regardless of the triggering mechanism.

---

## 24. Duplicate Events

Event-driven systems may deliver the same event more than once.

Example:

```
EVENT A → file X
EVENT A → file X
EVENT A → file X
```

The arrival operation must remain idempotent.

Use durable file identity and database constraints rather than assuming exactly-once event delivery.

---

## 25. Arrival Detection Python Skeleton

```
from dataclasses import dataclass
from datetime import datetime


@dataclass(frozen=True)
class ExpectedDelivery:
    source_system: str
    dataset: str
    business_date: str
    sequence_number: int
    expected_at: datetime
    deadline_at: datetime


def is_late(
    delivery: ExpectedDelivery,
    now: datetime,
) -> bool:
    return now >= delivery.deadline_at
```

Keep the pure time decision separate from database state transitions.

That makes the logic easy to test.

---

## 26. Database Arrival Worker

A simplified worker:

```
def mark_late_deliveries(conn):
    with conn.cursor() as cur:
        cur.execute(
            """
            UPDATE expected_file_delivery
            SET
                status = 'LATE',
                updated_at = now()
            WHERE status = 'EXPECTED'
              AND deadline_at <= now()
            RETURNING delivery_id
            """
        )

        return cur.fetchall()
```

Commit the state transition as a transaction.

The scheduler should be able to run this repeatedly without creating duplicate late records.

---

## 27. Testing

### Unit tests

Test:

- before expected time;
- exactly at expected time;
- inside grace period;
- exactly at deadline;
- after deadline;
- already arrived;
- already late;
- late arrival;
- unexpected file;
- duplicate arrival;
- no-file business day.

Use explicit timestamps in tests rather than relying on wall-clock time.

### Integration tests

Test:

- expected-delivery insertion;
- discovery-to-arrival matching;
- idempotent registration;
- late transition;
- late arrival transition;
- concurrent arrival handling;
- duplicate events.

---

## 28. Intentional Failure Drills

### Drill 1 — Do not deliver the expected file

Verify the delivery becomes LATE only after the configured deadline.

### Drill 2 — Deliver just before the deadline

Verify it is classified as on time.

### Drill 3 — Deliver after the deadline

Verify ARRIVED_LATE preserves the late condition.

### Drill 4 — Send an unexpected sequence

Verify it does not silently satisfy another expectation.

### Drill 5 — Run two arrival workers

Verify state transitions remain correct.

### Drill 6 — Send the same event repeatedly

Verify arrival registration remains idempotent.

### Drill 7 — Configure the wrong timezone

Verify tests detect the schedule mismatch.

### Drill 8 — Configure a source holiday

Verify a non-delivery day is not falsely classified as missing.

---

## 29. Observability

Track:

```
file_arrival_expected_total
file_arrival_detected_total
file_arrival_late_total
file_arrival_unexpected_total
file_arrival_duplicate_total
file_arrival_detection_latency_seconds
file_arrival_deadline_breaches_total
```

Useful structured fields:

```
source_system
dataset
business_date
sequence_number
delivery_id
expected_at
deadline_at
arrived_at
status
discovery_run_id
```

Important operational measurements include:

- number of pending deliveries;
- number of late deliveries;
- time since deadline;
- arrival detection latency;
- source-specific late-arrival rate.

Avoid putting delivery IDs or filenames into metric labels if cardinality becomes large.

---

## 30. Alerts

Arrival alerts should represent meaningful operational conditions.

Examples:

### Warning

A delivery is approaching its deadline.

### Critical

A required delivery has passed its deadline.

### Recovery

A previously late delivery has arrived.

An alert should contain enough context to act:

```
source
dataset
business_date
sequence
expected_at
deadline_at
current_status
```

Do not alert merely because the scanner found zero files before the expected arrival window.

---

## 31. Recovery Runbook

### Expected file is missing

1. Confirm the delivery was actually expected.
2. Check source schedule/calendar.
3. Verify source connectivity.
4. Check discovery runs.
5. Check source-side delivery status.
6. Check whether the file arrived under an unexpected name.
7. Check whether another location contains the delivery.
8. Escalate to the producer when the deadline is exceeded.
9. Preserve the LATE state until the delivery arrives or is formally waived.

### File arrives late

1. Preserve the original late state.
2. Register the arrival timestamp.
3. Transition to ARRIVED_LATE.
4. Continue readiness/content validation.
5. Process according to late-data policy.
6. Record the delay for source performance monitoring.

### Unexpected file arrives

1. Do not attach it to a different expected delivery automatically.
2. Parse its identity.
3. Compare against the delivery contract.
4. Determine whether it is a replay, extra partition, correction, or producer error.
5. Quarantine or accept according to policy.

### False late alert

1. Check timezone.
2. Check expected-delivery generation.
3. Check source calendar.
4. Check clock synchronization.
5. Check whether arrival registration failed.
6. Correct the control-plane issue.
7. Reconcile affected alerts.

---

## 32. Production Tools You Should Know

### 1. Airflow

Learn scheduled expectation generation, sensors, task state, retries, and deadline-oriented pipeline operations.

### 2. Object-storage event systems

Learn object-created events, duplicate event handling, event routing, and durable arrival registration.

### 3. Prometheus/Grafana

Learn how to expose pending deliveries, late arrivals, detection latency, and source-specific arrival health.

These tools implement scheduling, triggering, and monitoring. The underlying mechanism remains:

```
EXPECTED
   ↓
OBSERVED
   ↓
MATCH
   ↓
ARRIVED / LATE / UNEXPECTED
```

---

## 33. Common Mistakes

1. Treating successful discovery as proof of expected arrival.
2. Not creating expected-delivery records.
3. Defining late without a deadline.
4. Ignoring source timezones.
5. Assuming every calendar day requires a file.
6. Using processing time as arrival time.
7. Treating missing and incomplete files as the same problem.
8. Automatically matching unexpected files to nearby expectations.
9. Assuming event delivery is exactly once.
10. Updating arrival timestamps on every repeated discovery.
11. Having no durable delivery identity.
12. Ignoring late-arrival history.
13. Hard-coding source schedules in application logic.
14. Using polling intervals as the definition of lateness.
15. Alerting before the expected delivery window.
16. Ignoring source calendars and holidays.
17. Failing to distinguish duplicate events from duplicate deliveries.
18. Having no recovery state for late arrivals.

---

## 34. Definition of Done

You can independently:

- define an expected file-delivery contract;
- generate durable delivery expectations;
- define arrival windows and deadlines;
- handle source timezones;
- account for source calendars;
- distinguish discovery from arrival;
- match observed files to logical deliveries;
- register arrivals idempotently;
- detect unexpected deliveries;
- detect late deliveries;
- preserve late-arrival history;
- handle polling and event-driven detection;
- handle duplicate events;
- test timing boundaries;
- monitor arrival health;
- create actionable alerts;
- recover missing and late deliveries safely.

---

## 35. What You Learned

> **File arrival detection is the mechanism that converts "we expected data" into measurable operational state.**

The production pattern is:

```
DEFINE DELIVERY CONTRACT
        ↓
CREATE EXPECTATION
        ↓
WAIT FOR DELIVERY WINDOW
        ↓
DISCOVER / RECEIVE EVENT
        ↓
MATCH BUSINESS IDENTITY
        ↓
REGISTER ARRIVAL
        ↓
CLASSIFY ON-TIME / LATE / UNEXPECTED
        ↓
HAND OFF TO READINESS AND VALIDATION
```

The key question is:

> **Can I prove which deliveries were expected, which arrived, which were late, which were unexpected, and why?**

That is the foundation for reliable file-based ETL operations.
