# Recipe 26 — Quarantine Failed Data

Not every failed record should be retried forever.

Some records are invalid.

Some contain unexpected values.

Some violate business rules.

Some cannot be processed because the source data is incomplete.

Some failures need human investigation before processing can continue.

If these records remain inside the normal processing path, they can repeatedly fail and block useful work.

A quarantine mechanism gives these records a separate path.

The basic idea is:

~~~text
normal pipeline
      |
      v
   process
      |
      +------ success ------> completed
      |
      +------ failure ------> classify
                                  |
                                  v
                              quarantine
~~~

Quarantine is not simply a place to put bad data.

It is a controlled recovery boundary.

A good quarantine system should preserve the failed record, explain why it failed, prevent repeated processing, and provide a path back into the pipeline when the problem is fixed.

---

## 1. Goal

The goal is to build a controlled quarantine path for records that cannot safely continue through normal processing.

By the end of this recipe, you should understand how to:

- identify data that should be quarantined
- distinguish quarantine from retry
- distinguish quarantine from rejection
- preserve the original failed data
- store useful failure metadata
- design quarantine storage
- prevent repeated processing
- review quarantined records
- correct invalid data
- replay quarantined records
- track quarantine state
- define retention
- protect sensitive information
- monitor quarantine volume
- test quarantine behavior
- recover quarantined records safely

The central idea is:

> Quarantine should isolate failed data without losing the information needed to understand and recover it.

---

## 2. Problem

Consider a pipeline processing incoming records:

~~~text
Record A -> success
Record B -> success
Record C -> invalid
Record D -> success
Record E -> invalid
~~~

If the pipeline treats every failure as a retryable error:

~~~text
Record C
   |
   v
retry
   |
   v
invalid
   |
   v
retry
   |
   v
invalid
~~~

The same record can fail repeatedly.

This wastes resources.

It also creates noise.

A better approach is:

~~~text
Record C
   |
   v
validation failure
   |
   v
quarantine
~~~

The normal pipeline can continue:

~~~text
Record A -> completed
Record B -> completed
Record C -> quarantined
Record D -> completed
Record E -> quarantined
~~~

The failed records remain available for investigation and recovery.

---

## 3. Why This Matters

Quarantine protects the normal processing path from records that cannot currently be processed safely.

It helps with:

- invalid input
- malformed records
- schema mismatches
- business-rule failures
- corrupted source data
- unsupported values
- permanent processing failures
- manual review cases

Without quarantine, teams often fall back to unsafe alternatives:

- deleting records
- repeatedly retrying permanent failures
- manually editing production tables
- ignoring failed records
- stopping the entire pipeline

A quarantine path gives the failure a defined place to go.

---

## 4. Quarantine vs Retry

Retry and quarantine solve different problems.

### Retry

Retry asks:

> Could the same operation succeed later without changing the input?

Example:

~~~text
API timeout
   |
   v
wait
   |
   v
retry
~~~

### Quarantine

Quarantine asks:

> Can this record safely continue through the normal pipeline right now?

Example:

~~~text
invalid currency
   |
   v
quarantine
~~~

A useful decision model is:

~~~text
failure
   |
   v
classify
   |
   +---- temporary ----> retry
   |
   +---- permanent ----> quarantine
   |
   +---- unknown -------> investigate / controlled failure path
~~~

The exact classification depends on the system.

---

## 5. Quarantine vs Rejection

Quarantine and rejection can also be different.

### Rejection

A record may be rejected immediately when the system knows it is not acceptable.

For example:

~~~text
invalid request
   |
   v
reject
~~~

The source may receive an error response.

### Quarantine

A record may be accepted into the system but isolated for later investigation or recovery.

For example:

~~~text
received record
      |
      v
stored safely
      |
      v
processing failure
      |
      v
quarantine
~~~

This distinction is useful because quarantine usually implies that the record remains available for recovery.

---

## 6. When to Use Quarantine

Quarantine can be appropriate when:

- validation fails permanently
- required data is missing
- the schema is unsupported
- a business rule is violated
- a record cannot be transformed
- a downstream dependency cannot accept the record
- processing requires manual review
- a permanent failure has exhausted retries
- the record needs correction before processing

These are general examples.

The actual quarantine policy should be defined by the pipeline.

---

## 7. When Not to Quarantine

Not every error should become a quarantined record.

For example, a temporary database outage may not require quarantine:

~~~text
database unavailable
      |
      v
retry
~~~

A worker crash may not require quarantine either:

~~~text
worker crash
      |
      v
recover processing state
~~~

Quarantine should represent a meaningful processing outcome.

Do not use it as a general-purpose error bucket.

---

## 8. Quarantine Architecture

A simple architecture looks like this:

~~~text
                 Incoming record
                       |
                       v
                   validate
                       |
                       v
                   process
                       |
              +--------+--------+
              |                 |
           success            failure
              |                 |
              v                 v
          completed          classify
                                |
                     +----------+----------+
                     |                     |
                  retryable             permanent
                     |                     |
                     v                     v
                   retry              quarantine
                                           |
                                           v
                                      investigate
                                           |
                              +------------+------------+
                              |                         |
                           correct                   discard
                              |
                              v
                           replay
~~~

The quarantine path should eventually lead to a defined decision.

---

## 9. Before You Start

Before implementing quarantine, investigate the existing pipeline.

Answer:

1. Where does processing fail?
2. Which errors are permanent?
3. Which errors are temporary?
4. Where is the original input stored?
5. Is raw data preserved?
6. How is processing status stored?
7. How are errors currently recorded?
8. Where should quarantined data live?
9. Who or what reviews quarantined records?
10. How can a quarantined record be corrected?
11. How can it be replayed?
12. How long should quarantine data be retained?
13. Does quarantine contain sensitive information?
14. How will quarantine volume be monitored?

Do not create quarantine storage before understanding the existing failure path.

---

## 10. Preserve the Original Record

A quarantine system is only useful if the failed data can be investigated.

Suppose a record fails because:

~~~text
currency = "XYZ"
~~~

If the system stores only:

~~~text
error = invalid currency
~~~

the original input may be lost.

A better design preserves enough information to understand the failure.

Depending on the data sensitivity and storage model, this may include:

- original payload
- source identifier
- event identifier
- failure reason
- failure type
- processing timestamp
- source information
- processing version
- schema version

Do not copy sensitive information into additional locations without a clear reason.

---

## 11. Quarantine Metadata

A generic quarantine record might contain:

~~~text
quarantine_id
source_id
event_id
payload
failure_type
failure_message
status
attempt_count
created_at
quarantined_at
resolved_at
processing_version
schema_version
~~~

This is a **generic example**.

The exact schema depends on the pipeline.

The important principle is:

> Store enough information to investigate and recover the record without creating unnecessary copies of sensitive data.

---

## 12. Quarantine Status

A quarantine record can have states such as:

~~~text
quarantined
     |
     v
under_review
     |
     +------> corrected
     |           |
     |           v
     |        replay_pending
     |           |
     |           v
     |        completed
     |
     +------> rejected
     |
     +------> discarded
~~~

These are generic example states.

The actual state machine should reflect the recovery process.

---

## 13. Why Quarantine Needs a State Machine

Without clear states, teams can lose track of what happened.

For example:

~~~text
record is quarantined
~~~

But nobody knows whether:

- someone reviewed it
- it was corrected
- replay was attempted
- replay succeeded
- replay failed again
- it was intentionally discarded

A state machine makes the lifecycle visible.

---

## 14. Quarantine and Processing Status

The normal processing status and quarantine status can be separate.

For example:

~~~text
processing_status = failed
quarantine_status = quarantined
~~~

Or the system can use one unified state model.

Both approaches can work.

The important requirement is that the state meaning is clear.

Avoid having two fields that can contradict each other without defined rules.

---

## 15. Quarantine and Retry Exhaustion

A record may enter quarantine after retry attempts are exhausted.

For example:

~~~text
pending
   |
   v
processing
   |
   X
retry_pending
   |
   v
processing
   |
   X
retry_pending
   |
   v
maximum attempts reached
   |
   v
quarantined
~~~

This is one common pattern.

However, a permanent validation error may go directly to quarantine without retrying.

---

## 16. Validation Failure

Consider an incoming record:

~~~json
{
  "id": 1001,
  "currency": "INVALID"
}
~~~

Suppose the pipeline only supports:

~~~text
EUR
USD
GBP
~~~

The validation layer can identify the problem:

~~~text
invalid currency
~~~

Instead of retrying the same input:

~~~text
validate
   |
   X
invalid
   |
   v
quarantine
~~~

The record can later be corrected if the source value was wrong.

---

## 17. Schema Mismatch

Schema mismatch is another common quarantine case.

Suppose the pipeline expects:

~~~text
user_id
amount
currency
timestamp
~~~

But the incoming record contains:

~~~text
user
amount
currency
timestamp
~~~

If the pipeline cannot safely map user to user_id, it should not guess.

Possible flow:

~~~text
schema mismatch
      |
      v
quarantine
      |
      v
investigate source change
~~~

This is safer than silently producing incorrect data.

---

## 18. Business-Rule Failure

A record can be structurally valid but still violate a business rule.

For example:

~~~text
amount = -500
~~~

If negative values are not allowed for the operation, the record may be quarantined.

This is different from a malformed record.

The quarantine metadata should preserve the reason clearly.

For example:

~~~text
failure_type = business_rule
failure_message = amount must be positive
~~~

The exact classification depends on the system.

---

## 19. Quarantine Storage

There are several possible storage approaches.

### Same database

~~~text
main database
   |
   +--> normal tables
   |
   +--> quarantine table
~~~

This can be convenient for small or moderate workloads.

### Separate schema

~~~text
database
   |
   +--> application schema
   |
   +--> quarantine schema
~~~

This provides clearer logical separation.

### Separate storage system

For large raw payloads, quarantine data may be stored separately from operational tables.

The correct choice depends on:

- data volume
- payload size
- retention
- access patterns
- security
- operational requirements

Do not choose storage only because it is technically available.

---

## 20. Quarantine Table Example

A generic PostgreSQL table could look like:

~~~sql
CREATE TABLE quarantine_records (
    quarantine_id BIGSERIAL PRIMARY KEY,
    source_id TEXT,
    event_id TEXT,
    payload JSONB,
    failure_type TEXT NOT NULL,
    failure_message TEXT,
    status TEXT NOT NULL,
    attempt_count INTEGER NOT NULL DEFAULT 0,
    quarantined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at TIMESTAMPTZ
);
~~~

This is a **generic example**.

Before using such a design, decide:

- which identifiers are stable
- whether payload should be stored
- which fields require indexes
- retention requirements
- security controls
- uniqueness rules
- whether processing versions are needed

---

## 21. Do Not Automatically Copy Everything

A common mistake is copying the entire original record into quarantine without considering sensitivity.

Suppose the payload contains:

- personal information
- authentication data
- documents
- financial information
- tokens
- credentials

Copying it into another table increases the number of places where sensitive information exists.

Before storing payloads, determine:

- what information is necessary
- whether sensitive fields can be removed
- who can access quarantine
- how long it must be retained
- whether encryption is required

Quarantine is not an exception to security rules.

---

## 22. Quarantine Access

Quarantine data often contains failed production records.

Access should therefore be controlled.

Possible controls include:

- database permissions
- application-level authorization
- restricted operational roles
- audit logging
- encryption
- limited read access

Do not make quarantine publicly accessible just because it is operational data.

---

## 23. Quarantine Reason

A useful quarantine record should answer:

> Why was this record quarantined?

Avoid vague messages such as:

~~~text
processing failed
~~~

Prefer meaningful categories:

~~~text
validation_error
schema_mismatch
business_rule_error
unsupported_value
retry_exhausted
manual_review
~~~

The exact values depend on the system.

The reason should be structured when possible.

---

## 24. Quarantine Error Message

The error message should provide enough information for investigation.

For example:

~~~text
expected currency in {EUR, USD, GBP}; received XYZ
~~~

But error messages should not expose secrets or unnecessary sensitive data.

Do not include:

~~~text
password=...
access_token=...
full_document=...
~~~

unless there is an explicitly justified and secure reason.

---

## 25. Quarantine and Error Classification

Chapter 23 introduced error classification.

Quarantine should consume that classification.

A simple model is:

~~~text
error
 |
 +---- retryable
 |
 +---- permanent
 |
 +---- unknown
          |
          v
       investigate
~~~

Permanent failures can move to quarantine.

Unknown failures may require a safer path until the team understands them.

Do not automatically quarantine every unexpected exception without understanding its consequences.

---

## 26. Quarantine and Idempotency

Moving a record into quarantine should also be safe.

Suppose the worker fails after writing the quarantine record but before updating processing status.

The operation may be attempted again.

Without idempotency:

~~~text
same record
   |
   +----> quarantine row A
   |
   +----> quarantine row B
~~~

This creates duplicate quarantine entries.

A stable identifier can help:

~~~text
source/event identifier
        |
        v
unique quarantine identity
~~~

The exact uniqueness rule depends on whether the same logical record can be quarantined more than once.

---

## 27. Multiple Quarantine Events

Sometimes the same logical record can fail more than once for different reasons.

For example:

~~~text
attempt 1 -> schema mismatch
attempt 2 -> database failure
attempt 3 -> business rule failure
~~~

A single quarantine row may not represent this history well.

Another design is:

~~~text
record
  |
  +--> quarantine event 1
  +--> quarantine event 2
  +--> quarantine event 3
~~~

This preserves history.

The correct model depends on whether quarantine represents:

- the current state
- each quarantine event
- both

Define this before implementation.

---

## 28. Correcting Quarantined Data

Quarantine is useful only if recovery is possible.

A common recovery flow is:

~~~text
quarantined
     |
     v
review
     |
     v
identify problem
     |
     v
correct source/data
     |
     v
replay
~~~

The correction should be traceable.

Avoid silently changing the original data without recording what happened.

---

## 29. Where Should Corrections Happen?

Suppose the record contains:

~~~text
currency = XYZ
~~~

There are several possible places to correct it.

### Correct at the source

Best when the source system owns the incorrect value.

### Correct in an approved transformation

Useful when the mapping is deterministic and part of the pipeline rules.

### Correct manually

Sometimes required for exceptional cases.

The correction strategy should respect data ownership.

Do not permanently patch production records simply because it is the quickest solution.

---

## 30. Replay From Quarantine

Once the problem is corrected, the record can be replayed.

For example:

~~~text
quarantined
    |
    v
corrected
    |
    v
replay_pending
    |
    v
processing
    |
    v
completed
~~~

The replay should still use normal safeguards:

- validation
- idempotency
- transaction boundaries
- error handling
- retry logic
- observability

Quarantine does not bypass pipeline safety.

---

## 31. Quarantine Replay History

Suppose a record is quarantined twice.

The system should be able to show:

~~~text
Record 1001

First quarantine:
  reason = schema mismatch

Correction:
  source mapping fixed

First replay:
  result = completed

Later quarantine:
  reason = invalid amount

Second correction:
  amount corrected

Second replay:
  result = completed
~~~

This level of history may not be necessary for every system, but the operational requirement should be considered.

---

## 32. Quarantine Review

A review workflow can be as simple as:

~~~text
quarantined
     |
     v
review
     |
     +---- valid after investigation ---> replay
     |
     +---- source problem -------------> fix source
     |
     +---- invalid permanently --------> reject/discard
     |
     +---- unclear --------------------> investigate
~~~

For larger systems, review may require a dedicated operational interface.

For smaller systems, SQL or controlled administrative tooling may be sufficient.

The important requirement is controlled access.

---

## 33. Quarantine and Manual Review

Some records cannot be automatically classified.

For example:

~~~text
record
   |
   v
unusual transaction
   |
   v
manual review required
~~~

The quarantine system can provide a controlled waiting state.

But manual review should have:

- ownership
- status
- timestamps
- reason
- resolution
- audit information

Otherwise quarantine becomes a permanent storage area for forgotten records.

---

## 34. Quarantine Aging

Quarantined records should not remain unnoticed forever.

Useful metrics include:

~~~text
records quarantined today
records currently quarantined
oldest quarantined record
quarantine age
records by failure type
records awaiting review
records replayed successfully
records permanently rejected
~~~

An aging view can reveal operational problems.

For example:

~~~text
10 records quarantined today
2,000 records older than 30 days
~~~

The second number may indicate that the recovery process is not working.

---

## 35. Quarantine Retention

Quarantine data requires a retention policy.

Questions include:

- How long should failed records remain?
- Are they needed for audit?
- Can they be deleted after successful replay?
- Should metadata remain after payload deletion?
- Are there legal or compliance requirements?
- Does the payload contain personal information?

A possible lifecycle is:

~~~text
quarantined
    |
    v
resolved
    |
    v
retention period
    |
    v
archive/delete
~~~

The exact policy is system-specific.

Do not delete quarantine data automatically without understanding its operational and regulatory role.

---

## 36. Payload Retention vs Metadata Retention

It may be useful to retain metadata longer than the original payload.

For example:

~~~text
payload
   |
   v
deleted after retention period

metadata
   |
   v
retained for audit
~~~

This can reduce storage and sensitive-data exposure.

But whether this is appropriate depends on the system's requirements.

---

## 37. Quarantine Monitoring

Quarantine should be observable.

Useful metrics include:

~~~text
quarantine_records_total
quarantine_records_current
quarantine_by_failure_type
quarantine_by_source
quarantine_resolution_time
quarantine_replay_success_total
quarantine_replay_failure_total
~~~

A sudden increase can indicate a production problem.

For example:

~~~text
normal:
20 quarantined records/day

today:
5,000 quarantined records
~~~

That should trigger investigation.

---

## 38. Alerting on Quarantine

Alerts should be meaningful.

Possible alert conditions include:

- quarantine volume exceeds a known threshold
- a new failure type appears
- quarantine backlog grows continuously
- oldest quarantine age exceeds an operational limit
- replay failure rate increases
- a specific source begins producing invalid data

Avoid alerting on every individual record unless the business requires it.

One bad record may be normal.

A rapidly growing quarantine population is usually more useful as an operational signal.

---

## 39. Quarantine and Observability

A quarantine system should connect with normal pipeline observability.

A useful investigation path is:

~~~text
alert
  |
  v
quarantine metric
  |
  v
failure type
  |
  v
specific record
  |
  v
original payload/source
  |
  v
processing logs
  |
  v
code/version
  |
  v
root cause
~~~

This makes quarantine part of the operational system rather than an isolated table.

---

## 40. Testing Quarantine

Quarantine behavior must be tested.

At minimum, test:

### Test 1 — Permanent validation failure

Expected:

~~~text
record enters quarantine
~~~

### Test 2 — Temporary failure

Expected:

~~~text
record retries
~~~

It should not be quarantined immediately if retry is appropriate.

### Test 3 — Retry exhaustion

Expected:

~~~text
record eventually enters quarantine
~~~

### Test 4 — Quarantine idempotency

Expected:

~~~text
same failure does not create unintended duplicates
~~~

### Test 5 — Replay from quarantine

Expected:

~~~text
corrected record can return to normal processing
~~~

### Test 6 — Replay failure

Expected:

~~~text
new failure is recorded correctly
~~~

### Test 7 — Sensitive-data handling

Expected:

~~~text
protected fields are not unnecessarily exposed
~~~

### Test 8 — Retention behavior

Expected:

~~~text
expired data follows the defined retention policy
~~~

---

## 41. Testing the State Machine

If quarantine has states, test valid and invalid transitions.

For example:

Valid:

~~~text
quarantined -> under_review
under_review -> replay_pending
replay_pending -> completed
~~~

Invalid:

~~~text
completed -> quarantined
~~~

unless the system explicitly supports that transition.

State-transition tests prevent accidental lifecycle corruption.

---

## 42. Transaction Boundaries

Moving a record to quarantine should be designed carefully.

Consider:

~~~text
process record
     |
     X
failure
     |
     v
write quarantine
     |
     v
update processing status
~~~

What happens if the worker crashes between the two writes?

You could end up with:

~~~text
quarantine row exists
processing status still says processing
~~~

Or the opposite:

~~~text
processing status says quarantined
quarantine row does not exist
~~~

The correct transaction boundary depends on the storage model.

The important point is:

> The state change and quarantine write must not leave the system in an ambiguous state.

---

## 43. Quarantine With PostgreSQL

For a PostgreSQL-based pipeline, quarantine can often be represented with:

- a dedicated table
- status columns
- timestamps
- error metadata
- JSONB payload where appropriate
- unique constraints
- indexes for operational queries

For example, useful indexes might support queries for:

~~~text
status = quarantined
failure_type = ...
created_at = ...
source_id = ...
~~~

The exact indexes should be based on real access patterns.

Do not add indexes simply because every column could be queried.

---

## 44. Quarantine and Large Payloads

Large payloads can make quarantine tables expensive.

Suppose a failed record contains a large document.

Storing every payload directly in PostgreSQL may increase:

- database size
- backup size
- query cost
- vacuum work
- storage requirements

A different architecture may store:

~~~text
metadata -> database
payload  -> object storage
~~~

The correct design depends on the pipeline.

If payloads contain sensitive data, storage and access controls become even more important.

---

## 45. Quarantine and Security

Quarantine often contains exactly the records that caused problems.

That does not make them less sensitive.

Security considerations include:

- access control
- encryption
- secret redaction
- PII minimization
- retention
- audit logging
- secure replay
- secure administrative tooling

Never use quarantine as an excuse to bypass normal data-protection rules.

---

## 46. Production Recovery Workflow

A practical recovery workflow is:

~~~text
1. Detect quarantine growth
        |
        v
2. Identify failure type
        |
        v
3. Inspect sample records
        |
        v
4. Determine root cause
        |
        v
5. Decide correction
        |
        v
6. Apply controlled fix
        |
        v
7. Test on small sample
        |
        v
8. Replay corrected records
        |
        v
9. Verify results
        |
        v
10. Reconcile remaining quarantine
        |
        v
11. Document the incident
~~~

This is the same engineering cycle used throughout the book:

~~~text
Understand
    ->
Investigate
    ->
Design
    ->
Implement
    ->
Test
    ->
Verify
    ->
Observe
    ->
Recover
    ->
Improve
~~~

---

## 47. Troubleshooting

### Problem: records never enter quarantine

Check:

1. failure classification
2. retry limits
3. quarantine transition logic
4. transaction handling
5. quarantine write errors
6. processing status updates

### Problem: same record appears in quarantine many times

Check:

1. stable record identity
2. uniqueness constraints
3. idempotency logic
4. retry behavior
5. whether each failure is intentionally stored as a separate event

First determine whether multiple entries are actually duplicates.

### Problem: quarantine is growing continuously

Check:

1. new failure rate
2. failure types
3. source systems
4. retry exhaustion
5. review backlog
6. replay success rate
7. recent deployments
8. schema changes

A growing quarantine population usually needs root-cause investigation.

### Problem: replayed records return to quarantine

Do not keep replaying blindly.

Check:

1. whether the original problem was actually corrected
2. validation rules
3. transformation code
4. schema compatibility
5. external dependencies
6. configuration
7. source data

The repeated failure may be expected if the underlying problem remains.

### Problem: quarantine affects normal processing

Check:

1. worker concurrency
2. database connection usage
3. API request volume
4. queue capacity
5. storage throughput
6. scheduler configuration

Quarantine is production workload and should be controlled accordingly.

---

## 48. Production Considerations

Before deploying quarantine, consider:

### Failure classification

Know what belongs in quarantine.

### Storage

Choose appropriate storage for payload size and access patterns.

### Security

Protect quarantined information.

### Idempotency

Prevent accidental duplicate quarantine entries.

### State management

Define valid lifecycle transitions.

### Recovery

Provide a path from quarantine back to processing.

### Observability

Monitor volume, age, failure types, and resolution.

### Retention

Define how long data remains available.

### Capacity

Plan for abnormal spikes in quarantine volume.

### Auditability

Preserve enough information to explain what happened.

---

## 49. Definition of Done

Quarantine is complete when:

- [ ] Permanent failure categories are identified.
- [ ] Retryable failures are separated from quarantine failures.
- [ ] Quarantine states are defined.
- [ ] Valid state transitions are defined.
- [ ] Original data is preserved where required.
- [ ] Failure reason is stored.
- [ ] Failure type is structured where appropriate.
- [ ] Processing version is recorded where required.
- [ ] Schema version is recorded where required.
- [ ] Quarantine writes are safe and consistent.
- [ ] Duplicate quarantine behavior is defined.
- [ ] Sensitive data handling is reviewed.
- [ ] Quarantine access is controlled.
- [ ] Review workflow is defined.
- [ ] Correction workflow is defined.
- [ ] Replay from quarantine is supported where required.
- [ ] Replay failure is handled.
- [ ] Quarantine aging is observable.
- [ ] Quarantine volume is monitored.
- [ ] Retention policy is defined.
- [ ] Payload and metadata retention are considered separately where appropriate.
- [ ] State transitions are tested.
- [ ] Permanent failures are tested.
- [ ] Retry exhaustion is tested.
- [ ] Replay from quarantine is tested.
- [ ] Duplicate behavior is tested.
- [ ] Security behavior is tested.
- [ ] Recovery procedures are documented.

---

## 50. What You Learned

Quarantine is not a database table called failed_records.

It is a controlled failure path.

The basic model is:

~~~text
record
   |
   v
process
   |
   v
failure
   |
   v
classify
   |
   +---- temporary ----> retry
   |
   +---- permanent ----> quarantine
                              |
                              v
                           review
                              |
                    +---------+---------+
                    |                   |
                 correct              reject
                    |
                    v
                 replay
                    |
                    v
                completed
~~~

The most important lessons are:

1. Do not retry permanent failures forever.
2. Preserve enough information to investigate failed records.
3. Make quarantine state explicit.
4. Keep quarantine separate from normal processing when appropriate.
5. Protect sensitive data in quarantine.
6. Make quarantine operations idempotent.
7. Provide a clear recovery path.
8. Treat replay from quarantine as a normal controlled processing operation.
9. Monitor quarantine volume and age.
10. Define retention before production data starts accumulating.
11. Test failure, quarantine, correction, and replay paths.
12. Preserve enough history to explain what happened.

The final goal is not to hide failed records.

The goal is to isolate them safely, understand why they failed, and provide a reliable path to recovery.

---

# Chapter 27 Preview

The next recipe will cover **Backfill Historical Data**.

We will move from individual recovery into intentional historical processing.

We will cover:

- what a backfill is
- when backfills are needed
- defining a historical range
- selecting the correct source data
- backfill planning
- small-batch execution
- backfill checkpoints
- protecting normal workloads
- idempotency during backfill
- handling late and missing data
- validating historical results
- reconciliation
- backfill failures
- stopping and resuming a backfill
- monitoring progress
- production safety

The key question will be:

> How do we process a large historical dataset safely without damaging the normal production pipeline?## Production Tools You Should Know

These tools solve or provide production implementations of concepts covered in this recipe. Learn what the tool provides, but understand the underlying problem first.

| Tool | What to know |
|---|---|
| **Great Expectations** | Validation failures and data-quality workflows. |
| **Soda** | Data quality checks and failed-data detection. |
| **Apache Kafka** | Dead-letter topic pattern for failed events. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the engineering mechanism.

---




## Implementation Lab — Quarantine and Recovery

### 1. Quarantine model

~~~python
from dataclasses import dataclass

@dataclass
class QuarantinedRecord:
    event_id: str
    reason: str
    payload: dict
    status: str = "QUARANTINED"

def quarantine(event_id, payload, reason):
    return QuarantinedRecord(event_id, reason, payload)
~~~

### 2. Test preservation

~~~python
def test_quarantine_preserves_original_payload():
    record = quarantine("evt-1", {"amount": -10}, "INVALID_AMOUNT")
    assert record.event_id == "evt-1"
    assert record.payload["amount"] == -10
    assert record.status == "QUARANTINED"
~~~

### 3. Intentional failure drill

Replace the original payload with only the error message. Observe that repair evidence is lost. Restore original-payload preservation.

### 4. Recovery

Repair, validate again, then move the record to replay. Moving out of quarantine is not proof of successful processing.

---

