# Chapter 7 — Data Quality

A pipeline can run successfully and still produce bad data.

This is one of the most important ideas in Data Engineering.

Imagine a pipeline that:

- Connected to the source successfully
- Read all expected files
- Processed every record
- Inserted everything into PostgreSQL
- Returned no application errors

The pipeline may look healthy.

But what if:

- 20% of records are missing?
- A required field is empty?
- The same event appears five times?
- A date is in the wrong format?
- A payment amount is negative when it should not be?
- A foreign key points to a record that does not exist?

The pipeline technically succeeded.

The data did not.

Data quality is the set of checks and practices used to determine whether data is correct, complete, consistent, valid, and useful for its intended purpose.

This chapter explains the main data quality concepts and how to build them into a pipeline.

---

# What Is Data Quality?

Data quality asks a simple question:

> Can we trust the data for the purpose we want to use it for?

Trust does not mean that every value must be perfect.

It means the data satisfies the rules that matter for the system or business use case.

For example, an email address may need to:

- Exist
- Be stored as text
- Follow an accepted format

A payment amount may need to:

- Exist
- Be numeric
- Be greater than or equal to zero
- Use a supported currency

A customer identifier may need to:

- Exist
- Be unique where required
- Refer to a valid customer

Different datasets need different quality rules.

---

# Why Data Quality Matters

Bad data can move through an entire system without causing a technical error.

Consider:

```text
Source
  |
  v
Ingestion
  |
  v
Validation
  |
  v
Storage
  |
  v
Reporting
  |
  v
Business decision
```

If bad data is not detected near the beginning, it can affect every later stage.

This makes data quality part of pipeline reliability.

Good data quality checks can help detect:

- Missing data
- Duplicate data
- Invalid values
- Unexpected schema changes
- Broken relationships
- Late data
- Sudden volume changes
- Corrupted records

---

# Data Quality Is Not the Same as Data Validation

These terms are related but are not always used in exactly the same way.

Validation often happens while data is entering or moving through a pipeline.

For example:

```text
Incoming record
      |
      v
Validate fields
      |
   +--+--+
   |     |
 valid  invalid
   |     |
   v     v
Process Quarantine
```

Data quality can be broader.

It may include checks across an entire dataset or across multiple tables.

For example:

- Are all expected records present?
- Are customer IDs unique?
- Do all orders have valid customers?
- Did today's volume change unexpectedly?
- Is the dataset fresh enough?

A pipeline can therefore use validation and data quality checks at different stages.

---

# The Main Data Quality Dimensions

There is no single universal list of quality dimensions.

For this book, we will focus on the dimensions that are especially useful when building pipelines:

1. Completeness
2. Uniqueness
3. Validity
4. Referential integrity
5. Freshness
6. Consistency
7. Accuracy, where it can be meaningfully verified
8. Volume and anomaly checks

These dimensions overlap in some cases.

The important thing is to define the actual rule that the pipeline needs to enforce.

---

# 1. Completeness

Completeness asks:

> Did we receive and process the data we expected?

Completeness can be checked at different levels.

## Record completeness

Did all required records arrive?

For example, if a source says there should be 10,000 records and only 9,200 arrive, the pipeline should detect the difference.

## Field completeness

Are required fields populated?

For example:

```text
customer_id = NULL
```

If `customer_id` is required, the record is incomplete.

---

# Completeness Is Context Dependent

Not every empty value is a quality problem.

Some fields are optional.

For example:

```text
customer_id: 12345
middle_name: null
```

`middle_name` may legitimately be empty.

Therefore, the rule should be:

> This field must exist for this type of record.

Not:

> No field is ever allowed to be empty.

Quality rules need business context.

---

# Completeness Checks

A simple SQL check for required values might look like this:

```sql
SELECT COUNT(*)
FROM customer
WHERE customer_id IS NULL;
```

**GENERIC EXAMPLE**

If the result is greater than zero, the dataset contains records without the required identifier.

Another example:

```sql
SELECT COUNT(*)
FROM orders
WHERE order_id IS NULL
   OR created_at IS NULL;
```

These queries are examples of the type of checks a pipeline can perform.

They are not claims about a specific repository.

---

# 2. Uniqueness

Uniqueness asks:

> Does a value that should be unique appear more than once?

For example, an event may have a unique event ID.

Expected:

```text
event_001
event_002
event_003
```

Problem:

```text
event_001
event_002
event_002
```

If the event ID is supposed to be unique, the dataset contains a duplicate.

---

# Finding Duplicates

A common SQL pattern is:

```sql
SELECT event_id, COUNT(*)
FROM events
GROUP BY event_id
HAVING COUNT(*) > 1;
```

**GENERIC EXAMPLE**

This identifies values that appear more than once.

Uniqueness is especially important for:

- Event IDs
- Transaction IDs
- Customer IDs
- Order IDs
- Source record IDs
- Idempotency keys

---

# Uniqueness and Idempotency

Uniqueness and idempotency are closely related.

A uniqueness constraint can help prevent the same logical record from being stored multiple times.

For example:

```sql
CREATE UNIQUE INDEX ...;
```

The exact index definition depends on the table and business key.

**GENERIC EXAMPLE**

But uniqueness alone does not solve every duplicate problem.

The pipeline still needs to decide:

- What identifies the same logical event?
- At which stage should duplicates be removed?
- Should duplicates be ignored?
- Should duplicates be recorded?
- Should conflicting duplicates be investigated?

These are pipeline design decisions.

---

# 3. Validity

Validity asks:

> Does the value follow the rules defined for that field?

Examples:

| Field | Example validity rule |
|---|---|
| Age | Must be within an allowed range |
| Currency | Must be a supported code |
| Status | Must be one of the allowed values |
| Amount | Must be numeric |
| Date | Must be a valid date |
| Email | Must follow the accepted format |

These are examples only.

The actual rules depend on the dataset.

---

# Allowed Values

Suppose a status field supports:

```text
pending
completed
failed
```

Now a record contains:

```text
status = "finished"
```

The value may be understandable to a human, but it is not part of the defined contract.

That makes it invalid for the expected schema.

A simple check could be:

```sql
SELECT COUNT(*)
FROM events
WHERE status NOT IN ('pending', 'completed', 'failed');
```

**GENERIC EXAMPLE**

---

# Range Checks

Some values must stay inside a defined range.

For example:

```text
percentage >= 0
percentage <= 100
```

Or:

```text
quantity > 0
```

A range check catches values that are structurally valid but logically outside the allowed range.

---

# 4. Referential Integrity

Referential integrity asks:

> Do relationships between records remain valid?

Consider two tables:

```text
customers
---------
customer_id

orders
------
order_id
customer_id
```

An order references a customer.

If an order contains:

```text
customer_id = 99999
```

but customer `99999` does not exist, the relationship is broken.

That is a data quality problem.

---

# Database Constraints

A relational database can enforce some relationships directly.

For example, a foreign key can prevent references to non-existent records.

**GENERIC EXAMPLE**

```sql
FOREIGN KEY (customer_id)
REFERENCES customers(customer_id)
```

This is valuable because the database becomes another layer of protection.

However, not every quality rule can be represented as a database constraint.

Some checks require pipeline logic or dataset-level analysis.

---

# 5. Freshness

Freshness asks:

> How old is the data?

Some pipelines can tolerate data that is several hours old.

Others may require data within a few minutes.

Freshness is therefore a requirement, not a universal number.

Suppose the latest event arrived at 09:00.

The current time is 12:00.

Data age is approximately:

```text
12:00 - 09:00 = 3 hours
```

If the expected maximum age is 30 minutes, the dataset is stale.

---

# Freshness Is Different from Completeness

A dataset can be complete but stale.

For example:

```text
Expected records: 100,000
Received records: 100,000
Latest record: yesterday
```

Completeness looks good.

Freshness does not.

The opposite can also happen.

You may receive recent records but miss a large part of the expected data.

Therefore, both checks may be needed.

---

# 6. Consistency

Consistency asks whether the same information follows the same rules across the dataset or across systems.

Suppose one system stores a country as:

```text
Pakistan
```

Another uses:

```text
PK
```

Another uses:

```text
PAK
```

All may represent the same country.

But if downstream systems expect one standard representation, inconsistent values create problems.

Consistency checks help detect these differences.

---

# 7. Accuracy

Accuracy asks:

> Does the data represent the real-world value correctly?

This is often harder to verify than completeness or uniqueness.

For example, a pipeline can verify that a phone number is present and has an accepted format.

It cannot automatically prove that the phone number actually belongs to the customer.

That distinction matters.

A value can be:

- Complete
- Unique
- Valid according to format

and still be inaccurate.

Accuracy often requires an external source, business verification, reconciliation, or another trusted reference.

Do not claim that a pipeline has verified accuracy when it has only checked format or structure.

---

# 8. Volume Checks

Volume checks ask:

> Did the amount of data change in an unexpected way?

Suppose a pipeline normally receives around 100,000 events per day.

One day it receives:

```text
1,200 events
```

The pipeline may have technically completed successfully.

But the large change should be investigated.

Volume checks can detect:

- Missing source data
- Broken extraction
- Source outages
- Unexpected spikes
- Duplicate ingestion
- Filtering bugs

---

# Anomaly Detection

Volume checks are one simple form of anomaly detection.

The idea is to compare current behavior with expected behavior.

For example:

```text
Yesterday: 98,400
Today:     99,100
Tomorrow:  1,900
```

The sudden drop is a signal.

More advanced systems can use historical distributions, thresholds, statistical methods, or machine learning.

This book starts with simpler checks because they are easier to understand, test, and operate.

---

# Where Should Quality Checks Run?

Data quality checks can run at multiple stages.

A useful model is:

```text
Source
  |
  v
Acquisition
  |
  v
Raw
  |
  v
Validation
  |
  v
Staging
  |
  v
Curated
  |
  v
Warehouse
```

Different checks belong at different boundaries.

For example:

- Schema validation can happen when data enters the pipeline.
- Required-field checks can happen during staging.
- Referential integrity can be checked after related datasets are available.
- Freshness can be monitored continuously.
- Warehouse-level reconciliation can run after loading.

There is no need to put every check in one function.

---

# Fail Fast vs Quarantine

When bad data is detected, the pipeline needs a response.

Two common approaches are:

## Fail the operation

The pipeline stops or marks the processing unit as failed.

This can be appropriate when continuing would make the resulting data unsafe.

## Quarantine the bad data

Valid records continue while invalid records are separated.

This can be useful when one bad record should not block thousands of valid records.

The correct choice depends on the data and business consequences.

---

# Example: Record-Level Validation

Suppose a pipeline receives three events:

```text
Event A -> valid
Event B -> missing event_id
Event C -> valid
```

A record-level validation strategy could produce:

```text
Valid:
  Event A
  Event C

Invalid:
  Event B
```

Event B can be quarantined for investigation.

Again, this is a generic example.

Whether a real pipeline behaves this way must be verified from its implementation.

---

# Quality Checks and the Raw Layer

The raw layer can help preserve the original source data before transformation.

This is useful when a validation rule changes later.

Consider:

```text
Source data
    |
    v
Raw storage
    |
    v
Validation
    |
    v
Curated data
```

If a record fails validation, the original data may still be available for investigation or reprocessing, depending on the storage and retention design.

This is one reason raw data and processed data are often separated.

---

# Data Quality and Schema Changes

A source can change its schema.

For example:

```text
Before:
customer_id
name
email

After:
customer_id
name
email
country
```

Adding a field may be harmless if the pipeline allows optional fields.

Other changes can be breaking.

For example:

```text
amount: number
      ->
amount: object
```

A pipeline should understand which schema changes are compatible and which require code or contract changes.

Data quality checks can help detect unexpected changes.

---

# Data Quality and Contracts

A data contract defines expectations between producers and consumers.

A simple contract may define:

- Required fields
- Data types
- Allowed values
- Identifier rules
- Optional fields
- Event version
- Semantics

Quality checks can enforce parts of that contract.

Conceptually:

```text
Producer
   |
   v
Data Contract
   |
   v
Validation
   |
   v
Consumer
```

This reduces the chance that an unexpected change silently reaches downstream systems.

---

# Data Quality and Transactions

Database transactions can protect atomic changes.

For example:

```text
Begin
  |
  v
Write records
  |
  v
Run required checks
  |
  v
Commit
```

If a critical check fails before commit, the transaction may be rolled back.

However, not every quality check belongs inside a database transaction.

Large dataset checks can be expensive.

Some checks are better handled as separate validation or monitoring jobs.

The quality strategy should match the size and purpose of the check.

---

# Quality Checks Should Produce Evidence

A quality check should not only return:

```text
PASS
```

when possible, it should provide useful evidence.

For example:

```text
Check: required customer_id
Status: FAILED
Invalid records: 37
Checked at: 2026-09-26 10:00
```

The exact output format depends on the system.

The important point is that failures should be explainable.

---

# Quality Check Results

A useful quality system can track:

- Check name
- Dataset
- Check type
- Execution time
- Result
- Number of records checked
- Number of failures
- Failure percentage
- Threshold
- Error details

This makes quality measurable over time.

---

# Thresholds

Not every quality failure should stop a pipeline.

Suppose a dataset contains 1,000,000 records and 2 records have an optional field missing.

That may be acceptable.

Suppose 400,000 records are missing the same required field.

That is a much larger problem.

A quality rule may therefore have a threshold.

For example:

```text
Invalid records <= allowed threshold
    -> continue

Invalid records > allowed threshold
    -> fail or alert
```

The threshold must be defined according to the actual requirement.

Do not invent a threshold simply because a number is convenient.

---

# Quality Gates

A **quality gate** is a point in the pipeline where data must satisfy required checks before moving forward.

For example:

```text
Raw
 |
 v
Quality Gate
 |
+---- fail ----> Quarantine
 |
 v
Staging
```

A quality gate can protect downstream systems from known bad data.

Not every pipeline needs a formal gate at every stage.

Use gates where a quality failure would have meaningful downstream consequences.

---

# Data Quality in Batch Pipelines

Batch pipelines often have natural quality boundaries.

For example:

```text
Daily source
    |
    v
Load batch
    |
    v
Run quality checks
    |
    +---- fail ----> Alert / stop
    |
    v
Publish batch
```

A batch should ideally not be marked successful simply because the processing code finished.

Quality checks can be part of the batch completion criteria.

---

# Data Quality in Streaming Pipelines

Streaming systems have different constraints.

Data arrives continuously.

You may not be able to inspect the entire dataset before accepting each event.

Instead, checks may run at several levels:

- Event-level validation
- Window-level counts
- Aggregate monitoring
- Consumer lag
- Late-event checks
- Duplicate detection

For example:

```text
Event
  |
  v
Validate
  |
  v
Process
  |
  v
Store

Every few minutes:
  |
  v
Check volume / freshness / anomalies
```

Streaming quality often combines immediate validation with continuous monitoring.

---

# Late Data

Late data is data that arrives after the expected processing time.

Consider an event that occurred at 09:00 but arrives at 09:20.

```text
Event time:      09:00
Arrival time:    09:20
Processing time: 09:21
```

The event may still be valid.

It is simply late.

A quality system should distinguish:

- Invalid data
- Duplicate data
- Late but valid data

Treating all late data as invalid can cause unnecessary data loss.

---

# Reconciliation

Reconciliation compares two sources or stages to determine whether they agree.

For example:

```text
Source system
Records: 100,000
Amount: 50,000,000

Pipeline output
Records: 99,950
Amount: 49,800,000
```

The difference requires investigation.

Reconciliation is especially useful for financial, operational, and regulated workflows where totals need to agree across systems.

The exact reconciliation rules depend on the domain.

---

# Data Quality and Observability

Data quality is part of pipeline observability.

Traditional observability often focuses on:

- Logs
- Metrics
- Traces

Data pipelines also need visibility into the data itself.

For example:

```text
System health
    +
    |
    +-- Logs
    +-- Metrics
    +-- Traces
    +-- Data quality
```

A pipeline can be technically healthy while producing unhealthy data.

That is why system observability and data observability complement each other.

---

# What Should Happen When a Quality Check Fails?

The answer depends on severity.

Possible responses include:

- Reject the record
- Quarantine the record
- Fail the batch
- Stop downstream publication
- Mark the dataset as degraded
- Raise an alert
- Continue processing and record the issue
- Start a recovery workflow

The response should be defined before production.

Otherwise, operators have to make the decision during an incident.

---

# Testing Data Quality

Quality rules are code or configuration.

They should therefore be tested.

## Test 1 — Valid data

Provide valid records.

Verify that they pass.

## Test 2 — Missing required field

Remove a required value.

Verify that the record fails the expected check.

## Test 3 — Duplicate identifier

Send the same identifier twice.

Verify that uniqueness behavior works as expected.

## Test 4 — Invalid value

Use a value outside the accepted set or range.

Verify that validation detects it.

## Test 5 — Broken relationship

Reference a non-existent parent record.

Verify that referential integrity is detected or enforced.

## Test 6 — Stale data

Use a timestamp older than the allowed freshness window.

Verify that the freshness check reports the problem.

## Test 7 — Volume anomaly

Simulate an unusually small or large input.

Verify that the expected monitoring or quality response occurs.

---

# Testing the Quality Checks Themselves

There is another important question:

> How do we know the quality checks are correct?

A broken quality check can create false confidence.

For example, suppose a test checks:

```text
invalid_count == 0
```

but the query accidentally ignores half of the table.

The test may pass while the real dataset contains bad records.

Quality checks therefore need their own code review and tests.

---

# Common Mistakes

## Mistake 1: Checking only whether the pipeline ran

A successful job does not automatically mean good data.

## Mistake 2: Treating every NULL as an error

Some fields are optional.

Define required fields explicitly.

## Mistake 3: Using format checks as proof of accuracy

A value can have the correct format and still be wrong.

## Mistake 4: Checking only one record

Dataset-level problems may only appear when looking at the whole dataset.

## Mistake 5: No historical comparison

A dataset can pass basic validation while its volume suddenly drops.

Volume and anomaly checks can catch this.

## Mistake 6: Quality checks with no response

A failed check is not useful if nobody knows what happens next.

Define the response.

## Mistake 7: Putting every check into one giant validation function

Separate checks by purpose so they are easier to understand, test, and operate.

## Mistake 8: Silently discarding bad records

Rejected records may need investigation or recovery.

Do not make data disappear without a clear policy.

---

# Production Considerations

A production data quality strategy should define:

- Which fields are required
- Which fields are optional
- Which values are allowed
- Which identifiers must be unique
- Which relationships must exist
- Freshness requirements
- Volume expectations
- Reconciliation rules
- Quality thresholds
- Quality gate behavior
- Failure handling
- Quarantine behavior
- Alerting
- Retention of failed records
- Recovery and replay procedures

Teams should also know which checks are:

- Blocking
- Warning-only
- Informational

This prevents confusion during incidents.

---

# Practical Data Quality Example

Consider a generic payment-event pipeline.

An incoming event might conceptually contain:

```text
event_id
account_id
amount
currency
occurred_at
event_type
```

Possible quality rules could be:

| Field / Dataset | Example rule |
|---|---|
| event_id | Required and unique |
| account_id | Required |
| amount | Numeric and within allowed range |
| currency | Supported value |
| occurred_at | Valid timestamp |
| event_type | Allowed value |
| Events | Expected volume |
| Events | Fresh enough for the use case |

These are generic examples.

The actual rules for a real payment system must come from its data contract and business requirements.

---

# A Simple Quality Architecture

A practical pipeline can organize quality checks like this:

```text
                Source
                  |
                  v
              Acquisition
                  |
                  v
                 Raw
                  |
                  v
          +------------------+
          | Quality Checks   |
          +------------------+
             |            |
          valid         invalid
             |            |
             v            v
          Staging     Quarantine
             |
             v
          Curated
             |
             v
         Warehouse
```

This design separates normal processing from failed data.

It also creates clear points where quality can be measured.

---

# Practical Checklist

Before calling a pipeline production-ready, ask:

- [ ] Are required fields defined?
- [ ] Are optional fields defined?
- [ ] Are data types checked?
- [ ] Are allowed values defined?
- [ ] Are uniqueness rules defined?
- [ ] Are duplicate records detected or prevented?
- [ ] Are relationships checked?
- [ ] Is freshness measured?
- [ ] Is expected volume understood?
- [ ] Are anomalies detected?
- [ ] Are reconciliation checks needed?
- [ ] Are quality thresholds defined?
- [ ] Are quality gates defined?
- [ ] Is the response to a failed check defined?
- [ ] Are invalid records preserved when needed?
- [ ] Can failed data be investigated?
- [ ] Can failed data be replayed or recovered?
- [ ] Are quality checks tested?
- [ ] Are quality failures observable?
- [ ] Are quality results retained where useful?

---

# What You Learned

In this recipe, you learned:

- A pipeline can succeed technically while producing bad data.
- Data quality measures whether data is trustworthy for its intended use.
- Completeness checks whether expected data is present.
- Uniqueness checks identify unexpected duplicates.
- Validity checks whether values follow defined rules.
- Referential integrity protects relationships between records.
- Freshness measures how old the available data is.
- Consistency checks whether data follows the same expected representation.
- Accuracy is harder to prove than format correctness.
- Volume and anomaly checks can detect unexpected changes.
- Quality checks can run at different pipeline stages.
- Bad records can sometimes be rejected or quarantined instead of stopping all processing.
- Quality gates can protect downstream systems.
- Data quality is part of pipeline observability.
- Quality checks themselves need testing.

The main lesson is:

**A pipeline is not healthy just because it finishes. A reliable pipeline also checks whether the data it produced is complete, valid, consistent, fresh, and fit for its intended use.**

---
