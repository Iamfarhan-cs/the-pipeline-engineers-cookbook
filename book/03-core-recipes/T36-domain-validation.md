# T36 — Domain Validation

> **Goal:** Learn how to verify that correctly typed values belong to an explicitly permitted business domain, including enums, reference data, effective-dated codes, unknown values, and controlled handling of domain violations.

## 1. Problem Recognition

Domain validation answers:

    Does this correctly typed value belong to the set of values permitted by the business contract?

Examples:

    status = 'approved'
    allowed = {'pending', 'approved', 'rejected'}

    currency = 'EUR'
    allowed = supported currency codes

    country_code = 'PK'
    allowed = governed country-code reference domain

A value can be:

    correctly typed
    inside the technical range
    syntactically valid
    but still outside the business domain

Example:

    status = 'banana'

can be a perfectly valid string while being an invalid status.

### Common domain failures

- Unknown status code.
- Unsupported currency.
- Invalid country code.
- Deprecated product code.
- Misspelled category.
- Source-specific code not mapped to canonical code.
- Empty string accepted as a category.
- New producer value not present in the reference table.
- Historical code incorrectly treated as currently valid.
- Code valid for one product but invalid for another.

### Recognition questions

Before implementing domain validation, ask:

1. What is the authoritative domain?
2. Is the domain static or reference-driven?
3. Is comparison case-sensitive?
4. Is whitespace significant?
5. Are aliases allowed?
6. Are unknown values rejected, quarantined, or mapped?
7. Are deprecated values still valid historically?
8. Does validity depend on another field?
9. Does validity depend on time?
10. Does validity depend on tenant, product, country, or source?
11. How are new domain values introduced?
12. How is the domain versioned?

## 2. Concept and Reasoning

### 2.1 Domain is a business contract

Type validation says:

    status is TEXT

Range validation might say:

    score is between 0 and 100

Domain validation says:

    status must be one of the permitted statuses

These are different contracts.

### 2.2 Finite versus reference domains

A finite static domain can be embedded in code or SQL.

Example:

    status IN ('pending', 'approved', 'rejected')

A larger or changing domain should usually be represented as reference data.

Example:

    supported_currencies

### 2.3 Static domain

Static domains change rarely.

Examples:

- Boolean-like status categories.
- Fixed processing modes.
- Internal event types.

Even static domains should be version-controlled.

### 2.4 Reference domain

Reference domains are stored as data.

Example:

    country_code
    country_name
    active
    effective_from
    effective_to

Reference data allows controlled evolution without embedding every value into transformation code.

### 2.5 Canonical values

Different sources may represent the same concept as:

    Approved
    APPROVED
    approved
    app

Domain validation should normally happen against a canonical representation after explicitly approved normalization.

Do not use normalization to hide unexpected source values.

### 2.6 Normalization versus domain validation

Normalization:

    ' APPROVED ' → 'approved'

Domain validation:

    'approved' ∈ allowed_statuses

Keep the steps observable.

### 2.7 Case sensitivity

Decide whether:

    'EUR'
    'eur'

represent the same domain member.

Do not depend on database collation behavior accidentally.

### 2.8 Whitespace

Leading or trailing whitespace may be a source formatting issue.

Explicitly decide whether:

    'approved '

is normalized or rejected.

### 2.9 Aliases

Aliases should be explicit.

Example:

    source value 'A' → canonical 'approved'

Maintain an auditable mapping rather than scattering CASE statements throughout the pipeline.

### 2.10 Unknown values

When a source sends a value outside the known domain, possible policies are:

    reject
    quarantine
    map to UNKNOWN
    accept as future-compatible

Choose deliberately.

Mapping everything unknown to UNKNOWN can hide source drift.

### 2.11 Unknown is not the same as invalid

An unknown code may be:

- Truly invalid.
- Newly introduced upstream.
- Temporarily unsupported.
- Valid historically but not currently active.

Failure classification should preserve that distinction where it matters.

### 2.12 Domain evolution

Business domains change.

Example:

    pending
    approved
    rejected

later becomes:

    pending
    under_review
    approved
    rejected

Consumers must handle controlled domain evolution.

### 2.13 Additive domain evolution

Adding a valid value can break consumers that assume an exhaustive enum.

Example:

    CASE status
        WHEN 'pending' THEN ...
        WHEN 'approved' THEN ...
        WHEN 'rejected' THEN ...
    END

New values can otherwise produce NULL or unintended behavior.

### 2.14 Removal and deprecation

A domain member may become deprecated without immediately becoming invalid for historical records.

Separate:

    currently allowed
    historically valid
    deprecated
    unknown

### 2.15 Effective-dated domains

A code may be valid only during a specific period.

Example:

    code = 'LEGACY_A'
    effective_from = 2020-01-01
    effective_to = 2025-12-31

Historical records may remain valid even after the code is retired.

### 2.16 Context-dependent domains

Validity can depend on another attribute.

Example:

    payment_method = 'SEPA'
    currency = 'EUR'

while a combination such as:

    payment_method = 'SEPA'
    currency = 'JPY'

may not be allowed by the business contract.

This is a multi-field domain rule.

### 2.17 Tenant-specific domains

Different tenants may support different values.

Example:

    tenant A → products A, B
    tenant B → products B, C

Never validate a tenant-scoped value against an unscoped global domain.

### 2.18 Product-specific domains

Products may support different statuses, currencies, or features.

Domain validation may therefore require a product context.

### 2.19 Source-specific domains

Different upstream systems may use different code sets.

Example:

    CRM: 'A'
    ERP: 'APP'
    canonical: 'approved'

Use a controlled source-to-canonical mapping.

### 2.20 Domain mapping

A mapping table can contain:

    source_system
    source_code
    canonical_code
    active
    effective_from
    effective_to

Mapping is a transformation.

Validation confirms that the resulting canonical value belongs to the target domain.

### 2.21 Domain mapping versus defaulting

Bad:

    unknown code → approved

Defaulting an unknown business value to a valid value can create severe semantic corruption.

### 2.22 Domain validation and NULL

NULL means no value is present.

Domain validation should not automatically treat NULL as an invalid domain member.

Nullability is a separate contract.

### 2.23 Empty strings

An empty string is a string value.

It may be:

    invalid domain member
    missing-value representation

Normalize or classify it according to the data contract.

### 2.24 Sentinel values

Some sources use:

    'N/A'
    'UNKNOWN'
    'NONE'
    '-1'

These are not automatically legitimate domain members.

Document whether they mean missing, unknown, or actual business states.

### 2.25 Numeric domains

Domain validation can apply to discrete numeric values.

Example:

    priority IN (1, 2, 3, 4, 5)

This differs from range validation because values such as 1.5 may be inside the numeric range but not members of the discrete domain.

### 2.26 Composite domains

A domain may be defined by combinations.

Example:

    country + product

or:

    payment_method + currency

Validate the combination rather than validating each field independently.

### 2.27 Domain membership and referential integrity

Reference-table membership resembles referential integrity.

Domain validation focuses on whether a value belongs to an approved business set.

Referential integrity focuses on whether a relationship points to an existing parent entity.

The implementation can overlap, but the business meaning differs.

### 2.28 Domain validation and joins

A lookup join can validate membership.

If no reference row exists:

    DOMAIN_NOT_FOUND

Do not let an unsuccessful lookup silently produce a valid default.

### 2.29 Domain validation and aggregation

Validate categories before aggregation when invalid categories could create misleading groups.

Otherwise an unexpected source value can become a legitimate-looking group in reports.

### 2.30 Domain validation and partitioning

Do not allow uncontrolled domain values to create arbitrary storage partitions.

Validate or govern partition keys before materialization.

### 2.31 Domain validation and security

Some domain values affect authorization or routing.

Examples:

    account_type
    permission_level
    processing_mode

Unknown values must not silently inherit a permissive default.

### 2.32 Domain validation and downstream CASE logic

Every consumer should define behavior for unknown values.

Prefer explicit handling:

    known → business branch
    unknown → controlled failure

### 2.33 Domain validation and schema evolution

A schema can remain structurally unchanged while the domain evolves.

Therefore type checks alone cannot detect new enum values.

### 2.34 Domain contract ownership

Identify who owns each domain:

    business owner
    data owner
    source owner
    platform owner

Do not let the first pipeline developer become the accidental owner of business semantics.

### 2.35 Domain versioning

Record:

    domain_id
    domain_version
    effective_from
    effective_to

This makes historical validation reproducible.

### 2.36 Fail-open versus fail-closed

When the domain reference cannot be loaded, decide explicitly whether to:

    stop
    quarantine
    use a documented cached version

Do not silently accept arbitrary values.

## 3. Implementation

### 3.1 Define the domain contract

Example:

    status
      type: TEXT
      canonicalization: lowercase + trim
      allowed values: pending, approved, rejected
      unknown policy: quarantine

### 3.2 PostgreSQL static domain

    SELECT *
    FROM typed_customers
    WHERE status IN ('pending', 'approved', 'rejected');

Use a failure classification rather than silently dropping the other rows.

### 3.3 Domain failure classification

    CASE
        WHEN status IS NULL THEN NULL
        WHEN status IN ('pending', 'approved', 'rejected') THEN NULL
        ELSE 'UNKNOWN_STATUS'
    END AS domain_failure_code

Nullability remains a separate rule.

### 3.4 Normalize before validating

Example:

    LOWER(BTRIM(status)) AS normalized_status

Then validate:

    normalized_status IN ('pending', 'approved', 'rejected')

Only do this if the contract explicitly allows case and whitespace normalization.

### 3.5 PostgreSQL reference domain

    CREATE TABLE status_domain (
        status_code TEXT PRIMARY KEY,
        active BOOLEAN NOT NULL,
        effective_from TIMESTAMPTZ NOT NULL,
        effective_to TIMESTAMPTZ
    );

Reference data should itself be governed and validated.

### 3.6 Reference membership validation

    SELECT p.record_id, p.status, d.status_code
    FROM typed_payments p
    LEFT JOIN status_domain d
      ON d.status_code = p.status
     AND d.active = TRUE
    WHERE d.status_code IS NULL;

This identifies values without an active domain member.

### 3.7 Effective-dated domain

    SELECT p.record_id, p.status, d.status_code
    FROM typed_payments p
    LEFT JOIN status_domain d
      ON d.status_code = p.status
     AND p.event_time >= d.effective_from
     AND (d.effective_to IS NULL OR p.event_time < d.effective_to);

Use half-open effective intervals to avoid ambiguity at boundaries.

### 3.8 Source-to-canonical mapping

Example:

    CREATE TABLE status_mapping (
        source_system TEXT NOT NULL,
        source_code TEXT NOT NULL,
        canonical_code TEXT NOT NULL,
        effective_from TIMESTAMPTZ NOT NULL,
        effective_to TIMESTAMPTZ,
        PRIMARY KEY (source_system, source_code, effective_from)
    );

Map source values before validating the canonical domain.

### 3.9 Detect unmapped source values

    LEFT JOIN status_mapping m
      ON m.source_system = p.source_system
     AND m.source_code = p.source_status

Rows without a mapping should be explicitly classified.

### 3.10 Composite domain

Example reference:

    CREATE TABLE method_currency_domain (
        payment_method TEXT NOT NULL,
        currency TEXT NOT NULL,
        active BOOLEAN NOT NULL,
        PRIMARY KEY (payment_method, currency)
    );

Validate the pair as a single domain rule.

### 3.11 Tenant-scoped domain

Include tenant identity in the lookup key:

    tenant_id
    product_code

Do not validate tenant-scoped values against global reference data.

### 3.12 Python static domain

    ALLOWED_STATUS = {
        'pending',
        'approved',
        'rejected',
    }

    def validate_status(value):
        if value is None:
            return None
        if value not in ALLOWED_STATUS:
            return 'UNKNOWN_STATUS'
        return None

Keep the validation result separate from the value.

### 3.13 Python canonicalization

    def normalize_status(value):
        if value is None:
            return None
        return str(value).strip().lower()

Normalization should be a documented transformation, not an automatic rescue mechanism.

### 3.14 Python reference lookup

Represent reference data as a keyed structure when the dataset is small enough:

    allowed = {
        'pending',
        'approved',
        'rejected',
    }

Then perform membership checks.

### 3.15 Pandas domain validation

Conceptually:

    allowed = {'pending', 'approved', 'rejected'}
    invalid = ~df['status'].isin(allowed)

Handle missing values explicitly rather than relying on the mask alone.

### 3.16 Validation result table

Use structured failure records:

    CREATE TABLE domain_validation_failure (
        batch_id TEXT NOT NULL,
        record_id TEXT NOT NULL,
        field_name TEXT NOT NULL,
        observed_code TEXT,
        failure_code TEXT NOT NULL,
        domain_id TEXT NOT NULL,
        domain_version TEXT NOT NULL,
        detected_at TIMESTAMPTZ NOT NULL
    );

Avoid storing sensitive raw values unless required.

### 3.17 Valid and invalid outputs

Conceptually:

    typed records
          ↓
    canonicalization
          ↓
    domain validation
       ├── valid → trusted staging
       └── invalid → quarantine

### 3.18 Unknown-value policy

Possible implementation states:

    VALID
    UNKNOWN_CODE
    DEPRECATED_CODE
    UNMAPPED_SOURCE_CODE
    INVALID_COMBINATION
    NULL

Use separate codes when operational response differs.

### 3.19 Database constraints

For stable domains, a CHECK constraint can provide a final invariant:

    CHECK (status IN ('pending', 'approved', 'rejected'))

For frequently changing domains, use governed reference data instead.

### 3.20 Mapping-table constraints

Validate mapping data itself:

    source_system is not NULL
    source_code is not NULL
    canonical_code is not NULL
    no overlapping effective intervals

A broken mapping table can corrupt every downstream record.

### 3.21 Domain completeness

Monitor whether every expected source code has a canonical mapping.

Example reconciliation:

    distinct_source_codes
    mapped_source_codes
    unmapped_source_codes

### 3.22 Controlled fallback

If the business explicitly allows:

    unknown → UNKNOWN

store the original source code separately for traceability.

Never map an unknown value to a concrete business state such as `approved` unless that is explicitly defined.

## 4. Testing

Domain tests must cover membership, normalization, evolution, and context.

### 4.1 Known value

Input:

    approved

Expected:

    valid

### 4.2 Unknown value

Input:

    suspended

when not present in the domain.

Expected:

    UNKNOWN_CODE

### 4.3 Case normalization

Test:

    APPROVED
    Approved
    approved

Verify behavior according to the contract.

### 4.4 Whitespace normalization

Test:

    ' approved '

Verify whether trimming is permitted.

### 4.5 NULL

Test NULL separately.

Expected result should follow the nullability contract.

### 4.6 Empty string

Test empty string.

Verify whether it is treated as missing or as an invalid domain member.

### 4.7 Alias mapping

Test every approved source alias.

Example:

    source = 'APP'
    canonical = 'approved'

Also test an unapproved alias.

### 4.8 Unmapped source code

Provide a source value with no mapping.

Expected:

    UNMAPPED_SOURCE_CODE

not a default business state.

### 4.9 Deprecated code

Test a historically valid but currently inactive code.

Verify behavior for:

    historical event
    current event

### 4.10 Effective-date boundary

Test exactly at:

    effective_from
    effective_to

Verify half-open semantics.

### 4.11 New domain value

Add a new reference value.

Verify the pipeline can intentionally recognize it without changing unrelated values.

### 4.12 Removed domain value

Deactivate a value.

Verify current records fail according to policy while historical records remain reproducible.

### 4.13 Composite domain

Test valid and invalid combinations.

Example:

    SEPA + EUR → valid
    SEPA + unsupported currency → invalid

### 4.14 Tenant-specific domain

Test a value valid for tenant A but invalid for tenant B.

Verify tenant isolation.

### 4.15 Duplicate reference rows

Create duplicate active reference records.

Expected:

    ambiguous-domain failure

rather than nondeterministic selection.

### 4.16 Missing reference data

Remove the domain reference.

Expected:

    explicit reference-data failure

not universal acceptance.

### 4.17 Mapping idempotence

Run the same source value through canonicalization twice.

Expected:

    same canonical value

### 4.18 Domain idempotence

Run the same batch twice.

Expected:

    same valid population
    same invalid population
    no duplicate quarantine records

### 4.19 Reconciliation

Verify:

    input = valid + invalid + explicitly exempt

if the pipeline has an exemption state.

### 4.20 Consumer exhaustiveness

Test downstream CASE or mapping logic with every domain member.

Add a test for unknown values so new domain members do not silently disappear.

## 5. Observability

### Core metrics

| Metric | Meaning |
|---|---|
| `domain_validation_input_rows` | Records entering domain validation |
| `domain_validation_valid_rows` | Records with valid domain values |
| `domain_validation_invalid_rows` | Records violating domain rules |
| `domain_validation_unknown_codes` | Values not recognized |
| `domain_validation_unmapped_codes` | Source values without mappings |
| `domain_validation_deprecated_codes` | Deprecated values encountered |
| `domain_validation_invalid_combinations` | Context-dependent domain failures |
| `domain_validation_missing_reference` | Reference data unavailable or unmatched |
| `domain_validation_ambiguous_reference` | Multiple applicable domain rows |
| `domain_validation_domain_version` | Active domain version |

### Failure distribution

Break failures down by:

- Field.
- Source system.
- Source code.
- Canonical code.
- Tenant.
- Product.
- Country.
- Domain.
- Domain version.
- Batch.

### New-value detection

Track newly observed values not previously seen in the source.

A new value is not automatically invalid, but it is an important schema/domain-evolution signal.

### Mapping coverage

Track:

    mapped_source_codes / observed_source_codes

A decline can indicate upstream code-set changes.

### Domain cardinality

Monitor the number of distinct values.

Unexpected cardinality growth can indicate:

- Source corruption.
- New business states.
- Free-text leakage.
- Normalization failures.

### Reference-data health

Monitor:

- Duplicate active values.
- Missing mappings.
- Overlapping effective dates.
- Gaps in effective dates.
- Invalid canonical codes.

### Quarantine backlog

Track:

- Invalid count.
- Oldest failure.
- Failure rate.
- Retry count.
- Unmapped-code backlog.

## 6. Intentional Failure

### Failure 1 — Unknown value mapped to approved

Map an unrecognized status to `approved`.

Expected symptom:

- Semantic corruption.

Recovery:

Route unknown values to quarantine or an explicitly defined UNKNOWN state.

### Failure 2 — Case-sensitive comparison

Treat `APPROVED` and `approved` as different without a contract.

Expected symptom:

- False domain failures.

Recovery:

Define canonicalization policy.

### Failure 3 — Silent trimming

Normalize unexpected values without tracking the transformation.

Expected symptom:

- Source formatting drift becomes invisible.

Recovery:

Record normalization metrics and preserve source representation when required.

### Failure 4 — Missing reference data means pass

Allow unmatched lookup rows to bypass validation.

Expected symptom:

- Unknown values enter trusted datasets.

Recovery:

Classify missing reference data explicitly.

### Failure 5 — Overlapping effective rules

Create two active domain records for the same value and time.

Expected symptom:

- Nondeterministic or multiplied matches.

Recovery:

Quarantine ambiguity and repair the reference domain.

### Failure 6 — Historical code rejected

Validate historical data only against the current active domain.

Expected symptom:

- Previously valid records fail after domain retirement.

Recovery:

Use effective-dated validation.

### Failure 7 — New enum breaks downstream CASE

Introduce a new valid domain value without updating a consumer.

Expected symptom:

- NULL output or incorrect fallback.

Recovery:

Test consumer exhaustiveness and define explicit unknown behavior.

### Failure 8 — Tenant isolation failure

Validate a tenant-specific code against another tenant's domain.

Expected symptom:

- Values pass or fail incorrectly across tenants.

Recovery:

Include tenant scope in the domain key.

### Failure 9 — Duplicate mapping

Create two active source-to-canonical mappings for the same source code.

Expected symptom:

- One input produces multiple canonical rows.

Recovery:

Enforce uniqueness and effective-date integrity.

## 7. Recovery

### Recovery sequence

1. Preserve the original source value.
2. Identify the domain and version.
3. Identify the source system.
4. Determine whether the value is new, deprecated, unmapped, or invalid.
5. Check reference-data health.
6. Check effective dates.
7. Check tenant/product context.
8. Correct the mapping or domain if appropriate.
9. Revalidate affected records.
10. Reconcile valid and invalid populations.
11. Replay the affected scope.
12. Add regression coverage.

### Recovering from a new upstream code

1. Confirm the business meaning with the domain owner.
2. Add the canonical domain member if approved.
3. Add source mapping.
4. Version the domain.
5. Test downstream consumers.
6. Replay quarantined records.

### Recovering from an invalid mapping

1. Identify records mapped incorrectly.
2. Recover original source values.
3. Correct the mapping effective date.
4. Reprocess affected records.
5. Recalculate downstream outputs if necessary.

### Recovering from domain retirement

Do not rewrite historical records automatically.

Use effective dates to distinguish:

    historically valid
    currently valid
    future valid

### Recovering from reference outage

Depending on the contract:

    stop publication
    quarantine
    use a verified cached reference version

Never replace a missing reference domain with unrestricted acceptance.

## 8. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL supports CHECK constraints, lookup tables, foreign keys, unique constraints, and SQL membership tests.

Understand the relational model first; the database should enforce stable domain invariants where practical.

### 2. dbt

dbt can express accepted-value tests, relationships, and model-level domain contracts.

Use tests to make domain assumptions executable and reviewable.

### 3. Great Expectations

Great Expectations supports value-set expectations and configurable validation outcomes.

Use it to operationalize controlled domains without hiding the underlying business rule.

## 9. Production Runbook

### Before deployment

- [ ] Identify domain owner.
- [ ] Define canonical values.
- [ ] Define normalization rules.
- [ ] Define aliases.
- [ ] Define unknown-value policy.
- [ ] Define NULL policy.
- [ ] Define effective dates.
- [ ] Define tenant/product scope.
- [ ] Define source mappings.
- [ ] Define domain versioning.
- [ ] Define quarantine behavior.
- [ ] Test downstream consumers.

### During execution

- [ ] Record input count.
- [ ] Record valid count.
- [ ] Record invalid count.
- [ ] Record unknown values.
- [ ] Record unmapped values.
- [ ] Record deprecated values.
- [ ] Record ambiguous reference matches.
- [ ] Record domain version.
- [ ] Reconcile populations.

### If unknown-code rate spikes

1. Identify new source values.
2. Compare source release changes.
3. Check mapping coverage.
4. Check domain version.
5. Confirm business meaning.
6. Add approved values or quarantine them.

### If domain cardinality grows unexpectedly

1. Inspect distinct values.
2. Check normalization.
3. Check source-code changes.
4. Check free-text leakage.
5. Check mapping logic.

### If a domain value is retired

1. Set effective end date.
2. Verify historical behavior.
3. Test current consumers.
4. Monitor current usage.
5. Preserve historical reproducibility.

## 10. Common Mistakes

### Mistake 1 — Treating type validity as domain validity

TEXT does not mean every string is an accepted business value.

### Mistake 2 — Defaulting unknown values

Unknown business states should not become a valid state by convenience.

### Mistake 3 — Hardcoding dynamic domains

Frequently changing domains belong in governed reference data.

### Mistake 4 — No normalization contract

Case and whitespace handling must be explicit.

### Mistake 5 — No effective dating

Retired values may remain historically valid.

### Mistake 6 — Ignoring source mappings

Source codes often differ from canonical business codes.

### Mistake 7 — Ignoring context

Tenant, product, country, or currency can change domain validity.

### Mistake 8 — Missing reference data equals valid

An unavailable domain cannot safely mean unrestricted acceptance.

### Mistake 9 — No consumer exhaustiveness tests

New valid values can break downstream CASE logic.

### Mistake 10 — Mixing NULL with unknown

NULL means no value; unknown means a value exists but is not recognized.

### Mistake 11 — Silent aliasing

Every source-to-canonical mapping should be explicit and auditable.

### Mistake 12 — No domain ownership

Business semantics need an accountable owner.

## 11. Definition of Done

The domain-validation stage is complete when you can:

- [ ] Define static and reference-driven domains.
- [ ] Separate type, range, and domain validation.
- [ ] Define canonical values.
- [ ] Define case and whitespace policy.
- [ ] Implement explicit aliases.
- [ ] Detect unknown and unmapped codes.
- [ ] Distinguish unknown, deprecated, and invalid values.
- [ ] Validate effective-dated domains.
- [ ] Validate composite domains.
- [ ] Validate tenant- and product-scoped domains.
- [ ] Build source-to-canonical mappings.
- [ ] Detect duplicate and overlapping mappings.
- [ ] Handle NULL and sentinel values deliberately.
- [ ] Preserve raw source representations when needed.
- [ ] Quarantine invalid domain values.
- [ ] Monitor domain cardinality and new-value drift.
- [ ] Reconcile mapped and unmapped populations.
- [ ] Version domain rules.
- [ ] Intentionally break domain validation.
- [ ] Recover from source-code and reference-data changes.
- [ ] Explain how PostgreSQL, dbt, and Great Expectations support production domain validation.

## 12. What You Learned

Domain validation protects trusted data from values that are technically valid but not permitted by the business contract.

The production workflow is:

    DEFINE DOMAIN
         ↓
    CANONICALIZE WHERE ALLOWED
         ↓
    APPLY CONTEXT / EFFECTIVE-DATE RULES
         ↓
    VALIDATE MEMBERSHIP
         ↓
    CLASSIFY UNKNOWN / DEPRECATED / INVALID
         ↓
    QUARANTINE UNTRUSTED VALUES
         ↓
    PUBLISH VALID CANONICAL DATA
         ↓
    MONITOR DOMAIN EVOLUTION

> **A string can be perfectly valid data syntactically and still be invalid business data; domain validation makes the permitted value space explicit, governed, versioned, and observable.**

### Next recipe

**T37 — Referential Integrity**