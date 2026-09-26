# Recipe 20 — Boundary Validation

A production pipeline should never assume that data is safe simply because an earlier stage validated it.

Every important boundary is a place where data can become invalid:

```text
SOURCE
  ↓
[BOUNDARY VALIDATION]
  ↓
STAGING
  ↓
[BOUNDARY VALIDATION]
  ↓
TRANSFORMATION
  ↓
[BOUNDARY VALIDATION]
  ↓
TARGET
```

The core rule is:

> Validate data at the boundary where the next component is about to trust it.

This recipe teaches how to build that mechanism yourself.

---

## 1. Problem Recognition

### The production problem

Consider a pipeline:

```text
API
 ↓
Ingestion
 ↓
Staging
 ↓
Transformation
 ↓
Warehouse
```

The ingestion layer validates:

- `event_id` exists
- `amount` is numeric
- `currency` is valid
- `occurred_at` is a timestamp

It is tempting to conclude:

> The data is validated, so downstream code can trust it.

That assumption is unsafe.

Data can change after the first validation.

Examples:

- a transformation creates an invalid value
- a mapping converts a valid currency into an unsupported value
- a nullable field becomes required downstream
- a database migration changes a constraint
- a third-party source changes its schema
- a bug modifies a field
- a replay uses an older payload version
- a batch is manually repaired incorrectly
- a new producer bypasses the original validation path

The problem is not only **bad incoming data**.

The problem is **trusting data across boundaries without proving the contract still holds**.

### How do you recognize this problem?

Look for:

- validation exists only at ingestion
- downstream functions assume fields are valid
- transformation code has no post-condition checks
- target writes fail with unclear errors
- malformed records reach the database
- different pipeline stages implement different validation rules
- a single validation result is reused long after the data has changed
- production failures happen far away from the original corruption

A strong warning sign is:

```text
validate once
    ↓
many transformations
    ↓
target failure
```

The failure is detected too late.

### Recognition checklist

Ask:

- Where does data enter this component?
- What assumptions does this component make?
- What must be true before it accepts the data?
- Can the previous component guarantee those conditions still hold?
- Can this component modify the data?
- Where is the last safe place to reject invalid data?

If the answer to the last question is unclear, you probably need boundary validation.

---

## 2. Concept and Reasoning

### What is a boundary?

A boundary is a point where one component hands data to another component.

Examples:

```text
HTTP response → parser
parser → staging
staging → transformer
transformer → warehouse loader
producer → queue
consumer → processor
processor → database
batch file → ingestion job
```

The receiving component should define what it is willing to accept.

### Producer responsibility vs consumer responsibility

The producer should produce valid data.

The consumer should still validate what it receives.

These are not contradictory.

The producer controls:

```text
"What I send"
```

The consumer controls:

```text
"What I am willing to trust"
```

A reliable pipeline needs both.

### Boundary validation is a contract check

Suppose a transformation requires:

```text
amount > 0
currency ∈ {USD, EUR, GBP}
event_id is non-empty
occurred_at is a valid UTC timestamp
```

Those conditions form the input contract of the transformation.

The transformation should not silently assume them.

Instead:

```text
incoming data
      ↓
validate contract
   /       \
valid     invalid
  ↓          ↓
process    reject/quarantine
```

### Pre-condition and post-condition

A useful mental model is:

**Pre-condition**

> What must be true before this component starts?

**Post-condition**

> What must be true after this component finishes?

For a transformation:

```text
Input
 ↓
PRE-CONDITION
 ↓
Transform
 ↓
POST-CONDITION
 ↓
Output
```

This makes validation part of the component design rather than an afterthought.

### Validate at meaningful boundaries, not every line

Do not add arbitrary validation everywhere.

Validate where trust changes.

Good:

```text
API → service
service → staging
staging → transformation
transformation → target
```

Bad:

```text
function A
 ↓ validate
function B
 ↓ validate
function C
 ↓ validate
function D
 ↓ validate
```

The objective is not maximum validation.

The objective is **clear ownership of contracts at meaningful boundaries**.

### Validation should be deterministic

For the same input and contract:

```text
same input + same rules = same validation result
```

This makes failures reproducible and easier to debug.

### Validation should produce evidence

Do not return only:

```text
False
```

Return enough information to answer:

- which record failed?
- which boundary failed?
- which rule failed?
- what was the observed value?
- what was expected?
- when did validation occur?
- what pipeline version performed the check?

Never put secrets or unnecessary PII into validation logs.

---

## 3. Implementation

We will build a small boundary-validation system in Python.

The implementation will have:

1. a canonical record
2. a validation contract
3. a boundary validator
4. a transformation
5. a second boundary validation
6. explicit validation errors
7. tests
8. an intentional failure drill
9. recovery

### 3.1 Project structure

Create:

```text
boundary-validation/
├── validator.py
└── test_validator.py
```

### 3.2 Define the record

Create `validator.py`:

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from decimal import Decimal


@dataclass(frozen=True)
class Payment:
    event_id: str
    amount: Decimal
    currency: str
    occurred_at: datetime
```

The model represents the data contract used by the next stage.

---

### 3.3 Define validation errors

```python
@dataclass(frozen=True)
class ValidationFailure:
    boundary: str
    rule: str
    message: str


class BoundaryValidationError(Exception):
    def __init__(self, failures: list[ValidationFailure]):
        self.failures = failures
        summary = "; ".join(
            f"{failure.rule}: {failure.message}"
            for failure in failures
        )
        super().__init__(summary)
```

A validation failure contains the evidence required to diagnose the problem.

---

### 3.4 Implement the validator

```python
ALLOWED_CURRENCIES = {"USD", "EUR", "GBP"}


def validate_payment(payment: Payment, boundary: str) -> None:
    failures: list[ValidationFailure] = []

    if not payment.event_id.strip():
        failures.append(
            ValidationFailure(
                boundary=boundary,
                rule="event_id_required",
                message="event_id must not be empty",
            )
        )

    if payment.amount <= Decimal("0"):
        failures.append(
            ValidationFailure(
                boundary=boundary,
                rule="amount_positive",
                message="amount must be greater than zero",
            )
        )

    if payment.currency not in ALLOWED_CURRENCIES:
        failures.append(
            ValidationFailure(
                boundary=boundary,
                rule="currency_supported",
                message=f"unsupported currency: {payment.currency}",
            )
        )

    if payment.occurred_at.tzinfo is None:
        failures.append(
            ValidationFailure(
                boundary=boundary,
                rule="timestamp_timezone",
                message="occurred_at must contain timezone information",
            )
        )

    if payment.occurred_at.tzinfo is not None:
        if payment.occurred_at.utcoffset() is None:
            failures.append(
                ValidationFailure(
                    boundary=boundary,
                    rule="timestamp_offset",
                    message="occurred_at must have a valid UTC offset",
                )
            )

    if failures:
        raise BoundaryValidationError(failures)
```

The validator does not modify the record.

It answers one question:

> Does this record satisfy the contract required at this boundary?

---

### 3.5 Create a transformation

Suppose the transformation normalizes the amount into a reporting representation.

```python
def transform_payment(payment: Payment) -> Payment:
    validate_payment(payment, boundary="transform_input")

    normalized_currency = payment.currency.upper()

    transformed = Payment(
        event_id=payment.event_id,
        amount=payment.amount,
        currency=normalized_currency,
        occurred_at=payment.occurred_at.astimezone(timezone.utc),
    )

    validate_payment(transformed, boundary="transform_output")

    return transformed
```

Notice the two validation points:

```text
incoming record
      ↓
transform_input validation
      ↓
transformation
      ↓
transform_output validation
      ↓
next component
```

The transformation has both an input contract and an output contract.

---

### 3.6 Add a target boundary

A target may have stricter rules than staging.

For example:

```python
def validate_target_payment(payment: Payment) -> None:
    validate_payment(payment, boundary="target")

    if payment.amount.as_tuple().exponent < -2:
        raise BoundaryValidationError(
            [
                ValidationFailure(
                    boundary="target",
                    rule="amount_scale",
                    message="target accepts at most two decimal places",
                )
            ]
        )
```

Now the complete flow is:

```text
Source
  ↓
source boundary
  ↓
Staging
  ↓
transformation input boundary
  ↓
Transformation
  ↓
transformation output boundary
  ↓
Target boundary
  ↓
Warehouse
```

---

## 4. Testing

Testing must prove both acceptance and rejection.

Create `test_validator.py`:

```python
from datetime import datetime, timezone
from decimal import Decimal

import pytest

from validator import (
    BoundaryValidationError,
    Payment,
    transform_payment,
    validate_payment,
)


def valid_payment() -> Payment:
    return Payment(
        event_id="evt-001",
        amount=Decimal("125.50"),
        currency="USD",
        occurred_at=datetime(2026, 9, 26, 10, 0, tzinfo=timezone.utc),
    )


def test_valid_payment_passes():
    payment = valid_payment()

    validate_payment(payment, boundary="staging")


def test_missing_event_id_fails():
    payment = Payment(
        event_id="",
        amount=Decimal("10.00"),
        currency="USD",
        occurred_at=datetime.now(timezone.utc),
    )

    with pytest.raises(BoundaryValidationError) as error:
        validate_payment(payment, boundary="staging")

    assert "event_id_required" in str(error.value)


def test_negative_amount_fails():
    payment = Payment(
        event_id="evt-002",
        amount=Decimal("-1.00"),
        currency="USD",
        occurred_at=datetime.now(timezone.utc),
    )

    with pytest.raises(BoundaryValidationError):
        validate_payment(payment, boundary="staging")


def test_unsupported_currency_fails():
    payment = Payment(
        event_id="evt-003",
        amount=Decimal("10.00"),
        currency="XYZ",
        occurred_at=datetime.now(timezone.utc),
    )

    with pytest.raises(BoundaryValidationError):
        validate_payment(payment, boundary="staging")


def test_timezone_is_required():
    payment = Payment(
        event_id="evt-004",
        amount=Decimal("10.00"),
        currency="USD",
        occurred_at=datetime(2026, 9, 26, 10, 0),
    )

    with pytest.raises(BoundaryValidationError):
        validate_payment(payment, boundary="staging")


def test_transformation_validates_input_and_output():
    result = transform_payment(valid_payment())

    assert result.event_id == "evt-001"
    assert result.currency == "USD"
    assert result.occurred_at.tzinfo is not None
```

Run:

```bash
pytest -q
```

Expected result:

```text
6 passed
```

### Edge cases you should test

At minimum:

- empty identifier
- whitespace-only identifier
- zero amount
- negative amount
- unsupported currency
- timezone-naive timestamp
- invalid timezone offset
- excessive decimal scale
- transformation that creates an invalid output
- target contract stricter than staging

The important lesson is:

> Test the contract at the boundary, not only the transformation logic.

---

## 5. Observability

Validation is operationally useful only when failures are observable.

### 5.1 Log the boundary

A useful validation failure should identify:

```text
boundary
rule
event_id
pipeline_version
timestamp
```

Example:

```json
{
  "level": "ERROR",
  "event": "boundary_validation_failed",
  "boundary": "transform_input",
  "rule": "currency_supported",
  "event_id": "evt-1042",
  "pipeline_version": "2026.09.26"
}
```

Do not log:

- passwords
- access tokens
- authentication headers
- full documents
- unnecessary PII

### 5.2 Metrics

Track at least:

```text
boundary_validation_total
boundary_validation_failures_total
boundary_validation_failure_rate
boundary_validation_latency
```

Useful dimensions:

```text
boundary
rule
pipeline
version
source
```

Avoid unbounded labels such as:

```text
full_event_id
email
transaction_reference
raw_payload
```

Those can create metric-cardinality problems.

### 5.3 Operational questions

Your monitoring should let you answer:

- Which boundary is failing?
- Which validation rule is failing?
- Is the failure rate increasing?
- Did failures begin after a deployment?
- Is one source producing most failures?
- Are failures isolated or systemic?
- How many records are blocked?
- Are failures being quarantined correctly?

---

## 6. Intentional Failure

Do not consider the recipe complete until you deliberately break the boundary.

### Failure drill: remove validation before transformation

Temporarily change:

```python
def transform_payment(payment: Payment) -> Payment:
    validate_payment(payment, boundary="transform_input")
```

to:

```python
def transform_payment(payment: Payment) -> Payment:
    # INTENTIONAL FAILURE DRILL
    # validation removed
    ...
```

Then send an invalid payment:

```python
payment = Payment(
    event_id="evt-broken",
    amount=Decimal("-50.00"),
    currency="USD",
    occurred_at=datetime.now(timezone.utc),
)
```

Observe what happens.

The invalid record may now travel deeper into the pipeline.

This demonstrates the danger of relying on validation that happened earlier.

### Second failure drill: corrupt the transformation output

Temporarily introduce:

```python
transformed = Payment(
    event_id=payment.event_id,
    amount=Decimal("-999.00"),
    currency=payment.currency,
    occurred_at=payment.occurred_at,
)
```

If the output boundary is removed, the invalid value can reach the next component.

With output validation enabled:

```text
Transformation
      ↓
invalid output
      ↓
output boundary
      ↓
REJECT
```

Without it:

```text
Transformation
      ↓
invalid output
      ↓
next component
      ↓
corruption / target failure
```

---

## 7. Recovery

The recovery process should be evidence-driven.

### Step 1 — Identify the failing boundary

Example:

```text
boundary = transform_output
```

### Step 2 — Identify the rule

Example:

```text
rule = amount_positive
```

### Step 3 — Identify affected records

Use the stable event identifier:

```text
event_id = evt-broken
```

Do not recover an entire dataset when only a bounded set of records failed.

### Step 4 — Determine whether the failure is deterministic

Ask:

- Is the data invalid?
- Did the transformation introduce the problem?
- Did the contract change?
- Did the source change?
- Was the validation rule incorrect?

### Step 5 — Repair the root cause

Examples:

- fix transformation logic
- fix mapping
- update the contract
- repair malformed source data
- deploy compatible schema handling

Do not simply disable validation to make the pipeline green.

### Step 6 — Revalidate the affected records

Run the same boundary contract again.

```text
repair
  ↓
validate
  ↓
valid?
 / no  yes
 |    |
stop  replay
```

### Step 7 — Replay safely

Replay only the affected records.

Use the idempotency mechanism from Recipe 6.

### Step 8 — Reconcile

Confirm:

```text
failed records before recovery
        =
successfully recovered records
        +
permanently rejected records
        +
records still pending
```

There should be no unexplained records.

### Step 9 — Verify the target

Do not stop when validation passes.

Confirm that the repaired records actually reached the intended destination.

---

## 8. Production Tools You Should Know

These tools provide production implementations of concepts covered in this recipe.

| Tool | What to know |
|---|---|
| **Pydantic** | Defines typed Python data models and validates data at application/service boundaries. |
| **Great Expectations** | Defines and executes data-quality expectations against datasets at pipeline boundaries. |
| **dbt** | Applies SQL-based data tests and model contracts around transformation and warehouse boundaries. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the validation mechanism.

---

## 9. Production Runbook

### When validation failures increase

Check:

1. Which boundary is failing?
2. Which rule is failing?
3. Which source is producing the failures?
4. Did a deployment happen recently?
5. Did the source schema change?
6. Did the validation contract change?
7. Are failed records being quarantined?
8. Is the DLQ/quarantine backlog increasing?
9. Can the affected records be safely replayed?

### What to do

```text
Identify boundary
      ↓
Identify rule
      ↓
Identify affected records
      ↓
Determine root cause
      ↓
Fix producer / transformation / contract
      ↓
Revalidate
      ↓
Replay bounded records
      ↓
Reconcile
      ↓
Close incident
```

### What not to do

Do not:

- disable validation globally
- delete failed records to reduce the backlog
- replay the entire source unnecessarily
- modify production data manually without an audit trail
- silently coerce invalid values
- change validation rules only to make tests pass
- treat a target database error as the first validation mechanism
- assume upstream validation guarantees downstream validity

---

## 10. Common Mistakes

### Mistake 1 — Validate only at ingestion

**Problem:** Later stages can still corrupt or reinterpret data.

**Better:** Validate at meaningful trust boundaries.

### Mistake 2 — Duplicate validation rules everywhere

**Problem:** Rules drift.

**Better:** Define reusable contracts and make boundary ownership explicit.

### Mistake 3 — Validation returns only true/false

**Problem:** Operators cannot diagnose the failure.

**Better:** Return structured failure evidence.

### Mistake 4 — Log the entire payload

**Problem:** Can expose sensitive information and create huge logs.

**Better:** Log identifiers, rule names, boundary names, and safe diagnostic metadata.

### Mistake 5 — Put every failure into quarantine

**Problem:** Temporary infrastructure failures become data-quality records.

**Better:** Distinguish retryable operational failures from invalid data.

### Mistake 6 — Disable downstream validation because upstream validation exists

**Problem:** The data may have changed since the upstream check.

**Better:** Treat every meaningful boundary as a new trust decision.

### Mistake 7 — Validate after writing to the target

**Problem:** The target becomes the failure detector.

**Better:** Validate before the irreversible side effect whenever possible.

### Mistake 8 — Change validation rules without versioning

**Problem:** Historical records may have been processed under different contracts.

**Better:** Track contract/version information when validation behavior changes materially.

---

## 11. Definition of Done

You are done with this recipe when you can independently:

- explain what a pipeline boundary is
- identify meaningful trust boundaries
- distinguish producer responsibility from consumer responsibility
- define a boundary contract
- implement validation from scratch
- validate input before processing
- validate output after transformation
- produce structured validation failures
- write tests for valid and invalid records
- observe validation failures
- measure failure rates by boundary and rule
- intentionally remove a validation boundary
- reproduce the resulting failure
- identify the failed boundary from evidence
- repair the root cause
- revalidate affected records
- replay only the required records
- reconcile the recovery
- explain where Pydantic, Great Expectations, and dbt fit
- operate the validation mechanism using a runbook

A strong final test is:

> Given an unfamiliar pipeline, can you identify where data crosses trust boundaries and decide what must be validated at each one?

If yes, you have learned the mechanism rather than memorized the recipe.

---

## 12. What You Learned

The central lesson is:

> **Validation is not a one-time ingestion step. It is a boundary contract.**

A reliable pipeline treats each meaningful handoff as a new trust decision:

```text
DATA
 ↓
BOUNDARY
 ↓
VALIDATE
 ↓
TRUST
 ↓
PROCESS
 ↓
BOUNDARY
 ↓
VALIDATE AGAIN
 ↓
TRUST AGAIN
```

This protects the pipeline from a class of failures that ingestion-only validation cannot prevent.

The deeper engineering principle is:

> **Never make a component trust an assumption that it has no evidence for.**
