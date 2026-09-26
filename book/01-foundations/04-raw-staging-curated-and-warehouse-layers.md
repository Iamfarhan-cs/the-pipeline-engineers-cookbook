# Chapter 4 — Raw, Staging, Curated and Warehouse Layers

A data pipeline usually has more than one place where data lives.

This is because data has different purposes at different points in its lifecycle.

The data that first arrives from a source is not always ready for analysis.

It may be incomplete. It may contain duplicates. It may use a source-specific format. It may need validation. It may need transformation. It may need to be joined with other data.

Trying to keep all of this in one table or one storage location can make a pipeline difficult to understand and recover.

A common design is to separate data into layers.

~~~text
Source
  |
  v
Raw
  |
  v
Staging
  |
  v
Curated
  |
  v
Warehouse
~~~

Each layer has a different responsibility.

This chapter explains what those layers mean, why they exist, how data moves between them, and how to decide what belongs in each layer.

---

## What Is a Data Layer?

A data layer is a logical stage in the lifecycle of data.

The layer gives the data a purpose.

For example:

- **Raw** keeps what came from the source.
- **Staging** prepares data for reliable processing.
- **Curated** contains cleaned and business-ready data.
- **Warehouse** organizes data for analytics and reporting.

The exact names can differ between organizations.

You may see terms such as:

- Raw
- Bronze
- Landing
- Staging
- Silver
- Curated
- Gold
- Warehouse
- Analytics

The names are less important than the responsibilities.

A team should be able to explain why each layer exists.

---

# Why Do We Need Layers?

Consider a simple API pipeline.

The source sends:

~~~json
{
  "customerId": "123",
  "name": "Farhan",
  "country": "PK",
  "amount": "1250.50"
}
~~~

The source format may not match the format needed by the database or reporting system.

The pipeline may need to:

1. Preserve the original response.
2. Validate required fields.
3. Convert data types.
4. Remove duplicates.
5. Normalize values.
6. Apply business rules.
7. Join the record with other datasets.
8. Build reporting tables.

If all of these operations happen in one place, it becomes difficult to answer basic questions.

For example:

- What did the source actually send?
- Was the original data changed?
- Which records failed validation?
- Which transformation changed the value?
- Can we replay the original input?
- Which version of the data was used?
- Where should a bug be fixed?

Layers help answer these questions.

---

# The Basic Layer Model

A common logical design is:

~~~text
                +----------------+
                |     Source     |
                +-------+--------+
                        |
                        v
                +---------------+
                |      Raw      |
                +-------+-------+
                        |
                        v
                +---------------+
                |    Staging    |
                +-------+-------+
                        |
                        v
                +---------------+
                |    Curated    |
                +-------+-------+
                        |
                        v
                +---------------+
                |   Warehouse   |
                +---------------+
~~~

The flow is not always strictly linear.

A production system may have additional paths.

For example:

~~~text
                    Source
                      |
                      v
                    Raw
                  /     \
                 v       v
             Staging   Quarantine
                 |
                 v
              Curated
                 |
                 v
             Warehouse
~~~

Failed data may move to a quarantine area instead of continuing through the normal path.

The important idea is that each stage has a clear purpose.

---

# Layer 1 — Raw Data

Raw data is data captured from the source with minimal transformation.

The main purpose of raw storage is **preservation**.

You want to keep enough information to understand what the source actually provided.

For example:

~~~text
API response
     |
     v
Raw storage
~~~

The raw representation might be:

- JSON
- CSV
- XML
- Avro
- Parquet
- Message payload
- Database extract
- File received from another system

The exact format depends on the source.

---

## Why Keep Raw Data?

Raw data can be useful for several reasons.

### Reprocessing

Suppose a transformation has a bug.

If the original input is still available, the pipeline may be able to process it again after the bug is fixed.

### Debugging

Suppose a downstream value looks incorrect.

The engineer can inspect the original input.

### Auditing

Some systems need to know what data was received and when.

### Rebuilding

A downstream table may be deleted or rebuilt.

If the raw input still exists, it may be possible to reconstruct the downstream data.

### Source investigation

Raw data helps engineers understand source behavior.

The source may occasionally send unexpected fields or formats.

---

# Raw Does Not Mean No Metadata

A raw layer often needs metadata.

For example:

~~~text
raw record
    |
    +-- source
    +-- received_at
    +-- source_identifier
    +-- payload
    +-- checksum
    +-- ingestion_run
~~~

The exact metadata depends on the system.

Useful metadata can answer questions such as:

- Where did this data come from?
- When was it received?
- Which ingestion run received it?
- What identifies the source record?
- Has this exact payload already been received?

Do not assume the raw layer should contain only the original payload.

Operational metadata can be important for reliability.

---

# Raw Data Should Be Treated Carefully

Raw data may contain:

- Personal information
- Sensitive business information
- Authentication data
- Documents
- Financial information

Raw does not mean unrestricted.

The pipeline still needs appropriate security and access controls.

Later chapters will cover security and sensitive data handling in more detail.

---

# Layer 2 — Staging

Staging is an intermediate processing layer.

Its purpose is usually to make incoming data easier and safer to process.

A staging layer may contain data that has been:

- Parsed
- Validated at a basic level
- Normalized
- Type-converted
- Deduplicated
- Associated with ingestion metadata

The exact responsibilities depend on the architecture.

A simple flow might be:

~~~text
Raw
 |
 v
Parse
 |
 v
Validate
 |
 v
Staging
~~~

Staging is often closer to the structure needed by downstream processing than raw data is.

---

# Example: Raw to Staging

Suppose the source sends:

~~~json
{
  "customer_id": "123",
  "amount": "1250.50",
  "created_at": "2026-09-26T10:15:00Z"
}
~~~

The staging representation might use proper database types:

~~~text
customer_id   -> integer
amount        -> numeric
created_at    -> timestamp
~~~

This is a generic example.

The actual transformation depends on the source contract and target design.

The important point is that staging can convert source-oriented data into a predictable processing format.

---

# Staging Is Not Necessarily the Final Data Model

A common mistake is to treat staging as the final database design.

Staging usually exists to support processing.

It may contain:

- Temporary structures
- Normalized source records
- Processing metadata
- Validation results
- Intermediate values

It does not necessarily represent the final business model.

For example, one staging table may contain one row per source event.

A curated model may later split that information into multiple related tables.

---

# Layer 3 — Curated Data

Curated data is data that has been cleaned, validated, transformed, and organized for downstream use.

The word "curated" means the data has been deliberately prepared.

A curated dataset may have:

- Consistent data types
- Standardized values
- Validated relationships
- Duplicate handling
- Business rules applied
- Useful keys
- Clear semantics

For example:

~~~text
Staging
   |
   +-- validate
   +-- normalize
   +-- deduplicate
   +-- apply business rules
   |
   v
Curated
~~~

Curated data should be easier for downstream consumers to use than raw or staging data.

---

# Example of a Curated Dataset

Suppose a source sends customer records with different country representations:

~~~text
PK
Pakistan
PAK
pk
~~~

A transformation may standardize these values.

For example:

~~~text
PK
PK
PK
PK
~~~

This is a simple example of normalization.

The exact business rule should be defined by the system.

Curated data should not contain unexplained transformations.

A future engineer should be able to understand why a value was changed.

---

# Layer 4 — Warehouse

A data warehouse is usually designed for analytics and reporting.

The warehouse may organize curated information into structures that make analytical queries easier.

For example:

~~~text
Curated data
      |
      v
+----------------------+
|      Warehouse       |
|                      |
|  Fact tables         |
|  Dimension tables    |
|  Aggregations        |
+----------------------+
      |
      v
BI / Reports / Analysis
~~~

A warehouse is not simply another database.

Its data model and workload are usually designed around analytical use cases.

Later chapters will cover fact tables, dimension tables, slowly changing dimensions, indexing, and partitioning in more detail.

---

# Fact and Dimension Tables

A common warehouse pattern uses:

- Fact tables
- Dimension tables

A fact table usually contains measurable business events or values.

Examples:

- Transactions
- Orders
- Payments
- Page views

Dimension tables usually describe entities used to analyze those facts.

Examples:

- Customer
- Product
- Country
- Date

A simplified model might look like:

~~~text
              +---------------+
              |   Customer    |
              +-------+-------+
                      |
                      |
+-------------+       |
|    Date     |-------+
+-------------+       |
                      v
              +---------------+
              | Transactions  |
              +---------------+
                      ^
                      |
+-------------+       |
|   Product   |-------+
+-------------+
~~~

The exact warehouse model depends on the business.

---

# Raw vs Staging vs Curated vs Warehouse

A useful comparison is:

| Layer | Main Purpose | Typical State |
|---|---|---|
| Raw | Preserve source data | Close to source |
| Staging | Prepare data for processing | Parsed and normalized |
| Curated | Provide trusted processed data | Cleaned and business-ready |
| Warehouse | Support analytics | Analytical model |

These are logical responsibilities.

A real implementation may use different technologies or combine some layers.

---

# The Layers Do Not Have to Mean Four Physical Databases

This is an important point.

A layer is a logical concept.

You do not necessarily need:

~~~text
Database 1 = Raw
Database 2 = Staging
Database 3 = Curated
Database 4 = Warehouse
~~~

That would often be unnecessary.

The layers could exist as:

- Different PostgreSQL schemas
- Different tables
- Different object-storage prefixes
- Different databases
- Different buckets
- Different datasets
- Different warehouse schemas

The physical design should follow the requirements.

For a small pipeline, raw and staging may even be represented by a small number of tables.

For a large platform, each layer may have its own storage technology.

---

# A More Detailed Lifecycle

Consider this generic pipeline:

~~~text
Source
  |
  v
Ingestion
  |
  v
Raw
  |
  v
Parsing
  |
  v
Validation
  |
  v
Staging
  |
  v
Transformation
  |
  v
Curated
  |
  v
Warehouse
  |
  v
Analytics
~~~

Now imagine a record fails validation.

It should not simply disappear.

A production design may use:

~~~text
                    Raw
                     |
                     v
                Validation
                 /       \
              pass       fail
               |           |
               v           v
            Staging    Quarantine
               |
               v
            Curated
               |
               v
           Warehouse
~~~

This makes failure handling part of the architecture.

---

# Why Raw and Curated Should Not Be Mixed Carelessly

Suppose a pipeline receives this source value:

~~~text
amount = "1250.50"
~~~

A transformation changes it to:

~~~text
amount = 1250.50
~~~

That transformation may be correct.

But if the original value is overwritten, you lose evidence of what the source actually sent.

Keeping raw and processed representations separate gives you a useful boundary.

You can then say:

~~~text
Raw:
"1250.50"

Curated:
1250.50
~~~

The difference is visible.

This becomes valuable when debugging transformations.

---

# Data Lineage

Data lineage means being able to understand where data came from and how it changed.

A simple lineage chain is:

~~~text
Source Event
     |
     v
Raw Record
     |
     v
Staging Record
     |
     v
Curated Record
     |
     v
Warehouse Row
     |
     v
Dashboard
~~~

If a dashboard shows an incorrect number, an engineer should ideally be able to move backward through this chain.

Questions include:

- Which warehouse row is wrong?
- Which curated record produced it?
- Which staging record produced that?
- Which raw input produced that?
- Which source produced the raw input?

Good layer design helps make this investigation possible.

---

# Data Contracts Across Layers

Each layer should have clear expectations.

For example:

~~~text
Source -> Raw
Contract:
Capture the incoming payload.

Raw -> Staging
Contract:
Required fields can be parsed.

Staging -> Curated
Contract:
Business validation succeeds.

Curated -> Warehouse
Contract:
Data matches the analytical model.
~~~

These are conceptual contracts.

The actual rules should be documented for the system.

A pipeline becomes difficult to maintain when every transformation is based on assumptions that exist only in someone's memory.

---

# Schema Changes

Source systems change.

A source may add a field.

For example:

~~~text
Old:
id
amount
created_at

New:
id
amount
currency
created_at
~~~

Adding a field may be easy.

Removing or changing a field can be more difficult.

For example:

~~~text
amount
~~~

changes from a numeric value to a formatted string.

Each layer may need to handle the change.

This is one reason schema evolution matters.

A raw layer can sometimes preserve the original input while downstream layers adapt to the new structure.

---

# Layers and Idempotency

Layers do not remove the need for idempotency.

Suppose the same source event enters the pipeline twice.

The system might produce:

~~~text
Raw
  |
  +-- Event A
  +-- Event A
  |
  v
Staging
  |
  +-- Event A
  +-- Event A
~~~

If the duplicate is not detected, the problem can move downstream.

A pipeline needs an explicit strategy for duplicate handling.

Possible strategies include:

- Unique event IDs
- Unique source keys
- Checksums
- Upserts
- Processing state
- Deduplication windows

The correct strategy depends on the source and processing model.

---

# Layers and Replay

Layered storage can make replay easier.

Suppose the curated layer contains incorrect data because of a transformation bug.

If raw data is preserved, you may be able to:

~~~text
Raw
 |
 v
Corrected transformation
 |
 v
Staging
 |
 v
Curated
 |
 v
Warehouse
~~~

You do not necessarily need to request the data again from the source.

This is one of the strongest practical reasons to preserve useful raw input.

However, raw retention has storage, security, privacy, and cost implications.

---

# Layers and Backfills

A backfill means processing historical data again.

For example:

~~~text
January
February
March
April
~~~

A new business rule may need to be applied to January through April.

A layered pipeline can support this by processing historical raw or staging data through the updated logic.

A simplified flow is:

~~~text
Historical Raw
      |
      v
Updated Transformation
      |
      v
Curated
      |
      v
Warehouse
~~~

The backfill still needs careful handling of:

- Existing records
- Duplicates
- Time boundaries
- Dependencies
- Data quality
- Downstream consumers

Backfills are covered in more detail later in the book.

---

# Storage Choices

Different layers can use different storage systems.

For example:

~~~text
Raw
  -> Object storage

Staging
  -> PostgreSQL or object storage

Curated
  -> PostgreSQL or analytical storage

Warehouse
  -> Data warehouse
~~~

This is only a generic example.

Do not choose storage technology based only on the layer name.

Consider:

- Data volume
- Query patterns
- Retention
- Cost
- Latency
- Reliability
- Security
- Operational complexity
- Team skills

A small project may use PostgreSQL for several layers.

A larger platform may use object storage, processing engines, and a dedicated warehouse.

---

# Common Mistakes

## Mistake 1: Treating raw data as disposable

If raw input is deleted immediately, debugging and replay can become much harder.

Retention should be designed deliberately.

---

## Mistake 2: Putting business logic into raw storage

Raw data should generally preserve source information rather than become a heavily transformed business model.

If business transformations happen too early, source fidelity can be lost.

---

## Mistake 3: Treating staging as permanent business data

Staging is usually an intermediate processing area.

Do not automatically expose every staging table to analysts.

---

## Mistake 4: Making the warehouse a copy of raw data

A warehouse should serve its analytical purpose.

Simply copying raw records into a warehouse does not automatically create a useful analytical model.

---

## Mistake 5: Creating layers without defining responsibilities

Having folders or schemas called raw, staging, and curated does not make a layered architecture.

Each layer needs a clear purpose.

---

## Mistake 6: Hiding transformations

If a value changes between layers, the reason should be understandable.

Hidden transformations create difficult debugging problems.

---

## Mistake 7: Forgetting failed data

Not every record will pass every processing step.

The architecture should define what happens to invalid or failed data.

---

# Production Considerations

A production layered pipeline should define:

### Raw

- What is captured?
- How long is it retained?
- What metadata is stored?
- Can it be replayed?
- Who can access it?

### Staging

- What validation happens?
- Which types are converted?
- How are duplicates handled?
- How is processing status tracked?
- What happens to failed records?

### Curated

- Which business rules are applied?
- What makes a record trusted?
- How are relationships validated?
- How are transformations documented?

### Warehouse

- Who consumes the data?
- What analytical model is used?
- How are facts and dimensions organized?
- What freshness is expected?
- How are historical changes handled?

These questions should be answered as part of system design.

---

# A Practical Layering Example

Consider a generic transaction pipeline.

The source sends:

~~~text
Transaction event
~~~

The pipeline could use:

~~~text
+----------------------+
| Source               |
| Transaction Event    |
+----------+-----------+
           |
           v
+----------------------+
| Raw                  |
| Original payload     |
| Receipt metadata     |
+----------+-----------+
           |
           v
+----------------------+
| Staging              |
| Parsed fields        |
| Basic validation     |
+----------+-----------+
           |
           v
+----------------------+
| Curated              |
| Clean transactions   |
| Business rules       |
+----------+-----------+
           |
           v
+----------------------+
| Warehouse            |
| Facts + dimensions   |
+----------+-----------+
           |
           v
+----------------------+
| Reports / Analytics  |
+----------------------+
~~~

Now consider a bad transaction.

~~~text
Source
  |
  v
Raw
  |
  v
Validation
  |
  +---- valid ----> Staging -> Curated -> Warehouse
  |
  +---- invalid --> Quarantine
~~~

The invalid record is not silently lost.

That is a production mindset.

---

# What You Learned

In this recipe, you learned:

- Data layers separate different responsibilities.
- Raw data preserves source information.
- Staging prepares data for reliable processing.
- Curated data contains cleaned and business-ready information.
- Warehouses organize data for analytical workloads.
- Layers are logical concepts and do not always require separate databases.
- Raw data can support debugging, replay, auditing, and rebuilding.
- Layer boundaries make transformations easier to understand.
- Failed records need an explicit path.
- Idempotency, replay, and backfills still matter in layered pipelines.
- Data lineage becomes easier to reason about when layers are clearly defined.

The main lesson is simple:

**A data layer should exist because it has a clear responsibility, not simply because the architecture diagram has another box.**

---

# Practical Checklist

Before implementing a layered pipeline, ask:

- [ ] What does the source provide?
- [ ] What must be preserved exactly?
- [ ] What belongs in raw storage?
- [ ] What validation belongs before staging?
- [ ] What does staging represent?
- [ ] What makes data "curated"?
- [ ] What is the warehouse designed to answer?
- [ ] Where do invalid records go?
- [ ] How are duplicates handled?
- [ ] Can historical data be replayed?
- [ ] Can a backfill be performed?
- [ ] What metadata is required for lineage?
- [ ] How long is each layer retained?
- [ ] Who can access each layer?
- [ ] How will schema changes move through the layers?
- [ ] Which storage technology is appropriate for each layer?
- [ ] Can an engineer trace a warehouse value back to its source?

If these questions are clear, the layer design is much easier to implement.

---
