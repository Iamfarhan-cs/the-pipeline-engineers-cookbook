# Chapter 13 — Finding Tests

Tests are one of the best sources of information in an unfamiliar Data Engineering repository.

Documentation can become outdated.

Comments can become outdated.

Tests have a different advantage: they usually need to match behavior closely enough to execute successfully.

When investigating a repository, tests can help you answer questions such as:

- What is this function expected to do?
- What input is valid?
- What happens when input is invalid?
- Which database tables are required?
- What happens when a record already exists?
- How are failures handled?
- What happens when an external service fails?
- What does the pipeline do with bad data?
- Which behavior is considered important enough to protect?

This chapter explains how to find those tests and use them as part of repository investigation.

---

## 1. Why Tests Matter During Repository Investigation

Imagine you find a function called:

```text
process_event()
```

The function name tells you very little.

You could read the entire implementation.

But a test may immediately show the intended behavior:

```text
Input event
    ↓
process_event()
    ↓
database record created
```

Another test may show:

```text
Same event again
    ↓
process_event()
    ↓
no duplicate record
```

Now you know that duplicate processing matters.

Tests therefore help reveal the behavior that the repository expects.

---

## 2. What Counts as a Test?

A repository can contain many types of tests.

Common categories include:

- unit tests
- integration tests
- database tests
- API tests
- end-to-end tests
- pipeline tests
- contract tests
- data quality tests
- migration tests
- regression tests
- smoke tests
- performance tests

Not every repository uses all of them.

The first task is to discover what the repository actually contains.

Do not assume that a directory named `tests/` contains every test in the project.

Some tests may live beside the code they test.

For example:

```text
src/
├── ingestion/
│   ├── client.py
│   └── test_client.py
```

Others may use a central structure:

```text
tests/
├── unit/
├── integration/
└── e2e/
```

Search the repository rather than relying only on directory names.

---

## 3. Find the Test Framework

Before reading individual tests, identify the test framework.

Common examples include:

- pytest
- unittest
- JUnit
- Jest
- Vitest
- Mocha
- RSpec

These are generic examples.

Use the repository's actual dependencies and configuration to determine what it uses.

Look at files such as:

```text
pyproject.toml
requirements.txt
package.json
pom.xml
build.gradle
Makefile
tox.ini
pytest.ini
```

Also inspect CI configuration because the CI pipeline usually shows how tests are executed.

---

## 4. Find the Test Command

One of the first questions should be:

> How does this repository run its tests?

Search for commands such as:

```text
pytest
python -m pytest
npm test
npm run test
mvn test
gradle test
```

These are examples only.

The repository's actual command is the important one.

Look for it in:

- README files
- package scripts
- CI workflows
- Docker files
- shell scripts
- project configuration

A useful investigation path is:

```text
CI command
    ↓
Test runner
    ↓
Test discovery
    ↓
Test file
    ↓
Function under test
```

This tells you not only where tests exist, but how they are actually executed.

---

## 5. Find Test Directories

Start by listing the repository structure.

A generic repository might look like:

```text
project/
├── src/
│   ├── ingestion/
│   ├── processing/
│   └── storage/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── migrations/
├── Dockerfile
└── pyproject.toml
```

Now investigate each test area.

Ask:

1. What does this directory test?
2. Does it test one module or the entire pipeline?
3. Does it require a database?
4. Does it require Docker?
5. Does it call an external service?
6. Does it use fixtures?
7. Does it mock dependencies?

Do not judge a test only by its filename.

Read what it actually does.

---

## 6. Test Names Are Clues

Good test names can reveal expected behavior quickly.

Examples:

```text
test_insert_event
test_duplicate_event_is_ignored
test_invalid_payload_is_rejected
test_retry_on_timeout
test_failed_record_is_quarantined
```

These names immediately tell you what behavior the repository considers important.

However, names are only clues.

Read the test body before treating the name as proof.

A test named `test_retry` might verify only one retry.

Another test might verify exponential backoff.

The implementation details matter.

---

## 7. Follow a Test to the Code

A useful investigation technique is to start with a test and move toward the implementation.

Suppose you find:

```python
def test_duplicate_event_is_ignored():
    ...
```

Ask:

- Which function is being called?
- Which database table is involved?
- Which identifier is used for uniqueness?
- What happens on the first call?
- What happens on the second call?
- Is the duplicate detected in application code or by the database?

Your investigation may become:

```text
Test
 ↓
Pipeline function
 ↓
Repository/data-access function
 ↓
SQL
 ↓
Database constraint
```

This is one of the most useful ways to understand a repository.

---

## 8. Follow Code Back to the Tests

You can also investigate in the opposite direction.

Suppose you find this function:

```text
validate_payload()
```

Search the repository for:

```text
validate_payload
```

You may find several tests.

Now compare them.

For example:

```text
valid payload
invalid payload
missing required field
wrong data type
unexpected value
```

This can reveal the actual validation contract.

Tests often show edge cases that are not obvious from the function implementation.

---

## 9. Unit Tests

Unit tests usually focus on a small piece of code.

For example:

```text
input
  ↓
function
  ↓
output
```

A unit test may not require a real database or external API.

Generic example:

```python
def test_normalize_email():
    result = normalize_email("User@Example.COM")
    assert result == "user@example.com"
```

The exact code is only a generic example.

The important investigation question is:

> What behavior does this test define?

Unit tests are particularly useful for understanding:

- validation
- transformation
- parsing
- normalization
- calculations
- error handling
- small business rules

---

## 10. Integration Tests

Integration tests verify that multiple components work together.

For a Data Engineering system, this may mean:

```text
Pipeline
  ↓
Database
```

or:

```text
Pipeline
  ↓
External API
  ↓
Database
```

Integration tests are especially useful when investigating:

- database queries
- transactions
- migrations
- API clients
- message consumers
- storage systems
- pipeline stages

If a test uses a real database, it can reveal assumptions that unit tests cannot.

---

## 11. Database Tests

Database tests deserve special attention in Data Engineering repositories.

Look for tests that:

- create tables
- run migrations
- insert records
- update records
- query records
- verify constraints
- test transactions
- test duplicate handling
- test rollback behavior
- verify indexes or query behavior

Suppose a test does this:

```text
insert event
insert same event again
verify one record exists
```

That test is strong evidence that duplicate handling is part of the expected behavior.

Follow it into:

```text
test
 ↓
database operation
 ↓
constraint or SQL
 ↓
migration
```

This connects Chapter 12 and Chapter 13 directly.

---

## 12. Fixtures

Fixtures provide reusable test data or test setup.

They may create:

- database records
- sample payloads
- configuration
- temporary files
- mock responses
- test clients
- test databases

Generic example:

```text
fixture
  ↓
test input
  ↓
function under test
  ↓
assertion
```

When investigating tests, find the fixture definitions.

A test may appear simple because the difficult setup is hidden inside a fixture.

Ask:

- What data does the fixture create?
- Is the data realistic?
- Does it use fixed IDs?
- Does it create database state?
- Does it clean up afterward?
- Is the same fixture shared by many tests?

Fixtures can be as important as the test itself.

---

## 13. Mocking

Tests often replace external dependencies with mocks.

For example:

```text
Real API
    X
    |
Mock response
    ↓
Application
```

This allows a test to simulate conditions such as:

- successful API response
- timeout
- connection failure
- HTTP error
- invalid response
- rate limit

Mocks are useful, but they can hide integration problems.

When investigating a test, ask:

> Is this test checking the real dependency or a simulated dependency?

That distinction matters.

---

## 14. Tests for External APIs

Data pipelines often depend on external APIs.

Search for tests around:

- request construction
- authentication
- response parsing
- pagination
- rate limits
- timeouts
- retries
- malformed responses
- HTTP status handling

A good investigation follows the complete path:

```text
Test
 ↓
API client
 ↓
HTTP request
 ↓
response handling
 ↓
pipeline processing
```

If the test mocks the HTTP layer, inspect the mocked response carefully.

The response structure may reveal the expected external API contract.

---

## 15. Tests for Failure Behavior

Production systems are defined partly by how they handle failure.

Search for test names containing concepts such as:

```text
error
failure
timeout
retry
duplicate
invalid
missing
exception
rollback
quarantine
dead_letter
```

These tests can reveal the recovery design.

Example:

```text
temporary failure
      ↓
retry
      ↓
success
```

Another test might show:

```text
permanent failure
      ↓
no retry
      ↓
quarantine
```

This is often more useful than reading only the happy-path test.

---

## 16. Find Tests for Idempotency

Idempotency is especially important in Data Engineering.

Search for tests involving:

- duplicate events
- repeated requests
- repeated jobs
- unique identifiers
- upserts
- conflict handling
- reruns
- replay

Generic test sequence:

```text
Run pipeline
    ↓
Run pipeline again
    ↓
Compare database state
```

The expected result should depend on the pipeline design.

The important point is to determine what the repository actually expects.

Do not assume that every pipeline is idempotent.

Look for the test evidence.

---

## 17. Find Tests for Data Quality

Data quality rules are often encoded directly in tests.

Search for tests covering:

- missing values
- duplicate records
- invalid formats
- invalid ranges
- referential integrity
- unexpected row counts
- freshness
- schema mismatches

Example:

```text
Input data
    ↓
Validation
    ↓
invalid record
    ↓
rejected or quarantined
```

Find out what the repository actually does with the invalid record.

Do not assume that validation always stops the entire pipeline.

Some systems reject the batch.

Others isolate the bad record and continue.

Tests can tell you which behavior was intended.

---

## 18. End-to-End Tests

End-to-end tests exercise a larger workflow.

Generic example:

```text
Input event
    ↓
API
    ↓
Ingestion
    ↓
Validation
    ↓
Database
    ↓
Verification
```

These tests are useful for understanding the actual system flow.

They can answer:

- Which components participate?
- Which database is used?
- Which configuration is required?
- What output is expected?
- Which side effects occur?

End-to-end tests are usually more expensive than unit tests.

That is one reason repositories often have fewer of them.

---

## 19. Test Configuration

Tests may use different configuration from production.

Look for:

- test environment variables
- test configuration files
- temporary databases
- Docker Compose test services
- mock credentials
- test endpoints
- feature flags

Ask:

```text
What configuration does the test use?
What does it replace?
What does it share with production?
```

Configuration differences can explain why a test passes while production behaves differently.

Do not treat a passing test as proof that every production condition has been covered.

---

## 20. Test Database Lifecycle

Database-backed tests need special attention.

Determine how the test database is created and cleaned.

Possible approaches include:

```text
Create database
    ↓
Run migrations
    ↓
Run tests
    ↓
Rollback / cleanup
```

Another approach may reuse a database and reset its state between tests.

Find out which approach the repository actually uses.

Also investigate:

- transaction rollback
- truncation
- fixture cleanup
- temporary schemas
- containers
- test database isolation

Shared database state can make tests misleading.

---

## 21. Find Tests for Migrations

Chapter 12 focused on finding migrations.

Now connect migrations to tests.

Search for tests that verify:

- migrations can be applied
- schema objects exist
- constraints work
- data migrations produce expected results
- migrations can be rolled back where supported
- the application works against the migrated schema

A useful flow is:

```text
Migration
   ↓
Database schema
   ↓
Application
   ↓
Test
```

This gives you stronger evidence than reading the migration alone.

---

## 22. Test Markers and Categories

Some test frameworks allow tests to be grouped or marked.

Generic categories might include:

```text
unit
integration
e2e
slow
database
external
```

These markers can affect which tests run in CI.

For example, a CI job might run only fast tests on every pull request and run integration tests separately.

Therefore, finding the test file is not enough.

Also find out whether the test is actually executed in the normal development and CI workflow.

---

## 23. Read the CI Pipeline

CI configuration is one of the best places to understand test execution.

Look for:

- test commands
- test filters
- environment setup
- service containers
- database setup
- migration commands
- coverage commands
- parallel test execution
- test artifacts

A generic CI flow may look like:

```text
Pull request
    ↓
Install dependencies
    ↓
Start services
    ↓
Prepare database
    ↓
Run tests
    ↓
Collect results
    ↓
Build / deploy
```

Your repository may use a different sequence.

Follow the actual configuration.

---

## 24. Coverage Is Not the Same as Correctness

Test coverage can be useful.

But high coverage does not automatically mean a pipeline is correct.

For example, a test suite may execute a function without checking an important failure case.

Think about coverage in several dimensions:

```text
Code coverage
     +
Behavior coverage
     +
Failure coverage
     +
Data coverage
     +
Integration coverage
```

During repository investigation, ask what behavior is actually protected.

Do not treat a coverage percentage as a complete quality measurement.

---

## 25. Find the Missing Test

Sometimes the most useful discovery is a missing test.

Suppose you find code that:

```text
receives an event
    ↓
inserts into database
```

But you cannot find a test for duplicate events.

That does not prove the system is incorrect.

It means the repository does not provide visible test evidence for that behavior.

This distinction is important.

Use precise language:

```text
No test was found covering duplicate processing.
```

Do not claim:

```text
Duplicate processing is not handled.
```

The second statement requires additional evidence.

---

## 26. Following One Test Completely

A powerful investigation technique is to choose one important test and follow it from start to finish.

Suppose the test is:

```text
test_event_ingestion
```

Trace:

```text
Test
 ↓
Fixture
 ↓
Input payload
 ↓
Entry point
 ↓
Ingestion function
 ↓
Validation
 ↓
Database write
 ↓
Assertion
```

Now identify every important dependency.

Write them down.

After following one test completely, many repository concepts become easier to understand.

---

## 27. Following One Failure Test

Do the same thing with a failure test.

For example:

```text
test_api_timeout_retries
```

Trace:

```text
Test setup
 ↓
Mock timeout
 ↓
API client
 ↓
Retry logic
 ↓
Second request
 ↓
Success
 ↓
Assertion
```

Now you understand the retry behavior much better than a generic description of retries would provide.

This method is especially useful for troubleshooting production failures.

---

## 28. Test Isolation

Ask whether tests depend on each other.

A good test suite should make dependencies explicit.

Potential warning signs include:

- tests passing only in a specific order
- shared mutable state
- persistent database records
- global configuration
- reused external resources
- cleanup that happens only sometimes

If a test fails only when the entire suite runs, investigate shared state.

Do not immediately assume that the failing test itself is the only problem.

---

## 29. Parallel Test Execution

Some repositories run tests in parallel.

This can expose problems that sequential execution hides.

Potential problems include:

- shared database state
- duplicate fixture IDs
- temporary file conflicts
- shared ports
- race conditions
- external resource collisions

When investigating parallel tests, ask:

```text
Can these tests safely run at the same time?
What resources do they share?
How is isolation enforced?
```

This matters for both test reliability and pipeline correctness.

---

## 30. Regression Tests

Regression tests protect behavior that previously broke.

Suppose a production bug caused duplicate records.

A regression test may later be added to prevent the same problem from returning.

That test is valuable historical evidence.

It tells you:

- a particular behavior mattered
- the repository previously needed protection against it
- future changes should preserve the behavior

When investigating a strange-looking test, look at its history if repository history is available.

The test may exist because of a real incident or previous defect.

Do not invent the reason if the history does not show it.

---

## 31. Tests as Executable Documentation

Traditional documentation says:

```text
The system rejects invalid events.
```

A test can demonstrate the behavior:

```text
invalid event
    ↓
validation
    ↓
rejection
    ↓
assert expected result
```

This is why tests are often called executable documentation.

They express expected behavior in a form that can be executed repeatedly.

However, tests can also become outdated.

So use them together with:

- implementation code
- migrations
- configuration
- CI
- documentation
- actual runtime evidence

No single repository artifact should automatically be treated as the complete truth.

---

## 32. Practical Test Investigation Workflow

Use this workflow when entering an unfamiliar repository.

### Step 1 — Identify the test framework

Inspect dependencies and configuration.

### Step 2 — Find the test command

Check the README, scripts, Makefile, and CI.

### Step 3 — Find test locations

Search for test directories and test files.

### Step 4 — Identify test categories

Separate unit, integration, database, and end-to-end tests where possible.

### Step 5 — Find fixtures

Understand how test data and dependencies are created.

### Step 6 — Find important behavior tests

Search for duplicates, retries, failures, validation, and database behavior.

### Step 7 — Follow tests into the code

Trace the test to the function, SQL, API, or pipeline stage it exercises.

### Step 8 — Follow code back to tests

Check which behaviors have tests and which do not.

### Step 9 — Check CI

Determine which tests actually run automatically.

### Step 10 — Check environment setup

Understand databases, services, containers, and configuration.

### Step 11 — Investigate failures

Use failure tests to understand recovery behavior.

### Step 12 — Record gaps

Clearly separate verified behavior from behavior for which no test evidence was found.

---

## 33. Common Mistakes

### Mistake 1 — Looking only for `tests/`

Tests can exist elsewhere.

Search the repository.

### Mistake 2 — Reading only test names

Names are useful clues, but the test body contains the evidence.

### Mistake 3 — Ignoring fixtures

Important setup may be hidden in fixtures.

### Mistake 4 — Ignoring mocks

A mocked API is not the same as a real API integration.

### Mistake 5 — Assuming all tests run in CI

Check the workflow configuration.

### Mistake 6 — Treating coverage as correctness

Coverage does not guarantee meaningful behavior coverage.

### Mistake 7 — Ignoring failure tests

Failure tests often reveal more about production behavior than happy-path tests.

### Mistake 8 — Assuming missing tests mean missing behavior

No test evidence is not proof that the behavior does not exist.

### Mistake 9 — Assuming a passing test proves production safety

Tests cover specific scenarios and environments.

### Mistake 10 — Changing code before understanding the tests

Read the existing behavior first.

---

## 34. Production Investigation

When investigating a production incident, tests can help answer important questions.

Suppose production contains duplicate records.

Search for:

```text
duplicate
unique
idempotent
upsert
conflict
replay
```

Then find the relevant tests.

You may discover:

```text
Test expects duplicate to be ignored
        ↓
Production shows duplicate
        ↓
Investigate difference
        ↓
Code version?
Database constraint?
Migration version?
Configuration?
Different execution path?
```

The test gives you an expected behavior.

The production investigation determines why reality differs.

---

## 35. Build a Test Map

After investigating a repository, create a simple map.

Generic example:

```text
                 Tests
                   |
       +-----------+-----------+
       |           |           |
     Unit      Integration    E2E
       |           |           |
    Functions   Database     Pipeline
                  |             |
               Migrations     External API
```

Then connect important behaviors:

```text
Validation ------> validation tests
Duplicates ------> idempotency tests
Retries ---------> failure tests
Migrations ------> schema tests
Database --------> integration tests
Pipeline --------> end-to-end tests
```

This gives you a quick picture of what the repository protects.

---

## 36. Practical Checklist

Before saying that you understand the repository's tests, check:

- [ ] I know which test framework is used.
- [ ] I know how tests are executed.
- [ ] I found the main test locations.
- [ ] I identified major test categories.
- [ ] I found important fixtures.
- [ ] I understand important mocks.
- [ ] I found database tests.
- [ ] I found migration-related tests.
- [ ] I found failure-handling tests.
- [ ] I found duplicate/idempotency tests where they exist.
- [ ] I found data-quality tests where they exist.
- [ ] I checked CI test execution.
- [ ] I understand test environment setup.
- [ ] I followed at least one test into the implementation.
- [ ] I followed important code back to its tests.
- [ ] I identified meaningful test gaps without assuming behavior is absent.
- [ ] I understand which behaviors have automated evidence.

---

## 37. What You Learned

In this chapter, you learned how to use tests as a repository-investigation tool.

You learned how to:

- identify the test framework
- find test commands
- locate tests
- understand test categories
- follow tests into implementation code
- follow implementation code back to tests
- investigate fixtures and mocks
- inspect database tests
- investigate migration tests
- find failure and retry tests
- investigate idempotency tests
- understand CI test execution
- recognize test gaps
- follow a complete test path
- and build a test map

The main lesson is simple:

> Tests do not only verify code. They also explain how the repository expects the system to behave.

When you are learning an unfamiliar pipeline, read the tests.

They can show you the expected behavior faster than reading the entire repository from top to bottom.

---

## Recipe Preview

Repository investigation is now covering three important areas:

```text
Entry point
     ↓
Database code
     ↓
Migrations
     ↓
Tests
```

These pieces allow you to connect execution, storage, schema history, and expected behavior.

The next chapter focuses on another part of that investigation:

**Chapter 14 — Understanding Configuration**.