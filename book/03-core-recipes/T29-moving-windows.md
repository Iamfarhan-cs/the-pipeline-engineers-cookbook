# T29 — Moving Windows

> **Goal:** Learn how to calculate rolling metrics over recent rows or time periods, while making window boundaries, sparse data, ordering, late events, and production performance explicit.

## 1. Problem Recognition

A moving window answers:

    What is the metric over the recent window surrounding this row?

Typical production cases include:

- Seven-day rolling revenue.
- Thirty-day active-user counts.
- Rolling transaction averages.
- Recent error rates.
- Moving payment volume.
- Rolling inventory demand.
- Trailing latency metrics.
- Recent event counts per customer.
- Rolling fraud or risk indicators.

Moving windows are different from running totals.

A running total normally expands from the beginning of a sequence.

A moving window keeps a bounded recent region.

### Recognition questions

Before implementing a moving window, ask:

1. Is the window row-based or time-based?
2. What defines the ordering?
3. What is the exact lower and upper boundary?
4. Is the current row included?
5. Are boundaries inclusive or exclusive?
6. What happens with sparse data?
7. What happens with duplicate timestamps?
8. What timezone defines the business period?
9. How are late events handled?
10. What should happen when the window has insufficient history?

## 2. Concept and Reasoning

### 2.1 Moving window semantics

Suppose ordered values are:

    10
    20
    30
    40

A three-row trailing window can produce:

    row 1 → 10
    row 2 → 10 + 20
    row 3 → 10 + 20 + 30
    row 4 → 20 + 30 + 40

The window moves as the current row advances.

### 2.2 Running versus moving windows

Running:

    UNBOUNDED PRECEDING → CURRENT ROW

Moving:

    bounded preceding rows → CURRENT ROW

Example:

    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW

This is a seven-row trailing window.

### 2.3 Row-based windows

A row-based window counts physical rows.

Example:

    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW

means:

    current row + previous 6 rows

It does not mean seven calendar days.

### 2.4 Time-based windows

A time-based requirement asks:

    What happened during the previous seven days?

This is different from:

    What happened during the previous seven rows?

If events are sparse, seven rows could represent several months.

If events are dense, seven rows could represent a few minutes.

Never substitute row count for elapsed time without verifying that the business definition permits it.

### 2.5 Sparse data

Consider events:

    Jan 1
    Jan 20
    Feb 10

A seven-row window cannot represent a seven-day window because there are not seven daily observations.

For time-based analytics, the data model and query must preserve the intended time interval.

### 2.6 Current-row inclusion

Many trailing windows include the current row:

    ... PRECEDING AND CURRENT ROW

If the requirement is prior-period-only:

    ... PRECEDING AND 1 PRECEDING

These are different metrics.

Explicitly define whether the current observation participates.

### 2.7 Window boundaries

A moving-window contract should specify:

    start boundary
    end boundary
    inclusion semantics

Example:

    previous 7 complete days

is not necessarily:

    previous 7 days including the current partial day

### 2.8 Calendar windows versus elapsed-duration windows

These can differ around:

- Day boundaries.
- Month boundaries.
- Daylight-saving transitions.
- Business calendars.
- Timezone changes.

Use the business calendar when the requirement is calendar-based.

Use an elapsed interval when the requirement is duration-based.

### 2.9 Timezone semantics

A window such as:

    daily active users

requires a definition of day.

Possible definitions:

    UTC calendar day
    Europe/Berlin calendar day
    customer-local calendar day
    business timezone

The same event can belong to different days under different timezone policies.

### 2.10 Duplicate timestamps

Two events can share the same timestamp.

For row-based calculations, add a deterministic tie-breaker:

    ORDER BY occurred_at, event_id

For time-based calculations, decide whether equal timestamps should all belong to the same temporal boundary.

### 2.11 RANGE versus ROWS

`ROWS` uses physical row positions.

`RANGE` works from ordering values and can include peers with equal ordering values.

Example:

    RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW

expresses a temporal interval in systems that support this form.

The exact syntax and supported data types depend on the database engine.

### 2.12 GROUPS frames

`GROUPS` operates on peer groups rather than individual rows.

It can be useful when tied ordering values should be treated as one logical group.

Do not use it simply because it exists; choose the frame that matches the business semantics.

### 2.13 Moving SUM

Example:

    SUM(amount) OVER (
        PARTITION BY customer_id
        ORDER BY occurred_at, event_id
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_sum

### 2.14 Moving AVG

Example:

    AVG(amount) OVER (
        PARTITION BY customer_id
        ORDER BY occurred_at, event_id
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_average

### 2.15 Moving COUNT

Example:

    COUNT(*) OVER (
        PARTITION BY customer_id
        ORDER BY occurred_at, event_id
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_count

### 2.16 Moving MIN and MAX

Example:

    MAX(amount) OVER (
        PARTITION BY customer_id
        ORDER BY occurred_at, event_id
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_max

These can identify recent peaks and troughs.

### 2.17 Rolling rate

A rolling rate often requires two rolling quantities:

    successful_events / total_events

Use:

    rolling_successes / NULLIF(rolling_events, 0)

rather than averaging already-aggregated daily rates unless weighted and unweighted interpretations are intentionally equivalent.

### 2.18 Weighted versus unweighted rolling averages

Suppose daily conversion rates are:

    Day 1 → 1 / 10
    Day 2 → 90 / 100

Simple average:

    (10% + 90%) / 2 = 50%

Aggregate conversion:

    91 / 110 ≈ 82.7%

These answer different questions.

For event-level conversion, calculate rolling successes and rolling opportunities first.

### 2.19 Moving distinct counts

Rolling distinct counts are more difficult than ordinary rolling counts.

Example:

    unique customers active in the last 30 days

Simply using `COUNT(*)` is incorrect when the same customer appears repeatedly.

Possible implementations include:

- Generate a calendar or interval relation.
- Deduplicate to one entity/day before aggregation.
- Use database-specific distinct-window capabilities where available.
- Use an incremental stateful approach.

Choose the design based on scale and exactness requirements.

### 2.20 Moving windows and aggregation grain

If input is event-level but the requirement is daily rolling revenue, first establish daily grain when appropriate:

    events
       ↓
    daily aggregate
       ↓
    rolling window

This can dramatically reduce window size.

### 2.21 Rolling windows over daily facts

Example:

    date | revenue
    -----+--------
    D1   | 100
    D2   | 150
    D3   | 120

A seven-day rolling sum is naturally defined over daily rows if the dataset guarantees one row per day per entity.

If dates can be missing, determine whether missing days represent zero activity or missing data.

### 2.22 Missing dates are not automatically zero

Suppose:

    Jan 1 → 100
    Jan 3 → 50

Does Jan 2 mean:

    zero activity

or:

    missing data?

A rolling metric can change materially depending on this interpretation.

Create a calendar spine when the business model requires explicit zero-activity dates.

### 2.23 Calendar spine

Conceptual model:

    calendar dates
         ×
    business entities
         ↓
    left join activity
         ↓
    fill missing activity with zero
         ↓
    calculate rolling metric

Do not fill missing data with zero unless the domain contract says absence means zero.

### 2.24 Rolling windows and late events

A late event can change every subsequent window that contains it.

Example:

    Event on Jan 5 arrives on Jan 10.

It can affect rolling metrics for windows spanning Jan 5 through the configured trailing boundary.

Therefore late-data handling must define the affected date range.

### 2.25 Rolling windows and corrections

Changing one historical value can affect many later windows.

The impact radius depends on the window width.

For a seven-day trailing metric, a correction on day D can affect output dates roughly from D through D+6, subject to exact boundary semantics.

### 2.26 Incremental processing

Do not assume that a moving window can always be updated by processing only today's records.

A new event can affect historical output rows.

Incremental processing should maintain enough lookback and replay scope to recompute affected windows.

### 2.27 Lookback boundary

If the window is seven days and processing a new day:

    current output date = D
    required history = enough data to calculate D's seven-day window

If historical corrections are possible, the replay boundary may need to extend beyond the newly arrived data.

### 2.28 Window edge semantics

These are different:

    last 7 calendar days including today
    previous 7 complete calendar days
    previous 168 hours
    previous 7 observations

Use explicit language and implementation for each.

### 2.29 Moving window and data quality

Rolling metrics can hide missing data.

Example:

    seven-day average calculated from only two available days

That may be mathematically valid but operationally misleading.

Track:

    observation_count
    expected_observation_count
    coverage_ratio

when completeness matters.

### 2.30 Moving windows and denominator quality

For rates, always inspect the denominator.

A 100% rate based on one event is not equivalent to a 100% rate based on 100,000 events.

Track the rolling denominator alongside the rate.

## 3. Implementation

### 3.1 Define the contract

Example:

    Input grain:        one row per customer/day
    Metric:             daily revenue
    Window:             trailing 7 calendar days
    Current day:        included
    Timezone:            UTC
    Missing days:       explicit zero after calendar expansion
    Late data:          recompute affected days
    Coverage:            monitored

### 3.2 PostgreSQL example schema

    CREATE TABLE daily_customer_revenue (
        customer_id BIGINT NOT NULL,
        business_date DATE NOT NULL,
        revenue NUMERIC(20,4) NOT NULL,
        PRIMARY KEY (customer_id, business_date)
    );

### 3.3 Seven-row trailing sum

    SELECT
        customer_id,
        business_date,
        revenue,
        SUM(revenue) OVER (
            PARTITION BY customer_id
            ORDER BY business_date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS rolling_7_row_revenue
    FROM daily_customer_revenue;

This is a seven-row window, not automatically a seven-day window.

### 3.4 Seven-day temporal window

On PostgreSQL, when the business requirement is a temporal interval, one approach is:

    SELECT
        a.customer_id,
        a.business_date,
        SUM(b.revenue) AS rolling_7_day_revenue
    FROM daily_customer_revenue AS a
    JOIN daily_customer_revenue AS b
      ON b.customer_id = a.customer_id
     AND b.business_date >= a.business_date - 6
     AND b.business_date <= a.business_date
    GROUP BY
        a.customer_id,
        a.business_date;

This makes the date boundary explicit and works even when some dates are absent.

Validate whether missing dates represent zero or missing data before using this pattern.

### 3.5 Calendar-spine approach

First generate the required dates:

    SELECT
        d::date AS business_date
    FROM generate_series(
        DATE '2026-09-01',
        DATE '2026-09-30',
        INTERVAL '1 day'
    ) AS d;

Then cross the required entity population with the calendar, left join activity, and apply the rolling window.

### 3.6 Rolling average

    SELECT
        customer_id,
        business_date,
        AVG(revenue) OVER (
            PARTITION BY customer_id
            ORDER BY business_date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS rolling_7_row_avg
    FROM daily_customer_revenue;

### 3.7 Rolling count

    SELECT
        customer_id,
        business_date,
        COUNT(*) OVER (
            PARTITION BY customer_id
            ORDER BY business_date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS rolling_observation_count
    FROM daily_customer_revenue;

### 3.8 Rolling success rate

    WITH daily AS (
        SELECT
            business_date,
            customer_id,
            COUNT(*) AS attempts,
            COUNT(*) FILTER (WHERE status = 'SUCCESS') AS successes
        FROM payment_events
        GROUP BY business_date, customer_id
    ),
    rolling AS (
        SELECT
            customer_id,
            business_date,
            SUM(attempts) OVER (
                PARTITION BY customer_id
                ORDER BY business_date
                ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
            ) AS rolling_attempts,
            SUM(successes) OVER (
                PARTITION BY customer_id
                ORDER BY business_date
                ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
            ) AS rolling_successes
        FROM daily
    )
    SELECT
        customer_id,
        business_date,
        rolling_successes / NULLIF(rolling_attempts, 0)::numeric
            AS rolling_success_rate
    FROM rolling;

This computes the rate from rolling counts rather than averaging daily rates.

### 3.9 Rolling maximum

    MAX(revenue) OVER (
        PARTITION BY customer_id
        ORDER BY business_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7_row_max

### 3.10 Rolling minimum

    MIN(revenue) OVER (
        PARTITION BY customer_id
        ORDER BY business_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7_row_min

### 3.11 Rolling change

One useful pattern is:

    current_value - prior_window_value

where the prior value is calculated in a separate stage using `LAG` or another appropriate window.

Do not nest window functions directly when the SQL engine does not permit it; use a CTE.

### 3.12 Rolling distinct entities

For a metric such as:

    unique customers active in the last 30 days

one practical design is to first create one row per customer/day:

    customer_id + activity_date

Then construct the required date/entity population and count distinct customers over the temporal interval.

For large datasets, exact rolling distinct counts can become expensive. Consider whether approximate algorithms are acceptable for the business requirement.

### 3.13 Python implementation

    from collections import deque, defaultdict

    def rolling_sum(rows, window_size):
        partitions = defaultdict(list)

        for row in rows:
            partitions[row["customer_id"]].append(row)

        result = []

        for customer_id, customer_rows in partitions.items():
            customer_rows.sort(
                key=lambda row: (row["business_date"], row["event_id"])
            )

            values = deque()
            running = 0

            for row in customer_rows:
                values.append(row["amount"])
                running += row["amount"]

                if len(values) > window_size:
                    running -= values.popleft()

                output = dict(row)
                output["rolling_sum"] = running
                result.append(output)

        return result

This implements a row-based moving window efficiently without recomputing the entire sum for every row.

### 3.14 Time-based Python window

For a time-based window, maintain timestamped events and remove entries older than the configured boundary.

Conceptually:

    append current event
    add current value
    remove values outside time boundary
    emit current aggregate

The exact implementation must define whether boundaries are inclusive or exclusive.

### 3.15 Incremental recomputation

When late data arrives:

1. Identify the affected entity.
2. Identify the earliest affected date/time.
3. Calculate the maximum forward impact of the window.
4. Recompute the affected output range.
5. Replace it atomically.

For a trailing seven-day window, a correction can affect later output dates that still include the corrected date.

## 4. Testing

Moving-window tests must validate boundaries rather than only aggregate values.

### 4.1 Basic three-row window

Input:

    10
    20
    30
    40

Three-row trailing sums:

    10
    30
    60
    90

### 4.2 Current-row inclusion

Compare:

    2 preceding + current

with:

    3 preceding + 1 preceding

Verify the requirement explicitly.

### 4.3 Sparse-date test

Use:

    Jan 1
    Jan 10
    Jan 20

Verify that a seven-day temporal window does not behave like a three-row window.

### 4.4 Missing-date semantics

Create a missing calendar date.

Verify whether the contract treats it as:

    zero activity

or:

    missing data

Do not silently choose one.

### 4.5 Boundary test

Place events exactly at the lower boundary.

Verify whether they are included.

Also test one unit before and after the boundary.

### 4.6 Timestamp tie test

Create multiple events at the same timestamp.

Verify deterministic row ordering where row-based semantics require it.

### 4.7 Rolling-rate test

Use:

    Day 1 → 1 success / 1 attempt
    Day 2 → 9 successes / 99 attempts

Verify that:

    rolling event rate = 10 / 100 = 10%

rather than the unweighted average of 100% and approximately 9.09%.

### 4.8 Insufficient-history test

Verify the expected behavior when fewer observations than the configured window exist.

Possible policies:

- Return partial-window metric.
- Return NULL until the full window exists.
- Return metric plus coverage metadata.

Choose explicitly.

### 4.9 Late-event test

Calculate a seven-day metric.

Insert a historical event.

Verify every output date whose window includes that event is recomputed.

### 4.10 Correction test

Change a historical value.

Verify all affected windows change and unaffected windows remain unchanged.

### 4.11 Partition isolation

Create two customers with overlapping dates.

Verify no values cross customer boundaries.

### 4.12 Timezone test

Create events near midnight UTC and the business timezone boundary.

Verify that daily windows follow the documented timezone.

### 4.13 Calendar-spine test

Create activity on only two of seven dates.

Verify the expected coverage and zero/missing-day policy.

### 4.14 Idempotence

Run the same snapshot twice.

Expected:

    identical window boundaries
    identical derived values

## 5. Observability

Moving metrics need visibility into both calculation coverage and boundary behavior.

### Core metrics

| Metric | Meaning |
|---|---|
| `moving_window_input_rows` | Rows entering the calculation |
| `moving_window_output_rows` | Rows receiving metrics |
| `moving_window_partition_count` | Independent entity streams |
| `moving_window_max_partition_rows` | Largest partition |
| `moving_window_late_rows` | Late events affecting prior windows |
| `moving_window_boundary_rows` | Rows near window boundaries |
| `moving_window_partial_windows` | Outputs with insufficient history |
| `moving_window_missing_periods` | Expected periods with absent data |
| `moving_window_coverage_ratio` | Observed / expected coverage |
| `moving_window_rule_version` | Window semantics version |

### Coverage monitoring

A rolling metric can look normal while data coverage deteriorates.

Track:

    observed periods
    expected periods
    coverage ratio

when the domain expects regular observations.

### Boundary diagnostics

Track events entering and leaving the window.

Boundary bugs often appear as:

- One-day shifts.
- Off-by-one intervals.
- Duplicate inclusion.
- Unexpected exclusion.

### Late-data diagnostics

Track:

- Late event count.
- Earliest affected timestamp.
- Number of recomputed output rows.
- Reprocessing duration.

### Rate denominator monitoring

For rolling rates, track numerator and denominator separately.

A rate without its denominator can hide severe volume changes.

### Performance metrics

Monitor:

- Window query duration.
- Largest partition.
- Rows scanned.
- Sort time where available.
- Memory usage.
- Spill-to-disk indicators where available.

## 6. Intentional Failure

### Failure 1 — Use seven rows for seven days

Create sparse events.

Expected symptom:

- The metric covers the wrong elapsed period.

Recovery:

Use a temporal window or a calendar-spine design.

### Failure 2 — Shift the lower boundary

Change a seven-day inclusive boundary to an exclusive one.

Expected symptom:

- Events exactly on the boundary disappear.

Recovery:

Make interval semantics explicit and add boundary tests.

### Failure 3 — Average daily rates

Use unweighted averages for a volume-sensitive rate.

Expected symptom:

- Low-volume days have the same influence as high-volume days.

Recovery:

Aggregate rolling numerators and denominators first.

### Failure 4 — Treat missing dates as zero

Remove calendar rows and assume absence means zero.

Expected symptom:

- Coverage and averages are distorted when missing data means unknown.

Recovery:

Distinguish zero activity from missing observations.

### Failure 5 — Ignore timezone

Create events near midnight.

Expected symptom:

- Events appear in the wrong business day.

Recovery:

Define and apply the business timezone before windowing.

### Failure 6 — Ignore late events

Insert a historical event.

Expected symptom:

- Historical rolling metrics remain stale.

Recovery:

Recompute all affected output windows.

### Failure 7 — Use RANGE or ROWS without understanding peers

Create tied timestamps.

Expected symptom:

- Window values differ from the intended peer semantics.

Recovery:

Choose `ROWS`, `RANGE`, or `GROUPS` based on the actual requirement.

### Failure 8 — Recompute full history for every correction

Introduce a small historical correction.

Expected symptom:

- Excessive processing and unnecessary downstream churn.

Recovery:

Calculate the affected entity and forward impact boundary.

## 7. Recovery

Moving-window incidents usually involve incorrect boundaries, missing coverage, or insufficient replay.

### Recovery sequence

1. Identify the affected metric and reporting period.
2. Capture source snapshot and timezone.
3. Verify window definition.
4. Verify current-row inclusion.
5. Verify lower and upper boundaries.
6. Verify row-based versus time-based semantics.
7. Verify missing-period policy.
8. Verify late-data scope.
9. Identify affected partitions.
10. Recompute the affected output range.
11. Compare boundary cases.
12. Validate coverage metrics.
13. Replay downstream outputs idempotently.
14. Record the root cause.

### Recovering boundary errors

1. Reproduce the boundary with a minimal dataset.
2. Test exactly-on-boundary, just-before, and just-after values.
3. Correct interval semantics.
4. Add regression tests.
5. Recompute affected periods.

### Recovering from late data

1. Identify the late event timestamp.
2. Identify affected entity.
3. Determine the maximum forward window impact.
4. Recompute only the affected output range.
5. Validate unaffected rows remain unchanged.

### Recovering from coverage loss

If expected daily rows are missing:

1. Determine whether missing means zero or unknown.
2. Rebuild the calendar/entity spine if required.
3. Recalculate metrics.
4. Track coverage separately.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL supports window frames, temporal predicates, date generation, aggregation, and query planning useful for moving metrics.

Learn to inspect plans and understand sort and join costs for large windows.

### 2. dbt

dbt models can build daily facts, calendar spines, rolling metrics, and data-quality checks.

Use tests to validate coverage, grain, and boundary behavior.

### 3. DuckDB

DuckDB is useful for reproducing rolling calculations over Parquet and testing boundary behavior locally.

It is especially useful for comparing row-based and time-based window designs.

## 9. Production Runbook

### Before deployment

- [ ] Define metric.
- [ ] Define entity grain.
- [ ] Define row-based or time-based semantics.
- [ ] Define window width.
- [ ] Define boundary inclusion.
- [ ] Define current-row inclusion.
- [ ] Define timezone.
- [ ] Define missing-period semantics.
- [ ] Define late-event replay.
- [ ] Define insufficient-history behavior.
- [ ] Define coverage metrics.
- [ ] Test boundary cases.

### During execution

- [ ] Record input/output rows.
- [ ] Record partition count.
- [ ] Record largest partition.
- [ ] Record late events.
- [ ] Record partial windows.
- [ ] Record missing periods.
- [ ] Record coverage ratio.
- [ ] Record rule version.

### If rolling values shift unexpectedly

1. Check source changes.
2. Check boundary definitions.
3. Check timezone.
4. Check missing periods.
5. Check late events.
6. Check row-based versus time-based semantics.

### If rolling rates change unexpectedly

1. Check numerator.
2. Check denominator.
3. Check whether daily rates were averaged incorrectly.
4. Check missing-day handling.
5. Check the reporting population.

### If the calculation is slow

1. Inspect the query plan.
2. Measure partition sizes.
3. Aggregate to the correct grain before windowing.
4. Reduce unnecessary columns.
5. Restrict recomputation to affected ranges.
6. Evaluate whether approximate distinct methods are acceptable for expensive metrics.

## 10. Common Mistakes

### Mistake 1 — Treating rows as time

Seven rows are not necessarily seven days.

### Mistake 2 — Leaving boundaries implicit

Off-by-one errors are common in temporal analytics.

### Mistake 3 — Ignoring current-row inclusion

Current-day and prior-complete-period metrics differ.

### Mistake 4 — Ignoring sparse data

Missing observations can change the meaning of a rolling average.

### Mistake 5 — Treating missing as zero automatically

Missing data and zero activity are not universally equivalent.

### Mistake 6 — Averaging rates instead of aggregating counts

Unweighted rates can be misleading.

### Mistake 7 — Ignoring timezone

Daily windows require a day definition.

### Mistake 8 — Ignoring late events

Historical events can change multiple future windows.

### Mistake 9 — Ignoring frame semantics

`ROWS`, `RANGE`, and `GROUPS` do not mean the same thing.

### Mistake 10 — Ignoring coverage

A mathematically correct metric can still be operationally incomplete.

### Mistake 11 — Recomputing all history

Use bounded replay when the impact radius is known.

### Mistake 12 — Ignoring partition skew

Large entity partitions can dominate execution cost.

## 11. Definition of Done

The moving-window transformation is complete when you can:

- [ ] Explain moving-window semantics.
- [ ] Distinguish running and moving windows.
- [ ] Distinguish row-based and time-based windows.
- [ ] Define explicit boundaries.
- [ ] Define current-row inclusion.
- [ ] Handle sparse data.
- [ ] Handle missing periods.
- [ ] Build calendar spines when required.
- [ ] Handle timezone semantics.
- [ ] Explain `ROWS`, `RANGE`, and `GROUPS`.
- [ ] Calculate rolling sums, averages, counts, minimums, and maximums.
- [ ] Calculate rolling rates from appropriate numerators and denominators.
- [ ] Design rolling distinct metrics.
- [ ] Handle late events and corrections.
- [ ] Define incremental replay boundaries.
- [ ] Monitor coverage and partial windows.
- [ ] Test exact boundary cases.
- [ ] Intentionally break window semantics.
- [ ] Recover affected output ranges.
- [ ] Explain how PostgreSQL, dbt, and DuckDB support production moving-window workloads.

## 12. What You Learned

A moving window is a bounded analytical state over an ordered population.

The production workflow is:

    DEFINE GRAIN
         ↓
    DEFINE TIME / ROW SEMANTICS
         ↓
    DEFINE WINDOW WIDTH
         ↓
    DEFINE BOUNDARIES
         ↓
    DEFINE TIMEZONE
         ↓
    DEFINE MISSING-DATA POLICY
         ↓
    APPLY WINDOW
         ↓
    VALIDATE COVERAGE
         ↓
    HANDLE LATE DATA
         ↓
    RECOMPUTE AFFECTED RANGE

> **A rolling metric is correct only when its population, time semantics, boundaries, coverage, and replay behavior are explicitly defined.**

### Next recipe

**T30 — Deduplication with Window Functions**