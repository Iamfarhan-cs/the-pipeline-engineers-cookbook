# E39 — Database CDC

## 1. Problem Recognition

### The production problem

A database changes continuously. A downstream pipeline needs to capture inserts, updates, and deletes without repeatedly scanning the entire source table.

A naive approach is:

```text
Every run
   ↓
SELECT * FROM source_table
   ↓
Compare everything
   ↓
Find changes
```

This becomes expensive as the source grows. It also creates correctness problems around updates, deletes, transaction boundaries, concurrent writes, and missed changes.

**Change Data Capture (CDC)** solves this by extracting changes from the database's change stream rather than repeatedly discovering them by scanning the current table state.

### How to recognize the problem

CDC is appropriate when you see:

- large source tables with frequent changes
- a need to capture inserts, updates, and deletes
- incremental extraction based on database transaction history
- requirements for low-latency downstream updates
- repeated full-table scans becoming expensive
- source systems where `updated_at` is insufficient
- hard deletes that must be propagated
- a requirement to preserve transaction ordering or commit position

---

# 2. Concept and Reasoning

## 2.1 What CDC actually captures

CDC captures **changes**, not simply the latest state of a table.

A conceptual stream might look like:

```text
INSERT customer 101
UPDATE customer 101
UPDATE customer 101
DELETE customer 101
```

The downstream system can then reconstruct state or maintain its own change history.

A CDC record commonly contains some combination of:

- operation type
- table/schema identity
- primary key
- before-image
- after-image
- transaction identifier
- commit position
- source timestamp
- connector or capture metadata

The exact fields depend on the database and CDC implementation.

---

# 3. CDC vs Watermarks vs Timestamps

These mechanisms solve related but different problems.

| Mechanism | Main idea | Typical limitation |
|---|---|---|
| Full extraction | Read current source state | Expensive at scale |
| Timestamp extraction | Read rows changed after a time | Deletes and clock semantics |
| ID extraction | Read rows after an ID | IDs usually do not represent updates |
| Watermark | Track safe source progress | Needs a suitable ordered position |
| CDC | Capture committed changes | Requires source/log support and operational setup |

A timestamp query might say:

```sql
SELECT *
FROM customers
WHERE updated_at > :watermark;
```

CDC instead observes the database's change mechanism.

This distinction matters because an update to an existing row does not create a new primary-key ID, while CDC records the update itself.

---

# 4. CDC Mental Model

The general architecture is:

```text
SOURCE DATABASE
      ↓
DATABASE CHANGE LOG
      ↓
CDC CAPTURE
      ↓
CHANGE EVENTS
      ↓
DURABLE CDC BUFFER / BROKER
      ↓
CDC CONSUMER
      ↓
VALIDATE
      ↓
PERSIST
      ↓
COMMIT SOURCE POSITION
      ↓
DOWNSTREAM STATE
```

The exact technology can change. The underlying mechanism remains the same.

---

# 5. The Critical Correctness Boundary

The most important CDC rule is:

> **Do not acknowledge or advance the CDC position until the corresponding change has been durably processed.**

Conceptually:

```text
READ CHANGE
    ↓
VALIDATE
    ↓
PERSIST
    ↓
VERIFY / COMMIT
    ↓
ADVANCE CDC POSITION
```

If a process crashes before the position is committed, the change may be delivered again.

That is expected in many CDC systems.

Therefore:

```text
CDC
+
Idempotent consumer
+
Durable position
=
Recoverable change pipeline
```

---

# 6. Source Database Change Logs

Many databases expose a durable record of changes through mechanisms such as transaction logs or logical replication.

The important concept is not the product name. It is this:

```text
Database transaction
       ↓
Committed change record
       ↓
Durable source position
```

The CDC system reads from that ordered source of changes.

Examples include:

- PostgreSQL logical replication / WAL-derived changes
- MySQL binlog
- SQL Server transaction log / CDC mechanisms
- Oracle redo/log-based capture

The exact configuration must follow the database's supported CDC architecture.

---

# 7. Transaction Boundaries Matter

Suppose a transaction performs:

```text
BEGIN

UPDATE accounts SET balance = ... WHERE id = 10;
UPDATE ledger   SET amount = ... WHERE account_id = 10;

COMMIT
```

CDC should not expose an uncommitted transaction as if it were final source state.

The downstream pipeline must understand whether the CDC implementation exposes:

- transaction boundaries
- commit positions
- event ordering
- transaction identifiers

This matters when several changes belong to one business transaction.

---

# 8. Ordering

CDC streams commonly provide ordering guarantees at a specific scope, not necessarily globally.

For example:

```text
Partition 0: A → B → C
Partition 1: X → Y → Z
```

You may have ordering within each partition but no valid assumption that:

```text
A < X < B < Y
```

is meaningful.

Always identify the scope of the ordering guarantee:

- transaction
- table
- partition
- source database
- connector stream

Never invent global ordering where the source does not provide it.

---

# 9. Initial Snapshot vs Ongoing CDC

Many CDC systems need to establish an initial state before consuming ongoing changes.

There are two conceptual phases:

```text
INITIAL SNAPSHOT
       ↓
ESTABLISH BASELINE
       ↓
CONTINUE FROM CDC POSITION
       ↓
ONGOING CHANGES
```

The dangerous gap is:

```text
snapshot ends
      ↓
CDC begins later
      ↓
changes in between are missed
```

A correct CDC implementation must establish a consistent relationship between the snapshot boundary and the change-stream position.

Do not treat snapshot + CDC as two unrelated jobs.

---

# 10. Change Event Shape

A practical internal representation can be:

```python
from dataclasses import dataclass
from typing import Any


@dataclass(frozen=True)
class ChangeEvent:
    source: str
    schema: str
    table: str
    operation: str
    primary_key: dict[str, Any]
    before: dict[str, Any] | None
    after: dict[str, Any] | None
    position: str
    transaction_id: str | None = None
```

The event should preserve enough metadata to answer:

- Where did this change come from?
- What row did it affect?
- What operation happened?
- What was the source position?
- Which transaction produced it?

---

# 11. Operation Types

The consumer must distinguish at least:

```text
INSERT
UPDATE
DELETE
```

A common interpretation is:

| Operation | Before | After |
|---|---|---|
| INSERT | null | new row |
| UPDATE | old row | new row |
| DELETE | old row | null |

Some CDC systems expose different event models. Normalize them deliberately rather than assuming every connector uses the same shape.

---

# 12. Implement the Consumer from Scratch

The following example demonstrates the **consumer mechanism** independently of a specific CDC product.

## 12.1 Destination tables

```sql
CREATE TABLE IF NOT EXISTS customer_state (
    customer_id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    status TEXT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE IF NOT EXISTS cdc_processed_event (
    source TEXT NOT NULL,
    position TEXT NOT NULL,
    PRIMARY KEY (source, position)
);
```

The second table provides an idempotency boundary for replayed CDC events.

---

# 13. Apply CDC Events

A simplified consumer can look like:

```python
def apply_change(conn, event):
    with conn.transaction():
        with conn.cursor() as cur:
            cur.execute(
                """
                INSERT INTO cdc_processed_event (source, position)
                VALUES (%s, %s)
                ON CONFLICT DO NOTHING
                """,
                (event.source, event.position),
            )

            if cur.rowcount == 0:
                return "duplicate"

            if event.operation == "INSERT":
                cur.execute(
                    """
                    INSERT INTO customer_state
                        (customer_id, name, status, updated_at)
                    VALUES (%s, %s, %s, %s)
                    ON CONFLICT (customer_id)
                    DO UPDATE SET
                        name = EXCLUDED.name,
                        status = EXCLUDED.status,
                        updated_at = EXCLUDED.updated_at
                    """,
                    (
                        event.after["customer_id"],
                        event.after["name"],
                        event.after["status"],
                        event.after["updated_at"],
                    ),
                )

            elif event.operation == "UPDATE":
                cur.execute(
                    """
                    UPDATE customer_state
                    SET name = %s,
                        status = %s,
                        updated_at = %s
                    WHERE customer_id = %s
                    """,
                    (
                        event.after["name"],
                        event.after["status"],
                        event.after["updated_at"],
                        event.after["customer_id"],
                    ),
                )

            elif event.operation == "DELETE":
                cur.execute(
                    """
                    DELETE FROM customer_state
                    WHERE customer_id = %s
                    """,
                    (event.primary_key["customer_id"],),
                )

            else:
                raise ValueError(f"Unsupported CDC operation: {event.operation}")

    return "applied"
```

The event identity and destination change are handled in the same transaction. If the transaction rolls back, the event can safely be delivered again.

---

# 14. Event Identity

Do not assume that a database primary key uniquely identifies a CDC event.

The same row can generate many events:

```text
customer 101 INSERT
customer 101 UPDATE
customer 101 UPDATE
customer 101 DELETE
```

You need an event identity based on the source position or another provider-defined event identifier.

A useful conceptual identity is:

```text
(source, partition, position)
```

The exact fields depend on the CDC source.

---

# 15. Idempotency and Replay

A consumer may crash after applying a change but before acknowledging its source position.

Example:

```text
CDC event at position 500
       ↓
Destination committed
       ↓
Consumer crashes
       ↓
Position 500 not acknowledged
       ↓
Event 500 delivered again
```

The second delivery must be safe.

This is why CDC consumers commonly need:

- unique event identity
- destination uniqueness
- idempotent upserts
- transactional event bookkeeping

---

# 16. Delete Handling

Deletes are one of the major reasons CDC is valuable.

A timestamp-based query such as:

```sql
SELECT *
FROM customers
WHERE updated_at > :watermark;
```

cannot see a row that has been physically deleted unless the source keeps a deletion marker.

CDC can emit:

```text
DELETE customer_id=101
```

The downstream system can then remove or tombstone the corresponding state.

Choose explicitly between:

```text
DELETE physically
```

or:

```text
WRITE tombstone
```

based on downstream requirements.

---

# 17. Updates and Before/After Images

For an update:

```text
before = {
    status: "pending"
}

after = {
    status: "completed"
}
```

The consumer can apply the after-image directly.

Before-images are valuable for:

- auditing
- change history
- reverse operations
- debugging
- downstream diff calculations

Do not assume before-images are always available. Their availability depends on source configuration and CDC technology.

---

# 18. Schema Evolution

CDC is sensitive to source schema changes.

Examples:

```text
ADD column
RENAME column
CHANGE type
DROP column
```

A change event may evolve even though the CDC transport itself is still working.

The consumer should distinguish:

```text
transport failure
```

from:

```text
schema compatibility failure
```

Useful controls include:

- explicit event schemas
- versioned contracts
- tolerant readers where appropriate
- schema validation
- compatibility tests
- quarantine for invalid events

Do not silently discard fields because a new source version appeared.

---

# 19. CDC Backpressure

The source can generate changes faster than the consumer can process them.

Conceptually:

```text
Source change rate
       ↓
CDC stream
       ↓
Consumer capacity
```

If:

```text
production rate > consumption rate
```

backlog grows.

Monitor:

- source-to-consumer lag
- queue depth
- events per second
- processing latency
- batch size
- consumer errors

Do not solve persistent backlog by blindly adding workers. First determine whether the destination, source ordering, partitions, or downstream transaction boundaries are the bottleneck.

---

# 20. Poison Events

A single malformed event can repeatedly fail.

Example:

```text
Event 500
   ↓
validation failure
   ↓
retry
   ↓
validation failure
   ↓
retry forever
```

Classify failures.

Transient examples:

- temporary database outage
- network timeout
- temporary throttling

Permanent examples:

- invalid schema
- unsupported operation
- impossible field type
- corrupted event

Permanent failures should go through a controlled quarantine/DLQ path rather than blocking the entire stream indefinitely.

---

# 21. CDC Position Management

CDC consumers need durable source progress.

A conceptual state table might be:

```sql
CREATE TABLE IF NOT EXISTS cdc_position (
    source TEXT NOT NULL,
    partition_key TEXT NOT NULL,
    position TEXT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (source, partition_key)
);
```

The position might represent:

- an LSN
- binlog coordinate
- connector offset
- partition offset
- another source-specific position

Do not treat all positions as interchangeable numbers. The consumer must understand the source's ordering and comparison semantics.

---

# 22. Position Advancement

The safe sequence is:

```text
READ CHANGE
    ↓
VALIDATE
    ↓
APPLY CHANGE
    ↓
COMMIT DESTINATION
    ↓
COMMIT SOURCE POSITION
```

When the source position and destination are in different systems, a distributed transaction usually does not exist.

Therefore design for replay.

The goal is generally:

```text
at-least-once delivery
+
idempotent processing
=
exactly-once-like business result
```

Do not claim mathematical exactly-once semantics merely because duplicates are hidden by a unique key.

---

# 23. Initial Snapshot + CDC Correctness

A common production failure is an incorrect handoff between snapshot and streaming.

Unsafe sequence:

```text
snapshot starts
   ↓
snapshot completes
   ↓
CDC starts
```

Changes occurring during the snapshot can be missed.

A safe design establishes a source position associated with the snapshot boundary:

```text
CAPTURE CDC POSITION
       ↓
SNAPSHOT CONSISTENT STATE
       ↓
APPLY CHANGES FROM CAPTURED POSITION
       ↓
CONTINUE STREAMING
```

The exact mechanism is database-specific, but the correctness requirement is universal: there must be no gap between the baseline and the change stream.

---

# 24. CDC and Transactions

Suppose a source transaction modifies three tables:

```text
transaction 9001
  ├─ accounts
  ├─ ledger
  └─ audit
```

A downstream system may need to preserve the transaction relationship.

Useful metadata includes:

- transaction ID
- commit position
- event sequence within transaction
- table identity

If the downstream business logic requires atomic application across those tables, process the related changes according to the CDC system's transaction semantics instead of assuming event-by-event independence.

---

# 25. CDC and Multi-Table Dependencies

Suppose:

```text
customer
   ↓
account
   ↓
transaction
```

A transaction CDC event may arrive before the downstream dimension is available.

Possible strategies include:

- transaction-aware processing
- staging changes before publishing
- deferred foreign-key resolution
- retries
- reference-data reconciliation

Do not create artificial ordering assumptions from table names or application intuition.

---

# 26. Testing

## Essential tests

| Test | Expected result |
|---|---|
| INSERT event | Row created |
| UPDATE event | Current state updated |
| DELETE event | Row deleted or tombstoned |
| Duplicate event | No duplicate business effect |
| Crash before commit | Event can be replayed |
| Crash after data commit | Replay remains safe |
| Invalid schema | Controlled rejection/quarantine |
| Permanent failure | Does not retry forever |
| Transient failure | Retry according to policy |
| Position update conflict | Stale worker rejected |
| Source grows during snapshot | No change gap |
| Multiple partitions | Independent positions preserved |
| Out-of-order partitions | Only valid source guarantees assumed |
| Transaction metadata | Transaction relationship preserved |

---

# 27. Intentional Failure Drills

## Drill 1 — Crash after destination commit

Force a crash after applying an event but before acknowledging its source position.

Expected:

```text
Event delivered again
Duplicate safely detected
Final business state remains correct
```

## Drill 2 — Kill the consumer during a transaction

Expected:

```text
Incomplete destination transaction rolls back
Source position does not advance
Restart resumes safely
```

## Drill 3 — Inject a malformed event

Expected:

```text
Validation fails
Event is classified
Permanent failure is quarantined
Healthy events continue
```

## Drill 4 — Stop the destination

Expected:

```text
CDC backlog increases
Source position stops advancing
Consumer retries safely
Recovery catches up without skipping events
```

## Drill 5 — Break snapshot/stream handoff

Verify that the test detects a missing change between snapshot completion and CDC start.

This drill proves that the handoff boundary is actually understood.

---

# 28. Observability

## Logs

Record:

- source
- schema/table
- operation
- event position
- transaction ID where available
- consumer/run ID
- processing duration
- outcome
- retry count

Avoid logging sensitive row payloads unless explicitly permitted and protected.

## Metrics

Useful metrics include:

```text
cdc_events_received_total
cdc_events_processed_total
cdc_events_failed_total
cdc_duplicate_events_total
cdc_consumer_lag
cdc_processing_latency
cdc_position_advance_total
cdc_quarantined_events_total
cdc_backlog_size
```

## Alerts

Consider alerts for:

- rapidly increasing CDC lag
- position not advancing
- repeated permanent failures
- repeated transient failures
- unexpected schema changes
- connector/source disconnection
- growing quarantine volume
- snapshot/stream handoff failure

---

# 29. Reconciliation

CDC should not remove the need for reconciliation.

Periodically compare the downstream state with source state or source-derived control totals.

Examples:

```text
source customer count
vs
consumer customer count
```

and:

```text
source latest verified position
vs
consumer safe position
```

For critical pipelines, reconcile by partition, table, business date, or another appropriate boundary.

A healthy CDC stream can still have a configuration or mapping error. Reconciliation catches errors that transport metrics cannot.

---

# 30. Performance and Scaling

CDC scaling depends on source and transport semantics.

Important factors include:

- number of source partitions
- transaction size
- event size
- destination write throughput
- indexes
- batch size
- network bandwidth
- serialization/deserialization cost
- ordering requirements

A consumer may scale horizontally only where the source allows safe parallelism.

For example:

```text
Partition 0 → Worker A
Partition 1 → Worker B
Partition 2 → Worker C
```

is fundamentally different from allowing arbitrary workers to reorder one globally ordered transaction stream.

Measure before increasing concurrency.

---

# 31. CDC Retention and Recovery

A CDC source or broker normally retains changes for a bounded period.

If a consumer falls behind beyond retention:

```text
source position needed
        ↓
no longer available
```

The consumer may need to:

1. stop
2. determine the missing range
3. rebuild a snapshot
4. establish a new CDC position
5. replay available changes
6. reconcile

Do not assume a consumer can recover forever from any historical position.

Retention is part of the recovery design.

---

# 32. Common Mistakes

### 1. Treating CDC as just another polling query

CDC depends on source change semantics and durable positions.

### 2. Acknowledging before persistence

A crash can lose a change.

### 3. No idempotency

Redelivery creates duplicate business effects.

### 4. Ignoring deletes

The downstream state silently becomes stale.

### 5. Assuming global ordering

Most CDC systems have scoped ordering guarantees.

### 6. Ignoring transaction boundaries

Related source changes can be applied incorrectly.

### 7. No snapshot handoff design

Changes can be lost between snapshot and streaming.

### 8. Treating source position as an ordinary integer

LSNs, offsets, and connector positions have source-specific semantics.

### 9. No retention plan

A sufficiently delayed consumer may lose its recovery path.

### 10. Scaling consumers without understanding ordering

Parallelism can change correctness if the source requires ordered processing.

---

# 33. Production Tools You Should Know

### 1. Debezium

A widely used CDC platform for capturing database changes and publishing them to downstream systems. Learn its concepts: snapshots, offsets, connectors, schemas, and source positions.

### 2. PostgreSQL Logical Replication

Useful for understanding how PostgreSQL exposes committed database changes through WAL/logical replication mechanisms.

### 3. Kafka

Commonly used as a durable transport for CDC events, providing partitions, retention, consumer groups, offsets, and replay.

These tools are production vocabulary. The underlying CDC mechanism should remain understandable without them.

---

# 34. Production Runbook

## Symptom: CDC lag is increasing

Check:

1. source change rate
2. consumer throughput
3. destination latency
4. consumer errors
5. partition skew
6. transaction size
7. broker/transport backlog
8. retry volume

Do not immediately add consumers without identifying the bottleneck.

## Symptom: CDC position stopped advancing

Check:

1. consumer health
2. current failing event
3. destination connectivity
4. transaction locks
5. schema validation failures
6. poison events
7. source/connector connectivity

If a permanent event failure is blocking the stream, quarantine it according to the pipeline's policy rather than skipping it silently.

## Symptom: Consumer restarted and reprocessed events

This can be normal.

Check:

1. destination idempotency
2. duplicate-event metrics
3. final business state
4. source position
5. reconciliation results

Do not treat every redelivery as data corruption.

## Symptom: CDC retention was exceeded

1. Stop the consumer.
2. Determine the missing position range.
3. Establish a new consistent snapshot.
4. Establish a valid CDC starting position.
5. Replay available changes.
6. Reconcile the destination.
7. Resume continuous CDC.

---

# 35. Definition of Done

You understand this recipe when you can independently:

- explain what database CDC captures
- distinguish CDC from timestamp and ID extraction
- explain transaction and ordering boundaries
- design an initial snapshot + CDC handoff
- model INSERT, UPDATE, and DELETE events
- design event identity
- build an idempotent CDC consumer
- persist source positions safely
- handle redelivery
- classify permanent and transient failures
- handle poison events
- reason about partitioned CDC
- monitor CDC lag
- recover from consumer crashes
- reason about retention limits
- reconcile CDC state with the source
- explain why exactly-once-like results normally depend on idempotency and durable processing

---

# 36. What You Learned

> **Database CDC is the controlled capture of committed database changes from a source change mechanism. The hard part is not reading change events; it is preserving correctness across transactions, ordering, positions, snapshots, redelivery, deletes, failures, and recovery.**

The practical mental model is:

```text
SOURCE DATABASE
      ↓
COMMITTED CHANGE LOG
      ↓
CDC CAPTURE
      ↓
CHANGE EVENT
      ↓
VALIDATE
      ↓
IDEMPOTENT PERSISTENCE
      ↓
COMMIT
      ↓
ADVANCE SOURCE POSITION
      ↓
RECONCILE
```

The key question is always:

> **What source change position can this consumer prove it has safely processed?**
