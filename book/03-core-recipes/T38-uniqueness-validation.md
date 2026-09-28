# T38 — Uniqueness Validation

> **Goal:** Learn how to prove that records are unique at the intended business grain, distinguish legitimate repeats from duplicates, and prevent duplicate entities or facts from corrupting downstream models.

## 1. Problem Recognition

Uniqueness validation answers:

    Does each record have a unique identity at the grain required by the data contract?

Examples:

    customer.customer_id
    payment.payment_id
    tenant_id + account_number
    order_id + order_line_number

A table can have a technically unique database row identifier while still containing duplicate business entities.

### Common uniqueness failures

- Duplicate customer IDs.
- Duplicate transaction IDs.
- Duplicate source events.
- Duplicate composite business keys.
- Multiple active dimension records for one business key.
- Duplicate rows caused by a JOIN.
- Replayed batches without idempotency.
- NULL-heavy keys producing unexpected duplicate behavior.
- Case or whitespace variants representing the same business key.
- Same identifier reused across tenants.
- Historical records incorrectly treated as current duplicates.

### Recognition questions

Before implementing uniqueness validation, ask:

1. What is the intended grain?
2. What columns define identity?
3. Is identity global or tenant-scoped?
4. Is the key natural, business, surrogate, or generated?
5. Can legitimate records share the same key?
6. Are NULL components allowed?
7. Are case and whitespace significant?
8. Is uniqueness required globally or only among active rows?
9. Is uniqueness temporal?
10. Can source systems resend the same record?
11. What happens to duplicates?
12. Which record is authoritative when duplicates conflict?

## 2. Concept and Reasoning

### 2.1 Uniqueness is a grain contract

Before writing a duplicate query, define:

    one row represents one ______

Example:

    one row = one payment

Then define payment identity:

    payment_id

or:

    source_system + source_payment_id

Without a grain, uniqueness validation is arbitrary.

### 2.2 Technical uniqueness versus business uniqueness

A surrogate key can be unique while business identity is duplicated.

Example:

    customer_sk = 1001, customer_id = C100
    customer_sk = 1002, customer_id = C100

The surrogate keys differ.

The business customer may be duplicated.

### 2.3 Duplicate versus repeated event

Two rows with the same customer ID are not necessarily duplicates.

A customer can have many payments.

Two payments with different payment IDs can both be legitimate.

Uniqueness must match the intended grain.

### 2.4 Duplicate versus late-arriving record

A late-arriving record is not automatically a duplicate.

If the event has a unique event ID, it may be a legitimate late record.

### 2.5 Duplicate versus correction

A source may resend a record with changed attributes.

This can represent:

    duplicate
    correction
    replacement
    new version

Classify the business meaning before deleting anything.

### 2.6 Exact duplicates

Exact duplicates have identical values across the relevant record.

Example:

    same payment_id
    same amount
    same timestamp
    same status

These may be safely collapsible only when the contract says repeated copies are equivalent.

### 2.7 Conflicting duplicates

Two records share the same identity but disagree on attributes.

Example:

    payment_id = P100
    amount = 100

versus:

    payment_id = P100
    amount = 120

This requires conflict resolution, not blind deduplication.

### 2.8 Composite uniqueness

Identity can require multiple fields:

    tenant_id + account_number

Checking only account number may falsely identify valid records as duplicates.

### 2.9 Tenant-scoped uniqueness

A business key may be unique within a tenant but reusable globally.

Example:

    tenant A + customer 100
    tenant B + customer 100

Both may be valid.

### 2.10 Case sensitivity

These may or may not represent the same identity:

    ABC123
    abc123

Define canonicalization before uniqueness validation.

### 2.11 Whitespace

These may be semantically identical:

    'ACC-100'
    ' ACC-100 '

Do not allow formatting differences to create duplicate business entities if the contract treats them as equal.

### 2.12 NULL semantics

SQL UNIQUE constraints and `GROUP BY` behavior require careful interpretation when keys contain NULL.

Decide whether NULL means:

    missing identity
    unknown component
    not applicable

A partially NULL business key usually should not be treated as a trustworthy identity.

### 2.13 Partial keys

If identity is:

    tenant_id + account_number

then:

    tenant_id = 1
    account_number = NULL

does not identify a specific account.

Required-field validation should normally run before uniqueness validation.

### 2.14 Active uniqueness

Some domains require uniqueness only for current records.

Example:

    only one active customer profile per customer_id

Historical records may legitimately share the business key.

### 2.15 Temporal uniqueness

A business key may be unique within a time interval rather than globally.

Example:

    account_number valid from start_time to end_time

Intervals must not overlap when the contract requires one active version at a time.

### 2.16 SCD Type 2 uniqueness

SCD Type 2 can contain multiple rows for one business key.

Uniqueness should instead be enforced on:

    business_key + effective_from

or by the stronger invariant:

    no overlapping effective intervals

### 2.17 Snapshot uniqueness

A key can be unique per snapshot.

Example:

    snapshot_date + product_id

The same product can legitimately appear in multiple snapshots.

### 2.18 Event uniqueness

Event streams often use:

    event_id

or:

    source_system + source_event_id

for idempotency.

Timestamp alone is usually not a sufficient event identity.

### 2.19 Idempotency

An ingestion job should be safe to replay.

If the same event is processed twice, uniqueness controls should prevent two trusted copies.

### 2.20 Duplicate detection before insertion

Detect duplicates before publication when possible.

This provides better diagnostics than waiting for a target constraint to fail.

### 2.21 Duplicate detection after insertion

Target constraints remain valuable as a final invariant.

Use both:

    proactive validation
    target enforcement

### 2.22 Duplicate detection with window functions

`ROW_NUMBER()` can identify candidate survivors:

    partition by business key
    order by deterministic survivor priority

Keep the survivor selection rule explicit.

### 2.23 Survivor selection

Possible priority rules:

- Trusted source.
- Latest valid version.
- Highest source sequence.
- Latest ingestion time.
- Explicit correction flag.

Do not select an arbitrary row.

### 2.24 Deterministic tie-breaking

If two rows have identical priority, use a stable tie-breaker.

Example:

    source_record_id

The same input should produce the same survivor on every replay.

### 2.25 Source priority

Two systems may publish the same entity.

Define source authority explicitly.

Example:

    CRM > legacy_import

Do not infer authority from arrival order.

### 2.26 Latest-record selection

Latest is meaningful only if the timestamp is authoritative.

An ingestion timestamp can reflect pipeline delay rather than business chronology.

### 2.27 Business time versus processing time

Use business event time when the contract requires the latest business state.

Use processing time when the contract explicitly defines arrival precedence.

### 2.28 Duplicate groups

Do not only count duplicate rows.

Measure duplicate groups:

    duplicate_group_count
    duplicate_row_count
    conflicting_group_count

A group of ten duplicate rows has different operational significance from ten independent two-row groups.

### 2.29 Duplicate amplification

A JOIN can create apparent duplicates even when both input tables are unique.

Always compare row counts and grain before and after transformations.

### 2.30 Uniqueness after aggregation

An aggregation can restore uniqueness at a target grain.

Example:

    many payments
        ↓
    one customer-day row

Validate uniqueness at the output grain, not the input grain.

### 2.31 Uniqueness and partitioning

A partitioned dataset may be unique only within a partition.

Define whether uniqueness is:

    global
    partition-scoped
    tenant-scoped
    snapshot-scoped

### 2.32 Uniqueness and sharding

Distributed systems may generate locally unique IDs that collide globally.

Verify the actual namespace guarantees before assuming global uniqueness.

### 2.33 Uniqueness and UUIDs

UUID syntax validity does not prove business uniqueness.

Still enforce uniqueness where the contract requires it.

### 2.34 Uniqueness and hashes

Hashes can support duplicate detection but are not automatically collision-proof.

For critical identity, prefer the authoritative business key.

### 2.35 Fingerprints

A record fingerprint can detect exact content duplicates:

    hash(canonicalized_record)

Canonicalization must be deterministic.

### 2.36 Canonicalization before uniqueness

Normalize only according to an explicit identity contract.

Example:

    LOWER(TRIM(email))

may be appropriate for a particular identity rule, but should not be applied universally to every field.

### 2.37 Uniqueness and domain validation

A valid domain value can still appear multiple times when uniqueness is required.

Domain membership and uniqueness are separate controls.

### 2.38 Uniqueness and referential integrity

A parent key usually needs uniqueness so child relationships are deterministic.

Therefore uniqueness validation supports referential integrity.

### 2.39 Uniqueness and range validation

Range validation does not prevent duplicates.

Two records can both be valid and still violate the target grain.

### 2.40 Fail-open versus fail-closed

When uniqueness cannot be checked because the reference or target is unavailable, decide whether to:

    stop
    quarantine
    use a verified unique index

Do not silently publish data whose identity contract was not checked.

## 3. Implementation

### 3.1 Define the uniqueness contract

Example:

    payment
      grain: one row per payment
      identity: tenant_id + payment_id
      uniqueness: globally within tenant
      NULL identity components: forbidden

### 3.2 PostgreSQL duplicate detection

    SELECT tenant_id, payment_id, COUNT(*) AS duplicate_count
    FROM payments_stage
    GROUP BY tenant_id, payment_id
    HAVING COUNT(*) > 1;

This identifies duplicate identity groups.

### 3.3 Duplicate row extraction

    SELECT p.*
    FROM payments_stage p
    JOIN (
        SELECT tenant_id, payment_id
        FROM payments_stage
        GROUP BY tenant_id, payment_id
        HAVING COUNT(*) > 1
    ) d
      ON d.tenant_id = p.tenant_id
     AND d.payment_id = p.payment_id;

Use this for investigation or quarantine.

### 3.4 Window-based duplicate detection

    ROW_NUMBER() OVER (
        PARTITION BY tenant_id, payment_id
        ORDER BY event_time DESC, source_record_id
    ) AS duplicate_rank

Rows with:

    duplicate_rank > 1

are candidate duplicates under that survivor rule.

### 3.5 Deterministic survivor selection

Example priority:

    ORDER BY
        is_correction DESC,
        event_time DESC,
        source_record_id ASC

Every ordering component should have defined semantics.

### 3.6 Detect conflicting duplicates

After grouping by identity, compare relevant attributes.

Example:

    COUNT(DISTINCT amount)
    COUNT(DISTINCT status)

A group can be duplicate-identical or conflicting.

### 3.7 Exact duplicate detection

Compute a canonical fingerprint from selected business fields.

Then group by:

    identity_key + record_fingerprint

This separates exact copies from conflicting records.

### 3.8 Composite key

Example:

    SELECT tenant_id, account_number, COUNT(*)
    FROM accounts_stage
    GROUP BY tenant_id, account_number
    HAVING COUNT(*) > 1;

Use every identity component.

### 3.9 Active-only uniqueness

Example:

    CREATE UNIQUE INDEX ux_active_customer
    ON customer (tenant_id, customer_id)
    WHERE status = 'active';

This enforces uniqueness only for active records.

### 3.10 NULL-aware identity

Do not assume ordinary UNIQUE behavior matches business semantics for NULL.

If a key component is mandatory, validate it before uniqueness checks.

Where the database supports it, consider a `NOT NULL` constraint for mandatory identity columns.

### 3.11 PostgreSQL `NULLS NOT DISTINCT`

PostgreSQL can enforce uniqueness treating NULLs as equal:

    CREATE UNIQUE INDEX ux_key
    ON accounts (tenant_id, account_number) NULLS NOT DISTINCT;

Use this only when the business contract explicitly says NULL identity components should collide.

### 3.12 Target constraint

Stable uniqueness should be enforced at the target:

    CREATE UNIQUE INDEX ux_payment
    ON payments (tenant_id, payment_id);

This is a final invariant, not a substitute for diagnostic validation.

### 3.13 SCD Type 2 uniqueness

Do not use a global unique constraint on business key if multiple historical versions are expected.

Instead validate:

    business_key + effective_from

and separately ensure effective intervals do not overlap.

### 3.14 Detect overlapping intervals

Conceptually compare each interval to other intervals for the same business key.

An overlap exists when:

    start_a < end_b
    AND start_b < end_a

for half-open intervals.

### 3.15 Snapshot-scoped uniqueness

Example:

    CREATE UNIQUE INDEX ux_snapshot_product
    ON product_snapshot (snapshot_date, product_id);

This allows the same product in different snapshots.

### 3.16 Source-event idempotency

Use:

    source_system + source_event_id

as the idempotency identity when that combination is guaranteed by the source contract.

### 3.17 Safe insertion

PostgreSQL can use:

    ON CONFLICT DO NOTHING

when duplicate events should be ignored.

Do not use this blindly for conflicting duplicates because it can hide data-quality problems.

### 3.18 Upsert semantics

`ON CONFLICT DO UPDATE` can be appropriate when a source publishes corrections.

Define which fields are authoritative and under what version or timestamp.

### 3.19 Validation result table

    CREATE TABLE uniqueness_validation_failure (
        batch_id TEXT NOT NULL,
        record_id TEXT NOT NULL,
        identity_key_hash TEXT NOT NULL,
        failure_code TEXT NOT NULL,
        duplicate_group_id TEXT,
        detected_at TIMESTAMPTZ NOT NULL
    );

Hash sensitive identity values when operationally sufficient.

### 3.20 Failure categories

Useful categories:

    EXACT_DUPLICATE
    CONFLICTING_DUPLICATE
    DUPLICATE_ACTIVE_KEY
    NULL_IDENTITY
    AMBIGUOUS_SURVIVOR
    TEMPORAL_OVERLAP
    SOURCE_REPLAY

### 3.21 Quarantine strategy

Quarantine the duplicate group rather than randomly deleting rows.

Store enough metadata to reconstruct:

    identity
    candidate records
    survivor rule
    source
    rule version

### 3.22 Python duplicate detection

    seen = set()
    duplicates = []

    for row in rows:
        key = (row['tenant_id'], row['payment_id'])
        if key in seen:
            duplicates.append(row)
        else:
            seen.add(key)

This is useful for small streams but requires explicit memory considerations for large data.

### 3.23 Pandas duplicate detection

Conceptually:

    duplicate_mask = df.duplicated(
        subset=['tenant_id', 'payment_id'],
        keep=False,
    )

Use `keep=False` when the entire duplicate group must be inspected.

### 3.24 Reconciliation

Track:

    input_rows
    unique_rows
    duplicate_rows
    duplicate_groups
    conflicting_groups

Verify the accounting according to the chosen quarantine model.

### 3.25 Avoiding accidental duplicate creation

After joins, unions, and enrichments, validate the target grain again.

Uniqueness should be checked at important pipeline boundaries, not only at ingestion.

### 3.26 UNION versus UNION ALL

`UNION` removes duplicate rows based on the selected columns.

`UNION ALL` preserves all rows.

Do not use `UNION` as a generic duplicate repair mechanism because it may remove legitimate records and hides why duplicates occurred.

## 4. Testing

### 4.1 Unique input

Create distinct identity keys.

Expected:

    zero duplicate groups

### 4.2 Exact duplicate

Insert the same record twice.

Expected:

    EXACT_DUPLICATE

### 4.3 Conflicting duplicate

Use the same identity with different amount or status.

Expected:

    CONFLICTING_DUPLICATE

### 4.4 Composite-key duplicate

Repeat the complete composite key.

Expected:

    duplicate

### 4.5 Same partial key

Repeat one component while changing another.

Expected:

    valid if the full composite key differs

### 4.6 Tenant isolation

Use the same business key in two tenants.

Expected:

    valid when uniqueness is tenant-scoped

### 4.7 Case normalization

Test identity values differing only by case.

Verify the documented identity policy.

### 4.8 Whitespace normalization

Test leading/trailing whitespace.

Verify whether normalization occurs before uniqueness validation.

### 4.9 NULL identity

Test NULL in a mandatory identity component.

Expected:

    NULL_IDENTITY

rather than a normal duplicate result.

### 4.10 Active uniqueness

Create multiple historical rows and one active row.

Expected:

    historical duplicates allowed
    active uniqueness preserved

according to the contract.

### 4.11 Temporal overlap

Create two versions with overlapping effective intervals.

Expected:

    TEMPORAL_OVERLAP

### 4.12 Adjacent intervals

Create:

    [2026-01-01, 2026-02-01)
    [2026-02-01, 2026-03-01)

Expected:

    no overlap

### 4.13 Source replay

Process the same source event twice.

Expected:

    idempotent outcome

### 4.14 Conflicting replay

Replay the same event ID with changed business attributes.

Expected:

    conflict detected

not silently ignored unless the source contract explicitly defines replacement semantics.

### 4.15 Duplicate after JOIN

Start with unique inputs and create a one-to-many enrichment.

Expected:

    output-grain uniqueness failure detected

### 4.16 Survivor determinism

Shuffle input order.

Expected:

    same survivor

because ordering rules are deterministic.

### 4.17 Parent uniqueness

Create duplicate parent keys.

Expected:

    referential ambiguity detected

### 4.18 Fingerprint stability

Canonicalize the same record twice.

Expected:

    same fingerprint

### 4.19 Idempotence

Run the entire validation twice.

Expected:

    same duplicate groups
    no duplicate quarantine entries

### 4.20 Reconciliation

Verify the input population is completely accounted for.

### 4.21 Constraint enforcement

Attempt to insert a duplicate into a unique target.

Expected:

    database constraint rejects it.

## 5. Observability

### Core metrics

| Metric | Meaning |
|---|---|
| `uniqueness_input_rows` | Records checked |
| `uniqueness_unique_rows` | Records with unique identity |
| `uniqueness_duplicate_rows` | Rows participating in duplicate groups |
| `uniqueness_duplicate_groups` | Number of duplicate identity groups |
| `uniqueness_conflicting_groups` | Duplicate groups with conflicting attributes |
| `uniqueness_null_identity_rows` | Records with incomplete identity |
| `uniqueness_temporal_overlaps` | Overlapping historical versions |
| `uniqueness_source_replays` | Replayed source identities |

### Duplicate rate

Track:

    duplicate_rows / input_rows

Break down by:

- Source.
- Tenant.
- Entity.
- Batch.
- Identity rule.

### Duplicate-group size

Monitor group-size distribution.

Large groups can indicate:

- Retry storms.
- JOIN fan-out.
- Batch duplication.
- Source replay loops.

### Conflict rate

Track the proportion of duplicate groups with differing attributes.

Conflicts generally require more investigation than exact repeated copies.

### Survivor diagnostics

Record which rule selected survivors:

    source priority
    correction flag
    event time
    ingestion time
    tie-breaker

### Target constraint failures

Track database uniqueness violations separately from proactive validation failures.

A target violation can indicate a validation gap.

### Key cardinality

Monitor distinct identity counts over time.

Unexpected changes can indicate:

- Source duplication.
- Key-format changes.
- Missing partition scope.
- Upstream identifier reuse.

## 6. Intentional Failure

### Failure 1 — Wrong grain

Validate customer ID uniqueness in a payment table.

Expected symptom:

- Legitimate multiple payments are falsely classified as duplicates.

Recovery:

Define the correct target grain and identity.

### Failure 2 — Ignore tenant scope

Validate account number globally.

Expected symptom:

- Valid accounts across tenants appear duplicated.

Recovery:

Include tenant in the business key.

### Failure 3 — `DISTINCT` hides duplication

Apply DISTINCT after a fan-out join.

Expected symptom:

- Output appears smaller but the source relationship remains incorrect.

Recovery:

Identify and repair the multiplying relationship.

### Failure 4 — Arbitrary survivor

Use an unordered `ROW_NUMBER()` survivor selection.

Expected symptom:

- Different runs select different records.

Recovery:

Define deterministic ordering.

### Failure 5 — Latest ingestion wins

Use processing time when business event time is authoritative.

Expected symptom:

- Late corrections are overwritten by arrival order.

Recovery:

Use the correct business chronology.

### Failure 6 — NULL treated as normal identity

Allow incomplete composite keys into uniqueness logic.

Expected symptom:

- Missing identities appear valid.

Recovery:

Validate required identity components first.

### Failure 7 — Silent conflict discard

Use `ON CONFLICT DO NOTHING` for conflicting source records.

Expected symptom:

- Important corrections disappear.

Recovery:

Classify conflicting replays separately.

### Failure 8 — Historical versions rejected

Enforce global uniqueness on an SCD Type 2 business key.

Expected symptom:

- Legitimate historical records fail.

Recovery:

Use temporal uniqueness semantics.

## 7. Recovery

### Recovery sequence

1. Preserve all candidate records.
2. Identify the intended grain.
3. Identify the identity key.
4. Classify exact versus conflicting duplicates.
5. Check tenant or partition scope.
6. Check source replay behavior.
7. Check temporal semantics.
8. Determine the authoritative survivor rule.
9. Quarantine ambiguous groups.
10. Correct deterministic duplicates.
11. Reprocess affected outputs.
12. Reconcile counts and aggregates.
13. Add regression tests.

### Recovering exact duplicates

1. Confirm records are semantically identical.
2. Retain one canonical record.
3. Record duplicate evidence.
4. Reconcile output counts.

### Recovering conflicting duplicates

1. Preserve every conflicting record.
2. Identify source authority.
3. Compare business timestamps.
4. Apply the documented correction policy.
5. Quarantine if no deterministic policy exists.
6. Rebuild affected outputs.

### Recovering from wrong-grain validation

1. Stop the incorrect deduplication step.
2. Restore the pre-dedup population.
3. Define the correct grain.
4. Re-run uniqueness validation.
5. Reconcile downstream aggregates.

### Recovering from accidental duplicate creation

1. Identify the transformation that multiplied rows.
2. Measure the amplification.
3. Repair the JOIN or UNION logic.
4. Rebuild the affected partition.
5. Validate target uniqueness.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL provides UNIQUE constraints, unique indexes, partial unique indexes, `ON CONFLICT`, and window functions.

Use database constraints as final invariants and SQL analysis for diagnostics.

### 2. dbt

dbt supports uniqueness tests and model-level data contracts.

Use them to continuously test business-key assumptions in transformations.

### 3. Great Expectations

Great Expectations can express uniqueness expectations and produce validation results.

Use it to operationalize explicit identity contracts rather than treating uniqueness as a generic afterthought.

## 9. Production Runbook

### Before deployment

- [ ] Define target grain.
- [ ] Define identity columns.
- [ ] Define tenant/partition scope.
- [ ] Define NULL identity policy.
- [ ] Define canonicalization.
- [ ] Define active-only or temporal uniqueness.
- [ ] Define duplicate classification.
- [ ] Define survivor rules.
- [ ] Define tie-breakers.
- [ ] Define source replay behavior.
- [ ] Define quarantine policy.
- [ ] Add target uniqueness constraints where appropriate.

### During execution

- [ ] Record input rows.
- [ ] Record unique rows.
- [ ] Record duplicate rows.
- [ ] Record duplicate groups.
- [ ] Record conflicts.
- [ ] Record temporal overlaps.
- [ ] Record source replays.
- [ ] Reconcile populations.

### If duplicate rate spikes

1. Identify affected identity.
2. Compare source release changes.
3. Check replay behavior.
4. Check JOIN fan-out.
5. Check key normalization.
6. Check partition or tenant scope.

### If conflicts spike

1. Preserve all conflicting records.
2. Identify source authority.
3. Check event chronology.
4. Check correction semantics.
5. Avoid blind deduplication.

### If target unique constraint fails

1. Capture the failing key.
2. Compare proactive validation results.
3. Determine whether the validation has a race or coverage gap.
4. Repair the gap.
5. Replay safely.

## 10. Common Mistakes

### Mistake 1 — No target grain

Uniqueness without grain is meaningless.

### Mistake 2 — Confusing repeated entities with repeated facts

Customers can repeat across payments; payments should not repeat at their payment grain.

### Mistake 3 — Ignoring composite keys

Partial-key validation creates false duplicates or misses real ones.

### Mistake 4 — Ignoring tenant scope

Shared identifiers can be legitimate across tenants.

### Mistake 5 — Treating every duplicate as deletable

Conflicting duplicates may represent corrections.

### Mistake 6 — Arbitrary survivor selection

Without deterministic ordering, replays can change results.

### Mistake 7 — Using ingestion time blindly

Arrival order is not always business chronology.

### Mistake 8 — Using DISTINCT as a repair

Deduplication can hide upstream fan-out.

### Mistake 9 — Ignoring NULL identity

Incomplete identity should not enter trusted uniqueness logic.

### Mistake 10 — Global uniqueness for temporal entities

Historical versions may legitimately share a business key.

### Mistake 11 — No database constraint

Application validation alone leaves a final integrity gap.

### Mistake 12 — No duplicate observability

Duplicate groups and conflict rates should be measurable.

## 11. Definition of Done

The uniqueness-validation stage is complete when you can:

- [ ] Define target grain.
- [ ] Define business identity.
- [ ] Distinguish technical and business uniqueness.
- [ ] Validate composite keys.
- [ ] Handle tenant-scoped identity.
- [ ] Define NULL identity behavior.
- [ ] Handle case and whitespace canonicalization.
- [ ] Detect exact duplicates.
- [ ] Detect conflicting duplicates.
- [ ] Define deterministic survivor selection.
- [ ] Validate active-only uniqueness.
- [ ] Validate temporal uniqueness.
- [ ] Detect overlapping SCD intervals.
- [ ] Detect source replays.
- [ ] Prevent JOIN-generated duplicate output.
- [ ] Preserve duplicate evidence.
- [ ] Quarantine ambiguous records.
- [ ] Enforce stable uniqueness with database constraints.
- [ ] Reconcile duplicate populations.
- [ ] Monitor duplicate and conflict rates.
- [ ] Intentionally break uniqueness controls.
- [ ] Recover from wrong-grain, replay, and conflict failures.
- [ ] Explain how PostgreSQL, dbt, and Great Expectations support production uniqueness validation.

## 12. What You Learned

Uniqueness validation is a grain and identity problem, not simply a `COUNT(*)` problem.

The production workflow is:

    DEFINE TARGET GRAIN
         ↓
    DEFINE BUSINESS IDENTITY
         ↓
    CANONICALIZE IDENTITY WHERE REQUIRED
         ↓
    DETECT DUPLICATE GROUPS
         ↓
    CLASSIFY EXACT / CONFLICTING / TEMPORAL DUPLICATES
         ↓
    APPLY DETERMINISTIC SURVIVOR POLICY
         ↓
    QUARANTINE AMBIGUITY
         ↓
    ENFORCE TARGET UNIQUENESS
         ↓
    MONITOR DUPLICATE DRIFT

> **A unique database row is not necessarily a unique business entity; production uniqueness validation starts with the correct grain and ends with deterministic, observable identity enforcement.**

### Next recipe

**T39 — Completeness Checks**