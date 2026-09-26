# E57 — XML Extraction — ETL Application

## 1. Problem Recognition

XML remains common in enterprise ETL: payments, banking, healthcare, logistics, ERP, SOAP, government, and regulatory integrations.

XML extraction must handle elements, attributes, namespaces, repeated elements, optional fields, schema versions, malformed documents, large files, and source lineage.

Production flow:

```
SOURCE XML
    |
    v
VERIFY + PARSE
    |
    v
HANDLE NAMESPACES
    |
    v
LOCATE RECORDS
    |
    v
VALIDATE
    |
    v
NORMALIZE + LINEAGE
    |
    v
IDEMPOTENT STAGE
    |
    v
OBSERVE + RECOVER
```

Do not flatten XML blindly. The extractor should preserve source meaning and deterministic lineage.

## 2. XML Is a Tree

Example:

```
<payments>
  <payment>
    <id>P001</id>
    <amount>100</amount>
    <currency>EUR</currency>
  </payment>
  <payment>
    <id>P002</id>
    <amount>200</amount>
    <currency>USD</currency>
  </payment>
</payments>
```

The document has a root, child elements, and repeated record elements. Extraction paths must follow that structure.

## 3. Elements and Attributes

XML fields can be elements:

```
<payment><id>P001</id></payment>
```

or attributes:

```
<payment id="P001" status="completed" />
```

or both. Define the mapping in the source contract instead of guessing.

## 4. Basic Parsing

Python ElementTree handles ordinary XML:

```
import xml.etree.ElementTree as ET

tree = ET.parse("payments.xml")
root = tree.getroot()

print(root.tag)
```

For bounded documents this is straightforward. For very large documents, use incremental parsing.

## 5. Parsing XML Text

For XML already held as bytes or a small test string:

```
root = ET.fromstring(xml_bytes)
```

Prefer bytes when the XML declaration contains encoding information. Do not decode bytes using an assumed encoding unless the contract requires it.

## 6. Root Validation

Well-formed XML is not necessarily the expected document.

```
if root.tag != "payments":
    raise ValueError(f"Unexpected root: {root.tag}")
```

For namespaced XML, validate the namespace URI as well.

## 7. Finding Records

Simple XML:

```
for payment in root.findall("payment"):
    payment_id = payment.findtext("id")
    print(payment_id)
```

Nested records can use an explicit path:

```
for payment in root.findall("./batch/records/payment"):
    payment_id = payment.findtext("id")
```

## 8. Element Text

XML text is initially a string.

```
amount_text = payment.findtext("amount")

if amount_text is None:
    raise ValueError("Missing amount")

amount_text = amount_text.strip()
```

Convert it deliberately after validating the field contract.

## 9. Decimal and Timestamp Fields

For financial values use Decimal when exact decimal semantics are required:

```
from decimal import Decimal

amount = Decimal(amount_text)
```

XML timestamps are strings. Parse them explicitly and preserve timezone information.

```
from datetime import datetime

occurred_at = datetime.fromisoformat(
    timestamp_text.replace("Z", "+00:00")
)
```

Do not silently convert malformed values to zero or a local timezone.

## 10. Attributes

```
payment_id = payment.get("id")
status = payment.get("status")

if payment_id is None:
    raise ValueError("Missing payment id attribute")
```

Required attributes must be validated explicitly.

## 11. Missing and Empty Values

These can have different meanings:

```
<description />
<description></description>
<payment></payment>
```

Define whether each maps to missing, null, empty string, or empty collection. Do not apply one global rule without checking the source contract.

## 12. XML Namespaces

Namespaces are a major source of extraction failures.

```
<p:payments xmlns:p="urn:example:payments">
  <p:payment>
    <p:id>P001</p:id>
  </p:payment>
</p:payments>
```

ElementTree represents the tag using the namespace URI. The producer prefix is not the identity.

## 13. Namespace-Safe Extraction

Use the namespace URI explicitly:

```
namespace = "urn:example:payments"
payment_tag = "{" + namespace + "}payment"
id_tag = "{" + namespace + "}id"

for payment in root.iter(payment_tag):
    payment_id = payment.findtext(id_tag)
```

The following prefixes are equivalent when they map to the same URI:

```
<p:payment xmlns:p="urn:example:payments" />
<x:payment xmlns:x="urn:example:payments" />
```

Build selectors from the URI, not the prefix.

## 14. Default Namespaces

An XML document can have a default namespace:

```
<payments xmlns="urn:example:payments">
  <payment><id>P001</id></payment>
</payments>
```

These elements are still namespace-qualified. Treat the URI as part of the contract.

## 15. Parent Context

Nested records may depend on parent identity:

```
<account id="A001">
  <transactions>
    <transaction id="T001" />
  </transactions>
</account>
```

When producing the transaction record, preserve account_id. Do not lose parent context during flattening.

## 16. Repeated Elements

Example:

```
<payment>
  <tag>online</tag>
  <tag>priority</tag>
</payment>
```

Use findall when multiple values are expected:

```
tags = [tag.text for tag in payment.findall("tag")]
```

Define whether repeated values become an array, child rows, or another explicit model.

## 17. Mixed Content

XML can contain text around child elements. For example:

```
<description>Paid by <customer>Alice</customer> yesterday.</description>
```

Simple text extraction may not preserve all meaning. Define whether the target needs direct text, combined text, child structure, or the original XML.

## 18. XPath-Style Selection

ElementTree supports a useful subset of XPath:

```
for payment in root.findall("./batch/records/payment"):
    process(payment)
```

Use the simplest selector that correctly expresses the source contract. More advanced XPath requirements may justify lxml.

## 19. Malformed XML

Malformed XML is a document-level failure.

```
try:
    tree = ET.parse("payments.xml")
except ET.ParseError as exc:
    raise ValueError(f"Malformed XML: {exc}") from exc
```

Do not report successful extraction after a parser failure.

## 20. XML Schema Validation

Well-formedness and schema validity are different.

Well-formedness asks whether the XML syntax is valid. XSD validation asks whether the document conforms to the expected structural contract.

For regulated or tightly controlled feeds, validate against the producer's XSD when required.

## 21. Document vs Record Validation

Document checks may include:

- expected root;
- expected namespace;
- batch identifier;
- schema version;
- required envelope.

Record checks may include:

- required identifier;
- amount;
- currency;
- timestamp;
- allowed values.

Do not assume document validity guarantees business validity.

## 22. XML Security

XML is an input boundary. Untrusted XML can create parser security problems involving external entities, entity expansion, resource exhaustion, and oversized documents.

Use hardened parser settings or a security-focused XML parser for untrusted sources. Do not enable external entity processing merely because a producer asks for it without understanding the consequence.

Never execute XML values as code.

## 23. Large XML Documents

ET.parse builds an in-memory tree. That may be inappropriate for multi-gigabyte documents.

Use iterparse for repeated records:

```
for event, element in ET.iterparse(
    "large-payments.xml",
    events=("end",),
):
    if element.tag == "payment":
        process_payment(element)
        element.clear()
```

Clearing processed elements prevents the tree from retaining unnecessary content.

## 24. Namespace-Aware Streaming

```
namespace = "urn:example:payments"
payment_tag = "{" + namespace + "}payment"
id_tag = "{" + namespace + "}id"

for event, element in ET.iterparse(
    "payments.xml",
    events=("end",),
):
    if element.tag != payment_tag:
        continue

    payment_id = element.findtext(id_tag)
    yield {"payment_id": payment_id}
    element.clear()
```

This pattern works without building the complete XML tree.

## 25. Streaming Parent State

When records are nested under parents, capture the parent context explicitly before its element is cleared.

```
current_account_id = None

for event, element in ET.iterparse(
    "accounts.xml",
    events=("start", "end"),
):
    if event == "start" and element.tag == "account":
        current_account_id = element.get("id")

    if event == "end" and element.tag == "transaction":
        yield {
            "account_id": current_account_id,
            "transaction_id": element.get("id"),
        }
        element.clear()
```

For complex hierarchies, design the parser as an explicit state machine rather than relying on accidental tree retention.

## 26. Source Identity

Calculate a stable source hash:

```
import hashlib

def sha256_file(path):
    digest = hashlib.sha256()
    with path.open("rb") as source:
        for chunk in iter(
            lambda: source.read(1024 * 1024),
            b"",
        ):
            digest.update(chunk)
    return digest.hexdigest()
```

Use source identity to make replay deterministic.

## 27. Record Identity

Separate technical identity from business identity.

Technical identity can be:

```
(source_hash, record_number)
```

Business identity might be payment_id or transaction_id.

Do not generate a random identifier as the only identity during extraction if replay safety matters.

## 28. Idempotent Staging

Example PostgreSQL table:

```
CREATE TABLE xml_staging (
    source_hash TEXT NOT NULL,
    record_number BIGINT NOT NULL,
    record_id TEXT,
    payload JSONB NOT NULL,
    extracted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (source_hash, record_number)
);
```

Idempotent insert:

```
INSERT INTO xml_staging (
    source_hash, record_number, record_id, payload
)
VALUES (%s, %s, %s, %s)
ON CONFLICT (source_hash, record_number)
DO NOTHING;
```

A replay can therefore be safe.

## 29. Raw XML vs Normalized Data

Three strategies exist:

1. preserve raw XML and transform later;
2. normalize during extraction;
3. preserve raw source plus normalized staging.

For important feeds, source retention plus structured staging often gives the strongest replay and debugging capability, subject to retention and privacy requirements.

## 30. Do Not Convert XML to JSON Blindly

Automatic XML-to-JSON conversion can lose or distort:

- namespaces;
- attributes;
- repeated elements;
- ordering;
- mixed content.

Define the target representation explicitly.

## 31. Batch Inserts

Do not commit every XML record separately.

```
BATCH_SIZE = 1000
batch = []

for record_number, record in enumerate(stream_records(), start=1):
    batch.append((source_hash, record_number, record))

    if len(batch) >= BATCH_SIZE:
        insert_batch(batch)
        batch.clear()

if batch:
    insert_batch(batch)
```

Choose batch size based on record size, database capacity, transaction duration, and recovery requirements.

## 32. Transaction Policy

Choose explicitly among:

- whole-file atomicity;
- batch atomicity;
- record-level quarantine.

A financial or regulatory delivery may require whole-file success. Independently valid partner records may permit quarantine.

Database transaction boundaries must not accidentally define business semantics.

## 33. Extraction State

Track file processing separately from data:

```
CREATE TABLE xml_extraction_run (
    run_id BIGSERIAL PRIMARY KEY,
    source_uri TEXT NOT NULL,
    source_hash TEXT NOT NULL,
    extractor_version TEXT NOT NULL,
    status TEXT NOT NULL,
    records_seen BIGINT NOT NULL DEFAULT 0,
    records_staged BIGINT NOT NULL DEFAULT 0,
    records_rejected BIGINT NOT NULL DEFAULT 0,
    last_committed_record BIGINT,
    started_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ
);
```

Useful states include DISCOVERED, VALIDATING, EXTRACTING, COMPLETED, FAILED, and QUARANTINED.

## 34. Checkpointing

Safe ordering:

```
READ
  |
  v
VALIDATE
  |
  v
STAGE
  |
  v
COMMIT
  |
  v
CHECKPOINT
```

Never advance a durable checkpoint before its corresponding data is committed.

## 35. End-to-End Extractor

```
import xml.etree.ElementTree as ET
from decimal import Decimal

def stream_payments(path):
    namespace = "urn:example:payments"
    payment_tag = "{" + namespace + "}payment"
    id_tag = "{" + namespace + "}id"
    amount_tag = "{" + namespace + "}amount"
    currency_tag = "{" + namespace + "}currency"

    for event, element in ET.iterparse(
        path,
        events=("end",),
    ):
        if element.tag != payment_tag:
            continue

        payment_id = element.findtext(id_tag)
        amount_text = element.findtext(amount_tag)
        currency = element.findtext(currency_tag)

        if payment_id is None:
            raise ValueError("Missing payment id")
        if amount_text is None:
            raise ValueError("Missing amount")
        if currency is None:
            raise ValueError("Missing currency")

        yield {
            "payment_id": payment_id.strip(),
            "amount": Decimal(amount_text.strip()),
            "currency": currency.strip(),
        }

        element.clear()
```

Keep parsing independent from database code.

## 36. Parsing and Staging Separation

Recommended architecture:

```
XML source
   |
   v
XML reader
   |
   v
record extractor
   |
   v
record validator
   |
   v
staging writer
```

The parser should not contain database-specific behavior.

## 37. Error Categories

Use stable categories such as:

```
MALFORMED_XML
UNEXPECTED_ROOT
UNKNOWN_NAMESPACE
MISSING_REQUIRED_ELEMENT
MISSING_REQUIRED_ATTRIBUTE
INVALID_FIELD_TYPE
INVALID_VALUE
SCHEMA_VALIDATION_FAILED
RESOURCE_LIMIT
SOURCE_MUTATED
DATABASE_FAILURE
```

Stable error codes make metrics and recovery automation easier.

## 38. Quarantine

For record-level tolerant processing, preserve:

```
source_hash
record_number
record_id
error_code
error_message
detected_at
extractor_version
```

Preserve raw XML fragments only when permitted by retention and privacy rules.

## 39. Testing Strategy

Minimum tests:

1. valid XML;
2. malformed XML;
3. wrong root;
4. namespace-qualified XML;
5. changed namespace prefix;
6. default namespace;
7. missing element;
8. missing attribute;
9. repeated element;
10. nested records;
11. empty element;
12. Unicode;
13. large document;
14. idempotent replay;
15. database rollback;
16. checkpoint recovery;
17. schema validation failure;
18. security/resource-limit behavior.

## 40. Example Tests

```
import xml.etree.ElementTree as ET
import pytest

XML = """
<payments>
  <payment id="P001">
    <amount>100.00</amount>
    <currency>EUR</currency>
  </payment>
</payments>
"""

def test_extracts_payment():
    root = ET.fromstring(XML)
    payment = root.find("payment")

    assert payment is not None
    assert payment.get("id") == "P001"
    assert payment.findtext("currency") == "EUR"

def test_missing_attribute_fails():
    root = ET.fromstring("<payments><payment /></payments>")
    payment = root.find("payment")

    with pytest.raises(ValueError):
        if payment.get("id") is None:
            raise ValueError("Missing payment id")
```

## 41. Intentional Failure Drill — Namespace Change

Change the namespace URI from the configured value to a new URI.

Expected:

- the contract mismatch is detected;
- extraction does not silently return zero records;
- the run fails or routes to a versioned parser;
- source identity remains available.

A namespace change can represent a schema change.

## 42. Intentional Failure Drill — Missing Element

Remove a required currency element.

Expected:

- XML parsing succeeds;
- record validation fails;
- the missing field is identified;
- strict or quarantine policy is applied;
- the record remains traceable.

## 43. Intentional Failure Drill — Malformed XML

Remove a closing tag.

Expected:

- ParseError is raised;
- the extraction run fails;
- no false successful completion is recorded;
- the source remains available for investigation.

## 44. Intentional Failure Drill — Replay

Run the same XML twice.

Expected:

- identical source hash;
- identical technical record identities;
- no duplicate staging rows;
- second execution is safe.

## 45. Intentional Failure Drill — Database Failure

Stop the database during a batch.

Verify:

1. the failed transaction rolls back;
2. the checkpoint does not advance past uncommitted data;
3. retry is safe;
4. committed records remain;
5. no duplicate staging rows appear.

## 46. Observability

Track:

```
xml_files_discovered_total
xml_files_completed_total
xml_files_failed_total
xml_records_seen_total
xml_records_staged_total
xml_records_rejected_total
xml_parse_errors_total
xml_namespace_errors_total
xml_schema_errors_total
xml_extraction_duration_seconds
xml_bytes_read_total
```

Useful dimensions are source, dataset, schema version, extractor version, outcome, and error category.

Do not put arbitrary record IDs or raw payloads into metric labels.

## 47. Structured Logging

Success example:

```
{
  "event": "xml_extraction_completed",
  "source": "payments",
  "source_hash": "abc123",
  "records_seen": 100000,
  "records_staged": 100000,
  "records_rejected": 0,
  "schema_version": "v2"
}
```

Failure example:

```
{
  "event": "xml_extraction_failed",
  "source": "payments",
  "source_hash": "abc123",
  "error_code": "UNKNOWN_NAMESPACE"
}
```

Do not log full XML payloads when they can contain sensitive information.

## 48. Data Quality Checks

At completion verify:

- expected root exists;
- expected namespace is used;
- required identifiers exist;
- records seen reconcile with the delivery contract;
- staged and rejected counts match the policy;
- source hash is stable.

If a manifest provides an expected record count, reconcile it before completion.

## 49. Performance Testing

Measure:

- records per second;
- bytes per second;
- memory usage;
- CPU usage;
- database throughput;
- batch latency;
- checkpoint frequency.

Use realistic documents. A tiny XML fixture cannot prove that a streaming implementation handles a multi-gigabyte delivery.

## 50. Production Tools You Should Know

### 50.1 Python ElementTree

Use for standard XML parsing, simple XPath-style selection, and incremental parsing with iterparse.

Core APIs:

```
ET.parse()
ET.fromstring()
ET.find()
ET.findall()
ET.findtext()
ET.iterparse()
```

### 50.2 lxml

Use when advanced XPath or mature XML schema-validation workflows are required. Do not add it merely because XML is involved.

### 50.3 PostgreSQL JSONB

Use JSONB for structured staging when downstream transformations benefit from retaining the extracted record shape.

```
payload JSONB NOT NULL
```

Keep the original XML or a durable source reference according to retention requirements.

## 51. Common Mistakes

### Mistake 1 — Ignoring namespaces

Selectors return no records.

**Fix:** match namespace URIs explicitly.

### Mistake 2 — Treating prefixes as stable

The producer changes p: to x:.

**Fix:** use namespace URIs.

### Mistake 3 — Loading huge XML into memory

**Fix:** use iterparse and clear processed elements.

### Mistake 4 — Treating XML validity as business validity

**Fix:** validate required fields and values separately.

### Mistake 5 — Losing parent context

**Fix:** carry required parent identifiers into child records.

### Mistake 6 — Flattening without a target model

**Fix:** define attribute, element, and repeated-value semantics first.

### Mistake 7 — Blind XML-to-JSON conversion

**Fix:** define the target representation explicitly.

### Mistake 8 — Unsafe XML parser configuration

**Fix:** use hardened parsing for untrusted XML.

### Mistake 9 — One transaction per record

**Fix:** use bounded batches.

### Mistake 10 — Random extraction IDs

**Fix:** use deterministic source and record identity.

## 52. Production Implementation Sequence

### Step 1
Obtain representative XML.

### Step 2
Identify the root element and namespace.

### Step 3
Identify repeated record elements.

### Step 4
Identify attributes and child elements.

### Step 5
Identify required parent context.

### Step 6
Define required and optional fields.

### Step 7
Define schema/XSD requirements.

### Step 8
Define source and record identity.

### Step 9
Choose tree parsing or incremental parsing.

### Step 10
Implement namespace-safe extraction.

### Step 11
Implement record validation.

### Step 12
Implement lineage and idempotent staging.

### Step 13
Implement batching and checkpoints.

### Step 14
Add observability.

### Step 15
Test malformed XML, namespaces, replay, and recovery.

### Step 16
Document the runbook and production contract.

## 53. Production Checklist

- [ ] XML contract documented.
- [ ] Root element validated.
- [ ] Namespace URI documented.
- [ ] Prefix is not used as namespace identity.
- [ ] Record element identified.
- [ ] Attributes mapped explicitly.
- [ ] Required fields validated.
- [ ] Optional fields defined.
- [ ] Repeated elements modeled.
- [ ] Parent context preserved.
- [ ] Encoding behavior defined.
- [ ] Secure parser configuration used.
- [ ] Large-document strategy defined.
- [ ] Source identity deterministic.
- [ ] Record identity deterministic.
- [ ] Lineage preserved.
- [ ] Idempotent staging implemented.
- [ ] Transaction policy documented.
- [ ] Checkpoint policy documented.
- [ ] Schema evolution policy documented.
- [ ] Malformed XML tested.
- [ ] Namespace changes tested.
- [ ] Replay tested.
- [ ] Database recovery tested.
- [ ] Metrics and logs exist.
- [ ] Recovery runbook exists.

## 54. Definition of Done

E57 is complete when you can independently:

1. explain the XML tree model;
2. extract elements and attributes;
3. navigate nested records;
4. handle repeated elements;
5. preserve parent context;
6. identify namespace URIs;
7. handle default namespaces;
8. validate the root document;
9. validate record fields;
10. parse decimal and timestamp values safely;
11. detect malformed XML;
12. understand XSD validation;
13. stream large XML with iterparse;
14. release processed elements to control memory;
15. create deterministic record identity;
16. implement idempotent staging;
17. implement checkpoint-safe processing;
18. protect the XML parser boundary;
19. test namespace and schema evolution;
20. recover safely from failed extraction.

## 55. What You Learned

XML extraction is a tree-processing problem.

```
XML SOURCE
    |
    v
VERIFY ROOT + NAMESPACE
    |
    v
PARSE
    |
    v
LOCATE RECORDS
    |
    v
EXTRACT ELEMENTS + ATTRIBUTES
    |
    v
VALIDATE
    |
    v
PRESERVE LINEAGE
    |
    v
IDEMPOTENT STAGE
    |
    v
OBSERVE + RECOVER
```

The key rules are: namespace URI matters more than prefix; valid XML is not necessarily valid business data; large documents need incremental parsing; parent context can be essential; and deterministic identity makes replay safe.

## 56. Next Recipe

**E58 — Parquet Extraction**

The next recipe focuses on columnar extraction, schema inspection, predicate pushdown, column projection, partitioned Parquet datasets, row groups, and efficient ETL staging.