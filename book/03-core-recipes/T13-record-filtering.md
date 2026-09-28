# T13 — Record Filtering

> **Goal:** Select records for downstream processing using explicit, deterministic predicates without silently discarding data that may be required for audit, reconciliation, replay, or future processing.

Filtering looks simple:

    WHERE status = 'ACTIVE'

In production ETL, however, filtering is a data-disposition decision.

A record excluded from one downstream dataset may still be required for:

    audit
    reconciliation
    quarantine
    replay
    historical analysis
    another downstream consumer

The central rule is:

> **Filter for a declared downstream purpose, and make every exclusion explainable.**

---

## 1. Problem Recognition

### 1.1 Typical filtering problem

Suppose a source contains:

    ACTIVE
    CLOSED
    CANCELLED
    TEST
    UNKNOWN

A reporting model may require only:

    ACTIVE
    CLOSED

But filtering those records does not mean the other records are bad.

They may simply be outside this model's scope.

### 1.2 Filtering is not cleansing

Cleansing asks:

    Is this record valid?

Filtering asks:

    Should this valid or invalid record participate in this particular downstream operation?

Example:

    quantity = -5

may be invalid and therefore quarantined.

Example:

    status = CLOSED

may be perfectly valid but excluded from a current-active-customer model.

T12 teaches defect handling. T13 teaches **controlled record selection and disposition**.

### 1.3 Red flags

Investigate when:

- row counts suddenly fall
- a filter was added or changed
- a downstream report loses historical records
- analysts cannot explain why records disappeared
- filters depend on processing time instead of business time
- filtering occurs before required validation or enrichment
- different pipelines use slightly different inclusion rules
- records are deleted instead of excluded from a specific model.

---

## 2. Concept and Reasoning

### 2.1 Define the population first

Before writing a predicate, define:

    input population
    target population
    exclusion population
    business purpose
    effective time

Example:

    Input: all orders
    Target: completed orders for revenue reporting
    Excluded: pending/cancelled orders
    Time basis: order completion time

### 2.2 Filtering is a predicate

A filter should be expressible as a deterministic predicate:

    include(record) → TRUE / FALSE

Example:

    include = status == "COMPLETED"

More complex:

    include = (status == "COMPLETED")
              and (amount > 0)
              and (order_date >= reporting_start)

Keep the predicate explicit and testable.

### 2.3 Inclusion and exclusion rules

Prefer defining both:

    inclusion criteria
    exclusion criteria

Example:

    Include: account_status = ACTIVE
    Exclude: is_test_account = TRUE
    Exclude: deleted_at is not NULL

This makes the resulting population easier to explain.

### 2.4 Unknown values need an explicit policy

Consider:

    status = NULL

Do not automatically treat it as either:

    include

or:

    exclude

unless the contract defines that behavior.

For critical datasets, an unknown predicate result may be routed to quarantine or a separate review population.

### 2.5 SQL has three-valued logic

SQL predicates can evaluate to:

    TRUE
    FALSE
    UNKNOWN

Example:

    WHERE amount > 100

If `amount` is NULL, the predicate is UNKNOWN and the row is not returned by the WHERE clause.

This is different from explicitly defining a business rule such as:

    NULL amount → exclude

Know whether your filtering behavior comes from SQL semantics or an intentional business rule.

### 2.6 Filter after the required transformations

Sometimes filtering depends on standardized or cleansed data:

    raw
      ↓
    type conversion
      ↓
    normalization
      ↓
    standardization
      ↓
    validation
      ↓
    filtering

Do not filter on an unreliable raw representation when the business rule applies to the canonical value.

### 2.7 But do not over-transform before filtering

Filtering can also reduce expensive downstream work.

For example, if a source contains ten million rows and a cheap, trustworthy source predicate can reduce the population to one million, applying that predicate early may be appropriate.

The decision should consider:

    correctness
    cost
    predicate stability
    required fields
    pushdown behavior

### 2.8 Business time versus processing time

A common bug is:

    include records processed today

when the actual requirement is:

    include records whose business event occurred today

Define the time dimension explicitly.

Historical records arriving late should normally be evaluated using the business timestamp required by the model, not automatically by ingestion time.

### 2.9 Filtering should not destroy raw evidence

Prefer:

    raw → staged → filtered model

rather than:

    raw → delete excluded records

The source/staging layer should remain available according to retention policy so that an incorrect filter can be corrected and replayed.

### 2.10 Record filtering versus partition pruning

Do not confuse:

    business filtering

with:

    storage/query optimization

Partition pruning may reduce the amount of data physically scanned without changing the logical population.

Business filters determine which records belong in the result.

---

## 3. Implementation

## 3.1 Define a filter contract

Example:

| Rule | Predicate | Disposition |
|---|---|---|
| completed_order | status = COMPLETED | include |
| test_account | is_test = TRUE | exclude |
| deleted_order | deleted_at IS NOT NULL | exclude |
| unknown_status | status IS NULL | review |

Also define:

- time basis
- required fields
- NULL behavior
- precedence
- owner
- rule version
- whether exclusions need audit records.

## 3.2 Python predicate

    def include_order(record: dict) -> bool:
        if record.get("is_test") is True:
            return False

        if record.get("deleted_at") is not None:
            return False

        return record.get("status") == "COMPLETED"

This is simple, but the logic is explicit and testable.

## 3.3 Return a disposition instead of only a boolean

For important pipelines, return the reason:

    from dataclasses import dataclass

    @dataclass(frozen=True)
    class FilterResult:
        disposition: str
        reason: str

    def filter_order(record: dict) -> FilterResult:
        if record.get("is_test") is True:
            return FilterResult("excluded", "test_account")

        if record.get("deleted_at") is not None:
            return FilterResult("excluded", "deleted")

        if record.get("status") == "COMPLETED":
            return FilterResult("included", "completed")

        if record.get("status") is None:
            return FilterResult("review", "unknown_status")

        return FilterResult("excluded", "out_of_scope_status")

This produces operational evidence instead of a silent True/False decision.

## 3.4 Compose predicates

Keep individual rules understandable:

    def is_completed(record):
        return record.get("status") == "COMPLETED"

    def is_test(record):
        return record.get("is_test") is True

    def is_deleted(record):
        return record.get("deleted_at") is not None

    def include_order(record):
        return is_completed(record) and not is_test(record) and not is_deleted(record)

Composition makes unit testing easier.

## 3.5 SQL filtering

Example:

    SELECT
        order_id,
        customer_id,
        amount
    FROM standardized_orders
    WHERE status = 'COMPLETED'
      AND COALESCE(is_test, FALSE) = FALSE
      AND deleted_at IS NULL;

Be careful with `COALESCE`.

Using:

    COALESCE(is_test, FALSE)

means NULL is intentionally treated as FALSE.

That is a business decision, not merely a SQL convenience.

## 3.6 Make exclusions observable

Instead of only producing the included dataset, calculate dispositions:

    SELECT
        CASE
            WHEN is_test THEN 'test_account'
            WHEN deleted_at IS NOT NULL THEN 'deleted'
            WHEN status = 'COMPLETED' THEN 'included'
            WHEN status IS NULL THEN 'unknown_status'
            ELSE 'out_of_scope_status'
        END AS disposition,
        COUNT(*)
    FROM standardized_orders
    GROUP BY 1;

This provides a population reconciliation view.

## 3.7 Persist filtering metadata when required

For critical transformations, create an audit table:

    CREATE TABLE record_filter_audit (
        pipeline_run_id uuid NOT NULL,
        record_id text NOT NULL,
        rule_version text NOT NULL,
        disposition text NOT NULL,
        reason text NOT NULL,
        filtered_at timestamptz NOT NULL DEFAULT now()
    );

Do not persist every raw record automatically. Follow retention and privacy requirements.

## 3.8 Filter pushdown

If the source database can safely execute a predicate, push it down when it improves performance without changing semantics:

    SELECT ...
    FROM orders
    WHERE order_date >= $1
      AND order_date < $2;

Verify that the source column semantics, timezone, NULL behavior, and boundary conditions match the pipeline's contract.

## 3.9 Inclusive and exclusive boundaries

Prefer half-open intervals for time ranges:

    start <= event_time < end

Example:

    event_time >= '2026-09-01'
    AND event_time <  '2026-10-01'

This avoids overlapping adjacent windows.

---

## 4. Testing

Filtering tests must prove both inclusion and exclusion.

### 4.1 Included record

    def test_completed_order_is_included():
        result = filter_order({"status": "COMPLETED"})
        assert result.disposition == "included"

### 4.2 Excluded record

    def test_test_account_is_excluded():
        result = filter_order({"status": "COMPLETED", "is_test": True})
        assert result.disposition == "excluded"
        assert result.reason == "test_account"

### 4.3 Unknown values

    def test_unknown_status_requires_review():
        result = filter_order({"status": None})
        assert result.disposition == "review"

### 4.4 Precedence

Test records matching multiple rules.

Example:

    status = COMPLETED
    is_test = TRUE
    deleted_at = NULL

The contract must determine whether `test_account` or `completed` takes precedence.

### 4.5 Boundary tests

Test:

- exactly at the start timestamp
- exactly at the end timestamp
- one unit before the end
- NULL timestamps
- late-arriving records
- timezone boundaries.

### 4.6 SQL NULL semantics

Test explicitly:

    amount = NULL
    amount > 100

and verify whether the intended result is exclusion, review, or another disposition.

### 4.7 Population accounting

For a mutually exclusive disposition model:

    input_count = included + excluded + review

Test this invariant for every batch.

### 4.8 Idempotence

Filtering should be stable:

    filter(filter(record)) = filter(record)

More practically, running the same filter twice should produce the same disposition for the same input.

---

## 5. Observability

Monitor:

| Signal | Why it matters |
|---|---|
| input count | Population baseline |
| included count | Downstream population |
| excluded count | Scope/disposition volume |
| review count | Unknown/ambiguous records |
| exclusion rate | Detects unexpected population changes |
| exclusion reason | Identifies the active rule |
| rule version | Explains behavior |
| business-time distribution | Detects date-window errors |

Important alerts include:

- sudden inclusion-rate changes
- sudden exclusion-rate changes
- unexpected new exclusion reason
- review population above threshold
- zero included records where data is expected.

Do not alert only on job success. A pipeline can succeed technically while filtering out the entire business population.

---

## 6. Intentional Failure

### Failure A — Wrong status filter

Change:

    status = COMPLETED

to:

    status = SETTLED

when the model expects completed orders.

Expected result:

- included population changes
- reconciliation detects the difference
- distribution monitoring identifies the change.

### Failure B — NULL treated incorrectly

Use:

    COALESCE(status, 'COMPLETED')

without a contract allowing that behavior.

Expected result:

- NULL records become incorrectly included
- tests should fail.

### Failure C — Processing time instead of event time

Filter on:

    ingestion_date = today

when the requirement is:

    order_date = today

Expected result:

- late-arriving historical records are misclassified.

### Failure D — Boundary overlap

Use:

    start <= event_time <= end

for adjacent daily windows.

Expected result:

- the boundary record appears in both windows.

### Failure E — Silent deletion

Delete excluded records from the only persisted dataset.

Expected result:

- replay becomes impossible
- audit evidence is lost.

---

## 7. Recovery

### Scenario 1 — Filter excluded valid records

1. Identify the filter rule and version.
2. Find the first affected run.
3. Compare included/excluded distributions with the previous healthy run.
4. Determine whether the filter or source data changed.
5. Correct the predicate.
6. Reprocess from preserved raw/staged data.
7. Reconcile record counts and business measures.
8. Verify downstream consumers before closing the incident.

### Scenario 2 — Filter included invalid/out-of-scope records

1. Identify the incorrectly included population.
2. Determine the predicate defect.
3. Stop downstream publication if necessary.
4. Correct the filter.
5. Rebuild affected partitions/models.
6. Reconcile totals.
7. Record the incident and add a regression test.

### Scenario 3 — Boundary window error

1. Identify affected time windows.
2. Determine whether the interval should be `[start, end)`.
3. Reprocess overlapping or missing boundary records.
4. Reconcile adjacent windows.
5. Add boundary tests.

---

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

Useful for predicate pushdown, SQL filtering, population accounting, constraints, and audit queries.

The underlying mechanism remains a deterministic predicate with explicit NULL and boundary semantics.

### 8.2 Python

Useful for complex predicates, reusable filter functions, unit tests, and source-specific business logic.

Keep predicates small and composable.

### 8.3 dbt

Useful for version-controlled SQL models, tests, documentation, and controlled analytical filtering.

Filtering logic should remain visible in model SQL rather than hidden in opaque macros or undocumented application code.

---

## 9. Production Runbook

### Before deployment

- [ ] Input population is defined.
- [ ] Target population is defined.
- [ ] Inclusion rules are documented.
- [ ] Exclusion rules are documented.
- [ ] NULL behavior is explicit.
- [ ] Business-time basis is explicit.
- [ ] Boundary semantics are defined.
- [ ] Rule version is tracked.
- [ ] Population accounting is implemented.

### During execution

- [ ] Monitor inclusion rate.
- [ ] Monitor exclusion rate.
- [ ] Monitor review population.
- [ ] Monitor exclusion reasons.
- [ ] Monitor business-time distributions.
- [ ] Compare with recent healthy runs.

### When the population changes unexpectedly

1. Identify the first changed run.
2. Compare predicate version.
3. Compare source distributions.
4. Inspect NULL and boundary behavior.
5. Check upstream changes.
6. Determine whether the change is expected.
7. Correct the filter if necessary.
8. Replay affected data.
9. Reconcile before downstream publication.

---

## 10. Common Mistakes

### Mistake 1 — Treating excluded as invalid

Out-of-scope does not necessarily mean bad data.

### Mistake 2 — Silent row loss

Every important exclusion should be explainable.

### Mistake 3 — Ignoring SQL three-valued logic

NULL predicates can behave differently from application-language booleans.

### Mistake 4 — Using processing time

Business filters frequently depend on event/business time.

### Mistake 5 — Inclusive adjacent windows

Use half-open intervals to prevent overlap.

### Mistake 6 — Filtering before required semantic transformations

Do not evaluate canonical business rules against unreliable source representations.

### Mistake 7 — Hard-coding filters in many pipelines

Duplicate predicates eventually diverge.

### Mistake 8 — Deleting source data

Filtering a downstream model should not destroy the evidence required for replay.

---

## 11. Definition of Done

A production-grade filtering implementation is complete when you can:

- [ ] Define the target population precisely.
- [ ] Express inclusion and exclusion as explicit predicates.
- [ ] Explain NULL behavior.
- [ ] Define business-time semantics.
- [ ] Define boundary behavior.
- [ ] Distinguish exclusion from invalidity.
- [ ] Produce explainable dispositions.
- [ ] Implement filtering in Python or SQL.
- [ ] Use safe predicate pushdown where appropriate.
- [ ] Test inclusion and exclusion.
- [ ] Test precedence and NULL behavior.
- [ ] Test time boundaries.
- [ ] Prove population accounting.
- [ ] Observe population and exclusion changes.
- [ ] Intentionally break the filter.
- [ ] Diagnose the resulting population change.
- [ ] Recover through replay from preserved data.
- [ ] Explain the relevant production tools.
- [ ] Operate the mechanism using the runbook.

---

## 12. What You Learned

Record filtering is not merely a WHERE clause.

It defines which records participate in a specific downstream purpose.

The production reasoning pattern is:

    define population
          ↓
    define inclusion/exclusion rules
          ↓
    define NULL and time semantics
          ↓
    apply deterministic predicate
          ↓
    account for every disposition
          ↓
    observe population changes
          ↓
    replay safely when the rule changes

A technically successful pipeline can still be wrong if its filter silently removes valid business data. Production-grade filtering therefore requires explicit semantics, explainable dispositions, population accounting, tests, observability, and recoverability.

---

## Next Recipe

**T14 — Record Enrichment**