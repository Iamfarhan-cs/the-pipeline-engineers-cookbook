# T09 — Boolean Normalization

> **Goal:** Convert inconsistent true/false representations into explicit boolean semantics without turning unknown, missing, or invalid values into false.

Boolean transformation looks simple because the final type often contains only two values:

    true
    false

In production ETL, the source may contain:

    true
    false
    TRUE
    FALSE
    yes
    no
    Y
    N
    1
    0
    enabled
    disabled
    active
    inactive
    ""
    NULL

These values cannot safely be converted using generic truthiness.

The central rule is:

> **Unknown is not automatically false.**

---

## 1. Problem Recognition

### 1.1 Typical source representations

A boolean-like field can arrive as:

| Source value | Possible meaning |
|---|---|
| `true` | true |
| `false` | false |
| `"Y"` | true |
| `"N"` | false |
| `"1"` | true |
| `"0"` | false |
| `"enabled"` | true |
| `"disabled"` | false |
| `NULL` | unknown/missing |
| `"unknown"` | unknown or invalid |
| `"yes"` | possibly true |

The correct mapping is defined by the source contract.

### 1.2 The dangerous Python bug

Never assume this is a boolean parser:

    bool("false")

It evaluates to `True` because the string is non-empty.

Likewise:

    bool("0")

also evaluates to `True`.

Generic truthiness answers whether an object is considered truthy by the programming language. It does not answer whether a source value means business-level true.

### 1.3 Red flags

Investigate when:

- every non-empty string becomes true
- NULL counts suddenly drop after a migration
- false values become true
- unknown values disappear from quality metrics
- a source changes from `Y/N` to `true/false`
- a database integer flag is interpreted as an ordinary numeric field
- an API starts returning strings instead of JSON booleans.

---

## 2. Concept and Reasoning

## 2.1 Boolean versus tri-state semantics

A field may logically have more than two states:

    TRUE
    FALSE
    UNKNOWN

Examples:

- customer opted in → yes/no/unknown
- verification completed → yes/no/not evaluated
- feature enabled → yes/no/not configured.

If the source distinguishes missing from false, preserve that distinction.

### 2.2 NULL is not FALSE

Consider:

    opted_in = NULL

This does not necessarily mean:

    opted_in = FALSE

`NULL` can mean:

- not supplied
- not known
- not evaluated
- not applicable.

The downstream business rule must determine the interpretation.

### 2.3 Boolean normalization versus validation

Normalization answers:

    "Y" → true

Validation answers:

    "MAYBE" → reject

Keep the two concepts separate.

A normalization function should not silently accept arbitrary values simply to increase successful row counts.

### 2.4 Source-specific contracts

Do not create one universal mapping for every system.

Source A may define:

    Y = true
    N = false

Source B may define:

    1 = active
    0 = inactive

Source C may send:

    enabled
    disabled

The same source value can have different meaning in different fields.

### 2.5 Case and whitespace

Textual boolean values may contain:

    " TRUE "
    "true"
    "True"

A text parser may normalize case and surrounding whitespace **if the source contract permits it**.

Do not use broad string normalization that changes arbitrary field semantics.

### 2.6 Strict versus permissive parsing

Strict parser:

    "true" → true
    "false" → false
    anything else → error

Permissive parser:

    "yes", "Y", "1", "true" → true
    "no", "N", "0", "false" → false

Neither is universally correct.

Use the narrowest accepted set supported by the source contract.

### 2.7 Unknown versus invalid

These are different:

    NULL       → unknown/missing
    "unknown" → possibly an explicit domain state
    "MAYBE"    → invalid if not documented

Do not collapse all three into false.

### 2.8 Boolean fields in analytics

Boolean normalization affects:

- filters
- counts
- conversion rates
- feature adoption
- compliance reporting
- segmentation
- joins
- aggregations.

A single incorrect truthiness rule can change entire dashboards.

### 2.9 Boolean fields in databases

A relational schema may contain:

    BOOLEAN

or legacy representations such as:

    CHAR(1)
    SMALLINT
    TEXT.

Convert legacy representations at a controlled boundary and store the canonical boolean type when appropriate.

---

## 3. Implementation

## 3.1 Define a field contract

Example:

| Field | Accepted true | Accepted false | Missing | Invalid |
|---|---|---|---|---|
| `marketing_opt_in` | `Y`, `YES` | `N`, `NO` | NULL | reject |
| `is_active` | `1` | `0` | NULL | reject |
| `enabled` | `true` | `false` | NULL | reject |

Document whether case and whitespace are accepted.

## 3.2 Strict Python parser

    def parse_boolean(value):
        if value is True or value is False:
            return value

        if value is None:
            return None

        if not isinstance(value, str):
            raise ValueError("boolean value must be bool, string, or NULL")

        text = value.strip().lower()

        if text == "true":
            return True
        if text == "false":
            return False

        raise ValueError(f"invalid boolean value: {value!r}")

This parser is deliberately narrow.

## 3.3 Source-specific parser

For a source contract using `Y/N`:

    def parse_yes_no(value):
        if value is None:
            return None

        text = value.strip().upper()

        if text == "Y":
            return True
        if text == "N":
            return False

        raise ValueError(f"invalid Y/N value: {value!r}")

Do not automatically add `YES`, `1`, or `TRUE` unless the source contract allows them.

## 3.4 Numeric flag parser

    def parse_binary_flag(value):
        if value is None:
            return None

        if value == 1:
            return True
        if value == 0:
            return False

        raise ValueError(f"invalid binary flag: {value!r}")

Be careful with booleans in Python because `bool` is a subclass of `int`.

If the source contract requires strict numeric input, validate the input type explicitly.

## 3.5 Preserve unknown state

Return `None` when the target schema allows NULL and the source value means unknown:

    result = parse_yes_no(None)
    assert result is None

Do not replace it with:

    False

unless the business contract explicitly defines missing as false.

## 3.6 Explicit mapping tables

For larger domains, use a mapping table:

    TRUE_VALUES = {"true", "enabled", "active"}
    FALSE_VALUES = {"false", "disabled", "inactive"}

    def parse_status_boolean(value):
        if value is None:
            return None

        text = value.strip().lower()

        if text in TRUE_VALUES:
            return True
        if text in FALSE_VALUES:
            return False

        raise ValueError(f"invalid status value: {value!r}")

The accepted vocabulary should still be defined by the source contract.

## 3.7 Separate parsing from defaulting

Bad design:

    def parse_boolean(value):
        return bool(value) or False

Better:

    parsed = parse_boolean(value)
    normalized = parsed if parsed is not None else DEFAULT_VALUE

Defaulting is a separate business decision covered by T04.

## 3.8 PostgreSQL boolean conversion

Canonical target:

    CREATE TABLE customers (
        customer_id text PRIMARY KEY,
        is_active boolean,
        marketing_opt_in boolean
    );

Convert legacy source values before loading:

    CASE
        WHEN source_flag = 'Y' THEN TRUE
        WHEN source_flag = 'N' THEN FALSE
        ELSE NULL
    END

If invalid values should fail rather than become NULL, validate them before this SQL expression or use an explicit staging/quality path.

## 3.9 Normalize at the trust boundary

A robust flow is:

    raw source value
          ↓
    source-specific parser
          ↓
    validation
          ↓
    canonical BOOLEAN / NULL
          ↓
    downstream logic

Do not make every dashboard or downstream service understand `Y`, `N`, `1`, `0`, and `enabled`.

---

## 4. Testing

Boolean tests should prove both accepted mappings and rejected values.

## 4.1 Strict parser tests

    def test_true():
        assert parse_boolean("true") is True

    def test_false():
        assert parse_boolean("false") is False

    def test_case_and_whitespace():
        assert parse_boolean(" TRUE ") is True

    def test_null():
        assert parse_boolean(None) is None

### 4.2 Truthiness regression test

Explicitly protect against:

    assert parse_boolean("false") is False
    assert parse_boolean("0") is not True

The first assertion is especially important because `bool("false")` is true in Python.

### 4.3 Rejected values

Test values such as:

    "yes"
    "no"
    "Y"
    "N"
    "1"
    "0"
    "maybe"
    "unknown"
    ""

for the strict parser.

Some may be valid for another source-specific parser. The point is to enforce the correct contract.

### 4.4 Y/N tests

    assert parse_yes_no("Y") is True
    assert parse_yes_no("N") is False
    assert parse_yes_no(" y ") is True
    assert parse_yes_no(None) is None

Also verify unsupported values are rejected.

### 4.5 Numeric flag tests

    assert parse_binary_flag(1) is True
    assert parse_binary_flag(0) is False

Test invalid values:

    -1
    2
    100
    None

according to the contract.

### 4.6 Type tests

Test values that look similar but have different types:

    True
    False
    1
    0
    "1"
    "0"

Do not accidentally accept a value only because the programming language considers it truthy or equal under coercion.

### 4.7 Three-state tests

If NULL means unknown, test all three states:

    TRUE
    FALSE
    NULL

Then verify downstream counts preserve the distinction.

### 4.8 Idempotence

Once a value is canonical:

    normalize(True) == True
    normalize(False) == False
    normalize(None) is None

The transformation should not flip or reinterpret canonical values.

### 4.9 Database round-trip

Load:

    TRUE
    FALSE
    NULL

Read them back and verify that the semantic states are preserved.

---

## 5. Observability

Track boolean normalization quality explicitly.

| Signal | Why it matters |
|---|---|
| parse failures | Detect source vocabulary changes |
| NULL count | Detect missing/unknown changes |
| true count | Detect distribution changes |
| false count | Detect distribution changes |
| accepted source representation | Detect contract drift |
| rejected vocabulary | Detect new upstream values |
| defaulted boolean count | Detect business-default application |

Example:

    {
      "event": "boolean_transform_failed",
      "field": "marketing_opt_in",
      "reason": "unsupported_value",
      "source_system": "crm",
      "record_id": "customer_123"
    }

Do not log entire records just to identify an unsupported boolean value.

### 5.1 Distribution monitoring

Suppose a source historically contains:

    Y = 72%
    N = 26%
    NULL = 2%

A sudden change to:

    Y = 0%
    N = 0%
    NULL = 100%

could indicate an upstream schema or extraction failure rather than a real business change.

Monitor the distribution and accepted source vocabulary.

---

## 6. Intentional Failure

### Failure 1 — Use Python truthiness

Replace explicit parsing with:

    result = bool(source_value)

Expected symptom:

    "false" → True
    "0"     → True
    "N"     → True

Diagnosis:

- inspect the parser
- reproduce the values in a unit test
- compare language truthiness with domain semantics.

### Failure 2 — Collapse NULL into false

Change:

    None → False

Expected symptom:

- unknown records disappear
- opt-in/opt-out reporting changes
- completeness appears artificially improved.

Recovery:

- restore NULL semantics
- reprocess affected records
- recalculate downstream aggregates.

### Failure 3 — Overly broad vocabulary

Accept every value containing words such as:

    yes
    enabled
    active

Expected symptom:

- upstream contract violations are hidden
- unrelated domain values may become true.

### Failure 4 — Silent invalid-to-false conversion

Convert:

    "MAYBE" → False

Expected symptom:

- invalid source records appear valid
- quality failures are hidden.

---

## 7. Recovery

### 7.1 Wrong boolean mapping

1. Identify the source and field contract.
2. Determine the deployment window.
3. Compare raw values with canonical booleans.
4. Identify affected records.
5. Restore from trusted raw data.
6. Apply the corrected mapping.
7. Re-run downstream transformations.
8. Reconcile true/false/NULL distributions.

### 7.2 NULL incorrectly converted to false

1. Determine when the default was introduced.
2. Recover raw NULL states.
3. Restore NULL where the contract requires unknown.
4. Recompute affected analytics.
5. Review whether any business default is actually required.

### 7.3 New upstream vocabulary

If a source changes from `Y/N` to `YES/NO`:

1. Do not silently widen the parser.
2. Confirm the upstream contract change.
3. Add explicit accepted mappings.
4. Add regression tests.
5. Version the transformation if necessary.
6. Reprocess quarantined records.

### 7.4 Invalid values previously mapped to false

If values such as `MAYBE` were incorrectly converted:

1. Recover the original raw value.
2. Identify affected records.
3. Mark them invalid or unknown according to the contract.
4. Reprocess after the correct rule is deployed.
5. Recalculate downstream metrics.

---

## 8. Production Tools You Should Know

### 8.1 Python

Know:

- explicit equality checks
- type checks
- `None` semantics
- truthiness versus domain parsing.

Do not rely on `bool()` as a domain-specific parser.

### 8.2 PostgreSQL

Know:

- `BOOLEAN`
- `TRUE` / `FALSE`
- `NULL`
- `CASE` expressions
- three-valued SQL logic.

Understand that SQL's NULL behavior can make boolean expressions produce an unknown result rather than true or false.

### 8.3 Pandas

Know:

- boolean columns
- nullable boolean types
- missing-value behavior
- explicit mapping before conversion.

Do not assume an ordinary dataframe boolean dtype preserves the same semantics as a nullable business field.

---

## 9. Production Runbook

### Symptom: all non-empty values become true

Check:

1. Use of generic truthiness.
2. String parser.
3. Accepted vocabulary.
4. Source field type.

### Symptom: NULL count suddenly falls

Check:

1. Default values.
2. `NULL → FALSE` mapping.
3. Source extraction.
4. Database constraints.

### Symptom: true/false distribution suddenly changes

Check:

1. Source vocabulary.
2. Upstream release.
3. Mapping table.
4. Sampling of raw values.

### Symptom: database boolean conversion fails

Check:

1. Legacy source representation.
2. Invalid values.
3. CASE mapping.
4. Target column type.

### Symptom: analytics disagree between systems

Check:

1. NULL handling.
2. Defaulting.
3. Source-specific mappings.
4. SQL three-valued logic.

---

## 10. Common Mistakes

### Mistake 1 — Using `bool()` for strings

Non-empty strings are truthy regardless of whether they spell `false`.

### Mistake 2 — Treating NULL as false

Missing/unknown is not automatically negative.

### Mistake 3 — One universal mapping

Boolean vocabularies are source- and field-specific.

### Mistake 4 — Accepting every synonym

Broad parsers hide contract violations.

### Mistake 5 — Defaulting during parsing

Parsing and business defaulting are separate decisions.

### Mistake 6 — Ignoring SQL NULL semantics

SQL has three-valued logic around NULL.

### Mistake 7 — Converting invalid values to false

That creates plausible but incorrect data.

### Mistake 8 — Testing only true and false

Test NULL, invalid values, source-specific vocabulary, and type differences.

### Mistake 9 — Losing source vocabulary

Preserving raw evidence can make contract changes and incident recovery much easier.

### Mistake 10 — Hiding distribution changes

Boolean distributions are often useful data-quality signals.

---

## 11. Definition of Done

T09 is complete when you can:

- distinguish domain boolean semantics from programming-language truthiness
- explain why `bool("false")` is not a boolean parser
- define a field-specific boolean contract
- distinguish TRUE, FALSE, NULL, unknown, and invalid
- build strict parsers
- build source-specific parsers
- normalize case and whitespace only when permitted
- preserve NULL semantics
- separate parsing from defaulting
- validate legacy Y/N and 0/1 representations
- normalize into database BOOLEAN values
- test three-state behavior
- test invalid and unexpected vocabulary
- monitor true/false/NULL distributions
- intentionally reproduce truthiness and NULL-collapsing failures
- recover incorrect boolean transformations from trusted raw data.

---

## 12. What You Learned

The central lesson is:

> **Boolean normalization is a contract problem, not a truthiness problem.**

Before converting a boolean-like value, ask:

1. What values does this source actually define?
2. Does NULL mean unknown, not applicable, or false?
3. Which values are valid and which are invalid?
4. Is the parser strict enough to detect contract drift?
5. Is defaulting a separate business decision?
6. Can the original source representation be recovered?

If those answers are explicit, boolean normalization becomes deterministic, observable, and safe.

---

## Next Recipe

**T10 — Code/Status Mapping**

T10 will cover mapping source codes and statuses into canonical domain values, including lookup tables, unknown codes, effective dates, versioning, and safe handling of upstream vocabulary changes.