# T44 — Quarantine During Transformation — Transform Application
> **Goal:** Apply the quarantine lifecycle directly inside transformation logic without silently dropping records or publishing partially transformed data.

T43 established the general invalid-record lifecycle. T44 applies that lifecycle inside transformation stages, where records can become invalid through parsing, enrichment, joins, calculations, or business transformations.

The production goal is:

    valid records → continue
    invalid records → durable quarantine
    systemic failures → fail safely
    no record → silently disappear

## 1. Problem Recognition
Transformation is not guaranteed to succeed for every input record.

Common failures include malformed timestamps, missing reference data, ambiguous lookups, invalid derived values, numeric overflow, unsupported domains, unexpected join fan-out, and post-transform consistency failures.

### 1.1 Why transform-time quarantine matters
Validation before transformation can catch source defects. Transformation can create new defects.

Example:

    raw_amount = "100.00"
    currency = "USD"
    exchange_rate = missing

The source record may be valid, but the transformation cannot safely produce a converted amount.

### 1.2 Dangerous implementation
Do not silently discard failed transforms.

    transformed = [
        transform(row)
        for row in rows
        if transform(row) is not None
    ]

This can execute work twice, lose failure evidence, and make reconciliation impossible.

### 1.3 Failure categories
Typical transform-time failures:

- parse failure;
- type failure;
- lookup failure;
- ambiguous lookup;
- derived-value failure;
- overflow;
- domain failure;
- referential failure;
- consistency failure;
- business-rule failure;
- unexpected fan-out;
- missing transformation context.

### 1.4 Define the transformation contract
Every transformation should define:

- input grain;
- output grain;
- expected cardinality;
- required references;
- allowed record-level failures;
- batch-level failure conditions;
- quarantine behavior;
- publication boundary;
- replay scope.

### 1.5 Record-level versus systemic failure
Record-level:

    one input row cannot be transformed

Systemic:

    transformation code is broken
    required schema is incompatible
    reference dependency is unavailable
    database cannot persist results

These must not share one generic error path.

### 1.6 Partial success
If policy permits partial success:

    input = accepted + quarantined

If the counts do not reconcile, the transformation is incomplete.

### 1.7 Atomic publication
Use a publication boundary:

    input
      ↓
    transform
      ↓
    valid staging + quarantine
      ↓
    reconcile
      ↓
    publish

Do not publish before reconciliation.

### 1.8 Side-output principle
Treat transformation as producing two explicit outputs:

    transform(input) →
        valid_records
        invalid_records

The invalid output is a first-class dataset.

## 2. Concept and Reasoning
### 2.1 Transformation as a partition
For each record the result should be explicit:

    ACCEPT
    or
    QUARANTINE

For systemic failures:

    FAIL_BATCH

### 2.2 Result envelope
Return an outcome rather than only transformed data:

    {
        status: ACCEPTED,
        input_record_id: "...",
        output_record: {...}
    }

or:

    {
        status: QUARANTINED,
        input_record_id: "...",
        findings: [...]
    }

Expected failure becomes data instead of an exception-only path.

### 2.3 Exceptions versus expected invalidity
An invalid business value should normally quarantine the record.

A programming exception should normally fail the transformation.

Do not catch every exception and label it bad source data.

### 2.4 Failure taxonomy
Use explicit classes:

    EXPECTED_RECORD_FAILURE
    TRANSIENT_DEPENDENCY_FAILURE
    SYSTEM_FAILURE
    PROGRAMMING_FAILURE

Example policy:

| Failure | Default action |
|---|---|
| Invalid source value | Quarantine record |
| Temporary reference outage | Retry or fail stage |
| Database outage | Fail stage |
| Programmer exception | Fail stage |
| Schema incompatibility | Fail batch |
| Business-rule violation | Quarantine record |

### 2.5 Transaction boundaries
Valid rows, quarantine rows, and reconciliation state need a consistent persistence strategy.

A relational pattern is:

    transform staging
      ↓
    write valid rows
    write quarantine rows
    write accounting
      ↓
    reconcile
      ↓
    publish

The exact transaction boundary depends on external side effects.

### 2.6 Preserve source identity
Retain identifiers such as:

    source_batch_id
    source_record_id
    event_id
    transformation_run_id

### 2.7 Transformation run identity
A run identifier should identify:

- source batch;
- pipeline version;
- rule version;
- configuration;
- reference snapshot;
- execution time.

### 2.8 Reference-data snapshot
If transformation depends on mutable reference data, record a snapshot ID or effective version.

Otherwise replay can produce a different result from the original run.

### 2.9 Determinism
A replay should use the same input, transformation version, configuration, and reference version when deterministic reproduction is required.

### 2.10 Output cardinality
Explicitly define:

    1 input → 1 output
    1 input → 0 outputs
    1 input → many outputs

Unexpected fan-out can itself be a transformation defect.

### 2.11 Error locality
A useful error identifies:

    run
    batch
    record
    step
    rule
    category

"Transformation failed" is not enough for diagnosis.

### 2.12 No silent fallback
Do not silently replace missing reference data with arbitrary defaults.

For example:

    missing exchange rate → rate = 1

can produce financially incorrect but apparently valid data.

### 2.13 Quarantine versus fail-fast
Quarantine expected record-level defects when valid records can safely continue.

Fail the stage for systemic conditions such as broken code, incompatible schema, unavailable required reference data, or unsafe publication.

### 2.14 Atomicity versus throughput
Possible publication boundaries include:

- record;
- partition;
- batch;
- manifest.

Choose the smallest boundary that preserves correctness and operational efficiency.

### 2.15 Idempotency
A replay must not duplicate target effects.

Use an appropriate source identity, output identity, or business idempotency key.

### 2.16 Privacy
Keep sensitive payloads out of logs and metrics. Use controlled payload references when original evidence must be retained.

## 3. Implementation
### 3.1 Transformation run table
    CREATE TABLE transformation_run (
        run_id uuid PRIMARY KEY,
        pipeline_name text NOT NULL,
        pipeline_version text NOT NULL,
        rule_version text NOT NULL,
        source_batch_id text NOT NULL,
        reference_snapshot_id text,
        status text NOT NULL,
        input_count bigint NOT NULL DEFAULT 0,
        accepted_count bigint NOT NULL DEFAULT 0,
        quarantined_count bigint NOT NULL DEFAULT 0,
        failed_count bigint NOT NULL DEFAULT 0,
        started_at timestamptz NOT NULL DEFAULT now(),
        completed_at timestamptz
    );

### 3.2 Transform quarantine
    CREATE TABLE transform_quarantine (
        quarantine_id bigserial PRIMARY KEY,
        run_id uuid NOT NULL,
        source_batch_id text NOT NULL,
        source_record_id text NOT NULL,
        step_name text NOT NULL,
        failure_category text NOT NULL,
        retryable boolean NOT NULL,
        rule_version text NOT NULL,
        safe_context jsonb NOT NULL DEFAULT '{}'::jsonb,
        payload_reference text,
        created_at timestamptz NOT NULL DEFAULT now()
    );

### 3.3 Transform staging
    CREATE TABLE transformed_staging (
        run_id uuid NOT NULL,
        source_record_id text NOT NULL,
        output_record_id text NOT NULL,
        transformed_at timestamptz NOT NULL DEFAULT now(),
        payload jsonb NOT NULL,
        PRIMARY KEY (run_id, source_record_id, output_record_id)
    );

### 3.4 Explicit transform result
    from dataclasses import dataclass
    from typing import Any

    @dataclass
    class TransformResult:
        status: str
        source_record_id: str
        output: dict[str, Any] | None
        findings: list[dict[str, Any]]

Possible statuses:

    ACCEPTED
    QUARANTINED
    FAILED

### 3.5 Record-level transformation
    def transform_record(row, rate_lookup):
        try:
            amount = float(row["amount"])
        except (TypeError, ValueError):
            return TransformResult(
                status="QUARANTINED",
                source_record_id=row["id"],
                output=None,
                findings=[{
                    "category": "TYPE_INVALID",
                    "rule": "amount_numeric"
                }],
            )

        rate = rate_lookup.get(row["currency"])

        if rate is None:
            return TransformResult(
                status="QUARANTINED",
                source_record_id=row["id"],
                output=None,
                findings=[{
                    "category": "REFERENCE_MISSING",
                    "rule": "currency_rate"
                }],
            )

        converted = amount * rate

        return TransformResult(
            status="ACCEPTED",
            source_record_id=row["id"],
            output={
                "id": row["id"],
                "converted_amount": converted,
            },
            findings=[],
        )

The example separates expected record failures from successful transformation.

### 3.6 Batch transformation
    def transform_batch(rows, rate_lookup):
        accepted = []
        quarantined = []

        for row in rows:
            result = transform_record(row, rate_lookup)

            if result.status == "ACCEPTED":
                accepted.append(result)
            elif result.status == "QUARANTINED":
                quarantined.append(result)
            else:
                raise RuntimeError(
                    f"Unexpected transform status: {result.status}"
                )

        return accepted, quarantined

Do not catch broad programming exceptions here and turn them into quarantine.

### 3.7 Accounting
    input_count = len(rows)
    accepted_count = len(accepted)
    quarantined_count = len(quarantined)

    assert input_count == accepted_count + quarantined_count

Persist this invariant in durable run state as well.

### 3.8 Persist valid and invalid sides
Write:

    valid_records → transformed_staging
    invalid_records → transform_quarantine

Then persist:

    input_count
    accepted_count
    quarantined_count

### 3.9 Transactional publication
A relational pattern is:

    BEGIN
    insert valid rows
    insert quarantine rows
    update transformation_run
    verify input = accepted + quarantined
    COMMIT

Only after the successful commit should publication continue.

### 3.10 Publish after reconciliation
Use:

    transformed_staging
          ↓
    reconciliation
          ↓
    publish target
          ↓
    mark run complete

If reconciliation fails, do not publish.

### 3.11 Quarantine idempotency
Use an appropriate unique identity.

    CREATE UNIQUE INDEX uq_transform_quarantine
    ON transform_quarantine (
        run_id,
        source_record_id,
        step_name,
        failure_category
    );

Adjust the key if multiple findings are expected for the same step.

### 3.12 Output idempotency
Target publication must have its own idempotency strategy.

Do not assume deterministic transformation automatically makes publication idempotent.

### 3.13 Reference snapshot
Record the lookup version:

    UPDATE transformation_run
    SET reference_snapshot_id = 'fx-2026-09-28-1400'
    WHERE run_id = '...';

Use the documented snapshot policy during replay.

### 3.14 Transformation-step lineage
For multi-stage pipelines:

    raw
      ↓
    parse
      ↓
    normalize
      ↓
    enrich
      ↓
    calculate
      ↓
    publish

Record the first step where the record became invalid.

### 3.15 Partial transformation failure
If:

    input = 100
    accepted = 97
    quarantined = 3

the run can complete if partial success is allowed.

If the transformation code itself raises an unexpected exception, follow systemic-failure policy instead.

### 3.16 Run state
Use explicit states such as:

    RUNNING
    RECONCILING
    READY_TO_PUBLISH
    PUBLISHED
    PARTIAL_SUCCESS
    FAILED

Do not infer publication state from row counts.

### 3.17 Replay scope
Select retryable quarantine records explicitly:

    SELECT
        source_record_id,
        payload_reference
    FROM transform_quarantine
    WHERE run_id = '...'
      AND failure_category = 'REFERENCE_MISSING'
      AND retryable = true;

Replay only the eligible population.

### 3.18 Replay after dependency recovery
    reference fixed
       ↓
    select retryable quarantine
       ↓
    transform
       ↓
    accepted → staging
    still invalid → quarantine
       ↓
    reconcile
       ↓
    publish

### 3.19 Corrected record
If manual correction is required:

    original
       ↓
    correction record
       ↓
    corrected input
       ↓
    replay

Never overwrite original evidence.

### 3.20 Unexpected fan-out
If the contract is one-to-one:

    expected_output_count = accepted_input_count

Check the actual output count before publication.

### 3.21 Post-transform consistency
Validate derived invariants before publication.

Example:

    transformed_total =
        transformed_net + transformed_tax

If the invariant fails, quarantine the affected record rather than publishing an inconsistent result.

### 3.22 Batch manifest
A manifest can capture:

    run_id
    input_count
    accepted_count
    quarantined_count
    output_count
    checksum
    status

The manifest becomes a publication contract.

### 3.23 Atomic object publication
For object-based output:

    write files
       ↓
    validate
       ↓
    write manifest
       ↓
    mark manifest COMPLETE

Consumers read only COMPLETE manifests.

### 3.24 Partial-output recovery
If a run fails after partial staging:

1. identify run ID;
2. mark run FAILED;
3. block publication;
4. isolate partial staging;
5. replay from durable input;
6. reconcile again.

Do not guess which partial rows are safe.

## 4. Testing
### 4.1 All-valid transform
Input 100 records.

Expected:

    accepted = 100
    quarantined = 0
    failed = 0

### 4.2 Mixed valid and invalid
Input 100 records with 5 invalid.

Expected:

    accepted = 95
    quarantined = 5

### 4.3 Multiple failure categories
Create parse, lookup, and domain failures.

Verify each record receives the correct category.

### 4.4 Programming exception
Inject an unexpected application error.

Expected:

    transformation run fails

It must not become ordinary record quarantine.

### 4.5 Missing lookup
Remove one reference row.

Expected:

    affected records quarantined
    unaffected records continue
    accounting reconciles

### 4.6 Lookup outage
Make the reference dependency unavailable.

Expected policy must be explicit:

    retry stage
    or
    fail stage

Do not create misleading millions-of-record quarantine output for a systemic outage.

### 4.7 Accounting invariant
Verify:

    input = accepted + quarantined

for partial-success mode.

### 4.8 Unexpected fan-out
Make one input produce multiple outputs when one-to-one is required.

Expected:

    cardinality check fails

### 4.9 Derived-value failure
Break a post-transform invariant.

Expected:

    record quarantined before publication

### 4.10 Transaction rollback
Force an error after staging and quarantine writes begin.

Expected:

    no partial publication

### 4.11 Replay success
Restore the missing reference and replay.

Expected:

    quarantined records become accepted
    output remains idempotent
    accounting remains correct

### 4.12 Replay failure
Replay while the dependency remains invalid.

Expected:

    new attempt recorded
    original quarantine evidence retained

### 4.13 Replay idempotence
Replay the same quarantine scope twice.

Expected:

    no duplicate target effect

### 4.14 Reference snapshot
Run the same input against two reference versions.

Expected:

    results are distinguishable by reference_snapshot_id

### 4.15 Determinism
Use identical input, transformation version, configuration, and reference snapshot twice.

Expected:

    identical output

### 4.16 Privacy test
Force failure with sensitive input.

Expected:

    logs and safe_context do not expose prohibited values

### 4.17 Run-state test
Verify valid transitions such as:

    RUNNING → RECONCILING
    RECONCILING → READY_TO_PUBLISH
    READY_TO_PUBLISH → PUBLISHED

Reject invalid transitions such as:

    FAILED → PUBLISHED

### 4.18 Partial-output recovery
Simulate a crash after staging but before publication.

Expected:

    run remains non-publishable
    replay can safely reconstruct output

## 5. Observability
### 5.1 Core metrics

| Metric | Meaning |
|---|---|
| transform_input_records_total | Records entering transformation |
| transform_accepted_records_total | Successfully transformed records |
| transform_quarantined_records_total | Records sent to quarantine |
| transform_failed_runs_total | Systemic transform failures |
| transform_quarantine_rate | Quarantined / input |
| transform_output_records_total | Records produced |
| transform_replay_attempts_total | Replay attempts |
| transform_replay_success_total | Successful replays |
| transform_replay_failure_total | Failed replays |
| transform_unexplained_records_total | Accounting gaps |
| transform_run_duration_seconds | Transformation duration |

### 5.2 Failure-category metrics
Track failures by:

- transformation step;
- failure category;
- rule version;
- pipeline version.

Avoid high-cardinality record identifiers.

### 5.3 Stage metrics
For each stage expose:

    input
    accepted
    quarantined
    failed
    output

This identifies where quality degrades.

### 5.4 Reference dependency health
For enrichment-heavy transforms monitor:

- lookup hit rate;
- lookup miss rate;
- reference freshness;
- reference snapshot;
- dependency latency;
- dependency errors.

### 5.5 Reconciliation metrics
Every run should expose:

    input_count
    accepted_count
    quarantined_count
    output_count
    unexplained_count

Expected:

    unexplained_count = 0

### 5.6 Run-state dashboard
Show:

    RUNNING
    RECONCILING
    READY_TO_PUBLISH
    PUBLISHED
    PARTIAL_SUCCESS
    FAILED

Stuck runs should be visible.

### 5.7 Quarantine aging
Monitor:

- oldest transform quarantine record;
- retry queue depth;
- P95 quarantine age;
- records by failure category.

### 5.8 Alerting
Alert on:

- unexplained records > 0;
- quarantine rate above threshold;
- systemic transformation failure;
- lookup miss rate spike;
- output/input cardinality mismatch;
- replay failures;
- runs stuck in RECONCILING;
- partial output detected.

### 5.9 Deployment correlation
Correlate quarantine changes with:

- pipeline deployments;
- transformation-rule versions;
- reference-data changes.

## 6. Intentional Failure
### Failure 1 — Drop failed transforms
Modify the transformation to discard exceptions.

Expected:

    reconciliation exposes missing records

### Failure 2 — Missing reference row
Remove one currency rate.

Expected:

    affected records quarantined
    unaffected records continue

### Failure 3 — Reference outage
Make the lookup dependency unavailable.

Expected:

    systemic policy activates
    not one misleading quarantine entry per record

### Failure 4 — Unexpected programming exception
Raise an unhandled application error.

Expected:

    transformation run fails

Do not silently convert a programming defect into source-data invalidity.

### Failure 5 — Fan-out explosion
Make one input record produce multiple outputs.

Expected:

    cardinality contract detects the problem

### Failure 6 — Post-transform invariant failure
Create an incorrect derived total.

Expected:

    record quarantined before publication

### Failure 7 — Crash before commit
Terminate the process after staging writes.

Expected:

    publication does not occur
    partial state remains recoverable

### Failure 8 — Duplicate replay
Replay the same quarantine scope twice.

Expected:

    target remains idempotent

### Failure 9 — Wrong reference snapshot
Replay using a different reference version.

Expected:

    run metadata exposes the difference

### Failure 10 — Quarantine persistence failure
Make quarantine persistence fail.

Expected:

    pipeline does not continue as if invalid records were safely isolated

## 7. Recovery
### 7.1 Record-level transform failure
1. Identify transformation run.
2. Identify source record.
3. Inspect failure step and category.
4. Determine permanent versus retryable.
5. Correct source, reference, or rule as appropriate.
6. Replay the smallest affected scope.
7. Reconcile.
8. Publish only after reconciliation.

### 7.2 Systemic transformation failure
1. Stop publication.
2. Mark run FAILED.
3. Preserve run metadata.
4. Identify code, schema, or dependency defect.
5. Fix the defect.
6. Re-run from durable input.
7. Reconcile.
8. Publish the corrected run.

### 7.3 Reference-data outage
1. Confirm dependency failure.
2. Pause affected transformation according to policy.
3. Preserve source batch.
4. Restore reference availability.
5. Replay affected records or partitions.
6. Verify reference snapshot.
7. Reconcile.
8. Publish.

### 7.4 Rule defect
If a rule incorrectly quarantined valid data:

1. identify rule version;
2. isolate affected scope;
3. fix rule;
4. test representative records;
5. create a new rule version;
6. replay affected quarantine;
7. compare old and new outcomes;
8. preserve original decisions.

### 7.5 Partial staging failure
1. identify run ID;
2. block publication;
3. isolate partial staging;
4. replay from durable input;
5. verify idempotency;
6. reconcile;
7. publish once.

### 7.6 Output-cardinality failure
1. stop publication;
2. identify the first stage where cardinality changed;
3. inspect joins, splits, and aggregation;
4. fix transformation;
5. replay affected scope;
6. verify target grain;
7. publish after reconciliation.

### 7.7 Replay failure
Do not erase the original quarantine state.

Record:

    original quarantine
    + replay attempt
    + new result

Then apply the same lifecycle again.

### 7.8 Safe completion
A transformation run is complete only when:

- input is accounted for;
- valid output is staged;
- invalid output is quarantined;
- systemic failures are resolved;
- cardinality contract passes;
- reconciliation passes;
- publication succeeds;
- run state is updated.

## 8. Production Tools You Should Know
### 8.1 PostgreSQL
PostgreSQL can provide:

- transformation staging;
- quarantine persistence;
- reconciliation;
- transaction boundaries;
- unique constraints;
- publication metadata.

The core skill is making partial transformation outcomes durable and auditable.

### 8.2 dbt
dbt can model transformation stages and quality checks close to warehouse transformations.

Use it to make transformation logic and validation dependencies explicit.

For record-level quarantine, combine model results with an operational persistence pattern appropriate to the platform.

### 8.3 Great Expectations
Great Expectations can validate transformed datasets before publication and produce structured evidence.

Use it for dataset-level expectations alongside explicit record-level quarantine handling.

## 9. Production Runbook
### Alert
1. Identify pipeline and transformation run.
2. Check input count.
3. Check accepted and quarantined counts.
4. Check systemic failures.
5. Check unexplained count.
6. Check reference and dependency health.

### Diagnose
7. Identify the first transformation stage with abnormal failure rate.
8. Inspect failure categories.
9. Compare with recent deployments.
10. Compare reference-data versions.
11. Check output cardinality.
12. Check run state.
13. Determine record-level versus systemic failure.

### Recover
14. Restore dependencies or correct transformation logic.
15. Replay the smallest safe scope.
16. Re-run validation.
17. Reconcile input, accepted, quarantined, and output counts.
18. Verify publication idempotency.
19. Publish only after all gates pass.

### Close
20. Mark run state correctly.
21. Record root cause.
22. Preserve run and replay evidence.
23. Confirm quarantine backlog is handled.
24. Confirm no unexplained records remain.

### Incident decision table

| Observation | Classification | Action |
|---|---|---|
| Small set of invalid transformed records | Record-level | Quarantine and continue |
| Entire reference dependency unavailable | Systemic | Pause/fail affected stage |
| Programmer exception | Systemic/code defect | Fail run and fix code |
| One-to-one transform produces extra rows | Cardinality defect | Stop publication and investigate |
| Derived invariant fails | Record-level semantic failure | Quarantine affected record |
| Quarantine storage unavailable | Safety failure | Pause/fail rather than discard |
| Partial staging after crash | Recovery state | Block publication and replay |
| Rule version incorrectly rejects records | Transformation defect | Fix rule and replay affected scope |
| Replay succeeds twice | Idempotency test | Verify no duplicate target effect |
| Unexplained count > 0 | Accounting failure | Block completion and recover missing state |

## 10. Common Mistakes
### Mistake 1 — Catching every exception as a bad record
Programming defects and source-data defects are different.

### Mistake 2 — Dropping failed transformations
A failed record must have an explicit disposition.

### Mistake 3 — Publishing before reconciliation
Partial output can create inconsistent targets.

### Mistake 4 — Replaying against unknown reference state
Record the reference snapshot or effective version.

### Mistake 5 — Ignoring output cardinality
A transform can produce too many or too few records even when rows look valid.

### Mistake 6 — Making fallback values implicit
A fallback can hide a real data-quality defect.

### Mistake 7 — Replaying the entire dataset unnecessarily
Prefer narrow controlled replay.

### Mistake 8 — Failing the whole batch for expected record-level defects
Partial success can be safer when explicitly supported.

### Mistake 9 — Continuing after quarantine persistence fails
If invalid records cannot be durably isolated, data may be lost.

### Mistake 10 — Treating staging rows as published rows
Staging needs an explicit publication boundary.

### Mistake 11 — Ignoring rule and code versions
Without versions, diagnosis and replay become ambiguous.

### Mistake 12 — Allowing duplicate replay effects
Replay requires target-side idempotency.

### Mistake 13 — Logging raw payloads
Keep transformation diagnostics privacy-safe.

### Mistake 14 — Treating quarantine as permanent storage
Quarantine needs retention, replay, aging, and closure policies.

## 11. Definition of Done
You are done with T44 when you can:

- distinguish record-level transformation failures from systemic failures;
- design explicit valid and quarantine side outputs;
- define transformation input and output grain;
- define expected output cardinality;
- preserve source and run identity;
- store transformation-run metadata;
- persist quarantine evidence;
- implement explicit transform result states;
- separate expected invalidity from programming exceptions;
- implement partial-success accounting;
- enforce input-to-outcome reconciliation;
- stage transformed data before publication;
- implement atomic or manifest-based publication;
- make quarantine writes idempotent;
- make target publication idempotent;
- capture reference-data snapshots;
- support bounded replay;
- preserve original evidence during correction;
- detect unexpected fan-out;
- validate post-transform invariants;
- test rollback and partial output;
- intentionally break transform-time quarantine;
- recover from reference, rule, parser, and storage failures;
- operate the transformation lifecycle with the production runbook.

## 12. What You Learned
Transform-time quarantine is where data-quality engineering becomes part of transformation architecture.

The production mental model is:

    INPUT
      ↓
    TRANSFORM
      ↓
    ┌─────────────────────┐
    ↓                     ↓
    VALID OUTPUT       QUARANTINE
    ↓                     ↓
    STAGING            CLASSIFY
    ↓                     ↓
    RECONCILE       RETRY / CORRECT
    ↓                     ↓
    PUBLISH            REPLAY
                          ↓
                       RECONCILE

The transformation must preserve:

    input identity
    + transformation version
    + reference context
    + failure evidence
    + output identity

You learned to:

- isolate expected record-level transformation failures;
- distinguish systemic failures from bad records;
- use side outputs instead of silent drops;
- preserve invalid records for replay;
- reconcile accepted and quarantined populations;
- protect publication with staging and commit boundaries;
- enforce output-cardinality contracts;
- capture reference-data versions;
- make replay deterministic and idempotent;
- recover partial transformations safely;
- operate transformation quarantine as a first-class pipeline capability.

The production principle is:

> **A transformation is complete only when every input has an explicit outcome, every invalid record is recoverable, and no output is published before the transformation is reconciled.**

### Next Recipe

**T45 — Slowly Changing Dimensions**

T45 will introduce temporal dimension modeling: how dimensional attributes change over time, how historical truth is preserved, and how facts resolve against the correct dimension state.
