# E13 — Extract Data from Kafka

Kafka extraction is different from ordinary queue consumption because a consumer reads an ordered log rather than simply removing messages from a queue.

Unlike a traditional queue, Kafka consumers control their progress through offsets. Reliable Kafka extraction therefore depends on understanding partitions, offsets, consumer groups, commits, rebalancing, retention, ordering, lag, and replay.

The central problem is:

> How do I consume Kafka records, persist them safely, and advance offsets without losing data or creating uncontrolled duplicate processing?

## 1. Problem Recognition

Kafka extraction problems commonly appear as:

- records are processed twice
- offsets advance before data is durable
- records disappear from the expected processing path
- consumers fall behind
- partitions become unevenly loaded
- consumers rebalance unexpectedly
- partition ordering is accidentally destroyed
- a consumer restarts from the wrong offset
- long-running processing causes consumer-group instability
- retention removes records before they are consumed
- malformed records repeatedly fail
- a schema changes unexpectedly
- a consumer needs to replay historical data
- one partition becomes a throughput bottleneck

The fundamental Kafka relationship is:

```
PARTITION
   ↓
OFFSET
   ↓
RECORD
   ↓
PROCESS
   ↓
PERSIST
   ↓
COMMIT OFFSET
```

Do not treat a Kafka offset as proof that downstream data is already durable unless your processing architecture guarantees that relationship.

## 2. Kafka Mental Model

A Kafka topic contains partitions:

```
TOPIC
 ├── PARTITION 0
 │     ├── offset 0
 │     ├── offset 1
 │     └── offset 2
 │
 ├── PARTITION 1
 │     ├── offset 0
 │     ├── offset 1
 │     └── offset 2
 │
 └── PARTITION 2
       ├── offset 0
       ├── offset 1
       └── offset 2
```

An offset identifies a record position within a partition.

Therefore:

```
(topic, partition, offset)
```

is a useful technical identity for a Kafka record.

## 3. Consumer Groups

A consumer group allows multiple consumers to divide partitions:

```
TOPIC
 ├── P0 ── Consumer A
 ├── P1 ── Consumer B
 ├── P2 ── Consumer C
 └── P3 ── Consumer A
```

Within a consumer group, a partition is normally assigned to one consumer at a time.

Adding consumers does not automatically increase throughput beyond the number of partitions available to the group.

Example:

```
4 partitions + 8 consumers
        ↓
4 consumers can own partitions
4 consumers may be idle
```

This is an important capacity constraint.

## 4. Ordering

Kafka guarantees ordering within a partition.

It does not provide one global ordering across all partitions.

Therefore:

```
PARTITION ORDER ≠ GLOBAL ORDER
```

If events belonging to one business entity must remain ordered, producers commonly use a stable key so related records are routed to the same partition.

The consumer should preserve partition order when processing ordered records.

## 5. Offset Semantics

Important offset concepts include:

- current position
- committed offset
- earliest available offset
- latest offset

Conceptually:

```
EARLIEST
   ↓
OFFSET 100
   ↓
CURRENT POSITION
   ↓
OFFSET 150
   ↓
LATEST
```

The consumer's current position is not necessarily the same as its committed position.

This distinction matters during crashes.

## 6. Critical Processing Pattern

Unsafe:

```
POLL
 ↓
COMMIT OFFSET
 ↓
DATABASE WRITE
 ↓
CRASH
```

The database write may never happen, but Kafka considers the record consumed.

Safer:

```
POLL
 ↓
VALIDATE
 ↓
PERSIST
 ↓
COMMIT DATABASE TRANSACTION
 ↓
COMMIT KAFKA OFFSET
```

This still requires idempotent persistence because a crash can occur after database commit but before Kafka offset commit.

## 7. At-Least-Once Extraction

A practical default is:

```
PERSIST
 ↓
CRASH
 ↓
OFFSET NOT COMMITTED
 ↓
RESTART
 ↓
REPROCESS RECORD
```

The result is duplicate processing.

Therefore:

> At-least-once Kafka consumption normally requires an idempotent sink or another mechanism that makes replay safe.

## 8. Record Identity

A Kafka record can be identified by:

```
topic
partition
offset
```

For many ingestion systems this tuple is an excellent technical ingestion identity.

Example database table:

```
CREATE TABLE kafka_ingestion (
    topic TEXT NOT NULL,
    partition_id INTEGER NOT NULL,
    record_offset BIGINT NOT NULL,
    received_at TIMESTAMPTZ NOT NULL,
    payload JSONB NOT NULL,
    PRIMARY KEY (topic, partition_id, record_offset)
);
```

This gives the database a durable uniqueness boundary.

## 9. Implementation — Minimal Consumer

Python example using the Confluent Kafka Python client:

```
from confluent_kafka import Consumer


config = {
    "bootstrap.servers": "localhost:9092",
    "group.id": "payment-ingestion",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": False,
}

consumer = Consumer(config)
consumer.subscribe(["payments"])

try:
    while True:
        message = consumer.poll(1.0)

        if message is None:
            continue

        if message.error():
            raise RuntimeError(message.error())

        print(
            message.topic(),
            message.partition(),
            message.offset(),
            message.value(),
        )

finally:
    consumer.close()
```

The important setting is:

```
enable.auto.commit = False
```

The consumer now controls when offsets are committed.

## 10. Durable Processing

A simplified processing boundary:

```
def process_message(message, database):
    identity = (
        message.topic(),
        message.partition(),
        message.offset(),
    )

    inserted = database.insert_if_new(
        identity=identity,
        payload=message.value(),
    )

    return inserted
```

Then:

```
processed = process_message(message, database)

database.commit()

consumer.commit(message=message)
```

The exact database transaction and offset commit strategy must be designed carefully for the application's failure semantics.

## 11. Database Idempotency

Example:

```
INSERT INTO kafka_ingestion (
    topic,
    partition_id,
    record_offset,
    received_at,
    payload
)
VALUES (
    %(topic)s,
    %(partition)s,
    %(offset)s,
    %(received_at)s,
    %(payload)s
)
ON CONFLICT (
    topic,
    partition_id,
    record_offset
) DO NOTHING;
```

If the same Kafka record is replayed, the database does not create a second ingestion record.

## 12. Polling

Kafka consumers continuously poll for records.

Conceptually:

```
POLL
 ↓
RECEIVE BATCH
 ↓
PROCESS
 ↓
POLL AGAIN
```

The poll loop must remain healthy.

Long processing inside the consumer thread can cause group-management problems if the consumer fails to poll within the configured limits.

This is one reason to separate polling from heavy processing when processing duration is unpredictable.

## 13. Batch Polling

Consumers commonly process several records per poll:

```
POLL
 ↓
R1 R2 R3 R4 R5
 ↓
PROCESS
 ↓
COMMIT SAFE PROGRESS
```

Batching can improve throughput but complicates failure handling.

If:

```
R1 → success
R2 → success
R3 → failure
R4 → not processed
R5 → not processed
```

the consumer must not blindly commit an offset that would skip R3–R5.

Offset advancement must reflect the successfully durable processing boundary.

## 14. Per-Partition Progress

Kafka offsets are partition-specific.

Example:

```
P0 → committed offset 100
P1 → committed offset 240
P2 → committed offset 75
```

Do not store a single global Kafka offset for a multi-partition topic.

Use:

```
(topic, partition) → committed offset
```

## 15. Consumer Rebalancing

A rebalance occurs when partition ownership changes.

Typical causes:

- consumer joins
- consumer leaves
- consumer crashes
- group membership changes
- partition count changes
- group-management conditions trigger reassignment

During rebalancing, a partition may move from one consumer to another.

The new owner resumes from the group's committed position.

Therefore, committed offsets must represent a safe restart point.

## 16. Rebalance Safety

Unsafe:

```
PROCESS RECORD
 ↓
NO DURABLE COMMIT
 ↓
REBALANCE
 ↓
NEW CONSUMER
 ↓
PROCESS AGAIN
```

This is not necessarily data loss.

With idempotent processing it becomes safe duplicate processing.

The dangerous case is:

```
COMMIT OFFSET
 ↓
DATA NOT DURABLE
 ↓
REBALANCE
 ↓
RECORD SKIPPED
```

This can cause data loss.

## 17. Kafka Lag

Consumer lag represents how far the consumer is behind the available records.

Conceptually:

```
LATEST OFFSET
      -
COMMITTED OFFSET
      =
LAG
```

Exact lag reporting depends on the monitoring system and offset state being compared.

Useful signals include:

- records behind
- oldest record age
- processing latency
- records consumed per second
- records produced per second

Lag should be interpreted together with message age and processing throughput.

## 18. Backpressure

If producers generate:

```
10,000 records/sec
```

but consumers process:

```
7,000 records/sec
```

backlog grows.

Possible responses:

- increase consumer capacity
- increase partition count when appropriate
- optimize processing
- batch downstream writes
- reduce unnecessary work
- control downstream concurrency
- temporarily slow producers where the architecture permits it

Do not simply add consumers if the topic has too few partitions.

## 19. Partition Skew

Suppose:

```
P0 → 90% of records
P1 → 5%
P2 → 5%
```

Consumers assigned to P1/P2 may be mostly idle while P0 becomes the bottleneck.

This can be caused by a poor partitioning key.

Monitor:

- records per partition
- bytes per partition
- lag per partition
- processing time per partition

## 20. Replay

Kafka's retained log makes historical replay possible when the required records still exist.

A replay can be modeled as:

```
CHOOSE TOPIC
      ↓
CHOOSE PARTITIONS
      ↓
CHOOSE START OFFSET / TIME
      ↓
READ RECORDS
      ↓
REPROCESS
```

Replay should normally use a separate consumer group so production consumption is not unintentionally moved.

## 21. Starting from a Timestamp

A consumer may need to process records from a particular time rather than a known offset.

Conceptually:

```
TIMESTAMP
   ↓
FIND OFFSET PER PARTITION
   ↓
SEEK
   ↓
CONSUME
```

This is useful for backfills and incident recovery.

The exact API depends on the Kafka client.

## 22. Retention

Kafka is not an infinite archive.

Records can disappear according to retention policies.

Therefore:

```
RETENTION WINDOW
      ↓
CONSUMER MUST CATCH UP
      ↓
OR
      ↓
REQUIRED DATA MUST EXIST ELSEWHERE
```

If a consumer falls behind beyond retention, the original records may no longer be available.

This makes lag and consumer health operationally important.

## 23. Schema Handling

Kafka payloads can be:

- JSON
- Avro
- Protobuf
- plain bytes
- another application-defined format

The consumer should know the serialization contract.

A safe ingestion boundary validates:

```
TOPIC
 ↓
MESSAGE METADATA
 ↓
DESERIALIZATION
 ↓
SCHEMA VALIDATION
 ↓
DURABLE STORAGE
```

Schema evolution should be compatible with the producer/consumer contract.

## 24. Tombstones and Null Values

Kafka records may contain a null value.

This can have semantic meaning, particularly in compacted topics.

For example:

```
KEY = customer-123
VALUE = null
```

may represent a deletion/tombstone.

Do not automatically classify a null value as a malformed record.

Understand the topic's contract first.

## 25. Error Handling

Separate failures into categories.

### Retryable

- database temporarily unavailable
- transient network failure
- temporary downstream timeout

### Non-retryable

- malformed serialization
- unsupported schema version
- permanently invalid business payload

### Operational

- authentication failure
- incorrect topic configuration
- missing permissions
- unavailable broker

Each category needs a different response.

## 26. Poison Records

A permanently invalid record can repeatedly block processing.

A safe design should:

1. capture the record identity
2. capture the failure
3. preserve the original payload when permitted
4. record the processing attempt
5. move the record into an appropriate quarantine/DLQ workflow
6. advance past it only according to an explicit policy

Never silently skip an offset because a record is difficult.

## 27. Graceful Shutdown

A consumer should stop safely:

```
STOP ACCEPTING NEW WORK
       ↓
FINISH SAFE IN-FLIGHT WORK
       ↓
PERSIST
       ↓
COMMIT SAFE OFFSETS
       ↓
LEAVE GROUP
       ↓
CLOSE CONSUMER
```

If work cannot be completed safely, allow it to be replayed rather than committing an unsafe offset.

## 28. Testing

Test at least:

1. One record.
2. Multiple partitions.
3. Multiple consumers.
4. Duplicate processing.
5. Crash after database commit.
6. Crash before database commit.
7. Commit failure.
8. Rebalance.
9. Consumer restart.
10. Invalid serialization.
11. Unsupported schema version.
12. Null/tombstone record.
13. Poison record.
14. Retryable database failure.
15. Growing consumer lag.
16. Partition skew.
17. Batch partial failure.
18. Replay from an offset.
19. Replay from a timestamp.
20. Retention boundary.
21. Graceful shutdown.
22. Topic partition expansion.

## 29. Example Unit Tests

```
def test_kafka_identity_is_partition_specific():
    first = ("payments", 0, 100)
    second = ("payments", 1, 100)

    assert first != second


def test_same_topic_partition_offset_is_duplicate():
    processed = {
        ("payments", 0, 100)
    }

    identity = ("payments", 0, 100)

    assert identity in processed


def test_same_offset_in_different_partition_is_not_duplicate():
    processed = {
        ("payments", 0, 100)
    }

    identity = ("payments", 1, 100)

    assert identity not in processed
```

These tests teach the key identity rule. Integration tests should use a real Kafka-compatible environment to verify actual offset and consumer-group behavior.

## 30. Observability

Monitor at minimum:

```
kafka_records_received_total
kafka_records_processed_total
kafka_records_failed_total
kafka_records_retried_total
kafka_duplicate_records_total
kafka_consumer_lag
kafka_oldest_record_age
kafka_processing_duration_seconds
kafka_poll_errors_total
kafka_commit_errors_total
kafka_rebalances_total
kafka_records_per_partition
kafka_bytes_per_partition
kafka_dead_letter_total
```

Useful log fields:

```
consumer_group
consumer_id
topic
partition
offset
record_key
delivery_attempt
processing_status
processing_duration
error_type
```

Do not log sensitive payloads by default.

## 31. Intentional Failure

### Failure 1 — Commit before persistence

Commit the Kafka offset before writing to the database.

Then simulate a database failure.

Observe that the record can be skipped after restart.

Fix the ordering.

### Failure 2 — Crash after persistence

Persist a record and crash before committing the offset.

Restart the consumer.

Verify the record is redelivered and the sink remains idempotent.

### Failure 3 — Consumer lag

Artificially slow processing.

Observe lag increasing.

Restore processing capacity and verify lag decreases.

### Failure 4 — Rebalance

Stop one consumer while processing a partition.

Observe partition reassignment.

Verify the new consumer resumes from the committed offset.

### Failure 5 — Poison record

Inject an invalid record.

Verify the consumer does not retry it indefinitely.

### Failure 6 — Partition skew

Generate disproportionate traffic for one key.

Observe per-partition lag.

Investigate the partitioning strategy.

### Failure 7 — Commit failure

Force offset commit failure.

Verify the consumer does not assume the offset was durably recorded.

## 32. Recovery

1. Identify the affected consumer group.
2. Check topic and partition health.
3. Check consumer lag.
4. Identify the affected partition and offset.
5. Determine whether the record was persisted.
6. Inspect committed offsets.
7. Check consumer rebalances.
8. Inspect database/downstream failures.
9. Reprocess from the safe committed position when required.
10. Verify idempotent sink behavior.
11. Reconcile source and destination counts/state.
12. Confirm lag returns to normal.

For replay, prefer a separate consumer group unless intentionally changing production consumption state.

## 33. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Apache Kafka** | Distributed event log using topics, partitions, offsets, consumer groups, and retention. |
| **Confluent Kafka Python Client** | High-performance Python client for Kafka consumers and producers. |
| **Redpanda** | Kafka-compatible streaming platform useful for local development and Kafka-style workloads. |

The goal is to understand Kafka's mechanics rather than memorize one client library.

## 34. Production Runbook

### Consumer lag is increasing

Check:

1. Producer rate.
2. Consumer processing rate.
3. Partition-level lag.
4. Consumer count.
5. Partition count.
6. Database latency.
7. Downstream dependency latency.
8. Rebalance frequency.
9. Consumer errors.

### Records are duplicated

Check:

1. Offset commit timing.
2. Consumer crashes.
3. Rebalances.
4. Database idempotency.
5. Processing duration.
6. Batch commit boundaries.

### Records appear to be missing

Check:

1. Committed offsets.
2. Retention.
3. Offset-reset behavior.
4. Consumer-group changes.
5. Database persistence.
6. Whether offsets were committed before durable processing.

### One partition is far behind

Check:

1. Partition traffic.
2. Partition key distribution.
3. Consumer assignment.
4. Processing time.
5. Hot keys.

### What not to do

Do not:

- commit offsets before durable processing
- assume offsets are globally ordered
- assume Kafka provides global ordering
- assume more consumers always increase throughput
- ignore partition-level lag
- reset offsets without understanding the recovery consequence
- silently skip poison records
- treat tombstones as automatically invalid
- use production consumer groups casually for replay

## 35. Definition of Done

You are done when you can:

- explain topics and partitions
- explain offsets
- explain consumer groups
- preserve partition ordering
- identify Kafka records using topic/partition/offset
- manually control offset commits
- build an idempotent consumer
- understand at-least-once processing
- handle consumer rebalancing
- measure consumer lag
- diagnose partition skew
- handle poison records
- perform controlled replay
- understand retention
- handle schema and serialization failures
- handle tombstone records
- perform batch consumption safely
- recover after consumer crashes
- intentionally break offset handling
- diagnose the resulting failure
- recover without silently losing records

## 36. What You Learned

Kafka extraction is fundamentally about coordinating **record position and durable processing**.

The core pattern is:

```
POLL
  ↓
IDENTIFY
  ↓
VALIDATE
  ↓
PROCESS
  ↓
PERSIST
  ↓
COMMIT SAFE OFFSET
  ↓
MONITOR LAG
  ↓
REPLAY / RECOVER WHEN REQUIRED
```

The most important rule is:

> Never advance a Kafka offset past data that you cannot prove is safely durable.

Once that principle is understood, consumer groups, rebalancing, replay, lag, batching, and recovery become operational extensions of the same idea.
