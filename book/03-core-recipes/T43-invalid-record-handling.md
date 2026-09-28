# T43 — Invalid Record Handling

> **Goal:** Define how invalid records are classified, isolated, audited, replayed, and reconciled without silently dropping data.

A validation rule tells you that a record is invalid.

Production engineering still has to answer:

- What happens to the invalid record?
- Is it rejected, quarantined, corrected, or retried?
- Can the original record be recovered?
- Which rule failed?
- Was the failure permanent or temporary?
- How many valid records continued?
- How are quarantined records replayed?
- How do we prove that no records disappeared?

This recipe turns validation outcomes into an operational lifecycle.

## 1. Problem Recognition

A pipeline that detects invalid records but silently discards them is not a production-quality pipeline.

Consider a batch of 1,000,000 records:

    998,500 valid
      1,200 invalid
        300 malformed
        500 missing required fields
        200 invalid references
        200 temporary reference failures
    --------------------------------
    1,000,000 input

If the pipeline simply filters invalid rows:

    output = 998,500

the missing 1,500 records may never be explained.

### 1.1 Invalid does not mean disposable

An invalid record can represent:

- bad source data;
- a transient reference-data problem;
- a parsing defect;
- a business-rule violation;
- an upstream contract violation;
- a schema migration mismatch;
- an incomplete record;
- a duplicate;
- a record that requires manual review.

The record should normally remain recoverable.

### 1.2 Invalid versus rejected versus quarantined

These terms should have explicit meanings.

| Term | Meaning |
|---|---|
| Invalid | Failed one or more validation rules |
| Rejected | Deliberately excluded from the current target flow |
| Quarantined | Isolated while preserving evidence for diagnosis or replay |
| Corrected | Invalid data was repaired under an explicit policy |
| Retriable | Failure may succeed later without changing the source record |
| Permanent | Record cannot succeed without data or rule change |
| Accepted | Passed the applicable quality contract |

A record can be:

    invalid + quarantined + retriable

or:

    invalid + rejected + permanent

### 1.3 Define the record lifecycle

A useful lifecycle is:

    RECEIVED
       ↓
    VALIDATING
       ↓
    ACCEPTED ─────────→ PUBLISHED
       ↓
    INVALID
       ↓
    CLASSIFIED
       ↓
    QUARANTINED
       ↓
    RETRIABLE / PERMANENT / MANUAL_REVIEW
       ↓
    REPLAYED / CORRECTED / CLOSED
       ↓
    RECONCILED

The exact state machine must be explicit.

### 1.4 Preserve the original evidence

At minimum, preserve:

- source identifier;
- source batch;
- record identity if available;
- original payload or a recoverable reference;
- validation rule;
- rule version;
- failure category;
- failure details;
- first-seen timestamp;
- processing attempt;
- pipeline version.

Never preserve sensitive payload fields unnecessarily.

### 1.5 Define the target grain

Before handling invalid records, define whether one quarantine entry represents:

- one source record;
- one validation failure;
- one record plus multiple failures;
- one batch-level failure.

These are different models.

A common design is:

    one quarantine record per source record
    +
    many validation findings per quarantine record

### 1.6 Do not confuse row filtering with invalid-record handling

This:

    SELECT *
    FROM staging
    WHERE is_valid = true

is filtering.

Production invalid-record handling additionally needs:

- evidence;
- disposition;
- state;
- counts;
- replay;
- reconciliation.

### 1.7 Decide the failure boundary

Some failures should reject a record.

Others should stop the entire batch.

Examples:

    one invalid customer address
        → record-level quarantine

    corrupted input file
        → batch-level failure

    missing required schema column
        → pipeline-stage failure

The failure scope must be part of the contract.

## 2. Concept and Reasoning

### 2.1 Classify before disposition

A validation failure should first become a structured classification.

Example:

    rule = "required_customer_id"
    category = "MISSING_REQUIRED_FIELD"
    disposition = "QUARANTINE"
    retryable = false

Classification makes operations deterministic.

### 2.2 Validation result model

A useful result contains:

    record_id
    rule_name
    rule_version
    status
    failure_category
    severity
    retryable
    observed_value
    expected_value
    evidence

Keep raw sensitive values out of error messages whenever possible.

### 2.3 One record can have many failures

Example:

    customer_id = NULL
    country = "XX"
    amount = -5

The same record may fail:

- required-field validation;
- domain validation;
- range validation.

Do not arbitrarily retain only the last error.

### 2.4 Primary failure versus all failures

There are two useful representations.

Primary failure:

    one disposition-driving failure

All findings:

    every detected rule failure

Store both when operationally useful.

### 2.5 Failure precedence

If multiple failures exist, precedence may be needed.

Example:

    malformed payload
        >
    missing required field
        >
    invalid domain
        >
    statistical warning

The precedence should be policy-driven.

Do not use accidental validation order as business logic.

### 2.6 Permanent versus retriable

A retriable failure can succeed later.

Examples:

- reference table temporarily unavailable;
- network timeout;
- service unavailable;
- temporary database failure.

A permanent failure usually requires data or rule change.

Examples:

- impossible date;
- invalid domain value;
- missing mandatory identifier;
- structurally malformed payload.

Do not retry permanent failures indefinitely.

### 2.7 Batch-level versus record-level failure

Record-level failures allow valid records to continue.

Batch-level failures protect against systemic corruption.

Examples of batch-level conditions:

- missing required source column;
- corrupt file;
- wrong schema version;
- unexpected encoding;
- broken parser;
- database transaction failure.

### 2.8 Quarantine is not deletion

Quarantine means:

    remove from active processing
    +
    preserve recoverability

The quarantined record remains part of the input accounting.

### 2.9 Quarantine should be durable

Do not keep quarantine state only in application memory.

Use durable storage such as:

- relational quarantine tables;
- object storage for original artifacts;
- immutable audit records.

The storage design depends on payload size and sensitivity.

### 2.10 Preserve source identity

Every quarantined record should be traceable back to its source.

Useful identifiers:

    source_system
    source_file
    source_batch_id
    source_record_id
    event_id
    ingestion_id

If the source provides no identifier, derive a deterministic ingestion identity carefully.

### 2.11 Idempotent quarantine

A replayed invalid record must not create uncontrolled duplicates.

Use a stable identity such as:

    source_batch_id + source_record_id + validation_scope

or another domain-appropriate key.

### 2.12 Quarantine state machine

Example:

    OPEN
      ↓
    CLASSIFIED
      ↓
    RETRY_PENDING
      ↓
    RETRYING
      ↓
    ACCEPTED
      or
    QUARANTINED
      ↓
    MANUAL_REVIEW
      ↓
    CORRECTED
      ↓
    REPLAYED
      ↓
    CLOSED

Not every system needs every state.

The important requirement is that state transitions are explicit and auditable.

### 2.13 Correction policy

There are three broad approaches:

1. reject;
2. repair;
3. defer.

Automatic repair should only be used when the transformation is deterministic and approved.

Example:

    "  US  " → "US"

may be safe normalization.

Guessing a missing customer ID is not.

### 2.14 Do not silently mutate source data

If a record is corrected:

- preserve the original;
- record the correction;
- record the rule version;
- identify who or what made the correction;
- replay from the corrected representation.

### 2.15 Quarantine reason taxonomy

A stable taxonomy helps operations.

Example categories:

    SCHEMA_ERROR
    PARSE_ERROR
    REQUIRED_FIELD_MISSING
    TYPE_INVALID
    RANGE_INVALID
    DOMAIN_INVALID
    REFERENTIAL_FAILURE
    UNIQUENESS_FAILURE
    CONSISTENCY_FAILURE
    ANOMALY_REVIEW
    BUSINESS_RULE_FAILURE
    TEMPORARY_DEPENDENCY_FAILURE

Keep categories stable while allowing detailed rule names underneath.

### 2.16 Severity

Severity can be:

    INFO
    WARNING
    ERROR
    CRITICAL

Severity should describe impact, not merely how unusual a value looks.

### 2.17 Privacy-safe diagnostics

Avoid putting sensitive payloads into logs.

Prefer:

    record_id
    rule_name
    failure_category
    source_batch_id
    safe metadata

If original data must be retained, use controlled storage and access.

### 2.18 Replay boundary

Replay should have a precise scope:

    one record
    one quarantine batch
    one source batch
    one partition
    one time range

Avoid replaying the entire pipeline when a narrow replay is sufficient.

### 2.19 Replay semantics

A replay should execute the same or explicitly versioned validation and transformation logic.

Record:

    replay_id
    original_quarantine_id
    rule_version
    pipeline_version
    replay_started_at
    replay_finished_at
    result

### 2.20 Poison records

Some records repeatedly fail.

Without limits:

    retry
      ↓
    fail
      ↓
    retry
      ↓
    fail
      ↓
    infinite loop

Use bounded retries and move persistent failures to a terminal state.

### 2.21 Corrected-record replay

A corrected record should not be confused with the original input.

Track:

    original_record
    correction
    corrected_record
    replay_result

This preserves auditability.

### 2.22 Accounting invariant

The central invariant is:

    input = accepted + quarantined + rejected + unresolved

The categories must be mutually defined.

Do not let records disappear between states.

### 2.23 Duplicate accounting

If one record has five validation errors, it still counts as one record for record-level reconciliation.

Failure counts and record counts are different metrics.

### 2.24 Quarantine rate

Track:

    quarantine_rate =
        quarantined_records / input_records

But also track by reason.

A stable overall rate can hide a new critical failure category.

### 2.25 Recovery rate

For retriable or corrected records:

    recovery_rate =
        records_reaccepted / records_quarantined

Interpret recovery by failure category.

### 2.26 Aging

A quarantine queue can grow even when daily quarantine rate is stable.

Track:

- oldest open record;
- median age;
- P95 age;
- records by state;
- records by failure category.

### 2.27 Quarantine capacity

A quarantine system is part of the pipeline.

Monitor:

- storage usage;
- insertion failures;
- retention;
- replay throughput;
- retry queue depth.

## 3. Implementation

### 3.1 Quarantine table

A relational implementation can begin with:

    CREATE TABLE record_quarantine (
        quarantine_id bigserial PRIMARY KEY,
        source_system text NOT NULL,
        source_batch_id text NOT NULL,
        source_record_id text,
        record_fingerprint text,
        failure_category text NOT NULL,
        severity text NOT NULL,
        disposition text NOT NULL,
        state text NOT NULL,
        retryable boolean NOT NULL DEFAULT false,
        rule_name text NOT NULL,
        rule_version text NOT NULL,
        pipeline_version text,
        payload_reference text,
        safe_context jsonb NOT NULL DEFAULT '{}'::jsonb,
        first_seen_at timestamptz NOT NULL DEFAULT now(),
        last_attempt_at timestamptz,
        attempt_count integer NOT NULL DEFAULT 0,
        closed_at timestamptz
    );

Do not put sensitive payloads into this table unless storage policy explicitly allows it.

### 3.2 Validation findings table

Keep multiple failures separately:

    CREATE TABLE quarantine_finding (
        finding_id bigserial PRIMARY KEY,
        quarantine_id bigint NOT NULL,
        rule_name text NOT NULL,
        rule_version text NOT NULL,
        failure_category text NOT NULL,
        severity text NOT NULL,
        observed_value text,
        expected_value text,
        created_at timestamptz NOT NULL DEFAULT now()
    );

Sensitive observed values should be redacted or omitted.

### 3.3 State-transition table

For an auditable lifecycle:

    CREATE TABLE quarantine_transition (
        transition_id bigserial PRIMARY KEY,
        quarantine_id bigint NOT NULL,
        from_state text,
        to_state text NOT NULL,
        reason text NOT NULL,
        actor text NOT NULL,
        replay_id text,
        created_at timestamptz NOT NULL DEFAULT now()
    );

The transition history should be append-only.

### 3.4 Stable identity

Create a deterministic fingerprint when a source identifier is unavailable.

Conceptually:

    fingerprint =
        SHA256(
            source_system
            + source_batch_id
            + canonical_record_identity
        )

Do not hash arbitrary raw payloads when field ordering or serialization can change.

Canonicalize first.

### 3.5 Idempotent quarantine insert

A suitable pattern is:

    INSERT INTO record_quarantine (
        source_system,
        source_batch_id,
        source_record_id,
        record_fingerprint,
        failure_category,
        severity,
        disposition,
        state,
        retryable,
        rule_name,
        rule_version
    )
    VALUES (...)
    ON CONFLICT DO NOTHING;

The exact unique constraint should reflect the intended identity.

### 3.6 Validation pipeline

Conceptually:

    raw record
        ↓
    parse
        ↓
    validate
        ↓
    collect findings
        ↓
    classify
        ↓
    ┌───────────────┬─────────────────┐
    ↓               ↓                 ↓
    ACCEPTED     QUARANTINE       BATCH FAILURE

Do not mix classification logic with persistence side effects unnecessarily.

### 3.7 Python validation result

A simple internal representation:

    from dataclasses import dataclass

    @dataclass
    class Finding:
        rule_name: str
        rule_version: str
        category: str
        severity: str
        retryable: bool
        message: str

Keep the message privacy-safe.

### 3.8 Classification

    def classify(findings):
        if not findings:
            return {
                "status": "ACCEPTED",
                "retryable": False,
            }

        retryable = any(item.retryable for item in findings)

        return {
            "status": "INVALID",
            "retryable": retryable,
        }

A production implementation should additionally apply explicit severity and precedence policy.

### 3.9 Failure precedence

Represent precedence as configuration:

    FAILURE_PRECEDENCE = {
        "SCHEMA_ERROR": 100,
        "PARSE_ERROR": 90,
        "REQUIRED_FIELD_MISSING": 80,
        "TYPE_INVALID": 70,
        "REFERENTIAL_FAILURE": 60,
        "DOMAIN_INVALID": 50,
        "RANGE_INVALID": 40,
        "ANOMALY_REVIEW": 20,
    }

Then select the highest applicable policy priority rather than relying on rule execution order.

### 3.10 Record-level accounting

For every source batch, calculate:

    input_count
    accepted_count
    quarantined_count
    rejected_count
    unresolved_count

Then verify:

    input_count =
        accepted_count
        + quarantined_count
        + rejected_count
        + unresolved_count

### 3.11 Finding-level accounting

Separately calculate:

    total_findings
    records_with_findings
    findings_by_category
    findings_by_rule

A single record can produce multiple findings.

### 3.12 Retry queue

A retriable quarantine record can transition to:

    RETRY_PENDING

Only the retry worker should move it to:

    RETRYING

Use bounded attempts.

### 3.13 Exponential backoff

A simple policy:

    delay = base_delay * 2^attempt

Add a maximum delay and jitter in distributed systems.

Do not retry forever.

### 3.14 Terminal state

After a retry limit:

    RETRYING
       ↓
    PERMANENT_FAILURE

The record remains available for manual investigation or future policy-driven replay.

### 3.15 Replay transaction

For database-backed pipelines, use a transaction around:

    fetch quarantine record
    validate
    transform
    publish
    update quarantine state

The exact transaction boundary depends on external side effects.

### 3.16 Corrected data

If manual correction is allowed, store a correction record:

    CREATE TABLE quarantine_correction (
        correction_id bigserial PRIMARY KEY,
        quarantine_id bigint NOT NULL,
        corrected_payload_reference text NOT NULL,
        correction_reason text NOT NULL,
        corrected_by text NOT NULL,
        correction_version text NOT NULL,
        created_at timestamptz NOT NULL DEFAULT now()
    );

Never overwrite the original evidence.

### 3.17 Reconciliation table

Store batch accounting:

    CREATE TABLE invalid_record_reconciliation (
        batch_id text PRIMARY KEY,
        input_count bigint NOT NULL,
        accepted_count bigint NOT NULL,
        quarantined_count bigint NOT NULL,
        rejected_count bigint NOT NULL,
        unresolved_count bigint NOT NULL,
        reconciled boolean NOT NULL,
        reconciled_at timestamptz
    );

### 3.18 Reconciliation query

    SELECT
        batch_id,
        input_count,
        accepted_count,
        quarantined_count,
        rejected_count,
        unresolved_count,
        input_count
            - accepted_count
            - quarantined_count
            - rejected_count
            - unresolved_count AS unexplained_count
    FROM invalid_record_reconciliation;

Expected:

    unexplained_count = 0

### 3.19 Quarantine rate

    SELECT
        batch_id,
        quarantined_count::numeric
            / NULLIF(input_count, 0) AS quarantine_rate
    FROM invalid_record_reconciliation;

Trend this by failure category as well.

### 3.20 Replay result

Record:

    replay_id
    quarantine_id
    attempt_number
    result
    rule_version
    pipeline_version
    started_at
    finished_at
    failure_category

A successful replay should produce evidence that the record returned to the normal path.

### 3.21 Batch-safe publication

A useful architecture is:

    raw
      ↓
    staging
      ↓
    validation
      ↓
    valid path ─────────→ target
      ↓
    invalid path ───────→ quarantine

The invalid path must not be an implicit DROP.

### 3.22 Preserve ordering where required

For event streams, quarantine handling must not accidentally violate business ordering.

Some systems can continue processing valid records.

Others require:

    failed event
        ↓
    stop subsequent dependent events

Choose explicitly.

### 3.23 Tenant isolation

If data is multi-tenant, include tenant identity in:

- quarantine key;
- authorization checks;
- replay scope;
- observability;
- reconciliation.

Never allow a replay worker to accidentally mix tenants.

## 4. Testing

### 4.1 All-valid batch

Input:

    100 records
    100 valid

Expected:

    accepted = 100
    quarantined = 0
    unexplained = 0

### 4.2 One invalid record

Input:

    100 records
     99 valid
      1 invalid

Expected:

    accepted = 99
    quarantined = 1
    unexplained = 0

### 4.3 Multiple failures on one record

Create one record failing three rules.

Expected:

    quarantined_records = 1
    findings = 3

Do not count three findings as three quarantined records.

### 4.4 Retryable failure

Create a temporary dependency failure.

Expected:

    retryable = true
    state = RETRY_PENDING

### 4.5 Permanent failure

Create an impossible domain value.

Expected:

    retryable = false
    no infinite retry

### 4.6 Retry exhaustion

Force repeated transient failures.

Expected:

    attempts stop at configured limit
    state = PERMANENT_FAILURE

### 4.7 Idempotent quarantine

Process the same invalid record twice.

Expected:

    one logical quarantine record

### 4.8 Duplicate finding

Run the same validation rule twice.

Expected behavior should be explicitly defined.

### 4.9 Correction and replay

Correct a quarantined record.

Expected:

    original preserved
    correction recorded
    replay succeeds
    quarantine state updated
    reconciliation remains balanced

### 4.10 Failed replay

Replay an invalid record after correction but force another failure.

Expected:

    original quarantine remains recoverable
    new attempt is recorded
    state remains appropriate

### 4.11 Batch-level schema failure

Remove a required source column.

Expected:

    batch-level failure
    no misleading row-level quarantine storm

### 4.12 Accounting invariant

For every test batch verify:

    input =
        accepted
        + quarantined
        + rejected
        + unresolved

Expected:

    unexplained = 0

### 4.13 Privacy test

Ensure sensitive fields do not appear in:

- logs;
- exception strings;
- metrics labels;
- quarantine diagnostics.

### 4.14 Tenant-isolation test

Create records from two tenants.

Replay one tenant.

Expected:

    only that tenant's records are affected

### 4.15 Replay idempotence

Replay the same quarantine record twice under the same replay policy.

Expected:

    no duplicate target effect

Use an idempotency key where the target supports it.

### 4.16 State-transition test

Attempt invalid transitions such as:

    CLOSED → RETRYING

Expected:

    transition rejected

The lifecycle must behave like a state machine.

## 5. Observability

Invalid-record handling requires record-level and batch-level visibility.

### 5.1 Core metrics

| Metric | Meaning |
|---|---|
| invalid_records_total | Records classified invalid |
| quarantined_records_total | Records moved to quarantine |
| rejected_records_total | Records permanently rejected |
| retriable_records_total | Records eligible for retry |
| replay_attempts_total | Replay attempts |
| replay_success_total | Successful replays |
| replay_failure_total | Failed replays |
| quarantine_queue_depth | Open quarantine population |
| quarantine_oldest_age_seconds | Age of oldest unresolved record |
| quarantine_rate | Quarantined / input |
| recovery_rate | Reaccepted / quarantined |
| unexplained_records_total | Records missing from accounting |

### 5.2 Failure-category metrics

Track:

    invalid_records_total{
        category="DOMAIN_INVALID"
    }

Avoid high-cardinality labels such as raw record IDs.

### 5.3 Rule-level metrics

Track:

- failures by rule;
- failures by version;
- records affected;
- finding count;
- recovery rate.

This reveals which validation rule is generating operational load.

### 5.4 State metrics

Track:

    OPEN
    RETRY_PENDING
    RETRYING
    MANUAL_REVIEW
    ACCEPTED
    PERMANENT_FAILURE
    CLOSED

Monitor queue growth by state.

### 5.5 Aging

Monitor:

- oldest open record;
- P50 age;
- P95 age;
- P99 age.

A low daily quarantine rate can still hide an accumulating backlog.

### 5.6 Batch reconciliation dashboard

For each batch show:

    INPUT
      ├── ACCEPTED
      ├── QUARANTINED
      ├── REJECTED
      └── UNRESOLVED

The unexplained branch must always be zero before a batch is considered reconciled.

### 5.7 Replay monitoring

Track:

- replay throughput;
- retry attempts;
- successful replay rate;
- failure categories;
- retry exhaustion;
- replay latency.

### 5.8 Alerting

Alert when:

- quarantine rate exceeds policy;
- a new failure category appears;
- retry queue grows;
- oldest quarantine age exceeds SLA;
- unexplained records > 0;
- replay failures spike;
- a previously rare rule suddenly dominates failures.

### 5.9 Privacy observability

Do not expose sensitive payload values in dashboards.

Use:

- stable identifiers;
- categories;
- counts;
- hashes where appropriate;
- controlled payload references.

## 6. Intentional Failure

Break invalid-record handling deliberately.

### Failure 1 — Drop invalid rows without quarantine

Expected:

    reconciliation detects unexplained records

### Failure 2 — Duplicate quarantine insertion

Process the same record twice.

Expected:

    idempotent identity prevents uncontrolled duplication

### Failure 3 — Permanent failure sent to retry queue

Expected:

    classification prevents infinite retry

### Failure 4 — Retryable dependency failure

Temporarily break a reference service.

Expected:

    RETRY_PENDING
    bounded retries
    eventual recovery when dependency returns

### Failure 5 — Retry exhaustion

Keep dependency unavailable.

Expected:

    terminal state after configured attempts

### Failure 6 — Multiple validation findings

Cause three rules to fail.

Expected:

    one quarantined record
    three findings

### Failure 7 — Corrupt batch schema

Remove a required source column.

Expected:

    batch-level failure rather than millions of misleading row failures

### Failure 8 — Invalid state transition

Attempt:

    CLOSED → RETRYING

Expected:

    transition rejected

### Failure 9 — Cross-tenant replay

Attempt replay with the wrong tenant scope.

Expected:

    authorization or scope validation blocks the operation

### Failure 10 — Sensitive diagnostic logging

Force an exception containing a sensitive value.

Expected:

    privacy-safe logging removes or prevents the sensitive value

## 7. Recovery

### 7.1 Record-level invalid data

1. Identify quarantine record.
2. Inspect failure category and rule version.
3. Determine whether the defect is permanent or correctable.
4. Correct only under approved policy.
5. Preserve original evidence.
6. Replay the smallest scope.
7. Re-run validation.
8. Update quarantine state.
9. Reconcile batch accounting.

### 7.2 Temporary dependency failure

1. Confirm the dependency failure.
2. Keep records in RETRY_PENDING.
3. Wait according to backoff policy.
4. Retry within the configured limit.
5. Confirm successful validation.
6. Publish accepted records.
7. Reconcile.

Do not modify source records for a temporary dependency outage.

### 7.3 Validation-rule defect

If valid records were incorrectly quarantined:

1. identify affected rule version;
2. stop or isolate the defective rule;
3. determine affected quarantine scope;
4. deploy corrected rule version;
5. replay affected records;
6. compare old and new results;
7. preserve original decisions.

### 7.4 Parser defect

If malformed parsing caused valid records to be quarantined:

1. preserve raw source artifact;
2. identify parser version;
3. reproduce failure;
4. fix parser;
5. validate against representative artifacts;
6. replay affected records;
7. reconcile counts.

### 7.5 Quarantine storage failure

If quarantine writes fail:

1. do not silently continue as if records were handled;
2. fail or pause according to pipeline policy;
3. preserve source batch;
4. restore quarantine storage;
5. replay from durable input;
6. reconcile accepted and quarantined populations.

### 7.6 Replay failure

A failed replay should not erase the original quarantine record.

Keep:

    original quarantine
    + replay attempt
    + replay result

Then classify the new failure.

### 7.7 Safe closure

A quarantine record can be CLOSED only when:

- disposition is known;
- replay or rejection is complete;
- evidence is preserved;
- accounting is reconciled;
- no further operational action is required.

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

PostgreSQL is useful for:

- durable quarantine state;
- validation findings;
- state transitions;
- reconciliation;
- replay metadata;
- unique constraints for idempotency.

The key skill is designing a recoverable lifecycle rather than merely inserting failed rows.

### 8.2 dbt

dbt can help materialize validation outputs and identify records that violate model-level expectations.

Use it to produce structured quality evidence that downstream quarantine workflows can consume.

### 8.3 Great Expectations

Great Expectations can produce structured validation results and expectation metadata.

Use those results as evidence for classification.

Do not let the validation framework become the quarantine state machine. Operational disposition still requires explicit lifecycle design.

## 9. Production Runbook

### Alert

1. Identify source system and batch.
2. Check input count.
3. Check quarantine rate.
4. Check unexplained count.
5. Check failure categories.
6. Check queue depth and oldest age.

### Diagnose

7. Determine record-level versus batch-level failure.
8. Inspect the dominant validation rule.
9. Determine retryable versus permanent failures.
10. Check recent deployments.
11. Check dependencies and reference data.
12. Check tenant scope where applicable.
13. Inspect representative safe evidence.

### Recover

14. Restore missing dependencies.
15. Correct source or transformation defects.
16. Replay the smallest affected scope.
17. Re-run validation.
18. Update quarantine state.
19. Reconcile batch counts.

### Close

20. Confirm unexplained count is zero.
21. Confirm quarantine backlog is within SLA.
22. Record root cause.
23. Preserve original and replay evidence.
24. Confirm no sensitive payload leaked into logs or metrics.

### Incident decision table

| Observation | Classification | Action |
|---|---|---|
| One malformed record | Record-level permanent failure | Quarantine |
| Temporary reference outage | Record-level retriable failure | Retry with backoff |
| Missing required source column | Batch-level failure | Stop and recover source/schema |
| High quarantine rate after deployment | Transformation/rule regression | Investigate release and replay affected scope |
| Repeated same-record failures | Poison record | Bound retries and move to terminal state |
| Valid record quarantined by bad rule | Validation defect | Fix rule and replay affected records |
| Quarantine storage unavailable | Pipeline safety failure | Pause/fail rather than silently discard |
| Unexplained records > 0 | Accounting failure | Block reconciliation and recover missing state |
| Cross-tenant replay request | Scope/security failure | Reject operation |

## 10. Common Mistakes

### Mistake 1 — Dropping invalid rows

Filtering is not operational handling.

### Mistake 2 — Retrying permanent failures forever

Use explicit retryability and retry limits.

### Mistake 3 — Storing only the error message

Preserve source identity, rule version, state, and replay evidence.

### Mistake 4 — Overwriting the original record during correction

Preserve original evidence.

### Mistake 5 — Counting findings as records

One record can have many findings.

### Mistake 6 — Using validation order as failure precedence

Precedence should be explicit policy.

### Mistake 7 — Treating every failure as record-level

Systemic schema and parser failures may require batch-level stopping.

### Mistake 8 — Ignoring quarantine aging

A growing backlog can become a second production incident.

### Mistake 9 — Missing replay idempotency

A successful replay can create duplicate target effects without idempotency protection.

### Mistake 10 — Mixing tenants during replay

Replay scope must preserve tenant isolation.

### Mistake 11 — Logging raw sensitive payloads

Diagnostics should be privacy-safe.

### Mistake 12 — Closing quarantine records without reconciliation

Closure must be evidence-backed.

### Mistake 13 — Repairing data just to improve the quality score

Fix the underlying data or rule, then remeasure.

### Mistake 14 — Making quarantine the final destination

Quarantine is a controlled lifecycle state, not an undocumented data graveyard.

## 11. Definition of Done

You are done with T43 when you can:

- distinguish invalid, rejected, quarantined, retriable, and permanent records;
- define record-level and batch-level failure boundaries;
- collect multiple validation findings;
- classify failures deterministically;
- implement failure precedence;
- distinguish retryable from permanent failures;
- design a durable quarantine store;
- preserve source identity and evidence;
- implement idempotent quarantine insertion;
- model explicit quarantine states;
- record append-only state transitions;
- implement bounded retries and backoff;
- support corrected-record replay;
- preserve original evidence;
- implement batch accounting;
- prove input equals accounted-for outcomes;
- calculate quarantine and recovery rates;
- monitor quarantine aging;
- implement replay metadata;
- enforce tenant-safe replay;
- protect sensitive values in diagnostics;
- test permanent and transient failures;
- test duplicate processing;
- test retry exhaustion;
- test batch-level failures;
- intentionally break quarantine handling;
- recover from rule, parser, dependency, and storage failures;
- operate the quarantine lifecycle with the production runbook.

## 12. What You Learned

Invalid-record handling is the operational layer between validation and reliable data delivery.

The production mental model is:

    INPUT
      ↓
    VALIDATE
      ↓
    CLASSIFY
      ↓
    ┌───────────┬──────────────┬──────────────┐
    ↓           ↓              ↓
  ACCEPTED   QUARANTINED    BATCH FAILURE
    ↓           ↓
  PUBLISH    CLASSIFY
                ↓
        RETRY / CORRECT / REVIEW
                ↓
             REPLAY
                ↓
          ACCEPT / TERMINATE
                ↓
           RECONCILE

You learned to:

- preserve invalid records instead of silently dropping them;
- distinguish record-level from batch-level failures;
- classify failures into stable operational categories;
- separate validation findings from disposition;
- preserve original evidence;
- implement retryable and permanent states;
- bound retries;
- make quarantine and replay idempotent;
- maintain batch accounting invariants;
- monitor quarantine rate, aging, and recovery;
- protect tenant boundaries and sensitive diagnostics;
- recover safely from source, parser, rule, dependency, and storage failures.

The production principle is:

> **An invalid record is not a missing record. It must have a traceable disposition, recoverable evidence, and an auditable place in the pipeline's accounting.**

### Next Recipe

**T44 — Quarantine During Transformation — Transform Application**

T44 will apply the quarantine lifecycle directly inside transformation logic, covering transform-time validation, partial transformation failures, side-output design, atomic publication, and safe replay.
