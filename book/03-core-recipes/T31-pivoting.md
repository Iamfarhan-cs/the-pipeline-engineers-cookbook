# T31 — Pivoting

> **Goal:** Learn how to transform repeated row values into columns without losing grain, silently merging records, or creating ambiguous dynamic schemas.

## 1. Problem Recognition

Pivoting converts values from rows into separate columns.

Typical production cases include:

- Monthly metrics becoming month columns.
- Payment statuses becoming count columns.
- Product attributes becoming columns.
- Survey responses becoming feature columns.
- Event types becoming analytical columns.
- Key-value configuration data becoming a wide record.

Example input:

    customer_id | status  | amount
    ------------+---------+-------
    C1          | SUCCESS | 100
    C1          | FAILED  | 20
    C2          | SUCCESS | 80

Possible output:

    customer_id | success_amount | failed_amount
    ------------+----------------+--------------
    C1          | 100            | 20
    C2          | 80             | 0

The central problem is not syntax.

It is deciding:

    What should one output row represent?

and:

    Which row values become columns?

### Recognition questions

Before pivoting, ask:

1. What is the input grain?
2. What is the target grain?
3. Which column contains the pivot category?
4. Which measure is being pivoted?
5. Are categories known in advance?
6. Can multiple source rows map to one output cell?
7. If so, what aggregation resolves them?
8. What does a missing category mean?
9. Are NULL values different from zero?
10. Can new categories appear?
11. Does the output schema need to remain stable?
12. Can the transformation be reversed safely?

## 2. Concept and Reasoning

### 2.1 Pivot as a grain transformation

Pivoting changes representation and often changes grain.

Example:

    input grain  = customer + status
    output grain = customer

The status values become columns.

That means the pivot must resolve every set of input rows mapping to one customer/status cell.

### 2.2 Pivot without aggregation is ambiguous

Suppose the input contains:

    C1 | SUCCESS | 100
    C1 | SUCCESS | 200

If the output has one row per customer, what should `success_amount` be?

Possible policies:

- SUM = 300
- MAX = 200
- MIN = 100
- latest value = 200
- reject as duplicate

Pivoting requires an explicit collision policy.

### 2.3 Conditional aggregation

In SQL, the most portable pivot pattern is conditional aggregation:

    SUM(CASE WHEN status = 'SUCCESS' THEN amount ELSE 0 END)

or, where supported:

    SUM(amount) FILTER (WHERE status = 'SUCCESS')

This makes the category-to-column mapping explicit.

### 2.4 COUNT pivot

Example:

    COUNT(*) FILTER (WHERE status = 'SUCCESS')

produces a success count.

### 2.5 Boolean existence pivot

Sometimes the requirement is not a count but existence:

    MAX(CASE WHEN status = 'SUCCESS' THEN 1 ELSE 0 END)

This answers:

    Did this customer have at least one successful event?

### 2.6 SUM versus COUNT

These answer different questions.

    COUNT(*)

asks:

    How many records?

while:

    SUM(amount)

asks:

    What is the total amount?

Do not substitute one for the other.

### 2.7 Pivoting dimensions

A dimension such as:

    status

can become columns:

    success_count
    failed_count
    pending_count

This is useful for reporting but creates a fixed schema.

### 2.8 Pivoting measures

Multiple measures can be pivoted at once:

    success_count
    success_amount
    failed_count
    failed_amount

Each measure needs an explicit aggregation.

### 2.9 NULL semantics

These are different:

    no matching row

and:

    matching row with NULL amount

Example:

    SUM(CASE WHEN status = 'SUCCESS' THEN amount ELSE 0 END)

can produce zero for no success rows.

But a matched success row with NULL amount may also contribute no numeric value.

Define whether NULL means:

- Unknown.
- Not applicable.
- Missing source value.
- Zero.

Do not use `COALESCE(..., 0)` without understanding the business meaning.

### 2.10 Pivoting and zero

Zero is a value.

NULL is missing or unknown.

For reporting:

    no failed transactions → failed_count = 0

may be correct.

But:

    failed_amount = NULL

may be more appropriate if an existing failed transaction has an unknown amount.

### 2.11 Category completeness

A pivot with:

    SUCCESS
    FAILED
    PENDING

may silently omit a newly introduced status such as:

    REVERSED

Static pivot definitions must be monitored for unexpected categories.

### 2.12 Static pivot

A static pivot explicitly lists columns.

Advantages:

- Stable schema.
- Easy downstream contracts.
- Easy testing.
- Predictable query plans.

Disadvantage:

- New categories require code changes.

### 2.13 Dynamic pivot

A dynamic pivot generates columns from source values.

Advantages:

- Automatically accommodates new categories.

Disadvantages:

- Schema instability.
- More complex deployment.
- Harder downstream contracts.
- More difficult testing.

Dynamic SQL must never interpolate untrusted category values directly into executable SQL.

### 2.14 Pivoting and schema contracts

A production table or model should define:

    expected columns
    data types
    nullability
    semantic meaning

Dynamic columns can break consumers unexpectedly.

### 2.15 Pivoting and source grain

Suppose the source grain is:

    customer + status + transaction

and the target grain is:

    customer

then aggregation is required.

If the source is already:

    customer + status

then the pivot may not need additional aggregation.

### 2.16 Pivoting after aggregation

A useful flow is:

    raw events
       ↓
    validate
       ↓
    aggregate to business grain
       ↓
    pivot
       ↓
    publish wide model

This reduces duplicate ambiguity.

### 2.17 Pivoting before aggregation

Sometimes pivoting at event level is appropriate.

But if multiple events map to one output cell, the aggregation must still be explicit.

### 2.18 Pivoting and joins

Joining after a pivot can be simpler because the target grain is explicit.

However, joining before pivoting can multiply source rows and inflate aggregates.

Validate join cardinality before applying pivot aggregation.

### 2.19 Pivoting and fact tables

Wide pivoted facts can be useful for analytical consumption.

But repeated category columns can become difficult to maintain when the category domain changes frequently.

Use a normalized structure when category cardinality is high or highly dynamic.

### 2.20 Pivoting and dimensions

Pivoting is often appropriate for stable reporting dimensions.

Example:

    account_id | country

becomes feature columns such as:

    is_germany
    is_france
    is_spain

when the analytical requirement explicitly needs those features.

### 2.21 Pivoting key-value data

Input:

    entity_id | key        | value
    ----------+------------+------
    C1        | risk_level | HIGH
    C1        | segment    | SMB

Output:

    entity_id | risk_level | segment
    ----------+------------+--------
    C1        | HIGH       | SMB

This requires a uniqueness contract for:

    entity_id + key

or an explicit collision policy.

### 2.22 Duplicate key-value attributes

Suppose:

    C1 | segment | SMB
    C1 | segment | ENTERPRISE

The pivot cannot safely choose one value without a rule.

Possible policies:

- Latest value.
- Highest source priority.
- Reject.
- Aggregate if mathematically meaningful.

### 2.23 Pivoting dates

Months can become columns:

    Jan
    Feb
    Mar

This is common in reports but creates a time-dependent schema.

Prefer stable date-grain models for reusable analytical data and pivot only at the presentation boundary when possible.

### 2.24 Pivoting and financial data

Financial amounts should use exact numeric types.

Do not introduce floating-point rounding merely because the output is wide.

### 2.25 Pivoting and percentages

Do not average percentages unless the intended statistic is the unweighted average.

For a category rate:

    category_successes / category_attempts

should usually be calculated from aggregated counts.

### 2.26 Pivoting and reversibility

A pivot is reversible only if enough information survives the transformation.

Example:

    SUM(amount) by status

cannot reconstruct individual transactions.

Therefore pivoting is often lossy.

Treat the pivoted model as a derived representation, not a replacement for the normalized source.

### 2.27 Wide versus long

Long format:

    customer | status  | amount

Wide format:

    customer | success_amount | failed_amount

Long format is often easier to extend.

Wide format is often easier for reporting and machine-learning features.

Choose based on consumers and schema stability.

### 2.28 Pivoting and machine-learning features

Pivoting can produce feature columns such as:

    login_count
    payment_count
    refund_count

Ensure the feature window and cutoff time prevent future information leakage.

### 2.29 Pivoting and temporal leakage

If a feature for date D includes events after D, it can leak future information.

Define the feature cutoff before pivoting:

    source events <= feature_cutoff

### 2.30 Pivoting and sparse categories

Wide data can become extremely sparse.

Thousands of possible categories can produce thousands of mostly NULL or zero columns.

Consider:

- Normalized long storage.
- Top-category selection.
- Feature hashing.
- Separate dimension tables.

## 3. Implementation

### 3.1 Define the contract

Example:

    Input grain:       customer + payment
    Target grain:      customer + business_date
    Pivot dimension:   status
    Measures:          count and amount
    Categories:        SUCCESS, FAILED, PENDING
    Missing category:  count = 0
    Amount NULL:       preserve unknown semantics
    New category:      fail validation

### 3.2 Example schema

    CREATE TABLE payment_events (
        payment_id TEXT NOT NULL,
        customer_id BIGINT NOT NULL,
        business_date DATE NOT NULL,
        status TEXT NOT NULL,
        amount NUMERIC(20,4),
        PRIMARY KEY (payment_id)
    );

### 3.3 Basic conditional pivot

    SELECT
        customer_id,
        SUM(amount) FILTER (WHERE status = 'SUCCESS') AS success_amount,
        SUM(amount) FILTER (WHERE status = 'FAILED') AS failed_amount,
        SUM(amount) FILTER (WHERE status = 'PENDING') AS pending_amount
    FROM payment_events
    GROUP BY customer_id;

### 3.4 Pivot counts

    SELECT
        customer_id,
        COUNT(*) FILTER (WHERE status = 'SUCCESS') AS success_count,
        COUNT(*) FILTER (WHERE status = 'FAILED') AS failed_count,
        COUNT(*) FILTER (WHERE status = 'PENDING') AS pending_count
    FROM payment_events
    GROUP BY customer_id;

### 3.5 Combined measures

    SELECT
        customer_id,
        COUNT(*) FILTER (WHERE status = 'SUCCESS') AS success_count,
        SUM(amount) FILTER (WHERE status = 'SUCCESS') AS success_amount,
        COUNT(*) FILTER (WHERE status = 'FAILED') AS failed_count,
        SUM(amount) FILTER (WHERE status = 'FAILED') AS failed_amount
    FROM payment_events
    GROUP BY customer_id;

### 3.6 CASE-based portable pattern

    SELECT
        customer_id,
        SUM(CASE WHEN status = 'SUCCESS' THEN amount ELSE 0 END) AS success_amount,
        SUM(CASE WHEN status = 'FAILED' THEN amount ELSE 0 END) AS failed_amount
    FROM payment_events
    GROUP BY customer_id;

### 3.7 Preserve NULL amount semantics

If NULL amount means unknown rather than zero, do not automatically convert it to zero.

A safer design can expose both:

    success_count
    success_amount
    success_unknown_amount_count

This makes missing monetary values visible.

### 3.8 Pivot by date

    SELECT
        customer_id,
        SUM(amount) FILTER (WHERE business_date = DATE '2026-09-01') AS d1_amount,
        SUM(amount) FILTER (WHERE business_date = DATE '2026-09-02') AS d2_amount,
        SUM(amount) FILTER (WHERE business_date = DATE '2026-09-03') AS d3_amount
    FROM payment_events
    GROUP BY customer_id;

Use this only when a fixed reporting period and stable schema are appropriate.

### 3.9 Pivot key-value data

    SELECT
        entity_id,
        MAX(value) FILTER (WHERE key = 'risk_level') AS risk_level,
        MAX(value) FILTER (WHERE key = 'segment') AS segment
    FROM entity_attributes
    GROUP BY entity_id;

`MAX` here is not necessarily a business aggregation.

It is being used to collapse a value after the uniqueness contract has been established.

Validate that one entity/key has at most one authoritative value.

### 3.10 Validate key-value uniqueness

    SELECT
        entity_id,
        key,
        COUNT(*) AS value_count
    FROM entity_attributes
    GROUP BY entity_id, key
    HAVING COUNT(*) > 1;

If rows appear, resolve the collision before treating `MAX` as a safe pivot operation.

### 3.11 Pivot after daily aggregation

    WITH daily AS (
        SELECT
            customer_id,
            business_date,
            status,
            COUNT(*) AS payment_count,
            SUM(amount) AS payment_amount
        FROM payment_events
        GROUP BY customer_id, business_date, status
    )
    SELECT
        customer_id,
        business_date,
        SUM(payment_count) FILTER (WHERE status = 'SUCCESS') AS success_count,
        SUM(payment_count) FILTER (WHERE status = 'FAILED') AS failed_count,
        SUM(payment_amount) FILTER (WHERE status = 'SUCCESS') AS success_amount,
        SUM(payment_amount) FILTER (WHERE status = 'FAILED') AS failed_amount
    FROM daily
    GROUP BY customer_id, business_date;

This makes the input grain to the final pivot explicit.

### 3.12 Detect unexpected categories

    SELECT DISTINCT status
    FROM payment_events
    WHERE status NOT IN ('SUCCESS', 'FAILED', 'PENDING');

Use the result as a data-quality signal rather than silently dropping the category.

### 3.13 Dynamic pivot concept

A dynamic pivot typically follows:

    discover categories
         ↓
    validate categories
         ↓
    generate SQL
         ↓
    execute generated SQL
         ↓
    validate resulting schema

Generated identifiers must be safely quoted by the database mechanism.

Do not concatenate arbitrary source values into SQL.

### 3.14 Python pivot

    from collections import defaultdict

    def pivot_status(rows):
        result = defaultdict(lambda: {
            'SUCCESS': 0,
            'FAILED': 0,
            'PENDING': 0,
        })

        for row in rows:
            customer_id = row['customer_id']
            status = row['status']
            amount = row['amount'] or 0
            result[customer_id][status] += amount

        output = []
        for customer_id, values in result.items():
            output.append({
                'customer_id': customer_id,
                'success_amount': values['SUCCESS'],
                'failed_amount': values['FAILED'],
                'pending_amount': values['PENDING'],
            })

        return output

Only use `or 0` when NULL and zero are intentionally equivalent.

### 3.15 Pandas pivot

    pivoted = df.pivot_table(
        index='customer_id',
        columns='status',
        values='amount',
        aggfunc='sum',
        fill_value=0,
    ).reset_index()

Verify that `fill_value=0` matches the business meaning of missing categories.

### 3.16 Stable column naming

Normalize generated column names:

    success_amount
    failed_amount
    pending_amount

Do not allow arbitrary source text to become uncontrolled database identifiers.

### 3.17 Pivot with multiple dimensions

Suppose the desired output is:

    customer + business_date

with columns by:

    status

Aggregate at the target grain and pivot status.

If the output also needs currency, decide whether currency belongs in the grain or requires prior currency conversion.

### 3.18 Currency-aware pivot

Do not combine amounts across currencies merely because the status is the same.

Possible designs:

    customer + date + currency

or:

    convert to reporting currency first

Then pivot according to the reporting contract.

### 3.19 Safe target publication

Publish pivoted output to a table whose schema is explicit.

Example:

    CREATE TABLE customer_payment_summary (
        customer_id BIGINT NOT NULL,
        business_date DATE NOT NULL,
        success_count BIGINT NOT NULL,
        failed_count BIGINT NOT NULL,
        pending_count BIGINT NOT NULL,
        success_amount NUMERIC(20,4),
        failed_amount NUMERIC(20,4),
        pending_amount NUMERIC(20,4),
        PRIMARY KEY (customer_id, business_date)
    );

## 4. Testing

Pivot tests must validate both values and output grain.

### 4.1 Basic category pivot

Input:

    C1 SUCCESS 100
    C1 FAILED  20

Expected:

    C1 success_amount = 100
    C1 failed_amount = 20

### 4.2 Missing category

Input:

    C1 SUCCESS 100

Verify the expected failed representation:

    0

or:

    NULL

according to the contract.

### 4.3 Multiple rows in one cell

Input:

    C1 SUCCESS 100
    C1 SUCCESS 200

Verify the configured aggregation:

    SUM → 300

or another explicitly defined policy.

### 4.4 Conflicting key-value values

Input:

    C1 segment SMB
    C1 segment ENTERPRISE

Expected:

    collision detected

unless an explicit latest/source-priority rule exists.

### 4.5 NULL measure

Create a matching category with NULL amount.

Verify that the result follows the documented NULL policy.

### 4.6 Zero versus NULL

Create:

    no SUCCESS row

and separately:

    SUCCESS row with amount NULL

Verify that the output distinguishes these states if required.

### 4.7 Unexpected category

Insert:

    REVERSED

Verify that category validation detects it rather than silently dropping it.

### 4.8 Output grain

Create multiple input rows per customer/date/status.

Verify exactly one output row per:

    customer + business_date

### 4.9 Tenant isolation

Use the same customer identifier in multiple tenants.

Verify tenant remains part of the output grain when required.

### 4.10 Aggregation reconciliation

Before pivot:

    total_amount = SUM(amount)

After pivot:

    success_amount + failed_amount + pending_amount

should reconcile only if those categories are exhaustive and NULL semantics are accounted for.

### 4.11 Count reconciliation

Compare:

    source row count

with:

    sum of pivoted category counts

for an exhaustive mutually exclusive category domain.

### 4.12 Idempotence

Run the same input twice.

Expected:

    identical schema
    identical values
    identical output grain

### 4.13 Schema stability

Run the model with expected categories.

Then introduce an unexpected category.

Verify the configured policy:

    fail
    quarantine
    or explicitly evolve schema

### 4.14 Feature leakage

For feature-engineering pivots, insert an event after the feature cutoff.

Verify it does not appear in the feature vector.

## 5. Observability

Pivoting requires visibility into categories, collisions, and schema changes.

### Core metrics

| Metric | Meaning |
|---|---|
| `pivot_input_rows` | Rows entering the transformation |
| `pivot_output_rows` | Rows produced at target grain |
| `pivot_input_groups` | Input groups before pivot |
| `pivot_unexpected_categories` | Categories outside the contract |
| `pivot_collision_groups` | Multiple rows mapping to one output cell |
| `pivot_null_measure_rows` | Rows with missing measures |
| `pivot_missing_category_cells` | Output cells without source category rows |
| `pivot_schema_columns` | Number of output columns |
| `pivot_rule_version` | Active pivot contract |

### Grain monitoring

Compare expected output grain with actual output rows.

Example:

    expected = distinct(customer_id, business_date)
    actual   = output row count

Unexpected divergence indicates aggregation or grouping errors.

### Category monitoring

Track:

- Known category counts.
- Unknown category counts.
- New categories.
- Disappeared categories.

### Collision monitoring

Track groups where multiple rows map to one target cell.

Unexpected collisions often indicate:

- Duplicate source data.
- Wrong grouping grain.
- Missing key columns.
- Join fan-out.

### Schema monitoring

Track changes to:

    column count
    column names
    column types

Dynamic pivots can create silent downstream breaking changes.

### Reconciliation monitoring

For exhaustive category domains, compare:

    source total
    pivoted category total

Investigate unexplained differences.

## 6. Intentional Failure

### Failure 1 — Wrong target grain

Remove a grouping column.

Expected symptom:

- Multiple logical entities collapse into one output row.

Recovery:

Restore the complete target grain.

### Failure 2 — Hide collisions with MAX

Create multiple conflicting values and use `MAX` to collapse them.

Expected symptom:

- A value survives without proving it is authoritative.

Recovery:

Validate uniqueness or define an explicit survivor policy.

### Failure 3 — Treat missing category as zero

Use `COALESCE` on a metric where NULL means unknown.

Expected symptom:

- Missing information is presented as zero.

Recovery:

Restore semantic distinction between missing and zero.

### Failure 4 — Ignore unexpected categories

Insert a new status.

Expected symptom:

- The new category disappears from the pivot.

Recovery:

Detect and handle category evolution explicitly.

### Failure 5 — Pivot after join fan-out

Create a one-to-many join before aggregation.

Expected symptom:

- Pivoted amounts are inflated.

Recovery:

Fix join cardinality or aggregate the right side before joining.

### Failure 6 — Dynamic SQL injection

Use raw category text to construct SQL identifiers.

Expected symptom:

- Invalid SQL or unsafe query construction.

Recovery:

Validate allowed categories and use safe identifier quoting.

### Failure 7 — Future leakage

Include events after the feature cutoff.

Expected symptom:

- Features contain future information.

Recovery:

Apply the cutoff before aggregation and pivoting.

### Failure 8 — Currency mixing

Aggregate different currencies into one amount column.

Expected symptom:

- Financial totals become meaningless.

Recovery:

Keep currency in the grain or convert to a documented reporting currency first.

## 7. Recovery

### Recovery sequence

1. Identify the affected model and reporting period.
2. Capture the input snapshot.
3. Verify target grain.
4. Verify category domain.
5. Check unexpected categories.
6. Check collision groups.
7. Check joins before pivot.
8. Check NULL and zero semantics.
9. Recalculate reconciliation totals.
10. Rebuild the affected output.
11. Validate schema stability.
12. Replay downstream consumers idempotently.

### Recovering inflated totals

1. Compare source totals with pivot totals.
2. Inspect pre-pivot row counts.
3. Inspect join cardinality.
4. Find duplicate target cells.
5. Correct aggregation grain.
6. Recompute the output.

### Recovering missing categories

1. Inspect source category distribution.
2. Compare with the static category contract.
3. Determine whether the new category is valid.
4. Update schema intentionally if required.
5. Add regression coverage.

### Recovering schema drift

If a dynamic category creates a new column:

1. Detect the schema change.
2. Identify downstream consumers.
3. Determine whether the category belongs in the contract.
4. Version the schema.
5. Deploy the change intentionally.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL supports conditional aggregation, filtered aggregates, grouping, constraints, and query planning needed for production pivots.

### 2. dbt

dbt is useful for maintaining stable pivot models, documenting output columns, and testing target grain and accepted categories.

### 3. DuckDB

DuckDB is useful for local pivot experimentation over CSV and Parquet data and for comparing wide and long representations.

## 9. Production Runbook

### Before deployment

- [ ] Define input grain.
- [ ] Define target grain.
- [ ] Define pivot dimension.
- [ ] Define supported categories.
- [ ] Define aggregation for every measure.
- [ ] Define NULL semantics.
- [ ] Define missing-category semantics.
- [ ] Define unexpected-category behavior.
- [ ] Define collision policy.
- [ ] Define schema contract.
- [ ] Define reconciliation checks.
- [ ] Define feature cutoff if used for ML.

### During execution

- [ ] Record input rows.
- [ ] Record output rows.
- [ ] Record collision groups.
- [ ] Record unexpected categories.
- [ ] Record NULL measures.
- [ ] Record missing-category cells.
- [ ] Record schema column count.
- [ ] Record rule version.

### If totals increase unexpectedly

1. Check join fan-out.
2. Check duplicate source cells.
3. Check target grain.
4. Check aggregation functions.
5. Check category overlap.

### If totals decrease unexpectedly

1. Check unexpected categories.
2. Check NULL behavior.
3. Check category filters.
4. Check source completeness.
5. Check reconciliation logic.

### If consumers break

1. Compare output schema with previous version.
2. Identify new or removed categories.
3. Verify whether schema evolution was authorized.
4. Restore the previous contract if necessary.
5. Version the intended change.

## 10. Common Mistakes

### Mistake 1 — Ignoring target grain

Pivoting without a defined output grain can silently merge unrelated records.

### Mistake 2 — Using aggregation as a hiding mechanism

`MAX` or `SUM` should reflect business semantics, not simply make SQL return one row.

### Mistake 3 — Treating NULL as zero

Missing information is not automatically zero.

### Mistake 4 — Ignoring category evolution

New source values can disappear from static pivots.

### Mistake 5 — Dynamic columns without schema governance

Changing columns can break downstream consumers.

### Mistake 6 — Pivoting after an incorrect join

Aggregation can hide join multiplication.

### Mistake 7 — Averaging rates

Category rates should generally be derived from appropriate counts and denominators.

### Mistake 8 — Mixing currencies

Amounts require a currency contract.

### Mistake 9 — Forgetting reversibility

Pivoting often loses row-level detail.

### Mistake 10 — Ignoring feature cutoffs

Wide feature tables can leak future information.

### Mistake 11 — Allowing uncontrolled identifiers

Dynamic category values must be validated and safely quoted.

### Mistake 12 — No reconciliation

Every production pivot should prove what happened to the source population.

## 11. Definition of Done

The pivot transformation is complete when you can:

- [ ] Define input and target grain.
- [ ] Explain row-to-column transformation.
- [ ] Use conditional aggregation.
- [ ] Pivot counts and measures.
- [ ] Handle multiple source rows per output cell.
- [ ] Distinguish NULL from zero.
- [ ] Detect unexpected categories.
- [ ] Explain static versus dynamic pivots.
- [ ] Maintain a schema contract.
- [ ] Pivot key-value attributes safely.
- [ ] Validate key-value uniqueness.
- [ ] Reconcile source and pivoted totals.
- [ ] Handle currency-aware measures.
- [ ] Prevent feature leakage.
- [ ] Test target grain.
- [ ] Test collisions and category evolution.
- [ ] Monitor schema drift.
- [ ] Intentionally break pivot semantics.
- [ ] Recover inflated or missing totals.
- [ ] Explain how PostgreSQL, dbt, and DuckDB support production pivoting.

## 12. What You Learned

Pivoting is a controlled change from long representation to wide representation.

The production workflow is:

    DEFINE INPUT GRAIN
         ↓
    DEFINE TARGET GRAIN
         ↓
    DEFINE PIVOT DIMENSION
         ↓
    DEFINE CATEGORY CONTRACT
         ↓
    DEFINE AGGREGATION
         ↓
    HANDLE NULL / ZERO SEMANTICS
         ↓
    APPLY PIVOT
         ↓
    VALIDATE COLLISIONS
         ↓
    RECONCILE SOURCE TOTALS
         ↓
    VALIDATE SCHEMA

> **A production pivot is correct only when the target grain, category domain, aggregation policy, NULL semantics, and schema contract are explicit.**

### Next recipe

**T32 — Unpivoting**