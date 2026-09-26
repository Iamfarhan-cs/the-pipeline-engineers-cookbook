# E12 — Extract Data from Message Queues

Message queues are a common extraction boundary when another system publishes work or events asynchronously.

Unlike file extraction or API polling, a queue gives the consumer a stream of messages that must be **received, acknowledged, retried, and checkpointed according to queue-specific delivery semantics**.

The extraction problem is not simply reading messages. A production consumer must prevent message loss, tolerate duplicate delivery, handle poison messages, control concurrency, and make downstream persistence idempotent.

## 1. Problem Recognition

Typical production problems include:

- the same message is delivered more than once
- a consumer crashes after receiving a message
- a message is acknowledged before persistence
- processing takes longer than the visibility/ack deadline
- messages arrive faster than workers can process them
- one malformed message repeatedly fails
- consumers compete for the same queue
- messages are reordered
- a queue grows continuously
- a consumer restarts with work still in flight
- the downstream database becomes unavailable
- a message is too large or malformed
- retry behavior creates a poison-message loop

The fundamental distinction is:

```text
MESSAGE RECEIVED
      ≠
MESSAGE PROCESSED
      ≠
MESSAGE ACKNOWLEDGED
```

## 2. Queue Extraction Architecture

A reliable queue consumer should use:

```text
MESSAGE QUEUE
     ↓
RECEIVE
     ↓
VALIDATE
     ↓
PROCESS
     ↓
PERSIST
     ↓
ACKNOWLEDGE
     ↓
METRICS / CHECKPOINT STATE
```

For durable ingestion:

```text
MESSAGE QUEUE
     ↓
RECEIVE
     ↓
VALIDATE
     ↓
DURABLE INGESTION
     ↓
COMMIT
     ↓
ACK
     ↓
ASYNC TRANSFORMATION
```

Which architecture is appropriate depends on the queue and the required processing semantics.

## 3. Queue Semantics

Different systems expose different guarantees.

Important concepts include:

- at-least-once delivery
- visibility timeout
- acknowledgement
- negative acknowledgement
- redelivery
- dead-letter queues
- consumer groups
- offsets
- partitions
- ordering
- retention
- delivery attempts

Do not assume that "queue" means exactly-once processing.

For many queues, the safe application assumption is:

```text
DELIVERY IS AT-LEAST-ONCE
        ↓
CONSUMER MUST BE IDEMPOTENT
```

## 4. Acknowledgement Ordering

Bad:

```text
RECEIVE
  ↓
ACK
  ↓
DATABASE WRITE
  ↓
CRASH
```

The message can be lost.

Better:

```text
RECEIVE
  ↓
VALIDATE
  ↓
PERSIST
  ↓
COMMIT
  ↓
ACK
```

If the consumer crashes after persistence but before acknowledgement, the queue may redeliver the message. Idempotent persistence handles that duplicate.

## 5. Implementation — Build the Mechanism

Start with a provider-neutral message model.

### 5.1 Message model

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass(frozen=True)
class Message:
    message_id: str
    payload: bytes
    received_at: datetime
    delivery_attempt: int = 1
```

The message ID should be stable across redelivery when the provider supports it.

### 5.2 Message identity

```python
def message_identity(message: Message) -> str:
    return message.message_id
```

If a provider does not supply a stable ID, derive identity from a documented event ID or another source-defined identifier. Avoid using arrival time as identity.

### 5.3 Idempotent message processing

Teaching implementation:

```python
def process_once(message: Message, processed: set[str]) -> bool:
    if message.message_id in processed:
        return False

    persist_message(message)
    processed.add(message.message_id)

    return True
```

Production code must make the uniqueness check and durable persistence atomic.

## 6. Durable Message Ingestion

A relational database can provide the idempotency boundary.

```sql
CREATE TABLE queue_message (
    message_id TEXT PRIMARY KEY,
    received_at TIMESTAMPTZ NOT NULL,
    delivery_attempt INTEGER NOT NULL,
    payload JSONB NOT NULL,
    processing_status TEXT NOT NULL DEFAULT 'received',
    processed_at TIMESTAMPTZ
);
```

Insert safely:

```sql
INSERT INTO queue_message (
    message_id,
    received_at,
    delivery_attempt,
    payload
)
VALUES (
    %(message_id)s,
    %(received_at)s,
    %(delivery_attempt)s,
    %(payload)s
)
ON CONFLICT (message_id) DO NOTHING;
```

This makes duplicate delivery safe at the storage boundary.

## 7. Consumer Loop

A simple consumer loop:

```python
def consume(queue, processed):
    while True:
        messages = queue.receive(max_messages=10)

        if not messages:
            continue

        for message in messages:
            try:
                if process_once(message, processed):
                    queue.ack(message)
                else:
                    queue.ack(message)

            except Exception:
                queue.retry(message)
```

This is a teaching model.

Production consumers need explicit timeout, retry, acknowledgement, shutdown, and concurrency handling.

## 8. Better Consumer Boundary

Separate receipt from business processing:

```python
def consume_one(queue, store):
    message = queue.receive_one()

    if message is None:
        return

    validate_message(message)

    inserted = store.persist_if_new(message)

    if inserted:
        queue.ack(message)
    else:
        queue.ack(message)
```

The important rule is that acknowledgement occurs after durable acceptance.

## 9. Visibility Timeout

Some queues hide a message temporarily after delivery.

Example:

```text
MESSAGE
   ↓
RECEIVED
   ↓
INVISIBLE FOR 60s
   ↓
PROCESSING
   ↓
ACK
```

If processing takes longer than the visibility period:

```text
CONSUMER A
   ↓
PROCESSING
   ↓
TIMEOUT EXPIRES

CONSUMER B
   ↓
RECEIVES SAME MESSAGE
```

This creates concurrent duplicate processing.

The consumer may need to extend the visibility deadline while long work is running.

## 10. Visibility Extension

Generic example:

```python
def process_with_heartbeat(
    queue,
    message,
    process,
    extend_visibility,
    interval_seconds=20,
):
    # Real implementations should run the heartbeat
    # concurrently with the work.
    process(message)
```

The exact mechanism is provider-specific.

The important design principle is:

> The acknowledgement deadline must be compatible with the maximum expected processing time, or the consumer must extend it safely.

## 11. Poison Messages

A poison message repeatedly fails:

```text
MESSAGE
  ↓
FAIL
  ↓
RETRY
  ↓
FAIL
  ↓
RETRY
  ↓
FAIL
  ↓
...
```

This can consume worker capacity indefinitely.

Use bounded retries:

```text
ATTEMPT 1
   ↓
ATTEMPT 2
   ↓
ATTEMPT 3
   ↓
DEAD-LETTER / QUARANTINE
```

The exact retry limit depends on the business and queue contract.

## 12. Dead-Letter Handling

A dead-letter record should preserve enough information to investigate and replay:

```text
message_id
original_queue
failure_reason
delivery_attempts
first_received_at
last_failed_at
payload
error_type
```

Do not throw away the only copy of a failed message.

## 13. Retry Classification

Not every failure should be retried.

### Usually retryable

- temporary database outage
- connection reset
- timeout
- temporary dependency failure
- rate limit

### Usually not retryable

- invalid schema
- permanently invalid payload
- unsupported event type
- authentication/configuration error
- deterministic business-rule rejection

The exact classification belongs to the application contract.

## 14. Message Validation

Validate before expensive processing.

```python
def validate_message(message: Message) -> None:
    if not message.message_id:
        raise ValueError("missing message_id")

    if not message.payload:
        raise ValueError("empty payload")
```

For JSON messages:

```python
import json


def parse_json_message(message: Message) -> dict:
    try:
        return json.loads(message.payload)
    except json.JSONDecodeError as exc:
        raise ValueError("invalid JSON") from exc
```

Detailed JSON extraction belongs to E06. This recipe focuses on the queue boundary.

## 15. Ordering

Queues differ in ordering guarantees.

Possible models:

```text
GLOBAL ORDER
PARTITION ORDER
KEY ORDER
NO ORDER GUARANTEE
```

Do not assume global ordering unless the provider explicitly guarantees it.

If business ordering matters, use a documented ordering key or sequence number.

## 16. Consumer Concurrency

Multiple workers can process messages concurrently:

```text
QUEUE
 ├── WORKER A
 ├── WORKER B
 ├── WORKER C
 └── WORKER D
```

Higher concurrency can increase throughput.

But it can also increase:

- database load
- downstream API load
- duplicate concurrent processing
- memory usage
- contention

Concurrency is a capacity decision, not simply a performance switch.

## 17. Bounded Concurrency

Teaching example:

```python
from concurrent.futures import ThreadPoolExecutor


def process_batch(messages, process, workers=4):
    with ThreadPoolExecutor(max_workers=workers) as executor:
        list(executor.map(process, messages))
```

In production, the concurrency limit should be based on downstream capacity and queue behavior.

## 18. Backpressure

If:

```text
MESSAGE ARRIVAL RATE > PROCESSING RATE
```

queue depth increases.

Useful controls include:

- bounded worker concurrency
- batch receives
- controlled polling
- downstream rate limits
- autoscaling
- backoff
- circuit breakers where appropriate

Do not blindly increase workers when the database is already overloaded.

## 19. Batch Receiving

Many queues support receiving multiple messages.

Conceptually:

```text
RECEIVE 10
   ↓
PROCESS 10
   ↓
ACK SUCCESSFUL MESSAGES
   ↓
RETRY FAILED MESSAGES
```

Batching reduces network overhead but makes failure handling more complex.

Do not acknowledge an entire batch if only some messages succeeded unless the provider semantics explicitly support that behavior.

## 20. Graceful Shutdown

A consumer should stop accepting new work and finish or safely release in-flight work.

```text
SIGTERM
   ↓
STOP POLLING
   ↓
FINISH SAFE WORK
   ↓
ACK COMPLETED
   ↓
RELEASE / LET UNFINISHED WORK RETRY
   ↓
CLOSE CONNECTION
```

This prevents deployments from unnecessarily losing or duplicating work.

## 21. Large Messages

Message size limits vary by provider.

For large payloads, a common pattern is:

```text
QUEUE MESSAGE
    ↓
OBJECT STORAGE REFERENCE
    ↓
LARGE PAYLOAD
```

The queue carries metadata while the large artifact lives in object storage.

Do not assume this pattern is always necessary; use it when message-size constraints justify it.

## 22. Message Schema Evolution

Consumers should handle compatible schema changes deliberately.

Example:

```json
{
  "event_version": 2,
  "payment_id": "p-123"
}
```

Use explicit versions when the producer contract supports them.

A consumer should distinguish:

- supported version
- unknown version
- malformed payload
- missing required field

## 23. Checkpointing

For many queue systems, the acknowledgement itself represents the consumer's progress.

For systems with explicit offsets or partitions:

```text
MESSAGE
   ↓
PROCESS
   ↓
PERSIST
   ↓
COMMIT OFFSET
```

The same principle remains:

> Advance consumer state only after the corresponding data is safely persisted.

## 24. Recovery After Consumer Crash

Consider:

```text
RECEIVE MESSAGE
      ↓
PERSIST SUCCESSFULLY
      ↓
CRASH
      ↓
NO ACK
      ↓
MESSAGE REDELIVERED
```

The second attempt must be safe.

Durable idempotency turns this from a data-loss problem into a duplicate-delivery event.

## 25. Testing

Test at least:

1. Valid message.
2. Empty message.
3. Invalid JSON.
4. Duplicate message.
5. Same ID with different payload.
6. Database failure.
7. Database timeout.
8. Ack failure.
9. Consumer crash after persistence.
10. Consumer crash before persistence.
11. Visibility timeout.
12. Long-running processing.
13. Poison message.
14. Dead-letter routing.
15. Retryable failure.
16. Permanent failure.
17. Out-of-order messages.
18. High queue depth.
19. Batch processing.
20. Partial batch failure.
21. Graceful shutdown.
22. Schema version change.
23. Large message.
24. Concurrent consumers.

## 26. Example Unit Tests

```python
def test_duplicate_message_is_processed_once():
    processed = set()

    message = Message(
        message_id="m-1",
        payload=b'{"value": 10}',
        received_at=now(),
    )

    assert process_once(message, processed)
    assert not process_once(message, processed)


def test_redelivery_with_same_id_is_duplicate():
    processed = {"m-1"}

    redelivery = Message(
        message_id="m-1",
        payload=b'{"value": 10}',
        received_at=now(),
        delivery_attempt=2,
    )

    assert not process_once(redelivery, processed)


def test_different_message_ids_are_independent():
    processed = {"m-1"}

    message = Message(
        message_id="m-2",
        payload=b'{"value": 20}',
        received_at=now(),
    )

    assert process_once(message, processed)
```

The production implementation should enforce these properties in durable storage.

## 27. Observability

Useful metrics:

```text
queue_messages_received_total
queue_messages_processed_total
queue_messages_failed_total
queue_messages_retried_total
queue_messages_dead_lettered_total
queue_duplicate_messages_total
queue_processing_duration_seconds
queue_receive_latency_seconds
queue_ack_failures_total
queue_depth
queue_oldest_message_age_seconds
queue_visibility_extensions_total
queue_consumer_concurrency
```

Useful log fields:

```text
consumer_id
queue_name
message_id
delivery_attempt
received_at
processing_started_at
processing_duration
processing_status
error_type
```

Do not log sensitive message payloads by default.

## 28. Intentional Failure

### Failure 1 — Duplicate delivery

Deliver the same message twice.

Verify only one logical result is persisted.

### Failure 2 — Crash after persistence

Persist successfully, then simulate a crash before acknowledgement.

Verify redelivery does not duplicate downstream state.

### Failure 3 — Crash before persistence

Fail before the database transaction commits.

Verify the message remains available for retry.

### Failure 4 — Visibility timeout

Make processing longer than the visibility period.

Observe redelivery.

Then fix the design using appropriate visibility extension or processing-time configuration.

### Failure 5 — Poison message

Create a permanently invalid message.

Verify bounded retries and dead-letter handling.

### Failure 6 — Database outage

Stop the database during processing.

Verify transient failures retry without acknowledging messages prematurely.

### Failure 7 — Partial batch failure

Make one message fail while others succeed.

Verify only successfully processed messages are acknowledged when the queue semantics permit per-message acknowledgement.

## 29. Recovery

1. Identify queue and consumer.
2. Inspect queue depth and oldest-message age.
3. Identify failing message IDs.
4. Determine whether failures are transient or permanent.
5. Inspect delivery attempts.
6. Check database and downstream dependencies.
7. Inspect dead-letter messages.
8. Fix the underlying issue.
9. Replay eligible dead-letter messages.
10. Verify idempotency during replay.
11. Reconcile downstream state.
12. Confirm queue depth returns to the expected operating range.

Never purge a growing queue merely to make queue-depth metrics look healthy.

## 30. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Amazon SQS** | Managed queue with visibility timeout, acknowledgements/deletes, retries, and dead-letter queues. |
| **RabbitMQ** | Message broker with acknowledgements, consumer delivery, queues, exchanges, and routing. |
| **Azure Service Bus** | Managed messaging platform with queues, locks, retries, dead-lettering, and delivery semantics. |

The objective is to understand queue mechanics first, then recognize how each platform implements them.

## 31. Production Runbook

### Queue depth is increasing

Check:

1. Consumer health.
2. Processing latency.
3. Consumer concurrency.
4. Database health.
5. Downstream dependency latency.
6. Retry rate.
7. Dead-letter rate.
8. Message arrival rate.

### Messages are duplicated

Check:

1. Acknowledgement timing.
2. Visibility timeout.
3. Consumer crashes.
4. Durable idempotency constraint.
5. Concurrent processing.

### Messages are disappearing

Check:

1. Whether acknowledgement happens before persistence.
2. Consumer shutdown behavior.
3. Queue retention.
4. Visibility configuration.
5. Dead-letter routing.

### Poison messages are consuming workers

Check:

1. Delivery attempts.
2. Error classification.
3. Retry policy.
4. Dead-letter configuration.

### What not to do

Do not:

- acknowledge before persistence
- assume exactly-once processing
- retry permanent validation errors forever
- increase concurrency without checking downstream capacity
- ignore visibility timeout
- purge queues to hide backlog
- discard dead-letter messages without investigation
- assume message arrival order is business order

## 32. Common Mistakes

### Mistake 1 — Treating acknowledgement as processing

Acknowledgement is a queue-state operation, not proof that downstream business state is correct.

### Mistake 2 — No idempotency

Redelivery is normal in many queue systems.

### Mistake 3 — Ignoring visibility timeout

Long-running work can create concurrent duplicate consumers.

### Mistake 4 — Infinite retries

Poison messages can consume the entire worker pool.

### Mistake 5 — Unbounded concurrency

More consumers can overload the database or downstream services.

### Mistake 6 — No dead-letter strategy

Permanent failures need a controlled destination.

### Mistake 7 — Ignoring queue age

Queue depth alone does not show how long the oldest message has waited.

## 33. Definition of Done

You are done when you can:

- explain queue delivery semantics
- distinguish receive, process, and acknowledge
- implement durable message identity
- make processing idempotent
- use acknowledgements correctly
- understand visibility timeouts
- handle redelivery
- classify retryable and permanent failures
- design dead-letter handling
- control consumer concurrency
- apply backpressure
- process batches safely
- handle partial batch failure
- handle graceful shutdown
- handle schema evolution
- recover after consumer crashes
- observe queue depth and message age
- intentionally break the consumer
- recover without losing messages
- operate the queue consumer with a runbook

## 34. What You Learned

The central principle is:

> A queue consumer must treat delivery as potentially repeated and acknowledge a message only after the corresponding state is safely persisted.

The production pattern is:

```text
RECEIVE
   ↓
VALIDATE
   ↓
PERSIST
   ↓
COMMIT
   ↓
ACKNOWLEDGE
   ↓
MONITOR
   ↓
RETRY / DEAD-LETTER / REPLAY
```

The key question is:

> If the consumer crashes after processing, the message is redelivered, or the database becomes unavailable, can I prove that the message will neither be silently lost nor create uncontrolled duplicate state?
