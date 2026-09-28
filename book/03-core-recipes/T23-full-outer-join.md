# T23 — FULL OUTER JOIN

> **Goal:** Learn how to combine two populations while preserving matched records and unmatched records from both sides, with explicit reconciliation categories, cardinality controls, and duplicate detection.

## 1. Problem Recognition

A FULL OUTER JOIN is appropriate when both input populations matter and neither side should disappear merely because a match is missing.

Typical production uses include:

- Reconciling source and target populations.
- Comparing two snapshots of entities.
- Finding records present in one system but missing from another.
- Comparing reference datasets from different providers.
- Building migration validation reports.
- Reconciling two independently maintained systems.
- Identifying matched, left-only, and right-only populations in one result.

The defining behavior is:

    LEFT + RIGHT MATCH     → MATCHED ROW
    LEFT WITHOUT RIGHT     → LEFT-ONLY ROW + NULL RIGHT
    RIGHT WITHOUT LEFT     → RIGHT-ONLY ROW + NULL LEFT

A FULL OUTER JOIN therefore exposes discrepancies on both sides.

Common production failures include:

- Treating a FULL JOIN as a simple UNION.
- Missing duplicate keys and producing row multiplication.
- Comparing records with incompatible keys.
- Treating NULL-extended columns as source NULLs.
- Using `COALESCE` before determining which side actually existed.
- Losing track of left-only and right-only populations.
- Using the join for reconciliation without defining business equivalence.
- Joining on an incomplete composite key.
- Ignoring historical or snapshot timing.

### Recognition questions

Before using a FULL OUTER JOIN, ask:

1. What population does each side represent?
2. What makes a record equivalent across the two systems?
3. Which keys define identity?
4. What cardinality is expected?
5. What does a left-only record mean?
6. What does a right-only record mean?
7. What does a matched record mean?
8. How should conflicting values be classified?
9. How will duplicates be handled?
10. Which snapshot or business time is being compared?

## 2. Concept and Reasoning

### 2.1 FULL OUTER JOIN semantics

Given relations `A` and `B`, a FULL OUTER JOIN returns:

- Every matching pair.
- Every unmatched row from A.
- Every unmatched row from B.

Conceptually:

    A FULL OUTER JOIN B

preserves the union of both populations at the join relationship.

### 2.2 Three fundamental reconciliation categories

For a unique-key reconciliation, every output row can be classified as:

| Category | Left exists | Right exists | Meaning |
|---|---:|---:|---|
| MATCHED | Yes | Yes | Same business key exists on both sides |
| LEFT_ONLY | Yes | No | Exists only in left population |
| RIGHT_ONLY | No | Yes | Exists only in right population |

This classification is one of the most valuable uses of FULL OUTER JOIN.

### 2.3 FULL JOIN is not UNION

`UNION` combines rows vertically.

A FULL OUTER JOIN combines related rows horizontally when their keys match.

Example:

    source A: customer_id, name
    source B: customer_id, status

A FULL JOIN can produce:

    customer_id | name | status

whereas UNION requires compatible column structures and represents separate rows.

Use the operation that matches the business relationship.

### 2.4 FULL JOIN is not automatically a reconciliation

A FULL JOIN only establishes row relationships according to the join predicate.

It does not determine whether matched records are actually equivalent.

Two rows can share a customer ID while having:

    different name
    different address
    different status

Therefore reconciliation usually has two stages:

    identity matching
         ↓
    attribute comparison

Do not call every matched key a fully reconciled record.

### 2.5 Cardinality matters

For a unique key on both sides:

    one left row + one right row → one output row

If the left side has three rows for key `A` and the right side has four:

    3 × 4 = 12 matched output rows

This can overwhelm a reconciliation result.

Before trusting a FULL JOIN, validate the expected uniqueness of the comparison key.

### 2.6 Duplicate keys

Duplicate keys can have different meanings:

- Bad source duplication.
- Multiple legitimate versions.
- Multiple events sharing an entity identifier.
- Different records at different effective times.

Do not blindly deduplicate them.

First define the comparison grain.

If the reconciliation is one row per customer, each side must be reduced to one comparable customer record before joining.

### 2.7 Attribute comparison

After identity matching, compare business attributes explicitly.

Example:

    source_name = target_name
    source_status = target_status
    source_balance = target_balance

Classify:

    MATCHED_EQUAL
    MATCHED_DIFFERENT
    LEFT_ONLY
    RIGHT_ONLY

This is more useful than one generic matched/unmatched status.

### 2.8 NULL-aware attribute comparison

Ordinary SQL equality can produce UNKNOWN when either value is NULL.

For reconciliation, NULL may need to be compared as a meaningful state.

PostgreSQL supports:

    IS NOT DISTINCT FROM

which treats two NULLs as equal for comparison purposes.

Example:

    source.name IS NOT DISTINCT FROM target.name

Use null-safe comparison when the reconciliation contract defines two missing values as equivalent.

### 2.9 Composite identity

If identity is:

    tenant_id + customer_id

the FULL JOIN must use both:

    ON a.tenant_id = b.tenant_id
   AND a.customer_id = b.customer_id

Otherwise records from different tenants may be falsely classified as matched.

### 2.10 NULL join keys

Ordinary equality does not match NULL to NULL.

Therefore two rows with NULL keys will normally appear as separate unmatched records.

This can be correct because NULL usually means unknown identity.

If the business defines a different null-safe identity rule, implement that rule explicitly and test it.

### 2.11 Snapshot timing

Reconciliation is meaningful only when the populations are comparable.

Record:

    left_snapshot_at
    right_snapshot_at
    business_date
    extraction_run_id

Otherwise a right-only record may simply be newer than the left snapshot.

### 2.12 FULL JOIN and temporal data

Historical reconciliation may require comparing records valid at the same business time.

If each side contains effective-dated records, first establish the comparison grain and valid interval before joining.

Do not compare current state on one side with historical state on the other and call the result a data-quality difference.

### 2.13 FULL JOIN for migration validation

A common migration pattern is:

    old_system FULL JOIN new_system

using a stable business key.

Then classify:

    MATCHED_EQUAL
    MATCHED_DIFFERENT
    LEFT_ONLY
    RIGHT_ONLY

This provides a structured migration reconciliation report.

### 2.14 FULL JOIN and COALESCE

After classification, `COALESCE` can provide a convenient display value:

    COALESCE(a.customer_id, b.customer_id)

However, do not use COALESCE as a substitute for existence classification.

First determine whether the row is left-only, right-only, or matched.

Otherwise the output may hide which system actually contained the record.

## 3. Implementation

### 3.1 Define the reconciliation contract

Example:

    Left input:       legacy_customers
    Right input:      new_customers
    Grain:            one row per tenant + customer
    Join key:         tenant_id + customer_id
    Expected cardinality: one-to-one
    Output categories:
        MATCHED_EQUAL
        MATCHED_DIFFERENT
        LEFT_ONLY
        RIGHT_ONLY
    Duplicate keys: fail before reconciliation
    Snapshot:         same business date

### 3.2 PostgreSQL schema

    CREATE TABLE legacy_customers (
        tenant_id   BIGINT NOT NULL,
        customer_id BIGINT NOT NULL,
        name        TEXT,
        status      TEXT,
        PRIMARY KEY (tenant_id, customer_id)
    );

    CREATE TABLE new_customers (
        tenant_id   BIGINT NOT NULL,
        customer_id BIGINT NOT NULL,
        name        TEXT,
        status      TEXT,
        PRIMARY KEY (tenant_id, customer_id)
    );

### 3.3 Basic FULL OUTER JOIN

    SELECT
        COALESCE(l.tenant_id, n.tenant_id) AS tenant_id,
        COALESCE(l.customer_id, n.customer_id) AS customer_id,
        l.name AS legacy_name,
        n.name AS new_name,
        l.status AS legacy_status,
        n.status AS new_status
    FROM legacy_customers AS l
    FULL OUTER JOIN new_customers AS n
      ON n.tenant_id = l.tenant_id
     AND n.customer_id = l.customer_id;

### 3.4 Classify row existence

Use non-null identity columns to determine which side exists:

    SELECT
        COALESCE(l.tenant_id, n.tenant_id) AS tenant_id,
        COALESCE(l.customer_id, n.customer_id) AS customer_id,
        CASE
            WHEN l.customer_id IS NOT NULL
             AND n.customer_id IS NOT NULL THEN 'MATCHED'
            WHEN l.customer_id IS NOT NULL THEN 'LEFT_ONLY'
            ELSE 'RIGHT_ONLY'
        END AS presence_status
    FROM legacy_customers AS l
    FULL OUTER JOIN new_customers AS n
      ON n.tenant_id = l.tenant_id
     AND n.customer_id = l.customer_id;

Do not use a nullable business attribute as the existence marker.

### 3.5 Compare attributes

    SELECT
        COALESCE(l.tenant_id, n.tenant_id) AS tenant_id,
        COALESCE(l.customer_id, n.customer_id) AS customer_id,
        CASE
            WHEN l.customer_id IS NULL THEN 'RIGHT_ONLY'
            WHEN n.customer_id IS NULL THEN 'LEFT_ONLY'
            WHEN l.name IS NOT DISTINCT FROM n.name
             AND l.status IS NOT DISTINCT FROM n.status
                THEN 'MATCHED_EQUAL'
            ELSE 'MATCHED_DIFFERENT'
        END AS reconciliation_status
    FROM legacy_customers AS l
    FULL OUTER JOIN new_customers AS n
      ON n.tenant_id = l.tenant_id
     AND n.customer_id = l.customer_id;

### 3.6 Produce attribute-level differences

    SELECT
        COALESCE(l.customer_id, n.customer_id) AS customer_id,
        l.name AS legacy_name,
        n.name AS new_name,
        l.status AS legacy_status,
        n.status AS new_status,
        (l.name IS NOT DISTINCT FROM n.name) AS name_equal,
        (l.status IS NOT DISTINCT FROM n.status) AS status_equal
    FROM legacy_customers AS l
    FULL OUTER JOIN new_customers AS n
      ON n.customer_id = l.customer_id;

This makes the difference evidence explicit.

### 3.7 Validate uniqueness before joining

    SELECT
        tenant_id,
        customer_id,
        COUNT(*) AS row_count
    FROM legacy_customer_stage
    GROUP BY tenant_id, customer_id
    HAVING COUNT(*) > 1;

Repeat for the right side.

If the reconciliation contract is one row per customer, stop if either side violates uniqueness.

### 3.8 Build reconciliation counts

    WITH reconciliation AS (
        SELECT
            CASE
                WHEN l.customer_id IS NOT NULL
                 AND n.customer_id IS NOT NULL THEN 'MATCHED'
                WHEN l.customer_id IS NOT NULL THEN 'LEFT_ONLY'
                ELSE 'RIGHT_ONLY'
            END AS presence_status
        FROM legacy_customers AS l
        FULL OUTER JOIN new_customers AS n
          ON n.tenant_id = l.tenant_id
         AND n.customer_id = l.customer_id
    )
    SELECT
        presence_status,
        COUNT(*) AS row_count
    FROM reconciliation
    GROUP BY presence_status;

This produces a compact population reconciliation.

### 3.9 Python implementation

    def full_outer_reconcile(left_rows, right_rows):
        left_index = {}
        right_index = {}

        for row in left_rows:
            key = (row["tenant_id"], row["customer_id"])
            if key in left_index:
                raise ValueError(f"duplicate left key: {key!r}")
            left_index[key] = row

        for row in right_rows:
            key = (row["tenant_id"], row["customer_id"])
            if key in right_index:
                raise ValueError(f"duplicate right key: {key!r}")
            right_index[key] = row

        result = []
        for key in sorted(set(left_index) | set(right_index)):
            left = left_index.get(key)
            right = right_index.get(key)

            if left is not None and right is not None:
                status = "MATCHED"
            elif left is not None:
                status = "LEFT_ONLY"
            else:
                status = "RIGHT_ONLY"

            result.append({
                "tenant_id": key[0],
                "customer_id": key[1],
                "left": left,
                "right": right,
                "presence_status": status,
            })

        return result

The explicit indexes make uniqueness assumptions visible.

### 3.10 Attribute comparison in Python

    def null_safe_equal(left_value, right_value):
        return left_value == right_value

    def classify_pair(left, right):
        if left is None:
            return "RIGHT_ONLY"
        if right is None:
            return "LEFT_ONLY"

        if (
            null_safe_equal(left.get("name"), right.get("name"))
            and null_safe_equal(left.get("status"), right.get("status"))
        ):
            return "MATCHED_EQUAL"

        return "MATCHED_DIFFERENT"

Use explicit field-level comparison rules rather than comparing serialized records blindly.

### 3.11 Snapshot metadata

Persist or emit:

    reconciliation_run_id
    left_snapshot_id
    right_snapshot_id
    left_snapshot_time
    right_snapshot_time
    business_date
    join_rule_version
    left_row_count
    right_row_count
    matched_count
    left_only_count
    right_only_count
    matched_equal_count
    matched_different_count

Without this metadata, historical reconciliation results become difficult to explain.

### 3.12 Idempotence

A reconciliation should be deterministic for the same input snapshots and rule version.

Ensure:

1. Both snapshots are stable.
2. Duplicate keys are rejected or deterministically resolved.
3. Attribute comparison rules are explicit.
4. Output classification is deterministic.
5. Run metadata identifies the compared snapshots.

## 4. Testing

FULL OUTER JOIN tests must validate all three presence categories and attribute differences.

### 4.1 Matched-equal test

    left:  customer 7, name = Alice, status = active
    right: customer 7, name = Alice, status = active

Expected:

    MATCHED_EQUAL

### 4.2 Matched-different test

    left:  customer 7, status = active
    right: customer 7, status = suspended

Expected:

    MATCHED_DIFFERENT

### 4.3 Left-only test

Given a key that exists only in the left input:

    LEFT_ONLY

The output must retain the left record.

### 4.4 Right-only test

Given a key that exists only in the right input:

    RIGHT_ONLY

The output must retain the right record.

### 4.5 Complete reconciliation test

Example:

    left keys  = {1, 2, 3}
    right keys = {2, 3, 4}

Expected:

    matched   = {2, 3}
    left_only = {1}
    right_only = {4}

### 4.6 Duplicate-key test

Put two left records with the same reconciliation key.

Expected:

    fail before trusted reconciliation output

Repeat on the right side.

### 4.7 NULL attribute comparison

Compare:

    left.name  = NULL
    right.name = NULL

If the contract treats both missing values as equal, the result should classify the attribute as equal.

Use null-safe comparison explicitly.

### 4.8 NULL join-key test

Give both sides NULL identity values.

Ordinary equality should not match them.

Verify that they are classified according to the documented unknown-identity policy.

### 4.9 Composite-key test

Use the same customer ID in multiple tenants.

Verify that the FULL JOIN produces tenant-correct classifications.

### 4.10 Snapshot mismatch test

Compare snapshots from different business dates.

The test should demonstrate why a right-only or left-only result may represent timing rather than a data defect.

### 4.11 Attribute-level reconciliation test

Change only one field on the right side.

Expected:

    MATCHED_DIFFERENT

and the changed field should be identifiable.

### 4.12 Idempotence test

Run the same reconciliation twice against the same snapshots.

Verify identical classifications and counts.

## 5. Observability

FULL OUTER JOIN observability should describe the entire relationship between the two populations.

### Core metrics

| Metric | Meaning |
|---|---|
| `full_join_left_rows` | Input population on the left |
| `full_join_right_rows` | Input population on the right |
| `full_join_matched_rows` | Keys present on both sides |
| `full_join_left_only_rows` | Keys only on the left |
| `full_join_right_only_rows` | Keys only on the right |
| `full_join_matched_equal_rows` | Matched keys with equivalent attributes |
| `full_join_matched_different_rows` | Matched keys with attribute differences |
| `full_join_duplicate_left_keys` | Left uniqueness violations |
| `full_join_duplicate_right_keys` | Right uniqueness violations |
| `full_join_output_rows` | Physical reconciliation output |
| `full_join_rule_version` | Version of reconciliation semantics |

### Population equation

For unique keys:

    output_keys = left_only + matched + right_only

and:

    left_rows  = left_only + matched
    right_rows = right_only + matched

These equations provide powerful reconciliation assertions.

### Match rate

Two useful rates are:

    left_match_rate  = matched / left_rows
    right_match_rate = matched / right_rows

Both are important.

A 95% left match rate and a 60% right match rate tell a different story from a 95% rate on both sides.

### Difference rate

For matched records:

    difference_rate = matched_different / matched

Track this separately from presence differences.

### Attribute-level metrics

For important fields, track:

    field_difference_count
    field_difference_rate

Examples:

- name differences
- status differences
- currency differences
- balance differences

### Snapshot observability

Record:

- left snapshot identifier
- right snapshot identifier
- business date
- extraction timestamps
- pipeline run IDs

This allows operators to distinguish data drift from snapshot timing issues.

## 6. Intentional Failure

### Failure 1 — Duplicate key on the left

Insert two left records for one comparison key.

Expected symptom:

- Join output multiplies.
- Reconciliation counts become misleading.

Recovery:

Determine the intended grain, resolve or quarantine duplicates, then rerun.

### Failure 2 — Duplicate key on the right

Repeat on the right side.

Expected symptom:

- Matched combinations multiply.

Recovery:

Correct the comparison population before joining.

### Failure 3 — Remove part of a composite key

Join only on `customer_id`.

Expected symptom:

- Cross-tenant matches.
- False equality.
- Potential multiplication.

Recovery:

Restore the complete identity predicate and replay.

### Failure 4 — Use COALESCE before classification

Collapse left and right IDs into one display field without recording side existence.

Expected symptom:

- Operators cannot tell whether a key was left-only, right-only, or matched.

Recovery:

Restore explicit presence classification.

### Failure 5 — Compare NULL with ordinary equality

Use:

    left.name = right.name

Expected symptom:

- Two NULL values may not be classified as equal.

Recovery:

Use null-safe comparison according to the reconciliation contract.

### Failure 6 — Compare mismatched snapshots

Use different business dates.

Expected symptom:

- Large left-only/right-only populations.

Recovery:

Align snapshots or explicitly label the comparison as asynchronous.

### Failure 7 — Reconcile different grains

Join customer-level data to transaction-level data using customer ID.

Expected symptom:

- Severe row multiplication.
- Misleading presence counts.

Recovery:

Aggregate or otherwise transform both sides to the same comparison grain before joining.

## 7. Recovery

FULL OUTER JOIN recovery is primarily about determining whether a difference is caused by identity, timing, source quality, or the reconciliation rule.

### Recovery sequence

1. Identify the reconciliation run.
2. Record both snapshot identifiers and business dates.
3. Verify the comparison grain.
4. Check uniqueness on both sides.
5. Verify the complete join key.
6. Review matched, left-only, and right-only counts.
7. Inspect attribute-level differences.
8. Check source freshness.
9. Check temporal validity.
10. Determine whether the difference is expected.
11. Correct source data or reconciliation logic.
12. Rerun against stable snapshots.
13. Compare the new reconciliation with the previous result.
14. Record the final disposition.

### Classify before repairing

Do not treat every mismatch as a source defect.

Classify differences as:

    EXPECTED_BUSINESS_DIFFERENCE
    SOURCE_DATA_DEFECT
    REFERENCE_LAG
    SNAPSHOT_TIMING_DIFFERENCE
    KEY_MAPPING_DEFECT
    TRANSFORMATION_DEFECT

Then determine the appropriate remediation.

### Recovery after duplicate-key failure

If a source violates the expected unique comparison grain:

1. Stop trusted reconciliation output.
2. Identify duplicate keys.
3. Determine whether duplicates represent versions, events, or corruption.
4. Reduce the source to the documented comparison grain using an explicit rule.
5. Revalidate uniqueness.
6. Rerun reconciliation.

Do not add `DISTINCT` without knowing which record should represent the business entity.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL provides FULL OUTER JOIN semantics, constraints, indexes, query planning, and `IS NOT DISTINCT FROM` for null-safe comparison.

Use database constraints to enforce uniqueness whenever the comparison grain permits it.

### 2. dbt

dbt is useful for version-controlled reconciliation models and data tests.

Custom tests can assert population equations, uniqueness, relationship coverage, and acceptable difference thresholds.

### 3. DuckDB

DuckDB is useful for reproducing snapshot reconciliation locally from CSV and Parquet datasets.

It is well suited to building small controlled datasets that demonstrate matched, left-only, and right-only behavior.

## 9. Production Runbook

### Before deployment

- [ ] Define both input populations.
- [ ] Define the common comparison grain.
- [ ] Define the complete identity key.
- [ ] Validate uniqueness on both sides.
- [ ] Define matched semantics.
- [ ] Define left-only semantics.
- [ ] Define right-only semantics.
- [ ] Define attribute comparison rules.
- [ ] Define NULL comparison semantics.
- [ ] Align snapshot timing.
- [ ] Add population reconciliation assertions.

### During execution

- [ ] Record left and right input counts.
- [ ] Record duplicate-key counts.
- [ ] Record matched count.
- [ ] Record left-only count.
- [ ] Record right-only count.
- [ ] Record matched-equal count.
- [ ] Record matched-different count.
- [ ] Record snapshot identifiers.

### If left-only increases

1. Inspect newly missing right-side keys.
2. Check right-source freshness.
3. Check key normalization.
4. Check reference availability.
5. Check snapshot timing.
6. Determine whether the difference is expected.

### If right-only increases

1. Inspect newly appearing right-side keys.
2. Check left-source freshness.
3. Check migration or replication lag.
4. Check key mapping.
5. Check snapshot alignment.

### If matched-different increases

1. Group differences by field.
2. Identify affected source or tenant.
3. Compare business dates.
4. Check transformation versions.
5. Determine whether the difference is expected business change or data defect.

### If output explodes

1. Check duplicate keys on both sides.
2. Check comparison grain.
3. Check incomplete join predicates.
4. Check for accidental one-to-many or many-to-many relationships.
5. Stop downstream propagation until the grain is validated.

## 10. Common Mistakes

### Mistake 1 — Treating FULL JOIN as UNION

JOIN relates rows; UNION appends populations.

### Mistake 2 — Ignoring the comparison grain

Reconcile like with like.

### Mistake 3 — Assuming matched means equal

Identity matching and attribute equality are separate questions.

### Mistake 4 — Using nullable attributes to detect side existence

Use guaranteed non-null identity columns.

### Mistake 5 — Ignoring duplicate keys

Duplicates can multiply matched combinations.

### Mistake 6 — Using COALESCE as the classification mechanism

Classify presence first, then create display values.

### Mistake 7 — Comparing asynchronous snapshots

Timing differences can look like data defects.

### Mistake 8 — Using ordinary equality for NULL-aware comparison

Define and implement explicit NULL comparison semantics.

### Mistake 9 — Joining different grains

Customer-level and transaction-level data should not be compared directly unless that grain change is intentional.

### Mistake 10 — Treating every mismatch as an incident

Some differences are legitimate business changes.

### Mistake 11 — Fixing duplicate keys with arbitrary selection

Define the authoritative representation first.

### Mistake 12 — Looking only at total mismatch counts

Break differences down by tenant, source, partition, field, and reason.

## 11. Definition of Done

The FULL OUTER JOIN transformation is complete when you can:

- [ ] Explain FULL OUTER JOIN semantics.
- [ ] Distinguish MATCHED, LEFT_ONLY, and RIGHT_ONLY.
- [ ] Define the comparison grain.
- [ ] Define and validate the complete identity key.
- [ ] Detect duplicate keys on both sides.
- [ ] Explain row multiplication.
- [ ] Compare attributes separately from identity.
- [ ] Implement NULL-safe attribute comparison.
- [ ] Explain NULL join-key behavior.
- [ ] Align snapshot timing.
- [ ] Build a migration or reconciliation report.
- [ ] Validate population equations.
- [ ] Observe presence and attribute differences.
- [ ] Intentionally break the reconciliation and diagnose it.
- [ ] Recover from duplicate keys, timing issues, and predicate defects.
- [ ] Replay deterministically using stable snapshots.
- [ ] Explain how PostgreSQL, dbt, and DuckDB support production FULL JOIN work.

## 12. What You Learned

A FULL OUTER JOIN is most valuable when both populations matter and the pipeline must explain the relationship between them.

The production reconciliation model is:

    LEFT POPULATION       RIGHT POPULATION
          \                    /
           \                  /
            DEFINE IDENTITY
                  ↓
          VALIDATE CARDINALITY
                  ↓
            FULL OUTER JOIN
                  ↓
       ┌──────────┼──────────┐
       ↓          ↓          ↓
   LEFT_ONLY   MATCHED   RIGHT_ONLY
                  ↓
        COMPARE ATTRIBUTES
                  ↓
          EQUAL / DIFFERENT
                  ↓
             RECONCILE

> **A FULL OUTER JOIN is not merely a way to keep all rows; it is a mechanism for making two populations explain their differences.**

### Next recipe

**T24 — Anti-Joins**