# T32 — Unpivoting

> **Goal:** Learn how to transform wide columns into long records while preserving meaning, grain, NULL semantics, data types, and downstream schema contracts.

## 1. Problem Recognition

Unpivoting converts multiple columns representing one logical dimension into rows.

Typical production cases include:

- Monthly columns becoming date/value rows.
- Status-specific columns becoming category/value rows.
- Wide survey answers becoming question/value records.
- Feature columns becoming feature-name/value records.
- Legacy reports becoming normalized analytical tables.
- Wide operational extracts becoming event-like structures.

Example input:

    customer_id | success_amount | failed_amount
    ------------+----------------+---------------
    C1          | 100            | 20
    C2          | 80             | 0

Possible output:

    customer_id | status  | amount
    ------------+---------+-------
    C1          | SUCCESS | 100
    C1          | FAILED  | 20
    C2          | SUCCESS | 80
    C2          | FAILED  | 0

The important question is:

    What does each output row represent?

### Recognition questions

Before unpivoting, ask:

1. What is the source grain?
2. What is the target grain?
3. Which columns encode a hidden dimension?
4. What category does each column represent?
5. Should NULL columns produce rows?
6. Should zero-valued columns produce rows?
7. Are all source columns semantically equivalent?
8. Do source columns have compatible data types?
9. Can new columns appear?
10. Is the transformation intended to be reversible?
11. What metadata identifies the source column?
12. Are the resulting category values controlled?

## 2. Concept and Reasoning

### 2.1 Unpivot as a grain transformation

Suppose the source grain is:

    customer + business_date

and the output contains:

    customer + business_date + status

then unpivoting expands one source row into multiple category rows.

### 2.2 Wide versus long

Wide:

    customer | success | failed | pending

Long:

    customer | status  | amount

Long representation is often easier to extend because adding a new category does not necessarily require a new column.

### 2.3 Unpivoting is not simply renaming columns

Each source column contains two pieces of information:

    column name → category
    cell value  → measure

Unpivoting makes the category explicit as data.

### 2.4 Target grain

Example:

    input grain  = customer + date
    output grain = customer + date + metric_name

The transformation therefore multiplies rows by the number of selected metric columns.

### 2.5 Row multiplication is intentional

If one source row contains five metrics, an unpivot can produce five output rows.

This is not accidental join fan-out.

It is the intended representation change.

Still, the expansion factor should be measured and validated.

### 2.6 NULL semantics

Suppose:

    success_amount = 100
    failed_amount = NULL

Possible output:

    SUCCESS | 100
    FAILED  | NULL

or:

    SUCCESS | 100

If NULL means unknown, dropping the row may destroy information.

If NULL means not applicable, suppressing the row may be correct.

Define the policy explicitly.

### 2.7 Zero semantics

Zero is different from NULL.

Example:

    failed_amount = 0

may mean:

    the category exists and the measured value is zero.

If you filter out zero values during unpivoting, you may change the analytical meaning.

### 2.8 Missing category versus missing value

These are different:

    category absent

and:

    category present with NULL value

An unpivot transformation should preserve the distinction when downstream consumers need it.

### 2.9 Static unpivot

A static unpivot explicitly lists source columns.

Advantages:

- Stable schema.
- Easy review.
- Predictable output.
- Strong downstream contracts.

Disadvantage:

- New source columns require transformation changes.

### 2.10 Dynamic unpivot

A dynamic unpivot discovers source columns and converts them automatically.

Advantages:

- Handles evolving wide schemas.

Disadvantages:

- More complex governance.
- Harder schema testing.
- Potentially unsafe generated SQL.
- Unclear semantic meaning for arbitrary columns.

Dynamic behavior should be constrained by an explicit column contract.

### 2.11 Category mapping

Source columns may not have ideal analytical names.

Example:

    success_amt → SUCCESS
    fail_amt    → FAILED

Maintain an explicit mapping when source names and business categories differ.

### 2.12 Do not infer semantics from arbitrary names

A column called:

    status_1

does not necessarily mean:

    SUCCESS

Use metadata or a governed mapping rather than guessing.

### 2.13 Multiple measures

Suppose the source contains:

    success_count
    success_amount
    failed_count
    failed_amount

A naïve unpivot produces mixed measures.

Better target designs include:

    category + measure_name + value

or:

    category + count + amount

depending on downstream usage.

### 2.14 Measure type compatibility

Unpivoting multiple numeric columns into one value column requires compatible types.

Do not mix:

    numeric amount
    timestamp
    text

into one value column without a deliberate type representation.

### 2.15 Typed long models

One option is:

    entity_id
    category
    amount

Another is:

    entity_id
    attribute_name
    attribute_value_text

The first preserves numeric semantics.

The second supports heterogeneous attributes but moves type validation downstream.

### 2.16 Unpivoting and dates

Wide monthly columns:

    jan_amount
    feb_amount
    mar_amount

can become:

    month | amount

This creates a proper time dimension in the data.

### 2.17 Date parsing

Do not infer dates from column names without validating:

    year
    month
    fiscal period
    timezone

A column called `jan_amount` may not mean January 2026.

### 2.18 Unpivoting and metric identity

If columns represent different metrics:

    revenue
    cost
    profit

the output should preserve the metric identity:

    metric_name
    metric_value

Do not merge metrics into one category column without a clear model.

### 2.19 Unpivoting and category identity

If columns represent categories:

    success_amount
    failed_amount

then the output can use:

    status
    amount

Category and measure are different concepts.

### 2.20 Unpivoting after pivoting

If a pivot created:

    success_amount
    failed_amount
    pending_amount

unpivoting can restore:

    status + amount

but only if the pivot preserved enough information and used an invertible aggregation.

### 2.21 Pivot/unpivot is not always perfectly reversible

If a pivot aggregated:

    100
    200

into:

    300

unpivoting returns 300, not the original two records.

Aggregation is lossy.

### 2.22 Unpivoting and auditability

Record the source column when required:

    source_column = 'success_amount'

This can help trace output rows back to the original wide field.

### 2.23 Unpivoting and source lineage

Useful metadata can include:

    source_record_id
    source_column
    transformation_version
    extraction_batch_id

This is particularly useful for legacy-source migrations.

### 2.24 Unpivoting and schema evolution

New wide columns can create new long categories.

That may be easier for downstream systems than adding physical columns.

But new categories still require semantic governance.

### 2.25 Unpivoting and sparse data

A wide table with hundreds of mostly NULL columns can become very large after unpivoting if NULL rows are retained.

Decide whether sparse attributes should remain explicit.

### 2.26 Expansion factor

If there are 100 source columns and 1 million source rows, an unpivot can theoretically produce up to:

    100 million rows

before NULL filtering or other policies.

Estimate output volume before deployment.

### 2.27 Unpivoting and downstream joins

After unpivoting, category becomes data.

Downstream joins can then use:

    entity_id + category

This can simplify extensible models.

### 2.28 Unpivoting and aggregation

Once long, category-specific metrics can be grouped naturally:

    GROUP BY entity_id, category

This often simplifies subsequent analytical transformations.

### 2.29 Unpivoting and normalization

Unpivoting often moves a model closer to first-normal-form style data:

    repeated attributes → rows

But analytical models do not need to be fully normalized.

Choose the representation based on access patterns.

### 2.30 Unpivoting and feature stores

Feature systems may prefer:

    entity_id + feature_name + feature_value

over one physical column for every feature.

However, typed feature values and point-in-time correctness still need explicit contracts.

## 3. Implementation

### 3.1 Define the contract

Example:

    Source grain:      customer + business_date
    Target grain:      customer + business_date + status
    Source columns:    success_amount, failed_amount, pending_amount
    Category mapping:  explicit
    NULL policy:       preserve row
    Zero policy:       preserve row
    Output type:       NUMERIC
    Source lineage:    retained

### 3.2 Example schema

    CREATE TABLE customer_payment_wide (
        customer_id BIGINT NOT NULL,
        business_date DATE NOT NULL,
        success_amount NUMERIC(20,4),
        failed_amount NUMERIC(20,4),
        pending_amount NUMERIC(20,4),
        PRIMARY KEY (customer_id, business_date)
    );

### 3.3 PostgreSQL with CROSS JOIN LATERAL

PostgreSQL can represent a controlled unpivot using `VALUES`:

    SELECT
        w.customer_id,
        w.business_date,
        x.status,
        x.amount
    FROM customer_payment_wide AS w
    CROSS JOIN LATERAL (
        VALUES
            ('SUCCESS', w.success_amount),
            ('FAILED', w.failed_amount),
            ('PENDING', w.pending_amount)
    ) AS x(status, amount);

This preserves a row even when `amount` is NULL.

### 3.4 Filter NULL rows intentionally

If the business contract says NULL means category not applicable:

    SELECT
        w.customer_id,
        w.business_date,
        x.status,
        x.amount
    FROM customer_payment_wide AS w
    CROSS JOIN LATERAL (
        VALUES
            ('SUCCESS', w.success_amount),
            ('FAILED', w.failed_amount),
            ('PENDING', w.pending_amount)
    ) AS x(status, amount)
    WHERE x.amount IS NOT NULL;

Do not use this pattern until NULL semantics are defined.

### 3.5 UNION ALL pattern

A portable pattern is:

    SELECT customer_id, business_date, 'SUCCESS' AS status, success_amount AS amount
    FROM customer_payment_wide
    UNION ALL
    SELECT customer_id, business_date, 'FAILED', failed_amount
    FROM customer_payment_wide
    UNION ALL
    SELECT customer_id, business_date, 'PENDING', pending_amount
    FROM customer_payment_wide;

`UNION ALL` is important because the categories are intentionally separate rows.

Using `UNION` would add duplicate elimination work and could incorrectly remove legitimate identical rows.

### 3.6 Preserve zero values

If zero is meaningful, do not filter it:

    WHERE amount <> 0

unless the business contract explicitly says zero-valued categories should disappear.

### 3.7 Explicit category mapping

    SELECT
        customer_id,
        business_date,
        'SUCCESS' AS status,
        success_amt AS amount
    FROM source_table
    UNION ALL
    SELECT
        customer_id,
        business_date,
        'FAILED' AS status,
        fail_amt AS amount
    FROM source_table;

Explicit mapping is preferable when source column names do not equal business categories.

### 3.8 Unpivot monthly columns

    SELECT
        customer_id,
        DATE '2026-01-01' AS month_start,
        jan_amount AS amount
    FROM monthly_revenue
    UNION ALL
    SELECT
        customer_id,
        DATE '2026-02-01',
        feb_amount
    FROM monthly_revenue
    UNION ALL
    SELECT
        customer_id,
        DATE '2026-03-01',
        mar_amount
    FROM monthly_revenue;

Keep the date mapping explicit and versioned.

### 3.9 Unpivot multiple measures

One safe representation is:

    customer_id
    business_date
    status
    metric_name
    metric_value

Example:

    SUCCESS | COUNT  | 10
    SUCCESS | AMOUNT | 1000

This is extensible but may require typed metric storage or separate measure columns.

### 3.10 Typed category/value model

A cleaner analytical model for one numeric measure is:

    CREATE TABLE payment_status_metric (
        customer_id BIGINT NOT NULL,
        business_date DATE NOT NULL,
        status TEXT NOT NULL,
        amount NUMERIC(20,4),
        PRIMARY KEY (customer_id, business_date, status)
    );

The primary key enforces the intended long-form grain.

### 3.11 Preserve source lineage

Add:

    source_record_id
    source_column
    batch_id

when traceability is required.

Example:

    SELECT
        customer_id,
        business_date,
        'SUCCESS' AS status,
        success_amount AS amount,
        'success_amount' AS source_column
    FROM customer_payment_wide;

### 3.12 Python implementation

    def unpivot(rows, columns):
        output = []

        for row in rows:
            for column, category in columns.items():
                output.append({
                    'customer_id': row['customer_id'],
                    'business_date': row['business_date'],
                    'category': category,
                    'value': row[column],
                    'source_column': column,
                })

        return output

Example mapping:

    columns = {
        'success_amount': 'SUCCESS',
        'failed_amount': 'FAILED',
        'pending_amount': 'PENDING',
    }

### 3.13 Pandas melt

    long_df = df.melt(
        id_vars=['customer_id', 'business_date'],
        value_vars=['success_amount', 'failed_amount', 'pending_amount'],
        var_name='source_column',
        value_name='amount',
    )

Then map:

    category_map = {
        'success_amount': 'SUCCESS',
        'failed_amount': 'FAILED',
        'pending_amount': 'PENDING',
    }

    long_df['status'] = long_df['source_column'].map(category_map)

### 3.14 Validate mapping completeness

Every selected source column should map to exactly one category.

Do not silently allow:

    source column → NULL category

unless that is intentional.

### 3.15 Dynamic metadata-driven unpivot

A governed metadata table can define:

    source_column
    output_category
    data_type
    active_flag

Then the pipeline can validate that every expected column is mapped before execution.

### 3.16 Safe dynamic SQL

If SQL must be generated dynamically:

1. Discover source columns from trusted metadata.
2. Validate against an allow-list.
3. Generate identifiers using database-safe quoting.
4. Validate resulting schema.
5. Record the mapping version.

Never treat arbitrary source column names as trusted SQL syntax.

### 3.17 Output-volume estimation

Estimate:

    source_rows × selected_columns

then subtract rows removed by the NULL or filtering policy.

Use this estimate for capacity planning.

### 3.18 Atomic publication

Build the long-form output in a staging relation.

Then validate:

    row count
    key uniqueness
    category domain
    NULL policy

before publishing the final table.

## 4. Testing

Unpivot tests must prove row expansion, mapping, semantics, and target grain.

### 4.1 Basic expansion

Input:

    C1 | success=100 | failed=20

Expected:

    C1 SUCCESS 100
    C1 FAILED  20

### 4.2 Row-count expansion

If every source row has three selected columns and NULL rows are retained:

    output_rows = source_rows × 3

Verify this invariant.

### 4.3 NULL preservation

Input:

    success = 100
    failed  = NULL

Verify whether the output contains:

    FAILED | NULL

according to the contract.

### 4.4 Zero preservation

Input:

    failed = 0

Verify that zero is retained when zero represents a valid measurement.

### 4.5 Category mapping

Verify every source column maps to the correct category.

Test similar names such as:

    failed_amount
    failure_amount

to ensure mappings are explicit rather than inferred.

### 4.6 Duplicate output key

Create source data that can produce two rows for:

    customer + date + category

Verify the target uniqueness test detects the violation.

### 4.7 Source-column lineage

Verify every output row records the expected source column when lineage is enabled.

### 4.8 Data-type preservation

Verify numeric columns remain numeric.

Do not silently convert monetary values into strings.

### 4.9 Monthly mapping

Test:

    jan_amount → January
    feb_amount → February

and verify year/month semantics.

### 4.10 Unexpected source column

Add a new wide column.

Verify the configured policy:

    reject
    ignore with alert
    or include through governed metadata

### 4.11 Idempotence

Run the same input twice.

Expected:

    same output rows
    same categories
    same values

### 4.12 Round-trip test

If a pivot followed by unpivot is intended to preserve information, compare the normalized representations.

Do not expect row-level reconstruction if the pivot used lossy aggregation.

### 4.13 Reconciliation

For additive measures:

    SUM(wide.success_amount)

should equal:

    SUM(long.amount WHERE status = 'SUCCESS')

subject to NULL and filtering policy.

### 4.14 Tenant isolation

Verify tenant/account identifiers remain attached to every generated row.

### 4.15 Output-size test

Use a large source and verify the expansion factor stays within expected capacity.

## 5. Observability

Unpivoting can create large row expansions and hidden semantic loss.

### Core metrics

| Metric | Meaning |
|---|---|
| `unpivot_input_rows` | Source rows |
| `unpivot_output_rows` | Long-form rows |
| `unpivot_expansion_factor` | Output rows / input rows |
| `unpivot_selected_columns` | Number of unpivoted columns |
| `unpivot_null_rows` | Generated rows with NULL values |
| `unpivot_filtered_rows` | Rows removed by policy |
| `unpivot_unmapped_columns` | Source columns without mapping |
| `unpivot_duplicate_keys` | Duplicate target-grain rows |
| `unpivot_unexpected_categories` | Categories outside contract |
| `unpivot_rule_version` | Mapping/semantic version |

### Expansion monitoring

Track:

    output_rows / input_rows

Unexpected increases may indicate:

- New columns included accidentally.
- NULL filtering changed.
- Dynamic metadata drift.
- Duplicate source rows.

### Mapping monitoring

Track unmapped and inactive columns.

A source column disappearing from the output can otherwise go unnoticed.

### Category monitoring

Track category distribution after unpivoting.

Unexpected category proportions can reveal incorrect column mapping.

### NULL monitoring

Track generated NULL rows separately from filtered rows.

This shows whether source sparsity is becoming operationally significant.

### Grain monitoring

Verify uniqueness at the target grain:

    entity + category + period

or whatever the model defines.

### Reconciliation monitoring

Compare additive source totals with long-form totals by category.

## 6. Intentional Failure

### Failure 1 — Drop NULL rows without defining semantics

Filter `value IS NOT NULL`.

Expected symptom:

- Unknown or not-applicable states disappear.

Recovery:

Restore the documented NULL policy.

### Failure 2 — Drop zero values

Filter `value <> 0`.

Expected symptom:

- Valid zero measurements disappear.

Recovery:

Remove the filter unless zero is explicitly non-material.

### Failure 3 — Map a column to the wrong category

Swap SUCCESS and FAILED mappings.

Expected symptom:

- Category-level reports become incorrect while totals may still reconcile.

Recovery:

Correct the mapping and add category-specific fixtures.

### Failure 4 — Omit a source column

Leave one metric out of the mapping.

Expected symptom:

- Source total no longer reconciles with the long model.

Recovery:

Validate mapping completeness.

### Failure 5 — Allow duplicate target keys

Generate two rows for the same target identity.

Expected symptom:

- Downstream aggregation can double count.

Recovery:

Enforce target uniqueness and fix source/mapping grain.

### Failure 6 — Mix incompatible data types

Unpivot amounts and timestamps into one value column.

Expected symptom:

- Type coercion or unusable values.

Recovery:

Separate measures or use an explicitly typed attribute model.

### Failure 7 — Dynamic schema drift

Allow every new source column to become a category automatically.

Expected symptom:

- Unexpected output categories and uncontrolled data growth.

Recovery:

Introduce governed metadata and schema validation.

### Failure 8 — Wrong date interpretation

Map monthly columns to incorrect year or period.

Expected symptom:

- Time-series metrics shift into the wrong period.

Recovery:

Use explicit period metadata and boundary tests.

### Failure 9 — Lose tenant scope

Drop tenant ID during expansion.

Expected symptom:

- Records become ambiguous or collide across tenants.

Recovery:

Restore tenant/account identity to every output row.

## 7. Recovery

### Recovery sequence

1. Preserve the original wide snapshot.
2. Identify the affected source columns.
3. Verify the mapping contract.
4. Verify target grain.
5. Compare input and output counts.
6. Check NULL and zero policies.
7. Reconcile measures by category.
8. Correct mapping or filtering.
9. Rebuild the affected long-form partition.
10. Validate uniqueness.
11. Validate downstream totals.
12. Replay downstream models idempotently.

### Recovering from missing output categories

1. Compare source column inventory with mapping metadata.
2. Find unmapped columns.
3. Determine whether each is required.
4. Correct the mapping.
5. Reprocess affected batches.

### Recovering from excessive row growth

1. Compare expansion factor with expected column count.
2. Inspect newly included columns.
3. Check NULL filtering changes.
4. Check duplicate source rows.
5. Recalculate capacity.

### Recovering from wrong category mapping

1. Identify affected source column.
2. Determine affected output period.
3. Rebuild only affected partitions.
4. Reconcile category-level totals.
5. Correct regression tests.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL supports `CROSS JOIN LATERAL`, `VALUES`, `UNION ALL`, constraints, and metadata-driven SQL patterns useful for production unpivoting.

### 2. dbt

dbt is useful for maintaining explicit wide-to-long mappings, testing target grain, documenting categories, and validating source-to-target reconciliation.

### 3. DuckDB

DuckDB is useful for local experimentation with wide CSV/Parquet datasets and for validating expansion behavior before production execution.

## 9. Production Runbook

### Before deployment

- [ ] Define source grain.
- [ ] Define target grain.
- [ ] Define selected source columns.
- [ ] Define category mapping.
- [ ] Define NULL semantics.
- [ ] Define zero semantics.
- [ ] Define data types.
- [ ] Define schema evolution policy.
- [ ] Define lineage requirements.
- [ ] Estimate expansion factor.
- [ ] Define reconciliation checks.

### During execution

- [ ] Record input rows.
- [ ] Record output rows.
- [ ] Record expansion factor.
- [ ] Record selected columns.
- [ ] Record unmapped columns.
- [ ] Record NULL rows.
- [ ] Record filtered rows.
- [ ] Record duplicate target keys.
- [ ] Record unexpected categories.
- [ ] Record rule version.

### If output row count is too high

1. Check expansion factor.
2. Check selected columns.
3. Check NULL filtering.
4. Check source duplicates.
5. Check metadata changes.

### If output row count is too low

1. Check omitted columns.
2. Check NULL filtering.
3. Check source completeness.
4. Check mapping configuration.

### If downstream totals do not reconcile

1. Compare by category.
2. Compare NULL handling.
3. Compare zero handling.
4. Check data-type conversion.
5. Check missing source columns.

## 10. Common Mistakes

### Mistake 1 — Ignoring target grain

Unpivoting intentionally multiplies rows; the new grain must be explicit.

### Mistake 2 — Dropping NULL without policy

NULL can carry important information.

### Mistake 3 — Dropping zero values

Zero can be a valid measurement.

### Mistake 4 — Using `UNION` instead of `UNION ALL`

Duplicate elimination can remove legitimate generated rows.

### Mistake 5 — Guessing category mappings

Source column names do not always equal business semantics.

### Mistake 6 — Mixing incompatible measures

Keep typed measures separate.

### Mistake 7 — Ignoring output expansion

Wide-to-long transformations can multiply data volume dramatically.

### Mistake 8 — Allowing uncontrolled dynamic columns

Schema evolution still requires governance.

### Mistake 9 — Losing source lineage

Legacy transformations become difficult to debug without source-column traceability.

### Mistake 10 — Assuming round-trip reversibility

Aggregation and filtering can make the transformation lossy.

### Mistake 11 — Losing tenant identity

Every generated row must retain the required business scope.

### Mistake 12 — No reconciliation

Source totals should be explainable in the long representation.

## 11. Definition of Done

The unpivot transformation is complete when you can:

- [ ] Define source and target grain.
- [ ] Explain wide versus long representation.
- [ ] Identify the hidden dimension encoded by columns.
- [ ] Build an explicit category mapping.
- [ ] Use `CROSS JOIN LATERAL` and `VALUES` where appropriate.
- [ ] Use `UNION ALL` safely.
- [ ] Use Pandas `melt` correctly.
- [ ] Preserve or intentionally filter NULL values.
- [ ] Preserve meaningful zero values.
- [ ] Handle multiple measures.
- [ ] Preserve compatible data types.
- [ ] Estimate output expansion.
- [ ] Detect unmapped columns.
- [ ] Detect duplicate target keys.
- [ ] Preserve lineage where required.
- [ ] Handle schema evolution intentionally.
- [ ] Reconcile source and long-form totals.
- [ ] Test round-trip behavior where applicable.
- [ ] Intentionally break mapping and filtering.
- [ ] Recover from missing, duplicated, or misclassified output.
- [ ] Explain how PostgreSQL, dbt, and DuckDB support production unpivoting.

## 12. What You Learned

Unpivoting turns column-encoded dimensions into explicit rows.

The production workflow is:

    DEFINE SOURCE GRAIN
         ↓
    DEFINE TARGET GRAIN
         ↓
    IDENTIFY ENCODED DIMENSION
         ↓
    DEFINE COLUMN → CATEGORY MAPPING
         ↓
    DEFINE NULL / ZERO SEMANTICS
         ↓
    UNPIVOT
         ↓
    VALIDATE EXPANSION
         ↓
    VALIDATE TARGET GRAIN
         ↓
    RECONCILE SOURCE TOTALS
         ↓
    PUBLISH LONG MODEL

> **A production unpivot is correct only when the encoded dimension, target grain, mapping, NULL semantics, data types, expansion factor, and reconciliation behavior are explicit.**

### Next recipe

**T33 — Required-Field Validation**