# Recipe 24 — Add Retry Logic

A temporary failure does not always mean that processing should stop.

A network request can time out.

An API can return a temporary server error.

A database connection can fail for a short period.

A service can temporarily reject requests because of rate limits.

In these situations, trying the operation again may succeed.

That is the purpose of retry logic.

But retrying is not as simple as running the same code again.

A retry system must answer:

- Should this error be retried?
- How many times?
- When should the next attempt happen?
- Should the delay increase?
- Should random jitter be added?
- What happens after the final attempt?
- Is the operation safe to repeat?
- What happens if the worker crashes during a retry?

This chapter builds retry logic step by step.

---

## 1. Goal

The goal is to add controlled retry behavior for temporary failures.

By the end of this recipe, you should understand how to:

- identify retryable failures
- separate retryable and permanent errors
- define retry limits
- calculate retry delays
- use exponential backoff
- use jitter
- handle API rate limits
- retry database operations safely
- retry individual records
- retry batches
- connect retries with processing status
- preserve idempotency during retries
- avoid retry storms
- record retry history
- test retry behavior
- monitor retries
- recover records after retry exhaustion

The central idea is:

> A retry is another attempt to complete the same operation safely.

---

## 2. Problem

Suppose a pipeline calls an external API.

The request fails because the service is temporarily unavailable.

A simple implementation might do this:

~~~text
request
   |
   X
error
   |
   v
stop
~~~

The pipeline has no chance to recover automatically.

A different implementation might immediately retry:

~~~text
request
   |
   X
error
   |
   v
request
   |
   X
error
   |
   v
request
~~~

This can be dangerous.

If the external service is already overloaded, sending requests continuously can make the problem worse.

A better design controls:

~~~text
failure
   |
   v
is retryable?
   |
   +---- no ----> fail / quarantine
   |
   +---- yes
          |
          v
      retry limit?
          |
          +---- reached ----> failed
          |
          +---- not reached
                    |
                    v
                  wait
                    |
                    v
                  retry
~~~

Retry logic is therefore a reliability mechanism, not simply a loop.

---

## 3. Why This Matters

Temporary failures are normal in production systems.

Examples include:

- network interruptions
- temporary service outages
- database connection failures
- rate limits
- temporary infrastructure problems
- deadlocks
- temporary storage failures

Without retries, the pipeline may fail unnecessarily.

With badly designed retries, the pipeline may create:

- duplicate work
- excessive load
- retry storms
- delayed processing
- repeated side effects
- noisy logs
- hidden permanent failures

The goal is controlled recovery.

---

## 4. When to Use This Recipe

Retry logic is useful when an operation can fail temporarily.

Common examples:

- HTTP requests
- database operations
- object storage access
- message processing
- remote service calls
- temporary filesystem operations
- distributed service communication

Retry logic is usually not useful for errors where the input itself is invalid.

For example:

~~~text
missing required field
invalid currency
malformed JSON
unsupported value
~~~

Retrying the same input will normally produce the same failure.

This is why Chapter 23 classified errors before retry behavior was added.

---

## 5. Architecture

A basic retry flow looks like this:

~~~text
                 Process operation
                        |
                        v
                     Success?
                    /       \
                  Yes        No
                  |           |
                  v           v
              completed    classify
                              |
                    +---------+---------+
                    |                   |
                retryable          permanent
                    |                   |
                    v                   v
              attempts left?        fail/quarantine
                 /     \
               Yes      No
                |        |
                v        v
              wait     failed
                |
                v
              retry
~~~

The retry system should make each decision explicit.

---

## 6. Before You Start

Before adding retry logic, investigate the operation.

Answer these questions:

1. What operation can fail?
2. What errors can it produce?
3. Which errors are temporary?
4. Which errors are permanent?
5. Is the operation safe to repeat?
6. What is the maximum number of attempts?
7. What delay should be used?
8. Does the remote service provide retry guidance?
9. What happens after retries are exhausted?
10. How is retry state stored?
11. How is retry activity observed?
12. How will the retry behavior be tested?

Do not start by adding a generic retry decorator to everything.

First understand the operation.

---

## 7. Retryable Errors

A retryable error is a failure where another attempt may succeed without changing the underlying input.

Typical examples:

~~~text
network timeout
temporary connection failure
HTTP 503
HTTP 502
temporary database connection failure
rate limit
deadlock
~~~

These are general examples.

The actual classification depends on the system and its contracts.

---

## 8. Non-Retryable Errors

A non-retryable error usually requires some other action.

Examples:

~~~text
invalid input
missing required field
invalid authentication credentials
unsupported operation
malformed request
business-rule violation
~~~

The correct response may be:

- reject
- quarantine
- alert
- fix the input
- fix the code
- update configuration

Retrying should not be used to avoid investigating a permanent failure.

---

## 9. Retry Classification Comes First

The retry system should not decide whether an error is retryable based only on the fact that an exception occurred.

The flow should be:

~~~text
operation
   |
   v
error
   |
   v
classify error
   |
   +---- retryable
   |
   +---- non-retryable
   |
   +---- unknown
~~~

Unknown errors should be handled deliberately.

Depending on the system, they may:

- stop processing
- trigger an alert
- enter a safe retry path
- require investigation

Do not silently classify every unknown error as retryable.

---

## 10. Retry Count

Every retry system needs a limit.

For example:

~~~text
max_attempts = 4
~~~

The attempts could be:

~~~text
attempt 1 -> initial attempt
attempt 2 -> retry
attempt 3 -> retry
attempt 4 -> final retry
~~~

After the final attempt fails:

~~~text
failed
   |
   v
quarantine / alert / manual recovery
~~~

The exact number is a policy decision.

There is no universal value that is correct for every pipeline.

---

## 11. Attempt Count vs Retry Count

These terms can be confused.

Suppose:

~~~text
attempt 1 -> fails
attempt 2 -> fails
attempt 3 -> succeeds
~~~

The operation had:

- 3 total attempts
- 2 retries

A system should define its terminology clearly.

For example:

~~~text
attempt_count = 3
retry_count = 2
~~~

Do not mix these meanings in database fields, logs, and dashboards.

---

## 12. Simple Retry Loop

A generic retry implementation might look like:

~~~python
for attempt in range(1, max_attempts + 1):
    try:
        process()
        break
    except RetryableError:
        if attempt == max_attempts:
            raise
        wait_before_retry(attempt)
~~~

This is a **generic example**.

The important structure is:

1. attempt the operation
2. catch a retryable failure
3. check the attempt limit
4. wait
5. try again
6. fail after the limit

The real implementation may need additional state and error handling.

---

## 13. Why Immediate Retry Is Usually a Bad Idea

Suppose an API is temporarily overloaded.

The pipeline sends:

~~~text
request
request
request
request
request
~~~

within a very short period.

The service may become even more overloaded.

Immediate retries can create a feedback loop:

~~~text
service slows down
      |
      v
requests fail
      |
      v
clients retry immediately
      |
      v
more requests
      |
      v
service slows down further
~~~

This is one reason retry delays are important.

---

## 14. Fixed Delay

The simplest retry strategy uses a fixed delay.

For example:

~~~text
attempt 1 -> fail
wait 5 seconds
attempt 2 -> fail
wait 5 seconds
attempt 3 -> fail
~~~

The delay remains constant.

Conceptually:

~~~text
delay = 5 seconds
~~~

Advantages:

- simple
- predictable
- easy to understand

Limitations:

- many workers can retry at the same time
- it may not adapt to increasing failure duration

For some simple systems, fixed delay can be sufficient.

---

## 15. Exponential Backoff

Exponential backoff increases the delay after each failed attempt.

A common generic formula is:

~~~text
delay = base_delay × 2^(attempt - 1)
~~~

For example, with a base delay of 2 seconds:

~~~text
attempt 1 -> 2 seconds
attempt 2 -> 4 seconds
attempt 3 -> 8 seconds
attempt 4 -> 16 seconds
~~~

The exact formula can vary.

The important idea is:

> Wait longer when repeated attempts continue to fail.

---

## 16. Backoff With a Maximum Delay

Unbounded exponential growth is usually not desirable.

A maximum delay can be applied:

~~~text
delay = min(calculated_delay, maximum_delay)
~~~

For example:

~~~text
calculated delay: 64 seconds
maximum delay:    30 seconds

actual delay:     30 seconds
~~~

This prevents a single retry interval from becoming excessively long.

---

## 17. Jitter

Even exponential backoff can create a problem.

Imagine 1,000 workers all fail at the same time.

They calculate the same retry schedule:

~~~text
worker 1 -> retry at 10:00:05
worker 2 -> retry at 10:00:05
worker 3 -> retry at 10:00:05
...
worker 1000 -> retry at 10:00:05
~~~

They all hit the dependency again at the same time.

This can create a retry storm.

Jitter adds controlled randomness to the delay.

Conceptually:

~~~text
calculated delay
       |
       v
add jitter
       |
       v
actual retry delay
~~~

For example:

~~~text
worker A -> 4.2 seconds
worker B -> 5.1 seconds
worker C -> 4.7 seconds
worker D -> 5.6 seconds
~~~

The exact jitter algorithm depends on the system.

---

## 18. Backoff and Jitter Together

A common pattern is:

~~~text
failure
  |
  v
exponential backoff
  |
  v
jitter
  |
  v
wait
  |
  v
retry
~~~

This helps prevent many workers from retrying simultaneously.

It is especially useful in distributed systems.

---

## 19. Retry-After

Some APIs provide explicit retry guidance.

For example, an API may return a response indicating that the client should wait before trying again.

A generic flow is:

~~~text
API response
     |
     v
retry guidance?
     |
     +---- yes ----> use server guidance
     |
     +---- no -----> use client backoff policy
~~~

The API contract should be followed.

Do not blindly ignore a server-provided retry interval.

---

## 20. HTTP Rate Limits

Rate limits are a common reason for retries.

For example:

~~~text
request
   |
   v
HTTP 429
   |
   v
rate limited
   |
   v
wait according to policy
   |
   v
retry
~~~

The retry policy should consider:

- server-provided retry guidance
- maximum attempts
- request cost
- concurrency
- overall processing rate

If a service allows only a limited number of requests, retrying aggressively is counterproductive.

---

## 21. Authentication Errors

Authentication failures require special care.

Suppose an API returns an authentication error because a credential has expired.

Retrying the same request immediately will probably not fix it.

For example:

~~~text
request
   |
   v
authentication failure
   |
   v
same credentials
   |
   v
authentication failure
~~~

This should normally lead to investigation or credential refresh logic, not an unlimited retry loop.

The exact behavior depends on the authentication system.

---

## 22. Database Retries

Database operations can also have temporary failures.

Examples include:

- connection interruption
- temporary unavailability
- deadlock
- transient network problem

But database retries must respect transaction boundaries.

For example:

~~~text
BEGIN
  operation A
  operation B
COMMIT
~~~

If the transaction fails, the retry should normally repeat the appropriate transaction rather than continuing halfway through it.

A useful conceptual pattern is:

~~~text
start transaction
      |
      v
perform operation
      |
   success?
    /    \
  yes     no
  |        |
commit   rollback
           |
           v
       retry if safe
~~~

The exact behavior depends on the database error.

---

## 23. Deadlocks

Database deadlocks are a useful example of a potentially retryable error.

Two transactions may wait for each other:

~~~text
Transaction A
    |
    +--> locks row 1
    |
    +--> waits for row 2

Transaction B
    |
    +--> locks row 2
    |
    +--> waits for row 1
~~~

The database may abort one transaction.

The application can sometimes retry the entire transaction.

But the transaction must be designed so that repeating it is safe.

---

## 24. Retry and Idempotency

Retry logic depends heavily on idempotency.

Suppose a worker performs:

~~~text
create payment
~~~

The request reaches the remote service.

The remote service creates the payment.

The response is lost.

The worker sees:

~~~text
timeout
~~~

It retries.

If the remote operation is not idempotent, the system might create the payment twice.

The flow becomes:

~~~text
worker
  |
  +---- request ----> service
  |                     |
  |                     v
  |                  payment created
  |
  X response lost
  |
  v
retry
  |
  +---- request ----> service
                        |
                        v
                     duplicate
~~~

This is why retries and idempotency must be designed together.

Chapter 5 covered idempotency in detail.

---

## 25. Idempotency Key

A common pattern for external operations is an idempotency key.

Conceptually:

~~~text
operation
    |
    v
stable idempotency key
    |
    v
external service
~~~

If the same operation is sent again with the same key, the service can recognize it as the same operation.

The exact support depends on the external service.

Do not assume every API provides idempotency keys.

---

## 26. Retry and Processing Status

The processing status from Chapter 22 should represent retry state clearly.

For example:

~~~text
pending
   |
   v
processing
   |
   X
failed
   |
   v
retry_pending
   |
   v
processing
~~~

After the retry limit:

~~~text
retry_pending
      |
      v
attempt limit reached
      |
      v
failed / quarantined
~~~

The exact states depend on the pipeline.

The important part is that workers can determine:

- whether work should be attempted
- when it should be attempted
- how many attempts have happened
- whether another retry is allowed

---

## 27. Retry Scheduling

A retry does not always need to happen inside the same worker execution.

For example:

~~~text
failure
   |
   v
store retry state
   |
   v
retry_at = future time
   |
   v
worker later claims record
   |
   v
retry
~~~

This can be more reliable for long delays.

Instead of keeping a worker blocked:

~~~text
worker
   |
   v
sleep 30 minutes
~~~

the system can store the future retry time:

~~~text
status = retry_pending
retry_at = 10:30
~~~

Then a worker can pick it up when it becomes eligible.

---

## 28. Retry Scheduling With Database State

A generic schema might contain:

~~~text
status
attempt_count
next_retry_at
last_error
updated_at
~~~

For example:

~~~text
status          = retry_pending
attempt_count   = 2
next_retry_at   = 2026-09-26 10:30:00
last_error      = timeout
~~~

This is a **generic example**.

The purpose is to make retry decisions durable.

If the worker crashes, the retry information is still available.

---

## 29. Worker Crash During Retry

Suppose a worker does:

~~~text
claim record
   |
   v
process
   |
   X
worker crashes
~~~

The system needs a recovery mechanism.

Possible approaches include:

- processing leases
- heartbeats
- stale-state detection
- transaction rollback
- retryable processing states

The correct mechanism depends on the architecture.

The important point is:

> Retry state should not exist only in worker memory when recovery depends on it.

---

## 30. Record-Level Retry

For independent records, retry can happen at record level.

For example:

~~~text
Record A -> completed
Record B -> retry_pending
Record C -> completed
Record D -> failed
~~~

The worker can continue processing A and C while B waits for its retry time.

This can improve throughput.

But it requires clear state management.

---

## 31. Batch-Level Retry

Some operations are naturally batch-level.

For example:

~~~text
download entire file
~~~

If the download fails, retrying the entire operation may make sense.

Another example:

~~~text
load a transaction into a warehouse table
~~~

If the entire transaction fails, the batch may need to be retried.

The retry unit should match the operation's failure boundary.

---

## 32. Do Not Retry at Multiple Layers Without a Plan

A dangerous design is:

~~~text
HTTP client retries 3 times
       |
       v
worker retries 3 times
       |
       v
job retries 3 times
~~~

This can create many more attempts than expected.

For example:

~~~text
3 × 3 × 3 = 27 attempts
~~~

The application may think it has a small retry policy while the actual system performs many more attempts.

Define which layer owns retry responsibility.

---

## 33. Retry Storms

A retry storm occurs when many clients retry a failing dependency at the same time.

The pattern can be:

~~~text
dependency fails
      |
      v
many workers fail
      |
      v
many workers retry
      |
      v
dependency receives more traffic
      |
      v
dependency remains unhealthy
      |
      v
workers retry again
~~~

This can make an outage worse.

Protection can include:

- exponential backoff
- jitter
- retry limits
- concurrency limits
- rate limits
- circuit breakers
- queue-based retry scheduling

The appropriate controls depend on the architecture.

---

## 34. Circuit Breakers

A circuit breaker can stop repeated calls to a failing dependency.

Conceptually:

~~~text
normal
  |
  v
failures increase
  |
  v
open circuit
  |
  v
stop requests temporarily
  |
  v
test dependency later
  |
  v
recover
~~~

This is different from retry.

Retry says:

> Try the operation again.

A circuit breaker says:

> Stop sending requests for a period because the dependency is failing.

They can work together.

---

## 35. Retry Budget

A retry budget limits how much additional work retries are allowed to create.

For example, suppose a system normally processes:

~~~text
10,000 requests
~~~

If every request can be retried five times, the dependency could receive far more traffic.

A retry budget helps control this additional load.

The exact policy depends on the system.

The important idea is:

> Retries consume system capacity.

They are not free.

---

## 36. Retry History

A useful retry system should make the history understandable.

For example:

~~~text
record 1001

attempt 1 -> timeout
attempt 2 -> timeout
attempt 3 -> rate limited
attempt 4 -> completed
~~~

This can help answer:

- how often is the dependency failing?
- how many retries are normal?
- which records require many attempts?
- are retry delays too long?
- is the same error repeating?

History can be stored in a dedicated table or represented through logs and metrics, depending on requirements.

---

## 37. Retry Metrics

Useful retry metrics include:

~~~text
retries_total
retries_by_error_type
retries_by_service
retry_exhausted_total
retry_success_total
retry_attempts_per_record
retry_delay
~~~

These metrics can show whether retries are helping.

For example:

~~~text
retry attempts: 10,000
successful retries: 9,500
exhausted retries: 500
~~~

The exact metrics should match the system's operational needs.

---

## 38. Retry Success Rate

A useful measurement is the percentage of retries that eventually succeed.

Conceptually:

~~~text
retry success rate =
successful retry recoveries / retry attempts
~~~

The exact denominator should be defined carefully.

A high retry volume with very few successful recoveries may indicate that the retry policy is wasting resources.

A lower retry volume with good recovery may indicate that the retry policy is targeting temporary failures effectively.

Metrics should be interpreted together rather than in isolation.

---

## 39. Testing Retry Logic

Retry behavior must be tested directly.

Do not assume it works because a loop exists.

At minimum, test:

### Test 1 — First attempt succeeds

Expected:

~~~text
one attempt
no retry
~~~

### Test 2 — First attempt fails, second succeeds

Expected:

~~~text
two attempts
one retry
final status = completed
~~~

### Test 3 — All attempts fail

Expected:

~~~text
maximum attempts reached
final failure state
~~~

### Test 4 — Non-retryable error

Expected:

~~~text
no retry
~~~

### Test 5 — Retryable error

Expected:

~~~text
retry occurs
~~~

### Test 6 — Backoff

Expected:

~~~text
delay follows configured policy
~~~

### Test 7 — Retry state persistence

Expected:

~~~text
retry information survives worker restart
~~~

### Test 8 — Idempotent retry

Expected:

~~~text
repeated operation does not create duplicate results
~~~

---

## 40. Test Retry Limits

A common bug is an off-by-one error.

Suppose:

~~~text
max_attempts = 3
~~~

You need to verify exactly how many calls occur.

Expected:

~~~text
attempt 1
attempt 2
attempt 3
stop
~~~

Not:

~~~text
attempt 1
attempt 2
attempt 3
attempt 4
stop
~~~

Define whether the configured value means total attempts or retry count.

Then test that exact behavior.

---

## 41. Test Backoff Without Making Tests Slow

Real retry delays can make automated tests unnecessarily slow.

For example:

~~~text
2 seconds
4 seconds
8 seconds
16 seconds
~~~

A test suite should not need to wait 30 seconds to verify the retry calculation.

A common testing approach is to:

- inject a sleep function
- inject a clock
- calculate delay without actually sleeping
- mock the waiting mechanism

The exact approach depends on the implementation.

The important goal is to test behavior without making the test suite unnecessarily slow.

---

## 42. Test Jitter

Jitter is random by design.

Tests should therefore not require an exact random delay unless randomness is controlled.

A better test can verify:

~~~text
delay >= minimum
delay <= maximum
~~~

Or use an injectable random-number generator.

The test should verify the policy, not depend on an uncontrolled random value.

---

## 43. Testing Rate Limits

For an API client, simulate a rate-limit response.

For example:

~~~text
request 1 -> HTTP 429
request 2 -> HTTP 429
request 3 -> success
~~~

Verify that:

- the 429 is classified correctly
- retry occurs
- the delay policy is applied
- the maximum attempt limit is respected
- the final operation succeeds

Also test the case where every attempt receives 429.

---

## 44. Testing Database Retry

Database retry tests should simulate the relevant temporary failure.

For example:

~~~text
transaction attempt 1 -> deadlock
transaction attempt 2 -> success
~~~

Verify:

- the failed transaction is rolled back
- the operation is retried
- the transaction starts cleanly again
- the final result exists once
- no partial result remains

This is where transaction correctness and idempotency become important.

---

## 45. Practical Implementation Sequence

For an existing repository, use this order:

~~~text
1. Find existing error handling
        |
        v
2. Identify operations that can be retried
        |
        v
3. Identify retryable errors
        |
        v
4. Identify non-retryable errors
        |
        v
5. Confirm idempotency
        |
        v
6. Define retry ownership
        |
        v
7. Define maximum attempts
        |
        v
8. Define backoff policy
        |
        v
9. Define jitter if needed
        |
        v
10. Define rate-limit behavior
        |
        v
11. Define retry state
        |
        v
12. Define retry scheduling
        |
        v
13. Implement retry handling
        |
        v
14. Connect processing status
        |
        v
15. Add retry metrics and logs
        |
        v
16. Test successful retry
        |
        v
17. Test exhausted retry
        |
        v
18. Test non-retryable errors
        |
        v
19. Test worker restart/recovery
        |
        v
20. Verify final state
~~~

Do not implement backoff before deciding what is actually retryable.

---

## 46. Troubleshooting

### Problem: retries never happen

Check:

1. Is the error classified as retryable?
2. Is the retry limit already reached?
3. Is the record in a retryable processing state?
4. Is the retry scheduler running?
5. Is the next retry time correct?
6. Is another worker holding the record?

### Problem: retries happen too quickly

Check:

1. Is backoff configured?
2. Is the calculated delay correct?
3. Is the delay actually applied?
4. Is another retry layer performing immediate retries?
5. Is the server-provided retry guidance being ignored?

### Problem: too many retries

Check:

1. retry limits
2. nested retry layers
3. worker retries
4. job retries
5. client-library retries
6. queue redelivery
7. scheduler retries

The effective retry count may be larger than expected.

### Problem: duplicate results after retry

Investigate:

1. idempotency key
2. database uniqueness constraints
3. transaction boundaries
4. external side effects
5. whether the first attempt actually succeeded before the response was lost

A timeout does not prove that the operation did not happen.

### Problem: workers keep retrying the same record

Check:

1. whether the error is actually permanent
2. whether the retry limit is persisted
3. whether attempt count is incremented
4. whether the worker can update status
5. whether retry state is being reset accidentally
6. whether the error classification is wrong

---

## 47. Production Considerations

Retry logic affects production traffic.

Before deploying it, consider:

### Dependency capacity

Retries create additional requests.

### Concurrency

Many workers can retry simultaneously.

### Rate limits

External services may reject excessive requests.

### Latency

Backoff increases processing time.

### Storage

Retry history and error information require retention policies.

### Observability

Retries should be visible in logs and metrics.

### Recovery

Exhausted retries need a clear next step.

### Idempotency

Repeated operations must be safe.

### Configuration

Retry limits and delays should be configurable where appropriate.

---

## 48. Retry Configuration

A retry policy may contain values such as:

~~~text
max_attempts
base_delay
maximum_delay
jitter_enabled
retryable_errors
retryable_status_codes
~~~

This is a **generic example**.

Do not expose secrets through configuration.

Also avoid creating dozens of configuration values without a clear operational reason.

Configuration should be understandable and validated.

---

## 49. Retry Ownership

Every system should have a clear answer to:

> Which layer is responsible for retrying this operation?

Possible layers include:

~~~text
HTTP client
worker
queue consumer
scheduler
job runner
orchestrator
~~~

If multiple layers retry independently, the total number of attempts can become difficult to predict.

A clean design defines the ownership explicitly.

---

## 50. Retry vs Replay

Retry and replay are related but different.

### Retry

Retry usually means:

> Try the same failed operation again because the failure may be temporary.

### Replay

Replay usually means:

> Process an event or record again through the pipeline.

For example:

~~~text
timeout
  |
  v
retry
~~~

But:

~~~text
bad code deployed
  |
  v
fix code
  |
  v
replay historical failed records
~~~

Replay may happen much later and may use a different processing version.

Chapter 25 will cover replay and reprocessing in more detail.

---

## 51. Retry vs Backfill

A retry usually handles a failed current operation.

A backfill processes historical data intentionally.

For example:

~~~text
today's API call failed
        |
        v
retry
~~~

Whereas:

~~~text
historical records from January
        |
        v
new transformation
        |
        v
backfill
~~~

The two can interact, but they should not be treated as the same operation.

---

## 52. Definition of Done

Retry logic is complete when:

- [ ] Retryable failures are identified.
- [ ] Non-retryable failures are identified.
- [ ] Unknown failures have a defined handling path.
- [ ] Maximum attempts are defined.
- [ ] Attempt count and retry count are clearly defined.
- [ ] Backoff behavior is defined.
- [ ] Maximum delay is defined where needed.
- [ ] Jitter is considered where multiple workers can retry together.
- [ ] API retry guidance is respected where applicable.
- [ ] Rate-limit behavior is defined.
- [ ] Database transaction retry behavior is defined.
- [ ] Retry ownership is clear.
- [ ] Nested retry behavior has been reviewed.
- [ ] Idempotency has been verified.
- [ ] Retry state is durable where required.
- [ ] Processing status represents retry state correctly.
- [ ] Retry metrics are available.
- [ ] Retry logs contain useful context.
- [ ] Successful retries are tested.
- [ ] Exhausted retries are tested.
- [ ] Non-retryable errors are tested.
- [ ] Backoff behavior is tested.
- [ ] Worker restart behavior is tested where relevant.
- [ ] Duplicate effects are tested.
- [ ] Recovery after retry exhaustion is documented.

---

## 53. What You Learned

Retry logic is not:

~~~python
try_again()
~~~

It is a controlled recovery mechanism.

A good retry system answers:

~~~text
What failed?
    |
    v
Can it succeed later?
    |
    +---- no ----> fail / quarantine
    |
    +---- yes
           |
           v
       attempts left?
           |
        +--+--+
        |     |
       yes    no
        |     |
        v     v
      wait   fail
        |
        v
      retry
~~~

The most important lessons are:

1. Do not retry permanent failures.
2. Put a limit on retries.
3. Use backoff when repeated attempts can overload a dependency.
4. Add jitter when many workers may retry together.
5. Follow server-provided retry guidance when available.
6. Make repeated operations safe through idempotency.
7. Keep retry state durable when recovery depends on it.
8. Avoid multiple independent retry layers unless their interaction is understood.
9. Test retry behavior directly.
10. Make exhausted retries recoverable.

The final goal is not to retry more.

The goal is to recover from temporary failures without creating new failures.
