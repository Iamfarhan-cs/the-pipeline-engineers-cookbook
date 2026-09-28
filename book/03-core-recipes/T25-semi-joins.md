# T25 — Semi-Joins

> **Goal:** Learn how to keep left-side records when at least one qualifying related record exists, without multiplying the left population or unnecessarily retrieving right-side attributes.

## 1. Problem Recognition

A semi-join answers the existence question opposite to an anti-join:

    Which rows from A have at least one qualifying match in B?

Typical production cases include:

- Customers who have placed at least one order.
- Orders that have at least one successful payment.
- Accounts that have an approved compliance review.
- Events associated with a known entity.
- Products that appear in a valid catalog.
- Source records already represented in a reference dataset.
- Tenants that have activity during a reporting period.
- Selecting entities that satisfy a relationship without returning relationship rows.

The output contains rows from the left relation only.

A semi-join is about **existence**, not right-side attribute retrieval.

### Recognition questions

Before implementing a semi-join, ask:

1. What is the left population?
2. What exactly counts as a qualifying match?
3. Do I need right-side columns in the final output?
4. Can the right side contain duplicates?
5. Is the relationship key composite?
6. Are tenant, status, time, or partition predicates part of eligibility?
7. How should NULL join keys be handled?
8. How will the retained population be reconciled against the source?

## 2. Concept and Reasoning

### 2.1 Semi-join semantics

Conceptually:

    A SEMI JOIN B

means:

    keep a ∈ A
    when at least one b ∈ B
    satisfies the match predicate

The right-side row is used only to establish existence.

### 2.2 The canonical SQL form: EXISTS

The clearest SQL expression is:

    SELECT o.*
    FROM orders AS o
    WHERE EXISTS (
        SELECT 1
        FROM payments AS p
        WHERE p.order_id = o.order_id
          AND p.status = 'SUCCESS'
    );

Read this as:

    return the order when at least one successful payment exists.

`EXISTS` stops being logically interested in additional matching rows after existence has been established.

### 2.3 Why EXISTS preserves left-side grain

Suppose one order has five successful payments.

An ordinary INNER JOIN can produce:

    order 100 → payment 1
    order 100 → payment 2
    order 100 → payment 3
    order 100 → payment 4
    order 100 → payment 5

A semi-join produces:

    order 100

because the requirement is only:

    does at least one qualifying payment exist?

This makes `EXISTS` the natural expression for existence filtering.

### 2.4 Semi-join versus INNER JOIN

| Requirement | Pattern |
|---|---|
| Keep left rows with at least one qualifying match | `EXISTS` |
| Return columns from both sides | `JOIN` |
| Keep left rows with no qualifying match | `NOT EXISTS` |
| Return only matching key combinations | `JOIN` or set operation |

If the right-side data is not required in the output, an ordinary join may introduce unnecessary row multiplication.

### 2.5 Semi-join versus DISTINCT after JOIN

A common workaround is:

    SELECT DISTINCT o.*
    FROM orders AS o
    JOIN payments AS p
      ON p.order_id = o.order_id
     AND p.status = 'SUCCESS';

This may produce the desired final rows, but `DISTINCT` is repairing a multiplication introduced by the join.

The direct expression is:

    SELECT o.*
    FROM orders AS o
    WHERE EXISTS (
        SELECT 1
        FROM payments AS p
        WHERE p.order_id = o.order_id
          AND p.status = 'SUCCESS'
    );

`EXISTS` states the intent directly.

### 2.6 Qualifying match is part of the contract

Suppose an order has only:

    payment status = FAILED

If the requirement is:

    orders with at least one successful payment

then the order must not be retained.

Correct:

    EXISTS successful payment

Incorrect:

    EXISTS any payment

The predicate defines the relationship.

### 2.7 Predicate scope

Suppose the relationship is tenant-scoped:

    tenant_id + order_id

The semi-join must use both values:

    p.tenant_id = o.tenant_id
    AND p.order_id = o.order_id

Otherwise a matching order from another tenant can incorrectly satisfy the existence condition.

### 2.8 Duplicate right-side rows

Duplicates do not multiply a semi-join result.

If ten qualifying rows match one order:

    EXISTS = TRUE

The order is retained once.

That makes semi-joins especially useful when the right side is event-like or naturally one-to-many.

However, duplicates can still indicate a source-quality issue and should be monitored when uniqueness is expected.

### 2.9 NULL join keys

With ordinary equality:

    p.order_id = o.order_id

a NULL does not compare equal to another NULL.

Therefore a left row with NULL identity does not automatically find a matching right row.

If the business key is mandatory, classify NULL-key records separately rather than allowing them to silently disappear from the retained population.

If NULL should have special matching semantics, implement that rule explicitly rather than relying on ordinary equality.

### 2.10 EXISTS and NULL values in the right side

Unlike `NOT IN`, `EXISTS` does not suffer from the classic subquery-NULL trap.

For example:

    WHERE EXISTS (
        SELECT 1
        FROM payments AS p
        WHERE p.order_id = o.order_id
    )

Other NULL-valued payment rows do not make the entire predicate UNKNOWN merely because they exist.

The correlated predicate is evaluated row by row.

### 2.11 Correlated subqueries

An `EXISTS` subquery is often correlated to the current left row:

    WHERE EXISTS (
        SELECT 1
        FROM child AS c
        WHERE c.parent_id = parent.parent_id
    )

The inner query asks whether a qualifying child exists for this specific parent.

This is relational existence expressed directly in SQL.

### 2.12 Temporal semi-joins

Sometimes existence must be true at a particular point in time.

Example:

    SELECT t.*
    FROM transactions AS t
    WHERE EXISTS (
        SELECT 1
        FROM customer_segments AS s
        WHERE s.customer_id = t.customer_id
          AND t.occurred_at >= s.effective_from
          AND t.occurred_at <  s.effective_to
    );

This returns transactions with a valid segment at transaction time.

A current segment row is not necessarily valid historical evidence.

### 2.13 Semi-join as a filtering operator

A semi-join can be thought of as:

    filter A by membership in qualifying B

but the relational implementation matters.

The right side contributes eligibility, not output columns.

### 2.14 Semi-join versus lookup

Requirement:

    Keep customers that exist in the active customer table.

Use:

    EXISTS

Requirement:

    Keep customers and attach customer_segment and region.

Use:

    JOIN

The distinction is whether right-side attributes are required.

### 2.15 Semi-join and reconciliation

If:

    source_population = 10,000
    qualifying_matches = 7,500

then the semi-join output should contain:

    7,500 left records

assuming the existence predicate defines a binary matched/unmatched population and no additional left-side filters apply.

The complementary anti-join should contain:

    2,500 left records

and:

    source_population = semi_join_population + anti_join_population

when both use identical snapshots and predicates.

## 3. Implementation

### 3.1 Define the contract

Example:

    Left input:        orders
    Left grain:       one row per order
    Right input:      payments
    Match key:        tenant_id + order_id
    Qualifying match: payment status = SUCCESS
    Output:            orders with at least one successful payment
    Right attributes: not required
    Duplicate payments: irrelevant to existence, monitored

### 3.2 PostgreSQL schema

    CREATE TABLE orders (
        tenant_id BIGINT NOT NULL,
        order_id BIGINT PRIMARY KEY,
        amount NUMERIC(18,2) NOT NULL
    );

    CREATE TABLE payments (
        tenant_id BIGINT NOT NULL,
        payment_id BIGINT PRIMARY KEY,
        order_id BIGINT,
        status TEXT NOT NULL,
        amount NUMERIC(18,2) NOT NULL
    );

### 3.3 Canonical EXISTS implementation

    SELECT
        o.tenant_id,
        o.order_id,
        o.amount
    FROM orders AS o
    WHERE EXISTS (
        SELECT 1
        FROM payments AS p
        WHERE p.tenant_id = o.tenant_id
          AND p.order_id = o.order_id
          AND p.status = 'SUCCESS'
    );

This preserves the order grain regardless of how many successful payments exist.

### 3.4 Equivalent INNER JOIN

An ordinary join can express the same logical membership condition:

    SELECT DISTINCT
        o.tenant_id,
        o.order_id,
        o.amount
    FROM orders AS o
    JOIN payments AS p
      ON p.tenant_id = o.tenant_id
     AND p.order_id = o.order_id
     AND p.status = 'SUCCESS';

However, the join creates intermediate matching rows before `DISTINCT` removes duplicates.

When only existence matters, `EXISTS` communicates the contract more directly.

### 3.5 Semi-join with a temporal predicate

    SELECT
        t.transaction_id,
        t.customer_id
    FROM transactions AS t
    WHERE EXISTS (
        SELECT 1
        FROM customer_segments AS s
        WHERE s.customer_id = t.customer_id
          AND t.occurred_at >= s.effective_from
          AND t.occurred_at <  s.effective_to
    );

### 3.6 Semi-join with multiple eligibility conditions

    SELECT c.*
    FROM customers AS c
    WHERE EXISTS (
        SELECT 1
        FROM compliance_reviews AS r
        WHERE r.customer_id = c.customer_id
          AND r.status = 'APPROVED'
          AND r.reviewed_at IS NOT NULL
    );

The existence predicate should contain every condition that defines a qualifying review.

### 3.7 Semi-join for incremental processing

A source record can be filtered by whether it already exists in a target:

    SELECT s.*
    FROM source_stage AS s
    WHERE EXISTS (
        SELECT 1
        FROM target AS t
        WHERE t.business_key = s.business_key
    );

This identifies source records already represented in the target.

Do not confuse this with a complete synchronization algorithm. Updates, deletes, concurrent writes, and transaction isolation still require separate design.

### 3.8 Composite-key semi-join

    SELECT s.*
    FROM source_stage AS s
    WHERE EXISTS (
        SELECT 1
        FROM target AS t
        WHERE t.tenant_id = s.tenant_id
          AND t.business_key = s.business_key
    );

Every component of the identity must participate in the predicate.

### 3.9 Semi-join with a derived qualifying population

Sometimes the right side needs its own transformation before existence is evaluated:

    SELECT o.*
    FROM orders AS o
    WHERE EXISTS (
        SELECT 1
        FROM (
            SELECT DISTINCT order_id
            FROM payments
            WHERE status = 'SUCCESS'
        ) AS successful
        WHERE successful.order_id = o.order_id
    );

The `DISTINCT` is not required for `EXISTS`, but it can make an intermediate business rule explicit when the derived relation is reused or independently validated.

### 3.10 Python implementation

    def find_orders_with_successful_payment(orders, payments):
        successful = set()

        for payment in payments:
            if payment["status"] != "SUCCESS":
                continue

            successful.add((payment["tenant_id"], payment["order_id"]))

        result = []
        for order in orders:
            key = (order["tenant_id"], order["order_id"])
            if key in successful:
                result.append(order)

        return result

This models the semi-join as a membership test.

### 3.11 Preserve left-side columns only

If the final requirement is one row per customer, avoid selecting right-side event columns unless the business requirement needs them.

Bad design:

    customer + every qualifying event

Correct existence design:

    customer if qualifying event exists

This keeps output grain aligned with the business contract.

### 3.12 Idempotence

A semi-join should produce the same retained business-key set when evaluated against the same snapshots and predicates.

Production runs should record:

1. Input snapshot or watermark.
2. Right-side snapshot or watermark.
3. Existence rule version.
4. Left population count.
5. Retained population count.
6. Excluded population count.

## 4. Testing

### 4.1 Matching row

Given an order with one successful payment:

    order → retained

### 4.2 No matching row

Given an order with no payment:

    order → excluded

### 4.3 Non-qualifying relationship

Given an order with only a FAILED payment:

    order → excluded

This verifies that the status predicate is part of existence.

### 4.4 Multiple qualifying rows

Given one order with three successful payments:

    output → one order row

Never three order rows.

### 4.5 Right-side NULL values

Add unrelated payment rows containing NULL `order_id`.

Verify that valid order matches still work correctly.

### 4.6 NULL left key

Create a left record with a NULL join key.

Verify the documented identity policy.

If the key is mandatory, record the invalid identity separately rather than silently treating it as a normal non-match.

### 4.7 Composite-key isolation

Create:

    tenant 1 + order 10
    tenant 2 + order 10

Add a payment for tenant 1 only.

Expected:

    tenant 1 order 10 → retained
    tenant 2 order 10 → excluded

### 4.8 Temporal validity

Create a reference row outside the event's valid interval.

Expected:

    event → excluded

because no qualifying reference exists at event time.

### 4.9 EXISTS versus JOIN equivalence

On a controlled dataset, compare:

    EXISTS result

against:

    DISTINCT JOIN result

The business-key sets should match when the predicates are logically equivalent.

### 4.10 Duplicate invariance

Duplicate the qualifying right-side record.

Expected:

    semi-join output unchanged

This proves that existence rather than multiplicity drives the result.

### 4.11 Complementary anti-join test

Run the corresponding anti-join with the exact same match predicate.

Verify:

    source = semi_join + anti_join

and that the two output populations do not overlap.

### 4.12 Idempotence test

Run the semi-join twice against identical snapshots.

Expected:

    identical business-key set
    identical count

## 5. Observability

Semi-join observability should make membership measurable.

### Core metrics

| Metric | Meaning |
|---|---|
| `semi_join_left_rows` | Left population evaluated |
| `semi_join_matched_rows` | Left rows with a qualifying match |
| `semi_join_unmatched_rows` | Left rows without a qualifying match |
| `semi_join_match_rate` | Matched / left population |
| `semi_join_unmatched_rate` | Unmatched / left population |
| `semi_join_null_key_rows` | Left rows with NULL identity |
| `semi_join_right_duplicate_keys` | Duplicate right-side keys |
| `semi_join_rule_version` | Version of eligibility semantics |

### Population accounting

For a binary membership condition:

    left_rows = matched_rows + unmatched_rows

The semi-join output should equal `matched_rows`.

The complementary anti-join should equal `unmatched_rows`.

### Match-rate monitoring

A sudden change in match rate can indicate:

- Reference-data freshness problems.
- Source-key normalization changes.
- Eligibility rule changes.
- Tenant scoping defects.
- Status-code changes.
- Temporal boundary changes.
- Upstream ingestion failures.

Monitor the metric by useful dimensions when the dataset is large:

- Tenant.
- Source system.
- Partition/date.
- Entity type.
- Processing version.

### Duplicate monitoring

Existence semantics hide duplicate multiplication, but duplicates may still indicate broken upstream uniqueness.

Track duplicate right-side keys when uniqueness is expected.

### Reason-level diagnostics

Classify non-matches when useful:

    NO_RELATION
    RELATION_NOT_ELIGIBLE
    RELATION_EXPIRED
    NULL_JOIN_KEY
    REFERENCE_LAG

This prevents every excluded record from being reduced to one opaque count.

## 6. Intentional Failure

### Failure 1 — Replace EXISTS with ordinary JOIN

Create multiple qualifying right-side rows per left record.

Expected symptom:

- Left-side row count increases.
- Downstream grain is violated.

Recovery:

Use `EXISTS` when only membership is required, or explicitly deduplicate/pre-aggregate before joining.

### Failure 2 — Remove the status predicate

Create a left record with only a FAILED relationship.

Expected symptom:

- The record becomes incorrectly retained.

Recovery:

Restore the complete qualifying predicate.

### Failure 3 — Remove tenant from a composite key

Create identical business keys across two tenants.

Expected symptom:

- A relationship in one tenant incorrectly satisfies another tenant.

Recovery:

Restore the tenant predicate and add a cross-tenant regression test.

### Failure 4 — Ignore temporal validity

Use a reference record that was not valid at event time.

Expected symptom:

- Historical records become incorrectly retained.

Recovery:

Restore effective-time predicates.

### Failure 5 — Add an unnecessary DISTINCT workaround

Implement the semi-join with an INNER JOIN and DISTINCT.

Expected symptom:

- Query may become more expensive and the underlying existence intent is obscured.

Recovery:

Use `EXISTS` when no right-side attributes are needed.

### Failure 6 — Treat NULL identity as a normal relationship

Introduce NULL keys in the left population.

Expected symptom:

- Records silently fail membership checks.

Recovery:

Separate required-key validation from existence logic and define explicit NULL policy.

### Failure 7 — Use semi-join when attributes are required

Attempt to obtain the latest payment amount with only `EXISTS`.

Expected symptom:

- The existence test proves membership but cannot provide the required right-side value.

Recovery:

Use a controlled join after deterministically selecting the required right-side record.

### Failure 8 — Change the predicate without changing the rule version

Modify eligibility from `SUCCESS` to another status without versioning the rule.

Expected symptom:

- Historical match-rate changes become difficult to explain.

Recovery:

Version the business rule and record it with each run.

## 7. Recovery

Semi-join incidents usually involve incorrect membership, unintended multiplication, or changed eligibility.

### Recovery sequence

1. Identify the affected run.
2. Record the existence-rule version.
3. Capture left and right snapshots.
4. Verify the complete match key.
5. Verify status and eligibility predicates.
6. Verify tenant and partition scope.
7. Verify temporal boundaries.
8. Compare semi-join output with its complementary anti-join.
9. Compare against the last known-good run.
10. Correct the transformation.
11. Re-run the affected scope.
12. Reconcile source population.
13. Replay downstream processing idempotently.
14. Record the root cause.

### Recovering from row multiplication

If an INNER JOIN was used accidentally:

1. Measure duplicate output keys.
2. Identify right-side multiplicity.
3. Replace the join with `EXISTS` if attributes are unnecessary.
4. Re-run the affected partition.
5. Verify one-row-per-left-key output.

### Recovering from false matches

If the semi-join retained records incorrectly:

1. Identify the right-side row that satisfied the predicate.
2. Check key completeness.
3. Check status/eligibility conditions.
4. Check temporal validity.
5. Check tenant scope.
6. Correct the predicate.
7. Recalculate the retained population.

### Recovering from reference lag

If qualifying right-side records arrive late:

- Record unmatched records as a distinct operational category.
- Preserve the source snapshot.
- Wait for the documented dependency.
- Reprocess the affected scope after reference arrival.
- Avoid broad historical replay when a bounded replay is sufficient.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL supports `EXISTS`, correlated subqueries, indexes, and query planning for relational membership checks.

Learn to inspect `EXPLAIN` and `EXPLAIN ANALYZE` when semi-joins operate over large tables.

### 2. dbt

dbt can model relationship-based filters and expose membership or orphan datasets as explicit transformation and quality models.

Use tests to make relationship assumptions executable.

### 3. DuckDB

DuckDB is useful for reproducing semi-join behavior locally against CSV and Parquet datasets.

It is particularly useful for comparing `EXISTS`, joins, and duplicate behavior on realistic analytical data.

## 9. Production Runbook

### Before deployment

- [ ] Define the left population.
- [ ] Define the qualifying relationship.
- [ ] Define the complete identity key.
- [ ] Decide NULL-key handling.
- [ ] Confirm right-side attributes are not required.
- [ ] Add duplicate-invariance tests.
- [ ] Add composite-key tests where applicable.
- [ ] Add temporal tests where applicable.
- [ ] Define population reconciliation.
- [ ] Version the existence rule.

### During execution

- [ ] Record left input count.
- [ ] Record semi-join output count.
- [ ] Record complementary anti-join count.
- [ ] Record match rate.
- [ ] Record NULL-key count.
- [ ] Record duplicate right-side keys when relevant.
- [ ] Record source/reference snapshots or watermarks.

### If match rate suddenly increases

1. Check whether the reference population expanded.
2. Check eligibility predicates.
3. Check key normalization.
4. Check tenant scoping.
5. Check temporal boundaries.
6. Compare newly matched keys with the previous run.

### If match rate suddenly decreases

1. Check reference freshness.
2. Check upstream ingestion.
3. Check key normalization.
4. Check status mappings.
5. Check temporal validity.
6. Check for NULL or malformed keys.

### If output contains duplicate left keys

1. Confirm whether the implementation is actually a JOIN.
2. Inspect right-side multiplicity.
3. Replace existence-only joins with `EXISTS`.
4. Re-run and verify output grain.

### If the query is slow

1. Inspect the query plan.
2. Check indexes on correlated predicates.
3. Restrict the right-side population using selective conditions.
4. Restrict the left population when the pipeline contract allows it.
5. Check for unnecessary joins or DISTINCT operations.
6. Compare execution plans before and after changes.

## 10. Common Mistakes

### Mistake 1 — Joining when only existence is required

This can multiply the left population.

### Mistake 2 — Using DISTINCT to hide join multiplication

Often the better fix is to express existence directly.

### Mistake 3 — Forgetting qualifying conditions

Any relationship is not necessarily a qualifying relationship.

### Mistake 4 — Forgetting composite-key components

Partial identity creates false matches.

### Mistake 5 — Ignoring tenant boundaries

Cross-tenant matches can silently corrupt membership logic.

### Mistake 6 — Ignoring temporal validity

Current relationships are not automatically historical relationships.

### Mistake 7 — Ignoring NULL identity

NULL keys require an explicit policy.

### Mistake 8 — Assuming duplicates are harmless everywhere

Duplicates do not change `EXISTS`, but they can indicate upstream data-quality defects.

### Mistake 9 — Using semi-join when right attributes are needed

Existence does not retrieve values.

### Mistake 10 — Forgetting the complementary population

Match and non-match populations should reconcile when the rule is binary.

### Mistake 11 — Not versioning business eligibility

Changing a status or temporal rule can change historical match rates.

### Mistake 12 — Reprocessing the entire dataset unnecessarily

Use bounded replay when the affected population is known.

## 11. Definition of Done

The semi-join transformation is complete when you can:

- [ ] Define semi-join semantics.
- [ ] Explain why `EXISTS` is the canonical SQL expression.
- [ ] Distinguish semi-joins from INNER JOINs.
- [ ] Explain why joins can multiply left-side rows.
- [ ] Implement `EXISTS` correctly.
- [ ] Handle qualifying status and eligibility predicates.
- [ ] Handle composite keys.
- [ ] Handle tenant-scoped relationships.
- [ ] Handle temporal membership.
- [ ] Explain NULL-key behavior.
- [ ] Prove duplicate right-side rows do not multiply `EXISTS` output.
- [ ] Compare `EXISTS` with `DISTINCT JOIN` on a controlled dataset.
- [ ] Test matching, missing, non-qualifying, duplicate, NULL, and temporal cases.
- [ ] Reconcile semi-join and anti-join populations.
- [ ] Monitor match rate and reason-level diagnostics.
- [ ] Intentionally introduce membership and grain failures.
- [ ] Recover safely from false matches and row multiplication.
- [ ] Explain how PostgreSQL, dbt, and DuckDB support production semi-join work.

## 12. What You Learned

A semi-join answers one of the most common relational questions in Data Engineering:

    DOES AT LEAST ONE QUALIFYING RELATED RECORD EXIST?

If yes, retain the left record.

The production workflow is:

    DEFINE LEFT POPULATION
         ↓
    DEFINE QUALIFYING RELATIONSHIP
         ↓
    DEFINE COMPLETE KEY
         ↓
    HANDLE NULL SEMANTICS
         ↓
    APPLY EXISTS
         ↓
    PRESERVE LEFT GRAIN
         ↓
    RECONCILE WITH ANTI-JOIN
         ↓
    OBSERVE MATCH RATE
         ↓
    REPLAY SAFELY IF REQUIRED

> **When the requirement is membership rather than attribute retrieval, express membership directly with `EXISTS` and protect the left-side grain.**

### Next recipe

**T26 — Window Functions**