# T28 — Running Totals

> **Goal:** Learn how to calculate cumulative values correctly over an ordered event stream, including deterministic ordering, partitioned totals, resets, negative adjustments, late events, and production reconciliation.

## 1. Problem Recognition

A running total answers:

    What is the cumulative value up to this row?

Typical production cases include:

- Account balances.
- Cumulative payment volume.
- Running sales totals.
- Inventory quantities.
- Cumulative event counts.
- Cumulative fees.
- Usage counters.
- Progressive budget consumption.
- Running operational KPIs.

A running total is order-sensitive.

Unlike a simple partition total, the answer depends on where the current row sits in the sequence.

### Recognition questions

Before implementing a running total, ask:

1. What entity owns the cumulative state?
2. What defines event order?
3. Is ordering based on event time or ingestion time?
4. Is there a deterministic tie-breaker?
5. What is the starting balance?
6. Can values be negative?
7. Can the cumulative sequence reset?
8. Can events arrive late?
9. Are corrections or reversals possible?
10. What invariant can independently verify the result?

## 2. Concept and Reasoning

### 2.1 Running total semantics

For ordered values:

    10
    20
    -5
    15

the running totals are:

    10
    30
    25
    40

Each row includes the cumulative contribution of all preceding rows in the defined sequence, including the current row.

### 2.2 The canonical SQL pattern

    SUM(amount) OVER (
        PARTITION BY account_id
        ORDER BY occurred_at, transaction_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total

The three important components are:

    PARTITION BY → independent cumulative streams
    ORDER BY     → sequence
    ROWS frame   → cumulative row range

### 2.3 Running total versus partition total

Partition total:

    SUM(amount) OVER (PARTITION BY account_id)

Every account row receives the same final total.

Running total:

    SUM(amount) OVER (
        PARTITION BY account_id
        ORDER BY occurred_at, transaction_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )

Each row receives the total as of that point in the sequence.

### 2.4 Why ORDER BY is mandatory

Without an ordering definition, there is no concept of 'up to this row'.

A cumulative metric requires a sequence.

Bad:

    SUM(amount) OVER (PARTITION BY account_id)

This is a partition total, not a running total.

Correct:

    SUM(amount) OVER (
        PARTITION BY account_id
        ORDER BY occurred_at, transaction_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )

### 2.5 Deterministic ordering

Suppose two transactions have the same timestamp:

    T1 → 10:00 → +100
    T2 → 10:00 → -50

The timestamp alone does not define which transaction happens first.

Use a deterministic tie-breaker:

    ORDER BY occurred_at, transaction_id

Without it, intermediate running totals can be unstable.

### 2.6 ROWS versus RANGE

For running totals, an explicit `ROWS` frame is often important.

Example:

    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

This means physical rows up to the current row.

`RANGE` can include peer rows sharing the same ordering value.

Therefore tied timestamps can produce different cumulative behavior depending on the frame.

### 2.7 Event-time running totals

If the business meaning is:

    balance as of when the transaction occurred

order by event time.

If the meaning is:

    balance as transactions were received

order by ingestion time.

These are different metrics.

Do not substitute ingestion order for business event order without an explicit requirement.

### 2.8 Negative values

Running totals commonly include negative movements:

- Refunds.
- Withdrawals.
- Inventory consumption.
- Chargebacks.
- Corrections.
- Reversals.

The transformation should not assume that all amounts are positive.

### 2.9 Starting balance

Sometimes the cumulative sequence begins with an opening balance.

Conceptually:

    running_balance = opening_balance + cumulative_movements

Example:

    opening balance = 1,000
    movement       = +100
    movement       = -50

balances:

    1,100
    1,050

The opening balance must be associated with the correct account and period.

### 2.10 Opening balance as a separate input

A robust design can keep:

    account_opening_balance

separate from:

    account_transactions

Then calculate:

    opening_balance
        + SUM(movement) OVER (...)

This makes the starting state auditable.

### 2.11 Resetting a running total

Some cumulative metrics reset by business period.

Examples:

- Daily cumulative sales.
- Monthly budget consumption.
- Per-session event count.
- Per-batch sequence.

Use the reset boundary as part of the partition definition.

Example:

    PARTITION BY account_id, business_date

Then the running total starts again for each account/day.

### 2.12 Reset events

Sometimes a reset is represented by a row in the event stream rather than a date boundary.

Example:

    +100
    +50
    RESET
    +20

The sequence after RESET requires a new logical segment.

This generally requires a preparatory stage that identifies reset boundaries and creates a segment identifier before applying the window.

### 2.13 Segmenting after reset

Conceptually:

    event stream
        ↓
    identify reset rows
        ↓
    cumulative reset counter
        ↓
    segment_id
        ↓
    running SUM partitioned by segment

The reset logic should be separated from the cumulative calculation.

### 2.14 Late-arriving events

A late event can change every running total after its insertion point.

Example:

Original:

    09:00 +100 → 100
    10:00 +50  → 150
    11:00 +25  → 175

Late event:

    09:30 +20

Corrected:

    09:00 +100 → 100
    09:30 +20  → 120
    10:00 +50  → 170
    11:00 +25  → 195

Therefore late events require recomputation from the earliest affected sequence point.

### 2.15 Corrections and reversals

A correction may be represented as:

    original movement
    reversal movement
    corrected movement

The running total should reflect the event ledger according to the business contract.

Do not silently overwrite historical movements if auditability requires an immutable event trail.

### 2.16 Running total versus current balance

A current balance is the latest cumulative state.

A running total contains the state at every sequence point.

Example:

    transaction 1 → balance 100
    transaction 2 → balance 150
    transaction 3 → balance 125

The current balance is 125.

The running-total dataset preserves all three states.

### 2.17 Running totals and precision

Financial or quantity calculations require appropriate numeric types.

Do not calculate monetary cumulative values using binary floating-point arithmetic when exact decimal semantics are required.

Use an appropriate database numeric/decimal type and preserve scale.

### 2.18 Running totals and overflow

Cumulative values can exceed the range of the input type.

Choose a type that safely represents expected cumulative magnitude.

Do not assume that because each individual movement fits a type, the cumulative value must also fit.

### 2.19 Running totals and duplicate events

Duplicate source events can inflate cumulative values.

Before calculating a financial or operational running total, determine whether the source is:

- At-least-once.
- Exactly-once.
- Deduplicated upstream.
- Identified by a unique event key.

Idempotent ingestion and deduplication are separate concerns from the running-total calculation.

### 2.20 Running totals and filtering

Filtering before the window changes the cumulative population.

Example:

    WHERE status = 'SUCCESS'

before the running total means failed movements are excluded from the balance calculation.

That may be correct or incorrect depending on the business definition.

Define whether filtering belongs before or after the cumulative calculation.

### 2.21 Running totals and partition grain

Every independent cumulative stream requires its own partition.

Examples:

    account_id
    warehouse_id
    customer_id + currency
    tenant_id + account_id

Missing a partition component can combine unrelated state.

### 2.22 Currency-specific balances

Never combine currencies into one cumulative monetary balance unless a conversion rule has been applied.

Prefer:

    PARTITION BY account_id, currency_code

for native-currency running balances.

A EUR balance and USD balance are different state streams.

### 2.23 Reconciliation invariant

For a simple cumulative movement stream:

    final_running_total = opening_balance + SUM(all_valid_movements)

This is one of the most valuable independent checks.

If it fails, investigate:

- Missing events.
- Duplicate events.
- Wrong partition.
- Wrong filter.
- Wrong ordering.
- Incorrect opening balance.
- Numeric conversion.

## 3. Implementation

### 3.1 Define the contract

Example:

    Grain:             one row per transaction
    State owner:       account_id + currency_code
    Sequence:          occurred_at + transaction_id
    Starting state:    opening_balance
    Movement:          amount
    Frame:             UNBOUNDED PRECEDING → CURRENT ROW
    Late events:       recompute affected account from earliest event
    Currency:          never mixed
    Precision:         NUMERIC

### 3.2 PostgreSQL schema

    CREATE TABLE account_transactions (
        transaction_id BIGINT PRIMARY KEY,
        account_id BIGINT NOT NULL,
        currency_code TEXT NOT NULL,
        occurred_at TIMESTAMPTZ NOT NULL,
        amount NUMERIC(20,4) NOT NULL
    );

### 3.3 Basic running total

    SELECT
        transaction_id,
        account_id,
        currency_code,
        occurred_at,
        amount,
        SUM(amount) OVER (
            PARTITION BY account_id, currency_code
            ORDER BY occurred_at, transaction_id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS running_movement
    FROM account_transactions;

### 3.4 Running balance with opening balance

    SELECT
        t.transaction_id,
        t.account_id,
        t.currency_code,
        t.occurred_at,
        t.amount,
        o.opening_balance
        + SUM(t.amount) OVER (
            PARTITION BY t.account_id, t.currency_code
            ORDER BY t.occurred_at, t.transaction_id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS running_balance
    FROM account_transactions AS t
    JOIN account_opening_balance AS o
      ON o.account_id = t.account_id
     AND o.currency_code = t.currency_code;

### 3.5 Daily running total

    SELECT
        transaction_id,
        account_id,
        occurred_at::date AS business_date,
        amount,
        SUM(amount) OVER (
            PARTITION BY account_id, occurred_at::date
            ORDER BY occurred_at, transaction_id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS daily_running_total
    FROM account_transactions;

The date boundary creates a new cumulative stream each day.

### 3.6 Monthly running total

    SELECT
        transaction_id,
        account_id,
        date_trunc('month', occurred_at)::date AS month_start,
        amount,
        SUM(amount) OVER (
            PARTITION BY account_id, date_trunc('month', occurred_at)
            ORDER BY occurred_at, transaction_id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS monthly_running_total
    FROM account_transactions;

### 3.7 Running count

    COUNT(*) OVER (
        PARTITION BY account_id
        ORDER BY occurred_at, transaction_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS transaction_count

This can track cumulative event counts.

### 3.8 Running maximum

    MAX(amount) OVER (
        PARTITION BY account_id
        ORDER BY occurred_at, transaction_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_max_amount

### 3.9 Running minimum

    MIN(amount) OVER (
        PARTITION BY account_id
        ORDER BY occurred_at, transaction_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_min_amount

### 3.10 Segment-based reset

Suppose rows contain `is_reset`.

First create a segment number:

    WITH segmented AS (
        SELECT
            e.*,
            SUM(
                CASE WHEN is_reset THEN 1 ELSE 0 END
            ) OVER (
                PARTITION BY account_id
                ORDER BY occurred_at, event_id
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
            ) AS reset_segment
        FROM account_events AS e
    )
    SELECT
        segmented.*,
        SUM(amount) OVER (
            PARTITION BY account_id, reset_segment
            ORDER BY occurred_at, event_id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS segment_total
    FROM segmented;

The reset segmentation and running calculation are deliberately separate stages.

### 3.11 Running total after deduplication

If the source can contain duplicate event IDs, deduplicate before the cumulative calculation.

Conceptual pipeline:

    raw events
        ↓
    deterministic duplicate selection
        ↓
    unique events
        ↓
    running total

Do not calculate the running total first and attempt to remove duplicate effects afterward.

### 3.12 Python implementation

    from collections import defaultdict

    def running_balances(rows, opening_balances):
        partitions = defaultdict(list)

        for row in rows:
            key = (row["account_id"], row["currency_code"])
            partitions[key].append(row)

        result = []

        for key, partition in partitions.items():
            account_id, currency = key
            partition.sort(
                key=lambda row: (row["occurred_at"], row["transaction_id"])
            )

            running = opening_balances.get(key, 0)

            for row in partition:
                running += row["amount"]
                output = dict(row)
                output["running_balance"] = running
                result.append(output)

        return result

The Python version makes partitioning, ordering, and starting state explicit.

### 3.13 Bounded recomputation

When a late event arrives, identify:

    affected partition
    earliest affected sequence position

Then recompute from that boundary rather than rebuilding unrelated accounts.

For example:

    account A
    earliest affected event = 2026-09-10 14:00

Only account A records at or after the affected boundary may need recalculation for a simple cumulative stream.

### 3.14 Reconciliation query

An independent final-total check can use:

    SELECT
        account_id,
        currency_code,
        SUM(amount) AS total_movement
    FROM account_transactions
    GROUP BY account_id, currency_code;

Compare this with the final running value plus the opening-state contract.

## 4. Testing

Running-total tests must validate both values and sequence semantics.

### 4.1 Basic cumulative values

Input:

    +10
    +20
    +5

Expected:

    10
    30
    35

### 4.2 Negative movement

Input:

    +100
    -40
    +10

Expected:

    100
    60
    70

### 4.3 Partition isolation

Create two accounts.

Verify that each account has an independent cumulative stream.

### 4.4 Currency isolation

Create EUR and USD movements for the same account.

Verify that each currency has an independent running balance.

### 4.5 Deterministic timestamp ties

Create two transactions with identical timestamps.

Use transaction ID as the tie-breaker.

Verify that the cumulative sequence is stable across repeated runs.

### 4.6 Frame test

Compare explicit `ROWS` behavior against a peer-tied ordering dataset.

Verify that the chosen frame matches the intended cumulative semantics.

### 4.7 Opening balance

Given:

    opening = 1,000
    movements = +100, -50

Expected:

    1,100
    1,050

### 4.8 Daily reset

Create transactions across two dates.

Verify that the daily running total starts again on the second date.

### 4.9 Segment reset

Create:

    +100
    +50
    RESET
    +20

Verify that the post-reset segment starts according to the documented reset semantics.

### 4.10 Duplicate-event test

Insert the same event twice.

Verify that the deduplication stage prevents double counting when the event contract requires uniqueness.

### 4.11 Late-event test

Initial sequence:

    09:00 +100
    10:00 +50
    11:00 +25

Insert:

    09:30 +20

Expected recalculated values:

    09:00 → 100
    09:30 → 120
    10:00 → 170
    11:00 → 195

### 4.12 Reconciliation test

Verify:

    final_running_balance
    = opening_balance + SUM(valid_movements)

for every independent state partition.

### 4.13 Precision test

Use decimal values that cannot be represented exactly in binary floating point.

Verify exact decimal results.

### 4.14 Idempotence

Run the same snapshot twice.

Expected:

    identical running values
    identical sequence

## 5. Observability

Running totals need visibility into state continuity and event ordering.

### Core metrics

| Metric | Meaning |
|---|---|
| `running_input_rows` | Movement rows processed |
| `running_output_rows` | Rows with cumulative values |
| `running_partition_count` | Independent cumulative streams |
| `running_max_partition_rows` | Largest cumulative partition |
| `running_duplicate_events` | Duplicate event identities |
| `running_late_events` | Events arriving behind the processed frontier |
| `running_reset_count` | Number of reset boundaries |
| `running_negative_movements` | Negative movement count |
| `running_reconciliation_failures` | Partitions failing final-total checks |
| `running_rule_version` | Version of cumulative semantics |

### State reconciliation

For each partition:

    expected_final = opening_balance + total_valid_movement

Compare it with the last running value.

A mismatch is a strong diagnostic signal.

### Sequence diagnostics

Track:

- Duplicate timestamps.
- Duplicate sequence IDs.
- Out-of-order events.
- Late-arriving events.
- Missing sequence values where sequence continuity is expected.

### Partition diagnostics

Monitor unusually large partitions.

A single high-volume account can dominate sort and window execution.

### Reset diagnostics

If cumulative metrics reset periodically, monitor:

- Reset count.
- Missing resets.
- Unexpected resets.
- Multiple resets at the same sequence position.

## 6. Intentional Failure

### Failure 1 — Remove ORDER BY

Replace the running window with a partition-only SUM.

Expected symptom:

- Every row receives the final partition total rather than a cumulative value.

Recovery:

Restore deterministic ordering and an explicit cumulative frame.

### Failure 2 — Remove the tie-breaker

Create same-timestamp movements.

Expected symptom:

- Intermediate cumulative values become unstable or ambiguous.

Recovery:

Add a deterministic sequence key.

### Failure 3 — Mix currencies

Remove `currency_code` from the partition.

Expected symptom:

- EUR and USD movements are combined.

Recovery:

Partition by account and currency or apply an explicit conversion model.

### Failure 4 — Calculate before deduplication

Duplicate one event.

Expected symptom:

- Cumulative state is inflated.

Recovery:

Deduplicate at the correct business-event grain before cumulative calculation.

### Failure 5 — Filter out legitimate movements

Filter failed or reversal records without confirming the balance contract.

Expected symptom:

- Running balance no longer reconciles to the ledger.

Recovery:

Restore the correct movement population.

### Failure 6 — Use ingestion time instead of event time

Introduce a late event.

Expected symptom:

- Historical cumulative values reflect arrival order rather than business order.

Recovery:

Use the correct sequence definition and replay the affected boundary.

### Failure 7 — Ignore opening balance

Calculate only cumulative movements.

Expected symptom:

- Running balance differs from actual account state by the starting balance.

Recovery:

Join or otherwise apply the correct opening state.

### Failure 8 — Use floating-point arithmetic

Use repeated decimal movements.

Expected symptom:

- Small precision differences accumulate.

Recovery:

Use exact numeric/decimal arithmetic for exact-value domains.

## 7. Recovery

Running-total incidents require identifying where cumulative state diverged.

### Recovery sequence

1. Identify affected state partitions.
2. Capture source snapshot and watermark.
3. Verify event uniqueness.
4. Verify partition key.
5. Verify sequence definition.
6. Verify tie-breaker.
7. Verify opening balance.
8. Verify movement filters.
9. Verify reset boundaries.
10. Identify earliest incorrect cumulative value.
11. Recompute from the earliest affected boundary.
12. Compare final state with an independent aggregate.
13. Replace affected outputs atomically.
14. Replay downstream consumers idempotently.
15. Record the root cause.

### Recovering from a late event

1. Identify the event's business-time position.
2. Identify the owning partition.
3. Find the earliest affected row.
4. Recalculate from that row forward.
5. Validate final state.
6. Record the late-event impact.

### Recovering from duplicate events

1. Identify duplicate event IDs.
2. Determine the canonical event according to source rules.
3. Remove or quarantine duplicate effects.
4. Recompute from the earliest affected position.
5. Reconcile the final total.

### Recovering from reconciliation failure

If:

    final_running != opening + SUM(valid_movements)

investigate:

- Missing movement.
- Duplicate movement.
- Wrong opening balance.
- Wrong partition.
- Incorrect filter.
- Incorrect sequence.
- Numeric conversion.
- Reset logic.

Do not patch the final cumulative value manually without fixing the underlying event set or state contract.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL provides window functions, exact numeric types, query planning, and transactional updates useful for cumulative transformations.

Learn to inspect window-query plans and manage large partition sizes.

### 2. dbt

dbt models can implement running metrics and validate cumulative outputs through reconciliation tests.

Use tests to compare final cumulative state with independently aggregated movement totals.

### 3. DuckDB

DuckDB is useful for reproducing cumulative calculations over Parquet and analytical datasets.

It is valuable for testing frames, ordering, and late-event scenarios locally.

## 9. Production Runbook

### Before deployment

- [ ] Define state owner.
- [ ] Define partition key.
- [ ] Define event sequence.
- [ ] Define deterministic tie-breaker.
- [ ] Define opening state.
- [ ] Define movement population.
- [ ] Define currency/state separation.
- [ ] Define reset semantics.
- [ ] Define late-event replay.
- [ ] Define reconciliation equation.
- [ ] Test duplicate events.
- [ ] Test negative movements.
- [ ] Test timestamp ties.

### During execution

- [ ] Record input rows.
- [ ] Record output rows.
- [ ] Record partition count.
- [ ] Record largest partition.
- [ ] Record duplicate events.
- [ ] Record late events.
- [ ] Record reset count.
- [ ] Record reconciliation failures.
- [ ] Record rule version.

### If final totals do not reconcile

1. Check duplicate events.
2. Check missing events.
3. Check opening balances.
4. Check partition keys.
5. Check filters.
6. Check reset boundaries.
7. Check numeric precision.

### If historical balances change

1. Check late events.
2. Check corrected source records.
3. Check ordering changes.
4. Identify earliest affected sequence point.
5. Recompute forward from that point.

### If the query is slow

1. Inspect the execution plan.
2. Measure partition sizes.
3. Reduce unnecessary columns.
4. Restrict processing to affected partitions where valid.
5. Check sorting and memory pressure.

## 10. Common Mistakes

### Mistake 1 — Calling a partition total a running total

A cumulative calculation requires an order.

### Mistake 2 — Omitting a deterministic tie-breaker

Equal timestamps need a stable sequence.

### Mistake 3 — Using the wrong time

Event time and ingestion time represent different business questions.

### Mistake 4 — Mixing currencies

Different monetary units must not share a native balance stream.

### Mistake 5 — Calculating before deduplication

Duplicate events inflate state.

### Mistake 6 — Filtering without defining balance semantics

Removing movements can break reconciliation.

### Mistake 7 — Ignoring opening state

Cumulative movements alone may not equal account balance.

### Mistake 8 — Ignoring late data

One late event can change every later cumulative value in its partition.

### Mistake 9 — Ignoring reset semantics

Some cumulative metrics intentionally restart.

### Mistake 10 — Using floating point for exact-value domains

Precision errors can accumulate.

### Mistake 11 — Ignoring partition skew

One huge state stream can dominate execution.

### Mistake 12 — Manually patching final totals

Fix the event/state contract instead.

## 11. Definition of Done

The running-total transformation is complete when you can:

- [ ] Explain cumulative semantics.
- [ ] Distinguish running totals from partition totals.
- [ ] Define deterministic event ordering.
- [ ] Use explicit `ROWS` frames.
- [ ] Handle timestamp ties.
- [ ] Handle negative movements.
- [ ] Apply opening balances.
- [ ] Implement daily/monthly resets.
- [ ] Implement event-driven reset segments.
- [ ] Separate currencies and independent state streams.
- [ ] Deduplicate events before cumulative calculation.
- [ ] Handle late-arriving events.
- [ ] Handle corrections and reversals.
- [ ] Reconcile final cumulative state independently.
- [ ] Test precision.
- [ ] Monitor partition size and sequence quality.
- [ ] Intentionally break ordering, partitioning, and deduplication.
- [ ] Recover by bounded recomputation.
- [ ] Replay downstream processing idempotently.
- [ ] Explain how PostgreSQL, dbt, and DuckDB support production cumulative workloads.

## 12. What You Learned

A running total is a state reconstruction problem expressed through an ordered window.

The production workflow is:

    DEFINE STATE OWNER
         ↓
    DEFINE PARTITION
         ↓
    DEFINE EVENT ORDER
         ↓
    DEFINE TIE-BREAKER
         ↓
    DEFINE OPENING STATE
         ↓
    DEDUPLICATE EVENTS
         ↓
    APPLY CUMULATIVE WINDOW
         ↓
    HANDLE RESETS / LATE DATA
         ↓
    RECONCILE FINAL STATE
         ↓
    RECOMPUTE AFFECTED BOUNDARIES

> **A running total is only correct when the event population, partition, sequence, starting state, and replay semantics are correct.**

### Next recipe

**T29 — Moving Windows**