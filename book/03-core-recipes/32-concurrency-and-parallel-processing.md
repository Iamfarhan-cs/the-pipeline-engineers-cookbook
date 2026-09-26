# Recipe 32 — Concurrency & Parallel Processing

A pipeline often becomes slow because work is performed sequentially even though independent work could run at the same time.

Concurrency can improve throughput, but uncontrolled parallelism can also overload databases, APIs, memory, CPUs, and downstream systems.

This recipe teaches how to recognize when parallel processing is useful, implement bounded concurrency, test it, intentionally overload the system, and recover safely.

## 1. Problem Recognition

Consider processing 1,000 independent API requests.

Sequential execution:

    request 1 → wait
    request 2 → wait
    request 3 → wait
    ...
    request 1000 → wait

If each request takes 100 ms, the theoretical sequential time is approximately:

    1,000 × 0.1
    = 100 seconds

If ten independent requests can safely run concurrently:

    1,000 / 10 × 0.1
    ≈ 10 seconds

The exact result depends on the dependency, scheduling overhead, contention, and available capacity.

Concurrency is useful when work can safely overlap.

Typical candidates:

- independent API requests
- independent file processing
- independent partitions
- parallel transformations
- batch record processing
- database operations that do not conflict

But more workers do not automatically mean more throughput.

## 2. Concept and Reasoning

### Concurrency vs parallelism

**Concurrency** means multiple units of work can make progress during the same period.

**Parallelism** means multiple units of work execute simultaneously, usually on multiple CPU cores or workers.

For Data Engineering, both concepts matter.

A simple model:

    Work
      ↓
    Split
      ↓
    Worker 1 ─┐
    Worker 2 ─┤
    Worker 3 ─┤
    Worker 4 ─┘
      ↓
    Combine

### The important constraint

Concurrency must be bounded.

Without a limit:

    10 tasks
      ↓
    1,000 tasks
      ↓
    100,000 tasks
      ↓
    dependency overload

The goal is:

> **Use enough concurrency to improve throughput without exceeding system capacity.**

## 3. When Concurrency Helps

Concurrency is especially useful for I/O-bound work.

Examples:

    API calls
    database waits
    network requests
    object storage operations
    filesystem reads

CPU-bound work is different.

For CPU-heavy Python workloads, adding threads may not provide the expected speedup. Processes or distributed workers may be more appropriate.

Always measure rather than assuming.

## 4. When Concurrency Hurts

Parallelism can make a pipeline worse.

Example:

    Database capacity = 20 concurrent queries

Pipeline:

    workers = 100

Possible result:

    connection contention
    query queueing
    increased latency
    timeouts
    retries
    database overload

The pipeline has increased concurrency but decreased useful throughput.

This is a classic feedback loop:

    more workers
        ↓
    more load
        ↓
    dependency slows
        ↓
    timeouts
        ↓
    retries
        ↓
    even more load

## 5. Bounded Worker Pool

Build a small worker pool from scratch.

    from concurrent.futures import ThreadPoolExecutor

    def process(item):
        return transform(item)

    with ThreadPoolExecutor(max_workers=8) as executor:
        results = list(
            executor.map(process, items)
        )

The important parameter is:

    max_workers=8

The pipeline explicitly limits concurrent work.

## 6. Do Not Submit Unlimited Work Blindly

This pattern can be dangerous for very large workloads:

    futures = [
        executor.submit(process, item)
        for item in millions_of_items
    ]

Even if only a few workers execute simultaneously, the application may hold millions of future objects in memory.

Prefer bounded submission.

Conceptually:

    input
      ↓
    bounded work queue
      ↓
    fixed workers
      ↓
    output

This connects directly to backpressure.

## 7. Build a Bounded Executor

A learning implementation can use a queue and worker threads.

    from queue import Queue
    from threading import Thread

    class WorkerPool:
        def __init__(self, worker_count, queue_size):
            self.queue = Queue(maxsize=queue_size)
            self.workers = []

            for _ in range(worker_count):
                worker = Thread(target=self._worker)
                worker.start()
                self.workers.append(worker)

        def submit(self, item):
            self.queue.put(item)

        def _worker(self):
            while True:
                item = self.queue.get()

                if item is None:
                    self.queue.task_done()
                    break

                try:
                    process(item)
                finally:
                    self.queue.task_done()

The bounded queue is important.

If workers cannot keep up, producers eventually block instead of creating unlimited memory pressure.

## 8. Concurrency Limits Are Capacity Controls

Suppose:

    API capacity = 50 requests/sec

Do not automatically configure:

    500 workers

Instead determine:

    request duration
    allowed request rate
    dependency capacity
    retry behavior
    timeout behavior

Concurrency and rate limiting solve different problems.

### Concurrency

Controls how many operations are active at once.

### Rate limiting

Controls how frequently operations are started or accepted.

You may need both:

    concurrency limit
          +
    rate limit

## 9. Database Concurrency

Databases require special care.

Suppose:

    max DB connections = 20

Using:

    100 worker threads

may cause connection exhaustion.

A safe design might be:

    workers = 10
    DB pool = 10

The correct values depend on the workload and database capacity.

Never assume that increasing worker count is free.

## 10. Ordering

Parallel processing can change completion order.

Input:

    A B C D

Completion:

    B D A C

If ordering matters, you need an explicit strategy.

Options include:

- preserve input order
- attach sequence numbers
- partition by ordered key
- process dependent records sequentially

Do not accidentally introduce ordering bugs by parallelizing work that has dependencies.

## 11. Shared State

Parallel workers make shared mutable state dangerous.

Unsafe pattern:

    total += amount

from many workers without synchronization.

Possible solutions:

- thread-safe structures
- locks
- worker-local state
- message passing
- aggregation after workers finish

Prefer reducing shared mutable state rather than adding locks everywhere.

## 12. Idempotency

Concurrency increases the importance of idempotency.

A worker can:

1. process an item
2. complete the external operation
3. crash before recording success
4. retry the item

The same operation may run again.

Therefore:

    concurrency
        +
    retries
        +
    ambiguous outcomes
        ↓
    idempotency required

This connects directly to Recipe 6 and Recipe 26.

## 13. Partial Failure

Suppose 100 tasks run concurrently.

Results:

    94 SUCCESS
     4 RETRYABLE_FAILURE
     2 PERMANENT_FAILURE

Do not throw away the successful work simply because some workers failed.

Record results independently and aggregate the final outcome.

This connects to Recipe 24.

## 14. Implementation Pattern

A robust parallel batch can follow:

    INPUT
      ↓
    VALIDATE
      ↓
    SPLIT INTO WORK
      ↓
    BOUNDED WORKERS
      ↓
    COLLECT RESULT
      ↓
    CLASSIFY FAILURE
      ↓
    RETRY RETRYABLE WORK
      ↓
    QUARANTINE PERMANENT FAILURE
      ↓
    AGGREGATE RESULT

The worker should perform one clearly defined unit of work.

The coordinator should own the overall outcome.

## 15. Testing

### Test 1 — Worker limit

Submit many items and verify that active workers never exceed the configured limit.

### Test 2 — Throughput

Compare sequential and bounded-concurrent execution using the same workload.

### Test 3 — Failure isolation

Make one worker fail and verify other work continues.

### Test 4 — Retry

Make a transient task fail and verify only the appropriate task is retried.

### Test 5 — Permanent failure

Verify permanent failures are reported without repeatedly retrying them.

### Test 6 — Ordering

Verify output ordering matches the contract when ordering is required.

### Test 7 — Shared state

Run concurrent updates and verify the final result is correct.

### Test 8 — Queue pressure

Make workers slower than producers and verify the bounded queue applies backpressure.

### Test 9 — Shutdown

Verify workers stop cleanly and queued work is handled according to the shutdown policy.

## 16. Observability

Measure:

    active_workers
    queue_depth
    tasks_started_total
    tasks_completed_total
    tasks_failed_total
    task_duration_seconds
    throughput
    retries_total
    worker_utilization

Also measure downstream pressure:

    database_connections
    API_latency
    API_429_total
    timeout_total

Concurrency should be evaluated together with dependency health.

## 17. Intentional Failure

### Failure drill 1 — Remove the concurrency limit

Allow far more workers than the dependency can handle.

Observe:

    latency ↑
    timeouts ↑
    errors ↑
    retries ↑

### Failure drill 2 — Queue explosion

Allow unlimited work submission.

Observe:

    memory usage ↑
    pending tasks ↑

Then restore bounded submission.

### Failure drill 3 — Slow worker

Make one worker much slower.

Observe how completion time changes and whether work becomes imbalanced.

### Failure drill 4 — Shared-state race

Create concurrent updates to shared state without synchronization.

Observe incorrect results.

Then redesign the state handling.

### Failure drill 5 — Dependency overload

Increase workers against a deliberately constrained dependency.

Observe whether throughput reaches a ceiling and errors begin increasing.

## 18. Recovery

When excessive concurrency causes instability:

1. Reduce worker count.
2. Stop creating additional work if necessary.
3. Allow active work to drain.
4. Monitor dependency health.
5. Classify failed operations.
6. Retry only retryable failures.
7. Reconcile ambiguous operations.
8. Resume with a safe concurrency limit.
9. Compare throughput before and after.

Do not recover by immediately increasing workers again.

## 19. Concurrency Tuning

There is no universal worker count.

Measure:

    throughput
    latency
    error rate
    dependency utilization

Then increase concurrency gradually.

Example experiment:

    2 workers → 100 records/sec
    4 workers → 180 records/sec
    8 workers → 320 records/sec
    16 workers → 350 records/sec
    32 workers → 340 records/sec + errors

The useful range is near the point where additional concurrency stops producing proportional throughput improvement.

The exact limit must be established experimentally for the workload and dependency.

## 20. Concurrency and Backpressure

The relationship is:

    Producer
       ↓
    Bounded Queue
       ↓
    Workers
       ↓
    Dependency

If workers cannot consume work quickly enough, queue depth increases.

If the queue reaches capacity, the producer must slow down.

That is controlled backpressure.

Without it:

    producer rate > consumer capacity
             ↓
       unbounded memory
             ↓
           failure

## 21. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Python concurrent.futures** | Thread and process pools for bounded local concurrency. |
| **Celery** | Distributed task execution using workers, queues and task controls. |
| **Apache Spark** | Distributed parallel processing across partitions and workers. |

> These are reference tools for recognition and vocabulary. Understand worker limits, queues, shared state, ordering and failure semantics before relying on a framework.

## 22. Production Runbook

### Throughput is too low

Check:

1. Current concurrency.
2. Stage duration.
3. Queue depth.
4. CPU/memory.
5. Dependency latency.
6. Rate limits.
7. Database connection pool.

### Errors increase after scaling workers

Check:

1. API/database capacity.
2. Connection limits.
3. Rate limits.
4. Timeouts.
5. Retry volume.
6. Queue depth.

Reduce concurrency before making other changes.

### Memory increases continuously

Check:

1. Unbounded task submission.
2. Queue size.
3. Futures retained in memory.
4. Large result objects.
5. Worker leaks.

### Results are incorrect

Check:

1. Shared mutable state.
2. Ordering assumptions.
3. Duplicate execution.
4. Race conditions.
5. Missing idempotency.

### What not to do

Do not:

- maximize workers without measuring
- create unlimited pending futures
- share mutable state unnecessarily
- parallelize dependent operations
- ignore downstream capacity
- assume threads improve CPU-bound Python work
- retry every failed concurrent task blindly

## 23. Common Mistakes

### Mistake 1 — More workers equals more speed

Only true while the system has spare capacity.

### Mistake 2 — Ignoring the dependency

The pipeline may simply move the bottleneck downstream.

### Mistake 3 — Unlimited work submission

This can turn a throughput problem into a memory problem.

### Mistake 4 — Ignoring ordering

Parallel completion order is not guaranteed.

### Mistake 5 — Unsafe shared state

Race conditions can silently corrupt results.

### Mistake 6 — No idempotency

Retries can duplicate side effects.

### Mistake 7 — Measuring only pipeline duration

You need concurrency, queue, throughput, and dependency signals together.

## 24. Definition of Done

You are done when you can:

- explain concurrency and parallelism
- recognize when parallel execution is appropriate
- distinguish concurrency from rate limiting
- implement a bounded worker pool
- implement bounded work submission
- control worker count
- reason about dependency capacity
- handle ordering requirements
- avoid unsafe shared state
- combine concurrency with idempotency
- handle partial worker failure
- test worker limits
- measure throughput and latency
- intentionally overload a dependency
- diagnose excessive concurrency
- recover safely by reducing load
- tune concurrency using measurements
- explain Python concurrent.futures, Celery and Spark
- operate concurrent pipeline workloads using a production runbook

## 25. What You Learned

The central principle is:

> **Concurrency is a capacity-control problem, not simply a speed setting.**

The correct execution model is:

    INPUT
      ↓
    BOUNDED QUEUE
      ↓
    CONTROLLED WORKERS
      ↓
    DEPENDENCY
      ↓
    RESULT
      ↓
    RETRY / QUARANTINE / SUCCESS

And the tuning loop is:

    MEASURE
       ↓
    INCREASE CONCURRENCY GRADUALLY
       ↓
    OBSERVE THROUGHPUT + LATENCY + ERRORS
       ↓
    FIND CAPACITY LIMIT
       ↓
    SET SAFE BOUND
       ↓
    MONITOR

A strong Data Engineer should be able to answer:

> **How much parallelism can this pipeline safely use, what limits it, and what evidence proves that limit?**
