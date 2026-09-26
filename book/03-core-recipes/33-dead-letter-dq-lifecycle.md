# Recipe 33 — Dead-Letter / Data-Quality Lifecycle

A production pipeline must answer more than:

> Did this record fail?

It must answer:

> What failed, why did it fail, where is the failed data now, who or what can fix it, how can it be safely replayed, and how do we prove the corrected data reached the destination?

This recipe builds the complete **Dead-Letter / Data-Quality lifecycle**.

The lifecycle is:

    Detect
       |
       v
    Classify
       |
       v
    Capture
       |
       v
    Quarantine / Dead Letter
       |
       v
    Diagnose
       |
       v
    Decide
      / \
   Reject  Repair
      |      |
      |      v
      |    Validate
      |      |
      |      v
      |    Replay
      |      |
      +------+
         |
         v
      Reconcile
         |
         v
       Close

The important principle is:

> A failed record is not handled when it is moved to a dead-letter queue. It is handled when its lifecycle reaches a verified final state.

---

## 1. Goal

By the end of this recipe, you should be able to:

- recognize a data-quality failure in a real pipeline
- distinguish transient failures from data-quality failures
- distinguish malformed data from valid-but-unexpected data
- decide when to retry, quarantine, reject, or dead-letter
- capture the failed record without losing diagnostic context
- preserve the original payload safely
- record the exact validation failure
- classify failures consistently
- build a dead-letter/quarantine store
- track failure lifecycle state
- diagnose root cause
- repair eligible records
- validate repaired records
- replay records safely
- prevent duplicate processing during replay
- reconcile replayed data
- close or permanently reject records
- measure the dead-letter backlog
- alert on failure patterns
- intentionally create failures
- recover from them
- prove that recovery worked

---

## 2. What Is a Dead-Letter Record?

A dead-letter record is data that cannot continue through the normal processing path and therefore requires a separate handling path.

Example:

    Source Event
         |
         v
    Parse
         |
         v
    Validate
       /   \
    valid invalid
      |      |
      v      v
    Process Dead Letter
              |
              v
          Diagnose
              |
              v
            Repair
              |
              v
            Replay

A dead-letter record should not simply disappear.

It should retain enough information to answer:

- what record failed?
- where did it fail?
- when did it fail?
- what rule failed?
- what pipeline version processed it?
- what source produced it?
- can it be repaired?
- has it already been replayed?
- what happened during replay?

---

## 3. Dead Letter vs Retry

Do not put every failure into a dead-letter queue.

### Retry

Use retry when the problem is likely temporary.

Examples:

- network timeout
- temporary database connection failure
- HTTP 503
- transient dependency failure
- temporary rate limit

Conceptually:

    process
       |
       v
    transient failure
       |
       v
      retry
       |
       v
    process again

### Dead Letter

Use dead letter or quarantine when the data itself cannot currently pass processing.

Examples:

- malformed JSON
- missing required identifier
- invalid timestamp
- unsupported enum
- schema violation
- invalid business relationship
- impossible value

Conceptually:

    process
       |
       v
    data-quality failure
       |
       v
    dead letter
       |
       v
    investigate / repair / reject

The distinction is critical.

Retrying permanently invalid data can create retry storms.

Dead-lettering a temporary infrastructure failure can hide an operational outage.

---

## 4. Dead Letter vs Quarantine

These terms are often used interchangeably, but a useful distinction is:

### Dead Letter

The normal processing path has exhausted its allowed handling and the item requires separate processing.

### Quarantine

The item is deliberately isolated because it is unsafe or invalid to continue processing.

A system may implement both using one physical table or queue.

What matters is the lifecycle semantics, not the name.

---

## 5. Data Quality Failure Classes

A practical classification can include:

    PARSE_ERROR
    SCHEMA_ERROR
    REQUIRED_FIELD_MISSING
    TYPE_ERROR
    FORMAT_ERROR
    DOMAIN_ERROR
    REFERENTIAL_INTEGRITY_ERROR
    DUPLICATE_ERROR
    BUSINESS_RULE_ERROR
    SECURITY_POLICY_ERROR
    UNKNOWN_DATA_ERROR

Infrastructure failures should normally be classified separately:

    NETWORK_ERROR
    DATABASE_ERROR
    RATE_LIMIT
    DEPENDENCY_UNAVAILABLE
    TIMEOUT

This allows the pipeline to make different decisions.

---

## 6. The Failure Decision Tree

When processing fails, ask:

    Did the dependency fail?
          |
        yes ---> retry
          |
         no
          |
          v
    Is the input malformed?
          |
        yes ---> quarantine
          |
         no
          |
          v
    Is the value invalid?
          |
        yes ---> quarantine
          |
         no
          |
          v
    Is the business rule violated?
          |
        yes ---> quarantine / reject
          |
         no
          |
          v
    Is the failure unknown?
          |
        yes ---> quarantine + alert

Do not encode this as an arbitrary list of exceptions.

Define the failure policy explicitly.

---

## 7. Preserve the Original Evidence

When a record enters the dead-letter path, preserve the original evidence.

Useful fields:

    event_id
    source
    source_location
    received_at
    raw_payload
    content_type
    schema_version
    pipeline_version
    processing_attempt
    failure_stage
    failure_code
    failure_message
    failure_details
    quarantined_at

Do not store only:

    "invalid record"

That destroys the information needed for diagnosis.

---

## 8. Protect Sensitive Data

Dead-letter storage can accidentally become a second uncontrolled copy of sensitive data.

Apply the same or stronger security controls as the main pipeline.

Consider:

- encryption at rest
- encryption in transit
- access control
- retention limits
- audit logging
- masking
- field-level redaction
- restricted replay permissions

Never assume that because data failed processing it is safe to expose.

---

## 9. Preserve Raw Data Separately When Appropriate

For important pipelines, consider separating:

    raw evidence
        |
        v
    failure metadata

Instead of putting an unlimited payload directly into a relational row.

Possible architecture:

    Object Storage
         |
      raw payload
         |
         v
    Dead-letter metadata
         |
         v
    PostgreSQL

The correct design depends on payload size, retention, security, and replay requirements.

Do not introduce object storage simply because it is fashionable. Use it when the operational requirements justify it.

---

## 10. Dead-Letter Record Model

A practical relational model could look like:

~~~sql
CREATE TABLE dead_letter_records (
    dead_letter_id BIGSERIAL PRIMARY KEY,
    event_id TEXT NOT NULL,
    source TEXT NOT NULL,
    failure_stage TEXT NOT NULL,
    failure_code TEXT NOT NULL,
    failure_message TEXT,
    raw_payload JSONB,
    pipeline_version TEXT,
    attempt_count INTEGER NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'QUARANTINED',
    first_failed_at TIMESTAMPTZ NOT NULL,
    last_failed_at TIMESTAMPTZ NOT NULL,
    resolved_at TIMESTAMPTZ,
    replayed_at TIMESTAMPTZ
);
~~~

This is only an example.

The real schema should reflect:

- retention
- payload size
- privacy
- replay requirements
- operational ownership
- source semantics

---

## 11. Dead-Letter Lifecycle States

Define explicit states.

A practical model:

    DETECTED
       |
       v
    QUARANTINED
       |
       v
    INVESTIGATING
      /       \
     v         v
  REPAIRABLE  REJECTED
     |
     v
   REPAIRED
     |
     v
  REPLAY_PENDING
     |
     v
   REPLAYING
    /     \
   v       v
SUCCESS   FAILED
   |        |
   v        v
 RESOLVED  QUARANTINED

The exact state machine can be simpler or more detailed.

The important requirement is that the lifecycle is explicit.

---

## 12. Never Use an Ambiguous Status

Avoid:

    status = "failed"

That does not tell you:

- failed where?
- failed why?
- can it be repaired?
- was it replayed?
- is someone investigating it?
- is the failure permanent?

Prefer structured state:

    lifecycle_status
    failure_code
    failure_stage
    attempt_count

Status represents lifecycle.

Failure code represents cause.

These are different concepts.

---

## 13. Capture the Failure at the Boundary

The best place to capture a data-quality failure is as close as possible to where it is detected.

Example:

    parse()
       |
       +-- ParseError
              |
              v
          quarantine()

Or:

    validate()
       |
       +-- ValidationError
              |
              v
          quarantine()

Do not allow invalid records to travel through several transformations before recording the original failure context.

---

## 14. Make Failure Codes Stable

Failure messages can change.

For example:

    "amount cannot be negative"

might become:

    "payment amount must be >= 0"

Do not use the human message as the machine classification.

Use:

    failure_code = NEGATIVE_AMOUNT

and separately store:

    failure_message = "amount cannot be negative"

This enables reliable:

- dashboards
- grouping
- alerts
- trend analysis
- automated remediation

---

## 15. Data Quality Rules

A data-quality layer should define explicit rules.

Examples:

### Completeness

    event_id IS NOT NULL

### Type

    amount can be parsed as numeric

### Range

    amount >= 0

### Domain

    currency IN ('EUR', 'USD', 'GBP')

### Format

    timestamp follows the expected format

### Uniqueness

    event_id is unique

### Referential Integrity

    account_id exists in the account dimension

### Business Rule

    settled transaction must have settlement timestamp

Each rule should produce a stable failure code.

---

## 16. One Record Can Have Multiple Failures

Suppose:

    event_id = NULL
    currency = "XYZ"
    amount = -50

The record has multiple violations.

Decide whether the system:

1. records the first failure only
2. records all failures
3. records a primary failure plus detailed violations

For production data-quality systems, capturing all relevant validation violations is often more useful.

Example:

~~~json
{
  "failure_code": "DATA_QUALITY_FAILED",
  "violations": [
    "REQUIRED_FIELD_MISSING",
    "INVALID_CURRENCY",
    "NEGATIVE_AMOUNT"
  ]
}
~~~

The choice should match downstream operational needs.

---

## 17. Data Quality Result Model

Instead of only returning TRUE/FALSE, return structured results.

Example:

~~~json
{
  "valid": false,
  "violations": [
    {
      "code": "NEGATIVE_AMOUNT",
      "field": "amount",
      "message": "amount must be >= 0"
    }
  ]
}
~~~

This makes the validation layer reusable.

Processing can then decide:

    valid
       -> continue

    invalid
       -> dead letter

---

## 18. Row-Level vs Batch-Level Quality

Not every quality problem belongs to one record.

### Row-level

    missing event_id
    invalid currency
    negative amount

The individual record can be quarantined.

### Batch-level

    expected file has 100 partitions but only 99 arrived
    total amount differs from control total
    schema version changed unexpectedly

The entire batch may need to be blocked.

Do not dead-letter 1,000,000 records individually when the actual problem is one broken source batch.

---

## 19. Data Quality Gates

A pipeline can use gates:

    Ingest
      |
      v
    Validate
      |
    +---+---+
    |       |
   PASS   FAIL
    |       |
    v       v
 Process  Quarantine
    |
    v
 Reconcile
    |
    v
 Publish

Batch-level quality can become a hard gate.

Record-level failures can often be isolated.

The policy should be explicit.

---

## 20. Prevent Poison-Pill Loops

A poison pill is a record that always fails processing.

Bad architecture:

    receive
      |
      v
    fail
      |
      v
    retry
      |
      v
    fail
      |
      v
    retry
      |
      v
    infinite loop

Correct architecture:

    receive
      |
      v
    classify
      |
      v
    permanent data failure
      |
      v
    dead letter

Track:

    attempt_count

and enforce:

    max_attempts

for retryable failures.

---

## 21. Dead-Letter Backlog

A dead-letter queue is operational debt.

Track:

    total quarantined
    currently unresolved
    resolved
    permanently rejected
    oldest unresolved age
    failures by code
    failures by source
    failures by pipeline stage

Useful metrics:

    dead_letter_records_total
    dead_letter_open_total
    dead_letter_resolved_total
    dead_letter_rejected_total
    dead_letter_oldest_age_seconds
    dead_letter_replay_total

A queue that silently grows is a pipeline problem even if the main pipeline remains green.

---

## 22. Alert on Patterns, Not Just Individual Records

One bad record may be normal.

10,000 records failing with:

    INVALID_CURRENCY

is an incident.

Alert on:

- rate
- volume
- percentage of input
- oldest unresolved record
- repeated failure code
- repeated source
- repeated pipeline stage

Example:

    INVALID_CURRENCY
    12,000 failures
    18% of today's input

This is more actionable than 12,000 individual alerts.

---

## 23. Diagnose Before Repair

When a dead-letter backlog appears:

    1. Group by failure code
    2. Group by source
    3. Group by pipeline version
    4. Group by failure stage
    5. Inspect representative payloads
    6. Determine whether failures share one root cause
    7. Determine whether the source or pipeline is responsible

Do not manually repair records one by one before understanding the failure pattern.

One root cause may explain thousands of failures.

---

## 24. Root Cause Categories

Classify the cause.

### Source defect

The producer generated invalid data.

### Contract drift

Producer and consumer disagree about the schema or semantics.

### Pipeline defect

The consumer incorrectly rejects valid data.

### Reference-data problem

A valid record cannot be processed because required reference data is missing or stale.

### Configuration defect

A valid value is rejected because the pipeline configuration is wrong.

### Operational dependency failure

A required service or database is unavailable.

Root-cause classification should be separate from the original failure code.

---

## 25. Repairability

Every dead-letter record should eventually have a disposition.

Possible outcomes:

    REPLAYABLE
    REPAIRED
    PERMANENTLY_REJECTED
    DUPLICATE
    SOURCE_CORRECTION_REQUIRED
    WAITING_FOR_DEPENDENCY
    UNKNOWN

Do not leave records permanently in:

    FAILED

with no owner or next action.

---

## 26. Repair the Data, Not the Evidence

Never overwrite the original failed payload without preserving what originally arrived.

Prefer:

    original payload
          |
          v
    repair operation
          |
          v
    repaired payload
          |
          v
    validation
          |
          v
    replay

Record:

    original
    repaired
    who/what repaired it
    when
    repair reason
    repair version

This creates an audit trail.

---

## 27. Automated vs Manual Repair

Automate deterministic repairs.

Example:

    currency = "EURO"
        ->
    currency = "EUR"

if that mapping is explicitly defined.

Do not automate ambiguous corrections.

Example:

    amount = 100

but there are multiple possible interpretations of the currency.

That should require a controlled decision rather than guessing.

---

## 28. Repair Must Be Idempotent

If a repair job runs twice, it should not corrupt the record.

Example:

    "EURO"
        ->
    "EUR"

Running the same repair again should produce:

    "EUR"

not:

    "EURR"

Use:

- deterministic transformations
- versioned repair rules
- unique repair identifiers
- transactional updates
- immutable original evidence

---

## 29. Validate After Repair

Never replay a repaired record without validating it again.

Flow:

    original invalid
         |
         v
       repair
         |
         v
      validate
       /    \
    valid   invalid
      |        |
      v        v
   replay    remain
            quarantined

A successful repair operation does not prove the repaired record is valid.

---

## 30. Replay Must Be Explicit

Do not automatically replay every dead-letter record after repair.

A replay operation should define:

    dead_letter_id
    replay_reason
    replay_scope
    replay_version
    requested_by
    requested_at

For batch replay:

    batch_id
    failure_code
    time window
    source
    repair version

Bound the replay scope.

---

## 31. Replay Must Be Idempotent

Suppose:

    event_id = E123

is replayed twice.

The target should not contain two logical copies of E123.

Use the idempotency and deduplication mechanisms from earlier recipes.

Replay should be safe even when:

- the operator retries
- the worker crashes
- acknowledgement is lost
- the replay command is submitted twice

---

## 32. Replay State Machine

A useful replay state machine:

    REPLAY_PENDING
         |
         v
      REPLAYING
       /     \
      v       v
   SUCCESS   FAILED
      |        |
      v        v
   RESOLVED  QUARANTINED

Record:

    replay_attempts
    replay_started_at
    replay_completed_at
    replay_error
    target_reference

This makes replay auditable.

---

## 33. Do Not Delete Dead-Letter Records After Replay

A successful replay does not mean the evidence should immediately disappear.

Keep the lifecycle record according to the retention policy.

Example:

    original failure
         |
         v
    repair history
         |
         v
    replay result
         |
         v
    reconciliation result
         |
         v
    resolved

This provides an audit trail and helps investigate recurring failures.

---

## 34. Reconcile After Replay

After replay:

    source
       |
       v
    target

must be reconciled.

For example:

    before:
    missing = 100

    replay:
    100 records

    after:
    missing = 0

But also verify:

    extra = 0
    duplicates = 0
    changed = 0

Replay success is not the final proof.

Reconciliation is.

---

## 35. Dependency-Blocked Records

Sometimes a record is valid but cannot be processed yet.

Example:

    transaction references account X
    account X has not arrived in the reference dataset

Do not permanently reject the transaction.

Possible lifecycle:

    WAITING_FOR_DEPENDENCY
          |
          v
    dependency arrives
          |
          v
       validate
          |
          v
        replay

This is different from malformed data.

---

## 36. Data Quality Metrics

Useful metrics include:

    dq_records_checked_total
    dq_records_failed_total
    dq_failure_rate
    dq_failures_by_rule
    dq_failures_by_source
    dq_failures_by_stage
    dead_letter_open_total
    dead_letter_replay_total
    dead_letter_replay_success_total
    dead_letter_replay_failure_total
    dead_letter_oldest_age_seconds

Track both counts and rates.

A failure count of 100 means something different when input volume is:

    100,000

versus:

    100,000,000

---

## 37. Data Quality SLOs

Define operational expectations such as:

    invalid-record rate < threshold
    dead-letter backlog = 0 for critical feeds
    oldest unresolved record < threshold
    replay success rate > threshold
    batch-level quality failures = 0

Do not create thresholds without understanding the dataset's normal behavior and business risk.

---

## 38. Test — Valid Record

Input:

~~~json
{
  "event_id": "E1",
  "amount": 100,
  "currency": "EUR"
}
~~~

Expected:

    validation = PASS
    processing = continues
    dead_letter = 0

---

## 39. Test — Missing Required Field

Input:

~~~json
{
  "amount": 100,
  "currency": "EUR"
}
~~~

Expected:

    failure_code = REQUIRED_FIELD_MISSING
    lifecycle_status = QUARANTINED

The original payload must remain recoverable.

---

## 40. Test — Invalid Domain Value

Input:

~~~json
{
  "event_id": "E2",
  "amount": 100,
  "currency": "XYZ"
}
~~~

Expected:

    failure_code = INVALID_CURRENCY
    record = quarantined

---

## 41. Test — Transient Dependency Failure

Simulate:

    database unavailable

Expected:

    retry

Not:

    permanent dead letter

After the dependency recovers, the record should continue normally.

---

## 42. Test — Poison Pill

Create a record that always fails validation.

Expected:

    bounded attempts
    no infinite retry loop
    eventual quarantine

Verify that worker throughput remains healthy.

---

## 43. Test — Repair and Replay

Start with:

    currency = "EURO"

Repair to:

    currency = "EUR"

Then:

    validate
    replay
    reconcile

Expected:

    validation = PASS
    replay = SUCCESS
    target contains one logical record
    reconciliation = PASS

---

## 44. Test — Replay Twice

Replay the same event twice.

Expected:

    one logical target record
    no duplicate
    replay history shows both attempts if the system records attempts

This verifies replay idempotency.

---

## 45. Test — Repair Fails

Apply an invalid repair.

Expected:

    validation fails again
    record remains quarantined
    original evidence remains unchanged
    replay does not occur

---

## 46. Test — Dependency-Blocked Record

Create a valid transaction referencing unavailable reference data.

Expected:

    WAITING_FOR_DEPENDENCY

After reference data arrives:

    validate
    replay
    reconcile

Expected:

    RESOLVED

---

## 47. Test — Batch-Level Failure

Create a batch with an invalid schema or broken control total.

Expected:

    batch gate fails
    downstream publication is blocked
    individual records are not blindly dead-lettered
    incident contains batch-level evidence

This test teaches the difference between row-level and batch-level quality failures.

---

## 48. Intentionally Break the Pipeline

Perform these drills in a safe environment.

### Drill 1 — Remove a required field

Expected:

    REQUIRED_FIELD_MISSING

### Drill 2 — Send an unsupported enum

Expected:

    DOMAIN_ERROR / INVALID_* failure

### Drill 3 — Send malformed JSON

Expected:

    PARSE_ERROR

### Drill 4 — Break a reference-data lookup

Expected:

    dependency-aware handling

### Drill 5 — Force a permanent validation failure

Expected:

    bounded retries
    quarantine

### Drill 6 — Repair the wrong field

Expected:

    validation remains failed

### Drill 7 — Replay the same event twice

Expected:

    no duplicate business record

### Drill 8 — Corrupt a batch control total

Expected:

    batch-level gate fails

### Drill 9 — Create a large failure spike

Expected:

    metrics and alerts identify the pattern

---

## 49. Incident Investigation Exercise

Suppose:

    input records = 100,000
    valid records = 91,000
    dead-letter records = 9,000

Do not start repairing 9,000 records individually.

First group them.

Example:

    INVALID_CURRENCY       8,500
    MISSING_EVENT_ID         400
    INVALID_TIMESTAMP        100

Now investigate the dominant failure.

Suppose all 8,500 INVALID_CURRENCY records came from one source version.

That suggests a shared root cause.

The correct response may be:

    identify source contract change
         |
         v
    correct producer/configuration
         |
         v
    repair affected records if safe
         |
         v
    validate
         |
         v
    bounded replay
         |
         v
    reconcile

The important skill is recognizing when thousands of bad records represent one systemic defect.

---

## 50. Dead-Letter Ownership

Every unresolved class should have an owner.

Record:

    owning_team
    escalation_policy
    next_action
    due_at

A dead-letter queue without ownership becomes a permanent data graveyard.

---

## 51. Retention and Cleanup

Dead-letter data should have a retention policy.

Define:

    retention period
    legal requirements
    privacy requirements
    audit requirements
    replay window
    deletion process

Do not delete records simply because they are old.

But do not retain sensitive failed payloads forever without a justified policy.

---

## 52. Security and Replay Permissions

Replay can modify production data.

Therefore replay should normally require stronger authorization than inspection.

Consider:

    read dead-letter records
    investigate
    repair
    approve replay
    execute replay
    purge

These can be separate permissions.

For high-risk datasets, record:

    who requested replay
    who approved it
    who executed it
    what scope was replayed
    what version performed the replay

---

## 53. Observability Dashboard

A useful dashboard contains:

### Volume

    records checked
    records failed
    failure rate

### Failure causes

    top failure codes
    failures by source
    failures by stage

### Backlog

    open dead letters
    oldest unresolved age

### Recovery

    repaired
    replay pending
    replay success
    replay failure

### Quality

    current DQ rate
    DQ rate by source
    DQ rate by pipeline version

The dashboard should help answer:

> Is this a normal isolated failure or a systemic pipeline/source problem?

---

## 54. Practical Implementation Sequence

For an existing pipeline:

    1. Inventory current validation rules
    2. Identify failure points
    3. Separate transient failures from data failures
    4. Define stable failure codes
    5. Define dead-letter/quarantine storage
    6. Preserve original evidence
    7. Add lifecycle status
    8. Add failure metadata
    9. Add validation result structure
    10. Add row-level quarantine handling
    11. Add batch-level quality gates
    12. Add backlog metrics
    13. Add alerts
    14. Define ownership
    15. Define repair policies
    16. Implement safe repair operations
    17. Validate repaired data
    18. Implement bounded replay
    19. Make replay idempotent
    20. Reconcile after replay
    21. Add retention and security controls
    22. Test every lifecycle state
    23. Break the pipeline intentionally
    24. Recover the failures
    25. Document the runbook

---

## 55. Example End-to-End Lifecycle

Input:

~~~json
{
  "event_id": "PAY-1001",
  "amount": 250,
  "currency": "EURO"
}
~~~

Validation:

    INVALID_CURRENCY

Capture:

    dead_letter_id = 5001
    event_id = PAY-1001
    status = QUARANTINED

Diagnosis:

    source uses "EURO"
    consumer contract expects "EUR"

Repair:

    "EURO" -> "EUR"

Validation:

    PASS

Replay:

    REPLAY_PENDING
        ->
    REPLAYING
        ->
    SUCCESS

Reconciliation:

    source IDs = target IDs
    amount = 250
    currency = EUR
    duplicate count = 0

Final:

    RESOLVED

The important point is that the lifecycle has evidence at every step.

---

## 56. Example Lifecycle Record

~~~text
dead_letter_id = 5001
event_id = PAY-1001

failure_code = INVALID_CURRENCY
failure_stage = validation

original_payload = preserved
repair_version = currency-map-v3

status:
    QUARANTINED
    REPAIRED
    REPLAY_PENDING
    REPLAYING
    RESOLVED

replay_attempts = 1
replayed_at = <timestamp>
resolved_at = <timestamp>
~~~

A real implementation may normalize this into multiple audit tables rather than storing a state history in one row.

---

## 57. Common Mistakes

### Mistake 1 — Retry everything

Permanent bad data creates retry storms.

### Mistake 2 — Dead-letter everything

Transient infrastructure problems become hidden operational failures.

### Mistake 3 — Store only the error message

The original evidence is lost.

### Mistake 4 — Overwrite the original payload during repair

You lose auditability.

### Mistake 5 — Manually fix thousands of records

A systemic defect remains unresolved.

### Mistake 6 — Replay without validation

Invalid repaired records return to the pipeline.

### Mistake 7 — Replay without idempotency

Duplicates are created.

### Mistake 8 — Delete successful dead-letter records immediately

The audit trail disappears.

### Mistake 9 — No ownership

The backlog becomes permanent.

### Mistake 10 — No reconciliation after replay

You cannot prove recovery.

### Mistake 11 — Mix row-level and batch-level failures

The system quarantines symptoms instead of stopping the broken batch.

### Mistake 12 — Treat every failure as a data-quality problem

Some failures are infrastructure or dependency failures.

---

## 58. Production Runbook

When dead-letter volume increases:

    1. Check current failure rate
    2. Compare against normal baseline
    3. Group by failure code
    4. Group by source
    5. Group by pipeline version
    6. Group by failure stage
    7. Inspect representative records
    8. Determine shared root cause
    9. Decide retry vs quarantine vs repair
    10. Stop unsafe downstream processing if required
    11. Fix the producer or pipeline defect
    12. Define the affected scope
    13. Repair only when deterministic
    14. Revalidate repaired data
    15. Replay in a bounded scope
    16. Reconcile the target
    17. Verify duplicate safety
    18. Monitor backlog reduction
    19. Record the incident
    20. Close only after the lifecycle reaches a verified terminal state

---

## 59. Definition of Done

- [ ] Data-quality rules are explicitly defined.
- [ ] Transient and permanent failure classes are distinguished.
- [ ] Stable failure codes exist.
- [ ] Original failed evidence is preserved.
- [ ] Sensitive dead-letter data is protected.
- [ ] Dead-letter/quarantine storage exists.
- [ ] Lifecycle states are explicit.
- [ ] Failure stage is recorded.
- [ ] Failure message and structured failure code are separated.
- [ ] Row-level failures are handled independently where appropriate.
- [ ] Batch-level failures can block the batch.
- [ ] Poison-pill loops are prevented.
- [ ] Retry attempts are bounded.
- [ ] Dead-letter backlog is observable.
- [ ] Failure rates are observable.
- [ ] Alerts identify meaningful failure patterns.
- [ ] Ownership is defined.
- [ ] Repairability is explicit.
- [ ] Original evidence is immutable.
- [ ] Repairs are deterministic where automated.
- [ ] Repairs are idempotent.
- [ ] Repaired records are revalidated.
- [ ] Replay scope is bounded.
- [ ] Replay is idempotent.
- [ ] Replay attempts are auditable.
- [ ] Successful replay does not immediately destroy evidence.
- [ ] Dependency-blocked records have a separate lifecycle.
- [ ] Reconciliation runs after replay.
- [ ] Retention is defined.
- [ ] Replay permissions are controlled.
- [ ] Failure scenarios are intentionally tested.
- [ ] Recovery has been performed successfully.
- [ ] Final state is independently verified.

---

## 60. What You Learned

The dead-letter path is not a trash can.

It is a controlled data lifecycle:

    Detect
       |
       v
    Classify
       |
       v
    Capture evidence
       |
       v
    Quarantine
       |
       v
    Diagnose
       |
       +----------------+
       |                |
       v                v
    Repair           Reject
       |
       v
    Validate
       |
       v
    Replay
       |
       v
    Reconcile
       |
       v
    Resolve

The most important lessons are:

1. Not every failure belongs in a dead-letter queue.
2. Transient infrastructure failures usually need retry handling.
3. Invalid data needs isolation rather than infinite retries.
4. Original evidence must be preserved.
5. Failure codes should be stable and machine-readable.
6. Lifecycle status and failure cause are different concepts.
7. Row-level and batch-level quality failures require different handling.
8. Poison pills must not create infinite retry loops.
9. Large failure spikes should be investigated by pattern, not record by record.
10. Repairs must be deterministic and auditable.
11. Repaired records must be validated before replay.
12. Replay must be idempotent.
13. Replay must be bounded.
14. Dead-letter records should remain auditable after successful replay.
15. Dependency-blocked data is not necessarily invalid data.
16. Reconciliation is the proof that replay actually restored the expected state.
17. A dead-letter backlog is an operational signal, not merely a storage location.

The real objective is not:

    move bad data somewhere else

It is:

    detect
    -> understand
    -> correct or reject
    -> safely replay
    -> reconcile
    -> close

When you can independently operate that lifecycle, you can handle one bad event, a thousand malformed records, or a systemic data-quality incident without turning the pipeline into an uncontrolled retry or manual-repair process.

---

# Recipe 34 Preview

The next recipe will focus on **Handle Schema Changes**.

We will cover:

- detecting source schema changes
- added columns
- removed columns
- renamed columns
- changed data types
- nullable vs non-nullable changes
- breaking vs non-breaking changes
- schema snapshots
- schema comparison
- schema validation
- backward compatibility
- forward compatibility
- safe migrations
- versioned schemas
- producer/consumer contracts
- intentional schema-change testing
- recovery from a breaking schema deployment

The distinction is:

    Dead-Letter / DQ Lifecycle
        |
        v
    What happens when data violates the current contract?

    Schema Changes
        |
        v
    What happens when the contract itself changes?

A production pipeline must be able to handle both.