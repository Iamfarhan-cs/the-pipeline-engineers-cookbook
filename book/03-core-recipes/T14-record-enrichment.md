# T14 — Record Enrichment

> **Goal:** Add trusted attributes or context to records using explicit enrichment sources, controlled joins, correct cardinality, temporal semantics, and observable match behavior.

Enrichment means adding information that was not present in the original record.

Examples:

    order → customer_segment
    payment → merchant_country
    transaction → FX_rate
    event → device_metadata
    account → risk_category

The dangerous part is that an enrichment can look correct while silently attaching the wrong information.

The central rule is:

> **An enrichment is correct only when the source, join key, cardinality, and time semantics are correct.**

---

## 1. Problem Recognition

### 1.1 Typical enrichment problem

Suppose an order contains:

    order_id
    customer_id
    amount

and the customer dimension contains:

    customer_id
    customer_segment
    country

The downstream model needs the segment and country.

That requires an enrichment operation:

    orders + customer_dimension → enriched_orders

The challenge is not writing the JOIN.

The challenge is proving that exactly the intended customer record was attached.

### 1.2 Enrichment is not mapping

Mapping transforms a value:

    P → PENDING

Enrichment adds information from another source:

    customer_id = 123 → customer_segment = PREMIUM

T10 teaches code/status mapping. T14 teaches **adding trusted context from another dataset or service**.

### 1.3 Enrichment is not aggregation

Aggregation reduces multiple records into a summary:

    transactions → daily_total

Enrichment normally preserves the original grain:

    transaction → transaction + customer_attributes

If an enrichment unexpectedly changes row count, investigate the join cardinality.

### 1.4 Red flags

Investigate when:

- row count increases after enrichment
- one source record matches multiple enrichment records
- match rate suddenly falls
- NULL enrichment values spike
- enrichment values change for historical records
- a join uses a non-unique key
- current dimension data is applied to historical events
- analysts cannot explain which source produced an attribute.

---

## 2. Concept and Reasoning

### 2.1 Define the enrichment contract

Before implementing the join, define:

    input grain
    enrichment source
    join key
    expected cardinality
    required/optional match
    temporal rule
    conflict policy
    provenance
    freshness requirement

Example:

| Property | Contract |
|---|---|
| Input grain | one order |
| Source | customer dimension |
| Join key | customer_id |
| Cardinality | many orders → one customer |
| Match required | yes |
| Historical behavior | use record effective at order time |
| Missing match | quarantine |

### 2.2 Cardinality is the first safety check

Suppose the input contains 1,000 orders.

A correct many-to-one enrichment should normally produce:

    1,000 input rows
    1,000 enriched rows

If the result contains 1,250 rows, the enrichment source probably contains duplicate keys or the join condition is too broad.

Common relationships:

    many-to-one
    one-to-one
    one-to-many
    many-to-many

Know which one the model expects before writing SQL.

### 2.3 Validate enrichment keys

An enrichment key should have a defined uniqueness contract.

Example:

    customer_id → exactly one current customer record

Test this independently:

    SELECT customer_id, COUNT(*)
    FROM customer_dimension
    GROUP BY customer_id
    HAVING COUNT(*) > 1;

Duplicates should be handled explicitly, not accidentally resolved with `DISTINCT`.

### 2.4 Missing matches need a policy

A source record may have no enrichment match.

Possible policies:

    reject
    quarantine
    preserve with NULL
    use an explicit UNKNOWN dimension
    retry later

The correct policy depends on whether the enrichment is required for downstream correctness.

### 2.5 Current versus historical enrichment

Consider a customer whose segment changes:

    2026-01-01 → STANDARD
    2026-07-01 → PREMIUM

An order from March should not automatically receive the customer's September/current segment if the business model requires historical accuracy.

Temporal enrichment requires:

    input_event_time
          ↓
    dimension effective interval
          ↓
    matching version

### 2.6 Effective-dated enrichment

Use half-open intervals:

    effective_from <= event_time < effective_to

Example:

    customer_id | segment  | from       | to
    123         | STANDARD | 2026-01-01 | 2026-07-01
    123         | PREMIUM  | 2026-07-01 | NULL

An order on 2026-06-30 gets STANDARD.

An order on 2026-07-01 gets PREMIUM.

### 2.7 Snapshot enrichment

Sometimes the contract explicitly requires the current state.

Example:

    Add current account risk category to today's operational dashboard.

In that case, current enrichment may be correct.

The important point is that “current” versus “historical” must be intentional.

### 2.8 Conflicting enrichment sources

Two systems may provide different values.

Example:

    CRM country = US
    Billing country = CA

Do not silently choose whichever source appears first.

Define:

    source precedence
    conflict detection
    ownership
    reconciliation process.

### 2.9 Preserve enrichment provenance

Useful metadata includes:

    enrichment_source
    enrichment_version
    matched_key
    matched_effective_time
    enrichment_run_id

This makes downstream values explainable.

### 2.10 Freshness is part of correctness

An enrichment source can be available but stale.

Example:

    customer risk data last updated 3 days ago

Availability does not prove freshness.

Define an acceptable freshness threshold where the business requires it.

---

## 3. Implementation

## 3.1 Example datasets

Input:

    orders

    order_id | customer_id | amount | order_time

Enrichment source:

    customer_dimension

    customer_id | segment | country

Desired output:

    order_id | customer_id | amount | segment | country

## 3.2 Basic PostgreSQL enrichment

    SELECT
        o.order_id,
        o.customer_id,
        o.amount,
        c.segment,
        c.country
    FROM orders AS o
    LEFT JOIN customer_dimension AS c
      ON c.customer_id = o.customer_id;

`LEFT JOIN` makes missing matches visible instead of silently removing the input record.

Use `INNER JOIN` only when the contract explicitly says unmatched records should be excluded.

## 3.3 Detect duplicate enrichment keys

    SELECT
        customer_id,
        COUNT(*) AS row_count
    FROM customer_dimension
    GROUP BY customer_id
    HAVING COUNT(*) > 1;

If this returns rows where uniqueness is required, stop before publishing the enriched dataset.

Do not “fix” the issue with:

    SELECT DISTINCT ...

unless deduplication itself is a defined and validated business rule.

## 3.4 Enforce cardinality

Before and after the join:

    SELECT COUNT(*) FROM orders;

    SELECT COUNT(*)
    FROM orders AS o
    LEFT JOIN customer_dimension AS c
      ON c.customer_id = o.customer_id;

For a many-to-one enrichment, these counts should normally match.

Also check input key multiplicity and matched/unmatched counts.

## 3.5 Handle missing matches

    SELECT
        o.order_id,
        o.customer_id,
        c.segment,
        CASE
            WHEN c.customer_id IS NULL THEN 'UNMATCHED'
            ELSE 'MATCHED'
        END AS enrichment_status
    FROM orders AS o
    LEFT JOIN customer_dimension AS c
      ON c.customer_id = o.customer_id;

Do not convert an unmatched record into a fake customer unless an explicit UNKNOWN dimension is part of the model.

## 3.6 Historical enrichment

Suppose the dimension contains:

    customer_id
    segment
    effective_from
    effective_to

Use:

    SELECT
        o.order_id,
        o.customer_id,
        o.order_time,
        c.segment
    FROM orders AS o
    LEFT JOIN customer_segment_history AS c
      ON c.customer_id = o.customer_id
     AND c.effective_from <= o.order_time
     AND (c.effective_to IS NULL OR o.order_time < c.effective_to);

Validate that exactly one dimension row can match each input record.

## 3.7 Prevent overlapping dimension records

Two effective-dated records for the same key must not overlap when the model expects a single applicable record.

Conceptually:

    customer_id = 123
    STANDARD: [2026-01-01, 2026-07-01)
    PREMIUM:  [2026-07-01, 2026-10-01)

is valid.

But:

    STANDARD: [2026-01-01, 2026-08-01)
    PREMIUM:  [2026-07-01, 2026-10-01)

creates an ambiguous period.

Detect overlaps before enrichment.

## 3.8 Python enrichment

Small in-memory reference data can be represented explicitly:

    customer_by_id = {
        101: {"segment": "PREMIUM", "country": "US"},
        102: {"segment": "STANDARD", "country": "GB"},
    }

    def enrich_order(order: dict) -> dict:
        customer = customer_by_id.get(order["customer_id"])

        if customer is None:
            raise ValueError(
                f"customer not found: {order['customer_id']}"
            )

        return {
            **order,
            "customer_segment": customer["segment"],
            "customer_country": customer["country"],
        }

For large reference datasets, use an appropriate database join or bounded lookup strategy rather than loading everything into memory.

## 3.9 Enrichment provenance

    def enrich_order(order: dict, customer: dict) -> dict:
        return {
            **order,
            "customer_segment": customer["segment"],
            "customer_country": customer["country"],
            "enrichment_source": "customer_dimension",
            "enrichment_version": customer["version"],
        }

Do not add provenance fields blindly to every dataset. Include what the audit and operational contract requires.

## 3.10 Enrichment freshness

Example freshness check:

    SELECT MAX(updated_at) AS latest_update
    FROM customer_dimension;

Compare the result against the pipeline's freshness threshold before using the dimension.

---

## 4. Testing

### 4.1 Successful match

    def test_order_is_enriched():
        order = {"order_id": 1, "customer_id": 101, "amount": 50}
        customer = {"segment": "PREMIUM", "country": "US", "version": "v1"}

        result = enrich_order(order, customer)

        assert result["customer_segment"] == "PREMIUM"
        assert result["customer_country"] == "US"

### 4.2 Missing match

Test that an absent enrichment record follows the documented disposition.

### 4.3 Duplicate key

Test that duplicate dimension keys fail validation rather than being silently selected.

### 4.4 Cardinality

Prove:

    output_count == input_count

for a many-to-one enrichment.

Also test a deliberately one-to-many source to ensure the pipeline detects unexpected row multiplication.

### 4.5 Historical correctness

Test events:

    before effective_from
    exactly at effective_from
    exactly at effective_to
    after effective_to

Verify the correct dimension version is selected.

### 4.6 No-match and multi-match accounting

Track:

    input_count
    matched_count
    unmatched_count
    multi_match_count

Validate:

    input_count = matched_count + unmatched_count

when multi-match records are rejected before final enrichment.

### 4.7 Freshness

Test:

- fresh dimension
- stale dimension
- missing freshness timestamp
- future-dated update if invalid.

### 4.8 Idempotence

Running the same enrichment against the same source versions should produce the same output.

Do not allow repeated enrichment to create duplicated columns or change values unexpectedly.

---

## 5. Observability

Monitor:

| Signal | Why it matters |
|---|---|
| input count | Population baseline |
| matched count | Successful enrichment |
| unmatched count | Missing reference data |
| match rate | Source/key health |
| multi-match count | Cardinality violation |
| output count | Detects row multiplication |
| source freshness | Detects stale enrichment |
| enrichment version | Explains attached values |
| conflict count | Detects source disagreement |

Useful structured fields:

    pipeline_run_id
    input_dataset
    enrichment_dataset
    enrichment_version
    match_status
    rule_version

Alert on:

- sudden match-rate decline
- unexpected row multiplication
- multi-match records
- stale enrichment source
- unexpected version change.

Do not log sensitive enriched attributes merely to prove a match. Use safe keys or controlled identifiers.

---

## 6. Intentional Failure

### Failure A — Duplicate enrichment key

Insert two current customer records with the same `customer_id`.

Expected result:

- uniqueness validation fails
- enrichment does not arbitrarily choose one.

### Failure B — INNER JOIN loses input records

Replace a `LEFT JOIN` with an `INNER JOIN` when unmatched records must remain visible.

Expected result:

- output count drops
- unmatched count becomes invisible.

### Failure C — Historical enrichment uses current data

Use today's customer segment for an old transaction.

Expected result:

- historical test fails
- reconciliation identifies changed historical values.

### Failure D — Join key is too broad

Join only on:

    customer_country

instead of the intended customer identifier.

Expected result:

- row count may multiply
- unrelated attributes become attached.

### Failure E — Stale enrichment source

Use a dimension older than the allowed freshness threshold.

Expected result:

- freshness validation blocks or flags the enrichment.

---

## 7. Recovery

### Scenario 1 — Duplicate enrichment records

1. Stop publication if cardinality is unsafe.
2. Identify the duplicate key and effective period.
3. Determine the correct source record.
4. Fix the upstream/reference data.
5. Revalidate uniqueness.
6. Re-run enrichment.
7. Reconcile output counts and affected values.

### Scenario 2 — Wrong historical enrichment

1. Identify affected event-time range.
2. Determine the enrichment version used.
3. Identify the correct historical dimension records.
4. Rebuild affected records from preserved input data.
5. Re-run temporal enrichment.
6. Compare old and corrected attributes.
7. Reconcile downstream models.

### Scenario 3 — Match rate collapse

1. Identify the first run with reduced matches.
2. Compare key distributions between input and enrichment source.
3. Check source schema and identifier changes.
4. Check source freshness.
5. Determine whether the key mapping changed.
6. Correct the source or enrichment logic.
7. Replay affected data.
8. Monitor match rate after recovery.

---

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

Useful for joins, uniqueness constraints, temporal lookups, cardinality checks, and freshness queries.

The underlying mechanism remains a controlled relationship between an input record and an enrichment source.

### 8.2 Python

Useful for small reference lookups, complex enrichment functions, deterministic tests, and API-backed enrichment adapters.

Use bounded caches or database joins for large reference datasets rather than unbounded in-memory dictionaries.

### 8.3 dbt

Useful for version-controlled enrichment models, relationship tests, uniqueness tests, freshness checks, and documented analytical transformations.

Tests should prove the relationship rather than merely verify that a query executes.

---

## 9. Production Runbook

### Before deployment

- [ ] Input grain is documented.
- [ ] Enrichment source is documented.
- [ ] Join key is documented.
- [ ] Expected cardinality is documented.
- [ ] Key uniqueness is tested.
- [ ] Missing-match policy is documented.
- [ ] Historical/current semantics are explicit.
- [ ] Freshness requirement is defined.
- [ ] Provenance requirements are defined.

### During execution

- [ ] Monitor match rate.
- [ ] Monitor unmatched records.
- [ ] Monitor multi-match records.
- [ ] Monitor output/input row ratio.
- [ ] Monitor source freshness.
- [ ] Monitor enrichment version.

### When enrichment changes unexpectedly

1. Identify the first affected run.
2. Compare input and enrichment key distributions.
3. Check uniqueness and temporal overlap.
4. Check freshness.
5. Check enrichment version.
6. Determine whether the source or join changed.
7. Correct the issue.
8. Replay affected records.
9. Reconcile before downstream publication.

---

## 10. Common Mistakes

### Mistake 1 — Joining on a non-unique key

A syntactically valid join can produce semantically incorrect row multiplication.

### Mistake 2 — Using DISTINCT to hide cardinality problems

Duplicate enrichment records should be investigated, not masked.

### Mistake 3 — Ignoring missing matches

NULL enrichment values can represent a source defect, late arrival, or valid absence.

### Mistake 4 — Applying current dimensions to historical events

Historical models require event-time correctness when attributes are time-dependent.

### Mistake 5 — Ignoring freshness

A stale source can produce incorrect enrichment even when every join succeeds.

### Mistake 6 — Using an INNER JOIN by habit

INNER JOIN can silently remove input records.

### Mistake 7 — Ignoring provenance

Downstream users should be able to determine where important enriched attributes came from.

### Mistake 8 — Unbounded API enrichment

Calling an external service once per record can create latency, rate-limit, and consistency problems.

---

## 11. Definition of Done

A production-grade enrichment implementation is complete when you can:

- [ ] Define input grain and enrichment purpose.
- [ ] Identify the enrichment source.
- [ ] Define the join key.
- [ ] Prove expected cardinality.
- [ ] Validate key uniqueness.
- [ ] Define missing-match behavior.
- [ ] Handle historical versus current enrichment explicitly.
- [ ] Validate effective-date boundaries where applicable.
- [ ] Check source freshness.
- [ ] Preserve required enrichment provenance.
- [ ] Implement the enrichment deterministically.
- [ ] Test missing, duplicate, and multi-match cases.
- [ ] Prove row-count behavior.
- [ ] Observe match rate and source health.
- [ ] Intentionally break the join.
- [ ] Diagnose the resulting evidence.
- [ ] Recover and replay safely.
- [ ] Explain the relevant production tools.
- [ ] Operate the mechanism using the runbook.

---

## 12. What You Learned

Record enrichment is not just a JOIN.

It is the controlled attachment of trusted context to an existing record grain.

The production reasoning pattern is:

    define input grain
          ↓
    identify trusted enrichment source
          ↓
    validate key uniqueness/cardinality
          ↓
    apply current or historical temporal rule
          ↓
    account for matches and misses
          ↓
    validate output grain
          ↓
    observe freshness and provenance
          ↓
    replay safely when the source or rule changes

The most dangerous enrichment failures are often silent: the query succeeds, the columns are populated, and the values are wrong. Production-grade enrichment therefore requires relationship validation, temporal correctness, freshness, provenance, and explicit recovery.

---

## Next Recipe

**T15 — Record Splitting**