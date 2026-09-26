# Recipe 21 — Transactions & Atomicity

A pipeline can validate correct data and still corrupt the target if a multi-step operation fails halfway through.

For example:

```text
1. Insert payment
2. Update account balance
3. Write processing status
4. Commit
```

If step 2 succeeds and step 3 fails, the database may contain a partial result.

This recipe teaches how to recognize, implement, test, intentionally break, and recover from partial database operations using transactions and atomicity.

---

## 1. Problem Recognition

### The production problem

A pipeline often performs several related changes:

```text
Receive event
    ↓
Insert record
    ↓
Update aggregate
    ↓
Write audit state
    ↓
Mark processing complete
```

These operations may represent **one logical unit of work**.

If they are committed independently:

```text
Insert        ✓
Update        ✓
Audit         ✗
Status        not reached
```

The system is now partially updated.

### How do you recognize this problem?

Look for:

- multiple database writes for one event
- several related tables updated by one worker
- commits between related operations
- retry logic around individual writes
- inconsistent state after worker crashes
- records marked completed while related writes are missing
- balance/aggregate tables disagreeing with event tables
- recovery procedures that require manual database repair

Ask:

> If the process crashes after this statement but before the next one, is the database still correct?

If the answer is no, you need an explicit transaction boundary.

---

## 2. Concept and Reasoning

### Atomicity

Atomicity means a logical operation is treated as one indivisible unit:

```text
ALL changes succeed
       OR
NO changes are committed
```

Conceptually:

```text
BEGIN
  ↓
change A
  ↓
change B
  ↓
change C
  ↓
COMMIT
```

If something fails:

```text
BEGIN
  ↓
change A
  ↓
change B
  ↓
ERROR
  ↓
ROLLBACK
```

The database returns to the state before the transaction.

### Transaction vs retry

A transaction protects a group of database changes.

A retry determines what happens after a failure.

They solve different problems:

```text
Transaction
    ↓
prevents partial commit

Retry
    ↓
attempts the operation again
```

Retries without transactions can repeat partial work.

Transactions without appropriate retry handling can still leave the pipeline unable to make progress.

### Choose the transaction boundary around the logical unit of work

Do not automatically put an entire pipeline run into one transaction.

Prefer:

```text
one logical unit of work
        ↓
one appropriate transaction
```

For example, processing one payment event may require:

```text
event insert
+ ledger update
+ processing status
```

These may belong in one transaction.

A million unrelated events may not.

---

## 3. Implementation

We will build a small PostgreSQL transaction example.

### 3.1 Example schema

```sql
CREATE TABLE payments (
    event_id TEXT PRIMARY KEY,
    amount NUMERIC NOT NULL,
    status TEXT NOT NULL
);

CREATE TABLE account_balances (
    account_id TEXT PRIMARY KEY,
    balance NUMERIC NOT NULL
);

CREATE TABLE processing_audit (
    event_id TEXT PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

The logical operation is:

```text
payment insert
    +
balance update
    +
audit insert
```

These changes should succeed or fail together.

### 3.2 Unsafe implementation

An unsafe approach can look like:

```python
insert_payment()
connection.commit()

update_balance()
connection.commit()

write_audit()
connection.commit()
```

If `write_audit()` fails, the first two changes remain committed.

That is partial state.

### 3.3 Transactional implementation

Use one transaction:

```python
with connection:
    with connection.cursor() as cursor:
        cursor.execute(
            """
            INSERT INTO payments (event_id, amount, status)
            VALUES (%s, %s, %s)
            """,
            (event_id, amount, "completed"),
        )

        cursor.execute(
            """
            UPDATE account_balances
            SET balance = balance + %s
            WHERE account_id = %s
            """,
            (amount, account_id),
        )

        cursor.execute(
            """
            INSERT INTO processing_audit (event_id)
            VALUES (%s)
            """,
            (event_id,),
        )
```

With PostgreSQL transaction handling, leaving the context successfully commits the transaction; an exception causes rollback.

The exact transaction API depends on the driver.

### 3.4 Make the unit of work explicit

A better design is to expose one operation:

```python
def process_payment(connection, event_id, account_id, amount):
    with connection:
        with connection.cursor() as cursor:
            cursor.execute(
                """
                INSERT INTO payments (event_id, amount, status)
                VALUES (%s, %s, %s)
                """,
                (event_id, amount, "completed"),
            )

            cursor.execute(
                """
                UPDATE account_balances
                SET balance = balance + %s
                WHERE account_id = %s
                """,
                (amount, account_id),
            )

            cursor.execute(
                """
                INSERT INTO processing_audit (event_id)
                VALUES (%s)
                """,
                (event_id,),
            )
```

The function represents one logical database operation.

---

## 4. Testing

Test both successful commit and rollback.

### Test 1 — All operations succeed

Expected:

```text
payments        → row exists
balance         → updated
audit           → row exists
```

### Test 2 — Balance update fails

Expected:

```text
payments        → no committed row
balance         → unchanged
audit           → no row
```

### Test 3 — Audit insert fails

Expected:

```text
payments        → no committed row
balance         → unchanged
audit           → no committed row
```

### Test 4 — Transaction can be retried safely

Combine the transaction with Recipe 6's idempotency mechanism.

The retry should not create duplicate logical results.

### Test 5 — Concurrent processing

Two workers processing the same logical event should not corrupt the target.

Use:

- stable event identity
- database constraints
- appropriate transaction isolation
- idempotent writes

Do not assume a transaction alone solves duplicate processing.

---

## 5. Observability

Track transaction behavior as an operational signal.

Useful metrics include:

```text
transactions_total
transactions_committed_total
transactions_rolled_back_total
transaction_duration_seconds
transaction_failures_total
```

Useful logs:

```text
transaction_started
transaction_committed
transaction_rolled_back
transaction_failed
```

Include safe identifiers such as:

- event ID
- operation type
- transaction duration
- failure class
- pipeline version

Do not log credentials or full sensitive payloads.

### Long transactions

Watch for transactions that remain open too long.

Long transactions can:

- hold locks
- increase contention
- delay vacuum cleanup
- increase resource usage
- block other work

A transaction that is technically correct can still be operationally dangerous if it is too large.

---

## 6. Intentional Failure

Break the transaction deliberately.

### Failure drill

Place an intentional exception after the payment insert:

```python
cursor.execute(
    """
    INSERT INTO payments (event_id, amount, status)
    VALUES (%s, %s, %s)
    """,
    (event_id, amount, "completed"),
)

raise RuntimeError("intentional transaction failure")
```

Run the operation.

Then query the database.

Expected:

```text
payment row      → absent
balance update   → absent
audit row        → absent
```

This proves rollback is working.

### Second failure drill

Move the failure after the balance update.

Expected:

```text
payment row      → absent
balance update   → unchanged
audit row        → absent
```

The transaction should protect all three changes.

---

## 7. Recovery

After an intentional or real transaction failure:

### Step 1 — Confirm rollback

Check all tables involved in the logical operation.

### Step 2 — Identify the failed unit of work

Use a stable identifier such as:

```text
event_id
```

### Step 3 — Determine the failure type

Ask:

- database connection failure?
- constraint violation?
- serialization conflict?
- deadlock?
- application bug?
- invalid data?
- external dependency failure?

### Step 4 — Fix the root cause

Do not manually insert missing rows just to make the data appear consistent.

### Step 5 — Retry the complete logical operation

Run the transaction again.

### Step 6 — Verify atomic outcome

Confirm:

```text
payment exists
AND
balance is correct
AND
audit exists
```

### Step 7 — Reconcile

Compare the final state against the expected result.

---

## 8. Transaction Isolation

Atomicity is only one transaction property.

Isolation determines how concurrent transactions interact.

Common PostgreSQL isolation levels include:

| Level | Practical idea |
|---|---|
| **Read Committed** | Statements see committed data available when they execute. |
| **Repeatable Read** | Transaction reads use a stable snapshot. |
| **Serializable** | Database enforces behavior equivalent to serial execution, with possible serialization failures requiring retry. |

Do not choose the strictest isolation level automatically.

Higher isolation can increase contention and retries.

Choose based on the correctness requirement.

---

## 9. Deadlocks

Two transactions can wait for each other:

```text
Transaction A              Transaction B

locks row 1                locks row 2
     ↓                          ↓
waits for row 2             waits for row 1
     ↓                          ↓
       DEADLOCK
```

Databases can detect deadlocks and abort one transaction.

The application must be prepared to retry when the failure is retryable.

A useful prevention strategy is consistent lock ordering:

```text
Always lock account A
before account B
```

rather than allowing workers to acquire locks in arbitrary order.

---

## 10. Transactions and External Systems

A database transaction does not automatically include external APIs.

This is unsafe:

```text
BEGIN
  ↓
database update
  ↓
external API call
  ↓
COMMIT
```

Suppose the external API succeeds and the database transaction rolls back.

The external side effect still happened.

Or the database commits and the external call fails.

Now the systems disagree.

The database transaction only provides atomicity for resources participating in that transaction.

For external side effects, investigate patterns such as:

- idempotency keys
- transactional outbox
- retryable consumers
- reconciliation
- compensating actions

The appropriate pattern depends on the architecture.

---

## 11. Transactional Outbox

A common pattern for reliable database-to-message publication is the transactional outbox.

Instead of:

```text
update database
    ↓
publish message
```

use:

```text
BEGIN
  ↓
update business data
  ↓
insert outbox event
  ↓
COMMIT
```

A separate publisher then reads the outbox:

```text
outbox
  ↓
publisher
  ↓
message broker
```

This ensures the database state and the intent to publish are committed together.

The actual message publication still requires idempotency because publishing may be retried.

---

## 12. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **PostgreSQL** | Provides ACID transactions, isolation levels, locking, constraints, and rollback. |
| **SQLAlchemy** | Provides transaction management and database abstractions for Python applications. |
| **Apache Kafka** | Supports transactional producer/consumer patterns for specific streaming workflows; understand its transaction model separately from database transactions. |

> These tools implement production mechanisms. The goal of this recipe is to understand the transaction and atomicity principles underneath them.

---

## 13. Production Runbook

### Transaction failures increase

Check:

1. failure rate
2. database health
3. constraint violations
4. deadlocks
5. serialization failures
6. connection pool exhaustion
7. transaction duration
8. recent deployments

### If deadlocks increase

Check:

1. which tables are involved
2. lock acquisition order
3. transaction duration
4. concurrent workloads
5. recent query changes

Do not simply increase retries without investigating the locking pattern.

### If transactions become slow

Check:

- query plans
- indexes
- lock waits
- batch size
- transaction scope
- external calls accidentally placed inside transactions

Do not keep external network calls open inside database transactions unless there is a deliberate reason.

### What not to do

Do not:

- commit every statement when changes belong to one logical operation
- hold transactions open while waiting for slow external APIs
- assume transactions prevent duplicates
- use transactions as a substitute for idempotency
- increase isolation blindly
- ignore deadlock patterns
- manually repair partial state without reconciliation

---

## 14. Common Mistakes

### Mistake 1 — Commit after every statement

This defeats atomicity across related changes.

### Mistake 2 — One enormous transaction

A transaction covering an entire day's pipeline can create locks, memory pressure, and poor recovery behavior.

### Mistake 3 — Transaction as duplicate protection

Transactions do not automatically prevent repeated logical events.

Use Recipe 6's idempotency mechanism as well.

### Mistake 4 — External API inside a long transaction

A slow network call can keep database locks open unnecessarily.

### Mistake 5 — Ignore rollback testing

A transaction that has never been intentionally failed has not been properly exercised.

### Mistake 6 — Retry every database exception

Some errors are permanent.

Classify the failure before retrying.

### Mistake 7 — Ignore isolation requirements

Correctness under concurrent workers depends on more than simply using `BEGIN` and `COMMIT`.

---

## 15. Definition of Done

You are done when you can:

- identify a multi-step logical database operation
- define its transaction boundary
- implement atomic commit/rollback
- test successful commit
- test rollback after each important failure point
- observe commits, rollbacks, and transaction duration
- explain transaction isolation
- recognize deadlocks
- recover from retryable transaction failures
- distinguish transactions from idempotency
- explain why external APIs are outside normal database atomicity
- explain the transactional outbox pattern
- intentionally break a transaction
- prove that no partial state remains
- reconcile the recovered result
- operate the mechanism using a production runbook

---

## 16. What You Learned

The central principle is:

> **A logical operation should not leave a partially committed database state.**

The basic mechanism is:

```text
BEGIN
  ↓
all related changes
  ↓
validate result
  ↓
COMMIT
```

or:

```text
BEGIN
  ↓
failure
  ↓
ROLLBACK
  ↓
safe previous state
```

But transactions are only one part of reliable pipeline execution.

A production pipeline often combines:

```text
Boundary Validation
        +
Transaction / Atomicity
        +
Idempotency
        +
Retry
        +
Reconciliation
```

That combination is what turns individual database operations into reliable pipeline behavior.
