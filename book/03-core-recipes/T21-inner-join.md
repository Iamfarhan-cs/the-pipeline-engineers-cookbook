# T21 — INNER JOIN

> **Goal:** Learn how to use an INNER JOIN when the output should contain only records that have a valid match on both sides, while preserving explicit grain, cardinality, and reconciliation guarantees.

## 1. Problem Recognition

An INNER JOIN is appropriate when a downstream dataset should contain only records that satisfy a relationship between two inputs.

Typical production cases include:

- Orders that belong to known customers.
- Transactions associated with valid accounts.
- Events associated with registered entities.
- Fact records that have a required dimension relationship.
- Records that need a matching reference row before downstream processing.
- Comparing two populations where only matching entities are relevant.

The central behavior is simple:

    LEFT ROW + MATCHING RIGHT ROW → OUTPUT
    LEFT ROW + NO MATCH            → NO OUTPUT
    RIGHT ROW + NO MATCH           → NO OUTPUT

The danger is that this row-removal behavior can silently change the population.

Common production symptoms include:

- Row counts unexpectedly decrease.
- New source records disappear.
- A supposedly optional relationship removes valid records.
- Match rates change after a reference-data update.
- Duplicate right-side keys multiply output rows.
- A missing tenant predicate creates false matches.
- NULL keys fail to match and are mistaken for missing source data.
- A join used for validation accidentally becomes a data-loss operation.

### Recognition questions

Before using an INNER JOIN, ask:

1. Is a matching record genuinely required?
2. Which input population is allowed to disappear?
3. What exactly constitutes a match?
4. Is the join key complete?
5. What cardinality is expected?
6. How many records are expected to remain?
7. How will unmatched records be accounted for?
8. Are unmatched records errors, expected exclusions, or another workflow?

An INNER JOIN is a population-selection decision as much as it is a relational operation.

## 2. Concept and Reasoning

### 2.1 INNER JOIN semantics

Given relations `A` and `B`, an INNER JOIN returns rows where the join predicate evaluates to TRUE.

Conceptually:

    A ⋈ B

Only matching combinations appear in the result.

Unlike a LEFT JOIN, unmatched rows from either side are not preserved.

### 2.2 INNER JOIN is not the same as existence testing

Suppose `orders` has one row per order and `payments` has many rows per order.

An INNER JOIN:

    orders JOIN payments

can produce one output row per payment, not one row per order.

If the real requirement is:

    keep orders that have at least one payment

then an existence-oriented semi-join is usually the appropriate mechanism. T25 covers semi-joins in depth.

Do not use INNER JOIN merely because another table is involved.

### 2.3 Population semantics

Suppose:

    orders = 1,000,000 rows
    matching customers = 970,000 orders

An INNER JOIN may produce approximately 970,000 rows if the customer side is unique.

The missing 30,000 rows are not necessarily bad data. They may represent:

- New customers not yet replicated.
- Deleted or archived reference records.
- Invalid source identifiers.
- Late-arriving dimensions.
- Cross-system synchronization delay.
- Legitimately excluded records.

The pipeline must distinguish these possibilities.

### 2.4 Expected cardinality

For a many-to-one relationship:

    orders:     many rows per customer
    customers:  one row per customer
    result:     one row per order that has a customer

For a one-to-many relationship:

    orders:     one row per order
    events:     many rows per order
    result:     potentially many rows per order

The same INNER JOIN syntax can therefore produce completely different grains.

### 2.5 INNER JOIN and duplicate keys

Suppose one order matches two customer rows.

Then:

    1 order × 2 customer rows = 2 output rows

If the contract says one output row per order, this is a correctness failure.

Do not solve it with `DISTINCT` unless duplicate elimination is explicitly part of the business contract.

### 2.6 INNER JOIN and NULL

With ordinary equality:

    a.customer_id = b.customer_id

NULL does not equal NULL.

Therefore a row with a NULL join key will normally not match.

This can be correct if NULL means 'unknown identity'. It can also reveal an upstream completeness problem.

Always measure NULL join keys separately from unmatched non-NULL keys.

### 2.7 Composite-key INNER JOINs

If the identity is:

    tenant_id + customer_id

then both columns belong in the predicate:

    ON a.tenant_id = b.tenant_id
   AND a.customer_id = b.customer_id

Using only `customer_id` can match records across tenants.

This is particularly dangerous because the output can look structurally valid while containing semantically incorrect attributes.

### 2.8 Filter placement

For an INNER JOIN, predicates in `ON` and many predicates in `WHERE` can often be logically equivalent, but production SQL should still make relationship logic and population filtering clear.

Relationship:

    ON o.customer_id = c.customer_id

Reference eligibility:

    ON o.customer_id = c.customer_id
   AND c.status = 'active'

Additional output population filter:

    WHERE o.order_status = 'completed'

Separating these concepts makes intent easier to review and test.

### 2.9 Join filters can change match rates

Consider:

    INNER JOIN customers c
      ON o.customer_id = c.customer_id

versus:

    INNER JOIN customers c
      ON o.customer_id = c.customer_id
     AND c.status = 'active'

The second query requires a stronger definition of 'match'.

Inactive customers now count as unmatched for this transformation.

This should be visible in the transformation contract.

### 2.10 Temporal INNER JOIN

An INNER JOIN can require a historically valid reference record:

    ON t.customer_id = s.customer_id
   AND t.occurred_at >= s.effective_from
   AND t.occurred_at <  s.effective_to

An event with no valid historical version is excluded.

That may be correct, but the pipeline must separately account for these unmatched events.

### 2.11 Match rate is not correctness

A 99.9% match rate can still hide incorrect matches.

Examples:

- Wrong tenant.
- Wrong composite-key predicate.
- Current dimension used for historical data.
- Overly broad normalized key.
- Duplicate right-side records.

Match rate is an operational signal, not a correctness proof.

### 2.12 INNER JOIN as validation

An INNER JOIN is sometimes used to identify valid records:

    SELECT source.*
    FROM source
    INNER JOIN valid_reference
      ON source.code = valid_reference.code;

This can be useful, but it also removes invalid rows.

If rejected records need diagnosis or quarantine, explicitly materialize the unmatched population rather than discarding it.

## 3. Implementation

### 3.1 Define an explicit contract

Example:

    Left input:       orders
    Left grain:       one row per order
    Right input:      customers
    Right grain:      one row per tenant + customer
    Join key:         tenant_id + customer_id
    Cardinality:      many-to-one
    Match required:   yes
    Output grain:     one row per matched order
    Unmatched orders: accounted for separately
    Duplicate keys:   fail

This contract is more important than the SQL syntax.

### 3.2 PostgreSQL schema

    CREATE TABLE customers (
        tenant_id   BIGINT NOT NULL,
        customer_id BIGINT NOT NULL,
        segment     TEXT NOT NULL,
        status      TEXT NOT NULL,
        PRIMARY KEY (tenant_id, customer_id)
    );

    CREATE TABLE orders (
        tenant_id   BIGINT NOT NULL,
        order_id    BIGINT PRIMARY KEY,
        customer_id BIGINT,
        amount      NUMERIC(18,2) NOT NULL
    );

### 3.3 Basic INNER JOIN

    SELECT
        o.tenant_id,
        o.order_id,
        o.customer_id,
        o.amount,
        c.segment
    FROM orders AS o
    INNER JOIN customers AS c
      ON c.tenant_id = o.tenant_id
     AND c.customer_id = o.customer_id;

Only orders with a matching customer survive.

### 3.4 Validate the right-side uniqueness contract

Before joining a staging reference dataset:

    SELECT
        tenant_id,
        customer_id,
        COUNT(*) AS row_count
    FROM customer_stage
    GROUP BY tenant_id, customer_id
    HAVING COUNT(*) > 1;

If this returns rows and the contract requires one customer row per key, stop the transformation.

### 3.5 Account for unmatched rows

Do not lose the population merely because the final output is an INNER JOIN.

Measure unmatched records using a diagnostic LEFT JOIN:

    SELECT
        COUNT(*) AS source_rows,
        COUNT(c.customer_id) AS matched_rows,
        COUNT(*) - COUNT(c.customer_id) AS unmatched_rows
    FROM orders AS o
    LEFT JOIN customers AS c
      ON c.tenant_id = o.tenant_id
     AND c.customer_id = o.customer_id;

For nullable reference identity columns, use a guaranteed non-null right-side marker or an explicit existence expression.

### 3.6 Materialize unmatched records

If unmatched records require remediation:

    CREATE TABLE order_customer_unmatched AS
    SELECT
        o.*
    FROM orders AS o
    LEFT JOIN customers AS c
      ON c.tenant_id = o.tenant_id
     AND c.customer_id = o.customer_id
    WHERE c.customer_id IS NULL;

In production, prefer a durable run-scoped quarantine or exception table rather than recreating an untracked table for every execution.

### 3.7 Apply reference eligibility

Suppose only active customers should be accepted:

    SELECT
        o.order_id,
        o.amount,
        c.segment
    FROM orders AS o
    INNER JOIN customers AS c
      ON c.tenant_id = o.tenant_id
     AND c.customer_id = o.customer_id
     AND c.status = 'active';

Now an inactive customer is semantically an unmatched reference for this transformation.

Track that category separately if operational diagnosis requires it.

### 3.8 Pre-aggregate a one-to-many input

Suppose payments contain many rows per order, but the output needs one row per order:

    WITH payments AS (
        SELECT
            tenant_id,
            order_id,
            SUM(amount) AS paid_amount
        FROM payment_events
        GROUP BY tenant_id, order_id
    )
    SELECT
        o.order_id,
        o.amount,
        p.paid_amount
    FROM orders AS o
    INNER JOIN payments AS p
      ON p.tenant_id = o.tenant_id
     AND p.order_id = o.order_id;

The aggregation changes the right-side grain before the join.

### 3.9 Temporal INNER JOIN

    SELECT
        t.transaction_id,
        t.customer_id,
        s.segment
    FROM transactions AS t
    INNER JOIN customer_segments AS s
      ON s.customer_id = t.customer_id
     AND t.occurred_at >= s.effective_from
     AND t.occurred_at <  s.effective_to;

Validate that effective-dated rows do not overlap for the same customer.

### 3.10 Python implementation

    def inner_join_orders(orders, customers):
        customer_index = {}
        for customer in customers:
            key = (customer["tenant_id"], customer["customer_id"])
            if key in customer_index:
                raise ValueError(f"duplicate customer key: {key!r}")
            customer_index[key] = customer

        result = []
        for order in orders:
            key = (order["tenant_id"], order["customer_id"])
            customer = customer_index.get(key)
            if customer is None:
                continue

            result.append({
                **order,
                "customer_segment": customer["segment"],
            })

        return result

This implementation makes the population-removal behavior explicit.

### 3.11 Preserve exclusion accounting

Instead of returning only the matched output, classify every left record:

    matched
    missing_key
    unmatched_key
    duplicate_reference
    invalid_reference

This is especially useful when an INNER JOIN forms a trust boundary.

### 3.12 Idempotence

An INNER JOIN itself does not create persistent state. Replay safety depends on the surrounding transformation and load contract.

For deterministic replay:

1. Use stable input snapshots or source versions.
2. Use deterministic predicates.
3. Validate duplicate-key assumptions.
4. Record the rule version.
5. Reconcile matched and excluded populations.
6. Write the result through an idempotent load mechanism.

## 4. Testing

INNER JOIN tests must prove both which rows survive and how many rows each survivor can generate.

### 4.1 Basic match test

    def test_inner_join_keeps_only_matching_orders():
        orders = [
            {"tenant_id": 1, "order_id": 101, "customer_id": 7},
            {"tenant_id": 1, "order_id": 102, "customer_id": 8},
        ]
        customers = [
            {"tenant_id": 1, "customer_id": 7, "segment": "gold"},
        ]

        result = inner_join_orders(orders, customers)

        assert [row["order_id"] for row in result] == [101]

### 4.2 Test complete match

Given 100 valid orders and 100 matching unique customers:

    output_rows == 100
    unmatched_rows == 0

This is the baseline preservation case.

### 4.3 Test missing match

Given one order with no matching customer:

    output_rows == source_rows - 1
    unmatched_rows == 1

The excluded record must be observable even though it is absent from the INNER JOIN result.

### 4.4 Test NULL key

Given an order with:

    customer_id = NULL

verify that ordinary equality does not produce a customer match.

Record the NULL-key category separately from ordinary unmatched identifiers.

### 4.5 Test composite-key isolation

Create:

    tenant 1 + customer 7 → Gold
    tenant 2 + customer 7 → Silver

Verify that an order from tenant 1 receives Gold and cannot match Silver.

### 4.6 Test duplicate reference key

Provide two reference records with the same expected unique key.

Expected behavior:

    transformation fails before producing trusted output

Do not silently select one row.

### 4.7 Test one-to-many multiplication

Create one order and two payment records.

If an INNER JOIN is used directly, expect two output rows.

If the output contract is one row per order, the test should require pre-aggregation or another explicit strategy.

### 4.8 Test temporal boundaries

Use:

    A = [2026-01-01, 2026-07-01)
    B = [2026-07-01, 2027-01-01)

An event at `2026-07-01` must match B, not A.

### 4.9 Test eligibility filters

Given a matching but inactive customer, verify whether the contract expects:

- exclusion from the INNER JOIN output,
- a separate invalid-reference disposition, or
- another business workflow.

### 4.10 Test reconciliation

For a many-to-one inner join:

    source_rows = matched_rows + unmatched_rows

and:

    output_rows = matched_rows

provided there is exactly one right-side match per surviving left key.

These equations are useful production assertions.

### 4.11 Test idempotence

Execute the transformation twice against identical inputs and verify identical output keys, values, and accounting.

## 5. Observability

An INNER JOIN should make population loss visible.

### Core metrics

| Metric | Meaning |
|---|---|
| `inner_join_left_rows` | Input population considered for matching |
| `inner_join_right_rows` | Reference or comparison population |
| `inner_join_matched_rows` | Left rows with a valid match |
| `inner_join_unmatched_rows` | Left rows without a valid match |
| `inner_join_match_rate` | Matched / left input |
| `inner_join_output_rows` | Rows produced |
| `inner_join_duplicate_right_keys` | Right keys violating uniqueness |
| `inner_join_null_key_rows` | Left rows with NULL join keys |
| `inner_join_multiplication_factor` | Output relative to matched left keys |
| `inner_join_rule_version` | Version of the matching contract |

### Population accounting

For a many-to-one relationship:

    left_input = matched_left + unmatched_left

and:

    output = matched_left

when each matched left key produces exactly one output row.

Persist these counts by pipeline run.

### Match-rate monitoring

A useful derived metric is:

    match_rate = matched_left / left_input

Track it over time and segment it by:

- source
- tenant
- partition
- business date
- reference version
- pipeline run

A global match rate can hide failures affecting one tenant or source.

### Exclusion reasons

Do not collapse every missing row into `unmatched`.

Useful categories include:

    NULL_JOIN_KEY
    KEY_NOT_FOUND
    REFERENCE_INACTIVE
    REFERENCE_EXPIRED
    DUPLICATE_REFERENCE_KEY
    INVALID_REFERENCE

Reason-level metrics make remediation faster.

### Join multiplication

Track:

    output_rows / distinct_matched_left_keys

For a many-to-one contract, the expected value is `1`.

Unexpected values above `1` indicate a cardinality defect.

## 6. Intentional Failure

### Failure 1 — Delete a reference record

Remove one customer required by an order.

Expected evidence:

- Match count decreases.
- Unmatched count increases.
- Output row count decreases.

Diagnosis:

Compare the unmatched key against the reference snapshot and prior successful run.

### Failure 2 — Duplicate a reference key

Insert a second customer record for the same composite key.

Expected evidence:

- Duplicate-key validation fails.
- If validation is bypassed, output may multiply.

Recovery:

Correct or quarantine the duplicate before replay.

### Failure 3 — Remove tenant from the predicate

Use only `customer_id`.

Expected evidence:

- Cross-tenant attributes.
- Unexpected multiplication.
- Incorrect customer segments.

Recovery:

Restore the complete composite key and replay the affected partition.

### Failure 4 — Add an overly restrictive filter

Require `c.status = 'active'` when inactive records were previously valid.

Expected evidence:

- Match rate drops.
- Output population decreases.

Recovery:

Verify the business contract, then restore or version the eligibility rule.

### Failure 5 — Use current reference data for historical events

Remove the temporal condition.

Expected evidence:

- Historical outputs change after reference updates.
- Backfills disagree with prior results.

Recovery:

Restore event-time validity and replay the affected historical range.

### Failure 6 — Join a one-to-many table directly

Join orders directly to payment events.

Expected evidence:

- Output rows exceed matched orders.
- Monetary measures can be duplicated downstream.

Recovery:

Pre-aggregate payments or change the target grain explicitly.

## 7. Recovery

An INNER JOIN failure can manifest as data loss, incorrect matching, or multiplication.

### Recovery sequence

1. Identify the affected run and rule version.
2. Preserve the input snapshots used by the run.
3. Compare source, matched, unmatched, and output counts.
4. Identify newly unmatched keys.
5. Check NULL-key rates.
6. Check reference freshness.
7. Check duplicate reference keys.
8. Check composite-key predicates.
9. Check eligibility and temporal predicates.
10. Determine the affected partitions or business dates.
11. Correct the source or transformation.
12. Re-run validation.
13. Replay the affected scope through an idempotent load.
14. Reconcile output counts and business measures.
15. Record the root cause.

### Recovering apparent data loss

An INNER JOIN does not necessarily delete source data. It excludes rows from its result.

Therefore recovery usually means:

    identify excluded population
         ↓
    classify exclusion reason
         ↓
    correct reference or rule
         ↓
    rerun the affected scope
         ↓
    reconcile matched + unmatched = source

Do not manually insert missing rows into the output without fixing the transformation contract.

### Recovering multiplication

If a many-to-one INNER JOIN unexpectedly produced multiple rows per key:

1. Identify duplicated right-side keys.
2. Determine whether the duplicates are legitimate versions or invalid duplicates.
3. Apply the correct temporal or survivorship rule.
4. Rebuild the affected output.
5. Reconcile distinct business keys and measures.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL provides INNER JOIN semantics, constraints, indexes, query planning, and `EXPLAIN ANALYZE`.

Learn to inspect join plans and verify that indexes support the actual join predicates.

### 2. dbt

dbt is useful for version-controlled SQL transformations and relationship-oriented data tests.

Use uniqueness and relationship tests to make INNER JOIN assumptions executable rather than relying only on code review.

### 3. DuckDB

DuckDB is useful for reproducing INNER JOIN behavior locally against analytical files and small controlled datasets.

It is valuable for creating minimal failure cases before changing a production transformation.

## 9. Production Runbook

### Before deployment

- [ ] Define the left population.
- [ ] Define the right population.
- [ ] Define the complete join key.
- [ ] Define expected cardinality.
- [ ] Confirm that a match is genuinely required.
- [ ] Define unmatched-row categories.
- [ ] Validate right-side uniqueness where required.
- [ ] Define temporal validity if applicable.
- [ ] Add population reconciliation.
- [ ] Add multiplication detection.

### During execution

- [ ] Record left input rows.
- [ ] Record right input rows.
- [ ] Record NULL-key rows.
- [ ] Record matched rows.
- [ ] Record unmatched rows.
- [ ] Record output rows.
- [ ] Record duplicate-key counts.
- [ ] Monitor match rate.
- [ ] Monitor fan-out.

### If output is lower than expected

1. Check match rate.
2. Inspect newly unmatched keys.
3. Check source freshness.
4. Check reference freshness.
5. Check composite-key completeness.
6. Check NULL-key volume.
7. Check eligibility filters.
8. Check temporal boundaries.

### If output is higher than expected

1. Compare output rows with distinct matched left keys.
2. Check duplicate right-side keys.
3. Check one-to-many relationships.
4. Check for missing predicates.
5. Check whether a pre-aggregation was removed.
6. Check for temporal interval overlap.

### If business measures change unexpectedly

1. Compare aggregate measures before and after the join.
2. Identify duplicated business keys.
3. Inspect fan-out by key.
4. Determine whether the join changed grain.
5. Reconcile at the intended business grain before trusting downstream metrics.

## 10. Common Mistakes

### Mistake 1 — Using INNER JOIN for optional enrichment

If unmatched left records must survive, an INNER JOIN is the wrong preservation semantics.

### Mistake 2 — Treating missing matches as invisible

Every excluded record should be explainable.

### Mistake 3 — Assuming the reference key is unique

Validate uniqueness instead of assuming it.

### Mistake 4 — Joining on an incomplete business key

Composite identity must be represented completely.

### Mistake 5 — Ignoring NULL join keys

Measure them explicitly.

### Mistake 6 — Confusing existence with multiplication

If you only need to know whether a match exists, evaluate a semi-join pattern instead of joining every matching child row.

### Mistake 7 — Using `DISTINCT` to hide duplicate matches

Fix cardinality at the source or relationship boundary.

### Mistake 8 — Ignoring historical validity

Reference data can have different values at different event times.

### Mistake 9 — Monitoring only total output rows

Total counts can remain stable while one source or tenant loses records.

### Mistake 10 — Assuming a high match rate means correct matching

Incorrect keys can produce highly successful but semantically wrong joins.

## 11. Definition of Done

The INNER JOIN transformation is complete when you can:

- [ ] Explain exactly which rows survive an INNER JOIN.
- [ ] Define the input and output grain.
- [ ] Define the complete join predicate.
- [ ] State the expected cardinality.
- [ ] Validate right-side uniqueness when required.
- [ ] Explain NULL join behavior.
- [ ] Handle composite keys correctly.
- [ ] Account for unmatched records.
- [ ] Distinguish missing matches from invalid reference data.
- [ ] Detect one-to-many multiplication.
- [ ] Explain when a semi-join is more appropriate.
- [ ] Implement temporal matching where required.
- [ ] Test boundary, NULL, duplicate, and missing-key cases.
- [ ] Measure match rate and exclusion reasons.
- [ ] Intentionally break the join and diagnose the evidence.
- [ ] Recover and replay affected data safely.
- [ ] Explain how PostgreSQL, dbt, and DuckDB support production INNER JOIN work.

## 12. What You Learned

An INNER JOIN is a controlled intersection between relational populations.

Its defining behavior is not merely that rows are combined. It is that rows without a valid match are excluded from the result.

The production workflow is:

    DEFINE REQUIRED MATCH
         ↓
    DEFINE GRAIN
         ↓
    DEFINE COMPLETE KEY
         ↓
    DEFINE CARDINALITY
         ↓
    VALIDATE REFERENCE DATA
         ↓
    APPLY INNER JOIN
         ↓
    ACCOUNT FOR UNMATCHED ROWS
         ↓
    CHECK MULTIPLICATION
         ↓
    RECONCILE OUTPUT
         ↓
    REPLAY SAFELY IF REQUIRED

> **An INNER JOIN is safe only when every excluded row and every surviving row can be explained by the join contract.**

### Next recipe

**T22 — LEFT JOIN**