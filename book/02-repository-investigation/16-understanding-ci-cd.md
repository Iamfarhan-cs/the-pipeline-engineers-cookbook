# Chapter 16 — Understanding CI/CD

CI/CD is where the repository's code moves from a developer's change toward an automatically tested and deployable system.

For a Data Engineering repository, CI/CD may run:

- unit tests
- integration tests
- database migrations
- linting
- type checks
- Docker builds
- security checks
- data-quality checks
- deployment steps
- post-deployment verification

If you only understand the application code, you may still not understand how changes reach a running environment.

This chapter explains how to investigate CI/CD as part of repository investigation.

---

## 1. What Is CI/CD?

CI usually means **Continuous Integration**.

It is the automated process that checks changes when code is pushed or a pull request is created.

CD can mean **Continuous Delivery** or **Continuous Deployment**.

Continuous Delivery means the system is kept ready for deployment.

Continuous Deployment means approved changes can be deployed automatically.

The exact process depends on the repository.

A simple model is:

```text
Developer change
      ↓
CI
      ↓
Tests and checks
      ↓
Build
      ↓
Artifact
      ↓
CD
      ↓
Deployment
      ↓
Verification
```

---

## 2. Why CI/CD Matters During Repository Investigation

Suppose you find a test directory.

That tells you tests exist.

But it does not tell you whether those tests actually run automatically.

CI configuration can answer that question.

Suppose you find a migration directory.

That tells you migrations exist.

But CI/CD may reveal:

- when migrations run
- which database they run against
- whether migrations run before tests
- whether migrations run during deployment
- whether migration failures stop deployment

CI/CD therefore connects many parts of the repository.

---

## 3. Where CI/CD Configuration Lives

Start by looking for common CI/CD locations.

For GitHub Actions, a repository may contain:

```text
.github/
└── workflows/
    ├── test.yml
    ├── build.yml
    └── deploy.yml
```

Other systems may use files such as:

```text
.gitlab-ci.yml
Jenkinsfile
azure-pipelines.yml
bitbucket-pipelines.yml
```

These are examples only.

Use the repository's actual CI/CD platform as evidence.

Also search for:

- Makefiles
- shell scripts
- deployment scripts
- Docker files
- infrastructure directories
- Helm charts
- deployment manifests

---

## 4. Find the CI Platform

Before reading individual jobs, identify which system executes them.

Possible systems include:

- GitHub Actions
- GitLab CI/CD
- Jenkins
- Azure Pipelines
- CircleCI
- Buildkite

These are generic examples.

Look at the repository files and project settings available to you.

The platform matters because the configuration syntax and execution model are different.

---

## 5. Start With the Workflow File

Once you find a workflow, read it from top to bottom.

Look for:

- trigger
- job
- runner
- steps
- environment variables
- secrets
- services
- dependencies
- artifacts
- conditions
- deployment environments

A generic workflow may look like:

```text
Trigger
  ↓
Job
  ↓
Checkout
  ↓
Install dependencies
  ↓
Run tests
  ↓
Build
  ↓
Deploy
```

The actual repository may use a different sequence.

Follow the configuration.

---

## 6. Understand Triggers

A CI workflow needs a reason to start.

Common triggers include:

- push
- pull request
- tag
- manual execution
- scheduled execution
- another workflow
- release event

Generic example:

```text
Pull request created
       ↓
CI workflow starts
```

Another workflow may run only after a release tag is created.

This distinction matters.

A test workflow that runs on pull requests is different from a deployment workflow that runs only after a release event.

---

## 7. Find the Jobs

A workflow can contain multiple jobs.

Generic example:

```text
Workflow
   |
   +---- lint
   |
   +---- unit tests
   |
   +---- integration tests
   |
   +---- build
   |
   +---- deploy
```

Jobs may run in parallel or depend on one another.

When investigating, identify:

- what each job does
- what it requires
- what it produces
- whether another job depends on it

---

## 8. Job Dependencies

Some jobs must finish before others can start.

Generic flow:

```text
Tests
  ↓
Build
  ↓
Deploy
```

Another workflow might run independent checks in parallel:

```text
        +--> Unit tests --+
Trigger +--> Lint --------+--> Build
        +--> Security ----+
```

Find the actual dependency configuration.

This tells you which failures block later stages.

---

## 9. Runners

A CI job needs an execution environment.

This may be a hosted runner, self-hosted machine, or another execution environment.

The runner provides things such as:

- operating system
- shell
- runtime tools
- Docker
- language runtimes
- system packages

When investigating a CI failure, check the runner environment.

A command that works on a developer's machine may fail because the CI runner does not have the same tools installed.

---

## 10. Checkout

Most CI workflows first retrieve the repository source.

Generic flow:

```text
Workflow starts
    ↓
Repository checkout
    ↓
Source code available
```

Do not assume the workflow always checks out exactly the same revision you expect.

Tags, branches, pull requests, and reusable workflows can affect what source is tested.

---

## 11. Dependency Installation

CI usually needs to install project dependencies.

Examples include:

```text
Python dependencies
Node dependencies
Java dependencies
system packages
```

The repository may use commands such as:

```text
pip install ...
npm ci
mvn ...
```

These are generic examples.

Find the actual dependency installation command.

Also investigate whether dependency versions are locked.

---

## 12. Reproducibility

CI is useful partly because it creates a repeatable environment.

Suppose a developer says:

> It works on my machine.

CI provides another environment in which the repository can be tested.

A reproducible pipeline should make important dependencies explicit.

Look for:

- pinned versions
- lock files
- container images
- runtime versions
- package managers
- infrastructure versions

Reproducibility reduces environment-specific surprises.

---

## 13. Environment Variables in CI

CI workflows commonly provide environment variables.

Generic example:

```text
DATABASE_URL
API_URL
ENVIRONMENT
LOG_LEVEL
```

Trace them just as you did in Chapter 14.

```text
CI variable
    ↓
Workflow
    ↓
Process environment
    ↓
Application configuration
```

This can explain why code behaves differently in CI and locally.

---

## 14. Secrets in CI

CI systems often provide secrets to jobs.

Examples include:

- API tokens
- database passwords
- cloud credentials
- signing keys
- deployment credentials

Never print secret values during investigation.

When documenting a workflow, record:

```text
DEPLOYMENT_TOKEN is required
```

not the actual value.

Also check whether secrets are exposed to jobs that do not need them.

---

## 15. Find the Test Stage

One of the first CI investigation tasks should be finding where tests run.

Look for commands such as:

```text
pytest
npm test
mvn test
```

Again, these are generic examples.

Determine:

- which tests run
- which tests are excluded
- whether integration services are started
- whether migrations run first
- whether test results are saved

This connects directly to Chapter 13.

---

## 16. Unit Tests in CI

Unit tests are often part of an early CI stage.

Generic flow:

```text
Code change
   ↓
Dependencies
   ↓
Unit tests
   ↓
Pass / fail
```

Unit tests are usually fast enough to run frequently.

If they fail, later stages may not run.

Check the workflow rather than assuming this behavior.

---

## 17. Integration Tests in CI

Integration tests may require services such as:

- PostgreSQL
- Redis
- Kafka
- object storage
- external test services

A generic CI setup may be:

```text
CI runner
   ↓
Start service containers
   ↓
Prepare database
   ↓
Run integration tests
```

Find how the repository creates these dependencies.

This may use Docker services, Compose, or CI-native service containers.

---

## 18. Database Migrations in CI

Database migrations are especially important for Data Engineering repositories.

A workflow may do:

```text
Start PostgreSQL
      ↓
Run migrations
      ↓
Run integration tests
```

Another workflow may do:

```text
Build application
      ↓
Deploy
      ↓
Run migrations
```

Do not assume which sequence is used.

Find the actual command and its position in the workflow.

---

## 19. CI as a Schema Verification Tool

Running migrations in CI can reveal problems before deployment.

For example:

```text
Migration
   ↓
Fresh database
   ↓
Application starts
   ↓
Tests run
```

This can catch:

- invalid SQL
- missing dependencies
- incompatible schema changes
- incorrect migration ordering
- application/schema mismatches

CI therefore provides more than code testing.

It can test the relationship between application code and database schema.

---

## 20. Build Stage

After validation, CI may build an artifact.

Examples include:

- Docker image
- Python package
- Java artifact
- JavaScript bundle
- deployment package

Generic flow:

```text
Source code
   ↓
Build
   ↓
Artifact
```

Find what artifact the repository produces.

---

## 21. Docker Builds in CI

A repository may build a Docker image during CI.

Generic flow:

```text
Source
  ↓
Docker build
  ↓
Image
  ↓
Registry
```

Investigate:

- which Dockerfile is used
- build context
- build arguments
- image tag
- registry
- caching
- build failures

This connects Chapter 15 with CI/CD.

---

## 22. Image Tags

Container images need identifiable versions.

Possible tag strategies include:

- commit identifier
- release version
- branch name
- date
- environment label

These are generic examples.

Find the actual tagging strategy.

A good investigation question is:

> Which exact image does production run?

The answer should be traceable from the deployment configuration.

---

## 23. Artifact Flow

Follow the artifact through the pipeline.

Generic example:

```text
Source code
   ↓
CI build
   ↓
Docker image
   ↓
Container registry
   ↓
Deployment
   ↓
Running container
```

Another system may use a package instead of a Docker image.

The important concept is the same:

the artifact built by CI should be traceable to the code that produced it.

---

## 24. Deployment Stage

CD begins when the system moves a validated artifact toward an environment.

Generic flow:

```text
Validated artifact
      ↓
Deployment
      ↓
Environment
      ↓
Running application
```

Investigate:

- deployment trigger
- target environment
- credentials
- configuration
- artifact version
- migration behavior
- health checks
- rollback behavior

---

## 25. Deployment Environments

Repositories may have multiple deployment environments.

Generic example:

```text
development
     ↓
staging
     ↓
production
```

The actual repository may use different names or a different flow.

Find:

- how environments are selected
- which workflow targets each environment
- which configuration is used
- who or what can trigger deployment

---

## 26. Deployment Approvals

Some deployment workflows require approval before a production change.

Generic flow:

```text
Build
  ↓
Tests
  ↓
Approval
  ↓
Production deployment
```

If approvals exist, understand where they are configured.

Do not assume every successful build automatically becomes a production deployment.

---

## 27. Post-Deployment Verification

Deployment is not necessarily complete when a command succeeds.

A stronger flow is:

```text
Deploy
  ↓
Application starts
  ↓
Health check
  ↓
Smoke test
  ↓
Deployment verified
```

Search for:

- health checks
- smoke tests
- readiness checks
- endpoint checks
- database checks
- deployment verification scripts

These steps tell you how the repository determines whether deployment actually worked.

---

## 28. CI/CD and Observability

CI/CD can also produce operational evidence.

Look for:

- build logs
- test reports
- deployment logs
- health-check results
- metrics
- release annotations
- artifacts

A failed deployment should leave enough evidence to investigate what happened.

This connects CI/CD with Chapter 8.

---

## 29. Failed CI Jobs

When CI fails, investigate the failed stage first.

Generic structure:

```text
Workflow
   ↓
Job
   ↓
Step
   ↓
Command
   ↓
Error
```

Do not treat the entire workflow as failed without locating the actual failing step.

Examples:

```text
dependency installation failed
test failed
migration failed
Docker build failed
authentication failed
deployment failed
```

Each requires a different investigation path.

---

## 30. Failed Tests vs Failed Infrastructure

A CI job may report a test failure.

But the test may have failed because its dependency was unavailable.

For example:

```text
Integration test
      ↓
PostgreSQL unavailable
      ↓
test fails
```

The application behavior may not be the actual problem.

Check:

- service startup logs
- health checks
- database readiness
- network configuration
- environment variables
- credentials

This is why understanding CI infrastructure matters.

---

## 31. CI Caching

CI systems may cache dependencies or build layers.

Caching can make workflows faster.

But stale or incorrect caches can sometimes make investigation confusing.

Look for:

- dependency caches
- Docker layer caches
- build caches
- cache keys

When behavior differs unexpectedly, determine whether cached artifacts are involved.

Do not disable all caching immediately.

First understand what is being cached and why.

---

## 32. Parallel Jobs

CI jobs may run in parallel.

For example:

```text
              +--> unit tests
Trigger ----- +--> lint
              +--> security checks
```

This reduces total execution time.

But it also means jobs may have different environments and dependencies.

When investigating, identify whether a failing job depends on another job or service.

---

## 33. Artifacts and Test Reports

CI systems often store artifacts.

Examples include:

- test reports
- coverage reports
- logs
- Docker metadata
- generated files
- build packages

Artifacts can help investigate failures after the job has finished.

When reading a workflow, find which artifacts are uploaded.

---

## 34. CI/CD and Branches

Deployment behavior may depend on branches.

Generic example:

```text
feature branch
     ↓
tests

main branch
     ↓
tests
     ↓
build
     ↓
deploy
```

This is only a generic example.

Read the actual branch conditions.

A workflow may also depend on tags, pull requests, or manual approval.

---

## 35. CI/CD and Pull Requests

Pull-request workflows commonly protect the main branch.

Possible checks include:

- tests
- linting
- type checking
- security scanning
- build verification

Generic flow:

```text
Pull request
     ↓
CI checks
     ↓
All required?
  /        \
yes        no
 ↓          ↓
merge      fix
```

Find which checks are actually required.

Do not assume every workflow is a required merge check.

---

## 36. CI/CD and Secrets Safety

CI/CD configuration can expose secrets if handled incorrectly.

Investigate whether:

- secrets are stored in the platform's secret mechanism
- logs mask secret values
- secrets are passed only to required jobs
- pull-request workflows can access protected credentials
- generated artifacts contain sensitive values

Never add real secrets to workflow files.

Never print secret values for debugging.

---

## 37. CI/CD and Data Engineering Pipelines

Data Engineering repositories often have additional CI concerns.

Look for checks around:

- schema changes
- migrations
- data validation
- pipeline tests
- replay logic
- idempotency
- Docker images
- infrastructure
- configuration

Generic flow:

```text
Code change
    ↓
Tests
    ↓
Schema validation
    ↓
Build
    ↓
Deploy
    ↓
Health check
    ↓
Pipeline ready
```

The actual repository may use a smaller or larger flow.

---

## 38. Deployment Rollback

Production deployment investigation should include rollback behavior.

Ask:

- Can the previous artifact be restored?
- How is the previous version identified?
- What happens if a migration has already run?
- Can application and database versions remain compatible?
- Is rollback automatic or manual?

A simple application rollback may look like:

```text
Version B
   ↓
failure
   ↓
Version A
```

Database schema changes can make rollback more complicated.

This is why migration strategy and deployment strategy must be understood together.

---

## 39. CI/CD and Database Compatibility

Suppose a new application version expects a new database column.

A dangerous deployment sequence could be:

```text
Deploy new application
       ↓
Old database schema
       ↓
application failure
```

A safer migration strategy depends on the system, but often requires thinking about compatibility between application versions and schema versions.

During repository investigation, find the actual migration and deployment sequence.

Do not assume that migrations and deployments are automatically safe.

---

## 40. CI/CD Failure Investigation Workflow

Use this workflow when investigating a failed CI/CD pipeline.

### Step 1 — Identify the workflow

Determine which workflow was triggered.

### Step 2 — Identify the trigger

Was it a push, pull request, tag, schedule, or manual run?

### Step 3 — Find the failed job

Do not debug unrelated jobs first.

### Step 4 — Find the failed step

Locate the exact command that failed.

### Step 5 — Read the error

Separate the first useful error from later cascading errors.

### Step 6 — Identify the dependency

Determine whether the failure is application, database, Docker, network, configuration, or CI infrastructure related.

### Step 7 — Compare local and CI environments

Look for configuration and version differences.

### Step 8 — Check artifacts and logs

Use test reports, logs, and generated artifacts.

### Step 9 — Check recent changes

Identify what changed between the last successful run and the failing run.

### Step 10 — Reproduce the smallest failing step

A smaller reproduction is usually easier to understand.

---

## 41. Build a CI/CD Map

After investigating the repository, create a simple map.

Generic example:

```text
Developer change
       ↓
   CI trigger
       ↓
   Test jobs
       ↓
     Build
       ↓
    Artifact
       ↓
   Deployment
       ↓
 Environment
       ↓
Health checks
       ↓
  Verified
```

Add important dependencies:

```text
Tests ------> database service
Build ------> Dockerfile
Deploy -----> image registry
Deploy -----> environment secrets
Migrations --> database
Verification -> health endpoint
```

This gives you a complete picture of how source code becomes a running system.

---

## 42. Common Mistakes

### Mistake 1 — Looking only at application code

CI/CD may change how the application is built and run.

### Mistake 2 — Assuming every workflow deploys

Some workflows only test or build.

### Mistake 3 — Ignoring triggers

A workflow may not run for the event you expect.

### Mistake 4 — Ignoring job dependencies

A later job may depend on an earlier job.

### Mistake 5 — Treating every CI failure as an application failure

Infrastructure and configuration can fail first.

### Mistake 6 — Ignoring secrets and environment variables

CI environments can differ significantly from local environments.

### Mistake 7 — Assuming migrations run automatically

Find the actual migration command.

### Mistake 8 — Ignoring artifact versions

The deployed image may not be the image you expected.

### Mistake 9 — Ignoring post-deployment verification

A successful deployment command does not prove the application is healthy.

### Mistake 10 — Ignoring rollback behavior

Production recovery requires knowing how to return to a known-good version.

---

## 43. Production Considerations

Production CI/CD should make changes traceable and recoverable.

Important areas include:

- reproducible builds
- versioned artifacts
- protected environments
- secret management
- migration safety
- deployment verification
- health checks
- rollback procedures
- audit logs
- least privilege
- monitoring
- failure notifications

A production deployment should not be treated as one command.

It is a controlled sequence of build, release, deployment, and verification steps.

---

## 44. Practical Checklist

Before saying that you understand the repository's CI/CD setup, check:

- [ ] I know which CI/CD platform is used.
- [ ] I found the workflow or pipeline configuration.
- [ ] I understand the workflow triggers.
- [ ] I know the major jobs.
- [ ] I understand job dependencies.
- [ ] I know which runner or execution environment is used.
- [ ] I know how dependencies are installed.
- [ ] I know which tests run.
- [ ] I know how integration services are provided.
- [ ] I know how migrations are executed.
- [ ] I know what artifact is built.
- [ ] I know how artifacts are versioned.
- [ ] I know where artifacts are stored.
- [ ] I understand deployment triggers.
- [ ] I know which environments can be deployed to.
- [ ] I understand environment variables and secrets.
- [ ] I know how deployment is verified.
- [ ] I know how failures are investigated.
- [ ] I know whether rollback exists and how it works.
- [ ] I can trace a code change from commit to running application.

---

## 45. What You Learned

In this chapter, you learned how to investigate CI/CD as part of understanding a Data Engineering repository.

You learned how to:

- find CI/CD configuration
- identify the CI platform
- understand triggers and jobs
- follow job dependencies
- understand runners
- inspect dependency installation
- investigate environment variables and secrets
- find unit and integration test stages
- understand migrations in CI/CD
- investigate build and Docker stages
- follow artifacts
- understand deployment environments
- investigate post-deployment verification
- troubleshoot failed jobs
- distinguish application failures from infrastructure failures
- understand rollback considerations
- and build a CI/CD map

The main lesson is simple:

> CI/CD is the path that turns a code change into a tested, built, deployed, and verified system.

When investigating a repository, do not stop at the source code. Follow the change through the pipeline until you understand what happens in the target environment.

---

## Repository Investigation Complete

You have now covered the main repository investigation areas in Part II:

```text
Chapter 9  — How to Read a Data Engineering Repository
Chapter 10 — Finding the Entry Point
Chapter 11 — Finding Database Code
Chapter 12 — Finding Migrations
Chapter 13 — Finding Tests
Chapter 14 — Understanding Configuration
Chapter 15 — Understanding Docker
Chapter 16 — Understanding CI/CD
```

Together, these chapters give you a practical investigation path:

```text
Repository
    ↓
Entry Point
    ↓
Application Code
    ↓
Database Code
    ↓
Migrations
    ↓
Tests
    ↓
Configuration
    ↓
Docker
    ↓
CI/CD
    ↓
Running System
```

You should now be able to approach an unfamiliar Data Engineering repository systematically instead of opening files randomly.

Part III moves from investigation into implementation.

Next: **Chapter 17 — Create an Ingestion Pipeline**.