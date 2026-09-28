# T24 — Anti-Joins

> **Goal:** Learn how to find rows for which no qualifying related row exists, without accidentally losing records because of NULL semantics, duplicate matches, or incorrect predicate scope.

## 1. Problem Recognition

An anti-join answers an existence question:

    Which rows from A have no qualifying match in B?

Typical production cases include:

- Customers with no transactions.
- Orders with no successful payment.
- Source records missing from a reference dataset.
- Events without a corresponding entity.
- Accounts without required compliance records.
- Files that have no successful processing record.
- Records that have not yet been loaded downstream.
- Detecting orphaned relationships.

The output grain is usually the left-side population, filtered to records for which no qualifying right-side relationship exists.

Common production failures include:

- Using `NOT IN` against a nullable column and unexpectedly returning zero rows.
- Using an ordinary INNER JOIN and trying to infer non-existence afterward.
- Using `LEFT JOIN ... IS NULL` with the wrong nullable column.
- Forgetting part of a composite key.
- Placing predicates incorrectly so an unrelated right-side row prevents the expected match.
- Using a one-to-many join that creates duplicates before attempting exclusion.
- Confusing 'no row exists' with 'row exists but does not satisfy a condition'.
- Building an anti-join without a deterministic definition of what constitutes a qualifying match.

### Recognition questions

Before implementing an anti-join, ask:

1. What is the left population?
2. What exactly counts as a qualifying match?
3. Do I need only existence information or right-side attributes?
4. Can the right-side key contain NULL?
5. Is the identity composite?
6. Does relationship eligibility depend on status, time, tenant, or another predicate?
7. Should duplicate right-side rows have any effect?
8. How will excluded and retained populations be reconciled?

## 2. Concept and Reasoning

### 2.1 Anti-join semantics

An anti-join returns left rows for which no qualifying right row exists.

Conceptually:

    A ANTI JOIN B

means:

    keep a ∈ A
    when there is no b ∈ B
    such that the join predicate is TRUE

The right-side row itself does not appear in the output.

### 2.2 Anti-join is an existence operation

Suppose:

    orders: one row per order
    payments: many rows per order

Requirement:

    find orders with no successful payment

The desired output is still one row per order.

An anti-join expresses:

    no successful payment exists

rather than:

    join every payment row and then remove matches

This distinction protects the intended grain.

### 2.3 The canonical SQL form: NOT EXISTS

The most explicit form is:

    SELECT o.*
    FROM orders AS o
    WHERE NOT EXISTS (
        SELECT 1
        FROM payments AS p
        WHERE p.order_id = o.order_id
          AND p.status = 'SUCCESS'
    );

Read this as:

    return the order when no successful payment exists.

`NOT EXISTS` is usually the clearest expression of anti-join intent.

### 2.4 LEFT JOIN ... IS NULL

Another common form is:

    SELECT o.*
    FROM orders AS o
    LEFT JOIN payments AS p
      ON p.order_id = o.order_id
     AND p.status = 'SUCCESS'
    WHERE p.order_id IS NULL;

This can express the same existence logic when the right-side identity column is guaranteed non-null.

Critical detail:

    p.status = 'SUCCESS'

belongs in `ON`, not `WHERE`, because the LEFT JOIN must first identify qualifying matches.

### 2.5 NOT IN and the NULL trap

Consider:

    SELECT *
    FROM orders
    WHERE order_id NOT IN (
        SELECT order_id
        FROM payments
    );

If the subquery contains NULL, SQL three-valued logic can make the predicate UNKNOWN for every candidate that is not an explicit match.

This can produce an unexpectedly empty result.

Prefer `NOT EXISTS` when exclusion is based on a relational existence condition.

### 2.6 Why NOT EXISTS is robust

With:

    WHERE NOT EXISTS (...)

the question is whether at least one qualifying row exists.

Duplicate right-side records do not multiply the output.

Five successful payments for one order still mean:

    EXISTS = TRUE
    NOT EXISTS = FALSE

The order is excluded once, not five times.

### 2.7 Anti-join versus LEFT JOIN

| Requirement | Recommended pattern |
|---|---|
| Find left rows with no qualifying match | `NOT EXISTS` |
| Preserve left rows and attach right attributes | `LEFT JOIN` |
| Detect missing reference rows with diagnostics | `LEFT JOIN ... IS NULL` or `NOT EXISTS` |
| Find rows that have at least one match | Semi-join / `EXISTS` |

Use the pattern that expresses the business question directly.

### 2.8 Qualifying match is part of the definition

Suppose an order has:

    payment: FAILED

Requirement:

    find orders with no successful payment

The failed payment is not a qualifying match.

Correct:

    NOT EXISTS successful payment

Incorrect:

    NOT EXISTS any payment

These are different business questions.

### 2.9 Predicate scope

Suppose the relationship is tenant-scoped:

    tenant_id + order_id

Then the anti-join must include both:

    p.tenant_id = o.tenant_id
    AND p.order_id = o.order_id

Omitting `tenant_id` can make a payment from another tenant incorrectly satisfy the existence condition.

### 2.10 Temporal anti-joins

Sometimes the question is:

    Which events have no valid reference record at event time?

The anti-join must include temporal validity:

    NOT EXISTS (
        SELECT 1
        FROM customer_segments AS s
        WHERE s.customer_id = t.customer_id
          AND t.occurred_at >= s.effective_from
          AND t.occurred_at <  s.effective_to
    )

A current reference record is not enough if the event belongs to a historical period.

### 2.11 NULL join keys

With ordinary equality:

    p.order_id = o.order_id

an NULL `o.order_id` does not match a right-side value.

Therefore a left row with NULL identity may satisfy `NOT EXISTS` simply because no equality predicate can match it.

This may be correct or may indicate invalid source identity.

Classify NULL-key records separately when the identity is required.

### 2.12 Anti-join and duplicates

Duplicate right-side rows do not change the existence result.

If ten qualifying rows match one order:

    EXISTS = TRUE
    NOT EXISTS = FALSE

This makes anti-joins safer than ordinary joins for existence-only requirements.

However, duplicates still matter operationally because they may indicate source-quality problems.

Monitor them separately when uniqueness is expected.

### 2.13 Anti-join and data quality

Anti-joins are useful for finding:

- Orphans.
- Missing reference data.
- Unprocessed records.
- Missing downstream representations.
- Broken relationships.
- Incomplete migrations.

Treat the result as an evidence set, not automatically as a defect.

Some entities legitimately have no related records.

### 2.14 Anti-join and reconciliation

Suppose:

    source_orders = 1,000
    orders_with_successful_payment = 930

Then the anti-join population should be:

    unpaid_or_unmatched = 70

assuming the populations and predicates are aligned.

This makes anti-joins powerful reconciliation tools.

## 3. Implementation

### 3.1 Define the contract

Example:

    Left input:        orders
    Left grain:       one row per order
    Right input:      payments
    Match key:        tenant_id + order_id
    Qualifying match: payment status = SUCCESS
    Output:            orders with no successful payment
    NULL order_id:     separately classified
    Duplicate payments: allowed for existence, monitored

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

### 3.3 Canonical NOT EXISTS implementation

    SELECT
        o.tenant_id,
        o.order_id,
        o.amount
    FROM orders AS o
    WHERE NOT EXISTS (
        SELECT 1
        FROM payments AS p
        WHERE p.tenant_id = o.tenant_id
          AND p.order_id = o.order_id
          AND p.status = 'SUCCESS'
    );

This returns each qualifying order at most once because the right-side rows are used only for existence testing.

### 3.4 LEFT JOIN anti-join implementation

    SELECT
        o.tenant_id,
        o.order_id,
        o.amount
    FROM orders AS o
    LEFT JOIN payments AS p
      ON p.tenant_id = o.tenant_id
     AND p.order_id = o.order_id
     AND p.status = 'SUCCESS'
    WHERE p.payment_id IS NULL;

`payment_id` is a suitable existence marker because it is the primary key and therefore non-null.

### 3.5 Demonstrate the wrong filter placement

Incorrect:

    SELECT o.*
    FROM orders AS o
    LEFT JOIN payments AS p
      ON p.tenant_id = o.tenant_id
     AND p.order_id = o.order_id
    WHERE p.status <> 'SUCCESS';

This does not mean 'no successful payment exists'.

It can produce rows for failed payments while still failing to express the existence logic correctly.

Use `NOT EXISTS` for the business question instead.

### 3.6 NOT IN demonstration

Potentially unsafe when the subquery can contain NULL:

    SELECT *
    FROM orders
    WHERE order_id NOT IN (
        SELECT order_id
        FROM payments
    );

Safer relational expression:

    SELECT *
    FROM orders AS o
    WHERE NOT EXISTS (
        SELECT 1
        FROM payments AS p
        WHERE p.order_id = o.order_id
    );

### 3.7 Anti-join with a temporal condition

    SELECT
        t.transaction_id
    FROM transactions AS t
    WHERE NOT EXISTS (
        SELECT 1
        FROM customer_segments AS s
        WHERE s.customer_id = t.customer_id
          AND t.occurred_at >= s.effective_from
          AND t.occurred_at <  s.effective_to
    );

This finds transactions without a valid segment at transaction time.

### 3.8 Anti-join against a target table

A common incremental-loading pattern is finding source records absent from the target:

    SELECT s.*
    FROM source_stage AS s
    WHERE NOT EXISTS (
        SELECT 1
        FROM target AS t
        WHERE t.business_key = s.business_key
    );

This is useful for identifying candidate inserts, but it is not by itself a complete incremental-loading strategy. Concurrent changes, updates, deletes, and transaction isolation still need to be handled.

### 3.9 Anti-join with composite identity

    SELECT s.*
    FROM source_stage AS s
    WHERE NOT EXISTS (
        SELECT 1
        FROM target AS t
        WHERE t.tenant_id = s.tenant_id
          AND t.business_key = s.business_key
    );

Every component of the business identity must be represented.

### 3.10 Python implementation

    def find_unmatched_orders(orders, payments):
        successful = set()

        for payment in payments:
            if payment["status"] != "SUCCESS":
                continue

            successful.add((payment["tenant_id"], payment["order_id"]))

        result = []
        for order in orders:
            key = (order["tenant_id"], order["order_id"])
            if key not in successful:
                result.append(order)

        return result

This uses a set because the question is existence, not multiplication.

### 3.11 Classify exclusion reasons

An anti-join output can be enriched with a reason when multiple non-match categories exist.

Example conceptual categories:

    NO_PAYMENT
    ONLY_FAILED_PAYMENTS
    REFERENCE_NOT_FOUND
    REFERENCE_EXPIRED
    NULL_JOIN_KEY

Do not collapse all categories into `UNMATCHED` if operators need to distinguish them.

### 3.12 Idempotence

Anti-join selection is deterministic when the input snapshots and predicates are stable.

Repeated execution should produce the same set of unmatched business keys.

For production use:

1. Record input snapshot or watermark.
2. Record rule version.
3. Keep the existence predicate deterministic.
4. Reconcile the anti-join count against the total population.
5. Make downstream handling idempotent.

## 4. Testing

Anti-join tests must focus on existence semantics and NULL behavior.

### 4.1 No matching row

Given an order with no payment:

    order → returned

### 4.2 Matching qualifying row

Given an order with a successful payment:

    order → excluded

### 4.3 Non-qualifying matching row

Given:

    payment status = FAILED

and the requirement is 'no successful payment':

    order → returned

This is a critical predicate test.

### 4.4 Multiple qualifying rows

Given three successful payments for one order:

    order → excluded once

The output must not contain three copies.

### 4.5 NULL in right-side subquery

Construct a case where `payments.order_id` contains NULL.

Verify that `NOT EXISTS` still returns the correct unmatched orders.

Then compare with `NOT IN` to demonstrate the NULL trap.

### 4.6 NULL left key

Create a left row with a NULL join key.

Verify that the documented policy is applied.

If the key is required, the anti-join result should be separately classified as an invalid-identity case rather than silently treated as a legitimate orphan.

### 4.7 Composite-key test

Use:

    tenant 1 + order 7
    tenant 2 + order 7

and ensure a payment for tenant 1 cannot satisfy tenant 2's existence check.

### 4.8 Temporal test

Create a reference row that exists but is outside the event's effective interval.

Expected:

    event is returned by the anti-join

because no valid reference exists at event time.

### 4.9 Predicate-placement test

Compare the correct `NOT EXISTS` query with an incorrect LEFT JOIN that places the qualifying condition in WHERE.

Use a failed payment and a successful payment for different orders.

The test should expose the semantic difference.

### 4.10 Reconciliation equation

For a mutually exclusive existence condition:

    total_left = matched_population + anti_join_population

where `matched_population` is defined by the exact opposite qualifying-match predicate.

### 4.11 Idempotence test

Run the anti-join twice against identical snapshots.

Verify identical business-key sets and counts.

## 5. Observability

Anti-join observability should make absence measurable.

### Core metrics

| Metric | Meaning |
|---|---|
| `anti_join_left_rows` | Population evaluated for absence |
| `anti_join_matched_rows` | Left rows with a qualifying match |
| `anti_join_unmatched_rows` | Left rows with no qualifying match |
| `anti_join_match_rate` | Matched / left population |
| `anti_join_unmatched_rate` | Unmatched / left population |
| `anti_join_null_key_rows` | Left rows with NULL identity |
| `anti_join_right_duplicate_keys` | Duplicate right-side keys |
| `anti_join_rule_version` | Version of existence semantics |

### Population accounting

For a binary existence condition:

    left_rows = matched_rows + anti_join_rows

This should hold when both populations use the same snapshot and exact opposite predicates.

### Reason-level metrics

Track categories such as:

    NULL_JOIN_KEY
    KEY_NOT_FOUND
    ONLY_FAILED_RELATION
    RELATION_EXPIRED
    RELATION_NOT_YET_AVAILABLE

This makes absence actionable.

### Trend monitoring

A sudden increase in anti-join results can indicate:

- Reference-data lag.
- Source ingestion failure.
- Broken key normalization.
- Changed business behavior.
- Contract changes.
- Tenant-specific failures.
- Expired reference records.

An anti-join spike is a signal requiring diagnosis, not automatic proof of a defect.

### Cardinality monitoring

Although duplicates do not change `NOT EXISTS` results, monitor them when the right-side relationship is expected to be unique.

Duplicate existence records may reveal upstream data-quality problems that the anti-join itself hides.

## 6. Intentional Failure

### Failure 1 — Replace NOT EXISTS with NOT IN

Introduce a NULL into the right-side key column.

Expected symptom:

- The NOT IN result can unexpectedly become empty or otherwise differ from the intended anti-join.

Recovery:

Use `NOT EXISTS` or make the NULL semantics explicit and safe.

### Failure 2 — Remove tenant from the predicate

Expected symptom:

- A relation from another tenant satisfies the existence condition.

Recovery:

Restore the complete composite identity.

### Failure 3 — Check any payment instead of successful payment

Expected symptom:

- Orders with only failed payments disappear from the anti-join result.

Recovery:

Put the qualifying status predicate inside the existence condition.

### Failure 4 — Use a nullable existence marker

Construct a LEFT JOIN anti-join where the selected right-side marker can itself be NULL.

Expected symptom:

- A real match may be misclassified as unmatched.

Recovery:

Use a guaranteed non-null identity column or prefer `NOT EXISTS`.

### Failure 5 — Ignore temporal validity

Use a current reference row as proof that a historical relation existed.

Expected symptom:

- Historical events incorrectly disappear from the anti-join population.

Recovery:

Include the effective-time predicate.

### Failure 6 — Assume duplicates are irrelevant operationally

Create many duplicate qualifying right-side rows.

Expected anti-join output may remain unchanged, but right-side quality metrics should detect the duplication.

Recovery:

Investigate the source relationship even though existence semantics remain correct.

### Failure 7 — Use anti-join for a value retrieval problem

Try to retrieve the latest payment amount using only existence logic.

Expected symptom:

- Required right-side attributes are unavailable.

Recovery:

Use a controlled join or windowed selection for attribute retrieval, then apply the appropriate existence logic.

## 7. Recovery

Anti-join incidents usually involve incorrect absence classification rather than row multiplication.

### Recovery sequence

1. Identify the affected run and rule version.
2. Capture the left and right snapshots.
3. Verify the exact definition of a qualifying match.
4. Check NULL behavior.
5. Check composite-key completeness.
6. Check tenant and partition scoping.
7. Check temporal validity.
8. Compare matched and anti-join populations.
9. Inspect source/reference freshness.
10. Correct the predicate or source data.
11. Re-run the anti-join.
12. Reconcile the total population.
13. Replay downstream handling idempotently.
14. Record the root cause.

### Recovering a NOT IN NULL incident

If an anti-join unexpectedly returns no records:

1. Check whether the right-side subquery contains NULL.
2. Reproduce the query with a small dataset.
3. Replace `NOT IN` with `NOT EXISTS` where appropriate.
4. Add regression coverage for NULL.
5. Recalculate the affected output.

### Recovering false matches

If records are incorrectly excluded because another entity satisfied the existence predicate:

1. Identify the incorrect matching key.
2. Compare tenant/business-key components.
3. Restore the complete predicate.
4. Re-run the affected partition.
5. Reconcile the anti-join count.

### Recovering reference lag

If records are temporarily unmatched because reference data has not arrived:

- Preserve the anti-join result.
- Record the reference-lag reason.
- Wait for the documented dependency.
- Reprocess the affected scope.
- Avoid repeatedly scanning unrelated historical data.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL supports `NOT EXISTS`, `EXISTS`, joins, indexes, and query planning for relational existence checks.

Learn to inspect the query plan when large anti-joins become expensive.

### 2. dbt

dbt can express relationship tests and custom models that identify orphaned or missing records.

Use anti-join models as explicit quality evidence rather than hiding them inside complex transformations.

### 3. DuckDB

DuckDB is useful for reproducing anti-join and NULL behavior locally against CSV and Parquet data.

It is especially useful for building minimal examples that demonstrate the `NOT IN` NULL trap.

## 9. Production Runbook

### Before deployment

- [ ] Define the left population.
- [ ] Define the qualifying match.
- [ ] Define the complete identity key.
- [ ] Decide how NULL keys are handled.
- [ ] Decide whether right-side duplicates are valid.
- [ ] Define temporal validity if applicable.
- [ ] Prefer `NOT EXISTS` for relational existence logic.
- [ ] Add population reconciliation.
- [ ] Add NULL-key tests.

### During execution

- [ ] Record left input rows.
- [ ] Record matched rows.
- [ ] Record anti-join rows.
- [ ] Record NULL-key rows.
- [ ] Record duplicate right-side keys where relevant.
- [ ] Record rule version.

### If anti-join count suddenly increases

1. Check source freshness.
2. Check reference freshness.
3. Check key normalization.
4. Check tenant scoping.
5. Check status/eligibility predicates.
6. Check temporal boundaries.
7. Compare newly unmatched keys with the previous successful run.

### If anti-join count suddenly drops

1. Verify the qualifying predicate.
2. Check whether a reference population expanded.
3. Check whether a broad match was accidentally introduced.
4. Check composite-key completeness.
5. Check whether NULL handling changed.

### If the query is slow

1. Inspect the query plan.
2. Check indexes on the correlated key.
3. Reduce unnecessary columns.
4. Restrict the left population early when appropriate.
5. Check right-side filtering and selectivity.
6. Verify that the query is not performing an unintended broad scan.

## 10. Common Mistakes

### Mistake 1 — Using NOT IN with nullable subqueries

This is the classic SQL NULL trap.

### Mistake 2 — Using an ordinary join for an existence question

Existence should usually be expressed as existence.

### Mistake 3 — Forgetting qualifying conditions

'No payment' and 'no successful payment' are different requirements.

### Mistake 4 — Forgetting composite-key components

Partial identity creates false matches.

### Mistake 5 — Using a nullable right-side marker

A real match can look like no match.

### Mistake 6 — Ignoring NULL left keys

A NULL identity can create legitimate anti-join output while also representing a source defect.

### Mistake 7 — Ignoring temporal validity

Current existence does not prove historical existence.

### Mistake 8 — Treating every unmatched record as bad

Some relationships are intentionally optional.

### Mistake 9 — Ignoring right-side duplicates

Existence semantics hide multiplication, but duplicate source data can still matter operationally.

### Mistake 10 — Using anti-join when right-side values are required

Use a value-producing join when attributes need to be retrieved.

### Mistake 11 — Forgetting reconciliation

Always prove that the retained and matched populations account for the source.

### Mistake 12 — Reprocessing everything

Use bounded replay when the affected population can be identified.

## 11. Definition of Done

The anti-join transformation is complete when you can:

- [ ] Define a qualifying match precisely.
- [ ] Explain anti-join semantics.
- [ ] Implement `NOT EXISTS` correctly.
- [ ] Explain the `NOT IN` NULL trap.
- [ ] Implement `LEFT JOIN ... IS NULL` safely.
- [ ] Choose a non-null existence marker.
- [ ] Handle composite keys.
- [ ] Handle tenant-scoped relationships.
- [ ] Handle temporal existence conditions.
- [ ] Explain why duplicates do not multiply `NOT EXISTS` results.
- [ ] Detect right-side quality problems separately.
- [ ] Test missing, qualifying, non-qualifying, duplicate, and NULL cases.
- [ ] Reconcile matched and anti-join populations.
- [ ] Observe reason-level absence metrics.
- [ ] Intentionally break the anti-join and diagnose it.
- [ ] Recover from NULL, predicate, scope, and reference-lag failures.
- [ ] Replay affected data safely.
- [ ] Explain how PostgreSQL, dbt, and DuckDB support production anti-join work.

## 12. What You Learned

An anti-join answers one of the most important relational questions in Data Engineering:

    DOES A QUALIFYING RELATED RECORD EXIST?

If the answer is no, the left record belongs in the anti-join population.

The production workflow is:

    DEFINE LEFT POPULATION
         ↓
    DEFINE QUALIFYING MATCH
         ↓
    DEFINE COMPLETE KEY
         ↓
    HANDLE NULL SEMANTICS
         ↓
    APPLY NOT EXISTS
         ↓
    CLASSIFY ABSENCE
         ↓
    RECONCILE POPULATIONS
         ↓
    OBSERVE TRENDS
         ↓
    REPLAY SAFELY IF REQUIRED

> **For relational absence, express the question as existence: return the left record only when no qualifying right record exists.**

### Next recipe

**T25 — Semi-Joins**