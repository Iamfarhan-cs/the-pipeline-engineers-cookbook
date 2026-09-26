# E28 — API Schema Changes

## 1. Problem Recognition

An API can keep returning HTTP 200 while changing the shape, type, meaning, or availability of fields.

Typical changes include:
- a field being renamed
- a field being removed
- a field changing type
- a new required field appearing
- an object becoming an array
- an enum gaining a new value
- a timestamp changing format
- pagination metadata changing
- nested objects changing structure

Recognize the problem when response-validation failures suddenly increase, downstream transformations break, fields become unexpectedly null, or provider documentation announces a new API version.

The production objective is:

> Detect, classify, and safely absorb API contract changes without silently corrupting pipeline state.

## 2. Concept and Reasoning

Schema change handling is not the same as response validation.

- **E27 — Response Validation:** Is this response valid against the current contract?
- **E28 — Schema Changes:** What should the pipeline do when the contract itself changes?

The key distinction is:

```text
CURRENT RESPONSE
      ↓
VALIDATE AGAINST CONTRACT
      ↓
CHANGE DETECTED?
   /          \
 NO           YES
 ↓             ↓
PROCESS     CLASSIFY CHANGE
               ↓
        COMPATIBLE / BREAKING
               ↓
        MIGRATE / ADAPT / STOP
```

## 3. Schema Compatibility

Not every change has the same risk.

| Change | Typical classification |
|---|---|
| Add optional field | additive |
| Add unknown field | usually compatible |
| Remove optional field | potentially compatible |
| Remove required field | breaking |
| Rename field | breaking |
| Change string to object | breaking |
| Change integer to string | potentially breaking |
| Add enum value | potentially breaking |
| Change timestamp semantics | breaking risk |
| Change pagination contract | potentially breaking |

Classification must consider how the pipeline actually consumes the field.

## 4. Additive Changes

Example:

```json
{
  "id": "123",
  "name": "Alice",
  "phone": "+123456789"
}
```

If `phone` is new and optional, an existing consumer may continue working.

However, detect additive fields when useful because they can indicate provider evolution.

## 5. Removed Fields

Suppose the previous contract contained:

```json
{
  "id": "123",
  "name": "Alice",
  "email": "alice@example.com"
}
```

and the provider removes `email`.

If downstream logic requires `email`, this is a breaking change.

Never replace a removed required field with an invented default without an explicit business decision.

## 6. Renamed Fields

A rename is usually a breaking change for a strict consumer.

```text
customer_id
    ↓
id
```

Treat the old and new fields as separate contract versions until migration is deliberately completed.

## 7. Type Changes

Example:

```text
Before: "amount": 19.95
After:  "amount": "19.95"
```

A type change may be safely normalized in some pipelines, but only if the semantic meaning is unchanged and precision is preserved.

Do not automatically coerce every type change.

## 8. Enum Changes

Suppose the old status set is:

```text
pending
completed
failed
```

The provider adds:

```text
cancelled
```

A pipeline that assumes only three values may fail or misclassify the new status.

Unknown enum values should produce an explicit signal rather than silently becoming an arbitrary default.

## 9. Nested Object Changes

APIs may change:

```json
"customer": {"id": "123"}
```

into:

```json
"customer": {"identity": {"id": "123"}}
```

Field-path changes can break transformations even when the overall JSON remains valid.

Track important field paths, not just top-level keys.

## 10. Schema Versioning

When the provider exposes explicit versions, record them.

Useful metadata includes:

```text
provider
endpoint
api_version
schema_version
observed_at
contract_fingerprint
```

Do not assume a URL version is the same thing as a payload schema version. Record what the provider actually documents.

## 11. Versioned Contracts

Keep contracts versioned in source control.

Example:

```text
contracts/
├── customer_v1.json
├── customer_v2.json
└── customer_v3.json
```

This lets the pipeline explicitly support known versions rather than guessing which structure it received.

## 12. Contract Detection

A simple field/type representation can detect structural changes.

```python
def describe_record(row):
    return {
        key: type(value).__name__
        for key, value in sorted(row.items())
    }


observed = describe_record(record)
```

For production use, the representation should also account for nested structures, nullability, arrays, and important semantic constraints.

## 13. Schema Fingerprinting

Create a deterministic representation and hash it:

```python
import hashlib
import json


def fingerprint_schema(schema):
    canonical = json.dumps(
        schema,
        sort_keys=True,
        separators=(",", ":"),
    )
    return hashlib.sha256(
        canonical.encode("utf-8")
    ).hexdigest()
```

A fingerprint is useful for detecting that something changed.

It does not tell you whether the change is safe.

## 14. Compare Old and New Contracts

Conceptually:

```text
KNOWN CONTRACT
      ↓
NEW RESPONSE STRUCTURE
      ↓
DIFF
      ↓
CLASSIFY
      ↓
DECIDE
```

A useful schema diff should identify:
- added fields
- removed fields
- changed types
- changed nullability
- changed enum values
- changed nested structures
- changed array/object cardinality

## 15. Breaking vs Non-Breaking Changes

Use explicit compatibility rules.

Example:

```python
def classify_change(change):
    if change.kind in {
        "required_field_removed",
        "field_renamed",
        "type_changed",
    }:
        return "breaking"

    if change.kind == "optional_field_added":
        return "compatible"

    return "review_required"
```

This is intentionally conservative. Real compatibility depends on the consumer contract.

## 16. Schema Evolution Strategies

Common strategies include:

### Strategy A — Adapt in place

Use when the change is small and backwards compatibility is clear.

### Strategy B — Version the parser

Support both schemas temporarily.

```text
V1 RESPONSE ──→ V1 PARSER ──→ COMMON MODEL
V2 RESPONSE ──→ V2 PARSER ──→ COMMON MODEL
```

### Strategy C — Version the source contract

Run separate extraction contracts for different provider versions.

### Strategy D — Stop safely

For an unsafe breaking change, stop extraction at the last known safe position rather than loading untrusted data.

## 17. The Common Internal Model

A useful architecture is to translate provider-specific schemas into a stable internal representation.

```text
PROVIDER V1 ──→ PARSER V1 ──┐
                            ├──→ INTERNAL MODEL
PROVIDER V2 ──→ PARSER V2 ──┘
```

This isolates provider evolution from downstream transformations.

Do not let every downstream table understand every provider version.

## 18. Field Mapping During Migration

Example:

```python
def parse_v2(row):
    return {
        "customer_id": row["id"],
        "name": row["display_name"],
    }
```

The internal model remains stable even when the provider changes field names.

## 19. Dual-Read Migration

For high-risk migrations, temporarily read both old and new representations.

```text
V1 DATA ──→ INTERNAL MODEL ──┐
                             ├──→ COMPARE
V2 DATA ──→ INTERNAL MODEL ──┘
```

Compare outputs before removing the old parser.

Useful comparison signals include:
- record counts
- identities
- field population
- normalized values
- business totals

## 20. Shadow Validation

A new parser can run without becoming authoritative.

```text
PRODUCTION RESPONSE
       ↓
OLD PARSER ──→ PRODUCTION OUTPUT
       ↓
NEW PARSER ──→ SHADOW OUTPUT
       ↓
COMPARE
```

This reduces migration risk because the new contract can be tested against real traffic before cutover.

## 21. Schema Change and Incremental Extraction

Schema migration must not corrupt extraction state.

```text
READ SAFE CHECKPOINT
       ↓
EXTRACT
       ↓
VALIDATE CURRENT CONTRACT
       ↓
ADAPT / MIGRATE
       ↓
PERSIST
       ↓
ADVANCE CHECKPOINT
```

If a breaking change is detected before persistence, keep the previous checkpoint.

## 22. Schema Change and Pagination

Do not advance pagination state based only on successful HTTP communication.

```text
PAGE N
 ↓
VALIDATE SCHEMA
 ↓
PROCESS
 ↓
PERSIST
 ↓
CHECKPOINT PAGE N+1
```

A schema failure on page N means page N+1 must not become the new durable position.

## 23. Schema Change and Partial Responses

Be careful when providers deploy changes gradually.

One page may contain V1 records while another contains V2 records.

Therefore:

- detect schema per response where necessary
- do not assume one successful response defines the whole run
- support mixed-version behavior only when explicitly designed
- record observed schema versions

## 24. Schema Registry Concept

A schema registry provides a centralized place to store and retrieve known schemas and compatibility information.

The important DE concept is not the product itself:

```text
PRODUCER CONTRACT
      ↓
REGISTERED SCHEMA
      ↓
COMPATIBILITY CHECK
      ↓
CONSUMER
```

For API extraction, the provider's documented contract remains authoritative unless your organization defines an additional internal contract.

## 25. Schema Drift Detection

Track signals such as:

| Signal | Meaning |
|---|---|
| new field | additive evolution |
| missing field | possible breaking change |
| type change | contract drift |
| enum change | behavior drift |
| nested-path change | structural drift |
| schema fingerprint change | structural signal |
| validation failure spike | operational drift |

Schema drift detection should produce actionable evidence, not noise.

## 26. Observability

Track:

| Metric | Purpose |
|---|---|
| schema validation failures | detects incompatible responses |
| schema fingerprint changes | detects structural evolution |
| unknown fields | detects additive changes |
| missing required fields | detects breaking changes |
| type changes | detects contract drift |
| enum violations | detects new states |
| records by schema version | measures migration progress |
| parser version | identifies deployed behavior |
| migration comparison mismatches | validates migrations |

Useful log fields:

```text
run_id
request_id
endpoint
api_version
schema_version
parser_version
schema_fingerprint
change_type
compatibility
field_name
```

Do not log sensitive values merely to prove that a field changed.

## 27. Testing

### Unit Tests

Test:
- optional field addition
- required field removal
- field rename
- type change
- enum addition
- nested structure change
- nullability change
- schema fingerprint changes
- compatible and breaking classifications

### Contract Tests

Keep fixtures for every supported provider schema version.

### Migration Tests

Verify that V1 and V2 parsers produce the same internal model when the source data is semantically equivalent.

### Regression Tests

Run existing extraction tests against the new contract before deployment.

## 28. Intentional Failure

### Failure Drill A — Remove Required Field

Remove a required field from the fixture.

Expected:
- change detected
- classified as breaking
- data is not persisted
- checkpoint does not advance

### Failure Drill B — Change Field Type

Change a numeric field to an object.

Expected:
- type drift detected
- parser rejects or routes to explicit migration logic

### Failure Drill C — Add Enum Value

Introduce a previously unknown status.

Expected:
- unknown state is visible
- no arbitrary default classification occurs

### Failure Drill D — Simulate V1/V2 Mixed Responses

Return different schema versions across pages.

Expected:
- schema version is detected per response
- only supported versions are processed
- unsupported versions stop or quarantine safely

### Failure Drill E — Break the New Parser

Deploy a deliberately incorrect V2 mapping in shadow mode.

Expected:
- production V1 output remains authoritative
- comparison detects mismatches
- migration is not promoted

## 29. Recovery

When a schema change is detected:

1. Preserve the last safe checkpoint.
2. Identify the exact contract difference.
3. Classify the change as compatible, breaking, or review-required.
4. Determine whether the provider changed the API intentionally.
5. Capture a representative response safely.
6. Update or version the parser deliberately.
7. Add regression fixtures.
8. Replay the affected source range.
9. Reconcile source and target counts and business totals.
10. Retire the old contract only after the migration is proven.

Never solve a breaking schema change by weakening validation without understanding the new contract.

## 30. Production Tools You Should Know

### Pydantic
Useful for versioned Python response models and explicit validation.

### JSON Schema
Useful for machine-readable JSON contracts and schema comparison.

### Apicurio Registry
A schema-registry example worth recognizing when working with centralized schema governance.

These tools support contract management; the underlying engineering problem is schema compatibility and controlled evolution.

## 31. Production Runbook

### When schema drift is detected

1. Identify endpoint and schema version.
2. Compare the current response with the known contract.
3. Identify added, removed, or changed fields.
4. Determine whether the change is compatible.
5. Check provider release notes or documentation.
6. Preserve the last safe checkpoint.
7. Decide whether to adapt, version, or stop.

### For a compatible additive change

- update contract documentation
- add regression coverage
- monitor the new field
- deploy deliberately

### For a breaking change

- stop unsafe ingestion
- preserve checkpoint
- implement and test migration
- replay affected data
- reconcile downstream state

### What not to do

- Do not silently coerce breaking type changes.
- Do not invent values for removed required fields.
- Do not advance checkpoints after unsupported schema responses.
- Do not delete old contracts before migration validation.
- Do not assume an API version change is harmless.
- Do not make schema changes invisible to operations.

## 32. Common Mistakes

1. Treating every schema change as harmless.
2. Treating every new field as a breaking change.
3. Ignoring enum evolution.
4. Silently coercing incompatible types.
5. Keeping only one unversioned parser.
6. Allowing provider-specific schema details into every downstream transformation.
7. Advancing checkpoints after a schema failure.
8. Migrating without replay tests.
9. Migrating without reconciliation.
10. Logging sensitive payloads during schema investigation.
11. Removing the old parser before the new parser is proven.
12. Using fingerprints without actually classifying the difference.

## 33. Definition of Done

- [ ] Current API contracts are documented.
- [ ] Schema versions are identifiable where supported.
- [ ] Important fields and types are explicitly defined.
- [ ] Compatibility rules exist.
- [ ] Additive changes are distinguished from breaking changes.
- [ ] Schema drift is observable.
- [ ] Required-field removal is detected.
- [ ] Type changes are detected.
- [ ] Enum changes are detected.
- [ ] Nested structure changes are tested.
- [ ] Supported schema versions have explicit parsers or models.
- [ ] Breaking changes cannot silently advance checkpoints.
- [ ] Migration fixtures exist.
- [ ] Regression tests exist.
- [ ] Replay and reconciliation procedures exist.
- [ ] Rollback or safe-stop behavior is documented.

## 34. What You Learned

After this recipe, you should be able to independently:
- identify API schema evolution
- distinguish compatible and breaking changes
- detect field, type, enum, and structural changes
- version API contracts
- build version-specific parsers
- map multiple provider versions into one internal model
- use schema fingerprints for detection
- perform shadow migrations
- protect checkpoints during schema changes
- test migrations with fixtures and replay
- recover safely from provider contract changes
- operate schema evolution in production

### Core Mental Model

```text
OBSERVE CURRENT RESPONSE
          ↓
VALIDATE AGAINST KNOWN CONTRACT
          ↓
CHANGE?
    /             \
  NO               YES
  ↓                 ↓
PROCESS       DIFF THE SCHEMA
                    ↓
             CLASSIFY CHANGE
                    ↓
          COMPATIBLE / BREAKING
                ↓
       ADAPT / VERSION / STOP
                ↓
             TEST + REPLAY
                ↓
             RECONCILE
```

> Schema evolution is a controlled migration problem. Detect the difference, classify its compatibility, preserve the last safe position, and change the parser deliberately.