# T27 — Ranking

> **Goal:** Learn how to rank records correctly within a population, choose deterministic top-N rows, handle ties and NULLs, and distinguish business ranking from arbitrary row numbering.

## 1. Problem Recognition

Ranking problems appear whenever a pipeline must order records relative to other records.

Typical production cases include:

- Top customers by revenue.
- Highest-value transactions per customer.
- Best-performing products per category.
- Latest record per business key.
- Top N events per tenant.
- Rank candidates within a partition.
- Identify first, second, and third occurrences.
- Detect percentile or relative position.
- Select one deterministic representative from tied records.

Ranking sounds simple, but production correctness depends on:

- Partition definition.
- Ranking metric.
- Sort direction.
- Tie policy.
- NULL policy.
- Deterministic tie-breakers.
- Whether ties should consume positions.
- Whether the requirement is top-N rows or top-N ranks.

### Recognition questions

Before implementing ranking, ask:

1. What population is being ranked?
2. Is ranking global or partitioned?
3. What metric defines rank?
4. Is higher or lower better?
5. How should ties behave?
6. Should tied records all be retained?
7. If only one row is required, what deterministic tie-breaker is used?
8. How are NULL metric values treated?
9. Does top-N mean N rows or N distinct ranks?
10. Can late-arriving data change the ranking?

## 2. Concept and Reasoning

### 2.1 Ranking is relative ordering

A rank is meaningful only relative to a defined population and ordering rule.

Example:

    customer_id | revenue
    ------------+--------
    C1          | 900
    C2          | 700
    C3          | 400

Ranking by revenue descending gives:

    C1 → 1
    C2 → 2
    C3 → 3

Change the population and the ranks can change.

Therefore a production ranking should document its population boundary.

### 2.2 Global versus partitioned ranking

Global ranking:

    RANK() OVER (ORDER BY revenue DESC)

All rows compete in one population.

Partitioned ranking:

    RANK() OVER (
        PARTITION BY country
        ORDER BY revenue DESC
    )

Each country has an independent ranking.

Confusing these two can produce completely different business results.

### 2.3 ROW_NUMBER

`ROW_NUMBER()` assigns a unique sequence position.

Example:

    ROW_NUMBER() OVER (
        PARTITION BY customer_id
        ORDER BY amount DESC, transaction_id
    )

Even tied amounts receive different row numbers when the tie-breaker is deterministic.

Use `ROW_NUMBER()` when the requirement is:

    select exactly one ordered row

or:

    select exactly N rows

provided the ordering is deterministic.

### 2.4 RANK

`RANK()` gives equal ordering values the same rank.

Example values:

    100
    100
     50

Ranks:

    1
    1
    3

The gap exists because two rows occupy rank 1.

Use `RANK()` when tied records should share the same position and subsequent positions should reflect the number of preceding rows.

### 2.5 DENSE_RANK

`DENSE_RANK()` also gives ties the same rank, but does not create gaps.

Example:

    values: 100, 100, 50
    ranks:  1,   1,   2

Use it when rank numbers represent distinct metric levels rather than row positions.

### 2.6 Ranking functions are not interchangeable

| Function | Ties share rank? | Gaps after ties? | Unique row position? |
|---|---:|---:|---:|
| `ROW_NUMBER` | No | No | Yes |
| `RANK` | Yes | Yes | No |
| `DENSE_RANK` | Yes | No | No |

Choosing the wrong function changes the business meaning.

### 2.7 Ranking versus sorting

Sorting answers:

    In what order should rows be displayed?

Ranking answers:

    What position does this row occupy relative to the ranking population?

A query can be sorted without calculating rank.

Ranking should be introduced only when the position itself is needed.

### 2.8 Deterministic ranking

Suppose two transactions both have amount 500.

Using:

    ROW_NUMBER() OVER (
        ORDER BY amount DESC
    )

does not define which transaction receives row number 1.

Add a stable tie-breaker:

    ORDER BY amount DESC, transaction_id ASC

The tie-breaker must be deterministic and appropriate to the business contract.

### 2.9 Business tie-breakers

Possible tie-breakers include:

- Event ID.
- Transaction ID.
- Created timestamp.
- Sequence number.
- Source-system offset.

Do not use arbitrary physical row order.

Do not use a random function unless nondeterminism is explicitly required.

### 2.10 Ranking and NULL values

NULL is not automatically equivalent to zero.

If ranking by a metric that can be NULL, define the policy.

Example:

    ORDER BY score DESC NULLS LAST

or:

    ORDER BY score ASC NULLS FIRST

The correct choice depends on business semantics.

### 2.11 Top-N rows

To select the top three rows globally:

    WITH ranked AS (
        SELECT
            product_id,
            revenue,
            ROW_NUMBER() OVER (
                ORDER BY revenue DESC, product_id
            ) AS rn
        FROM products
    )
    SELECT *
    FROM ranked
    WHERE rn <= 3;

This returns exactly three rows when at least three valid rows exist.

### 2.12 Top-N ranks

If the requirement is:

    include everyone tied within the top three ranks

use `RANK()` or `DENSE_RANK()` according to the business definition.

Example:

    WITH ranked AS (
        SELECT
            product_id,
            revenue,
            RANK() OVER (
                ORDER BY revenue DESC
            ) AS rnk
        FROM products
    )
    SELECT *
    FROM ranked
    WHERE rnk <= 3;

This can return more than three rows.

### 2.13 Top-N rows versus top-N distinct values

These requirements are different:

    top 3 rows

versus:

    top 3 revenue levels

`ROW_NUMBER`, `RANK`, and `DENSE_RANK` answer different versions of these questions.

Define the requirement before selecting the function.

### 2.14 Partitioned top-N

Example:

    WITH ranked AS (
        SELECT
            product_id,
            category_id,
            revenue,
            ROW_NUMBER() OVER (
                PARTITION BY category_id
                ORDER BY revenue DESC, product_id
            ) AS rn
        FROM products
    )
    SELECT *
    FROM ranked
    WHERE rn <= 5;

This returns up to five products per category.

### 2.15 Latest record is a ranking problem

Latest-row selection is often implemented using:

    ROW_NUMBER() OVER (
        PARTITION BY business_key
        ORDER BY effective_at DESC, record_id DESC
    )

Then:

    WHERE rn = 1

The ranking contract must define what 'latest' means.

### 2.16 Ranking by derived metrics

Ranking can use a calculated expression:

    RANK() OVER (
        PARTITION BY customer_id
        ORDER BY amount * quantity DESC
    )

For maintainability, calculate complex metrics in an earlier stage when useful, then rank the resulting column.

### 2.17 Ranking after aggregation

Often the business entity must first be aggregated.

Example:

    WITH customer_totals AS (
        SELECT
            customer_id,
            SUM(amount) AS total_revenue
        FROM orders
        GROUP BY customer_id
    )
    SELECT
        customer_id,
        total_revenue,
        RANK() OVER (
            ORDER BY total_revenue DESC
        ) AS revenue_rank
    FROM customer_totals;

Do not rank raw transactions when the business requirement is ranking customers by total revenue.

### 2.18 Ranking and filtering order

Window functions are calculated at a defined stage of query processing.

If you filter source rows before ranking, you change the ranking population.

Example:

    WHERE country = 'DE'

before the window means the ranking is within German rows.

If the requirement is global ranking followed by selecting German records, the query must rank first and filter later.

This is a critical semantic distinction.

### 2.19 Ranking and time windows

Ranking may be scoped to a time period:

    rank customers by revenue during the current month

First define the period, then aggregate/rank within that population.

Do not accidentally rank across all historical data.

### 2.20 Ranking and late-arriving data

A new record can change many existing ranks.

Example:

    Customer A → rank 3

After a late high-value record arrives:

    Customer A → rank 4

Ranking is therefore sensitive to population changes.

Incremental pipelines need a clear recomputation strategy.

### 2.21 Ranking and snapshot semantics

Production rankings should identify the snapshot or reporting period they represent.

Record:

    ranking_period
    source_watermark
    rule_version
    generated_at

Without snapshot context, historical rankings become difficult to reproduce.

### 2.22 Percentiles and relative ranking

Some ranking problems require relative position rather than integer rank.

Useful functions include:

    PERCENT_RANK()
    CUME_DIST()
    NTILE()

These answer different questions and should not be substituted casually.

### 2.23 PERCENT_RANK

`PERCENT_RANK()` represents relative rank between 0 and 1.

Conceptually:

    (rank - 1) / (rows - 1)

Single-row partitions require special attention because there is no meaningful denominator for ordinary relative position.

### 2.24 CUME_DIST

`CUME_DIST()` measures the proportion of rows at or below the current ordering position according to the function's ordering semantics.

It can be useful for percentile-style thresholds.

Validate the exact boundary behavior against the database engine and business requirement.

### 2.25 NTILE

`NTILE(n)` distributes rows into approximately equal buckets.

Example:

    NTILE(4) OVER (
        ORDER BY revenue DESC, customer_id
    )

can create four groups.

These are buckets, not percentile ranks in every business sense.

## 3. Implementation

### 3.1 Define the ranking contract

Example:

    Population:       active customers
    Metric:            monthly revenue
    Direction:         DESC
    Tie policy:        shared rank
    Tie-breaker:       customer_id for deterministic display only
    Scope:             country
    Period:            calendar month
    NULL revenue:      zero after explicit aggregation
    Snapshot:          month-end

### 3.2 PostgreSQL example data

    CREATE TABLE customer_monthly_revenue (
        customer_id BIGINT NOT NULL,
        country_code TEXT NOT NULL,
        revenue_month DATE NOT NULL,
        revenue NUMERIC(18,2),
        PRIMARY KEY (customer_id, revenue_month)
    );

### 3.3 Global RANK

    SELECT
        customer_id,
        revenue,
        RANK() OVER (
            ORDER BY revenue DESC NULLS LAST
        ) AS revenue_rank
    FROM customer_monthly_revenue;

### 3.4 Partitioned RANK

    SELECT
        customer_id,
        country_code,
        revenue,
        RANK() OVER (
            PARTITION BY country_code
            ORDER BY revenue DESC NULLS LAST
        ) AS country_rank
    FROM customer_monthly_revenue;

### 3.5 Deterministic top-N rows

    WITH ranked AS (
        SELECT
            customer_id,
            revenue,
            ROW_NUMBER() OVER (
                ORDER BY revenue DESC NULLS LAST,
                         customer_id ASC
            ) AS rn
        FROM customer_monthly_revenue
    )
    SELECT
        customer_id,
        revenue
    FROM ranked
    WHERE rn <= 10;

This returns at most ten rows.

### 3.6 Top-N with ties

    WITH ranked AS (
        SELECT
            customer_id,
            revenue,
            RANK() OVER (
                ORDER BY revenue DESC NULLS LAST
            ) AS rnk
        FROM customer_monthly_revenue
    )
    SELECT
        customer_id,
        revenue
    FROM ranked
    WHERE rnk <= 10;

This may return more than ten rows.

### 3.7 Top-N per country

    WITH ranked AS (
        SELECT
            customer_id,
            country_code,
            revenue,
            ROW_NUMBER() OVER (
                PARTITION BY country_code
                ORDER BY revenue DESC NULLS LAST,
                         customer_id
            ) AS rn
        FROM customer_monthly_revenue
    )
    SELECT *
    FROM ranked
    WHERE rn <= 5;

### 3.8 Rank after aggregation

    WITH monthly AS (
        SELECT
            customer_id,
            country_code,
            SUM(amount) AS revenue
        FROM orders
        WHERE order_date >= DATE '2026-09-01'
          AND order_date <  DATE '2026-10-01'
        GROUP BY customer_id, country_code
    ),
    ranked AS (
        SELECT
            monthly.*,
            RANK() OVER (
                PARTITION BY country_code
                ORDER BY revenue DESC, customer_id
            ) AS rnk
        FROM monthly
    )
    SELECT *
    FROM ranked
    WHERE rnk <= 10;

Be careful: adding `customer_id` to the ORDER BY of `RANK()` changes tie semantics because the full ordering becomes unique.

For shared ranks based only on revenue, rank by revenue only and use the ID only in a later display sort.

### 3.9 Separate ranking from display order

Correct shared-rank design:

    WITH ranked AS (
        SELECT
            customer_id,
            revenue,
            RANK() OVER (
                ORDER BY revenue DESC
            ) AS rnk
        FROM customer_monthly_revenue
    )
    SELECT *
    FROM ranked
    WHERE rnk <= 10
    ORDER BY rnk, customer_id;

The rank is based on revenue.

The final `ORDER BY` uses customer ID only to make output display deterministic.

### 3.10 Latest record per entity

    WITH ranked AS (
        SELECT
            customer_id,
            status,
            occurred_at,
            event_id,
            ROW_NUMBER() OVER (
                PARTITION BY customer_id
                ORDER BY occurred_at DESC NULLS LAST,
                         event_id DESC
            ) AS rn
        FROM customer_events
    )
    SELECT *
    FROM ranked
    WHERE rn = 1;

### 3.11 PERCENT_RANK

    SELECT
        customer_id,
        revenue,
        PERCENT_RANK() OVER (
            ORDER BY revenue
        ) AS revenue_percent_rank
    FROM customer_monthly_revenue;

### 3.12 CUME_DIST

    SELECT
        customer_id,
        revenue,
        CUME_DIST() OVER (
            ORDER BY revenue
        ) AS revenue_cume_dist
    FROM customer_monthly_revenue;

### 3.13 NTILE

    SELECT
        customer_id,
        revenue,
        NTILE(4) OVER (
            ORDER BY revenue DESC, customer_id
        ) AS revenue_quartile
    FROM customer_monthly_revenue;

The unique customer ID makes bucket assignment deterministic.

### 3.14 Python implementation

    from collections import defaultdict

    def top_n_per_country(rows, n):
        partitions = defaultdict(list)

        for row in rows:
            partitions[row["country_code"]].append(row)

        result = []

        for country, country_rows in partitions.items():
            country_rows.sort(
                key=lambda row: (-row["revenue"], row["customer_id"])
            )

            for position, row in enumerate(country_rows, start=1):
                if position > n:
                    break
                output = dict(row)
                output["row_number"] = position
                result.append(output)

        return result

Python implementations must define the same partition and ordering contract as the SQL implementation.

### 3.15 Ranking after an aggregate stage in Python

Use explicit stages:

    raw transactions
         ↓
    aggregate by business entity
         ↓
    partition by ranking scope
         ↓
    deterministic sort
         ↓
    assign rank
         ↓
    apply top-N policy

Do not rank raw rows when the business metric is an entity-level aggregate.

## 4. Testing

Ranking tests must verify both numeric positions and population semantics.

### 4.1 Basic ranking

Given:

    900
    700
    400

Expected RANK:

    1
    2
    3

### 4.2 Tie behavior

Given:

    900
    900
    400

Expected:

    ROW_NUMBER → 1, 2, 3
    RANK       → 1, 1, 3
    DENSE_RANK → 1, 1, 2

### 4.3 Deterministic ROW_NUMBER

Create equal metrics with unique IDs.

Run the query multiple times.

Expected:

    identical row-number assignments

### 4.4 Partition isolation

Create identical revenue values in two countries.

Verify each country receives an independent rank sequence.

### 4.5 NULL metric

Create NULL revenue values.

Verify the documented NULL ordering.

### 4.6 Top-N rows

Create ten records and request top three rows.

Expected:

    exactly three rows

assuming all records are eligible.

### 4.7 Top-N ties

Create more than N rows sharing the boundary rank.

Verify whether the business rule expects:

    exactly N rows

or:

    everyone tied through rank N

### 4.8 Ranking after aggregation

Create multiple transactions per customer.

Verify that ranking uses customer totals rather than individual transactions.

### 4.9 Filter-order test

Compare:

    rank globally, then filter country

against:

    filter country, then rank

Verify that the implementation matches the business definition.

### 4.10 Late-data test

Run a monthly ranking.

Insert a late high-value record.

Verify that affected ranks change according to the documented replay policy.

### 4.11 Snapshot reproducibility

Run the same ranking against the same source snapshot and rule version.

Expected:

    identical ranks
    identical top-N population

### 4.12 Population accounting

For a top-N row selection:

    selected_rows <= eligible_rows

For a shared-rank selection:

    selected_rows may exceed N

Do not use an exact-N assertion for a tied-rank requirement.

## 5. Observability

Ranking pipelines need visibility into both result distribution and ranking stability.

### Core metrics

| Metric | Meaning |
|---|---|
| `ranking_input_rows` | Eligible ranking population |
| `ranking_output_rows` | Selected ranking population |
| `ranking_partition_count` | Number of ranking partitions |
| `ranking_max_partition_rows` | Largest partition |
| `ranking_tie_rows` | Rows involved in metric ties |
| `ranking_null_metric_rows` | Rows with NULL ranking metric |
| `ranking_boundary_tie_rows` | Rows tied at the top-N boundary |
| `ranking_rule_version` | Version of ranking semantics |
| `ranking_snapshot` | Source/reporting snapshot |

### Top-N boundary monitoring

The boundary is operationally important.

If top 10 is the requirement and many records share rank 10, the output can expand significantly under a tied-rank policy.

Track:

    boundary_rank
    boundary_metric
    boundary_tie_count

### Ranking stability

Compare the current ranking with the previous snapshot when business monitoring requires it.

Useful measurements include:

- Rank changes.
- Entries entering top-N.
- Entries leaving top-N.
- Large rank movements.
- New entities appearing in the ranking.

These are monitoring signals, not automatic proof of an error.

### Partition monitoring

Large partitions can dominate sort cost.

Track partition size distribution and identify unusually large partitions.

### Data-quality monitoring

Monitor:

- NULL metrics.
- Duplicate business keys.
- Duplicate ordering keys.
- Unexpected negative or impossible metric values.

Ranking should not be the first place where fundamental source-quality problems are discovered.

## 6. Intentional Failure

### Failure 1 — Add a tie-breaker inside RANK

Change:

    RANK() OVER (ORDER BY revenue DESC)

to:

    RANK() OVER (ORDER BY revenue DESC, customer_id)

Expected symptom:

- Previously tied customers receive different ranks.

Recovery:

Keep the business ranking expression separate from display-level tie-breaking.

### Failure 2 — Use ROW_NUMBER for shared top-N

Create ties at the boundary.

Expected symptom:

- Some equally ranked entities are excluded.

Recovery:

Use `RANK` or `DENSE_RANK` according to the business definition.

### Failure 3 — Use RANK for exact top-N rows

Create a large tie at rank 1.

Expected symptom:

- Output contains far more than N rows.

Recovery:

Use deterministic `ROW_NUMBER` when the contract requires exactly N rows.

### Failure 4 — Filter before global ranking

Filter to one country before applying the ranking.

Expected symptom:

- Country-local ranks replace global ranks.

Recovery:

Move the filter to the correct query stage.

### Failure 5 — Remove partitioning

Rank all countries together.

Expected symptom:

- Country-local ranking disappears.

Recovery:

Restore the partition key.

### Failure 6 — Remove deterministic tie-breaking from ROW_NUMBER

Create tied metrics.

Expected symptom:

- Selected top-N records can change between executions or plans.

Recovery:

Add a stable unique tie-breaker.

### Failure 7 — Ignore NULL ranking values

Introduce NULL metrics.

Expected symptom:

- NULL rows appear in unexpected ranking positions.

Recovery:

Define explicit NULL ordering.

### Failure 8 — Rank raw transactions instead of aggregated entities

Create multiple transactions per customer.

Expected symptom:

- Customers with more transaction rows dominate the ranking incorrectly.

Recovery:

Aggregate to the business ranking grain before ranking.

### Failure 9 — Ignore late data

Insert a late record that should rank highly.

Expected symptom:

- Historical ranking becomes stale.

Recovery:

Recompute the affected snapshot or partition according to the replay contract.

## 7. Recovery

Ranking incidents require separating population, metric, ordering, and tie-policy failures.

### Recovery sequence

1. Identify the affected ranking snapshot.
2. Capture source watermark.
3. Verify eligible population.
4. Verify aggregation grain.
5. Verify ranking metric.
6. Verify sort direction.
7. Verify NULL policy.
8. Verify tie semantics.
9. Verify deterministic tie-breakers.
10. Check late-arriving records.
11. Recompute affected partitions.
12. Compare top-N population before and after correction.
13. Validate ranking invariants.
14. Replay downstream consumers if required.
15. Record the root cause.

### Recovering an incorrect top-N

1. Identify the boundary rank.
2. Compare metric values around the boundary.
3. Check whether ties were handled correctly.
4. Check whether the ranking population was filtered incorrectly.
5. Check whether aggregation occurred at the correct grain.
6. Re-run the ranking.

### Recovering unstable latest-row selection

If different records are repeatedly selected as latest:

1. Inspect timestamp ties.
2. Inspect the tie-breaker.
3. Verify NULL timestamp handling.
4. Add a deterministic unique ordering key.
5. Recompute affected business keys.

### Recovering after late data

1. Identify affected ranking period.
2. Identify affected partitions.
3. Rebuild the ranking from the required historical boundary.
4. Replace the affected snapshot atomically.
5. Validate rank changes.
6. Replay downstream outputs.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL provides `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `PERCENT_RANK`, `CUME_DIST`, and `NTILE`, along with query planning tools.

Learn to inspect sorting, partitioning, and memory behavior for large ranking workloads.

### 2. dbt

dbt models frequently use ranking for latest-record selection, deduplication, top-N analysis, and entity-level prioritization.

Use dbt tests to enforce expected grain and uniqueness after ranking filters.

### 3. DuckDB

DuckDB is useful for validating ranking semantics against analytical datasets and Parquet files.

It is particularly useful for testing ties, partitions, and top-N behavior locally.

## 9. Production Runbook

### Before deployment

- [ ] Define the ranking population.
- [ ] Define ranking grain.
- [ ] Define metric.
- [ ] Define sort direction.
- [ ] Define tie behavior.
- [ ] Define NULL behavior.
- [ ] Define whether top-N means rows or ranks.
- [ ] Define deterministic tie-breakers.
- [ ] Define snapshot period.
- [ ] Define late-data replay.
- [ ] Test partition isolation.
- [ ] Test boundary ties.

### During execution

- [ ] Record eligible population.
- [ ] Record selected population.
- [ ] Record partition count.
- [ ] Record boundary metric.
- [ ] Record boundary tie count.
- [ ] Record NULL metric count.
- [ ] Record rule version.
- [ ] Record source snapshot/watermark.

### If top-N membership changes unexpectedly

1. Check source changes.
2. Check aggregation changes.
3. Check ranking population.
4. Check metric values.
5. Check ties.
6. Check NULL handling.
7. Check rule version.

### If output has too many rows

1. Determine whether shared-rank semantics are intended.
2. Inspect the boundary tie.
3. If exact N rows are required, use deterministic `ROW_NUMBER`.
4. If ties must be preserved, document variable output size.

### If output has too few rows

1. Check eligibility filters.
2. Check NULL policy.
3. Check aggregation grain.
4. Check whether a tied-rank policy was accidentally replaced with row numbering.

### If ranking query is slow

1. Inspect the execution plan.
2. Measure partition sizes.
3. Reduce input columns before sorting.
4. Restrict the ranking population early when semantically valid.
5. Avoid unnecessary repeated ranking expressions.

## 10. Common Mistakes

### Mistake 1 — Treating RANK and ROW_NUMBER as equivalent

They encode different tie semantics.

### Mistake 2 — Adding a tie-breaker to RANK

This can destroy intended shared ranks.

### Mistake 3 — Using RANK for exact top-N rows

Ties can make the result larger than N.

### Mistake 4 — Using ROW_NUMBER when ties must be preserved

Some tied entities will be excluded.

### Mistake 5 — Forgetting PARTITION BY

Independent ranking populations become one population.

### Mistake 6 — Filtering at the wrong stage

Filtering before ranking changes the ranking population.

### Mistake 7 — Ignoring NULL ordering

Missing metrics can land in unexpected positions.

### Mistake 8 — Ranking the wrong grain

Raw event rows should not be ranked when the business entity is an aggregate.

### Mistake 9 — Using arbitrary physical order

Database storage order is not a business tie-breaker.

### Mistake 10 — Ignoring snapshot semantics

Historical rankings become difficult to reproduce.

### Mistake 11 — Ignoring late-arriving data

Ranking is population-sensitive.

### Mistake 12 — Recomputing everything unnecessarily

Use bounded recomputation when the affected ranking scope is known.

## 11. Definition of Done

The ranking transformation is complete when you can:

- [ ] Define the ranking population and grain.
- [ ] Distinguish global and partitioned ranking.
- [ ] Explain ROW_NUMBER, RANK, and DENSE_RANK.
- [ ] Define tie behavior.
- [ ] Define deterministic tie-breakers.
- [ ] Handle NULL ranking values explicitly.
- [ ] Implement exact top-N rows.
- [ ] Implement top-N with ties.
- [ ] Implement partitioned top-N.
- [ ] Rank after aggregation.
- [ ] Explain filter-order effects.
- [ ] Use PERCENT_RANK, CUME_DIST, and NTILE appropriately.
- [ ] Handle late-arriving ranking data.
- [ ] Define ranking snapshot semantics.
- [ ] Test tie, NULL, partition, and boundary behavior.
- [ ] Monitor partition size and boundary ties.
- [ ] Intentionally break ranking semantics.
- [ ] Recover incorrect or unstable rankings.
- [ ] Reproduce rankings from a defined snapshot.
- [ ] Explain how PostgreSQL, dbt, and DuckDB support production ranking workloads.

## 12. What You Learned

Ranking is not simply sorting rows.

A production ranking is a contract:

    DEFINE POPULATION
         ↓
    DEFINE BUSINESS GRAIN
         ↓
    DEFINE METRIC
         ↓
    DEFINE PARTITION
         ↓
    DEFINE SORT DIRECTION
         ↓
    DEFINE TIE POLICY
         ↓
    DEFINE NULL POLICY
         ↓
    DEFINE DETERMINISTIC TIE-BREAKER
         ↓
    APPLY RANKING
         ↓
    APPLY TOP-N POLICY
         ↓
    VALIDATE BOUNDARY
         ↓
    SNAPSHOT AND MONITOR

> **The ranking function is only the middle of the design. Correct ranking requires a precisely defined population, metric, partition, tie policy, and reproducible ordering.**

### Next recipe

**T28 — Running Totals**