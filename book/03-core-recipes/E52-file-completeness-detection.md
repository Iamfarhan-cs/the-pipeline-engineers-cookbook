# E52 — File Completeness Detection

## 1. Problem Recognition

File arrival answers:

> Did the expected delivery arrive?

Completeness answers:

> **Did we receive everything that belongs to that delivery?**

These are different production questions.

A file can arrive successfully while being incomplete.

Example:

```
EXPECTED:
001.csv
002.csv
003.csv

OBSERVED:
001.csv
002.csv
```

The delivery has arrived, but the expected set is incomplete.

Completeness detection prevents downstream ETL from processing partial deliveries as if they were complete.

---

## 2. Arrival Versus Completeness

The distinction is fundamental.

### Arrival

```
EXPECTED DELIVERY
      ↓
FILE OBSERVED
      ↓
ARRIVED
```

### Completeness

```
EXPECTED DELIVERY SET
      ↓
OBSERVED DELIVERY SET
      ↓
COMPARE
      ↓
COMPLETE / INCOMPLETE
```

A production pipeline should not assume:

```
ARRIVED = COMPLETE
```

Instead:

```
ARRIVED
   ↓
COMPLETENESS CHECK
   ↓
COMPLETE
   ↓
READY FOR EXTRACTION
```

---

## 3. Define a Completeness Contract

Before implementing detection, define how completeness is proven.

Possible contracts include:

1. known expected file count;
2. known sequence range;
3. producer manifest;
4. expected partition list;
5. control-total record;
6. end-of-delivery marker;
7. source API completion status;
8. stable object set after a defined window.

The correct mechanism comes from the producer contract.

Do not invent completeness rules from whatever files happen to be visible.

---

## 4. Completeness Contract Example

Suppose a source sends:

```
payments_20260926_001.csv
payments_20260926_002.csv
payments_20260926_003.csv
```

The contract might state:

```
business_date = 2026-09-26
expected_sequences = 1..3
```

Then:

```
{001, 002, 003} = COMPLETE
{001, 002}      = INCOMPLETE
{001, 003}      = INCOMPLETE
{002, 003}      = INCOMPLETE
```

This is deterministic.

---

## 5. Why File Count Alone Is Dangerous

Suppose three files are expected.

Observed:

```
file_a.csv
file_b.csv
file_x.csv
```

Count comparison gives:

```
expected = 3
observed = 3
```

But the delivery is still incomplete because the identities differ.

Therefore:

> **Completeness is an identity-set problem, not merely a count problem.**

Compare logical identities.

---

## 6. Expected Set and Observed Set

Represent the expected files as a set:

```
EXPECTED = {001, 002, 003}
```

Observed:

```
OBSERVED = {001, 002}
```

Missing:

```
EXPECTED - OBSERVED = {003}
```

Unexpected:

```
OBSERVED - EXPECTED = {}
```

Complete when:

```
EXPECTED == OBSERVED
```

This simple model is useful for both code and database design.

---

## 7. Sequence-Based Completeness

If a source contract defines contiguous sequences:

```
expected_start = 1
expected_end = 5
```

Then the expected set is:

```
{1, 2, 3, 4, 5}
```

Observed:

```
{1, 2, 3, 5}
```

The missing sequence is:

```
4
```

Do not infer completeness merely from the highest observed sequence:

```
max(observed) = 5
```

That does not prove sequence 4 arrived.

---

## 8. Manifest-Based Completeness

A producer manifest can explicitly declare the delivery set.

Example:

```
manifest_20260926.json
```

Conceptually:

```
{
  "business_date": "2026-09-26",
  "files": [
    "payments_20260926_001.csv",
    "payments_20260926_002.csv",
    "payments_20260926_003.csv"
  ]
}
```

The manifest becomes the expected set.

Then:

```
MANIFEST
   ↓
EXPECTED FILES
   ↓
DISCOVER
   ↓
OBSERVED FILES
   ↓
SET COMPARISON
```

A manifest can also contain checksums and record counts.

---

## 9. End-of-Delivery Marker

Some producers send a completion marker.

Example:

```
payments_001.csv
payments_002.csv
payments_003.csv
payments_20260926.done
```

The marker means:

> The producer considers the delivery complete.

However, the marker should not automatically override missing-file checks.

A safer flow is:

```
DATA FILES
   +
DONE MARKER
   ↓
VERIFY EXPECTED SET
   ↓
COMPLETE
```

If the producer contract explicitly defines the marker as authoritative, document that rule.

---

## 10. Control Totals

Completeness can also involve totals.

Example:

```
manifest:
file_count = 3
record_count = 1,250,000
amount_total = 9,840,000.25
```

Observed files may contain:

```
file_count = 3
record_count = 1,180,000
amount_total = 9,100,000.25
```

The file set is complete by identity but the data delivery is not complete according to the producer's control totals.

This leads to an important distinction:

- file completeness;
- content completeness;
- business-total reconciliation.

Do not collapse these into one check.

---

## 11. Completeness State Machine

A useful state model is:

```
ARRIVED
   ↓
ASSEMBLING
   ↓
COMPLETENESS_CHECK
   ├── COMPLETE
   ├── INCOMPLETE
   └── UNEXPECTED
```

Late data can produce:

```
ARRIVED_LATE
   ↓
ASSEMBLING
   ↓
COMPLETENESS_CHECK
```

A complete late delivery is still late.

Do not lose arrival-timing information when completeness becomes true.

---

## 12. Completeness Table

A delivery-level table can store the state:

```
CREATE TABLE file_delivery_completeness (
    delivery_id UUID PRIMARY KEY,
    expected_file_count INTEGER,
    observed_file_count INTEGER NOT NULL DEFAULT 0,
    missing_file_count INTEGER NOT NULL DEFAULT 0,
    unexpected_file_count INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL,
    checked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

For high-value pipelines, also persist the individual expected/observed identities.

---

## 13. Delivery File Ledger

A normalized ledger is often more useful than storing only counts.

```
CREATE TABLE delivery_file (
    delivery_id UUID NOT NULL,
    file_id UUID NOT NULL,
    logical_sequence INTEGER,
    expected BOOLEAN NOT NULL,
    observed_at TIMESTAMPTZ,
    status TEXT NOT NULL,

    PRIMARY KEY (delivery_id, file_id)
);
```

Possible statuses:

```
EXPECTED
OBSERVED
MISSING
UNEXPECTED
DUPLICATE
```

The exact model depends on the source contract.

---

## 14. Idempotent File Registration

The same file may be discovered multiple times.

Registration should be idempotent.

Example:

```
INSERT INTO delivery_file (
    delivery_id,
    file_id,
    logical_sequence,
    expected,
    observed_at,
    status
)
VALUES (
    :delivery_id,
    :file_id,
    :sequence,
    TRUE,
    now(),
    'OBSERVED'
)
ON CONFLICT (delivery_id, file_id)
DO NOTHING;
```

A repeated discovery should not create another logical delivery member.

---

## 15. Duplicate Logical Files

Physical file identity and logical delivery identity are different.

Example:

```
payments_20260926_001.csv
payments_20260926_001_copy.csv
```

Both might contain the same logical sequence.

The completeness system must determine whether the second object is:

- a duplicate;
- a replacement;
- a replay;
- a correction;
- an unexpected file.

Do not count every physical object as a unique delivery member.

---

## 16. Content Hashes

A content hash can help identify identical payloads.

Example:

```
SHA-256(file_bytes)
```

If two objects have the same expected logical identity and same content hash, they may be duplicate observations of the same payload.

If the logical identity is the same but hashes differ, investigate whether the producer replaced the file.

Hashing is evidence for identity/reconciliation; it is not by itself a business completeness rule.

---

## 17. File Count Completeness

Count-based validation is valid when the producer contract guarantees a fixed count.

Example:

```
expected_count = 24
observed_count = 24
```

Then count completeness can be true.

But also verify duplicates and identities when possible.

A safer rule is:

```
expected_unique_members = observed_unique_expected_members
AND
unexpected_members = 0
```

---

## 18. Dynamic File Counts

Some sources do not know the file count before delivery starts.

For example:

```
one file per customer region
one file per partition
one file per transaction shard
```

The pipeline needs another completeness signal.

Possible approaches:

- producer manifest;
- completion marker;
- source API;
- known partition metadata;
- control total;
- explicit final sequence.

Do not use a fixed count if the source does not guarantee one.

---

## 19. Final Sequence Number

Some sources send a final sequence.

Example:

```
001
002
003
004
DONE: 004
```

The final sequence tells the consumer:

```
expected = 1..4
```

Then the consumer compares the observed set against the expected range.

If sequence 003 is missing:

```
OBSERVED = {1,2,4}
EXPECTED = {1,2,3,4}
MISSING = {3}
```

---

## 20. Completeness After a Quiet Period

For sources without manifests or completion markers, some pipelines use a quiet period.

Example:

```
observe files
    ↓
wait 10 minutes
    ↓
observe again
    ↓
no new files
    ↓
evaluate completeness
```

This can reduce premature processing but does not prove completeness.

A quiet period is an operational heuristic, not a source-authoritative guarantee.

Document it explicitly if used.

---

## 21. Stability Versus Completeness

These concepts are related but different.

### Stability

The current files are no longer changing.

### Completeness

All expected delivery members are present.

Example:

```
001.csv = stable
002.csv = stable

003.csv = never arrived
```

The set is stable but incomplete.

Therefore:

```
STABLE ≠ COMPLETE
```

E53 will handle file validation after the delivery is considered ready.

---

## 22. Completeness Evaluation

A pure Python implementation:

```
from dataclasses import dataclass


@dataclass(frozen=True)
class CompletenessResult:
    complete: bool
    missing: frozenset[str]
    unexpected: frozenset[str]


def evaluate_completeness(
    expected: set[str],
    observed: set[str],
) -> CompletenessResult:
    missing = expected - observed
    unexpected = observed - expected

    return CompletenessResult(
        complete=not missing and not unexpected,
        missing=frozenset(missing),
        unexpected=frozenset(unexpected),
    )
```

Keeping this logic pure makes boundary testing straightforward.

---

## 23. Example Evaluation

```
expected = {"001", "002", "003"}
observed = {"001", "002"}

result = evaluate_completeness(expected, observed)

assert not result.complete
assert result.missing == {"003"}
assert not result.unexpected
```

Another example:

```
expected = {"001", "002", "003"}
observed = {"001", "002", "003"}

assert evaluate_completeness(
    expected,
    observed,
).complete
```

---

## 24. SQL Completeness Query

For a sequence-based delivery:

```
SELECT
    delivery_id,
    COUNT(*) FILTER (WHERE expected = TRUE) AS expected_count,
    COUNT(*) FILTER (
        WHERE expected = TRUE
        AND status = 'OBSERVED'
    ) AS observed_expected_count,
    COUNT(*) FILTER (
        WHERE expected = FALSE
    ) AS unexpected_count
FROM delivery_file
WHERE delivery_id = :delivery_id
GROUP BY delivery_id;
```

The application can use these values to determine the state.

---

## 25. Detect Missing Members

A normalized expected-members table makes missing detection explicit.

```
SELECT
    expected.logical_sequence
FROM expected_delivery_file expected
LEFT JOIN delivery_file observed
    ON observed.delivery_id = expected.delivery_id
   AND observed.logical_sequence = expected.logical_sequence
   AND observed.status = 'OBSERVED'
WHERE expected.delivery_id = :delivery_id
  AND observed.file_id IS NULL;
```

The returned rows are missing delivery members.

This is stronger than comparing aggregate counts.

---

## 26. Detect Unexpected Members

The inverse query identifies observed members that were not expected.

```
SELECT
    observed.file_id,
    observed.logical_sequence
FROM delivery_file observed
LEFT JOIN expected_delivery_file expected
    ON expected.delivery_id = observed.delivery_id
   AND expected.logical_sequence = observed.logical_sequence
WHERE observed.delivery_id = :delivery_id
  AND expected.logical_sequence IS NULL;
```

Unexpected members should be investigated rather than silently included.

---

## 27. Completeness Decision

A practical decision rule:

```
COMPLETE IF:

all expected members observed
AND
no required member missing
AND
no blocking unexpected members
AND
producer completion signal satisfied when required
```

This allows the contract to define which conditions are mandatory.

---

## 28. Completeness and Late Files

Suppose:

```
02:00 expected
02:30 deadline
02:35 first two files
02:50 final file
```

The delivery is:

```
ARRIVED_LATE
+
COMPLETE
```

Do not change the arrival SLA result merely because completeness eventually becomes true.

Both dimensions matter.

---

## 29. Completeness and Missing Files

Suppose:

```
expected = {001,002,003}
observed = {001,003}
deadline passed
```

State:

```
LATE
+
INCOMPLETE
```

This tells operations two things:

1. the delivery missed its expected timing;
2. sequence 002 is still missing.

That is much more useful than a single FAILED status.

---

## 30. Completeness and Replacement Files

A producer may replace a file.

Example:

```
001.csv
```

later becomes:

```
001.csv
hash = new_hash
```

The pipeline must know whether replacement is permitted.

Possible policy:

```
EXPECTED
   ↓
OBSERVED
   ↓
REPLACED
   ↓
REVALIDATE
```

Never silently replace already-ingested data unless the source contract explicitly supports correction.

---

## 31. Completeness Check Timing

Do not necessarily evaluate completeness once.

Possible strategy:

```
ARRIVAL DETECTED
      ↓
FIRST COMPLETENESS CHECK
      ↓
WAIT / RETRY
      ↓
SECOND CHECK
      ↓
COMPLETE or INCOMPLETE
```

This is useful when files are expected to arrive over a delivery window.

The number and spacing of retries should be part of the pipeline policy.

---

## 32. Retry Policy

A retryable incomplete delivery should not create unlimited checks.

Example:

```
attempt 1 → incomplete
wait 5 min
attempt 2 → incomplete
wait 10 min
attempt 3 → incomplete
escalate
```

Store the evaluation history when operational diagnosis matters.

---

## 33. Completeness Evaluation History

For important pipelines, keep an audit trail.

```
CREATE TABLE completeness_check (
    check_id UUID PRIMARY KEY,
    delivery_id UUID NOT NULL,
    checked_at TIMESTAMPTZ NOT NULL,
    expected_count INTEGER,
    observed_count INTEGER,
    missing_count INTEGER,
    unexpected_count INTEGER,
    status TEXT NOT NULL
);
```

This allows operators to answer:

> When did we first know that the delivery was incomplete?

---

## 34. Testing

### Unit tests

Test:

- empty expected set;
- empty observed set;
- exact match;
- missing one member;
- missing multiple members;
- unexpected member;
- missing and unexpected together;
- duplicate physical observations;
- fixed-count deliveries;
- sequence gaps;
- final sequence handling.

Example:

```
def test_missing_sequence_is_detected():
    result = evaluate_completeness(
        {"001", "002", "003"},
        {"001", "003"},
    )

    assert not result.complete
    assert result.missing == {"002"}
```

### Integration tests

Test:

- delivery registration;
- expected-member registration;
- observed-member registration;
- duplicate registration;
- completeness query;
- state transition;
- concurrent observations;
- late-arrival interaction.

---

## 35. Intentional Failure Drills

### Drill 1 — Remove one expected file

Verify the delivery becomes INCOMPLETE.

### Drill 2 — Add an unexpected file

Verify the system identifies the unexpected member.

### Drill 3 — Duplicate one file

Verify the duplicate does not falsely increase logical completeness.

### Drill 4 — Send files out of order

Verify ordering does not affect set-based completeness.

### Drill 5 — Send final sequence with a gap

Verify the missing sequence is identified.

### Drill 6 — Send a valid completion marker with a missing file

Verify the configured contract determines whether the delivery can become complete.

### Drill 7 — Replace a file

Verify replacement policy is applied and the payload is revalidated when required.

### Drill 8 — Evaluate during an active delivery window

Verify the system does not prematurely escalate a delivery that is still within its allowed arrival window.

---

## 36. Observability

Track:

```
file_completeness_check_total
file_completeness_complete_total
file_completeness_incomplete_total
file_completeness_unexpected_total
file_completeness_missing_total
file_completeness_check_latency_seconds
file_completeness_retry_total
```

Useful structured fields:

```
delivery_id
source_system
dataset
business_date
expected_count
observed_count
missing_count
unexpected_count
status
checked_at
```

Useful dashboards show:

- incomplete deliveries;
- missing sequences;
- unexpected files;
- time waiting for completeness;
- source-specific completeness failures;
- repeated completeness retries.

---

## 37. Alerts

Alert on meaningful states.

### Warning

Delivery is still incomplete but remains inside its allowed completion window.

### Critical

Required delivery remains incomplete after the deadline.

### Recovery

Previously incomplete delivery becomes complete.

Include:

```
source
dataset
business_date
delivery_id
missing_members
unexpected_members
deadline
status
```

Do not create an incident merely because a delivery is temporarily incomplete during its normal arrival window.

---

## 38. Recovery Runbook

### Delivery is incomplete

1. Identify the delivery.
2. Retrieve the expected member set.
3. Retrieve the observed member set.
4. Calculate missing members.
5. Check source-side delivery status.
6. Check whether missing files arrived under unexpected identities.
7. Check transport failures.
8. Check producer logs when available.
9. Wait or retry if still inside the allowed window.
10. Escalate after the defined deadline.

### Unexpected file exists

1. Preserve the physical file.
2. Determine its logical identity.
3. Compare it with the source contract.
4. Check whether it is a replay or replacement.
5. Do not silently count it toward completeness unless policy allows it.

### Delivery remains incomplete after deadline

1. Mark the delivery incomplete/late according to the state model.
2. Record the missing members.
3. Alert the responsible team.
4. Follow late-data policy.
5. Do not release downstream processing unless the pipeline explicitly supports partial delivery.

### A missing file arrives later

1. Register the new observation idempotently.
2. Re-run completeness evaluation.
3. Preserve the original incomplete/late history.
4. Transition to COMPLETE only when all contract conditions are satisfied.
5. Continue downstream readiness checks.

---

## 39. Production Tools You Should Know

### 1. Airflow

Learn sensors, task dependencies, retries, timeouts, and orchestration around delivery completeness.

### 2. Object-storage event systems

Learn object-created events, event ordering, duplicate notifications, and event-driven delivery assembly.

### 3. Prometheus/Grafana

Learn how to monitor incomplete deliveries, missing members, retry counts, and time-to-complete.

The tools provide orchestration and monitoring. The underlying completeness mechanism remains:

```
EXPECTED SET
     ↓
OBSERVED SET
     ↓
SET DIFFERENCE
     ↓
MISSING / UNEXPECTED
     ↓
COMPLETE or INCOMPLETE
```

---

## 40. Common Mistakes

1. Treating arrival as proof of completeness.
2. Comparing only file counts.
3. Ignoring logical file identity.
4. Assuming the highest sequence means all lower sequences arrived.
5. Ignoring duplicate files.
6. Treating unexpected files as valid expected members.
7. Using a quiet period as absolute proof of completeness.
8. Ignoring producer manifests.
9. Ignoring completion markers.
10. Mixing file completeness with content validation.
11. Mixing file completeness with business-total reconciliation.
12. Losing late-arrival information when completeness becomes true.
13. Evaluating too early in the delivery window.
14. Retrying indefinitely.
15. Having no audit history for completeness checks.
16. Silently replacing previously observed files.
17. Releasing partial data without an explicit business policy.
18. Using aggregate counts when a member-level comparison is possible.

---

## 41. Definition of Done

You can independently:

- define a file-completeness contract;
- distinguish arrival from completeness;
- represent expected delivery members;
- represent observed delivery members;
- compare expected and observed sets;
- identify missing members;
- identify unexpected members;
- detect sequence gaps;
- use manifests when appropriate;
- use completion markers safely;
- reconcile control totals separately;
- handle duplicate observations;
- handle replacement files;
- evaluate completeness repeatedly;
- preserve evaluation history;
- test incomplete and complete deliveries;
- monitor completeness health;
- alert on meaningful completeness failures;
- recover incomplete deliveries safely.

---

## 42. What You Learned

> **A delivered file is not necessarily a complete delivery.**

The production pattern is:

```
DEFINE COMPLETENESS CONTRACT
        ↓
CREATE EXPECTED MEMBER SET
        ↓
OBSERVE DELIVERY MEMBERS
        ↓
DEDUPLICATE
        ↓
COMPARE EXPECTED VS OBSERVED
        ↓
IDENTIFY MISSING / UNEXPECTED
        ↓
COMPLETE OR INCOMPLETE
        ↓
HAND OFF TO FILE VALIDATION
```

The key question is:

> **Can I prove that every required member of the delivery arrived, while also identifying anything missing or unexpected?**

That is the foundation for safely moving from file arrival into ETL validation.
