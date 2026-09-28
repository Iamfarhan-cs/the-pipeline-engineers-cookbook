# T20 — JOIN Transformations

> **Goal:** Learn how to combine relational datasets safely without accidental row multiplication, incorrect matches, hidden NULL behavior, or loss of auditability.

## 1. Problem Recognition

A JOIN combines rows from two or more relations according to a matching predicate. In production ETL, the difficult part is rarely writing `JOIN`; it is proving that the join produces the intended grain and cardinality.

Typical symptoms of a broken join include:

- Output row counts suddenly increase.
- Measures become inflated after joining reference or transaction data.
- Expected records disappear.
- Unmatched records are silently lost.
- A supposedly unique dimension matches multiple rows.
- Historical records receive current reference values.
- A `LEFT JOIN` behaves like an inner join.
- Different systems use incompatible key types or representations.
- A many-to-many relationship creates a join explosion.
- A pipeline becomes dramatically slower after adding a join.

### Production recognition checklist

Before implementing a join, answer:

1. What is the grain of the left input?
2. What is the grain of the right input?
3. What is the intended grain of the output?
4. Which columns define identity on each side?
5. Is each join key unique?
6. What cardinality is expected?
7. What should happen when there is no match?
8. What should happen when there are multiple matches?
9. Does the relationship depend on time?
10. How will row counts and measures be reconciled?

Never treat a JOIN as merely a SQL syntax operation. It is a cardinality-changing transformation.

## 2. Concept and Reasoning

### 2.1 The relational model

Suppose `orders` contains one row per order and `customers` contains one row per customer.

An order may contain `customer_id`. A customer may contain profile attributes. Joining them can enrich an order:

    orders:    one row per order
    customers: one row per customer
    output:    one row per order

This is a many-to-one relationship from orders to customers.

If the customer table accidentally contains two rows for the same `customer_id`, the relationship is no longer many-to-one. One order can become two output rows.

That is not a harmless duplicate. It changes the output grain.

### 2.2 JOIN versus other transformations

| Operation | Primary purpose | Typical grain effect |
|---|---|---|
| JOIN | Combine related relations | Can preserve, multiply, or remove rows |
| Enrichment | Add contextual attributes | Usually preserves left-side grain |
| Aggregation | Summarize records | Reduces grain |
| Merge | Consolidate competing records | Usually reduces multiple representations into one logical record |
| Filtering | Select a population | Reduces or preserves row count |
| Deduplication | Remove duplicate representations | Reduces row count |

JOIN is the general relational mechanism. T14 covered enrichment as a controlled application of joining. T21 onward will examine specific join forms in greater depth.

### 2.3 Join cardinality

The most important question is how many rows can match each input row.

| Relationship | Left match count | Right match count | Main risk |
|---|---:|---:|---|
| One-to-one | 1 | 1 | Missing or duplicate keys |
| Many-to-one | many | 1 | Duplicate dimension keys |
| One-to-many | 1 | many | Intentional row multiplication |
| Many-to-many | many | many | Join explosion |

Cardinality must be an explicit contract.

### 2.4 Row multiplication

If one left row matches `m` right rows, that left row contributes `m` output rows for an inner or left join.

If there are several left rows, the total output can be much larger than either input.

For a many-to-many key `k`, if the left side contains `L_k` rows and the right side contains `R_k` rows, the matched output for that key contains:

    L_k × R_k

rows.

For example:

    left key A:  3 rows
    right key A: 4 rows
    output:    12 rows

This is the fundamental mechanism behind join explosions.

### 2.5 Join predicates

A join predicate defines which rows are related.

Simple equality:

    left.customer_id = right.customer_id

Composite key:

    left.tenant_id = right.tenant_id
    AND left.customer_id = right.customer_id

Temporal relationship:

    left.customer_id = right.customer_id
    AND left.occurred_at >= right.effective_from
    AND left.occurred_at < right.effective_to

A predicate that omits part of the business key can create false matches.

### 2.6 Composite keys

If identity is `(tenant_id, customer_id)`, joining only on `customer_id` is unsafe.

Incorrect:

    orders.customer_id = customers.customer_id

Correct:

    orders.tenant_id = customers.tenant_id
    AND orders.customer_id = customers.customer_id

This matters especially in multi-tenant systems where identifiers may only be unique within a tenant.

### 2.7 NULL join keys

SQL equality does not treat two NULLs as equal.

Conceptually:

    NULL = NULL

produces UNKNOWN rather than TRUE.

Therefore ordinary equality joins do not match NULL keys.

If NULL represents 'unknown customer', that is usually different from a customer whose identifier is also unknown. Do not invent equality semantics merely to increase match rates.

PostgreSQL also supports `IS NOT DISTINCT FROM` when null-safe equality is explicitly part of the data contract. It should not be used casually.

### 2.8 Join type is part of the business contract

At a conceptual level:

- **INNER JOIN** retains rows with matches on both sides.
- **LEFT JOIN** preserves the left population and attaches matches when available.
- **FULL OUTER JOIN** preserves unmatched rows from both sides.
- **CROSS JOIN** intentionally produces the Cartesian product.
- **SEMI JOIN** asks whether a match exists without multiplying by matching rows.
- **ANTI JOIN** asks whether a match does not exist.

T21–T25 cover these forms individually. T20 focuses on the common mechanism and how to reason about it safely.

### 2.9 `ON` versus `WHERE`

Predicate placement can change the meaning of a left join.

Consider:

    SELECT o.order_id, c.segment
    FROM orders o
    LEFT JOIN customers c
      ON o.customer_id = c.customer_id
     AND c.status = 'active';

Here inactive customers do not match, but the order remains because the join is still left-preserving.

Compare:

    SELECT o.order_id, c.segment
    FROM orders o
    LEFT JOIN customers c
      ON o.customer_id = c.customer_id
    WHERE c.status = 'active';

The `WHERE` condition removes rows whose joined customer is NULL, so the result behaves like an inner-filtered population.

This is a common production bug.

### 2.10 Cross joins

A cross join has no matching predicate:

    left_rows × right_rows

Even moderate inputs can create enormous outputs.

Never use a cross join accidentally because a join predicate was omitted.

### 2.11 Temporal joins

Some relationships depend on the state of the right-side record at the time of the left-side event.

Example:

    transaction occurred_at = 2026-09-10
    customer segment effective_from = 2026-01-01
    customer segment effective_to   = 2026-10-01

The transaction should use the version effective at the transaction time, not whichever row happens to be current when the pipeline runs.

Use half-open intervals:

    effective_from <= event_time < effective_to

and enforce that effective-dated records do not overlap for the same business key.

### 2.12 Join key quality

Before joining, validate:

- Data type compatibility.
- Normalization compatibility.
- Case and whitespace rules.
- Unit and encoding compatibility.
- Composite-key completeness.
- Nullability.
- Uniqueness where required.
- Referential coverage.
- Historical validity.

A join can be syntactically correct while being semantically wrong.

### 2.13 Pre-aggregation before joining

If the right-side dataset contains multiple rows per business key but only a summary is needed, aggregate it before joining.

Instead of:

    orders JOIN payment_events

when the output only needs total paid amount, use:

    orders JOIN
        (SELECT order_id, SUM(amount) AS paid_amount
         FROM payment_events
         GROUP BY order_id) payments

This changes the right side to one row per order and prevents transaction-level multiplication.

### 2.14 Many-to-many joins require an explicit reason

A many-to-many join is not automatically wrong. It is wrong when the output grain does not intentionally represent every pair.

Examples where pair generation may be intentional include:

- Product compatibility matrices.
- Entity-to-entity relationship analysis.
- Calendar expansion.
- Matching candidate pairs.

Otherwise, many-to-many behavior is a major warning signal.

## 3. Implementation

### 3.1 Define the contract first

Example contract:

    Input A: orders — one row per order
    Input B: customers — one row per customer
    Key: tenant_id + customer_id
    Expected relationship: many-to-one
    Output: one row per order
    Missing customer: preserve order with NULL customer attributes
    Duplicate customer key: fail the transformation
    Null customer key: do not match

The implementation should enforce this contract rather than merely assume it.

### 3.2 PostgreSQL implementation

Reference tables:

    CREATE TABLE customers (
        tenant_id     BIGINT NOT NULL,
        customer_id   BIGINT NOT NULL,
        segment       TEXT NOT NULL,
        status        TEXT NOT NULL,
        PRIMARY KEY (tenant_id, customer_id)
    );

Orders:

    CREATE TABLE orders (
        tenant_id     BIGINT NOT NULL,
        order_id      BIGINT PRIMARY KEY,
        customer_id   BIGINT,
        amount        NUMERIC(18,2) NOT NULL,
        occurred_at   TIMESTAMPTZ NOT NULL
    );

Controlled enrichment join:

    SELECT
        o.tenant_id,
        o.order_id,
        o.customer_id,
        o.amount,
        o.occurred_at,
        c.segment,
        c.status AS customer_status
    FROM orders AS o
    LEFT JOIN customers AS c
      ON c.tenant_id = o.tenant_id
     AND c.customer_id = o.customer_id;

Because `(tenant_id, customer_id)` is the primary key, the right side has at most one matching row for each order.

### 3.3 Validate the right-side key before joining

Even when a schema is expected to enforce uniqueness, validation is useful at transformation boundaries or when the source is a staging table.

    SELECT
        tenant_id,
        customer_id,
        COUNT(*) AS row_count
    FROM customer_stage
    GROUP BY tenant_id, customer_id
    HAVING COUNT(*) > 1;

If this query returns rows and the contract is many-to-one, stop before producing the target dataset.

### 3.4 Measure match coverage

    SELECT
        COUNT(*) AS total_orders,
        COUNT(c.customer_id) AS matched_orders,
        COUNT(*) - COUNT(c.customer_id) AS unmatched_orders
    FROM orders AS o
    LEFT JOIN customers AS c
      ON c.tenant_id = o.tenant_id
     AND c.customer_id = o.customer_id;

Be careful when using a nullable right-side column as the match indicator. Prefer a guaranteed non-null right-side identity column when possible.

### 3.5 Detect row multiplication

Compare the left population to the joined population:

    WITH joined AS (
        SELECT o.order_id
        FROM orders AS o
        JOIN customer_stage AS c
          ON c.tenant_id = o.tenant_id
         AND c.customer_id = o.customer_id
    )
    SELECT
        COUNT(*) AS joined_rows,
        COUNT(DISTINCT order_id) AS distinct_orders
    FROM joined;

If `joined_rows` is greater than `distinct_orders` while the contract says one output row per order, the right side is multiplying orders.

### 3.6 Use pre-aggregation when the target needs one row per key

    WITH payments_by_order AS (
        SELECT
            tenant_id,
            order_id,
            SUM(amount) AS paid_amount
        FROM payment_events
        GROUP BY tenant_id, order_id
    )
    SELECT
        o.tenant_id,
        o.order_id,
        o.amount,
        p.paid_amount
    FROM orders AS o
    LEFT JOIN payments_by_order AS p
      ON p.tenant_id = o.tenant_id
     AND p.order_id = o.order_id;

This explicitly changes the payment input grain before joining.

### 3.7 Guard against accidental cross joins

Do not generate SQL dynamically without validating that a join predicate exists.

An explicit intentional cross join should look like:

    SELECT
        c.calendar_date,
        p.product_id
    FROM calendar AS c
    CROSS JOIN products AS p;

The absence of a predicate should be visible and deliberate.

### 3.8 Python implementation

Python can express the same contract, but production behavior still depends on explicit cardinality validation.

    from collections import defaultdict

    def index_unique(rows, key_fn):
        index = {}
        for row in rows:
            key = key_fn(row)
            if key in index:
                raise ValueError(f"duplicate join key: {key!r}")
            index[key] = row
        return index

    def enrich_orders(orders, customers):
        customer_index = index_unique(
            customers,
            lambda r: (r["tenant_id"], r["customer_id"]),
        )

        output = []
        for order in orders:
            key = (order["tenant_id"], order["customer_id"])
            customer = customer_index.get(key)
            result = dict(order)
            if customer is not None:
                result["customer_segment"] = customer["segment"]
                result["customer_status"] = customer["status"]
            else:
                result["customer_segment"] = None
                result["customer_status"] = None
            output.append(result)

        return output

This implementation deliberately fails when a supposed unique right-side key is duplicated.

### 3.9 Deterministic join behavior

A production join should produce the same output for the same input snapshot and rule version.

Avoid selecting an arbitrary duplicate with patterns such as:

    SELECT DISTINCT ON (customer_id) ...

unless the selection rule is explicit and deterministic.

If multiple versions exist, define a survivorship or temporal rule first. Do not rely on database scan order.

### 3.10 Temporal join implementation

Example:

    SELECT
        t.transaction_id,
        t.customer_id,
        t.occurred_at,
        s.segment
    FROM transactions AS t
    LEFT JOIN customer_segments AS s
      ON s.customer_id = t.customer_id
     AND t.occurred_at >= s.effective_from
     AND t.occurred_at <  s.effective_to;

The temporal table must be validated so that overlapping intervals cannot produce multiple valid matches for the same business key and event time.

### 3.11 Materialize join accounting

For a production transformation, persist or emit at least:

    run_id
    join_name
    left_input_rows
    right_input_rows
    output_rows
    matched_left_rows
    unmatched_left_rows
    duplicate_right_keys
    multiplication_events
    rule_version
    processed_at

This turns a hidden SQL behavior into operational evidence.

### 3.12 Idempotence

A deterministic join is naturally replayable when its input snapshots and rules are stable.

To make replay safe:

1. Identify the input snapshot or source versions.
2. Record the transformation rule version.
3. Avoid nondeterministic duplicate selection.
4. Persist run and reconciliation metadata.
5. Replace or upsert the target according to its load contract.

Joining the same inputs twice should not create additional logical records merely because the transformation was rerun.

## 4. Testing

JOIN tests must validate both values and cardinality.

### 4.1 Unit test: many-to-one join

    def test_many_to_one_join_preserves_order_grain():
        orders = [
            {"tenant_id": 1, "order_id": 101, "customer_id": 7},
            {"tenant_id": 1, "order_id": 102, "customer_id": 7},
        ]
        customers = [
            {"tenant_id": 1, "customer_id": 7, "segment": "gold", "status": "active"},
        ]

        result = enrich_orders(orders, customers)

        assert len(result) == 2
        assert {row["order_id"] for row in result} == {101, 102}
        assert all(row["customer_segment"] == "gold" for row in result)

### 4.2 Duplicate right-side key must fail

    def test_duplicate_customer_key_fails():
        orders = [
            {"tenant_id": 1, "order_id": 101, "customer_id": 7},
        ]
        customers = [
            {"tenant_id": 1, "customer_id": 7, "segment": "gold", "status": "active"},
            {"tenant_id": 1, "customer_id": 7, "segment": "silver", "status": "active"},
        ]

        try:
            enrich_orders(orders, customers)
            assert False, "expected duplicate-key failure"
        except ValueError as exc:
            assert "duplicate join key" in str(exc)

### 4.3 Test missing matches

Verify that the selected join type produces the intended result when the right-side key is absent.

    order 101 -> customer 7 exists
    order 102 -> customer 8 does not exist

For a left-preserving contract, both orders must remain in the output.

### 4.4 Test composite keys

Include two tenants that reuse the same customer identifier.

    tenant 1 + customer 7 -> Gold
    tenant 2 + customer 7 -> Silver

A join on `customer_id` alone should be detected as incorrect.

### 4.5 Test NULL keys

Verify that rows with NULL join keys follow the documented policy.

Do not assume NULL matches NULL.

### 4.6 Test predicate placement

Create a test where the right-side record is inactive and verify the intended difference between:

    LEFT JOIN ... ON right.status = 'active'

and:

    LEFT JOIN ... WHERE right.status = 'active'

This catches accidental conversion of a left-preserving population into an inner-filtered population.

### 4.7 Test row multiplication

Provide one left row and two right rows with the same key.

Expected result depends on the contract:

- Fail before joining.
- Intentionally produce two rows.
- Pre-aggregate the right side to one row.

The test must make the choice explicit.

### 4.8 Test temporal boundaries

Given:

    version A: [2026-01-01, 2026-07-01)
    version B: [2026-07-01, 2027-01-01)

Verify that an event exactly at `2026-07-01` selects version B.

### 4.9 Test idempotence

Run the same join twice against the same input snapshot and verify identical logical output and accounting.

### 4.10 Test SQL cardinality

Useful database assertions include:

    SELECT COUNT(*) FROM joined_result;

    SELECT COUNT(DISTINCT order_id) FROM joined_result;

    SELECT order_id, COUNT(*)
    FROM joined_result
    GROUP BY order_id
    HAVING COUNT(*) > 1;

These tests expose accidental grain changes.

## 5. Observability

A join should expose enough evidence to distinguish a business change from a join defect.

### Core metrics

| Metric | Meaning |
|---|---|
| `join_left_rows` | Number of input rows on the left |
| `join_right_rows` | Number of input rows on the right |
| `join_output_rows` | Rows produced by the join |
| `join_matched_rows` | Left rows with at least one match |
| `join_unmatched_rows` | Left rows without a match |
| `join_match_rate` | Matched-left / left-input population |
| `join_duplicate_key_count` | Right-side keys violating uniqueness assumptions |
| `join_multiplication_factor` | Output rows relative to the expected grain |
| `join_null_key_count` | Rows unable to match because the key is NULL |
| `join_rule_version` | Version of the join contract |

### Multiplication factor

For a contract that should preserve one row per left entity:

    multiplication_factor = output_rows / left_rows

A value materially above `1` requires investigation.

Do not define a universal alert threshold without understanding the intended cardinality. A one-to-many join may legitimately have a factor above one.

### Match-rate monitoring

A sudden drop in match rate may indicate:

- New source identifiers.
- Source lag.
- Broken normalization.
- Incorrect tenant scoping.
- Reference-data freshness issues.
- Schema or contract changes.
- A changed business population.

Match rate is a signal, not proof of correctness.

### High-cardinality key monitoring

Monitor the distribution of matches per key.

Useful statistics include:

- maximum matches per left key
- p95/p99 matches per left key
- number of keys with more than one match
- number of keys with unexpectedly high fan-out

This often identifies join explosions earlier than total row count alone.

## 6. Intentional Failure

The purpose of this section is to break the transformation deliberately and learn what the evidence looks like.

### Failure 1 — Remove part of a composite key

Change:

    ON c.tenant_id = o.tenant_id
   AND c.customer_id = o.customer_id

to:

    ON c.customer_id = o.customer_id

Expected symptom:

- Cross-tenant matches.
- Output multiplication.
- Incorrect attributes.

Diagnosis:

Compare key cardinality by tenant and inspect mismatched source identifiers.

Recovery:

Restore the full business key and replay the affected transformation.

### Failure 2 — Duplicate a right-side key

Insert two customer records for the same `(tenant_id, customer_id)`.

Expected symptom:

- More output rows than expected.
- Duplicate logical orders.

Recovery:

Quarantine or correct the conflicting reference records, then replay.

### Failure 3 — Move a right-side filter from `ON` to `WHERE`

Expected symptom:

- Unmatched left records disappear.
- Match rate may appear artificially high.

Recovery:

Restore predicate placement according to the population contract and rerun reconciliation.

### Failure 4 — Introduce a cross join

Remove the join predicate.

Expected symptom:

- Output count approaches left rows × right rows.
- Database resource consumption increases sharply.

Recovery:

Stop the transformation, terminate the runaway execution if necessary, restore the predicate, and verify output counts before rerunning.

### Failure 5 — Use the current reference row for historical data

Replace a temporal predicate with a simple current-state join.

Expected symptom:

- Historical events receive current attributes.
- Backfills disagree with previously produced results.

Recovery:

Restore event-time semantics and replay the affected historical interval.

## 7. Recovery

JOIN failures require both code correction and data-impact analysis.

### Recovery sequence

1. Stop or isolate the affected transformation if downstream corruption is still possible.
2. Identify the pipeline run and join rule version.
3. Capture left and right input versions or snapshots.
4. Compare expected and observed cardinality.
5. Identify duplicate or unmatched keys.
6. Inspect predicate changes and filter placement.
7. Determine the affected time or partition range.
8. Correct the join contract or source data.
9. Re-run validation before producing the target.
10. Replay the affected scope idempotently.
11. Reconcile row counts and important measures.
12. Record the incident and root cause.

### Recovery principle

Never repair a join explosion with `DISTINCT` unless duplicate elimination is itself part of the business contract.

`DISTINCT` can hide incorrect cardinality while silently removing legitimate records.

### Data repair example

If a duplicate reference key caused two outputs per order:

    1. Identify affected keys.
    2. Determine the authoritative reference record.
    3. Correct or quarantine the invalid duplicate.
    4. Rebuild the affected join result.
    5. Verify one output row per order.
    6. Reconcile monetary and count measures.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL provides relational JOIN semantics, constraints, indexes, query plans, and execution statistics. Learn to inspect `EXPLAIN` and `EXPLAIN ANALYZE` when a join is unexpectedly expensive.

Use it to understand the actual relational mechanism rather than treating joins as an ORM abstraction.

### 2. dbt

dbt is useful for expressing transformation joins as version-controlled SQL models and validating assumptions with data tests.

Useful tests around joins include uniqueness, relationships, accepted values, and custom cardinality assertions.

### 3. DuckDB

DuckDB is useful for local analytical experimentation and reproducing relational transformations against files such as Parquet and CSV.

It is particularly useful for building a small reproducible dataset to investigate join behavior before changing a production model.

## 9. Production Runbook

### Before deployment

- [ ] Define left input grain.
- [ ] Define right input grain.
- [ ] Define output grain.
- [ ] Document the join key.
- [ ] Document expected cardinality.
- [ ] Validate key types and normalization.
- [ ] Validate required key uniqueness.
- [ ] Define unmatched-row behavior.
- [ ] Define duplicate-match behavior.
- [ ] Define temporal semantics if applicable.
- [ ] Add cardinality tests.
- [ ] Add reconciliation metrics.

### During execution

- [ ] Record input row counts.
- [ ] Record key-null counts.
- [ ] Record duplicate-key counts.
- [ ] Measure match rate.
- [ ] Measure output row count.
- [ ] Monitor fan-out and multiplication.
- [ ] Monitor query duration and resource usage.

### If output count increases unexpectedly

1. Compare output rows with distinct left business keys.
2. Check right-side key uniqueness.
3. Check for incomplete composite predicates.
4. Check for many-to-many relationships.
5. Check whether an upstream aggregation disappeared.
6. Check for duplicate or overlapping temporal records.
7. Inspect the query plan for accidental cross joins.

### If match rate falls

1. Compare unmatched keys with the previous successful run.
2. Check source freshness.
3. Check type and normalization compatibility.
4. Check tenant or partition predicates.
5. Check reference-data coverage.
6. Determine whether the business population genuinely changed.
7. Replay only after the cause is understood.

### If the join is slow

1. Inspect `EXPLAIN ANALYZE`.
2. Check indexes on join keys.
3. Check data volume and skew.
4. Check whether the join can be filtered earlier.
5. Check whether the right side can be projected or pre-aggregated.
6. Check for stale statistics.
7. Reassess whether the join is unintentionally many-to-many.

## 10. Common Mistakes

### Mistake 1 — Joining without defining grain

Every join should have an explicit input and output grain.

### Mistake 2 — Assuming a key is unique

Names such as `customer_id` do not prove uniqueness. Validate it.

### Mistake 3 — Joining on only part of a composite key

Include every field required by the business identity.

### Mistake 4 — Using `DISTINCT` to hide multiplication

Fix the relationship instead of masking the symptom.

### Mistake 5 — Ignoring NULL semantics

NULL is not an ordinary value in SQL equality.

### Mistake 6 — Treating match rate as correctness

100% matching can still produce wrong matches if the key is too broad.

### Mistake 7 — Filtering a left join incorrectly

Know whether a condition belongs in `ON` or `WHERE`.

### Mistake 8 — Ignoring historical validity

Current reference data is not necessarily valid for historical events.

### Mistake 9 — Allowing arbitrary duplicate selection

If duplicates exist, define a deterministic rule or fail.

### Mistake 10 — Forgetting resource impact

Join cardinality affects CPU, memory, network traffic, disk usage, and downstream storage.

### Mistake 11 — Assuming optimizer behavior changes semantics

Database optimizers may change execution strategy, but a logically incorrect predicate remains incorrect.

### Mistake 12 — Treating many-to-many as a performance problem only

The first question is whether the resulting grain is logically valid.

## 11. Definition of Done

The JOIN transformation is complete when you can:

- [ ] State the grain of every input.
- [ ] State the grain of the output.
- [ ] Identify the complete join key.
- [ ] Explain the expected cardinality.
- [ ] Implement the join without relying on accidental uniqueness.
- [ ] Explain NULL join behavior.
- [ ] Explain composite-key behavior.
- [ ] Distinguish one-to-one, many-to-one, one-to-many, and many-to-many joins.
- [ ] Detect join multiplication.
- [ ] Detect an accidental cross join.
- [ ] Explain `ON` versus `WHERE` behavior.
- [ ] Implement a controlled temporal join.
- [ ] Pre-aggregate a relation when the target grain requires it.
- [ ] Test missing, duplicate, NULL, and boundary cases.
- [ ] Observe match rate and cardinality.
- [ ] Intentionally break the join and diagnose it.
- [ ] Recover and replay the affected data safely.
- [ ] Explain how PostgreSQL, dbt, and DuckDB support production join work.

## 12. What You Learned

A JOIN is not simply a way to retrieve columns from another table.

It is a relational transformation that can change the population, grain, cardinality, historical meaning, and resource requirements of a pipeline.

The production mindset is:

    DEFINE GRAIN
         ↓
    DEFINE JOIN KEY
         ↓
    DEFINE CARDINALITY
         ↓
    VALIDATE KEY QUALITY
         ↓
    APPLY JOIN
         ↓
    MEASURE MATCHING
         ↓
    CHECK MULTIPLICATION
         ↓
    RECONCILE OUTPUT
         ↓
    REPLAY SAFELY IF NEEDED

The most important rule is:

> **Never trust a JOIN until you can explain exactly how many output rows each input row is allowed to produce.**

### Next recipe

**T21 — INNER JOIN**