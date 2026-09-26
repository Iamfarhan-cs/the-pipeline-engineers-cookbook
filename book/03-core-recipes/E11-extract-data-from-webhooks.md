# E11 — Extract Data from Webhooks

Webhooks are an event-driven extraction source.

Instead of repeatedly polling a provider, the provider sends an event to your endpoint. The production problem is making that delivery durable, authenticated, idempotent, replayable, observable, and safe under retries and duplicates.

## 1. Problem Recognition

Typical problems:

- duplicate event delivery
- provider retries after timeout
- endpoint crashes after receiving a request
- invalid signatures
- out-of-order events
- malformed payloads
- duplicate event IDs with different payloads
- database success followed by HTTP failure
- HTTP success followed by database failure
- bursts that exceed downstream processing capacity
- events that need replay after a bug

The fundamental distinction is:

    HTTP REQUEST RECEIVED
            ≠
    EVENT DURABLY INGESTED
            ≠
    EVENT SUCCESSFULLY PROCESSED

## 2. Webhook Extraction Architecture

    PROVIDER
       ↓
    HTTPS WEBHOOK ENDPOINT
       ↓
    AUTHENTICATE / VERIFY SIGNATURE
       ↓
    VALIDATE ENVELOPE
       ↓
    PERSIST RAW EVENT
       ↓
    COMMIT
       ↓
    RETURN SUCCESS
       ↓
    ASYNC PROCESSOR
       ↓
    TRANSFORM
       ↓
    PERSIST DOWNSTREAM

The endpoint should normally perform only the work required to safely accept the event.

## 3. Delivery Semantics

Most webhook integrations should be designed for at-least-once delivery:

    ONE EVENT
       ↓
    DELIVERY
       ↓
    TIMEOUT
       ↓
    RETRY
       ↓
    SAME EVENT AGAIN

Therefore a webhook consumer must assume duplicate delivery unless the provider explicitly documents stronger semantics.

Even then, defensive idempotency is useful.

## 4. Event Identity

Prefer a provider-generated event ID.

Example:

    {
      "id": "evt_12345",
      "type": "payment.completed"
    }

Store the provider event ID as unique within the provider namespace.

Useful ingestion state:

    provider
    event_id
    event_type
    received_at
    occurred_at
    signature_status
    payload
    processing_status
    attempt_count
    processed_at

## 5. Acknowledgement Race

Bad design:

    WEBHOOK ARRIVES
          ↓
    RETURN 200
          ↓
    PROCESS EVENT
          ↓
    WORKER CRASHES

The provider believes delivery succeeded while your system can lose the event.

Correct ordering:

    RECEIVE
      ↓
    VERIFY
      ↓
    VALIDATE
      ↓
    PERSIST DURABLY
      ↓
    COMMIT
      ↓
    ACKNOWLEDGE

## 6. Implementation — Build the Mechanism

### 6.1 Event model

    from dataclasses import dataclass
    from datetime import datetime

    @dataclass(frozen=True)
    class WebhookEvent:
        provider: str
        event_id: str
        event_type: str
        occurred_at: datetime | None
        received_at: datetime
        payload: bytes
        signature_valid: bool

Keep the raw payload available for replay and investigation.

### 6.2 Event identity

    def event_identity(event: WebhookEvent) -> tuple[str, str]:
        return event.provider, event.event_id

### 6.3 Idempotent ingestion

A teaching implementation:

    def persist_if_new(event, seen: set[tuple]) -> bool:
        identity = event_identity(event)

        if identity in seen:
            return False

        persist_raw_event(event)
        seen.add(identity)
        return True

Production code must enforce uniqueness and persistence atomically in durable storage.

## 7. Database Idempotency

A relational database can enforce the uniqueness boundary.

    CREATE TABLE webhook_event (
        provider TEXT NOT NULL,
        event_id TEXT NOT NULL,
        event_type TEXT NOT NULL,
        received_at TIMESTAMPTZ NOT NULL,
        occurred_at TIMESTAMPTZ,
        signature_valid BOOLEAN NOT NULL,
        payload JSONB NOT NULL,
        processing_status TEXT NOT NULL DEFAULT 'received',
        attempt_count INTEGER NOT NULL DEFAULT 0,
        processed_at TIMESTAMPTZ,
        PRIMARY KEY (provider, event_id)
    );

Insert with duplicate protection:

    INSERT INTO webhook_event (
        provider,
        event_id,
        event_type,
        received_at,
        occurred_at,
        signature_valid,
        payload
    )
    VALUES (
        %(provider)s,
        %(event_id)s,
        %(event_type)s,
        %(received_at)s,
        %(occurred_at)s,
        %(signature_valid)s,
        %(payload)s
    )
    ON CONFLICT (provider, event_id) DO NOTHING;

A duplicate delivery is then a database-enforced property.

## 8. Signature Verification

Webhook providers commonly sign the request body.

The signature may depend on:

    SECRET
      +
    RAW REQUEST BODY
      +
    PROVIDER-SPECIFIC SIGNATURE FORMAT

Do not parse and reserialize JSON before verifying a signature when the provider signs the raw bytes.

Example HMAC verification:

    import hashlib
    import hmac

    def verify_hmac(raw_body: bytes, received_signature: str, secret: bytes) -> bool:
        expected = hmac.new(
            secret,
            raw_body,
            hashlib.sha256,
        ).hexdigest()

        return hmac.compare_digest(
            expected,
            received_signature,
        )

The actual algorithm, header format, timestamp handling, and canonicalization rules must come from the provider contract.

## 9. Replay Protection

A valid signature may not prevent replay of an old valid request.

If the provider includes a signed timestamp, enforce the documented timestamp window.

    from datetime import datetime, timedelta, timezone

    def is_recent(event_time: datetime, max_age: timedelta) -> bool:
        now = datetime.now(timezone.utc)
        return now - event_time <= max_age

Do not invent timestamp semantics that the provider does not define.

## 10. Envelope Validation

Validate before business processing.

Typical checks:

- provider identity
- event ID
- event type
- signature
- payload encoding
- required envelope fields
- timestamp when required
- payload size

Example:

    def validate_event(event: WebhookEvent) -> None:
        if not event.provider:
            raise ValueError("missing provider")

        if not event.event_id:
            raise ValueError("missing event_id")

        if not event.event_type:
            raise ValueError("missing event_type")

        if not event.signature_valid:
            raise ValueError("invalid signature")

## 11. Raw Payload Preservation

The raw event can support:

- debugging
- replay
- auditing
- schema investigation
- downstream reprocessing

Use:

    RAW WEBHOOK
         ↓
    IMMUTABLE INGESTION RECORD
         ↓
    PARSER
         ↓
    NORMALIZED EVENT

Do not mutate the only copy of the source event before persistence.

## 12. Separate Ingestion from Processing

Use two logical stages:

    STAGE 1
    WEBHOOK RECEIVER
        ↓
    RAW EVENT STORE

    STAGE 2
    WORKER
        ↓
    RAW EVENT
        ↓
    TRANSFORM
        ↓
    BUSINESS STORAGE

This prevents slow downstream work from controlling provider HTTP latency.

## 13. Processing Status

A useful lifecycle is:

    RECEIVED
       ↓
    PROCESSING
       ↓
    PROCESSED

Failure can move to:

    PROCESSING
       ↓
    FAILED

Store retry metadata and the latest error.

## 14. Safe Receiver

A simplified receiver:

    def receive_webhook(provider, raw_body, signature):
        validate_size(raw_body)

        if not verify_hmac(
            raw_body,
            signature,
            get_provider_secret(provider),
        ):
            raise ValueError("invalid signature")

        payload = decode_json(raw_body)

        event = build_event(
            provider=provider,
            raw_body=raw_body,
            payload=payload,
        )

        validate_event(event)

        inserted = persist_webhook_event(event)

        return {
            "accepted": True,
            "duplicate": not inserted,
        }

The HTTP framework should return success only after durable acceptance.

## 15. Do Not Process Inside the HTTP Request

Avoid:

    HTTP REQUEST
        ↓
    CALL MULTIPLE SERVICES
        ↓
    TRANSFORM
        ↓
    WRITE DATABASE
        ↓
    RETURN 200

Prefer:

    HTTP REQUEST
        ↓
    VERIFY
        ↓
    PERSIST
        ↓
    RETURN 2xx
        ↓
    WORKER
        ↓
    PROCESS

## 16. Duplicate Delivery

Duplicate events should produce one logical event.

    evt_123
       ↓
    FIRST DELIVERY → INSERT
       ↓
    SECOND DELIVERY → CONFLICT / IGNORE

The second delivery can return success if the event was already durably accepted.

A duplicate is not necessarily an error.

## 17. Same Event ID, Different Payload

This is more serious.

    evt_123 → payload A
    evt_123 → payload B

Do not silently overwrite payload A.

Detect the conflict.

    SAME EVENT ID
         +
    SAME CONTENT
         → DUPLICATE

    SAME EVENT ID
         +
    DIFFERENT CONTENT
         → IDENTITY CONFLICT

A payload hash can support this check:

    import hashlib

    def payload_hash(raw_body: bytes) -> str:
        return hashlib.sha256(raw_body).hexdigest()

## 18. Ordering

Webhook delivery order may differ from business-event order.

For example:

    payment.completed
        arrives first

    payment.created
        arrives later

Do not assume network arrival order equals event occurrence order.

If ordering matters, use documented timestamps, sequence numbers, or provider-specific ordering mechanisms.

## 19. Late Events

A late event may arrive after downstream state has advanced.

Possible policies:

- process based on event time
- update current state
- create a correction
- quarantine
- replay related events

Do not silently discard late events without an explicit policy.

## 20. Missing Events and Reconciliation

Webhook delivery alone may not prove completeness.

If the provider supports a reconciliation API:

    WEBHOOKS
       +
    PERIODIC RECONCILIATION
       ↓
    COMPLETE DATASET

This is especially important for financial and operational data.

## 21. Provider Retries

Providers may retry when:

- the endpoint times out
- the endpoint returns an error
- the connection fails
- the response is not considered successful

The consumer should therefore:

- respond quickly after durable acceptance
- make ingestion idempotent
- record delivery attempts
- monitor retry rates
- avoid success before persistence

## 22. Backpressure

Webhook traffic can arrive faster than downstream processing.

Use:

    WEBHOOK
       ↓
    DURABLE BUFFER
       ↓
    WORKERS
       ↓
    DOWNSTREAM

The buffer decouples ingress rate from processing rate.

## 23. Payload Size Limits

Protect the endpoint from unexpectedly large requests.

    MAX_BODY_BYTES = 5 * 1024 * 1024

    def validate_size(raw_body: bytes) -> None:
        if len(raw_body) > MAX_BODY_BYTES:
            raise ValueError("webhook payload too large")

The actual limit should match provider and infrastructure constraints.

## 24. Schema Validation

Validate the event envelope before business processing.

    def validate_payment_event(payload: dict) -> None:
        required = {"id", "type", "data"}
        missing = required - payload.keys()

        if missing:
            raise ValueError(
                f"missing fields: {sorted(missing)}"
            )

## 25. Testing

Test at least:

1. Valid webhook.
2. Invalid signature.
3. Missing signature.
4. Missing event ID.
5. Duplicate event.
6. Same event ID with different payload.
7. Malformed JSON.
8. Oversized payload.
9. Old timestamp.
10. Future timestamp.
11. Provider retry.
12. Database failure.
13. Database timeout.
14. Worker failure after ingestion.
15. Duplicate processing.
16. Out-of-order events.
17. Late event.
18. Unknown event type.
19. Provider schema change.
20. Burst traffic.
21. Reconciliation of missing events.
22. Replay of stored event.

## 26. Example Unit Tests

    def test_duplicate_event_is_not_inserted_twice():
        seen = set()

        event = WebhookEvent(
            provider="payments",
            event_id="evt-1",
            event_type="payment.completed",
            occurred_at=None,
            received_at=now(),
            payload=b'{"id":"evt-1"}',
            signature_valid=True,
        )

        assert persist_if_new(event, seen)
        assert not persist_if_new(event, seen)

    def test_same_id_different_payload_can_be_detected():
        first = b'{"id":"evt-1","amount":100}'
        second = b'{"id":"evt-1","amount":999}'

        assert payload_hash(first) != payload_hash(second)

The production implementation should enforce the identity rule at the durable persistence boundary.

## 27. Observability

Useful metrics:

    webhook_requests_total
    webhook_signature_failures_total
    webhook_validation_failures_total
    webhook_events_received_total
    webhook_events_accepted_total
    webhook_duplicate_events_total
    webhook_identity_conflicts_total
    webhook_processing_failures_total
    webhook_processing_retries_total
    webhook_processing_duration_seconds
    webhook_queue_depth
    webhook_event_age_seconds
    webhook_payload_bytes_total

Useful log fields:

    run_id
    provider
    event_id
    event_type
    received_at
    occurred_at
    signature_status
    processing_status
    attempt
    duration
    error_type

Never log secrets or complete sensitive payloads unless explicitly permitted.

## 28. Intentional Failure

### Failure 1 — Duplicate delivery

Send the same event twice.

Verify that one logical event exists.

### Failure 2 — Database failure

Make persistence fail before the response.

Verify that the provider receives a failure response and can retry safely.

### Failure 3 — Crash after persistence

Persist the event, then simulate a crash before processing.

Verify that the stored event can be picked up by the worker.

### Failure 4 — Invalid signature

Change the request body without updating the signature.

Verify that the event is rejected.

### Failure 5 — Identity conflict

Send the same event ID with a different payload.

Verify that the conflict is detected rather than overwritten.

### Failure 6 — Slow processing

Make downstream processing intentionally slow.

Verify that the receiver acknowledges after durable ingestion rather than waiting for the full pipeline.

### Failure 7 — Out-of-order delivery

Deliver related events in reverse order.

Verify that event-time or sequence semantics handle the ordering correctly.

## 29. Recovery

1. Identify provider and event ID.
2. Check signature and validation status.
3. Check whether the raw event was durably stored.
4. Inspect processing status.
5. Check attempt count and latest failure.
6. Determine whether the event is a duplicate or identity conflict.
7. Retry transient processing failures.
8. Replay the stored raw event after fixing downstream logic.
9. Reconcile against the provider when missing events are suspected.
10. Verify final downstream state.
11. Mark processing complete only after successful persistence.

Never delete a failed event before understanding whether it is the only durable copy.

## 30. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **FastAPI** | Python HTTP framework commonly used to implement webhook endpoints and request validation. |
| **Stripe Webhooks** | Real-world reference for signed events, event IDs, retries, and webhook delivery semantics. |
| **ngrok** | Development tool for exposing a local webhook endpoint to external providers during testing. |

The important skill is understanding the durable webhook ingestion mechanism independently of any one tool.

## 31. Production Runbook

### Webhook delivery failures are increasing

Check:

1. Endpoint availability.
2. Response latency.
3. Signature failures.
4. Validation failures.
5. Database health.
6. Queue depth.
7. Provider retry behavior.

### Duplicate events are increasing

Check:

1. Provider delivery attempts.
2. Event IDs.
3. Durable uniqueness constraint.
4. Checkpoint timing.
5. Whether the endpoint acknowledges before persistence.

### Events are accepted but not processed

Check:

1. Raw event table.
2. Processing status.
3. Worker health.
4. Queue depth.
5. Worker errors.
6. Retry state.

### Same event ID has different payloads

Treat it as an identity conflict.

Do not overwrite the original event silently.

### What not to do

Do not:

- acknowledge before durable persistence
- process the entire business workflow inside the HTTP request
- trust event IDs without enforcing uniqueness
- overwrite an existing event when the payload differs
- log secrets
- disable signature verification
- assume webhook arrival order equals event order
- discard late events without a documented policy
- retry permanent validation failures forever

## 32. Common Mistakes

### Mistake 1 — Treating HTTP 200 as processing success

A successful HTTP response should normally mean the event was durably accepted, not that the entire downstream workflow completed.

### Mistake 2 — No idempotency

At-least-once delivery makes duplicates normal.

### Mistake 3 — Verifying transformed data

Signature verification can fail if the raw signed request body is transformed before verification.

### Mistake 4 — Processing synchronously

Slow downstream processing causes provider timeouts and retries.

### Mistake 5 — No raw event retention

Without the original event, debugging and replay become harder.

### Mistake 6 — Ignoring ordering

Network delivery order is not necessarily business-event order.

### Mistake 7 — No reconciliation

Webhooks alone may not prove that every expected event was received.

## 33. Definition of Done

You are done when you can:

- explain webhook delivery semantics
- distinguish receipt from processing
- implement a durable webhook boundary
- verify signatures
- preserve raw payloads
- enforce event identity
- handle duplicate delivery
- detect identity conflicts
- separate ingestion from processing
- handle provider retries
- design for out-of-order events
- handle late events
- protect the endpoint with payload limits
- implement durable processing status
- test failure after persistence
- replay stored events
- reconcile missing events
- observe delivery and processing health
- intentionally break the webhook pipeline
- recover without losing accepted events
- operate it with a production runbook

## 34. What You Learned

The central principle is:

> A webhook is an event-delivery mechanism, not a guaranteed processing mechanism. Accept the event durably first, then process it asynchronously and idempotently.

The production pattern is:

    PROVIDER
       ↓
    VERIFY
       ↓
    VALIDATE
       ↓
    PERSIST RAW EVENT
       ↓
    COMMIT
       ↓
    ACKNOWLEDGE
       ↓
    PROCESS ASYNCHRONOUSLY
       ↓
    RETRY / REPLAY / RECONCILE

The key question is:

> If the provider sends this event twice, the worker crashes after acceptance, or the event arrives out of order, can I prove that the event was not lost and recover the correct downstream state?
