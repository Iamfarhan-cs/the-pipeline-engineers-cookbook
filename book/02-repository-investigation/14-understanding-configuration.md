# Recipe 14 — Understanding Configuration

Configuration is one of the easiest parts of a Data Engineering repository to misunderstand.

A pipeline can have perfectly correct code and still fail because it is running with the wrong configuration.

A database host can be wrong.

An API URL can point to the wrong environment.

A required environment variable can be missing.

A retry limit can be different from what you expected.

A feature flag can change the execution path.

When investigating a repository, you therefore need to understand not only **what the code does**, but also **which configuration tells the code how to run**.

This chapter explains how to find configuration, understand where values come from, determine which value wins when multiple sources exist, and investigate configuration-related failures safely.

---

## 1. What Is Configuration?

Configuration is information that controls how software runs.

Examples include:

- database host
- database port
- database name
- API URL
- timeout
- retry count
- log level
- feature flag
- queue name
- topic name
- storage bucket
- environment name
- batch size
- polling interval

Configuration is different from application logic.

Application logic answers:

> What should the program do?

Configuration answers:

> How, where, or under which conditions should the program do it?

A simple model is:

```text
Code
  +
Configuration
  ↓
Running application
```

---

## 2. Why Configuration Investigation Matters

Consider a pipeline that connects to PostgreSQL.

The code may contain:

```text
connect_to_database()
```

But the function still needs values such as:

```text
host
port
database
user
password
```

Those values may come from environment variables or another configuration source.

If the application connects to the wrong database, the SQL code may still be completely correct.

The problem is configuration.

This is why configuration should be part of every repository investigation.

---

## 3. Where Configuration Can Live

Configuration can come from many places.

Common sources include:

- environment variables
- `.env` files
- YAML files
- JSON files
- TOML files
- Python or application configuration modules
- command-line arguments
- Docker Compose
- Dockerfiles
- CI/CD variables
- deployment manifests
- secret managers
- cloud configuration
- database configuration

These are generic examples.

Do not assume a repository uses all of them.

The first task is to identify which sources the repository actually uses.

---

## 4. Start at the Repository Root

When investigating configuration, start with the repository root.

Look for files such as:

```text
.env
.env.example
config/
settings/
pyproject.toml
package.json
docker-compose.yml
Dockerfile
Makefile
.github/
```

Also inspect directories related to deployment and infrastructure.

For example:

```text
deploy/
infra/
scripts/
k8s/
helm/
.github/workflows/
```

The exact structure depends on the repository.

The goal is to build a map of where configuration may enter the system.

---

## 5. Search for Environment Variables

Environment variables are common in pipeline systems.

Generic examples include:

```text
DATABASE_URL
DB_HOST
DB_PORT
API_URL
LOG_LEVEL
RETRY_COUNT
ENVIRONMENT
```

Search the repository for the variable names.

You may find them in application code, Docker Compose, CI workflows, deployment scripts, documentation, or `.env.example`.

This lets you trace a configuration value through the system.

---

## 6. Find Where a Configuration Value Is Read

Finding a variable name is only the beginning.

Suppose you find:

```text
DATABASE_URL
```

Search for where it is read.

Generic examples include:

```python
os.getenv("DATABASE_URL")
```

or:

```python
os.environ["DATABASE_URL"]
```

The actual repository may use a configuration library instead.

Once you find the read operation, follow the value.

For example:

```text
Environment variable
       ↓
Configuration loader
       ↓
Database settings
       ↓
Connection factory
       ↓
Database connection
```

This is the configuration equivalent of following a data flow.

---

## 7. Find the Configuration Object

Some applications load configuration into an object.

Generic example:

```text
Settings
├── database_url
├── api_url
├── retry_count
├── timeout
└── environment
```

Then application code uses the object instead of reading environment variables directly.

The investigation path becomes:

```text
Environment
   ↓
Configuration loader
   ↓
Settings object
   ↓
Application component
```

Find the configuration object and identify all important fields.

Do not stop after finding the first environment variable.

---

## 8. Required vs Optional Configuration

Configuration values can be required or optional.

A required value might cause startup to fail when missing.

An optional value might have a default.

Generic example:

```text
DATABASE_URL       required
LOG_LEVEL          optional
RETRY_COUNT        optional
API_URL            required
```

This distinction matters during troubleshooting.

If the application fails because a required value is missing, the correct solution may be to provide the configuration rather than change the application code.

Look for validation logic or default values.

---

## 9. Find Default Values

Configuration often has defaults.

Generic example:

```text
RETRY_COUNT = 3
LOG_LEVEL = INFO
TIMEOUT = 30
```

A default may exist in code, configuration files, or a configuration library.

When investigating a value, ask:

1. Is there a default?
2. Where is it defined?
3. Can an environment variable override it?
4. Can a command-line argument override it?
5. Is the default different between environments?

Do not assume the value you see in one file is the final value used at runtime.

---

## 10. Configuration Precedence

Multiple configuration sources can exist at the same time.

For example:

```text
default value
     ↓
config file
     ↓
.env
     ↓
environment variable
     ↓
command-line argument
```

The actual precedence depends on the application and tools involved.

Your job is to find that precedence rather than assume it.

Suppose a timeout appears as:

```text
Code default:       30
.env:               60
CI variable:        90
```

The final value might be 90.

But that is only true if the repository's configuration rules give the CI variable highest precedence.

Verify it.

---

## 11. `.env` Files

`.env` files are commonly used for local development.

A generic file might contain:

```text
DATABASE_URL=postgresql://localhost/example
LOG_LEVEL=DEBUG
API_URL=http://localhost:8000
```

Treat this as a generic example.

`.env` files can contain secrets.

Never assume that a repository should commit real credentials.

Check whether `.env` is ignored by Git.

Also look for `.env.example` or similar files.

An example configuration file can document the required variables without containing real secrets.

---

## 12. Secrets Are Configuration Too

Passwords, API keys, tokens, and private credentials are configuration values, but they require additional care.

Look for references to:

```text
secret
token
password
api_key
credential
private_key
```

Do not print secret values while investigating.

Do not copy secrets into tickets, logs, documentation, or chat.

When documenting configuration, record the variable name rather than the secret value.

Good:

```text
DATABASE_PASSWORD must be configured.
```

Bad:

```text
DATABASE_PASSWORD=actual-secret-value
```

---

## 13. Configuration and Docker

Docker often becomes part of the configuration path.

A Docker Compose file may define environment variables.

Generic example:

```yaml
services:
  pipeline:
    environment:
      DATABASE_URL: ${DATABASE_URL}
```

The exact syntax depends on the repository.

When investigating Docker configuration, trace:

```text
Host environment
      ↓
Docker Compose
      ↓
Container environment
      ↓
Application configuration
      ↓
Application
```

A container may therefore receive configuration from outside the container.

Do not look only inside the application source code.

---

## 14. Configuration and Dockerfiles

Dockerfiles can also define configuration-related behavior.

Look for instructions such as:

```text
ENV
ARG
ENTRYPOINT
CMD
```

These can affect how the application starts.

For example:

```text
Docker image
    ↓
ENTRYPOINT
    ↓
application startup
    ↓
configuration loading
```

Remember that build-time configuration and runtime configuration are not necessarily the same thing.

Investigate both.

---

## 15. Configuration and CI/CD

CI/CD systems frequently provide environment variables.

Look at workflow files and deployment configuration.

Search for:

- environment variables
- secrets
- deployment environments
- test configuration
- database URLs
- service URLs
- feature flags
- migration commands

A generic CI flow may look like:

```text
GitHub Actions
      ↓
Environment / secrets
      ↓
Test process
      ↓
Application configuration
```

The exact CI platform may be different.

The important question is where the runtime values come from.

---

## 16. Configuration by Environment

Most production systems have more than one environment.

Common examples include:

```text
development
test
staging
production
```

The same application code may run in each environment with different configuration.

Example:

```text
                 Development   Production
Database          local          managed DB
API URL            test API       production API
Log level          DEBUG          INFO
```

These are generic examples.

The repository may use different names.

When investigating an issue, always identify the environment.

A configuration value that is correct for development may be completely wrong for production.

---

## 17. Find Environment Selection

Some repositories select an environment using a variable.

Generic example:

```text
ENVIRONMENT=staging
```

That value may influence database selection, API endpoints, logging, feature flags, storage, message brokers, and external services.

Search for where the environment value is read.

Then follow the branches in the configuration code.

Your goal is to answer:

> What configuration does the application actually load for this environment?

---

## 18. Configuration and Database Connections

Database configuration deserves special attention.

Find all values related to:

```text
host
port
database
user
password
SSL
connection pool
timeout
```

Then follow them into the connection code.

A useful map is:

```text
Configuration
    ↓
Connection settings
    ↓
Connection pool
    ↓
Database
```

This helps when investigating connection failures, wrong database selection, authentication errors, SSL problems, connection pool exhaustion, and unexpected timeouts.

---

## 19. Configuration and External APIs

API configuration often includes:

- base URL
- timeout
- authentication
- retry settings
- page size
- rate limits

Suppose a pipeline works locally but fails in production.

Check whether it is calling the same endpoint.

Generic investigation:

```text
Application
    ↓
API configuration
    ↓
Base URL
    ↓
External service
```

A wrong URL can send valid requests to the wrong system.

---

## 20. Configuration and Message Systems

Streaming and event-driven pipelines have additional configuration.

Examples include:

- broker address
- topic
- consumer group
- partition settings
- offset behavior
- acknowledgement settings
- polling interval
- batch size

Generic example:

```text
Consumer configuration
        ↓
Broker
        ↓
Topic
        ↓
Consumer group
        ↓
Application
```

If a consumer receives no events, configuration should be one of the first investigation areas.

Do not immediately assume the producer is broken.

---

## 21. Configuration and Storage

Object storage and file-based pipelines can have configuration such as:

- bucket name
- endpoint
- region
- credentials
- prefix
- path
- retention settings

Generic flow:

```text
Configuration
    ↓
Storage client
    ↓
Bucket / filesystem
    ↓
Object or file
```

If the pipeline cannot find a file or object, verify the configured location before changing parsing logic.

---

## 22. Feature Flags

Feature flags allow behavior to change without changing the main code path.

Generic example:

```text
ENABLE_NEW_PROCESSOR=true
```

The application may then choose:

```text
                 feature flag
                     ↓
             +-------+-------+
             |               |
           old path        new path
```

When investigating unexpected behavior, search for feature flags.

A feature flag can explain why two environments running the same code behave differently.

---

## 23. Timeouts and Retries

Configuration can control failure behavior.

Examples:

```text
TIMEOUT=30
RETRY_COUNT=3
BACKOFF=2
```

If a pipeline is retrying too many times, do not inspect only the retry function.

Find the configuration that controls it.

Then determine the default value, configured value, environment-specific value, and maximum allowed value.

This connects configuration investigation to the retry concepts from Chapter 6.

---

## 24. Batch Size and Processing Limits

Pipeline throughput is often affected by configuration.

Examples include:

- batch size
- page size
- maximum records
- worker count
- concurrency
- polling interval

A generic flow is:

```text
Configuration
    ↓
Processing limit
    ↓
Records per operation
    ↓
Database / API / queue
```

If a pipeline suddenly processes fewer records, configuration may be involved.

Do not assume the processing code changed.

---

## 25. Configuration and Logging

Logging configuration determines how much information the application produces.

Common settings include:

```text
DEBUG
INFO
WARNING
ERROR
```

During troubleshooting, a different log level can make a major difference.

But debug logging can also expose sensitive information if the application logs unsafe values.

Always investigate what the configured log level does and what data the application logs.

---

## 26. Configuration Validation

Good configuration handling validates important values early.

Generic example:

```text
Application starts
       ↓
Load configuration
       ↓
Validate required values
       ↓
Valid?
   /       \
 yes       no
  ↓         ↓
start      fail early
```

Failing early is often easier to troubleshoot than allowing an invalid value to cause a failure much later.

When investigating configuration code, check whether validation happens at startup or only when a feature is used.

---

## 27. Configuration Errors

Common configuration failures include:

- missing variable
- wrong variable name
- wrong environment
- wrong URL
- invalid port
- invalid boolean value
- incorrect timeout
- missing secret
- wrong database
- wrong queue or topic
- configuration file not loaded
- unexpected precedence

A useful investigation sequence is:

```text
Failure
  ↓
Which component failed?
  ↓
Which configuration does it use?
  ↓
Where did the value come from?
  ↓
Was the value overridden?
  ↓
Is the value valid?
```

---

## 28. Never Debug Configuration by Guessing

Configuration problems often tempt engineers to change values until the system works.

That is risky.

Instead, build evidence.

For example:

```text
Expected database: staging_db
Actual configured database: staging_db
Expected API: staging API
Actual configured API: production API
```

Now you have a concrete problem.

Compare expected and actual configuration systematically.

Do not make random changes to production configuration.

---

## 29. Safe Configuration Investigation

When configuration contains secrets, separate the value from the configuration structure.

Instead of recording the actual secret value, record only that the required secret is configured.

Good documentation:

```text
DATABASE_PASSWORD is configured.
```

Also acceptable:

```text
DATABASE_URL points to the expected environment.
```

without exposing credentials.

The goal is to document configuration behavior without creating a new security problem.

---

## 30. Configuration and Tests

Tests can reveal expected configuration.

Search for:

- test environment variables
- test configuration files
- fixtures that set configuration
- mocked settings
- test database URLs
- feature flags

Suppose a test explicitly sets:

```text
RETRY_COUNT=1
```

That may explain why the test behaves differently from production.

Follow configuration from the test into the application.

---

## 31. Configuration and Migrations

Migrations can also depend on configuration.

For example, the migration command may need a database URL, credentials, environment, or migration directory.

A generic deployment flow is:

```text
Deployment configuration
        ↓
Database connection
        ↓
Migration tool
        ↓
Database schema
```

If migrations work locally but fail in CI, compare the configuration sources.

This connects Chapter 12 and Chapter 14.

---

## 32. Configuration and CI Failures

Suppose tests pass locally but fail in CI.

One possible investigation path is:

```text
Local
  ↓
configuration
  ↓
tests pass

CI
  ↓
different configuration
  ↓
tests fail
```

Compare environment variables, database services, URLs, credentials, feature flags, dependency versions, and configuration files.

Do not assume that the code is different.

The runtime environment may be different.

---

## 33. Build a Configuration Map

After investigating the repository, create a simple map.

Generic example:

```text
                    Configuration
                         |
       +-----------------+-----------------+
       |                 |                 |
   Environment        Database          External API
       |                 |                 |
   feature flags     pool/timeout       URL/auth
       |                 |                 |
       +-----------------+-----------------+
                         |
                    Application
```

A more detailed map can show where values originate:

```text
`.env` -----------+
                  |
CI variables -----+----> Config loader ---> Application
                  |
Docker Compose ---+
```

This makes configuration flow easier to reason about.

---

## 34. Practical Configuration Investigation Workflow

Use this workflow when entering an unfamiliar repository.

### Step 1 — List configuration sources

Find environment files, config directories, deployment files, Docker files, and CI workflows.

### Step 2 — Identify important variables

Start with database, API, storage, queue, logging, retry, and environment settings.

### Step 3 — Find where values are read

Trace each important variable into the configuration loader.

### Step 4 — Find defaults

Record which values have defaults.

### Step 5 — Determine precedence

Find out which source wins when multiple values exist.

### Step 6 — Identify environment differences

Compare development, test, staging, and production configuration where available.

### Step 7 — Trace configuration into components

Follow values into database clients, API clients, consumers, storage clients, and workers.

### Step 8 — Inspect Docker

Find how configuration enters containers.

### Step 9 — Inspect CI/CD

Find how automated environments provide configuration.

### Step 10 — Inspect tests

Find test-specific configuration and overrides.

### Step 11 — Check secrets safely

Verify that required secrets exist without exposing their values.

### Step 12 — Build the configuration map

Document the source, path, purpose, and environment of important settings.

---

## 35. Example: Wrong Database Investigation

Suppose a developer says:

> The pipeline inserted records, but I cannot find them in the database.

Do not immediately inspect the INSERT query.

First verify configuration.

Investigation:

```text
Pipeline
   ↓
Database configuration
   ↓
Database host
   ↓
Database name
   ↓
Connected database
```

Suppose the application is configured for:

```text
staging_database
```

but the developer is checking:

```text
development_database
```

The SQL can be correct.

The observed result is still explained by configuration.

This is why configuration investigation should happen early.

---

## 36. Example: API Failure Investigation

Suppose a pipeline reports that an API is unavailable.

Check:

```text
API base URL
API environment
timeout
authentication configuration
proxy configuration
retry configuration
```

Then trace:

```text
Configuration
    ↓
API client
    ↓
Request
    ↓
External service
```

This helps separate an application bug from a configuration problem.

---

## 37. Common Mistakes

### Mistake 1 — Assuming the value in `.env` is the final value

Another source may override it.

### Mistake 2 — Searching only application code

Docker, CI, deployment files, and scripts may provide configuration.

### Mistake 3 — Exposing secrets during investigation

Never include secret values in logs, tickets, or documentation.

### Mistake 4 — Assuming all environments use the same configuration

Environment differences are often intentional.

### Mistake 5 — Ignoring defaults

A missing environment variable may silently activate a default.

### Mistake 6 — Ignoring feature flags

Feature flags can change execution paths.

### Mistake 7 — Changing production configuration without evidence

First determine the expected and actual values.

### Mistake 8 — Assuming local success proves production configuration is correct

Local and production environments can be very different.

### Mistake 9 — Ignoring configuration used by tests

Test behavior can depend heavily on configuration.

### Mistake 10 — Treating configuration as separate from architecture

Configuration controls important parts of the running system.

---

## 38. Production Considerations

Configuration becomes more important as systems become more complex.

In production, consider:

- secret management
- configuration versioning
- environment separation
- least privilege
- configuration validation
- safe defaults
- auditability
- deployment consistency
- configuration drift
- rollback procedures

A configuration change can alter system behavior without changing application code.

That means configuration changes should be treated as engineering changes, not as invisible operational details.

---

## 39. Practical Checklist

Before saying that you understand a repository's configuration, check:

- [ ] I know where configuration is defined.
- [ ] I know where configuration is loaded.
- [ ] I know the important environment variables.
- [ ] I know which values are required.
- [ ] I know which values have defaults.
- [ ] I understand configuration precedence.
- [ ] I understand environment-specific configuration.
- [ ] I know how Docker receives configuration.
- [ ] I know how CI/CD receives configuration.
- [ ] I understand database connection configuration.
- [ ] I understand external API configuration.
- [ ] I understand queue or streaming configuration where applicable.
- [ ] I know how feature flags affect execution where applicable.
- [ ] I know how retry and timeout settings are configured.
- [ ] I know how tests override configuration.
- [ ] I can investigate configuration without exposing secrets.
- [ ] I can trace an important configuration value from source to runtime.

---

## 40. What You Learned

In this recipe, you learned how to investigate configuration as part of understanding a Data Engineering repository.

You learned how to:

- find configuration sources
- identify environment variables
- trace configuration values into application code
- find defaults
- understand configuration precedence
- investigate `.env` files
- handle secrets safely
- follow configuration through Docker
- investigate CI/CD configuration
- compare environments
- understand database and API configuration
- investigate message and storage configuration
- find feature flags
- understand timeout and retry settings
- investigate configuration failures
- connect configuration to tests and migrations
- and build a configuration map

The main lesson is simple:

> The code tells you what the application can do. Configuration helps determine what it actually does when it runs.

When investigating a pipeline, always ask where its important runtime values come from.

---

## Recipe Preview

The repository investigation path is now becoming a complete system map:

```text
Entry Point
    ↓
Database Code
    ↓
Migrations
    ↓
Tests
    ↓
Configuration
```

These pieces help you understand not only the code, but also how the system is started, stored, verified, and configured.

Next: **Chapter 15 — Understanding Docker**.