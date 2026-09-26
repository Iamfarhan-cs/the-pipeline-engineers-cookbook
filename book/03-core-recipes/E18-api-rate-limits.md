# E18 — API Rate Limits

## 1. Problem Recognition

An API can accept valid requests and still reject a pipeline because the client is sending requests faster than the provider permits.

Recognize this problem when you see:

- HTTP `429 Too Many Requests` responses
- rate-limit headers such as `Retry-After` or provider-specific quota headers
- sudden request failures after sustained throughput
- quota-exceeded messages
- documentation describing requests per second, minute, hour, or day
- different limits for different endpoints, tenants, credentials, or plans

Rate limiting is not simply an HTTP error-handling problem. It is a **throughput-control problem**.

### Typical failure modes

| Failure | Consequence |
|---|---|
| Requests exceed provider limit | 429 responses |
| Multiple workers share one quota | Aggregate traffic exceeds the limit |
| Ignoring `Retry-After` | Repeated throttling |
| Retry storm | Traffic increases while the API is already overloaded |
| Per-tenant quota is ignored | One tenant can consume another tenant's capacity |
| Daily quota is exhausted | Pipeline cannot continue until quota resets |
| Concurrent jobs are unaware of each other | Combined request rate exceeds the quota |
| Rate limiter is too conservative | Pipeline wastes available capacity |

## 2. Concept and Reasoning

A rate limit defines how much request traffic a provider allows over a period.

Examples:

```text
100 requests / minute
10 requests / second
10,000 requests / day
5 concurrent requests
1,000 records / minute
```

These are different constraints. A pipeline must understand which dimension is actually limited.

### Rate limit vs quota

- **Rate limit** usually constrains request frequency over a time window.
- **Quota** usually constrains total consumption over a longer period.

A provider can enforce both:

```text
10 requests/second
AND
100,000 requests/day
```

Passing the per-second limit does not guarantee that the daily quota remains available.

### The core mechanism

```text
REQUEST
   ↓
CHECK LOCAL RATE BUDGET
   ↓
CALL API
   ↓
SUCCESS ───────────────→ CONTINUE
   ↓
429 / THROTTLE
   ↓
READ PROVIDER SIGNAL
   ↓
WAIT
   ↓
RETRY SAFELY
```

The goal is not to eliminate throttling at all costs. The goal is to **control traffic so the pipeline makes predictable progress without creating retry storms**.

## 3. Implementation

### Step 1 — Discover the provider's limits

Document:

- requests per second/minute/hour/day
- whether limits are per IP, API key, user, tenant, endpoint, or account
- burst allowance
- concurrency limits
- record-based limits
- response headers
- `Retry-After` semantics
- quota-reset information
- whether failed requests consume quota
- whether pagination requests share the same quota

Never infer a limit from a small successful test. Use the provider's documented contract and observed response metadata.

### Step 2 — Separate provider limits from local policy

Suppose the provider permits 100 requests/minute.

Your pipeline might deliberately operate at 80 requests/minute:

```python
PROVIDER_LIMIT = 100
SAFETY_RATE = 0.80
LOCAL_LIMIT = int(PROVIDER_LIMIT * SAFETY_RATE)
```

The local limit is an operational decision. It should not be confused with the provider's actual contract.

### Step 3 — Build a simple token-bucket limiter

A token bucket allows controlled bursts while enforcing an average rate.

```python
import time


class RateLimiter:
    def __init__(self, rate_per_second: float, capacity: int):
        self.rate = rate_per_second
        self.capacity = capacity
        self.tokens = float(capacity)
        self.updated_at = time.monotonic()

    def acquire(self, tokens: int = 1) -> None:
        if tokens <= 0:
            raise ValueError("tokens must be positive")

        while True:
            now = time.monotonic()
            elapsed = now - self.updated_at
            self.updated_at = now

            self.tokens = min(
                self.capacity,
                self.tokens + elapsed * self.rate,
            )

            if self.tokens >= tokens:
                self.tokens -= tokens
                return

            missing = tokens - self.tokens
            sleep_for = missing / self.rate
            time.sleep(sleep_for)
```

Example:

```python
limiter = RateLimiter(
    rate_per_second=5,
    capacity=5,
)

for request in requests_to_make:
    limiter.acquire()
    response = call_api(request)
```

This controls one process. It does **not** automatically coordinate multiple pipeline workers.

### Step 4 — Handle `Retry-After`

When the provider explicitly tells you when to retry, use that information.

```python
def retry_after_seconds(response) -> float | None:
    value = response.headers.get("Retry-After")

    if value is None:
        return None

    try:
        seconds = float(value)
    except ValueError:
        return None

    return max(0.0, seconds)
```

The actual header may contain either a delay or an HTTP date depending on the provider contract. Parse according to the provider's documented semantics.

### Step 5 — Use bounded backoff when no retry delay is provided

```python
import random


def exponential_backoff(attempt: int, base: float = 1.0, maximum: float = 60.0) -> float:
    delay = min(maximum, base * (2 ** attempt))
    return random.uniform(0, delay)
```

Jitter prevents many workers from waking at the same instant.

Example:

```python
for attempt in range(5):
    response = call_api()

    if response.status_code != 429:
        break

    delay = retry_after_seconds(response)
    if delay is None:
        delay = exponential_backoff(attempt)

    time.sleep(delay)
```

Rate limiting and retries are related but different:

- **Rate limiter:** controls requests before they are sent.
- **Retry policy:** controls what happens after a failed request.

Both are needed.

### Step 6 — Classify throttling responses

Do not treat every 429 as the same operational event.

Useful classifications include:

```text
429 + short Retry-After
    → temporary throttling

429 + quota exhausted
    → longer wait / quota reset

401/403 + quota message
    → provider-specific authorization/quota condition

5xx
    → server failure, not necessarily rate limiting
```

Provider behavior differs. Classification must follow the API contract.

### Step 7 — Coordinate multiple workers

A local limiter per worker can still violate the provider limit:

```text
Worker A → 5 req/s
Worker B → 5 req/s
Worker C → 5 req/s
                 ↓
           15 req/s total
                 ↓
Provider limit = 10 req/s
```

If the limit applies globally, the rate budget must also be coordinated globally.

Possible designs:

- one shared worker
- centralized rate-limit service
- distributed token bucket
- queue-based dispatch
- database-backed coordination for low-volume workloads

Do not introduce distributed coordination unless the workload actually requires it.

### Step 8 — Tenant-aware rate limiting

For multi-tenant APIs, the limit may apply independently to each tenant.

Model the rate budget explicitly:

```python
limiters = {}


def limiter_for(tenant_id: str) -> RateLimiter:
    if tenant_id not in limiters:
        limiters[tenant_id] = RateLimiter(
            rate_per_second=2,
            capacity=2,
        )
    return limiters[tenant_id]
```

In production, determine whether the actual quota is tenant-specific before using this design.

### Step 9 — Do not retry forever

Every retry policy needs a stopping rule.

```python
MAX_RETRIES = 5
MAX_TOTAL_WAIT_SECONDS = 300
```

A pipeline should eventually classify the run as failed or deferred instead of waiting indefinitely.

## 4. Rate-Limit Signals

Providers may expose headers such as:

```text
Retry-After: 10
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 12
X-RateLimit-Reset: 1730000000
```

Names and semantics vary by provider.

Useful information to capture:

- configured limit
- remaining budget
- reset time
- retry-after duration
- endpoint
- tenant/account
- response status

Do not assume headers are always present or trustworthy unless the provider documents them.

## 5. Pagination Interaction

Pagination can multiply API traffic dramatically.

Example:

```text
1,000,000 records
1,000 records/page
        ↓
1,000 API requests
```

If each request is rate-limited, pagination throughput becomes rate-limit throughput.

Therefore page size and rate limiting should be considered together.

```text
LARGER PAGE
   ↓
FEWER REQUESTS
   ↓
LESS RATE-LIMIT CONSUMPTION
```

But larger pages can increase latency, memory usage, timeout risk, and provider-side processing.

Optimize the complete extraction, not one variable in isolation.

## 6. Rate Limits and Retries

A dangerous implementation is:

```python
while True:
    response = call_api()
    if response.status_code == 429:
        continue
```

This creates a retry storm.

Better:

```text
429
 ↓
READ RETRY-AFTER
 ↓
WAIT
 ↓
RETRY WITH BOUND
 ↓
STOP AFTER POLICY LIMIT
```

Never increase request concurrency in response to throttling.

## 7. Testing

### Unit tests

Test:

- requests stay within local rate budget
- burst capacity behaves as intended
- `Retry-After` is respected
- missing `Retry-After` uses bounded backoff
- jitter produces varied delays
- retry count is bounded
- total wait is bounded
- invalid rate-limit headers are handled safely
- multiple tenants maintain separate budgets when appropriate
- repeated 429 responses eventually fail or defer

Example:

```python
def test_retry_after_is_non_negative():
    class Response:
        headers = {"Retry-After": "-10"}

    assert retry_after_seconds(Response()) == 0.0
```

### Integration tests

Build a fake API that returns:

1. normal responses
2. one 429 followed by success
3. several 429 responses
4. `Retry-After`
5. missing retry information
6. long quota exhaustion
7. multiple concurrent workers

Verify that request volume remains within the intended policy.

## 8. Observability

Rate-limit behavior must be measurable.

| Metric | Purpose |
|---|---|
| Requests sent | Measures traffic |
| Requests throttled | Measures provider pressure |
| 429 count | Detects explicit throttling |
| Retry count | Measures recovery activity |
| Retry wait seconds | Measures throttling cost |
| Current request rate | Compares actual traffic with policy |
| Remaining quota | Shows available budget when provided |
| Quota reset time | Predicts availability |
| Request latency | Shows API performance |
| Requests by endpoint | Finds noisy endpoints |
| Requests by tenant | Finds quota consumers |

Useful structured log fields:

```text
run_id
source
endpoint
tenant_id
status_code
retry_attempt
retry_after_seconds
rate_limit_remaining
rate_limit_reset
request_latency_ms
```

Do not log API credentials or sensitive authorization headers.

### Important operational signal

A rising 429 rate is not merely an API error metric. It can indicate that the pipeline's configured throughput is incompatible with the provider's current capacity or quota.

## 9. Intentional Failure

### Failure drill A — Remove the limiter

Run the extraction with no local rate control against a test API with a strict limit.

Expected:

- 429 responses increase
- extraction throughput becomes unstable
- logs show throttling

Restore the limiter and compare behavior.

### Failure drill B — Ignore `Retry-After`

Force the fake API to return `Retry-After: 10`.

Temporarily ignore the value and observe repeated 429s. Restore correct handling.

### Failure drill C — Create a retry storm

Use several workers with immediate retries.

Expected:

- request volume spikes
- throttling increases
- recovery becomes slower

Replace immediate retries with bounded backoff and jitter.

### Failure drill D — Exhaust the daily quota

Simulate a provider that reports quota exhaustion rather than temporary throttling.

Expected:

- pipeline does not retry indefinitely
- run is classified as deferred or failed according to policy
- quota reset information is recorded

## 10. Recovery

When throttling occurs:

1. Inspect the response status and provider-specific headers.
2. Determine whether the condition is temporary throttling or quota exhaustion.
3. Check the current request rate.
4. Check whether other workers are consuming the same budget.
5. Honor `Retry-After` when documented.
6. Apply bounded backoff with jitter when no provider delay is available.
7. Reduce concurrency if necessary.
8. Resume only within the retry policy.
9. Record the final throttling state.

If the daily quota is exhausted, waiting a few seconds is not a valid recovery strategy. Resume after the provider's documented reset or use an approved alternative extraction window.

## 11. Production Tools You Should Know

### Requests
Simple Python HTTP client suitable for implementing explicit rate limiting and response handling.

### HTTPX
Python HTTP client useful for synchronous or asynchronous API extraction where concurrency and connection management matter.

### Airbyte
Production data integration platform whose connectors provide practical examples of API throttling, retries, and source-specific request pacing.

Tools implement mechanisms, but the engineer still needs to understand the provider's quota model.

## 12. Production Runbook

### Before running

- Document provider limits.
- Confirm whether limits are per key, tenant, endpoint, or account.
- Confirm burst and concurrency rules.
- Confirm `Retry-After` semantics.
- Configure local safety margins.
- Configure bounded retries.
- Configure maximum total wait.

### During a run

- Monitor 429 rate.
- Monitor request rate.
- Monitor remaining quota when available.
- Monitor retry wait time.
- Watch endpoint and tenant distribution.
- Watch for quota exhaustion.

### If throttling increases

1. Check whether traffic increased.
2. Check whether multiple workers share the same quota.
3. Check recent provider limit changes.
4. Honor provider retry instructions.
5. Reduce concurrency if necessary.
6. Verify that retry logic is not amplifying traffic.

### What not to do

- Do not retry 429 immediately in a tight loop.
- Do not assume every 429 has the same cause.
- Do not treat a daily quota as a short-term rate limit.
- Do not create one independent limiter per worker when the provider quota is global.
- Do not hide throttling by merely increasing retries.
- Do not log authorization credentials while debugging rate limits.

## 13. Common Mistakes

1. Hard-coding a request rate without checking the provider contract.
2. Ignoring `Retry-After`.
3. Retrying immediately.
4. Retrying forever.
5. Using local rate limits without considering aggregate worker traffic.
6. Confusing rate limits with daily quotas.
7. Treating all endpoints as if they have identical limits.
8. Increasing concurrency when the API is already throttling.
9. Ignoring page size when optimizing a paginated extraction.
10. Failing to measure throttling cost.

## 14. Definition of Done

- [ ] Provider rate-limit contract is documented.
- [ ] Rate-limit scope is known.
- [ ] Local request pacing exists.
- [ ] Provider retry signals are handled.
- [ ] Backoff is bounded.
- [ ] Jitter is used where appropriate.
- [ ] Retry count is bounded.
- [ ] Total retry wait is bounded.
- [ ] Shared quota behavior is understood.
- [ ] Pagination traffic is included in throughput planning.
- [ ] Rate-limit metrics exist.
- [ ] Throttling failures have been intentionally tested.
- [ ] Recovery behavior is documented.
- [ ] Quota exhaustion has a defined outcome.

## 15. What You Learned

After this recipe, you should be able to independently:

- identify the rate-limit model of an unfamiliar API
- distinguish rate limits from quotas
- implement local request pacing
- handle `Retry-After` correctly
- implement bounded exponential backoff and jitter
- coordinate traffic across workers when required
- reason about tenant-specific limits
- connect pagination throughput to API quotas
- diagnose retry storms
- recover from temporary throttling and quota exhaustion
- measure the operational cost of API rate limiting

### Core Mental Model

```text
UNDERSTAND PROVIDER LIMIT
          ↓
CONTROL LOCAL REQUEST RATE
          ↓
CALL API
          ↓
SUCCESS ─────────────→ CONTINUE
          ↓
THROTTLED
          ↓
READ PROVIDER SIGNAL
          ↓
WAIT WITH BOUNDED BACKOFF
          ↓
RETRY SAFELY
          ↓
STOP / DEFER WHEN POLICY IS EXHAUSTED
```

> A reliable extractor does not try to defeat an API's rate limit. It turns the provider's capacity constraint into controlled pipeline behavior.