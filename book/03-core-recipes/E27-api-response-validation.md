# E27 — API Response Validation

## 1. Problem Recognition

An HTTP request can succeed while the data inside the response is wrong, incomplete, malformed, or incompatible with the pipeline.

HTTP 200 does not prove that the payload is safe to ingest.

Recognize this problem when APIs change fields or types, providers return partial responses, pagination metadata disappears, or downstream failures appear far away from the extraction boundary.

> Reject or quarantine responses that do not satisfy the extraction contract before they corrupt downstream state.

## 2. Concept and Reasoning

Response validation operates at multiple levels:

```text
HTTP RESPONSE
      ↓
HTTP VALIDATION
      ↓
CONTENT VALIDATION
      ↓
STRUCTURE VALIDATION
      ↓
FIELD / TYPE VALIDATION
      ↓
BUSINESS VALIDATION
      ↓
PERSIST
```

A response can be HTTP-valid but structurally invalid, structurally valid but type-invalid, or type-valid but semantically invalid.

## 3. HTTP-Level Validation

Validate status code, content type, relevant headers, response size, and provider-specific error indicators.

```python
def validate_http_response(response):
    if response.status_code >= 400:
        raise RuntimeError(
            f"HTTP failure: {response.status_code}"
        )

    content_type = response.headers.get("Content-Type", "")

    if "application/json" not in content_type:
        raise ValueError(
            f"unexpected content type: {content_type}"
        )
```

Do not assume every successful status contains JSON.

## 4. JSON Parsing

Separate transport validation from parsing.

```python
def parse_json(response):
    try:
        return response.json()
    except ValueError as exc:
        raise ValueError(
            "response is not valid JSON"
        ) from exc
```

Never silently convert malformed JSON into an empty result. That can turn a provider failure into apparent zero-data success.

## 5. Root Structure Validation

If the contract requires `{data, next_cursor}`, validate the root object and required fields explicitly.

```python
def validate_root(payload):
    if not isinstance(payload, dict):
        raise ValueError("expected JSON object")

    required = {"data", "next_cursor"}
    missing = required - payload.keys()

    if missing:
        raise ValueError(
            f"missing required fields: {sorted(missing)}"
        )
```

Do not silently invent defaults for required fields.

## 6. Collection and Record Validation

```python
def validate_records(payload):
    rows = payload["data"]

    if not isinstance(rows, list):
        raise ValueError("data must be a list")

    for row in rows:
        if not isinstance(row, dict):
            raise ValueError("each record must be an object")

    return rows
```

## 7. Required vs Optional Fields

Define the contract explicitly:

| Field | Required | Expected type |
|---|---|---|
| `id` | yes | string |
| `name` | yes | string |
| `email` | no | string/null |
| `updated_at` | yes | timestamp |
| `status` | yes | string |

A missing optional field may be acceptable. A missing identity field usually is not.

## 8. Type Validation

APIs can unexpectedly change an ID from an integer to a string or change a scalar into an object.

Decide whether each type change should be rejected, normalized, coerced, or quarantined. Do not perform implicit conversions without understanding their correctness implications.

## 9. Strict vs Tolerant Validation

**Strict validation** rejects the response when a contract violation occurs. It is useful when partial ingestion is dangerous.

**Tolerant validation** accepts valid records while isolating invalid records. It is useful when one bad record should not block a whole batch and quarantine is available.

The choice depends on the source contract and business impact.

## 10. Response-Level vs Record-Level Failure

Distinguish a whole-page failure from one bad record.

```text
RESPONSE FAILURE
      ↓
WHOLE PAGE MAY BE UNSAFE
```

versus:

```text
RESPONSE VALID
      ↓
ONE RECORD INVALID
      ↓
QUARANTINE RECORD
```

Partial acceptance is safe only when the pipeline contract permits it.

## 11. Pagination Metadata Validation

Pagination fields are part of the extraction contract.

Validate cursor type, presence, termination semantics, and repeated-cursor behavior.

A missing cursor can be a correctness failure, not merely a warning.

## 12. Pydantic Response Contracts

```python
from pydantic import BaseModel

class Customer(BaseModel):
    id: str
    name: str
    email: str | None = None

class CustomerResponse(BaseModel):
    data: list[Customer]
    next_cursor: str | None = None

payload = CustomerResponse.model_validate(
    response.json()
)
```

Pydantic turns schema violations into explicit validation failures.

## 13. Manual Validation

Understand the mechanism even when using a library:

```python
def validate_customer(row):
    if not isinstance(row.get("id"), str):
        raise ValueError("id must be a string")

    if not isinstance(row.get("name"), str):
        raise ValueError("name must be a string")

    email = row.get("email")
    if email is not None and not isinstance(email, str):
        raise ValueError("email must be string or null")

    return row
```

## 14. Semantic Validation

Types alone are not enough. A structurally valid amount can still be invalid for the business domain.

Semantic checks can include amount ranges, supported currencies, valid statuses, timestamp rules, identifier formats, and relationship consistency.

Keep transport/schema validation separate from business validation where possible.

## 15. Timestamp Validation

Validate timezone, format, precision, and semantic plausibility before using timestamps for incremental extraction.

```python
from datetime import datetime

def parse_timestamp(value):
    parsed = datetime.fromisoformat(
        value.replace("Z", "+00:00")
    )

    if parsed.tzinfo is None:
        raise ValueError("timestamp must include timezone")

    return parsed
```

Do not silently assume a timezone for ambiguous source data.

## 16. Numeric Validation

For exact financial or measurement values, avoid converting arbitrary numeric strings directly to floating point.

```python
from decimal import Decimal

amount = Decimal("19.95")
```

Validate range, precision, scale, sign, and currency relationship.

## 17. Unknown Fields

When an API adds a field, decide whether it should be accepted, preserved, logged, treated as a schema signal, or rejected.

Rejecting every additive field makes pipelines brittle. Ignoring every new field can hide important contract changes.

## 18. Null vs Missing

These states can have different meanings:

```json
{}
```

versus:

```json
{"email": null}
```

Define the difference explicitly in the contract.

## 19. Empty vs Missing Collections

These are not automatically equivalent:

```json
{"data": []}
```

and:

```json
{}
```

An empty collection can be a valid zero-record result. A missing collection may indicate an invalid response.

## 20. Response Size Limits

Protect the pipeline from unexpectedly large payloads.

```python
MAX_BYTES = 10 * 1024 * 1024

if len(response.content) > MAX_BYTES:
    raise ValueError("response exceeds size limit")
```

For streaming responses, enforce the limit while reading.

## 21. Duplicate Records Inside a Response

A page can contain duplicates. Validate identity and decide whether duplicates should be accepted, deduplicated, quarantined, or treated as a provider defect.

Do not use row count as the only measure of extracted records.

## 22. Identity Validation

Every record should have a reliable source identity.

```python
def validate_identity(row):
    source_id = row.get("id")
    if not source_id:
        raise ValueError("missing source identity")
    return source_id
```

Stable identity enables deduplication and reconciliation.

## 23. Contract Fingerprints

A schema fingerprint can help detect response-shape changes operationally.

```text
response structure
      ↓
normalized field/type representation
      ↓
fingerprint
      ↓
compare with expected/current fingerprint
```

A fingerprint is a detection aid, not a substitute for actual validation.

## 24. Invalid Response Preservation

When validation fails, preserve enough safe evidence to diagnose the problem.

Useful metadata:

```text
run_id
request_id
endpoint
status_code
content_type
validation_error
received_at
payload_checksum
```

Be careful with raw payload storage because responses may contain sensitive information.

## 25. Validation Before Checkpointing

Validation belongs before durable progress advancement:

```text
REQUEST
  ↓
PARSE
  ↓
VALIDATE
  ↓
PERSIST
  ↓
CHECKPOINT
```

Never checkpoint merely because the network request succeeded.

## 26. Testing

### Unit tests

Test valid responses, invalid JSON, wrong root types, missing fields, wrong types, unexpected nulls, invalid timestamps, invalid numbers, malformed pagination metadata, duplicate records, and oversized responses.

```python
def test_missing_data_field_is_rejected():
    payload = {"next_cursor": None}

    try:
        validate_payload(payload)
    except ValueError:
        return

    raise AssertionError("expected validation failure")
```

### Contract tests

Maintain fixtures representing the provider's documented responses and run them against extraction-client changes.

### Negative tests

Malformed responses must be tested explicitly, not only successful responses.

## 27. Intentional Failure

### Failure Drill A — Wrong Root Type

Return a JSON array when an object is expected. Expected: response rejected, no persistence, no checkpoint advancement.

### Failure Drill B — Missing Required Field

Remove the record identity field. Expected: the page or record is rejected according to policy.

### Failure Drill C — Type Drift

Change a scalar field into an object. Expected: schema validation detects the change.

### Failure Drill D — HTML Instead of JSON

Return an HTML gateway page with HTTP 200. Expected: content validation rejects it rather than interpreting it as empty data.

### Failure Drill E — Missing Pagination Metadata

Return records without required continuation metadata. Expected: pipeline refuses to claim complete pagination unless the contract allows it.

### Failure Drill F — Oversized Response

Return a payload beyond the configured limit. Expected: response is rejected before expensive downstream processing.

## 28. Observability

| Metric / Signal | Purpose |
|---|---|
| validation failures | Contract violations |
| invalid JSON count | Malformed payloads |
| schema failures | Structural/type changes |
| semantic failures | Business-invalid data |
| quarantined records | Partial data problems |
| unexpected content types | Gateway/provider anomalies |
| response size | Abnormal payloads |
| unknown-field count | Additive schema changes |
| duplicate-record count | Source quality problems |
| validation latency | Expensive validation |

Useful logs:

```text
run_id
request_id
endpoint
status_code
content_type
validation_stage
error_class
field_name
schema_version
response_size_bytes
payload_checksum
```

Never log sensitive field values merely to diagnose validation failures.

## 29. Recovery

When validation fails:

1. Identify the validation layer that failed.
2. Determine whether the failure is transient, contract-related, or data-specific.
3. Preserve safe diagnostic metadata.
4. Do not advance the extraction checkpoint.
5. Retry only if the failure is actually retryable.
6. Quarantine individual records when the contract permits partial acceptance.
7. Update the parser only after understanding the provider change.
8. Reprocess the affected source position.
9. Reconcile if invalid data may already have been loaded.

Do not disable validation merely to make the pipeline green.

## 30. Production Tools You Should Know

### Pydantic
Useful for explicit Python data contracts and typed response validation.

### JSON Schema
A language-independent way to describe JSON structure and validate payloads.

### OpenAPI
Documents HTTP API contracts, schemas, parameters, and response definitions.

Use tools to express contracts, but understand the validation layers underneath them.

## 31. Production Runbook

### Before deployment

- Document expected status codes.
- Document content types.
- Define response schemas.
- Define required and optional fields.
- Define null behavior.
- Define pagination metadata.
- Define type normalization rules.
- Define semantic validation.
- Define partial-record handling.
- Define response-size limits.
- Define quarantine behavior.
- Define schema-change handling.

### If validation failures suddenly increase

1. Check provider/API status.
2. Check endpoint and response status.
3. Check content type.
4. Compare payload structure with the contract.
5. Check schema/type drift.
6. Check whether a gateway is returning unexpected content.
7. Inspect the affected validation stage.
8. Preserve the last safe checkpoint.
9. Reprocess after the contract issue is understood.

### If unknown fields appear

1. Determine whether they are additive.
2. Check whether required fields changed.
3. Check whether types changed.
4. Decide whether to preserve or ignore the new fields.
5. Update the contract deliberately.

### What not to do

- Do not treat HTTP 200 as proof of valid data.
- Do not convert malformed responses into empty responses.
- Do not silently drop required fields.
- Do not disable validation during an incident.
- Do not advance checkpoints after validation failure.
- Do not log sensitive payloads indiscriminately.

## 32. Common Mistakes

1. Validating only the HTTP status code.
2. Assuming Content-Type is always correct.
3. Treating missing fields as empty values.
4. Ignoring type drift.
5. Treating null and missing as identical without a contract.
6. Using floats for exact financial values.
7. Treating every unknown field as a breaking change.
8. Accepting malformed pagination metadata.
9. Advancing checkpoints before validation.
10. Silently converting malformed responses into empty datasets.
11. Logging sensitive payload values during debugging.
12. Testing only successful responses.

## 33. Definition of Done

- [ ] HTTP status is validated.
- [ ] Content type is validated.
- [ ] JSON parsing failures are explicit.
- [ ] Root structure is validated.
- [ ] Required fields are defined.
- [ ] Optional/null behavior is documented.
- [ ] Field types are validated.
- [ ] Pagination metadata is validated.
- [ ] Identity fields are validated.
- [ ] Semantic validation exists where required.
- [ ] Response-size limits are defined.
- [ ] Duplicate-record behavior is defined.
- [ ] Unknown-field behavior is intentional.
- [ ] Invalid data cannot advance checkpoints.
- [ ] Quarantine behavior is documented where partial acceptance is allowed.
- [ ] Contract fixtures exist.
- [ ] Negative tests pass.
- [ ] Validation metrics exist.
- [ ] Recovery procedures are documented.

## 34. What You Learned

After this recipe, you should be able to independently:
- distinguish HTTP success from data validity
- validate API response structure
- validate fields and types
- handle null versus missing values
- validate pagination metadata
- detect schema and type drift
- perform semantic validation
- protect pipelines from oversized or malformed responses
- choose strict versus tolerant validation
- quarantine invalid records safely
- preserve checkpoint correctness
- test malformed API responses
- diagnose provider contract changes
- operate response validation in production

### Core Mental Model

```text
HTTP RESPONSE
      ↓
STATUS / HEADERS
      ↓
PARSE
      ↓
STRUCTURE
      ↓
FIELDS / TYPES
      ↓
SEMANTICS
      ↓
PERSIST
      ↓
CHECKPOINT
```

> A successful HTTP request is not successful data extraction. The pipeline should advance only after the response satisfies the extraction contract and the resulting data is safely persisted.