# E19 — API Retries

## 1. Problem Recognition

API extraction pipelines depend on networks and remote services that can fail temporarily. A valid request can fail because of a connection reset, timeout, throttling, or temporary provider outage.

Recognize a retry problem when you see:
- connection resets or timeouts
- HTTP 429 responses
- transient HTTP 5xx responses
- intermittent provider failures
- requests that succeed when manually repeated

The production problem is not simply retrying failed requests. You must decide which failures are retryable, when to retry, how long to wait, how many times to retry, whether the operation is safe to repeat, and when to stop.

## 2. Concept and Reasoning

A retry is another attempt at an operation after a previous attempt failed.

```text
REQUEST
   ↓
SUCCESS ─────────────→ CONTINUE
   ↓
TRANSIENT FAILURE
   ↓
WAIT
   ↓
RETRY
   ↓
SUCCESS / FAILURE
```

### Retryable vs non-retryable

| Failure | Typical treatment |
|---|---|
| Connection reset | Retry |
| Temporary timeout | Usually retry |
| HTTP 429 | Retry according to rate-limit policy |
| HTTP 500 | Usually retry |
| HTTP 502 | Usually retry |
| HTTP 503 | Usually retry |
| HTTP 504 | Usually retry |
| HTTP 400 | Usually do not retry |
| HTTP 401 | Refresh credentials or fail |
| HTTP 403 | Usually fail unless provider documents temporary behavior |
| Invalid request | Do not blindly retry |
| Schema validation failure | Do not retry indefinitely |

These are defaults. The provider contract determines the final classification.

## 3. Retry Is a State Machine

Do not implement retries as an infinite loop.

```text
ATTEMPT
   ↓
FAIL
   ↓
CLASSIFY ERROR
   ↓
RETRYABLE?
  /      \
NO        YES
↓          ↓
FAIL      WAIT
           ↓
         RETRY
           ↓
      RETRY LIMIT
           ↓
          FAIL
```

A production retry policy has explicit bounds.

## 4. Implementation

### Step 1 — Define the retry policy

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class RetryPolicy:
    max_attempts: int = 5
    base_delay_seconds: float = 1.0
    max_delay_seconds: float = 60.0
    max_total_wait_seconds: float = 300.0
```

Keep retry policy in configuration rather than scattering constants through the extractor.

### Step 2 — Classify HTTP failures

```python
RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 504}

def is_retryable_status(status_code: int) -> bool:
    return status_code in RETRYABLE_STATUS_CODES
```

Network exceptions also need classification:

```python
import requests

RETRYABLE_EXCEPTIONS = (
    requests.Timeout,
    requests.ConnectionError,
)
```

Do not put every exception into the retryable category. Programming errors, malformed data, invalid configuration, and permanent validation failures should normally fail fast.

### Step 3 — Exponential backoff

```python
def exponential_delay(attempt: int, base: float, maximum: float) -> float:
    return min(maximum, base * (2 ** (attempt - 1)))
```

Example:

```text
Attempt 1 → 1s
Attempt 2 → 2s
Attempt 3 → 4s
Attempt 4 → 8s
Attempt 5 → 16s
```

### Step 4 — Add jitter

Without jitter, many workers can fail together and retry together:

```text
Worker A → wait 2s → retry
Worker B → wait 2s → retry
Worker C → wait 2s → retry
                         ↓
                    traffic spike
```

Use randomized delay:

```python
import random

def jittered_delay(attempt: int, base: float, maximum: float) -> float:
    delay = min(maximum, base * (2 ** (attempt - 1)))
    return random.uniform(0, delay)
```

### Step 5 — Respect provider retry instructions

If the provider supplies `Retry-After`, use it when its documented semantics require a wait.

```python
def retry_after_seconds(response) -> float | None:
    value = response.headers.get("Retry-After")
    if value is None:
        return None
    try:
        return max(0.0, float(value))
    except ValueError:
        return None
```

Some providers encode an HTTP date instead of seconds. Parse according to the provider contract.

### Step 6 — Build a bounded retry loop

```python
import time

def request_with_retry(session, url, policy):
    total_wait = 0.0

    for attempt in range(1, policy.max_attempts + 1):
        try:
            response = session.get(url, timeout=30)
        except RETRYABLE_EXCEPTIONS:
            if attempt == policy.max_attempts:
                raise
            delay = jittered_delay(
                attempt,
                policy.base_delay_seconds,
                policy.max_delay_seconds,
            )
            if total_wait + delay > policy.max_total_wait_seconds:
                raise RuntimeError("retry wait budget exhausted")
            time.sleep(delay)
            total_wait += delay
            continue

        if not is_retryable_status(response.status_code):
            response.raise_for_status()
            return response

        if attempt == policy.max_attempts:
            response.raise_for_status()

        delay = retry_after_seconds(response)
        if delay is None:
            delay = jittered_delay(
                attempt,
                policy.base_delay_seconds,
                policy.max_delay_seconds,
            )

        if total_wait + delay > policy.max_total_wait_seconds:
            raise RuntimeError("retry wait budget exhausted")

        time.sleep(delay)
        total_wait += delay

    raise RuntimeError("unreachable")
```

This separates error classification, delay calculation, provider instructions, retry count, total wait, and final failure.

## 5. Retry the Operation, Not the Whole Pipeline

A common mistake is restarting an entire extraction after one request fails.

Bad:

```text
1,000 API pages
       ↓
page 900 fails
       ↓
restart all pages
```

Better:

```text
page 900 fails
       ↓
retry page 900
       ↓
success
       ↓
continue page 901
```

Retries should normally happen at the smallest safe operation boundary. For paginated extraction, the request/page is usually the retry boundary.

## 6. Retry Safety and Idempotency

Retrying a request means the server may receive the operation more than once.

Reads such as `GET` are generally safe to repeat. Writes such as `POST` may create duplicate side effects.

When supported, use an idempotency key:

```python
headers = {
    "Idempotency-Key": operation_id,
}
```

The key should represent the logical operation, not each network attempt.

```text
LOGICAL OPERATION
       │
       ├── attempt 1
       ├── attempt 2
       └── attempt 3
             ↓
       SAME IDEMPOTENCY KEY
```

Do not assume every API supports idempotency keys.

## 7. Retry and Rate Limiting

Retries and rate limiting must work together.

```text
REQUEST
   ↓
429
   ↓
RATE-LIMIT POLICY
   ↓
WAIT
   ↓
RETRY POLICY
   ↓
RETRY
```

Never retry a 429 immediately in a tight loop. The rate-limiting mechanism controls request pressure; the retry mechanism controls failure recovery.

## 8. Retry and Timeouts

A retry policy without a request timeout can hang indefinitely on one attempt.

```text
ATTEMPT TIMEOUT
       +
RETRY LIMIT
       +
TOTAL WAIT LIMIT
       =
BOUNDED REQUEST BEHAVIOR
```

Example:

```python
response = session.get(
    url,
    timeout=(5, 30),
)
```

The correct timeout depends on the provider and operation.

## 9. Retry and Pagination

Each page should have its own request/retry lifecycle.

```text
PAGE 1 → SUCCESS
          ↓
PAGE 2 → TIMEOUT
          ↓
       RETRY PAGE 2
          ↓
        SUCCESS
          ↓
PAGE 3
```

Do not advance pagination state because a request was attempted. Advance it only after the page is successfully validated and durably persisted.

## 10. Retry and Checkpointing

Use this sequence:

```text
REQUEST
  ↓
RETRY IF TRANSIENT
  ↓
VALIDATE
  ↓
PERSIST
  ↓
VERIFY
  ↓
CHECKPOINT
```

A failed request must not move the extraction checkpoint.

## 11. Retry Budgets

Retries consume time, request quota, and compute.

Useful budgets include:
- maximum attempts per request
- maximum retry delay
- maximum total retry wait
- maximum pipeline runtime
- maximum retry volume

Example:

```python
MAX_ATTEMPTS = 5
MAX_TOTAL_WAIT = 300
```

Once a budget is exhausted, classify the operation as failed or deferred instead of retrying forever.

## 12. Retry Classification

Preserve why a retry occurred.

```text
NETWORK_TIMEOUT
CONNECTION_RESET
RATE_LIMITED
SERVER_ERROR
PROVIDER_UNAVAILABLE
AUTH_REFRESH_REQUIRED
UNKNOWN_TRANSIENT
```

Do not collapse all failures into `RETRY_ERROR`. Classification makes metrics and incident diagnosis useful.

## 13. Testing

### Unit tests

Test:
- retryable status is retried
- non-retryable status fails immediately
- connection timeout is retried
- connection error is retried
- `Retry-After` is honored
- exponential delay is bounded
- jitter stays within expected bounds
- maximum attempts stop retries
- total wait budget stops retries
- successful retry returns the response
- final failure preserves error context

Example:

```python
def test_server_error_is_retryable():
    assert is_retryable_status(503)

def test_bad_request_is_not_retryable():
    assert not is_retryable_status(400)

def test_delay_is_capped():
    assert exponential_delay(20, 1, 60) == 60
```

### Integration tests

Use a fake API that produces controlled sequences:

```text
503 → 503 → 200
429 → 200
timeout → 200
400
401 → credential refresh → 200
500 → 500 → 500 → 500 → 500 → failure
```

Verify the request sequence, delay policy, and failure classification.

## 14. Observability

| Metric | Purpose |
|---|---|
| Requests attempted | Measures request volume |
| Retries | Measures transient failure pressure |
| Retry rate | Shows instability relative to traffic |
| Retry delay seconds | Measures recovery cost |
| Final request failures | Measures unrecovered failures |
| Failures by class | Identifies failure causes |
| 429 retries | Measures throttling |
| 5xx retries | Measures provider instability |
| Network retries | Measures transport instability |
| Success after retry | Measures retry effectiveness |

Useful structured log fields:

```text
run_id
request_id
endpoint
attempt
max_attempts
error_class
status_code
delay_seconds
total_wait_seconds
final_attempt
```

Never log authorization headers, access tokens, or sensitive request bodies.

## 15. Intentional Failure

### Failure drill A — Temporary 503

Configure a fake API to return:

```text
503
503
200
```

Expected:
- two retries occur
- delays are applied
- third attempt succeeds
- extraction continues

### Failure drill B — Permanent 400

Return `400` for every request.

Expected:
- no retry storm
- request fails immediately
- failure classification is recorded

### Failure drill C — Repeated 429

Return repeated `429` responses.

Expected:
- provider retry instructions are honored
- retry count is bounded
- total wait is bounded
- request eventually fails or is deferred

### Failure drill D — Crash after successful retry

Make a request succeed on retry, then simulate a process crash before persistence.

Expected:
- the request can be repeated after restart
- destination idempotency prevents corruption
- checkpoint does not advance prematurely

### Failure drill E — Retry storm

Temporarily remove backoff and run multiple workers. Observe the request spike. Restore bounded backoff and jitter.

## 16. Recovery

When a request repeatedly fails:

1. Inspect the error class.
2. Determine whether the failure is actually transient.
3. Check attempt count.
4. Check total retry wait.
5. Check rate-limit state.
6. Check provider health if relevant.
7. Check credentials if authentication-related.
8. Check whether the request is safe to repeat.
9. Resume from the last durable pipeline state.
10. Reconcile destination state if an ambiguous outcome is possible.

If the retry budget is exhausted, stop retrying. Investigate or defer the work rather than creating an infinite loop.

## 17. Production Tools You Should Know

### Requests
Python HTTP client commonly used for explicit retry and response-handling logic.

### HTTPX
Python HTTP client useful when retry behavior must work with synchronous or asynchronous extraction.

### urllib3 Retry
Reusable retry-policy support for HTTP clients built around urllib3, including backoff and status-based retry configuration.

These tools provide mechanisms. The engineer still owns error classification, retry safety, and operational limits.

## 18. Production Runbook

### Before running
- Define retryable failures.
- Define maximum attempts.
- Define maximum total wait.
- Define request timeout.
- Confirm rate-limit behavior.
- Confirm idempotency behavior.
- Confirm checkpoint ordering.

### If retries increase
1. Identify the failure class.
2. Check endpoint-specific error rates.
3. Check provider throttling.
4. Check network health.
5. Check provider availability.
6. Verify workers are not producing a retry storm.
7. Reduce concurrency if required.

### If retries are exhausted
- Preserve the final error.
- Preserve retry history.
- Mark the operation failed or deferred.
- Keep the last safe checkpoint.
- Reconcile destination state if the outcome was ambiguous.
- Retry later only according to an explicit recovery policy.

### What not to do
- Do not retry every exception.
- Do not retry forever.
- Do not use zero-delay retries.
- Do not ignore `Retry-After`.
- Do not retry non-idempotent writes blindly.
- Do not restart the entire pipeline for one failed request.
- Do not advance checkpoints after an unsuccessful request.

## 19. Common Mistakes

1. Treating all HTTP failures as retryable.
2. Retrying authentication failures forever.
3. Using fixed delays for every worker.
4. Omitting jitter.
5. Having no maximum retry count.
6. Having no maximum total wait.
7. Retrying non-idempotent operations without protection.
8. Ignoring rate limits.
9. Retrying at the whole-pipeline level instead of the request level.
10. Losing the original error context.

## 20. Definition of Done

- [ ] Retryable and non-retryable failures are defined.
- [ ] Retry policy is configuration-driven.
- [ ] Attempt count is bounded.
- [ ] Total retry wait is bounded.
- [ ] Request timeouts are configured.
- [ ] Exponential backoff is implemented.
- [ ] Jitter is implemented where appropriate.
- [ ] `Retry-After` is respected.
- [ ] Retry classification is observable.
- [ ] Request-level retry is used instead of whole-pipeline restart.
- [ ] Idempotency implications are understood.
- [ ] Pagination checkpoints advance only after successful persistence.
- [ ] Retry behavior is tested with a fake API.
- [ ] Intentional failure drills pass.
- [ ] Recovery behavior is documented.

## 21. What You Learned

After this recipe, you should be able to independently:
- classify API failures into retryable and non-retryable categories
- design bounded retry policies
- implement exponential backoff
- add jitter
- honor provider retry instructions
- combine retries with rate limiting and timeouts
- reason about idempotency
- retry at the correct operation boundary
- preserve checkpoint safety
- observe retry behavior
- diagnose retry storms
- recover after retry exhaustion

### Core Mental Model

```text
REQUEST
   ↓
CLASSIFY FAILURE
   ↓
TRANSIENT?
  /      \
NO        YES
↓          ↓
FAIL      WAIT
           ↓
         RETRY
           ↓
      BOUNDED?
      /      \
    YES       NO
     ↓         ↓
 CONTINUE     FAIL / DEFER
```

> A reliable retry policy is not a loop around an HTTP request. It is a bounded recovery mechanism that understands failure type, timing, idempotency, rate limits, and pipeline progress.