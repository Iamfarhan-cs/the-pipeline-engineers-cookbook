# T33 — Required-Field Validation

> **Goal:** Learn how to enforce required-field contracts in production pipelines, distinguish missing, blank, invalid, and conditionally required values, and route failures without silently corrupting downstream data.

## 1. Problem Recognition

Required-field validation answers:

    Does every record contain the information that must exist for this pipeline stage?

Typical required fields include:

- Customer identifiers.
- Transaction identifiers.
- Event timestamps.
- Account identifiers.
- Currency codes.
- Status values.
- Source-system identifiers.
- Partition keys.
- Business dates.
- Mandatory reference fields.

A required field can fail in more ways than SQL `NULL`.

Examples:

    NULL
    empty string
    whitespace-only string
    malformed placeholder
    missing JSON property
    missing nested object
    invalid sentinel such as 'N/A'

### Recognition questions

Before implementing validation, ask:

1. Which fields are mandatory?
2. At which pipeline stage are they mandatory?
3. Is the rule unconditional or conditional?
4. Does blank text count as missing?
5. Are sentinel values treated as missing?
6. Is a zero value valid?
7. Is an empty array valid?
8. What happens to invalid records?
9. Which failures are recoverable?
10. Must invalid records be quarantined?
11. What percentage of failures is acceptable?
12. Can the rule change by schema version?

## 2. Concept and Reasoning

### 2.1 Required is a data contract

A required field is not merely a database preference.

It is a contract between:

    source
      ↓
    ingestion
      ↓
    transformation
      ↓
    downstream consumer

The contract should state what must exist and what constitutes a valid presence.

### 2.2 NULL versus empty string

These are different SQL values:

    NULL
    ''

A validation rule that checks only:

    column IS NOT NULL

can incorrectly accept an empty string.

### 2.3 Whitespace-only values

This is also not meaningful:

    '   '

For text fields, a common presence rule is conceptually:

    value IS NOT NULL
    AND TRIM(value) <> ''

Apply trimming according to the field's semantics.

### 2.4 Sentinel values

Some sources use:

    N/A
    UNKNOWN
    NONE
    NULL
    -

as placeholders.

Do not globally classify these as missing without a source-specific contract.

A real business field may legitimately contain a value such as `UNKNOWN`.

### 2.5 Zero is not missing

For numeric fields:

    0

may be valid.

Never use generic truthiness rules that classify zero as absent.

### 2.6 False is not missing

For Boolean fields:

    FALSE

is a valid value.

Do not write application validation equivalent to:

    if not value:

when false and missing have different meanings.

### 2.7 Empty collections

For an array or JSON field:

    []

may be valid presence but invalid business content.

Required-field validation and minimum-content validation are separate concepts.

### 2.8 Field presence versus field validity

These are different rules.

Presence:

    customer_id exists

Validity:

    customer_id matches the required format

Keep them separately observable.

### 2.9 Required fields depend on pipeline stage

A field may not exist in raw source data but become mandatory after enrichment.

Example:

    raw source
        ↓
    customer_id optional
        ↓
    customer mapping
        ↓
    customer_id required

Define requiredness per stage rather than globally.

### 2.10 Conditional requiredness

Some fields are required only when another field has a certain value.

Example:

    payment_method = CARD
    → card_token required

while:

    payment_method = BANK_TRANSFER
    → bank_account_id required

This is a business rule, not a simple null check.

### 2.11 Mutually exclusive requirements

A contract may require one of several fields:

    email OR phone

or:

    iban OR local_account_number

This is not equivalent to requiring both.

### 2.12 At-least-one requirements

Example:

    one of address_line_1, postal_code, or address_reference must exist

Define the rule explicitly rather than applying individual required-field checks.

### 2.13 All-or-none groups

Some fields must appear together.

Example:

    latitude + longitude

If one exists and the other does not, the record is incomplete.

### 2.14 Nested required fields

For JSON:

    customer.profile.first_name

you must distinguish:

    profile missing
    first_name missing
    first_name null
    first_name blank

These may require different error reasons.

### 2.15 Required fields and schema evolution

A field may become required in schema version 2.

Therefore validation should know:

    schema_version
    rule_version

This makes historical reprocessing explainable.

### 2.16 Required fields and source systems

Different sources may have different contracts.

Example:

    Source A → email required
    Source B → phone required

Do not force one source's assumptions onto another without an explicit normalization contract.

### 2.17 Required fields and normalization

Validation can occur before or after normalization.

Example:

    raw = '   ABC123   '

After normalization:

    'ABC123'

A field that appears blank before transformation may become valid after an approved normalization step.

Document the validation stage.

### 2.18 Required-field validation should be deterministic

The same input and rule version should produce the same result.

A validator should not depend on:

- Processing order.
- Current wall-clock time unless explicitly part of the rule.
- Random values.
- Non-deterministic source ordering.

### 2.19 Validation result model

A useful result contains:

    record_id
    field_name
    rule_id
    rule_version
    failure_code
    observed_value_state
    processing_timestamp

Do not necessarily store sensitive raw values.

### 2.20 Fail-fast versus collect-all

Two strategies exist.

Fail-fast:

    stop after first required-field failure

Collect-all:

    evaluate all required fields and return every failure

Collect-all is usually more useful for batch data-quality remediation because one record can have multiple missing fields.

### 2.21 Record-level versus field-level failure

Record-level:

    record is invalid

Field-level:

    customer_id missing
    currency missing

Use field-level diagnostics even if the downstream decision is record-level rejection.

### 2.22 Severity

Not every missing field has equal operational impact.

Possible severity classes:

    ERROR
    WARNING

Do not downgrade a business-critical identity field merely to improve quality metrics.

### 2.23 Reject versus quarantine

Rejecting a record can mean:

    do not publish it

Quarantine means:

    preserve the invalid record and reason for later handling

Quarantine is valuable when the source record may be corrected and replayed.

### 2.24 Required-field validation and ingestion

Raw data should usually be preserved before validation.

Preferred flow:

    acquire raw
       ↓
    preserve raw
       ↓
    validate
       ↓
    publish valid
       ↓
    quarantine invalid

### 2.25 Required-field validation and schema enforcement

Database constraints can enforce required fields:

    NOT NULL

But constraints do not replace transformation-level validation.

Why?

- Constraints usually do not explain source-specific failure reasons.
- Conditional requirements need transformation logic.
- Quarantine needs explicit handling.
- Raw source preservation may happen before the target table.

Use both where appropriate.

### 2.26 Validation before joins

Validate required join keys before joining.

Missing join keys can cause records to disappear or fail enrichment.

### 2.27 Validation after enrichment

Enrichment can create new required fields.

Example:

    customer_id
    ↓
    reference lookup
    ↓
    risk_segment required

Validate the enriched field at the stage where it becomes mandatory.

### 2.28 Validation and data contracts

A source contract should specify:

    field
    requiredness
    type
    format
    allowed values
    conditional dependencies

Required-field validation handles the requiredness dimension.

### 2.29 Requiredness and privacy

Validation logs should not expose sensitive values unnecessarily.

Prefer:

    field = email
    state = MISSING

over logging the full record.

### 2.30 Required-field validation and metrics

Useful measurements include:

    valid_records / input_records
    invalid_records / input_records
    missing_field_count
    failure_rate_by_field

Always retain the input population denominator.

## 3. Implementation

### 3.1 Define the contract

Example:

    record_id:       required
    customer_id:     required
    occurred_at:     required
    currency:        required
    amount:          required
    description:     optional
    payment_method:  required

Text presence:

    NULL and whitespace-only = missing

Numeric semantics:

    zero = valid

### 3.2 Example schema

    CREATE TABLE payment_stage (
        record_id TEXT,
        customer_id BIGINT,
        occurred_at TIMESTAMPTZ,
        currency TEXT,
        amount NUMERIC(20,4),
        description TEXT,
        payment_method TEXT
    );

### 3.3 Basic SQL validation

    SELECT
        *,
        CASE
            WHEN record_id IS NULL THEN 'MISSING_RECORD_ID'
            WHEN customer_id IS NULL THEN 'MISSING_CUSTOMER_ID'
            WHEN occurred_at IS NULL THEN 'MISSING_OCCURRED_AT'
            WHEN currency IS NULL OR BTRIM(currency) = ''
                THEN 'MISSING_CURRENCY'
            WHEN amount IS NULL THEN 'MISSING_AMOUNT'
            WHEN payment_method IS NULL OR BTRIM(payment_method) = ''
                THEN 'MISSING_PAYMENT_METHOD'
            ELSE NULL
        END AS validation_error
    FROM payment_stage;

This gives one primary error.

### 3.4 Collect all failures

For diagnostics, produce one row per failed field:

    SELECT record_id, 'customer_id' AS field_name, 'REQUIRED' AS rule_id
    FROM payment_stage
    WHERE customer_id IS NULL
    UNION ALL
    SELECT record_id, 'occurred_at', 'REQUIRED'
    FROM payment_stage
    WHERE occurred_at IS NULL
    UNION ALL
    SELECT record_id, 'currency', 'REQUIRED'
    FROM payment_stage
    WHERE currency IS NULL OR BTRIM(currency) = '';

This makes field-level quality reporting easier.

### 3.5 Valid-record filter

After validation, publish only records satisfying the required contract:

    SELECT *
    FROM payment_stage
    WHERE record_id IS NOT NULL
      AND customer_id IS NOT NULL
      AND occurred_at IS NOT NULL
      AND currency IS NOT NULL
      AND BTRIM(currency) <> ''
      AND amount IS NOT NULL
      AND payment_method IS NOT NULL
      AND BTRIM(payment_method) <> '';

Keep the rejected population separately rather than simply dropping it.

### 3.6 Conditional requirement

Example:

    SELECT
        *,
        CASE
            WHEN payment_method = 'CARD'
                 AND (card_token IS NULL OR BTRIM(card_token) = '')
                THEN 'MISSING_CARD_TOKEN'
            WHEN payment_method = 'BANK_TRANSFER'
                 AND (bank_account_id IS NULL OR BTRIM(bank_account_id) = '')
                THEN 'MISSING_BANK_ACCOUNT_ID'
            ELSE NULL
        END AS validation_error
    FROM payment_stage;

### 3.7 At-least-one requirement

Example:

    WHERE NULLIF(BTRIM(email), '') IS NOT NULL
       OR NULLIF(BTRIM(phone), '') IS NOT NULL

This means at least one contact method exists.

### 3.8 All-or-none requirement

Example:

    CASE
        WHEN (latitude IS NULL) <> (longitude IS NULL)
            THEN 'INCOMPLETE_GEOLOCATION'
        ELSE NULL
    END

This identifies exactly-one-present states.

### 3.9 Required JSON property

PostgreSQL JSONB can be checked explicitly:

    payload ? 'customer_id'

Then validate the extracted value:

    NULLIF(BTRIM(payload ->> 'customer_id'), '') IS NOT NULL

Presence and value validation remain separate.

### 3.10 Validation result table

    CREATE TABLE required_field_failures (
        record_id TEXT NOT NULL,
        field_name TEXT NOT NULL,
        rule_id TEXT NOT NULL,
        rule_version TEXT NOT NULL,
        failure_code TEXT NOT NULL,
        detected_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
    );

A production implementation may also include batch ID, source system, schema version, and quarantine reference.

### 3.11 Valid and invalid branches

Conceptually:

    input
      ↓
    required-field validation
      ├── valid → transformation
      └── invalid → quarantine

### 3.12 Python implementation

    def is_present(value):
        if value is None:
            return False
        if isinstance(value, str):
            return value.strip() != ''
        return True

    REQUIRED_FIELDS = [
        'record_id',
        'customer_id',
        'occurred_at',
        'currency',
        'amount',
        'payment_method',
    ]

    def validate_required_fields(row):
        failures = []

        for field in REQUIRED_FIELDS:
            if not is_present(row.get(field)):
                failures.append({
                    'field_name': field,
                    'failure_code': 'REQUIRED',
                })

        return failures

This deliberately does not classify zero or `False` as missing.

### 3.13 Conditional Python validation

    def validate_payment_method(row):
        method = row.get('payment_method')

        if method == 'CARD' and not is_present(row.get('card_token')):
            return 'MISSING_CARD_TOKEN'

        if method == 'BANK_TRANSFER' and not is_present(row.get('bank_account_id')):
            return 'MISSING_BANK_ACCOUNT_ID'

        return None

### 3.14 Record-level result

Return a structured result:

    {
        'record_id': 'P100',
        'valid': False,
        'failures': [
            {
                'field_name': 'currency',
                'failure_code': 'REQUIRED',
            }
        ]
    }

This separates validation from routing.

### 3.15 Do not log sensitive values

Prefer:

    record_id=P100 field=currency failure=MISSING

over:

    record={...full customer payload...}

### 3.16 Rule versioning

Attach:

    rule_version = required_fields_v1

to validation decisions.

When requiredness changes, publish a new rule version.

### 3.17 Batch validation

For batch processing, produce:

    input_count
    valid_count
    invalid_count
    failure_count

and:

    invalid_count + valid_count = input_count

unless records are explicitly classified into additional terminal states.

### 3.18 Database constraints

After validation, the target table can reinforce required fields:

    CREATE TABLE payment_fact (
        record_id TEXT PRIMARY KEY,
        customer_id BIGINT NOT NULL,
        occurred_at TIMESTAMPTZ NOT NULL,
        currency TEXT NOT NULL,
        amount NUMERIC(20,4) NOT NULL
    );

The transformation should still retain diagnostic failure records upstream.

### 3.19 Required-field validation before deduplication

If identity is missing, deduplication may be unsafe.

Preferred flow:

    required identity validation
         ↓
    deduplication

unless the source contract explicitly defines another sequence.

### 3.20 Required-field validation before enrichment

Validate keys needed for reference joins before attempting enrichment.

Missing keys should not silently become missing enrichment values.

## 4. Testing

Required-field validation must test every semantic state.

### 4.1 NULL

Input:

    customer_id = NULL

Expected:

    MISSING_CUSTOMER_ID

### 4.2 Empty string

Input:

    currency = ''

Expected:

    MISSING_CURRENCY

### 4.3 Whitespace

Input:

    currency = '   '

Expected:

    MISSING_CURRENCY

### 4.4 Zero

Input:

    amount = 0

Expected:

    valid

unless the business contract separately prohibits zero.

### 4.5 False

Input:

    is_verified = FALSE

Expected:

    valid presence

### 4.6 Optional field

Input:

    description = NULL

Expected:

    valid

### 4.7 Conditional field

Input:

    payment_method = CARD
    card_token = NULL

Expected:

    MISSING_CARD_TOKEN

### 4.8 Alternative field

Input:

    email = NULL
    phone = '+123'

Expected:

    valid

if the contract requires at least one contact method.

### 4.9 All-or-none group

Input:

    latitude = 10
    longitude = NULL

Expected:

    INCOMPLETE_GEOLOCATION

### 4.10 Multiple failures

Create a record missing:

    customer_id
    currency
    amount

Expected:

three field-level failures

when using collect-all validation.

### 4.11 Nested JSON

Test:

    property missing
    property NULL
    property blank
    property valid

### 4.12 Sentinel values

Test source-specific sentinel values such as:

    N/A
    UNKNOWN

and verify only configured sentinels are treated as missing.

### 4.13 Unicode whitespace

Test whitespace beyond a normal ASCII space where the source can contain it.

Use the chosen runtime's documented trimming semantics and normalize deliberately.

### 4.14 Schema-version test

Run the same record against two rule versions.

Verify requiredness changes only when the rule version changes.

### 4.15 Idempotence

Run validation twice.

Expected:

    same failure classifications
    same valid/invalid routing

### 4.16 Reconciliation

Verify:

    valid_records + invalid_records = input_records

and:

    field_failure_count >= invalid_record_count

when records can have multiple failures.

### 4.17 Quarantine preservation

Verify every invalid record has:

    original record reference
    failure reason
    rule version

## 5. Observability

Required-field validation needs field-level and record-level metrics.

### Core metrics

| Metric | Meaning |
|---|---|
| `required_validation_input_rows` | Records entering validation |
| `required_validation_valid_rows` | Records passing |
| `required_validation_invalid_rows` | Records failing |
| `required_validation_failure_count` | Total field-level failures |
| `required_validation_missing_field_rows` | Failures caused by absence |
| `required_validation_quarantine_rows` | Invalid records routed to quarantine |
| `required_validation_rule_version` | Active rule version |
| `required_validation_failure_rate` | Invalid / input |

### Field-level metrics

Track:

    failure_count by field

Example:

    currency → 0.3%
    customer_id → 0.02%
    amount → 4.1%

A field-level spike is often more actionable than the aggregate invalid rate.

### Failure-code distribution

Track:

    MISSING
    BLANK
    CONDITIONAL_MISSING
    INCOMPLETE_GROUP

or your governed failure taxonomy.

### Source-specific monitoring

Break quality metrics down by:

- Source system.
- Batch.
- Schema version.
- Pipeline stage.

This helps identify whether one producer is responsible for a quality regression.

### Quarantine aging

Monitor:

    oldest unresolved invalid record
    quarantine backlog
    remediation time

Validation is operationally incomplete if invalid records accumulate indefinitely.

### Privacy-safe diagnostics

Do not expose raw sensitive field values in metrics or logs.

Track field names and failure states instead.

## 6. Intentional Failure

### Failure 1 — Check only IS NOT NULL

Insert whitespace-only currency.

Expected symptom:

- Invalid record passes validation.

Recovery:

Treat blank text according to the field contract.

### Failure 2 — Use truthiness for required fields

Set amount to zero or a Boolean to false.

Expected symptom:

- Valid values are incorrectly rejected.

Recovery:

Use type-aware presence checks.

### Failure 3 — Require conditional fields unconditionally

Require `card_token` for bank transfers.

Expected symptom:

- Valid bank-transfer records fail.

Recovery:

Implement conditional requiredness.

### Failure 4 — Drop invalid records silently

Filter invalid rows without writing diagnostics.

Expected symptom:

- Input and output counts no longer explain the difference.

Recovery:

Preserve invalid records and failure reasons.

### Failure 5 — Validate after a required-key join

Join on a missing customer ID before validation.

Expected symptom:

- Records disappear before the quality layer can explain why.

Recovery:

Validate required join keys before enrichment.

### Failure 6 — Log complete invalid records

Write full payloads into logs.

Expected symptom:

- Sensitive data exposure risk increases.

Recovery:

Log identifiers and failure metadata only.

### Failure 7 — Ignore rule versions

Change required fields without recording the rule version.

Expected symptom:

- Historical validation results become difficult to explain.

Recovery:

Version the validation contract.

### Failure 8 — Treat every source the same

Apply one requiredness rule to incompatible source contracts.

Expected symptom:

- Valid source-specific records are rejected.

Recovery:

Define source-aware validation contracts.

## 7. Recovery

### Recovery sequence

1. Preserve the raw source batch.
2. Identify the failing field and rule.
3. Determine whether the failure is source, mapping, or validator logic.
4. Check the active rule version.
5. Inspect field-level failure rates.
6. Correct the source or validation rule.
7. Reprocess quarantined records.
8. Reconcile valid and invalid counts.
9. Verify downstream target constraints.
10. Record the root cause and regression test.

### Recovering from false rejection

1. Identify records incorrectly classified as invalid.
2. Correct the requiredness rule.
3. Revalidate from preserved raw data.
4. Compare old and new decision sets.
5. Publish newly valid records.

### Recovering from false acceptance

1. Identify records that bypassed required validation.
2. Determine affected downstream outputs.
3. Correct the validator.
4. Reprocess affected input.
5. Reconcile downstream data.

### Recovering from quarantine backlog

1. Group failures by reason.
2. Fix systematic source problems first.
3. Reprocess safe records in batches.
4. Keep unresolved exceptions isolated.
5. Monitor backlog age.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL provides `NOT NULL`, `CHECK`, filtered queries, constraints, and relational structures for enforcing and diagnosing required-field contracts.

### 2. dbt

dbt is useful for expressing model-level requiredness and testing accepted null behavior while keeping rules version-controlled.

### 3. Great Expectations

Great Expectations provides declarative data-quality expectations and validation reporting useful for field-level completeness checks.

Use it as a validation framework, not as a substitute for understanding the underlying contract.

## 9. Production Runbook

### Before deployment

- [ ] List every required field.
- [ ] Define blank semantics.
- [ ] Define sentinel semantics.
- [ ] Define zero/false semantics.
- [ ] Define conditional requirements.
- [ ] Define alternative-field rules.
- [ ] Define all-or-none groups.
- [ ] Define JSON/nested-field rules.
- [ ] Define rule version.
- [ ] Define quarantine behavior.
- [ ] Define privacy-safe logging.

### During execution

- [ ] Record input rows.
- [ ] Record valid rows.
- [ ] Record invalid rows.
- [ ] Record field-level failures.
- [ ] Record failure-code distribution.
- [ ] Record source-level quality.
- [ ] Record quarantine backlog.

### If invalid rate spikes

1. Check source system.
2. Check schema version.
3. Check recent mapping changes.
4. Check normalization changes.
5. Check validator deployment.
6. Compare failure rate by field.

### If valid count drops

1. Compare rule version.
2. Inspect new required fields.
3. Inspect conditional logic.
4. Check whitespace/sentinel handling.
5. Compare with previous batch.

### If target rejects records

1. Compare target constraints with validation rules.
2. Identify fields missing from pre-validation.
3. Add the missing validation rule.
4. Reprocess from preserved input.

## 10. Common Mistakes

### Mistake 1 — Checking only NULL

Blank and sentinel values can also represent missing data.

### Mistake 2 — Treating zero as missing

Numeric zero can be a valid business value.

### Mistake 3 — Treating false as missing

Boolean false is not absence.

### Mistake 4 — Requiring conditional fields globally

Requiredness can depend on other fields.

### Mistake 5 — Dropping invalid rows silently

Every rejected record should have an explainable reason.

### Mistake 6 — Validating too late

Missing identity can cause data loss during joins.

### Mistake 7 — Logging sensitive values

Validation diagnostics should be privacy-aware.

### Mistake 8 — No rule version

Changing validation rules without versioning damages reproducibility.

### Mistake 9 — Ignoring source-specific contracts

Different producers can have different valid representations.

### Mistake 10 — Treating presence as validity

A present value can still fail type, domain, or range validation.

### Mistake 11 — No denominator

An invalid count without the input population is difficult to interpret.

### Mistake 12 — No quarantine lifecycle

Invalid records require ownership, remediation, and replay.

## 11. Definition of Done

The required-field validation stage is complete when you can:

- [ ] Define requiredness as a data contract.
- [ ] Distinguish NULL, blank, whitespace, and sentinel states.
- [ ] Preserve valid zero and false values.
- [ ] Implement unconditional required fields.
- [ ] Implement conditional requiredness.
- [ ] Implement at-least-one rules.
- [ ] Implement all-or-none groups.
- [ ] Validate nested JSON fields.
- [ ] Version validation rules.
- [ ] Produce field-level failure diagnostics.
- [ ] Route invalid records to quarantine.
- [ ] Preserve raw input for replay.
- [ ] Enforce target constraints.
- [ ] Measure field-level failure rates.
- [ ] Reconcile valid + invalid populations.
- [ ] Protect sensitive values in logs.
- [ ] Intentionally break validation semantics.
- [ ] Recover from false acceptance and false rejection.
- [ ] Explain how PostgreSQL, dbt, and Great Expectations support production required-field validation.

## 12. What You Learned

Required-field validation is the first layer of a data-quality contract.

The production workflow is:

    DEFINE FIELD CONTRACT
         ↓
    DEFINE PRESENCE SEMANTICS
         ↓
    DEFINE CONDITIONAL RULES
         ↓
    VALIDATE
         ↓
    CLASSIFY FIELD FAILURES
         ↓
    ROUTE VALID / INVALID
         ↓
    QUARANTINE INVALID
         ↓
    MONITOR FAILURE RATES
         ↓
    REPLAY AFTER REMEDIATION

> **A required field is not simply a non-NULL column; it is a versioned contract defining when information must exist, what counts as present, and what happens when it does not.**

### Next recipe

**T34 — Type Validation**