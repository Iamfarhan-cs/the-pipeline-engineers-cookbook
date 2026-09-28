# T47 — SCD Type 2

> **Goal:** Implement full historical dimension versioning so each governed business state remains queryable at the correct point in time.

## 1. Problem Recognition

SCD Type 2 preserves historical versions instead of overwriting the previous dimension state.

Suppose a customer changes segment:

    Before
    C123 → SMB

    Change effective 2026-06-15
    C123 → Enterprise

A Type 2 dimension keeps both states:

    customer_sk | customer_id | segment     | valid_from | valid_to   | is_current
    ------------+-------------+-------------+------------+------------+-----------
    101         | C123        | SMB         | 2026-01-01 | 2026-06-15 | false
    145         | C123        | Enterprise  | 2026-06-15 | NULL       | true

The business key identifies the entity. The surrogate key identifies the historical version.

### When Type 2 is required

Use Type 2 when the business needs questions such as:

- Which customer segment applied when this payment occurred?
- Which sales territory owned this customer at the time?
- Which product category was active when the order was placed?
- How did the customer population change over time?
- What dimension state was used when an earlier report was produced?

### Symptoms of a broken Type 2 implementation

- historical reports change after dimension updates
- more than one current row exists for a business key
- effective intervals overlap
- effective intervals unexpectedly contain gaps
- a fact joins to multiple dimension versions
- rerunning a batch creates duplicate versions
- late changes are incorrectly appended as current
- surrogate keys are unstable
- old rows are overwritten instead of closed

### Questions before implementation

1. What is the dimension grain?
2. What is the business key?
3. Which attributes are Type 2?
4. What timestamp defines business effectiveness?
5. What timestamp defines warehouse observation?
6. Are intervals continuous?
7. How are deletes represented?
8. How are late and out-of-order changes handled?
9. What is the idempotency identity?
10. Can facts be re-resolved when history changes?

## 2. Concept and Reasoning

### 2.1 The Type 2 state model

For one business key, history is a sequence of non-overlapping intervals:

    version 1: [t1, t2)
    version 2: [t2, t3)
    version 3: [t3, infinity)

Each interval contains the dimension attributes that were valid during that period.

The standard temporal predicate is:

    valid_from <= point_in_time
    AND
    (point_in_time < valid_to OR valid_to IS NULL)

Half-open intervals avoid double-counting the exact boundary between versions.

### 2.2 Type 2 versus Type 1

| Concern | Type 1 | Type 2 |
|---|---|---|
| Previous value | Overwritten | Preserved |
| Historical reporting | Not available from dimension alone | Supported |
| Rows per business key | Usually one | Multiple |
| Surrogate key on change | Usually unchanged | New version key |
| Change operation | UPDATE | Close + INSERT |
| Temporal joins | Not required | Required |

T46 covers Type 1. T47 applies the historical mechanism introduced in T45.

### 2.3 Business key and surrogate key

Example:

    customer_id = C123

is the business identity.

Each historical version receives a different surrogate key:

    customer_sk = 101 → SMB
    customer_sk = 145 → Enterprise

Facts should normally reference the surrogate key resolved for the fact's event time.

### 2.4 Effective time and load time

Keep these concepts separate:

- `valid_from`: business-effective time
- `valid_to`: end of business validity
- `source_updated_at`: source observation/version time
- `loaded_at`: warehouse processing time

Example:

    Business change:  June 10
    Source reports:   June 15
    Warehouse loads:  June 16

If June 10 is the trusted effective timestamp, the new version belongs at June 10 even though the warehouse learned about it later.

### 2.5 Current-row invariant

For each business key:

    count(is_current = true) <= 1

Most implementations require exactly one current row after the entity has been introduced.

Use a database constraint where possible rather than relying only on application logic.

### 2.6 Non-overlap invariant

Two versions for one business key must not represent the same instant unless the model explicitly supports simultaneous states.

For normal SCD Type 2:

    previous.valid_to = next.valid_from

when continuous coverage is required.

If gaps are allowed, that must be an explicit business rule.

### 2.7 Change detection

Only tracked Type 2 attributes should trigger a new version.

Example:

    Type 2: segment, region
    Type 1: corrected display_name

A display-name correction should not automatically create a new Type 2 version if the policy says the name is current-state-only.

Use NULL-safe comparison or a canonical fingerprint over exactly the tracked attributes.

### 2.8 New, unchanged, and changed states

Every staged entity should resolve to one of these logical outcomes:

| State | Action |
|---|---|
| New business key | Insert initial version |
| Existing + unchanged | No new version |
| Existing + tracked change | Close old + insert new |
| Existing + Type 1-only change | Apply governed Type 1 correction |
| Stale observation | Ignore/reject according to authority policy |
| Invalid | Quarantine/reject |

Making these dispositions explicit prevents accidental history churn.

### 2.9 Multiple changes in one batch

If one source batch contains several effective states:

    C123  2026-01-01  SMB
    C123  2026-04-01  Mid-Market
    C123  2026-07-01  Enterprise

the pipeline must sort by effective time and apply them deterministically.

Expected history:

    [2026-01-01, 2026-04-01) SMB
    [2026-04-01, 2026-07-01) Mid-Market
    [2026-07-01, infinity) Enterprise

Database row order must never determine business history.

### 2.10 Same-time changes

Two source records can have the same effective timestamp.

Define a deterministic tie-breaker such as:

    effective_at
    source_sequence
    source_record_id

Do not invent a one-second or one-millisecond offset simply to make rows fit. Artificial time changes alter business meaning.

### 2.11 Late-arriving historical changes

A late record can fall inside an already-existing interval.

Example:

    Existing: [Jan 1, Jun 30) SMB
    Late event: effective Apr 15 → Enterprise

The correct result may require:

    [Jan 1, Apr 15) SMB
    [Apr 15, Jun 30) Enterprise

and subsequent versions must continue from the existing timeline.

This is **interval surgery**, not a normal append.

Whether historical correction is permitted must be an explicit source and warehouse policy.

### 2.12 Out-of-order processing

Do not confuse:

    event/effective order
with
    arrival order

Type 2 correctness depends on the chosen effective-time model. If the source cannot provide trustworthy effective time, the pipeline should not manufacture historical truth.

### 2.13 Deletes

A delete may be represented as:

- a final inactive Type 2 version
- `is_deleted = true`
- a tombstone
- another governed state

The important property is that historical states remain queryable if the business requires them.

### 2.14 Unknown and inferred dimensions

A fact may arrive before its dimension.

A controlled unknown member can preserve referential integrity:

    customer_sk = 0
    customer_id = UNKNOWN

Later resolution must follow an explicit correction policy.

Do not silently mutate unknown history into fabricated customer attributes.

## 3. Implementation

### 3.1 Target schema

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
    ON dim_customer(customer_id)
    WHERE is_current;

Consider a stronger interval constraint when the database and warehouse design support it. The partial unique index is a minimum current-row guard, not a complete temporal-integrity solution.

### 3.2 Source staging

Stage normalized records before mutating history:

    CREATE TEMP TABLE customer_stage AS
    SELECT
        customer_id,
        customer_name,
        segment,
        region,
        effective_at,
        source_updated_at,
        source_record_id,
        source_run_id
    FROM raw_customer_changes;

Validate:

- business-key presence
- effective timestamp
- source identity
- tracked attribute types
- source grain
- duplicate records

### 3.3 Deterministic source ordering

For multiple source records per business key:

    SELECT *
    FROM (
        SELECT
            s.*,
            row_number() OVER (
                PARTITION BY customer_id, effective_at
                ORDER BY source_sequence DESC, source_record_id DESC
            ) AS rn
        FROM customer_stage s
    ) ranked
    WHERE rn = 1;

The exact deduplication rule depends on the source contract.

### 3.4 Initial version

For a new business key:

    INSERT INTO dim_customer (
        customer_id, customer_name, segment, region,
        valid_from, valid_to, is_current,
        source_updated_at, source_run_id
    )
    VALUES (
        :customer_id, :customer_name, :segment, :region,
        :effective_at, NULL, TRUE,
        :source_updated_at, :source_run_id
    );

The first effective timestamp becomes the beginning of known history.

### 3.5 Normal forward change

For a changed current state:

    BEGIN

      UPDATE dim_customer
      SET valid_to = :effective_at,
          is_current = FALSE
      WHERE customer_id = :customer_id
        AND is_current = TRUE;

      INSERT INTO dim_customer (
          customer_id, customer_name, segment, region,
          valid_from, valid_to, is_current,
          source_updated_at, source_run_id
      ) VALUES (
          :customer_id, :customer_name, :segment, :region,
          :effective_at, NULL, TRUE,
          :source_updated_at, :source_run_id
      );

    COMMIT;

Both mutations must be atomic.

### 3.6 Preventing duplicate versions

Before creating a new version, compare the incoming tracked attributes with the currently effective state.

Example:

    SELECT d.*
    FROM dim_customer d
    WHERE d.customer_id = :customer_id
      AND d.valid_from <= :effective_at
      AND (:effective_at < d.valid_to OR d.valid_to IS NULL);

If the incoming state matches the effective version, it is a no-op.

This is essential for replay safety.

### 3.7 Key-level concurrency

Two workers updating one business key can otherwise create invalid history.

Possible controls:

- partition work by business key
- lock the current row
- use advisory locks
- serialize through a key-aware queue
- enforce database constraints as a final guard

A transaction alone does not automatically prevent every logical race.

### 3.8 Effective-date update

Suppose current history is:

    [Jan 1, Jun 30) SMB
    [Jun 30, infinity) Enterprise

A late event effective Apr 15 says `Mid-Market`.

A correction workflow may reconstruct:

    [Jan 1, Apr 15) SMB
    [Apr 15, Jun 30) Mid-Market
    [Jun 30, infinity) Enterprise

Do not implement this by simply inserting another row with `valid_to = Jun 30`. The neighboring intervals must be recalculated and the result must be reconciled.

### 3.9 Interval reconstruction

For affected business keys, a robust correction algorithm can:

1. Collect existing versions.
2. Add the new effective state.
3. Sort all states by effective time and deterministic tie-breaker.
4. Remove or merge exact duplicate states according to policy.
5. Set each version's `valid_to` to the next version's `valid_from`.
6. Mark only the final version current.
7. Replace or update the affected history atomically.
8. Reconcile overlaps and gaps.

For large histories, limit reconstruction to the smallest affected range.

### 3.10 Rebuilding an affected timeline

A conceptual SQL pattern:

    SELECT
        customer_id,
        effective_at AS valid_from,
        lead(effective_at) OVER (
            PARTITION BY customer_id
            ORDER BY effective_at, source_sequence
        ) AS valid_to
    FROM customer_state_events;

Then:

    is_current = valid_to IS NULL

Window functions calculate the timeline, but publication should occur only after validating the reconstructed result.

### 3.11 Temporal fact resolution

For a fact at `occurred_at`:

    SELECT d.customer_sk
    FROM dim_customer d
    WHERE d.customer_id = :customer_id
      AND d.valid_from <= :occurred_at
      AND (d.valid_to > :occurred_at OR d.valid_to IS NULL);

Expected cardinality:

    exactly one dimension version

Zero means missing history. More than one means overlapping history.

Both are data-quality failures requiring investigation.

### 3.12 Fact re-resolution after historical correction

If a late dimension change modifies the historical version covering existing facts, the fact's dimension surrogate key may need to change.

Example:

    Fact F100 occurred Apr 20
    Before correction → customer_sk 101
    After correction  → customer_sk 155

If facts physically store the dimension surrogate key, the pipeline must define whether and how that foreign key is repaired.

This is a major reason late Type 2 corrections need bounded impact analysis.

### 3.13 Python orchestration pattern

Python should coordinate policy while PostgreSQL provides transactional storage.

    from dataclasses import dataclass
    from datetime import datetime

    @dataclass(frozen=True)
    class SCD2State:
        business_key: str
        effective_at: datetime
        segment: str | None
        region: str | None
        source_record_id: str

    def state_changed(old, new) -> bool:
        return (
            old.segment != new.segment
            or old.region != new.region
        )

Production code should use explicit NULL-safe semantics and canonicalization rather than relying blindly on Python equality for database values.

### 3.14 Idempotent mutation identity

Use durable source identity where possible:

    source_system
    + source_record_id
    + source_version

or another source-defined unique identity.

A run ID alone is insufficient because the same source record can be replayed in a different run.

### 3.15 Publication sequence

A safe workflow is:

    raw source
        ↓
    normalized staging
        ↓
    source deduplication
        ↓
    effective-time ordering
        ↓
    change detection
        ↓
    affected-key calculation
        ↓
    SCD2 mutation/reconstruction
        ↓
    interval reconciliation
        ↓
    temporal fact reconciliation
        ↓
    publish

Do not expose a partially reconstructed timeline to downstream readers.

## 4. Testing

### 4.1 Core scenario matrix

| Scenario | Expected result |
|---|---|
| New business key | One initial version |
| Unchanged state | No new version |
| Tracked attribute change | Close old + insert new |
| Type 1-only attribute change | No unintended version |
| NULL → value | New version |
| Value → NULL | New version |
| NULL → NULL | No new version |
| Duplicate source event | One logical application |
| Replay | No duplicate version |
| Multiple ordered changes | Correct interval chain |
| Same-time events | Deterministic winner/order |
| Late effective change | Correct interval split |
| Out-of-order events | Effective-time order preserved |
| Delete | Governed terminal state |
| Overlap attempt | Reconciliation blocks publication |
| Missing interval | Policy-specific handling |
| Concurrent change | One valid timeline |
| Fact at boundary | Exactly one dimension version |

### 4.2 Interval invariant tests

For every business key assert:

    valid_from < valid_to

for closed rows.

Then assert no pair of rows overlaps.

Also assert:

    count(*) FILTER (WHERE is_current) <= 1

and, where continuous coverage is required:

    previous.valid_to = current.valid_from

### 4.3 Temporal join tests

Create facts at:

- just before a change
- exactly at the change boundary
- just after the change
- inside a later interval

Each fact must resolve to exactly one expected dimension surrogate key.

### 4.4 Replay test

Run the same source batch twice.

Expected:

    first run  → required versions created
    second run → zero additional logical versions

Compare:

- row count
- surrogate keys
- validity intervals
- current-row flags
- source identities

### 4.5 Late-change test

Create two existing intervals, then inject a historical change between them.

Verify:

- old interval closes at the late event
- new interval begins there
- following interval begins at its original boundary
- no overlap exists
- facts in the affected range resolve correctly

### 4.6 Failure-path tests

Force failures:

- after closing the old version
- before inserting the new version
- during timeline reconstruction
- during temporal fact reconciliation
- during publication

Expected: no partially visible history.

## 5. Observability

### 5.1 Transformation metrics

Track:

- source rows
- new entities
- unchanged states
- new versions
- late changes
- out-of-order changes
- duplicate source events
- rejected states
- reconstructed timelines
- affected fact count

### 5.2 Dimension health

Track:

- dimension row count
- distinct business keys
- current-row count
- versions per business key
- overlap violations
- gap violations
- unknown-member usage
- historical correction count
- SCD churn

### 5.3 Temporal resolution metrics

Track:

- facts resolved to exactly one version
- facts with zero matching versions
- facts with multiple matching versions
- fact surrogate-key repairs
- unresolved historical corrections

Multiple matches should normally be zero.

### 5.4 Structured mutation logs

Useful fields:

    run_id
    business_key
    source_record_id
    old_dimension_sk
    new_dimension_sk
    effective_at
    previous_valid_to
    action
    policy_version

Log identifiers and metadata rather than unrestricted sensitive dimension values.

### 5.5 Alerts

Alert on:

- duplicate current rows
- interval overlaps
- unexpected gaps
- high SCD churn
- high late-event rate
- unusual number of fact re-resolutions
- zero/multiple temporal matches
- unknown-member growth

## 6. Intentional Failure

### Failure 1 — Duplicate current version

Attempt to create a second current row for the same business key.

Expected protection: current-row uniqueness constraint.

### Failure 2 — Overlapping history

Insert a version whose effective interval overlaps an existing version.

Expected: reconciliation detects it before publication.

### Failure 3 — Duplicate replay

Run the same source event twice.

Expected: one historical version, not two.

### Failure 4 — Wrong effective timestamp

Use load time instead of trusted business-effective time.

Expected symptom: facts resolve to incorrect historical states.

### Failure 5 — Late event treated as current

Insert an old effective event as a new open-ended current row.

Expected symptom: historical ordering is corrupted.

### Failure 6 — Crash after closing old row

Force a transaction failure between close and insert.

Expected: transaction rollback restores the old valid state.

### Failure 7 — Fact fan-out

Create overlapping dimension intervals and run a temporal fact join.

Expected: multiple matches expose the broken dimension invariant.

## 7. Recovery

### 7.1 Failed transaction

If close-and-insert or reconstruction is transactional, rollback and retry after diagnosing the cause.

### 7.2 Incorrect historical version

1. Identify affected business keys.
2. Capture source evidence and run identity.
3. Determine intended effective states.
4. Reconstruct only the affected timelines.
5. Reconcile interval invariants.
6. Re-resolve affected facts if required.
7. Publish atomically.
8. Record the correction.

### 7.3 Accidental overlap

Do not merely delete one row without understanding the intended timeline.

Rebuild the affected business key from authoritative source states when possible.

### 7.4 Fact repair

If historical correction changes dimension surrogate-key resolution:

    identify affected fact time range
          ↓
    re-run temporal resolution
          ↓
    compare old/new dimension keys
          ↓
    update only governed foreign keys
          ↓
    reconcile fact counts and measures

Do not alter fact measures simply because their dimension key changed.

### 7.5 Backfill strategy

For large corrections:

- isolate affected business keys
- bound the effective-time range
- stage reconstructed history
- validate before publication
- process affected facts separately
- reconcile before releasing downstream

### 7.6 Recovery verification

Verify:

- exactly one current row where required
- no prohibited overlaps
- expected gap behavior
- correct effective boundaries
- idempotent rerun
- correct temporal fact resolution
- downstream aggregates remain reconciled

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

Important mechanisms:

- partial unique indexes
- transactions
- row locking
- `IS DISTINCT FROM`
- window functions such as `LEAD`
- exclusion constraints for interval-oriented designs
- `EXPLAIN` for temporal joins

Learn the underlying SQL semantics before depending on framework abstractions.

### 8.2 dbt

dbt supports warehouse implementations of historical dimensions through models and snapshot functionality.

Useful concepts:

- snapshots
- unique keys
- incremental models
- schema tests
- relationship tests
- model dependencies

Understand the generated SQL and snapshot semantics. The tool does not decide your business effective-time policy for you.

### 8.3 Great Expectations

Useful expectations include:

- business-key uniqueness under defined conditions
- non-null effective timestamps
- accepted domain values
- expected row-count ranges
- schema contracts

For interval overlap and temporal cardinality, database queries and dedicated reconciliation tests are often still required.

## 9. Production Runbook

### Before deployment

- Define dimension grain.
- Define business key.
- Define surrogate-key policy.
- Identify Type 2 attributes.
- Define effective-time source.
- Define processing-time metadata.
- Define interval convention.
- Define gap policy.
- Define overlap policy.
- Define delete policy.
- Define late/out-of-order policy.
- Define idempotency identity.
- Define unknown-member behavior.
- Define fact re-resolution policy.
- Add current-row protection.
- Add interval reconciliation.

### Normal run

1. Register run.
2. Stage and normalize source.
3. Validate source grain.
4. Deduplicate observations.
5. Sort effective states deterministically.
6. Detect new/unchanged/changed states.
7. Apply normal forward versions.
8. Reconstruct affected timelines for approved late changes.
9. Reconcile intervals.
10. Resolve affected facts if required.
11. Publish atomically.
12. Record metrics and lineage.

### If current-row count is wrong

Check:

1. concurrent workers
2. duplicate source events
3. failed partial mutation
4. uniqueness constraint
5. replay behavior

### If overlap appears

Stop downstream publication.

Identify the affected business keys, reconstruct their timelines from authoritative states, validate, and republish only after reconciliation succeeds.

### If late changes spike

Check source delivery latency, source ordering, effective timestamps, timezone conversions, and upstream corrections.

Do not automatically widen the late-data acceptance window without understanding why lateness changed.

## 10. Common Mistakes

1. **Treating Type 2 as an ordinary upsert.** History requires version creation and interval management.
2. **Using load time as effective time without justification.** This changes historical meaning.
3. **Using arrival order as event order.** Late data can corrupt timelines.
4. **Not deduplicating source events.** Duplicate changes create duplicate versions.
5. **Allowing overlapping intervals.** Temporal joins can multiply facts.
6. **Ignoring gaps.** Whether gaps are valid must be explicit.
7. **Creating a new version for every snapshot.** Identical states should normally be no-ops.
8. **Using a single row hash for mixed Type 1/Type 2 attributes.** Corrections can create false historical changes.
9. **Failing to serialize business-key updates.** Concurrent workers can create competing timelines.
10. **Changing history without considering fact foreign keys.** Existing facts may require re-resolution.
11. **Inferring deletes from incomplete extracts.** Missing data is not necessarily deletion.
12. **Artificially adjusting timestamps to remove overlaps.** This changes business meaning.
13. **Deleting incorrect history without retaining source evidence.** Corrections need auditability.
14. **Skipping post-mutation reconciliation.** A successful transaction can still encode incorrect business logic.

## 11. Definition of Done

T47 is complete when you can independently:

- Explain Type 2 historical versioning.
- Define business and surrogate keys.
- Design effective-dated half-open intervals.
- Separate effective time from processing time.
- Detect tracked attribute changes safely.
- Create initial dimension versions.
- Close old versions and insert new versions atomically.
- Enforce one current row per business key.
- Detect and prevent interval overlaps.
- Define gap semantics.
- Handle multiple changes deterministically.
- Handle same-time and duplicate source events.
- Explain and implement bounded late-change correction.
- Perform temporal fact-to-dimension resolution.
- Identify when fact surrogate keys need repair.
- Make Type 2 processing idempotent.
- Test boundary timestamps.
- Test failure between close and insert.
- Observe SCD churn and temporal resolution health.
- Recover corrupted history safely.
- Explain the relevant production tools.

## 12. What You Learned

SCD Type 2 is fundamentally a temporal state-management problem.

The core mechanism is:

    BUSINESS KEY
         ↓
    EFFECTIVE STATE
         ↓
    CHANGE DETECTION
         ↓
    VERSION CREATION
         ↓
    NON-OVERLAPPING INTERVALS
         ↓
    TEMPORAL FACT RESOLUTION
         ↓
    RECONCILIATION
         ↓
    SAFE RECOVERY

The difficult part is not writing `UPDATE` and `INSERT`. The difficult part is preserving a trustworthy timeline when data arrives late, duplicates occur, workers race, source order differs from effective order, and historical corrections affect downstream fact keys.

Once this mechanism is understood, dimensional history becomes an explicit temporal model rather than an opaque warehouse feature.

### Next Recipe

**T48 — Fact Table Transformation**

Next, move from dimension history to fact modeling: establish fact grain, resolve dimension keys, handle measures and degenerate dimensions, validate cardinality, and publish reconciled fact rows safely.