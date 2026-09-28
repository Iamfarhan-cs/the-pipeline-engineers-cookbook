# T04 — Default Values

## 1. Problem Recognition

A pipeline often receives a missing value where the target system expects a value.

Examples:

    retry_count = NULL
    country = NULL
    status = NULL
    currency = NULL

The tempting solution is:

    NULL -> default

But a default is not a cleanup operation.

A default is a business or technical rule that says:

> When the source does not provide a usable value, this specific replacement is semantically correct.

If that statement is not true, the default can create false data.

---

## 2. What You Are Building

A safe default-value boundary:

    source value
         |
         v
    classify state
         |
         v
    determine field policy
         |
         +----------------------+
         |                      |
         v                      v
    value supplied         value absent
         |                      |
         v                      v
    validate value         check default rule
                                |
                         +------+------+
                         |             |
                         v             v
                      default       no default
                         |             |
                         +------+------+
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
                      staging    quarantine/fail

The goal is not to eliminate NULLs.

The goal is to apply only intentional, documented defaults.

---

## 3. Learning Objectives

By the end of this recipe you should be able to:

1. Explain why defaults are semantic rules.
2. Distinguish missing, NULL, empty, and invalid values.
3. Define field-level default contracts.
4. Distinguish business defaults from technical defaults.
5. Apply defaults only at the correct pipeline boundary.
6. Avoid overwriting legitimate values.
7. Handle defaults in Python and SQL.
8. Handle defaults during inserts and upserts.
9. Version and observe default application.
10. Test normal and failure paths.
11. Recover from an incorrect default.
12. Implement default handling independently.

---

## 4. Default vs Fallback

A default is not the same as an arbitrary fallback.

Example:

    retry_count missing -> 0

can be valid if the domain defines a missing retry count as zero attempts.

But:

    transaction_amount missing -> 0

may be invalid if NULL means the amount was never received.

The difference is semantic.

---

## 5. Types of Defaults

### 5.1 Business Default

A domain-defined value.

Example:

    retry_count -> 0

because zero retries is the defined initial state.

### 5.2 Technical Default

A value required to make a technical component operate.

Example:

    batch_size -> 1000

This is configuration rather than business data.

### 5.3 Presentation Default

A display fallback such as:

    country -> "Unknown"

This should normally happen at the presentation boundary, not in the canonical data model.

### 5.4 Schema Default

A database-level default:

    DEFAULT 0

This is useful for enforcing storage behavior, but it does not replace source validation or business reasoning.

---

## 6. Default Contract

For each defaultable field define:

    field
    allowed states
    default value
    trigger state
    validation
    owner
    version

Example:

    field: retry_count
    trigger: missing or NULL
    default: 0
    validation: integer >= 0
    owner: payment-domain
    version: 1

This makes the rule reviewable.

---

## 7. The Most Important Question

Before writing:

    fillna(default)

ask:

> What does the missing value mean?

Possible meanings:

    unknown
    not applicable
    not collected
    not yet available
    producer failure
    legitimate initial state

Only some of these justify a default.

---

## 8. Default Contract Example

A configuration object:

```python
DEFAULT_POLICIES = {
    "retry_count": {
        "default": 0,
        "apply_to": {"MISSING", "NULL"},
        "validate": lambda value: (
            isinstance(value, int) and value >= 0
        ),
    },
    "status": {
        "default": "PENDING",
        "apply_to": {"MISSING", "NULL"},
        "validate": lambda value: value in {
            "PENDING",
            "PROCESSING",
            "COMPLETED",
            "FAILED",
        },
    },
}
```

The policy should be explicit rather than hidden inside a generic utility.

---

## 9. Missing vs NULL

These may have different meanings.

JSON:

```json
{}
```

means the field is absent.

While:

```json
{"retry_count": null}
```

contains an explicit NULL.

A policy may say:

    missing -> default
    explicit NULL -> preserve

or:

    missing -> default
    explicit NULL -> default

Both are possible.

The source contract decides.

---

## 10. Empty Values

An empty string is not automatically missing.

For example:

    country = ""

could mean:

    invalid source value

rather than:

    use default country

Do not write:

```python
value = value or default
```

because it also collapses:

    0
    False
    ""
    None

into the same result.

Use explicit state checks.

---

## 11. Safe Python Defaulting

Prefer:

```python
def apply_default(value, default):
    if value is None:
        return default

    return value
```

For dictionaries where missing and NULL have the same contract:

```python
value = record.get("retry_count")

if value is None:
    value = 0
```

If missing and explicit NULL differ, test membership first:

```python
if "retry_count" not in record:
    value = 0
elif record["retry_count"] is None:
    value = None
else:
    value = record["retry_count"]
```

---

## 12. Validate After Defaulting

Defaulting does not prove that the final value is valid.

Example:

```python
value = None
value = 0
```

Then validate:

```python
if not isinstance(value, int) or value < 0:
    raise ValueError("INVALID_RETRY_COUNT")
```

The safe sequence is:

    classify
      ↓
    default
      ↓
    type validation
      ↓
    domain validation

---

## 13. Do Not Default Invalid Values

These are different:

    NULL
    "abc"

Suppose:

    retry_count NULL -> 0

That does not imply:

    retry_count "abc" -> 0

The second value was supplied but is invalid.

Silently defaulting it hides a source defect.

Prefer:

    invalid supplied value -> validation failure

---

## 14. Default Only the Trigger States

A field-level policy can explicitly define trigger states.

Example:

```python
DEFAULT_TRIGGERS = {"MISSING", "NULL"}
```

Then:

```python
if state in DEFAULT_TRIGGERS:
    value = default
```

An empty string should only trigger the default if the contract includes:

    EMPTY

---

## 15. Default Application Function

A reusable implementation:

```python
def apply_field_default(
    *,
    state: str,
    value,
    default,
    trigger_states: set[str],
):
    if state in trigger_states:
        return default, True

    return value, False
```

Return whether a default was applied.

That boolean is useful for:

- metrics;
- audit;
- testing;
- debugging.

---

## 16. Structured Result

A richer result:

```python
from dataclasses import dataclass


@dataclass
class DefaultResult:
    value: object
    applied: bool
    reason: str | None = None
```

Implementation:

```python
def apply_default(
    value,
    state,
    default,
    trigger_states,
):
    if state in trigger_states:
        return DefaultResult(
            value=default,
            applied=True,
            reason="DEFAULT_APPLIED",
        )

    return DefaultResult(
        value=value,
        applied=False,
    )
```

This makes the transformation observable.

---

## 17. Default Reason Codes

Use stable reason codes such as:

    DEFAULT_MISSING
    DEFAULT_NULL
    DEFAULT_EMPTY
    DEFAULT_SOURCE_MARKER

Avoid relying on free-form messages.

Reason codes make metrics and audits queryable.

---

## 18. Defaults and Data Types

The default must match the target type.

Example:

    retry_count -> 0

not:

    retry_count -> "0"

unless the target contract explicitly uses text.

For a date:

    processed_at -> current timestamp

has very different semantics from:

    processed_at -> NULL

Do not choose a default based only on type compatibility.

---

## 19. Defaults and Time

Time defaults require particular caution.

Bad example:

    missing occurred_at -> now()

This changes business event history.

A technical ingestion timestamp can be appropriate:

    ingested_at -> current timestamp

because it describes when the pipeline observed the record.

Do not substitute ingestion time for business event time.

---

## 20. Defaults and Identifiers

Never invent identifiers merely to eliminate NULLs.

Bad:

    missing customer_id -> 0

or:

    missing customer_id -> "UNKNOWN"

Identifiers participate in joins and referential integrity.

A missing required identifier should normally result in:

    validation failure
    quarantine
    or explicit rejection

according to the contract.

---

## 21. Defaults and Foreign Keys

A missing foreign key should not normally be replaced with an arbitrary parent ID.

Bad:

    missing country_id -> 1

unless ID 1 is explicitly defined as a valid "unknown" or "not applicable" dimension member.

Even then, the model should document that semantic choice.

---

## 22. Sentinel Values

A sentinel is a special value representing a state.

Examples:

    country_id = 0
    status = "UNKNOWN"
    date = "9999-12-31"

Sentinels can be useful in dimensional models.

But they introduce a second representation of missingness.

Use them only when the target model explicitly requires them.

---

## 23. NULL vs Sentinel

Compare:

    country_id = NULL

with:

    country_id = 0

These are not automatically equivalent.

NULL can mean:

    value unavailable

while 0 can mean:

    explicitly assigned unknown member

The distinction should be documented.

---

## 24. Defaults in SQL

SQL can use:

```sql
COALESCE(retry_count, 0)
```

This is useful when NULL should become zero for a specific output.

But:

```sql
SELECT COALESCE(amount, 0)
FROM payments;
```

does not repair the source data.

It only changes the query result.

Use it intentionally.

---

## 25. INSERT Defaults

A database can define:

```sql
CREATE TABLE job (
    job_id BIGINT PRIMARY KEY,
    retry_count INTEGER NOT NULL DEFAULT 0
);
```

Then:

```sql
INSERT INTO job(job_id)
VALUES (1001);
```

produces:

    retry_count = 0

This is a storage-level default.

It does not tell you whether the source omitted the field because it legitimately meant zero.

---

## 26. Explicit NULL and Database Defaults

A database default normally applies when the column is omitted.

Example:

```sql
INSERT INTO job(job_id)
VALUES (1001);
```

can use the default.

But:

```sql
INSERT INTO job(job_id, retry_count)
VALUES (1001, NULL);
```

explicitly supplies NULL.

With a NOT NULL constraint, this can fail rather than use the default.

This distinction matters in ETL loaders.

---

## 27. Upserts

Defaults and upserts require explicit semantics.

Suppose:

    existing retry_count = 3
    incoming retry_count = NULL

Possible contracts:

    NULL -> clear
    NULL -> preserve
    NULL -> default to 0

Do not assume the correct behavior.

Write it into the upsert contract.

---

## 28. Preserve Existing vs Default

If NULL means preserve:

```sql
UPDATE customer
SET retry_count = COALESCE(EXCLUDED.retry_count, customer.retry_count);
```

If NULL means default:

```sql
UPDATE customer
SET retry_count = COALESCE(EXCLUDED.retry_count, 0);
```

If NULL means clear:

```sql
UPDATE customer
SET retry_count = EXCLUDED.retry_count;
```

These are three different data contracts.

---

## 29. Defaults and CDC

For change events, distinguish:

    field absent
    field explicitly NULL
    field contains value

Example:

```json
{}
```

can mean:

    do not update retry_count

while:

```json
{"retry_count": null}
```

can mean:

    clear retry_count

If the pipeline applies a default to both, it may corrupt updates.

---

## 30. Default Timing

Defaulting can happen at:

    source ingestion
    staging
    transformation
    load
    presentation

Choose the boundary deliberately.

A useful rule:

> Apply a default as late as possible, but before the layer that requires the defaulted representation.

For example:

    canonical warehouse data -> preserve NULL
    dashboard display -> "Unknown"

may be better than storing "Unknown" in the canonical table.

---

## 31. Default Too Early

Suppose:

    source country = NULL

and the pipeline immediately writes:

    country = "UNKNOWN"

Later, another transformation cannot determine whether:

    source was NULL

or:

    source explicitly contained "UNKNOWN"

Early defaulting destroys information.

---

## 32. Default Too Late

If a target database requires:

    retry_count NOT NULL

and the loader sends NULL, the load may fail.

The default must therefore be applied before the boundary that requires it.

---

## 33. Canonical Data vs Presentation

Canonical data:

    country = NULL

Presentation:

    country_display = "Unknown"

This separation often preserves better analytical semantics.

Do not contaminate the canonical dataset merely to make a dashboard readable.

---

## 34. Default Configuration

Store defaults in a versioned configuration where appropriate.

Example:

```yaml
defaults:
  retry_count:
    value: 0
    triggers:
      - MISSING
      - NULL

  status:
    value: PENDING
    triggers:
      - MISSING
      - NULL
```

Version the configuration with the pipeline.

A historical run must be reproducible.

---

## 35. Versioned Default Policies

A production record may need:

    pipeline_version
    default_policy_version

This helps answer:

> Why did this record receive this value?

Do not over-engineer metadata, but preserve the information needed for important transformations.

---

## 36. Default Ownership

Each important default should have an owner.

Example:

    retry_count -> payment-domain
    country_display -> analytics
    batch_size -> platform

This prevents unrelated teams from changing business semantics casually.

---

## 37. Default Metrics

Track:

    defaults_applied_total
    defaults_by_field
    defaults_by_source
    defaults_by_reason
    default_rate

Example:

    1,000,000 records
    20,000 retry_count defaults

    default rate = 2%

A sudden increase can indicate a producer regression.

---

## 38. Default Rate

For field f:

    default_rate(f) =
        records_defaulted(f) / records_processed

Keep numerator and denominator.

Example:

    processed = 500,000
    defaulted = 25,000

    rate = 25,000 / 500,000
         = 0.05
         = 5%

The rate alone is less useful without the volume.

---

## 39. Baselines

A default rate needs context.

Historical:

    1%–2%

Current:

    2.1%

may be normal.

Historical:

    1%–2%

Current:

    38%

requires investigation.

Use both:

    explicit threshold

and:

    historical baseline

where appropriate.

---

## 40. Alerting

Useful alerts:

    required field unexpectedly defaulted
    default rate above threshold
    new field begins receiving defaults
    default count spikes
    default policy version changes
    defaulted records increase after a source deployment

Do not alert on every individual default.

Valid defaults are expected behavior.

---

## 41. Default Audit Event

For important transformations, emit structured metadata:

```python
{
    "field": "retry_count",
    "reason": "DEFAULT_NULL",
    "policy_version": "v1",
    "source": "payments",
}
```

Avoid logging sensitive original values unnecessarily.

---

## 42. Testing Strategy

For every defaultable field test:

    missing
    explicit NULL
    empty
    whitespace
    source null marker
    valid value
    invalid value
    zero
    false

Then verify which states trigger the default.

---

## 43. Unit Test — Missing

Input:

```python
record = {}
```

Policy:

    missing -> 0

Expected:

```python
0
```

and:

    applied = True

---

## 44. Unit Test — Explicit NULL

Input:

```python
record = {"retry_count": None}
```

If NULL triggers the default:

    result = 0
    applied = True

If NULL means preserve:

    result = None
    applied = False

The test must encode the selected contract.

---

## 45. Unit Test — Valid Value

Input:

```python
{"retry_count": 3}
```

Expected:

    3

Never replace a supplied valid value with the default.

---

## 46. Unit Test — Zero

Input:

```python
{"retry_count": 0}
```

Expected:

    0

and:

    applied = False

This catches:

```python
value = value or default
```

---

## 47. Unit Test — False

If a boolean field has:

    default = True

then:

```python
{"enabled": False}
```

must remain:

    False

A false value is not missing.

---

## 48. Unit Test — Invalid Value

Input:

```python
{"retry_count": "abc"}
```

Expected:

    validation failure

Not:

    0

---

## 49. Unit Test — Empty String

If the policy says empty text does not trigger the default:

```python
{"status": ""}
```

must fail validation.

If the contract explicitly says empty means missing:

    "" -> default

Test that mapping explicitly.

---

## 50. Integration Test — Database Default

Create:

```sql
CREATE TEMP TABLE test_job (
    retry_count INTEGER NOT NULL DEFAULT 0
);
```

Insert without the column.

Verify:

    retry_count = 0

Then insert explicit NULL and verify the database behavior.

This proves the difference between omitted and explicit NULL.

---

## 51. Integration Test — Upsert

Test:

    existing = 3
    incoming = NULL

under each supported contract:

    clear
    preserve
    default

Only one should be selected by the application contract.

---

## 52. Integration Test — Replay

Run the same input twice with the same default policy.

Verify:

    deterministic output
    stable default count
    no duplicate side effects

Defaulting itself should be deterministic.

---

## 53. Intentional Failure Drill — Truthiness

Introduce:

```python
value = value or default
```

Use:

    0
    False
    ""

Verify that valid falsey values are corrupted.

Replace the implementation with explicit state handling.

Add regression tests.

---

## 54. Intentional Failure Drill — Dangerous Default

Configure:

    transaction_amount NULL -> 0

for a dataset where NULL means unknown.

Run the pipeline.

Observe the incorrect business metric.

Then:

1. remove the default;
2. restore NULL semantics;
3. replay affected records;
4. reconcile downstream metrics.

---

## 55. Intentional Failure Drill — Wrong Time Default

Configure:

    occurred_at NULL -> current_timestamp

Observe how historical event time changes.

Replace it with a separate:

    ingested_at

technical timestamp.

Never fabricate business event time.

---

## 56. Intentional Failure Drill — Identifier Default

Configure:

    customer_id NULL -> 0

Observe broken joins or referential integrity.

Remove the default and quarantine the invalid record.

---

## 57. Observability

Track:

    records_processed
    defaults_applied
    records_with_defaults
    defaults_by_field
    defaults_by_source
    defaults_by_reason
    default_policy_version
    validation_failures_after_default

Useful dimensions:

    pipeline
    dataset
    source
    field
    run_id
    schema_version

---

## 58. Production Runbook

When default usage unexpectedly increases:

### Step 1 — Identify the field

Check:

    source
    dataset
    field
    policy version

### Step 2 — Identify the trigger

Determine whether the values were:

    missing
    NULL
    empty
    source marker

### Step 3 — Compare baseline

Check:

    current default rate
    historical default rate
    configured threshold

### Step 4 — Inspect source changes

Check:

    producer deployments
    schema changes
    API changes
    file-format changes

### Step 5 — Verify the policy

Confirm:

    default value
    trigger states
    validation rules

### Step 6 — Determine impact

Identify:

    affected runs
    affected records
    downstream tables
    metrics

### Step 7 — Stop propagation if necessary

If the default creates false business data, stop or quarantine the affected path.

### Step 8 — Correct the policy

Change the appropriate mapping or validation rule.

### Step 9 — Replay

Replay from durable source evidence.

### Step 10 — Reconcile

Compare:

    row counts
    default counts
    default rates
    downstream aggregates

### Step 11 — Add regression coverage

Ensure the source failure cannot silently reintroduce the same problem.

---

## 59. Recovery — Incorrect Default

If a bad default has already been loaded:

1. identify the default policy version;
2. identify affected runs;
3. identify affected records;
4. recover original source evidence;
5. determine intended NULL/default semantics;
6. correct the transformation;
7. replay;
8. repair downstream derived data;
9. reconcile;
10. add regression tests.

Prefer replay from durable evidence over manual edits.

---

## 60. Recovery — Default Applied Too Early

If a canonical NULL was replaced with a sentinel:

1. determine whether raw evidence still preserves NULL;
2. restore the canonical representation;
3. move the presentation default to the presentation layer;
4. replay downstream transformations;
5. verify analytics semantics.

---

## 61. Recovery — Wrong Default Policy Version

If different runs used different policies:

1. identify policy version per run;
2. freeze the incorrect version;
3. determine affected time range;
4. replay using the corrected version;
5. compare outputs;
6. retain an audit trail of the correction.

Historical reproducibility matters.

---

## 62. Common Mistakes

### Mistake 1 — Defaulting every NULL

**Fix:** Define defaults field by field.

### Mistake 2 — Using truthiness

**Fix:** Explicitly distinguish None, zero, false, and empty text.

### Mistake 3 — Defaulting invalid values

**Fix:** Validate supplied values instead of hiding them.

### Mistake 4 — Inventing identifiers

**Fix:** Reject or quarantine missing required identifiers.

### Mistake 5 — Replacing business timestamps with now()

**Fix:** Keep technical ingestion time separate.

### Mistake 6 — Applying defaults too early

**Fix:** Preserve canonical semantics until the boundary that needs the default.

### Mistake 7 — Using presentation defaults in storage

**Fix:** Keep display fallbacks at the presentation layer where possible.

### Mistake 8 — Ignoring CDC semantics

**Fix:** Distinguish missing update fields from explicit NULL.

### Mistake 9 — Hiding default spikes

**Fix:** Measure default counts and rates.

### Mistake 10 — Changing defaults without versioning

**Fix:** Version important default policies.

### Mistake 11 — Treating sentinel values as universal NULL replacements

**Fix:** Use sentinels only when the target model explicitly defines them.

---

## 63. Production Tools You Should Know

### PostgreSQL

Understand:

- column DEFAULT clauses;
- NOT NULL constraints;
- COALESCE;
- NULL semantics;
- upsert behavior;
- partial indexes and constraints.

### Python

Understand:

- None;
- explicit state checks;
- policy-driven defaults;
- dataclasses for structured transformation results;
- deterministic transformations.

### Pydantic

Useful for:

- typed defaults;
- required vs optional fields;
- validation after defaulting;
- structured validation errors.

Pydantic should express the application contract; it should not silently invent business defaults.

---

## 64. Package Structure

A practical implementation:

    etl/
      defaults/
        policies.py
        application.py
        validation.py
        metrics.py

      transformations/
        customers.py
        payments.py

      staging/
        loader.py
        quarantine.py

      tests/
        test_default_policies.py
        test_default_application.py
        test_default_validation.py
        test_upserts.py
        test_database_defaults.py
        test_replay.py

Keep these responsibilities separate:

    state classification
    default policy
    default application
    validation
    persistence
    observability

---

## 65. Design Principle — Defaults Are Semantics

A default changes the meaning of data.

Therefore:

    default != cleanup

A default should be treated like a business rule.

---

## 66. Design Principle — Preserve Information

If NULL is meaningful, keep NULL in the canonical model.

Use a default only where the contract requires one.

---

## 67. Design Principle — Defaults Must Be Deterministic

Given:

    same input
    same policy version

the pipeline should produce:

    same defaulted result

Avoid defaults based on:

    random values
    current time
    external mutable state

unless the contract explicitly requires that behavior.

---

## 68. Design Principle — Defaults Must Be Observable

You should be able to answer:

> How many records received a default, on which field, and why?

If you cannot answer that, the transformation is difficult to operate safely.

---

## 69. Design Principle — Defaults Should Be Reversible

A good pipeline retains enough evidence to determine:

    original state
    applied policy
    resulting value

This makes replay and incident recovery possible.

---

## 70. Definition of Done

You are done with T04 when you can independently implement default handling that:

1. distinguishes missing, NULL, empty, and invalid values;
2. defines field-level default contracts;
3. distinguishes business and technical defaults;
4. applies defaults only to explicit trigger states;
5. preserves valid zero values;
6. preserves valid false values;
7. does not default malformed supplied values;
8. validates the result after defaulting;
9. understands SQL DEFAULT and COALESCE semantics;
10. understands omitted vs explicit NULL during inserts;
11. handles default behavior correctly in upserts;
12. understands CDC update semantics;
13. keeps presentation defaults separate from canonical data where appropriate;
14. tracks default application metrics;
15. versions important default policies;
16. tests normal and failure paths;
17. can intentionally demonstrate default corruption;
18. can recover from an incorrect default;
19. can replay affected data safely;
20. can explain why every default exists.

If you can implement these mechanisms independently, you understand production default-value handling.

---

## 71. What You Learned

The core model is:

    source state
         |
         v
    classify
         |
         v
    field policy
         |
         +--------------------+
         |                    |
         v                    v
    supplied value       trigger state
         |                    |
         v                    v
      validate             default
         |                    |
         +---------+----------+
                   |
                   v
                validate
                   |
            +------+------+
            |             |
            v             v
          valid        invalid
            |             |
            v             v
         staging      quarantine

Key rules:

1. A default is a semantic rule.
2. NULL is not automatically a default trigger.
3. Missing and explicit NULL may have different meanings.
4. Empty text is not automatically missing.
5. Zero and false are valid values.
6. Invalid supplied values should not be hidden by defaults.
7. Database defaults and application defaults solve different problems.
8. COALESCE changes query output; it does not repair source data.
9. Default timing affects information preservation.
10. Canonical data and presentation defaults should often remain separate.
11. CDC and upsert semantics must define what NULL means.
12. Default application must be observable.
13. Important policies should be versioned.
14. Durable source evidence makes incorrect defaults recoverable.
15. The safest default is the one whose semantic meaning you can explain and test.

Next:

    T05 — String Normalization
