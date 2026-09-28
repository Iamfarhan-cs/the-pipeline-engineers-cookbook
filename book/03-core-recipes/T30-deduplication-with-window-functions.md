# T30 — Deduplication with Window Functions

> **Goal:** Learn how to detect duplicate records, define a deterministic survivor, remove or quarantine duplicates safely, and prove that the resulting dataset has the intended grain.

## 1. Problem Recognition

Deduplication is required when a pipeline receives multiple records representing the same logical entity or event.

Common causes include:

- Retries without idempotency.
- Replayed files.
- At-least-once delivery.
- CDC reprocessing.
- Duplicate API responses.
- Multiple source extracts.
- Source-system corrections.
- Manual imports.
- Concurrent writes.
- Join fan-out accidentally materialized into a target.

The difficult part is not finding repeated values.

The difficult part is deciding:

    Which rows represent the same logical record?

and:

    Which row should survive?

### Recognition questions

Before deduplicating, ask:

1. What is the business identity of a record?
2. Is the identity a natural key, event ID, or composite key?
3. Are duplicates exact copies or conflicting versions?
4. Which row is authoritative?
5. What timestamp determines recency?
6. What happens when timestamps tie?
7. Can key columns be NULL?
8. Is the source tenant-scoped?
9. Should duplicates be deleted, quarantined, or retained historically?
10. What downstream records depend on the duplicate?

## 2. Concept and Reasoning

### 2.1 Duplicate versus repeated value

A repeated value is not automatically a duplicate.

Example:

    customer_id = 1001

can legitimately appear many times in an order or payment table.

Deduplication requires a defined logical identity.

### 2.2 Grain is the foundation

Suppose the intended grain is:

    one row per payment_id

Then two rows with the same payment ID violate the target grain if both represent the same logical payment.

If the intended grain is:

    one row per customer per day

then the duplicate key is:

    customer_id + business_date

not merely `customer_id`.

### 2.3 Exact duplicates versus conflicting duplicates

Exact duplicates have the same relevant attributes.

Conflicting duplicates share identity but differ in one or more attributes.

Example:

    payment_id = P100
    amount = 100
    status = SUCCESS

versus:

    payment_id = P100
    amount = 100
    status = FAILED

This is not simply a duplicate-count problem.

It is a record-version conflict.

### 2.4 Deduplication is a policy

A production deduplication rule must define a survivor policy.

Examples:

- Keep latest source update.
- Keep earliest accepted event.
- Keep highest source priority.
- Keep successfully validated version.
- Keep the row from the trusted system.
- Keep the version with the greatest sequence number.

Never silently use arrival order unless arrival order is explicitly authoritative.

### 2.5 Window functions expose duplicate groups

A standard pattern is:

    ROW_NUMBER() OVER (
        PARTITION BY business_key
        ORDER BY authoritative_timestamp DESC, stable_tiebreaker DESC
    )

Then:

    row_number = 1

is the survivor.

Rows with:

    row_number > 1

are duplicate candidates under the selected policy.

### 2.6 Why ROW_NUMBER is usually the survivor tool

`ROW_NUMBER()` assigns one unique position to each row.

`RANK()` can assign the same rank to tied rows.

`DENSE_RANK()` also preserves ties.

If the requirement is exactly one survivor per duplicate group, use a deterministic `ROW_NUMBER()` ordering.

### 2.7 Deterministic ordering

This is unsafe:

    ORDER BY updated_at DESC

when two rows can have the same timestamp.

Use a stable tie-breaker:

    ORDER BY updated_at DESC, source_row_id DESC

or another deterministic authority field.

### 2.8 Determinism is a correctness requirement

Running the same deduplication twice should produce the same survivor set when the input is unchanged.

If tied rows can alternate between executions, downstream results become non-reproducible.

### 2.9 Source priority

Sometimes recency is not the correct survivor rule.

Example:

    ERP > API > manual import

Then the ordering can be:

    source_priority DESC, updated_at DESC, source_row_id DESC

Business authority must be explicit.

### 2.10 Composite business keys

Many identities require multiple columns.

Example:

    tenant_id + external_customer_id

Using only `external_customer_id` can incorrectly merge customers belonging to different tenants.

### 2.11 NULL business keys

NULL identity values require special handling.

Two records with NULL in a key column are not necessarily the same business entity.

Do not assume:

    NULL = NULL

in SQL.

If a required identity field is NULL, a safer policy is often to quarantine the record or classify it as an identity-quality failure rather than deduplicate it automatically.

### 2.12 Duplicate events versus duplicate entities

An event table may legitimately contain multiple events for one entity.

Example:

    customer 1001 → LOGIN
    customer 1001 → PAYMENT
    customer 1001 → LOGOUT

These are not duplicates merely because the customer ID repeats.

The event identity might be:

    event_id

or:

    customer_id + event_type + event_time + source_event_id

### 2.13 Duplicate files versus duplicate rows

File-level duplication can cause row-level duplication.

Example:

    same file processed twice

can create:

    same business records twice

Preventing duplicate files upstream is useful, but row-level deduplication may still be required.

### 2.14 Deduplication versus aggregation

Deduplication chooses representative records.

Aggregation combines multiple records into a summary.

Do not deduplicate financial transactions simply because several transactions share a customer and date.

### 2.15 Deduplication versus latest-state selection

Latest-state selection often uses the same window pattern:

    ROW_NUMBER() OVER (...) = 1

But the semantic purpose differs.

Deduplication means multiple rows represent one logical record.

Latest-state selection may intentionally choose one current version from a valid history.

### 2.16 Deduplication before joins

If a dimension contains duplicate keys, joining it to a fact table can multiply fact rows.

Deduplicating or validating the dimension before the join can prevent unintended fan-out.

### 2.17 Deduplication after joins

Do not automatically use `ROW_NUMBER()` after a join to hide row multiplication.

If a join produced duplicates unexpectedly, fix the join cardinality problem instead.

Otherwise deduplication can discard legitimate information.

### 2.18 Idempotency

A good deduplication stage should be idempotent.

Running it twice over the same source snapshot should produce the same result.

### 2.19 Stable identifiers

Prefer immutable source identifiers when available:

    event_id
    transaction_id
    source_record_id

Generated row numbers should not become the business identity.

### 2.20 Duplicate detection versus duplicate removal

These are separate operations.

Detection produces:

    duplicate_group
    duplicate_count
    survivor
    discarded_candidates

Removal changes data.

Production pipelines should make the detection result observable before destructive action.

### 2.21 Quarantine as an alternative

Some duplicates should not be silently removed.

Examples:

- Conflicting payment amounts.
- Conflicting customer identities.
- Missing business keys.
- Contradictory status history.

In such cases, quarantine the group for investigation.

### 2.22 Duplicate groups

A useful diagnostic is:

    COUNT(*) OVER (PARTITION BY business_key) AS duplicate_group_size

This lets the pipeline distinguish:

    unique row → group size 1
    duplicate row → group size > 1

### 2.23 Duplicate ratio

Useful metrics include:

    duplicate_rows / input_rows

and:

    duplicate_groups / total_groups

Track these over time because a sudden increase often indicates a source or ingestion regression.

### 2.24 Survivor ratio

Also track:

    survivors / duplicate_candidates

Expected values depend on the policy.

### 2.25 Conflict ratio

Not every duplicate group is equivalent.

Track groups containing conflicting values separately from exact duplicates.

### 2.26 Duplicate key distribution

One duplicate group with 100,000 rows is operationally different from 100,000 groups with two rows each.

Monitor maximum and percentile duplicate-group sizes.

### 2.27 Deduplication timing

Deduplicate as early as practical when duplicate records are clearly invalid and downstream fan-out would be harmful.

But do not deduplicate before enough information exists to determine identity safely.

### 2.28 Historical retention

If auditability matters, preserve:

    original record
    deduplication decision
    survivor identifier
    rule version
    processing timestamp

Do not destroy evidence required for reconciliation.

## 3. Implementation

### 3.1 Define the contract

Example:

    Target grain:       one row per tenant + payment_id
    Identity:           tenant_id + payment_id
    Survivor:           latest trusted update
    Tie-breaker:        source_row_id
    NULL identity:      quarantine
    Conflicting amount: quarantine
    Exact duplicates:   retain one
    Rule version:       dedupe_v1

### 3.2 Example schema

    CREATE TABLE payment_staging (
        tenant_id BIGINT NOT NULL,
        payment_id TEXT,
        amount NUMERIC(20,4),
        status TEXT,
        updated_at TIMESTAMPTZ,
        source_priority INTEGER NOT NULL,
        source_row_id TEXT NOT NULL
    );

### 3.3 Detect duplicate groups

    SELECT
        tenant_id,
        payment_id,
        COUNT(*) AS duplicate_group_size
    FROM payment_staging
    WHERE payment_id IS NOT NULL
    GROUP BY tenant_id, payment_id
    HAVING COUNT(*) > 1;

This identifies groups but does not choose a survivor.

### 3.4 Assign deterministic row numbers

    SELECT
        p.*,
        COUNT(*) OVER (
            PARTITION BY tenant_id, payment_id
        ) AS duplicate_group_size,
        ROW_NUMBER() OVER (
            PARTITION BY tenant_id, payment_id
            ORDER BY
                source_priority DESC,
                updated_at DESC NULLS LAST,
                source_row_id DESC
        ) AS survivor_rank
    FROM payment_staging AS p
    WHERE payment_id IS NOT NULL;

Interpretation:

    survivor_rank = 1 → survivor
    survivor_rank > 1 → duplicate candidate

### 3.5 Materialize the decision

Use a CTE when the downstream operation should consume the decision once:

    WITH ranked AS (
        SELECT
            p.*,
            ROW_NUMBER() OVER (
                PARTITION BY tenant_id, payment_id
                ORDER BY
                    source_priority DESC,
                    updated_at DESC NULLS LAST,
                    source_row_id DESC
            ) AS survivor_rank
        FROM payment_staging AS p
        WHERE payment_id IS NOT NULL
    )
    SELECT *
    FROM ranked
    WHERE survivor_rank = 1;

### 3.6 Keep duplicates for audit

Instead of deleting immediately:

    CREATE TABLE payment_dedup_decisions AS
    SELECT
        tenant_id,
        payment_id,
        source_row_id,
        survivor_rank,
        CURRENT_TIMESTAMP AS decision_at
    FROM ranked;

Use an explicit table design in production rather than repeatedly creating ad hoc tables.

### 3.7 Split survivors and duplicates

    WITH ranked AS (
        SELECT
            p.*,
            ROW_NUMBER() OVER (
                PARTITION BY tenant_id, payment_id
                ORDER BY
                    source_priority DESC,
                    updated_at DESC NULLS LAST,
                    source_row_id DESC
            ) AS survivor_rank
        FROM payment_staging AS p
        WHERE payment_id IS NOT NULL
    )
    SELECT *
    FROM ranked
    WHERE survivor_rank = 1;

and:

    WITH ranked AS (
        SELECT
            p.*,
            ROW_NUMBER() OVER (
                PARTITION BY tenant_id, payment_id
                ORDER BY
                    source_priority DESC,
                    updated_at DESC NULLS LAST,
                    source_row_id DESC
            ) AS survivor_rank
        FROM payment_staging AS p
        WHERE payment_id IS NOT NULL
    )
    SELECT *
    FROM ranked
    WHERE survivor_rank > 1;

### 3.8 NULL identity handling

Keep records with missing identity outside automatic survivor selection:

    SELECT *
    FROM payment_staging
    WHERE payment_id IS NULL;

These can be sent to a quarantine path with a reason such as:

    MISSING_DEDUPLICATION_KEY

### 3.9 Exact duplicate detection

First determine whether duplicates differ on relevant attributes.

Example:

    SELECT
        tenant_id,
        payment_id,
        COUNT(*) AS rows_in_group,
        COUNT(DISTINCT amount) AS amount_versions,
        COUNT(DISTINCT status) AS status_versions
    FROM payment_staging
    WHERE payment_id IS NOT NULL
    GROUP BY tenant_id, payment_id
    HAVING COUNT(*) > 1;

A group with one value for every relevant attribute is more likely to be an exact replay.

A group with multiple versions requires conflict policy.

### 3.10 Preserve source authority

If source priority is part of the policy:

    ORDER BY
        source_priority DESC,
        updated_at DESC NULLS LAST,
        source_row_id DESC

Do not replace source priority with arrival time unless that is intentional.

### 3.11 Deduplication after validation

A robust flow can be:

    raw
      ↓
    validate identity
      ↓
    classify duplicate groups
      ↓
    choose survivor
      ↓
    quarantine conflicts
      ↓
    publish unique target

### 3.12 Enforce the target grain

After deduplication, add a database constraint where possible:

    CREATE UNIQUE INDEX uq_payment_target
    ON payment_target (tenant_id, payment_id);

The transformation and storage layer should reinforce the same business invariant.

### 3.13 Safe deletion

Do not delete duplicates directly from the source before proving the survivor decision.

A safer pattern is:

    source
      ↓
    ranked decision
      ↓
    audit decision
      ↓
    publish survivor
      ↓
    remove or archive duplicate candidate

### 3.14 Python implementation

    from collections import defaultdict

    def deduplicate(rows):
        groups = defaultdict(list)

        for row in rows:
            key = (row["tenant_id"], row["payment_id"])
            groups[key].append(row)

        survivors = []
        duplicates = []
        quarantine = []

        for key, group in groups.items():
            if key[1] is None:
                quarantine.extend(group)
                continue

            ordered = sorted(
                group,
                key=lambda row: (
                    row["source_priority"],
                    row["updated_at"] or "",
                    row["source_row_id"],
                ),
                reverse=True,
            )

            survivor = ordered[0]
            survivors.append(survivor)
            duplicates.extend(ordered[1:])

        return survivors, duplicates, quarantine

This mirrors the SQL policy but should use typed timestamps and explicit comparison rules in production.

### 3.15 Stable comparison keys in Python

Do not rely on string ordering for timestamps when the input can contain inconsistent formats.

Parse timestamps into timezone-aware datetime values before sorting.

### 3.16 Idempotent publication

After choosing survivors, publish using a target uniqueness constraint.

Conceptually:

    validate
      ↓
    deduplicate
      ↓
    write unique target
      ↓
    record decision

If the same input is replayed, the survivor set should remain stable.

## 4. Testing

Deduplication tests must prove both survivor correctness and duplicate accounting.

### 4.1 Unique rows

Input:

    P1
    P2
    P3

Expected:

    survivors = 3
    duplicates = 0

### 4.2 Exact duplicate

Input:

    P1 version A
    P1 version A

Expected:

    one survivor
    one duplicate candidate

### 4.3 Conflicting duplicate

Input:

    P1 amount = 100
    P1 amount = 200

Expected:

    conflict classification

if the contract says conflicting values require investigation.

### 4.4 Deterministic tie

Create two records with identical timestamps.

Verify the stable tie-breaker always selects the same survivor.

### 4.5 Source priority

Create an older record from a trusted source and a newer record from a lower-priority source.

Verify the configured authority rule wins.

### 4.6 Composite key

Input:

    tenant A + P1
    tenant B + P1

Expected:

    two independent survivors

### 4.7 NULL key

Input:

    tenant A + NULL
    tenant A + NULL

Expected:

    quarantine or explicit policy outcome

not silent automatic merging unless the business contract explicitly defines NULL as an identity.

### 4.8 Duplicate event versus entity

Create multiple legitimate events for one entity.

Verify they are retained when the event identity differs.

### 4.9 Large duplicate group

Create one key with many records.

Verify:

    group size
    survivor
    duplicate count

and ensure memory/query behavior remains acceptable.

### 4.10 Partition isolation

Create the same business key in multiple tenants.

Verify no cross-tenant deduplication occurs.

### 4.11 Idempotence

Run the transformation twice on the same input.

Expected:

    same survivors
    same duplicate classifications

### 4.12 Target uniqueness

Attempt to publish two survivors for one target key.

Expected:

    database uniqueness constraint rejects the invalid state.

### 4.13 Join-fan-out regression

Create a source dimension with duplicate keys.

Join it to a fact table.

Verify that the test detects increased row counts instead of allowing a later deduplication step to hide the problem.

### 4.14 Reprocessing

Replay the same source batch.

Expected:

    no additional logical records
    stable survivor selection

## 5. Observability

Deduplication must expose both volume and decision quality.

### Core metrics

| Metric | Meaning |
|---|---|
| `dedupe_input_rows` | Rows entering deduplication |
| `dedupe_unique_groups` | Logical identity groups |
| `dedupe_duplicate_groups` | Groups containing multiple rows |
| `dedupe_duplicate_rows` | Rows ranked below survivor |
| `dedupe_survivor_rows` | Published survivor count |
| `dedupe_quarantine_rows` | Rows requiring investigation |
| `dedupe_conflict_groups` | Duplicate groups with conflicting attributes |
| `dedupe_null_key_rows` | Rows missing identity |
| `dedupe_max_group_size` | Largest duplicate group |
| `dedupe_rule_version` | Active survivor policy |

### Duplicate rate

Monitor:

    duplicate_rows / input_rows

A sudden increase can indicate:

- Source replay.
- Broken idempotency.
- API retry regression.
- CDC offset issue.
- Join multiplication.

### Conflict rate

Monitor:

    conflicting_duplicate_groups / duplicate_groups

An increase may indicate source-system divergence rather than simple replay.

### Survivor diagnostics

Record why the survivor won:

    source_priority
    updated_at
    sequence_number
    source_row_id

This makes incident investigation possible.

### Duplicate-group size

Monitor high-cardinality duplicate groups separately.

A group with two rows is usually a different operational problem from a group with millions of rows.

### Quarantine monitoring

Track:

- Quarantine count.
- Quarantine rate.
- Reason distribution.
- Oldest unresolved group.
- Reprocessing status.

### Rule-version monitoring

Persist the deduplication rule version with decisions.

Changing survivor policy without versioning can make historical outputs impossible to explain.

## 6. Intentional Failure

### Failure 1 — No deterministic tie-breaker

Remove the stable final ordering field.

Expected symptom:

- Equal records can produce unstable survivor selection.

Recovery:

Add a deterministic immutable tie-breaker.

### Failure 2 — Deduplicate on an incomplete key

Remove tenant ID from a tenant-scoped identity.

Expected symptom:

- Records from different tenants are merged.

Recovery:

Restore the complete business key and add cross-tenant tests.

### Failure 3 — Use arrival order as authority

Process the same records in a different order.

Expected symptom:

- Survivor changes despite identical source facts.

Recovery:

Use an authoritative business ordering.

### Failure 4 — Treat NULL as a valid identity

Group rows with missing business keys.

Expected symptom:

- Unrelated unknown records collapse into one group.

Recovery:

Quarantine missing identity unless the domain explicitly defines NULL identity semantics.

### Failure 5 — Deduplicate legitimate events

Partition only by customer ID in an event table.

Expected symptom:

- Legitimate events disappear.

Recovery:

Define event identity correctly.

### Failure 6 — Hide join multiplication

Create a one-to-many join and then apply `ROW_NUMBER()` to keep one row.

Expected symptom:

- Legitimate right-side records disappear.

Recovery:

Fix the join cardinality or aggregate intentionally before joining.

### Failure 7 — Delete before audit

Delete duplicate rows before recording decisions.

Expected symptom:

- Survivor reasoning becomes difficult or impossible to reconstruct.

Recovery:

Restore from source and create decision/audit records before destructive cleanup.

### Failure 8 — Ignore conflicting duplicates

Treat every duplicate group as an exact replay.

Expected symptom:

- Contradictory business states are silently discarded.

Recovery:

Classify conflicts separately and quarantine when required.

## 7. Recovery

### Recovery sequence

1. Stop destructive cleanup if the survivor rule is suspected.
2. Preserve the original source snapshot.
3. Identify the business key.
4. Measure duplicate groups.
5. Identify conflicting groups.
6. Verify survivor ordering.
7. Verify tenant or partition scope.
8. Re-run the decision stage.
9. Compare old and new survivor sets.
10. Restore affected target records if necessary.
11. Rebuild downstream outputs from the corrected survivor set.
12. Add a regression test for the root cause.

### Recovering from wrong survivor selection

1. Capture the old decision set.
2. Correct the ordering policy.
3. Recompute all affected groups.
4. Compare survivor IDs.
5. Publish the corrected target atomically.
6. Reconcile downstream counts.

### Recovering from over-deduplication

If legitimate records were removed:

1. Recover original source data.
2. Correct the identity definition.
3. Reconstruct legitimate records.
4. Rebuild downstream aggregates.
5. Reconcile counts and financial totals where relevant.

### Recovering from under-deduplication

If duplicates were published:

1. Identify duplicate target keys.
2. Determine the intended survivor.
3. Correct target state.
4. Rebuild affected aggregates.
5. Add target uniqueness enforcement.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL provides window functions, unique constraints, indexes, CTEs, and query plans needed for robust deduplication.

Learn to combine transformation logic with storage-level uniqueness guarantees.

### 2. dbt

dbt is useful for expressing deduplication models and validating target grain with uniqueness tests.

Use it to make survivor logic reviewable and version-controlled.

### 3. DuckDB

DuckDB is useful for reproducing deduplication behavior locally over CSV or Parquet data.

It is especially useful for testing ranking and tie-breaking rules before production deployment.

## 9. Production Runbook

### Before deployment

- [ ] Define target grain.
- [ ] Define complete business key.
- [ ] Define source authority.
- [ ] Define survivor rule.
- [ ] Define deterministic tie-breaker.
- [ ] Define NULL-key policy.
- [ ] Define conflict policy.
- [ ] Define quarantine policy.
- [ ] Define audit fields.
- [ ] Define rule version.
- [ ] Add target uniqueness enforcement.

### During execution

- [ ] Record input rows.
- [ ] Record logical groups.
- [ ] Record duplicate groups.
- [ ] Record duplicate rows.
- [ ] Record conflict groups.
- [ ] Record null-key rows.
- [ ] Record quarantine rows.
- [ ] Record maximum duplicate-group size.

### If duplicate rate spikes

1. Check ingestion retries.
2. Check file replay.
3. Check CDC offsets.
4. Check source changes.
5. Check join fan-out upstream.
6. Check idempotency keys.

### If survivor selection looks wrong

1. Inspect business key.
2. Inspect ordering fields.
3. Inspect source priority.
4. Inspect timestamp ties.
5. Inspect stable tie-breaker.
6. Compare rule version.

### If the target rejects a uniqueness constraint

1. Preserve the failing batch.
2. Identify duplicate target keys.
3. Compare target and source identity definitions.
4. Correct the transformation or source state.
5. Re-run idempotently.

## 10. Common Mistakes

### Mistake 1 — Calling every repeated key a duplicate

Repeated keys can be legitimate at a different grain.

### Mistake 2 — Deduplicating without defining identity

Without identity, survivor selection is arbitrary.

### Mistake 3 — Using `RANK()` when exactly one survivor is required

Ties can produce multiple rows with rank 1.

### Mistake 4 — Omitting a stable tie-breaker

Equal timestamps do not guarantee deterministic output.

### Mistake 5 — Ignoring tenant scope

Composite identities often require tenant or account context.

### Mistake 6 — Treating NULL keys as equal business identities

Missing identity should usually be handled separately.

### Mistake 7 — Using arrival time as business authority

Arrival order often reflects infrastructure behavior rather than business truth.

### Mistake 8 — Deduplicating after a broken join

Fix unintended cardinality instead of hiding it.

### Mistake 9 — Deleting without an audit trail

Production deduplication needs explainable decisions.

### Mistake 10 — Ignoring conflicting duplicates

Conflicting records can indicate a source-system integrity problem.

### Mistake 11 — Forgetting target constraints

Transformation logic should be reinforced by database invariants where possible.

### Mistake 12 — Not testing reprocessing

Replay behavior is a core production requirement.

## 11. Definition of Done

The deduplication transformation is complete when you can:

- [ ] Define the target grain.
- [ ] Define the complete business identity.
- [ ] Distinguish exact duplicates from conflicting versions.
- [ ] Explain why `ROW_NUMBER()` is useful for survivor selection.
- [ ] Explain the difference between `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()`.
- [ ] Build deterministic ordering.
- [ ] Handle composite keys.
- [ ] Handle NULL identity values safely.
- [ ] Preserve legitimate repeated events.
- [ ] Detect duplicate groups.
- [ ] Select one deterministic survivor.
- [ ] Separate duplicate candidates.
- [ ] Classify conflicting duplicates.
- [ ] Build an audit trail.
- [ ] Enforce target uniqueness.
- [ ] Test idempotent replay.
- [ ] Monitor duplicate and conflict rates.
- [ ] Intentionally break survivor selection.
- [ ] Recover from over- and under-deduplication.
- [ ] Explain how PostgreSQL, dbt, and DuckDB support production deduplication.

## 12. What You Learned

Deduplication is not simply deleting repeated rows.

The production workflow is:

    DEFINE TARGET GRAIN
         ↓
    DEFINE BUSINESS IDENTITY
         ↓
    CLASSIFY DUPLICATE GROUPS
         ↓
    DEFINE SURVIVOR POLICY
         ↓
    ORDER DETERMINISTICALLY
         ↓
    RANK WITH ROW_NUMBER
         ↓
    CLASSIFY SURVIVORS / DUPLICATES / CONFLICTS
         ↓
    AUDIT THE DECISION
         ↓
    PUBLISH UNIQUE TARGET
         ↓
    ENFORCE TARGET GRAIN

> **A production deduplication rule is correct only when identity, survivor authority, tie-breaking, conflict handling, auditability, and target grain are explicit.**

### Next recipe

**T31 — Pivoting**