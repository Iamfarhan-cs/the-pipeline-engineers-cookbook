# Chapter 9 — How to Read a Data Engineering Repository

Until now, we focused on how data pipelines work.

Now we change the skill.

In real work, you will often join a project that already exists.

You may be given a repository and a task such as:

- Add a new ingestion source.
- Fix duplicate processing.
- Add a database migration.
- Change a pipeline stage.
- Add retry handling.
- Investigate missing data.
- Add tests.
- Improve observability.

The first mistake is to start coding immediately.

Before changing an existing pipeline, you need to understand how the repository is organized.

You need to answer questions such as:

- Where does the application start?
- Where is the pipeline entry point?
- Where is database access implemented?
- Where are migrations stored?
- Where are tests?
- Where is configuration loaded?
- How does Docker start the system?
- How does CI run the tests?
- Where does data enter?
- Where does data leave?

This chapter gives you a practical method for answering those questions.

---

# What Does It Mean to Read a Repository?

Reading a repository does not mean opening every file.

It means building a useful mental model of the system.

A good first pass should help you understand:

```text
Repository
   |
   +-- Application entry point
   +-- Pipeline stages
   +-- Database layer
   +-- Configuration
   +-- Tests
   +-- Infrastructure
   +-- CI/CD
```

You are trying to understand relationships between these parts.

For example:

```text
API / Scheduler / Worker
          |
          v
      Pipeline Code
          |
          v
      Database Layer
          |
          v
       PostgreSQL
```

The exact architecture will differ from repository to repository.

The investigation method stays useful.

---

# Why Repository Investigation Matters

Existing systems contain assumptions.

A function may depend on a database transaction.

A table may be created by a migration rather than application startup.

A worker may receive data from a queue instead of an HTTP endpoint.

A configuration value may come from an environment variable.

A test fixture may create the database schema automatically.

If you do not understand these relationships, a small code change can have unexpected effects.

Repository investigation reduces that risk.

---

# The Investigation Mindset

When entering an unfamiliar repository, do not immediately ask:

> Which file should I edit?

Start with:

> How does this system work?

Then ask:

1. Where does execution begin?
2. How does data enter?
3. How is data processed?
4. Where is data stored?
5. How are failures handled?
6. How is the system tested?
7. How is it configured?
8. How is it built and deployed?

Only after that should you decide which files need changes.

---

# Start With the Repository Root

The repository root is your map.

A typical Data Engineering repository may contain directories such as:

```text
project/
|
+-- src/
+-- tests/
+-- migrations/
+-- config/
+-- scripts/
+-- docs/
+-- docker/
+-- .github/
+-- Dockerfile
+-- docker-compose.yml
+-- pyproject.toml
+-- README.md
```

**GENERIC EXAMPLE**

Do not assume every repository has this exact structure.

Your first job is to discover what actually exists.

---

# Step 1 — Read the README

The README is usually the first document to inspect.

It may tell you:

- What the project does
- How to install dependencies
- How to start the application
- How to run tests
- Required environment variables
- Database setup
- Docker commands
- Development workflow

Do not assume the README is complete.

It is a starting point, not necessarily the final source of truth.

Later, verify important claims against the actual code and configuration.

---

# Step 2 — Inspect the Top-Level Structure

After reading the README, inspect the repository tree.

You want to identify the major areas of the project.

For example:

```text
src/       -> application code
tests/     -> automated tests
migrations/ -> database schema changes
scripts/   -> operational or development scripts
docs/      -> project documentation
.github/   -> GitHub workflows
```

These meanings are common patterns, not guaranteed rules.

A directory named `src` could contain application code, libraries, or something else.

Open files before deciding what a directory means.

---

# Step 3 — Identify the Technology Stack

Before following application code, identify the main technologies.

Look for dependency and configuration files.

Common examples include:

| File | What it may tell you |
|---|---|
| `pyproject.toml` | Python project configuration and dependencies |
| `requirements.txt` | Python dependencies |
| `package.json` | Node.js dependencies and scripts |
| `go.mod` | Go dependencies |
| `pom.xml` | Java/Maven configuration |
| `build.gradle` | Gradle configuration |
| `Dockerfile` | Container build process |
| `docker-compose.yml` | Local multi-service setup |

These are generic examples.

The actual repository may use different files or tools.

---

# Why the Technology Stack Matters

The stack changes how you investigate the repository.

For example, a Python pipeline may use:

- Python modules
- PostgreSQL
- psycopg
- SQLAlchemy
- pytest
- Docker

A Java pipeline may use:

- Java packages
- Maven or Gradle
- JDBC
- JUnit
- Spring

The investigation questions remain similar.

The files and conventions change.

---

# Step 4 — Find the Entry Point

One of the most important questions is:

> What starts this system?

Possible entry points include:

- Command-line script
- Python module
- Web application
- Worker
- Scheduler
- Airflow DAG
- Kafka consumer
- Batch job
- Shell script
- Container entrypoint

Do not assume there is only one entry point.

A repository can contain several independent processes.

For example:

```text
Repository
   |
   +-- API service
   +-- Worker
   +-- Scheduler
   +-- Migration command
   +-- Data quality job
```

Each may have its own entry point.

---

# How to Find an Entry Point

Start by looking for commands in:

- README files
- `Makefile`
- package scripts
- Dockerfiles
- Docker Compose files
- CI workflows
- shell scripts
- deployment configuration

For Python, also look for patterns such as:

```python
if __name__ == "__main__":
    ...
```

**GENERIC EXAMPLE**

You may also find command-line entry points configured in project metadata.

The important point is to trace the actual startup path rather than guessing from filenames.

---

# Step 5 — Follow the Execution Path

Once you find an entry point, follow what it calls.

Imagine:

```text
main()
  |
  v
run_pipeline()
  |
  v
fetch_data()
  |
  v
validate()
  |
  v
store_data()
```

Now you have a basic execution path.

Keep following the important branches.

You do not need to understand every helper function immediately.

Focus on the functions that control:

- Data movement
- State changes
- Database writes
- External calls
- Error handling
- Retries
- Transactions

---

# Step 6 — Find Where Data Enters

A Data Engineering repository must have some way of receiving data.

Search for input boundaries such as:

- HTTP requests
- API clients
- File readers
- Object storage downloads
- Kafka consumers
- Queue consumers
- Database reads
- Scheduled queries
- Webhooks

Then ask:

> What happens immediately after the data arrives?

Follow the data.

---

# Step 7 — Find Where Data Is Validated

Look for validation logic.

Possible patterns include:

- Schema classes
- Validation functions
- Pydantic models
- JSON schema
- Database constraints
- SQL checks
- Data quality functions
- Conditional checks

Do not assume a function named `validate` is the only validation.

Some validation may happen implicitly through database constraints or serializers.

---

# Step 8 — Find Where Data Is Stored

Search for database access.

Common signs include:

- PostgreSQL connection setup
- Database client creation
- SQL queries
- ORM models
- Repository classes
- DAO classes
- Transactions
- Insert/update functions
- Connection pools

Then answer:

> What table or storage location receives the data?

Do not stop at finding a database connection.

You need to trace the write operation.

---

# Step 9 — Find the Database Schema

Application code tells you what the program does.

Database schema tells you what the database expects.

Look for:

- Migration files
- SQL schema files
- ORM models
- Table definitions
- Index definitions
- Constraints
- Foreign keys
- Views

Later chapters will examine database investigation in more detail.

For now, the goal is simply to locate these parts.

---

# Step 10 — Find Migrations

Migrations describe how database structure changes over time.

They can tell you:

- When tables were introduced
- When columns were added
- When indexes were created
- When constraints changed
- How the current schema evolved

Suppose application code uses:

```text
processing_status
```

Before assuming the column exists everywhere, find the migration that created it or changed it.

This is especially important when investigating production bugs.

---

# Step 11 — Find Tests

Tests are one of the best sources of system knowledge.

They often show:

- Expected behavior
- Input formats
- Database setup
- Error handling
- Retry behavior
- Idempotency assumptions
- Processing states
- API contracts

Tests can sometimes explain behavior more clearly than comments.

---

# Read Tests as Documentation

Suppose you find a test named conceptually like:

```text
test_duplicate_event_is_ignored
```

That test tells you something important.

It suggests the system has an expectation around duplicate events.

Then inspect the test body.

Find out:

- What input is created?
- What identifier is duplicated?
- What operation is called?
- What database state is expected?
- Is an error expected?
- Is the second operation ignored?

Tests are evidence.

Do not infer more than the test actually proves.

---

# Step 12 — Find Configuration

Configuration controls how the application behaves in different environments.

Look for:

- Environment variables
- `.env` examples
- Configuration modules
- YAML files
- TOML files
- JSON configuration
- Docker environment sections
- CI secrets

Typical configuration categories include:

- Database connection
- API URL
- Credentials
- Retry settings
- Timeouts
- Feature flags
- Logging
- Environment name

Never assume that a value in local configuration is the production value.

---

# Step 13 — Find Docker Configuration

Docker configuration can reveal the architecture quickly.

Look for:

- `Dockerfile`
- `docker-compose.yml`
- Compose overrides
- Entrypoint scripts
- Environment variables
- Mounted volumes
- Service dependencies
- Health checks

For example:

```text
docker compose
     |
     +-- application
     +-- postgres
     +-- queue
     +-- monitoring
```

**GENERIC EXAMPLE**

Even when the application code is large, Docker configuration can provide a quick overview of the services required to run it.

---

# Step 14 — Find CI/CD

CI/CD configuration shows how the project is tested and delivered.

Look for:

- GitHub Actions
- GitLab CI
- Jenkins
- CircleCI
- Other CI systems

Common CI steps include:

```text
Checkout
  |
  v
Install dependencies
  |
  v
Lint
  |
  v
Run tests
  |
  v
Build
  |
  v
Deploy
```

Do not assume this exact sequence.

Read the actual workflow.

---

# Build a Repository Map

After the first investigation pass, write down what you found.

For example:

```text
Repository
  |
  +-- Entry point -> worker.py
  |
  +-- Input -> API client
  |
  +-- Validation -> validator module
  |
  +-- Storage -> PostgreSQL repository
  |
  +-- Schema -> migrations
  |
  +-- Tests -> tests/
  |
  +-- Config -> environment variables
  |
  +-- Runtime -> Docker Compose
  |
  +-- CI -> GitHub Actions
```

**GENERIC EXAMPLE**

The names above are illustrative.

Your actual repository map should use verified paths and components.

---

# Build a Data Flow Map

A repository map tells you where code lives.

A data flow map tells you how data moves.

For example:

```text
External API
     |
     v
API client
     |
     v
Raw storage
     |
     v
Validation
     |
     v
Staging table
     |
     v
Transformation
     |
     v
Curated table
```

**GENERIC EXAMPLE**

This map is often more useful than a long list of filenames.

---

# Build a Failure Map

Do not map only the happy path.

Ask where the system can fail.

For example:

```text
API request
    |
    +---- timeout ----> retry
    |
    +---- invalid data ----> quarantine
    |
    +---- success ----> validation
                              |
                              +---- fail ----> error state
                              |
                              +---- pass ----> database
```

**GENERIC EXAMPLE**

A failure map is especially useful when a task involves retries, recovery, replay, or missing data.

---

# Build a State Map

If the pipeline tracks processing state, identify the states.

For example:

```text
RECEIVED
   |
   v
PROCESSING
   |
   +----> COMPLETED
   |
   +----> RETRY_PENDING
   |          |
   |          v
   |      PROCESSING
   |
   +----> FAILED
```

**GENERIC EXAMPLE**

Then find where each transition happens in the code.

---

# Search Before Opening Files

A common repository investigation mistake is opening files randomly.

Search is usually faster.

Useful search targets include:

- Function names
- Table names
- Environment variable names
- Error messages
- API paths
- Event names
- Status values
- Migration numbers
- Test names

For example, if a ticket mentions:

```text
processing_status
```

search the repository for that exact term.

You may find:

```text
migration
model
SQL query
service
test
documentation
```

This creates a much faster path through an unfamiliar codebase.

---

# Search for Behavior, Not Only Names

Sometimes you do not know the function name.

Search for behavior instead.

For database writes, search for patterns such as:

- `INSERT`
- `UPDATE`
- `SELECT`
- transaction calls
- database cursor execution

For HTTP calls, search for:

- HTTP client usage
- request methods
- endpoint URLs
- timeout configuration

For retries, search for:

- retry
- backoff
- attempt
- timeout
- exception handling

The exact search terms depend on the language and framework.

---

# Follow One Record

A powerful investigation technique is to follow one record through the system.

Start with a known identifier.

For example:

```text
event_id = 123
```

Then search for that identifier's usage or the code responsible for processing that type of record.

Trace:

```text
Input
  -> validation
  -> transformation
  -> database write
  -> processing state
  -> output
```

This can reveal the actual path more clearly than reading the repository from top to bottom.

---

# Follow One Failure

The same method works for failures.

Start with an actual error message.

For example:

```text
database connection timeout
```

Search for the error or the code that can produce it.

Then follow:

```text
Error
  -> exception
  -> caller
  -> retry logic
  -> state update
  -> logging
  -> test
```

This creates a failure path through the repository.

---

# Do Not Trust File Names

A file called `pipeline.py` does not necessarily contain the whole pipeline.

A file called `database.py` may only create connections.

A file called `utils.py` may contain important business logic.

A directory called `tests` may contain integration tests that reveal critical architecture.

Names are clues.

Code behavior is evidence.

---

# Read the Smallest Useful Set of Files

You do not need to read the entire repository before starting an investigation.

Start with the smallest set that answers the current question.

For a database bug, that might be:

```text
1. Entry point
2. Service function
3. Database function
4. Migration
5. Relevant test
```

For a CI failure, it might be:

```text
1. CI workflow
2. Test command
3. Dependency configuration
4. Relevant test
```

Expand the investigation only when necessary.

---

# Repository Investigation Is Iterative

You will often discover new information while investigating.

For example:

```text
Start
  |
  v
Find entry point
  |
  v
Find database call
  |
  v
Find migration
  |
  v
Discover trigger
  |
  v
Return to entry point
```

This is normal.

Do not expect to understand the entire system in one pass.

---

# A Practical Investigation Sequence

Here is a useful sequence for an unfamiliar repository.

### Step 1 — Read the README

Understand the project purpose and basic commands.

### Step 2 — Inspect the root

Identify major directories and configuration files.

### Step 3 — Identify the stack

Find language, framework, database, queue, orchestration, and infrastructure tools.

### Step 4 — Find entry points

Determine what starts each relevant process.

### Step 5 — Follow data input

Find where data enters.

### Step 6 — Follow processing

Find validation, transformation, and business logic.

### Step 7 — Follow storage

Find database writes and external outputs.

### Step 8 — Find schema and migrations

Understand what the database expects.

### Step 9 — Read relevant tests

Learn expected behavior.

### Step 10 — Inspect configuration

Understand environment-dependent behavior.

### Step 11 — Inspect Docker and infrastructure

Understand how components run together.

### Step 12 — Inspect CI/CD

Understand how the repository is tested and delivered.

### Step 13 — Draw the maps

Create repository, data-flow, failure, and state maps where useful.

---

# Repository Investigation for a Ticket

Suppose you receive this task:

> Add retry handling for failed database writes.

Do not immediately edit the database function.

Investigate:

1. Where does the database write happen?
2. Who calls it?
3. Is it already inside a transaction?
4. What exceptions can it raise?
5. Is the operation idempotent?
6. Does the caller already retry?
7. Is there processing state?
8. How are failures logged?
9. Are there existing retry utilities?
10. Are there tests for database failures?
11. How does Docker provide PostgreSQL locally?
12. How does CI run the relevant tests?

Now the task becomes much more concrete.

---

# Repository Investigation for a Missing Data Problem

Suppose someone reports:

> Yesterday's records are missing.

Start with the data flow.

Ask:

1. Did the source produce the records?
2. Did acquisition receive them?
3. Were they stored in raw form?
4. Did validation reject them?
5. Did staging receive them?
6. Did transformation process them?
7. Did the final database write succeed?
8. Did processing state mark them as complete?
9. Did a downstream query filter them?

Then inspect the relevant code, database tables, logs, tests, and configuration.

This is repository investigation applied to an incident.

---

# What Not to Do

## Mistake 1: Start coding immediately

You may solve the wrong problem.

## Mistake 2: Read every file

You will spend time on unrelated code.

## Mistake 3: Trust the README completely

Documentation can become outdated.

Verify important behavior.

## Mistake 4: Ignore tests

You may miss the actual expected behavior.

## Mistake 5: Ignore migrations

You may misunderstand the database schema.

## Mistake 6: Ignore configuration

The same code can behave differently across environments.

## Mistake 7: Ignore infrastructure

Docker and CI can reveal important runtime assumptions.

## Mistake 8: Look only at the happy path

Production problems often happen in the failure path.

---

# Production Considerations

Repository investigation becomes especially important in production systems.

Before changing a pipeline, identify:

- The process that runs it
- The source system
- The destination system
- Database dependencies
- Queue or broker dependencies
- Configuration sources
- Secrets handling
- Retry behavior
- Failure states
- Monitoring
- Alerting
- Deployment process
- Rollback process
- Relevant tests

A code change is only one part of a production change.

---

# Practical Repository Investigation Checklist

Before modifying an unfamiliar Data Engineering repository, ask:

- [ ] Have I read the README?
- [ ] Have I inspected the repository root?
- [ ] Do I know the main technologies?
- [ ] Do I know the relevant entry point?
- [ ] Do I know where data enters?
- [ ] Do I know where validation happens?
- [ ] Do I know where processing happens?
- [ ] Do I know where data is stored?
- [ ] Have I found the relevant schema?
- [ ] Have I found the relevant migrations?
- [ ] Have I found the relevant tests?
- [ ] Do I understand the relevant configuration?
- [ ] Do I understand the local runtime?
- [ ] Do I know how CI runs the code?
- [ ] Have I traced the happy path?
- [ ] Have I traced the failure path?
- [ ] Have I identified processing states?
- [ ] Have I checked retry and idempotency behavior?
- [ ] Can I explain the data flow in a simple diagram?

---

# What You Learned

In this recipe, you learned:

- Repository investigation is a core Data Engineering skill.
- The goal is to build a mental model before changing code.
- The repository root provides the first map of the project.
- The README is a useful starting point but important behavior should be verified.
- Entry points show where execution begins.
- Data-flow investigation shows where data enters, changes, and leaves.
- Database investigation shows where data is stored and how the schema is maintained.
- Tests reveal expected behavior.
- Configuration explains environment-dependent behavior.
- Docker and CI/CD reveal how the system runs and is delivered.
- Searching for behavior can be more useful than searching only for filenames.
- Following one record or one failure is a powerful investigation technique.
- Repository maps, data-flow maps, failure maps, and state maps make complex systems easier to understand.

The main lesson is:

**Before changing an existing pipeline, understand how the repository actually works. The fastest engineer is often not the one who starts coding first, but the one who finds the correct execution path first.**

---
