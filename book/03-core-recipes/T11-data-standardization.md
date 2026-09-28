# T11 — Data Standardization

> **Goal:** Align equivalent data from different sources to a governed canonical representation without losing source meaning, units, provenance, or auditability.

Data standardization is the point where independently designed source systems are made compatible with a shared downstream contract.

Example:

    CRM:        United States
    Billing:    USA
    ERP:        US
    Canonical:  US

Or:

    Source A:  1000 g
    Source B:  1 kg
    Canonical: 1 kg

The central rule is:

> **Standardize representations to a declared canonical standard; do not silently invent equivalence.**

---

## 1. Problem Recognition

### 1.1 Typical standardization problem

Imagine three systems provide customer data:

    CRM:       country = "United States"
    Payments:  country = "US"
    Support:   country = "USA"

Without standardization, joins, aggregations, validation, and reporting can treat these as different values.

A second source may report:

    weight = 2.2 lb

while another reports:

    weight = 1 kg

These values cannot be combined safely until the unit convention is explicit.

### 1.2 Standardization is not normalization

Normalization changes the representation of a value within a field:

    "  US  " → "US"

Standardization aligns representations with a shared external or domain standard:

    "United States" → "US"
    "USA" → "US"

T05 teaches string normalization. T10 teaches code/status mapping. T11 focuses on **cross-source alignment to a governed standard**.

### 1.3 Red flags

Investigate when:

- the same business value appears under several names
- joins fail because sources use different representations
- units differ between source systems
- country, currency, language, or category values are inconsistent
- analysts repeatedly maintain source-specific mappings
- two systems disagree about spelling or abbreviations
- a standard changes and historical data becomes ambiguous
- an ETL job performs conversions without recording the source unit or standard version.

---

## 2. Concept and Reasoning

### 2.1 Define the canonical standard first

Do not start by writing transformation code.

First define:

    source representation
    canonical representation
    allowed canonical values
    conversion rules
    unknown-value policy
    provenance requirements
    version/effective-date policy

Example:

| Field | Canonical standard | Example |
|---|---|---|
| country | ISO 3166-1 alpha-2 | `US` |
| currency | ISO 4217 alphabetic | `USD` |
| language | ISO 639-1 | `en` |
| weight | kilograms | `1.5` |

The exact standard is a data-contract decision, not a coding convenience.

### 2.2 Equivalent representation versus different meaning

These may be equivalent:

    USA → US
    United States → US

But similar-looking values may not be equivalent:

    UK → United Kingdom
    GB → Great Britain / United Kingdom code context

The transformation must follow the chosen standard and source contract rather than guessing from text.

### 2.3 Preserve source evidence

A standardized record should often retain:

    source_value = "USA"
    canonical_value = "US"
    standard = "ISO_3166_1_ALPHA_2"
    standard_version = "..."

For unit conversion, preserve the source unit when it matters:

    source_value = 2204.62
    source_unit = "lb"
    canonical_value = 1000
    canonical_unit = "kg"

This makes transformations explainable and reversible where required.

### 2.4 Standardization versus mapping

Mapping answers:

    "P" → "PENDING"

Standardization answers:

    "USA" → "US"
    "United States" → "US"

These mechanisms can overlap. A governed reference table is often appropriate for both, but their business purpose should remain explicit.

### 2.5 Units require dimensional reasoning

A numeric value without a unit is incomplete.

    100

could mean:

    100 kg
    100 g
    100 lb
    100 USD

Never standardize a number before establishing its dimension and source unit.

Use:

    canonical_value = convert(source_value, source_unit, canonical_unit)

not:

    canonical_value = source_value * guessed_factor

### 2.6 Categories need governed vocabularies

Suppose sources contain:

    ecommerce:  electronics
    ERP:        ELEC
    marketplace: consumer-electronics

A canonical category may be:

    ELECTRONICS

But only if the business has explicitly defined those values as equivalent.

Do not collapse distinct categories merely because they look similar.

### 2.7 Reference data is often the control plane

For governed standards, store reference data centrally.

Example:

    standard_name
    source_system
    source_value
    canonical_value
    effective_from
    effective_to
    standard_version
    owner
    is_active

This is preferable to copying mappings into many pipelines.

### 2.8 Effective dates matter

Standards and reference mappings can change.

Historical data may need the standard that was valid when the source event occurred.

Prefer:

    source_value + event_time → canonical_value

when the business rule is time-dependent.

Do not automatically apply today's mapping to a historical record.

### 2.9 Avoid over-standardization

Not every field should be converted to a common representation.

Examples that often require preservation:

- legal names
- free-text descriptions
- source identifiers
- opaque external IDs
- signed payloads
- hashes
- document references.

Standardization is useful only when the canonical form preserves the field's intended meaning.

### 2.10 Idempotence

A good standardization transformation should normally be idempotent:

    standardize(standardize(x)) = standardize(x)

Example:

    USA → US → US

A second pipeline run should not convert an already canonical value again.

---

## 3. Implementation

## 3.1 Define a standardization contract

Example:

| Source | Field | Source values | Canonical standard | Unknown policy |
|---|---|---|---|---|
| crm | country | `USA`, `United States` | ISO alpha-2 | quarantine |
| billing | country | `US`, `USA` | ISO alpha-2 | quarantine |
| warehouse | weight | `g`, `kg`, `lb` | kilograms | reject |

Also define:

- owner
- source system
- canonical type
- unit
- reference-data version
- effective dates
- rounding policy
- whether raw evidence must be retained.

## 3.2 Simple categorical standardization

    COUNTRY_MAP = {
        "US": "US",
        "USA": "US",
        "United States": "US",
        "GB": "GB",
        "UK": "GB",
        "United Kingdom": "GB",
    }

    def standardize_country(value: str) -> str:
        try:
            return COUNTRY_MAP[value]
        except KeyError as exc:
            raise ValueError(f"unknown country representation: {value!r}") from exc

Keep the mapping explicit. Do not silently return an arbitrary default.

## 3.3 Normalize before standardizing when appropriate

If the source contract permits whitespace/case cleanup:

    def clean_token(value: str) -> str:
        return " ".join(value.strip().split())

    def standardize_country(value: str) -> str:
        cleaned = clean_token(value)
        key = cleaned.casefold()

        mapping = {
            "us": "US",
            "usa": "US",
            "united states": "US",
        }

        try:
            return mapping[key]
        except KeyError as exc:
            raise ValueError(f"unknown country representation: {value!r}") from exc

Do not use this pattern when case or whitespace is semantically significant.

## 3.4 Preserve provenance

    def standardize_country_record(source_value: str) -> dict:
        canonical = standardize_country(source_value)

        return {
            "source_country": source_value,
            "canonical_country": canonical,
            "standard_name": "ISO_3166_1_ALPHA_2",
        }

## 3.5 Unit conversion

For a controlled example, kilograms can be used as the canonical unit:

    from decimal import Decimal

    LB_TO_KG = Decimal("0.45359237")
    G_TO_KG = Decimal("0.001")

    def to_kg(value: Decimal, unit: str) -> Decimal:
        if unit == "kg":
            return value
        if unit == "g":
            return value * G_TO_KG
        if unit == "lb":
            return value * LB_TO_KG
        raise ValueError(f"unsupported weight unit: {unit!r}")

Use Decimal when the domain requires exact decimal arithmetic and an explicit precision policy.

## 3.6 Standardization with explicit unknown policy

    def standardize(value: str, mapping: dict[str, str], unknown_policy: str = "reject") -> str | None:
        if value in mapping:
            return mapping[value]

        if unknown_policy == "reject":
            raise ValueError(f"unknown value: {value!r}")
        if unknown_policy == "null":
            return None
        if unknown_policy == "quarantine":
            raise RuntimeError(f"quarantine value: {value!r}")

        raise ValueError(f"unsupported unknown policy: {unknown_policy!r}")

Do not use `mapping.get(value, canonical_default)` unless the default is explicitly part of the contract.

## 3.7 PostgreSQL reference table

    CREATE TABLE standardization_reference (
        source_system text NOT NULL,
        field_name text NOT NULL,
        source_value text NOT NULL,
        canonical_value text NOT NULL,
        standard_name text NOT NULL,
        standard_version text NOT NULL,
        effective_from timestamptz NOT NULL,
        effective_to timestamptz,
        PRIMARY KEY (source_system, field_name, source_value, effective_from),
        CHECK (effective_to IS NULL OR effective_to > effective_from)
    );

Effective-dated lookup:

    SELECT canonical_value, standard_name, standard_version
    FROM standardization_reference
    WHERE source_system = $1
      AND field_name = $2
      AND source_value = $3
      AND effective_from <= $4
      AND (effective_to IS NULL OR $4 < effective_to)
    ORDER BY effective_from DESC
    LIMIT 1;

Use half-open intervals:

    [effective_from, effective_to)

This avoids ambiguity at exact boundary timestamps.

## 3.8 Prevent overlapping reference rules

Two active rules for the same source value and overlapping effective periods can make a pipeline nondeterministic.

At minimum, validate overlaps during deployment or data-quality checks.

For PostgreSQL, an exclusion constraint can enforce non-overlapping time ranges when the schema and PostgreSQL range types are appropriate:

    CREATE EXTENSION IF NOT EXISTS btree_gist;

    ALTER TABLE standardization_reference
    ADD CONSTRAINT no_overlapping_standardization_rules
    EXCLUDE USING gist (
        source_system WITH =,
        field_name WITH =,
        source_value WITH =,
        tstzrange(effective_from, effective_to, '[)') WITH &&
    );

Treat an open-ended `effective_to` as an unbounded range in the production implementation.

## 3.9 Validate the canonical output

After standardization, validate against the canonical domain.

    ALLOWED_COUNTRIES = {"US", "GB", "DE", "FR"}

    def validate_country(value: str) -> str:
        if value not in ALLOWED_COUNTRIES:
            raise ValueError(f"invalid canonical country: {value!r}")
        return value

The pipeline should fail or quarantine when the canonical contract is violated.

---

## 4. Testing

Standardization needs tests for both **equivalence** and **non-equivalence**.

### 4.1 Known representations

    def test_country_standardization():
        assert standardize_country("USA") == "US"
        assert standardize_country("United States") == "US"
        assert standardize_country("US") == "US"

### 4.2 Unknown values

    import pytest

    def test_unknown_country_is_rejected():
        with pytest.raises(ValueError):
            standardize_country("Atlantis")

### 4.3 Idempotence

    def test_standardization_is_idempotent():
        value = standardize_country("USA")
        assert standardize_country(value) == value

### 4.4 Unit conversion

    def test_weight_conversion():
        assert to_kg(Decimal("1000"), "g") == Decimal("1.000")

### 4.5 Boundary tests

Test:

- zero
- negative values where permitted
- maximum/minimum domain values
- exact effective-from timestamp
- exact effective-to timestamp
- unknown unit
- unknown category
- null/missing values
- duplicate reference rules.

### 4.6 Reference-data completeness

For every required source value, prove that exactly one applicable rule exists for the event time.

Conceptually:

    required source values
             ↓
    reference table
             ↓
    exactly one match

Zero matches should be observable and handled according to policy.

### 4.7 Non-equivalence tests

Do not only test what should map together.

Test that genuinely different values remain different:

    assert standardize_country("US") != "GB"

For categories, explicitly test values that must not collapse into the same canonical category.

---

## 5. Observability

At minimum, monitor:

| Signal | Why it matters |
|---|---|
| standardization count | Shows transformation volume |
| unchanged/canonical count | Detects already-standardized input |
| mapped count by source | Detects source-specific drift |
| unknown value count | Detects contract changes |
| unknown value rate | Detects sudden source changes |
| conversion count by unit | Detects unit mix changes |
| reference-data misses | Detects incomplete mappings |
| reference-data overlaps | Detects nondeterministic rules |
| standard version | Explains which standard was applied |
| collision count | Detects distinct source values collapsing unexpectedly |

Useful structured fields:

    pipeline_run_id
    source_system
    field_name
    source_value
    canonical_value
    standard_name
    standard_version
    event_time
    outcome

Do not log sensitive source values merely for observability. Prefer safe identifiers, hashes where appropriate, or controlled samples.

---

## 6. Intentional Failure

Break the pipeline deliberately and diagnose the evidence.

### Failure A — Wrong unit

Send:

    value = 1000
    unit = "lb"

while the producer actually meant grams.

Expected result:

- conversion technically succeeds
- business result is wrong
- reconciliation or range checks should expose the problem.

Lesson: a mathematically valid conversion can still be semantically wrong.

### Failure B — Unknown representation

Send:

    country = "U.S.A."

without a defined mapping.

Expected result:

- standardization fails or quarantines
- unknown-value metric increases
- source evidence remains available.

### Failure C — Double conversion

Apply kilograms-to-kilograms conversion as though the value were still in grams.

Expected result:

- magnitude becomes incorrect
- range or reconciliation checks detect the anomaly.

Lesson: every transformed value must have an explicit unit state.

### Failure D — Overlapping reference rules

Create two mappings for the same source value with overlapping effective periods.

Expected result:

- deployment/data-quality validation fails
- the pipeline does not choose arbitrarily.

### Failure E — Lost source evidence

Drop the source representation after conversion.

Expected result:

- downstream result may still look correct
- incident investigation becomes harder
- reprocessing may become impossible.

Lesson: correctness includes explainability when provenance is required.

---

## 7. Recovery

### Scenario 1 — New source representation

1. Stop or quarantine affected records if the value cannot be safely interpreted.
2. Identify the upstream contract change.
3. Decide whether the new representation is genuinely equivalent.
4. Add the governed reference mapping.
5. Version the mapping if required.
6. Test known and unknown values.
7. Replay quarantined records.
8. Reconcile output counts and canonical distributions.

### Scenario 2 — Wrong unit configuration

1. Stop downstream publication if incorrect values have escaped.
2. Identify the first affected run.
3. Determine the correct source unit.
4. Preserve the incorrect output for audit if required.
5. Correct the transformation configuration.
6. Reprocess affected raw/staged records.
7. Reconcile totals and ranges.
8. Document the configuration change.

### Scenario 3 — Incorrect reference mapping

1. Identify affected source values and time range.
2. Determine which mapping version produced them.
3. Correct the reference data through the governed process.
4. Do not silently overwrite historical evidence.
5. Replay affected records using the intended mapping version.
6. Compare old and corrected canonical distributions.
7. Record the incident and rule change.

---

## 8. Production Tools You Should Know

### 8.1 PostgreSQL reference tables

Useful for centrally governed mappings, effective dates, constraints, ownership metadata, and auditable reference data.

The underlying mechanism remains the same: explicit source-to-canonical rules plus validation.

### 8.2 Python

Useful for source-specific adapters, unit conversion, deterministic transformation functions, and automated tests.

Python should implement declared rules rather than become an undocumented mapping store.

### 8.3 dbt

Useful for expressing standardization transformations as version-controlled SQL models with tests and documentation.

Typical pattern:

    raw source
       ↓
    standardized model
       ↓
    canonical downstream model

Use dbt tests to prove canonical constraints and reference-data expectations.

---

## 9. Production Runbook

### Before deployment

- [ ] Canonical standard is explicitly documented.
- [ ] Source representations are inventoried.
- [ ] Units are explicitly defined.
- [ ] Unknown-value policy is documented.
- [ ] Reference data has an owner.
- [ ] Effective-date behavior is defined where required.
- [ ] Raw source evidence is retained where required.
- [ ] Canonical output constraints are tested.

### During execution

- [ ] Monitor unknown-value rate.
- [ ] Monitor reference-data misses.
- [ ] Monitor conversion counts by unit.
- [ ] Monitor canonical distribution changes.
- [ ] Monitor standard/reference version.
- [ ] Track pipeline run and source system.

### When standardization fails

1. Identify the affected source and field.
2. Inspect the source representation.
3. Determine whether it is unknown, invalid, or genuinely new.
4. Check the active reference rule and version.
5. Check the source unit and canonical unit.
6. Inspect the first affected event/run.
7. Quarantine or stop publication if correctness is uncertain.
8. Correct the rule or upstream contract.
9. Replay affected data.
10. Reconcile before reopening downstream publication.

---

## 10. Common Mistakes

### Mistake 1 — Treating normalization as standardization

Trimming or lowercasing a value does not establish its canonical business meaning.

### Mistake 2 — Converting numbers without units

    1000 → 1

is meaningless unless the source and target units are known.

### Mistake 3 — Using arbitrary defaults

Unknown source values should not silently become a canonical value just to keep the pipeline green.

### Mistake 4 — Hard-coding mappings everywhere

Duplicated mappings drift across pipelines.

### Mistake 5 — Dropping source evidence

Once the source representation is lost, explaining or correcting a transformation becomes harder.

### Mistake 6 — Ignoring time-dependent standards

A historical record can be misclassified when today's reference data is applied to yesterday's event.

### Mistake 7 — Double standardization

An already canonical value must not be converted again.

### Mistake 8 — Over-standardizing free text

Do not force names, descriptions, or legal text into a canonical vocabulary unless the domain explicitly requires it.

---

## 11. Definition of Done

A production-grade standardization implementation is complete when you can:

- [ ] Define the canonical standard.
- [ ] Explain the difference between normalization, mapping, and standardization.
- [ ] Identify source-specific representations.
- [ ] Define explicit unit and conversion rules.
- [ ] Maintain governed reference data.
- [ ] Handle unknown values without guessing.
- [ ] Preserve required source evidence.
- [ ] Implement deterministic transformations.
- [ ] Prove idempotence where required.
- [ ] Test equivalent and non-equivalent values.
- [ ] Test boundary and effective-date behavior.
- [ ] Detect overlapping reference rules.
- [ ] Observe unknowns, conversions, misses, and collisions.
- [ ] Intentionally break the implementation.
- [ ] Diagnose the failure from evidence.
- [ ] Recover and replay affected records safely.
- [ ] Explain the relevant production tools.
- [ ] Operate the mechanism using the runbook.

---

## 12. What You Learned

Data standardization is not cosmetic formatting.

It is the controlled process of making independently produced data compatible with a shared canonical contract.

The important reasoning pattern is:

    source representation
            ↓
    explicit standard
            ↓
    governed reference/conversion rule
            ↓
    canonical value
            ↓
    validation
            ↓
    observable output

When the standard is wrong, the pipeline can produce perfectly valid-looking data that is semantically incorrect. Production-grade standardization therefore requires explicit standards, units, reference data, provenance, tests, observability, and safe recovery.

---

## Next Recipe

**T12 — Data Cleansing**