# T03 — Null Handling

## 1. Problem Recognition

Production data contains more states than simply "has a value" or "does not have a value":

    field missing
    field = NULL
    field = ""
    field = "   "
    field = "NULL"
    field = "N/A"
    field = 0
    field = false

These values are not interchangeable.

A pipeline that collapses them can create:

- false business values;
- incorrect aggregates;
- broken joins;
- accidental updates;
- misleading completeness metrics;
- hidden source-system failures.

The core rule is:

> Never treat absence, NULL, empty text, zero, false, and literal null markers as equivalent without an explicit contract.

---

## 2. What You Are Building

A production null-handling boundary:

    source record
         |
         v
    classify source state
         |
         v
    source-specific normalization
         |
         v
    field null policy
         |
         +----------------+
         |                |
         v                v
       valid            invalid
         |                |
         v                v
    type validation    quarantine/fail
         |
         v
       staging

The goal is not to eliminate NULLs.

The goal is to make every NULL intentional and explainable.

---

## 3. Learning Objectives

By the end of this recipe you should be able to:

1. Distinguish missing, NULL, empty, whitespace, zero, and false.
2. Define field-level null contracts.
3. Normalize source-specific null markers.
4. Apply defaults safely.
5. Understand SQL three-valued logic.
6. Handle NULLs in joins and aggregations.
7. Handle NULLs in Python without truthiness bugs.
8. Preserve nullable fields through transformations.
9. Handle NULL semantics in CDC and upserts.
10. Measure null rates and missingness.
11. Quarantine invalid null states.
12. Replay corrected null policies safely.

---

## 4. What NULL Means

NULL can represent:

    unknown
    unavailable
    not applicable
    not provided
    not yet occurred

The exact meaning comes from the data contract.

NULL is not automatically:

    0
    false
    empty string
    "NULL"

---

## 5. Missing vs Explicit NULL

JSON can contain:

    {}

or:

    {"email": null}

The first means the field is absent.

The second means the field is explicitly present with a null value.

For some pipelines this distinction is essential.

Example:

    missing -> producer did not send the field
    null    -> producer explicitly sent no value

Preserve the distinction when update or audit semantics require it.

---

## 6. Empty String

An empty string is a value:

    ""

It is not SQL NULL.

Whether it should become NULL is a source-specific contract decision.

Do not use generic truthiness logic such as:

    value or None

because it can also collapse:

    0
    false

into NULL.

---

## 7. Whitespace

These are different source representations:

    ""
    " "
    "   "

A source may use whitespace accidentally or intentionally.

A safe normalization step can trim text:

```
def normalize_text(value):
    if value is None:
        return None

    if isinstance(value, str):
        return value.strip()

    return value
```

Do not automatically convert the resulting empty string to NULL unless the field contract says so.

---

## 8. Literal Null Markers

Sources sometimes send:

    NULL
    null
    N/A
    NA
    unknown
    -

These are ordinary strings until the source contract says otherwise.

For one source:

    "N/A" -> NULL

For another:

    "N/A" -> business value

Never create a global null-marker dictionary and apply it to every dataset.

---

## 9. Field Null Contract

Every important field should define:

    missing behavior
    explicit NULL behavior
    empty behavior
    whitespace behavior
    null-marker behavior
    default behavior
    invalid behavior

Example:

```
{
  "field": "middle_name",
  "missing": "NULL",
  "null": "NULL",
  "empty": "NULL",
  "default": null,
  "invalid": "QUARANTINE"
}
```

A required identifier might instead be:

```
{
  "field": "customer_id",
  "missing": "REJECT",
  "null": "REJECT",
  "empty": "REJECT"
}
```

---

## 10. Classify Before Normalizing

A useful classifier:

```
def classify_null_state(record, field):
    if field not in record:
        return "MISSING"

    value = record[field]

    if value is None:
        return "NULL"

    if isinstance(value, str):
        if value.strip() == "":
            return "EMPTY"

    return "PRESENT"
```

This should happen before destructive normalization.

Once the source representation is overwritten, you may not be able to determine how the NULL was created.

---

## 11. Extended Classification

For stronger observability:

    MISSING
    NULL
    EMPTY
    WHITESPACE
    NULL_MARKER
    PRESENT

Example:

```
NULL_MARKERS = {"null", "n/a", "na"}

def classify_value(value):
    if value is None:
        return "NULL"

    if not isinstance(value, str):
        return "PRESENT"

    stripped = value.strip()

    if stripped == "":
        return "EMPTY"

    if stripped.lower() in NULL_MARKERS:
        return "NULL_MARKER"

    return "PRESENT"
```

The marker set must belong to the source contract.

---

## 12. Correct Processing Order

Use this sequence:

    raw value
       |
       v
    classify
       |
       v
    source normalization
       |
       v
    null policy
       |
       v
    type validation
       |
       v
    semantic validation
       |
       v
    typed value

Do not turn every abnormal value into NULL before validation.

---

## 13. Null Policy Types

### Allow

NULL is valid.

Example:

    middle_name

### Reject

NULL violates the record contract.

Example:

    customer_id

### Default

A contract-defined value is applied.

Example:

    retry_count -> 0

### Quarantine

The record is retained but excluded from valid processing.

Example:

    required transaction amount is NULL

### Derive

The value is calculated from other fields.

Example:

    full_name from first_name and last_name

Derivation is a transformation, not generic null handling.

---

## 14. Do Not Default Everything

This is dangerous:

    missing transaction_amount -> 0

It changes:

    unknown amount

into:

    known zero amount

Defaults should exist only when the domain contract explicitly defines them.

---

## 15. Default vs Unknown

Suppose:

    country = NULL

Do not automatically assign:

    country = "US"

A guessed value is worse than an explicit unknown state.

Default values must be:

- documented;
- deterministic;
- testable;
- contract-approved.

---

## 16. SQL NULL

SQL NULL represents an unknown or absent value.

This is not the correct comparison:

```
WHERE email = NULL
```

Use:

```
WHERE email IS NULL
```

and:

```
WHERE email IS NOT NULL
```

Equality with NULL does not evaluate to TRUE.

---

## 17. SQL Three-Valued Logic

SQL predicates can evaluate to:

    TRUE
    FALSE
    UNKNOWN

Example:

```
SELECT 1 = NULL;
```

The result is UNKNOWN.

A WHERE clause keeps rows only when the predicate evaluates to TRUE.

This is why NULL can disappear from query results unexpectedly.

---

## 18. NULL and Not-Equal

This query:

```
WHERE status <> 'ACTIVE'
```

does not include rows where status is NULL.

If the intended behavior is to include NULL:

```
WHERE status <> 'ACTIVE'
   OR status IS NULL
```

Make the desired NULL behavior explicit.

---

## 19. COALESCE

SQL can replace NULL with another expression:

```
COALESCE(country, 'UNKNOWN')
```

This is useful when the target output explicitly requires a fallback.

It is dangerous when used only to hide missing data.

Ask:

> Is UNKNOWN really the correct business value?

before applying COALESCE.

---

## 20. NULLIF

NULLIF can convert a specific value to NULL.

Example:

```
NULLIF(TRIM(email), '')
```

This is appropriate when the contract says blank text means no value.

It should not be used globally.

---

## 21. NULL in Arithmetic

Example:

```
SELECT 100 + NULL;
```

The result is NULL.

This is NULL propagation.

Do not automatically convert NULL to zero before arithmetic unless the business meaning requires that behavior.

---

## 22. NULL in Aggregation

Many SQL aggregates ignore NULL values.

For example:

```
SELECT AVG(amount)
FROM payments;
```

If values are:

    100
    200
    NULL

the average is calculated from the two numeric observations.

It does not treat NULL as zero.

This distinction is critical for financial and operational reporting.

---

## 23. COUNT and NULL

These are different:

```
COUNT(*)
COUNT(amount)
```

If there are 100 rows and 20 amount values are NULL:

    COUNT(*) = 100
    COUNT(amount) = 80

A useful completeness query is:

```
SELECT
    COUNT(*) AS total_rows,
    COUNT(amount) AS non_null_amount,
    COUNT(*) - COUNT(amount) AS null_amount
FROM payments;
```

---

## 24. SUM and NULL

If no non-NULL values exist, SUM can produce NULL.

A query can explicitly request zero:

```
COALESCE(SUM(amount), 0)
```

Use this only if:

    no observed values

should mean:

    zero

rather than:

    unknown

---

## 25. NULL in GROUP BY

NULL values can form a group:

```
SELECT country, COUNT(*)
FROM customer
GROUP BY country;
```

All NULL country values can appear as one NULL group.

That is different from a literal:

    "UNKNOWN"

group.

Do not normalize them together unless required.

---

## 26. NULL in JOINs

Normal equality joins do not match NULL keys.

This:

```
a.customer_id = b.customer_id
```

does not match:

    NULL
    NULL

Therefore nullable join keys require explicit reasoning.

Do not replace NULL join keys with arbitrary values just to force a match.

---

## 27. LEFT JOIN NULL Semantics

Example:

```
SELECT
    c.customer_id,
    p.payment_id
FROM customer c
LEFT JOIN payment p
    ON p.customer_id = c.customer_id;
```

If no payment row exists, payment columns become NULL.

That does not mean the payment row existed and its payment_id was NULL.

It can mean:

    no matching payment row exists

Always interpret NULL relative to query semantics.

---

## 28. NULL in Deduplication

Consider:

```
ROW_NUMBER() OVER (
    PARTITION BY customer_id
    ORDER BY updated_at DESC
)
```

If updated_at can be NULL, ordering behavior matters.

Use explicit ordering:

```
ORDER BY updated_at DESC NULLS LAST
```

and a deterministic tie-breaker:

```
ORDER BY
    updated_at DESC NULLS LAST,
    source_sequence DESC,
    record_id DESC
```

This prevents unstable record selection.

---

## 29. Nullable Uniqueness

A UNIQUE constraint involving NULL may not mean:

    only one NULL is allowed

Database semantics matter.

In PostgreSQL, a useful pattern can be:

```
CREATE UNIQUE INDEX ux_customer_email
ON customer(email)
WHERE email IS NOT NULL;
```

This means:

- supplied emails must be unique;
- NULL email values are allowed multiple times.

Use this only when it matches the business rule.

---

## 30. Python NULL Handling

Python normally represents a null value with:

```
value = None
```

Use:

```
if value is None:
    ...
```

Avoid:

```
if not value:
    ...
```

because that treats all of these as false:

    None
    ""
    0
    False
    []
    {}

---

## 31. Safe Python Checks

Check the exact semantic state.

For NULL:

```
if value is None:
    ...
```

For empty text:

```
if isinstance(value, str) and value.strip() == "":
    ...
```

For zero:

```
if value == 0:
    ...
```

For false:

```
if value is False:
    ...
```

Do not collapse these states accidentally.

---

## 32. Optional Text Normalization

Example:

```
def normalize_optional_text(value):
    if value is None:
        return None

    if not isinstance(value, str):
        raise TypeError("EXPECTED_TEXT")

    value = value.strip()

    return value if value else None
```

This deliberately maps empty text to NULL.

Only use it for fields whose contracts allow that mapping.

---

## 33. Nullability vs Validation

These are different questions.

### Nullability

Can the field be absent?

### Validation

If present, is the value valid?

Example:

    age = NULL

may be valid.

But:

    age = "unknown"

may be invalid.

A nullable field is not an unvalidated field.

---

## 34. Required Fields

For a required field:

    missing -> failure
    NULL -> failure
    empty -> failure
    valid value -> continue

Example:

```
REQUIRED_FIELDS = {
    "customer_id",
    "transaction_id",
}
```

Validate before loading into a NOT NULL target.

---

## 35. Optional Fields

An optional field may allow:

    missing -> NULL
    NULL -> NULL

but still reject malformed values.

Example:

    phone = NULL
    phone = "bad-format"

The first may be valid.

The second may be invalid.

Optional does not mean unrestricted.

---

## 36. Defaultable Fields

Example:

    retry_count

Contract:

    missing -> 0
    NULL -> 0

But:

    "abc" -> invalid

The default policy and type validation remain separate.

---

## 37. Source-Specific Null Normalizer

Example:

```
def normalize_null_marker(value, markers):
    if value is None:
        return None

    if isinstance(value, str):
        text = value.strip()

        if text == "":
            return None

        if text.lower() in markers:
            return None

        return text

    return value
```

This is intentionally source-specific.

Do not make the entire platform treat every empty or marker value as NULL.

---

## 38. Field-Level Policy

Example:

```
NULL_POLICY = {
    "middle_name": {
        "allow_null": True,
        "empty_to_null": True,
    },
    "customer_id": {
        "allow_null": False,
        "empty_to_null": True,
    },
    "retry_count": {
        "allow_null": False,
        "default": 0,
    },
}
```

This makes null behavior visible and reviewable.

---

## 39. Applying a Policy

Example:

```
def apply_null_policy(field, value, policy):
    if value is None:
        if "default" in policy:
            return policy["default"]

        if policy.get("allow_null", True):
            return None

        raise ValueError("NULL_NOT_ALLOWED")

    if (
        policy.get("empty_to_null")
        and isinstance(value, str)
        and value.strip() == ""
    ):
        if "default" in policy:
            return policy["default"]

        if policy.get("allow_null", True):
            return None

        raise ValueError("EMPTY_NOT_ALLOWED")

    return value
```

In production, distinguish missing from explicit NULL when update semantics require it.

---

## 40. Required-Field Validation

Example:

```
def validate_required_fields(record, required_fields):
    errors = []

    for field in required_fields:
        if field not in record:
            errors.append({
                "field": field,
                "reason": "REQUIRED_FIELD_MISSING",
            })
            continue

        if record[field] is None:
            errors.append({
                "field": field,
                "reason": "REQUIRED_FIELD_NULL",
            })

    return errors
```

Keep validation results structured so they can feed quarantine and metrics.

---

## 41. Null Failure Reason Codes

Useful stable reason codes:

    REQUIRED_FIELD_MISSING
    REQUIRED_FIELD_NULL
    EMPTY_NOT_ALLOWED
    NULL_NOT_ALLOWED
    INVALID_NULL_MARKER
    DEFAULT_NOT_ALLOWED

Reason codes should be machine-readable.

Do not depend on free-form error text for operational reporting.

---

## 42. Quarantine

A required NULL can follow:

    source record
         |
         v
    null validation
         |
         v
    quarantine
         |
         +----> reason code
         |
         +----> source lineage
         |
         +----> raw evidence
         |
         v
    remediation and replay

Never silently discard the record.

---

## 43. Null Metrics

Track at least:

    null_values_total
    missing_fields_total
    empty_values_total
    null_markers_total
    required_null_failures_total
    defaults_applied_total
    quarantined_null_records_total

Useful dimensions:

    source_system
    dataset
    field
    pipeline
    schema_version

---

## 44. Null Rate

A simple field null rate is:

    null_rate =
        null_or_missing_values / total_records

Track numerator and denominator separately.

Example:

    total records = 1,000,000
    missing/null email = 150,000

    null rate = 15%

A null-rate increase may indicate:

- producer regression;
- schema change;
- source outage;
- legitimate business behavior.

Use contract thresholds and historical baselines together.

---

## 45. Measure Missingness by State

Do not only track one NULL count.

Track:

    missing
    explicit NULL
    empty
    whitespace
    null marker
    defaulted

This helps identify how the source system is behaving.

---

## 46. Nulls in Data Quality

A required field can have:

    allowed null rate = 0%

An optional field may have:

    expected null rate = 20%

Completeness can be measured as:

    completeness = 1 - null_rate

But completeness does not prove correctness.

A non-NULL value can still be invalid.

---

## 47. Referential Integrity

A nullable foreign key can be valid.

Example:

    address_id = NULL

may mean:

    address not provided

A non-NULL foreign key can still be invalid if the parent does not exist.

Therefore:

    NULL foreign key
        !=
    invalid foreign key

The data model decides the rule.

---

## 48. NULL in CDC

CDC can distinguish:

    field unchanged
    field changed to NULL
    field absent
    record deleted

Do not flatten these states before applying the CDC contract.

This is especially important for update events.

---

## 49. PATCH and Update Semantics

For an update payload:

```
{}
```

can mean:

    do not change anything

while:

```
{"phone": null}
```

can mean:

    clear the phone value

If the pipeline collapses both into NULL, it can generate incorrect updates.

---

## 50. Upsert Semantics

Suppose:

    existing email = "a@example.com"
    incoming email = NULL

Question:

> Should NULL clear the email or leave it unchanged?

There are two possible contracts.

### NULL means clear

Use the incoming NULL.

### NULL means no update

Preserve the existing value.

Neither is universally correct.

The source contract decides.

---

## 51. Conditional Upsert

If NULL means no update:

```
UPDATE customer
SET email = COALESCE(EXCLUDED.email, customer.email);
```

But this prevents intentional clearing.

If NULL means clear:

```
UPDATE customer
SET email = EXCLUDED.email;
```

Never choose between these patterns without understanding update semantics.

---

## 52. Temporal NULLs

A NULL timestamp may mean:

    event has not happened yet

Example:

    completed_at = NULL

can mean:

    transaction not completed

Do not replace it with:

    1970-01-01

or:

    current_timestamp

just to eliminate NULLs.

---

## 53. NULL and Event Ordering

If:

    occurred_at = NULL

the event cannot safely be ordered by that timestamp alone.

Possible contract-approved alternatives include:

- source sequence;
- ingestion time as technical metadata;
- quarantine;
- explicit fallback ordering.

Do not invent business event time.

---

## 54. Nulls in JSON

JSON distinguishes:

    field absent

from:

    "field": null

These can have different semantics.

Preserve the structure until the source contract determines the normalization.

---

## 55. Nested NULLs

These are different:

    customer = null

    customer = {}

    customer = {
        "email": null
    }

A recursive transformer should not collapse all three without a documented reason.

---

## 56. Nulls in Arrays

An array can contain:

    [1, 2, null, 4]

The NULL is an element.

That is different from a missing array or missing array member.

Nested data requires explicit null semantics at each structural level.

---

## 57. Schema Evolution

A newly introduced source field may be missing from historical records.

Do not automatically classify old records as invalid.

Use:

    schema version
    field introduction version
    effective date

to determine whether missingness is expected.

---

## 58. Backfills

A backfill may reduce historical NULL rates by deriving values from another source.

That does not mean the original source had complete data.

Track:

    source value
    derived value
    transformation version

when lineage matters.

---

## 59. Null Lineage

When a downstream field is NULL, ask:

    Was it NULL at source?
    Was it normalized to NULL?
    Did a join introduce NULL?
    Did conversion fail?
    Did aggregation produce NULL?
    Did an update explicitly clear it?

Null lineage turns a vague data-quality problem into a traceable pipeline event.

---

## 60. Preserve Evidence

When required for auditability, preserve technical metadata such as:

    original field presence
    original null state
    normalization rule
    transformation version

Do not add excessive metadata to every table.

Preserve enough lineage to explain important transformations.

---

## 61. Generic Null Normalizer

Example:

```
from dataclasses import dataclass

@dataclass
class NullState:
    state: str
    value: object

def classify_and_normalize(
    value,
    null_markers=None,
    empty_to_null=False,
):
    markers = {
        marker.lower()
        for marker in (null_markers or set())
    }

    if value is None:
        return NullState("NULL", None)

    if isinstance(value, str):
        text = value.strip()

        if text == "":
            if empty_to_null:
                return NullState("EMPTY_TO_NULL", None)

            return NullState("EMPTY", value)

        if text.lower() in markers:
            return NullState("NULL_MARKER", None)

        return NullState("PRESENT", text)

    return NullState("PRESENT", value)
```

The state makes normalization observable and testable.

---

## 62. Required Field Validator

Example:

```
def validate_required(record, fields):
    errors = []

    for field in fields:
        if field not in record:
            errors.append({
                "field": field,
                "reason": "REQUIRED_FIELD_MISSING",
            })
            continue

        if record[field] is None:
            errors.append({
                "field": field,
                "reason": "REQUIRED_FIELD_NULL",
            })

    return errors
```

Add empty-string checks for required text fields.

---

## 63. Testing Strategy

For every important field, test:

    missing
    None
    empty
    whitespace
    null marker
    valid value
    invalid value
    zero
    false

This catches most null-handling bugs.

---

## 64. Unit Test — Missing

Input:

```
{}
```

Expected classification:

    MISSING

Do not access the field first and accidentally turn absence into NULL without recording the distinction.

---

## 65. Unit Test — Explicit NULL

Input:

```
{"email": null}
```

Expected:

    NULL

Not:

    MISSING

---

## 66. Unit Test — Empty

Input:

```
{"email": ""}
```

Expected:

    EMPTY

or:

    NULL

if the field contract enables empty-to-null normalization.

---

## 67. Unit Test — Whitespace

Input:

```
{"email": "   "}
```

Verify whether whitespace becomes:

    EMPTY

or:

    NULL

according to the contract.

---

## 68. Unit Test — Zero and False

Input:

```
{"count": 0, "active": false}
```

Both should remain:

    PRESENT

This catches the dangerous pattern:

```
if not value:
    value = None
```

---

## 69. Unit Test — Null Marker

Input:

```
{"country": "N/A"}
```

If the source contract defines N/A as a null marker:

    NULL_MARKER -> NULL

Otherwise it remains a value.

---

## 70. Unit Test — Required Field

Test all of:

    missing -> failure
    NULL -> failure
    empty -> failure
    valid -> success

Do not test only explicit NULL.

---

## 71. Unit Test — Optional Field

Test:

    missing -> NULL
    NULL -> NULL
    empty -> contract-defined behavior
    valid -> preserved

---

## 72. Unit Test — Default

Test:

    missing -> default
    NULL -> default

only when the contract says so.

Also test:

    invalid non-null -> failure

The default must not hide malformed data.

---

## 73. Integration Test — SQL NULL

Example setup:

```
CREATE TEMP TABLE test_null (
    value INTEGER
);

INSERT INTO test_null VALUES
    (1),
    (NULL),
    (2);
```

Verify:

```
SELECT COUNT(*) FROM test_null;
SELECT COUNT(value) FROM test_null;
```

Expected:

    COUNT(*) = 3
    COUNT(value) = 2

---

## 74. Integration Test — LEFT JOIN

Create a parent row with no child row.

Run a LEFT JOIN.

Verify that child columns become NULL because the child row is absent.

This must not be confused with a child row whose actual field contains NULL.

---

## 75. Integration Test — Upsert

Test both contracts:

    NULL means clear
    NULL means preserve existing value

Verify the SQL matches the selected contract.

This should be mandatory for CDC and update pipelines.

---

## 76. Integration Test — Aggregation

Use:

    10
    20
    NULL

Verify:

    SUM
    AVG
    COUNT
    COUNT(*)

Document the business interpretation.

---

## 77. Integration Test — Deduplication

Create records with:

    updated_at = valid timestamp
    updated_at = NULL

Verify:

- explicit NULL ordering;
- deterministic tie-breaking;
- stable selected record.

---

## 78. Intentional Failure Drill — Truthiness

Introduce:

```
if not value:
    value = None
```

Run data containing:

    0
    false
    ""

Verify the corruption.

Then remove the bug and add regression tests.

---

## 79. Intentional Failure Drill — Required Field

Remove:

    customer_id

Expected:

    REQUIRED_FIELD_MISSING

The record must not enter the valid staging path.

---

## 80. Intentional Failure Drill — Dangerous Default

Configure:

    missing_amount -> 0

for a field where zero is not equivalent to missing.

Observe the false business value.

Remove the default and enforce the correct policy.

---

## 81. Intentional Failure Drill — NULL Upsert

Existing:

    email = "a@example.com"

Incoming:

    email = NULL

Run the update.

Verify whether the contract says:

    clear

or:

    preserve

Then add a regression test.

---

## 82. Intentional Failure Drill — NULL Join Key

Create a child record with:

    customer_id = NULL

Run an equality join.

Verify that NULL does not match another NULL key.

Then route the record according to the data contract.

---

## 83. Observability

At minimum measure:

    total_records
    missing_fields
    null_values
    empty_values
    null_markers
    defaults_applied
    required_null_failures
    quarantined_records

Track by:

    pipeline
    dataset
    source
    field
    schema_version

---

## 84. Structured Logging

Example:

```
logger.info(
    "null_policy_applied",
    extra={
        "dataset": dataset,
        "field": field_name,
        "null_state": null_state,
        "policy": policy_name,
        "record_id": record_id,
    },
)
```

Do not log sensitive source values just to explain NULL behavior.

---

## 85. Alerting

Useful alert conditions include:

    required field null rate > 0
    null rate above contract threshold
    sudden empty-string increase
    new null marker appears
    default application spikes
    null quarantine rate increases

Do not alert on every NULL.

Valid NULLs are normal in many datasets.

---

## 86. Baselines

A null rate requires context.

Example:

    historical = 12–16%
    current = 14%

may be normal.

But:

    historical = 12–16%
    current = 71%

requires investigation.

Use historical baselines together with explicit source and business thresholds.

---

## 87. Production Runbook

When NULL behavior changes unexpectedly:

### Step 1 — Identify the field

Check:

    source
    dataset
    field
    schema version

### Step 2 — Classify the state

Determine:

    missing
    NULL
    empty
    whitespace
    null marker
    present

### Step 3 — Compare the contract

Check:

    nullability
    defaults
    marker list
    schema version

### Step 4 — Trace lineage

Determine whether NULL originated:

    at source
    during normalization
    during conversion
    during join
    during aggregation
    during update

### Step 5 — Determine scope

Identify:

    first affected run
    affected files
    affected records
    affected downstream tables

### Step 6 — Isolate if necessary

Stop propagation when the NULL semantics are demonstrably wrong.

### Step 7 — Correct the rule

Update the appropriate:

    source mapping
    null policy
    SQL
    transformation

### Step 8 — Replay

Replay from durable raw evidence or the appropriate staging checkpoint.

### Step 9 — Reconcile

Compare:

    record counts
    null rates
    defaults
    quarantines
    downstream values

### Step 10 — Add regression coverage

Make the failure detectable in CI.

---

## 88. Recovery — Wrong Default

If NULL was incorrectly converted to a business value:

1. identify affected records;
2. identify the transformation version;
3. recover source evidence;
4. correct the default policy;
5. replay affected data;
6. repair downstream aggregates;
7. reconcile;
8. add a regression test.

Prefer replay over manual final-table edits.

---

## 89. Recovery — Wrong Null Marker

If:

    "N/A"

was incorrectly treated as NULL:

1. identify source and field;
2. determine intended meaning;
3. version the source mapping;
4. replay affected records;
5. compare null rates before and after;
6. reconcile downstream consumers.

---

## 90. Recovery — Missing vs NULL Collapsed

If missing and explicit NULL were accidentally merged:

1. check whether raw evidence preserves the original representation;
2. recover the original states;
3. correct classification;
4. replay affected records;
5. reconcile downstream updates.

If raw evidence does not preserve the distinction, do not invent historical values. Document the ambiguity and its impact.

---

## 91. Recovery — Join-Generated NULL

If a LEFT JOIN unexpectedly produces NULL:

1. determine whether the related row exists;
2. inspect join keys;
3. check key nullability;
4. check normalization;
5. check source completeness;
6. correct the join or mapping;
7. replay;
8. reconcile unmatched counts.

---

## 92. Common Mistakes

### Mistake 1 — Treating NULL as zero

**Fix:** Preserve unknown vs zero semantics.

### Mistake 2 — Treating NULL as false

**Fix:** Use explicit boolean semantics.

### Mistake 3 — Using truthiness for null detection

**Fix:** Check None and empty text separately.

### Mistake 4 — Comparing SQL NULL with equals

**Fix:** Use IS NULL and IS NOT NULL.

### Mistake 5 — Defaulting every NULL

**Fix:** Define defaults field by field.

### Mistake 6 — Collapsing missing and explicit NULL

**Fix:** Preserve the distinction when required.

### Mistake 7 — Misreading LEFT JOIN NULLs

**Fix:** Determine whether the related row was absent.

### Mistake 8 — Ignoring NULL ordering

**Fix:** Make ordering explicit for deterministic transformations.

### Mistake 9 — Overwriting values with NULL during upsert

**Fix:** Define clear-vs-preserve semantics.

### Mistake 10 — Using sentinel values everywhere

**Fix:** Use them only when the target model explicitly requires them.

### Mistake 11 — Logging sensitive raw values

**Fix:** Log state, reason, and lineage.

### Mistake 12 — Treating completeness as correctness

**Fix:** Validate non-NULL values too.

---

## 93. Debugging Questions

Ask:

1. Was the field missing or explicitly NULL?
2. Was an empty string converted to NULL?
3. Was a source null marker configured?
4. Did a default apply?
5. Did a join introduce the NULL?
6. Did conversion failure become NULL?
7. Did aggregation ignore NULLs?
8. Did an upsert overwrite a value?
9. What database NULL semantics apply?
10. Which schema version was active?
11. Which null policy version was active?
12. Is the field required, optional, or defaultable?
13. What does NULL mean in this domain?
14. Can raw evidence reconstruct the original state?
15. Can the affected records be replayed?

---

## 94. Production Implementation Sequence

### Step 1 — Inventory fields

Classify fields as:

    required
    optional
    defaultable
    derived

### Step 2 — Define null semantics

Specify behavior for:

    missing
    NULL
    empty
    whitespace
    source markers

### Step 3 — Preserve source state

Classify before destructive normalization.

### Step 4 — Implement normalization

Apply only source-approved mappings.

### Step 5 — Apply null policy

Choose:

    allow
    reject
    default
    quarantine
    derive

### Step 6 — Validate present values

Nullability does not replace type or domain validation.

### Step 7 — Integrate with staging

Use T01 lineage and quarantine mechanisms.

### Step 8 — Implement SQL semantics

Explicitly handle:

    IS NULL
    COALESCE
    NULLIF
    joins
    aggregates
    ordering

### Step 9 — Define update semantics

Especially for:

    CDC
    upserts
    PATCH-style events

### Step 10 — Add observability

Measure null states and changes.

### Step 11 — Test failure modes

Test missing, NULL, empty, false, zero, markers, joins, and updates.

### Step 12 — Drill recovery

Replay a corrected rule from durable source evidence.

---

## 95. Production Checklist

### Contract

- [ ] Every field has explicit nullability.
- [ ] Missing behavior is defined.
- [ ] NULL behavior is defined.
- [ ] Empty behavior is defined.
- [ ] Whitespace behavior is defined.
- [ ] Source null markers are defined.
- [ ] Defaults are explicitly justified.
- [ ] Contract version is retained.

### Transformation

- [ ] Missing and NULL can be distinguished where required.
- [ ] Empty strings are handled intentionally.
- [ ] Zero is not treated as NULL.
- [ ] False is not treated as NULL.
- [ ] Invalid values are not silently converted to NULL.
- [ ] Technical conversion remains separate from business transformation.

### SQL

- [ ] IS NULL is used correctly.
- [ ] COALESCE is used intentionally.
- [ ] NULLIF is used intentionally.
- [ ] NULL behavior in joins is understood.
- [ ] NULL behavior in aggregates is understood.
- [ ] NULL ordering is explicit where required.

### Incremental processing

- [ ] CDC NULL semantics are defined.
- [ ] Upsert NULL semantics are defined.
- [ ] Missing update fields are distinguished from explicit NULL where required.

### Observability

- [ ] Null rates are measured.
- [ ] Missing rates are measured.
- [ ] Empty values are measured.
- [ ] Defaults are measured.
- [ ] Quarantine is measured.
- [ ] Baselines and thresholds exist.

### Recovery

- [ ] Raw evidence is preserved.
- [ ] Replay is supported.
- [ ] Null policy changes are versioned.
- [ ] Reconciliation is available.
- [ ] Regression tests exist.

---

## 96. Production Tools You Should Know

### PostgreSQL

Use for:

- NULL-aware predicates;
- COALESCE and NULLIF;
- nullable columns;
- partial indexes;
- constraints;
- joins and aggregations.

Understand the actual behavior of the target database rather than assuming all SQL systems are identical.

### Python

Use for:

- None;
- explicit state classification;
- source-specific normalization;
- field-level null policies;
- structured validation.

The critical skill is avoiding truthiness-based corruption.

### Pydantic

Useful when nullability is part of an explicit typed data model.

It can help express:

- required fields;
- optional fields;
- typed values;
- structured validation errors.

It does not replace source-system semantics.

---

## 97. Package Structure

A practical implementation:

    etl/
      null_handling/
        classification.py
        normalization.py
        policies.py
        validation.py
        errors.py

      transformations/
        customers.py
        payments.py

      staging/
        loader.py
        quarantine.py

      tests/
        test_classification.py
        test_normalization.py
        test_required_fields.py
        test_defaults.py
        test_sql_nulls.py
        test_upserts.py
        test_joins.py

Keep these responsibilities separate:

    classification
    normalization
    policy
    validation
    transformation
    persistence

---

## 98. Design Principle — NULL Is Data

NULL is not merely a nuisance to remove.

It can communicate:

    unknown
    unavailable
    not applicable
    not yet occurred
    not provided

Treat NULL as part of the data model.

---

## 99. Design Principle — Missingness Can Be Meaningful

The absence of a field can contain information.

Examples:

    no cancellation_date
    no completion_date
    no consent_date

Do not fill missing values merely because the table looks cleaner.

---

## 100. Design Principle — Defaults Are Business Rules

Changing:

    NULL -> 0

can change business meaning.

Therefore defaults should be:

    documented
    reviewed
    tested
    versioned

---

## 101. Design Principle — Preserve Before Normalizing

Use:

    observe
       |
       v
    classify
       |
       v
    normalize
       |
       v
    validate
       |
       v
    transform

Once states are collapsed, the original evidence may be impossible to reconstruct.

---

## 102. Design Principle — Update Semantics Matter

For incremental data:

    missing field
    NULL field
    existing value

can represent three different operations.

This is especially important for:

- CDC;
- APIs;
- event streams;
- upserts.

Null handling must follow the update contract.

---

## 103. Design Principle — Do Not Hide Quality Problems

Bad pipeline:

    invalid value -> NULL
    NULL -> default
    default -> valid record

This can turn a source defect into apparently valid data.

Prefer:

    invalid value
         |
         v
    explicit failure
         |
         v
    quarantine / correction / replay

---

## 104. Definition of Done

You are done with T03 when you can independently implement null handling that:

1. distinguishes missing, NULL, empty, whitespace, and null markers;
2. defines field-level null contracts;
3. handles required and optional fields;
4. applies defaults only when justified;
5. avoids truthiness-based corruption;
6. understands SQL three-valued logic;
7. handles NULL correctly in joins;
8. handles NULL correctly in aggregations;
9. handles NULL ordering in deterministic transformations;
10. understands nullable foreign keys;
11. distinguishes missing update fields from explicit NULL;
12. handles CDC and upsert semantics;
13. classifies null-related failures;
14. quarantines invalid records when required;
15. measures null and missing rates;
16. preserves raw evidence and lineage;
17. supports replay after policy changes;
18. tests boundary and failure cases;
19. documents recovery procedures;
20. can explain the semantic meaning of every NULL introduced by the pipeline.

If you can implement these mechanisms independently, you understand production null handling.

---

## 105. What You Learned

The central model is:

    source record
         |
         v
    classify state
         |
         +---------------------+
         |                     |
         v                     v
      present              missing/null
         |                     |
         v                     v
     normalize             null policy
         |                     |
         +----------+----------+
                    |
                    v
                validate
                    |
             +------+------+
             |             |
             v             v
           valid       invalid
             |             |
             v             v
          staging     quarantine
             |
             v
        transformation

Key rules:

1. NULL is not zero.
2. NULL is not false.
3. NULL is not an empty string.
4. Missing is not always the same as explicit NULL.
5. Null markers are source-specific.
6. Defaults are business rules.
7. SQL uses three-valued logic.
8. NULL does not equal NULL in normal SQL equality.
9. LEFT JOIN NULL can mean a missing related row.
10. NULL behavior must be explicit in upserts and CDC.
11. Preserve source state before normalization.
12. Measure null rates and their underlying states.
13. Never turn invalid data into NULL merely to make a pipeline succeed.
14. Preserve evidence so incorrect null policies can be replayed safely.
15. A complete null strategy explains every NULL introduced by the pipeline.

Next:

    T04 — Default Values
