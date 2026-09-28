# T46 — SCD Type 1

> **Goal:** Implement overwrite-style dimension changes safely when historical versions are intentionally not required.

## 1. Problem Recognition

SCD Type 1 means the dimension keeps the latest governed value by **overwriting the existing value** rather than creating a historical version.

Example:

    Before
    customer_id | segment
    ------------+---------
    C123        | SMB

    Source correction
    customer_id | segment
    ------------+---------
    C123        | Enterprise

    After Type 1
    customer_id | segment
    ------------+---------
    C123        | Enterprise

The old `SMB` value is no longer represented as a separate warehouse version.

Type 1 is appropriate when:

- historical changes are intentionally irrelevant
- a value is being corrected
- the warehouse wants current descriptive state
- preserving prior values would create misleading history
- a downstream system explicitly requires the latest mastered value

Type 1 is **not** appropriate merely because it is easier to implement. If historical reporting depends on the prior state, use a historical strategy such as Type 2.

### Symptoms of a broken Type 1 implementation

- duplicate rows exist for one business key
- an old value survives beside the new value
- rerunning the same source creates duplicates
- source corrections are applied inconsistently
- NULL-to-value changes are missed
- source updates can overwrite newer data
- a failed update leaves partially changed attributes
- a source snapshot is treated as authoritative when it is actually partial

### Questions before implementation

1. What is the dimension grain?
2. What is the business key?
3. Which columns are Type 1 attributes?
4. Which columns must never be overwritten?
5. What proves the source observation is newer or authoritative?
6. Is the incoming source a full snapshot or a partial change feed?
7. How are deletes represented?
8. What is the idempotency identity?
9. How are concurrent updates serialized?
10. How are audit fields maintained?

## 2. Concept and Reasoning

### 2.1 Type 1 is state replacement

The central invariant is:

    one business key → one current dimension state

The pipeline transforms:

    old state + authoritative source state
                    ↓
              latest state

There is no requirement to preserve previous attribute values as warehouse versions.

### 2.2 Type 1 versus Type 2

| Concern | Type 1 | Type 2 |
|---|---|---|
| Historical versions | Not retained | Retained |
| Typical row count | One row/entity | Multiple rows/entity |
| Change operation | Update existing row | Close + insert version |
| Current-state query | Simple | Requires current-row semantics |
| Historical reporting | Cannot reconstruct overwritten state | Supported by versions |
| Common use | Corrections/current descriptive state | Historical business state |

The choice is a data-model decision, not simply a SQL syntax choice.

### 2.3 Business key

A Type 1 dimension needs a stable identity.

Example:

    customer_id = C123

should identify the same customer across source changes.

The business key is normally protected by a unique constraint or unique index at the dimension grain.

Do not use a mutable descriptive field such as customer name as the identity.

### 2.4 Surrogate keys

A Type 1 dimension can still use a surrogate primary key:

    customer_sk | customer_id | segment
    ------------+-------------+---------
    1001        | C123        | Enterprise

The surrogate key provides warehouse identity; the business key determines which entity is updated.

If facts reference the surrogate key, a Type 1 update normally leaves the surrogate key unchanged.

### 2.5 Snapshot versus change feed

Do not use the same update algorithm for every source.

A **full snapshot** says something close to:

    this is the complete authoritative population

A **change feed** says:

    these entities changed

If a partial change feed omits a customer, omission usually means nothing about that customer's current state.

Conversely, if a complete authoritative snapshot omits a customer and the source contract defines omission as deletion, the pipeline can apply the governed delete policy.

### 2.6 Authoritativeness and ordering

An incoming row should not automatically overwrite a newer target state.

Possible ordering evidence includes:

- source version
- source sequence
- source updated timestamp
- CDC log position
- monotonically increasing revision number

Load time alone is weak evidence of business ordering.

### 2.7 NULL semantics

Type 1 change detection must distinguish:

    old = NULL, new = NULL       → unchanged
    old = NULL, new = 'SMB'      → changed
    old = 'SMB', new = NULL      → changed
    old = 'SMB', new = 'SMB'      → unchanged

In PostgreSQL, `IS DISTINCT FROM` provides NULL-safe comparison.

### 2.8 Type 1 attribute scope

Not every source column should be blindly copied.

Define an explicit allowlist:

    Type 1 attributes:
      customer_name
      segment
      region

    Protected warehouse fields:
      customer_sk
      created_at
      source_created_at
      first_seen_at

An explicit mapping prevents source data from accidentally overwriting warehouse-managed metadata.

### 2.9 Change detection versus update execution

Separate these decisions:

1. Is the source row valid?
2. Does the business key exist?
3. Is the incoming state newer/authoritative?
4. Did a Type 1 attribute actually change?
5. Should the row be updated?

Keeping these decisions separate makes the pipeline easier to test and observe.

### 2.10 Idempotency

A replay of the same source state should produce no additional logical change.

For Type 1, this usually means:

    same business key + same authoritative source state
        → no-op

An update statement that writes the same values repeatedly may still create unnecessary database work, trigger side effects, or update `updated_at`. Prefer change-aware updates.

## 3. Implementation

### 3.1 Example schema

    CREATE TABLE dim_customer (
        customer_sk BIGSERIAL PRIMARY KEY,
        customer_id TEXT NOT NULL UNIQUE,
        customer_name TEXT NOT NULL,
        segment TEXT,
        region TEXT,
        is_active BOOLEAN NOT NULL DEFAULT TRUE,
        source_updated_at TIMESTAMPTZ,
        created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
        updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
        source_run_id TEXT NOT NULL
    );

The unique business key prevents duplicate current-state entities.

### 3.2 Stage the source

Normalize the incoming source before mutation.

    CREATE TEMP TABLE customer_stage AS
    SELECT
        customer_id,
        customer_name,
        segment,
        region,
        is_active,
        source_updated_at,
        source_run_id
    FROM raw_customer;

Then validate:

- business key is present
- source grain is correct
- source timestamps are parseable
- enum/domain values are valid
- duplicate business keys are resolved deterministically

### 3.3 Detect changes

A direct comparison can identify changed entities:

    SELECT
        s.customer_id
    FROM customer_stage s
    JOIN dim_customer d
      ON d.customer_id = s.customer_id
    WHERE d.customer_name IS DISTINCT FROM s.customer_name
       OR d.segment IS DISTINCT FROM s.segment
       OR d.region IS DISTINCT FROM s.region
       OR d.is_active IS DISTINCT FROM s.is_active;

This avoids unnecessary updates.

### 3.4 Insert new entities

New business keys need an insert:

    INSERT INTO dim_customer (
        customer_id, customer_name, segment, region,
        is_active, source_updated_at, source_run_id
    )
    SELECT
        s.customer_id, s.customer_name, s.segment, s.region,
        s.is_active, s.source_updated_at, s.source_run_id
    FROM customer_stage s
    LEFT JOIN dim_customer d
      ON d.customer_id = s.customer_id
    WHERE d.customer_id IS NULL;

Use a uniqueness constraint as the database-level guard against duplicate creation.

### 3.5 Update existing entities

An explicit update can be written as:

    UPDATE dim_customer d
    SET customer_name = s.customer_name,
        segment = s.segment,
        region = s.region,
        is_active = s.is_active,
        source_updated_at = s.source_updated_at,
        source_run_id = s.source_run_id,
        updated_at = now()
    FROM customer_stage s
    WHERE d.customer_id = s.customer_id
      AND (
          d.customer_name IS DISTINCT FROM s.customer_name
          OR d.segment IS DISTINCT FROM s.segment
          OR d.region IS DISTINCT FROM s.region
          OR d.is_active IS DISTINCT FROM s.is_active
      );

PostgreSQL does not allow the same unrestricted `UPDATE ... FROM` pattern to resolve arbitrary duplicate source rows safely. Source deduplication must happen before this statement.

### 3.6 Upsert pattern

PostgreSQL can combine insert and update with `ON CONFLICT`:

    INSERT INTO dim_customer (
        customer_id, customer_name, segment, region,
        is_active, source_updated_at, source_run_id
    ) VALUES (...)
    ON CONFLICT (customer_id)
    DO UPDATE SET
        customer_name = EXCLUDED.customer_name,
        segment = EXCLUDED.segment,
        region = EXCLUDED.region,
        is_active = EXCLUDED.is_active,
        source_updated_at = EXCLUDED.source_updated_at,
        source_run_id = EXCLUDED.source_run_id,
        updated_at = now()
    WHERE dim_customer.customer_name IS DISTINCT FROM EXCLUDED.customer_name
       OR dim_customer.segment IS DISTINCT FROM EXCLUDED.segment
       OR dim_customer.region IS DISTINCT FROM EXCLUDED.region
       OR dim_customer.is_active IS DISTINCT FROM EXCLUDED.is_active;

The `WHERE` clause makes the update conditional on an actual Type 1 change.

### 3.7 Protect against stale updates

If the source provides an ordering column, add an authority predicate.

    ...
    DO UPDATE SET
        segment = EXCLUDED.segment,
        source_updated_at = EXCLUDED.source_updated_at
    WHERE EXCLUDED.source_updated_at >= dim_customer.source_updated_at;

Be careful with NULL source timestamps. Define a policy before implementing the comparison.

A source sequence or revision is usually preferable when timestamps are not guaranteed to be unique or monotonic.

### 3.8 Deduplicate source records first

Suppose a batch contains:

    customer_id | source_updated_at | segment
    ------------+-------------------+----------
    C123        | 2026-09-01 10:00  | SMB
    C123        | 2026-09-01 11:00  | Enterprise

Choose the authoritative row deterministically before upserting.

    SELECT *
    FROM (
        SELECT
            s.*,
            row_number() OVER (
                PARTITION BY customer_id
                ORDER BY source_updated_at DESC, source_record_id DESC
            ) AS rn
        FROM customer_stage s
    ) x
    WHERE rn = 1;

The tie-breaker must itself be deterministic.

### 3.9 Full transaction

A safe database mutation commonly follows:

    BEGIN
      stage and validate
      deduplicate source
      verify source authority
      insert new business keys
      update changed existing keys
      reconcile uniqueness and counts
    COMMIT

If reconciliation fails, roll back the mutation rather than publishing a partially transformed dimension.

### 3.10 Delete handling

Type 1 does not define deletion semantics by itself.

Possible policies include:

- set `is_active = false`
- apply a tombstone
- physically delete from a current-state dimension
- retain inactive records for referential reasons

For a partial change feed, missing rows are not proof of deletion.

For a complete snapshot, deletion can be inferred only if the source contract explicitly says omission means deletion.

### 3.11 Python implementation pattern

Keep the decision logic deterministic and let the database perform the atomic mutation.

    from dataclasses import dataclass
    from datetime import datetime

    @dataclass(frozen=True)
    class CustomerState:
        customer_id: str
        segment: str | None
        region: str | None
        source_updated_at: datetime | None

    def is_newer(incoming, existing) -> bool:
        if existing is None:
            return True
        if incoming.source_updated_at is None:
            return False
        if existing.source_updated_at is None:
            return True
        return incoming.source_updated_at >= existing.source_updated_at

Unit-test the policy independently from database I/O.

### 3.12 Batch accounting

Record counts such as:

    input_rows
    invalid_rows
    duplicate_source_rows
    new_entities
    changed_entities
    unchanged_entities
    stale_rows
    rejected_rows

A useful identity is:

    input_rows = invalid_rows + duplicate_source_rows + new_entities
               + changed_entities + unchanged_entities + stale_rows

Only use this exact equation if the categories are mutually exclusive. Otherwise define explicit accounting precedence.

## 4. Testing

### 4.1 Core scenarios

| Scenario | Expected result |
|---|---|
| New business key | Insert one row |
| Unchanged row | No logical update |
| Changed Type 1 attribute | Existing row overwritten |
| NULL → value | Update |
| Value → NULL | Update |
| NULL → NULL | No update |
| Duplicate source rows | One deterministic survivor |
| Same batch replay | No duplicate entity |
| Older source state | Rejected or ignored by authority policy |
| Newer source state | Applied |
| Equal timestamp | Deterministic tie policy |
| Delete event | Governed delete behavior |
| Partial snapshot omission | No accidental delete |
| Full snapshot omission | Delete only if contract allows it |
| Concurrent upserts | One valid current row |
| Failed transaction | No partial mutation |

### 4.2 Constraint tests

Attempt to insert two rows with the same business key.

Expected:

    unique constraint violation

Then verify the existing dimension remains valid.

### 4.3 Idempotency test

Run the same source state twice.

Verify:

- row count is unchanged
- surrogate key is unchanged
- Type 1 values are unchanged
- `updated_at` does not move if no logical change is intended
- downstream triggers or CDC do not emit unnecessary changes

### 4.4 Stale update test

Load a newer target state, then replay an older source record.

Expected:

    target remains at newer state

Do not test only the timestamp. Test the actual source ordering contract.

### 4.5 Concurrency test

Run two workers against the same business key.

Verify:

- no duplicate business key
- final state follows the authority policy
- transaction errors are retriable
- no partially updated row is observable

### 4.6 Failure-path tests

Force failures:

- before insert
- during update
- after one key in a batch
- during reconciliation
- during commit

Verify atomicity and rerun safety.

## 5. Observability

### 5.1 Batch metrics

Track:

- source rows received
- invalid rows
- duplicate source rows
- new entities
- changed entities
- unchanged entities
- stale observations
- delete events
- rejected updates
- database conflicts

### 5.2 Dimension metrics

Track:

- total rows
- distinct business keys
- inactive rows
- rows missing source timestamps
- update rate
- no-op rate
- source-to-target reconciliation differences

### 5.3 Change-rate monitoring

A sudden jump in Type 1 updates can indicate:

- source corruption
- changed normalization
- unstable reference mappings
- a bad CDC position
- a source timestamp problem
- a deployment bug
- an upstream mass correction

Do not treat every spike as an application failure; investigate the source evidence.

### 5.4 Logs

Useful structured fields:

    run_id
    business_key
    source_record_id
    source_version
    source_updated_at
    action = INSERT | UPDATE | NOOP | STALE | REJECT
    changed_attributes
    policy_version

Avoid logging sensitive attribute values unless the logging policy explicitly permits them.

### 5.5 Alerts

Useful alerts include:

- duplicate business keys
- unexpected update-rate spike
- high stale-update rate
- high invalid-source rate
- reconciliation mismatch
- repeated uniqueness conflicts
- missing authority timestamps

## 6. Intentional Failure

### Failure 1 — Duplicate source key

Load two source rows for the same business key without deduplication.

Expected lesson: source grain must be established before mutation.

### Failure 2 — Stale overwrite

Apply an older source record after a newer state.

Expected lesson: arrival order is not necessarily business order.

### Failure 3 — NULL comparison bug

Use `=` instead of NULL-safe comparison.

Expected symptom: NULL-to-value or value-to-NULL changes are missed.

### Failure 4 — Accidental metadata overwrite

Allow a generic source-to-target mapper to update `created_at` or `customer_sk`.

Expected lesson: Type 1 updates need an explicit writable-column policy.

### Failure 5 — Partial mutation

Force a failure after some rows have been updated.

Expected result: the transaction rolls back, or batch-level recovery explicitly isolates the affected unit.

### Failure 6 — Snapshot mistaken for change feed

Treat an incomplete extract as authoritative and deactivate every omitted entity.

Expected lesson: deletion inference depends on source completeness semantics.

## 7. Recovery

### 7.1 Stale overwrite recovery

If a stale value overwrote a newer state:

1. Identify affected business keys.
2. Find the authoritative source version.
3. Reapply the correct current state.
4. Verify source ordering metadata.
5. Reconcile downstream consumers.
6. Record the corrective run.

Type 1 cannot reconstruct the overwritten warehouse value unless another source, audit log, CDC stream, backup, or snapshot contains it.

### 7.2 Accidental mass update

If a faulty deployment updates too many rows:

1. Stop further mutation.
2. Capture the affected run ID and SQL/deployment version.
3. Determine the intended state from an authoritative source.
4. Restore using a controlled corrective batch.
5. Reconcile counts and key samples.
6. Investigate why validation did not catch the scope error.

Do not use an uncontrolled rollback query if the correct state depends on source history.

### 7.3 Partial failure

If the database transaction was atomic, rollback and retry.

If processing was intentionally chunked into independent transactions:

- identify completed chunks
- identify failed chunks
- use durable run/chunk identity
- replay only failed chunks
- reconcile the entire batch

### 7.4 Recovery verification

Verify:

- one row per business key
- no protected columns were corrupted
- latest authoritative state is present
- stale state is absent
- row counts reconcile
- rerun is idempotent
- downstream dependencies are consistent

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

Understand:

- `INSERT ... ON CONFLICT DO UPDATE`
- unique constraints and indexes
- transactions
- row locks
- `IS DISTINCT FROM`
- `RETURNING`
- query plans for large upserts

These mechanisms are the foundation of reliable Type 1 loading.

### 8.2 dbt

dbt can express Type 1 dimension transformations through models and incremental strategies.

Useful concepts include:

- incremental models
- `unique_key` configuration
- merge-style incremental behavior
- schema tests
- source freshness and documentation

The underlying business-key and authority policy still belongs in your design.

### 8.3 Great Expectations

Use expectations to validate source and target conditions such as:

- business-key uniqueness
- non-null required attributes
- accepted domains
- expected row-count ranges
- schema contracts

Database constraints should remain the final enforcement layer for relational invariants.

## 9. Production Runbook

### Before deployment

- Define Type 1 attribute allowlist.
- Define business key.
- Add unique constraint/index.
- Define source authority/order.
- Define snapshot versus change-feed semantics.
- Define delete policy.
- Define NULL behavior.
- Define idempotency identity.
- Define protected warehouse-managed fields.
- Define batch accounting.
- Add reconciliation checks.

### Normal run

1. Register run.
2. Stage source data.
3. Validate schema and business-key grain.
4. Deduplicate source records.
5. Determine authoritative state.
6. Insert new entities.
7. Update changed entities.
8. Reconcile counts and uniqueness.
9. Publish completion state.
10. Record metrics and lineage.

### If update volume spikes

Check:

1. source changes
2. normalization changes
3. mapping/reference changes
4. CDC position
5. source timestamps
6. deployment version
7. duplicate source population

### If stale updates appear

Check source ordering evidence and verify that the upsert contains the authority predicate.

### If duplicate keys appear

Stop publication, identify the violating run, repair through the controlled recovery path, then restore the unique constraint or correct the source mutation path.

## 10. Common Mistakes

1. **Using Type 1 when historical state is required.** Overwritten values cannot be reconstructed from the Type 1 table alone.
2. **Treating every source snapshot as a complete authoritative state.** Partial extracts have different semantics.
3. **Using arrival time as source truth.** A late record can overwrite newer state.
4. **Skipping source deduplication.** Multiple source rows for one key make update results nondeterministic.
5. **Using `=` for NULL-sensitive comparisons.** Changes can be silently missed.
6. **Updating every row every run.** This creates unnecessary writes and downstream change events.
7. **Allowing source columns to overwrite warehouse-managed metadata.** Use explicit column mappings.
8. **Assuming `ON CONFLICT` alone solves correctness.** It solves one uniqueness problem, not source authority, deletes, validation, or reconciliation.
9. **Ignoring concurrent writers.** Final state can depend on race timing.
10. **Using mutable descriptive fields as business keys.** Identity must come from a stable domain key.
11. **Inferring deletion from omission without a completeness contract.** This can deactivate valid entities.
12. **Failing to preserve recovery evidence.** Type 1 intentionally destroys previous state.
13. **Changing normalization without monitoring update churn.** Small transformations can cause mass rewrites.

## 11. Definition of Done

T46 is complete when you can independently:

- Explain Type 1 overwrite semantics.
- Decide when Type 1 is appropriate and when historical modeling is required.
- Define a stable business key.
- Separate business identity from surrogate warehouse identity.
- Distinguish full snapshots from change feeds.
- Define source authority and ordering.
- Implement NULL-safe change detection.
- Deduplicate source records deterministically.
- Insert new dimension entities.
- Update changed entities without unnecessary writes.
- Implement a safe PostgreSQL upsert.
- Prevent stale source states from overwriting newer state.
- Protect warehouse-managed columns.
- Define explicit delete semantics.
- Make reruns idempotent.
- Handle concurrent updates safely.
- Test normal, stale, duplicate, NULL, delete, and failure paths.
- Observe update churn and source quality.
- Recover from stale or incorrect Type 1 updates.
- Explain how PostgreSQL, dbt, and data-quality tooling support the mechanism.

## 12. What You Learned

SCD Type 1 is simple in appearance but still requires production-grade reasoning.

The essential flow is:

    SOURCE
      ↓
    VALIDATE
      ↓
    DEDUPLICATE
      ↓
    ESTABLISH AUTHORITY
      ↓
    COMPARE CURRENT STATE
      ↓
    INSERT NEW / UPDATE CHANGED / NO-OP UNCHANGED
      ↓
    RECONCILE
      ↓
    PUBLISH

The key property is not merely that the latest value appears in the dimension. The key property is that the latest **authoritative** value appears exactly once, stale observations do not corrupt it, reruns are safe, and the mutation can be diagnosed and recovered.

### Next Recipe

**T47 — SCD Type 2**

Next, implement full historical versioning with surrogate keys, effective intervals, current-row enforcement, late changes, temporal correctness, and safe history reconstruction.