# Recipe 06 — Payment Telemetry Pipeline Foundation

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 6  
> **Source implementation commit:** <code>0fa3f6f</code> — <code>feat: add payment telemetry pipeline foundation</code>  
> **Source repository:** <code>devops</code>

## 1. What This Recipe Teaches

This recipe establishes the Python batch-processing foundation that reads telemetry events from PostgreSQL and prepares them for later staging, validation, and incremental processing.

By completing it, you should understand how to:
- bootstrap a modern Python 3.13 data-processing package;
- use a <code>src/</code> package layout;
- manage dependencies with <code>pyproject.toml</code>;
- load database configuration from environment variables;
- fail fast when required configuration is missing;
- create PostgreSQL connections with Psycopg 3;
- represent database records as strongly typed Python dataclasses;
- extract telemetry using parameterized SQL;
- validate batch limits;
- keep extraction read-only;
- test configuration, connection creation, row mapping, and validation;
- establish clear boundaries between extraction and later pipeline stages.

The supplied source describes Stage 6 as the foundation connecting the telemetry source table to a decoupled Python batch pipeline. fileciteturn14file0L3-L15

## 2. The Problem

At this point in the system, the frontend can generate telemetry and <code>svc</code> can authenticate and persist those events in:

~~~text
public.frontend_telemetry_event
~~~

The operational application should not also become responsible for heavy analytical or data-processing work.

Instead, introduce a separate batch-processing package:

~~~text
svc
 |
 | writes telemetry
 v
PostgreSQL
 |
 | public.frontend_telemetry_event
 v
payment-telemetry-pipeline
 |
 +--> extract
 +--> validate      [later]
 +--> stage         [later]
 +--> checkpoint   [later]
 +--> audit        [later]
~~~

The source explicitly states that the Python pipeline provides asynchronous isolation from the operational OLTP APIs. fileciteturn14file0L8-L15

## 3. Target Architecture

~~~text
PostgreSQL
public.frontend_telemetry_event
          |
          | SELECT
          v
    extract_events()
          |
          v
list[TelemetryEvent]
          |
          +--> Stage 9: staging
          +--> Stage 10: validation
          +--> Stage 11: incremental extraction
~~~

The extractor performs only the source read and object mapping. fileciteturn14file0L50-L63

## 4. Step 1 — Bootstrap the Python Package

Create the project using a modern Python package layout:

~~~text
payment-telemetry-pipeline/
├── pyproject.toml
├── .env.example
├── src/
│   └── payment_telemetry/
│       ├── __init__.py
│       ├── config.py
│       ├── db.py
│       └── extractor.py
└── tests/
    ├── test_package.py
    ├── test_config.py
    ├── test_db.py
    └── test_extractor.py
~~~

The supplied implementation uses Python <code>&gt;=3.13</code>, Psycopg 3 binary support, and pytest. It also uses setuptools with the standard <code>src</code> layout. fileciteturn14file0L19-L25

A representative dependency configuration is:

~~~toml
[project]
requires-python = ">=3.13"
dependencies = [
    "psycopg[binary]>=3.3,<4",
]

[project.optional-dependencies]
test = [
    "pytest>=8",
]
~~~

Use the exact dependency/version configuration from the project when reproducing the implementation.

## 5. Step 2 — Add Environment Configuration

The pipeline needs PostgreSQL connection information.

The required environment variables are:

~~~text
DB_HOST
DB_PORT
DB_NAME
DB_USER
DB_PASS
~~~

Create an example environment file:

~~~text
.env.example
~~~

The example file documents the expected configuration without committing real credentials.

Do not place real passwords in source control.

## 6. Step 3 — Implement a Typed Database Configuration

Create:

~~~text
src/payment_telemetry/config.py
~~~

Define a <code>DatabaseConfig</code> representation and a <code>get_database_config()</code> function.

Its responsibility is:
1. read the required environment variables;
2. verify every mandatory variable exists;
3. construct the database configuration;
4. raise an explicit <code>RuntimeError</code> if configuration is incomplete.

The supplied implementation deliberately fails immediately rather than allowing an invalid database connection attempt to happen later. fileciteturn14file0L32-L36

The conceptual flow is:

~~~text
process environment
       |
       v
get_database_config()
       |
       +-- DB_HOST ?
       +-- DB_PORT ?
       +-- DB_NAME ?
       +-- DB_USER ?
       +-- DB_PASS ?
       |
       +-- missing -> RuntimeError
       |
       +-- complete -> DatabaseConfig
~~~

### Why Fail Fast?

Without validation, a missing environment variable might result in:
- confusing connection errors;
- accidental defaults;
- failures far away from the configuration problem;
- harder CI/CD diagnosis.

The pipeline should fail at the configuration boundary instead.

## 7. Step 4 — Implement PostgreSQL Connection Management

Create:

~~~text
src/payment_telemetry/db.py
~~~

Implement <code>get_connection()</code> using Psycopg 3.

The function should obtain the validated database configuration and create a PostgreSQL connection using Psycopg's connection/context-management facilities.

The source specifically uses Psycopg 3 and identifies context-manager transaction semantics as part of the driver choice. fileciteturn14file0L41-L45

The conceptual flow is:

~~~text
get_connection()
      |
      v
get_database_config()
      |
      v
DatabaseConfig
      |
      v
psycopg connection
~~~

Keep connection creation separate from extraction logic.

This gives the pipeline a reusable database boundary:

~~~text
config.py
   |
   v
db.py
   |
   v
extractor.py
~~~

## 8. Why Psycopg 3?

The source selected Psycopg 3 instead of the legacy Psycopg 2 driver.

The stated reasons are:
- native binary protocol performance;
- structured type support;
- first-class context-manager transaction semantics.

This is a project design decision recorded by the Stage 6 implementation, not a requirement that every Python pipeline must use Psycopg 3. fileciteturn14file0L41-L44

## 9. Step 5 — Define the Telemetry Domain Model

Create:

~~~text
src/payment_telemetry/extractor.py
~~~

Define a strongly typed <code>TelemetryEvent</code> dataclass.

The supplied implementation contains these fields:

~~~text
event_id
user_id
event_name
event_version
occurred_at
route
properties
received_at
~~~

The purpose is to establish a Python representation of the source database record.

Conceptually:

~~~text
PostgreSQL row
      |
      v
TelemetryEvent
      |
      v
Python pipeline
~~~

The source explicitly defines these fields as the Stage 6 domain model. fileciteturn14file0L34-L37

## 10. Why Use a Domain Dataclass?

Do not make every downstream pipeline stage operate directly on raw cursor tuples.

Raw tuples are positional:

~~~text
(row[0], row[1], row[2], ...)
~~~

A domain model is explicit:

~~~text
event.event_id
event.user_id
event.event_name
event.occurred_at
event.properties
~~~

This makes later stages easier to understand and reduces accidental column-position mistakes.

The dataclass also becomes the contract between extraction and subsequent processing stages.

## 11. Step 6 — Implement Batch Extraction

Implement <code>extract_events(conn, limit=1000)</code>.

The extractor queries:

~~~text
public.frontend_telemetry_event
~~~

The source query selects:

~~~sql
SELECT
    event_id,
    user_id,
    event_name,
    event_version,
    occurred_at,
    route,
    properties,
    received_at
FROM public.frontend_telemetry_event
ORDER BY received_at ASC
LIMIT %s
~~~

The supplied Stage 6 architecture specifies this ordering and parameterized limit. fileciteturn14file0L50-L63

The resulting cursor rows are mapped into <code>list[TelemetryEvent]</code>.

## 12. Why Order by <code>received_at</code>?

Stage 6 establishes extraction order using:

~~~sql
ORDER BY received_at ASC
~~~

This provides deterministic oldest-first batch extraction for the foundational extractor.

It is important not to confuse this with the later incremental-processing strategy.

The source explicitly defers keyset pagination and checkpointing to Stage 11. fileciteturn14file0L117-L121

## 13. Step 7 — Parameterize the Query

Do not construct SQL by interpolating the limit.

Bad:

~~~python
query = f"SELECT ... LIMIT {limit}"
~~~

Instead, use a database parameter:

~~~sql
LIMIT %s
~~~

and pass the value separately through the database driver.

The source explicitly identifies parameterized extraction as an SQL-injection safety requirement. fileciteturn14file0L43-L46

The general rule is:

~~~text
SQL structure -> query string
Runtime value -> parameter
~~~

This separation should remain standard throughout the pipeline.

## 14. Step 8 — Validate the Batch Limit

The extractor must reject invalid limits.

Required rule:

~~~text
limit <= 0
      |
      v
ValueError
~~~

Valid example:

~~~text
limit = 1000
      |
      v
execute extraction
~~~

Invalid examples:

~~~text
limit = 0
limit = -1
limit = -100
~~~

The source explicitly requires the extraction limit to be a positive integer and raises <code>ValueError</code> for non-positive values. fileciteturn14file0L43-L46

## 15. Architecture and Data Flow

The complete Stage 6 flow is:

~~~text
Environment
    |
    v
get_database_config()
    |
    +-- missing config -> RuntimeError
    |
    v
get_connection()
    |
    v
PostgreSQL
    |
    v
public.frontend_telemetry_event
    |
    | SELECT ... ORDER BY received_at ASC LIMIT %s
    v
extract_events()
    |
    +-- validate limit
    +-- execute parameterized query
    +-- read cursor rows
    +-- map rows
    v
list[TelemetryEvent]
~~~

The extractor is intentionally read-only. fileciteturn14file0L68-L70

## 16. Transaction Safety

Stage 6 performs read-only extraction.

There are no INSERT, UPDATE, DELETE, or staging writes in this layer.

The source explicitly states that the extractor executes read-only <code>SELECT</code> queries and does not perform database mutations. fileciteturn14file0L68-L70

This is an important architectural boundary:

~~~text
Stage 6
   |
   +--> READ source
   |
   X--> WRITE source
~~~

Later stages will introduce controlled writes into processing/staging structures.

## 17. Idempotency

The Stage 6 read itself is idempotent.

Running <code>extract_events(conn, limit=1000)</code> does not modify the source rows.

The same source state can therefore be read multiple times without changing the database.

The source explicitly describes database reads as inherently idempotent because extraction does not mutate row state. fileciteturn14file0L74-L76

However, do not interpret this as the complete incremental-processing strategy.

Checkpointing and keyset pagination are explicitly deferred to Stage 11.

## 18. Integration With the Existing Pipeline

Stage 6 connects directly to the telemetry database created by the earlier application stages.

~~~text
Frontend
   |
   v
svc
   |
   v
public.frontend_telemetry_event
   |
   v
Stage 6 extractor
   |
   v
TelemetryEvent objects
   |
   +--> Stage 9 staging
   +--> Stage 10 validation
   +--> Stage 11 incremental extraction
~~~

The source identifies Stage 4's <code>public.frontend_telemetry_event</code> table as the source and the in-memory <code>TelemetryEvent</code> instances as the output. fileciteturn14file0L80-L83

## 19. Database Schema Boundary

Stage 6 does **not** introduce a database migration.

It reads the table created previously:

~~~text
public.frontend_telemetry_event
~~~

The source explicitly states that no new migrations were introduced in this stage. fileciteturn14file0L87-L89

This matters because the pipeline foundation should consume the operational source rather than prematurely redesigning its schema.

## 20. Testing Strategy

The initial Stage 6 test suite contains five tests:

~~~text
test_package_import
test_database_config_requires_all_values
test_get_connection_uses_database_config
test_extract_events_maps_database_rows
test_extract_events_rejects_invalid_limit
~~~

The source records all five as passing. fileciteturn14file0L93-L100

### Test 1 — Package Import

Verify that the package installs correctly and its namespace resolves.

This catches basic packaging problems before pipeline logic is tested.

### Test 2 — Configuration Requires All Values

Remove one or more mandatory environment variables.

Expected result:

~~~text
get_database_config()
      |
      v
RuntimeError
~~~

### Test 3 — Connection Uses Database Configuration

Mock the Psycopg connection call and verify the expected configuration values are passed through.

This confirms the configuration layer is actually connected to the connection layer.

### Test 4 — Database Rows Map to <code>TelemetryEvent</code>

Provide representative cursor tuples and verify every field is mapped to the correct domain-model field.

This is particularly important because positional database rows are easy to map incorrectly.

### Test 5 — Invalid Limit Is Rejected

Verify:

~~~text
limit = 0   -> ValueError
limit < 0   -> ValueError
~~~

The source confirms these five foundational tests and their intended coverage. fileciteturn14file0L93-L100

## 21. PostgreSQL Validation

The implementation also validated the query syntax and column compatibility against PostgreSQL 16 ANSI standards. fileciteturn14file0L103-L106

The important practical lesson is to verify the extractor against the actual source schema instead of assuming that column names or types match the application model.

## 22. Privacy and Data Protection

The pipeline reads potentially sensitive telemetry data, so credential and data handling must be controlled.

The Stage 6 implementation establishes two explicit rules:

### Database credentials

Credentials come only from process environment variables.

They are excluded from Git through <code>.gitignore</code>.

~~~text
Environment
    |
    v
Database credentials
    |
    X--> Git repository
~~~

### Extracted properties

The JSON <code>properties</code> field is loaded into memory but is not logged to disk or the console.

~~~text
PostgreSQL properties
        |
        v
Python memory
        |
        X--> raw console logging
        X--> raw file logging
~~~

These rules are explicitly recorded in the source. fileciteturn14file0L110-L113

## 23. Common Implementation Mistakes

### Mistake 1 — Putting database credentials in source code

Bad:

~~~python
DB_PASS = "real-password"
~~~

Use environment-driven configuration instead.

### Mistake 2 — Silently accepting missing configuration

Do not allow a missing variable to become a confusing downstream connection failure.

Fail at <code>get_database_config()</code>.

### Mistake 3 — Building SQL with string interpolation

Bad:

~~~python
f"... LIMIT {limit}"
~~~

Use parameterized SQL.

### Mistake 4 — Accepting zero or negative batch sizes

Reject invalid limits before touching the database.

### Mistake 5 — Writing to the source table

Stage 6 is an extraction layer.

Do not mutate <code>public.frontend_telemetry_event</code>.

### Mistake 6 — Logging raw telemetry properties

The <code>properties</code> field may contain sensitive values. Do not print or persist raw extracted payloads merely for debugging.

### Mistake 7 — Implementing checkpointing too early

Stage 6 intentionally uses a simple ordered batch extractor.

Keyset pagination and incremental checkpointing belong to Stage 11 according to the supplied implementation plan. fileciteturn14file0L117-L121

### Mistake 8 — Adding staging logic here

The Stage 6 extractor should produce <code>TelemetryEvent</code> objects. Staging is deferred to Stage 9.

## 24. Independent Implementation Exercise

Build a small Python batch extractor for another PostgreSQL-backed event table.

Requirements:

1. Use Python 3.13 or the project's required Python version.
2. Use a <code>src/</code> package layout.
3. Define dependencies in <code>pyproject.toml</code>.
4. Load database configuration from environment variables.
5. Fail fast when required variables are missing.
6. Create a reusable PostgreSQL connection factory.
7. Define a typed event dataclass.
8. Extract events with parameterized SQL.
9. Order extraction deterministically.
10. Require a positive batch limit.
11. Return typed domain objects rather than raw tuples.
12. Keep extraction read-only.
13. Do not log raw event properties.
14. Write unit tests for configuration, connection creation, mapping, and limit validation.

Then extend the exercise by deliberately attempting to add checkpointing. Before implementing it, compare your design against the stated Stage 11 boundary and identify what additional state the pipeline would need.

## 25. Validation Checklist

### Package
- [ ] Python package uses a <code>src/</code> layout.
- [ ] <code>pyproject.toml</code> defines project metadata and dependencies.
- [ ] Psycopg 3 binary dependency is configured.
- [ ] pytest is configured.
- [ ] Package imports successfully.

### Configuration
- [ ] <code>DB_HOST</code> is required.
- [ ] <code>DB_PORT</code> is required.
- [ ] <code>DB_NAME</code> is required.
- [ ] <code>DB_USER</code> is required.
- [ ] <code>DB_PASS</code> is required.
- [ ] Missing configuration raises <code>RuntimeError</code>.
- [ ] Real credentials are excluded from Git.

### Database
- [ ] Connection creation is isolated in <code>db.py</code>.
- [ ] Psycopg 3 is used.
- [ ] Connection parameters come from validated configuration.

### Extraction
- [ ] <code>TelemetryEvent</code> contains all eight source fields.
- [ ] Extractor reads <code>public.frontend_telemetry_event</code>.
- [ ] Query uses parameterized SQL.
- [ ] Rows are ordered by <code>received_at ASC</code>.
- [ ] Batch limit is positive.
- [ ] Invalid limits raise <code>ValueError</code>.
- [ ] Rows map into typed domain objects.
- [ ] Source rows are not mutated.

### Privacy
- [ ] Credentials are not hard-coded.
- [ ] Raw properties are not logged.
- [ ] Extracted data is not unnecessarily written to disk.

### Tests
- [ ] Package import test passes.
- [ ] Missing configuration test passes.
- [ ] Connection configuration test passes.
- [ ] Row-to-domain mapping test passes.
- [ ] Invalid-limit test passes.
- [ ] Full foundational suite passes.

## 26. Scope Boundaries

### Implemented in Stage 6
- Python package bootstrap;
- modern packaging;
- environment-driven configuration;
- configuration validation;
- PostgreSQL connection management;
- <code>TelemetryEvent</code> domain model;
- initial batch extractor;
- foundational unit tests.

### Deferred to Stage 9
- staging layer;
- <code>app_version</code> column inclusion.

### Deferred to Stage 10
- telemetry event validation logic.

### Deferred to Stage 11
- keyset pagination;
- incremental checkpointing.

These boundaries are explicitly documented by the source implementation. fileciteturn14file0L117-L121

## 27. What You Should Understand Before the Next Recipe

After this recipe, the pipeline has crossed an important architectural boundary:

~~~text
Operational application
        |
        v
PostgreSQL telemetry source
        |
        v
Python batch-processing package
        |
        v
Typed TelemetryEvent objects
~~~

The key engineering lessons are:
- configuration should fail fast;
- database access should have a dedicated boundary;
- extraction should use parameterized SQL;
- raw database rows should become explicit domain objects;
- foundational extraction should remain read-only;
- privacy rules apply during data engineering, not only at the API layer;
- simple extraction and incremental processing are different problems;
- staging, validation, and checkpointing should be introduced deliberately rather than prematurely.

The supplied Stage 6 implementation records the foundation as complete with five passing tests. fileciteturn14file0L93-L100

## Source Traceability

This recipe is derived from the supplied Stage 6 implementation record:
- Source commit: <code>0fa3f6f</code>
- Subject: <code>feat: add payment telemetry pipeline foundation</code>
- Repository: <code>devops</code>
- Overview and technical objective: fileciteturn14file0L3-L15
- Implementation sequence: fileciteturn14file0L19-L26
- Technical changes: fileciteturn14file0L30-L37
- Design decisions: fileciteturn14file0L41-L46
- Architecture and data flow: fileciteturn14file0L50-L63
- Transaction safety and idempotency: fileciteturn14file0L68-L76
- Pipeline integration and schema boundary: fileciteturn14file0L80-L89
- Tests and PostgreSQL validation: fileciteturn14file0L93-L106
- Privacy and scope boundaries: fileciteturn14file0L110-L121