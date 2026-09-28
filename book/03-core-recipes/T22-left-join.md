# T22 — LEFT JOIN

> **Goal:** Learn how to preserve the complete left-side population while optionally attaching matching right-side records, with explicit unmatched semantics, cardinality controls, and reconciliation.

## 1. Problem Recognition

A LEFT JOIN is used when the left-side population must remain present even when the right-side relationship is missing.

Typical production cases include:

- Enriching transactions with optional reference data.
- Keeping every order while attaching customer information when available.
- Preserving source events when a dimension has not arrived yet.
- Comparing a primary population against a reference population.
- Building completeness reports that must include unmatched entities.
- Detecting missing relationships without dropping source records.

The defining behavior is:

    LEFT ROW + MATCHING RIGHT ROW → OUTPUT
    LEFT ROW + NO MATCH            → LEFT ROW + NULLS

The left input is therefore population-preserving, but only if the rest of the query does not accidentally remove NULL-extended rows.

Common production failures include:

- A LEFT JOIN unexpectedly drops rows.
- A `WHERE` condition on the right table turns the result into an inner-filtered population.
- Duplicate right keys multiply left rows.
- NULL values are mistaken for actual right-side records.
- Missing reference data is silently accepted.
- A left join is used when existence-only logic would be safer.
- Historical reference data is joined using the current version.

### Recognition questions

Before using a LEFT JOIN, ask:

1. Must every left record survive?
2. What should unmatched right-side attributes contain?
3. Is no match expected, exceptional, or invalid?
4. How many right-side matches are allowed?
5. What is the intended output grain?
6. Which conditions belong in the `ON` clause?
7. Which conditions belong in the `WHERE` clause?
8. How will unmatched rows be measured?

## 2. Concept and Reasoning

### 2.1 LEFT JOIN semantics

Given a left relation `A` and right relation `B`, a LEFT JOIN returns:

- Every row from `A`.
- Matching rows from `B` when the predicate is TRUE.
- NULL-extended right-side columns when no right-side row matches.

Conceptually:

    A LEFT JOIN B

preserves the left population.

### 2.2 LEFT JOIN versus INNER JOIN

| Behavior | INNER JOIN | LEFT JOIN |
|---|---|---|
| Unmatched left row | Removed | Preserved |
| Unmatched right row | Removed | Removed |
| Right attributes when unmatched | Not applicable | NULL |
| Typical use | Required relationship | Optional relationship / population preservation |

Changing INNER JOIN to LEFT JOIN is therefore a semantic change, not merely a way to get more rows.

### 2.3 LEFT JOIN and output grain

A LEFT JOIN preserves the left population only when the right side has at most the intended number of matches.

If the left side contains:

    one order

and the right side contains:

    three payment events

then a direct LEFT JOIN produces three rows for that order.

The left row is preserved, but its grain has changed.

Population preservation does not mean grain preservation.

### 2.4 One-to-one and many-to-one LEFT JOINs

For a typical enrichment:

    orders:     one row per order
    customers:  one row per customer
    output:     one row per order

the right side must be unique at the customer key.

If the right side is not unique, the join may multiply orders.

### 2.5 NULL-extended rows

When no right-side match exists, PostgreSQL returns NULL for right-side columns.

Example:

    order_id | customer_id | customer_segment
    ---------+-------------+-----------------
    101      | 7           | gold
    102      | 8           | NULL

The NULL does not necessarily mean the source customer attribute itself was NULL.

It may mean there was no matching customer row at all.

This distinction is critical for data quality analysis.

### 2.6 Missing row versus NULL attribute

Suppose a customer exists but `phone_number` is NULL.

That differs from an order with no customer match.

Use a guaranteed non-null right-side identity column to distinguish:

    right row exists + attribute is NULL

from:

    right row does not exist

Do not use an arbitrary nullable attribute as the match indicator.

### 2.7 `ON` versus `WHERE` — the critical LEFT JOIN rule

Consider:

    SELECT o.order_id, c.segment
    FROM orders AS o
    LEFT JOIN customers AS c
      ON c.customer_id = o.customer_id
     AND c.status = 'active';

Inactive customers do not match, but the order remains.

Now consider:

    SELECT o.order_id, c.segment
    FROM orders AS o
    LEFT JOIN customers AS c
      ON c.customer_id = o.customer_id
    WHERE c.status = 'active';

For an unmatched customer, `c.status` is NULL. The WHERE predicate is not TRUE, so the left row is removed.

The second query therefore behaves like an inner-filtered population for that condition.

This is one of the most important LEFT JOIN failure modes to recognize.

### 2.8 Filtering the left side

A predicate on the left table can safely belong in `WHERE` when the intention is to reduce the left population:

    WHERE o.order_status = 'completed'

This is different from filtering the optional right-side relationship.

Keep population filtering and relationship eligibility conceptually separate.

### 2.9 Filtering the right side

If only active customers should be eligible for matching, put that condition into the relationship when left preservation is required:

    LEFT JOIN customers AS c
      ON c.customer_id = o.customer_id
     AND c.status = 'active'

This means:

    every order survives
    active customer → match
    inactive customer → unmatched

### 2.10 LEFT JOIN for missing-data detection

A LEFT JOIN can identify missing relationships:

    SELECT o.*
    FROM orders AS o
    LEFT JOIN customers AS c
      ON c.customer_id = o.customer_id
    WHERE c.customer_id IS NULL;

This pattern is useful for detecting referential gaps.

However, make the distinction explicit between:

- source key is NULL
- source key is non-NULL but not found
- right record exists but is ineligible

### 2.11 LEFT JOIN versus SEMI JOIN

If the requirement is simply:

    keep orders that have at least one payment

an INNER JOIN may multiply orders when there are multiple payments.

A semi-join pattern can preserve the left grain while testing existence.

T25 covers semi-joins in detail.

Use LEFT JOIN when you need right-side attributes or explicit missing-reference information.

### 2.12 LEFT JOIN versus ANTI JOIN

If the requirement is:

    find orders with no customer

a LEFT JOIN with a NULL test can express the logic, but an anti-join pattern may be clearer depending on the database and query design.

T24 covers anti-joins.

### 2.13 Temporal LEFT JOIN

Historical enrichment may require:

    LEFT JOIN customer_segments AS s
      ON s.customer_id = t.customer_id
     AND t.occurred_at >= s.effective_from
     AND t.occurred_at <  s.effective_to

An event without a valid historical version remains in the output with NULL segment attributes.

This is useful when late-arriving reference data must not remove source events.

### 2.14 Composite keys

For multi-tenant data, use the complete business identity:

    ON c.tenant_id = o.tenant_id
   AND c.customer_id = o.customer_id

Do not join only on `customer_id` if it is only unique within a tenant.

### 2.15 NULL join keys

Ordinary equality does not match NULL to NULL.

Therefore:

    order.customer_id IS NULL

normally produces an unmatched left row.

That row survives the LEFT JOIN with NULL right-side columns.

Measure NULL-key records separately from non-NULL keys that fail to find a reference.

### 2.16 LEFT JOIN and many-to-many relationships

A LEFT JOIN can still produce a large many-to-many multiplication.

For a key with:

    left_count = 4
    right_count = 5

the matched portion can contain:

    4 × 5 = 20

rows.

The unmatched-left preservation property does not protect against multiplication.

## 3. Implementation

### 3.1 Define the contract

Example:

    Left input:       orders
    Left grain:      one row per order
    Right input:     customers
    Right grain:     one row per tenant + customer
    Join key:        tenant_id + customer_id
    Cardinality:     many-to-one
    Preservation:    every order
    Missing match:   retain order with NULL customer fields
    Duplicate key:   fail
    Output grain:    one row per order

### 3.2 PostgreSQL schema

    CREATE TABLE customers (
        tenant_id   BIGINT NOT NULL,
        customer_id BIGINT NOT NULL,
        segment     TEXT,
        status      TEXT NOT NULL,
        PRIMARY KEY (tenant_id, customer_id)
    );

    CREATE TABLE orders (
        tenant_id   BIGINT NOT NULL,
        order_id    BIGINT PRIMARY KEY,
        customer_id BIGINT,
        amount      NUMERIC(18,2) NOT NULL
    );

### 3.3 Basic LEFT JOIN

    SELECT
        o.tenant_id,
        o.order_id,
        o.customer_id,
        o.amount,
        c.segment,
        c.status AS customer_status
    FROM orders AS o
    LEFT JOIN customers AS c
      ON c.tenant_id = o.tenant_id
     AND c.customer_id = o.customer_id;

Every order remains in the result.

### 3.4 Verify left-population preservation

Before trusting the result, compare:

    SELECT COUNT(*) FROM orders;

with:

    SELECT COUNT(DISTINCT order_id)
    FROM (
        SELECT o.order_id
        FROM orders AS o
        LEFT JOIN customers AS c
          ON c.tenant_id = o.tenant_id
         AND c.customer_id = o.customer_id
    ) AS joined;

For a many-to-one contract, these values should be equal.

### 3.5 Detect right-side multiplication

    SELECT
        o.order_id,
        COUNT(*) AS joined_rows
    FROM orders AS o
    LEFT JOIN customer_stage AS c
      ON c.tenant_id = o.tenant_id
     AND c.customer_id = o.customer_id
    GROUP BY o.order_id
    HAVING COUNT(*) > 1;

If rows are returned, the right side is producing multiple matches for orders.

### 3.6 Validate right-side uniqueness

    SELECT
        tenant_id,
        customer_id,
        COUNT(*) AS row_count
    FROM customer_stage
    GROUP BY tenant_id, customer_id
    HAVING COUNT(*) > 1;

For a many-to-one contract, this query should return no rows.

### 3.7 Correct right-side filtering

Requirement:

    Preserve every order.
    Match only active customers.

Correct:

    SELECT
        o.order_id,
        c.segment
    FROM orders AS o
    LEFT JOIN customers AS c
      ON c.tenant_id = o.tenant_id
     AND c.customer_id = o.customer_id
     AND c.status = 'active';

Incorrect for the same requirement:

    SELECT
        o.order_id,
        c.segment
    FROM orders AS o
    LEFT JOIN customers AS c
      ON c.tenant_id = o.tenant_id
     AND c.customer_id = o.customer_id
    WHERE c.status = 'active';

### 3.8 Distinguish no match from NULL attributes

Add a non-null right-side identity marker:

    SELECT
        o.order_id,
        c.customer_id AS matched_customer_id,
        c.segment
    FROM orders AS o
    LEFT JOIN customers AS c
      ON c.tenant_id = o.tenant_id
     AND c.customer_id = o.customer_id;

Then:

    matched_customer_id IS NULL

means no right-side customer row matched, assuming `customer_id` is non-null in the customer table.

### 3.9 Measure match and unmatched populations

    SELECT
        COUNT(*) AS left_rows,
        COUNT(c.customer_id) AS matched_rows,
        COUNT(*) - COUNT(c.customer_id) AS unmatched_rows
    FROM orders AS o
    LEFT JOIN customers AS c
      ON c.tenant_id = o.tenant_id
     AND c.customer_id = o.customer_id;

For a unique non-null customer key, this provides simple population accounting.

### 3.10 Materialize unmatched records

    SELECT
        o.*
    FROM orders AS o
    LEFT JOIN customers AS c
      ON c.tenant_id = o.tenant_id
     AND c.customer_id = o.customer_id
    WHERE c.customer_id IS NULL;

This is useful for diagnostics, quarantine, or late-arriving reference workflows.

### 3.11 Pre-aggregate one-to-many reference data

Suppose an order has multiple payment events but the target needs one payment summary:

    WITH payment_summary AS (
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
    LEFT JOIN payment_summary AS p
      ON p.tenant_id = o.tenant_id
     AND p.order_id = o.order_id;

The LEFT JOIN preserves orders with no payments while keeping one row per order.

### 3.12 Temporal LEFT JOIN

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

Transactions without a valid historical segment remain present.

### 3.13 Python implementation

    def left_join_orders(orders, customers):
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

            output = dict(order)
            if customer is None:
                output["customer_segment"] = None
                output["customer_status"] = None
                output["match_status"] = "UNMATCHED"
            else:
                output["customer_segment"] = customer["segment"]
                output["customer_status"] = customer["status"]
                output["match_status"] = "MATCHED"

            result.append(output)

        return result

This implementation explicitly preserves every left record and records whether a match occurred.

### 3.14 Idempotence

A LEFT JOIN is deterministic when:

- Input snapshots are stable.
- Join predicates are deterministic.
- Right-side uniqueness or survivorship is enforced.
- Temporal rules are explicit.
- The surrounding load is idempotent.

Repeated execution should produce the same logical output for the same inputs and rule version.

## 4. Testing

LEFT JOIN tests must prove both population preservation and right-side cardinality.

### 4.1 Basic preservation test

    def test_left_join_preserves_unmatched_orders():
        orders = [
            {"tenant_id": 1, "order_id": 101, "customer_id": 7},
            {"tenant_id": 1, "order_id": 102, "customer_id": 8},
        ]
        customers = [
            {"tenant_id": 1, "customer_id": 7, "segment": "gold", "status": "active"},
        ]

        result = left_join_orders(orders, customers)

        assert len(result) == 2
        assert result[0]["match_status"] == "MATCHED"
        assert result[1]["match_status"] == "UNMATCHED"

### 4.2 Test complete matching

Given 100 left rows and 100 unique matching right rows:

    left_rows == 100
    output_rows == 100
    unmatched_rows == 0

### 4.3 Test missing reference

Given one left row without a right match:

    left_rows == 100
    distinct_left_keys_in_output == 100
    unmatched_rows == 1

The output must retain the left row.

### 4.4 Test duplicate right key

Provide two right rows for the same expected unique key.

Expected behavior:

    transformation fails before trusted output

Do not arbitrarily select a customer record.

### 4.5 Test NULL left key

Given:

    customer_id = NULL

verify that:

- the left row survives;
- the right side is NULL-extended;
- the row is classified as a NULL-key case rather than an ordinary reference miss.

### 4.6 Test NULL right attribute

Provide a valid customer row whose `segment` is NULL.

Verify that the result can distinguish:

    customer exists + segment NULL

from:

    customer does not exist.

### 4.7 Test `ON` versus `WHERE`

Create an inactive customer and an order referencing it.

Verify that:

    LEFT JOIN ... ON customer.status = 'active'

preserves the order, while:

    LEFT JOIN ... WHERE customer.status = 'active'

removes it.

This should be a mandatory regression test for important LEFT JOIN transformations.

### 4.8 Test one-to-many multiplication

Create one order and two payment records.

Verify that a direct LEFT JOIN produces two rows.

Then test the pre-aggregated version and verify one row per order.

### 4.9 Test composite keys

Use identical customer IDs in two tenants and verify that each order receives only its tenant's customer.

### 4.10 Test temporal boundaries

Use:

    A = [2026-01-01, 2026-07-01)
    B = [2026-07-01, 2027-01-01)

Verify that an event at `2026-07-01` matches B.

An event outside all valid intervals should remain in the LEFT JOIN output with NULL reference attributes.

### 4.11 Test match accounting

For a unique many-to-one relationship:

    left_rows = matched_rows + unmatched_rows

and:

    distinct_left_keys_in_output = left_rows

These are useful assertions for every production run.

### 4.12 Test idempotence

Run the same transformation twice against identical snapshots and compare output keys, values, match statuses, and accounting.

## 5. Observability

LEFT JOIN observability focuses on population preservation and match quality.

### Core metrics

| Metric | Meaning |
|---|---|
| `left_join_left_rows` | Total left-side population |
| `left_join_right_rows` | Right-side population |
| `left_join_matched_rows` | Left rows with a valid match |
| `left_join_unmatched_rows` | Left rows without a valid match |
| `left_join_match_rate` | Matched / left population |
| `left_join_output_rows` | Physical output rows |
| `left_join_duplicate_right_keys` | Right-side uniqueness violations |
| `left_join_null_key_rows` | Left rows with NULL join keys |
| `left_join_multiplication_factor` | Output relative to distinct left keys |
| `left_join_rule_version` | Version of join semantics |

### Population preservation

For a many-to-one LEFT JOIN:

    left_rows = distinct_left_keys_in_output

and normally:

    output_rows = left_rows

when each left key has at most one right match.

If `output_rows > left_rows`, investigate right-side fan-out.

### Match rate

    match_rate = matched_left_rows / left_rows

Track this over time and segment it by:

- tenant
- source
- business date
- reference version
- partition

A global match rate can hide localized reference failures.

### Unmatched reasons

Useful classifications include:

    NULL_JOIN_KEY
    KEY_NOT_FOUND
    REFERENCE_INACTIVE
    REFERENCE_EXPIRED
    REFERENCE_NOT_YET_ARRIVED

This is more actionable than one generic `UNMATCHED` count.

### Fan-out monitoring

For each left business key, calculate the number of right matches.

Useful statistics include:

- maximum right matches per left key
- p95/p99 fan-out
- number of left keys with more than one match
- output-to-left row ratio

For a many-to-one contract, any fan-out above one is a correctness signal.

## 6. Intentional Failure

### Failure 1 — Move a right-side filter into WHERE

Change:

    ON c.customer_id = o.customer_id
   AND c.status = 'active'

to:

    ON c.customer_id = o.customer_id
    WHERE c.status = 'active'

Expected symptom:

- Unmatched and inactive-reference orders disappear.
- Left population is no longer preserved.

Recovery:

Restore the predicate placement and rerun population reconciliation.

### Failure 2 — Duplicate the right-side key

Insert two customer rows for one `(tenant_id, customer_id)`.

Expected symptom:

- One order produces multiple rows.
- `output_rows > left_rows`.

Recovery:

Correct or quarantine the duplicate and replay.

### Failure 3 — Remove part of a composite key

Join only on `customer_id`.

Expected symptom:

- Cross-tenant matches.
- Incorrect attributes.
- Potential multiplication.

Recovery:

Restore the complete key and replay the affected scope.

### Failure 4 — Treat NULL attribute as no match

Create a real customer with a NULL `segment`.

Expected symptom:

- Monitoring incorrectly reports the customer as missing.

Recovery:

Use a guaranteed non-null identity column to determine whether a right-side row exists.

### Failure 5 — Join raw one-to-many events

Join orders directly to payment events.

Expected symptom:

- Multiple rows per order.
- Downstream sums may be inflated.

Recovery:

Pre-aggregate the event relation or explicitly change the target grain.

### Failure 6 — Use current reference data historically

Remove temporal validity from a historical join.

Expected symptom:

- Previously generated historical values change after reference updates.

Recovery:

Restore event-time validity and replay affected historical partitions.

## 7. Recovery

LEFT JOIN incidents usually fall into three categories:

1. Population loss.
2. Unexpected multiplication.
3. Incorrect matching.

### Recovery sequence

1. Identify the affected pipeline run and rule version.
2. Preserve the input snapshots.
3. Compare left rows, output rows, matched rows, and unmatched rows.
4. Check right-side key uniqueness.
5. Inspect `ON` and `WHERE` predicates.
6. Check NULL-key volume.
7. Check composite-key completeness.
8. Check reference freshness.
9. Check temporal overlap or expiration.
10. Identify affected partitions or business dates.
11. Correct the transformation or reference data.
12. Re-run validation.
13. Replay the affected scope.
14. Reconcile counts and business measures.
15. Record the incident.

### Recovering unexpected population loss

Start from the left population:

    source left rows
          ↓
    classify match status
          ↓
    identify missing keys
          ↓
    determine whether missing is expected
          ↓
    correct source/reference/rule
          ↓
    replay

Do not manually add lost records to the target while leaving the transformation broken.

### Recovering fan-out

If one left key has multiple right matches:

1. Identify the duplicate right records.
2. Determine whether they represent valid versions or invalid duplicates.
3. Apply the correct temporal or survivorship rule.
4. Rebuild the affected result.
5. Verify one output row per left key where required.
6. Reconcile important measures.

### Late-arriving reference data

If the right-side record arrives after the left-side event:

- Preserve the left record.
- Record the unmatched state.
- Track the missing reference.
- Reprocess when the reference becomes available.

Do not repeatedly rerun the entire historical dataset when a bounded replay can repair the affected scope.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL provides LEFT JOIN semantics, constraints, indexes, query planning, and `EXPLAIN ANALYZE`.

Use primary keys and unique constraints to make many-to-one assumptions enforceable.

### 2. dbt

dbt provides version-controlled SQL models and data tests for relationship assumptions, uniqueness, and custom population assertions.

Use tests to make LEFT JOIN cardinality expectations executable.

### 3. DuckDB

DuckDB is useful for creating small reproducible LEFT JOIN datasets from CSV or Parquet and investigating NULL and cardinality behavior locally.

## 9. Production Runbook

### Before deployment

- [ ] Define the left population that must survive.
- [ ] Define the right-side matching contract.
- [ ] Define expected cardinality.
- [ ] Define unmatched semantics.
- [ ] Define NULL-key behavior.
- [ ] Define whether right-side filters belong in `ON`.
- [ ] Validate right-side uniqueness.
- [ ] Define temporal semantics if applicable.
- [ ] Add population reconciliation.
- [ ] Add fan-out detection.

### During execution

- [ ] Record left input rows.
- [ ] Record right input rows.
- [ ] Record matched rows.
- [ ] Record unmatched rows.
- [ ] Record NULL-key rows.
- [ ] Record output rows.
- [ ] Monitor match rate.
- [ ] Monitor fan-out.
- [ ] Monitor query resource usage.

### If left rows disappear

1. Check whether a right-side predicate was moved into WHERE.
2. Check downstream filters.
3. Compare distinct left keys before and after.
4. Inspect the join predicate.
5. Check whether the query was changed from LEFT JOIN to INNER JOIN.
6. Reconcile unmatched categories.

### If output rows increase

1. Compare output rows with distinct left keys.
2. Check right-side uniqueness.
3. Identify fan-out by left key.
4. Check for many-to-many relationships.
5. Check whether pre-aggregation was removed.
6. Check temporal overlap.

### If match rate falls

1. Inspect newly unmatched keys.
2. Check source and reference freshness.
3. Check normalization and data types.
4. Check tenant scoping.
5. Check reference eligibility.
6. Check whether the business population changed.

## 10. Common Mistakes

### Mistake 1 — Assuming LEFT JOIN guarantees one row per left record

It guarantees preservation, not uniqueness.

### Mistake 2 — Putting right-side filters in WHERE

This can remove NULL-extended rows and defeat population preservation.

### Mistake 3 — Using a nullable attribute to detect a match

Use a guaranteed non-null right-side identity.

### Mistake 4 — Ignoring right-side duplicates

A LEFT JOIN can multiply the left population.

### Mistake 5 — Treating unmatched as automatically bad

Some reference relationships are legitimately optional.

### Mistake 6 — Treating unmatched as automatically acceptable

Required reference data may make an unmatched row a data-quality incident.

### Mistake 7 — Using `DISTINCT` to hide fan-out

Fix the relationship rather than masking it.

### Mistake 8 — Ignoring NULL join keys

Separate missing identity from missing reference data.

### Mistake 9 — Forgetting temporal semantics

Historical enrichment must use historically valid reference records.

### Mistake 10 — Measuring only output row count

Population preservation requires distinct left-key accounting.

### Mistake 11 — Using LEFT JOIN for existence tests

When right-side attributes are unnecessary, an existence-oriented semi-join may better express the requirement.

### Mistake 12 — Assuming match rate proves correctness

Incorrect predicates can still produce high match rates.

## 11. Definition of Done

The LEFT JOIN transformation is complete when you can:

- [ ] Explain left-population preservation.
- [ ] Define input and output grain.
- [ ] Define expected right-side cardinality.
- [ ] Explain NULL-extended rows.
- [ ] Distinguish no right row from a NULL right attribute.
- [ ] Use composite keys correctly.
- [ ] Explain the difference between right-side predicates in ON and WHERE.
- [ ] Account for unmatched left rows.
- [ ] Detect right-side fan-out.
- [ ] Pre-aggregate one-to-many inputs when necessary.
- [ ] Implement temporal LEFT JOINs.
- [ ] Test NULL, missing, duplicate, composite-key, and temporal cases.
- [ ] Measure match rate and unmatched reasons.
- [ ] Intentionally break population preservation and diagnose it.
- [ ] Recover and replay affected data safely.
- [ ] Explain how PostgreSQL, dbt, and DuckDB support production LEFT JOIN work.

## 12. What You Learned

A LEFT JOIN is the primary relational pattern for preserving a required left-side population while optionally attaching information from another relation.

The important distinction is:

    PRESERVE LEFT POPULATION
              ≠
       PRESERVE LEFT GRAIN

A LEFT JOIN can preserve every order while still producing multiple rows per order if the right side is not unique.

The production workflow is:

    DEFINE LEFT POPULATION
         ↓
    DEFINE OUTPUT GRAIN
         ↓
    DEFINE MATCHING RULE
         ↓
    VALIDATE RIGHT CARDINALITY
         ↓
    APPLY LEFT JOIN
         ↓
    ACCOUNT FOR MATCHED / UNMATCHED
         ↓
    CHECK FAN-OUT
         ↓
    RECONCILE LEFT POPULATION
         ↓
    REPLAY SAFELY IF REQUIRED

> **A LEFT JOIN is correct only when every left record is preserved exactly as the contract requires and every missing or multiplied match is explainable.**

### Next recipe

**T23 — FULL OUTER JOIN**