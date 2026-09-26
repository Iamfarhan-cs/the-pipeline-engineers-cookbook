# Recipe 25 — Rate Limiting

A pipeline can be correct, fast, and still damage its dependencies by sending requests too quickly.

This recipe teaches how to control request rate, distinguish rate limiting from concurrency limiting, implement a simple limiter, test it, intentionally trigger limits, recover safely, and understand production rate-limiting systems.

## 1. Problem Recognition

A dependency may permit 100 requests/second while a pipeline attempts 1,000 requests/second. The dependency may return HTTP 429, increase latency, reject requests, throttle the client, or temporarily block it.

Recognize the problem through:
- HTTP 429 responses
- database connection exhaustion
- service-specific throttling errors
- latency increases as concurrency rises
- repeated timeouts
- provider quota warnings
- request bursts
- retry storms

The core question is:

> How fast can this pipeline safely send work to this dependency?

## 2. Concept and Reasoning

### Rate limiting vs concurrency limiting

**Rate limiting** controls how many operations happen over time.

```text
100 requests / second
```

**Concurrency limiting** controls how many operations are active simultaneously.

```text
maximum 10 requests in flight
```

A production pipeline may need both.

### Why rate limiting matters

Without a limit:

```text
workers
  ↓
burst
  ↓
dependency
  ↓
overload
  ↓
errors
  ↓
retries
  ↓
more load
  ↓
retry storm
```

A rate limiter breaks this feedback loop.

## 3. Common Rate-Limiting Models

### Fixed window

Allow a fixed number of requests during a fixed interval.

Simple, but requests can cluster at window boundaries.

### Sliding window

Measure requests over the most recent interval. This produces smoother behavior but requires more state.

### Token bucket

Tokens are added at a fixed rate and each request consumes a token.

```text
token generation
      ↓
[● ● ● ● ●]
      ↓
request consumes token
      ↓
[● ● ● ●]
```

A bucket can also permit a controlled burst.

### Leaky bucket

Requests enter a queue and leave at a controlled rate.

```text
requests
   ↓
[queue]
   ↓
controlled output
   ↓
dependency
```

The correct model depends on the dependency contract and workload.

## 4. Implementation

We will implement a simple token-bucket limiter.

### 4.1 Basic limiter

```python
import time


class RateLimiter:
    def __init__(self, rate_per_second: float, capacity: int):
        if rate_per_second <= 0:
            raise ValueError("rate_per_second must be positive")

        if capacity <= 0:
            raise ValueError("capacity must be positive")

        self.rate = rate_per_second
        self.capacity = capacity
        self.tokens = float(capacity)
        self.last_refill = time.monotonic()

    def acquire(self):
        while True:
            now = time.monotonic()
            elapsed = now - self.last_refill

            self.tokens = min(
                self.capacity,
                self.tokens + elapsed * self.rate,
            )

            self.last_refill = now

            if self.tokens >= 1:
                self.tokens -= 1
                return

            wait_time = (1 - self.tokens) / self.rate
            time.sleep(wait_time)
```

### 4.2 Use it before the dependency call

```python
limiter = RateLimiter(
    rate_per_second=100,
    capacity=100,
)

for record in records:
    limiter.acquire()
    send_to_api(record)
```

The ordering is:

```text
get permission
     ↓
send request
```

not:

```text
send request
     ↓
sleep
```

## 5. Burst Capacity

Suppose:

```text
rate = 10 requests/second
capacity = 20
```

The system can initially consume up to 20 available tokens. After the burst, tokens refill at approximately 10 tokens/second.

This allows controlled bursts without unlimited traffic.

Do not choose a large capacity without understanding the dependency's burst tolerance.

## 6. Configuration

Do not hard-code limits inside pipeline logic.

Use configuration:

```text
API_RATE_LIMIT=100
API_BURST_CAPACITY=100
```

Then:

```python
import os

rate = float(os.environ["API_RATE_LIMIT"])
capacity = int(os.environ["API_BURST_CAPACITY"])

limiter = RateLimiter(rate, capacity)
```

Production configuration should be based on the dependency's documented or agreed limits.

## 7. Testing

Rate limiting is time-sensitive, so tests should avoid unnecessary real wall-clock delays.

Test:

1. invalid rate and capacity
2. initial burst capacity
3. token consumption
4. token refill
5. capacity ceiling
6. waiting when the bucket is empty
7. dependency responses such as HTTP 429

For integration tests, use a test server that deliberately returns 429 and verify correct classification.

## 8. Observability

Track:

```text
requests_allowed_total
requests_delayed_total
rate_limit_wait_seconds
rate_limit_rejections_total
dependency_429_total
dependency_latency_seconds
```

Useful event:

```json
{
  "event": "rate_limit_wait",
  "dependency": "payments-api",
  "wait_ms": 42,
  "configured_rate": 100
}
```

Monitor:
- request rate
- wait time
- 429 rate
- dependency latency
- queue depth
- retry count

If the limiter constantly delays requests, investigate whether the configured rate is appropriate before simply increasing it.

## 9. Intentional Failure

Configure a test dependency to allow 10 requests/second.

Temporarily disable the limiter and send approximately 100 requests/second.

Observe:
- 429 responses
- latency
- failure rate
- retries

Then enable the limiter and verify controlled traffic.

### Retry-storm drill

Deliberately create:

```text
request burst
   ↓
429
   ↓
immediate retry
   ↓
429
   ↓
immediate retry
   ↓
more load
```

Then add rate limiting and backoff and compare the behavior.

## 10. Recovery

A 429 response is usually a signal to slow down.

Safe pattern:

```text
request
  ↓
429
  ↓
read Retry-After if available
  ↓
wait
  ↓
retry
```

For transient failures, use bounded exponential backoff with jitter.

Conceptually:

```text
attempt 1 → ~1s
attempt 2 → ~2s
attempt 3 → ~4s
attempt 4 → ~8s
```

Use an upper bound and retry limit.

Do not allow retries forever.

## 11. Rate Limiting and Retries

They solve different problems.

**Rate limiter:**

> How quickly should normal traffic leave this worker?

**Retry backoff:**

> How long should this failed operation wait before trying again?

A production system may need both:

```text
             ┌── rate limiter ──→ normal request
worker ──────┤
             └── retry backoff ─→ retry request
```

Every retry should still pass through the appropriate rate/concurrency controls.

## 12. Distributed Rate Limiting

A local limiter controls one process.

Suppose:

```text
10 workers
×
100 requests/sec
=
potentially 1,000 requests/sec
```

If the dependency allows only 100 requests/sec, ten independent local limiters are insufficient.

You may need shared coordination:

```text
Worker A ─┐
Worker B ─┤
Worker C ─┼→ shared rate limit → dependency
Worker D ─┤
Worker E ─┘
```

Possible implementations include shared state or infrastructure-level gateways.

## 13. Rate Limiting and Concurrency

A pipeline may need both:

```text
100 requests/sec
+
10 concurrent requests
```

For example:

```text
10 requests/sec
×
10-second latency
≈
100 requests in flight
```

Therefore:

> Rate limiting does not automatically control concurrency.

Use the mechanism that addresses the actual bottleneck.

## 14. Testing Recovery

Test:

### Case 1 — Dependency recovers

```text
429
429
200
```

Expected:

```text
wait → retry → success
```

### Case 2 — Dependency remains unavailable

Verify:
- bounded retries
- eventual failure/quarantine
- no infinite loop

### Case 3 — Multiple workers

When a global limit is required, verify that aggregate traffic stays within that limit.

## 15. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Redis** | Can provide shared counters, token buckets, and distributed rate-limiting state. |
| **Envoy** | Provides traffic management and rate-limiting capabilities at service boundaries. |
| **Kong** | API gateway with request-rate limiting and traffic-control capabilities. |

> These tools provide production mechanisms for enforcing limits. First understand rate, burst, concurrency, retry, and shared-state problems.

## 16. Production Runbook

### Dependency returns 429

Check:
1. configured rate
2. actual dependency limit
3. burst capacity
4. worker count
5. retry rate
6. whether a global limit exists
7. Retry-After information

### Pipeline becomes slow

Determine whether rate limiting is intentionally throttling the workload.

Then determine whether the limit is:
- correct
- unnecessarily conservative
- imposed by the dependency
- imposed by another downstream resource

### Multiple workers overwhelm a dependency

Calculate:

```text
worker_count × per_worker_rate
```

Compare it with the dependency limit.

### What not to do

Do not:
- ignore 429 responses
- immediately retry 429 responses
- use unlimited retries
- assume local limits are global limits
- increase limits without checking dependency capacity
- confuse rate limiting with concurrency limiting

## 17. Common Mistakes

### Mistake 1 — No rate limit

Workers can overwhelm dependencies.

### Mistake 2 — Fixed sleep everywhere

A constant sleep does not properly model burst capacity or shared limits.

### Mistake 3 — Unlimited retries

A transient dependency problem becomes a retry storm.

### Mistake 4 — Ignoring Retry-After

The server may explicitly tell the client when to try again.

### Mistake 5 — Per-worker limits treated as global

Ten workers can multiply actual request rate.

### Mistake 6 — No jitter

Many workers can wake and retry simultaneously.

### Mistake 7 — Monitoring only application throughput

A pipeline can be fast while damaging its dependency.

## 18. Definition of Done

You are done when you can:
- recognize rate-limit problems
- distinguish rate limiting from concurrency limiting
- explain fixed-window, sliding-window, token-bucket, and leaky-bucket concepts
- implement a basic token bucket from scratch
- configure rate and burst capacity
- test token consumption and refill
- observe request rate and limiter waits
- intentionally trigger dependency throttling
- handle HTTP 429 safely
- implement bounded exponential backoff with jitter
- avoid retry storms
- explain local vs distributed rate limiting
- calculate aggregate traffic across workers
- explain why rate limiting and concurrency control may both be required
- explain how Redis, Envoy, and Kong relate to production rate limiting

## 19. What You Learned

The central principle is:

> **A pipeline must control the rate at which it consumes shared resources and dependencies.**

The basic pattern is:

```text
WORK
  ↓
RATE LIMIT
  ↓
CONCURRENCY CONTROL
  ↓
DEPENDENCY
  ↓
SUCCESS
```

When throttled:

```text
429 / THROTTLE
      ↓
READ SERVER GUIDANCE
      ↓
BACKOFF + JITTER
      ↓
RATE LIMIT
      ↓
RETRY
```

The deeper lesson is that pipeline performance is not just about how fast your workers can run.

A production pipeline must operate at a rate the entire system can safely sustain.
