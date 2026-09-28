# T26 — Window Functions

> **Goal:** Learn how to calculate row-aware analytics while preserving the original row grain, including partitioned calculations, ranking, running totals, frames, peer groups, and production-safe ordering.

## 1. Problem Recognition

Window functions solve problems where a row needs information about related rows without collapsing the result set.

Typical production cases include:

- Ranking transactions within each customer.
- Selecting the latest record per business key.
- Calculating running balances.
- Calculating cumulative revenue.
- Comparing a row with the previous or next row.
- Calculating percentages of a partition total.
- Detecting changes between consecutive events.
- Calculating rolling metrics.
- Measuring time between events.
- Identifying first and last records within a group.

The defining property is:

    aggregate or compare across related rows
    while keeping one output row per input row

An ordinary `GROUP BY` reduces multiple rows to one row per group.

A window function annotates rows without collapsing them.

### Recognition questions

Before writing a window expression, ask:

1. What is the input grain?
2. What defines a partition?
3. What defines deterministic row order?
4. What window frame is required?
5. Are ties expected?
6. What should happen when ordering values are NULL?
7. Is the calculation row-based or range/time-based?
8. Does the result need to preserve every input row?
9. Is the window calculation being used for filtering?
10. Can the ordering change between pipeline runs?

## 2. Concept and Reasoning

### 2.1 What is a window?

A window is the set of rows visible to a window function for the current row.

Example:

    customer_id | occurred_at | amount
    ------------+-------------+-------
    C1          | 09:00       | 100
    C1          | 10:00       |  50
    C1          | 11:00       |  25
    C2          | 09:30       | 200

A window partitioned by `customer_id` gives each C1 row access to the C1 rows and each C2 row access to the C2 rows.

### 2.2 The three core clauses

A typical window specification contains:

    OVER (
        PARTITION BY ...
        ORDER BY ...
        frame
    )

`PARTITION BY` defines which rows belong to the same logical group.

`ORDER BY` defines the sequence used by order-sensitive functions.

The frame defines which subset of the partition is visible for the current row.

### 2.3 Window functions preserve row grain

Given ten transaction rows, a window calculation normally returns ten rows.

Example:

    SELECT
        transaction_id,
        customer_id,
        amount,
        SUM(amount) OVER (
            PARTITION BY customer_id
        ) AS customer_total
    FROM transactions;

Every transaction remains present.

### 2.4 GROUP BY versus window functions

With:

    SELECT customer_id, SUM(amount)
    FROM transactions
    GROUP BY customer_id;

the output is one row per customer.

With:

    SELECT
        transaction_id,
        customer_id,
        amount,
        SUM(amount) OVER (PARTITION BY customer_id) AS customer_total
    FROM transactions;

the output remains one row per transaction.

Use `GROUP BY` when you want aggregation to collapse the population.

Use a window when the row-level population must remain available.

### 2.5 PARTITION BY

Example:

    SUM(amount) OVER (PARTITION BY customer_id)

Each customer receives an independent total.

Without `PARTITION BY`:

    SUM(amount) OVER ()

the entire result set is one window.

### 2.6 ORDER BY

Order-sensitive functions require a meaningful sequence.

Example:

    ROW_NUMBER() OVER (
        PARTITION BY customer_id
        ORDER BY occurred_at
    )

assigns a sequence within each customer.

If multiple rows have identical `occurred_at`, the ordering may be nondeterministic unless a tie-breaker is added.

Prefer:

    ORDER BY occurred_at, transaction_id

when `transaction_id` is unique.

### 2.7 Deterministic ordering

Production pipelines should not assume that database row order is stable.

Bad:

    ROW_NUMBER() OVER (
        PARTITION BY customer_id
        ORDER BY occurred_at
    )

when multiple events can have the same timestamp.

Better:

    ROW_NUMBER() OVER (
        PARTITION BY customer_id
        ORDER BY occurred_at, event_id
    )

Deterministic ordering is part of the transformation contract.

### 2.8 Window frames

A window frame controls the rows used for the current calculation.

Common concepts include:

    UNBOUNDED PRECEDING
    CURRENT ROW
    UNBOUNDED FOLLOWING

Example:

    SUM(amount) OVER (
        PARTITION BY customer_id
        ORDER BY occurred_at, transaction_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )

This calculates a running total.

### 2.9 ROWS versus RANGE

`ROWS` describes a physical row-based frame.

`RANGE` describes a value-based frame and can include peer rows with equal ordering values.

These can produce different results when ordering values tie.

For deterministic row-by-row running calculations, explicitly specifying `ROWS` is often clearer.

### 2.10 Peer groups

Rows with equal ordering values can be peers.

Peer behavior matters for functions and frames that operate on ordering values rather than strictly physical row positions.

If the business definition requires one deterministic sequence, include a unique tie-breaker.

### 2.11 Ranking functions

Common ranking functions:

    ROW_NUMBER()
    RANK()
    DENSE_RANK()

`ROW_NUMBER()` assigns a unique sequence position.

`RANK()` gives tied rows the same rank and leaves gaps after ties.

`DENSE_RANK()` gives tied rows the same rank without gaps.

Example:

    RANK() OVER (
        PARTITION BY customer_id
        ORDER BY amount DESC
    )

### 2.12 ROW_NUMBER and tie-breaking

`ROW_NUMBER()` must have a deterministic order if its result controls business logic.

Example use:

    latest row per customer

Use:

    ROW_NUMBER() OVER (
        PARTITION BY customer_id
        ORDER BY occurred_at DESC, event_id DESC
    ) AS rn

Then select `rn = 1` in an outer query.

### 2.13 LAG and LEAD

`LAG()` accesses a previous row.

`LEAD()` accesses a following row.

Example:

    LAG(amount) OVER (
        PARTITION BY customer_id
        ORDER BY occurred_at, transaction_id
    ) AS previous_amount

These are useful for:

- Change detection.
- Event intervals.
- State transitions.
- Previous-value comparisons.

### 2.14 FIRST_VALUE and LAST_VALUE

These functions can retrieve values relative to a window.

Example:

    FIRST_VALUE(amount) OVER (
        PARTITION BY customer_id
        ORDER BY occurred_at, transaction_id
    )

`LAST_VALUE()` requires particular care because the default frame may end at the current row rather than the end of the partition.

If the requirement is the final partition value, define the frame explicitly.

### 2.15 Aggregate windows

Aggregates such as:

    SUM()
    AVG()
    MIN()
    MAX()
    COUNT()

can be used as window functions.

Example:

    AVG(amount) OVER (PARTITION BY customer_id)

calculates the customer's average while preserving transaction rows.

### 2.16 Running totals

Canonical pattern:

    SUM(amount) OVER (
        PARTITION BY account_id
        ORDER BY occurred_at, transaction_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_balance

The ordering defines when each transaction contributes to the cumulative result.

### 2.17 Moving windows

A rolling calculation can use a bounded frame.

Example concept:

    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW

This represents the current row plus the previous six rows.

Important:

    seven rows is not necessarily seven days.

If the business requirement is a seven-day time window, use an appropriate time-based design rather than assuming row counts represent elapsed time.

### 2.18 Percentage of partition total

Example:

    amount / NULLIF(
        SUM(amount) OVER (PARTITION BY customer_id),
        0
    )

This calculates each transaction's share of its customer's total.

### 2.19 Filtering on window results

SQL evaluation order means a window result generally cannot be referenced directly in the same query's `WHERE` clause.

Use a subquery or CTE:

    WITH ranked AS (
        SELECT
            o.*,
            ROW_NUMBER() OVER (
                PARTITION BY customer_id
                ORDER BY occurred_at DESC, order_id DESC
            ) AS rn
        FROM orders AS o
    )
    SELECT *
    FROM ranked
    WHERE rn = 1;

This pattern is fundamental for latest-row selection.

### 2.20 QUALIFY

Some analytical engines support `QUALIFY` for filtering after window evaluation.

Conceptually:

    SELECT
        o.*,
        ROW_NUMBER() OVER (...) AS rn
    FROM orders AS o
    QUALIFY rn = 1;

PostgreSQL does not provide `QUALIFY` as a native clause, so use a subquery or CTE there.

### 2.21 Window functions and NULL ordering

NULL ordering can affect ranking and latest-row logic.

Make the policy explicit when necessary:

    ORDER BY occurred_at DESC NULLS LAST

or:

    ORDER BY occurred_at ASC NULLS FIRST

The correct policy depends on the business meaning of missing timestamps.

### 2.22 Window functions and grain

A window does not automatically change the row grain.

But a later filter such as:

    WHERE rn = 1

does change the output population.

Document both:

    pre-window grain
    post-filter grain

### 2.23 Window functions and business time

An event-time window should use the event timestamp, not ingestion timestamp, when the business definition is based on when the event occurred.

Otherwise late-arriving data can be placed in the wrong analytical sequence.

### 2.24 Window functions and late data

If events arrive late, recomputing a running or ranking window over only the newly arrived rows may be incorrect.

A late event can change:

- Row numbers.
- Previous/next relationships.
- Running totals.
- Rolling metrics.
- First/last records.

Window transformations therefore require an explicit reprocessing boundary when historical data can change.

### 2.25 Window functions are not inherently stateful

A SQL window calculation is evaluated over the rows visible to the query.

It does not automatically maintain persistent state between pipeline runs.

Incremental pipelines must define which historical rows are re-evaluated when new or corrected records can affect existing windows.

### 2.26 Nested window calculations

Window functions generally cannot be directly nested inside another window function at the same query level.

Use stages:

    raw rows
        ↓
    first window
        ↓
    derived relation
        ↓
    second window

This keeps transformation stages explicit.

## 3. Implementation

### 3.1 Define the contract

Example:

    Input grain:        one row per transaction
    Partition key:      customer_id
    Sequence:           occurred_at + transaction_id
    Ranking:            latest transaction
    Running metric:     cumulative amount
    NULL timestamp:     last
    Late events:        recompute affected customer scope

### 3.2 PostgreSQL example data

    CREATE TABLE transactions (
        transaction_id BIGINT PRIMARY KEY,
        customer_id BIGINT NOT NULL,
        occurred_at TIMESTAMPTZ,
        amount NUMERIC(18,2) NOT NULL
    );

### 3.3 Partition total

    SELECT
        transaction_id,
        customer_id,
        amount,
        SUM(amount) OVER (
            PARTITION BY customer_id
        ) AS customer_total
    FROM transactions;

Every transaction remains in the result.

### 3.4 Running total

    SELECT
        transaction_id,
        customer_id,
        occurred_at,
        amount,
        SUM(amount) OVER (
            PARTITION BY customer_id
            ORDER BY occurred_at, transaction_id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS running_total
    FROM transactions;

### 3.5 Previous transaction

    SELECT
        transaction_id,
        customer_id,
        occurred_at,
        amount,
        LAG(amount) OVER (
            PARTITION BY customer_id
            ORDER BY occurred_at, transaction_id
        ) AS previous_amount
    FROM transactions;

### 3.6 Time since previous event

    SELECT
        transaction_id,
        customer_id,
        occurred_at,
        occurred_at - LAG(occurred_at) OVER (
            PARTITION BY customer_id
            ORDER BY occurred_at, transaction_id
        ) AS elapsed_since_previous
    FROM transactions;

### 3.7 Rank transactions

    SELECT
        transaction_id,
        customer_id,
        amount,
        RANK() OVER (
            PARTITION BY customer_id
            ORDER BY amount DESC
        ) AS amount_rank,
        DENSE_RANK() OVER (
            PARTITION BY customer_id
            ORDER BY amount DESC
        ) AS amount_dense_rank
    FROM transactions;

### 3.8 Latest row per business key

    WITH ranked AS (
        SELECT
            t.*,
            ROW_NUMBER() OVER (
                PARTITION BY customer_id
                ORDER BY occurred_at DESC NULLS LAST,
                         transaction_id DESC
            ) AS rn
        FROM transactions AS t
    )
    SELECT *
    FROM ranked
    WHERE rn = 1;

This is a common deduplication and latest-state pattern.

### 3.9 Running total with explicit frame

    SELECT
        account_id,
        transaction_id,
        occurred_at,
        amount,
        SUM(amount) OVER (
            PARTITION BY account_id
            ORDER BY occurred_at, transaction_id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS cumulative_amount
    FROM account_transactions;

Use an explicit frame when the calculation depends on physical row progression.

### 3.10 Moving seven-row calculation

    SELECT
        customer_id,
        occurred_at,
        amount,
        AVG(amount) OVER (
            PARTITION BY customer_id
            ORDER BY occurred_at, transaction_id
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS seven_row_average
    FROM transactions;

This is a seven-row window, not a seven-calendar-day window.

### 3.11 Partition share

    SELECT
        transaction_id,
        customer_id,
        amount,
        amount / NULLIF(
            SUM(amount) OVER (PARTITION BY customer_id),
            0
        ) AS customer_share
    FROM transactions;

### 3.12 Detect value changes

    WITH changes AS (
        SELECT
            customer_id,
            occurred_at,
            status,
            LAG(status) OVER (
                PARTITION BY customer_id
                ORDER BY occurred_at, event_id
            ) AS previous_status
        FROM account_events
    )
    SELECT *
    FROM changes
    WHERE status IS DISTINCT FROM previous_status;

`IS DISTINCT FROM` gives explicit NULL-safe comparison semantics.

### 3.13 Two-stage window calculation

Suppose the requirement is:

    calculate each transaction's running total
    then rank customers by their final running total

Use stages:

    WITH running AS (
        SELECT
            customer_id,
            transaction_id,
            occurred_at,
            amount,
            SUM(amount) OVER (
                PARTITION BY customer_id
                ORDER BY occurred_at, transaction_id
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
            ) AS running_total
        FROM transactions
    ),
    customer_final AS (
        SELECT
            customer_id,
            MAX(running_total) AS final_total
        FROM running
        GROUP BY customer_id
    )
    SELECT
        customer_id,
        final_total,
        RANK() OVER (ORDER BY final_total DESC) AS customer_rank
    FROM customer_final;

Separate transformation stages when one window depends on the result of another calculation.

### 3.14 Python implementation

    from collections import defaultdict

    def running_totals(rows):
        partitions = defaultdict(list)

        for row in rows:
            partitions[row["customer_id"]].append(row)

        result = []

        for customer_id, customer_rows in partitions.items():
            customer_rows.sort(
                key=lambda row: (row["occurred_at"], row["transaction_id"])
            )

            running = 0
            for row in customer_rows:
                running += row["amount"]
                output = dict(row)
                output["running_total"] = running
                result.append(output)

        return result

The Python implementation makes the two core requirements explicit:

    partition
    deterministic order

### 3.15 Memory considerations in Python

Window-style processing can require buffering a partition.

For large partitions:

- Avoid loading the entire dataset into memory.
- Process sorted partitions when possible.
- Use database window execution when appropriate.
- Bound state when the business rule permits it.
- Monitor unusually large partitions.

## 4. Testing

Window-function tests must validate partitioning, ordering, frames, ties, and grain.

### 4.1 Partition isolation

Create rows for two customers.

Verify that one customer's totals never include another customer's rows.

### 4.2 Deterministic ordering

Create two events with identical timestamps.

Verify that the unique tie-breaker determines their sequence consistently.

### 4.3 Running-total test

Given amounts:

    10
    20
    5

Expected cumulative values:

    10
    30
    35

### 4.4 LAG test

Given ordered amounts:

    10
    20
    5

Expected previous values:

    NULL
    10
    20

### 4.5 Ranking ties

Given amounts:

    100
    100
    50

Verify:

    ROW_NUMBER → 1, 2, 3
    RANK       → 1, 1, 3
    DENSE_RANK → 1, 1, 2

### 4.6 NULL ordering

Create NULL and non-NULL timestamps.

Verify the explicit `NULLS FIRST` or `NULLS LAST` policy.

### 4.7 Frame test

Compare `ROWS` and `RANGE` on tied ordering values.

Verify that the selected frame matches the business requirement.

### 4.8 Latest-row test

Create multiple records per business key.

Verify exactly one row receives `rn = 1`.

Repeat with tied timestamps and confirm the unique tie-breaker determines the result.

### 4.9 Duplicate-input test

Introduce duplicate events.

Determine whether duplicates are expected or represent source defects.

Verify the window output against the documented grain.

### 4.10 Temporal-order test

Create events whose ingestion time differs from event time.

Verify that the chosen ordering column matches the business definition.

### 4.11 Late-event test

Process an initial dataset.

Then insert an event whose event timestamp belongs in the middle of the existing sequence.

Verify that affected running totals, ranks, and previous/next relationships are recalculated as required.

### 4.12 Grain preservation

Before filtering:

    output row count = input row count

for a pure window annotation.

If a later `rn = 1` filter is applied, explicitly test the resulting new grain.

### 4.13 Idempotence

Run the same window transformation twice against identical snapshots.

Expected:

    identical ordered business-key results
    identical derived values

## 5. Observability

Window functions can be logically correct while still producing operationally dangerous results when partitions or ordering change.

### Core metrics

| Metric | Meaning |
|---|---|
| `window_input_rows` | Rows entering the window stage |
| `window_output_rows` | Rows leaving the stage |
| `window_partition_count` | Number of partitions |
| `window_max_partition_rows` | Largest partition size |
| `window_null_order_rows` | Rows with NULL ordering values |
| `window_tie_count` | Ordering ties requiring tie-breakers |
| `window_late_event_rows` | Late records affecting prior windows |
| `window_rule_version` | Version of window semantics |

### Partition-size monitoring

Very large partitions can cause:

- Memory pressure.
- Sort pressure.
- Long-running queries.
- Skewed execution.

Monitor maximum and high-percentile partition sizes.

### Ordering diagnostics

Track:

- Number of NULL ordering values.
- Number of duplicate ordering keys.
- Number of records requiring tie-breakers.

A sudden increase can change ranking and latest-row results.

### Window correctness checks

For a running total, compare final partition totals with an independent aggregation.

For a latest-row model, verify one selected row per business key.

For a ranking model, verify expected partition counts.

### Late-data monitoring

Track how many new records fall before the latest processed event time.

Late records are especially important when windows depend on historical sequence.

## 6. Intentional Failure

### Failure 1 — Remove the tie-breaker

Create two events with identical timestamps.

Expected symptom:

- `ROW_NUMBER()` assignment can become unstable.

Recovery:

Add a deterministic unique ordering key.

### Failure 2 — Remove PARTITION BY

Calculate a customer running total without partitioning.

Expected symptom:

- Different customers contribute to one cumulative sequence.

Recovery:

Restore the correct partition key.

### Failure 3 — Use the wrong ordering timestamp

Order events by ingestion time when the business requirement is event time.

Expected symptom:

- Historical sequence and derived metrics are incorrect for late-arriving events.

Recovery:

Use the correct business-time column and define replay behavior.

### Failure 4 — Use a seven-row window for a seven-day metric

Create sparse events.

Expected symptom:

- The calculation covers seven records rather than seven days.

Recovery:

Implement an actual time-based window strategy.

### Failure 5 — Use default frame assumptions

Apply `LAST_VALUE()` without understanding the current-row frame.

Expected symptom:

- The result may represent the current row rather than the final partition value.

Recovery:

Define the frame explicitly.

### Failure 6 — Use JOIN instead of a window

Implement previous-row logic using a self-join with ambiguous ordering.

Expected symptom:

- Duplicate or missing relationships.

Recovery:

Use `LAG` or another appropriate window function.

### Failure 7 — Filter before calculating

Remove historical rows before computing a running total.

Expected symptom:

- The cumulative value starts from an incomplete population.

Recovery:

Apply filtering at the correct stage relative to the window calculation.

### Failure 8 — Ignore partition skew

Create one partition containing millions of rows.

Expected symptom:

- One partition dominates execution time or memory usage.

Recovery:

Inspect partition distribution and redesign the calculation or processing strategy if required.

## 7. Recovery

Window-function incidents require identifying which component of the window contract changed.

### Recovery sequence

1. Identify affected partitions and runs.
2. Capture input snapshot and processing watermark.
3. Verify partition keys.
4. Verify ordering keys.
5. Verify tie-breakers.
6. Verify NULL ordering policy.
7. Verify window frame.
8. Check late-arriving records.
9. Compare derived values against an independent calculation.
10. Correct the transformation.
11. Recompute the affected partition or replay boundary.
12. Reconcile output grain and counts.
13. Replay downstream processing idempotently.
14. Record the root cause.

### Recovering unstable ranking

If latest-row or ranking results changed unexpectedly:

1. Check for ordering ties.
2. Check whether the tie-breaker is unique.
3. Check NULL ordering.
4. Compare the previous and current ordering definitions.
5. Re-run affected partitions.

### Recovering running totals after late data

If a late event belongs inside an already processed sequence:

1. Identify the earliest affected event time.
2. Identify the affected partition keys.
3. Recompute from the earliest affected boundary.
4. Replace affected derived records atomically.
5. Validate final partition totals.

### Recovering partition changes

If the partition key changed:

- Treat it as a semantic transformation change.
- Version the rule.
- Identify all affected records.
- Recompute old and new partitions as required.
- Reconcile population and derived metrics.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL provides extensive window-function support and query planning tools.

Learn `EXPLAIN`, sort behavior, indexes, memory settings, and execution plans for large window queries.

### 2. dbt

dbt models commonly use window functions for deduplication, latest-state selection, ranking, and analytical transformations.

Use dbt tests to verify grain, uniqueness, and expected partition behavior.

### 3. DuckDB

DuckDB provides strong analytical SQL support and is useful for reproducing window behavior against Parquet and other local datasets.

It is valuable for testing frames, ranking, and large analytical windows before deploying a production transformation.

## 9. Production Runbook

### Before deployment

- [ ] Define input grain.
- [ ] Define partition key.
- [ ] Define deterministic ordering.
- [ ] Define tie-breaker.
- [ ] Define NULL ordering.
- [ ] Define frame semantics.
- [ ] Decide whether the window is row-based or time-based.
- [ ] Define late-data replay behavior.
- [ ] Test partition isolation.
- [ ] Test duplicate ordering values.
- [ ] Test NULL ordering.
- [ ] Test output grain.

### During execution

- [ ] Record input and output counts.
- [ ] Record partition count.
- [ ] Record largest partition.
- [ ] Record ordering ties.
- [ ] Record NULL ordering values.
- [ ] Record late-event count.
- [ ] Record rule version.

### If ranking changes unexpectedly

1. Check ordering ties.
2. Check tie-breaker stability.
3. Check NULL ordering.
4. Check new or corrected events.
5. Compare rule versions.

### If running totals change unexpectedly

1. Check late events.
2. Check event ordering.
3. Check partition boundaries.
4. Check frame definition.
5. Compare final totals with independent aggregation.

### If execution becomes slow

1. Inspect the query plan.
2. Inspect partition distribution.
3. Find the largest partitions.
4. Check sorting requirements.
5. Reduce unnecessary columns before the window stage.
6. Restrict the input scope where the contract allows it.

## 10. Common Mistakes

### Mistake 1 — Treating window functions like GROUP BY

A window preserves rows; GROUP BY collapses them.

### Mistake 2 — Using non-deterministic ordering

Business logic based on `ROW_NUMBER()` requires stable ordering.

### Mistake 3 — Ignoring ties

Equal ordering values can produce ambiguous sequence results.

### Mistake 4 — Forgetting PARTITION BY

Rows from unrelated entities can enter the same calculation.

### Mistake 5 — Ignoring window frames

Especially dangerous with `LAST_VALUE`, running calculations, and peer groups.

### Mistake 6 — Confusing ROWS with RANGE

They can produce different results when ordering values tie.

### Mistake 7 — Using row count as elapsed time

Seven rows do not necessarily represent seven days.

### Mistake 8 — Filtering before a required historical window

Removing rows too early changes the calculation.

### Mistake 9 — Ignoring late-arriving data

Late events can change previously calculated windows.

### Mistake 10 — Ignoring partition skew

One enormous partition can dominate resource usage.

### Mistake 11 — Nesting windows at the same query level

Use explicit transformation stages.

### Mistake 12 — Using a window to retrieve arbitrary related attributes

Define deterministic ordering and selection criteria first.

## 11. Definition of Done

The window-function transformation is complete when you can:

- [ ] Explain the difference between GROUP BY and window functions.
- [ ] Define a window partition.
- [ ] Define deterministic ordering.
- [ ] Add stable tie-breakers.
- [ ] Explain window frames.
- [ ] Distinguish ROWS from RANGE.
- [ ] Use aggregate window functions.
- [ ] Use ROW_NUMBER, RANK, and DENSE_RANK correctly.
- [ ] Use LAG and LEAD.
- [ ] Use FIRST_VALUE and LAST_VALUE safely.
- [ ] Build running totals.
- [ ] Build bounded moving windows.
- [ ] Calculate partition shares.
- [ ] Filter window results using a later query stage.
- [ ] Handle NULL ordering explicitly.
- [ ] Handle late-arriving events.
- [ ] Test partition isolation and grain.
- [ ] Detect partition skew.
- [ ] Intentionally break ordering and frame logic.
- [ ] Recover affected partitions safely.
- [ ] Reconcile derived metrics with independent calculations.
- [ ] Explain how PostgreSQL, dbt, and DuckDB support production window workloads.

## 12. What You Learned

Window functions let you reason across related rows without immediately collapsing them.

The production workflow is:

    DEFINE INPUT GRAIN
         ↓
    DEFINE PARTITION
         ↓
    DEFINE DETERMINISTIC ORDER
         ↓
    DEFINE FRAME
         ↓
    APPLY WINDOW FUNCTION
         ↓
    VALIDATE GRAIN
         ↓
    CHECK TIES / NULLS / LATE DATA
         ↓
    RECONCILE DERIVED VALUES
         ↓
    REPLAY AFFECTED PARTITIONS

> **A window function is not just SQL syntax. It is a contract about population, partition, order, frame, and time. Make all five explicit when correctness depends on them.**

### Next recipe

**T27 — Ranking**