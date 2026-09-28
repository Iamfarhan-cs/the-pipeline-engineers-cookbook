# T39 — Completeness Checks

> **Goal:** Prove that the expected data population was received, retained, and transformed at the expected grain and time coverage.

Completeness is not the same as required-field validation.

- **T33 Required-Field Validation:** Is a field present when a record exists?
- **T39 Completeness Checks:** Did the records, partitions, files, keys, events, or time buckets that should exist actually arrive and survive?

## 1. Problem Recognition

Production pipelines can succeed technically while producing an incomplete dataset.

A job may report `SUCCESS` even when:

- one hourly partition never arrived;
- a source file contained only half of its expected records;
- a watermark advanced past an unprocessed interval;
- an expected customer or account disappeared from the extract;
- a sequence of events contains gaps;
- a multi-file delivery is missing one member;
- a valid zero-activity period is incorrectly treated as missing data;
- late-arriving records have not yet crossed the completeness deadline;
- duplicates make row counts look healthy while unique business entities are missing;
- a transformation silently drops a population.

Completeness asks:

> **What population did we expect, what population did we observe, and what evidence proves the difference is acceptable?**

### 1.1 Completeness dimensions

| Dimension | Question | Example |
|---|---|---|
| Row | Did the expected number of rows arrive? | 10,000 expected, 9,998 observed |
| Partition | Did every expected partition arrive? | 23 of 24 hourly partitions |
| Time | Is every expected time bucket represented? | Missing 14:00–15:00 |
| Key | Did every expected entity/key appear? | 99,800 of 100,000 accounts |
| Event | Did the expected event sequence arrive? | Missing sequence 91822 |
| File | Did every expected delivery member arrive? | 4 of 5 files |
| Batch | Did the complete source batch arrive? | Partial daily extract |
| Column population | Is a metric populated where it should be? | Amount missing for 2% of rows |

Column population is useful, but it must not be confused with dataset completeness.

### 1.2 Empty and missing are different

These states are materially different:

- **Missing partition:** no evidence that the partition was delivered.
- **Empty partition:** evidence exists that the partition was delivered and contains zero valid records.
- **Expected zero:** business logic says zero records is correct.
- **Not yet due:** the completeness deadline has not passed.
- **Late:** data is expected but has not arrived by the SLA.
- **Invalid:** data arrived but failed validation.

Never infer `missing` solely from `row_count = 0`.

### 1.3 The completeness contract

A production completeness check needs:

- expected population;
- expected grain;
- expected time window;
- expected partitions or delivery members;
- source identity;
- arrival deadline;
- allowed lateness;
- acceptable tolerance;
- evidence used to determine completion;
- failure classification;
- recovery action.

Without an expected population, a completeness percentage is only a number, not a correctness check.

## 2. Concept and Reasoning

### 2.1 Expected versus observed

The basic model is:

    expected_population = what the contract says should exist
    observed_population = what the pipeline actually received
    missing_population = expected_population - observed_population

For counts:

    completeness_pct = observed_count / expected_count * 100

Use exact set comparison when identity matters. Use counts only when a count contract is sufficient.

### 2.2 Correct grain

Suppose a source contains:

    account_id | transaction_date | transaction_id

If the contract expects one row per transaction, comparing account counts is insufficient.

The check must operate at the transaction grain:

    expected(transaction_id) = observed(transaction_id)

Always define:

1. source grain;
2. expected grain;
3. observed grain;
4. reconciliation grain.

### 2.3 Row-count completeness

Counts are useful when the source provides a trustworthy expected count.

Example:

    expected_rows = 1,000,000
    observed_rows = 999,700
    missing_rows = 300
    completeness = 99.97%

Do not automatically call 99.97% complete. The acceptable threshold belongs to the contract.

A payment ledger may require 100% completeness while a documented telemetry stream may permit bounded tolerance.

### 2.4 Partition completeness

For partitioned data, create the expected partition set first.

Example hourly expectation:

    2026-09-28 00:00
    2026-09-28 01:00
    ...
    2026-09-28 23:00

Then compare expected partitions with observed partitions.

Missing partition detection is usually more informative than aggregate row-count comparison because missing intervals can be hidden by unusually large later partitions.

### 2.5 Calendar spine

A calendar or time spine explicitly represents every expected time bucket.

Example:

    generate_series(
        '2026-09-28 00:00:00'::timestamptz,
        '2026-09-28 23:00:00'::timestamptz,
        interval '1 hour'
    )

Left join observed data to the spine.

Rows with no observed match represent candidate gaps.

This is preferable to checking only observed timestamps because a missing interval has no row from which to infer its existence.

### 2.6 Key completeness

Sometimes the expected population is a reference set rather than a count.

Example:

    expected_accounts = active_accounts_as_of('2026-09-28')
    observed_accounts = accounts_in_daily_extract('2026-09-28')

Then:

    missing_accounts = expected_accounts EXCEPT observed_accounts

This detects population loss even when the total row count remains unchanged.

### 2.7 Set completeness versus count completeness

Count equality does not prove identity equality.

Example:

    expected IDs:  A, B, C, D
    observed IDs:  A, B, C, E

Both populations contain four rows, but D is missing and E is unexpected.

Production reconciliation should therefore prefer:

- set comparison;
- key-level anti-joins;
- partition comparison;
- sequence validation;
- manifest comparison.

Counts remain valuable as summary metrics.

### 2.8 Manifest-based completeness

A manifest is durable metadata describing an expected delivery.

| Field | Purpose |
|---|---|
| delivery_id | Stable delivery identity |
| source | Source system |
| expected_date | Business/data date |
| expected_members | Number or list of required members |
| expected_rows | Source-declared count when available |
| deadline_at | Completeness deadline |
| observed_members | Received members |
| observed_rows | Loaded rows |
| status | OPEN, COMPLETE, LATE, FAILED |
| rule_version | Completeness contract version |

Manifest state makes the completeness decision durable and auditable.

### 2.9 Watermarks and completeness

A watermark answers where processing has reached. It does not automatically prove that everything before the watermark is complete.

Bad assumption:

    watermark = 10:00
    therefore all data through 10:00 exists

A source can advance its watermark while a partition, file, or late event remains missing.

Use a completeness boundary that combines:

- watermark;
- expected partitions;
- arrival state;
- allowed lateness;
- reconciliation evidence.

### 2.10 Late data

Do not classify recent missing data as permanently incomplete before the allowed lateness window expires.

Example:

    event_time = 09:00
    completeness_deadline = 09:15
    current_time = 09:08

The 09:00 bucket is not yet overdue.

After the deadline, classify it as late or incomplete according to the operational contract.

### 2.11 Tolerance

Possible policies include:

- exact: `missing_count = 0`;
- percentage: `completeness >= 99.9%`;
- absolute: `missing_count <= 100`;
- partition: all critical partitions must exist;
- weighted: critical records require stricter treatment than low-value telemetry.

Tolerance must be explicit and versioned.

Do not hide missing records by choosing a convenient threshold after the failure occurs.

### 2.12 Completeness equations

For count-based checks:

    missing = max(expected - observed, 0)
    excess = max(observed - expected, 0)
    completeness_pct = observed / expected * 100

These are summaries, not proof of identity.

For key populations:

    missing_keys = expected_keys - observed_keys
    unexpected_keys = observed_keys - expected_keys

For partitions:

    missing_partitions = expected_partitions - observed_partitions
    unexpected_partitions = observed_partitions - expected_partitions

### 2.13 Completeness through transformations

Measure population movement at each meaningful boundary.

Example:

    source → raw → staging → validated → curated

Track:

    source_rows = 1,000,000
    raw_rows = 1,000,000
    staging_rows = 1,000,000
    valid_rows = 998,500
    curated_rows = 998,500

If the 1,500 rejected rows are expected and accounted for, this can be complete from a pipeline perspective.

If 20,000 rows disappeared without a disposition, the pipeline is incomplete even though the final table loaded successfully.

### 2.14 Duplicate impact

Duplicates can mask missing records.

Example:

    expected unique keys = 100,000
    observed rows = 100,000
    observed unique keys = 99,500

Count completeness appears perfect while key completeness is only 99.5%.

Pair completeness checks with uniqueness checks from T38 whenever identity matters.

### 2.15 Completeness and required fields

Example:

    100,000 expected transactions
    100,000 transactions received
    2,000 transactions have NULL merchant_id

The dataset may be complete in population while failing a required-field rule.

Conversely:

    98,000 complete records received
    2,000 expected records missing

Every received record may satisfy required fields while the dataset is incomplete.

Use separate quality dimensions.

## 3. Implementation

### 3.1 Durable expectation and result state

Use durable expectation and result tables.

    CREATE TABLE completeness_expectation (
        expectation_id uuid PRIMARY KEY,
        source_name text NOT NULL,
        dataset_name text NOT NULL,
        data_date date NOT NULL,
        expected_grain text NOT NULL,
        expected_rows bigint,
        deadline_at timestamptz NOT NULL,
        tolerance_pct numeric(9,4) NOT NULL DEFAULT 0,
        status text NOT NULL DEFAULT 'OPEN',
        rule_version text NOT NULL,
        created_at timestamptz NOT NULL DEFAULT now()
    );

    CREATE TABLE completeness_result (
        expectation_id uuid PRIMARY KEY REFERENCES completeness_expectation(expectation_id),
        observed_rows bigint NOT NULL,
        missing_rows bigint NOT NULL,
        excess_rows bigint NOT NULL,
        completeness_pct numeric(12,6),
        evaluated_at timestamptz NOT NULL DEFAULT now(),
        status text NOT NULL,
        details jsonb NOT NULL DEFAULT '{}'::jsonb
    );

### 3.2 Expected hourly partitions

Generate the expected population explicitly.

    SELECT bucket
    FROM generate_series(
        '2026-09-28 00:00:00+00'::timestamptz,
        '2026-09-28 23:00:00+00'::timestamptz,
        interval '1 hour'
    ) AS bucket;

Store the expectation or derive it from a governed schedule.

### 3.3 Detect missing partitions

Assume the observed table contains `event_hour`.

    WITH expected AS (
        SELECT bucket AS event_hour
        FROM generate_series(
            '2026-09-28 00:00:00+00'::timestamptz,
            '2026-09-28 23:00:00+00'::timestamptz,
            interval '1 hour'
        ) AS bucket
    ),
    observed AS (
        SELECT DISTINCT event_hour
        FROM staging_events
        WHERE event_hour >= '2026-09-28 00:00:00+00'
          AND event_hour <  '2026-09-29 00:00:00+00'
    )
    SELECT e.event_hour
    FROM expected e
    LEFT JOIN observed o USING (event_hour)
    WHERE o.event_hour IS NULL
    ORDER BY e.event_hour;

This returns missing time buckets, not merely buckets with low row counts.

### 3.4 Count-based completeness

    WITH observed AS (
        SELECT count(*)::bigint AS observed_rows
        FROM staging_events
        WHERE event_date = DATE '2026-09-28'
    )
    SELECT
        expected.expected_rows,
        observed.observed_rows,
        greatest(expected.expected_rows - observed.observed_rows, 0) AS missing_rows,
        greatest(observed.observed_rows - expected.expected_rows, 0) AS excess_rows,
        CASE
            WHEN expected.expected_rows = 0 THEN NULL
            ELSE round(
                observed.observed_rows * 100.0 / expected.expected_rows,
                4
            )
        END AS completeness_pct
    FROM completeness_expectation expected
    CROSS JOIN observed
    WHERE expected.expectation_id = :expectation_id;

Treat zero expected count as an explicit contract state, not as a divide-by-zero error.

### 3.5 Key-level completeness

    SELECT e.account_id
    FROM expected_accounts e
    LEFT JOIN observed_accounts o
      ON o.account_id = e.account_id
    WHERE o.account_id IS NULL;

For multi-tenant data, include tenant identity in the key.

    ON o.tenant_id = e.tenant_id
   AND o.account_id = e.account_id

An incomplete composite key can create false missing or false present results.

### 3.6 Unexpected population

Check the reverse direction as well.

    SELECT o.account_id
    FROM observed_accounts o
    LEFT JOIN expected_accounts e
      ON e.account_id = o.account_id
    WHERE e.account_id IS NULL;

Unexpected records are not necessarily bad. They may indicate:

- source expansion;
- contract change;
- wrong extraction window;
- stale reference data;
- duplicate or replayed delivery;
- tenant or environment leakage.

### 3.7 Manifest-based file completeness

Represent expected members explicitly.

    CREATE TABLE delivery_manifest_member (
        delivery_id uuid NOT NULL,
        member_name text NOT NULL,
        expected boolean NOT NULL DEFAULT true,
        received_at timestamptz,
        observed_rows bigint,
        status text NOT NULL DEFAULT 'EXPECTED',
        PRIMARY KEY (delivery_id, member_name)
    );

Then:

    SELECT member_name
    FROM delivery_manifest_member
    WHERE delivery_id = :delivery_id
      AND expected = true
      AND received_at IS NULL;

Those members are still missing.

### 3.8 Empty versus missing

Record arrival separately from row count.

    received_at IS NOT NULL
    observed_rows = 0

This means:

    delivery received + zero rows

not:

    delivery missing

That distinction prevents false incidents.

### 3.9 Sequence completeness

For a source that promises contiguous integer sequence numbers:

    SELECT
        min(sequence_id) AS min_sequence,
        max(sequence_id) AS max_sequence,
        count(*) AS observed_count,
        max(sequence_id) - min(sequence_id) + 1 AS expected_count
    FROM staging_events
    WHERE batch_id = :batch_id;

If observed count differs from the sequence span, investigate gaps and duplicates.

Do not use this method when the source does not promise contiguous sequences.

### 3.10 Gap detection

With PostgreSQL:

    SELECT gs AS missing_sequence
    FROM generate_series(:min_sequence, :max_sequence) gs
    LEFT JOIN staging_events s
      ON s.sequence_id = gs
    WHERE s.sequence_id IS NULL
    ORDER BY gs;

### 3.11 Population accounting across stages

Create pipeline accounting state or equivalent run metrics.

    CREATE TABLE population_accounting (
        run_id uuid NOT NULL,
        stage_name text NOT NULL,
        input_rows bigint NOT NULL,
        output_rows bigint NOT NULL,
        rejected_rows bigint NOT NULL DEFAULT 0,
        quarantined_rows bigint NOT NULL DEFAULT 0,
        duplicate_rows bigint NOT NULL DEFAULT 0,
        accounted_rows bigint NOT NULL,
        PRIMARY KEY (run_id, stage_name)
    );

The accounting equation should be explicit for each stage.

    input_rows = output_rows
              + rejected_rows
              + quarantined_rows
              + intentionally_removed_rows
              + duplicate_rows

The exact equation depends on the stage contract. Do not blindly apply one universal formula.

### 3.12 Python count checker

    from dataclasses import dataclass
    from decimal import Decimal

    @dataclass(frozen=True)
    class CompletenessResult:
        expected: int
        observed: int
        missing: int
        excess: int
        percentage: Decimal | None
        complete: bool

    def check_count_completeness(
        expected: int,
        observed: int,
        tolerance_pct: Decimal = Decimal('0'),
    ) -> CompletenessResult:
        if expected < 0 or observed < 0:
            raise ValueError('counts must be non-negative')

        missing = max(expected - observed, 0)
        excess = max(observed - expected, 0)

        if expected == 0:
            percentage = None
            complete = observed == 0
        else:
            percentage = (Decimal(observed) * Decimal('100')) / Decimal(expected)
            complete = percentage >= (Decimal('100') - tolerance_pct)

        return CompletenessResult(
            expected=expected,
            observed=observed,
            missing=missing,
            excess=excess,
            percentage=percentage,
            complete=complete,
        )

Keep calculation separate from source-specific extraction logic so it can be unit tested independently.

### 3.13 Python set comparison

    def compare_keys(expected: set[str], observed: set[str]) -> dict[str, set[str]]:
        return {
            'missing': expected - observed,
            'unexpected': observed - expected,
            'matched': expected & observed,
        }

For very large populations, do not blindly materialize millions of keys in application memory. Push comparison into PostgreSQL or use bounded/partitioned processing.

### 3.14 Threshold evaluation

    def within_tolerance(
        expected: int,
        observed: int,
        tolerance_pct: Decimal,
    ) -> bool:
        if expected == 0:
            return observed == 0
        completeness = Decimal(observed) * Decimal('100') / Decimal(expected)
        return completeness >= Decimal('100') - tolerance_pct

Store the applied threshold with the result.

### 3.15 Version the rule

Persist a rule version such as:

    rule_version = 'completeness-v3'

If expected-population logic changes, historical results should remain explainable under the rule that produced them.

### 3.16 Preserve evidence

A completeness result should point to evidence such as:

- delivery ID;
- source batch ID;
- partition range;
- expected-count source;
- observed-count query/window;
- missing keys or durable exception table;
- watermark;
- evaluation timestamp;
- rule version.

Do not rely on a transient log line as the only evidence.

## 4. Testing

Completeness tests must cover correct data and misleadingly healthy data.

### 4.1 Count tests

Test:

- expected equals observed;
- observed below expected;
- observed above expected;
- expected zero and observed zero;
- expected zero and observed above zero;
- negative inputs rejected;
- exact threshold boundary;
- tolerance just inside boundary;
- tolerance just outside boundary.

### 4.2 Partition tests

Test:

- all expected partitions present;
- one partition missing;
- several partitions missing;
- unexpected partition present;
- partition present with zero rows;
- partition late but within allowed lateness;
- partition late beyond deadline.

### 4.3 Key tests

Test:

- identical expected and observed sets;
- one missing key;
- one unexpected key;
- duplicate observed key;
- NULL key when NULL is prohibited;
- composite-key mismatch;
- tenant-scoped key collision.

### 4.4 Sequence tests

Test:

- contiguous sequence;
- gap in the middle;
- missing first sequence;
- missing final sequence;
- duplicate sequence;
- non-contiguous source contract rejected.

### 4.5 Transformation accounting tests

Construct:

    input = 100
    valid = 92
    rejected = 5
    quarantined = 3

Assert that the accounting equation balances.

Also test an intentionally unaccounted record and ensure the check fails.

### 4.6 Idempotence tests

Run the completeness check twice.

The second evaluation should:

- produce the same result for unchanged evidence;
- not create duplicate incidents;
- not mutate source data;
- retain clear evaluation history when audit history is required.

### 4.7 False-confidence tests

These are especially important:

- expected count equals observed count but keys differ;
- row count is zero but the file was delivered;
- watermark advanced while one partition is missing;
- duplicate rows compensate for missing unique keys;
- late data is evaluated before its deadline.

These cases prove why completeness cannot be reduced to one count.

## 5. Observability

Completeness checks should produce operational signals, not only pass/fail logs.

### 5.1 Core metrics

| Metric | Meaning |
|---|---|
| `completeness_expected_rows` | Contracted population |
| `completeness_observed_rows` | Received population |
| `completeness_missing_rows` | Count shortfall |
| `completeness_excess_rows` | Unexpected count surplus |
| `completeness_pct` | Count-based coverage |
| `completeness_missing_partitions` | Number of absent expected buckets |
| `completeness_missing_keys` | Expected identities not observed |
| `completeness_unexpected_keys` | Observed identities outside expectation |
| `completeness_lag_seconds` | Time until/after completeness boundary |
| `completeness_failures_total` | Failed evaluations |

### 5.2 Useful dimensions

Use:

- source;
- dataset;
- environment;
- tenant where appropriate;
- data date;
- partition;
- rule version;
- delivery ID.

Do not attach high-cardinality raw identifiers to general metrics. Put detailed missing identities in structured diagnostic storage.

### 5.3 Dashboard questions

A production dashboard should answer:

1. Which datasets are incomplete?
2. Which partitions are missing?
3. How many rows or keys are missing?
4. Is the dataset merely late?
5. When did the completeness deadline pass?
6. Is the issue isolated to one source or widespread?
7. Did completeness degrade after a deployment?
8. Are missing records accumulating?

### 5.4 Alerts

Alert on actionable states:

- critical dataset incomplete after deadline;
- missing partition after SLA;
- completeness below contract threshold;
- unexpected population spike;
- manifest member missing;
- watermark advanced without complete coverage.

Avoid alerting on every temporary pre-deadline gap.

## 6. Intentional Failure

Break the pipeline deliberately to prove the completeness mechanism works.

### Failure 1 — Remove one expected partition

Withhold one partition.

Expected:

    missing_partitions = 1
    status = INCOMPLETE

### Failure 2 — Reduce a file by 10%

Keep the file present but remove records.

Expected:

    file_arrival = COMPLETE
    row_completeness = INCOMPLETE

This proves arrival and content completeness are separate checks.

### Failure 3 — Replace one key

Remove expected key `A100` and add unexpected key `A900`.

Expected count remains unchanged. The key-level check must still fail.

### Failure 4 — Advance the watermark incorrectly

Set the checkpoint beyond a deliberately missing partition.

The completeness layer should detect the population gap instead of trusting the watermark.

### Failure 5 — Create duplicate compensation

Remove 100 unique records and duplicate another 100.

Observed row count stays unchanged.

Count completeness may pass, but uniqueness/key completeness must expose the defect.

### Failure 6 — Deliver an empty file

Register the file as received and give it zero rows.

The system must classify it according to the source contract rather than automatically treating it as missing.

### Failure 7 — Evaluate before the deadline

Introduce a late partition but evaluate before its allowed lateness expires.

Expected:

    status = NOT_YET_DUE

Do not create a permanent missing-data incident prematurely.

## 7. Recovery

Recovery depends on the failure classification.

### 7.1 Missing partition

1. Identify the exact partition.
2. Confirm whether the source generated it.
3. Check delivery/manifest state.
4. Re-extract only the missing interval where possible.
5. Load through an idempotent boundary.
6. Re-run completeness.
7. Record recovery evidence.

### 7.2 Partial file

1. Preserve the original artifact.
2. Compare source-declared and observed counts.
3. Determine whether the source can regenerate the file.
4. Replace or replay using a stable delivery identity.
5. Reconcile before publication.

Do not silently append a replacement file to a partially loaded file.

### 7.3 Missing keys

1. Materialize the missing-key set.
2. Determine whether the reference population is correct.
3. Query source availability for those keys.
4. Re-extract or replay missing records.
5. Re-run key-level completeness.
6. Investigate persistent omissions.

### 7.4 Late data

Do not treat every late record as corruption.

Use:

    late → accepted within policy → replay/recompute

or:

    late → beyond policy → incident + controlled backfill

### 7.5 Unexpected records

Unexpected population can indicate valid source expansion.

Before deleting anything, determine whether the expectation was stale.

Possible recovery:

- update the contract;
- isolate unexpected records;
- correct extraction scope;
- fix tenant filtering;
- deduplicate replayed data.

### 7.6 Safe replay

Replay must preserve:

- delivery identity;
- partition identity;
- business key;
- source lineage;
- idempotency key;
- completeness evidence.

After replay, run completeness and uniqueness checks again.

### 7.7 Recompute derived data

If missing data affected downstream aggregates:

1. identify affected partitions;
2. invalidate or mark stale derived results;
3. replay the corrected source range;
4. recompute downstream transformations;
5. reconcile before reopening the dataset.

Do not patch a final aggregate manually when the source-level gap can be corrected and replayed.

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

Use PostgreSQL for:

- manifests;
- expectation/result state;
- calendar spines;
- anti-joins;
- set comparisons;
- sequence-gap detection;
- durable completeness evidence.

The important mechanism remains the explicit expected-versus-observed comparison.

### 8.2 dbt

dbt can operationalize data tests and expose completeness failures as part of transformation workflows.

Use it when completeness belongs naturally beside model-level tests and warehouse transformations.

A successful model test does not prove that an expected upstream partition ever arrived.

### 8.3 Great Expectations

Great Expectations can express expectation-style checks around row counts, distributions, and data conditions.

It is useful when a team wants declarative validation and reporting.

The underlying concepts remain:

    expectation → observation → evaluation → evidence → action

## 9. Production Runbook

### Alert

1. Identify dataset and delivery ID.
2. Check current completeness status.
3. Check deadline and allowed lateness.
4. Identify missing partitions, members, rows, or keys.

### Diagnose

5. Confirm expected-population source.
6. Verify manifest and watermark state.
7. Compare expected and observed keys when identity matters.
8. Check duplicates and unexpected records.
9. Determine whether the issue is missing, late, invalid, or incorrectly expected data.

### Recover

10. Re-extract the smallest safe missing scope.
11. Preserve original artifacts.
12. Load idempotently.
13. Re-run completeness.
14. Reconcile downstream affected data.

### Close

15. Record root cause.
16. Record rule version and evidence.
17. Confirm no unexplained population remains.
18. Confirm alerts return to normal.
19. Update the source contract if the expectation itself was wrong.

### Incident decision table

| Observation | Classification | Action |
|---|---|---|
| Missing partition before deadline | Not yet due | Wait/monitor |
| Missing partition after deadline | Incomplete/late | Investigate and recover |
| File received with zero valid rows | Contract-dependent | Validate content/expected-zero semantics |
| Counts equal but keys differ | Incomplete | Perform key-level reconciliation |
| Unexpected keys only | Contract drift or scope error | Investigate before deletion |
| Watermark advanced with gap | State inconsistency | Correct watermark and replay |
| Missing rows accounted as rejected | Complete population, lower validity | Follow DQ policy |

## 10. Common Mistakes

### Mistake 1 — Using row count as the only completeness check

Equal counts do not imply equal populations.

### Mistake 2 — Treating zero rows as missing

An arrived empty partition can be valid.

### Mistake 3 — Trusting the watermark

A watermark is processing state, not proof of source completeness.

### Mistake 4 — Ignoring late-arrival windows

Evaluate completeness against the declared deadline.

### Mistake 5 — Checking the wrong grain

Account-level completeness cannot prove transaction-level completeness.

### Mistake 6 — Ignoring duplicates

Duplicate records can compensate for missing unique entities.

### Mistake 7 — Hard-coding today's expected population

Expected populations should come from explicit contracts, calendars, manifests, or governed reference data.

### Mistake 8 — Hiding missing records behind tolerance

Tolerance is a policy, not a mechanism for making unexplained loss disappear.

### Mistake 9 — Overwriting evidence during recovery

Keep original artifacts and evaluation history so the incident remains reconstructable.

### Mistake 10 — Mixing completeness with validity

A complete dataset can contain invalid records, and a valid subset can still be incomplete.

### Mistake 11 — Creating incidents before the deadline

Temporary lateness should not automatically become a permanent missing-data incident.

### Mistake 12 — Treating unexpected data as automatically corrupt

Unexpected population may indicate legitimate source evolution or a stale contract.

## 11. Definition of Done

You are done with T39 when you can:

- define completeness independently from field validation;
- state the expected population for a dataset;
- define the correct comparison grain;
- distinguish missing, empty, late, invalid, and not-yet-due data;
- calculate count-based completeness safely;
- compare expected and observed keys;
- detect missing partitions with a calendar spine;
- use manifests for multi-file deliveries;
- reason about watermarks without treating them as completeness proof;
- account for duplicates when measuring completeness;
- apply explicit tolerances;
- preserve rule versions and evidence;
- implement the checks in SQL;
- implement core calculations in Python;
- write tests for normal, boundary, and deceptive cases;
- emit actionable metrics and alerts;
- intentionally create a completeness failure;
- diagnose it from evidence;
- recover the smallest safe scope;
- replay idempotently;
- reconcile downstream effects;
- explain PostgreSQL, dbt, and Great Expectations;
- operate the mechanism using the runbook.

## 12. What You Learned

Completeness is a population contract, not a single percentage.

The core mental model is:

    EXPECTED
       ↓
    OBSERVED
       ↓
    COMPARE
       ↓
    CLASSIFY
       ↓
    RECORD EVIDENCE
       ↓
    RECOVER
       ↓
    RECHECK

You learned to:

- define what should exist before checking what did exist;
- compare data at the correct grain;
- use counts for summary and identities for proof;
- represent expected time buckets explicitly;
- distinguish empty from missing;
- handle late data using deadlines and watermarks carefully;
- detect missing keys even when row counts look healthy;
- preserve completeness evidence and rule versions;
- connect completeness with uniqueness, validation, and reconciliation;
- recover missing populations without corrupting existing data.

The production principle is simple:

> **A successful pipeline run is not evidence of complete data. Completeness must be explicitly defined, measured, evidenced, and rechecked.**

### Next Recipe

**T40 — Consistency Checks**

T40 will build on completeness by asking a different question: **when data exists, do related values, records, aggregates, and states agree with each other?**
