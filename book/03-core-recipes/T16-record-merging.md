# T16 — Record Merging

> **Goal:** Combine multiple input records into one logical output record while preserving identity, deterministic survivorship, conflict handling, temporal correctness, provenance, accounting, and replay safety.

Record merging occurs when several records represent one logical entity or one output-grain record.

Examples:

    duplicate customer profiles → one canonical customer
    multiple source rows → one business entity
    partial records from systems A/B → one canonical record
    several observations → one latest-state record
    multiple records for one logical key → one consolidated record

The central risk is that merging can silently destroy information or choose a different winner on different runs.

The central rule is:

> **A merge is correct only when the merge identity, target grain, survivorship rules, conflict policy, and provenance are explicit and deterministic.**

---

## 1. Problem Recognition

### 1.1 Typical merging problem

Suppose three source records represent the same customer:

    source=A, customer_ref=42, name=Farhan, email=NULL
    source=B, customer_ref=42, name=NULL, email=farhan@example.com
    source=C, customer_ref=42, name=Farhan Khan, email=farhan@example.com

The target model requires one canonical customer record.

The transformation is:

    3 input records → 1 logical output record

The difficult part is deciding:

    which records belong together
    which value wins for each field
    what happens when values conflict
    how the decision can be reproduced later

### 1.2 Merging is not joining

A JOIN combines columns from related datasets according to a relationship.

A merge combines records that are intentionally being represented as one output-grain record.

Example:

    orders JOIN customers

adds customer attributes to orders.

Whereas:

    customer records A + B + C → canonical customer

creates one consolidated customer record.

### 1.3 Merging is not aggregation

Aggregation intentionally reduces records using mathematical or business functions:

    transactions → daily revenue

A merge may reduce record count without summing values.

Example:

    record A provides phone
    record B provides email
    record C provides address

These values may be consolidated into one entity without being aggregated.

### 1.4 Merging is not deduplication

Deduplication identifies duplicate records and decides which duplicate representation survives.

Merging may require combining complementary information from records that are not exact duplicates.

Example:

    A = {name, email}
    B = {name, phone}

Deduplication might discard B.

Merge logic can produce:

    {name, email, phone}

### 1.5 Red flags

Investigate when:

- multiple records share a supposed business identity
- different sources disagree about the same field
- output changes between repeated runs
- NULL values overwrite known values
- `first()` or `last()` appears without a deterministic ordering rule
- merge groups unexpectedly contain many records
- one merge group contains different real-world entities
- historical values are overwritten by current values
- downstream users cannot explain why a value won.

---

## 2. Concept and Reasoning

### 2.1 Define the merge contract

Before implementation, define:

    input grain
    target grain
    merge identity
    grouping rule
    expected group cardinality
    field-level survivorship rules
    conflict policy
    temporal rule
    source precedence
    provenance
    rule version
    accounting expectations

Example:

| Property | Contract |
|---|---|
| Input | one source customer record |
| Output | one canonical customer |
| Merge identity | normalized customer_id |
| Group size | one or more source records |
| Email rule | verified value wins; otherwise newest trusted value |
| Name rule | trusted source wins; otherwise newest non-null value |
| Conflict | quarantine if equally trusted values disagree |
| History | preserve source evidence |

### 2.2 Define the target grain first

The target grain answers:

> What does exactly one output row represent?

Examples:

    one customer
    one account
    one product
    one current entity state

If the answer is unclear, the merge is not ready to implement.

### 2.3 Define merge identity

A merge key must represent the same logical entity, not merely similar values.

Examples:

    customer_id
    account_id
    source_system + source_customer_id
    a governed canonical entity ID

Do not merge on weak keys such as:

    first_name
    city
    amount
    email alone

unless the business contract explicitly establishes that identity rule.

### 2.4 One logical group should represent one logical entity

Suppose:

    merge_key = 123

and five records belong to that group.

The merge contract must establish that all five records can safely represent one target entity.

If one merge key can represent multiple real-world entities, the merge will collapse distinct records and create silent corruption.

### 2.5 Field-level survivorship

Do not use one global rule such as:

    take the first row

Different fields may require different rules.

Example:

| Field | Survivorship rule |
|---|---|
| email | verified value, then source priority, then newest |
| phone | verified value, then newest |
| name | trusted source, then newest non-null |
| country | authoritative master source |
| created_at | minimum valid timestamp |
| updated_at | maximum valid timestamp |

### 2.6 Source precedence must be explicit

Suppose:

    CRM says phone = 111
    Billing says phone = 222

If CRM is authoritative for phone, encode that rule.

Do not rely on:

    database insertion order
    physical row order
    query planner behavior
    arbitrary `LIMIT 1`

### 2.7 Deterministic winner selection

A valid winner rule should produce the same result for the same input.

A typical ordering might be:

    verified DESC
    source_priority ASC
    updated_at DESC
    source_record_id ASC

The final stable tie-breaker matters.

Without it, two records can remain equally eligible and the selected value may depend on execution details.

### 2.8 NULL must not automatically win

Suppose:

    record A: email = a@example.com
    record B: email = NULL

A merge should normally not let the missing value erase known information unless the source contract explicitly says NULL means deletion.

Therefore distinguish:

    missing
    explicit null
    deletion/tombstone
    invalid value

The correct behavior depends on source semantics.

### 2.9 Conflict is different from missing data

These are different:

    A.email = NULL
    A.email = a@example.com
    B.email = b@example.com

The first is absence.
The latter two are a conflict.

Do not solve conflicts with `COALESCE`.

`COALESCE` answers:

    which non-null value should be used?

It does not answer:

    which conflicting non-null value is authoritative?

### 2.10 Temporal merge semantics

A merge may be:

    current-state merge
    event-time merge
    as-of historical merge

If the target represents current state, the latest valid record may win.

If the target represents historical state, collapsing all records into one current record destroys history.

Use event time or effective intervals when historical correctness matters.

### 2.11 Preserve source evidence

A canonical record should normally retain enough provenance to answer:

    which source records contributed?
    which value won?
    why did it win?
    which rule version made the decision?

Useful metadata:

    source_system
    source_record_id
    contributing_record_ids
    winning_source
    winning_record_id
    merge_rule_version
    merged_at

### 2.12 Merge accounting

For a batch:

    input_records = 1000
    merge_groups = 800
    output_records = 800
    conflicts = 12
    quarantined_groups = 3

The counts should be explainable.

A merge should not simply make rows disappear.

---

## 3. Implementation

### 3.1 Python implementation

Start with explicit data structures and a field-level winner function.

    from dataclasses import dataclass
    from datetime import datetime
    from typing import Optional

    @dataclass(frozen=True)
    class SourceRecord:
        merge_key: str
        source: str
        source_record_id: str
        name: Optional[str]
        email: Optional[str]
        verified_email: bool
        updated_at: datetime
        source_priority: int

    def choose_email(records: list[SourceRecord]) -> SourceRecord | None:
        candidates = [r for r in records if r.email is not None]
        if not candidates:
            return None

        return min(
            candidates,
            key=lambda r: (
                not r.verified_email,
                r.source_priority,
                -r.updated_at.timestamp(),
                r.source_record_id,
            ),
        )

    def merge_group(records: list[SourceRecord]) -> dict:
        if not records:
            raise ValueError("cannot merge an empty group")

        email_winner = choose_email(records)

        return {
            "merge_key": records[0].merge_key,
            "email": email_winner.email if email_winner else None,
            "email_source": email_winner.source if email_winner else None,
            "email_source_record_id": (
                email_winner.source_record_id if email_winner else None
            ),
            "merge_rule_version": "v1",
        }

The important design point is not the Python syntax.

It is that the winner rule is:

    explicit
    deterministic
    testable
    versioned

### 3.2 Validate merge keys before grouping

Normalize the key using the relevant contract, then validate:

    missing merge keys
    malformed keys
    unexpected key cardinality
    suspiciously large groups

Never silently put records with missing identity into one shared group such as:

    merge_key = NULL

unless that behavior is explicitly intended.

### 3.3 PostgreSQL grouping

Basic grouping identifies merge candidates:

    SELECT
        merge_key,
        COUNT(*) AS source_record_count
    FROM source_records
    GROUP BY merge_key;

Then inspect suspicious groups:

    SELECT
        merge_key,
        COUNT(*) AS source_record_count
    FROM source_records
    GROUP BY merge_key
    HAVING COUNT(*) > 10;

The threshold must come from the domain, not from an arbitrary universal value.

### 3.4 Deterministic winner with PostgreSQL

Use `ROW_NUMBER()` when selecting one winning record:

    SELECT *
    FROM (
        SELECT
            r.*,
            ROW_NUMBER() OVER (
                PARTITION BY merge_key
                ORDER BY
                    verified_email DESC,
                    source_priority ASC,
                    updated_at DESC,
                    source_record_id ASC
            ) AS rn
        FROM source_records r
        WHERE email IS NOT NULL
    ) ranked
    WHERE rn = 1;

The final `source_record_id` tie-breaker makes the ordering total.

### 3.5 Field-level merging

Different fields can use different ranked candidate sets.

Example:

    email → verified + source priority + recency
    phone → verification + recency
    country → authoritative source only

Do not select one winning row and copy every field from that row unless the business contract explicitly defines whole-record survivorship.

### 3.6 Conflict detection

Detect multiple distinct non-null values before choosing a winner:

    SELECT
        merge_key,
        COUNT(DISTINCT email) AS distinct_emails
    FROM source_records
    WHERE email IS NOT NULL
    GROUP BY merge_key
    HAVING COUNT(DISTINCT email) > 1;

This gives you a conflict population that can be:

    resolved by policy
    quarantined
    escalated
    audited

### 3.7 Safe use of COALESCE

This is acceptable when the contract says the first available source wins:

    COALESCE(authoritative_email, trusted_email, fallback_email)

It is not a conflict-resolution strategy when multiple populated fields disagree.

### 3.8 Temporal winner selection

For current-state merging, rank by the business update timestamp:

    ROW_NUMBER() OVER (
        PARTITION BY merge_key
        ORDER BY
            updated_at DESC,
            source_priority ASC,
            source_record_id ASC
    )

Do not use ingestion time as a substitute for business event time unless the contract explicitly defines ingestion order as authoritative.

### 3.9 Preserve contributing records

For an auditable canonical model, consider storing:

    merge_key
    canonical_record_id
    contributing_source
    contributing_source_record_id
    winning_field
    merge_rule_version

This can be a separate lineage table rather than duplicating all source payloads into the canonical table.

### 3.10 Idempotent merge

Given the same source snapshot and same rule version:

    run 1 → canonical result A
    run 2 → canonical result A

Do not generate a new canonical identity on every run unless the model explicitly requires versioned identities.

Persist deterministic keys or use a stable natural/canonical identity.

---

## 4. Testing

### 4.1 Minimum test matrix

| Test | Expected result |
|---|---|
| one input record | one output record |
| two complementary records | fields combine correctly |
| two conflicting records | documented winner or quarantine |
| all values NULL | documented null behavior |
| one NULL and one known value | known value survives when policy allows |
| equal-priority conflict | deterministic tie-breaker |
| missing merge key | reject/quarantine according to contract |
| same input repeated | identical output |
| oversized group | detected and handled |
| historical records | temporal rule respected |
| duplicate source record | handled according to identity contract |

### 4.2 Determinism test

Shuffle the same input records before every execution.

    input order A,B,C → result X
    input order C,A,B → result X
    input order B,C,A → result X

If the result changes, the merge rule depends on input order.

That is a production defect unless input order is explicitly part of the contract.

### 4.3 Conflict tests

Test at least:

    verified vs unverified
    authoritative vs non-authoritative
    newer vs older
    equal priority
    two different non-null values
    NULL vs known value
    invalid vs valid value

### 4.4 Accounting tests

Assert that the batch explains:

    input records
    distinct merge groups
    output records
    resolved conflicts
    unresolved conflicts
    quarantined groups

### 4.5 Idempotence test

Run the merge twice against identical input.

Verify:

    same canonical identity
    same field values
    same winner metadata
    no duplicate output

---

## 5. Observability

At minimum track:

    input_record_count
    merge_group_count
    output_record_count
    average_group_size
    maximum_group_size
    conflict_count
    unresolved_conflict_count
    quarantined_group_count
    winning_source_distribution
    missing_merge_key_count
    merge_rule_version

Useful metrics include:

    merge_groups_per_run
    records_per_merge_group
    conflict_rate
    quarantine_rate
    source_win_rate

### 5.1 Watch group-size changes

A sudden increase from:

    average group size = 1.3

to:

    average group size = 4.8

may indicate:

    identity collision
    source duplication
    key normalization change
    upstream regression

### 5.2 Watch winner distribution

If source A normally supplies 80% of email values and suddenly supplies 5%, investigate.

The merge output may still pass schema validation while its business semantics have changed.

### 5.3 Log decisions, not sensitive payloads

Prefer structured metadata:

    merge_key_hash
    winning_source
    winning_source_record_id
    rule_version
    conflict_type

Avoid logging complete personal records or sensitive fields merely for debugging.

---

## 6. Intentional Failure

### Failure 1 — Nondeterministic first row

Break the merge by selecting:

    first record

without an explicit ordering.

Run the query repeatedly or change execution conditions.

Expected lesson:

> Physical row order is not a business rule.

### Failure 2 — NULL overwrites known data

Create:

    A.email = known@example.com
    B.email = NULL

Apply an incorrect whole-row overwrite.

Expected lesson:

> Missing values and deletion semantics must be distinguished.

### Failure 3 — Conflicting non-null values

Create:

    A.email = a@example.com
    B.email = b@example.com

Remove the source-priority rule.

Expected lesson:

> Conflict resolution must be explicit rather than accidental.

### Failure 4 — Identity collision

Create two real entities sharing the merge key.

Expected lesson:

> A bad merge identity can destroy entity boundaries before any field-level rule runs.

### Failure 5 — Historical overwrite

Provide an older business record and a newer current record.

Use ingestion order incorrectly.

Expected lesson:

> Current-state merging and historical reconstruction are different problems.

### Failure 6 — Hidden group explosion

Change key normalization so unrelated values collapse into one key.

Expected lesson:

> Identity normalization can change merge cardinality and must be monitored.

---

## 7. Recovery

### 7.1 Stop downstream publication

If the merge rule is producing incorrect canonical records, stop publication of affected output when operationally possible.

Do not continue generating more incorrect state while investigating.

### 7.2 Identify the affected merge groups

Use:

    pipeline_run_id
    merge_rule_version
    affected source systems
    time range
    merge keys

to isolate the affected population.

### 7.3 Restore from preserved source evidence

Do not reconstruct the correct result from an already-corrupted canonical table when original inputs are available.

Replay from:

    immutable/raw input
    trusted staging data
    preserved source records

### 7.4 Fix the rule

Version the merge policy.

Example:

    merge_rule_version = v1
    merge_rule_version = v2

Document exactly what changed:

    source precedence
    field survivorship
    identity logic
    temporal ordering
    conflict handling

### 7.5 Re-run deterministically

Replay the affected groups with the corrected rule.

Verify:

    output count
    canonical identity
    field values
    conflict count
    provenance
    downstream reconciliation

### 7.6 Audit downstream consumers

Determine whether incorrect canonical values were propagated to:

    dimensions
    facts
    reports
    APIs
    exports
    downstream services

Correct the downstream state according to its own recovery contract.

---

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

Know how to use:

    GROUP BY
    COUNT(DISTINCT ...)
    ROW_NUMBER()
    CASE
    COALESCE
    unique constraints
    indexes

PostgreSQL helps implement deterministic candidate selection and conflict detection.

### 8.2 Python

Know how to implement:

    field-level survivorship
    deterministic sorting
    conflict classification
    merge-group validation
    idempotent canonical output

Python is useful when merge policy is too complex or domain-specific for a single SQL statement.

### 8.3 dbt

Know how to use dbt for:

    modelized merge logic
    tests for uniqueness and relationships
    documentation of business rules
    version-controlled transformation SQL

Do not treat dbt tests as a substitute for explicit survivorship logic.

---

## 9. Production Runbook

### Before deployment

Verify:

- target grain is documented
- merge identity is documented
- merge-key quality is validated
- field-level survivorship rules exist
- source precedence is explicit
- conflict policy is explicit
- temporal semantics are explicit
- deterministic tie-breakers exist
- provenance is captured
- rule version is recorded
- accounting metrics exist
- representative conflict cases are tested.

### During execution

Monitor:

    input count
    merge-group count
    output count
    group-size distribution
    conflict rate
    quarantine count
    source winner distribution
    rule version

### If output looks wrong

1. Stop or isolate affected publication.
2. Identify the merge rule version.
3. Compare input groups before and after the merge.
4. Check merge-key cardinality.
5. Inspect field-level conflicts.
6. Verify source precedence.
7. Verify timestamps and temporal logic.
8. Replay from preserved source data.
9. Reconcile downstream outputs.
10. Document the rule correction.

---

## 10. Common Mistakes

### Mistake 1 — `first()` without ordering

Why it fails:

    first ≠ authoritative

### Mistake 2 — One winner for every field

Why it fails:

Different fields may have different authoritative sources.

### Mistake 3 — Using COALESCE to resolve conflicts

Why it fails:

COALESCE handles null availability, not conflicting populated values.

### Mistake 4 — Merging on weak identity

Why it fails:

Similar values do not necessarily represent the same entity.

### Mistake 5 — Ignoring temporal semantics

Why it fails:

A current value can overwrite historically correct state.

### Mistake 6 — Dropping source provenance

Why it fails:

You cannot explain or reproduce the canonical value.

### Mistake 7 — Ignoring oversized groups

Why it fails:

A sudden group explosion can indicate identity corruption or upstream duplication.

### Mistake 8 — Treating merge as deduplication

Why it fails:

Complementary records may contain information that would be lost if one record were simply discarded.

### Mistake 9 — Generating unstable canonical IDs

Why it fails:

Repeated processing creates new identities for the same logical entity.

### Mistake 10 — Mutating the only copy of source evidence

Why it fails:

Recovery becomes reconstruction instead of replay.

---

## 11. Definition of Done

A record-merging implementation is complete when:

- [ ] target grain is explicitly defined
- [ ] merge identity is documented
- [ ] merge-key quality is validated
- [ ] expected group cardinality is understood
- [ ] field-level survivorship rules are explicit
- [ ] source precedence is explicit
- [ ] conflict behavior is defined
- [ ] NULL and deletion semantics are understood
- [ ] temporal behavior is defined
- [ ] deterministic tie-breakers exist
- [ ] provenance is retained
- [ ] rule version is recorded
- [ ] input/output/group accounting exists
- [ ] normal cases are tested
- [ ] conflict cases are tested
- [ ] ordering independence is tested
- [ ] idempotence is tested
- [ ] intentional failure has been exercised
- [ ] recovery has been tested
- [ ] observability is available
- [ ] production runbook exists
- [ ] downstream reconciliation is understood.

---

## 12. What You Learned

Record merging is not simply putting rows together.

You learned how to:

    define a merge target grain
    establish a safe merge identity
    separate merge from join, aggregation, and deduplication
    define field-level survivorship
    resolve conflicts deterministically
    handle NULL correctly
    preserve temporal semantics
    retain provenance
    account for every input and output
    implement merging in Python and PostgreSQL
    test order independence and idempotence
    observe merge behavior
    intentionally break a merge
    recover from a bad merge rule
    operate the transformation in production

The key lesson is:

> **Never let row order, NULL behavior, or query execution details decide which business record survives. The merge contract must decide.**

### Next Recipe

**T17 — SQL SELECT Transformations**

After learning how to merge records safely, the next recipe moves into SQL-specific transformation mechanics.