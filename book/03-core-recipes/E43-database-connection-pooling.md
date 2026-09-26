# E43 — Database Connection Pooling

## 1. Problem Recognition

### The production problem

A data pipeline needs database connections for extraction.

The naive implementation opens a new connection for every operation:

```
read batch 1 → connect → query → close
read batch 2 → connect → query → close
read batch 3 → connect → query → close
...
```

This creates unnecessary overhead.

A different mistake is keeping one connection per worker forever:

```
100 workers
   ↓
100 database connections
```

The database may have far fewer connections available.

Connection pooling solves the resource-management problem by maintaining a controlled set of reusable database connections.

```
PIPELINE WORKERS
      ↓
CONNECTION POOL
      ↓
LIMITED DATABASE CONNECTIONS
      ↓
SOURCE DATABASE
```

### How to recognize the problem

Look for:

- connection creation on every batch
- high connection latency
- frequent authentication/TLS handshakes
- connection exhaustion
- database errors such as too many connections
- workers waiting for connections
- idle connections consuming database capacity
- long-lived connections with stale state
- connection leaks
- pool exhaustion during concurrency spikes

The key question is:

> **How many database connections does the pipeline actually need, and how do we guarantee that it never exceeds the safe source-side budget?**

---

# 2. Concept and Reasoning

## 2.1 What is a connection pool?

A connection pool is a managed collection of reusable database connections.

Instead of creating a connection for every operation:

```
WORKER
  ↓
POOL.acquire()
  ↓
CONNECTION
  ↓
QUERY
  ↓
POOL.release()
```

The connection returns to the pool for another worker.

A pool controls:

- minimum connections
- maximum connections
- acquisition waiting
- connection lifetime
- idle lifetime
- health checking
- cleanup
- shutdown

---

# 3. Why Connection Creation Is Expensive

Opening a database connection can involve:

1. TCP connection
2. TLS negotiation
3. authentication
4. session initialization
5. database protocol setup

For a pipeline processing thousands of batches, repeatedly paying this cost is inefficient.

Without pooling:

```
1000 batches
×
connection setup
=
1000 connection lifecycles
```

With a pool:

```
1000 batches
↓
reuse a controlled connection set
```

The pool does not make the database infinitely scalable.

It makes connection reuse and concurrency explicit.

---

# 4. Pooling Is a Concurrency Control Mechanism

A connection pool is not merely a performance optimization.

It creates a hard limit.

Suppose:

```
workers = 50
pool max = 8
```

Only eight workers can hold database connections at the same time.

The remaining workers wait.

```
50 workers
    ↓
8 database connections
    ↓
database
```

This can protect the source database from an uncontrolled connection storm.

---

# 5. Pool Size Is Not the Same as Worker Count

Do not automatically configure:

```
pool_size = number_of_workers
```

Suppose:

```
workers = 32
database safe budget = 10 connections
```

A pool of 32 could overwhelm the database.

A pool of 6–8 may be more appropriate depending on:

- source workload
- query duration
- CPU
- I/O
- database connection limits
- other applications
- replicas
- concurrent pipelines

The correct pool size is an operational decision, not a magic formula.

---

# 6. Source Connection Budget

Always think in terms of the entire database.

Suppose the database allows 100 connections.

Other applications already consume:

```
API service          30
admin/monitoring      5
background jobs      20
reserved capacity    15
-----------------------
available            30
```

A pipeline should not simply claim all 30.

Reserve headroom.

A practical model is:

```
database connection limit
        ↓
subtract existing workloads
        ↓
subtract safety reserve
        ↓
pipeline connection budget
        ↓
divide across pipeline instances
```

If there are five pipeline instances, each instance cannot independently assume the full global budget.

---

# 7. Pooling and Pipeline Architecture

A typical architecture is:

```
                 PIPELINE
                    ↓
             WORKER COORDINATOR
                    ↓
              CONNECTION POOL
              /      |       \
             /       |        \
        connection connection connection
             \       |       /
              \      |      /
                 DATABASE
```

For multiple application processes:

```
worker process A → pool A ─┐
worker process B → pool B ─┼→ database
worker process C → pool C ─┘
```

Each process has its own pool unless an external pooler is used.

Therefore:

> **Total connections = pool size × active application processes**, approximately, subject to pool configuration and lifecycle.

---

# 8. Build a Small Pool from Scratch

The following example uses Python and PostgreSQL.

First install psycopg:

```bash
pip install "psycopg[binary,pool]"
```

Create a pool:

```python
from psycopg_pool import ConnectionPool

pool = ConnectionPool(
    conninfo="postgresql://app:secret@localhost:5432/appdb",
    min_size=2,
    max_size=6,
)
```

Acquire and return a connection:

```python
def read_batch(pool, limit: int):
    with pool.connection() as conn:
        with conn.cursor() as cur:
            cur.execute(
                """
                SELECT id, updated_at
                FROM events
                ORDER BY id
                LIMIT %s
                """,
                (limit,),
            )
            return cur.fetchall()
```

The context manager returns the connection to the pool.

It does not normally mean:

```
close physical database connection
```

It means:

```
release connection back to pool
```

---

# 9. Always Return Connections

A connection leak is one of the most dangerous pooling mistakes.

Bad:

```python
conn = pool.getconn()

# query

# exception happens
# connection never returned
```

Repeated enough times:

```
pool
 ↓
connections checked out
 ↓
connections never returned
 ↓
pool exhausted
 ↓
workers block
```

Prefer managed lifecycle:

```python
with pool.connection() as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT 1")
```

The same principle applies when using other database libraries.

---

# 10. Pool Exhaustion

Suppose:

```
max pool size = 4
active workers = 12
```

Four workers acquire connections.

The other eight wait.

This is not automatically an error.

Waiting is often safer than opening eight more database connections.

But excessive waiting indicates a capacity problem.

Possible causes:

- queries are too slow
- transactions are too long
- pool is too small
- worker concurrency is too high
- connections are leaked
- database is overloaded

---

# 11. Pool Acquisition Timeout

Do not allow workers to wait forever.

Use a bounded acquisition timeout where supported.

Conceptually:

```
REQUEST CONNECTION
       ↓
WAIT
       ↓
AVAILABLE?
   /       \
 YES       NO
  ↓         ↓
USE      TIMEOUT
           ↓
       CLASSIFY FAILURE
```

A timeout tells you:

> The pipeline could not obtain a database connection within its operational budget.

That is different from:

> The database query failed.

Keep these failures distinct in logs and metrics.

---

# 12. Connection Pool vs Query Timeout

These are different controls.

| Timeout | Protects |
|---|---|
| Pool acquisition timeout | Waiting for a connection |
| Connection timeout | Establishing a connection |
| Query timeout | Executing a database query |
| Transaction timeout | Long-running transaction |
| Pipeline deadline | Entire operation |

Example:

```
worker
 ↓
wait 5s for pool connection
 ↓
connect
 ↓
query max 30s
 ↓
release
```

Do not use one timeout value as a substitute for all of them.

---

# 13. Connection Health

A pooled connection may become unusable while sitting idle.

Possible causes:

- network interruption
- database restart
- firewall timeout
- load balancer timeout
- server-side termination
- connection lifetime exceeded

The pool should detect or recover from stale connections according to the library's capabilities.

Conceptually:

```
POOL
 ↓
borrow connection
 ↓
health check / operational validation
 ↓
healthy → use
unhealthy → discard/recreate
```

Do not blindly assume that a connection created two hours ago is still valid.

---

# 14. Connection Lifetime

Long-lived connections can accumulate session state or become invalid.

A pool can enforce maximum connection lifetime.

Conceptually:

```
connection created
       ↓
reused
       ↓
reused
       ↓
maximum lifetime reached
       ↓
retire
       ↓
new connection
```

This can help with:

- infrastructure changes
- network path changes
- server restarts
- stale sessions
- gradual connection recycling

Do not set an extremely short lifetime without reason; excessive churn recreates the overhead pooling was intended to avoid.

---

# 15. Session State Is Dangerous

Database connections can retain session-level state.

Examples include:

- \`SET\` configuration
- temporary tables
- prepared statements
- session variables
- advisory locks
- transaction state

Suppose worker A changes:

```sql
SET search_path = special_schema;
```

The connection returns to the pool.

Worker B receives the same connection.

If the state is not reset, worker B may execute against an unexpected environment.

Therefore:

> **A pooled connection must be treated as reusable shared infrastructure, not as a permanently private connection.**

---

# 16. Transactions and Pooling

A connection should not be returned to the pool with an open transaction.

Bad lifecycle:

```
acquire
 ↓
BEGIN
 ↓
query
 ↓
exception
 ↓
return connection
 ↓
transaction remains open
```

This can create:

- locks
- stale snapshots
- idle-in-transaction sessions
- MVCC pressure
- incorrect later behavior

Use explicit transaction boundaries and ensure rollback on failure.

```python
with pool.connection() as conn:
    try:
        with conn.transaction():
            with conn.cursor() as cur:
                cur.execute(
                    "SELECT COUNT(*) FROM events"
                )
    except Exception:
        raise
```

The exact transaction API varies by driver.

The invariant does not:

> **Never return a connection to the pool with an unintended transaction state.**

---

# 17. Pooling and Consistent Snapshots

E42 introduced database snapshot consistency.

Pooling adds an important constraint.

A snapshot belongs to a transaction/connection context.

Therefore:

```
snapshot transaction
        ↓
must remain associated with
        ↓
the same connection
```

Do not:

```
acquire connection A
create snapshot
release A
acquire connection B
continue snapshot
```

unless the database explicitly supports the required snapshot handoff mechanism.

For a normal transaction snapshot:

```
acquire
 ↓
BEGIN
 ↓
establish snapshot
 ↓
read
 ↓
commit/rollback
 ↓
release
```

This is one reason large consistent snapshots need careful connection-pool design.

---

# 18. Pooling and Streaming Extraction

Large database extraction often uses a server-side cursor or streaming mechanism.

The connection may need to remain checked out for the entire stream.

```
acquire connection
       ↓
BEGIN
       ↓
stream batch 1
       ↓
stream batch 2
       ↓
stream batch 3
       ↓
...
       ↓
COMMIT / ROLLBACK
       ↓
release
```

If ten workers each hold a connection for a long-running stream, the pool needs at least ten connections for full concurrency.

But increasing the pool can increase source pressure.

This is a throughput-versus-source-load trade-off.

---

# 19. Pooling and Parallel Extraction

Suppose:

```
workers = 20
pool max = 5
```

Only five database operations can use pooled connections concurrently.

The other workers wait.

If each query takes 10 seconds, increasing the pool may improve throughput.

But if the database is already CPU-bound, increasing connections may make performance worse.

More connections do not necessarily mean more throughput.

A common pattern is:

```
more concurrency
       ↓
more active queries
       ↓
more CPU / I/O contention
       ↓
longer queries
       ↓
connections held longer
       ↓
pool exhaustion
```

This is why connection pooling must be considered together with query performance.

---

# 20. Pool Size and Query Duration

A useful mental model is:

```
throughput ≈ concurrent database work / average work duration
```

If:

```
pool = 4
average query = 1 second
```

you may have approximately four active query slots.

If:

```
pool = 20
average query = 10 seconds
```

you may simply have twenty long-running queries competing for the same database resources.

Measure before changing the pool.

---

# 21. Database Connection Limits

PostgreSQL exposes a database/server connection limit.

Inspect the configuration:

```sql
SHOW max_connections;
```

Also inspect active sessions:

```sql
SELECT
    state,
    count(*)
FROM pg_stat_activity
GROUP BY state;
```

For operational investigation:

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query_start
FROM pg_stat_activity
ORDER BY query_start;
```

Do not treat the maximum connection count as the pipeline's available budget.

Other workloads need capacity.

---

# 22. Application Name

Give extraction connections a recognizable application identity where supported.

For PostgreSQL:

```
application_name = pipeline-extractor
```

Then operational queries can identify pipeline sessions.

This makes incidents easier to diagnose.

Example:

```sql
SELECT
    application_name,
    state,
    count(*)
FROM pg_stat_activity
GROUP BY application_name, state
ORDER BY application_name, state;
```

A production connection should be identifiable without exposing secrets.

---

# 23. Multi-Process Pooling

Suppose a deployment has:

```
4 worker processes
pool max = 8 per process
```

The theoretical connection capacity becomes:

```
4 × 8 = 32
```

This is frequently missed.

If the database budget is 20, a per-process pool of 8 is already too large.

Model total capacity:

```
maximum pipeline connections
=
processes × pool max
```

Then include other services using the same database.

---

# 24. External Connection Poolers

An external pooler sits between applications and the database.

```
pipeline processes
       ↓
application pools
       ↓
external pooler
       ↓
database
```

A common PostgreSQL example is PgBouncer.

External pooling can be useful when many application processes would otherwise create too many physical database connections.

But it introduces another operational layer.

Understand:

- session pooling
- transaction pooling
- statement pooling
- prepared statement compatibility
- session state
- transaction semantics

Do not add an external pooler merely because connection pooling sounds useful.

---

# 25. Connection Pooling and Prepared Statements

Some database drivers maintain prepared statement state on a connection.

With pooling, the same physical connection may be reused by different logical workers.

With external transaction-level pooling, a logical application session may move between physical connections.

This can affect prepared statements and session state.

The lesson:

> **Connection pooling changes the lifetime and ownership model of database sessions.**

Always verify driver and pooler compatibility before changing pooling modes.

---

# 26. Connection Leaks

A connection leak looks like:

```
pool max = 10

time 1 → 2 checked out
time 2 → 5 checked out
time 3 → 8 checked out
time 4 → 10 checked out
time 5 → 10 checked out forever
```

New workers now wait indefinitely or time out.

Common causes:

- missing context manager
- exception path does not release
- generator keeps connection alive
- streaming result not closed
- transaction never completed
- cancellation interrupts cleanup

Test failure paths, not only successful queries.

---

# 27. Cancellation and Cleanup

Pipelines can be stopped by:

- deployment
- orchestration timeout
- SIGTERM
- manual cancellation
- worker crash

Cleanup should be designed explicitly.

```
shutdown signal
      ↓
stop accepting new work
      ↓
finish/cancel active operations
      ↓
rollback unfinished transactions
      ↓
return/close connections
      ↓
close pool
```

A graceful shutdown reduces stale sessions and incomplete transactions.

---

# 28. Pool Shutdown

Application shutdown should close the pool.

Example:

```python
pool.close()
pool.wait()
```

The exact API depends on the library version.

The important lifecycle is:

```
START APPLICATION
       ↓
CREATE POOL
       ↓
RUN PIPELINE
       ↓
STOP WORK
       ↓
CLOSE POOL
       ↓
EXIT
```

Do not create a new pool for every batch.

Create it at the appropriate application/pipeline lifecycle boundary.

---

# 29. Configuration

Keep connection settings separate from code.

Example:

```
DATABASE_HOST
DATABASE_PORT
DATABASE_NAME
DATABASE_USER
DATABASE_PASSWORD
DATABASE_POOL_MIN
DATABASE_POOL_MAX
DATABASE_CONNECTION_TIMEOUT
DATABASE_POOL_ACQUIRE_TIMEOUT
```

Secrets should come from a secret-management mechanism or protected environment configuration.

Never log:

```
postgresql://user:password@host/database
```

Redact credentials in diagnostics.

---

# 30. A Production-Oriented Configuration

Example:

```python
from psycopg_pool import ConnectionPool
import os

pool = ConnectionPool(
    conninfo=(
        f"host={os.environ['DATABASE_HOST']} "
        f"port={os.environ.get('DATABASE_PORT', '5432')} "
        f"dbname={os.environ['DATABASE_NAME']} "
        f"user={os.environ['DATABASE_USER']} "
        f"password={os.environ['DATABASE_PASSWORD']}"
    ),
    min_size=int(os.environ.get("DATABASE_POOL_MIN", "2")),
    max_size=int(os.environ.get("DATABASE_POOL_MAX", "6")),
    open=True,
)
```

Do not copy these numbers blindly.

The values are examples. Measure the actual source workload and deployment topology.

---

# 31. Testing

## Unit tests

Test:

- pool configuration
- acquisition timeout handling
- release on success
- release on exception
- transaction rollback
- pool shutdown
- retry after stale connection
- maximum concurrency

Example concurrency test concept:

```
pool max = 2
workers = 5

expected:
2 active connections
3 waiting
```

---

# 32. Integration Test: Connection Reuse

Run several queries through the pool.

Observe the database session identifiers where supported.

Expected behavior:

```
query 1 → connection A
query 2 → connection B
query 3 → connection A
```

The exact reuse order is implementation-dependent.

The important property is controlled reuse rather than unbounded connection creation.

---

# 33. Integration Test: Connection Leak

Intentionally create an exception after acquiring a connection.

Verify:

```
before test:
pool available = N

after failed operation:
pool available = N
```

If available connections decrease permanently, there is a leak.

---

# 34. Integration Test: Pool Exhaustion

Configure:

```
pool max = 2
```

Start three or more long-running operations.

Verify:

- two acquire connections
- additional workers wait
- acquisition timeout eventually fires if configured
- no connections leak after completion

---

# 35. Integration Test: Database Restart

While a pipeline is running:

1. stop/restart the database
2. allow pooled connections to become invalid
3. observe the next operation
4. verify stale connections are discarded/recreated
5. verify retry policy does not duplicate unsafe work

Do not assume every connection error is safely retryable.

---

# 36. Intentional Failure Drills

## Drill 1 — Leak a connection

Deliberately omit release in a test-only path.

Expected:

```
pool available decreases
        ↓
pool exhaustion
        ↓
acquisition timeout
```

Then fix the lifecycle and confirm recovery.

## Drill 2 — Hold all connections

Run long queries equal to pool maximum.

Expected:

```
additional workers wait
```

Confirm the database connection count stays within the configured budget.

## Drill 3 — Kill database

Stop PostgreSQL while workers are active.

Observe:

- connection failures
- pool recovery
- retry classification
- transaction rollback
- eventual successful reconnection

## Drill 4 — Leave transaction open

In a controlled test, hold a transaction open.

Observe:

- active/idle-in-transaction session
- transaction age
- downstream impact

Then terminate it safely.

## Drill 5 — Increase workers without changing pool

Increase worker count sharply.

Verify:

- database connections remain bounded
- pool wait time increases
- throughput does not necessarily increase linearly

---

# 37. Observability

A connection pool should be observable as a resource.

## Metrics

Useful metrics include:

```
db_pool_size
db_pool_available_connections
db_pool_checked_out_connections
db_pool_waiting_workers
db_pool_acquisition_wait_seconds
db_pool_acquisition_timeouts_total
db_pool_connection_errors_total
db_pool_connection_creations_total
db_pool_connection_closes_total
db_query_duration_seconds
db_transaction_duration_seconds
```

Exact metric names depend on the instrumentation system.

## Logs

Record:

- database logical name
- pool configuration
- acquisition timeout
- query classification
- transaction duration
- connection errors
- pool exhaustion
- graceful shutdown

Never log passwords or complete DSNs containing credentials.

---

# 38. Operational Signals

A useful diagnostic relationship is:

```
pool wait ↑
      +
query duration ↑
      ↓
database may be saturated
```

Another:

```
pool wait ↑
      +
query duration normal
      ↓
pool may simply be too small
```

Another:

```
pool available ↓ permanently
      ↓
possible connection leak
```

And:

```
database connections ↑ unexpectedly
      ↓
check process count × pool size
```

Pool metrics must be interpreted together with database metrics.

---

# 39. Recovery

## Pool exhaustion

1. Check active workers.
2. Check checked-out connections.
3. Check acquisition wait time.
4. Check query duration.
5. Check transaction duration.
6. Look for leaked connections.
7. Check database CPU/I/O.
8. Check database connection limits.
9. Reduce worker concurrency if source pressure is high.
10. Increase pool only when database capacity supports it.

Do not immediately increase the pool.

---

# 40. Recovery After a Connection Failure

For a failed operation:

```
connection error
      ↓
classify
      ↓
transaction state?
      ↓
rollback/release
      ↓
obtain healthy connection
      ↓
retry only if operation is safe
```

For reads, retry is often simpler.

For writes, determine whether the previous operation committed before retrying.

An ambiguous database outcome must not automatically become a duplicate write.

---

# 41. Connection Pooling and Idempotency

Pooling does not provide idempotency.

Suppose:

```
worker
 ↓
INSERT
 ↓
connection failure before response
```

The database may have committed the insert even though the client did not receive confirmation.

Retrying may create a duplicate.

Therefore:

```
connection pooling
+
idempotent destination writes
+
safe retry policy
```

are separate mechanisms that must work together.

---

# 42. Production Runbook

## Symptom: Too many database connections

Check:

1. number of pipeline processes
2. pool maximum per process
3. other application pools
4. database connection limit
5. external poolers
6. orphaned sessions

Calculate:

```
maximum pipeline connections
=
processes × pool max
```

Then compare it with the actual database budget.

### Do not

- blindly increase \`max_connections\`
- give every worker its own connection
- assume idle connections are free
- ignore other applications

---

## Symptom: Pool acquisition timeout

Check:

1. available connections
2. checked-out connections
3. query duration
4. transaction duration
5. connection leaks
6. worker concurrency

If queries are slow, optimize the workload before simply increasing the pool.

---

## Symptom: Database is overloaded after increasing workers

Check:

- CPU
- I/O
- locks
- query latency
- active connections
- pool wait
- source replication lag

Reduce concurrency if the database is the bottleneck.

More workers are not automatically better.

---

## Symptom: Connections become invalid

Check:

- database restarts
- network infrastructure
- idle connection timeouts
- connection lifetime
- pool health checks

Discard stale connections and allow controlled recreation.

---

## Symptom: \`idle in transaction\`

Check:

- exception paths
- transaction context managers
- streaming operations
- worker cancellation
- connection release

Rollback unfinished transactions before returning connections to the pool.

---

# 43. Common Mistakes

### 1. One connection per worker forever

This can exhaust the source database.

### 2. Pool size equal to worker count by default

Worker concurrency and database capacity are different resources.

### 3. Ignoring multiple processes

Each process can have its own pool.

### 4. Returning connections with open transactions

This can create locks and MVCC pressure.

### 5. Treating a pool as a retry mechanism

A pool manages connections. It does not determine whether a failed operation is safe to retry.

### 6. Retrying writes after ambiguous connection failures

The write may already have committed.

### 7. Ignoring session state

Pooled connections can retain database session configuration.

### 8. Holding connections while doing non-database work

Do not acquire a connection and then perform unrelated CPU/network work for minutes.

### 9. Using a huge pool to fix slow queries

The database may become even more overloaded.

### 10. Creating a pool per batch

This defeats connection reuse.

### 11. Forgetting graceful shutdown

Processes can leave incomplete transactions or connections behind.

### 12. Logging credentials

Never expose database passwords in logs or error messages.

---

# 44. Production Tools You Should Know

### 1. Psycopg

Python PostgreSQL driver with connection-pooling support. Learn connection lifecycle, transactions, cursors, and pool behavior.

### 2. PgBouncer

External PostgreSQL connection pooler. Understand why transaction/session pooling changes connection and session semantics.

### 3. SQLAlchemy

Common Python database toolkit with engine-level connection pooling. Learn the distinction between an engine/pool and individual database connections.

These tools implement production mechanisms, but the underlying concepts should remain understandable without them.

---

# 45. Definition of Done

You understand this recipe when you can independently:

- explain why database connections are expensive resources
- build a reusable connection pool
- choose a bounded pool size
- calculate connection capacity across processes
- distinguish worker concurrency from database concurrency
- implement safe connection acquisition and release
- handle acquisition timeouts
- detect connection leaks
- manage transaction lifecycle
- handle stale connections
- reason about connection lifetime
- protect the source database from connection storms
- understand pooling with streaming extraction
- understand pooling with consistent snapshots
- design graceful shutdown
- instrument pool health
- recover from pool exhaustion
- distinguish connection failure from query failure
- reason about ambiguous database outcomes
- test the pool under failure

---

# 46. What You Learned

> **Database connection pooling is controlled reuse of a scarce database resource. The goal is not to create as many connections as possible; it is to provide enough controlled concurrency to meet pipeline throughput requirements without overwhelming the source database.**

The practical mental model is:

```
PIPELINE WORKERS
       ↓
BOUNDED CONNECTION POOL
       ↓
CONTROLLED DATABASE CONCURRENCY
       ↓
SOURCE DATABASE
       ↓
OBSERVE
       ↓
TUNE
```

The key question is:

> **What is the safe database connection budget for this pipeline across all workers and processes, and can I prove the pipeline stays within it?**
