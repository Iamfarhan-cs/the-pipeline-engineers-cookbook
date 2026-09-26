# Recipe 19 — Validate Incoming Data

Chapter 18 gave us a staging boundary:

```text
API
 ↓
Raw
 ↓
Staging
 ↓
Target
```

But there is still an important question:

> What exactly should be accepted into the pipeline?

A pipeline should not assume that every record received from a source is usable.

Sources can return incomplete records, unexpected types, invalid values, malformed responses, or data that violates business rules.

Validation is the step that checks incoming data before the pipeline trusts it.

This recipe adds validation to the learning pipeline from the previous chapters.

---

## Production Tools You Should Know

These tools solve or provide production implementations of concepts covered in this recipe. Learn what the tool provides, but understand the underlying problem first.

| Tool | What to know |
|---|---|
| **Pydantic** | Programmatic Python data validation and typed boundaries. |
| **Great Expectations** | Data validation and expectation-based quality checks. |
| **Soda** | Data quality checks and monitoring. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the engineering mechanism.

---

## Recipe Goal

By the end of this recipe:

```text
External source
      ↓
Receive record
      ↓
Validate
   ↙     ↘
Valid     Invalid
  ↓          ↓
Raw       Reject / Quarantine
  ↓
Staging
  ↓
Processing
  ↓
Target
```

We will learn how to:

- validate required fields
- validate data types
- validate values
- validate text fields
- separate validation from transformation
- return useful validation errors
- decide what happens to invalid records
- avoid blindly trusting source data
- test valid and invalid inputs
- understand validation's relationship with idempotency and deduplication

---

## 1. What Is Data Validation?

Data validation means checking whether incoming data satisfies the rules required by the next stage of the pipeline.

Suppose the source sends:

```json
{
  "userId": 7,
  "id": 42,
  "title": "hello",
  "body": "world"
}
```

We may need to verify that:

- userId exists
- id exists
- title exists
- body exists
- userId is an integer
- id is an integer
- title is a string
- body is a string

If a record does not satisfy the required rules, the pipeline should not treat it as valid simply because the source accepted it.

---

## 2. Why Validation Matters

Without validation:

```text
Bad source data
      ↓
Raw
      ↓
Staging
      ↓
Processing fails
      ↓
Debugging becomes harder
```

With an early validation boundary:

```text
Bad source data
      ↓
Validation
      ↓
Rejected or quarantined
```

Early validation does not mean every problem must stop the entire pipeline.

It means the pipeline makes an intentional decision instead of accidentally accepting bad data.

---

## 3. Validation vs Transformation

Validation asks:

> Is this record acceptable?

Transformation asks:

> How should this acceptable record be represented for the next stage?

For example:

```text
title = " Hello World "
```

Validation might check that the title exists and is a string.

Transformation might then produce:

```text
"Hello World"
```

Keep important validation rules separate from transformation code.

Clear boundaries make failures easier to understand.

---

## 4. Validation vs Data Quality

Validation is one part of data quality.

Data quality is broader and can include:

- completeness
- uniqueness
- validity
- consistency
- referential integrity
- freshness
- anomaly detection

Validation might ask:

> Is userId an integer?

A later quality check might ask:

> Does this userId actually exist?

Both are useful, but they answer different questions.

Chapter 7 introduced the broader data-quality concepts.

This recipe focuses on incoming-record validation.

---

## 5. Types of Validation

Useful validation categories include:

### Structural validation

Does the record have the expected structure?

### Type validation

Are fields the expected types?

### Value validation

Are values within allowed boundaries?

### Format validation

Does a value follow the required format?

### Business-rule validation

Does the record make sense according to application rules?

### Referential validation

Does a referenced entity exist?

Not every pipeline needs all of these.

Use the checks required by the actual data contract.

---

## 6. Structural Validation

Start with the simplest question:

> Is the input actually a record?

Expected structure:

```text
record
├── userId
├── id
├── title
└── body
```

This is invalid:

```json
{
  "id": 42,
  "title": "hello"
}
```

Required fields are missing.

These are also invalid for this contract:

```json
null
```

and:

```json
[1, 2, 3]
```

The validator should reject values that are not records.

---

## 7. Required Fields

Start with required fields.

**GENERIC EXAMPLE:**

```python
REQUIRED_FIELDS = {
    "userId",
    "id",
    "title",
    "body",
}


def validate_required_fields(record: dict) -> None:
    missing = REQUIRED_FIELDS - record.keys()

    if missing:
        raise ValueError(
            f"Missing required fields: {sorted(missing)}"
        )
```

Now the failure is explicit:

```text
Missing required fields: ['body']
```

This is much more useful than waiting for PostgreSQL to report a later NOT NULL violation.

---

## 8. Type Validation

A field can exist but still contain the wrong type.

For example:

```json
{
  "userId": "seven",
  "id": 42,
  "title": "hello",
  "body": "world"
}
```

userId exists, but it is not an integer.

Add type checks:

```python
def validate_types(record: dict) -> None:
    if not isinstance(record["userId"], int):
        raise ValueError("userId must be an integer")

    if not isinstance(record["id"], int):
        raise ValueError("id must be an integer")

    if not isinstance(record["title"], str):
        raise ValueError("title must be a string")

    if not isinstance(record["body"], str):
        raise ValueError("body must be a string")
```

The invalid record can now be rejected before database insertion.

---

## 9. Python Type Details

Python has an important detail:

```python
isinstance(True, int)
```

returns True because bool is a subclass of int.

If a field must be an actual integer, use a stricter check:

```python
def is_integer(value) -> bool:
    return isinstance(value, int) and not isinstance(value, bool)
```

This is a small example of why validation rules should be explicit.

Language-level type behavior does not always match a business requirement.

---

## 10. Value Validation

Correct types are not enough.

Consider:

```json
{
  "userId": -5,
  "id": 42,
  "title": "hello",
  "body": "world"
}
```

The type is correct.

The value may still be invalid.

A rule could be:

```python
def validate_values(record: dict) -> None:
    if record["userId"] <= 0:
        raise ValueError("userId must be greater than zero")

    if record["id"] <= 0:
        raise ValueError("id must be greater than zero")
```

Value rules should come from an actual requirement.

Do not invent arbitrary limits just because they seem reasonable.

---

## 11. String Validation

Strings can also be invalid.

Examples include:

- empty strings
- whitespace-only values
- values that exceed a defined maximum
- invalid formats
- unexpected content

A simple check is:

```python
def validate_text_fields(record: dict) -> None:
    if not record["title"].strip():
        raise ValueError("title cannot be empty")

    if not record["body"].strip():
        raise ValueError("body cannot be empty")
```

The exact rule should come from the contract or business requirement.

---

## 12. Format Validation

Some values have a required format.

Examples include:

- email addresses
- UUIDs
- dates
- timestamps
- currency codes
- country codes
- URLs

If a source contract requires an ISO timestamp, validation should check the agreed timestamp format.

Do not enforce a format only because another system happens to use it.

The format must be part of the pipeline requirement.

---

## 13. Business Validation

A structurally valid record can still violate a business rule.

For example:

```json
{
  "quantity": 5,
  "unit_price": 10
}
```

Both values may have valid types.

A business rule might require quantity to be greater than zero.

Another rule might prohibit a particular combination of status and amount.

Business rules should be named clearly.

Do not hide important business decisions inside generic parsing code.

---

## 14. Referential Validation

Some fields refer to another entity.

For example:

```text
order.user_id
      ↓
users.id
```

The value can be an integer and still be invalid if the referenced user does not exist.

A database lookup can enforce this:

```python
def validate_user_exists(connection, user_id: int) -> None:
    row = connection.execute(
        "SELECT 1 FROM users WHERE id = %s",
        (user_id,),
    ).fetchone()

    if row is None:
        raise ValueError(f"Unknown user_id: {user_id}")
```

This is more expensive than a simple type check.

Use database lookups where the requirement justifies the additional work.

---

## 15. Validation Order

A useful sequence is:

```text
1. Is the input an object?
          ↓
2. Are required fields present?
          ↓
3. Are field types correct?
          ↓
4. Are values valid?
          ↓
5. Are formats valid?
          ↓
6. Do business rules pass?
          ↓
7. Do references exist?
```

Not every pipeline needs every step.

The sequence mainly prevents confusing errors.

There is little value in checking whether a user exists if userId is not even a valid integer.

---

## 16. Build One Validation Entry Point

For the learning project, create one clear validation function.

**GENERIC EXAMPLE:**

```python
def validate_record(record: object) -> dict:
    if not isinstance(record, dict):
        raise ValueError("record must be an object")

    validate_required_fields(record)
    validate_types(record)
    validate_values(record)
    validate_text_fields(record)

    return record
```

The function does not transform the record.

It checks the record and returns it when the checks pass.

That keeps validation easy to test.

---

## 17. Add Validation to Ingestion

The ingestion flow now becomes:

```text
Source
  ↓
Receive record
  ↓
Validate
  ↓
Store raw
  ↓
Store staging
```

A generic implementation is:

```python
def ingest_records(records: list[dict]) -> int:
    staged = 0

    with get_connection() as connection:
        for raw_record in records:
            record = validate_record(raw_record)
            raw_id = store_raw_record(connection, record)
            store_staging_record(connection, raw_id, record)
            staged += 1

        connection.commit()

    return staged
```

The invalid record does not enter the normal raw/staging path in this version.

This is a deliberate learning-project choice.

A production system may instead preserve the raw input first and validate it afterward.

---

## 18. Validate Before Raw or Raw Before Validation?

There is no universal answer.

### Pattern A — Validate first

```text
Source
 ↓
Validate
 ↓
Raw
 ↓
Staging
```

Benefit:

- invalid data does not enter the normal raw path

Trade-off:

- the original invalid source record may not be preserved

### Pattern B — Store raw first

```text
Source
 ↓
Raw
 ↓
Validate
 ↓
Staging
```

Benefit:

- every received record can potentially be preserved

Trade-off:

- raw storage can contain invalid data

Raw-first can be useful when replay, auditability, or source preservation is important.

For this small learning pipeline, validation before the normal raw/staging path keeps the flow simple.

Do not treat one pattern as universally correct.

---

## 19. Validation Errors Should Be Useful

Bad error:

```text
invalid data
```

Better:

```text
userId must be an integer
```

A larger system might use structured information:

```text
field=userId
error=expected_integer
received_type=string
```

A useful validation error should help answer:

- Which record failed?
- Which field failed?
- Which rule failed?
- Can the record be corrected?
- Should it be retried?

Be careful not to place sensitive values into logs or error messages.

---

## 20. Use a Validation Error Type

A custom exception makes failures easier to classify.

**GENERIC EXAMPLE:**

```python
class ValidationError(Exception):
    pass
```

Then:

```python
def validate_types(record: dict) -> None:
    if not isinstance(record["userId"], int):
        raise ValidationError("userId must be an integer")
```

Now ingestion can distinguish validation failures from unrelated exceptions.

That distinction becomes important when retry behavior is added.

---

## 21. Validation Failures Usually Should Not Be Retried

Consider:

```text
userId = "seven"
```

Retrying the same input does not normally change anything.

This is usually a permanent failure.

Compare it with:

```text
database connection timeout
```

A retry may succeed because the failure can be temporary.

A useful mental model is:

```text
Validation failure
      ↓
Usually non-retryable

Temporary infrastructure failure
      ↓
Potentially retryable
```

Chapter 24 will build retry behavior in detail.

---

## 22. Record-Level vs Batch-Level Validation

Suppose one API response contains 1,000 records.

One invalid record does not automatically mean all 1,000 records should be rejected.

### Batch-level failure

```text
1,000 records
      ↓
1 invalid
      ↓
Reject whole batch
```

### Record-level handling

```text
1,000 records
      ↓
999 valid → continue
1 invalid → reject / quarantine
```

Record-level handling can be more resilient, but it requires more detailed error handling.

The correct choice depends on whether records are independent.

---

## 23. Validation and Transactions

Suppose ten records are processed inside one database transaction.

If record 10 fails after records 1–9 have already been inserted, the transaction can roll back.

That provides all-or-nothing behavior.

Another design may validate every record first and then insert only valid records.

Another may commit each record independently.

These choices affect recovery and partial processing.

Do not choose a transaction boundary without understanding its failure behavior.

---

## 24. Validation and the Staging Layer

Chapter 18 created the staging boundary.

Validation now gives that boundary a clearer contract.

One possible flow is:

```text
Source
 ↓
Validation
 ↓
Raw
 ↓
Staging
 ↓
Processing
 ↓
Target
```

Another architecture is:

```text
Source
 ↓
Raw
 ↓
Validation
 ↓
Staging
 ↓
Processing
```

The correct design depends on source-preservation requirements.

---

## 25. Validation and Idempotency

Validation does not solve duplicates.

Suppose the same valid record arrives twice:

```text
record 10
   ↓
valid

record 10
   ↓
valid
```

Both copies can pass validation.

Validation answers:

> Is this record acceptable?

Idempotency answers:

> What should happen if this operation runs again?

Chapter 20 handles idempotency.

---

## 26. Validation and Deduplication

Deduplication asks whether two records represent the same logical data.

A duplicate can be perfectly valid by itself.

For example:

```text
record A: id=10
record B: id=10
```

Both can pass every validation rule.

Deduplication requires a separate identity rule.

Chapter 21 covers this.

---

## 27. Validation and Quarantine

Some invalid records should not simply disappear.

A quarantine path can preserve them:

```text
             ┌→ Normal pipeline
Source → Validation
             └→ Quarantine
```

Quarantine is useful when:

- bad records need investigation
- records may be corrected later
- source teams need feedback
- replay after correction matters
- validation failures need operational tracking

Chapter 26 builds quarantine as a separate recipe.

---

## 28. Testing Valid Records

Start with a known-good record.

```python
def test_valid_record_passes_validation():
    record = {
        "userId": 7,
        "id": 42,
        "title": "hello",
        "body": "world",
    }

    assert validate_record(record) == record
```

This proves the validator does not reject valid input.

That matters because validation can become too strict.

---

## 29. Test Missing Fields

```python
import pytest


def test_missing_body_fails_validation():
    record = {
        "userId": 7,
        "id": 42,
        "title": "hello",
    }

    with pytest.raises(ValueError, match="body"):
        validate_record(record)
```

This verifies a specific validation rule.

---

## 30. Test Wrong Types

```python
def test_wrong_user_id_type_fails_validation():
    record = {
        "userId": "7",
        "id": 42,
        "title": "hello",
        "body": "world",
    }

    with pytest.raises(ValueError, match="userId"):
        validate_record(record)
```

---

## 31. Test Invalid Values

```python
def test_negative_user_id_fails_validation():
    record = {
        "userId": -1,
        "id": 42,
        "title": "hello",
        "body": "world",
    }

    with pytest.raises(ValueError, match="greater than zero"):
        validate_record(record)
```

---

## 32. Test Unexpected Input

The validator should also reject values that are not records.

```python
def test_list_is_not_a_valid_record():
    with pytest.raises(ValueError, match="object"):
        validate_record([1, 2, 3])
```

Also consider tests for:

- None
- strings
- integers
- empty dictionaries
- nested objects with invalid fields

The exact test set depends on the contract.

---

## 33. Test the Complete Ingestion Boundary

Unit tests are important, but also test the real boundary:

```text
Source record
     ↓
Validation
     ↓
Raw
     ↓
Staging
```

For a valid record, verify that the expected database rows exist.

For an invalid record, verify the intended failure behavior.

Do not test only the validation function and assume the ingestion code actually calls it.

---

## 34. What Should Happen to Invalid Data?

This is an architecture decision.

Common choices include:

1. Reject and stop the batch.
2. Reject the individual record and continue.
3. Store the record in quarantine.
4. Store raw data and mark validation as failed.
5. Log the failure and discard the record.

The correct choice depends on system requirements.

Do not silently discard invalid data unless that behavior is explicitly acceptable.

---

## 35. Validation Metrics

Once validation exists, measure it.

Useful metrics include:

- records received
- records validated
- records rejected
- records quarantined
- validation failures by rule
- validation failures by source
- validation failure rate

For example:

```text
Received:    10,000
Valid:        9,850
Rejected:       150
```

A sudden increase in rejected records can indicate a source change or a pipeline bug.

---

## 36. Validation Logging

Useful structured logging might contain:

```text
event=validation_failed
source=example_api
rule=required_field
field=body
record_id=42
```

Avoid logging the complete record when it may contain sensitive information.

Prefer safe identifiers and metadata.

---

## 37. Schema Validation

For larger systems, manually writing every type check may become difficult.

Schema-validation libraries can define the expected structure more formally.

They can support:

- required fields
- types
- nested objects
- enums
- constraints
- custom validators

The exact library is a project choice.

Do not add a framework simply because it exists.

For a small pipeline, straightforward Python validation may be easier to understand and maintain.

---

## 38. Validation Against a Data Contract

A data contract defines what a producer promises to send and what a consumer expects.

A simple contract might say:

```text
userId → integer, required
id     → integer, required
title  → string, required
body   → string, required
```

Validation turns those requirements into executable checks.

Instead of:

> The API normally sends this.

we can say:

> These fields and rules are part of the expected contract.

That is a stronger engineering boundary.

---

## 39. Schema Evolution

Suppose the source adds:

```json
"createdAt": "2026-09-26T10:00:00Z"
```

An additive field may be harmless if the validator allows unknown fields.

But suppose the source changes:

```text
userId: integer → string
```

That may break downstream processing.

Validation can detect the change early.

However, schema evolution should be handled deliberately rather than relying only on validation failures.

Chapter 69 covers schema evolution in the production section.

---

## 40. Strict vs Flexible Validation

### Strict

Reject unexpected fields or structures.

Benefit:

- contract violations are detected quickly

Trade-off:

- harmless source additions may break ingestion

### Flexible

Accept additional fields while validating required fields.

Benefit:

- more tolerant of additive changes

Trade-off:

- some contract changes may remain unnoticed

The correct choice depends on the source contract and compatibility requirements.

---

## 41. Common Mistakes

### Mistake 1 — Trusting the source

External data can be wrong even when the source is considered reliable.

### Mistake 2 — Validating only required fields

Presence does not prove correctness.

### Mistake 3 — Mixing validation and transformation

Keep responsibilities understandable.

### Mistake 4 — Retrying permanent validation failures

Repeating the same invalid input does not normally fix it.

### Mistake 5 — Silently dropping invalid records

Make failure behavior explicit.

### Mistake 6 — Over-validating

Rules unsupported by requirements can reject legitimate data.

### Mistake 7 — Logging sensitive values

Validation errors should not become a data-leak path.

### Mistake 8 — Testing only valid records

Many validation bugs appear with invalid or unexpected input.

### Mistake 9 — Validating only at the final table

Late failures are usually harder to diagnose and recover from.

### Mistake 10 — Treating validation as deduplication

Two duplicate records can both be valid.

---

## 42. Troubleshooting

### Problem: valid records are being rejected

Check:

- the validation rule
- the source contract
- recent source changes
- type conversion
- whitespace and formatting
- test fixtures

Do not immediately weaken the validator.

First confirm whether the input or the rule is wrong.

### Problem: too many records are invalid

Check:

- source schema changes
- recent deployments
- API version changes
- configuration changes
- upstream data quality
- validation metrics

### Problem: invalid records reach the target

Trace the execution path.

Check whether validation is actually called by the ingestion entry point.

The repository investigation methods from Part II are useful here.

### Problem: validation is slow

Look for expensive checks such as:

- database lookups
- external API calls
- large regular expressions
- repeated parsing
- repeated schema construction

Move cheap checks earlier and avoid unnecessary external calls.

### Problem: validation errors are hard to debug

Improve the error structure.

Include safe identifiers, field names, and rule names.

---

## 43. Production Considerations

A production validation layer may need:

- versioned schemas
- data contracts
- validation metrics
- structured errors
- quarantine
- source-specific rules
- backward compatibility
- schema evolution handling
- sensitive-data protection
- alerting
- replay support
- validation performance monitoring

The validator is part of the pipeline's contract boundary.

Changing a validation rule changes what data the system accepts.

Therefore validation changes should be tested and reviewed like other production code.

---

## 44. Definition of Done

The recipe is complete when:

- [ ] Required fields are defined.
- [ ] Expected data types are defined.
- [ ] Important value rules are defined.
- [ ] Validation happens at a deliberate pipeline boundary.
- [ ] Validation errors are understandable.
- [ ] Permanent validation failures are not blindly retried.
- [ ] Invalid-record behavior is defined.
- [ ] Valid records reach the next pipeline stage.
- [ ] Invalid records do not silently disappear.
- [ ] Unit tests cover valid and invalid inputs.
- [ ] Integration tests cover the ingestion boundary.
- [ ] Sensitive values are protected in logs.
- [ ] Validation metrics can be added where operational visibility is needed.

---

## 45. What You Learned

Validation is the pipeline's first serious answer to bad input.

The basic mental model is:

```text
Receive
  ↓
Validate
  ↓
Accept or reject
  ↓
Continue processing
```

The most important lesson is that validation should be intentional.

Do not trust incoming data simply because it came from an API.

At the same time, do not invent rules that the system does not actually require.

Good validation is specific, testable, observable, and connected to a real data contract or business requirement.

---

## 46. Recipe Progression

Part III now looks like:

```text
17. Basic ingestion
    ↓
API → PostgreSQL

18. Staging
    ↓
API → Raw → Staging → Target

19. Validation
    ↓
Source → Validate → Continue or Reject
```

The next question is:

> How do we make sure the same operation does not create unwanted effects when it runs more than once?

That leads to idempotency.

---
