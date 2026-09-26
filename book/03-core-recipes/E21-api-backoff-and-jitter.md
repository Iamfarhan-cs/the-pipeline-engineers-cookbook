# E21 — API Backoff and Jitter

## 1. Problem Recognition

When an API request fails temporarily, immediately sending another request can make the original problem worse. If many workers fail together, they can all retry at the same time and create a traffic spike.

Recognize this problem when:

- multiple workers retry simultaneously
- 429 or 5xx responses increase after a failure
- request volume spikes immediately after an outage
- a provider recovers slowly because clients keep sending traffic
- retries happen at fixed intervals
- logs show synchronized retry timestamps

The production problem is not merely adding `sleep()`. It is controlling **when failed work is allowed to return to the system**.

## 2. Concept and Reasoning

### Backoff

Backoff increases the waiting time between attempts after repeated failures.

Example:

```text
Attempt 1 → fail → wait 1s
Attempt 2 → fail → wait 2s
Attempt 3 → fail → wait 4s
Attempt 4 → fail → wait 8s
```

### Jitter

Jitter adds controlled randomness to the delay.

Without jitter:

```text
Worker A ── fail ── wait 4s ── retry
Worker B ── fail ── wait 4s ── retry
Worker C ── fail ── wait 4s ── retry
                              ↓
                         traffic spike
```

With jitter:

```text
Worker A ── wait 3.1s
Worker B ── wait 4.7s
Worker C ── wait 3.8s
             ↓
       retries spread out
```

Backoff controls **how quickly retries become less frequent**. Jitter controls **when different workers retry relative to one another**.

## 3. Why Fixed Retry Delays Fail

Consider:

```python
for attempt in range(5):
    try:
        return call_api()
    except TemporaryError:
        time.sleep(5)
```

If 1,000 workers fail together, many of them wake after approximately five seconds.

```text
1,000 failures
      ↓
5 second sleep
      ↓
1,000 retries together
      ↓
provider pressure
      ↓
more failures
      ↓
more retries
```

This feedback loop is a retry storm.

## 4. Implementation

### Step 1 — Define the policy

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class BackoffPolicy:
    base_delay_seconds: float = 1.0
    maximum_delay_seconds: float = 60.0
    max_attempts: int = 5
    max_total_wait_seconds: float = 300.0
```

Keep policy separate from the request implementation.

### Step 2 — Exponential backoff

```python
def exponential_backoff(
    attempt: int,
    base: float,
    maximum: float,
) -> float:
    return min(
        maximum,
        base * (2 ** (attempt - 1)),
    )
```

Example:

```text
Attempt 1 → 1s
Attempt 2 → 2s
Attempt 3 → 4s
Attempt 4 → 8s
Attempt 5 → 16s
Attempt 6 → 32s
Attempt 7 → 60s (capped)
```

The cap prevents the delay from growing indefinitely.

### Step 3 — Full jitter

Full jitter selects a random delay between zero and the calculated backoff:

```python
import random

def full_jitter_delay(
    attempt: int,
    base: float,
    maximum: float,
) -> float:
    ceiling = exponential_backoff(attempt, base, maximum)
    return random.uniform(0, ceiling)
```

Full jitter is simple and effective for spreading concurrent retries.

### Step 4 — Equal jitter

Equal jitter keeps part of the backoff while randomizing the remainder:

```python
def equal_jitter_delay(
    attempt: int,
    base: float,
    maximum: float,
) -> float:
    ceiling = exponential_backoff(attempt, base, maximum)
    return ceiling / 2 + random.uniform(0, ceiling / 2)
```

This produces a delay between half and the full exponential ceiling.

### Step 5 — Decorrelated jitter

Decorrelated jitter uses the previous delay to avoid repeatedly selecting highly synchronized ranges:

```python
def decorrelated_jitter(
    previous_delay: float,
    base: float,
    maximum: float,
) -> float:
    upper = max(base, previous_delay * 3)
    return min(maximum, random.uniform(base, upper))
```

Different systems use different jitter algorithms. The important skill is understanding the trade-off rather than memorizing one formula.

## 5. Choosing a Jitter Strategy

| Strategy | Behavior |
|---|---|
| No jitter | Deterministic retries; synchronization risk |
| Full jitter | Random delay from zero to backoff ceiling |
| Equal jitter | Preserves part of the calculated delay |
| Decorrelated jitter | Uses previous delay to spread future delays |

For many API clients, full jitter is a practical default because it is simple and spreads retry traffic.

## 6. Provider Retry Instructions Come First

If the API provides `Retry-After`, that signal can be more authoritative than a locally generated backoff.

```text
429
 ↓
Retry-After: 15
 ↓
WAIT ACCORDING TO PROVIDER CONTRACT
 ↓
RETRY
```

Do not replace a documented provider wait of 15 seconds with a locally generated 1-second delay merely because the local algorithm says so.

Provider-specific quota and reset semantics must also be respected.

## 7. Bounded Backoff

Backoff needs multiple boundaries:

- maximum attempts
- maximum individual delay
- maximum total wait
- overall pipeline deadline

Example:

```python
def within_budget(
    total_wait: float,
    delay: float,
    maximum_total: float,
) -> bool:
    return total_wait + delay <= maximum_total
```

Never let backoff become an infinite waiting mechanism.

## 8. Deadline-Aware Backoff

Retries should respect the remaining time of the logical operation.

```python
import time

def remaining_budget(deadline: float) -> float:
    return max(0.0, deadline - time.monotonic())
```

Before sleeping:

```python
remaining = remaining_budget(deadline)
delay = min(delay, remaining)

if delay <= 0:
    raise TimeoutError("operation deadline exceeded")

time.sleep(delay)
```

This prevents the retry mechanism from sleeping beyond the operation's deadline.

## 9. Backoff + Retry + Timeout

These mechanisms form one bounded recovery system:

```text
REQUEST
   ↓
TIMEOUT / 429 / 5xx
   ↓
CLASSIFY
   ↓
RETRYABLE?
   ↓
CALCULATE BACKOFF
   ↓
ADD JITTER
   ↓
CHECK DEADLINE
   ↓
WAIT
   ↓
RETRY
```

Timeout limits an individual attempt. Retry limits the number of attempts. Backoff controls the waiting interval. Jitter spreads concurrent retries.

## 10. Backoff + Rate Limiting

Rate limiting and backoff solve different problems.

- **Rate limiting:** controls traffic before requests are sent.
- **Backoff:** controls traffic after failures.

Use both:

```text
LOCAL RATE LIMITER
       ↓
     REQUEST
       ↓
     FAILURE
       ↓
BACKOFF + JITTER
       ↓
     RETRY
```

A retry mechanism should not bypass the normal rate limiter.

## 11. Backoff + Pagination

Each failed page should back off independently while preserving pagination state.

```text
PAGE 25
  ↓
503
  ↓
BACKOFF
  ↓
RETRY PAGE 25
  ↓
SUCCESS
  ↓
CHECKPOINT
  ↓
PAGE 26
```

Do not advance to page 26 while page 25 is still unresolved.

## 12. Backoff + Concurrency

Concurrency can amplify retry traffic.

```text
100 workers
    ↓
provider failure
    ↓
100 workers calculate backoff
    ↓
jitter spreads retries
    ↓
provider receives controlled traffic
```

Jitter reduces synchronization but does not eliminate the need for an overall rate limit.

For large distributed systems, a shared retry/rate budget may be required.

## 13. Avoiding Randomness Problems in Tests

Production jitter should be random. Tests should remain deterministic.

Inject a random-number generator:

```python
def full_jitter_delay(attempt, base, maximum, rng):
    ceiling = exponential_backoff(attempt, base, maximum)
    return rng.uniform(0, ceiling)
```

Test with a seeded or fake generator:

```python
import random

rng = random.Random(42)
delay = full_jitter_delay(3, 1.0, 60.0, rng)
assert 0.0 <= delay <= 4.0
```

This lets you test the algorithm without depending on uncontrolled randomness.

## 14. Testing

### Unit tests

Test:
- exponential growth
- maximum-delay cap
- full jitter bounds
- equal jitter bounds
- decorrelated jitter bounds
- maximum total wait
- deadline handling
- provider-specified delay precedence
- deterministic tests with injected randomness

Example:

```python
def test_full_jitter_stays_within_ceiling():
    import random

    rng = random.Random(42)
    delay = full_jitter_delay(4, 1.0, 60.0)
    assert 0 <= delay <= 8
```

For production-quality tests, inject `rng` rather than relying on the global random generator.

### Distribution testing

Do not expect a single random sample to prove jitter quality.

Generate many samples and inspect whether delays are distributed across the intended range.

### Integration tests

Use a fake API that produces:

```text
503 → 503 → 200
429 + Retry-After → 200
timeout → 200
multiple simultaneous failures
```

Verify that retries are bounded and that provider instructions are honored.

## 15. Observability

Track backoff behavior explicitly.

| Metric | Purpose |
|---|---|
| Retry count | Measures repeated attempts |
| Backoff delay seconds | Measures recovery waiting |
| Retry rate | Measures failure pressure |
| 429 retries | Measures throttling |
| 5xx retries | Measures provider errors |
| Timeout retries | Measures slow dependency failures |
| Maximum delay reached | Detects prolonged instability |
| Retry budget exhausted | Detects unrecovered failures |
| Success after retry | Measures recovery effectiveness |

Useful structured log fields:

```text
run_id
request_id
endpoint
attempt
error_class
base_delay_seconds
calculated_delay_seconds
jitter_strategy
actual_delay_seconds
total_wait_seconds
remaining_deadline_seconds
```

Do not log sensitive credentials or request payloads.

## 16. Intentional Failure

### Failure drill A — Fixed-delay storm

Temporarily replace jittered backoff with a fixed 5-second delay across many workers.

Observe synchronized retry traffic.

Restore jitter and compare the retry distribution.

### Failure drill B — Remove the maximum delay

Force repeated failures and observe exponential delay growth.

Restore the cap.

Expected:
- delay remains bounded
- retries remain operationally predictable

### Failure drill C — Ignore `Retry-After`

Return `429` with a provider-defined retry delay.

Temporarily ignore the provider signal.

Observe excessive throttling. Restore provider-aware behavior.

### Failure drill D — Exceed the total deadline

Make every attempt fail slowly.

Expected:
- backoff respects the remaining deadline
- no sleep extends beyond the logical operation budget
- operation eventually fails or is deferred

### Failure drill E — Concurrent failure

Start many workers and make the provider return 503 simultaneously.

Expected:
- jitter spreads retry times
- rate limiting still controls aggregate request pressure
- retry volume remains bounded

## 17. Recovery

When retry traffic becomes excessive:

1. Inspect retry rate.
2. Identify the dominant failure class.
3. Check whether workers are synchronized.
4. Check current rate-limit behavior.
5. Inspect actual backoff delays.
6. Check `Retry-After` handling.
7. Check retry and deadline budgets.
8. Reduce concurrency if necessary.
9. Preserve the last durable checkpoint.
10. Resume using the bounded retry policy.

If the provider remains unavailable, stop generating retry traffic and defer the work according to the pipeline's recovery policy.

## 18. Production Tools You Should Know

### urllib3 Retry
Provides reusable retry and backoff controls for HTTP clients built around urllib3.

### Tenacity
Python retry library with configurable stop conditions, waits, exception handling, and jitter strategies.

### backoff
Python decorator library for implementing bounded retry and backoff behavior.

Use these tools after understanding the underlying mechanism. A library configuration should be explainable in terms of attempts, delay, jitter, and stop conditions.

## 19. Production Runbook

### Before running

- Define retryable failures.
- Define base backoff.
- Define maximum delay.
- Select a jitter strategy.
- Define maximum attempts.
- Define maximum total wait.
- Define overall deadline.
- Confirm provider retry instructions.
- Confirm interaction with the rate limiter.

### If retry traffic spikes

1. Check whether a provider failure affected many workers.
2. Check whether retries are synchronized.
3. Check jitter configuration.
4. Check rate-limit enforcement.
5. Check retry budgets.
6. Check `Retry-After` handling.
7. Reduce concurrency if necessary.

### What not to do

- Do not use fixed delays for large concurrent workloads.
- Do not retry immediately.
- Do not let exponential delay grow without a cap.
- Do not ignore provider retry instructions.
- Do not let retries bypass rate limiting.
- Do not rely on randomness without bounded policy.
- Do not retry forever.

## 20. Common Mistakes

1. Using fixed retry delays.
2. Omitting jitter.
3. Using exponential backoff without a maximum.
4. Having no total wait budget.
5. Ignoring provider `Retry-After` instructions.
6. Treating backoff as a replacement for rate limiting.
7. Testing random behavior nondeterministically.
8. Allowing concurrent workers to share no retry budget.
9. Starting another retry after the overall deadline.
10. Hiding retry storms behind high worker counts.

## 21. Definition of Done

- [ ] Exponential backoff is understood and implemented.
- [ ] Maximum delay is configured.
- [ ] Jitter strategy is selected intentionally.
- [ ] Provider retry instructions are respected.
- [ ] Retry count is bounded.
- [ ] Total wait is bounded.
- [ ] Overall deadline is respected.
- [ ] Backoff does not bypass rate limiting.
- [ ] Concurrent retry behavior has been tested.
- [ ] Randomness is testable deterministically.
- [ ] Backoff metrics exist.
- [ ] Retry storms can be diagnosed.
- [ ] Intentional failure drills pass.
- [ ] Recovery behavior is documented.

## 22. What You Learned

After this recipe, you should be able to independently:
- explain why fixed retry delays can create retry storms
- implement exponential backoff
- implement full, equal, and decorrelated jitter
- choose and cap retry delays
- honor provider retry instructions
- combine backoff with rate limiting and timeouts
- make retry behavior deadline-aware
- reason about concurrent workers
- test randomized behavior deterministically
- observe retry traffic
- diagnose and recover from retry storms

### Core Mental Model

```text
REQUEST
   ↓
TRANSIENT FAILURE
   ↓
CLASSIFY
   ↓
CALCULATE BACKOFF
   ↓
ADD JITTER
   ↓
CHECK DEADLINE
   ↓
WAIT
   ↓
RATE-LIMITED RETRY
   ↓
SUCCESS / BOUNDED FAILURE
```

> Backoff controls how quickly retries return. Jitter controls how synchronized those retries are. Together they prevent temporary failures from becoming self-amplifying traffic storms.