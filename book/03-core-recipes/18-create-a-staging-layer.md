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
