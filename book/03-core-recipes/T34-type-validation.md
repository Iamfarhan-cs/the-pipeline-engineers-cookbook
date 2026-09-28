# T34 — Type Validation

> **Goal:** Learn how to verify that incoming values conform to their declared data types, distinguish safe parsing from silent coercion, and prevent malformed values from corrupting downstream models.

## 1. Problem Recognition

Type validation answers:

    Can this value be represented safely as the type required by the pipeline contract?

Typical fields include:

- Integer identifiers.
- Decimal monetary amounts.
- Boolean flags.
- Dates and timestamps.
- Strings.
- JSON objects.
- Arrays.
- UUIDs.

Common failures include:

    '123'          → expected integer
    '12.50'        → expected decimal
    'TRUE'         → expected boolean
    '2026-09-28'   → expected timestamp
    'abc'          → cannot parse as numeric
    9999999999999  → overflow for target integer

Type validation is not the same as type conversion.

Conversion asks:

    Can I transform this value?

Validation asks:

    Is that transformation permitted and safe under the contract?

### Recognition questions

Before implementing type validation, ask:

1. What is the source representation?
2. What is the target type?
3. Is coercion allowed?
4. Which formats are accepted?
5. What precision and scale are required?
6. What timezone is required?
7. What range does the target type support?
8. What happens on overflow?
9. What happens on malformed input?
10. Are NULL values handled separately?
11. Is the conversion lossless?
12. Should invalid values be quarantined or rejected?

## 2. Concept and Reasoning

### 2.1 Type is part of the data contract

A field contract should define:

    logical type
    physical representation
    accepted formats
    coercion policy
    failure policy

Example:

    amount
      logical type: DECIMAL
      scale: 2
      source format: decimal string
      coercion: allowed
      overflow: reject

### 2.2 Logical type versus physical representation

An API may send:

    amount = '125.50'

while the analytical model requires:

    NUMERIC(20,2)

The source representation is text.

The business type is monetary decimal.

Do not confuse the two.

### 2.3 Validation versus parsing

Parsing attempts to interpret a value.

Validation determines whether the interpretation satisfies the contract.

Example:

    '2026-09-28T10:00:00Z'

may parse successfully as a timestamp.

But if the contract requires:

    America/New_York local timestamp

additional validation is required.

### 2.4 Strict versus permissive conversion

Strict:

    '123abc' → reject

Permissive:

    '123abc' → 123

Permissive conversion can silently corrupt data.

For production pipelines, prefer explicit accepted formats.

### 2.5 Numeric conversion

Numeric validation includes:

- Syntax.
- Sign.
- Decimal separator.
- Precision.
- Scale.
- Range.

Example:

    123.45

may be valid for:

    NUMERIC(10,2)

while:

    123.456

may require rounding or rejection.

### 2.6 Precision versus scale

For a decimal type:

    NUMERIC(10,2)

means:

    precision = 10 total digits
    scale = 2 digits after decimal

The maximum integer portion is constrained accordingly.

Do not assume the database will preserve arbitrary precision.

### 2.7 Monetary values

Use exact decimal types for money.

A floating-point representation can introduce binary rounding artifacts.

Type validation should therefore enforce an appropriate decimal representation before financial aggregation.

### 2.8 Integer conversion

Check both syntax and range.

Example:

    '2147483648'

cannot safely fit in a signed 32-bit integer.

A parser that accepts the string but later overflows is not a complete validation strategy.

### 2.9 Boolean conversion

Boolean source formats vary:

    true
    false
    TRUE
    FALSE
    1
    0
    yes
    no

Do not accept all representations automatically.

Define the allowed domain.

### 2.10 Date validation

Dates can be syntactically valid but semantically invalid for the contract.

Examples:

    2026-09-28
    28/09/2026
    09/28/2026

These can represent the same date under different conventions.

Use explicit format contracts.

### 2.11 Timestamp validation

Timestamps require additional decisions:

- Date format.
- Time format.
- Fractional seconds.
- Offset.
- Timezone.
- DST behavior.

Example:

    2026-09-28T10:00:00Z

is explicitly UTC.

An offset-free timestamp is not automatically UTC.

### 2.12 Timezone-aware types

Prefer timezone-aware representations when the event has a real instant in time.

Do not attach UTC arbitrarily to an unknown local timestamp.

Unknown timezone information should be a validation or data-quality issue when the business contract requires an instant.

### 2.13 String validation

Strings can fail type contracts through:

- Binary data.
- Invalid encoding.
- Unexpected control characters.
- Excessive length.

Type validation can check the representation before later semantic validation.

### 2.14 UUID validation

A UUID-like string is not necessarily a valid UUID.

Validate:

    syntax
    version if required
    canonical representation if required

### 2.15 JSON validation

JSON can be syntactically valid but have the wrong top-level type.

Example:

    expected object
    received array

Type validation should distinguish:

    invalid JSON syntax
    valid JSON with wrong shape
    valid JSON with correct shape

### 2.16 Arrays

An array field may require:

    expected JSON array

rather than:

    any JSON value

Element types may require separate validation.

### 2.17 Nested types

A nested object can have a schema:

    customer = {
        id: integer,
        name: string
    }

Type validation should distinguish missing properties from wrong property types.

### 2.18 NULL is not a type error by itself

NULL often means unknown or absent.

Required-field validation decides whether NULL is allowed.

Type validation decides whether a non-NULL value has the expected type.

Keep the two failure classes separate.

### 2.19 Empty string is not automatically a type error

An empty string is still a string.

It may violate a required-field or semantic rule.

Do not incorrectly report every blank value as a type failure.

### 2.20 Type coercion and lossiness

Conversions can lose information.

Examples:

    DECIMAL → INTEGER
    TIMESTAMP → DATE
    high-precision decimal → float
    timezone-aware timestamp → timezone-free timestamp

If conversion loses information, require explicit policy.

### 2.21 Rounding

Converting:

    10.999

to a two-decimal value can produce:

    11.00

or:

    10.99

depending on rounding mode.

Financial transformations require a documented rounding rule.

### 2.22 Truncation

String truncation can silently destroy information.

Example:

    500-character description

into:

    VARCHAR(100)

Validate length before truncating unless truncation is explicitly permitted.

### 2.23 Overflow

Overflow is a type-validation failure.

Example:

    source integer > target integer maximum

Do not rely on downstream database exceptions as the only quality mechanism.

### 2.24 Underflow

Small numeric values can be lost when converted to lower-precision representations.

Scientific and financial data may require special handling.

### 2.25 Type validation and schema drift

A source can change:

    amount: string → object

without changing the field name.

Type validation catches this form of schema drift.

### 2.26 Type validation and ingestion

Preserve the raw representation before coercion when auditability matters.

Preferred flow:

    raw
      ↓
    type validation
      ↓
    safe conversion
      ↓
    normalized value

### 2.27 Type validation and joins

Join keys should have compatible logical types.

Joining:

    integer customer_id

to:

    free-form customer_id text

can hide malformed identifiers and cause unexpected casts.

Normalize keys before the join.

### 2.28 Type validation and partitioning

Partition fields must have predictable types.

A date partition represented sometimes as a date and sometimes as a string can create inconsistent storage layouts.

### 2.29 Type validation and ordering

Lexical ordering differs from numeric ordering.

Example:

    '100' < '20'

as strings may be true lexically, while:

    100 > 20

is true numerically.

Never rely on implicit casts for analytical ordering.

### 2.30 Type validation and reproducibility

Conversion rules must be deterministic.

Document:

    parser
    accepted formats
    timezone
    rounding mode
    overflow policy
    rule version

## 3. Implementation

### 3.1 Define the contract

Example:

    customer_id: BIGINT
    amount: NUMERIC(20,4)
    currency: TEXT
    occurred_at: TIMESTAMPTZ
    is_recurring: BOOLEAN

Accepted source forms:

    customer_id → decimal string
    amount → decimal string
    occurred_at → ISO-8601 timestamp with offset
    is_recurring → true/false

### 3.2 Example staging schema

Keep source representations in staging when required:

    CREATE TABLE payment_raw_stage (
        record_id TEXT,
        customer_id_raw TEXT,
        amount_raw TEXT,
        currency_raw TEXT,
        occurred_at_raw TEXT,
        is_recurring_raw TEXT
    );

### 3.3 PostgreSQL safe integer parsing

PostgreSQL casts can raise errors for malformed values.

Do not cast unvalidated arbitrary input blindly when one malformed row should not abort the whole batch.

A safer pattern is to classify obvious invalid syntax first or use a controlled staging function.

### 3.4 Regex-gated integer conversion

Example:

    CASE
        WHEN customer_id_raw ~ '^[0-9]+$'
            THEN customer_id_raw::BIGINT
        ELSE NULL
    END AS customer_id

This handles syntax but not every range requirement.

Add explicit range validation when needed.

### 3.5 Decimal conversion

Example:

    CASE
        WHEN amount_raw ~ '^[+-]?[0-9]+(\.[0-9]+)?$'
            THEN amount_raw::NUMERIC(20,4)
        ELSE NULL
    END AS amount

Validate accepted numeric formats before conversion.

### 3.6 Decimal precision validation

Do not assume casting to a target scale is always semantically safe.

Validate whether the input has more fractional digits than permitted when the policy is reject rather than round.

### 3.7 Boolean mapping

Explicit mapping:

    CASE LOWER(BTRIM(is_recurring_raw))
        WHEN 'true' THEN TRUE
        WHEN 'false' THEN FALSE
        ELSE NULL
    END AS is_recurring

If `1/0` or `yes/no` are allowed, add them explicitly to the contract.

### 3.8 Timestamp parsing

Use an explicit accepted timestamp format.

Example:

    '2026-09-28T10:15:00+00:00'

should be interpreted as a timestamp with an explicit offset.

Do not treat an offset-free value as UTC without a contract.

### 3.9 Date parsing

Use a fixed source format where possible.

For ISO date strings:

    CASE
        WHEN date_raw ~ '^[0-9]{4}-[0-9]{2}-[0-9]{2}$'
            THEN date_raw::DATE
        ELSE NULL
    END AS business_date

The database still validates whether the date itself exists.

### 3.10 UUID validation

Use a controlled UUID parser or database cast and classify failures separately.

Do not silently retain malformed UUID strings as identifiers.

### 3.11 JSON type validation

For JSONB payloads, validate expected shape before downstream extraction.

Conceptually:

    JSON syntax valid
        ↓
    top-level object?
        ↓
    required properties?
        ↓
    property types correct?

### 3.12 Python decimal validation

    from decimal import Decimal, InvalidOperation

    def parse_decimal(value):
        if value is None:
            return None

        try:
            return Decimal(str(value))
        except (InvalidOperation, ValueError):
            return None

Production code should separately report whether parsing failed rather than using `None` as the only signal.

### 3.13 Python integer validation

    def parse_integer(value):
        if value is None:
            return None

        text = str(value).strip()
        if not text.isdigit():
            return None

        return int(text)

This accepts only unsigned decimal digits.

If signed integers are allowed, define that explicitly.

### 3.14 Python Boolean validation

    TRUE_VALUES = {'true'}
    FALSE_VALUES = {'false'}

    def parse_boolean(value):
        if value is None:
            return None

        normalized = str(value).strip().lower()
        if normalized in TRUE_VALUES:
            return True
        if normalized in FALSE_VALUES:
            return False
        return None

Keep accepted representations intentionally narrow.

### 3.15 Python timestamp validation

Use a timezone-aware parser and reject values that violate the contract.

Do not silently attach a timezone to an offset-free timestamp unless the contract explicitly defines the source timezone.

### 3.16 Validation result model

Use a result structure such as:

    {
        'record_id': 'P100',
        'field_name': 'amount',
        'expected_type': 'NUMERIC(20,4)',
        'failure_code': 'INVALID_TYPE',
        'rule_version': 'type_v1'
    }

Do not store sensitive raw values unnecessarily.

### 3.17 Safe conversion pipeline

Use separate concepts:

    raw_value
        ↓
    syntax validation
        ↓
    parse
        ↓
    range / precision validation
        ↓
    normalized typed value

This prevents a successful parser from being mistaken for complete validation.

### 3.18 Invalid branch

Conceptually:

    raw
      ↓
    type validation
      ├── valid → typed staging
      └── invalid → quarantine

### 3.19 Typed target table

    CREATE TABLE payment_typed_stage (
        record_id TEXT PRIMARY KEY,
        customer_id BIGINT NOT NULL,
        amount NUMERIC(20,4) NOT NULL,
        currency TEXT NOT NULL,
        occurred_at TIMESTAMPTZ NOT NULL,
        is_recurring BOOLEAN NOT NULL
    );

Target constraints provide a final invariant after validation.

### 3.20 Range and type are separate

A value can be correctly typed but invalid for the business range.

Example:

    amount = NUMERIC
    amount = -5000000

Type validation succeeds.

Range validation may fail.

Keep these recipe responsibilities separate.

## 4. Testing

Type validation requires tests for syntax, boundaries, coercion, precision, and lossiness.

### 4.1 Valid integer

Input:

    '123'

Expected:

    123::BIGINT

### 4.2 Invalid integer

Input:

    '123abc'

Expected:

    INVALID_TYPE

### 4.3 Integer overflow

Provide a value above the target integer maximum.

Expected:

    overflow classification

not successful conversion.

### 4.4 Valid decimal

Input:

    '123.4500'

Expected:

    NUMERIC value

with the documented scale policy.

### 4.5 Excess precision

Input:

    '123.4567'

against a two-decimal contract.

Expected:

    reject

if rounding is not allowed.

### 4.6 Rounding policy

Test values exactly around rounding boundaries.

Verify the configured rounding mode.

### 4.7 Boolean domain

Test:

    true
    false
    TRUE
    FALSE
    1
    yes

and verify only explicitly accepted representations pass.

### 4.8 Date format

Test the accepted date format and reject ambiguous alternatives when the contract is strict.

### 4.9 Invalid date

Input:

    2026-02-30

Expected:

    INVALID_TYPE

or an appropriate date-parsing failure code.

### 4.10 Timestamp offset

Test:

    2026-09-28T10:00:00+00:00

and an offset-free timestamp.

Verify the contract's timezone policy.

### 4.11 JSON object versus array

Provide valid JSON with the wrong top-level type.

Expected:

    schema/type failure

not JSON syntax failure.

### 4.12 NULL

Input:

    NULL

Expected:

    separate requiredness outcome

when NULL is allowed or disallowed.

### 4.13 Lossy conversion

Convert decimal to integer.

Verify that fractional information is either rejected or transformed according to explicit policy.

### 4.14 String truncation

Provide a string longer than the target field.

Expected:

    reject

unless truncation is explicitly permitted.

### 4.15 Type drift

Run the same field first as a valid string representation and then as an object.

Expected:

    schema/type drift detected.

### 4.16 Idempotence

Run the same raw input through validation twice.

Expected:

    same typed values
    same failure codes

### 4.17 Reconciliation

Verify:

    valid_type_rows + invalid_type_rows = input_rows

unless explicit terminal states exist.

### 4.18 Target constraint test

Attempt to insert an invalid typed value into the target table.

Expected:

    target constraint rejects it.

## 5. Observability

Type validation needs visibility into both parser failures and conversion behavior.

### Core metrics

| Metric | Meaning |
|---|---|
| `type_validation_input_rows` | Records entering validation |
| `type_validation_valid_rows` | Records with valid typed fields |
| `type_validation_invalid_rows` | Records with type failures |
| `type_validation_failure_count` | Total field-level type failures |
| `type_validation_parse_failures` | Values that could not be parsed |
| `type_validation_overflow_failures` | Values exceeding target range |
| `type_validation_precision_failures` | Values violating precision/scale policy |
| `type_validation_schema_drift` | Type changes from expected schema |
| `type_validation_rule_version` | Active type contract |

### Failure distribution

Break failures down by:

- Field.
- Source.
- Failure code.
- Schema version.
- Batch.

### Conversion monitoring

Track how many values were:

    accepted without conversion
    safely coerced
    rejected

A sudden increase in coercion can indicate source drift.

### Precision monitoring

For monetary fields, monitor:

    scale violations
    rounding events
    overflow events

### Schema drift monitoring

Detect:

    expected type != observed type

before downstream models fail.

### Quarantine monitoring

Track:

- Invalid rows.
- Failure reasons.
- Oldest unresolved record.
- Reprocessing attempts.

## 6. Intentional Failure

### Failure 1 — Implicit numeric cast

Allow malformed numeric strings to reach a cast.

Expected symptom:

- Batch failure or uncontrolled coercion.

Recovery:

Validate syntax before conversion and quarantine invalid rows.

### Failure 2 — Silent rounding

Convert high-precision decimals into lower scale.

Expected symptom:

- Monetary information changes without an explicit decision.

Recovery:

Reject or apply a documented rounding rule.

### Failure 3 — Integer overflow

Provide a value beyond the target type range.

Expected symptom:

- Conversion failure or corrupted downstream behavior.

Recovery:

Validate range before publication.

### Failure 4 — Boolean over-coercion

Accept arbitrary values such as:

    'yes'
    'Y'
    '1'

without a contract.

Expected symptom:

- Different sources receive inconsistent semantics.

Recovery:

Define an explicit Boolean domain.

### Failure 5 — Assume offset-free timestamps are UTC

Remove timezone information from source data.

Expected symptom:

- Events shift across reporting boundaries.

Recovery:

Require explicit timezone semantics.

### Failure 6 — Truncate strings silently

Send oversized values into a bounded target field.

Expected symptom:

- Data is silently shortened.

Recovery:

Reject or explicitly govern truncation.

### Failure 7 — Treat NULL as type failure

Classify every NULL as malformed type.

Expected symptom:

- Requiredness and type metrics become mixed.

Recovery:

Separate nullability from type correctness.

### Failure 8 — Cast before validation

Convert the entire batch directly into target types.

Expected symptom:

- One malformed record can abort the batch.

Recovery:

Use staged parsing and row-level failure classification.

## 7. Recovery

### Recovery sequence

1. Preserve the raw source batch.
2. Identify failing field and type.
3. Identify parser or coercion behavior.
4. Check schema version.
5. Determine whether the source changed.
6. Correct mapping or source data.
7. Revalidate invalid records.
8. Rebuild typed staging.
9. Reconcile valid and invalid populations.
10. Publish only validated typed data.
11. Record the root cause and regression test.

### Recovering from a source type change

1. Compare observed and expected representations.
2. Confirm whether the source change is intentional.
3. Version the schema contract.
4. Update parsing logic if approved.
5. Reprocess preserved raw data.

### Recovering from precision loss

1. Identify affected field and batch.
2. Recover original raw values.
3. Reapply the correct decimal contract.
4. Recalculate affected downstream measures.
5. Reconcile financial totals.

### Recovering from timestamp ambiguity

1. Identify the source timezone assumption.
2. Determine the affected reporting period.
3. Reinterpret from raw values using the correct timezone.
4. Rebuild affected partitions.
5. Add timezone regression tests.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL provides strong native types, casts, constraints, numeric precision, timestamp types, JSONB, and query-level validation.

Learn how implicit casts and explicit casts behave before relying on them in production.

### 2. dbt

dbt is useful for documenting expected column types, testing models, and making schema contracts version-controlled.

### 3. Great Expectations

Great Expectations can express type expectations and produce validation results across data batches.

Use it to operationalize a contract you already understand rather than treating expectations as a black box.

## 9. Production Runbook

### Before deployment

- [ ] Define logical types.
- [ ] Define physical source representations.
- [ ] Define accepted formats.
- [ ] Define coercion rules.
- [ ] Define precision and scale.
- [ ] Define timezone policy.
- [ ] Define overflow behavior.
- [ ] Define truncation behavior.
- [ ] Define NULL policy.
- [ ] Define rule version.
- [ ] Define quarantine behavior.

### During execution

- [ ] Record input rows.
- [ ] Record valid rows.
- [ ] Record invalid rows.
- [ ] Record parse failures.
- [ ] Record overflow failures.
- [ ] Record precision failures.
- [ ] Record schema drift.
- [ ] Record rule version.

### If type failures spike

1. Compare source schema.
2. Check recent producer releases.
3. Compare failure distribution by field.
4. Check schema version.
5. Check parser changes.

### If financial values change

1. Inspect decimal precision.
2. Inspect scale.
3. Inspect rounding mode.
4. Compare raw and typed values.
5. Reconcile totals.

### If timestamps shift

1. Check source offset.
2. Check timezone conversion.
3. Check daylight-saving behavior.
4. Compare raw timestamps.
5. Reprocess affected periods.

## 10. Common Mistakes

### Mistake 1 — Treating parsing as validation

A parser succeeding does not prove the value satisfies the business type contract.

### Mistake 2 — Relying on implicit casts

Implicit conversion can hide source drift.

### Mistake 3 — Ignoring precision and scale

Numeric representation can lose information.

### Mistake 4 — Using floating point for money

Binary floating-point is not a substitute for exact monetary decimals.

### Mistake 5 — Silent rounding

Rounding is a business decision when precision matters.

### Mistake 6 — Silent truncation

Shortening data can create irreversible loss.

### Mistake 7 — Assuming timezone

Offset-free timestamps are not automatically UTC.

### Mistake 8 — Mixing NULL and type errors

Nullability and type correctness are separate contracts.

### Mistake 9 — Validating after conversion

Invalid source values may already have been corrupted by permissive coercion.

### Mistake 10 — No schema-drift monitoring

Type changes can occur while field names remain unchanged.

### Mistake 11 — One bad row aborts the whole batch

Use staged validation when row-level quarantine is required.

### Mistake 12 — No raw preservation

Without original representations, correcting conversion logic may be impossible.

## 11. Definition of Done

The type-validation stage is complete when you can:

- [ ] Define logical and physical types.
- [ ] Distinguish parsing from validation.
- [ ] Implement strict numeric validation.
- [ ] Handle integer range and overflow.
- [ ] Enforce decimal precision and scale.
- [ ] Define rounding policy.
- [ ] Validate Boolean domains.
- [ ] Validate dates and timestamps.
- [ ] Handle timezone-aware timestamps.
- [ ] Validate UUIDs.
- [ ] Validate JSON top-level types and nested types.
- [ ] Distinguish NULL from malformed values.
- [ ] Detect lossy conversions.
- [ ] Detect truncation risk.
- [ ] Detect schema type drift.
- [ ] Produce field-level failure diagnostics.
- [ ] Quarantine invalid records.
- [ ] Preserve raw representations.
- [ ] Enforce target types with database constraints.
- [ ] Monitor conversion and schema-drift metrics.
- [ ] Intentionally break type validation.
- [ ] Recover from precision, overflow, and timezone failures.
- [ ] Explain how PostgreSQL, dbt, and Great Expectations support production type validation.

## 12. What You Learned

Type validation protects the boundary between raw representation and trusted typed data.

The production workflow is:

    DEFINE TYPE CONTRACT
         ↓
    VALIDATE SOURCE REPRESENTATION
         ↓
    PARSE
         ↓
    VALIDATE RANGE / PRECISION
         ↓
    NORMALIZE TO TARGET TYPE
         ↓
    ROUTE INVALID VALUES
         ↓
    ENFORCE TARGET CONSTRAINTS
         ↓
    MONITOR SCHEMA DRIFT

> **A successful cast does not automatically mean valid data; production type validation must control accepted representation, precision, range, timezone, lossiness, and failure handling.**

### Next recipe

**T35 — Range Validation**