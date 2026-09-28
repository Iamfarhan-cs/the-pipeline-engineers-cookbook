# T08 — Numeric Transformation

> **Goal:** Convert, normalize, validate, round, and persist numeric values without silently changing magnitude, precision, scale, sign, or business meaning.

Numeric transformation is not simply calling `float()` or casting a database column.

In production pipelines, numeric errors can change money, quantities, rates, percentages, measurements, identifiers, and analytical results while still producing syntactically valid data.

Typical failures include:

- floating-point rounding changing monetary values
- cents interpreted as dollars
- percentages interpreted as fractions
- integers overflowing a target type
- decimal scale being silently truncated
- negative values becoming positive through absolute-value cleanup
- rounding being applied at the wrong stage
- `NaN` or infinity entering a relational table
- numeric strings containing locale-specific separators
- a transformation converting a large integer into a floating-point value and losing exactness.

By the end of this recipe, you should be able to design numeric contracts, implement safe transformations, test precision and boundaries, observe numeric quality, intentionally break numeric processing, and recover affected data.

---

## 1. Problem Recognition

### 1.1 Common source representations

A numeric field may arrive as:

    125
    125.50
    "125.50"
    "1,250.50"
    "1250.50"
    12500   # cents
    0.125
    "12.5%"

These representations do not necessarily have the same business meaning.

### 1.2 First question: what does the number mean?

Before transforming a value, identify:

| Question | Example |
|---|---|
| What quantity? | account balance |
| What unit? | USD |
| What scale? | cents or dollars |
| What numeric domain? | non-negative money |
| What precision? | 2 decimal places |
| What range? | 0 to 1,000,000 |
| What rounding rule? | half-even |
| Is exactness required? | yes for monetary amount |

The numeric type is only one part of the contract.

### 1.3 Red flags

Investigate when:

- amounts change by exactly 100× or 1,000×
- values suddenly gain or lose decimal places
- totals differ by small amounts after a migration
- percentages become 12.5 instead of 0.125, or vice versa
- very large IDs stop matching exactly
- negative numbers appear where the domain prohibits them
- `NaN` or infinity appears in analytical output
- values change after a load/reload cycle
- a source switches decimal separators or thousands separators.

---

## 2. Concept and Reasoning

## 2.1 Numeric type versus numeric meaning

Consider:

    1250

It could mean:

- $1,250
- 1,250 cents
- 1,250 units
- an identifier
- 1,250 basis points.

Do not infer business meaning from the numeric representation alone.

### 2.2 Integer

Use integers for quantities that are naturally discrete and exactly representable:

- counts
- sequence numbers
- whole units
- minor currency units such as cents, when explicitly modeled that way.

### 2.3 Decimal

Decimal arithmetic is appropriate when exact base-10 representation matters, especially for financial quantities.

Python example:

    from decimal import Decimal

    amount = Decimal("10.10")
    tax = Decimal("0.20")
    total = amount + amount * tax

Use strings when constructing `Decimal` from external decimal text.

Prefer:

    Decimal("0.1")

over:

    Decimal(0.1)

because the latter begins with the binary floating-point approximation already stored in the Python float.

### 2.4 Floating-point numbers

Binary floating-point is useful for many scientific and analytical workloads, but it does not represent every decimal fraction exactly.

Example:

    0.1 + 0.2

may not equal exactly:

    0.3

That does not make floating-point unusable. It means the representation and comparison strategy must match the domain.

### 2.5 Monetary values

For monetary amounts, define:

- currency
- scale
- precision
- rounding policy.

Example contract:

    amount: Decimal
    currency: ISO currency code
    scale: 2
    rounding: ROUND_HALF_EVEN

Do not assume every currency has the same number of decimal places.

### 2.6 Fixed-point minor units

An alternative financial representation is an integer minor unit:

    1250 cents = $12.50

This can be useful when the domain explicitly defines a fixed minor unit.

The critical requirement is that the unit is part of the contract.

### 2.7 Precision versus scale

These concepts should be distinguished.

- **Precision:** total number of significant digits.
- **Scale:** digits represented to the right of the decimal point.

Example:

    1234.56

has six total digits and two fractional digits.

A database schema such as `NUMERIC(12,2)` expresses both a maximum precision and scale.

### 2.8 Rounding is a business rule

These operations are different:

    truncate
    round half up
    round half even
    floor
    ceiling

Never add rounding merely because a target column has fewer decimal places.

First determine the business rule.

### 2.9 Rounding stage matters

Suppose several line items are transformed and then aggregated.

These can produce different results:

    round(each item) → sum

versus:

    sum(each item) → round(total)

The correct strategy depends on the domain contract.

### 2.10 Units matter

A numeric transformation should make units explicit.

Bad:

    amount = value

Better:

    amount_usd = Decimal(value)

or:

    amount_minor_units = int(value)

Names and schemas should communicate the unit when it is not obvious from the domain.

### 2.11 Percentages and ratios

Define whether:

    0.125

means 12.5% or 0.125%.

A safe contract might explicitly state:

    rate_fraction = 0.125

or:

    rate_percent = 12.5

Never convert between the two implicitly.

### 2.12 Numeric identifiers

Do not treat every numeric-looking field as a number.

Values such as:

    001234567890

may be identifiers whose leading zeros are meaningful.

An account number, postal code, phone number, or external identifier may need string semantics even when it contains only digits.

### 2.13 Missing, invalid, and non-finite values

Distinguish:

    NULL
    empty string
    invalid numeric text
    NaN
    +infinity
    -infinity

These should not automatically collapse into zero.

---

## 3. Implementation

## 3.1 Define a numeric contract

Example:

| Field | Contract |
|---|---|
| `amount` | Decimal monetary amount |
| `currency` | Currency code |
| `rate_fraction` | Decimal from 0 through 1 |
| `quantity` | Non-negative integer |
| `measurement` | Decimal with documented unit |

Also define:

- accepted source formats
- missing-value behavior
- valid range
- precision
- scale
- rounding
- overflow behavior.

## 3.2 Parse decimal text safely

    from decimal import Decimal, InvalidOperation

    def parse_decimal(value: str) -> Decimal:
        if value is None:
            raise ValueError("numeric value is required")

        text = value.strip()
        if not text:
            raise ValueError("numeric value cannot be empty")

        try:
            return Decimal(text)
        except InvalidOperation as exc:
            raise ValueError(f"invalid numeric value: {value!r}") from exc

Keep parsing separate from business validation.

## 3.3 Reject non-finite Decimal values

    def require_finite(value: Decimal) -> Decimal:
        if not value.is_finite():
            raise ValueError("numeric value must be finite")
        return value

Use this when the domain does not permit `NaN` or infinity.

## 3.4 Validate a range

    from decimal import Decimal

    def validate_rate(rate: Decimal) -> Decimal:
        if rate < Decimal("0") or rate > Decimal("1"):
            raise ValueError("rate must be between 0 and 1")
        return rate

Do not silently clamp:

    rate = max(Decimal("0"), min(rate, Decimal("1")))

unless clamping is explicitly part of the business contract.

## 3.5 Quantize to an explicit scale

    from decimal import Decimal, ROUND_HALF_EVEN

    def round_money(amount: Decimal) -> Decimal:
        return amount.quantize(
            Decimal("0.01"),
            rounding=ROUND_HALF_EVEN,
        )

Quantization should be an explicit policy, not an incidental side effect.

## 3.6 Minor-unit conversion

    from decimal import Decimal, ROUND_HALF_EVEN

    def dollars_to_cents(amount: Decimal) -> int:
        rounded = amount.quantize(Decimal("0.01"), rounding=ROUND_HALF_EVEN)
        return int(rounded * 100)

Only use this when the domain explicitly defines cents as the storage unit.

## 3.7 Parse percentages explicitly

    from decimal import Decimal

    def percent_to_fraction(value: str) -> Decimal:
        text = value.strip().rstrip("%")
        percent = Decimal(text)
        return percent / Decimal("100")

Then:

    assert percent_to_fraction("12.5%") == Decimal("0.125")

Do not apply this function to an input contract that already provides fractions.

## 3.8 Locale-aware input

A source may provide:

    "1,234.56"

or:

    "1.234,56"

Do not remove every punctuation character blindly.

Instead, define the source locale/format explicitly and parse according to that contract.

A generic cleanup such as:

    text.replace(",", "")

can turn a decimal separator into incorrect magnitude.

## 3.9 PostgreSQL numeric types

Example:

    CREATE TABLE payments (
        payment_id text PRIMARY KEY,
        amount numeric(18, 2) NOT NULL,
        currency text NOT NULL,
        quantity bigint
    );

Choose precision and scale from domain requirements and expected ranges.

Do not assume the database schema can compensate for incorrect upstream units.

## 3.10 Avoid accidental float conversion

Bad:

    amount = float(source_value)

followed later by:

    Decimal(amount)

Better for exact decimal text:

    amount = Decimal(source_value)

Keep the value in the appropriate numeric representation throughout the transformation.

## 3.11 Overflow validation

Before loading into a constrained target, validate the allowed range.

    from decimal import Decimal

    MAX_AMOUNT = Decimal("9999999999999999.99")

    def validate_amount(amount: Decimal) -> Decimal:
        if amount < Decimal("0") or amount > MAX_AMOUNT:
            raise ValueError("amount outside supported range")
        return amount

The exact range must come from the target schema and business contract.

## 3.12 Deterministic transformations

Given the same source value and configuration:

    transform(value) == transform(value)

should normally hold.

Do not make numeric transformation depend on machine locale, implicit floating-point behavior, or environment-specific formatting.

---

## 4. Testing

Numeric tests should emphasize exactness and boundaries.

## 4.1 Decimal parsing

    def test_parse_decimal():
        assert parse_decimal("125.50") == Decimal("125.50")

## 4.2 Invalid numeric text

    def test_invalid_decimal():
        try:
            parse_decimal("abc")
            assert False
        except ValueError:
            pass

Test:

    "abc"
    "1.2.3"
    ""
    "   "
    "$10.00"

according to the source contract.

## 4.3 Precision tests

    def test_decimal_does_not_use_binary_float():
        assert Decimal("0.1") + Decimal("0.2") == Decimal("0.3")

Also test values with many significant digits when the target schema supports them.

## 4.4 Rounding tests

Create explicit fixtures for the selected rounding policy.

    assert round_money(Decimal("10.125")) == Decimal("10.12")

The expected result depends on the configured rounding mode. The important rule is to encode the chosen policy in tests rather than relying on an undocumented default.

## 4.5 Scale tests

Verify:

    10
    10.1
    10.12
    10.123

against the target contract.

Test whether extra fractional digits should be rejected, rounded, or preserved.

## 4.6 Range tests

Test:

    minimum valid value
    maximum valid value
    one unit below minimum
    one unit above maximum

Do not test only ordinary values.

## 4.7 Sign tests

Test:

    0
    positive
    negative

according to the domain.

Do not silently apply `abs()` to invalid negative values.

## 4.8 Unit tests

Explicitly test unit conversions:

    1250 cents → 12.50 dollars
    12.5 percent → 0.125 fraction

Also test that an already-normalized value is not converted twice.

## 4.9 Idempotence

For a canonicalization function:

    normalize(normalize(x)) == normalize(x)

This is particularly important for rounding and unit normalization.

## 4.10 Large integer tests

Use values larger than the range of common 32-bit integers when the domain requires them.

Verify that identifiers and large quantities are not accidentally converted through floating-point representations.

## 4.11 Database round-trip tests

Insert representative values into the target database and read them back.

Verify:

- value
- precision
- scale
- sign
- NULL behavior.

---

## 5. Observability

Numeric transformations should produce operational signals without logging sensitive amounts unnecessarily.

| Signal | Why it matters |
|---|---|
| numeric parse failures | Detect source format problems |
| range violations | Detect domain or source anomalies |
| overflow/rejection count | Detect target capacity problems |
| rounding count | Detect scale mismatches |
| unit conversion count | Detect source contract application |
| non-finite value count | Detect invalid analytical values |
| negative-value violations | Detect domain violations |
| precision-loss count | Detect schema or transformation loss |
| source format distribution | Detect upstream formatting changes |

Prefer aggregate metrics or safe identifiers where monetary values are sensitive.

Example structured event:

    {
      "event": "numeric_transform_failed",
      "field": "amount",
      "reason": "out_of_range",
      "source_system": "payments-api",
      "record_id": "payment_123"
    }

Do not log complete financial payloads just to diagnose numeric parsing.

---

## 6. Intentional Failure

### Failure 1 — Float conversion for money

Change the implementation to:

    amount = float(source_value)

and perform subsequent arithmetic using floats.

Expected symptom:

- small representation differences
- inconsistent rounding at boundaries
- reconciliation differences after aggregation.

Diagnosis:

- compare exact decimal arithmetic with float arithmetic
- inspect the transformation path for implicit casts.

### Failure 2 — Cents interpreted as dollars

Take:

    1250

and interpret it directly as `$1,250` when the source contract says the value is cents.

Expected symptom:

    expected: $12.50
    actual:   $1,250.00

Diagnosis:

- inspect source unit
- compare representative raw values
- verify transformation configuration.

### Failure 3 — Percentage converted twice

Take:

    0.125

and pass it through a function intended for `12.5` percent.

Expected symptom:

    0.125 → 0.00125

Diagnosis:

- inspect field contract
- identify repeated unit conversion.

### Failure 4 — Silent clamping

Change validation to clamp values into an allowed range.

Expected symptom:

- invalid source values become apparently valid
- data quality incidents disappear from metrics
- downstream aggregates are biased.

Recovery:

- restore rejected source values
- remove silent clamping
- reprocess from trusted raw data.

### Failure 5 — Precision loss through float

Convert a large integer or high-precision decimal through a float.

Expected symptom:

- values differ after round-trip conversion
- identifiers or exact quantities no longer match.

---

## 7. Recovery

### 7.1 Wrong unit

1. Identify the source unit from the contract.
2. Determine the affected processing window.
3. Compare raw and transformed values.
4. Recover from raw data.
5. Apply the correct unit conversion.
6. Re-run downstream calculations.
7. Reconcile aggregates.
8. Add a regression test.

### 7.2 Wrong rounding policy

1. Identify the intended business rounding rule.
2. Determine whether rounding occurred before or after aggregation.
3. Recover the highest-precision source available.
4. Recompute using the approved policy.
5. Reconcile affected financial or analytical outputs.
6. Version the transformation policy if needed.

### 7.3 Float contamination

If exact values were converted through floats:

1. Identify affected transformations.
2. Recover original decimal text or high-precision source values.
3. Recompute using exact decimal arithmetic.
4. Compare before/after aggregates.
5. Prevent implicit float conversion in the transformation contract.

### 7.4 Target overflow

If values exceed the target schema:

1. Do not truncate the values.
2. Identify whether the target schema is incorrect or the source is invalid.
3. Determine the approved supported range.
4. Correct the schema or reject/quarantine invalid values.
5. Reprocess only after the contract is resolved.

---

## 8. Production Tools You Should Know

### 8.1 Python `decimal`

Know:

- `Decimal`
- `quantize()`
- rounding modes
- `is_finite()`
- decimal contexts.

Use it when exact decimal semantics are required.

### 8.2 PostgreSQL `NUMERIC`

Know:

- `NUMERIC(precision, scale)`
- numeric arithmetic
- casts
- overflow behavior
- rounding and scale interactions.

Database precision should reflect a deliberate domain contract.

### 8.3 Pandas / NumPy

Know where floating-point arithmetic is appropriate and where it is not.

For dataframe workloads, understand:

- integer and floating dtypes
- missing numeric values
- explicit conversion
- overflow behavior
- decimal/object representations when exact decimal semantics are required.

Do not choose a dtype merely because it is convenient.

---

## 9. Production Runbook

### Symptom: monetary totals differ slightly

Check:

1. Float conversions.
2. Rounding mode.
3. Rounding stage.
4. Decimal scale.
5. Aggregation order.

### Symptom: values are exactly 100× too large or small

Check:

1. Minor units versus major units.
2. Currency-specific scale.
3. Duplicate conversion.

### Symptom: percentage values look wrong

Check:

1. Fraction versus percent representation.
2. Double conversion.
3. Field-level contract.

### Symptom: database rejects numeric values

Check:

1. Precision.
2. Scale.
3. Range.
4. Sign constraints.
5. Source unit.

### Symptom: large integers no longer match

Check:

1. Float conversion.
2. JSON serialization.
3. Database type.
4. Application type.
5. Whether the value is actually an identifier and should be text.

### Symptom: numeric parse failures suddenly increase

Check:

1. Source formatting.
2. Locale.
3. Thousands/decimal separators.
4. New currency symbols or percent signs.
5. Upstream schema changes.

---

## 10. Common Mistakes

### Mistake 1 — Using float for every numeric field

Numeric representation must match the domain.

### Mistake 2 — Treating money as formatting

Money requires explicit currency, scale, precision, and rounding semantics.

### Mistake 3 — Guessing units

1250 without a unit is incomplete information.

### Mistake 4 — Rounding silently

Rounding changes values. It requires an explicit policy.

### Mistake 5 — Clamping invalid values

Turning invalid input into a valid-looking number hides source defects.

### Mistake 6 — Converting identifiers to numbers

Leading zeros and exact representation may matter.

### Mistake 7 — Ignoring non-finite values

Analytical systems can produce `NaN` or infinity; relational targets may not accept them as expected.

### Mistake 8 — Removing punctuation blindly

Locale-specific separators can change magnitude.

### Mistake 9 — Validating only after database failure

Business validation should happen before loading into the target.

### Mistake 10 — Losing original precision

Derived low-precision values are difficult or impossible to reconstruct.

---

## 11. Definition of Done

T08 is complete when you can:

- define a numeric field's business meaning and unit
- distinguish integer, decimal, and floating-point semantics
- explain precision and scale
- parse decimal text safely
- avoid unnecessary float conversion
- define explicit rounding behavior
- convert minor units only when the contract requires it
- distinguish percentages from fractions
- validate numeric ranges
- detect overflow
- distinguish numeric identifiers from quantities
- handle missing, invalid, and non-finite values explicitly
- write boundary and precision tests
- test database round trips
- observe numeric transformation failures
- intentionally reproduce unit and precision failures
- recover incorrect numeric transformations from trusted raw data.

---

## 12. What You Learned

The central lesson is:

> **A number is only correct when its representation, unit, precision, scale, range, and business meaning agree.**

Before transforming a numeric value, ask:

1. What quantity does this represent?
2. What unit is it in?
3. Does exact decimal representation matter?
4. What precision and scale are required?
5. What rounding rule applies?
6. What values are invalid?
7. Could the transformation overflow or lose information?
8. Can the original value be recovered from trusted raw data?

If those answers are explicit, numeric transformation becomes controlled engineering rather than silent mathematical corruption.

---

## Next Recipe

**T09 — Boolean Normalization**

T09 will cover the many representations of true/false values, strict versus permissive parsing, three-state logic, database NULL behavior, and safe boolean normalization.