# Recipe 26 — Network Failure

A production data pipeline depends on networks constantly. It may call HTTP APIs, databases, object storage, message brokers, internal services, or external providers.

Networks are not reliable execution boundaries.

This recipe teaches how to design pipeline operations that remain safe when network communication fails.

## 1. Problem Recognition

Consider:

```text
pipeline
   ↓
HTTP request
   ↓
remote service
   ↓
request succeeds
   ↓
network response is lost
   ↓
pipeline sees timeout
```

The pipeline now has an ambiguous outcome.

Did the remote operation fail, or did communication with the operation fail?

Common failures include:
- DNS resolution failure
- connection refused
- connection timeout
- read timeout
- connection reset
- broken pipe
- TLS or network handshake failure
- temporary routing failure
- proxy failure
- remote service disconnect
- network partition.

Recognize the problem through timeout exceptions, connection errors, sudden latency increases, intermittent failures, dependency-specific failures, or requests that succeed later without code changes.

## 2. Concept and Reasoning

### Network failure is not the same as operation failure

Suppose:

```text
POST /payment
     ↓
server processes payment
     ↓
server sends response
     ↓
network breaks
     ↓
client timeout
```

The client may conclude FAILED while the remote system may actually be SUCCEEDED.

Retrying blindly can therefore create a duplicate effect.

### Failure categories

| Category | Example | Typical action |
|---|---|---|
| Connection failure | connection refused | retry if transient |
| Timeout | response too slow | retry only if safe |
| Rate limit | HTTP 429 | respect server guidance |
| Server transient | HTTP 503 | bounded retry |
| Client error | HTTP 400 | usually do not retry |
| Authentication | HTTP 401/403 | fix credentials/authorization |
| Ambiguous outcome | response lost after operation | reconcile/idempotency |

Do not treat every network exception as automatically retryable.

## 3. Timeouts Are Mandatory

Never allow a pipeline request to wait indefinitely.

At minimum, distinguish connection timeout, read timeout, and overall operation timeout.

Example:

```python
import requests

response = requests.get(
    'https://example.com/data',
    timeout=(3, 30),
)
```

Here 3 seconds is the connection timeout and 30 seconds is the read timeout.

A timeout converts an unbounded wait into a controlled failure.

## 4. Implementation

### 4.1 Basic retry wrapper

```python
import time

RETRYABLE_ERRORS = (
    TimeoutError,
    ConnectionError,
)

def retry_operation(operation, attempts=3, base_delay=1.0):
    last_error = None

    for attempt in range(attempts):
        try:
            return operation()
        except RETRYABLE_ERRORS as exc:
            last_error = exc
            if attempt == attempts - 1:
                raise
            delay = base_delay * (2 ** attempt)
            time.sleep(delay)

    raise last_error
```

The mechanism is:

```text
failure
  ↓
classify
  ↓
bounded retry
  ↓
backoff
  ↓
retry
```

### 4.2 Add jitter

If many workers fail simultaneously, identical delays can create another burst.

Use:

```text
delay = exponential_backoff + random_jitter
```

Example:

```python
import random

delay = base_delay * (2 ** attempt)
delay += random.uniform(0, 0.25)
```

Use bounded delays in production.

## 5. The Most Dangerous Case: Ambiguous Outcome

Consider:

```text
client
  ↓
POST request
  ↓
server processes operation
  ↓
server sends response
  ↓
network fails
  ↓
client timeout
```

The client does not know the final state.

Possible state:

```text
UNKNOWN
```

This should not automatically become FAILED.

Safe strategies include:
1. idempotency key
2. operation ID
3. status lookup API
4. transactional outbox
5. destination reconciliation
6. durable request state

Example:

```text
operation_id = op-123
```

Retry using the same operation ID so the remote system can recognize the same logical operation.

## 6. Idempotency

Suppose:

```text
POST payment
   ↓
server succeeds
   ↓
response lost
   ↓
client retries
```

Without idempotency, a duplicate effect can occur.

With a stable idempotency key:

```text
same idempotency key
       ↓
same logical operation
       ↓
no duplicate effect
```

Network retries therefore depend heavily on Recipe 6 — Idempotency.

## 7. Testing

Test network failure deliberately.

### Connection failure

Expected:

```text
connection error
→ retry
→ bounded failure
```

### Read timeout

Delay the test server response and verify timeout handling.

### Temporary failure

```text
503
503
200
```

Expected:

```text
retry
retry
success
```

### Permanent failure

```text
400
```

Expected: no blind retry.

### Ambiguous outcome

Simulate the server performing the operation while the response is dropped. Retry with the same idempotency key and verify one logical operation.

### Retry exhaustion

Make every attempt fail and verify maximum attempts, final failure recording, no infinite loop, and recoverable state.

## 8. Observability

Track:

```text
requests_total
requests_succeeded_total
requests_failed_total
request_timeouts_total
connection_failures_total
retry_attempts_total
retry_exhausted_total
ambiguous_outcomes_total
request_duration_seconds
```

Useful dimensions:

```text
dependency
operation
error_type
attempt
status_code
```

Example:

```json
{
  "event": "network_retry",
  "dependency": "payments-api",
  "operation": "create_payment",
  "attempt": 2,
  "error_type": "ReadTimeout",
  "backoff_ms": 1800
}
```

Do not log secrets, credentials, tokens, or sensitive request bodies.

## 9. Intentional Failure

Build a test server with controlled failures:

```text
Attempt 1 → timeout
Attempt 2 → connection reset
Attempt 3 → 503
Attempt 4 → 200
```

Verify classification, backoff, bounded retry count, and final success.

### Ambiguous outcome drill

Simulate:

```text
server commits operation
       ↓
response intentionally dropped
       ↓
client timeout
       ↓
client retries
```

Verify that idempotency prevents a duplicate effect.

## 10. Recovery

When a network error occurs:

1. Classify the error.
2. Check whether the operation is safe to retry.
3. Apply bounded retry with exponential backoff and jitter.
4. Respect server guidance such as Retry-After.
5. Record final state after retry exhaustion.
6. Reconcile ambiguous operations instead of blindly replaying them.

Possible final states include:

```text
FAILED
RETRY_PENDING
QUARANTINED
UNKNOWN
```

Choose the state that accurately represents what is known.

## 11. Retry Policy

A retry policy should explicitly define:

```text
what errors are retryable
maximum attempts
initial delay
maximum delay
jitter
overall timeout
```

Example:

```text
retryable: timeout, connection reset, HTTP 503
max attempts: 4
base delay: 1 second
max delay: 30 seconds
```

Do not retry everything.

## 12. Network Failure and Rate Limiting

Retries must respect rate limits.

Unsafe:

```text
100 workers fail
     ↓
100 workers retry immediately
     ↓
dependency overload
```

Safer:

```text
failure
  ↓
backoff + jitter
  ↓
rate limiter
  ↓
retry
```

Rate limiting controls normal traffic. Backoff controls recovery traffic.

## 13. Network Failure and Partial Failure

Network errors may affect only some records:

```text
100 records
 ├── 95 success
 └── 5 network failures
```

Do not restart all 100 records unnecessarily. Keep the 95 completed records and recover the five affected records.

## 14. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **requests** | Python HTTP client where timeout and connection behavior should be explicit. |
| **Tenacity** | Python retry library supporting retry conditions, backoff, and stop policies. |
| **Envoy** | Service-proxy layer providing timeout, retry, circuit-breaking, and traffic-control mechanisms. |

> These tools implement production mechanisms around network communication. Understand timeout, retry, ambiguity, idempotency, and backoff first.

## 15. Production Runbook

### Timeouts suddenly increase

Check:
1. dependency latency
2. network path
3. DNS
4. connection pool
5. service health
6. request size
7. recent deployment changes

### Connection failures increase

Check DNS resolution, firewall/network policy, service availability, connection limits, proxy configuration, and TLS configuration.

### Retry volume increases

Check retry rate, failure rate, and dependency latency. Do not increase retries before understanding the cause.

### Ambiguous operations appear

Check idempotency key, operation ID, remote status, destination state, and audit record.

### What not to do

Do not:
- retry forever
- retry every exception
- retry state-changing operations blindly
- use no timeout
- ignore Retry-After
- retry without jitter in a large distributed system
- treat every timeout as proof that the remote operation failed

## 16. Common Mistakes

### Mistake 1 — No timeout
A worker can remain blocked indefinitely.

### Mistake 2 — Retry everything
Permanent errors waste resources and increase load.

### Mistake 3 — No backoff
Failures turn into request storms.

### Mistake 4 — No jitter
Many workers retry at the same instant.

### Mistake 5 — Ignoring ambiguous outcomes
A timeout after a successful remote operation can create duplicates.

### Mistake 6 — No retry limit
Transient failure becomes an infinite loop.

### Mistake 7 — Treating network reliability as application reliability
A healthy application can still depend on an unhealthy network path.

## 17. Definition of Done

You are done when you can:
- recognize common network failures
- explain why network failure and operation failure are different
- configure connection and read timeouts
- implement bounded retries
- implement exponential backoff
- add jitter
- classify retryable and permanent failures
- recognize ambiguous outcomes
- use idempotency for safe retries
- intentionally simulate timeout and connection failure
- test retry exhaustion
- test an ambiguous successful operation with a lost response
- observe network failure metrics
- combine retry with rate limiting
- recover only affected records
- explain how requests, Tenacity, and Envoy relate to production network reliability

## 18. What You Learned

The central principle is:

> **A network failure tells you that communication failed; it does not always tell you that the remote operation failed.**

The safe pattern is:

```text
REQUEST
  ↓
TIMEOUT / NETWORK FAILURE?
  ↓
CLASSIFY
  ↓
IS OPERATION SAFE TO RETRY?
  ├── YES → BACKOFF + JITTER → RATE LIMIT → RETRY
  │
  └── NO / UNKNOWN → RECONCILE
                         ↓
                    DETERMINE STATE
                         ↓
                       RECOVER
```

A production pipeline must assume that networks fail and must preserve correctness even when the final state of a remote operation is temporarily uncertain.