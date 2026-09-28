# T17 — SQL SELECT Transformations

> **Goal:** Use SQL `SELECT` as a controlled transformation layer to project columns, derive values, rename fields, apply expressions, preserve grain, and produce a deterministic output contract without accidentally filtering, joining, aggregating, or changing row cardinality.

`SELECT` is one of the most common transformation mechanisms in Data Engineering. It is also one of the easiest places to hide business logic.

Examples:

    raw amount → normalized amount
    first_name + last_name → display_name
    source_code → derived category
    timestamp → event_date
    source columns → canonical column names
    several input columns → one calculated metric

The central risk is that a query can look like simple column selection while silently changing types, semantics, null behavior, precision, or output grain.

The central rule is:

> **Every `SELECT` transformation should have an explicit output contract: what each output column means, how it is derived, what happens with NULLs, and whether input row cardinality is preserved.**

---

## 1. Problem Recognition

### 1.1 Typical SELECT transformation

Suppose staging contains:

    customer_id
    first_name
    last_name
    amount_cents
    event_timestamp

The canonical model requires:

    customer_id
    full_name
    amount
    event_date

A projection can transform the record:

    SELECT
        customer_id,
        first_name || ' ' || last_name AS full_name,
        amount_cents / 100.0 AS amount,
        event_timestamp::date AS event_date
    FROM staging;

Each output column is a transformation contract.

### 1.2 SELECT is not WHERE

`SELECT` determines what columns and expressions appear in the output.

`WHERE` determines which rows participate.

Example:

    SELECT customer_id, amount
    FROM payments
    WHERE status = 'COMPLETED';

`SELECT` projects:

    customer_id
    amount

`WHERE` filters:

    status = COMPLETED

T18 will cover filtering in detail. T17 focuses on projection and expressions.

### 1.3 SELECT is not JOIN

A `JOIN` changes the input relation by combining rows from multiple relations.

A basic `SELECT` from one relation should normally preserve its input row count.

Investigate when a supposedly simple projection unexpectedly changes cardinality.

### 1.4 SELECT is not GROUP BY aggregation

Aggregation changes grain:

    transaction rows → daily totals

A projection should normally preserve the input grain.

If a query contains:

    SUM(...)
    COUNT(...)
    GROUP BY

you have entered aggregation territory rather than simple projection.

### 1.5 Red flags

Investigate when:

- `SELECT *` is used in a production model without a schema contract
- a derived column has undocumented business meaning
- implicit casts appear in important calculations
- NULL behavior is accidental
- numeric precision changes silently
- column aliases are inconsistent
- a projection unexpectedly changes row count
- expressions are duplicated across many models
- downstream users cannot explain how a column was calculated.

---

## 2. Concept and Reasoning

### 2.1 Define the SELECT contract

Before writing SQL, document:

    input table
    input grain
    output table/model
    output grain
    selected columns
    derived columns
    data types
    NULL behavior
    units
    rounding rules
    naming rules
    lineage

Example:

| Output column | Meaning | Rule |
|---|---|---|
| customer_id | canonical customer identifier | copy from source |
| full_name | display name | first + last |
| amount | monetary amount in USD | cents / 100 |
| event_date | calendar date of event | event timestamp converted to date |

### 2.2 Projection should preserve grain

If the input contains:

    10,000 payment rows

then a pure projection should normally produce:

    10,000 output rows

provided there is no `DISTINCT`, grouping, set operation, or other row-changing mechanism.

This makes row-count reconciliation a useful test.

### 2.3 Explicit columns beat SELECT *

Prefer:

    SELECT
        customer_id,
        email,
        created_at
    FROM customers;

over:

    SELECT *
    FROM customers;

`SELECT *` can silently expose newly added source columns to downstream models.

That can create:

    schema drift
    unexpected PII propagation
    larger transfers
    unstable model interfaces
    accidental coupling to source schema

`SELECT *` can be appropriate during exploration, but production interfaces should normally be explicit.

### 2.4 Aliases are part of the contract

Use meaningful output names:

    amount_cents / 100.0 AS amount

rather than exposing an implementation expression as the downstream interface.

Good aliases describe meaning, not just calculation mechanics.

### 2.5 Derived columns should have one clear meaning

Consider:

    amount / 100 AS revenue

That expression is ambiguous if `amount` is already dollars.

Before deriving a column, establish:

    input unit
    output unit
    currency
    precision
    rounding behavior

T08 covers numeric transformation in detail. T17 applies those rules inside SQL projections.

### 2.6 SQL expressions have NULL semantics

Consider:

    first_name || ' ' || last_name

If `last_name` is NULL, PostgreSQL's concatenation operator can produce NULL.

Depending on the contract, a safer expression may be:

    concat_ws(' ', first_name, last_name)

The correct expression depends on the intended semantics.

Do not treat NULL behavior as an implementation detail.

### 2.7 CASE expressions encode business rules

A `CASE` expression can create a canonical derived field:

    CASE
        WHEN amount >= 10000 THEN 'HIGH'
        WHEN amount >= 1000 THEN 'MEDIUM'
        ELSE 'LOW'
    END AS amount_band

Order matters when conditions overlap.

Document:

    precedence
    boundary values
    NULL behavior
    ELSE behavior

### 2.8 ELSE is a data-quality decision

Consider:

    CASE
        WHEN status = 'A' THEN 'ACTIVE'
        WHEN status = 'I' THEN 'INACTIVE'
        ELSE 'UNKNOWN'
    END

`ELSE 'UNKNOWN'` may be correct, but it can also hide new source codes.

Depending on the contract, unknown values may need to:

    remain NULL
    become UNKNOWN
    fail validation
    enter quarantine

T10 covers code/status mapping. T17 focuses on expressing the rule safely in SQL.

### 2.9 Type conversion should be explicit

Prefer deliberate casts:

    CAST(amount_text AS NUMERIC(18,2))

or PostgreSQL syntax:

    amount_text::NUMERIC(18,2)

Do not depend on implicit conversion for important interfaces.

Explicit types make the transformation easier to review and test.

### 2.10 Date extraction can change meaning

Consider:

    event_timestamp::date

This produces a calendar date according to the timestamp's semantics.

If the source timestamp is an instant and the business date depends on a specific timezone, the transformation should establish that timezone before extracting the date.

Example:

    (event_timestamp AT TIME ZONE 'Europe/Berlin')::date

T06 and T07 cover the underlying temporal mechanisms.

### 2.11 DISTINCT is not harmless cleanup

`DISTINCT` changes cardinality.

Example:

    SELECT DISTINCT customer_id
    FROM payments;

Ten payment rows for one customer become one row.

That is a transformation with explicit semantics, not merely a formatting operation.

Do not add `DISTINCT` to hide duplicate records without understanding why the duplicates exist.

### 2.12 Determinism matters

A SELECT expression should produce the same output for the same input and reference state.

Avoid uncontrolled dependencies such as:

    random()
    current_timestamp
    unstable external functions

when deterministic historical replay is required.

If a time-dependent function is required, define whether the value represents:

    event time
    processing time
    pipeline run time

---

## 3. Implementation

### 3.1 Basic projection

Start with explicit columns:

    SELECT
        customer_id,
        email,
        created_at
    FROM staging.customers;

This creates a clear output contract and preserves the source row grain.

### 3.2 Rename columns

Use aliases:

    SELECT
        customer_id AS customer_key,
        created_at AS customer_created_at
    FROM staging.customers;

Keep naming conventions consistent across the project.

### 3.3 Derive a calculated column

Example:

    SELECT
        payment_id,
        amount_cents,
        amount_cents / 100.0 AS amount
    FROM staging.payments;

Verify that the output numeric type and unit match the target contract.

### 3.4 Handle NULLs deliberately

Example:

    SELECT
        customer_id,
        concat_ws(' ', first_name, last_name) AS full_name
    FROM staging.customers;

Do not replace every NULL with an empty string automatically.

An empty string can mean something different from missing data.

### 3.5 CASE transformation

Example:

    SELECT
        payment_id,
        amount,
        CASE
            WHEN amount IS NULL THEN NULL
            WHEN amount >= 10000 THEN 'HIGH'
            WHEN amount >= 1000 THEN 'MEDIUM'
            ELSE 'LOW'
        END AS amount_band
    FROM staging.payments;

Notice that NULL is handled explicitly.

### 3.6 Explicit type conversion

Example:

    SELECT
        customer_id,
        CAST(age_text AS INTEGER) AS age
    FROM staging.customers;

Only do this when the source contract guarantees the value is convertible.

If invalid values are possible, validation or a safe parsing strategy should occur before publication.

### 3.7 Conditional arithmetic

Example:

    SELECT
        payment_id,
        CASE
            WHEN currency = 'USD' THEN amount
            WHEN currency = 'EUR' THEN amount * eur_to_usd_rate
            ELSE NULL
        END AS amount_usd
    FROM staging.payments;

This should only be used when the FX rate source and temporal semantics are explicitly defined.

Otherwise the expression can produce a numerically valid but semantically incorrect result.

### 3.8 PostgreSQL example

Suppose the staging table contains:

    CREATE TABLE staging.orders (
        order_id BIGINT,
        customer_id BIGINT,
        first_name TEXT,
        last_name TEXT,
        amount_cents BIGINT,
        status_code TEXT,
        created_at TIMESTAMPTZ
    );

A canonical projection can be:

    SELECT
        order_id,
        customer_id,
        concat_ws(' ', first_name, last_name) AS customer_name,
        amount_cents / 100.0 AS amount,
        CASE status_code
            WHEN 'P' THEN 'PENDING'
            WHEN 'C' THEN 'COMPLETED'
            WHEN 'F' THEN 'FAILED'
            ELSE 'UNKNOWN'
        END AS status,
        created_at,
        created_at::date AS event_date
    FROM staging.orders;

The target contract should document each expression.

### 3.9 Use a CTE when intermediate logic improves correctness

A Common Table Expression can make complex projection logic readable:

    WITH normalized AS (
        SELECT
            order_id,
            amount_cents / 100.0 AS amount,
            status_code
        FROM staging.orders
    )
    SELECT
        order_id,
        amount,
        CASE status_code
            WHEN 'P' THEN 'PENDING'
            WHEN 'C' THEN 'COMPLETED'
            ELSE 'UNKNOWN'
        END AS status
    FROM normalized;

Do not introduce CTEs merely for style. Use them when separating stages makes the logic easier to validate.

### 3.10 Preserve source columns needed for auditability

A canonical model does not need every raw column.

But retain identifiers and timestamps needed to trace the output:

    source_record_id
    source_system
    event_timestamp
    ingestion_timestamp

Provenance requirements should be explicit rather than accidental.

### 3.11 Validate output shape

Before publication, verify:

    expected columns exist
    expected data types exist
    expected row count is reasonable
    key columns remain populated
    derived units are correct
    no unintended columns were introduced

Example:

    SELECT COUNT(*) FROM staging.orders;

Compare with:

    SELECT COUNT(*) FROM transformed_orders;

A pure projection should normally preserve the count.

### 3.12 SQL implementation pattern

A useful production pattern is:

    source
      ↓
    explicit SELECT
      ↓
    derived expressions
      ↓
    output contract validation
      ↓
    target

Keep the projection deterministic and explainable.

---

## 4. Testing

### 4.1 Minimum test matrix

| Test | Expected result |
|---|---|
| normal record | all expected columns produced |
| NULL input | documented NULL behavior |
| boundary value | correct CASE branch |
| invalid cast | rejected or handled according to contract |
| numeric calculation | correct unit and precision |
| timestamp transformation | correct business date |
| unknown status | documented ELSE behavior |
| duplicate input rows | preserved unless explicitly changed |
| same input repeated | identical output |
| newly added source column | does not silently alter explicit projection |

### 4.2 Row-count test

For a pure projection:

    input_count = output_count

unless the query intentionally contains a cardinality-changing construct.

This catches accidental additions such as:

    DISTINCT
    GROUP BY
    JOIN
    UNION

### 4.3 Column-contract test

Verify the output schema:

    column names
    order if contractually relevant
    data types
    nullability expectations

Do not rely on visual inspection of query output.

### 4.4 Boundary tests

For:

    CASE
        WHEN amount >= 10000 THEN 'HIGH'
        WHEN amount >= 1000 THEN 'MEDIUM'
        ELSE 'LOW'
    END

test:

    999.99
    1000
    1000.01
    9999.99
    10000
    10000.01

Boundary tests catch overlapping and missing conditions.

### 4.5 NULL tests

Test each expression with:

    NULL
    empty string
    valid value
    invalid value

Do not assume SQL's NULL behavior matches Python or application behavior.

### 4.6 Type tests

Verify important expressions produce the intended type.

For example:

    amount_cents / 100.0

should be checked against the target numeric contract rather than assuming the database will choose the desired representation.

### 4.7 Determinism test

Execute the same SELECT twice against the same source snapshot.

Expected:

    result A = result B

If a time-dependent function is intentionally present, the test should inject or control the relevant run time.

### 4.8 Idempotence test

Running the transformation twice against the same immutable input should not change the result.

Verify:

    same row count
    same values
    same data types
    same derived fields

---

## 5. Observability

A SELECT transformation is easier to operate when its output contract is measurable.

Track where appropriate:

    input_row_count
    output_row_count
    null_rate_by_important_column
    invalid_value_count
    unknown_code_count
    cast_failure_count
    min/max numeric values
    output_schema_version
    transformation_version

### 5.1 Row-count reconciliation

For a pure projection:

    source rows = transformed rows

A mismatch should trigger investigation.

### 5.2 Derived-column distributions

Monitor derived categorical fields:

    status
    amount_band
    customer_type

A new unexpected category may indicate source drift or an incorrect CASE rule.

### 5.3 Numeric sanity checks

Monitor:

    minimum
    maximum
    average
    zero rate
    NULL rate

for important numeric outputs.

A factor-of-100 error in money conversion can pass SQL execution successfully.

### 5.4 Schema drift detection

Explicit projection reduces schema drift, but source changes can still affect referenced columns.

Monitor:

    missing columns
    type changes
    renamed columns
    unexpected nullability changes

### 5.5 Do not log sensitive data

Prefer metadata:

    query/model name
    run ID
    row counts
    rule version
    schema version

Avoid logging complete customer records merely to diagnose a projection.

---

## 6. Intentional Failure

### Failure 1 — SELECT * schema drift

Change the source table by adding a column.

Observe how:

    SELECT *

can change the downstream shape without changing the transformation code.

Expected lesson:

> Production interfaces should not depend accidentally on every source column.

### Failure 2 — NULL concatenation

Provide:

    first_name = 'Farhan'
    last_name = NULL

Use:

    first_name || ' ' || last_name

Expected lesson:

> SQL NULL propagation must be part of the transformation contract.

### Failure 3 — Hidden unit conversion error

Treat cents as dollars.

Example:

    amount_cents = 1500

incorrect output:

    amount = 1500

correct semantic output:

    amount = 15.00

Expected lesson:

> SQL can execute successfully while producing semantically incorrect data.

### Failure 4 — Missing CASE boundary

Use:

    WHEN amount > 1000

instead of:

    WHEN amount >= 1000

Test exactly 1000.

Expected lesson:

> Boundary conditions are part of business logic.

### Failure 5 — DISTINCT hides duplicates

Introduce duplicate source rows and add:

    SELECT DISTINCT ...

Expected lesson:

> Removing duplicate output is not the same as understanding duplicate input.

### Failure 6 — Timezone mistake

Extract a date directly from an instant when the business date depends on a local timezone.

Expected lesson:

> Date extraction can change business meaning.

### Failure 7 — Implicit cast

Feed malformed numeric text into an implicit conversion path.

Expected lesson:

> Important type boundaries should be explicit and validated.

---

## 7. Recovery

### 7.1 Stop incorrect publication

If a projection is producing incorrect values, isolate the affected model or downstream publication where operationally possible.

Do not continue publishing known-invalid derived data.

### 7.2 Identify the transformation version

Record:

    pipeline run ID
    SQL/model version
    schema version
    source snapshot
    affected time range

### 7.3 Compare source and transformed values

For a representative population, compare:

    source columns
    derived columns
    expected rule output

Classify the defect:

    expression error
    type error
    NULL handling error
    unit error
    boundary error
    source schema change

### 7.4 Fix and version the rule

Do not silently edit historical logic without documenting the change.

Record:

    old transformation version
    new transformation version
    affected records
    expected correction

### 7.5 Replay from trusted input

Re-run from the preserved staging or raw input.

Verify:

    row count
    schema
    derived values
    quality metrics
    downstream reconciliation

### 7.6 Audit downstream consumers

Determine whether incorrect values reached:

    dimensions
    facts
    reports
    dashboards
    APIs
    exports

Correct downstream state according to its recovery contract.

---

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

Know:

    SELECT
    CASE
    CAST
    NULL handling
    expressions
    CTEs
    EXPLAIN

PostgreSQL provides the execution engine for production SQL transformations.

### 8.2 dbt

Know how dbt can organize SQL transformations with:

    models
    tests
    documentation
    dependency graphs
    version control

Use dbt to manage transformation projects, but understand the SQL mechanism underneath.

### 8.3 DuckDB

Know how DuckDB can be used for:

    local analytical SQL
    Parquet transformations
    reproducible development tests
    fast exploration of pipeline datasets

DuckDB is useful for developing and validating SQL transformations before deploying them to a production warehouse or database.

---

## 9. Production Runbook

### Before deployment

Verify:

- input grain is documented
- output grain is documented
- every production column is explicitly selected
- aliases are meaningful
- data types are intentional
- NULL behavior is documented
- units are documented
- CASE precedence is documented
- ELSE behavior is documented
- timezone semantics are documented where relevant
- row-count expectations are known
- schema tests exist
- representative boundary cases are tested.

### During execution

Monitor:

    input row count
    output row count
    NULL rates
    derived-value distributions
    cast failures
    unknown categories
    schema/version metadata

### If output looks wrong

1. Identify the transformation version.
2. Compare source and output rows.
3. Check whether cardinality changed.
4. Check NULL propagation.
5. Check units and data types.
6. Check CASE boundaries.
7. Check timezone interpretation.
8. Compare against the documented output contract.
9. Fix and version the SQL.
10. Replay from trusted input.
11. Reconcile downstream outputs.

---

## 10. Common Mistakes

### Mistake 1 — Using SELECT * in stable interfaces

Why it fails:

Source schema changes can become downstream schema changes automatically.

### Mistake 2 — Treating NULL as empty

Why it fails:

Missing data and empty values can have different meanings.

### Mistake 3 — Relying on implicit casts

Why it fails:

Database conversion behavior may not match the target contract.

### Mistake 4 — Hiding business rules inside unreadable expressions

Why it fails:

Reviewers and operators cannot verify the transformation.

### Mistake 5 — Adding DISTINCT to hide source problems

Why it fails:

Duplicates disappear without their cause being understood.

### Mistake 6 — Ignoring units

Why it fails:

A numerically valid result can still be 100 or 1000 times wrong.

### Mistake 7 — Forgetting timezone semantics

Why it fails:

The extracted date can belong to a different business day.

### Mistake 8 — Missing ELSE semantics

Why it fails:

New source values can silently become NULL or an unintended category.

### Mistake 9 — Testing only happy-path rows

Why it fails:

NULLs, boundaries, invalid values, and schema changes are where production defects often appear.

### Mistake 10 — Treating executable SQL as self-documenting

Why it fails:

SQL explains how a value is calculated, not necessarily why that calculation is the correct business rule.

---

## 11. Definition of Done

A SQL SELECT transformation is complete when:

- [ ] input grain is documented
- [ ] output grain is documented
- [ ] production columns are explicitly selected
- [ ] aliases are meaningful
- [ ] derived expressions have documented meaning
- [ ] data types are explicit where important
- [ ] units are documented
- [ ] NULL behavior is defined
- [ ] CASE precedence is tested
- [ ] ELSE behavior is defined
- [ ] timezone semantics are correct where applicable
- [ ] row-count behavior is understood
- [ ] output schema is tested
- [ ] normal cases are tested
- [ ] NULL cases are tested
- [ ] boundary cases are tested
- [ ] invalid-value behavior is tested
- [ ] determinism is tested
- [ ] idempotence is tested
- [ ] observability exists
- [ ] intentional failure has been exercised
- [ ] recovery has been tested
- [ ] SQL/model version is traceable
- [ ] downstream reconciliation is understood.

---

## 12. What You Learned

SQL `SELECT` is not merely a way to retrieve columns.

You learned how to:

    define a projection contract
    preserve row grain
    select explicit production columns
    create deterministic derived fields
    handle NULL semantics
    use CASE safely
    make type conversions explicit
    preserve units and precision
    handle temporal expressions correctly
    detect accidental cardinality changes
    test SQL transformations
    observe derived data
    intentionally break a projection
    recover from incorrect transformation logic
    operate SQL transformations in production

The key lesson is:

> **A production SELECT is an interface. Treat its columns, types, NULL behavior, units, and grain as a contract rather than as incidental query output.**

### Next Recipe

**T18 — WHERE Filtering**

After learning controlled SQL projection, the next recipe focuses specifically on row selection and filtering semantics.