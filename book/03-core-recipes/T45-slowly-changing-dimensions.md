# T45 — Slowly Changing Dimensions

> **Goal:** Understand the general mechanism for preserving dimension history in an ETL pipeline, including change detection, effective dating, current-row semantics, idempotency, late changes, temporal joins, and safe recovery.

## 1. Problem Recognition

A dimension describes the business entities that give facts meaning: customers, merchants, products, accounts, employees, locations, or organizations.

A normal current-state table answers questions such as:

- What is this customer's current segment?
- Which country is this merchant currently associated with?
- What is the product's current category?

Historical analytics asks a different question:

- Which segment did the customer belong to when the transaction happened?
- Which country was the merchant associated with when revenue was recognized?
- Which product category was in effect when the sale occurred?

If the pipeline overwrites the dimension row every time an attribute changes, the current state is preserved but the historical state disappears. A **Slowly Changing Dimension (SCD)** is a controlled way to decide whether, and how, attribute history is retained.

### Symptoms that indicate an SCD problem

- Historical reports change after a dimension attribute is updated.
- Facts can no longer be associated with the dimension state that existed at event time.
- Analysts keep snapshots of dimensions manually.
- A dimension has duplicate current rows for the same business entity.
- Effective periods overlap or contain unexplained gaps.
- A rerun creates another version of the same historical state.
- Late source changes cannot be placed correctly in history.

### The first questions to ask

Before writing SQL, establish:

1. What is the dimension grain?
2. What identifies the real-world entity?
3. Which attributes should retain history?
4. What timestamp represents when the change became effective?
5. What timestamp represents when the warehouse learned about it?
6. What does a current row mean?
7. How are deletes represented?
8. How are unknown or not-yet-arrived entities represented?
9. What should happen when changes arrive late or out of order?

## 2. Concept and Reasoning

### 2.1 Dimension grain

A dimension row must have a precise grain. For example:

    One row = one version of one customer business entity

That is different from:

    One row = one customer

once history is introduced. A Type 2 dimension can contain many rows for one business key, while each row still represents exactly one historical version.

### 2.2 Business key, natural key, and surrogate key

A **business key** identifies the real-world entity in the source domain, such as `customer_id = C123`.

A **surrogate key** identifies one warehouse dimension version, such as `customer_sk = 90127`.

A historical dimension commonly looks like:

    customer_sk | customer_id | segment | valid_from | valid_to | is_current
    ------------+-------------+---------+------------+----------+-----------
    101         | C123        | SMB     | 2026-01-01 | 2026-06-14 | false
    145         | C123        | Enterprise | 2026-06-15 | NULL | true

The business key ties versions together. The surrogate key distinguishes the versions.

### 2.3 Current-state versus historical-state modeling

A current-state dimension contains one authoritative row per business key.

A historical dimension contains multiple versions where required, with explicit rules for which version is active for each point in time.

Do not introduce historical rows merely because a source sends repeated snapshots. A repeated identical snapshot is not necessarily a business change.

### 2.4 SCD types

Common SCD strategies include:

| Type | Behavior | Typical use |
|---|---|---|
| Type 0 | Preserve the original value permanently | Original classification |
| Type 1 | Overwrite the old value | Corrections or attributes where history is irrelevant |
| Type 2 | Add a new version and preserve the old version | Historical reporting |
| Type 3 | Keep limited prior-state columns | Small, explicitly bounded history |
| Type 6 | Combine Type 1, 2, and 3 techniques | Advanced hybrid designs |

T46 and T47 cover Type 1 and Type 2 in implementation depth. T45 establishes the common reasoning needed before choosing one.

### 2.5 Effective time versus processing time

Two timestamps often matter:

- **Effective time:** when the business state became true.
- **Processing/load time:** when the pipeline received or processed the information.

They can differ substantially.

Example:

    Customer changes segment:     June 10
    Source reports change:        June 15
    Warehouse loads change:      June 16

If the warehouse stores only the load timestamp, a temporal query can produce the wrong answer. The model should explicitly decide which clock controls historical validity.

### 2.6 Standard Type 2 validity model

A common representation is:

    valid_from <= point_in_time
    AND
    point_in_time < valid_to

with `valid_to IS NULL` for the open-ended current row.

This half-open interval convention avoids ambiguity at boundaries. For example, a row ending at midnight on June 15 is not valid at exactly June 15 if the next version begins then.

### 2.7 Core invariants

A robust historical dimension normally enforces:

- At most one current row per business key.
- No overlapping effective periods for one business key.
- Every historical row has a valid interval.
- A current row has an open-ended or explicitly governed end.
- A non-current row has a closed interval.
- A new version is created only when tracked attributes actually change.
- Reprocessing the same source state does not create another version.
- Every dimension version has traceable source and run metadata.

### 2.8 Attribute classification

Classify attributes before choosing the SCD policy:

| Attribute behavior | Example | Possible treatment |
|---|---|---|
| Static | Date of birth | Type 0 or governed correction |
| Slowly changing | Customer segment | Type 2 |
| Rapidly changing | Account balance | Usually fact/event modeling instead |
| Corrective | Typo in legal name | Often Type 1 |

Do not force high-frequency operational state into a dimension simply because the column exists on a source customer table.

### 2.9 Row-level versus attribute-level change detection

A row can contain both Type 1 and Type 2 attributes.

For example:

    Type 2: segment, region
    Type 1: corrected display_name

A generic row hash can therefore be dangerous: a Type 1-only correction might accidentally create a historical Type 2 version. Change detection should be based on the attributes governed by the historical policy.

### 2.10 NULL-safe change detection

Ordinary SQL equality does not treat `NULL` and `NULL` as equal in the usual three-valued logic.

Use explicit NULL-safe comparison semantics. In PostgreSQL, `IS DISTINCT FROM` is useful:

    old.segment IS DISTINCT FROM new.segment

This distinguishes:

- `NULL` from a real value.
- a real value from `NULL`.
- two equal non-NULL values.
- two NULL values.

### 2.11 Late and out-of-order changes

An SCD pipeline must distinguish:

- A new state received late but effective in the future.
- A correction effective in the past.
- A duplicate observation.
- An event that arrives out of sequence.

These cases cannot all be handled by simply closing the current row and inserting a new row.

For a past-effective change, the pipeline may need to split an existing interval:

    existing:  [Jan 1, Jun 30)
    late change:          [Apr 15, Apr 30)

Whether such interval surgery is allowed should be an explicit policy. If the source cannot provide trustworthy effective time, the safer policy may be to record the change as observed time rather than invent historical truth.

### 2.12 Deletes

Delete semantics must be explicit.

Possible policies include:

- Type 2 inactive version.
- `is_deleted = true` on the current version.
- Type 1 removal from a current-state table.
- Tombstone dimension member.
- No physical deletion because facts require historical referential integrity.

Never assume a missing source row means deletion unless the source contract proves that the extract is complete.

### 2.13 Unknown and inferred members

Facts can arrive before the dimension entity.

A warehouse often uses a governed unknown member such as:

    customer_sk = 0
    customer_id = UNKNOWN

Later, the real dimension entity can be resolved according to the warehouse's correction policy.

This is different from silently inserting an incomplete customer row with fabricated business attributes.

### 2.14 Temporal fact-to-dimension resolution

When facts need the dimension state valid at event time, use the business key plus the validity interval:

    fact.customer_id = dim.customer_id
    AND fact.occurred_at >= dim.valid_from
    AND (fact.occurred_at < dim.valid_to OR dim.valid_to IS NULL)

The join must be one-to-one for each fact at the intended grain. Overlapping dimension periods can multiply facts and corrupt aggregates.

## 3. Implementation

### 3.1 Example target model

PostgreSQL example:

    CREATE TABLE dim_customer (
        customer_sk BIGSERIAL PRIMARY KEY,
        customer_id TEXT NOT NULL,
        customer_name TEXT NOT NULL,
        segment TEXT,
        region TEXT,
        valid_from TIMESTAMPTZ NOT NULL,
        valid_to TIMESTAMPTZ,
        is_current BOOLEAN NOT NULL,
        is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
        source_updated_at TIMESTAMPTZ,
        loaded_at TIMESTAMPTZ NOT NULL DEFAULT now(),
        source_run_id TEXT NOT NULL
    );

    CREATE UNIQUE INDEX ux_dim_customer_current
    ON dim_customer (customer_id)
    WHERE is_current;

This partial unique index protects the most important current-row invariant.

### 3.2 Stage the source first

Do not compare an uncontrolled source extract directly against the target.

Stage a normalized source population containing:

- business key
- tracked attributes
- source effective timestamp
- source update timestamp
- source record identity
- source run identity
- ingestion metadata

Example:

    CREATE TEMP TABLE customer_stage AS
    SELECT
        customer_id,
        customer_name,
        segment,
        region,
        effective_at,
        updated_at,
        source_run_id
    FROM raw_customer_snapshot;

Then enforce one source row per business key and effective point according to the source contract.

### 3.3 Detect new and changed entities

Conceptually:

    SELECT s.*
    FROM customer_stage s
    LEFT JOIN dim_customer d
      ON d.customer_id = s.customer_id
     AND d.is_current = TRUE
    WHERE d.customer_id IS NULL
       OR d.segment IS DISTINCT FROM s.segment
       OR d.region IS DISTINCT FROM s.region;

The actual implementation should include every attribute governed by the chosen historical policy.

### 3.4 Deterministic fingerprints

For wide dimensions, compute a fingerprint over canonicalized Type 2 attributes.

Example:

    md5(
        coalesce(segment, '<NULL>') || '|' ||
        coalesce(region, '<NULL>')
    ) AS tracked_attributes_hash

Production implementations should use an unambiguous encoding rather than delimiter concatenation when values can contain the delimiter. A structured canonical representation or length-prefixed encoding avoids collisions caused by ambiguous serialization.

Store the policy version if the hash definition can change.

### 3.5 Basic Type 2 change flow

For a normal current-state change:

    BEGIN
      1. Lock or otherwise serialize the affected business key.
      2. Read the current dimension version.
      3. Compare governed Type 2 attributes.
      4. If unchanged, record no-op processing.
      5. If changed, close the current version.
      6. Insert the new version.
      7. Reconcile current-row and interval invariants.
    COMMIT

Closing the old row:

    UPDATE dim_customer
    SET valid_to = :effective_at,
        is_current = FALSE
    WHERE customer_id = :customer_id
      AND is_current = TRUE;

Inserting the new version:

    INSERT INTO dim_customer (
        customer_id, customer_name, segment, region,
        valid_from, valid_to, is_current, source_updated_at, source_run_id
    ) VALUES (
        :customer_id, :customer_name, :segment, :region,
        :effective_at, NULL, TRUE, :updated_at, :source_run_id
    );

Both operations belong in one transaction for the normal current-to-new transition.

### 3.6 Protect against concurrency

Two workers can otherwise read the same current row and both attempt to create a new version.

Possible controls include:

- Partition work by business key.
- Serialize the key through a queue.
- Use row locking inside a database transaction.
- Use advisory locks for controlled key-level serialization.
- Enforce database uniqueness constraints as a final guard.

Concurrency control should be explicit rather than relying on timing.

### 3.7 Idempotent processing

An SCD operation is idempotent when replaying the same source state produces no additional history.

Useful identity components include:

    source_system + business_key + effective_at + source_version

or a durable source event identity when the source provides one.

Do not use a randomly generated load ID as the only duplicate detector.

### 3.8 Multiple changes in one batch

If a source can contain several effective states for one business key, sort them deterministically before applying them.

Example:

    customer_id | effective_at | segment
    ------------+--------------+----------
    C123        | 2026-01-01   | SMB
    C123        | 2026-04-01   | Mid-Market
    C123        | 2026-07-01   | Enterprise

The resulting history should be:

    [Jan 1, Apr 1) SMB
    [Apr 1, Jul 1) Mid-Market
    [Jul 1, infinity) Enterprise

Do not let arbitrary database row order decide history.

### 3.9 Out-of-order changes

For a late event whose effective time is older than the current row:

1. Locate the existing interval containing the effective timestamp.
2. Determine whether the late state differs from that interval.
3. Decide whether historical correction is allowed.
4. If allowed, split or replace the affected interval deterministically.
5. Recalculate neighboring `valid_to` values.
6. Reconcile overlap and gap invariants.
7. Re-resolve affected facts if their dimension surrogate keys can change.

Historical correction can be significantly more expensive than normal append-forward processing. The policy must be explicit.

### 3.10 Temporal interval reconciliation

Useful validation query:

    SELECT a.customer_id, a.customer_sk, b.customer_sk
    FROM dim_customer a
    JOIN dim_customer b
      ON a.customer_id = b.customer_id
     AND a.customer_sk <> b.customer_sk
     AND a.valid_from < coalesce(b.valid_to, 'infinity'::timestamptz)
     AND b.valid_from < coalesce(a.valid_to, 'infinity'::timestamptz);

Any returned pair represents a possible overlap and requires investigation.

Gap detection depends on the business rule. Some dimensions permit gaps; others require continuous coverage from the first known effective date.

### 3.11 Safe publication

A production flow should generally be:

    source
      ↓
    normalized staging
      ↓
    source deduplication
      ↓
    change detection
      ↓
    SCD mutation
      ↓
    interval/current-row reconciliation
      ↓
    downstream publication

Do not publish a dimension state that has not passed its own invariants.

### 3.12 Python implementation pattern

Python can orchestrate the policy while PostgreSQL enforces relational invariants.

    from dataclasses import dataclass
    from datetime import datetime

    @dataclass(frozen=True)
    class DimensionChange:
        business_key: str
        effective_at: datetime
        tracked_hash: str
        source_run_id: str

    def is_change(old_hash: str | None, new_hash: str) -> bool:
        return old_hash is None or old_hash != new_hash

Keep transformation logic deterministic and testable. The database should remain the authority for transactions and uniqueness constraints.

### 3.13 Reference algorithm

    for each business key:
        normalize source records
        deduplicate source observations
        sort by effective time and deterministic tie-breaker
        load existing dimension history
        compare governed attributes
        apply allowed changes
        reconcile intervals
        reconcile current-row uniqueness
        record source/run lineage

Then run batch-level accounting:

    source_rows = unchanged + inserted_versions + corrected_versions + rejected

Do not count unchanged source rows as newly created dimension versions.

## 4. Testing

Test the policy, not only the SQL syntax.

### 4.1 Required scenarios

| Scenario | Expected result |
|---|---|
| New business key | One initial dimension version |
| Unchanged source row | No new version |
| Tracked attribute change | Old version closes; new version opens |
| Type 1-only correction | No unintended Type 2 version |
| NULL → value | Change detected |
| Value → NULL | Change detected |
| NULL → NULL | No change |
| Duplicate source row | Deterministic single application |
| Same event replay | No duplicate history |
| Multiple ordered changes | Correct consecutive intervals |
| Late change | Policy-specific historical correction |
| Out-of-order event | Deterministic handling |
| Delete | Governed delete policy applied |
| Unknown member | Controlled resolution path |
| Overlap attempt | Transaction rejected or quarantined |
| Concurrent updates | One valid history produced |

### 4.2 Invariant tests

Test that:

    count(current rows per business key) <= 1

and, where continuity is required:

    previous.valid_to = next.valid_from

Also test:

- `valid_from < valid_to` for closed rows.
- current rows have the expected open-ended representation.
- no overlapping intervals exist.
- every surrogate key is unique.
- every dimension version has lineage.

### 4.3 Idempotency test

Run the same source batch twice.

Expected:

    first run  -> versions created as required
    second run -> zero additional versions

Compare row counts, version identities, and effective intervals.

### 4.4 Temporal join test

Create facts before, during, and after a dimension change.

Verify that each fact resolves to exactly one expected dimension version.

Also test the exact boundary timestamp where one interval ends and another begins.

### 4.5 Property-style test ideas

Generate multiple changes for one business key and assert:

- intervals remain ordered
- no overlaps exist
- only one current row exists
- replay is stable
- sorting/tie-breaking is deterministic

These tests catch state-machine defects that a few hand-written examples can miss.

## 5. Observability

SCD pipelines need both transformation metrics and history-health metrics.

### 5.1 Batch metrics

Track:

- source rows received
- source rows rejected
- unchanged entities
- new dimension entities
- new historical versions
- Type 1-only corrections
- late changes
- out-of-order changes
- duplicate source observations
- deletes
- unknown/inferred members

### 5.2 Dimension health metrics

Track:

- total dimension rows
- current-row count
- historical-row count
- current rows per business key
- interval overlap count
- required-gap count
- unknown-member usage
- SCD churn rate
- average versions per business key
- maximum versions per business key

Unexpected SCD churn can indicate a bad source timestamp, unstable normalization, or an incorrectly defined change hash.

### 5.3 Logs

Each mutation should be traceable through:

    run_id
    business_key
    source_record_id
    source_updated_at
    effective_at
    previous_dimension_sk
    new_dimension_sk
    change_type
    policy_version

Do not log sensitive dimension values merely to make debugging easier.

### 5.4 Alerts

Useful alerts include:

- more than one current row
- interval overlaps
- unexpected gaps
- sudden SCD churn
- large growth in historical versions
- high late-change rate
- rising unknown-member resolution
- reconciliation failure

Alert thresholds should be based on known operating ranges rather than arbitrary numbers.

## 6. Intentional Failure

Break the pipeline deliberately to learn what evidence proves the failure.

### Failure 1 — Duplicate current row

Attempt to insert a second current row for the same business key.

Expected protection:

    UNIQUE INDEX WHERE is_current = TRUE

Diagnose with:

    SELECT customer_id, count(*)
    FROM dim_customer
    WHERE is_current
    GROUP BY customer_id
    HAVING count(*) > 1;

### Failure 2 — Overlapping intervals

Insert a historical row whose effective interval intersects an existing version.

Expected result: reconciliation detects the overlap before publication.

### Failure 3 — Duplicate replay

Run the same source batch twice.

Expected result: second execution creates no duplicate version.

### Failure 4 — Wrong change hash

Change the fingerprint inputs or normalization logic.

Expected symptom: unexpected SCD churn or missed changes.

Diagnose by comparing canonicalized attribute values and policy versions.

### Failure 5 — Late event

Send an event effective inside an existing historical interval.

Expected result: the pipeline follows its explicit late-change policy rather than silently appending a current row.

### Failure 6 — Crash between close and insert

Force a transaction failure after the old row is closed but before the new row is inserted.

Expected result: database rollback restores the previous valid state.

## 7. Recovery

Recovery depends on whether the error changed history, facts, or only pipeline state.

### 7.1 Failed transaction

If the mutation is transactional, rollback should restore the prior state. Verify current-row and interval invariants before retrying.

### 7.2 Incorrect version created

Do not blindly delete history.

1. Identify the affected business keys.
2. Capture the incorrect run and source evidence.
3. Determine the intended effective intervals.
4. Correct the dimension in a controlled transaction.
5. Reconcile affected facts if surrogate-key resolution changes.
6. Record the correction and lineage.

### 7.3 Backfill

For a historical correction:

    source evidence
        ↓
    isolated staging
        ↓
    deterministic ordering
        ↓
    affected dimension keys
        ↓
    interval reconstruction
        ↓
    temporal fact re-resolution
        ↓
    reconciliation
        ↓
    controlled publication

Backfills should be bounded to affected business keys and time ranges whenever possible.

### 7.4 Recovery verification

After recovery verify:

- one current row per business key
- no prohibited overlaps
- expected gap policy
- correct effective boundaries
- stable rerun behavior
- fact-to-dimension resolution
- source-to-target accounting
- downstream aggregate reconciliation

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

Useful mechanisms include:

- partial unique indexes for current-row uniqueness
- transactions for close-and-insert atomicity
- row locks for key-level serialization
- exclusion constraints where appropriate for interval overlap protection
- `IS DISTINCT FROM` for NULL-safe comparisons
- window functions for interval diagnostics

Understand the SQL mechanics before relying on an abstraction.

### 8.2 dbt

dbt is useful for expressing warehouse transformation logic and tests around dimensional models.

Relevant concepts include:

- incremental models
- snapshots
- unique and relationship tests
- model dependencies
- generated documentation
- controlled deployment of model changes

Treat dbt snapshots as an implementation option, not as a substitute for understanding effective dating and change policy.

### 8.3 Great Expectations

Great Expectations can express reusable data-quality expectations around dimensions, such as:

- uniqueness of business keys under a defined condition
- non-null effective timestamps
- accepted status values
- row-count expectations
- schema expectations

It complements database constraints and transformation tests; it does not replace transactional integrity.

## 9. Production Runbook

### Before deployment

- Define dimension grain.
- Define business/natural key.
- Define surrogate-key strategy.
- Classify attributes by history policy.
- Define effective-time semantics.
- Define processing-time semantics.
- Define delete semantics.
- Define unknown/inferred-member behavior.
- Define late/out-of-order change policy.
- Define idempotency identity.
- Define overlap and gap policy.
- Add current-row and key constraints.
- Add reconciliation checks.

### During each run

1. Register the run.
2. Load and normalize source staging.
3. Validate source grain.
4. Deduplicate source observations.
5. Detect changes using governed attributes.
6. Apply changes transactionally.
7. Reconcile current rows and intervals.
8. Publish only if checks pass.
9. Record counts and lineage.

### If SCD churn spikes

Check, in order:

1. Source update timestamps.
2. String/NULL normalization.
3. Tracked attribute list.
4. Fingerprint implementation.
5. Source duplicate handling.
6. Effective-time conversion and timezone boundaries.
7. Policy-version changes.

### If historical queries return duplicates

Check:

1. Interval overlap.
2. Business-key completeness.
3. Temporal join predicates.
4. Fact grain.
5. Dimension duplicate versions.

### If a late change arrives

Do not automatically append it as current.

Determine:

- effective time
- source confidence
- affected interval
- correction policy
- affected facts
- required backfill scope

Then execute the governed recovery path.

## 10. Common Mistakes

1. **Using the business key as the dimension primary key.** Historical versions need distinct identities.
2. **Overwriting every attribute with Type 2 semantics.** Some changes are corrections, not historical business states.
3. **Using load time as effective time without checking source semantics.** This can move history to the wrong date.
4. **Using ordinary `=` for NULL-sensitive change detection.** NULL transitions can be missed.
5. **Creating a new version for every source snapshot.** Identical snapshots are not automatically business changes.
6. **Ignoring source duplicates.** Duplicate observations can create duplicate versions or nondeterministic history.
7. **Ignoring concurrency.** Two workers can create competing versions.
8. **Allowing overlapping validity intervals.** Temporal joins can then multiply facts.
9. **Assuming missing source rows are deletes.** A partial extract does not prove deletion.
10. **Treating late changes like current changes.** Historical corrections require different logic.
11. **Changing hash definitions without versioning the policy.** Reprocessing can suddenly rewrite history.
12. **Deleting bad history without preserving evidence.** Corrections need auditability.
13. **Ignoring fact re-resolution after historical changes.** Fact-to-dimension mappings can become stale.
14. **Skipping invariant checks because database constraints exist.** Constraints catch some failures; reconciliation catches others.

## 11. Definition of Done

T45 is complete when you can independently:

- Define the grain of a historical dimension.
- Distinguish business keys from surrogate keys.
- Explain current-state versus historical-state dimensions.
- Explain Type 0, Type 1, Type 2, Type 3, and hybrid SCD approaches.
- Classify attributes by history policy.
- Define effective time and processing time.
- Design half-open validity intervals.
- Detect tracked-attribute changes safely.
- Compare NULL values correctly.
- Make normal Type 2 transitions atomic.
- Make processing idempotent.
- Handle duplicate and multiple source observations deterministically.
- Explain late and out-of-order changes.
- Define delete and unknown-member semantics.
- Perform temporal fact-to-dimension joins safely.
- Detect interval overlaps and gaps.
- Protect current-row uniqueness.
- Test normal, failure, replay, and boundary cases.
- Observe SCD churn and history health.
- Recover an incorrect historical mutation.
- Explain the relevant production tools without depending on them for the underlying mechanism.

## 12. What You Learned

A Slowly Changing Dimension is not simply a table with `valid_from` and `valid_to` columns.

The real engineering problem is preserving trustworthy business state over time while controlling change policy, identity, effective time, concurrency, idempotency, temporal correctness, and recovery.

The central model is:

    BUSINESS KEY
         ↓
    CHANGE POLICY
         ↓
    CHANGE DETECTION
         ↓
    VERSIONED STATE
         ↓
    EFFECTIVE INTERVALS
         ↓
    TEMPORAL FACT RESOLUTION
         ↓
    RECONCILIATION + OBSERVABILITY
         ↓
    SAFE RECOVERY

Once this mechanism is understood, SCD Type 1 and Type 2 become concrete policy choices rather than isolated warehouse tricks.

### Next Recipe

**T46 — SCD Type 1**

Next, implement overwrite semantics in detail: when Type 1 is appropriate, how to perform deterministic upserts, how to distinguish corrections from historical changes, and how to make reruns safe.