# T19 — GROUP BY Aggregation

> **Goal:** Reduce detailed records into a well-defined summary grain using SQL `GROUP BY` and aggregate functions while preserving correct grouping semantics, NULL behavior, numeric meaning, reconciliation, and protection against double-counting.

Aggregation intentionally changes the grain of a dataset.

Examples:

    payments → daily payment totals
    orders → customer order counts
    events → hourly event volume
    transactions → account balances
    line items → invoice totals

The central risk is that an aggregation can execute perfectly while producing the wrong business result because the grouping grain, join cardinality, NULL semantics, or aggregate function is wrong.

The central rule is:

> **Define the output grain before writing `GROUP BY`. Every grouping key and aggregate must have an explicit business meaning and reconciliation path.**

---

## 1. Problem Recognition

### 1.1 Typical aggregation problem

Suppose payments contain:

    payment_id
    customer_id
    amount
    status
    event_time

The business needs:

    customer_id
    payment_count
    completed_amount

That transformation changes the grain:

    one payment → many payments per customer
    detailed rows → one row per customer

Example:

    SELECT
        customer_id,
        COUNT(*) AS payment_count,
        SUM(amount) AS total_amount
    FROM payments
    GROUP BY customer_id;

### 1.2 Aggregation changes grain

Before:

    one row = one payment

After:

    one row = one customer

This is fundamentally different from T17 projection and T18 filtering.

### 1.3 Aggregation is not deduplication

`GROUP BY customer_id` does not mean duplicate removal.

It intentionally combines all rows sharing the grouping key.

If the input contains:

    customer_id = 42, amount = 100
    customer_id = 42, amount = 100

the total becomes:

    200

not:

    100

If the duplicates are accidental, aggregation may hide the source defect.

### 1.4 Aggregation is not merging

Record merging can combine complementary records into one logical entity.

Aggregation computes measures over a group.

Example:

    customer records → canonical customer

is merging.

    payments → customer payment total

is aggregation.

T16 covers merging.

### 1.5 Red flags

Investigate when:

- output grain is undocumented
- `SUM` suddenly increases after adding a JOIN
- counts do not reconcile with source data
- `COUNT(*)` and `COUNT(column)` differ unexpectedly
- NULL values disappear from metrics
- integer division changes a metric
- a grouping key is accidentally omitted
- `DISTINCT` is added to hide duplicate inputs
- a metric changes when unrelated columns are added
- empty input behavior is misunderstood.

---

## 2. Concept and Reasoning

### 2.1 Define the aggregation contract

Before writing SQL, document:

    input grain
    output grain
    grouping keys
    aggregate measures
    NULL semantics
    numeric type
    time window
    timezone
    duplicate policy
    empty-input behavior
    reconciliation rules

Example:

| Property | Contract |
|---|---|
| Input grain | one payment |
| Output grain | one customer per UTC day |
| Group keys | customer_id, event_date |
| Count | all payment rows |
| Amount | completed payments only |
| NULL amount | excluded from SUM but counted separately |
| Timezone | UTC |
| Duplicate policy | duplicates are source records, not removed |

### 2.2 Grouping keys define the output grain

Consider:

    GROUP BY customer_id

Output grain:

    one row per customer

Now add:

    GROUP BY customer_id, currency

Output grain becomes:

    one row per customer + currency

That is a different model.

Every grouping column changes the grain.

### 2.3 The grain test

Ask:

> What real-world thing does one output row represent?

Examples:

    one customer
    one customer per day
    one account per currency
    one product per warehouse

If you cannot answer this precisely, the aggregation is not ready.

### 2.4 Aggregate functions have different semantics

Common functions:

    COUNT
    SUM
    AVG
    MIN
    MAX
    BOOL_AND
    BOOL_OR
    ARRAY_AGG

They do not mean the same thing.

Example:

    COUNT(*)

counts rows.

Whereas:

    COUNT(amount)

counts rows where `amount` is non-NULL.

### 2.5 COUNT semantics

These are different:

    COUNT(*)
    COUNT(customer_id)
    COUNT(DISTINCT customer_id)

`COUNT(*)` counts rows.

`COUNT(column)` ignores NULL values.

`COUNT(DISTINCT column)` counts distinct non-NULL values.

Do not substitute one for another without a metric definition.

### 2.6 SUM and NULL

Suppose a group contains:

    100
    50
    NULL

Then:

    SUM(amount) = 150

NULL does not contribute a numeric value.

But a group containing only NULL values can produce NULL rather than zero.

If the business metric requires zero, make that explicit:

    COALESCE(SUM(amount), 0)

Do not apply `COALESCE` blindly. NULL may mean “unknown” rather than zero.

### 2.7 AVG is not always the average of averages

Suppose:

    group A → average 10 over 100 rows
    group B → average 20 over 10 rows

The correct combined average is weighted by row count.

It is not:

    (10 + 20) / 2 = 15

Aggregation layers must preserve enough information to calculate the desired higher-level metric.

### 2.8 Numeric types matter

Aggregation can overflow or lose precision if the input type is inappropriate.

Financial values should normally use exact numeric types rather than binary floating point.

Example:

    SUM(amount_numeric)

should be evaluated against the target precision and scale.

T08 covers numeric transformation. T19 applies those principles to aggregates.

### 2.9 Conditional aggregation

Many business metrics are conditional aggregates:

    SELECT
        customer_id,
        COUNT(*) AS payment_count,
        COUNT(*) FILTER (WHERE status = 'COMPLETED') AS completed_count,
        COALESCE(SUM(amount) FILTER (WHERE status = 'COMPLETED'), 0) AS completed_amount
    FROM payments
    GROUP BY customer_id;

This produces multiple measures at the same grain without creating multiple scans in the logical SQL model.

### 2.10 Grouping NULL values

Consider:

    GROUP BY country

All NULL country values belong to the same grouping bucket.

That does not mean NULL countries are the same real-world country.

Interpret the NULL group as:

    country unknown/missing

unless the domain says otherwise.

### 2.11 Time-based grouping

Grouping by timestamp directly often creates one group per distinct timestamp.

Instead derive the intended bucket:

    date_trunc('hour', event_time)

or:

    event_time::date

The bucket's timezone semantics must be explicit.

### 2.12 Business timezone in aggregation

If the metric is:

    revenue per Berlin calendar day

then UTC midnight is not necessarily the business-day boundary.

Convert the event instant into the business timezone before deriving the date bucket.

### 2.13 GROUP BY and joins can multiply measures

Suppose one customer has:

    3 orders
    4 addresses

A naive join can create:

    3 × 4 = 12 rows

Then:

    SUM(order_amount)

can be multiplied by four.

This is one of the most dangerous aggregation defects because the SQL remains valid.

Aggregate at the correct grain before joining when necessary.

### 2.14 Pre-aggregation

Instead of:

    orders JOIN many_addresses → GROUP BY customer

consider:

    orders → aggregate by customer
    addresses → select/aggregate to one customer row
    customer aggregates → join

This prevents many-to-many multiplication.

### 2.15 HAVING is post-aggregation filtering

`WHERE` filters input rows before aggregation.

`HAVING` filters groups after aggregation.

Example:

    SELECT
        customer_id,
        SUM(amount) AS total_amount
    FROM payments
    WHERE status = 'COMPLETED'
    GROUP BY customer_id
    HAVING SUM(amount) >= 1000;

Semantics:

    WHERE → choose payment rows
    GROUP BY → create customer groups
    SUM → calculate measure
    HAVING → choose qualifying groups

T18 covers WHERE filtering. T19 introduces the post-aggregation population boundary.

### 2.16 Empty input versus zero-valued metric

These are different concepts:

    no rows exist
    rows exist but metric value is zero

A grouped query over an empty input returns no groups.

Do not assume it will return:

    customer_id = X, total = 0

unless the query begins from a population that already contains customer X and uses an appropriate outer join.

---

## 3. Implementation

### 3.1 Basic aggregation

    SELECT
        customer_id,
        COUNT(*) AS payment_count,
        SUM(amount) AS total_amount
    FROM payments
    GROUP BY customer_id;

The output grain is one row per customer.

### 3.2 Multiple grouping keys

    SELECT
        customer_id,
        currency,
        COUNT(*) AS payment_count,
        SUM(amount) AS total_amount
    FROM payments
    GROUP BY customer_id, currency;

The output grain is customer + currency.

### 3.3 Conditional aggregation

    SELECT
        customer_id,
        COUNT(*) AS payment_count,
        COUNT(*) FILTER (WHERE status = 'COMPLETED') AS completed_count,
        COALESCE(
            SUM(amount) FILTER (WHERE status = 'COMPLETED'),
            0
        ) AS completed_amount
    FROM payments
    GROUP BY customer_id;

Use this when the business wants multiple measures at the same grain.

### 3.4 COUNT variants

    SELECT
        customer_id,
        COUNT(*) AS row_count,
        COUNT(email) AS known_email_count,
        COUNT(DISTINCT email) AS distinct_email_count
    FROM customers
    GROUP BY customer_id;

Document the intended meaning of each metric.

### 3.5 Safe averages

Example:

    SELECT
        customer_id,
        AVG(amount) AS average_payment
    FROM payments
    WHERE amount IS NOT NULL
    GROUP BY customer_id;

If NULL amount rows matter to the quality metric, count them separately rather than silently losing that information.

### 3.6 Conditional SUM with CASE

Portable SQL can express conditional aggregation with `CASE`:

    SELECT
        customer_id,
        SUM(
            CASE
                WHEN status = 'COMPLETED' THEN amount
                ELSE 0
            END
        ) AS completed_amount
    FROM payments
    GROUP BY customer_id;

Be careful with NULL amounts and with the semantic difference between zero and unknown.

### 3.7 Time buckets

Example:

    SELECT
        date_trunc('hour', event_time) AS event_hour,
        COUNT(*) AS event_count
    FROM events
    GROUP BY date_trunc('hour', event_time);

For production models, define the timezone semantics of the bucket.

### 3.8 Business-day aggregation

Example:

    SELECT
        (event_time AT TIME ZONE 'Europe/Berlin')::date AS business_date,
        SUM(amount) AS revenue
    FROM payments
    GROUP BY (event_time AT TIME ZONE 'Europe/Berlin')::date;

Use IANA timezone semantics rather than a fixed offset when DST applies.

### 3.9 HAVING

    SELECT
        customer_id,
        SUM(amount) AS total_amount
    FROM payments
    WHERE status = 'COMPLETED'
    GROUP BY customer_id
    HAVING SUM(amount) >= 1000;

`WHERE` reduces detailed input rows.
`HAVING` reduces grouped output rows.

### 3.10 Aggregate before joining

Suppose orders can have multiple related events.

Prefer:

    WITH order_totals AS (
        SELECT
            customer_id,
            SUM(amount) AS total_order_amount
        FROM orders
        GROUP BY customer_id
    )
    SELECT
        c.customer_id,
        o.total_order_amount
    FROM customers c
    LEFT JOIN order_totals o
        ON o.customer_id = c.customer_id;

This prevents event-level joins from multiplying order amounts.

### 3.11 Reconciliation query

Before trusting an aggregate, compare it with an independent calculation:

    SELECT
        SUM(amount) AS source_total
    FROM payments
    WHERE status = 'COMPLETED';

Then compare with:

    SELECT
        SUM(total_amount) AS aggregate_total
    FROM customer_payment_totals;

For a complete, non-overlapping population, the totals should reconcile.

### 3.12 Group uniqueness check

The output should have exactly one row per intended grouping key.

Validate:

    SELECT
        customer_id,
        COUNT(*) AS rows_per_customer
    FROM customer_payment_totals
    GROUP BY customer_id
    HAVING COUNT(*) > 1;

Expected result:

    zero rows

---

## 4. Testing

### 4.1 Minimum test matrix

| Test | Expected result |
|---|---|
| one row in group | correct aggregate |
| multiple rows | correct reduction |
| NULL measure | documented behavior |
| all NULL measures | NULL or zero according to contract |
| zero measure | retained as numeric zero |
| negative measure | handled according to domain |
| duplicate source row | counted unless explicitly deduplicated |
| multiple grouping keys | correct target grain |
| empty input | documented empty behavior |
| timestamp boundary | correct bucket |
| timezone boundary | correct business bucket |
| post-aggregation threshold | HAVING works correctly |
| many-to-many join risk | detected/prevented |

### 4.2 Grain test

After aggregation, verify that the grouping key is unique:

    SELECT
        customer_id,
        COUNT(*)
    FROM aggregate_table
    GROUP BY customer_id
    HAVING COUNT(*) > 1;

Expected:

    zero rows

### 4.3 Source-to-aggregate reconciliation

Compare:

    source count
    aggregate count where appropriate
    source total
    aggregate total

Be careful: aggregate row count is not expected to equal source row count.

The correct reconciliation depends on the measure and grain.

### 4.4 NULL tests

Test groups containing:

    no NULL values
    one NULL value
    all NULL values

Verify `COUNT`, `SUM`, and `AVG` semantics explicitly.

### 4.5 Join multiplication test

Create:

    3 orders
    4 related rows

Perform the risky join and observe the 12-row multiplication.

Then aggregate before joining and verify the correct total.

### 4.6 Timezone tests

Create events around:

    UTC midnight
    local midnight
    DST transition

Verify the intended business-day or hour bucket.

### 4.7 Determinism test

Run the aggregation repeatedly against the same snapshot.

Expected:

    identical grouping keys
    identical measures
    identical output

### 4.8 Idempotence test

Rebuild the aggregate from the same source snapshot twice.

Expected:

    no accumulation
    same totals
    same group counts

---

## 5. Observability

Track:

    input_row_count
    output_group_count
    average_group_size
    maximum_group_size
    NULL_measure_count
    empty_group_count where applicable
    source_total
    aggregate_total
    reconciliation_difference
    group_key_null_count
    aggregation_rule_version

### 5.1 Reconciliation difference

Define:

    reconciliation_difference = source_total - aggregate_total

Expected for a complete, non-overlapping population:

    0

Non-zero differences require investigation.

### 5.2 Group-size distribution

Monitor:

    minimum group size
    average group size
    maximum group size
    percentile distribution where useful

A sudden increase can indicate:

    duplicate ingestion
    key collapse
    missing grouping key
    join multiplication

### 5.3 NULL metric rates

Track:

    NULL amount rate
    zero amount rate
    missing grouping-key rate

Do not combine NULL and zero into one metric.

### 5.4 Metric distribution

Monitor aggregate values:

    count
    sum
    average
    minimum
    maximum

Large distribution changes can reveal upstream or logic regressions.

### 5.5 Query performance

Aggregation can become expensive when large volumes require:

    sorting
    hashing
    disk spill
    large memory grants

Use query plans and execution metrics to understand bottlenecks.

---

## 6. Intentional Failure

### Failure 1 — Missing grouping key

Suppose the required grain is:

    customer + currency

but the query groups only by:

    customer

Expected lesson:

> A missing grouping key silently changes the meaning of the metric.

### Failure 2 — COUNT(column) instead of COUNT(*)

Introduce NULL values.

Compare:

    COUNT(*)
    COUNT(email)

Expected lesson:

> Different COUNT forms represent different populations.

### Failure 3 — NULL mistaken for zero

Create a group where every amount is NULL.

Observe:

    SUM(amount)

Expected lesson:

> Unknown/missing measure and numeric zero are not automatically equivalent.

### Failure 4 — Join multiplication

Join orders to multiple child records before summing.

Expected lesson:

> Correct aggregation over an incorrect joined grain still produces incorrect totals.

### Failure 5 — Wrong timezone bucket

Aggregate events by UTC date when the business requires a local date.

Expected lesson:

> A valid timestamp expression can still create the wrong business bucket.

### Failure 6 — Averaging averages

Aggregate daily averages and then average those daily averages when days have different row counts.

Expected lesson:

> Averages require appropriate weighting.

### Failure 7 — DISTINCT hides duplicate source rows

Add `DISTINCT` before aggregation.

Expected lesson:

> Removing duplicates changes the metric unless deduplication is part of the contract.

### Failure 8 — Empty input assumed to produce zero

Run a grouped query against an empty table.

Expected lesson:

> No groups is different from a group whose metric equals zero.

---

## 7. Recovery

### 7.1 Stop incorrect publication

If aggregate values are known to be wrong, isolate affected downstream models, reports, or exports where possible.

Do not continue publishing corrupted metrics.

### 7.2 Identify the aggregation version

Record:

    pipeline run ID
    SQL/model version
    source snapshot
    grouping keys
    time window
    timezone
    aggregation rule version

### 7.3 Compare detailed input with grouped output

Inspect:

    grouping-key distribution
    source totals
    aggregate totals
    duplicate populations
    NULL populations
    join cardinality

### 7.4 Correct the grain or expression

Typical corrections include:

    add missing grouping key
    remove unintended grouping key
    aggregate before joining
    fix NULL semantics
    correct numeric type
    correct timezone
    remove inappropriate DISTINCT

### 7.5 Replay from trusted input

Rebuild the aggregate from preserved source/staging data.

Verify:

    group uniqueness
    source-to-output reconciliation
    metric distributions
    boundary populations
    downstream reconciliation

### 7.6 Audit downstream consumers

Determine whether incorrect metrics reached:

    dashboards
    reports
    warehouse tables
    APIs
    exports
    financial summaries

Correct downstream state according to its own recovery procedure.

---

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

Know:

    GROUP BY
    aggregate functions
    FILTER
    HAVING
    date_trunc
    EXPLAIN
    indexes

PostgreSQL provides the execution engine and query planner for relational aggregation.

### 8.2 dbt

Know how dbt can provide:

    version-controlled aggregate models
    tests
    documentation
    dependencies
    model-level reconciliation checks

### 8.3 DuckDB

Know how DuckDB can support:

    local analytical aggregation
    Parquet workloads
    reproducible metric tests
    rapid experimentation with large local datasets

---

## 9. Production Runbook

### Before deployment

Verify:

- output grain is explicitly defined
- grouping keys are complete
- every aggregate has a metric definition
- NULL semantics are documented
- numeric precision is appropriate
- timezone is explicit for temporal buckets
- duplicate policy is explicit
- join cardinality is understood
- reconciliation queries exist
- boundary cases are tested.

### During execution

Monitor:

    input rows
    output groups
    group-size distribution
    NULL measure rates
    source totals
    aggregate totals
    reconciliation difference
    metric distributions
    query execution metrics

### If aggregate values look wrong

1. Identify the aggregation version.
2. Confirm the source snapshot and time window.
3. Confirm the output grain.
4. Inspect grouping keys.
5. Check for join multiplication.
6. Check NULL and zero semantics.
7. Check numeric precision.
8. Check timezone buckets.
9. Compare source and aggregate totals.
10. Replay from trusted input.
11. Reconcile downstream consumers.

---

## 10. Common Mistakes

### Mistake 1 — Not defining output grain

Why it fails:

You cannot determine whether the grouping keys are correct.

### Mistake 2 — Omitting a grouping key

Why it fails:

Different business populations become one metric.

### Mistake 3 — Counting the wrong thing

Why it fails:

`COUNT(*)`, `COUNT(column)`, and `COUNT(DISTINCT column)` have different meanings.

### Mistake 4 — Treating NULL as zero

Why it fails:

Unknown measure and zero measure are different states.

### Mistake 5 — Aggregating after a many-to-many join

Why it fails:

Measures can be multiplied silently.

### Mistake 6 — Using DISTINCT to repair source duplication

Why it fails:

The source defect is hidden and metric semantics change.

### Mistake 7 — Ignoring timezone

Why it fails:

Daily and hourly metrics can be assigned to the wrong business bucket.

### Mistake 8 — Averaging averages

Why it fails:

Unequal group sizes require weighting.

### Mistake 9 — Assuming empty input means zero

Why it fails:

No groups and zero-valued groups are different results.

### Mistake 10 — Skipping reconciliation

Why it fails:

A valid SQL query can still produce an incorrect business metric.

---

## 11. Definition of Done

A production aggregation is complete when:

- [ ] input grain is documented
- [ ] output grain is documented
- [ ] grouping keys are explicit
- [ ] every aggregate has a business definition
- [ ] COUNT semantics are explicit
- [ ] NULL semantics are explicit
- [ ] zero versus unknown is defined
- [ ] numeric precision is appropriate
- [ ] temporal bucket timezone is defined
- [ ] duplicate policy is defined
- [ ] join cardinality is validated
- [ ] source-to-output reconciliation exists
- [ ] group uniqueness is tested
- [ ] normal cases are tested
- [ ] NULL cases are tested
- [ ] boundary cases are tested
- [ ] join multiplication is tested
- [ ] determinism is tested
- [ ] idempotence is tested
- [ ] observability exists
- [ ] intentional failure has been exercised
- [ ] recovery has been tested
- [ ] aggregation version is traceable
- [ ] downstream reconciliation is understood.

---

## 12. What You Learned

SQL aggregation is not simply adding `GROUP BY` to a query.

You learned how to:

    define output grain
    choose grouping keys
    distinguish COUNT variants
    handle NULL measures
    perform conditional aggregation
    create time buckets
    handle business timezones
    prevent join multiplication
    aggregate before joining when required
    distinguish WHERE from HAVING
    reconcile detailed data with summaries
    test aggregation correctness
    observe metric drift
    intentionally break an aggregate
    recover from incorrect aggregation logic
    operate aggregate models in production

The key lesson is:

> **Aggregation is a change of grain. If you cannot state exactly what one output row represents and how its measures reconcile to source records, the aggregation is not production-ready.**

### Next Recipe

**T20 — JOIN Transformations**

After learning controlled aggregation, the next recipe covers relational joins and the cardinality risks they introduce.