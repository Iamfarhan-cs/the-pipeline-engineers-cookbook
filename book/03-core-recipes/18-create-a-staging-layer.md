# Chapter 18 — Create a Staging Layer

Chapter 17 gave us a working ingestion pipeline:

```text
HTTP API
   ↓
Python
   ↓
PostgreSQL
```

It works, but ingestion and processing are too closely connected.

A staging layer gives us a durable boundary between data that has arrived and data that is ready for processing.

This recipe builds that boundary using the same learning project from Chapter 17.

---

## Recipe Goal

By the end of this recipe:

```text
HTTP API
   ↓
Python ingestion
   ↓
Raw data
   ↓
Staging
   ↓
Processing
   ↓
Final table
```

We will learn how to:

- preserve the incoming source record
- create a staging table
- separate ingestion from processing
- map raw JSON into structured staging columns
- process staging records into a target table
- trace a target record back to its source
- test the staging boundary
- understand the limitations of a simple staging design

---

## 1. The Problem With Direct Ingestion

Chapter 17 used:

```text
API
 ↓
posts
```

That is easy to understand, but it connects the external source directly to the final data model.

Imagine that the source sends:

```json
{
  "userId": 1,
  "id": 10,
  "title": "hello",
  "body": "world"
}
```

Later processing may need to:

- rename fields
- validate business rules
- normalize values
- enrich the record
- calculate derived fields
- join another dataset
- filter records

If all of this happens inside ingestion, one job has too many responsibilities.

A staging layer separates those responsibilities.

---

## 2. What Is a Staging Layer?

A staging layer is an intermediate storage area between ingestion and later processing.

```text
Source
  ↓
Raw / landing
  ↓
Staging
  ↓
Curated / target
```

The exact meaning of raw and staging differs between systems.

Some systems use both.

Some systems use one landing area.

Some systems store raw objects in object storage and use database tables for staging.

For this recipe:

> A staging layer is a durable intermediate representation of incoming records that can be processed independently from the original source request.

---

## 3. Why This Matters

The main benefit is separation.

```text
Ingestion
   ↓
Receive data

Staging
   ↓
Hold incoming data

Processing
   ↓
Transform data

Target
   ↓
Serve processed data
```

Suppose the source API becomes unavailable after ingestion succeeds.

With direct processing, the next attempt may need the source again.

With staging, the already received data can remain available.

That makes recovery and replay easier.

---

## 4. When to Use a Staging Layer

A staging layer is useful when:

- ingestion and processing have different responsibilities
- processing may fail and need to run again
- incoming data needs additional validation
- transformations are more than simple field copying
- you need a durable intermediate state
- replay or reprocessing matters
- several processing steps use the same incoming data

It may be unnecessary for a very small pipeline where the source and destination are simple and no intermediate recovery point is needed.

Do not add layers only because architecture diagrams contain many boxes.

Add a layer when it solves a real problem.

---

## 5. Architecture

Our learning architecture is:

```text
                 HTTP API
                    │
                    │ JSON
                    ▼
             ┌──────────────┐
             │ Python       │
             │ Ingestion    │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │ raw_posts    │
             │ source data  │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │posts_staging │
             │ structured   │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │    posts     │
             │ target data  │
             └──────────────┘
```

For this learning recipe we use:

1. raw_posts — source representation.
2. posts_staging — process-ready representation.
3. posts — processed target.

We also keep ingestion_runs from Chapter 17.

---

## 6. Before You Start

This recipe builds on Chapter 17.

You should already understand:

- Python basics
- PostgreSQL basics
- HTTP requests
- JSON
- SQL inserts
- transactions
- the Chapter 17 learning project

The recipe uses Python, PostgreSQL, Docker, requests, psycopg, and pytest.

These are generic learning-project choices, not claims about a particular production repository.

---

## 7. Repository Investigation

In an existing repository, investigate before creating a new staging table.

Search for:

- raw or landing tables
- existing staging tables
- migration files
- processing jobs
- database access modules
- status columns
- batch identifiers
- lineage fields
- database tests

Ask:

> Does the repository already have a staging concept under another name?

Also check whether staging is implemented in PostgreSQL, object storage, a message system, or a warehouse schema.

Do not create a second staging mechanism simply because the first one was not named staging.

---

## 8. Project Structure

For the standalone learning project:

```text
api-ingestion-pipeline/
├── .env.example
├── .gitignore
├── docker-compose.yml
├── requirements.txt
├── sql/
│   ├── init.sql
│   └── 002_add_staging.sql
├── src/
│   ├── __init__.py
│   ├── config.py
│   ├── db.py
│   ├── ingest.py
│   └── process.py
└── tests/
    ├── test_ingest.py
    └── test_staging.py
```

In a real project, follow its existing layout and migration convention.

---

## 9. Database Design

Create a raw table that preserves the source representation:

```sql
CREATE TABLE IF NOT EXISTS raw_posts (
    raw_id BIGSERIAL PRIMARY KEY,
    source_record_id INTEGER NOT NULL,
    payload JSONB NOT NULL,
    received_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Then create the staging table:

```sql
CREATE TABLE IF NOT EXISTS posts_staging (
    staging_id BIGSERIAL PRIMARY KEY,
    raw_id BIGINT NOT NULL REFERENCES raw_posts(raw_id),
    source_record_id INTEGER NOT NULL,
    user_id INTEGER NOT NULL,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    staged_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

The final posts table from Chapter 17 remains the target.

---

## 10. Why Keep Two Identifiers?

The source record ID comes from the external system.

The raw ID comes from our database.

Example:

```text
source_record_id = 10
raw_id           = 501
```

The source knows the record as 10.

Our database knows the stored raw row as 501.

Keeping both gives us a useful lineage path:

```text
source record
     ↓
raw_posts
     ↓
raw_id
     ↓
posts_staging
```

This becomes important when debugging duplicates, replaying records, or tracing a final record back to its original input.

---

## 11. Create the Migration

Create sql/002_add_staging.sql:

```sql
CREATE TABLE IF NOT EXISTS raw_posts (
    raw_id BIGSERIAL PRIMARY KEY,
    source_record_id INTEGER NOT NULL,
    payload JSONB NOT NULL,
    received_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS posts_staging (
    staging_id BIGSERIAL PRIMARY KEY,
    raw_id BIGINT NOT NULL REFERENCES raw_posts(raw_id),
    source_record_id INTEGER NOT NULL,
    user_id INTEGER NOT NULL,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    staged_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

For the standalone learning project:

```bash
psql "$DATABASE_URL" -f sql/002_add_staging.sql
```

In an existing repository, use its actual migration tool instead.

---

## 12. Store the Raw Record

Change ingestion so the original source record is stored first.

```python
from psycopg.types.json import Jsonb


def store_raw_record(connection, record: dict) -> int:
    row = connection.execute(
        """
        INSERT INTO raw_posts (source_record_id, payload)
        VALUES (%s, %s)
        RETURNING raw_id
        """,
        (record["id"], Jsonb(record)),
    ).fetchone()

    return row[0]
```

The function returns the internal raw ID.

That ID will be used when the record enters staging.

---

## 13. Store the Staging Record

Now map the source record into structured staging columns.

```python
def store_staging_record(connection, raw_id: int, record: dict) -> None:
    connection.execute(
        """
        INSERT INTO posts_staging (
            raw_id,
            source_record_id,
            user_id,
            title,
            body
        )
        VALUES (%s, %s, %s, %s, %s)
        """,
        (
            raw_id,
            record["id"],
            record["userId"],
            record["title"],
            record["body"],
        ),
    )
```

The flow is now:

```text
Source JSON
    ↓
raw_posts.payload
    ↓
posts_staging columns
```

Raw storage preserves the source representation.

Staging provides a structured representation for processing.

---

## 14. Raw vs Staging

| Raw | Staging |
|---|---|
| Preserves source representation | Provides structured processing fields |
| Close to source | Closer to internal schema |
| Useful for replay | Useful for processing |
| May contain source-specific fields | Contains fields needed by processing |
| Changes less frequently | Can evolve with processing requirements |

The distinction is conceptual, not universal.

Some production systems use different names or combine these responsibilities.

---

## 15. Why Store Raw Data Before Processing?

Without a durable raw boundary:

```text
API
 ↓
processing fails
 ↓
source data may no longer be available
```

With raw storage:

```text
API
 ↓
raw_posts
 ↓
processing fails
 ↓
raw record remains available
```

The stored raw record becomes a recovery point.

It can also become an input for replay later.

---

## Next Part

The next part of this chapter will connect the raw and staging layers to the processing step, then cover verification, failure scenarios, idempotency, testing, troubleshooting, and the Definition of Done.

---

## 16. Separate Ingestion From Processing

Chapter 17 combined the basic source read and database write.

Now we separate the responsibilities.

### Ingestion

```text
API
 ↓
Fetch
 ↓
Basic validation
 ↓
Store raw
 ↓
Store staging
```

### Processing

```text
Staging
 ↓
Read records
 ↓
Transform
 ↓
Write target
```

The processor no longer needs to call the external API.

That is the main architectural change in this recipe.

---

## 17. Update the Ingestion Function

**GENERIC EXAMPLE:**

```python
def ingest_records(records: list[dict]) -> int:
    staged = 0

    with get_connection() as connection:
        for raw_record in records:
            record = validate_record(raw_record)
            raw_id = store_raw_record(connection, record)
            store_staging_record(connection, raw_id, record)
            staged += 1

        connection.commit()

    return staged
```

The ingestion function now ends at staging.

It does not write to the final target table.

That makes the boundary explicit.

---

## 18. Create the Processing Function

Create src/process.py:

```python
from .db import get_connection


def process_staged_records(connection) -> int:
    rows = connection.execute(
        """
        SELECT
            staging_id,
            source_record_id,
            user_id,
            title,
            body
        FROM posts_staging
        ORDER BY staging_id
        """
    ).fetchall()

    processed = 0

    for row in rows:
        connection.execute(
            """
            INSERT INTO posts (id, user_id, title, body)
            VALUES (%s, %s, %s, %s)
            ON CONFLICT (id) DO NOTHING
            """,
            (
                row[1],
                row[2],
                row[3],
                row[4],
            ),
        )
        processed += 1

    return processed


def main() -> None:
    with get_connection() as connection:
        processed = process_staged_records(connection)
        connection.commit()

    print(f"Processed {processed} staged records")


if __name__ == "__main__":
    main()
```

This is intentionally a small processing implementation.

It does not yet track processing state.

That limitation becomes the problem for Chapter 22.

---

## 19. Run the Two-Step Pipeline

Start PostgreSQL:

```bash
docker compose up -d
```

Apply the migration:

```bash
psql "$DATABASE_URL" -f sql/002_add_staging.sql
```

Run ingestion:

```bash
python -m src.ingest
```

Then process staging:

```bash
python -m src.process
```

The pipeline now has two operational steps.

---

## 20. Verify Raw Data

Run:

```sql
SELECT
    raw_id,
    source_record_id,
    received_at,
    payload
FROM raw_posts
ORDER BY raw_id DESC
LIMIT 5;
```

You should be able to see the original source representation.

---

## 21. Verify Staging Data

Run:

```sql
SELECT
    staging_id,
    raw_id,
    source_record_id,
    user_id,
    title,
    staged_at
FROM posts_staging
ORDER BY staging_id DESC
LIMIT 5;
```

Now the raw-to-staging relationship can be inspected directly.

---

## 22. Verify Final Data

Run:

```sql
SELECT
    id,
    user_id,
    title,
    ingested_at
FROM posts
ORDER BY id
LIMIT 10;
```

The three layers can now be inspected independently.

---

## 23. Verify Counts

Run:

```sql
SELECT COUNT(*) FROM raw_posts;
SELECT COUNT(*) FROM posts_staging;
SELECT COUNT(*) FROM posts;
```

For a simple first run, the counts may match.

Do not assume they must always match.

Later recipes may introduce filtering, validation failures, deduplication, failed processing, and cleanup.

Differences between counts can therefore be useful diagnostic information.

---

## 24. Trace One Record Through the Pipeline

Lineage means being able to answer where a record came from and what happened to it.

A simplified query is:

```sql
SELECT
    p.id AS post_id,
    s.staging_id,
    s.raw_id,
    s.source_record_id,
    r.payload
FROM posts p
JOIN posts_staging s
    ON s.source_record_id = p.id
JOIN raw_posts r
    ON r.raw_id = s.raw_id
WHERE p.id = 10;
```

The logical path is:

```text
Final record
    ↓
Staging record
    ↓
Raw record
    ↓
Original payload
```

This becomes extremely useful during production debugging.

---

## 25. Failure Scenario: Processing Fails

Suppose ingestion succeeds:

```text
API
 ↓
raw_posts
 ↓
posts_staging
```

But processing fails before the target table is updated.

The staged data remains available.

We can attempt processing again without requesting the source again.

That is a major operational benefit of the staging boundary.

---

## 26. Failure Scenario: API Is Down

Suppose the source API is unavailable.

If records have already been staged, processing can still continue.

```text
API unavailable
      X

posts_staging
      ↓
processing
      ↓
posts
```

Ingestion and processing no longer have to succeed at exactly the same moment.

---

## 27. Failure Scenario: Database Is Down

If PostgreSQL is unavailable during ingestion:

```text
API
 ↓
database connection fails
 ↓
transaction fails
```

The application should not report the records as successfully committed.

The exact recovery behavior depends on where the failure occurs and which transaction has already committed.

---

## 28. Failure Scenario: Invalid Source Record

Suppose a source record is missing a required field.

For this recipe:

```text
API response
   ↓
validation
   ↓
invalid record
   ↓
ingestion fails
```

This is intentionally simple.

Later, Chapter 26 introduces quarantine so invalid records can be isolated instead of stopping the normal path.

---

## 29. Failure Scenario: Processing Runs Twice

At this stage, staging has no processing status.

If you run:

```bash
python -m src.process
python -m src.process
```

the same staging rows are read again.

The final posts table uses a primary key and conflict handling, so duplicate final rows are not created for the same ID.

But the processor still scans the same staging records.

That is inefficient.

It also means the system cannot clearly answer:

> Which staging records are waiting, processing, completed, or failed?

Chapter 22 solves that problem.

---

## 30. Staging Is Not Idempotency

A staging layer does not automatically make ingestion idempotent.

Consider:

```text
Run 1
source id 10
    ↓
raw row 100
    ↓
staging row 200

Run 2
source id 10
    ↓
raw row 101
    ↓
staging row 201
```

The same source record can still be stored twice.

Staging provides separation.

Idempotency controls repeated effects.

They solve different problems.

Chapter 20 covers idempotency.

---

## 31. Staging Is Not Deduplication

Suppose the source sends:

```text
id=10
id=10
id=11
```

Staging can preserve all three records unless a deduplication rule exists.

That is often useful because staging should not silently destroy source information before the pipeline has decided how duplicates should be handled.

Chapter 21 will add a deliberate deduplication strategy.

---

## 32. Staging Is Not Validation

A staging row can still contain:

- an invalid value
- a missing business field
- an impossible date
- an unknown reference
- a duplicate
- a schema mismatch

Validation is a separate concern.

Chapter 19 will build that capability.

---

## 33. Processing Status

Our staging table currently lacks fields such as:

```text
status
processed_at
attempt_count
error_message
```

Without these fields, processing state is difficult to inspect.

A production-oriented staging table often needs to answer:

- Which records are waiting?
- Which records are being processed?
- Which records succeeded?
- Which records failed?
- How many times was a record attempted?

Chapter 22 builds this state model.

---

## 34. Transaction Design

One possible ingestion transaction is:

```text
Begin
  ↓
Write raw
  ↓
Write staging
  ↓
Commit
```

If the transaction rolls back, neither write is committed.

Another architecture may intentionally separate the operations.

Neither design is universally correct.

The correct boundary depends on consistency, recovery, throughput, and operational requirements.

For this learning recipe, one transaction keeps the behavior easy to understand.

---

## 35. Retention

Raw and staging data may have different retention requirements.

One possible model is:

```text
Raw
 ↓
longer retention

Staging
 ↓
retain until processing is complete

Target
 ↓
business retention
```

This is only a generic example.

Retention must consider storage cost, replay requirements, privacy, security, contractual requirements, and regulatory requirements.

Do not keep sensitive data forever just because it might be useful later.

---

## 36. Can Staging Be Something Other Than a Table?

Yes.

Staging can be implemented using:

- PostgreSQL tables
- object storage
- files
- message topics
- warehouse staging schemas
- temporary database structures

The correct choice depends on the architecture.

This recipe uses PostgreSQL because it is easy to run, query, and inspect while learning.

---

## 37. Testing the Staging Boundary

We need tests for both directions.

### Ingestion tests

Verify:

- raw payload is preserved
- raw ID is generated
- staging rows are created
- source IDs are copied correctly

### Processing tests

Verify:

- staging rows can be read
- target rows are created
- mappings are correct
- database errors are handled

---

## 38. Unit Test the Mapping

A useful unit-test target is the transformation from source record to staging values.

```python
def build_staging_values(raw_id: int, record: dict) -> tuple:
    return (
        raw_id,
        record["id"],
        record["userId"],
        record["title"],
        record["body"],
    )
```

Test it with:

```python
def test_build_staging_values():
    record = {
        "userId": 7,
        "id": 42,
        "title": "hello",
        "body": "world",
    }

    result = build_staging_values(100, record)

    assert result == (100, 42, 7, "hello", "world")
```

This verifies the mapping without requiring PostgreSQL.

---

## 39. Integration Test the Database Flow

A stronger test verifies:

```text
source record
    ↓
raw_posts
    ↓
posts_staging
    ↓
posts
```

Use a dedicated test database or test container.

Do not run integration tests against production data.

The test should verify actual SQL, foreign keys, transactions, and constraints where those behaviors matter.

---

## 40. Test Lineage

A useful integration test should verify that a staging row points to the correct raw row.

```sql
SELECT
    s.raw_id,
    r.raw_id
FROM posts_staging s
JOIN raw_posts r
    ON r.raw_id = s.raw_id;
```

The relationship should remain valid.

This is a small example of testing data lineage rather than only testing final row counts.

---

## 41. Common Mistakes

### Mistake 1 — Adding staging without a reason

More tables do not automatically mean a better pipeline.

### Mistake 2 — Treating staging as final data

Staging is normally an intermediate boundary.

### Mistake 3 — Transforming everything during ingestion

Keep ingestion responsibilities clear.

### Mistake 4 — Losing the original source payload

If replay matters, preserve enough information to reproduce processing.

### Mistake 5 — No lineage

Keep identifiers that allow records to be traced between layers.

### Mistake 6 — Assuming staging solves duplicates

Staging and deduplication are different concerns.

### Mistake 7 — No processing state

Repeated scans become expensive and difficult to control.

### Mistake 8 — Keeping sensitive data forever

Retention must be intentional.

### Mistake 9 — Different processing logic for backfills

Where appropriate, reuse the same processing path.

### Mistake 10 — Making staging impossible to inspect

Clear names and lineage fields make debugging much easier.

---

## 42. Troubleshooting

### Problem: raw row exists but staging row does not

Check:

- whether the transaction committed
- validation failures
- staging SQL errors
- foreign-key errors
- application logs

### Problem: staging row exists but final row does not

Run the processing step separately.

Then inspect processing logs and database errors.

### Problem: staging contains duplicates

This is expected until idempotency or deduplication is implemented.

Do not delete rows manually before understanding why they were created.

### Problem: processor scans the same rows repeatedly

Processing state has not been implemented yet.

Chapter 22 addresses this.

### Problem: raw and staging counts differ

Possible causes include:

- validation failures
- filtering
- duplicates
- partial processing
- previous runs
- cleanup

Investigate lineage and processing rules before deciding that the difference is an error.

---

## 43. Production Considerations

A production staging layer may need:

- indexes
- partitioning
- retention policies
- processing status
- error information
- batch IDs
- source IDs
- ingestion timestamps
- schema versions
- access controls
- encryption
- monitoring
- data-quality checks

The exact design depends on workload and requirements.

Do not copy fields from another pipeline without understanding why they exist.

Start with the operational questions the system needs to answer.

---

## 44. Definition of Done

The recipe is complete when:

- [ ] Incoming records have a durable raw representation.
- [ ] A staging table exists.
- [ ] Raw records can be linked to staging records.
- [ ] Ingestion and processing are separate responsibilities.
- [ ] Staging records can be processed without calling the source again.
- [ ] Final records can be traced back to raw input.
- [ ] The staging boundary has appropriate tests.
- [ ] Transaction behavior is understood.
- [ ] Failure behavior has been tested.
- [ ] Duplicate behavior is understood.
- [ ] Processing-status limitations are documented.
- [ ] Retention requirements have been considered.

---

## 45. What You Learned

You now have a clear boundary between ingestion and processing.

The mental model is:

```text
Source
  ↓
Raw
  ↓
Staging
  ↓
Processing
  ↓
Target
```

The most important benefit is separation.

Ingestion can focus on receiving and preserving data.

Processing can focus on transforming and applying business rules.

That separation makes failure handling, replay, testing, and troubleshooting easier.

---

## 46. Recipe Progression

Part III has now moved from:

```text
17. Basic ingestion
    ↓
API → PostgreSQL
```

to:

```text
18. Staging
    ↓
API → Raw → Staging → Target
```

The next question is:

> What exactly should be accepted into the pipeline?

That leads to validation.

---

## Next Recipe

**Next: Chapter 19 — Validate Incoming Data**## Production Tools You Should Know

These tools solve or provide production implementations of concepts covered in this recipe. Learn what the tool provides, but understand the underlying problem first.

| Tool | What to know |
|---|---|
| **PostgreSQL** | Common staging-layer storage. |
| **dbt** | Staging and transformation workflows in analytical systems. |
| **Apache Spark** | Large-scale staging and transformation workloads. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the engineering mechanism.

---


