# E07 — Extract Data from XML Files

XML remains common in enterprise integrations, banking, logistics, government feeds, SOAP systems, legacy exports, and partner data exchanges.

XML is more expressive than CSV and structurally different from JSON. Production extraction therefore has to handle:

- elements and attributes
- namespaces
- repeated elements
- mixed content
- nested structures
- schema validation
- malformed documents
- external entities and parser security
- very large documents
- schema evolution

The goal is:

> Extract XML only after proving that the document is complete, structurally valid, safely parsed, and compatible with the pipeline contract.

## 1. Problem Recognition

Common production failures include:

- namespace mismatches
- treating attributes as elements
- incorrect XPath expressions
- assuming element order is irrelevant when it is contractually meaningful
- repeated nodes being missed
- nested collections being flattened incorrectly
- malformed XML
- incomplete files
- external entity vulnerabilities
- very large documents exhausting memory
- schema changes
- missing vs empty elements
- duplicate files
- checkpoint reuse against changed files

Do not equate:

    valid XML syntax

with:

    valid pipeline data

## 2. XML Extraction Architecture

    FILE ARRIVES
        ↓
    DISCOVER
        ↓
    VERIFY COMPLETENESS
        ↓
    IDENTIFY FILE
        ↓
    SECURE PARSE
        ↓
    VALIDATE STRUCTURE
        ↓
    EXTRACT NODES
        ↓
    VALIDATE RECORDS
        ↓
    PERSIST
        ↓
    CHECKPOINT
        ↓
    MARK COMPLETE

Each stage has a separate responsibility.

## 3. XML Data Model

XML represents a tree.

Example:

    <payments>
      <payment id="1001">
        <amount>125.50</amount>
        <currency>EUR</currency>
      </payment>
    </payments>

The payment ID above is an attribute.

Amount and currency are child elements.

This distinction matters when mapping XML to relational or event data.

## 4. Elements vs Attributes

XML may encode the same concept in different ways.

Attribute:

    <payment id="1001">

Element:

    <payment>
      <id>1001</id>
    </payment>

Do not assume the representation from the field name alone.

The source contract determines how values should be extracted.

## 5. XML Namespaces

Namespaces are one of the most common sources of XML extraction bugs.

Example:

    <p:payment xmlns:p="urn:payments">
      <p:id>1001</p:id>
    </p:payment>

The visible prefix is not the namespace identity.

The namespace URI is the important identity.

Therefore an XPath that ignores namespaces may fail even though the XML visually contains the expected element.

## 6. Namespace-Aware Extraction

Conceptually:

    namespace:
        p → urn:payments

Then query using the namespace mapping.

Do not depend on a producer's chosen prefix.

These can represent the same namespace:

    <p:payment ...>

and:

    <payment xmlns="urn:payments">

The prefix changed; the namespace URI did not.

## 7. XPath

XPath provides a way to locate nodes in an XML tree.

Example:

    /payments/payment

or:

    /payments/payment/amount

With namespaces, queries must use namespace-aware mappings.

XPath should be treated as part of the extraction contract.

## 8. Basic Python XML Parsing

For a small trusted XML document:

    import xml.etree.ElementTree as ET

    tree = ET.parse('payments.xml')
    root = tree.getroot()

    for payment in root.findall('payment'):
        payment_id = payment.get('id')
        amount = payment.findtext('amount')
        currency = payment.findtext('currency')

This is appropriate only when the document size and security requirements are understood.

## 9. Large XML Documents

Do not automatically materialize a multi-gigabyte XML document into memory.

For large repetitive documents, use streaming parsing.

Conceptually:

    READ EVENT
       ↓
    FIND COMPLETE RECORD
       ↓
    VALIDATE
       ↓
    PERSIST
       ↓
    RELEASE ELEMENT
       ↓
    NEXT RECORD

Python's iterparse can support this pattern for suitable XML layouts.

## 10. Streaming with iterparse

Conceptually:

    import xml.etree.ElementTree as ET

    for event, element in ET.iterparse(
        'payments.xml',
        events=('end',)
    ):
        if element.tag == 'payment':
            process(element)
            element.clear()

Clearing processed elements prevents the in-memory tree from growing unnecessarily.

The exact implementation must account for namespaces and parent/child relationships.

## 11. Repeated Elements

XML commonly represents collections through repeated elements.

Example:

    <payments>
      <payment>...</payment>
      <payment>...</payment>
      <payment>...</payment>
    </payments>

Each payment can become a record.

Do not assume the collection is always at the same depth across source versions.

## 12. Nested Collections

A record can contain a collection:

    <payment>
      <id>1001</id>
      <items>
        <item>...</item>
        <item>...</item>
      </items>
    </payment>

Decide whether the pipeline should:

- preserve items as nested data
- create child records
- aggregate them
- extract them into a separate dataset

Do not flatten without understanding cardinality.

## 13. Cardinality Problems

Flattening multiple nested collections can multiply rows.

Example:

    1 payment
      × 5 items
      × 3 adjustments
      = 15 combinations

If the relationship is not modeled correctly, the extraction can create false records.

Understand parent-child relationships before flattening.

## 14. Missing vs Empty Elements

These are different:

    <payment />

and:

    <payment>
      <description />
    </payment>

A missing element and an empty element may carry different meanings.

Define the policy for:

- missing
- empty
- nil
- whitespace-only content

## 15. XML Schema Validation

XML ecosystems may use XSD schemas to define structure and types.

An XSD can define:

- required elements
- element order
- types
- cardinality
- attributes
- enumerations
- namespaces

Schema validation can catch structural problems before records enter the pipeline.

Do not assume that every XML producer enforces its own schema correctly.

## 16. XSD and Business Validation

Schema validation and business validation are different.

XSD may prove:

    amount is decimal

but not necessarily:

    amount must be positive

Therefore:

    XML syntax
        ↓
    XSD/schema validation
        ↓
    business validation

Keep these layers separate.

## 17. Secure XML Parsing

XML parsers can have security risks when processing untrusted input, particularly around external entities and related resource expansion behavior.

Production rules:

- use secure parser configuration
- disable unnecessary external entity processing
- avoid fetching external resources during parsing
- use trusted libraries and maintained parser configurations
- treat partner XML as untrusted input

Do not enable external resource resolution merely because a document references a DTD.

## 18. Entity Expansion

An XML document can contain entity declarations.

Unsafe parser configurations can allow an attacker to cause excessive resource consumption or unexpected external resource access.

The extraction worker should parse XML under a security policy appropriate for untrusted input.

Security is part of extraction correctness.

## 19. Encoding

XML can declare its encoding:

    <?xml version="1.0" encoding="UTF-8"?>

The file may also contain a byte-order mark or use another supported encoding.

Use a parser that correctly handles XML encoding declarations and byte input.

Do not decode bytes manually before the XML parser unless you understand the consequences.

## 20. XML Declarations

XML documents may begin with:

    <?xml version="1.0" encoding="UTF-8"?>

Do not assume the declaration is always present.

The parser should determine encoding according to the XML specification and actual input bytes.

## 21. CDATA

XML can contain CDATA:

    <description><![CDATA[Payment <pending> review]]></description>

CDATA is still element content.

The extraction contract should determine whether markup-like text is treated as plain data or requires additional processing.

Never execute or interpret extracted text as code.

## 22. Whitespace

XML can contain formatting whitespace:

    <payment>
      <id>1001</id>
    </payment>

Do not assume indentation represents business data.

At the same time, whitespace can be meaningful inside text nodes.

Normalize only according to the source contract.

## 23. File Completeness

An XML document can be truncated while still appearing as a file.

Prefer:

    WRITE TEMP
       ↓
    CLOSE
       ↓
    VERIFY
       ↓
    ATOMIC RENAME
       ↓
    PUBLISHED XML

If no explicit completion signal exists, stable-size checks are only heuristics.

A parser error caused by truncation should be distinguished from a genuine schema or content error.

## 24. File Identity

Track:

    source
    filename
    size
    checksum
    discovered_at

SHA-256 can provide a content identity.

File identity protects against duplicate delivery and accidental replacement.

## 25. Duplicate Files

The same XML file can arrive more than once.

Use durable file state keyed by a suitable identity such as:

    source + filename + checksum

Do not rely on filename alone when the producer can replace file contents.

## 26. Duplicate Records

Different XML files can contain the same logical record.

Use stable source identifiers where available.

File-level idempotency and record-level idempotency solve different problems.

## 27. Checkpointing

Checkpointing large XML extraction is more complicated than JSONL because one logical record can span many bytes and nested elements.

Possible state includes:

    file_id
    checksum
    record_number
    last_source_id
    batch_number

Byte-position checkpoints should only be used when parser behavior and encoding assumptions make them safe.

Always bind progress to immutable file identity.

## 28. Batch Persistence

For large XML feeds:

    PARSE RECORDS
         ↓
    BUILD BATCH
         ↓
    VALIDATE
         ↓
    PERSIST
         ↓
    CHECKPOINT

Keep the batch bounded so memory and failure scope remain controlled.

## 29. Malformed XML

A malformed XML document may fail before individual records can be safely extracted.

Examples:

- unclosed element
- invalid encoding
- malformed attribute
- invalid namespace declaration
- truncated document

At the document level, the safest policy may be:

    PARSE FAILURE
       ↓
    FILE FAILURE
       ↓
    QUARANTINE

Do not silently process an incomplete XML tree.

## 30. Record-Level XML Failures

If the XML document is valid but a particular record violates the data contract:

    VALID XML
       ↓
    RECORD VALIDATION
       ↓
    ACCEPT / REJECT

Define whether rejected records are quarantined or whether the entire file fails.

## 31. Error Thresholds

A source contract may define:

    max_bad_records = 0

or a controlled tolerance such as:

    max_bad_record_ratio = 0.01

When exceeded:

    STOP
      ↓
    QUARANTINE
      ↓
    ALERT

Do not invent tolerance rules during an incident.

## 32. Schema Evolution

XML schema changes can include:

- new element
- removed element
- renamed element
- type change
- namespace change
- cardinality change
- element-order change
- new required attribute

Protect extraction with:

- schema validation
- compatibility tests
- versioned contracts
- source-change monitoring
- controlled deployments

Namespace changes deserve special attention because they can make existing XPath expressions return nothing without obvious parser errors.

## 33. Testing

Test at least:

1. Valid XML.
2. Empty document.
3. Missing expected root.
4. Wrong root element.
5. Namespace variation.
6. Missing namespace.
7. Missing required element.
8. Empty element.
9. Missing attribute.
10. Invalid attribute value.
11. Repeated records.
12. Nested collection.
13. Cardinality edge case.
14. Malformed XML.
15. Truncated XML.
16. Invalid encoding.
17. Schema validation failure.
18. Duplicate file.
19. Duplicate record.
20. Large XML document.
21. Checkpoint restart.
22. Persistence failure.
23. File mutation.
24. Secure-parser behavior.

## 34. Observability

Useful metrics:

    xml_files_discovered_total
    xml_files_processed_total
    xml_files_failed_total
    xml_files_quarantined_total
    xml_files_duplicated_total
    xml_records_read_total
    xml_records_persisted_total
    xml_records_rejected_total
    xml_parse_errors_total
    xml_schema_errors_total
    xml_namespace_errors_total
    xml_processing_duration_seconds
    xml_bytes_processed_total

Useful log fields:

    extraction_run_id
    file_id
    filename
    checksum
    record_number
    batch_number
    source_schema_version
    namespace_set
    error_type
    records_processed

Do not log complete XML documents or sensitive record contents unnecessarily.

## 35. Intentional Failure

### Failure drill 1 — Namespace mismatch

Change the namespace URI or prefix in a test document.

Verify that namespace-aware extraction detects the contract mismatch rather than silently producing zero records.

### Failure drill 2 — Truncated document

Remove the closing portion of the XML file.

Verify the file is rejected and not partially accepted.

### Failure drill 3 — Schema violation

Remove a required element or change its type.

Verify schema/record validation fails.

### Failure drill 4 — Duplicate file

Deliver the same XML artifact twice.

Verify file-level idempotency.

### Failure drill 5 — Persistence failure

Fail persistence after several valid records.

Restart and verify checkpoint and downstream idempotency behavior.

### Failure drill 6 — Large document

Process a deliberately large XML file.

Verify memory remains bounded with streaming parsing.

### Failure drill 7 — File mutation

Modify the XML after checkpointing.

Verify the old checkpoint cannot be applied to the changed artifact.

## 36. Recovery

When XML extraction fails:

1. Identify the extraction run and file.
2. Verify filename, size, and checksum.
3. Confirm publication completeness.
4. Determine whether the failure is parsing, namespace, schema, record, or persistence related.
5. Inspect the source schema/version.
6. Verify checkpoint state.
7. Confirm the source artifact has not changed.
8. Resume only from a valid checkpoint.
9. Reconcile expected and persisted records.
10. Mark the file complete only after verification.

Never silently downgrade a namespace or schema error into an empty dataset.

## 37. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **lxml** | Python XML library with XPath, namespace handling, and streaming-related capabilities. |
| **defusedxml** | Security-focused XML parsing helpers for untrusted XML input. |
| **SAX** | Event-driven XML parsing model useful for understanding streaming extraction of large documents. |

> These are reference tools for production vocabulary. The underlying XML extraction mechanics should still be understood independently.

## 38. Production Runbook

### XML parser fails

Check:

1. File completeness.
2. Encoding declaration.
3. Truncation.
4. XML syntax.
5. Producer version.

### Extraction returns zero records

Check:

1. Namespace URI.
2. XPath.
3. Root element.
4. Schema version.
5. Source changes.

Never assume zero records means the source legitimately sent no data.

### Memory usage is too high

Check:

1. Full-tree parsing.
2. Document size.
3. Streaming parser usage.
4. Element clearing.
5. Batch size.

### Duplicate records appear

Check:

1. File identity.
2. Processed-file state.
3. Source identifiers.
4. Checkpoint restart.
5. Upstream duplicate delivery.

### What not to do

Do not:

- ignore XML namespaces
- assume prefixes identify namespaces
- manually parse XML with string operations
- parse huge documents into memory blindly
- enable external resource resolution unnecessarily
- silently ignore schema errors
- treat zero extracted records as success without validation
- reuse checkpoints against changed files

## 39. Common Mistakes

### Mistake 1 — Ignoring namespaces

A correct-looking XPath can return nothing when the namespace is wrong.

### Mistake 2 — Confusing attributes and elements

The source representation determines how values must be extracted.

### Mistake 3 — Flattening nested collections blindly

Incorrect cardinality can create false records.

### Mistake 4 — Treating valid XML as valid business data

Syntax does not establish business correctness.

### Mistake 5 — Unsafe XML parsing

Untrusted XML must be parsed under an appropriate security policy.

### Mistake 6 — Unbounded memory

Large XML documents require streaming strategies.

### Mistake 7 — Treating zero rows as success

Namespace or XPath bugs can produce zero records from a non-empty file.

## 40. Definition of Done

You are done when you can:

- understand the XML tree model
- distinguish elements from attributes
- handle XML namespaces correctly
- write namespace-aware XPath expressions
- extract repeated elements
- model nested collections
- reason about cardinality
- distinguish missing and empty elements
- understand XSD validation
- separate schema validation from business validation
- parse untrusted XML securely
- stream large XML documents
- batch records safely
- detect malformed and truncated XML
- identify duplicate files
- identify duplicate records
- checkpoint XML extraction safely
- detect namespace and schema changes
- test parser and contract failures
- intentionally break XML extraction
- recover from failed extraction
- operate XML extraction with a production runbook

## 41. What You Learned

The central principle is:

> XML extraction requires both structural correctness and semantic correctness, with namespace handling and parser security treated as first-class concerns.

A production XML pipeline follows:

    DISCOVER
       ↓
    VERIFY COMPLETENESS
       ↓
    IDENTIFY FILE
       ↓
    SECURE PARSE
       ↓
    VALIDATE STRUCTURE
       ↓
    EXTRACT NODES
       ↓
    VALIDATE RECORDS
       ↓
    PERSIST
       ↓
    CHECKPOINT
       ↓
    MARK COMPLETE

The key question is:

> What proves that this XML artifact contains the correct records, under the correct namespace and schema, and that I parsed it safely?

That question drives namespace handling, XPath, schema validation, secure parsing, streaming, checkpointing, testing, observability, and recovery.