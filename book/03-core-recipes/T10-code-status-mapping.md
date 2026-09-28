# T10 — Code/Status Mapping

> **Goal:** Translate source-system codes and statuses into stable canonical domain values without silently misclassifying unknown, retired, or time-dependent values.

Code and status mapping appears simple:

    source: "P"
    target: "PENDING"

But production systems rarely have one permanent vocabulary.

A source may use:

    P
    pending
    WAIT
    01
    A
    ACTIVE

while another system uses completely different values for the same business concept.

Even more importantly, a code can change meaning over time.

The central rule is:

> **Never guess the meaning of an unknown code.**

---

## 1. Problem Recognition

### 1.1 Typical mapping problem

Suppose a payment provider sends:

    INIT
    AUTH
    SETT
    FAIL
    REV

Your canonical model might require:

    INITIALIZED
    AUTHORIZED
    SETTLED
    FAILED
    REVERSED

The transformation must explicitly define the mapping.

### 1.2 Mapping is not string normalization

String normalization changes representation:

    " pending " → "pending"

Status mapping changes domain meaning:

    "P" → "PENDING"

These are different mechanisms.

T05 teaches string normalization. T10 teaches **semantic vocabulary translation**.

### 1.3 Red flags

Investigate when:

- unknown source codes start appearing
- a source adds a new status
- status counts suddenly change
- multiple source codes map to one canonical value
- one canonical value maps from different source meanings
- a code's meaning changed after an upstream release
- analysts maintain different CASE expressions for the same status
- mappings are embedded in application code with no ownership or version.

---

## 2. Concept and Reasoning

## 2.1 Source vocabulary versus canonical vocabulary

Keep the two concepts separate.

Example:

    Source system:
        OPN
        HLD
        CLS

    Canonical domain:
        OPEN
        ON_HOLD
        CLOSED

The source vocabulary belongs to the source contract.

The canonical vocabulary belongs to your domain model.

### 2.2 Mapping direction

Prefer:

    source_code → canonical_code

rather than relying on an informal collection of reverse mappings.

Why?

Different source codes can legitimately map to the same canonical state:

    "P"   → PENDING
    "WAIT" → PENDING

The reverse mapping is therefore not necessarily one-to-one.

### 2.3 Many-to-one mappings

Many source states can represent the same downstream business state.

Example:

    AUTH_PENDING → PENDING
    COMPLIANCE_PENDING → PENDING
    MANUAL_REVIEW → PENDING

This is valid only if the downstream domain intentionally does not need to distinguish those states.

Preserve the original source code when the distinction may matter for audit, reconciliation, or future modeling.

### 2.4 One-to-many mappings are dangerous

A single source code cannot safely map to different canonical meanings without additional context.

Example:

    source_code = "A"

If `A` means ACTIVE for one date range and ARCHIVED for another, the mapping needs effective dates or another discriminating field.

Do not make a context-dependent mapping look like a static dictionary.

### 2.5 Unknown versus invalid

An unknown code can mean:

- upstream introduced a legitimate new value
- the source contract changed
- data is corrupted
- the mapping configuration is incomplete.

Treat unknown values as an observable state requiring policy.

Possible policies:

    reject
    quarantine
    map to UNKNOWN
    preserve raw code and continue

The correct choice depends on downstream safety requirements.

### 2.6 Mapping tables are often better than hard-coded CASE logic

For stable, governed mappings, a lookup table can provide:

- ownership
- effective dates
- audit history
- review workflow
- versioning
- source-specific mappings.

Example conceptual table:

    source_system
    source_code
    canonical_code
    effective_from
    effective_to
    mapping_version
    is_active

### 2.7 Effective-dated mappings

Suppose:

    A → ACTIVE

until June 30, and afterward:

    A → ARCHIVED

The mapping is no longer simply:

    A → ACTIVE

It is:

    source_code + event_date → canonical_code

Effective dating prevents historical records from being interpreted using today's vocabulary.

### 2.8 Event time versus processing time

When mappings are time-dependent, prefer the timestamp defined by the business contract.

Do not automatically use pipeline processing time.

Historical data arriving today should normally be interpreted according to the mapping applicable to the event or source record's effective date, if that is what the contract specifies.

### 2.9 Versioned mappings

A mapping can be versioned:

    mapping_version = 1
    mapping_version = 2

Versioning is useful when a transformation rule changes and you need to explain which rule produced a historical canonical value.

### 2.10 Preserve source evidence

A useful canonical record can contain:

    source_status = "HLD"
    canonical_status = "ON_HOLD"
    mapping_version = 3

This supports auditability and reprocessing.

### 2.11 Status transitions are different from status mapping

Mapping asks:

    "What does this source status mean?"

State-machine validation asks:

    "Is this transition allowed?"

Example:

    PENDING → SETTLED

may be valid, while:

    SETTLED → PENDING

may require special handling.

Do not overload the mapping layer with transition logic. Transition validation is a separate mechanism.

---

## 3. Implementation

## 3.1 Define a mapping contract

Example:

| Source system | Source code | Canonical value | Unknown policy |
|---|---|---|---|
| payments-api | `P` | `PENDING` | quarantine |
| payments-api | `A` | `AUTHORIZED` | quarantine |
| payments-api | `S` | `SETTLED` | quarantine |
| payments-api | `F` | `FAILED` | quarantine |

Also define:

- case sensitivity
- whitespace policy
- effective dates
- owner
- mapping version
- whether raw source code is preserved.

## 3.2 Simple dictionary mapping

For small, static vocabularies:

    STATUS_MAP = {
        "P": "PENDING",
        "A": "AUTHORIZED",
        "S": "SETTLED",
        "F": "FAILED",
    }

    def map_status(value: str) -> str:
        try:
            return STATUS_MAP[value]
        except KeyError as exc:
            raise ValueError(f"unknown status code: {value!r}") from exc

Failing explicitly is safer than:

    STATUS_MAP.get(value, "UNKNOWN")

unless the business contract explicitly says unknown values may continue as `UNKNOWN`.

## 3.3 Preserve raw and canonical values

    def transform_status(source_code: str) -> dict:
        canonical = map_status(source_code)

        return {
            "source_status": source_code,
            "canonical_status": canonical,
        }

Preserving the raw value makes incident investigation and mapping evolution easier.

## 3.4 Mapping with an explicit unknown policy

    def map_status(value: str, unknown_policy: str = "reject") -> str | None:
        if value in STATUS_MAP:
            return STATUS_MAP[value]

        if unknown_policy == "null":
            return None

        if unknown_policy == "unknown":
            return "UNKNOWN"

        if unknown_policy == "reject":
            raise ValueError(f"unknown status code: {value!r}")

        raise ValueError(f"unsupported unknown policy: {unknown_policy!r}")

Make the policy explicit and observable.

## 3.5 Lookup-table implementation

Example PostgreSQL table:

    CREATE TABLE status_mapping (
        mapping_id bigserial PRIMARY KEY,
        source_system text NOT NULL,
        source_code text NOT NULL,
        canonical_code text NOT NULL,
        effective_from timestamptz NOT NULL,
        effective_to timestamptz,
        mapping_version integer NOT NULL,
        created_at timestamptz NOT NULL DEFAULT now(),
        UNIQUE (
            source_system,
            source_code,
            effective_from,
            mapping_version
        )
    );

Treat mapping data as governed configuration rather than arbitrary application data.

## 3.6 Effective-dated lookup

A simplified lookup can select the mapping valid at an event time:

    SELECT canonical_code, mapping_version
    FROM status_mapping
    WHERE source_system = %(source_system)s
      AND source_code = %(source_code)s
      AND effective_from <= %(event_time)s
      AND (effective_to IS NULL OR %(event_time)s < effective_to)
    ORDER BY effective_from DESC
    LIMIT 1;

The half-open interval:

    [effective_from, effective_to)

avoids overlapping boundary ownership when configured consistently.

Production implementations should also enforce that overlapping active mappings cannot exist for the same source vocabulary.

## 3.7 Mapping data as configuration

Keep mapping ownership explicit.

Example:

    source_system: payments-api
    mapping_owner: payments-data-team
    mapping_version: 4
    approved_by: data-governance

The exact governance process depends on the organization, but mapping changes should be reviewable.

## 3.8 Normalize before mapping only when required

If the source contract permits case/whitespace normalization:

    normalized = source_code.strip().upper()
    canonical = map_status(normalized)

Do not normalize blindly when codes are case-sensitive.

## 3.9 SQL CASE mapping

For small transformations near a trusted boundary:

    CASE source_status
        WHEN 'P' THEN 'PENDING'
        WHEN 'A' THEN 'AUTHORIZED'
        WHEN 'S' THEN 'SETTLED'
        WHEN 'F' THEN 'FAILED'
        ELSE NULL
    END

If unknown codes are unacceptable, do not allow `ELSE NULL` to hide the problem. Add an explicit validation path.

## 3.10 Validate canonical domain values

After mapping, validate the target vocabulary:

    ALLOWED_CANONICAL = {
        "PENDING",
        "AUTHORIZED",
        "SETTLED",
        "FAILED",
    }

    def validate_canonical(value: str) -> str:
        if value not in ALLOWED_CANONICAL:
            raise ValueError(f"invalid canonical status: {value!r}")
        return value

Never assume the mapping configuration itself is correct simply because it loaded successfully.

---

## 4. Testing

### 4.1 Known mapping tests

    def test_known_status():
        assert map_status("P") == "PENDING"
        assert map_status("A") == "AUTHORIZED"

### 4.2 Unknown code test

    def test_unknown_status_is_rejected():
        try:
            map_status("NEW_STATUS")
            assert False
        except ValueError:
            pass

Unknown behavior must match the configured policy.

### 4.3 Many-to-one mapping

Test that multiple source codes intentionally map to the same canonical value:

    assert map_status("AUTH_PENDING") == "PENDING"
    assert map_status("COMPLIANCE_PENDING") == "PENDING"

Also verify that the raw source code remains available when auditability requires it.

### 4.4 Mapping completeness

Maintain a fixture containing all source codes expected by the current contract.

Fail the test when a newly documented source code is not mapped.

### 4.5 Canonical vocabulary test

Every mapping result must belong to the approved canonical domain.

    for source_code, canonical in STATUS_MAP.items():
        assert canonical in ALLOWED_CANONICAL

### 4.6 Effective-date tests

Test a boundary:

    before effective_from
    exactly at effective_from
    immediately before effective_to
    exactly at effective_to

For a mapping:

    2026-01-01 → 2026-07-01

the first mapping should apply before July 1 and the next mapping should apply at July 1.

### 4.7 Historical replay

Replay historical records using their event dates.

Verify that a mapping change made today does not reinterpret historical records incorrectly.

### 4.8 Overlap detection

Test that two active mappings cannot both apply to the same:

    source_system + source_code + event_time

Overlapping mappings are a configuration defect.

### 4.9 Mapping version tests

Verify that a transformed record can identify the mapping version used when the domain requires auditability.

### 4.10 Idempotence

Mapping a canonical value should not accidentally remap it through a source vocabulary unless the contract explicitly supports that operation.

Keep source-to-canonical mapping one-directional.

---

## 5. Observability

Track mapping behavior as data-quality and contract signals.

| Signal | Why it matters |
|---|---|
| unknown code count | Detect vocabulary changes |
| mapped record count | Measure transformation volume |
| mapping distribution | Detect business/source changes |
| mapping version usage | Understand active rule versions |
| quarantine count | Detect unsafe unknown values |
| canonical status distribution | Detect downstream behavior changes |
| overlapping mapping count | Detect configuration defects |

Example structured event:

    {
      "event": "status_mapping_failed",
      "source_system": "payments-api",
      "source_code": "SUSPENDED_NEW",
      "reason": "unknown_code",
      "mapping_version": 4,
      "record_id": "payment_123"
    }

Be careful with source codes that may themselves contain sensitive information.

### 5.1 Distribution monitoring

A sudden change from:

    PENDING  → 40%
    AUTHORIZED → 30%
    SETTLED → 25%
    FAILED → 5%

to a radically different distribution can indicate either a real business change or a mapping/source problem.

Investigate before declaring either explanation.

---

## 6. Intentional Failure

### Failure 1 — Unknown codes silently become UNKNOWN

Change the parser to:

    STATUS_MAP.get(value, "UNKNOWN")

Expected symptom:

- upstream contract changes stop generating visible failures
- downstream data appears complete while new vocabulary is hidden.

Diagnosis:

- compare raw source vocabulary with mapping configuration
- inspect unknown-code metrics.

### Failure 2 — Wrong canonical mapping

Map:

    "F" → "SETTLED"

instead of:

    "F" → "FAILED"

Expected symptom:

- status distributions become incorrect
- downstream business logic may treat failed records as successful.

Recovery:

- restore the mapping
- identify the affected processing window
- reprocess from raw source data.

### Failure 3 — Current mapping applied to historical data

Change a mapping today and replay old events using today's rule.

Expected symptom:

- historical reports change unexpectedly
- old source statuses receive new meanings.

### Failure 4 — Overlapping effective dates

Create two active mappings for the same source code and overlapping time range.

Expected symptom:

- different executions may choose different mappings
- results become dependent on query ordering.

Recovery:

- stop affected transformation
- remove the overlapping configuration
- determine the authoritative mapping
- replay affected records.

### Failure 5 — Normalize case incorrectly

Uppercase a case-sensitive code system.

Expected symptom:

- valid source codes become invalid or map incorrectly.

---

## 7. Recovery

### 7.1 Wrong mapping deployed

1. Identify the mapping version and deployment window.
2. Freeze or isolate the incorrect mapping.
3. Determine affected source records.
4. Restore the approved mapping.
5. Reprocess from trusted raw evidence.
6. Reconcile canonical status counts.
7. Verify downstream aggregates and state transitions.
8. Preserve the incident and mapping version history.

### 7.2 New source code

1. Capture the raw unknown code.
2. Confirm its meaning with the source owner.
3. Decide whether it maps to an existing canonical state or requires a new state.
4. Add the mapping with an effective date.
5. Add tests.
6. Deploy the mapping.
7. Reprocess quarantined records if appropriate.

### 7.3 Incorrect historical mapping

1. Identify the historical effective period.
2. Determine the correct mapping for each period.
3. Recompute canonical values from raw source codes.
4. Reconcile affected historical reports.
5. Preserve mapping versions so the correction is explainable.

### 7.4 Overlapping mapping configuration

1. Disable the ambiguous configuration.
2. Identify all affected source/time combinations.
3. Select the approved mapping.
4. Reprocess deterministically.
5. Add a configuration-level overlap test.

---

## 8. Production Tools You Should Know

### 8.1 PostgreSQL lookup tables

Know how to model mapping configuration with:

- source system
- source code
- canonical code
- effective dates
- version
- ownership.

Database constraints can prevent invalid or duplicate configuration.

### 8.2 Python mapping structures

Know when to use:

- dictionaries for small static mappings
- explicit validation
- typed configuration
- mapping version metadata.

Do not let a dictionary become an undocumented source of business rules.

### 8.3 Data contract / governance systems

In larger environments, code/status vocabularies are often governed as reference data or data contracts.

Understand the operational concepts:

- ownership
- approval
- versioning
- effective dates
- change notification.

The specific platform may vary; the engineering mechanism remains the same.

---

## 9. Production Runbook

### Symptom: unknown codes suddenly increase

Check:

1. Upstream release notes.
2. Raw source vocabulary.
3. Mapping version.
4. Effective dates.
5. Source contract.

### Symptom: status distribution changes suddenly

Check:

1. Mapping changes.
2. New source values.
3. Upstream business changes.
4. Historical versus current event-time mapping.

### Symptom: historical reports changed after a deployment

Check:

1. Effective-dated mappings.
2. Mapping version.
3. Replay behavior.
4. Whether current rules were applied to historical events.

### Symptom: different jobs produce different canonical statuses

Check:

1. Mapping configuration version.
2. Embedded CASE expressions.
3. Effective-date logic.
4. Query ordering around overlapping mappings.

### Symptom: source code is valid but rejected

Check:

1. Case sensitivity.
2. Whitespace normalization.
3. Mapping activation date.
4. Source-system identifier.

---

## 10. Common Mistakes

### Mistake 1 — Treating status mapping as string replacement

Mapping is semantic translation, not text cleanup.

### Mistake 2 — Silently mapping unknown codes

Unknown vocabulary is an important contract signal.

### Mistake 3 — Losing the raw source code

Raw evidence makes investigation and replay possible.

### Mistake 4 — Hard-coding mappings everywhere

Multiple independent CASE expressions drift over time.

### Mistake 5 — Ignoring effective dates

Today's mapping may not describe yesterday's source semantics.

### Mistake 6 — Allowing overlapping mappings

Ambiguous configuration produces nondeterministic transformations.

### Mistake 7 — Using processing time for historical mappings

Mapping should use the contractually relevant event/effective time.

### Mistake 8 — Confusing mapping with state-transition validation

Knowing what a status means is different from validating whether a transition is allowed.

### Mistake 9 — Widening accepted vocabulary without review

Permissiveness can hide upstream changes.

### Mistake 10 — Testing only current records

Historical replay is essential when mappings are time-dependent.

---

## 11. Definition of Done

T10 is complete when you can:

- distinguish source vocabulary from canonical vocabulary
- implement a deterministic source-to-canonical mapping
- handle many-to-one mappings intentionally
- recognize dangerous one-to-many mappings
- distinguish unknown from invalid values
- choose an explicit unknown-value policy
- preserve raw source codes
- use lookup tables for governed mappings
- implement effective-dated mappings
- version mapping rules
- prevent overlapping mappings
- validate canonical vocabulary
- test historical replay
- monitor unknown-code and mapping distributions
- intentionally reproduce silent-unknown and wrong-mapping failures
- recover incorrect mappings from trusted raw evidence.

---

## 12. What You Learned

The central lesson is:

> **A status code is a domain vocabulary, not just a string.**

Before mapping a source code, ask:

1. What does the source code mean?
2. What canonical concept should it represent?
3. Is the mapping one-to-one or many-to-one?
4. Could the meaning change over time?
5. What happens to unknown codes?
6. Should the raw code be preserved?
7. Which mapping version produced the canonical value?
8. Can historical data be replayed using the correct rule?

If those answers are explicit, status mapping becomes governed data transformation rather than scattered business logic.

---

## Next Recipe

**T11 — Data Standardization**

T11 will cover cross-source standardization of representations, units, categories, naming conventions, and canonical domain formats.