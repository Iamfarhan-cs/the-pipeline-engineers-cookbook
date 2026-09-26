# T02 — Data Type Conversion

## 1. Problem Recognition

Data arrives in source-specific representations.

Examples:

    "100.50"
    "2026-09-26"
    "true"
    "00123"
    100.50
    null

Downstream systems need explicit types:

    NUMERIC
    TIMESTAMPTZ
    BOOLEAN
    INTEGER
    DATE

Data type conversion is the controlled process of turning source representations into the types required by the next pipeline layer.

The core rule is:

    never let an implicit cast decide business meaning

A production pipeline should know:

- what the source value means;
- what target type is required;
- which values are valid;
- what happens when conversion fails;
- whether precision can be lost;
- whether the conversion is reversible;
- how the conversion is observed and tested.

This recipe builds on T01 — Raw Data to Staging.

---

## 2. What You Are Building

Target architecture:

    source representation
            |
            v
       type contract
            |
            v
      normalization
            |
            v
      deterministic cast
            |
       +----+----+
       |         |
       v         v
     valid     invalid
       |         |
       v         v
   typed value  quarantine
       |
       v
 downstream transformation

Examples:

    "42"       -> INTEGER 42
    "42.50"    -> NUMERIC 42.50
    "2026-09-26" -> DATE
    "true"     -> BOOLEAN
    "2026-09-26T10:15:00Z" -> TIMESTAMPTZ

---

## 3. Learning Objectives

By the end of this recipe you should be able to:

1. Define a type-conversion contract.
2. Convert strings to numeric types safely.
3. Handle decimal precision correctly.
4. Convert timestamps without losing timezone meaning.
5. Normalize booleans explicitly.
6. Distinguish NULL, empty, and invalid values.
7. Handle integer overflow.
8. Handle malformed dates.
9. Detect lossy conversions.
10. Separate normalization from type conversion.
11. Quarantine conversion failures.
12. Test boundary values.
13. Make conversions idempotent.
14. Observe conversion failures in production.
15. Design conversion logic that survives schema evolution.

---

## 4. Source Type Is Not Target Type

A CSV column may technically contain:

    "100.25"

That does not mean the source business type is:

    NUMERIC

It is a textual representation of a numeric value.

Likewise:

    "2026-09-26T10:00:00Z"

is text representing a timestamp.

The pipeline must explicitly define the target type.

---

## 5. Type Contract

A type contract can describe:

    source_field
    source_representation
    target_type
    normalization
    accepted_formats
    invalid_policy
    precision
    scale
    timezone
    null_policy

Example:

```
{
  "field": "amount",
  "source_type": "text",
  "target_type": "NUMERIC(18,2)",
  "trim": true,
  "allow_empty": false,
  "rounding": "REJECT",
  "null_policy": "ALLOW"
}
```

Keep conversion rules explicit and versioned.

---

## 6. Conversion Pipeline

A safe sequence is:

    raw value
       |
       v
    presence check
       |
       v
    normalization
       |
       v
    syntax validation
       |
       v
    type conversion
       |
       v
    semantic validation
       |
       v
    typed value

Do not combine all of these into one opaque function.

---

## 7. Presence vs Value

These are different:

    field absent
    field = null
    field = ""
    field = "   "
    field = "0"

A conversion function should know which condition it received.

Example:

```
def classify_presence(record, field):
    if field not in record:
        return "MISSING"

    if record[field] is None:
        return "NULL"

    if isinstance(record[field], str) and not record[field].strip():
        return "EMPTY"

    return "PRESENT"
```

Do not convert empty strings to NULL unless the contract says to.

---

## 8. Integer Conversion

Basic conversion:

```
def to_int(value):
    return int(value)
```

This is acceptable only when the input contract is already strict.

Production conversion should define:

- whitespace handling;
- sign handling;
- decimal rejection;
- range limits;
- empty-value behavior.

---

## 9. Safe Integer Conversion

Example:

```
def parse_int(value):
    if value is None:
        return None

    if isinstance(value, str):
        value = value.strip()

    if value == "":
        raise ValueError("EMPTY_INTEGER")

    return int(value)
```

This handles simple text representations.

It does not automatically make all inputs valid.

---

## 10. Do Not Silently Convert Decimal to Integer

Example:

    "10.9"

must not silently become:

    10

That loses information.

A safer policy is:

    "10.9" -> reject

unless the contract explicitly says:

    truncate
    round
    floor
    ceiling

Rounding is a business rule, not merely a technical cast.

---

## 11. Integer Bounds

Target systems may have bounded integer types.

Examples:

    SMALLINT
    INTEGER
    BIGINT

Python integers can be arbitrarily large.

That does not mean PostgreSQL INTEGER can store every Python integer.

For a signed 32-bit integer:

    -2,147,483,648
    to
     2,147,483,647

Validate the target range explicitly when needed.

---

## 12. Integer Range Validation

Example:

```
INT32_MIN = -(2 ** 31)
INT32_MAX = (2 ** 31) - 1

def parse_int32(value):
    parsed = int(str(value).strip())

    if not INT32_MIN <= parsed <= INT32_MAX:
        raise ValueError("INTEGER_OUT_OF_RANGE")

    return parsed
```

This makes the target constraint explicit before database insertion.

---

## 13. Decimal Values

Financial and measurement data often requires exact decimal semantics.

Do not use:

    float

as the default representation for monetary values.

Example:

    0.1 + 0.2

with binary floating-point does not represent decimal arithmetic exactly.

Use:

    decimal.Decimal

for exact decimal values.

---

## 14. Decimal Conversion

Example:

```
from decimal import Decimal

def parse_decimal(value):
    if value is None:
        return None

    text = str(value).strip()

    if not text:
        raise ValueError("EMPTY_DECIMAL")

    return Decimal(text)
```

This preserves decimal representation better than converting through float.

Avoid:

```
Decimal(float(value))
```

because the float may already contain binary approximation.

---

## 15. Precision and Scale

A database type such as:

    NUMERIC(18,2)

means:

    precision = 18 total digits
    scale = 2 digits after decimal

Examples:

    1234567890123456.78 -> valid

but:

    12345678901234567.89

exceeds the precision.

The conversion contract should define target precision and scale.

---

## 16. Decimal Scale Validation

Example:

```
from decimal import Decimal

def decimal_scale(value):
    exponent = value.as_tuple().exponent
    return max(0, -exponent)

def validate_scale(value, scale):
    if decimal_scale(value) > scale:
        raise ValueError("DECIMAL_SCALE_EXCEEDED")
```

Do not silently round unless the business contract requires it.

---

## 17. Explicit Rounding

If rounding is required:

```
from decimal import Decimal, ROUND_HALF_UP

def round_currency(value):
    amount = Decimal(value)
    return amount.quantize(
        Decimal("0.01"),
        rounding=ROUND_HALF_UP,
    )
```

The rounding mode must be part of the contract.

Different modes produce different business results.

---

## 18. Currency Conversion Is Different

Do not confuse:

    decimal conversion

with:

    currency conversion

This recipe converts:

    "100.50" -> Decimal("100.50")

It does not decide:

    EUR 100 -> USD 117

FX conversion requires rate, date, source, and business rules.

---

## 19. Numeric Formatting

Inputs may contain:

    "1,234.50"
    "1234.50"
    " 1234.50 "

Whether commas are accepted must be explicit.

Example normalization:

```
def normalize_decimal_text(value):
    text = str(value).strip()

    if "," in text:
        raise ValueError("THOUSANDS_SEPARATOR_NOT_ALLOWED")

    return text
```

Do not remove commas automatically unless the source contract defines them as formatting separators.

---

## 20. Boolean Conversion

Do not use:

```
bool("false")
```

It returns:

    True

because the string is non-empty.

Boolean conversion requires explicit accepted representations.

---

## 21. Explicit Boolean Mapping

Example:

```
TRUE_VALUES = {"true", "1", "yes", "y", "t"}
FALSE_VALUES = {"false", "0", "no", "n", "f"}

def parse_bool(value):
    if value is None:
        return None

    text = str(value).strip().lower()

    if text in TRUE_VALUES:
        return True

    if text in FALSE_VALUES:
        return False

    raise ValueError("INVALID_BOOLEAN")
```

The accepted values should be source-specific.

---

## 22. Do Not Invent Boolean Semantics

Some sources use:

    ACTIVE
    INACTIVE

These are not necessarily booleans.

They may represent:

    status

Use a status mapping when that is the actual business meaning.

Type conversion should not erase semantic distinctions.

---

## 23. Date Conversion

A date contains:

    year
    month
    day

Example:

    2026-09-26

Use:

```
from datetime import date

value = date.fromisoformat("2026-09-26")
```

Do not parse dates using locale-dependent assumptions unless the source contract requires them.

---

## 24. Ambiguous Date Formats

These are ambiguous:

    09/10/2026

Could mean:

    September 10

or:

    October 9

Never guess.

The source contract must specify:

    DD/MM/YYYY

or:

    MM/DD/YYYY

---

## 25. Strict Date Parsing

Example:

```
from datetime import datetime

def parse_date_ddmmyyyy(value):
    text = str(value).strip()
    return datetime.strptime(text, "%d/%m/%Y").date()
```

A parsing failure should be explicit.

---

## 26. Date Semantics

Do not convert:

    2026-09-26

to a timestamp merely because the database column accepts it.

A DATE means:

    calendar date

A TIMESTAMPTZ means:

    point in time

These are different semantic types.

---

## 27. Timestamp Conversion

Example:

```
from datetime import datetime

value = datetime.fromisoformat(
    "2026-09-26T10:15:00+00:00"
)
```

For UTC:

    +00:00

or:

    Z

must be interpreted as UTC.

---

## 28. Naive vs Aware Datetimes

Naive:

```
datetime(2026, 9, 26, 10, 15)
```

Aware:

```
datetime(
    2026,
    9,
    26,
    10,
    15,
    tzinfo=timezone.utc,
)
```

Do not attach a timezone to a naive timestamp merely because the target column requires one.

You must know what timezone the source value represents.

---

## 29. Timestamp With Timezone

For an input:

    2026-09-26T10:15:00+02:00

the corresponding UTC instant is:

    2026-09-26T08:15:00Z

Timezone conversion changes representation while preserving the point in time.

---

## 30. Local Time Requires a Zone

Input:

    2026-09-26 10:15

is not a complete point-in-time value.

You need:

    timezone = Europe/Berlin

or another source-defined zone.

Never assume:

    local time = UTC

without a contract.

---

## 31. DST Ambiguity

During daylight-saving transitions, a local timestamp can be:

- nonexistent;
- ambiguous.

For example, a clock can skip from:

    02:00 -> 03:00

or repeat an hour.

A production conversion layer must define how these cases are handled.

Prefer source timestamps that include an offset or timezone.

---

## 32. Timezone Normalization

A useful internal policy is:

    parse source timestamp
          |
          v
    resolve timezone
          |
          v
    normalize to UTC
          |
          v
    store TIMESTAMPTZ

Keep the original source representation in raw evidence when auditability matters.

---

## 33. String Conversion

String normalization should not become arbitrary data cleaning.

Examples:

    " C1 " -> "C1"

may be correct.

But:

    "ACME LTD." -> "ACME"

is a business transformation.

Keep normalization rules explicit.

---

## 34. Unicode

Do not assume:

    one visible character = one byte

or:

    one visible character = one database character

UTF-8 uses variable-length encoding.

For text conversion:

- preserve Unicode;
- normalize only when required;
- avoid destructive ASCII conversion.

---

## 35. Decimal Separator Differences

Some sources use:

    1234.50

others:

    1234,50

Do not globally replace:

    "," -> "."

because a source may also contain:

    1,234.50

Define the source locale and grammar.

---

## 36. Scientific Notation

A numeric source may contain:

    1.25E+3

Decide whether scientific notation is accepted.

Python Decimal can parse many valid scientific forms:

```
from decimal import Decimal

value = Decimal("1.25E+3")
print(value)
```

The target database and business contract still determine whether it is acceptable.

---

## 37. Special Numeric Values

Some systems may produce:

    NaN
    Infinity
    -Infinity

These are not ordinary business numbers.

Define whether they are:

    accepted
    converted to NULL
    rejected

For financial data, explicit rejection is usually safer unless the source contract says otherwise.

---

## 38. NULL Policy

For every field define:

    missing -> ?
    null -> ?
    empty -> ?
    invalid -> ?

Example:

```
{
  "field": "amount",
  "missing": "REJECT",
  "null": "ALLOW",
  "empty": "REJECT",
  "invalid": "QUARANTINE"
}
```

Do not let each parser choose independently.

---

## 39. Default Values

Do not use default values to hide conversion failures.

Bad:

    invalid amount -> 0

This converts an unknown value into a known business value.

A default should be used only when:

    the business contract defines it

Example:

    missing retry_count -> 0

may be valid.

---

## 40. Conversion Failure Categories

Useful reason codes:

    INVALID_INTEGER
    INTEGER_OUT_OF_RANGE
    INVALID_DECIMAL
    DECIMAL_SCALE_EXCEEDED
    INVALID_BOOLEAN
    INVALID_DATE
    INVALID_TIMESTAMP
    TIMEZONE_REQUIRED
    AMBIGUOUS_DATE
    EMPTY_VALUE
    UNSUPPORTED_FORMAT

Stable reason codes make monitoring and remediation easier.

---

## 41. Generic Conversion Result

A useful Python pattern:

```
from dataclasses import dataclass
from typing import Any

@dataclass
class ConversionResult:
    value: Any
    valid: bool
    reason_code: str | None = None
```

This lets the pipeline carry conversion results without immediately crashing the entire record.

---

## 42. Safe Integer Converter

```
def convert_integer(value):
    if value is None:
        return ConversionResult(None, True)

    try:
        text = str(value).strip()

        if not text:
            return ConversionResult(
                None,
                False,
                "EMPTY_VALUE",
            )

        return ConversionResult(int(text), True)

    except (TypeError, ValueError):
        return ConversionResult(
            None,
            False,
            "INVALID_INTEGER",
        )
```

The production version should also enforce the target range.

---

## 43. Safe Decimal Converter

```
from decimal import Decimal, InvalidOperation

def convert_decimal(value):
    if value is None:
        return ConversionResult(None, True)

    try:
        text = str(value).strip()

        if not text:
            return ConversionResult(
                None,
                False,
                "EMPTY_VALUE",
            )

        return ConversionResult(
            Decimal(text),
            True,
        )

    except InvalidOperation:
        return ConversionResult(
            None,
            False,
            "INVALID_DECIMAL",
        )
```

Avoid converting through float.

---

## 44. Safe Boolean Converter

```
def convert_boolean(value):
    if value is None:
        return ConversionResult(None, True)

    text = str(value).strip().lower()

    if text in {"true", "1", "yes"}:
        return ConversionResult(True, True)

    if text in {"false", "0", "no"}:
        return ConversionResult(False, True)

    return ConversionResult(
        None,
        False,
        "INVALID_BOOLEAN",
    )
```

Keep accepted representations explicit.

---

## 45. Safe Date Converter

```
from datetime import date

def convert_date(value):
    if value is None:
        return ConversionResult(None, True)

    try:
        parsed = date.fromisoformat(str(value).strip())
        return ConversionResult(parsed, True)

    except ValueError:
        return ConversionResult(
            None,
            False,
            "INVALID_DATE",
        )
```

Use a format-specific parser when the source is not ISO.

---

## 46. Safe Timestamp Converter

```
from datetime import datetime

def convert_timestamp(value):
    if value is None:
        return ConversionResult(None, True)

    try:
        parsed = datetime.fromisoformat(
            str(value).strip().replace("Z", "+00:00")
        )

        if parsed.tzinfo is None:
            return ConversionResult(
                None,
                False,
                "TIMEZONE_REQUIRED",
            )

        return ConversionResult(parsed, True)

    except ValueError:
        return ConversionResult(
            None,
            False,
            "INVALID_TIMESTAMP",
        )
```

Do not silently assume a timezone for a naive timestamp.

---

## 47. Column-Level Conversion Map

A pipeline can define:

```
CONVERTERS = {
    "customer_id": str,
    "amount": convert_decimal,
    "is_active": convert_boolean,
    "transaction_date": convert_date,
    "created_at": convert_timestamp,
}
```

Apply converters explicitly.

This prevents accidental implicit casting.

---

## 48. Record Conversion

Example:

```
def convert_record(record):
    output = {}

    for field, converter in CONVERTERS.items():
        result = converter(record.get(field))

        if not result.valid:
            raise ValueError(
                f"{field}: {result.reason_code}"
            )

        output[field] = result.value

    return output
```

A production pipeline can instead return structured errors for quarantine.

---

## 49. Partial Conversion

Do not partially commit a record unless the staging contract allows it.

Example:

    amount -> valid
    date -> invalid
    currency -> valid

The record should normally become:

    INVALID

rather than:

    partially valid

unless the pipeline explicitly supports field-level error handling.

---

## 50. Strict vs Tolerant Mode

### Strict

One conversion error can fail the record, batch, or file.

Useful when:

- the source contract is strict;
- partial ingestion would create misleading data.

### Tolerant

Invalid records are quarantined while valid records continue.

Useful when:

- record independence is acceptable;
- large files should not fail because of isolated bad rows.

Choose explicitly.

---

## 51. Database Conversion

Avoid relying on:

```
INSERT INTO target(amount)
VALUES (%s)
```

to discover whether the source value is valid.

Validate and convert before the database boundary when the pipeline owns the conversion contract.

The database remains the final type constraint.

Use both:

    application validation
        +
    database type enforcement

---

## 52. PostgreSQL Numeric

Example:

```
CREATE TABLE staging_payment (
    amount NUMERIC(18,2)
);
```

The application should ensure:

- numeric syntax;
- expected scale;
- expected range.

PostgreSQL should remain the final enforcement layer.

---

## 53. PostgreSQL Timestamp

Prefer:

```
created_at TIMESTAMPTZ NOT NULL
```

when the field represents an actual point in time.

Use:

```
business_date DATE NOT NULL
```

when it represents only a calendar date.

Do not substitute one for the other.

---

## 54. Casting in SQL

Explicit casts are preferable to ambiguous implicit behavior.

Example:

```
SELECT amount_text::NUMERIC(18,2)
FROM staging_payment;
```

But if malformed data can reach this statement, one bad value may fail the query.

Validate or isolate invalid values before bulk conversion.

---

## 55. Safe SQL Conversion

A guarded pattern:

```
SELECT
    CASE
        WHEN amount_text ~ '^[0-9]+(\.[0-9]+)?$'
        THEN amount_text::NUMERIC
        ELSE NULL
    END AS amount
FROM staging_payment;
```

This is only an example grammar.

Production validation should match the actual source contract.

---

## 56. Conversion at Staging vs Curated

A useful rule:

Convert technical representations early when the type is unambiguous.

Examples:

    "42" -> INTEGER
    ISO timestamp -> TIMESTAMPTZ

Delay business interpretation when it requires domain rules.

Examples:

    "ACTIVE" -> customer status meaning

    "100" -> EUR amount

The first can be technical.

The second may require business context.

---

## 57. Idempotency

Type conversion should be deterministic.

Given the same source value and same contract:

    input
      |
      v
    converter
      |
      v
    same typed result

Do not make conversion depend on:

    current time
    random state
    external mutable configuration

unless that dependency is explicitly part of the contract.

---

## 58. Conversion Versioning

A rule may change.

Example:

    v1:
        round to 2 decimals

    v2:
        reject values with more than 2 decimals

Historical records should retain enough metadata to know which conversion contract was applied.

Possible field:

    conversion_version

---

## 59. Lossy Conversion

A conversion is lossy when information disappears.

Examples:

    10.999 -> 11.00
    timestamp with timezone -> local timestamp
    "00123" -> 123
    "true" -> 1

Lossy conversion can be valid, but only when explicitly required.

Document it.

---

## 60. Reversibility

Ask:

> Can I reconstruct the source representation from the typed value?

Examples:

    "00123" -> 123

cannot recover:

    "00123"

If the original formatting matters, preserve it in raw evidence or a source-text column.

Do not assume type conversion is reversible.

---

## 61. Identifier Conversion

Be careful with identifiers that look numeric.

Example:

    "00001234"

may be an account/customer identifier.

Converting to INTEGER produces:

    1234

which destroys leading zeros.

Do not convert identifiers merely because they contain digits.

---

## 62. Postal Codes and Similar Fields

These should usually remain text:

    postal_code
    phone_number
    account_reference
    customer_reference

They can contain:

- leading zeros;
- letters;
- separators;
- country-specific formats.

Semantic type matters more than visual appearance.

---

## 63. Money and Floating Point

For monetary amounts:

    source text
       |
       v
    Decimal
       |
       v
    NUMERIC

Avoid:

    source text
       |
       v
    float
       |
       v
    NUMERIC

The intermediate float can introduce precision artifacts.

---

## 64. Percentage Values

A source may send:

    "15"

meaning:

    15%

or:

    0.15

meaning:

    15%

Do not decide based on magnitude alone.

The source contract must define the representation.

---

## 65. Units

Numeric conversion does not establish units.

Example:

    "100"

could mean:

    100 EUR
    100 USD
    100 cents
    100 kilograms

Type conversion answers:

    what machine type?

The data contract answers:

    what does the value mean?

---

## 66. Enum Conversion

A string can map to a controlled enum.

Example:

```
STATUS_MAP = {
    "A": "ACTIVE",
    "I": "INACTIVE",
}
```

This is more than a primitive type cast.

Keep mapping logic separate from generic type conversion.

---

## 67. Conversion Metrics

Track:

    conversion_attempts_total
    conversion_success_total
    conversion_failures_total
    conversion_quarantined_total

Useful dimensions:

    source_system
    file_type
    field_name
    target_type
    reason_code

Be careful with field_name cardinality if schemas are highly dynamic.

---

## 68. Conversion Error Logs

Example:

```
logger.warning(
    "type_conversion_failed",
    extra={
        "source_file_id": source_file_id,
        "source_record_id": source_record_id,
        "field": field_name,
        "target_type": target_type,
        "reason_code": reason_code,
    },
)
```

Do not log sensitive raw values by default.

If debugging requires a value, apply the project's data-redaction policy.

---

## 69. Reconciliation

After conversion:

    extracted_records
        =
    converted_valid_records
      +
    conversion_failed_records

If conversion occurs as part of broader validation:

    extracted
        =
    staged_valid
      +
    quarantined
      +
    explicitly_skipped

The accounting equation must be defined for the pipeline.

---

## 70. Intentional Failure Drill — Invalid Integer

Input:

    customer_count = "ABC"

Expected:

    conversion failure
    reason = INVALID_INTEGER
    record handled according to policy

No silent zero should appear.

---

## 71. Intentional Failure Drill — Decimal Precision

Input:

    amount = "10.999"

Contract:

    NUMERIC(18,2)
    rounding = REJECT

Expected:

    DECIMAL_SCALE_EXCEEDED

No silent 11.00.

---

## 72. Intentional Failure Drill — Boolean

Input:

    is_active = "false"

Expected:

    False

Verify the implementation does not use:

    bool("false")

---

## 73. Intentional Failure Drill — Timezone

Input:

    2026-09-26T10:00:00+02:00

Expected UTC:

    2026-09-26T08:00:00Z

Verify the point in time is preserved.

---

## 74. Intentional Failure Drill — Naive Timestamp

Input:

    2026-09-26 10:00:00

Contract requires timezone-aware timestamps.

Expected:

    TIMEZONE_REQUIRED

Do not silently assume UTC.

---

## 75. Intentional Failure Drill — Identifier Leading Zeros

Input:

    customer_id = "000123"

Expected:

    "000123"

Do not convert it to integer 123.

---

## 76. Intentional Failure Drill — Overflow

Input:

    integer field receives value outside target range

Expected:

    INTEGER_OUT_OF_RANGE

Do not rely on database failure alone if the pipeline can detect it earlier.

---

## 77. Unit Test — Integer

Test:

    "42" -> 42
    "-42" -> -42
    " 42 " -> 42
    "" -> invalid
    "42.0" -> invalid if integer syntax is strict

---

## 78. Unit Test — Decimal

Test:

    "10.50" -> Decimal("10.50")
    "0.10" -> exact decimal
    "ABC" -> invalid
    "" -> invalid
    None -> according to null policy

---

## 79. Unit Test — Boolean

Test:

    "true" -> True
    "false" -> False
    "1" -> True
    "0" -> False
    "unknown" -> invalid

Also test:

    bool("false")

is never used as the parser.

---

## 80. Unit Test — Date

Test:

    "2026-09-26" -> valid
    "2026-02-29" -> invalid
    "09/26/2026" -> invalid under ISO contract

The exact accepted format belongs to the source contract.

---

## 81. Unit Test — Timestamp

Test:

    2026-09-26T10:00:00Z
    2026-09-26T10:00:00+02:00
    naive timestamp
    malformed timestamp

Verify timezone semantics.

---

## 82. Unit Test — Null, Missing, Empty

Test separately:

    field absent
    field = None
    field = ""
    field = "   "

Do not collapse them accidentally.

---

## 83. Unit Test — Decimal Boundaries

For NUMERIC(18,2), test:

    maximum valid precision
    one digit beyond precision
    exact scale boundary
    one extra decimal digit

Verify the contract's rounding/rejection behavior.

---

## 84. Integration Test — Mixed Record

Input:

    valid integer
    valid decimal
    valid boolean
    valid timestamp

Expected:

    all target values have correct database types.

---

## 85. Integration Test — One Invalid Field

Input:

    amount = "ABC"

Expected according to policy:

    record quarantined

or:

    file failed

but never:

    amount = 0

unless zero is an explicit default.

---

## 86. Integration Test — Replay

Run the same source through conversion twice.

Expected:

    same typed output
    same deterministic result
    no duplicate staging records

---

## 87. Integration Test — Contract Version

Run the same value under two conversion contracts.

Verify:

    conversion_version

or equivalent lineage metadata distinguishes the behavior.

---

## 88. Integration Test — Timezone

Load timestamps from multiple offsets.

Verify that equivalent instants normalize to the same UTC point.

Example:

    10:00 +02:00
    08:00 Z

represent the same instant.

---

## 89. Recovery — Invalid Data

When conversion fails:

1. preserve source evidence;
2. record field and reason code;
3. quarantine or fail according to policy;
4. notify only if operationally required;
5. correct the source or mapping;
6. replay from raw/staging;
7. verify reconciliation.

Do not manually edit raw data to make the conversion succeed.

---

## 90. Recovery — Contract Change

If the source changes format:

1. identify the affected field;
2. compare old and new contracts;
3. version the converter;
4. test representative samples;
5. deploy the new mapping;
6. preserve historical conversion metadata;
7. replay affected data if required.

Do not change conversion semantics silently.

---

## 91. Recovery — Bad Rounding Rule

If production discovers that a rounding policy was incorrect:

1. stop or isolate affected processing;
2. identify impacted records;
3. preserve original source values;
4. correct the conversion contract;
5. replay affected records;
6. reconcile downstream tables;
7. document the correction.

Raw evidence makes this recovery possible.

---

## 92. Recovery — Timezone Error

If timestamps were interpreted using the wrong timezone:

1. identify affected source and time range;
2. determine the intended timezone;
3. preserve the original source representation;
4. correct the parser;
5. replay affected records;
6. reconcile downstream time-based facts;
7. document the incident.

Do not simply update the final timestamp column without understanding the original meaning.

---

## 93. Production Runbook

When type conversion failures increase:

### Step 1 — Identify the field

Check:

    source_system
    file_type
    field_name
    target_type

### Step 2 — Check the reason code

Examples:

    INVALID_DECIMAL
    INVALID_DATE
    TIMEZONE_REQUIRED
    INTEGER_OUT_OF_RANGE

### Step 3 — Compare with source contract

Check:

    format
    allowed values
    precision
    timezone
    null policy

### Step 4 — Inspect source evidence

Use the raw artifact and source record lineage.

### Step 5 — Determine scope

Is the issue:

    one record
    one file
    one source
    all recent deliveries

### Step 6 — Choose recovery

Use:

    replay
    quarantine
    mapping correction
    contract version

### Step 7 — Reconcile

Verify:

    conversion successes
    conversion failures
    staged records
    downstream records

### Step 8 — Close

Preserve the corrected contract and incident history.

---

## 94. Common Mistakes

### Mistake 1 — Relying on implicit database casts

**Fix:** Convert explicitly before the database boundary.

### Mistake 2 — Converting monetary values through float

**Fix:** Use Decimal and database NUMERIC.

### Mistake 3 — Using bool on strings

**Fix:** Define explicit true/false representations.

### Mistake 4 — Assuming local timestamps are UTC

**Fix:** Require an explicit source timezone.

### Mistake 5 — Converting identifiers to numbers

**Fix:** Keep semantic identifiers as text.

### Mistake 6 — Silently rounding

**Fix:** Make rounding a versioned business rule.

### Mistake 7 — Turning invalid values into zero

**Fix:** Reject or quarantine invalid data.

### Mistake 8 — Treating DATE as TIMESTAMPTZ

**Fix:** Preserve semantic type.

### Mistake 9 — Removing separators blindly

**Fix:** Define the source numeric grammar.

### Mistake 10 — Collapsing missing, NULL, and empty

**Fix:** Preserve distinctions required by the contract.

### Mistake 11 — Losing original representation

**Fix:** Keep raw evidence and source lineage.

### Mistake 12 — Changing conversion rules without versioning

**Fix:** Version conversion contracts.

### Mistake 13 — Ignoring overflow

**Fix:** Validate target ranges explicitly.

### Mistake 14 — Converting units during type casting

**Fix:** Keep unit/business transformations separate.

---

## 95. Debugging Questions

When conversion behaves unexpectedly, ask:

1. What is the source representation?
2. What is the target type?
3. What does the source contract say?
4. Was normalization applied first?
5. Is the value missing, NULL, empty, or invalid?
6. Is the conversion lossy?
7. Is precision or scale being exceeded?
8. Is the timezone known?
9. Is the value actually an identifier rather than a number?
10. Is a business mapping being mistaken for a type conversion?
11. Which conversion version was applied?
12. Can the raw source record be replayed?
13. Did the database reject the value after application validation?
14. Are conversion failures increasing for one field or many?

---

## 96. Production Implementation Sequence

### Step 1 — Inventory source fields

For each field record:

    source representation
    semantic meaning
    target type

### Step 2 — Define conversion contract

Specify:

- accepted formats;
- null policy;
- empty policy;
- range;
- precision;
- scale;
- timezone;
- invalid policy.

### Step 3 — Separate normalization

Implement trimming, case handling, and source-format normalization independently.

### Step 4 — Implement deterministic converters

Create tested converters for:

    integer
    decimal
    boolean
    date
    timestamp
    string

### Step 5 — Add semantic guards

Check:

    ranges
    precision
    scale
    timezone
    identifier semantics

### Step 6 — Add failure classification

Assign stable reason codes.

### Step 7 — Connect to staging

Use T01 lineage and quarantine mechanisms.

### Step 8 — Enforce database types

Let PostgreSQL provide final type constraints.

### Step 9 — Add observability

Track conversion successes and failures.

### Step 10 — Version conversion rules

Preserve the version used for historical processing.

### Step 11 — Test boundaries

Test invalid, extreme, ambiguous, and lossy values.

### Step 12 — Run recovery drills

Replay corrected data from durable raw evidence.

---

## 97. Production Checklist

### Contract

- [ ] Every converted field has a target type.
- [ ] Accepted formats are explicit.
- [ ] Null behavior is defined.
- [ ] Empty behavior is defined.
- [ ] Invalid behavior is defined.
- [ ] Precision and scale are defined.
- [ ] Timezone rules are defined.
- [ ] Conversion version is retained.

### Numeric

- [ ] Monetary values use exact decimal semantics.
- [ ] Integer ranges are validated.
- [ ] Decimal scale is validated.
- [ ] Rounding is explicit.
- [ ] Special numeric values are handled explicitly.

### Temporal

- [ ] DATE and TIMESTAMP semantics are distinguished.
- [ ] Timestamps are timezone-aware when required.
- [ ] Naive timestamps are not silently assumed UTC.
- [ ] DST behavior is tested.
- [ ] Source timezone is preserved in the contract.

### Strings and identifiers

- [ ] Identifiers remain text when appropriate.
- [ ] Leading zeros are preserved.
- [ ] Unicode is preserved.
- [ ] Locale-specific numeric formats are explicit.

### Failure handling

- [ ] Conversion failures have reason codes.
- [ ] Invalid records are quarantined or explicitly fail.
- [ ] Raw evidence is preserved.
- [ ] Retryable and permanent failures are separated.

### Operations

- [ ] Conversion metrics exist.
- [ ] Structured logs exist.
- [ ] Reconciliation exists.
- [ ] Replay is supported.
- [ ] Conversion contracts are versioned.

### Testing

- [ ] Integer boundaries tested.
- [ ] Decimal precision tested.
- [ ] Boolean representations tested.
- [ ] Date boundaries tested.
- [ ] Timestamp timezone tested.
- [ ] Null/missing/empty tested.
- [ ] Overflow tested.
- [ ] Lossy conversions tested.
- [ ] Replay tested.

---

## 98. Production Tools You Should Know

### Python Decimal and datetime

Use for:

- exact decimal conversion;
- timestamp parsing;
- timezone-aware values;
- deterministic conversion tests.

Know:

- Decimal;
- datetime;
- zoneinfo;
- explicit parsing formats.

### PostgreSQL

Use for:

- NUMERIC;
- INTEGER/BIGINT;
- DATE;
- TIMESTAMPTZ;
- final type enforcement;
- constraints.

### Pydantic

Use when a pipeline benefits from declarative typed validation.

Useful capabilities include:

- field typing;
- validation;
- structured validation errors;
- explicit model contracts.

Do not use a validation framework as a substitute for understanding the source semantics.

---

## 99. Package Structure

A practical implementation:

    etl/
      conversion/
        contracts.py
        normalize.py
        integers.py
        decimals.py
        booleans.py
        dates.py
        timestamps.py
        results.py

      staging/
        loader.py
        quarantine.py

      tests/
        test_integers.py
        test_decimals.py
        test_booleans.py
        test_dates.py
        test_timestamps.py
        test_boundaries.py
        test_replay.py

Keep:

    normalization
    conversion
    semantic validation
    loading

as separate responsibilities.

---

## 100. Design Principle — Type Is Part of the Data Contract

A field is not fully defined by its name.

For example:

    amount

needs:

    numeric type
    precision
    scale
    currency semantics

Likewise:

    created_at

needs:

    timestamp semantics
    timezone semantics
    precision

Type is part of meaning.

---

## 101. Design Principle — Never Guess

When a value is ambiguous:

    09/10/2026
    100
    10:00
    "false"

do not guess.

Use the source contract.

A wrong conversion can create syntactically valid but semantically incorrect data, which is harder to detect than a hard failure.

---

## 102. Design Principle — Preserve Evidence

When conversion fails:

    source value
        |
        v
    conversion failure
        |
        +----> reason code
        |
        +----> quarantine
        |
        +----> raw evidence

Do not overwrite the source merely to make the pipeline pass.

---

## 103. Design Principle — Database Is the Final Guard

The application should perform explicit conversion and validation.

The database should still enforce:

    column type
    NOT NULL
    CHECK
    UNIQUE
    foreign key

Neither layer should be treated as the only line of defense.

---

## 104. Design Principle — Separate Technical Conversion From Business Transformation

Technical conversion:

    "42" -> 42

Business transformation:

    42 EUR -> reporting amount

Do not hide business rules inside generic type converters.

This separation keeps pipelines easier to test, reason about, and change.

---

## 105. Definition of Done

You are done with T02 when you can independently implement a conversion layer that:

1. defines target types explicitly;
2. distinguishes missing, NULL, empty, and invalid values;
3. converts integers safely;
4. converts decimals without float precision loss;
5. enforces precision and scale;
6. handles boolean representations explicitly;
7. parses dates without locale guessing;
8. handles timezone-aware timestamps correctly;
9. detects ambiguous or naive timestamps;
10. preserves identifiers as text when appropriate;
11. handles overflow and invalid numeric values;
12. separates normalization from conversion;
13. separates technical conversion from business transformation;
14. produces deterministic results;
15. provides stable conversion error codes;
16. supports quarantine and replay;
17. preserves source evidence and lineage;
18. enforces target types in PostgreSQL;
19. exposes conversion failures through observability;
20. passes boundary, failure, and recovery tests.

If you can implement these mechanisms independently, you understand production data type conversion.

---

## 106. What You Learned

The central model is:

    source representation
            |
            v
       type contract
            |
            v
        normalization
            |
            v
        conversion
            |
       +----+----+
       |         |
       v         v
     valid     invalid
       |         |
       v         v
   typed value quarantine
       |
       v
 downstream transformation

Key rules:

1. Never let implicit casts decide business meaning.
2. Define the target type explicitly.
3. Distinguish technical conversion from semantic transformation.
4. Use Decimal for exact decimal data.
5. Never guess ambiguous dates or timezones.
6. Keep identifiers as text when leading zeros or non-numeric semantics matter.
7. Make rounding explicit.
8. Validate target ranges and precision.
9. Preserve raw evidence when conversion fails.
10. Give failures stable reason codes.
11. Keep conversion deterministic and versioned.
12. Let the database provide final type enforcement.
13. Test boundaries rather than only happy paths.
14. Make replay safe through deterministic conversion and staging identity.

Next:

    T03 — Null Handling
