# Recipe 21 — Add Deduplication

A pipeline can receive the same logical data more than once.

Sometimes the duplicates have exactly the same fields.

Sometimes they have different technical IDs but represent the same business record.

Sometimes the duplicate is caused by a source system.

Sometimes it is created by a retry, replay, file reprocessing, or an upstream integration.

Idempotency makes repeated processing safe.

Deduplication answers a different question:

> Which records should be treated as duplicates?

This distinction is important.

A pipeline can be idempotent and still contain duplicate business records.

For example:

```text
Record A
    event_id = evt-1001
    customer_id = cust-42
    amount = 250

Record B
    event_id = evt-2007
    customer_id = cust-42
    amount = 250
```

The technical IDs are different.

But the two records may represent the same business event.

This chapter explains how to investigate, design, implement, test, and operate deduplication.

---

## 1. Goal

The goal is to identify and handle duplicate records without accidentally removing legitimate records.

By the end of this recipe, you should understand how to:

- define what a duplicate means
- distinguish exact duplicates from business duplicates
- choose a deduplication key
- investigate the source of duplicates
- decide where deduplication belongs
- use database constraints where appropriate
- deduplicate batch data
- deduplicate streaming data
- handle conflicting duplicates
- retain useful duplicate information
- test duplicate detection
- combine deduplication with idempotency
- verify the final data state

The central idea is:

```text
Multiple records
      |
      v
Identify duplicates
      |
      v
Apply an explicit policy
      |
      v
Keep, merge, update, quarantine, or reject
```

Deduplication is not simply "delete duplicates."

It is a data-model and business-rule decision.

---

## 2. Problem

Imagine a source sends this data:

```json
[
  {
    "customer_id": "cust-42",
    "amount": 250,
    "currency": "EUR"
  },
  {
    "customer_id": "cust-42",
    "amount": 250,
    "currency": "EUR"
  }
]
```

At first glance, these records look identical.

If the target table expects one transaction, storing both records creates incorrect data.

But now consider:

```json
[
  {
    "customer_id": "cust-42",
    "amount": 250,
    "currency": "EUR",
    "created_at": "2026-09-26T10:00:00Z"
  },
  {
    "customer_id": "cust-42",
    "amount": 250,
    "currency": "EUR",
    "created_at": "2026-09-26T11:00:00Z"
  }
]
```

These records look similar.

They may be two legitimate transactions.

If the pipeline deletes one just because the customer and amount match, it loses valid data.

This is the core difficulty:

> Similar records are not automatically duplicate records.

The pipeline needs a clear definition.

---

## 3. Why This Matters

Duplicates can affect almost every downstream calculation.

For example:

```text
Original transactions = 1,000
Duplicate transactions = 50

reported transactions = 1,050
```

The same problem can affect:

- revenue
- balances
- customer activity
- payment counts
- event counts
- inventory
- analytics
- reporting
- machine-learning features
- warehouse facts

Duplicates can also create referential problems.

For example:

```text
one business event
      |
      +--> duplicate fact A
      |
      +--> duplicate fact B
```

A downstream report may count both facts.

The result can look technically valid while being logically wrong.

---

## 4. When to Use This Recipe

Deduplication is useful when the source or pipeline can produce multiple records representing the same logical entity or event.

Common situations include:

- API ingestion
- webhooks
- file imports
- event processing
- database synchronization
- CDC pipelines
- batch ingestion
- streaming systems
- historical backfills
- replay systems
- third-party integrations

You should investigate deduplication when you see symptoms such as:

- unexpected row counts
- repeated business identifiers
- repeated events
- duplicate files
- repeated API responses
- duplicate warehouse facts
- repeated customer records
- unexpected aggregates

---

## 5. Architecture

A simple deduplication flow looks like this:

```text
Source
  |
  v
Acquire
  |
  v
Raw
  |
  v
Validate
  |
  v
Identify duplicate key
  |
  v
+--------------------+
| Duplicate exists?  |
+--------------------+
   |              |
  Yes             No
   |              |
   v              v
Apply policy     Process
   |              |
   +-------+------+
           |
           v
         Target
```

The important part is the phrase:

**Apply policy.**

A duplicate does not always mean "delete."

Possible policies include:

- skip
- keep first
- keep latest
- update existing
- merge
- quarantine
- reject
- replace
- retain all but mark duplicates

---

## 6. Before You Start

Before implementing deduplication, answer these questions:

1. What exactly is a duplicate?
2. Which fields define logical identity?
3. Can two legitimate records have the same values?
4. Does the source provide a stable business identifier?
5. Can the same record have different technical IDs?
6. Which record should survive if duplicates conflict?
7. Should duplicates be deleted, skipped, merged, or retained?
8. Where should duplicate detection happen?
9. Should raw duplicate data be preserved?
10. How will the decision be tested?

Do not start with a SQL DELETE statement.

Start with the definition.

---

## 7. Exact Duplicates

An exact duplicate has the same relevant values.

For example:

```text
Record A:
customer_id = cust-42
amount      = 250
currency    = EUR

Record B:
customer_id = cust-42
amount      = 250
currency    = EUR
```

If all fields that matter are identical, the records may be exact duplicates.

But even here, be careful.

Some fields should usually be ignored when comparing records.

For example:

- ingestion_time
- processing_time
- internal_row_id

These fields can be different even when the source records are the same.

Therefore:

> Exact duplicate detection usually means identical values for the fields that define source or business identity, not identical values for every database column.

---

## 8. Business-Key Duplicates

A business duplicate can have different technical values.

For example:

```text
Record A
source_id = A-100
customer_id = cust-42
transaction_ref = TX-500
amount = 250

Record B
source_id = B-900
customer_id = cust-42
transaction_ref = TX-500
amount = 250
```

The source IDs differ.

The transaction reference is the same.

If transaction_ref uniquely identifies a transaction in the business domain, these records may represent the same transaction.

This is why technical IDs and business keys should not automatically be treated as the same thing.

---

## 9. Choosing a Deduplication Key

The deduplication key should represent the identity of the data being processed.

Possible keys include:

```text
transaction_reference
external_customer_id
source_record_id
event_id
webhook_id
composite business key
```

Sometimes one field is enough.

Sometimes several fields are required.

For example:

```text
customer_id + transaction_date + transaction_reference
```

could represent a business key.

The correct key depends on the source and the business meaning of the record.

---

## 10. Composite Keys

Suppose the source does not provide one unique field.

You may need a combination:

```text
customer_id
+
transaction_date
+
transaction_reference
```

The combination becomes the logical identity.

Conceptually:

```text
(
    customer_id,
    transaction_date,
    transaction_reference
)

must be unique.
```

A database can enforce this with a composite unique constraint.

For example:

```sql
ALTER TABLE transactions
ADD CONSTRAINT uq_transactions_business_key
UNIQUE (
    customer_id,
    transaction_date,
    transaction_reference
);
```

This is a **generic example**.

Use the actual table and business columns from the repository and data model.

---

## 11. Do Not Assume Similarity Means Duplication

Consider:

```text
customer_id = cust-42
amount = 250
currency = EUR
```

appearing twice.

Are they duplicates?

Not necessarily.

They may be:

```text
transaction at 10:00
transaction at 11:00
```

The fields that distinguish legitimate events may be missing from the example.

This is why deduplication requires domain understanding.

A dangerous deduplication rule is:

> "If these three fields match, delete one."

A better approach is:

> "These fields define one business transaction because the source contract says so."

The second statement gives the rule a defensible meaning.

---

## 12. Repository Investigation

Before changing an existing pipeline, investigate where duplicates enter.

Start from the target table and work backwards.

```text
Target table
    |
    v
Database write
    |
    v
Processing function
    |
    v
Validation
    |
    v
Staging
    |
    v
Raw data
    |
    v
Source
```

Ask:

- Are duplicates already present in raw data?
- Are duplicates created during transformation?
- Is the same file processed twice?
- Is the same API page fetched twice?
- Is a message delivered more than once?
- Is a join multiplying rows?
- Is the database missing a uniqueness rule?
- Is the source itself sending duplicate records?

This distinction is important.

The correct fix depends on where the duplication is introduced.

---

## 13. Common Sources of Duplicates

Duplicates can come from many places.

### Source duplicates

The source sends the same logical record twice.

### Repeated ingestion

The same file or API page is ingested more than once.

### Message redelivery

A consumer receives the same event again.

### Retry behavior

A retry repeats an operation that already succeeded.

### Replay

Historical events are intentionally processed again.

### Backfill

Historical data is loaded into a target that already contains part of the same period.

### Transformation

A join or transformation accidentally multiplies rows.

For example:

```text
A has 1 row
B has 3 matching rows

A join can produce:
1 x 3 = 3 rows
```

The result may look like duplicate data even though the duplication was introduced by the transformation.

---

## 14. Deduplication Before or After Validation?

There is no universal answer.

One possible flow is:

```text
Raw
  |
  v
Validate
  |
  v
Deduplicate
  |
  v
Process
```

Another is:

```text
Raw
  |
  v
Identify duplicate
  |
  v
Validate unique records
  |
  v
Process
```

The correct order depends on what information is required to identify duplicates.

For example, if the duplicate key is:

```text
external_id
```

then the field must exist and be usable before duplicate detection.

If malformed records cannot provide the key, validation may need to happen first.

The important point is to define the order explicitly.

---

## 15. Deduplication and Raw Data

Do not automatically delete duplicate raw records.

Raw data can be valuable for:

- auditing
- debugging
- replay
- source investigation
- reconciliation
- historical analysis

A common design is:

```text
Raw
  |
  +--> preserve source record
  |
  v
Staging
  |
  v
Deduplicate
  |
  v
Curated
```

This allows the pipeline to preserve what the source actually sent while controlling duplicates in downstream layers.

The exact retention policy depends on the system.

---

## 16. Deduplication and Idempotency

Deduplication and idempotency work together.

Idempotency asks:

> What happens if I process this logical operation again?

Deduplication asks:

> Which records represent the same logical data?

For example:

```text
event_id = evt-1001
event_id = evt-1001
```

Idempotency can ensure that processing the same event twice does not create two results.

Now consider:

```text
event_id = evt-1001
event_id = evt-2009

transaction_reference = TX-500
transaction_reference = TX-500
```

Idempotency based on event_id will not identify them as the same operation.

Business-key deduplication may be needed.

The two mechanisms solve different problems.

---

## 17. Database-Level Deduplication

When the business key is well defined, the database can enforce uniqueness.

For example:

```sql
CREATE UNIQUE INDEX
    idx_transactions_transaction_ref
ON transactions (transaction_reference);
```

Then a duplicate insert produces a conflict.

A conflict-aware insert can prevent a second row:

```sql
INSERT INTO transactions (
    transaction_reference,
    customer_id,
    amount
)
VALUES (
    'TX-500',
    'cust-42',
    250
)
ON CONFLICT (transaction_reference) DO NOTHING;
```

This is a **generic example**.

This pattern is useful when:

- the key is trustworthy
- one record should exist per key
- the database is the authoritative target
- duplicate writes should be prevented

But not every deduplication problem can be solved with a unique constraint.

---

## 18. Cleaning Existing Duplicates

A unique constraint prevents future duplicates.

It does not automatically clean existing duplicates.

Suppose a table already contains:

```text
TX-500
TX-500
TX-500
```

Before adding a unique constraint, the existing data must be investigated.

A generic query can identify duplicate keys:

```sql
SELECT
    transaction_reference,
    COUNT(*) AS duplicate_count
FROM transactions
GROUP BY transaction_reference
HAVING COUNT(*) > 1;
```

This tells you which keys appear multiple times.

It does not tell you which record should be retained.

That requires a business rule.

---

## 19. Keep First vs Keep Latest

Suppose three records have the same logical key:

```text
TX-500
    created_at = 10:00

TX-500
    created_at = 10:05

TX-500
    created_at = 10:10
```

Which one should remain?

Possible rules:

### Keep first

Keep the earliest valid record.

### Keep latest

Keep the newest record.

### Keep most complete

Keep the record containing the most useful information.

### Merge

Combine fields from multiple records.

### Quarantine

Do not automatically decide.

Move the conflicting records to a review path.

There is no universal answer.

The correct policy depends on the source contract and business meaning.

---

## 20. Why "Keep Latest" Is Not Always Safe

A common deduplication rule is:

> Keep the latest record.

This can be wrong.

The latest record might be:

- a correction
- a partial update
- a duplicate with bad data
- a delayed event
- a replayed record
- an out-of-order event

For example:

```text
Original:
amount = 250

Later duplicate:
amount = 0
```

Keeping the latest record automatically could destroy valid information.

A deduplication rule must consider record semantics, not only timestamps.

---

## 21. Window-Based Deduplication

Streaming systems often need to deduplicate records within a time window.

For example:

```text
event_id = evt-1001
```

may appear several times within five minutes.

A streaming system can maintain recently seen keys:

```text
evt-1001 -> seen
evt-1002 -> seen
evt-1003 -> seen
```

When evt-1001 arrives again:

```text
duplicate -> ignore
```

This is a **generic streaming pattern**.

The challenge is deciding how long the system should remember keys.

If the window is too short:

```text
duplicate arrives after window
        |
        v
duplicate may pass through
```

If the window is too long:

```text
more state must be retained
```

The correct window depends on source behavior and event timing.

---

## 22. Exact vs Approximate Deduplication

There are two broad approaches.

### Exact deduplication

The system compares a well-defined key.

Example:

```text
transaction_reference = TX-500
```

This is usually easier to reason about.

### Approximate or fuzzy deduplication

The system tries to identify records that are similar.

For example:

```text
customer name
amount
date
address
```

Fuzzy matching is much more dangerous.

Two similar records may be legitimate.

Use fuzzy matching only when the business problem actually requires it and the matching rules have been carefully defined.

For a first implementation, prefer a clear stable key when one exists.

---

## 23. Deduplication Using a Fingerprint

Sometimes the source does not provide a usable identifier.

One option is to create a deterministic fingerprint from selected fields.

For example:

```text
fingerprint =
    hash(
        customer_id
        + transaction_date
        + amount
        + currency
    )
```

The important word is **deterministic**.

The same logical record must produce the same fingerprint.

A fingerprint is a **generic design pattern**.

It is not automatically correct to hash every field.

Fields such as these may change between ingestion attempts:

```text
ingestion_time
processing_time
internal database ID
```

Including them would produce different fingerprints for the same logical record.

Choose fields based on the identity definition.

---

## 24. Fingerprints Need a Clear Definition

Suppose:

```text
customer_id = cust-42
amount = 250
currency = EUR
timestamp = 10:00
```

If the timestamp is included in the fingerprint:

```text
hash(cust-42, 250, EUR, 10:00)
```

A retry at 10:01 with a changed processing timestamp may produce a different fingerprint.

The system may fail to detect the duplicate.

The fingerprint should use source or business fields that define identity.

---

## 25. Handling Conflicting Duplicates

Duplicates are not always identical.

Consider:

```text
Record A
transaction_ref = TX-500
amount = 250
status = completed

Record B
transaction_ref = TX-500
amount = 300
status = completed
```

These records share the same business key but disagree on an important value.

Do not silently choose one without a rule.

Possible handling:

```text
Duplicate detected
      |
      v
Compare fields
      |
   +--+--+
   |     |
 Same  Conflict
   |     |
   v     v
 Skip  Quarantine
```

This is often safer than automatically overwriting data.

A conflict may indicate:

- source correction
- data corruption
- schema change
- integration bug
- out-of-order processing
- incorrect business key

---

## 26. Quarantine as a Deduplication Safety Valve

When the pipeline cannot safely decide which duplicate is correct, quarantine can be used.

For example:

```text
Duplicate group
      |
      v
Can policy resolve it?
    /       \
  Yes        No
   |          |
   v          v
Process    Quarantine
              |
              v
          Investigate
```

Quarantine prevents uncertain data from silently entering a trusted layer.

It also preserves the records for later investigation.

---

## 27. Batch Deduplication

A simple batch workflow can be:

```text
Read batch
   |
   v
Validate
   |
   v
Group by logical key
   |
   v
Identify duplicate groups
   |
   v
Apply duplicate policy
   |
   v
Write clean records
```

Suppose the batch contains:

```text
TX-100
TX-101
TX-100
TX-102
TX-101
```

The logical groups are:

```text
TX-100 -> 2 records
TX-101 -> 2 records
TX-102 -> 1 record
```

The pipeline can then apply the configured rule.

For example:

```text
TX-100 -> keep one
TX-101 -> keep one
TX-102 -> keep one
```

But the actual rule must be defined before implementation.

---

## 28. Streaming Deduplication

Streaming deduplication introduces state.

The consumer may need to remember recently processed keys.

Conceptually:

```text
Incoming event
      |
      v
Extract key
      |
      v
Seen recently?
    /       \
  Yes        No
   |          |
   v          v
 Skip       Process
              |
              v
          Remember key
```

The state needs a retention policy.

Questions include:

- How long should keys be remembered?
- What happens after a restart?
- Where is the state stored?
- What happens when events arrive late?
- What happens when an old duplicate arrives?
- How is state recovered?

Streaming deduplication is therefore more than a simple set() in application memory.

---

## 29. Do Not Use an In-Memory Set as the Production Design

A simple learning example might use:

```python
seen = set()

if event_id in seen:
    return

seen.add(event_id)
process(event)
```

This is useful for understanding the idea.

But it has serious limitations.

If the process restarts:

```text
memory is lost
```

If there are two workers:

```text
Worker A has one set
Worker B has another set
```

The workers do not share duplicate state.

For production systems, deduplication state needs a design appropriate to the processing architecture.

Possible approaches include:

- database constraints
- durable state stores
- broker-supported mechanisms
- streaming state stores
- partition-local state
- warehouse merge logic

The correct choice depends on the system.

---

## 30. Deduplication and Ordering

Ordering can affect duplicate decisions.

Suppose two records arrive:

```text
event A
event B
```

Both use the same business key.

But event B may be a correction to event A.

If the system processes B first, the final result may differ from processing A first.

This is why duplicate handling and event ordering sometimes need to be designed together.

Ask:

- Is order meaningful?
- Does the source provide event time?
- Does the source provide sequence numbers?
- Can events arrive late?
- Can corrections arrive after original events?

Do not assume that the last record received is the latest business state.

---

## 31. Deduplication and Late Data

Late data can look like duplication.

Suppose:

```text
Event A
event_time = 10:00
```

arrives at 10:01.

Later another record with the same business key arrives at 10:10.

It may be:

- a duplicate
- a correction
- a late event
- a replacement
- a new business event

The pipeline needs enough information to distinguish these cases.

Useful fields may include:

```text
event_id
event_time
sequence_number
version
update_type
source_record_id
```

Again, the correct fields depend on the source contract.

---

## 32. Testing Deduplication

Deduplication should be tested as a business rule.

At minimum, test:

### Test 1 — Exact duplicate

Input:

```text
A
A
```

Expected:

```text
one logical record
```

### Test 2 — Different records

Input:

```text
A
B
```

Expected:

```text
two records
```

### Test 3 — Same business key

Input:

```text
transaction_ref = TX-500
transaction_ref = TX-500
```

Expected:

```text
duplicate policy is applied
```

### Test 4 — Similar but legitimate records

Input:

```text
customer = cust-42
amount = 250
time = 10:00

customer = cust-42
amount = 250
time = 11:00
```

Expected:

```text
both remain if the business rule says they are separate events
```

### Test 5 — Conflicting duplicates

Input:

```text
TX-500 / amount 250
TX-500 / amount 300
```

Expected:

```text
explicit conflict policy
```

Do not silently pick a record unless that behavior is part of the specification.

---

## 33. Test Existing Duplicate Data

If the feature is being added to an existing system, test with historical duplicate records.

For example:

```text
existing data
      |
      v
identify duplicates
      |
      v
apply cleanup policy
      |
      v
verify result
```

Also verify that adding a uniqueness constraint does not fail because existing data already violates it.

A migration that adds:

```sql
UNIQUE (transaction_reference)
```

can fail if duplicate values already exist.

This is why data cleanup and schema migration must be planned together.

---

## 34. Verify Duplicate Counts

After deduplication, do not only check that the job completed.

Measure the result.

For example:

```sql
SELECT
    COUNT(*) AS total_rows,
    COUNT(DISTINCT transaction_reference) AS unique_keys
FROM transactions;
```

If the business key is expected to be unique:

```text
total_rows
    =
unique_keys
```

If they differ, duplicates still exist.

This is a **generic verification query**.

Use the actual business key for the system being investigated.

---

## 35. Observability

Useful deduplication metrics can include:

```text
records_received_total
records_processed_total
duplicate_records_total
conflicting_duplicates_total
quarantined_records_total
```

These metrics help answer:

- How many records were received?
- How many were duplicates?
- How many were accepted?
- How many had conflicting values?
- How many required investigation?

A sudden increase in duplicates may indicate a source or pipeline problem.

For example:

```text
duplicate rate
      |
      v
suddenly increases
      |
      v
investigate upstream
```

The deduplication layer protects downstream data, but the metric helps identify the underlying issue.

---

## 36. Failure Scenarios

A production deduplication design should consider at least these cases.

### Failure 1 — Exact duplicate arrives

Expected:

```text
duplicate policy is applied
```

### Failure 2 — Same business key with different technical ID

Expected:

```text
business-key rule identifies the duplicate
```

### Failure 3 — Duplicate records conflict

Expected:

```text
explicit conflict policy is applied
```

### Failure 4 — Same file is ingested twice

Expected:

```text
file or record-level deduplication prevents unintended duplication
```

### Failure 5 — Replay sends old records again

Expected:

```text
replay follows the deduplication and idempotency policy
```

### Failure 6 — Database already contains duplicates

Expected:

```text
cleanup is performed before enforcing a new uniqueness rule
```

### Failure 7 — Two workers insert the same business key

Expected:

```text
database or shared state prevents an unintended duplicate
```

---

## 37. Deduplication Does Not Mean Delete

This is one of the most important lessons in this chapter.

When duplicates are found, there are several possible actions:

```text
skip
merge
update
quarantine
mark as duplicate
replace
delete
```

Deleting is only one option.

In a production system, raw records may be required for:

- audit
- reconciliation
- investigation
- replay
- compliance
- debugging

A safer architecture often separates:

```text
source preservation
        |
        v
duplicate decision
        |
        v
trusted downstream data
```

This keeps the original evidence available while controlling what enters the curated layer.

---

## 38. Practical Investigation Example

Suppose a warehouse report suddenly shows:

```text
expected transactions: 50,000
actual rows:           53,400
```

Do not immediately delete 3,400 rows.

Start by investigating.

### Step 1 — Find repeated business keys

```sql
SELECT
    transaction_reference,
    COUNT(*) AS count
FROM transactions
GROUP BY transaction_reference
HAVING COUNT(*) > 1;
```

### Step 2 — Inspect duplicate groups

Find out whether the repeated keys contain:

- identical records
- different values
- different timestamps
- corrections
- late events

### Step 3 — Trace the source

Ask:

```text
Did the source send duplicates?

Did ingestion run twice?

Did a replay run?

Did a join multiply rows?

Did a retry repeat a write?
```

### Step 4 — Identify the correct rule

Determine whether the duplicates should be:

```text
skipped
merged
updated
quarantined
retained
```

### Step 5 — Fix the source of duplication

Do not only clean the current table.

If the source of duplication remains, the problem will return.

---

## 39. Practical Implementation Sequence

For an existing repository, use an investigation-first sequence:

```text
1. Understand the target data
        |
        v
2. Define what "duplicate" means
        |
        v
3. Identify the business or source key
        |
        v
4. Find where duplicates enter
        |
        v
5. Check raw and staging data
        |
        v
6. Check transformations and joins
        |
        v
7. Inspect existing database constraints
        |
        v
8. Define the duplicate policy
        |
        v
9. Decide where deduplication belongs
        |
        v
10. Clean existing duplicates if required
        |
        v
11. Add the required protection
        |
        v
12. Implement duplicate handling
        |
        v
13. Add tests
        |
        v
14. Test conflict cases
        |
        v
15. Test replay and retry behavior
        |
        v
16. Verify final counts
        |
        v
17. Add duplicate metrics
        |
        v
18. Document the rule
```

The key step is number 2.

If the definition of a duplicate is wrong, perfect code will still produce the wrong result.

---

## 40. Definition of Done

The deduplication implementation is complete when:

- [ ] A duplicate is clearly defined.
- [ ] The logical or business key is identified.
- [ ] Legitimate similar records are distinguishable from duplicates.
- [ ] The source of duplication has been investigated.
- [ ] The deduplication location is explicitly chosen.
- [ ] The duplicate policy is documented.
- [ ] Existing duplicates have been assessed.
- [ ] Existing duplicates are cleaned when required.
- [ ] Database uniqueness is enforced where appropriate.
- [ ] Conflicting duplicates have an explicit policy.
- [ ] Replay behavior has been considered.
- [ ] Retry behavior has been considered.
- [ ] Exact duplicates are tested.
- [ ] Legitimate similar records are tested.
- [ ] Conflicting duplicates are tested.
- [ ] Concurrent processing is tested where relevant.
- [ ] Final row counts are verified.
- [ ] Duplicate metrics are available where appropriate.
- [ ] Raw evidence is preserved when required.
- [ ] The deduplication rule is documented for future engineers.

---

## 41. What You Learned

Deduplication is not simply removing repeated rows.

It is the process of deciding when multiple records represent the same logical data and then applying a deliberate policy.

The important flow is:

```text
Define identity
      |
      v
Find duplicates
      |
      v
Understand why they exist
      |
      v
Apply an explicit policy
      |
      v
Verify the result
```

The most important lesson is:

> Never delete a record merely because it looks similar to another record.

First determine what makes the record unique.

Then determine what should happen when that uniqueness rule is violated.

Idempotency and deduplication should also be understood together:

```text
Idempotency
    |
    v
Repeated processing is safe

Deduplication
    |
    v
Duplicate logical records are identified
```

A reliable pipeline often needs both.

---

# Chapter 22 Preview

The next recipe will add **Processing Status**.

We will build a clear state model for pipeline records:

```text
pending
   |
   v
processing
   |
   +------> failed
   |
   v
completed
```

We will cover:

- why processing status matters
- status columns
- state transitions
- valid and invalid transitions
- pending records
- processing records
- completed records
- failed records
- retry state
- stale processing records
- worker crashes
- transaction boundaries
- concurrent workers
- status history
- testing state transitions
- recovery from stuck records
- production observability

The key question will be:

> How does the pipeline know what happened to each record?## Production Tools You Should Know

These tools solve or provide production implementations of concepts covered in this recipe. Learn what the tool provides, but understand the underlying problem first.

| Tool | What to know |
|---|---|
| **Apache Kafka Streams** | Stateful stream processing and deduplication patterns. |
| **Apache Flink** | Stateful streaming deduplication. |
| **Apache Spark** | Batch and streaming deduplication at scale. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the engineering mechanism.

---


