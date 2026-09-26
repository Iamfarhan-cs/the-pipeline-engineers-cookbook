# E40 — Transaction Log CDC

## 1. Problem Recognition

A database changes continuously. Polling tables with timestamps or IDs can miss updates and hard deletes, and repeated scans become expensive. Transaction-log CDC reads the database's durable change history instead.

Recognize the problem when you need:

- INSERT, UPDATE, and DELETE capture
- low-latency change propagation
- a restartable database-native source position
- changes that have no reliable `updated_at`
- hard-delete detection
- source transaction and commit semantics

Core distinction:

    TABLE POLLING → observes current state
    LOG CDC      → observes committed change history

---

## 2. Concept and Reasoning

A transaction-log CDC pipeline is:

    DATABASE TRANSACTION
           ↓
         COMMIT
           ↓
     TRANSACTION LOG
           ↓
       CDC READER
           ↓
      CHANGE EVENT
           ↓
       VALIDATE
           ↓
       PERSIST
           ↓
    SAFE POSITION

The central rule is:

> Never advance the durable source position merely because a change was observed. Advance it only after the corresponding work is durably complete.

This produces a recoverable at-least-once system when combined with idempotent processing.

---

## 3. Transaction Log CDC vs Other Extraction

| Approach | Captures | Important limitation |
|---|---|---|
| Full extraction | Current state | Expensive at scale |
| Timestamp extraction | Changed rows | Deletes and clock semantics |
| ID extraction | Usually new rows | Ordinary updates are missed |
| Transaction-log CDC | Committed changes | More operational complexity |

For example, changing `customer_id=42` from `pending` to `completed` does not change its primary key. ID-based extraction may never see that update. Log-based CDC can capture the update itself.

---

## 4. Source Log Concepts

Different databases expose different log mechanisms:

| Database | Common CDC source concept |
|---|---|
| PostgreSQL | WAL + logical decoding/logical replication |
| MySQL | Binary log |
| SQL Server | Transaction-log-based CDC/change mechanisms |
| Oracle | Redo-log-based capture mechanisms |

The implementation is database-specific, but the engineering contract is similar:

    COMMITTED CHANGE
          ↓
    DURABLE SOURCE POSITION
          ↓
    CDC CONSUMER

Do not treat an LSN, binlog coordinate, GTID, or connector offset as an ordinary timestamp or generic integer. The source defines its comparison and recovery semantics.

---

## 5. Transaction Boundaries

Suppose one source transaction does:

    BEGIN
      UPDATE accounts ...
      UPDATE ledger ...
      INSERT audit ...
    COMMIT

CDC consumers must understand whether their source exposes transaction IDs, begin/commit markers, event sequence, and commit position.

If downstream processing requires atomic visibility, do not assume that processing each event independently preserves source transaction semantics.

---

## 6. Initial Snapshot + CDC

A common failure is:

    SNAPSHOT TABLE
         ↓
    SNAPSHOT FINISHES
         ↓
    START CDC

Changes committed during the handoff can be missed.

A correct design establishes a relationship between the snapshot boundary and the source log position:

    CAPTURE CONSISTENT CDC POSITION
             ↓
       ESTABLISH SNAPSHOT
             ↓
        LOAD BASELINE
             ↓
    CONSUME CHANGES FROM POSITION
             ↓
          RECONCILE
             ↓
        CONTINUE CDC

The exact procedure depends on the database and CDC implementation. The invariant is universal: there must be no unowned interval between the baseline and the change stream.

---

## 7. Durable Source Position

Use durable state for the consumer position.

    CREATE TABLE cdc_source_position (
        source_name TEXT NOT NULL,
        partition_key TEXT NOT NULL,
        position TEXT NOT NULL,
        version BIGINT NOT NULL DEFAULT 0,
        updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
        PRIMARY KEY (source_name, partition_key)
    );

The position should preserve the source-native representation where practical. If stored as text, validate its format at the application boundary.

---

## 8. Build the Consumer from Scratch

### 8.1 Normalize a change event

    from dataclasses import dataclass
    from typing import Any

    @dataclass(frozen=True)
    class LogChange:
        source: str
        partition: str
        position: str
        transaction_id: str | None
        operation: str
        table: str
        primary_key: dict[str, Any]
        before: dict[str, Any] | None
        after: dict[str, Any] | None

Keep source-position metadata together with the business change. This lets the consumer answer where the change came from and where safe progress can resume.

### 8.2 Create an idempotency boundary

    CREATE TABLE processed_cdc_change (
        source_name TEXT NOT NULL,
        partition_key TEXT NOT NULL,
        position TEXT NOT NULL,
        processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
        PRIMARY KEY (source_name, partition_key, position)
    );

The identity must represent a source change, not merely a row. One row can produce many changes.

### 8.3 Apply the change transactionally

    BEGIN
      INSERT processed_cdc_change
        ON CONFLICT DO NOTHING

      if newly inserted:
          apply INSERT / UPDATE / DELETE

      COMMIT

If the transaction rolls back, the source event can safely be delivered again.

A simplified PostgreSQL consumer is:

    def apply_change(conn, event):
        with conn.transaction():
            with conn.cursor() as cur:
                cur.execute(
                    """
                    INSERT INTO processed_cdc_change
                        (source_name, partition_key, position)
                    VALUES (%s, %s, %s)
                    ON CONFLICT DO NOTHING
                    """,
                    (event.source, event.partition, event.position),
                )

                if cur.rowcount == 0:
                    return "duplicate"

                if event.operation in {"INSERT", "UPDATE"}:
                    row = event.after
                    if row is None:
                        raise ValueError("Missing after image")

                    cur.execute(
                        """
                        INSERT INTO customer_state (customer_id, name, status)
                        VALUES (%s, %s, %s)
                        ON CONFLICT (customer_id)
                        DO UPDATE SET
                            name = EXCLUDED.name,
                            status = EXCLUDED.status
                        """,
                        (row["customer_id"], row["name"], row["status"]),
                    )

                elif event.operation == "DELETE":
                    cur.execute(
                        "DELETE FROM customer_state WHERE customer_id = %s",
                        (event.primary_key["customer_id"],),
                    )
                else:
                    raise ValueError("Unsupported CDC operation")

        return "applied"

---

## 9. Safe Position Advancement

Suppose the last safe position is P100 and the consumer reads P101, P102, and P103.

The safe sequence is:

    P103 observed
         ↓
    P103 applied
         ↓
    destination transaction committed
         ↓
    P103 becomes safe

Never do:

    P103 observed → immediately mark P103 complete

When the source position and destination are different systems, a distributed transaction may not exist. Design for replay instead of assuming atomicity across systems.

---

## 10. Batch Processing

Batching improves throughput but changes failure granularity.

### Transactional batch

    READ P100–P199
          ↓
       APPLY
          ↓
       COMMIT

If one change fails, the entire batch can roll back.

### Event-level transactions

    P100 → commit
    P101 → commit
    P102 → commit

This gives finer recovery but more transaction overhead.

Choose based on source ordering, transaction semantics, destination constraints, and throughput requirements. Batch size is a correctness decision as well as a performance decision.

---

## 11. Inserts, Updates, and Deletes

A normalized event commonly means:

| Operation | Before | After |
|---|---|---|
| INSERT | null | new row |
| UPDATE | old row | new row |
| DELETE | old row | null |

Deletes are a major advantage of CDC. A current-state query cannot see a physically deleted row unless the application maintains a deletion marker.

Choose deliberately whether downstream DELETE means:

- physical deletion
- tombstone
- historical deletion record

---

## 12. Before and After Images

For an update:

    before.status = pending
    after.status  = completed

Before-images are useful for auditing, diffs, debugging, reverse operations, and history.

Do not assume before-images exist. Their availability depends on source configuration and CDC implementation.

---

## 13. Ordering

Ordering guarantees are scoped.

A source may guarantee:

    transaction order
    partition order

without guaranteeing a single global order across independent partitions.

Never invent global ordering. Store partition-specific positions when the source is partitioned.

For example:

    partition 0 → P1000
    partition 1 → P900
    partition 2 → P1200

These are three safe positions, not automatically one global position of P1200.

---

## 14. Schema Evolution

CDC transport and event schema are separate contracts.

Potential changes include:

- added columns
- removed columns
- renamed columns
- type changes
- new tables
- changed event envelopes

Distinguish:

    log transport failure
          ≠
    event schema incompatibility
          ≠
    destination schema failure

Use explicit contracts, validation, compatibility tests, versioning, and controlled quarantine where required.

---

## 15. Backpressure and Source Impact

If change production exceeds consumption:

    source change rate > consumer capacity
                    ↓
                backlog grows

Monitor:

- CDC lag
- event rate
- processing latency
- destination latency
- transaction size
- retry volume
- partition skew

CDC also has source-side operational costs. A stalled log consumer can create log-retention pressure or replication-slot pressure, depending on the database implementation.

Do not solve backlog by blindly adding consumers. First identify whether the bottleneck is source decoding, transport, ordering, destination writes, locks, or transaction size.

---

## 16. Log Retention and Recovery

Transaction logs have finite retention.

If the consumer needs a position that the source no longer retains:

    required position
          ↓
    no longer available
          ↓
    direct resume impossible

Recovery may require:

1. stop the consumer
2. determine the missing range
3. establish a new consistent snapshot
4. establish a new CDC position
5. rebuild the baseline
6. resume streaming
7. reconcile

Retention is part of the CDC recovery design, not merely an infrastructure setting.

---

## 17. CDC Position Ownership

Avoid several workers blindly writing one position:

    worker A ─┐
    worker B ─┼──→ same position
    worker C ─┘

Use explicit ownership, partition assignment, or compare-and-set updates.

A stale worker must not overwrite newer progress.

A normal position update follows:

    current safe position
          ↓
    process changes
          ↓
    verify persistence
          ↓
    advance position

A backward move should be treated as an explicit recovery operation.

---

## 18. Testing

| Test | Expected result |
|---|---|
| INSERT event | Destination row created |
| UPDATE event | Destination row updated |
| DELETE event | Destination state removed/tombstoned |
| Duplicate event | No duplicate business effect |
| Crash before commit | Event safely replayed |
| Crash after destination commit | Replay remains idempotent |
| Invalid event | Controlled rejection |
| Position regression | Rejected |
| Multiple partitions | Independent positions preserved |
| Large transaction | Controlled memory/batch behavior |
| Retention exceeded | Recovery path invoked |
| Schema change | Contract handling invoked |
| Snapshot handoff | No changes missed |
| Transaction with many changes | Transaction semantics preserved |

---

## 19. Intentional Failure Drills

### Drill 1 — Crash after destination commit

Force a crash after applying an event but before advancing its source position.

Expected:

    event delivered again
          ↓
    duplicate identity detected
          ↓
    no second business effect

### Drill 2 — Consumer crash during a transaction

Verify that the chosen transaction model prevents partial destination state where atomicity is required.

### Drill 3 — Corrupt the source position

Expected:

    unsafe resume rejected
          ↓
    operator investigates history
          ↓
    controlled recovery

### Drill 4 — Pause the consumer

Observe CDC lag and source retention pressure, then recover without skipping changes.

### Drill 5 — Schema change

Introduce a compatible and an incompatible change. Verify that each follows the defined contract policy.

---

## 20. Observability

### Logs

Record:

- source
- schema/table
- operation
- source position
- transaction ID where available
- partition
- consumer/run ID
- processing duration
- outcome
- retry count

Avoid logging sensitive row contents by default.

### Metrics

Useful metrics:

    cdc_events_received_total
    cdc_events_processed_total
    cdc_events_failed_total
    cdc_duplicate_events_total
    cdc_consumer_lag
    cdc_processing_latency
    cdc_safe_position
    cdc_position_update_conflicts
    cdc_source_retention_pressure
    cdc_quarantined_events_total

### Alerts

Alert on:

- rapidly increasing lag
- position not advancing
- source retention approaching a recovery threshold
- repeated failures
- connector/source disconnection
- schema incompatibility
- unexpected position regression

---

## 21. Reconciliation

CDC health metrics do not prove that downstream data is correct.

Periodically compare:

    SOURCE CURRENT STATE
           vs
    DOWNSTREAM CURRENT STATE

and:

    SOURCE SAFE POSITION
           vs
    CONSUMER SAFE POSITION

For critical data, reconcile by table, partition, key range, business date, counts, or control totals.

A consumer can be processing every event successfully while still applying an incorrect field mapping. Reconciliation catches this class of error.

---

## 22. Performance and Scaling

Important variables include:

- transaction volume
- transaction size
- event size
- log decoding cost
- network throughput
- destination throughput
- serialization cost
- batch size
- ordering constraints

Large source transactions can produce sudden bursts. Monitor worst-case transaction size, not only average event rate.

Scale horizontally only where source ordering and transaction semantics permit it.

---

## 23. Common Mistakes

1. Treating the transaction log as an ordinary table.
2. Replacing the native source position with a wall-clock timestamp.
3. Advancing position before destination commit.
4. Ignoring transaction boundaries.
5. Ignoring log retention.
6. Assuming global ordering.
7. Omitting idempotent event identity.
8. Treating snapshot and streaming as unrelated jobs.
9. Increasing concurrency without understanding ordering.
10. Logging sensitive change payloads unnecessarily.

---

## 24. Production Tools You Should Know

### 1. PostgreSQL WAL / Logical Decoding

Learn how PostgreSQL exposes durable WAL-derived logical changes and source positions.

### 2. MySQL Binary Log

Learn how MySQL's binlog represents committed changes and source coordinates.

### 3. Debezium

Learn how a production CDC platform turns database log changes into structured events and durable connector offsets.

These are production vocabulary. They do not replace understanding the underlying mechanism.

---

## 25. Production Runbook

### Symptom: CDC lag is increasing

Check:

1. source transaction rate
2. large transactions
3. consumer throughput
4. destination latency
5. event failure rate
6. partition skew
7. network health
8. source log retention pressure

Do not add workers before identifying the bottleneck and confirming parallelism is safe.

### Symptom: Consumer lost its source position

1. Stop the consumer.
2. Inspect position history.
3. Determine whether the required log range still exists.
4. Resume from the last verified safe position if available.
5. Otherwise rebuild from a consistent snapshot.
6. Establish a valid CDC starting position.
7. Reconcile.

### Symptom: Log retention is approaching its limit

1. Measure lag.
2. Find the slowest partition or transaction.
3. Check destination bottlenecks.
4. Check connector failures.
5. Increase capacity only where safe.
6. Recover backlog.
7. Confirm retention headroom has returned.

### Symptom: Duplicate events after restart

Check that:

- the source position was not advanced before commit
- event identity is unique
- destination application is idempotent
- final business state reconciles correctly

Redelivery after restart can be normal.

### What not to do

- Do not manually skip an unknown source position merely to reduce lag.
- Do not delete CDC state without understanding its recovery effect.
- Do not assume timestamps replace native log positions.
- Do not silently discard malformed events.
- Do not claim global ordering without a source guarantee.
- Do not expose sensitive log payloads in operational logs.

---

## 26. Definition of Done

You can independently:

- explain transaction-log CDC
- distinguish it from table polling
- explain native source positions
- explain commit and transaction boundaries
- design a safe snapshot-to-log handoff
- model INSERT, UPDATE, and DELETE events
- build an idempotent CDC consumer
- persist and safely advance positions
- handle duplicate delivery
- reason about partitions and ordering
- monitor log-retention pressure
- recover when the required log range is unavailable
- test crash and replay behavior
- reconcile source and destination state
- explain why CDC does not automatically mean exactly-once processing

---

## 27. What You Learned

> **Transaction-log CDC captures committed database changes from a durable source log and turns them into a restartable change stream. The hard part is preserving correctness across transaction boundaries, native source positions, destination persistence, redelivery, retention, and recovery.**

The practical mental model is:

    DATABASE TRANSACTION
           ↓
         COMMIT
           ↓
     TRANSACTION LOG
           ↓
       CDC READER
           ↓
      CHANGE EVENT
           ↓
       VALIDATE
           ↓
   IDEMPOTENT PERSISTENCE
           ↓
         COMMIT
           ↓
   ADVANCE SAFE POSITION
           ↓
       RECONCILE

The key question is:

> **What transaction-log position can this consumer prove it has safely processed?**
