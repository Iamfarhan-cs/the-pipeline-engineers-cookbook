# Recipe 17 — Create an Ingestion Pipeline

Part III starts with the most basic pipeline we can build: receive data from a source, validate it, and store it in a database.

This chapter is a complete recipe.

We will build a small **API → Python → PostgreSQL** pipeline.

The goal is not to build a production platform in one chapter. The goal is to understand the complete path from source data to stored data and to have working code that we can improve in later chapters.

---

## Production Tools You Should Know

These tools solve or provide production implementations of concepts covered in this recipe. Learn what the tool provides, but understand the underlying problem first.

| Tool | What to know |
|---|---|
| **Requests** | HTTP/API source communication from Python. |
| **PostgreSQL** | Common durable destination for ingestion pipelines. |
| **Docker Compose** | Reproducible local pipeline infrastructure. |

> These are reference tools for recognition and vocabulary, not substitutes for understanding the engineering mechanism.

---

## Recipe Goal

By the end of this recipe, you will have a small ingestion pipeline that can:

1. call an HTTP API
2. receive JSON data
3. check the response
4. validate the basic shape of the records
5. connect to PostgreSQL
6. create the destination table
7. insert records
8. record an ingestion run
9. report useful errors
10. verify that the data arrived

The flow is:

```text
HTTP API
   ↓
Python ingestion script
   ↓
HTTP response
   ↓
Basic validation
   ↓
PostgreSQL
   ↓
Stored records
   ↓
Verification
```

Later chapters will improve this basic pipeline with staging, validation, idempotency, deduplication, processing status, retries, replay, quarantine, and backfills.

---

## 1. The Problem

Imagine a company has an API that exposes business records.

The data might represent:

- customers
- orders
- payments
- shipments
- products
- transactions
- telemetry events
- sanctions records

We want the data inside PostgreSQL so that another system can query it.

A first attempt might be:

```text
Call API
   ↓
Parse JSON
   ↓
Insert into database
```

That looks simple.

But even this small pipeline has important questions:

- What happens when the API is unavailable?
- What happens when the API returns an error?
- What happens when the JSON is malformed?
- What happens when one record is missing a field?
- What happens when the database is unavailable?
- What happens when the script is run twice?
- How do we know how many records were received?
- How do we know how many records were stored?
- How do we know whether the run succeeded?

A useful ingestion pipeline starts answering these questions explicitly.

---

## 2. What Are We Building?

For this recipe, we will use a simple public JSON API as the learning source.

**GENERIC LEARNING EXAMPLE:** We will use JSONPlaceholder's `/posts` endpoint, which returns JSON objects containing `userId`, `id`, `title`, and `body`.

The destination will be PostgreSQL.

The pipeline will store the following fields:

| Field | Meaning |
|---|---|
| `id` | Source record identifier |
| `user_id` | Source user identifier |
| `title` | Record title |
| `body` | Record body |
| `ingested_at` | Time the record was stored by our pipeline |

We will also keep a small `ingestion_runs` table so we can answer:

> Did the ingestion run? How many records did it receive? How many did it store? Did it succeed?

---

## 3. Architecture

Our learning architecture is intentionally small.

```text
                    ┌──────────────────────┐
                    │      HTTP API        │
                    │   JSON source data   │
                    └──────────┬───────────┘
                               │
                               │ HTTP GET
                               ▼
                    ┌──────────────────────┐
                    │ Python Ingestion     │
                    │ Worker               │
                    │                      │
                    │ 1. Request           │
                    │ 2. Check response    │
                    │ 3. Validate records  │
                    │ 4. Insert            │
                    │ 5. Record run        │
                    └──────────┬───────────┘
                               │
                               │ SQL
                               ▼
                    ┌──────────────────────┐
                    │     PostgreSQL       │
                    │                      │
                    │ posts                │
                    │ ingestion_runs      │
                    └──────────────────────┘
```

There is no Kafka, Airflow, warehouse, cloud storage, or orchestration system in this first recipe.

That is deliberate.

Learn the basic movement of data first.

---

## 4. Before You Start

You need:

- Python 3.11+
- PostgreSQL 14+ or a compatible PostgreSQL installation
- `pip`
- basic SQL knowledge
- basic Python knowledge
- an internet connection for the learning API

Docker is recommended because it gives us a repeatable PostgreSQL environment.

If you already have PostgreSQL, you can use it instead.

---

## 5. Repository Structure

**GENERIC RECIPE PROJECT:** Create a small project with this structure:

```text
api-ingestion-pipeline/
├── .env.example
├── .gitignore
├── docker-compose.yml
├── requirements.txt
├── sql/
│   └── init.sql
├── src/
│   ├── __init__.py
│   ├── config.py
│   ├── db.py
│   └── ingest.py
└── tests/
    └── test_ingest.py
```

The structure is intentionally simple.

We are separating configuration, database access, ingestion logic, and tests so that each part has a clear responsibility.

---

## 6. Create the Python Environment

Create the project directory:

```bash
mkdir api-ingestion-pipeline
cd api-ingestion-pipeline
```

Create a virtual environment:

```bash
python -m venv .venv
```

Activate it on Linux/macOS:

```bash
source .venv/bin/activate
```

On Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

---

## 7. Dependencies

Create `requirements.txt`:

```text
requests>=2.31,<3
psycopg[binary]>=3.2,<4
python-dotenv>=1.0,<2
pytest>=8,<9
```

We use:

- `requests` for HTTP
- `psycopg` for PostgreSQL
- `python-dotenv` for local configuration
- `pytest` for tests

Install them:

```bash
pip install -r requirements.txt
```

---

## 8. Configuration

Do not put database passwords directly inside Python code.

Create `.env.example`:

```dotenv
API_URL=https://jsonplaceholder.typicode.com/posts
DATABASE_URL=postgresql://pipeline:pipeline@localhost:5432/pipeline_db
HTTP_TIMEOUT_SECONDS=30
```

Then create a local `.env` from it.

```bash
cp .env.example .env
```

On Windows, you can create the file manually.

`.env` should normally be ignored by Git.

Add this to `.gitignore`:

```text
.venv/
.env
__pycache__/
.pytest_cache/
```

---

## 9. Configuration Code

Create `src/config.py`:

```python
import os

from dotenv import load_dotenv


load_dotenv()


def get_required(name: str) -> str:
    value = os.getenv(name)
    if not value:
        raise RuntimeError(f"Required environment variable is missing: {name}")
    return value


API_URL = get_required("API_URL")
DATABASE_URL = get_required("DATABASE_URL")
HTTP_TIMEOUT_SECONDS = int(os.getenv("HTTP_TIMEOUT_SECONDS", "30"))
```

This module has one job: read configuration.

It does not call the API.

It does not connect to PostgreSQL.

It does not insert data.

Keeping responsibilities separate makes the code easier to test and change.

---

## 10. Start PostgreSQL With Docker

Create `docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: pipeline_db
      POSTGRES_USER: pipeline
      POSTGRES_PASSWORD: pipeline
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Start PostgreSQL:

```bash
docker compose up -d
```

Check the container:

```bash
docker compose ps
```

You should see the PostgreSQL service running.

---

## 11. Understand the Database Connection

Our application connects to:

```text
localhost:5432
     ↓
PostgreSQL
     ↓
pipeline_db
```

The connection string is:

```text
postgresql://pipeline:pipeline@localhost:5432/pipeline_db
```

It contains:

```text
postgresql://
    ↓
username:password
    ↓
host:port
    ↓
database
```

In a real environment, credentials should come from a secure secret-management mechanism rather than being committed to the repository.

---

## 12. Create the Database Tables

Create `sql/init.sql`:

```sql
CREATE TABLE IF NOT EXISTS ingestion_runs (
    run_id BIGSERIAL PRIMARY KEY,
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    finished_at TIMESTAMPTZ,
    status TEXT NOT NULL,
    records_received INTEGER NOT NULL DEFAULT 0,
    records_stored INTEGER NOT NULL DEFAULT 0,
    error_message TEXT
);

CREATE TABLE IF NOT EXISTS posts (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    ingested_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

These tables are enough for the first recipe.

`posts` contains the business records.

`ingestion_runs` contains information about each execution of our ingestion process.

---

## 13. Apply the Schema

One simple way to apply the schema is with `psql`.

If `psql` is available on your machine:

```bash
psql "postgresql://pipeline:pipeline@localhost:5432/pipeline_db" -f sql/init.sql
```

Alternatively, the same SQL can be executed through a database client.

Verify the tables:

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name;
```

You should see the two tables created by this recipe.

---

## 14. Database Connection Code

Create `src/db.py`:

```python
from contextlib import contextmanager

import psycopg

from .config import DATABASE_URL


@contextmanager
def get_connection():
    with psycopg.connect(DATABASE_URL) as connection:
        yield connection
```

The context manager makes connection cleanup automatic.

When the block finishes, the connection is closed.

---

## 15. Start With the Source

Before writing the database code, understand the source response.

A typical response from the learning API looks like:

```json
[
  {
    "userId": 1,
    "id": 1,
    "title": "example title",
    "body": "example body"
  }
]
```

The pipeline therefore expects:

```text
response
  ↓
JSON array
  ↓
object
  ↓
userId + id + title + body
```

Do not write ingestion code until you understand the source contract.

That habit becomes increasingly important when working with real APIs.

---

## 16. Request the API

The first function in our ingestion code will fetch the source data.

Create `src/ingest.py`:

```python
import requests

from .config import API_URL, HTTP_TIMEOUT_SECONDS


def fetch_records() -> list[dict]:
    response = requests.get(
        API_URL,
        timeout=HTTP_TIMEOUT_SECONDS,
    )

    response.raise_for_status()

    data = response.json()

    if not isinstance(data, list):
        raise ValueError("Expected API response to be a JSON array")

    return data
```

This function performs four important actions:

1. sends the HTTP request
2. applies a timeout
3. converts HTTP errors into exceptions
4. checks that the top-level JSON value is a list

It does not write to the database.

---

## 17. Why Use a Timeout?

Never assume an external API will respond immediately.

Without a timeout, a network operation can wait much longer than expected.

With a timeout:

```text
API request
    ↓
response received
    OR
timeout/error
```

A timeout gives the pipeline a defined boundary.

The value used in this recipe is configurable.

In production, the correct timeout depends on the API and workload.

---

## 18. Why `raise_for_status()`?

An HTTP response can contain an error status.

For example:

```text
200 OK
404 Not Found
429 Too Many Requests
500 Internal Server Error
```

`response.raise_for_status()` turns unsuccessful HTTP responses into exceptions.

That means the ingestion code does not accidentally treat an error response as valid data.

Later, Chapter 24 will add a more detailed retry policy.

---

## 19. Validate Individual Records

Checking that the response is a list is only the first validation step.

We also need to check each record.

Add this function to `src/ingest.py`:

```python
def validate_record(record: object) -> dict:
    if not isinstance(record, dict):
        raise ValueError("Each record must be a JSON object")

    required_fields = {"userId", "id", "title", "body"}
    missing = required_fields - record.keys()

    if missing:
        raise ValueError(
            f"Record is missing required fields: {sorted(missing)}"
        )

    if not isinstance(record["id"], int):
        raise ValueError("Record id must be an integer")

    if not isinstance(record["userId"], int):
        raise ValueError("userId must be an integer")

    if not isinstance(record["title"], str):
        raise ValueError("title must be a string")

    if not isinstance(record["body"], str):
        raise ValueError("body must be a string")

    return record
```

This is deliberately basic validation.

We are not checking business rules yet.

Chapter 19 will build a more complete validation layer.

---

## 20. Why Validate Before Inserting?

Without validation, bad data can reach the database.

Example:

```text
API
 ↓
Bad record
 ↓
Database error
```

With validation:

```text
API
 ↓
Validate
 ↓
Invalid record detected
 ↓
Stop or quarantine
```

For this first recipe, invalid data stops the run.

Later chapters will introduce quarantine so one bad record does not necessarily stop the entire pipeline.

---

## 21. Insert One Record

Now create the database write function.

Add to `src/ingest.py`:

```python
from .db import get_connection


def store_record(connection, record: dict) -> None:
    connection.execute(
        """
        INSERT INTO posts (id, user_id, title, body)
        VALUES (%s, %s, %s, %s)
        ON CONFLICT (id) DO NOTHING
        """,
        (
            record["id"],
            record["userId"],
            record["title"],
            record["body"],
        ),
    )
```

Notice that the SQL uses parameters.

We are not constructing SQL like this:

```python
query = f"INSERT ... '{record['title']}' ..."
```

Parameterized SQL is safer and avoids SQL quoting problems.

---

## 22. Why `ON CONFLICT DO NOTHING`?

Our `posts.id` column is a primary key.

If the same source record is inserted again, PostgreSQL will detect the conflict.

`ON CONFLICT (id) DO NOTHING` tells PostgreSQL to leave the existing row unchanged.

This gives the small recipe a basic duplicate guard.

Important:

> This is not the complete idempotency design we will build later.

A real pipeline may need an explicit idempotency key, processing state, source version, conflict-update rules, and transaction boundaries.

Chapter 20 covers that in detail.

---

## 23. Insert the Records

Now connect fetching, validation, and storage.

Add:

```python
def store_records(records: list[dict]) -> int:
    stored = 0

    with get_connection() as connection:
        for raw_record in records:
            record = validate_record(raw_record)
            before = connection.execute(
                "SELECT 1 FROM posts WHERE id = %s",
                (record["id"],),
            ).fetchone()

            store_record(connection, record)

            if before is None:
                stored += 1

        connection.commit()

    return stored
```

This is intentionally straightforward so the lifecycle is easy to see.

The flow is:

```text
records
  ↓
validate
  ↓
insert
  ↓
commit
```

The explicit existence check is suitable for teaching, but it performs an extra database query per record.

For larger workloads, use more efficient insert accounting or PostgreSQL features rather than copying this pattern blindly.

---

## 24. Record the Ingestion Run

Now we want operational information about the run.

Add:

```python
def start_run(connection) -> int:
    row = connection.execute(
        """
        INSERT INTO ingestion_runs (status)
        VALUES ('running')
        RETURNING run_id
        """
    ).fetchone()

    return row[0]
```

To finish a successful run:

```python
def finish_run(
    connection,
    run_id: int,
    records_received: int,
    records_stored: int,
) -> None:
    connection.execute(
        """
        UPDATE ingestion_runs
        SET status = 'success',
            finished_at = NOW(),
            records_received = %s,
            records_stored = %s
        WHERE run_id = %s
        """,
        (records_received, records_stored, run_id),
    )
```

For a failed run:

```python
def fail_run(connection, run_id: int, error_message: str) -> None:
    connection.execute(
        """
        UPDATE ingestion_runs
        SET status = 'failed',
            finished_at = NOW(),
            error_message = %s
        WHERE run_id = %s
        """,
        (error_message[:2000], run_id),
    )
```

Now the database can tell us what happened.

---

## 25. The Complete Ingestion Function

Combine the pieces:

```python
def run_ingestion() -> int:
    with get_connection() as connection:
        run_id = start_run(connection)
        connection.commit()

    try:
        records = fetch_records()
        stored = store_records(records)

        with get_connection() as connection:
            finish_run(
                connection,
                run_id,
                records_received=len(records),
                records_stored=stored,
            )
            connection.commit()

        return stored

    except Exception as exc:
        with get_connection() as connection:
            fail_run(connection, run_id, str(exc))
            connection.commit()
        raise
```

At this point we have a complete pipeline.

```text
run_ingestion()
      │
      ├── start run
      │
      ├── fetch API
      │
      ├── validate records
      │
      ├── store records
      │
      ├── mark success
      │
      └── or mark failure
```

---

## 26. Add a Command-Line Entry Point

At the bottom of `src/ingest.py`, add:

```python
if __name__ == "__main__":
    count = run_ingestion()
    print(f"Stored {count} new records")
```

Now the module can be executed directly.

Run:

```bash
python -m src.ingest
```

If everything is configured correctly, the script should call the API and write records into PostgreSQL.

---

## 27. Verify the Data

Connect to PostgreSQL and run:

```sql
SELECT COUNT(*) FROM posts;
```

Then inspect a few records:

```sql
SELECT id, user_id, title, ingested_at
FROM posts
ORDER BY id
LIMIT 10;
```

You should see the source records in PostgreSQL.

---

## 28. Verify the Ingestion Run

Check the run table:

```sql
SELECT
    run_id,
    status,
    records_received,
    records_stored,
    started_at,
    finished_at,
    error_message
FROM ingestion_runs
ORDER BY run_id DESC
LIMIT 10;
```

A successful run should contain:

```text
status = success
records_received > 0
records_stored > 0
error_message = NULL
```

This is already better than simply printing `Done` to the terminal.

---

## 29. Test the Pipeline Manually

A first manual test should be:

```text
Start PostgreSQL
      ↓
Run pipeline
      ↓
Check exit result
      ↓
Query posts
      ↓
Query ingestion_runs
```

Commands:

```bash
docker compose up -d
python -m src.ingest
```

Then query PostgreSQL.

Manual verification is useful during the first implementation because it confirms that the complete path works.

Automated tests come next.

---

## 30. Unit Test the Response Validation

Create `tests/test_ingest.py`:

```python
import pytest

from src.ingest import validate_record


def test_validate_record_accepts_valid_record():
    record = {
        "userId": 1,
        "id": 10,
        "title": "hello",
        "body": "world",
    }

    result = validate_record(record)

    assert result == record


def test_validate_record_rejects_missing_field():
    record = {
        "userId": 1,
        "id": 10,
        "title": "hello",
    }

    with pytest.raises(ValueError, match="missing required fields"):
        validate_record(record)


def test_validate_record_rejects_wrong_id_type():
    record = {
        "userId": 1,
        "id": "10",
        "title": "hello",
        "body": "world",
    }

    with pytest.raises(ValueError, match="id must be an integer"):
        validate_record(record)
```

Run:

```bash
pytest
```

These tests do not require PostgreSQL or the external API.

That makes them fast and reliable.

---

## 31. Test the HTTP Request Separately

Network calls should not be required for every unit test.

A useful design is to keep `fetch_records()` small enough that its network behavior can be mocked later.

For example, the test can replace the HTTP client response with a fake response.

**GENERIC EXAMPLE:**

```python
from unittest.mock import Mock, patch

from src.ingest import fetch_records


@patch("src.ingest.requests.get")
def test_fetch_records(mock_get):
    response = Mock()
    response.json.return_value = [
        {
            "userId": 1,
            "id": 1,
            "title": "test",
            "body": "test body",
        }
    ]
    response.raise_for_status.return_value = None
    mock_get.return_value = response

    records = fetch_records()

    assert len(records) == 1
    assert records[0]["id"] == 1
    mock_get.assert_called_once()
```

This tests the Python logic without depending on the real API.

---

## 32. Test Database Writes

Database tests should use a real test database when possible.

Do not replace every database test with mocks.

A database integration test can verify:

```text
Python
   ↓
SQL
   ↓
PostgreSQL
   ↓
stored row
```

For this chapter, the exact test-database setup depends on your environment.

**RECOMMENDED PATTERN:** Use an isolated PostgreSQL database or disposable PostgreSQL container for integration tests.

Do not run destructive test SQL against a production database.

---

## 33. Failure Scenario: API Is Unavailable

Suppose the API cannot be reached.

The expected flow is:

```text
HTTP request
     ↓
connection error
     ↓
pipeline raises error
     ↓
run marked failed
```

The pipeline should not report success.

Check:

```sql
SELECT run_id, status, error_message
FROM ingestion_runs
ORDER BY run_id DESC
LIMIT 1;
```

You should see a failed run if the failure happened after the run was created.

---

## 34. Failure Scenario: API Returns HTTP 500

If the source returns a server error:

```text
HTTP 500
   ↓
raise_for_status()
   ↓
exception
   ↓
failed run
```

This is a temporary failure candidate.

But this chapter does not automatically retry it.

Retries need a deliberate policy.

Chapter 24 will add retry logic.

---

## 35. Failure Scenario: API Returns Invalid JSON

Suppose the server returns something that is not valid JSON.

`response.json()` will raise an exception.

The pipeline should fail rather than insert unknown data.

That is an important rule:

> Do not turn an unknown response into apparently valid database records.

---

## 36. Failure Scenario: Wrong Response Shape

Suppose the API returns:

```json
{
  "message": "temporary problem"
}
```

But the pipeline expects a list.

Our validation detects this:

```python
if not isinstance(data, list):
    raise ValueError("Expected API response to be a JSON array")
```

This prevents the pipeline from silently processing an unexpected response.

---

## 37. Failure Scenario: Missing Record Field

Suppose one record looks like:

```json
{
  "userId": 1,
  "id": 10,
  "title": "hello"
}
```

`body` is missing.

The validator raises an error.

For this first recipe, the entire run fails.

That is simple but not always ideal.

Later we will build quarantine behavior so that invalid records can be isolated instead of stopping all processing.

---

## 38. Failure Scenario: PostgreSQL Is Down

Suppose the database is unavailable.

The pipeline should fail rather than pretending that records were stored.

Check:

```bash
docker compose ps
docker compose logs postgres
```

Then check the database connection configuration.

Typical causes include:

- PostgreSQL is stopped
- wrong host
- wrong port
- wrong username
- wrong password
- wrong database name
- network problem

---

## 39. Failure Scenario: Duplicate Source ID

Suppose the pipeline receives record `id = 10` twice.

The primary key prevents two rows with the same `id`.

Because the insert uses:

```sql
ON CONFLICT (id) DO NOTHING
```

the second insert does not create another row.

This gives us basic duplicate protection.

But there is an important limitation.

What if the source record changes?

For example:

```text
First run:
id = 10
title = Old title

Later run:
id = 10
title = New title
```

`DO NOTHING` keeps the old row.

Whether that is correct depends on the source contract.

Later chapters will examine update semantics and idempotency more deeply.

---

## 40. Transaction Boundary

Transactions determine which database changes succeed together.

In our basic implementation, records are inserted and then committed.

Conceptually:

```text
Begin transaction
     ↓
Insert records
     ↓
Commit
```

If an exception occurs before the commit, PostgreSQL can roll back the uncommitted transaction.

This gives us an important property:

> A database transaction can prevent a partial set of inserts from being committed as if the whole batch succeeded.

However, our run-status update is stored separately from the record transaction.

That is a deliberate simplification for teaching.

Production pipelines often need a more carefully designed transaction model.

---

## 41. What If the Pipeline Crashes?

Consider this sequence:

```text
Fetch API
   ↓
Insert records
   ↓
Process crashes
   ↓
Commit?
```

If the transaction has not committed, PostgreSQL can roll back the uncommitted changes.

If the commit already happened, the data remains.

This is why transaction boundaries matter.

It is also why production pipelines need explicit recovery and replay strategies.

---

## 42. Idempotency in This Recipe

This recipe already has a small duplicate guard:

```sql
PRIMARY KEY (id)
ON CONFLICT (id) DO NOTHING
```

That means running the same source records twice does not create duplicate `posts` rows.

But do not conclude that the whole pipeline is fully idempotent.

We still need to think about:

- run records
- external side effects
- source updates
- partial processing
- retries
- concurrent executions
- changed records

Chapter 20 will turn this into a dedicated idempotency recipe.

---

## 43. Retry Behavior

This first recipe does not automatically retry failed HTTP requests.

That is intentional.

A retry should not simply mean:

```python
while True:
    try:
        request()
    except:
        request()
```

That can create retry storms and can overload an already failing service.

A proper retry design needs:

- retryable errors
- maximum attempts
- backoff
- jitter where appropriate
- timeout
- logging
- final failure behavior

Chapter 24 will implement this properly.

---

## 44. Replay and Reprocessing

Suppose the pipeline fails today.

How do you run it again?

For this simple recipe, replay is just another execution:

```text
Run pipeline again
      ↓
Fetch source again
      ↓
Insert records
```

The primary key prevents duplicate rows for unchanged source IDs.

But this is not yet a real replay system.

A production replay design should define:

- what data is replayed
- which time range is replayed
- whether the source can be fetched again
- how duplicates are handled
- how changed records are handled
- how successful records are avoided or reprocessed
- how the replay is tracked

Chapter 25 will build replay and reprocessing concepts explicitly.

---

## 45. Batch Size

Our example downloads the complete response into memory.

That is fine for a small learning API.

It becomes a problem when the source contains millions of records.

Instead of:

```text
API
 ↓
1,000,000 records in memory
 ↓
database
```

a larger system may use pagination:

```text
API page 1
    ↓
store
    ↓
API page 2
    ↓
store
    ↓
API page 3
    ↓
store
```

Pagination, batching, checkpointing, and incremental processing are covered later in the book.

---

## 46. Pagination

Many APIs use pagination.

Common patterns include:

```text
page=1&page_size=100
```

or:

```text
cursor=abc123
```

Do not assume the API uses page numbers.

Read its actual contract.

A paginated ingestion loop generally looks like:

```text
Request page
     ↓
Validate page
     ↓
Store records
     ↓
Next page?
  /       \
yes       no
 ↓         ↓
repeat    finish
```

Pagination becomes important when designing scalable ingestion.

---

## 47. Source Metadata

Real ingestion pipelines often need metadata beyond the business record.

Useful metadata can include:

- source URL
- request timestamp
- response timestamp
- HTTP status
- source version
- ingestion run ID
- source record ID
- checksum

Our `ingestion_runs` table begins this idea.

Later chapters will expand it.

---

## 48. Logging

At minimum, an ingestion pipeline should make failures understandable.

A basic learning implementation can print:

```python
print(f"Fetching records from {API_URL}")
print(f"Received {len(records)} records")
print(f"Stored {stored} new records")
```

For production code, structured logging is usually preferable.

Example:

```text
event=ingestion_completed
run_id=42
records_received=100
records_stored=100
status=success
```

Do not log sensitive payloads just because they are convenient for debugging.

Chapter 8 introduced this observability principle.

---

## 49. Data Privacy

Real ingestion sources can contain personal or financial information.

Do not assume that because data is available through an API it is safe to print everywhere.

A pipeline should define:

- what data is allowed
- what data is sensitive
- where raw data can be stored
- what can appear in logs
- who can access the database
- how long data is retained

The learning example uses non-sensitive demonstration records.

Real payment, identity, sanctions, or customer data requires stronger controls.

---

## 50. Definition of Done

For this recipe, the implementation is complete when:

- [ ] PostgreSQL starts successfully.
- [ ] Configuration loads from environment variables.
- [ ] The API request has a timeout.
- [ ] HTTP errors are detected.
- [ ] The response shape is validated.
- [ ] Required record fields are checked.
- [ ] PostgreSQL tables exist.
- [ ] Records are inserted using parameterized SQL.
- [ ] Duplicate source IDs do not create duplicate rows.
- [ ] An ingestion run is recorded.
- [ ] Successful runs show received and stored counts.
- [ ] Failed runs record an error.
- [ ] The pipeline can be run from the command line.
- [ ] Stored records can be verified with SQL.
- [ ] Basic unit tests pass.
- [ ] The pipeline does not report success when the source or database fails.

---

## 51. Complete `src/ingest.py`

After understanding each part, it is useful to see the complete learning implementation together.

```python
import requests

from .config import API_URL, HTTP_TIMEOUT_SECONDS
from .db import get_connection


def fetch_records() -> list[dict]:
    response = requests.get(
        API_URL,
        timeout=HTTP_TIMEOUT_SECONDS,
    )
    response.raise_for_status()

    data = response.json()

    if not isinstance(data, list):
        raise ValueError("Expected API response to be a JSON array")

    return data


def validate_record(record: object) -> dict:
    if not isinstance(record, dict):
        raise ValueError("Each record must be a JSON object")

    required_fields = {"userId", "id", "title", "body"}
    missing = required_fields - record.keys()

    if missing:
        raise ValueError(
            f"Record is missing required fields: {sorted(missing)}"
        )

    if not isinstance(record["id"], int):
        raise ValueError("Record id must be an integer")

    if not isinstance(record["userId"], int):
        raise ValueError("userId must be an integer")

    if not isinstance(record["title"], str):
        raise ValueError("title must be a string")

    if not isinstance(record["body"], str):
        raise ValueError("body must be a string")

    return record


def start_run(connection) -> int:
    row = connection.execute(
        """
        INSERT INTO ingestion_runs (status)
        VALUES ('running')
        RETURNING run_id
        """
    ).fetchone()

    return row[0]


def store_record(connection, record: dict) -> bool:
    cursor = connection.execute(
        """
        INSERT INTO posts (id, user_id, title, body)
        VALUES (%s, %s, %s, %s)
        ON CONFLICT (id) DO NOTHING
        """,
        (
            record["id"],
            record["userId"],
            record["title"],
            record["body"],
        ),
    )
    return cursor.rowcount == 1


def store_records(records: list[dict]) -> int:
    stored = 0

    with get_connection() as connection:
        for raw_record in records:
            record = validate_record(raw_record)
            if store_record(connection, record):
                stored += 1

        connection.commit()

    return stored


def finish_run(
    connection,
    run_id: int,
    records_received: int,
    records_stored: int,
) -> None:
    connection.execute(
        """
        UPDATE ingestion_runs
        SET status = 'success',
            finished_at = NOW(),
            records_received = %s,
            records_stored = %s
        WHERE run_id = %s
        """,
        (records_received, records_stored, run_id),
    )


def fail_run(connection, run_id: int, error_message: str) -> None:
    connection.execute(
        """
        UPDATE ingestion_runs
        SET status = 'failed',
            finished_at = NOW(),
            error_message = %s
        WHERE run_id = %s
        """,
        (error_message[:2000], run_id),
    )


def run_ingestion() -> int:
    with get_connection() as connection:
        run_id = start_run(connection)
        connection.commit()

    try:
        records = fetch_records()
        stored = store_records(records)

        with get_connection() as connection:
            finish_run(
                connection,
                run_id,
                records_received=len(records),
                records_stored=stored,
            )
            connection.commit()

        return stored

    except Exception as exc:
        with get_connection() as connection:
            fail_run(connection, run_id, str(exc))
            connection.commit()
        raise


if __name__ == "__main__":
    count = run_ingestion()
    print(f"Stored {count} new records")
```

This is the complete learning version.

It is intentionally not the final architecture for a large production ingestion system.

---

## 52. What This Code Teaches

The code is small, but it contains several important Data Engineering ideas.

### Source interaction

`fetch_records()` isolates the external API call.

### Validation

`validate_record()` prevents obviously invalid records from reaching the database.

### Storage

`store_record()` owns the SQL insert.

### Run tracking

`ingestion_runs` gives every execution a database record.

### Transaction handling

Record inserts are committed as a database transaction.

### Failure reporting

Failed runs are marked explicitly.

### Duplicate protection

The source ID is protected by a database primary key.

This is the beginning of a real pipeline design.

---

## 53. What This Recipe Does Not Solve Yet

A common mistake when learning Data Engineering is to build one working script and assume the production problem is solved.

This recipe intentionally leaves several problems for later chapters.

### No full staging layer

We write directly into the destination table.

Chapter 18 adds staging.

### Basic validation only

We check structure and types, but not complex business rules.

Chapter 19 expands validation.

### Basic duplicate protection

We use a primary key and `ON CONFLICT DO NOTHING`.

Chapter 20 explains proper idempotency.

### No dedicated deduplication strategy

Chapter 21 covers deduplication.

### No processing status

Chapter 22 introduces explicit processing states.

### No detailed error handling

Chapter 23 expands error handling.

### No retry policy

Chapter 24 introduces retries.

### No replay system

Chapter 25 covers replay and reprocessing.

### No quarantine

Chapter 26 introduces quarantine.

### No historical backfill

Chapter 27 covers backfills.

### No incremental checkpoint

Later recipes introduce incremental processing and checkpointing.

This progression is intentional.

First make the data move.

Then make the movement reliable.

---

## 54. Production Improvements

A real production ingestion pipeline may eventually need:

```text
API
 ↓
Authentication
 ↓
Pagination
 ↓
Rate-limit handling
 ↓
Raw storage
 ↓
Staging
 ↓
Schema validation
 ↓
Deduplication
 ↓
Idempotent processing
 ↓
Curated storage
 ↓
Quality checks
 ↓
Metrics
 ↓
Alerts
 ↓
Replay / recovery
```

Do not implement all of these at once just because they exist.

Add them when the pipeline's requirements justify them.

---

## 55. Troubleshooting

### `ModuleNotFoundError`

Make sure the virtual environment is active and dependencies are installed:

```bash
pip install -r requirements.txt
```

Run the module from the project root:

```bash
python -m src.ingest
```

### PostgreSQL connection refused

Check:

```bash
docker compose ps
```

Then inspect:

```bash
docker compose logs postgres
```

### Authentication failed

Check the `DATABASE_URL` in `.env`.

### API timeout

Check the network and source URL.

Do not immediately increase the timeout without understanding why the request is slow.

### JSON validation failure

Inspect the actual API response shape.

### Duplicate key error

Check whether the table schema matches the recipe and whether the insert contains the expected conflict handling.

### No records stored

Check:

```sql
SELECT COUNT(*) FROM posts;
```

Then inspect:

```sql
SELECT * FROM ingestion_runs ORDER BY run_id DESC LIMIT 5;
```

---

## 56. Practical Investigation Exercise

After implementing the recipe, deliberately break one thing at a time.

### Exercise 1 — Wrong API URL

Change `API_URL` to an invalid URL.

Observe:

- Python error
- ingestion run status
- database contents

### Exercise 2 — Stop PostgreSQL

Run:

```bash
docker compose stop postgres
```

Then run the pipeline.

Observe the failure.

Start it again:

```bash
docker compose start postgres
```

### Exercise 3 — Change the response contract

Modify the validation temporarily so a required field is missing.

Observe how the pipeline reacts.

### Exercise 4 — Run the pipeline twice

Run:

```bash
python -m src.ingest
python -m src.ingest
```

Then compare:

```sql
SELECT COUNT(*) FROM posts;
```

with:

```sql
SELECT COUNT(*) FROM ingestion_runs;
```

You should see multiple runs without creating duplicate primary-key rows.

### Exercise 5 — Inspect failed runs

Create a controlled failure and query:

```sql
SELECT
    run_id,
    status,
    error_message
FROM ingestion_runs
ORDER BY run_id DESC;
```

This exercise is important.

Data Engineering becomes much easier to understand when you intentionally create failures and observe what the system does.

---

## 57. Recipe Summary

We started with a simple problem:

> Move JSON records from an HTTP API into PostgreSQL.

We built:

```text
HTTP API
   ↓
Python
   ↓
Validation
   ↓
PostgreSQL
```

We added a small amount of operational information:

```text
ingestion_runs
     ↓
run status
record count
error message
timestamps
```

We also added basic duplicate protection using a primary key.

The important lesson is not the exact API or table name.

The important lesson is the shape of the pipeline:

```text
Source
  ↓
Acquire
  ↓
Validate
  ↓
Store
  ↓
Verify
```

That pattern appears again and again in Data Engineering.

---

## 58. What You Learned

After completing this recipe, you should understand:

- how an ingestion pipeline starts
- how to call an HTTP API from Python
- why timeouts matter
- how to detect HTTP errors
- how to validate JSON structure
- how to validate individual records
- how to connect Python to PostgreSQL
- how to create destination tables
- how to use parameterized SQL
- how to use a primary key as a basic duplicate guard
- how transactions affect database writes
- how to track an ingestion run
- how to record success and failure
- how to verify stored data with SQL
- how to write basic unit tests
- how to think about failure scenarios
- why retries need a policy
- why replay needs a design
- and why a working script is only the beginning of a production pipeline

---

## 59. Final Mental Model

When you see an ingestion requirement, think through these questions in order:

```text
1. What is the source?
        ↓
2. What does the source return?
        ↓
3. How do I acquire it?
        ↓
4. How do I know the response is valid?
        ↓
5. Where should the data go?
        ↓
6. What database transaction protects the write?
        ↓
7. How do I know what happened?
        ↓
8. What happens when something fails?
        ↓
9. What happens if I run it again?
        ↓
10. How will I recover later?
```

The first four questions get data moving.

The remaining questions turn a script into an engineering system.

That distinction is the foundation of the recipes in Part III.

---
