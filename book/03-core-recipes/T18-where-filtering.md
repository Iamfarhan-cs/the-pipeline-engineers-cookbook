# T18 — WHERE Filtering

> **Goal:** Filter rows with explicit, deterministic SQL predicates while correctly handling NULL, boundaries, dates, booleans, unknown values, predicate ordering, and population accounting.

`WHERE` defines which input rows are eligible to continue through a SQL transformation.

Examples:

    completed payments
    active customers
    events inside a time window
    records belonging to one tenant
    valid records for a downstream model

The central risk is that SQL filtering can silently exclude records when NULL semantics, boundary conditions, timezone interpretation, or boolean logic are misunderstood.

The central rule is:

> **A production filter is a population-definition contract. Every predicate must have an explicit meaning, boundary, NULL behavior, and accounting expectation.**

---

## 1. Problem Recognition

### 1.1 Typical filtering problem

Suppose a payment table contains:

    payment_id
    customer_id
    status
    amount
    created_at

The downstream model should contain completed payments only:

    SELECT *
    FROM payments
    WHERE status = 'COMPLETED';

The SQL is simple.

The engineering question is:

> What exactly does `COMPLETED` mean, and which records are intentionally excluded?

### 1.2 Filtering is not cleansing

Cleansing changes or repairs values.

Filtering decides whether a record participates.

Example:

    trim email → cleansing
    reject records with unusable email → filtering

T12 covers cleansing. T18 focuses on population selection.

### 1.3 Filtering is not projection

`SELECT` determines output columns.

`WHERE` determines participating rows.

T17 covers projection. T18 covers row selection.

### 1.4 Filtering is not deduplication

Consider:

    WHERE status = 'ACTIVE'

This selects active records.

`DISTINCT`, grouping, or window-based selection can change duplicate behavior.

Do not add deduplication merely because a filtered dataset contains repeated business entities.

### 1.5 Red flags

Investigate when:

- row counts change unexpectedly
- NULL records disappear without explanation
- a time-window filter loses boundary records
- `NOT` behaves differently from expected
- `OR` conditions include unintended records
- a filter uses local time against UTC timestamps
- a supposedly selective predicate returns almost everything
- a new source value is silently excluded
- `WHERE` logic differs between similar pipelines.

---

## 2. Concept and Reasoning

### 2.1 Define the filter contract

Before writing SQL, document:

    target population
    inclusion criteria
    exclusion criteria
    NULL behavior
    time-window boundaries
    timezone
    unknown-value policy
    tenant/security scope
    expected counts
    predicate version

Example:

| Property | Contract |
|---|---|
| Population | completed payments |
| Status | exactly `COMPLETED` |
| Amount | greater than or equal to 0 |
| Event time | start inclusive, end exclusive |
| Timezone | UTC |
| NULL status | excluded and counted |
| Unknown status | excluded and monitored |

### 2.2 SQL uses three-valued logic

SQL predicates can evaluate to:

    TRUE
    FALSE
    UNKNOWN

`WHERE` keeps rows only when the predicate is `TRUE`.

Consider:

    amount > 100

If `amount` is NULL:

    amount > 100 → UNKNOWN

The row is not returned.

This is different from application code that may represent missing data differently.

### 2.3 NULL is not FALSE

Consider:

    WHERE is_active = TRUE

Rows with:

    is_active = TRUE → included
    is_active = FALSE → excluded
    is_active = NULL → excluded because comparison is UNKNOWN

If the business rule needs to distinguish FALSE from missing, make that distinction explicit.

### 2.4 `IS NULL` and `IS NOT NULL`

Correct:

    WHERE email IS NULL

Correct:

    WHERE email IS NOT NULL

Incorrect:

    WHERE email = NULL

`= NULL` does not evaluate to TRUE.

### 2.5 AND and OR precedence

Consider:

    WHERE status = 'ACTIVE'
    AND country = 'DE'
    OR country = 'FR'

SQL interprets `AND` before `OR`.

That is effectively:

    (status = 'ACTIVE' AND country = 'DE')
    OR country = 'FR'

That may unintentionally include inactive French records.

Prefer explicit parentheses:

    WHERE status = 'ACTIVE'
      AND (country = 'DE' OR country = 'FR')

### 2.6 NOT and NULL

Consider:

    WHERE NOT is_deleted

If `is_deleted` is NULL, `NOT NULL` is UNKNOWN.

Therefore the row is excluded.

If the contract means:

    NULL should be treated as not deleted

write that explicitly:

    WHERE COALESCE(is_deleted, FALSE) = FALSE

Only do this when NULL-as-false is actually the business rule.

### 2.7 Equality is not always population membership

Consider:

    WHERE status IN ('PAID', 'SETTLED')

This defines an explicit status population.

If a new status such as `REVERSED` appears, it is excluded.

That can be correct, but the pipeline should detect the new value rather than silently treating it as irrelevant.

### 2.8 Time-window boundaries

A robust batch window often uses:

    start <= event_time < end

Example:

    WHERE event_time >= '2026-09-01 00:00:00+00'
      AND event_time <  '2026-10-01 00:00:00+00'

This avoids overlapping adjacent batches.

Do not use:

    event_time BETWEEN start_time AND end_time

when the end boundary should be exclusive.

`BETWEEN` is inclusive at both ends.

### 2.9 Half-open intervals compose cleanly

Suppose batches are:

    [00:00, 01:00)
    [01:00, 02:00)

A record at exactly 01:00 belongs to the second interval.

There is no overlap and no gap.

This pattern is especially useful for incremental ETL.

### 2.10 Date filtering versus timestamp filtering

Be precise about what the user means by a date.

Example:

    WHERE event_date = DATE '2026-09-28'

is different from:

    WHERE event_timestamp >= ...
    AND event_timestamp < ...

If `event_timestamp` is an instant, convert it according to the business timezone before deriving a calendar date.

T06 and T07 cover the underlying time semantics.

### 2.11 Timezone must be explicit

Suppose the business asks for:

    all events occurring on September 28 in Berlin

Do not assume that:

    UTC date = Berlin date

Convert the business window into the timestamp semantics used by the source.

### 2.12 Predicate order is a correctness and performance concern

SQL optimizers may reorder predicates.

Therefore never rely on textual predicate order to protect an unsafe expression.

Bad assumption:

    WHERE value <> 0
      AND numerator / value > 10

Do not assume the first predicate guarantees the second expression cannot encounter zero.

Use safe expressions or `NULLIF` when appropriate:

    numerator / NULLIF(value, 0)

Correctness should not depend on the optimizer evaluating conditions in written order.

### 2.13 Filter pushdown

Filtering early can reduce data processed by later stages.

Example:

    source
      ↓
    filter relevant partition/date
      ↓
    expensive transformation

But early filtering is valid only when the predicate is semantically equivalent to the intended final population.

Do not push a filter below a transformation if that changes its meaning.

### 2.14 Security filters are different

A tenant predicate such as:

    WHERE tenant_id = :tenant_id

may be a security boundary rather than a normal business filter.

Treat security predicates as mandatory controls.

Never allow a caller-controlled optional parameter to accidentally remove tenant isolation.

### 2.15 Filter accounting

Every important filter should make population loss explainable.

Example:

    input_rows = 1,000,000
    selected_rows = 820,000
    NULL_status = 10,000
    unsupported_status = 25,000
    outside_time_window = 145,000

The categories should reconcile when the classification is mutually exclusive.

---

## 3. Implementation

### 3.1 Basic equality filter

    SELECT
        payment_id,
        customer_id,
        amount
    FROM payments
    WHERE status = 'COMPLETED';

Use explicit status vocabulary.

### 3.2 Multiple predicates

    SELECT
        payment_id,
        amount
    FROM payments
    WHERE status = 'COMPLETED'
      AND amount >= 0;

Document whether negative amounts are invalid, refunds, or another legitimate business state.

### 3.3 Parenthesize OR conditions

Use:

    SELECT *
    FROM customers
    WHERE is_active = TRUE
      AND (country = 'DE' OR country = 'FR');

This makes the intended boolean structure visible.

### 3.4 Use IN for finite membership

    SELECT *
    FROM payments
    WHERE status IN ('COMPLETED', 'SETTLED');

`IN` makes finite membership easier to read and review than a long OR chain.

### 3.5 NULL filtering

    SELECT *
    FROM customers
    WHERE email IS NOT NULL;

For explicit missing population:

    SELECT *
    FROM customers
    WHERE email IS NULL;

### 3.6 Boolean filtering

Prefer:

    WHERE is_active IS TRUE

when the distinction between TRUE, FALSE, and NULL matters.

Use:

    WHERE COALESCE(is_active, FALSE) IS TRUE

only when NULL is contractually equivalent to FALSE.

### 3.7 Range filtering

Example:

    SELECT *
    FROM payments
    WHERE amount >= 100
      AND amount < 1000;

This defines:

    [100, 1000)

Use half-open ranges when adjacent ranges must compose without overlap.

### 3.8 Timestamp filtering

Use:

    SELECT *
    FROM events
    WHERE event_time >= TIMESTAMPTZ '2026-09-28 00:00:00+00'
      AND event_time <  TIMESTAMPTZ '2026-09-29 00:00:00+00';

This is safer for daily incremental processing than an inclusive end timestamp.

### 3.9 Filtering by business timezone

Example for a Berlin business day:

    SELECT *
    FROM events
    WHERE event_time >= TIMESTAMPTZ '2026-09-28 00:00:00+02'
      AND event_time <  TIMESTAMPTZ '2026-09-29 00:00:00+02';

Do not hardcode an offset for recurring timezone-aware logic when DST applies. Build the window using the relevant IANA timezone semantics in the pipeline or database layer.

### 3.10 Safe division

Instead of relying on predicate order:

    numerator / NULLIF(denominator, 0)

Then filter or validate the result according to the business contract.

### 3.11 Filter disposition for observability

A useful pattern is to classify records before final selection:

    SELECT
        payment_id,
        CASE
            WHEN status IS NULL THEN 'MISSING_STATUS'
            WHEN status NOT IN ('COMPLETED', 'SETTLED') THEN 'UNSUPPORTED_STATUS'
            WHEN amount < 0 THEN 'INVALID_AMOUNT'
            ELSE 'SELECTED'
        END AS filter_disposition
    FROM payments;

This can support accounting and quarantine workflows.

Do not duplicate complex filter rules in multiple places without versioning them.

### 3.12 PostgreSQL example

Suppose a daily model needs completed, non-negative payments for one tenant.

    SELECT
        payment_id,
        tenant_id,
        customer_id,
        amount,
        event_time
    FROM staging.payments
    WHERE tenant_id = :tenant_id
      AND status = 'COMPLETED'
      AND amount >= 0
      AND event_time >= :window_start
      AND event_time < :window_end;

The calling pipeline should supply a validated half-open window.

---

## 4. Testing

### 4.1 Minimum test matrix

| Test | Expected result |
|---|---|
| matching value | included |
| non-matching value | excluded |
| NULL compared with equality | excluded/handled according to contract |
| `IS NULL` | missing rows selected |
| `IS NOT NULL` | known rows selected |
| lower boundary | included when lower bound is inclusive |
| upper boundary | excluded when upper bound is exclusive |
| OR condition | only intended combinations included |
| NOT with NULL | documented behavior |
| unknown status | documented disposition |
| negative/zero numeric boundary | correct business behavior |
| timezone boundary | correct business date/window |
| tenant mismatch | excluded |

### 4.2 Three-valued logic tests

Test predicates with:

    TRUE
    FALSE
    NULL

For example:

    is_active = TRUE

Verify the intended treatment of:

    TRUE → TRUE
    FALSE → FALSE
    NULL → UNKNOWN → filtered out

### 4.3 Boolean precedence tests

Create records covering every combination of:

    active/inactive
    country DE/FR/other

Verify the predicate:

    is_active = TRUE
    AND (country = 'DE' OR country = 'FR')

does not accidentally include inactive records.

### 4.4 Boundary tests

For:

    amount >= 100
    AND amount < 1000

test:

    99.99
    100
    100.01
    999.99
    1000

### 4.5 Time-window tests

Test timestamps exactly at:

    window_start
    window_end

Expected:

    start → included
    end → excluded

Also test adjacent windows to prove there is no overlap or gap.

### 4.6 Filter accounting test

Classify a controlled dataset into:

    selected
    missing required value
    unsupported value
    invalid value

Verify that the classifications reconcile with the input population when the rules are designed to be mutually exclusive.

### 4.7 Determinism test

Run the same predicate against the same snapshot multiple times.

Expected:

    identical selected population

If the filter depends on `CURRENT_TIMESTAMP`, freeze the evaluation time for testing.

---

## 5. Observability

Track:

    input_row_count
    selected_row_count
    excluded_row_count
    NULL_predicate_count
    unknown_value_count
    invalid_value_count
    filter_rate
    time_window_start
    time_window_end
    timezone
    predicate_version

### 5.1 Filter rate

Example:

    selected / input = 82%

A sudden change may indicate:

    source distribution change
    upstream outage
    status vocabulary change
    incorrect predicate
    wrong time window

### 5.2 Track exclusion reasons

Instead of only logging:

    180,000 rows excluded

prefer categories such as:

    25,000 unsupported status
    10,000 missing status
    100,000 outside time window
    45,000 invalid amount

Reason categories make incidents diagnosable.

### 5.3 Watch boundary populations

Monitor records near important thresholds:

    amount around 100
    timestamps near midnight
    timestamps near window boundaries

This can reveal unit, timezone, or comparison errors.

### 5.4 Security-filter observability

For tenant-scoped processing, monitor:

    tenant ID
    requested scope
    actual selected scope
    cross-tenant match count

Do not log sensitive record contents.

---

## 6. Intentional Failure

### Failure 1 — `= NULL`

Change:

    WHERE email IS NULL

to:

    WHERE email = NULL

Expected lesson:

> SQL NULL is not compared with `=`.

### Failure 2 — AND/OR precedence

Use:

    WHERE is_active = TRUE
      AND country = 'DE'
      OR country = 'FR'

Create inactive French records.

Expected lesson:

> SQL operator precedence can enlarge the selected population.

### Failure 3 — Inclusive upper bound

Change:

    event_time < window_end

to:

    event_time <= window_end

Run adjacent batches.

Expected lesson:

> Inclusive end boundaries can duplicate boundary records.

### Failure 4 — Timezone mismatch

Filter UTC timestamps using a local calendar boundary without conversion.

Expected lesson:

> A syntactically correct timestamp filter can select the wrong business day.

### Failure 5 — NULL boolean assumption

Use:

    WHERE NOT is_deleted

with NULL values.

Expected lesson:

> NOT NULL is UNKNOWN, not TRUE.

### Failure 6 — Unsafe predicate ordering

Use an expression that can divide by zero and assume another predicate will run first.

Expected lesson:

> SQL optimizer behavior must not be treated as procedural evaluation order.

### Failure 7 — Silent vocabulary exclusion

Add a new status:

    REVERSED

to the source.

Keep:

    WHERE status IN ('COMPLETED', 'SETTLED')

Expected lesson:

> A filter can silently exclude new source states unless population drift is observed.

---

## 7. Recovery

### 7.1 Stop incorrect publication

If a filter selects the wrong population, isolate affected downstream publication where possible.

Do not continue propagating an incorrect population.

### 7.2 Identify the filter version

Record:

    pipeline run ID
    SQL/model version
    predicate version
    source snapshot
    window start
    window end
    timezone

### 7.3 Reconstruct the intended population

Compare:

    original input population
    actual selected population
    expected selected population

Break differences into predicate categories.

### 7.4 Correct the predicate

Fix the documented rule:

    NULL handling
    boolean grouping
    boundary
    timezone
    status vocabulary
    security scope

Do not simply increase the selected row count until it “looks right.”

### 7.5 Replay from trusted input

Re-run the corrected filter against the preserved source or staging snapshot.

Verify:

    selected count
    exclusion reasons
    boundary records
    tenant scope
    downstream reconciliation

### 7.6 Audit downstream impact

Determine whether the wrong population reached:

    facts
    dimensions
    aggregates
    reports
    dashboards
    exports
    APIs

Correct downstream state according to its recovery contract.

---

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

Know:

    WHERE
    AND / OR / NOT
    IS NULL / IS NOT NULL
    IN
    BETWEEN
    CASE
    EXPLAIN
    indexes

PostgreSQL provides both filtering semantics and execution plans.

### 8.2 dbt

Know how to use dbt to:

    version SQL filters
    test selected populations
    document model assumptions
    compare model changes
    run targeted transformations

### 8.3 DuckDB

Know how to use DuckDB for:

    local SQL testing
    Parquet filtering
    reproducible predicate experiments
    validating boundary and NULL behavior

---

## 9. Production Runbook

### Before deployment

Verify:

- target population is documented
- inclusion and exclusion rules are explicit
- NULL semantics are documented
- AND/OR grouping is parenthesized where useful
- time boundaries are explicit
- timezone is explicit
- tenant/security scope is protected
- unknown values are observable
- expected filter rate is known
- representative boundary tests exist.

### During execution

Monitor:

    input rows
    selected rows
    filter rate
    exclusion reasons
    unknown values
    NULL predicate population
    time-window metadata
    predicate version

### If selected population changes unexpectedly

1. Identify the predicate version.
2. Confirm the source snapshot.
3. Compare input and selected counts.
4. Inspect exclusion reasons.
5. Check NULL semantics.
6. Check boolean grouping.
7. Check time boundaries and timezone.
8. Check new source values.
9. Replay from trusted input.
10. Reconcile downstream outputs.

---

## 10. Common Mistakes

### Mistake 1 — Using `= NULL`

Why it fails:

SQL NULL requires `IS NULL` or `IS NOT NULL`.

### Mistake 2 — Forgetting parentheses around OR

Why it fails:

`AND` has higher precedence than `OR`.

### Mistake 3 — Using BETWEEN for half-open ETL windows

Why it fails:

`BETWEEN` includes both endpoints.

### Mistake 4 — Treating NULL as FALSE

Why it fails:

SQL uses UNKNOWN for many NULL comparisons.

### Mistake 5 — Filtering timestamps without timezone semantics

Why it fails:

The selected business day can be wrong.

### Mistake 6 — Trusting predicate order

Why it fails:

SQL is declarative and optimizers can reorder evaluation.

### Mistake 7 — Hiding new source states

Why it fails:

Finite filters can silently exclude newly introduced values.

### Mistake 8 — Adding DISTINCT to fix population problems

Why it fails:

`DISTINCT` changes cardinality and can hide upstream defects.

### Mistake 9 — Ignoring filter accounting

Why it fails:

You cannot explain where the missing rows went.

### Mistake 10 — Treating security filters as ordinary business logic

Why it fails:

A missing tenant predicate can expose records across trust boundaries.

---

## 11. Definition of Done

A production WHERE filter is complete when:

- [ ] target population is explicitly defined
- [ ] inclusion rules are documented
- [ ] exclusion rules are documented
- [ ] NULL semantics are understood
- [ ] boolean precedence is tested
- [ ] time-window boundaries are explicit
- [ ] timezone semantics are correct
- [ ] tenant/security scope is protected
- [ ] unknown source values are observable
- [ ] expected filter rate is understood
- [ ] exclusion reasons are measurable where needed
- [ ] normal cases are tested
- [ ] NULL cases are tested
- [ ] boundary cases are tested
- [ ] determinism is tested
- [ ] intentional failure has been exercised
- [ ] recovery has been tested
- [ ] predicate/model version is traceable
- [ ] downstream reconciliation is understood.

---

## 12. What You Learned

SQL filtering is not merely writing a condition after `FROM`.

You learned how to:

    define a target population
    reason about SQL three-valued logic
    handle NULL explicitly
    structure AND/OR predicates safely
    use half-open time windows
    handle timezone-aware filtering
    protect tenant boundaries
    avoid relying on predicate evaluation order
    account for selected and excluded records
    observe population drift
    intentionally break a filter
    recover from incorrect selection logic
    operate filters in production

The key lesson is:

> **A WHERE clause defines who is allowed into the next stage. Treat its predicate, boundaries, NULL behavior, and population accounting as production logic.**

### Next Recipe

**T19 — GROUP BY Aggregation**

After learning row selection, the next recipe covers controlled reduction of records into grouped business measures.