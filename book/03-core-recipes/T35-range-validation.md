# T35 — Range Validation

> **Goal:** Learn how to prove that correctly typed values fall inside the valid operational or business range, including boundary semantics, cross-field ranges, temporal limits, and safe handling of violations.

## 1. Problem Recognition

Type validation answers whether a value can be represented as the required type.

Range validation answers whether that typed value is allowed within the contract.

Examples:

    age = 37
    type = INTEGER
    range = 0..120

    amount = 125.50
    type = NUMERIC
    range = >= 0

    event_time = 2026-09-28T10:00:00Z
    type = TIMESTAMPTZ
    range = ingestion_time - 7 days .. ingestion_time + 5 minutes

A value can be correctly typed and still be invalid.

### Common range failures

- Negative amount where only non-negative amounts are allowed.
- Percentage greater than 100.
- Quantity below zero.
- Date outside the supported reporting period.
- Timestamp too far in the future.
- Transaction amount above an approved limit.
- Coordinate outside geographic bounds.
- Score outside its defined scale.
- Start date after end date.
- Minimum greater than maximum.

### Recognition questions

Before implementing range validation, ask:

1. What field is being bounded?
2. What is the lower bound?
3. What is the upper bound?
4. Are bounds inclusive or exclusive?
5. Are bounds static or dynamic?
6. What reference timestamp defines the range?
7. Is the range a technical limit or a business rule?
8. Are NULL values allowed?
9. Are bounds dependent on another field?
10. What happens when the value is outside the range?
11. Should the record be quarantined or rejected?
12. How is the rule versioned?

## 2. Concept and Reasoning

### 2.1 Range is a contract

A range rule should be explicit.

Example:

    amount >= 0

means zero is valid.

Example:

    amount > 0

means zero is invalid.

Those rules are not interchangeable.

### 2.2 Inclusive versus exclusive bounds

Four common forms are:

    x >= min AND x <= max
    x >  min AND x <  max
    x >= min AND x <  max
    x >  min AND x <= max

Boundary semantics must be documented.

### 2.3 Closed, open, and half-open intervals

Closed:

    [min, max]

Open:

    (min, max)

Half-open:

    [min, max)

Half-open intervals are especially useful for time windows because adjacent periods do not overlap.

### 2.4 Why boundary testing matters

Most range bugs occur around boundaries.

For a rule:

    0 <= amount <= 100

test:

    -0.01
     0
     0.01
    99.99
    100
    100.01

Do not test only an obviously valid and obviously invalid value.

### 2.5 Static versus dynamic ranges

Static:

    percentage between 0 and 100

Dynamic:

    event_time <= current_time + 5 minutes

Dynamic ranges require a clearly defined reference.

### 2.6 Business ranges versus physical ranges

A database type may support:

    INTEGER = large numeric range

while the business rule may allow:

    quantity between 0 and 100000

Do not confuse technical capacity with business validity.

### 2.7 Range validation after type validation

Preferred sequence:

    raw value
       ↓
    type validation
       ↓
    typed value
       ↓
    range validation
       ↓
    trusted value

Do not compare numeric values as strings.

### 2.8 String ranges are different

Lexical comparisons can be valid for some contracts but should not be confused with numeric ranges.

Example:

    '100' < '20'

may be true lexically.

Numeric range validation should use a numeric type.

### 2.9 Integer ranges

Example:

    quantity >= 0

may represent a count.

Also consider an upper business limit:

    quantity <= 100000

### 2.10 Decimal ranges

Example:

    amount >= 0

and:

    amount <= 1000000.00

Validate after exact decimal parsing.

### 2.11 Percentage ranges

A percentage may use:

    0..100

while a ratio may use:

    0..1

Do not mix the two representations.

### 2.12 Probability ranges

Probability is usually represented as:

    0 <= p <= 1

Values such as 1.2 are technically numeric but invalid probabilities.

### 2.13 Date ranges

Example:

    business_date >= DATE '2020-01-01'

Date rules may also have a maximum:

    business_date <= DATE '2099-12-31'

### 2.14 Timestamp ranges

Timestamp validation often needs a reference event.

Example:

    event_time <= ingestion_time + interval '5 minutes'

The correct reference is part of the contract.

### 2.15 Future timestamp tolerance

Small clock differences are normal.

A rule might allow:

    event_time <= ingestion_time + 5 minutes

rather than requiring:

    event_time <= ingestion_time

### 2.16 Historical timestamp tolerance

Events may arrive late.

Instead of rejecting all old events, define a supported lateness window:

    event_time >= ingestion_time - interval '30 days'

Anything older can be quarantined for separate historical processing.

### 2.17 Start and end dates

A common cross-field rule is:

    start_date <= end_date

Both fields may individually have valid ranges while the record remains invalid.

### 2.18 Duration ranges

Example:

    duration_seconds >= 0

and:

    duration_seconds <= 86400

Duration should be validated as a duration or numeric quantity, not inferred from formatted text.

### 2.19 Age ranges

Age might require:

    0 <= age <= 120

But age can also be derived from birth date.

When both exist, decide which is authoritative and validate their consistency separately.

### 2.20 Coordinate ranges

Latitude:

    -90 <= latitude <= 90

Longitude:

    -180 <= longitude <= 180

Correct numeric typing alone does not guarantee geographic validity.

### 2.21 Financial limits

Financial ranges can be:

- Product-specific.
- Currency-specific.
- Customer-specific.
- Risk-tier-specific.
- Time-dependent.

These should be treated as versioned business rules.

### 2.22 Currency-specific ranges

A maximum amount may differ by currency.

Example:

    EUR limit = 10000
    USD limit = 12000

Do not encode such rules as a universal constant when the business rule depends on reference data.

### 2.23 Reference-data ranges

Dynamic ranges can come from a reference table:

    product_limits

with:

    product_id
    min_amount
    max_amount
    effective_from
    effective_to

Validation then becomes a temporal lookup plus comparison.

### 2.24 Effective-dated ranges

A rule may change over time.

Validate against the rule effective when the business event occurred, not automatically the rule effective today.

### 2.25 Cross-field range rules

Examples:

    start <= end
    min <= max
    used <= limit
    debit <= available_balance

These are record-level constraints.

### 2.26 Range consistency

Individual values may be valid while their relationship is invalid.

Example:

    min_price = 100
    max_price = 50

Both values are numeric.

The record is still invalid.

### 2.27 Range validation and NULL

SQL comparisons involving NULL do not produce TRUE.

Define whether:

    NULL

means:

    not applicable
    unknown
    missing
    invalid

Required-field validation handles mandatory presence; range validation handles bounds for present values.

### 2.28 Range validation and NaN/infinity

Floating-point data may contain:

    NaN
    +Infinity
    -Infinity

These values require explicit policy.

Do not assume ordinary numeric comparisons handle them as business values.

### 2.29 Precision at boundaries

Decimal boundaries should use exact arithmetic.

Example:

    limit = 100.00
    amount = 100.0001

The result must not depend on floating-point representation.

### 2.30 Tolerance rules

Some measurements require tolerance:

    abs(measured - expected) <= tolerance

Tolerance must be documented rather than introduced to make failures disappear.

### 2.31 Range validation versus anomaly detection

Range validation asks:

    Is this value inside a defined acceptable interval?

Anomaly detection asks:

    Is this value unusual compared with expected behavior?

A value can be inside a valid range and still be anomalous.

### 2.32 Range validation versus domain validation

Range:

    score between 0 and 100

Domain:

    status IN ('pending', 'approved', 'rejected')

Keep numeric or temporal boundaries separate from enumerated domain membership.

### 2.33 Range validation and aggregation

Validate at the correct grain.

A daily aggregate may have a valid range even when individual source events contain invalid values.

Usually validate source values before aggregation when invalid inputs could contaminate the result.

### 2.34 Range validation and joins

Reference-driven limits often require joins.

Validate the join itself before trusting the resulting bounds.

A missing limit must not accidentally become an unlimited record.

### 2.35 Range validation and partitioning

Partition values can have operational limits.

Example:

    partition_date cannot be more than 1 day in the future

This can prevent accidental creation of incorrect future partitions.

### 2.36 Range rule versioning

Record:

    rule_id
    rule_version
    effective_from
    effective_to

This makes historical validation reproducible.

### 2.37 Fail-open versus fail-closed

If the range rule cannot be loaded, decide whether the pipeline should:

    stop
    quarantine
    use a documented fallback

Do not silently treat a missing rule as no limit.

## 3. Implementation

### 3.1 Define the range contract

Example:

    quantity: INTEGER
    minimum: 0
    maximum: 100000
    bounds: inclusive

    amount: NUMERIC(20,4)
    minimum: 0
    maximum: 1000000.0000
    bounds: inclusive

    event_time: TIMESTAMPTZ
    lower: ingestion_time - 30 days
    upper: ingestion_time + 5 minutes

### 3.2 PostgreSQL numeric range

    SELECT *
    FROM typed_payments
    WHERE amount >= 0
      AND amount <= 1000000.0000;

This returns valid records when NULL handling has already been defined.

### 3.3 Classify range failures

Instead of filtering silently:

    CASE
        WHEN amount < 0 THEN 'BELOW_MIN'
        WHEN amount > 1000000.0000 THEN 'ABOVE_MAX'
        ELSE NULL
    END AS range_failure_code

This preserves diagnostic information.

### 3.4 Inclusive boundary rule

    amount BETWEEN 0 AND 1000000.0000

`BETWEEN` is inclusive at both ends in PostgreSQL.

Use explicit comparisons when readability of the contract matters:

    amount >= 0
    AND amount <= 1000000.0000

### 3.5 Exclusive boundary rule

Example:

    amount > 0
    AND amount < 1000000.0000

Document why the endpoints are excluded.

### 3.6 Half-open time interval

Use:

    event_time >= window_start
    AND event_time < window_end

This avoids overlap between adjacent time windows.

### 3.7 Cross-field validation

    CASE
        WHEN start_date > end_date THEN 'START_AFTER_END'
        ELSE NULL
    END AS range_failure_code

### 3.8 Dynamic timestamp range

Example:

    CASE
        WHEN event_time < ingestion_time - INTERVAL '30 days'
            THEN 'TOO_OLD'
        WHEN event_time > ingestion_time + INTERVAL '5 minutes'
            THEN 'TOO_FAR_IN_FUTURE'
        ELSE NULL
    END AS range_failure_code

### 3.9 Reference-table range validation

Example reference table:

    CREATE TABLE product_limits (
        product_id TEXT NOT NULL,
        min_amount NUMERIC(20,4) NOT NULL,
        max_amount NUMERIC(20,4) NOT NULL,
        effective_from TIMESTAMPTZ NOT NULL,
        effective_to TIMESTAMPTZ,
        PRIMARY KEY (product_id, effective_from)
    );

Join each transaction to the rule effective at its event time.

### 3.10 Effective-dated lookup

    SELECT p.record_id, p.amount, l.min_amount, l.max_amount
    FROM typed_payments p
    JOIN product_limits l
      ON l.product_id = p.product_id
     AND p.event_time >= l.effective_from
     AND (l.effective_to IS NULL OR p.event_time < l.effective_to);

Use half-open effective intervals to prevent overlapping rules.

### 3.11 Detect missing reference rules

Use a LEFT JOIN when you need to identify records without a matching rule.

Do not let missing reference data turn into a false range pass.

### 3.12 Validate reference-table quality

Reference rules themselves need validation:

    min_amount <= max_amount
    no overlapping effective intervals
    no duplicate active rule

### 3.13 Python range validator

    from decimal import Decimal

    def validate_amount(value, minimum, maximum):
        if value is None:
            return 'NULL'

        if value < minimum:
            return 'BELOW_MIN'

        if value > maximum:
            return 'ABOVE_MAX'

        return None

Keep the return code separate from the validated value.

### 3.14 Cross-field Python validation

    def validate_interval(start, end):
        if start is None or end is None:
            return None
        if start > end:
            return 'START_AFTER_END'
        return None

### 3.15 Decimal boundaries in Python

    minimum = Decimal('0.00')
    maximum = Decimal('1000000.00')

Use `Decimal` for exact monetary comparisons.

### 3.16 Timestamp tolerance in Python

    from datetime import timedelta

    lower = ingestion_time - timedelta(days=30)
    upper = ingestion_time + timedelta(minutes=5)

Then compare timezone-aware timestamps consistently.

### 3.17 Vectorized validation

For large batches, avoid row-by-row Python calls where the processing engine can evaluate the predicate vectorially.

Conceptually:

    valid = (df['amount'] >= 0) & (df['amount'] <= 1000000)
    invalid = ~valid

Make NULL policy explicit because boolean masks can have missing values.

### 3.18 Validation result table

Create a structured failure record:

    CREATE TABLE range_validation_failure (
        batch_id TEXT NOT NULL,
        record_id TEXT NOT NULL,
        field_name TEXT NOT NULL,
        failure_code TEXT NOT NULL,
        observed_type TEXT,
        rule_id TEXT NOT NULL,
        rule_version TEXT NOT NULL,
        detected_at TIMESTAMPTZ NOT NULL
    );

Do not store sensitive raw values unless required and authorized.

### 3.19 Valid and invalid outputs

Conceptually:

    typed records
          ↓
    range validation
       ├── valid → trusted staging
       └── invalid → quarantine

### 3.20 Target constraints

Where practical, encode stable invariants in the database:

    CHECK (amount >= 0)

Database constraints provide a final protection layer.

Application-level validation should not be the only control.

### 3.21 Reconciliation

Track:

    input_rows = valid_rows + invalid_rows

and separately:

    invalid_rows = below_min + above_max + cross_field_failures + other_failures

when failure categories are mutually exclusive.

### 3.22 Avoid silent filtering

Bad:

    SELECT * FROM typed_payments WHERE amount BETWEEN 0 AND 1000;

when the rejected rows disappear without accounting.

Better:

    classify
    count
    quarantine
    publish valid rows

### 3.23 Rule configuration

Keep rules versioned and auditable.

Example:

    rule_id = 'payment_amount_limit'
    rule_version = '2026-09-01'

Do not bury frequently changing business limits across application code.

## 4. Testing

Range tests should concentrate on boundaries and failure classification.

### 4.1 Lower boundary

For `[0, 100]`, test:

    0

Expected:

    valid

### 4.2 Just below lower boundary

Test:

    -0.01

Expected:

    BELOW_MIN

### 4.3 Upper boundary

Test:

    100

Expected:

    valid

### 4.4 Just above upper boundary

Test:

    100.01

Expected:

    ABOVE_MAX

### 4.5 Exclusive lower boundary

For `(0, 100]`, test zero explicitly.

Expected:

    invalid

### 4.6 Exclusive upper boundary

For `[0, 100)`, test 100 explicitly.

Expected:

    invalid

### 4.7 NULL

Test NULL separately.

Verify that the result follows the nullability contract rather than being mislabeled as below or above range.

### 4.8 Negative zero

Test numeric representations such as:

    -0

and verify they follow the exact numeric semantics required by the system.

### 4.9 Decimal precision

Test values immediately around the boundary:

    100.00
    100.0001

using exact decimal arithmetic.

### 4.10 Integer boundaries

Test:

    minimum - 1
    minimum
    minimum + 1
    maximum - 1
    maximum
    maximum + 1

### 4.11 Timestamp lower boundary

Test exactly at:

    ingestion_time - 30 days

and just outside it.

### 4.12 Timestamp upper boundary

Test exactly at:

    ingestion_time + 5 minutes

and just beyond it.

### 4.13 Half-open interval

For `[start, end)`, verify:

    start → valid
    end   → invalid

### 4.14 Cross-field range

Test:

    start < end
    start = end
    start > end

and verify the contract.

### 4.15 Reference-rule lookup

Test:

- Matching rule.
- No matching rule.
- Multiple matching rules.
- Overlapping effective rules.

Missing or ambiguous reference rules should not silently pass.

### 4.16 Currency-specific limits

Verify that the correct currency rule is selected.

Test a value valid in one currency but invalid in another.

### 4.17 Effective-dated rules

Test records immediately before, at, and after a rule change.

Verify the event is validated against the correct effective version.

### 4.18 Rule version reproducibility

Run the same historical record under its historical rule version.

Expected:

    deterministic result

even if the current rule has changed.

### 4.19 Missing reference data

Remove the limit row.

Expected:

    explicit reference-data failure

not an unlimited pass.

### 4.20 Idempotence

Run the same batch twice.

Expected:

    identical valid population
    identical invalid population
    no duplicate quarantine records

### 4.21 Reconciliation

Verify:

    input = valid + invalid

and verify failure categories reconcile to invalid records.

### 4.22 Property-based boundary testing

For configurable bounds, generate values around each boundary rather than testing only fixed examples.

Test:

    boundary - epsilon
    boundary
    boundary + epsilon

where epsilon matches the field precision.

## 5. Observability

### Core metrics

| Metric | Meaning |
|---|---|
| `range_validation_input_rows` | Records entering range validation |
| `range_validation_valid_rows` | Records inside permitted ranges |
| `range_validation_invalid_rows` | Records violating range rules |
| `range_validation_below_min` | Values below lower bounds |
| `range_validation_above_max` | Values above upper bounds |
| `range_validation_cross_field_failures` | Relationship violations |
| `range_validation_missing_rule` | Records without applicable reference rules |
| `range_validation_ambiguous_rule` | Records matching multiple rules |
| `range_validation_rule_version` | Active rule version |

### Failure distribution

Break failures down by:

- Field.
- Source.
- Product.
- Currency.
- Rule.
- Rule version.
- Batch.
- Effective date.

### Boundary concentration

Monitor values close to limits.

A sudden concentration immediately below a maximum can indicate:

- Producer truncation.
- Unit conversion errors.
- Gaming of a business threshold.
- Changed source behavior.

Do not automatically classify boundary concentration as invalid; investigate it as a signal.

### Dynamic-rule health

Monitor reference rules for:

- Missing ranges.
- Overlapping ranges.
- Gaps.
- Invalid min/max relationships.
- Expired rules.

### Quarantine backlog

Track:

- Invalid record count.
- Oldest failure.
- Failure rate.
- Retry count.
- Unresolved rule version.

### Drift alerts

Alert when the invalid rate changes materially from its historical baseline.

Use field-level breakdowns to identify the source of the change.

## 6. Intentional Failure

### Failure 1 — Off-by-one boundary

Implement:

    amount > 0

when the contract requires:

    amount >= 0

Expected symptom:

- Valid zero-value records are rejected.

Recovery:

Correct boundary semantics and add explicit zero tests.

### Failure 2 — String comparison

Compare numeric values before type conversion.

Expected symptom:

- Lexical ordering produces incorrect range decisions.

Recovery:

Validate type first, then compare typed values.

### Failure 3 — Floating-point boundary

Use binary floating-point for an exact monetary threshold.

Expected symptom:

- Values near the boundary behave unexpectedly.

Recovery:

Use exact decimal arithmetic.

### Failure 4 — Missing reference rule treated as unlimited

Allow a failed lookup to produce NULL limits and skip validation.

Expected symptom:

- Records without business limits pass.

Recovery:

Classify missing rules explicitly and fail closed where required.

### Failure 5 — Overlapping effective rules

Create two active rules covering the same event time.

Expected symptom:

- One transaction matches multiple limits.

Recovery:

Quarantine ambiguous matches and repair reference data.

### Failure 6 — Wrong rule version

Validate historical events using today's limit.

Expected symptom:

- Historical results change when business rules change.

Recovery:

Use effective-dated rule selection.

### Failure 7 — Future timestamp rejection

Reject every event later than ingestion time.

Expected symptom:

- Small clock-skew differences create false failures.

Recovery:

Define a documented future tolerance.

### Failure 8 — Late-event rejection

Reject every event older than the ingestion batch.

Expected symptom:

- Legitimate late-arriving events disappear.

Recovery:

Define a supported lateness window and separate genuinely stale events.

### Failure 9 — Silent filtering

Apply a WHERE predicate and discard invalid rows without accounting.

Expected symptom:

- Output looks correct while input/output counts no longer reconcile.

Recovery:

Classify and quarantine invalid rows.

## 7. Recovery

### Recovery sequence

1. Preserve the raw input.
2. Identify the failing rule.
3. Confirm boundary semantics.
4. Check the rule version.
5. Check reference-data completeness.
6. Determine whether the source or rule changed.
7. Correct the rule or source mapping.
8. Revalidate affected records.
9. Reconcile valid and invalid populations.
10. Replay only the affected scope.
11. Record the root cause.
12. Add a regression test.

### Recovering from a bad static limit

1. Identify batches validated with the incorrect limit.
2. Identify the correct effective rule.
3. Reprocess preserved raw or typed data.
4. Rebuild affected outputs.
5. Reconcile downstream aggregates.

### Recovering from a reference-data outage

Do not invent a limit.

Depending on the contract:

    stop publication
    quarantine affected records
    retry reference lookup

Once the rule data is restored, replay the affected scope.

### Recovering from boundary-rule changes

Version the new rule.

Do not rewrite historical validation results unless the business explicitly requires restatement.

### Recovering from timezone errors

Re-evaluate timestamp ranges using the correct instant and timezone semantics.

Reprocess only affected time partitions when the impact scope is known.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL provides CHECK constraints, numeric types, date/time types, range operators, and strong comparison semantics.

Understand the underlying SQL predicates before depending on framework validation.

### 2. dbt

dbt can encode range tests and expose failures as part of model validation.

Use tests as executable data contracts.

### 3. Great Expectations

Great Expectations provides expectations for minimums, maximums, date ranges, and configurable validation outcomes.

Use it to operationalize explicit range rules rather than hiding business semantics inside generic expectations.

## 9. Production Runbook

### Before deployment

- [ ] Define minimum values.
- [ ] Define maximum values.
- [ ] Define inclusive/exclusive semantics.
- [ ] Define NULL behavior.
- [ ] Define decimal precision.
- [ ] Define timestamp tolerance.
- [ ] Define cross-field constraints.
- [ ] Define dynamic reference rules.
- [ ] Define effective dates.
- [ ] Define missing-rule behavior.
- [ ] Define quarantine behavior.
- [ ] Version every business rule.

### During execution

- [ ] Record input count.
- [ ] Record valid count.
- [ ] Record invalid count.
- [ ] Record below-min failures.
- [ ] Record above-max failures.
- [ ] Record cross-field failures.
- [ ] Record missing-rule failures.
- [ ] Record rule version.
- [ ] Reconcile all populations.

### If invalid rate spikes

1. Identify the failing field.
2. Identify the rule version.
3. Compare source distribution.
4. Check recent producer changes.
5. Check reference-data changes.
6. Check units and currency.
7. Check timezone assumptions.
8. Determine whether the threshold or source changed.

### If a threshold changes

1. Create a new rule version.
2. Define effective time.
3. Validate reference-data integrity.
4. Test boundary values.
5. Deploy.
6. Monitor the first affected batches.

### If reference data is missing

1. Stop or quarantine according to policy.
2. Repair reference data.
3. Verify rule coverage.
4. Replay affected records.
5. Reconcile.

## 10. Common Mistakes

### Mistake 1 — Off-by-one errors

Confusing `>` with `>=` changes valid populations.

### Mistake 2 — No boundary tests

Testing only normal values misses the highest-risk cases.

### Mistake 3 — Comparing strings

Numeric and temporal ranges require typed values.

### Mistake 4 — Floating-point monetary comparisons

Use exact decimal semantics for financial boundaries.

### Mistake 5 — Treating NULL as outside range

NULL needs explicit nullability semantics.

### Mistake 6 — Static constants for dynamic rules

Business limits often depend on product, currency, customer, or effective date.

### Mistake 7 — Missing reference data equals no limit

A failed lookup must not silently broaden the valid population.

### Mistake 8 — Ignoring rule version

Changing a rule can make historical validation non-reproducible.

### Mistake 9 — Rejecting all late events

Event-time systems must account for legitimate lateness.

### Mistake 10 — Rejecting all future timestamps

Small clock differences are normal; define a tolerance.

### Mistake 11 — Silent filtering

Discarded rows must remain accounted for.

### Mistake 12 — Mixing range and anomaly detection

Unusual does not necessarily mean invalid.

## 11. Definition of Done

The range-validation stage is complete when you can:

- [ ] Define lower and upper bounds.
- [ ] Define inclusive and exclusive boundaries.
- [ ] Use closed and half-open intervals correctly.
- [ ] Test values immediately around boundaries.
- [ ] Validate integers and decimals.
- [ ] Validate percentages and probabilities.
- [ ] Validate dates and timestamps.
- [ ] Define future and historical timestamp tolerances.
- [ ] Validate cross-field ranges.
- [ ] Validate dynamic reference-driven ranges.
- [ ] Handle currency-specific and product-specific limits.
- [ ] Use effective-dated business rules.
- [ ] Detect missing and overlapping reference rules.
- [ ] Separate NULL handling from range failures.
- [ ] Handle exact decimal boundaries.
- [ ] Classify failures rather than silently filtering them.
- [ ] Quarantine invalid records.
- [ ] Reconcile input, valid, and invalid populations.
- [ ] Monitor range failures and rule drift.
- [ ] Intentionally break boundary and reference-data logic.
- [ ] Recover safely from incorrect rules.
- [ ] Explain how PostgreSQL, dbt, and Great Expectations support production range validation.

## 12. What You Learned

Range validation protects typed data from values that are technically representable but operationally or business-wise invalid.

The production workflow is:

    DEFINE RANGE CONTRACT
         ↓
    TYPE VALIDATION
         ↓
    APPLY STATIC OR DYNAMIC BOUNDS
         ↓
    CHECK CROSS-FIELD RELATIONSHIPS
         ↓
    CLASSIFY FAILURES
         ↓
    QUARANTINE INVALID RECORDS
         ↓
    ENFORCE STABLE TARGET INVARIANTS
         ↓
    MONITOR BOUNDARIES AND RULE DRIFT

> **A correctly typed value is not automatically a valid value; production range validation makes boundaries explicit, testable, versioned, observable, and recoverable.**

### Next recipe

**T36 — Domain Validation**