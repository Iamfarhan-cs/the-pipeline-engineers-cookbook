# E20 — API Timeouts

## 1. Problem Recognition

An API request can fail without returning an HTTP error because the client is waiting indefinitely for a response.

Recognize a timeout problem when:

- requests hang for an unexpected amount of time
- workers remain busy without making progress
- connection establishment is slow
- the server accepts a request but responds too slowly
- large responses take longer than expected
- one slow dependency blocks many pipeline workers
- retries never start because the first attempt never finishes

A timeout is a **boundary on waiting**. It converts an unbounded dependency into bounded pipeline behavior.

### Typical failure modes

| Failure | Consequence |
|---|---|
| No timeout | Worker can hang indefinitely |
| One global timeout | Different network phases are treated identically |
| Very short timeout | Healthy requests fail unnecessarily |
| Very long timeout | Failed requests consume workers for too long |
| Read timeout ignored | Slow server can stall the pipeline |
| Connect timeout ignored | Network problems hold resources |
| Timeout retried without bounds | Slow dependency becomes retry storm |
| Ambiguous timeout outcome | Server may have processed a request that the client did not observe |

## 2. Concept and Reasoning

Timeouts define how long the client is willing to wait for a specific operation.

HTTP communication can have several waiting phases:

```text
DNS / CONNECTION
       ↓
TLS HANDSHAKE
       ↓
REQUEST SENT
       ↓
SERVER PROCESSING
       ↓
RESPONSE BYTES
       ↓
RESPONSE BODY
```

A production HTTP client should distinguish the phases where the chosen library supports it.

### Connect timeout

How long the client waits to establish the connection.

### Read timeout

How long the client waits for response data.

### Write timeout

How long the client waits while sending request data.

### Total or deadline timeout

An overall upper bound for the operation.

These controls solve different problems.

## 3. The Core Mechanism

```text
START REQUEST
     ↓
SET DEADLINE
     ↓
CONNECT WITH LIMIT
     ↓
SEND WITH LIMIT
     ↓
READ WITH LIMIT
     ↓
SUCCESS / TIMEOUT
     ↓
CLASSIFY
     ↓
RETRY OR FAIL
```

The important principle is:

> A timeout should bound waiting without accidentally allowing retries to extend the total operation forever.

## 4. Implementation

### Step 1 — Define a timeout policy

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class TimeoutPolicy:
    connect_seconds: float = 5.0
    read_seconds: float = 30.0
    write_seconds: float = 30.0
    pool_seconds: float = 5.0
    total_seconds: float = 60.0
```

Keep timeout values configurable and source-specific.

### Step 2 — Requests

Requests supports a simple tuple for connect and read timeouts:

```python
import requests

response = requests.get(
    url,
    timeout=(5, 30),
)
```

The first value limits connection establishment and the second limits waiting for response data.

Do not assume `timeout=30` means the entire operation always completes within exactly 30 seconds. Understand the timeout semantics of the HTTP library and operation.

### Step 3 — HTTPX

HTTPX exposes separate timeout components:

```python
import httpx

timeout = httpx.Timeout(
    30.0,
    connect=5.0,
    read=30.0,
    write=30.0,
    pool=5.0,
)

with httpx.Client(timeout=timeout) as client:
    response = client.get(url)
```

This makes the different waiting phases explicit.

### Step 4 — Catch timeout failures explicitly

```python
import requests

try:
    response = requests.get(
        url,
        timeout=(5, 30),
    )
except requests.Timeout as exc:
    raise RuntimeError("API request timed out") from exc
```

Do not convert every timeout into a generic error that loses the operation, endpoint, attempt, and timeout phase.

## 5. Timeout Classification

Useful categories include:

```text
CONNECT_TIMEOUT
READ_TIMEOUT
WRITE_TIMEOUT
POOL_TIMEOUT
TOTAL_DEADLINE_EXCEEDED
```

The exact categories depend on the HTTP client.

Classification helps answer:

- Is the network path slow?
- Is the server processing too slowly?
- Is the client pool exhausted?
- Is the response body unusually large?
- Is the operation exceeding its total deadline?

## 6. Timeout Values Are Engineering Decisions

Do not choose timeout values randomly.

Consider:

- provider latency
- endpoint behavior
- response size
- network distance
- normal percentile latency
- expected server processing time
- pipeline SLA
- retry policy
- rate limit
- worker concurrency

A useful starting process is:

```text
MEASURE NORMAL LATENCY
        ↓
UNDERSTAND TAIL LATENCY
        ↓
CHOOSE TIMEOUT ABOVE NORMAL TAIL
        ↓
OBSERVE TIMEOUT RATE
        ↓
ADJUST USING EVIDENCE
```

A timeout should not simply be set to an arbitrarily large value because some requests are slow.

## 7. Timeout and Retries

Timeouts and retries must be designed together.

Suppose:

```text
attempt timeout = 30s
retries = 5
```

Without an overall deadline, one logical request can consume several minutes.

Use a total retry budget:

```text
TOTAL DEADLINE = 90s
        ↓
attempt 1 → timeout
        ↓
attempt 2 → timeout
        ↓
attempt 3 → timeout
        ↓
deadline reached
        ↓
FAIL / DEFER
```

Each attempt needs its own timeout, while the logical operation needs an overall deadline.

## 8. Deadline-Aware Retries

A retry should not start if there is insufficient time left for a meaningful attempt.

```python
import time


def remaining_seconds(deadline: float) -> float:
    return max(0.0, deadline - time.monotonic())
```

Before each request:

```python
remaining = remaining_seconds(deadline)

if remaining <= 0:
    raise TimeoutError("operation deadline exceeded")

attempt_timeout = min(30.0, remaining)
```

This prevents a retry from starting after the logical operation has already expired.

## 9. Timeout and Rate Limits

A timeout can increase request pressure when combined with retries.

```text
SLOW API
   ↓
TIMEOUT
   ↓
RETRY
   ↓
TIMEOUT
   ↓
RETRY
   ↓
MORE TRAFFIC
   ↓
API PRESSURE INCREASES
```

Rate limiting prevents the retry system from amplifying pressure.

Never respond to timeouts by automatically increasing concurrency.

## 10. Timeout and Pagination

Large paginated extractions need per-page timeout behavior.

```text
PAGE 1 → 1.2s
PAGE 2 → 1.4s
PAGE 3 → TIMEOUT
          ↓
       RETRY PAGE 3
          ↓
        SUCCESS
          ↓
PAGE 4
```

Do not restart the entire pagination sequence because one page timed out.

The page remains incomplete until it is successfully validated and persisted.

## 11. Timeout and Large Responses

A large response can legitimately take longer to download.

Do not solve every large-response timeout by blindly increasing the timeout.

First investigate:

- page size
- compression
- response payload size
- server processing time
- network bandwidth
- client streaming behavior

Prefer streaming when supported:

```text
REQUEST
  ↓
STREAM RESPONSE
  ↓
PROCESS CHUNKS
  ↓
RELEASE MEMORY
```

## 12. Timeout and Connection Pools

With concurrent extraction, workers may wait for an available connection.

```text
WORKER A ── connection
WORKER B ── connection
WORKER C ── waiting
WORKER D ── waiting
```

A pool timeout can distinguish connection-pool exhaustion from a slow remote server.

Monitor:

- active connections
- pool wait time
- pool size
- worker count
- request latency

Do not solve pool exhaustion simply by creating unlimited connections.

## 13. Ambiguous Timeout Outcomes

A timeout does not always mean the server did nothing.

Example:

```text
CLIENT → POST /operation
           ↓
       SERVER PROCESSES
           ↓
CLIENT TIMEOUT
           ↓
SERVER MAY HAVE COMPLETED
```

This is especially important for non-idempotent operations.

Recovery options may include:

- idempotency keys
- operation-status lookup
- provider request IDs
- reconciliation
- safe retry semantics

Never blindly repeat a potentially completed side-effecting request.

## 14. Timeout and Idempotency

For reads, repeating after a timeout is generally safer.

For writes:

```text
POST
 ↓
TIMEOUT
 ↓
UNKNOWN OUTCOME
```

The correct question is not simply whether the client saw a response. It is whether the server may have committed the operation.

Use provider-supported idempotency or status reconciliation where available.

## 15. Testing

### Unit tests

Test:
- connect timeout classification
- read timeout classification
- write timeout classification where supported
- pool timeout classification where supported
- total deadline expiration
- remaining-time calculation
- retry stops at deadline
- timeout configuration is applied
- timeout values are not accidentally disabled

Example:

```python
def test_remaining_time_never_negative():
    deadline = 100.0

    assert max(0.0, deadline - 150.0) == 0.0
```

### Integration tests

Use a controlled test server that can:

1. delay connection establishment
2. delay response headers
3. delay response body
4. return a large response slowly
5. close the connection unexpectedly
6. hold requests longer than the deadline
7. simulate concurrent pool exhaustion

Verify that the client stops waiting according to the configured policy.

## 16. Observability

Useful metrics:

| Metric | Purpose |
|---|---|
| Request latency | Measures normal and tail behavior |
| Timeout count | Measures bounded-wait failures |
| Connect timeout count | Detects connection problems |
| Read timeout count | Detects slow responses |
| Write timeout count | Detects slow request transmission |
| Pool timeout count | Detects client-side connection pressure |
| Deadline exceeded count | Measures total-operation failures |
| Retry after timeout | Measures recovery activity |
| Response size | Explains slow downloads |

Useful structured log fields:

```text
run_id
request_id
endpoint
attempt
timeout_connect_seconds
timeout_read_seconds
timeout_total_seconds
elapsed_ms
timeout_type
response_size_bytes
retry_attempt
```

Do not log sensitive request or response bodies merely to diagnose timeouts.

### Important operational signal

A rising timeout rate should be investigated alongside latency percentiles, response sizes, connection-pool utilization, provider errors, and retry volume.

## 17. Intentional Failure

### Failure drill A — Connect timeout

Use a test endpoint or controlled network condition that prevents connection establishment within the configured limit.

Expected:
- connection timeout is raised
- worker does not wait indefinitely
- timeout is classified correctly

### Failure drill B — Slow response

Make the server delay response data beyond the read timeout.

Expected:
- read timeout occurs
- request is classified as transient only if policy allows
- retry budget remains bounded

### Failure drill C — Total deadline

Force every attempt to consume most of the available deadline.

Expected:
- retries stop when the overall deadline expires
- no new attempt starts after the deadline

### Failure drill D — Pool exhaustion

Run more workers than the configured connection pool can serve.

Expected:
- pool pressure is visible
- pool timeout is distinguishable from server latency
- the system does not create unlimited connections

### Failure drill E — Ambiguous write

Simulate a server that completes a write but delays the response until the client times out.

Expected:
- client recognizes ambiguous outcome
- operation is reconciled or retried with idempotency protection
- duplicate side effects are prevented

## 18. Recovery

When timeouts increase:

1. Identify the timeout phase.
2. Check latency percentiles.
3. Check response sizes.
4. Check connection-pool utilization.
5. Check provider health and status.
6. Check network conditions.
7. Check retry volume.
8. Determine whether the timeout is client-side or provider-side.
9. Adjust concurrency or page size if evidence supports it.
10. Preserve the last safe checkpoint.
11. Retry only within the retry policy.
12. Reconcile ambiguous writes.

Do not solve an incident by simply increasing every timeout. That can hide a slow dependency while consuming more workers.

## 19. Production Tools You Should Know

### Requests
Python HTTP client with straightforward connect/read timeout configuration.

### HTTPX
Python HTTP client with explicit connect, read, write, and pool timeout controls.

### urllib3
Underlying HTTP library used by many Python clients; useful for understanding connection pools, timeout behavior, and transport-level mechanics.

These tools implement HTTP behavior, but the engineer still owns timeout policy and operational boundaries.

## 20. Production Runbook

### Before running

- Define connect timeout.
- Define read timeout.
- Define write timeout when relevant.
- Define pool timeout when relevant.
- Define total operation deadline.
- Configure request retries separately.
- Confirm idempotency behavior for writes.

### If timeout rate increases

1. Check latency percentiles.
2. Identify timeout phase.
3. Check provider status.
4. Check response sizes.
5. Check connection-pool pressure.
6. Check worker concurrency.
7. Check retry volume.
8. Check network conditions.

### If requests hang

- Verify that a timeout is actually configured.
- Check whether the selected HTTP library's timeout semantics match your expectation.
- Inspect connection-pool behavior.
- Check whether streaming code has its own blocking operation.

### What not to do

- Do not run production HTTP requests without explicit timeout policy.
- Do not confuse read timeout with total operation timeout.
- Do not set enormous timeouts to hide slow dependencies.
- Do not retry timed-out writes blindly.
- Do not let retries bypass the overall deadline.
- Do not increase concurrency when the dependency is timing out.

## 21. Common Mistakes

1. No timeout at all.
2. One arbitrary timeout for every API endpoint.
3. Ignoring connect vs read behavior.
4. Forgetting connection-pool timeouts in concurrent clients.
5. Allowing retries to exceed the total operation deadline.
6. Treating every timeout as proof that the server did nothing.
7. Retrying non-idempotent writes without protection.
8. Increasing timeout values without measuring latency.
9. Ignoring response size and page size.
10. Failing to distinguish client-side pool pressure from provider latency.

## 22. Definition of Done

- [ ] Explicit timeout policy exists.
- [ ] Connect timeout is defined.
- [ ] Read timeout is defined.
- [ ] Write timeout is defined where relevant.
- [ ] Pool timeout is defined where relevant.
- [ ] Overall operation deadline is defined.
- [ ] Timeout errors are classified.
- [ ] Retry policy respects the overall deadline.
- [ ] Pagination retries preserve page state.
- [ ] Large responses are handled intentionally.
- [ ] Connection-pool behavior is understood.
- [ ] Ambiguous write outcomes are handled safely.
- [ ] Timeout metrics exist.
- [ ] Intentional timeout drills pass.
- [ ] Recovery behavior is documented.

## 23. What You Learned

After this recipe, you should be able to independently:
- explain why API requests need bounded waiting
- distinguish connection, read, write, pool, and total timeouts
- configure timeouts in Python HTTP clients
- choose timeout values using latency evidence
- combine timeouts with retries
- enforce an overall operation deadline
- reason about ambiguous timeout outcomes
- protect non-idempotent operations
- diagnose connection-pool pressure
- handle slow and large responses
- observe and recover from timeout failures

### Core Mental Model

```text
START REQUEST
     ↓
SET DEADLINE
     ↓
CONNECT WITH LIMIT
     ↓
SEND WITH LIMIT
     ↓
READ WITH LIMIT
     ↓
SUCCESS / TIMEOUT
     ↓
CLASSIFY
     ↓
RETRY WITHIN DEADLINE
     ↓
SUCCESS / FAIL / DEFER
```

> A timeout is not merely an HTTP setting. It is the boundary that prevents a slow dependency from turning into an unbounded pipeline failure.