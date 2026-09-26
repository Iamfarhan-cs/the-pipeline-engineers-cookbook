# E42 — Consistent Database Snapshots

## 1. Problem Recognition

### The production problem

A database is changing while a data pipeline is trying to read it.

If the pipeline executes many independent queries without a shared consistency boundary, it can build a dataset that never actually existed in the source at one point in time.

Example:

```text
10:00:00  read customers
10:00:01  application updates account
10:00:02  read accounts
10:00:03  application commits another change
10:00:04  read transactions
```

The resulting extract may contain:

- customers from one source state
- accounts from another
- transactions from a third

The pipeline may still report success because every query succeeded.

A **consistent database snapshot** solves the problem by making multiple reads observe one defined logical source state.

### How to recognize the problem

You need consistent snapshots when:

- extracting multiple related tables
- initializing a warehouse from a transactional database
- taking a backup-like analytical baseline
- building a snapshot before CDC
- running parallel extraction workers
- reconciling related tables
- foreign-key relationships must represent one source state
- a source is changing continuously during extraction

The central question is:

> **Can every piece of data in this snapshot be explained as belonging to the same source consistency boundary?**

---

# 2. Concept and Reasoning

## 2.1 What does consistent mean?

A consistent snapshot represents a source state that satisfies the database's snapshot semantics at a defined point or logical boundary.

It does **not** necessarily mean:

- no writes happen during extraction
- every query executes at exactly the same wall-clock time
- the source is globally frozen
- the destination is transactionally identical to the source during the entire extraction

Instead, it means the reads have a defined relationship to committed source state.

---

# 3. Why Independent Queries Can Produce Impossible State

Suppose the source has:

```text
customer 42
account 100 → customer 42
```

The pipeline first reads `customers`.

Then the application deletes customer 42 and its account in a transaction.

If the pipeline reads the tables independently without a shared snapshot, it may obtain:

```text
customers: customer 42 exists
accounts: account 100 does not exist
```

That state might be valid.

But another interleaving could produce:

```text
customers: customer 42 does not exist
accounts: account 100 exists
```

if the two queries observe different committed states.

The point is not that every mismatch is invalid. The point is that **independent reads do not automatically represent one source state**.

---

# 4. Consistency vs Completeness

These are different properties.

| Property | Question |
|---|---|
| Consistency | Do reads represent one defined source state? |
| Completeness | Did we capture everything required from that state? |
| Correctness | Are values mapped and transformed correctly? |
| Durability | Is the resulting snapshot safely persisted? |

A snapshot can be consistent but incomplete.

It can also be complete according to row counts but inconsistent.

You need all relevant properties.

---

# 5. Isolation Levels

Database isolation determines what a transaction can see while other transactions are running.

Common levels include:

- Read Uncommitted
- Read Committed
- Repeatable Read
- Serializable

For snapshot extraction, the important question is not simply:

> Which isolation level is strongest?

The question is:

> **Which isolation semantics provide the required consistent view without imposing unacceptable source-side cost?**

A stronger isolation level can create more contention or operational overhead than necessary.

---

# 6. PostgreSQL and MVCC

PostgreSQL uses MVCC (Multi-Version Concurrency Control).

Conceptually:

```text
transaction A writes row version 2
transaction B reads
        ↓
B sees a version according to its snapshot/isolation rules
```

This allows readers and writers to operate concurrently while maintaining defined visibility semantics.

For snapshot extraction, PostgreSQL's transaction isolation can provide a stable logical view across multiple queries in the same transaction.

---

# 7. Read Committed vs Repeatable Read

Consider two queries in one extraction transaction:

```sql
SELECT COUNT(*) FROM customers;
SELECT COUNT(*) FROM accounts;
```

Under some isolation semantics, each statement can obtain a different view.

Under PostgreSQL `REPEATABLE READ`, the transaction uses a stable snapshot for its reads.

Conceptually:

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT ... customers ...;
SELECT ... accounts ...;
SELECT ... transactions ...;

COMMIT;
```

This is useful when multiple queries must agree on one source state.

---

# 8. A Critical Operational Warning

A consistent snapshot does not mean that holding one transaction open for hours is automatically safe.

A long-lived snapshot can interact with:

- MVCC version retention
- vacuum
- table bloat
- transaction ID age
- replication behavior
- locks depending on the operations used
- connection pool capacity

Therefore:

> **Consistency must be designed together with source operational safety.**

The technically strongest consistency model can still be the wrong production design if it damages the source system.

---

# 9. Snapshot Boundary vs Extraction Duration

Suppose a snapshot starts at 10:00 and takes two hours.

The source continues changing during those two hours.

A consistent snapshot can still represent:

```text
source state at snapshot boundary
```

rather than:

```text
source state continuously changing from 10:00–12:00
```

This is exactly why consistency semantics matter.

The extraction duration does not redefine the snapshot boundary.

---

# 10. Build a Consistent Snapshot from Scratch

The implementation below uses PostgreSQL concepts because they make the mechanism easy to observe.

## 10.1 Example source tables

```sql
CREATE TABLE customers (
    customer_id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    status TEXT NOT NULL
);

CREATE TABLE accounts (
    account_id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(customer_id),
    balance NUMERIC NOT NULL
);

CREATE TABLE transactions (
    transaction_id BIGINT PRIMARY KEY,
    account_id BIGINT NOT NULL REFERENCES accounts(account_id),
    amount NUMERIC NOT NULL
);
```

The relationships make inconsistent cross-table reads easier to detect.

---

# 11. Establish One Snapshot Transaction

Open one transaction with the required isolation semantics:

```python
import psycopg


def extract_consistent_snapshot(dsn: str):
    with psycopg.connect(dsn) as conn:
        conn.execute("BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ")

        try:
            customers = conn.execute(
                """
                SELECT customer_id, name, status
                FROM customers
                ORDER BY customer_id
                """
            ).fetchall()

            accounts = conn.execute(
                """
                SELECT account_id, customer_id, balance
                FROM accounts
                ORDER BY account_id
                """
            ).fetchall()

            transactions = conn.execute(
                """
                SELECT transaction_id, account_id, amount
                FROM transactions
                ORDER BY transaction_id
                """
            ).fetchall()

            conn.commit()
            return customers, accounts, transactions

        except Exception:
            conn.rollback()
            raise
```

The important mechanism is not the Python syntax. It is the single transaction and its snapshot semantics.

For very large datasets, do not copy millions of rows into application memory as shown here. The example is intentionally small so the consistency mechanism is visible.

---

# 12. Why One Transaction Matters

The three reads occur under one snapshot boundary:

```text
BEGIN REPEATABLE READ
       ↓
customers read
       ↓
accounts read
       ↓
transactions read
       ↓
COMMIT
```

A source transaction committed after the snapshot boundary is not silently mixed into one table while excluded from another.

This gives the extractor a meaningful statement:

> These three datasets were read according to the same database snapshot semantics.

---

# 13. Snapshot Export for Parallel Workers

One transaction is not always enough for a large snapshot because one worker may not provide sufficient throughput.

A database may support exporting a snapshot so multiple workers can read the same logical source state.

Conceptually:

```text
Coordinator transaction
        ↓
CREATE SNAPSHOT
        ↓
EXPORT SNAPSHOT ID
        ↓
       / | \
      /  |  \
Worker A B  C
  ↓    ↓    ↓
Same logical source state
```

The workers must import/use the snapshot according to the database's rules.

Do not invent a generic snapshot-export implementation. Snapshot APIs are database-specific.

---

# 14. PostgreSQL Snapshot Export Concept

PostgreSQL can expose a transaction snapshot for use by other transactions under the appropriate transaction and privilege rules.

The conceptual flow is:

```text
BEGIN
   ↓
ESTABLISH SNAPSHOT
   ↓
EXPORT SNAPSHOT ID
   ↓
WORKERS START THEIR TRANSACTIONS
   ↓
IMPORT SAME SNAPSHOT
   ↓
READ DIFFERENT DATA RANGES
```

This allows multiple workers to read different tables or key ranges while maintaining one logical snapshot boundary.

The exact SQL function and lifecycle must follow the PostgreSQL version and snapshot-export rules in use.

---

# 15. Parallel Snapshot Architecture

A production architecture may look like:

```text
                 SNAPSHOT COORDINATOR
                         ↓
                 SNAPSHOT IDENTIFIER
                         ↓
       ┌─────────────────┼─────────────────┐
       ↓                 ↓                 ↓
  worker A           worker B           worker C
 customers           accounts          transactions
       ↓                 ↓                 ↓
       └─────────────────┼─────────────────┘
                         ↓
                    STAGING
                         ↓
                    VALIDATE
                         ↓
                    PUBLISH
```

The workers can operate concurrently without turning the snapshot into a collection of unrelated point-in-time reads.

---

# 16. Snapshot and Foreign Keys

Consistent snapshots are especially useful when related tables have foreign keys.

Suppose:

```text
customers
   ↓
accounts
   ↓
transactions
```

If all three tables are read from one consistent snapshot, the extracted relationship represents one source state.

This simplifies downstream validation:

```text
account.customer_id
       ↓
customer.customer_id
```

A referential-integrity failure then becomes more meaningful because it is less likely to be caused merely by different read times.

---

# 17. Source Writes During a Consistent Snapshot

A consistent snapshot does not stop normal source writes.

Suppose:

```text
10:00 snapshot established
10:01 customer updated
10:02 account inserted
```

The snapshot's visibility rules determine whether those changes appear in the snapshot.

The extractor should not try to manually decide this with application timestamps.

The database snapshot is the authority for the defined consistency boundary.

---

# 18. Snapshot Visibility and Uncommitted Data

A production snapshot should normally represent committed source state according to the chosen isolation semantics.

Do not build a snapshot that accidentally reads uncommitted data merely to improve throughput.

Dirty reads can create values that never become committed source state.

The snapshot's contract should explicitly state:

```text
Committed source state
+ database-defined visibility
+ defined snapshot boundary
```

---

# 19. Large Tables and Streaming

A consistent snapshot does not require loading the entire result into memory.

Use:

- server-side cursors where appropriate
- bounded fetch sizes
- streaming result consumption
- staged writes
- deterministic ordering

Conceptually:

```text
ONE SNAPSHOT
     ↓
READ BATCH
     ↓
WRITE STAGING
     ↓
READ NEXT BATCH
     ↓
WRITE STAGING
     ↓
...
```

The snapshot boundary remains stable even though batches are processed sequentially.

---

# 20. Transaction Lifetime Trade-Off

For a large snapshot, one long transaction may create operational pressure.

Possible alternatives include:

### Option A — One transaction

Simple consistency model, potentially long transaction.

### Option B — Exported snapshot + multiple workers

Shared logical state with parallel reads, if the database supports it.

### Option C — Database-native snapshot/export mechanism

Use the database's supported snapshot facility.

### Option D — Source replica

Take the snapshot from a replica when its consistency and lag are acceptable.

There is no universally correct option. Choose based on:

- dataset size
- extraction duration
- source workload
- database capabilities
- acceptable staleness
- operational constraints

---

# 21. Replica Snapshots

A read replica can protect the primary from heavy extraction workloads.

But it introduces another consistency dimension:

```text
PRIMARY
   ↓
replication
   ↓
REPLICA
   ↓
SNAPSHOT
```

If the replica is behind, the snapshot represents the replica's committed state, not the primary's latest state.

Record:

- replica identity
- replication lag
- snapshot start position/time
- consistency assumptions

Do not call a lagging replica snapshot "current primary state".

---

# 22. Consistency Across Multiple Tables

A common mistake is using separate transactions:

```text
transaction A → customers
transaction B → accounts
transaction C → transactions
```

Even if every transaction uses `REPEATABLE READ`, they can represent different source snapshots.

If the tables must describe one state, they need a shared snapshot boundary.

This is especially important for:

- dimensions + facts
- parent + child tables
- account + ledger data
- customer + transaction data
- reference + event data

---

# 23. Consistency Across Multiple Databases

A database snapshot normally applies to one database consistency domain.

Suppose a pipeline reads:

```text
PostgreSQL A
      +
PostgreSQL B
      ↓
One destination snapshot
```

There may be no single native snapshot spanning both systems.

You then need an explicit cross-system consistency strategy such as:

- common business timestamp where valid
- source transaction markers
- coordinated capture positions
- CDC positions
- reconciliation
- accepting bounded inconsistency

Do not claim a globally consistent snapshot when the sources cannot provide one.

---

# 24. Snapshot + CDC Handoff

A consistent snapshot is often the first half of a CDC pipeline.

The desired sequence is:

```text
CAPTURE CDC POSITION P
        ↓
ESTABLISH SNAPSHOT S
        ↓
READ SNAPSHOT S
        ↓
APPLY CDC FROM P
        ↓
CATCH UP
        ↓
CONTINUE STREAMING
```

The snapshot and CDC streams may overlap.

This is usually safer than trying to create a tiny gap-free handoff based on wall-clock timestamps.

The destination must be idempotent so an overlapping change does not create an incorrect business result.

---

# 25. Failure Scenario: Snapshot Worker Crashes

Suppose:

```text
Worker A → customers complete
Worker B → accounts 70%
Worker C → transactions complete
```

Worker B crashes.

Do not publish the snapshot as complete.

Instead:

```text
snapshot = INCOMPLETE
```

Then either:

- retry the failed range using the same valid snapshot boundary
- restart the snapshot
- create a new snapshot if the old boundary cannot safely be reused

The recovery strategy must preserve the meaning of the snapshot.

---

# 26. Failure Scenario: Source Connection Drops

A network failure does not automatically invalidate the snapshot boundary.

Ask:

1. Is the transaction still alive?
2. Can the same snapshot be resumed?
3. Did the database roll back the transaction?
4. Can the exported snapshot still be imported?
5. Is the source position still valid?

If the answer is uncertain, restart from a new known-consistent snapshot rather than pretending the old state is still valid.

---

# 27. Failure Scenario: Source Transaction ID Age

Long-running snapshots can prevent old row versions from being reclaimed efficiently.

Monitor database-specific signals such as:

- transaction age
- vacuum progress
- table bloat
- replica impact
- log retention

A snapshot that is logically correct but operationally damaging is not production-safe.

---

# 28. Snapshot Publication

Build the snapshot separately from the current destination state.

A safe publication flow is:

```text
SOURCE SNAPSHOT
      ↓
STAGING SNAPSHOT
      ↓
VALIDATION
      ↓
COMPLETION MARKER
      ↓
PUBLISH / SWAP
      ↓
CURRENT DATA
```

If validation fails, the current destination remains unchanged.

This separates extraction correctness from publication correctness.

---

# 29. Snapshot Metadata

Store enough information to reproduce the reasoning behind the snapshot.

Example:

```sql
CREATE TABLE snapshot_metadata (
    snapshot_id UUID PRIMARY KEY,
    source_name TEXT NOT NULL,
    consistency_model TEXT NOT NULL,
    source_snapshot_id TEXT,
    source_position TEXT,
    started_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    status TEXT NOT NULL,
    row_count BIGINT,
    notes TEXT
);
```

Avoid storing sensitive payloads in metadata. Record identifiers and operational facts instead.

---

# 30. Testing

## Essential tests

| Test | Expected result |
|---|---|
| Multiple tables | All read from one snapshot boundary |
| Source write during extraction | Snapshot follows defined visibility semantics |
| Concurrent source transaction | Uncommitted data is not accidentally included |
| Worker failure | Snapshot remains unpublished |
| Connection failure | Recovery preserves or deliberately recreates boundary |
| Parallel workers | Same logical snapshot is observed |
| Replica snapshot | Lag and source identity recorded |
| Foreign-key relationships | Consistent parent/child state |
| Validation failure | Publication blocked |
| Cross-database extraction | Consistency limitations explicitly handled |
| Snapshot + CDC | No unowned change interval |
| Long-running snapshot | Operational safeguards trigger |

---

# 31. Intentional Failure Drills

## Drill 1 — Modify source during extraction

Start a snapshot and concurrently commit source changes.

Verify that all snapshot queries obey the chosen visibility semantics.

## Drill 2 — Kill one parallel worker

Expected:

```text
snapshot incomplete
publication blocked
```

Recover only if the original snapshot boundary remains valid.

## Drill 3 — Force connection loss

Determine whether the snapshot transaction survives. If not, verify that the pipeline creates a new consistent boundary rather than mixing old and new snapshots.

## Drill 4 — Run an intentionally long snapshot

Observe:

- transaction age
- vacuum behavior
- table bloat indicators
- source load
- replica impact

## Drill 5 — Break snapshot/CDC handoff

Verify that the test detects missing or duplicated changes rather than relying on timestamps alone.

---

# 32. Observability

## Logs

Record:

- snapshot ID
- database/source identity
- isolation level
- source snapshot identifier where available
- worker ID
- partition/range
- start/end times
- rows processed
- publication status
- recovery events

## Metrics

Useful metrics include:

```text
snapshot_active_seconds
snapshot_rows_processed_total
snapshot_batch_duration_seconds
snapshot_worker_failures_total
snapshot_publication_failures_total
snapshot_source_replication_lag
snapshot_transaction_age
snapshot_retries_total
```

## Alerts

Alert when:

- snapshot duration exceeds the expected window
- transaction age becomes unsafe
- source replication lag increases materially
- worker failures repeat
- snapshot publication is blocked
- CDC handoff is approaching retention limits

---

# 33. Reconciliation

After the snapshot is published, reconcile using the **same source consistency boundary** whenever possible.

Possible checks:

```text
source count at snapshot boundary
        vs
snapshot count
```

and:

```text
source control total
        vs
snapshot control total
```

For related tables, verify relationships:

```text
accounts.customer_id
      ↓
customers.customer_id
```

A current source query performed hours later is not necessarily an appropriate comparison for a historical snapshot.

---

# 34. Common Mistakes

### 1. Assuming one query means one snapshot

A pipeline with many independent queries can observe different source states.

### 2. Using separate transactions for related tables

`REPEATABLE READ` in three different transactions does not create one shared snapshot.

### 3. Choosing the strongest isolation level automatically

Stronger isolation can impose unnecessary source cost.

### 4. Holding a huge transaction indefinitely

Long-lived snapshots can create MVCC and operational pressure.

### 5. Assuming replicas are current

Replica lag changes the snapshot's actual source state.

### 6. Ignoring parallel-worker consistency

Parallel reads can become unrelated snapshots without shared snapshot semantics.

### 7. Publishing before all workers complete

Consumers can see partial data.

### 8. Treating cross-database reads as one transaction

Independent databases normally do not share one native snapshot.

### 9. Using wall-clock timestamps as a substitute for snapshot semantics

Application time is not automatically a database consistency boundary.

### 10. Ignoring snapshot + CDC overlap

The handoff must be designed for overlap and replay.

---

# 35. Production Tools You Should Know

### 1. PostgreSQL

Learn MVCC, isolation levels, repeatable reads, exported snapshots, and source-side operational effects.

### 2. Debezium

Recognize how production CDC systems coordinate initial snapshots with ongoing change capture and source positions.

### 3. Apache Spark

Recognize distributed snapshot reads and the additional consistency/parallelism considerations when extracting very large datasets.

These are production tools to recognize. The underlying snapshot mechanism must remain understandable without them.

---

# 36. Production Runbook

## Symptom: Related tables do not reconcile

Check:

1. Were they read in the same transaction?
2. Was the same snapshot boundary used?
3. Did parallel workers share a snapshot?
4. Was a replica used?
5. Was the reconciliation performed against the same source state?

Do not immediately modify the destination. First determine whether the inconsistency came from the extraction boundary.

## Symptom: Snapshot transaction is too old

Check:

1. extraction duration
2. source write rate
3. vacuum/version-retention indicators
4. batch throughput
5. source load
6. replica impact

Possible actions:

- reduce snapshot duration
- improve extraction efficiency
- use a supported snapshot-export strategy
- move extraction to a suitable replica
- restart with a better boundary

Do not keep an unsafe long-running transaction open just to preserve a plan.

## Symptom: Parallel workers disagree

1. Verify all workers used the same snapshot boundary.
2. Check worker ranges.
3. Check duplicate/overlap handling.
4. Check missing partitions.
5. Reconcile counts and control totals.
6. Rebuild if the shared snapshot guarantee was broken.

## Symptom: Snapshot + CDC produces duplicates

Overlap can be expected.

Check:

- source position
- snapshot boundary
- event identity
- destination idempotency
- final state reconciliation

Do not remove CDC events simply because they overlap the snapshot.

## What not to do

- Do not call independent queries a consistent snapshot without evidence.
- Do not use `MAX(id)` as a substitute for database snapshot semantics.
- Do not hold a long transaction open without monitoring source impact.
- Do not assume replica state equals primary state.
- Do not publish before every required worker completes.
- Do not claim cross-database consistency without a cross-system mechanism.
- Do not use wall-clock time as a substitute for a database snapshot boundary.

---

# 37. Definition of Done

You understand this recipe when you can independently:

- explain database snapshot consistency
- distinguish consistency from completeness
- choose an appropriate isolation model
- explain PostgreSQL MVCC at a practical level
- build multi-table extraction under one snapshot
- reason about snapshot export for parallel workers
- handle concurrent source writes
- protect the source from long-lived snapshot damage
- account for replica lag
- design snapshot publication safely
- design snapshot + CDC handoff
- test consistency under failure
- reconcile against the correct source boundary
- explain why multiple `REPEATABLE READ` transactions are not one shared snapshot

---

# 38. What You Learned

> **A consistent database snapshot is not simply a successful collection of queries. It is a collection of reads tied to one defined database visibility boundary. The hard part is preserving that boundary across multiple tables, workers, failures, replicas, and the eventual CDC handoff without creating unacceptable source-side operational cost.**

The practical mental model is:

```text
DEFINE CONSISTENCY REQUIREMENT
          ↓
ESTABLISH SNAPSHOT BOUNDARY
          ↓
READ ALL REQUIRED DATA FROM THAT BOUNDARY
          ↓
STAGE
          ↓
VALIDATE
          ↓
PUBLISH COMPLETE SNAPSHOT
          ↓
RECONCILE
          ↓
CONTINUE CDC / INCREMENTAL PROCESSING
```

The key question is:

> **Can I prove that all data in this snapshot belongs to the same defined source consistency boundary?**
