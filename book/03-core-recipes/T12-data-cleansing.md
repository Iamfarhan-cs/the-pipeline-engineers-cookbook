# T12 — Data Cleansing

> **Goal:** Detect and safely correct, remove, isolate, or reject data that violates defined quality rules without silently changing valid business meaning.

Data cleansing is often described as “fixing bad data.” That is too vague for production systems.

A production cleansing step must answer:

    What is wrong?
    How do we know it is wrong?
    Can it be corrected deterministically?
    What evidence supports the correction?
    What happens when it cannot be corrected safely?

The central rule is:

> **Never turn an uncertain correction into a silent transformation.**

---

## 1. Problem Recognition

### 1.1 Typical cleansing problems

Real pipelines encounter:

    missing required fields
    malformed emails
    impossible dates
    negative quantities where negatives are forbidden
    duplicate identifiers
    broken relationships
    invalid codes
    inconsistent formats
    impossible combinations of fields
    corrupted numeric values
    stale or contradictory records

These are data-quality problems. They should not all be handled with the same generic cleaning function.

### 1.2 Cleansing is not normalization

Normalization changes representation:

    "  Alice  " → "Alice"

Standardization aligns equivalent representations:

    "USA" → "US"

Cleansing establishes whether data is acceptable and, when safe, repairs known defects:

    quantity = -3 → reject
    malformed date → quarantine
    known formatting defect → deterministic correction

T05 teaches string normalization. T11 teaches cross-source standardization. T12 focuses on **data-quality correction and disposition**.

### 1.3 Red flags

Investigate when:

- invalid-record rates increase
- downstream constraints reject rows
- the same defect appears repeatedly
- analysts manually repair source data
- cleansing logic differs between pipelines
- records are silently dropped
- corrections cannot be explained
- a “clean” dataset contains values outside the domain contract.

---

## 2. Concept and Reasoning

### 2.1 Define a cleansing contract

For every rule, define:

    field
    condition
    severity
    correction policy
    disposition
    evidence
    owner

Example:

| Rule | Condition | Action |
|---|---|---|
| quantity_nonnegative | quantity < 0 | reject |
| email_shape | invalid email structure | quarantine |
| trim_known_field | surrounding whitespace | normalize |
| country_valid | country not canonical | quarantine |
| start_before_end | start_time > end_time | reject |

The rule must be explicit before implementation.

### 2.2 Not every invalid value should be “fixed”

Consider:

    quantity = -5

If negative quantities are impossible, changing it to `5` is not cleansing. It invents a value.

Safer options may be:

    reject
    quarantine
    request source correction
    preserve but exclude from downstream model

Only apply a correction when the correct value can be determined from reliable evidence.

### 2.3 Correction confidence

Useful classification:

    deterministic correction
    rule-based correction with evidence
    ambiguous defect
    uncorrectable defect

Examples of deterministic correction:

    known accidental surrounding whitespace
    documented legacy token replacement
    known source formatting bug

Examples of ambiguous correction:

    13/14/2026 as a date
    quantity = 100 when source unit is unknown
    two possible customer identifiers.

Ambiguous data should not be guessed into validity.

### 2.4 Validation versus cleansing

Validation asks:

    “Is this acceptable?”

Cleansing asks:

    “If it is not acceptable, is there a safe, evidence-based correction or disposition?”

A useful pipeline is:

    raw
      ↓
    validate
      ↓
    classify defect
      ↓
    deterministic correction
      ↓
    validate again
      ↓
    accept / quarantine / reject

### 2.5 Preserve before/after evidence

When a value is corrected, retain enough metadata to explain it:

    source_value
    cleaned_value
    rule_id
    rule_version
    cleaned_at
    pipeline_run_id

This is particularly important for regulated, financial, and operational data.

### 2.6 Cleansing must be idempotent

A cleansing rule should normally satisfy:

    clean(clean(x)) = clean(x)

Example:

    " Alice " → "Alice" → "Alice"

If repeated execution keeps changing the value, the rule is not stable.

### 2.7 Order matters

Rules can depend on each other.

Example:

    trim → validate length

may be correct, while:

    validate length → trim

may incorrectly reject a value that only contains accidental surrounding whitespace.

Define the transformation sequence explicitly.

### 2.8 Never hide upstream defects

A successful pipeline run does not prove good data.

Track:

    accepted
    corrected
    quarantined
    rejected
    unchanged

A rising correction rate can be an upstream incident even when the ETL job remains green.

---

## 3. Implementation

## 3.1 Define explicit rules

Represent cleansing rules as named functions rather than a large undocumented cleaning block.

    from dataclasses import dataclass
    from typing import Any

    @dataclass(frozen=True)
    class CleanResult:
        status: str
        value: Any
        rule_id: str | None = None
        reason: str | None = None

    def clean_quantity(value: int | None) -> CleanResult:
        if value is None:
            return CleanResult("quarantine", value, "quantity_missing", "required field")

        if value < 0:
            return CleanResult("reject", value, "quantity_nonnegative", "negative quantity")

        return CleanResult("accepted", value)

This makes the disposition explicit.

## 3.2 Deterministic correction

Suppose the source contract guarantees that surrounding whitespace is accidental:

    def clean_customer_name(value: str | None) -> CleanResult:
        if value is None:
            return CleanResult("quarantine", None, "customer_name_missing", "required field")

        cleaned = " ".join(value.strip().split())

        if cleaned == "":
            return CleanResult("quarantine", value, "customer_name_empty", "empty after cleanup")

        if cleaned != value:
            return CleanResult("corrected", cleaned, "customer_name_whitespace_v1")

        return CleanResult("accepted", value)

Only perform this correction because the contract explicitly defines it as safe.

## 3.3 Validate after correction

    def validate_customer_name(value: str) -> None:
        if not value:
            raise ValueError("customer name cannot be empty")
        if len(value) > 200:
            raise ValueError("customer name exceeds maximum length")

    result = clean_customer_name(raw_name)

    if result.status in {"accepted", "corrected"}:
        validate_customer_name(result.value)

Never assume a corrected value is automatically valid.

## 3.4 Rule registry

For larger pipelines, keep rules discoverable:

    CLEANING_RULES = {
        "customer_name": [clean_customer_name],
        "quantity": [clean_quantity],
    }

Execute them in an explicit order and record the rules that changed the record.

## 3.5 Record-level cleansing

Some defects require multiple fields.

Example:

    def validate_order_window(start, end) -> None:
        if start is None or end is None:
            raise ValueError("order window is incomplete")
        if start > end:
            raise ValueError("start must not be after end")

Do not cleanse one field independently when the actual rule is relational.

## 3.6 SQL cleansing

SQL is useful when the transformation belongs close to relational data:

    SELECT
        id,
        NULLIF(BTRIM(customer_name), '') AS customer_name,
        quantity
    FROM staging_orders;

Then validate the result:

    SELECT id, quantity
    FROM staging_orders
    WHERE quantity IS NOT NULL
      AND quantity < 0;

Do not assume that replacing invalid values with NULL is always correct. The NULL policy must be part of the contract.

## 3.7 Quarantine instead of silent deletion

Create a durable quarantine record when required:

    CREATE TABLE data_quality_quarantine (
        quarantine_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
        pipeline_run_id uuid NOT NULL,
        record_id text NOT NULL,
        rule_id text NOT NULL,
        rule_version text NOT NULL,
        reason text NOT NULL,
        raw_payload jsonb,
        created_at timestamptz NOT NULL DEFAULT now()
    );

Store only the evidence required by the privacy and retention policy.

## 3.8 Batch cleansing with explicit accounting

For every batch, maintain:

    input_count
    accepted_count
    corrected_count
    quarantined_count
    rejected_count

A useful invariant is:

    input_count = accepted + corrected + quarantined + rejected

If corrected records are counted separately, document whether `accepted_count` includes corrected records. Avoid ambiguous metrics.

## 3.9 Prevent accidental repeated cleansing

Persist rule metadata where auditability matters:

    cleaning_rule_version = "customer_name_whitespace_v1"

A record already transformed by the same idempotent rule should not be repeatedly rewritten just because the pipeline was replayed.

---

## 4. Testing

Test the rule and the disposition, not only the final value.

### 4.1 Safe correction

    def test_customer_name_whitespace_is_corrected():
        result = clean_customer_name("  Alice  ")
        assert result.status == "corrected"
        assert result.value == "Alice"

### 4.2 Invalid data is not guessed

    def test_negative_quantity_is_rejected():
        result = clean_quantity(-5)
        assert result.status == "reject"
        assert result.value == -5

### 4.3 Idempotence

    def test_cleaning_is_idempotent():
        first = clean_customer_name("  Alice  ")
        second = clean_customer_name(first.value)
        assert second.value == "Alice"

### 4.4 Revalidation

Test that a correction is validated by the downstream rule.

Examples:

- whitespace correction followed by length validation
- date repair followed by date-range validation
- numeric correction followed by domain-range validation.

### 4.5 Record-level rules

Test combinations:

- missing start and present end
- present start and missing end
- start after end
- equal start and end when equality is allowed
- valid interval.

### 4.6 Accounting invariants

Test that every input record receives exactly one final disposition unless the pipeline explicitly models multiple stages.

Example:

    assert input_count == accepted_count + corrected_count + quarantined_count + rejected_count

### 4.7 Adversarial inputs

Include:

- empty strings
- whitespace-only strings
- nulls
- boundary lengths
- negative numbers
- huge values
- malformed dates
- Unicode edge cases
- unexpected types
- duplicate identifiers.

---

## 5. Observability

Monitor cleansing as a data-quality signal, not merely as ETL housekeeping.

| Signal | Why it matters |
|---|---|
| input records | Baseline for accounting |
| corrected records | Detects source-quality drift |
| correction rate | Detects changing source behavior |
| quarantined records | Detects unresolved defects |
| rejected records | Detects contract violations |
| rule-level counts | Identifies dominant defects |
| source-level counts | Identifies problematic producers |
| post-clean validation failures | Detects unsafe corrections |
| rule version | Explains transformation behavior |

Useful structured fields:

    pipeline_run_id
    source_system
    dataset
    record_id
    rule_id
    rule_version
    disposition

Do not log sensitive raw payloads by default.

---

## 6. Intentional Failure

### Failure A — Silent correction

Change:

    -5 → 5

without evidence that the sign is a source defect.

Expected result:

- the implementation should reject or quarantine
- tests should prevent arbitrary correction.

### Failure B — Correction creates a new invalid value

Use a cleaning rule that converts a value but leaves it outside the target domain.

Expected result:

- post-clean validation fails
- the record is not silently accepted.

### Failure C — Rule order is wrong

Run validation before a documented safe normalization step.

Expected result:

- valid-after-cleaning records may be incorrectly rejected.

### Failure D — Silent deletion

Drop invalid rows without recording their disposition.

Expected result:

- input/output accounting reveals a mismatch
- quarantine or rejection accounting is missing.

### Failure E — Non-idempotent cleaning

Create a rule that changes the value differently on each execution.

Expected result:

- replay produces different output
- idempotence tests fail.

---

## 7. Recovery

### Scenario 1 — Cleansing rule is too aggressive

1. Identify the rule and version.
2. Determine the first affected pipeline run.
3. Stop or quarantine downstream publication if necessary.
4. Compare raw/staged evidence with cleaned output.
5. Correct or disable the rule.
6. Reprocess affected records from preserved source evidence.
7. Re-run validation.
8. Reconcile record counts and affected business measures.

### Scenario 2 — New defect appears

1. Identify the source and failing rule.
2. Determine whether the value is invalid or a legitimate new representation.
3. Do not immediately add a cleanup rule.
4. Confirm the upstream contract or business meaning.
5. Add a deterministic rule only when justified.
6. Version and test the rule.
7. Replay quarantined records.
8. Monitor the defect rate after recovery.

### Scenario 3 — Cleansed data escaped downstream

1. Identify the affected time range and rule version.
2. Determine which downstream datasets consumed the values.
3. Preserve affected outputs for investigation.
4. Correct the rule.
5. Rebuild or replay affected partitions.
6. Reconcile downstream totals and quality metrics.
7. Record the incident and prevention action.

---

## 8. Production Tools You Should Know

### 8.1 Python

Useful for deterministic cleansing functions, rule composition, unit tests, and source-specific defect handling.

Keep the business rule explicit and testable.

### 8.2 PostgreSQL

Useful for SQL-side cleansing, constraints, quarantine tables, and accounting queries.

Database constraints are especially useful for proving that invalid data cannot cross a storage boundary.

### 8.3 dbt

Useful for version-controlled SQL transformations, data-quality tests, and documenting cleansing logic in analytical pipelines.

The tool does not decide whether a defect should be corrected; that remains a domain decision.

---

## 9. Production Runbook

### Before deployment

- [ ] Every cleansing rule has a documented reason.
- [ ] Corrections are deterministic or explicitly evidence-based.
- [ ] Unknown/ambiguous defects have a disposition.
- [ ] Rule order is documented.
- [ ] Rule versions are tracked.
- [ ] Source evidence is retained where required.
- [ ] Post-clean validation exists.
- [ ] Input/output accounting exists.

### During execution

- [ ] Monitor correction rate.
- [ ] Monitor quarantine rate.
- [ ] Monitor rejection rate.
- [ ] Monitor rule-level defect counts.
- [ ] Monitor source-level drift.
- [ ] Monitor post-clean validation failures.

### When the cleansing rate spikes

1. Identify the first affected source and run.
2. Identify the dominant rule.
3. Compare current values with previous healthy batches.
4. Check upstream releases and contract changes.
5. Determine whether the rule or source changed.
6. Quarantine uncertain records.
7. Correct the rule or upstream data contract.
8. Replay affected records.
9. Reconcile before downstream publication.

---

## 10. Common Mistakes

### Mistake 1 — Treating cleansing as “make everything valid”

Some values cannot be safely repaired.

### Mistake 2 — Guessing missing information

Turning unknown data into a plausible value creates false certainty.

### Mistake 3 — Silently dropping records

Every disposition should be explainable.

### Mistake 4 — Cleansing without revalidation

A corrected value can still violate the target contract.

### Mistake 5 — No rule ownership

Unowned cleansing logic becomes permanent technical debt.

### Mistake 6 — No provenance

Without before/after evidence, debugging and audit become difficult.

### Mistake 7 — Non-idempotent rules

Replay can progressively corrupt data.

### Mistake 8 — One giant cleaning function

Separate rules make testing, versioning, observability, and rollback possible.

---

## 11. Definition of Done

A production-grade cleansing implementation is complete when you can:

- [ ] Define explicit cleansing rules and dispositions.
- [ ] Distinguish validation from correction.
- [ ] Identify when correction is deterministic.
- [ ] Reject or quarantine ambiguous defects.
- [ ] Preserve required before/after evidence.
- [ ] Validate again after correction.
- [ ] Control rule order.
- [ ] Prove idempotence where required.
- [ ] Maintain input/output accounting.
- [ ] Test normal, invalid, boundary, and adversarial inputs.
- [ ] Observe correction, quarantine, and rejection rates.
- [ ] Intentionally break the implementation.
- [ ] Diagnose the failure from operational evidence.
- [ ] Recover by correcting the rule and replaying safely.
- [ ] Explain the relevant production tools.
- [ ] Operate the mechanism using the runbook.

---

## 12. What You Learned

Data cleansing is controlled defect handling, not cosmetic cleanup.

The production reasoning pattern is:

    detect
      ↓
    classify
      ↓
    correct only when justified
      ↓
    validate again
      ↓
    accept / quarantine / reject
      ↓
    account and observe

A pipeline that silently changes uncertain data can be more dangerous than one that rejects it. Good cleansing makes defects explicit, corrections deterministic, and recovery possible.

---

## Next Recipe

**T13 — Record Filtering**