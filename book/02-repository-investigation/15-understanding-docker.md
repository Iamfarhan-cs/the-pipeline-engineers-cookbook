# Chapter 15 — Understanding Docker

Docker is often part of the development and execution environment of a Data Engineering repository.

A repository may use Docker to run:

- the pipeline application
- PostgreSQL
- Redis
- Kafka
- object storage
- monitoring tools
- test services
- background workers
- supporting infrastructure

If you do not understand the Docker configuration, it can be difficult to understand how the repository actually runs.

A pipeline may work inside a container but fail on the host.

A database may be available inside Docker but not directly from the host.

An environment variable may exist in the container but not on your machine.

A volume may preserve data after a container is removed.

This chapter explains how to investigate Docker as part of understanding a Data Engineering repository.

---

## 1. What Is Docker?

Docker packages an application and its runtime environment into containers.

A container is an isolated runtime environment for a process and its dependencies.

A simple model is:

```text
Application
    ↓
Docker image
    ↓
Container
    ↓
Running process
```

Docker does not replace the application.

It provides a controlled way to package and run it.

For Data Engineering, this is useful because pipelines often depend on several services.

---

## 2. Why Docker Investigation Matters

Suppose a repository contains a Python ingestion service and PostgreSQL.

Without Docker, you may need to install and configure PostgreSQL manually.

With Docker Compose, the repository may define both services:

```text
+-------------------+
| pipeline container|
+---------+---------+
          |
          | network
          v
+-------------------+
| postgres container|
+-------------------+
```

The Docker configuration tells you how these components are expected to work together.

That makes Docker configuration part of repository architecture.

---

## 3. Docker Terms You Need to Know

Before investigating a Dockerized repository, understand a few basic terms.

### Image

An image is a packaged template used to create containers.

### Container

A container is a running instance created from an image.

### Dockerfile

A Dockerfile describes how an image is built.

### Docker Compose

Docker Compose describes multiple services and how they run together.

### Volume

A volume provides persistent storage that can exist independently from a container's writable layer.

### Network

A Docker network allows containers to communicate with each other.

These concepts are enough to begin repository investigation.

---

## 4. Find the Docker Files

Start at the repository root.

Look for:

```text
Dockerfile
Dockerfile.dev
Dockerfile.test
docker-compose.yml
docker-compose.yaml
compose.yml
compose.yaml
```

Also search directories such as:

```text
docker/
infra/
deploy/
scripts/
```

Do not assume there is only one Dockerfile.

A repository may have different images for development, testing, workers, or production.

---

## 5. Read the Dockerfile as a Build Recipe

A Dockerfile is a sequence of instructions used to build an image.

A generic Dockerfile may look like:

```dockerfile
FROM python:3.x
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

This is a generic example.

When investigating a real repository, use the actual Dockerfile.

Read it from top to bottom.

Ask:

- Which base image is used?
- Which files are copied?
- Which dependencies are installed?
- What commands run during the build?
- Which user runs the application?
- Which port is exposed?
- What command starts the container?

---

## 6. Understand `FROM`

`FROM` selects the base image.

Generic example:

```dockerfile
FROM python:3.x
```

The base image can affect:

- operating system
- runtime version
- installed system packages
- default environment
- available tools

If an application behaves differently inside Docker, inspect the base image.

A version difference between local and containerized environments can cause unexpected behavior.

---

## 7. Understand `WORKDIR`

`WORKDIR` sets the working directory inside the image.

Generic example:

```dockerfile
WORKDIR /app
```

Later commands are usually interpreted relative to that directory.

This matters when investigating:

- copied files
- startup commands
- relative paths
- configuration files
- log files
- application imports

If the application cannot find a file inside the container, check the working directory and copy paths.

---

## 8. Understand `COPY`

`COPY` moves files from the build context into the image.

Generic example:

```dockerfile
COPY src/ /app/src/
```

Ask:

- Which files are copied?
- Which files are excluded?
- Does `.dockerignore` remove files from the build context?
- Where do the files appear inside the image?

A common investigation mistake is assuming that every repository file exists inside the container.

That is not necessarily true.

---

## 9. Find `.dockerignore`

A `.dockerignore` file controls which files are excluded from the Docker build context.

Typical exclusions may include:

```text
.git
.venv
node_modules
__pycache__
*.log
.env
```

These are examples only.

Read the actual `.dockerignore` file.

It can explain why a file available on the host is missing during image building.

It is also important for security and build performance.

---

## 10. Understand `RUN`

`RUN` executes a command while the image is being built.

Generic example:

```dockerfile
RUN pip install -r requirements.txt
```

This is different from a command that runs when the container starts.

Think of it as:

```text
docker build
    ↓
RUN commands
    ↓
image
```

Do not confuse build-time behavior with runtime behavior.

---

## 11. Understand `CMD` and `ENTRYPOINT`

`CMD` and `ENTRYPOINT` help define how a container starts.

Generic example:

```dockerfile
CMD ["python", "app.py"]
```

Another image may use:

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
```

The exact behavior depends on how the image is built and invoked.

When investigating startup behavior, trace:

```text
Dockerfile
    ↓
ENTRYPOINT / CMD
    ↓
Container process
    ↓
Application entry point
```

This connects directly to Chapter 10.

---

## 12. Find the Real Container Entry Point

A Dockerfile command may start a script instead of the application directly.

For example:

```text
ENTRYPOINT
    ↓
start.sh
    ↓
migration command
    ↓
application startup
```

Do not stop at the Dockerfile.

Follow the startup script.

Search for:

- shell scripts
- Python modules
- package commands
- migration commands
- worker commands

This can reveal the actual startup sequence.

---

## 13. Docker Compose

Docker Compose is commonly used when a repository needs several services.

A generic Compose file might define:

```text
services:
  pipeline
  postgres
  redis
```

The services can communicate through a Docker network.

Think of Compose as a small local system definition.

---

## 14. Read Compose Services

For each service, identify:

- service name
- image
- build context
- Dockerfile
- command
- environment
- ports
- volumes
- networks
- dependencies

Create a table while investigating.

Example:

| Service | Purpose | Image/Build | Port | Storage |
|---|---|---|---|---|
| pipeline | application | local build | internal | none |
| postgres | database | PostgreSQL image | database port | volume |

This is a generic example.

The real repository may contain different services.

---

## 15. Service Names Matter

Inside a Compose network, services can often communicate using their service names.

Generic example:

```text
pipeline
   ↓
postgres
```

The application may therefore use a hostname such as:

```text
postgres
```

rather than:

```text
localhost
```

This is one of the most common Docker networking concepts to understand.

`localhost` inside a container refers to that container itself, not automatically to another container.

---

## 16. Docker Networking

Suppose Compose starts:

```text
+-------------+       +-------------+
| pipeline    | ----> | postgres    |
+-------------+       +-------------+
        Docker network
```

The pipeline container needs to know how to reach PostgreSQL.

Investigate:

- hostname
- port
- network
- protocol
- environment variables

Do not assume the host machine's network settings are the same as the container network.

---

## 17. Host Ports vs Container Ports

Port mappings can be confusing.

Generic Compose configuration might look like:

```yaml
ports:
  - "5433:5432"
```

This can mean:

```text
Host port 5433
       ↓
Container port 5432
```

The exact behavior depends on the configuration.

Inside the Docker network, another container may still connect to PostgreSQL using its service name and container port.

Do not automatically use the host port for container-to-container communication.

---

## 18. Volumes

Containers are not a substitute for persistent storage.

A database container may use a volume:

```text
Postgres container
       ↓
     Volume
       ↓
Persistent database files
```

Without appropriate persistence, removing a container can remove its writable data.

During repository investigation, identify:

- which volumes exist
- which service uses each volume
- what data is stored
- whether the volume is local or externally managed

---

## 19. Why Volumes Matter for Data Engineering

Data Engineering systems often contain stateful services.

Examples include:

- PostgreSQL
- Kafka
- object storage
- workflow metadata
- local test databases

If you restart or recreate containers, the persistence behavior matters.

Suppose a developer says:

> My database disappeared after I recreated the containers.

Check the volume configuration before assuming the database migration failed.

---

## 20. Environment Variables in Compose

Compose can pass environment variables into containers.

Generic example:

```yaml
environment:
  DATABASE_URL: ${DATABASE_URL}
```

Trace the value:

```text
Host / CI variable
       ↓
Compose
       ↓
Container environment
       ↓
Application configuration
       ↓
Database client
```

This connects Docker directly to Chapter 14.

---

## 21. `.env` and Compose

Compose projects often use environment files.

Possible sources include:

- `.env`
- explicit `env_file` configuration
- shell environment
- CI environment
- deployment configuration

The exact precedence depends on the Docker Compose setup.

Do not assume that the `.env` file is the only source.

When debugging a value, determine exactly where it came from.

---

## 22. `depends_on`

Compose can express relationships between services.

Generic example:

```text
pipeline
   ↓
depends on
postgres
```

This can help define startup relationships.

But service startup order is not automatically the same as service readiness.

A database container may be running while the database is still initializing.

This distinction matters.

---

## 23. Container Started vs Service Ready

Consider:

```text
Container starts
      ↓
PostgreSQL initializes
      ↓
Database accepts connections
```

Another service may attempt to connect before the final step.

This can produce errors such as connection refused or initialization failures.

During investigation, check whether the repository uses:

- health checks
- startup retries
- readiness checks
- dependency conditions
- application-level connection retries

Do not assume that a running container means the service is ready.

---

## 24. Health Checks

A health check can test whether a service is ready to perform useful work.

Generic flow:

```text
Container running
       ↓
Health check
       ↓
Healthy?
  /          \
yes          no
 ↓            ↓
continue     wait/retry
```

Find health checks in Compose or other infrastructure configuration.

Also inspect what they actually test.

A process being alive is not always enough.

---

## 25. Docker and Database Migrations

Dockerized repositories often run migrations as part of startup or deployment.

Possible flow:

```text
Container starts
      ↓
Migration command
      ↓
Database schema
      ↓
Application starts
```

Another repository may run migrations separately:

```text
Deployment
   ↓
Migration job
   ↓
Application deployment
```

Do not assume the sequence.

Find the actual command in Dockerfiles, Compose files, scripts, or CI/CD configuration.

---

## 26. Docker and Tests

Docker may provide services required by integration tests.

Generic example:

```text
Test runner
    ↓
PostgreSQL container
    ↓
Database tests
```

Or:

```text
Test runner
    ↓
Kafka container
    ↓
Consumer tests
```

When tests fail, determine whether the problem is:

- application code
- test code
- container startup
- service readiness
- networking
- configuration
- volume state

This prevents debugging the wrong layer.

---

## 27. Docker and CI

CI systems may run Docker images or Docker Compose services.

Investigate the CI workflow.

Ask:

- Is Docker available?
- Are services started before tests?
- Which image is built?
- Which Compose file is used?
- Are environment variables supplied?
- Are volumes used?
- Are ports required?
- Are logs collected when a service fails?

A local Docker workflow may not exactly match CI.

Find the difference instead of assuming they are identical.

---

## 28. Build Context

When Docker builds an image, it receives a build context.

Generic example:

```text
repository/
├── src/
├── Dockerfile
└── requirements.txt
```

The build context determines which files can be referenced by `COPY` and `ADD`.

If a Docker build fails because a file cannot be found, inspect:

- build context
- Dockerfile path
- `.dockerignore`
- COPY paths

---

## 29. Docker Build vs Docker Run

Separate build-time behavior from runtime behavior.

Build:

```text
Dockerfile
   ↓
docker build
   ↓
image
```

Run:

```text
image
   ↓
docker run / Compose
   ↓
container
   ↓
application
```

A package installed during the build is part of the image.

An environment variable supplied during runtime may not exist during the build.

This distinction is important when troubleshooting.

---

## 30. Inspect Running Containers

When investigating a live local environment, inspect what is actually running.

Useful questions include:

- Which containers are running?
- Which image is each container using?
- What command started it?
- What ports are exposed?
- What networks are attached?
- What environment is configured?
- Which volumes are mounted?

Use the repository's documented Docker commands where available.

Do not rely only on what the Compose file appears to say.

The running system is the final result of configuration, image builds, and commands.

---

## 31. Container Logs

Container logs are often the first place to investigate startup failures.

Look for:

- missing configuration
- connection errors
- migration failures
- import errors
- permission errors
- port conflicts
- service readiness problems

A useful path is:

```text
Container
   ↓
Startup log
   ↓
Error
   ↓
Configuration / dependency / code
```

Logs should be interpreted together with Docker configuration.

---

## 32. Inspecting the Container Filesystem

Sometimes you need to determine whether a file actually exists inside a container.

Examples of useful investigation questions:

- Is the application source present?
- Is the configuration file present?
- Is the migration directory present?
- Is the expected working directory correct?
- Are permissions correct?

This can quickly distinguish a host-side file problem from an image-build problem.

---

## 33. User and Permissions

Docker containers may run as a non-root user.

This is generally useful for reducing unnecessary privileges.

But it can create permission problems if files or mounted volumes have incompatible ownership.

When investigating:

```text
permission denied
```

check:

- container user
- file ownership
- directory permissions
- volume ownership
- mounted host paths

Do not solve permission problems by making everything world-writable without understanding the cause.

---

## 34. Docker Networking Failure Investigation

Suppose the pipeline cannot connect to PostgreSQL.

Investigate in this order:

```text
Is postgres container running?
        ↓
Is postgres ready?
        ↓
Are both services on the expected network?
        ↓
Is the hostname correct?
        ↓
Is the container port correct?
        ↓
Is authentication configuration correct?
        ↓
Is PostgreSQL accepting connections?
```

This is usually more effective than immediately changing application code.

---

## 35. Docker Volume Failure Investigation

Suppose a service starts with missing data.

Check:

```text
Expected volume
      ↓
Actual mounted volume
      ↓
Volume contents
      ↓
Container path
      ↓
Application path
```

A volume can be mounted correctly but at the wrong path.

Again, follow the actual configuration.

---

## 36. Docker Image Version Problems

Sometimes a container is running an image different from the one you expected.

Possible causes include:

- stale image
- wrong tag
- cached build
- different build context
- CI-built image
- registry image

When behavior seems inconsistent, identify the actual image and version being used.

Do not assume the latest source code automatically means the running container contains that source code.

---

## 37. Docker Compose Investigation Workflow

Use this workflow when entering an unfamiliar repository.

### Step 1 — Find Docker files

Locate Dockerfiles, Compose files, `.dockerignore`, and Docker-related scripts.

### Step 2 — Identify images

Determine whether each service uses a local build or a prebuilt image.

### Step 3 — Map services

Write down every service and its purpose.

### Step 4 — Trace entry points

Follow `ENTRYPOINT`, `CMD`, and Compose commands.

### Step 5 — Map configuration

Identify environment variables and configuration files.

### Step 6 — Map networks

Determine how services communicate.

### Step 7 — Map ports

Separate host ports from container ports.

### Step 8 — Map volumes

Identify which state is persistent.

### Step 9 — Check dependencies

Find service dependencies and readiness checks.

### Step 10 — Check migrations

Determine how and when database migrations run.

### Step 11 — Check tests

Determine which services are required for automated tests.

### Step 12 — Check CI

Compare local Docker execution with CI execution.

---

## 38. Example: Pipeline Cannot Reach PostgreSQL

Suppose the application reports:

```text
connection refused
```

Do not immediately change the database URL.

Build the map:

```text
pipeline container
      ↓
Docker network
      ↓
postgres service
      ↓
PostgreSQL
```

Then verify each boundary.

Possible findings include:

- PostgreSQL container is stopped
- database is not ready
- wrong hostname
- wrong port
- containers are on different networks
- credentials are wrong
- PostgreSQL is configured differently

The error message alone does not identify which layer failed.

---

## 39. Example: Container Starts Then Exits

Suppose a container repeatedly starts and stops.

Investigate:

```text
Container starts
      ↓
Entry point
      ↓
Startup command
      ↓
Application initialization
      ↓
Error
      ↓
Process exits
```

Check logs first.

Then check:

- missing environment variable
- invalid command
- missing file
- import failure
- database connection
- migration failure
- permission problem

Do not assume Docker itself is broken.

The application may be exiting normally because startup failed.

---

## 40. Common Mistakes

### Mistake 1 — Treating Docker as a black box

Read the Dockerfile and Compose configuration.

### Mistake 2 — Using `localhost` for another container

Inside a container, `localhost` normally refers to that container.

### Mistake 3 — Confusing host and container ports

Trace the actual network path.

### Mistake 4 — Assuming a running container means the service is ready

Check readiness and health behavior.

### Mistake 5 — Ignoring volumes

Persistent state may depend on volume configuration.

### Mistake 6 — Ignoring `.dockerignore`

A required file may never enter the build context.

### Mistake 7 — Confusing build-time and runtime configuration

They happen at different stages.

### Mistake 8 — Assuming local and CI Docker environments are identical

Compare their configuration.

### Mistake 9 — Rebuilding randomly

First determine whether the problem is image, container, configuration, network, or application behavior.

### Mistake 10 — Ignoring image versions

The running image may not contain the code you think it does.

---

## 41. Production Considerations

Docker in production introduces additional concerns.

Consider:

- image versioning
- immutable deployments
- resource limits
- health checks
- secret handling
- network isolation
- persistent storage
- logging
- monitoring
- graceful shutdown
- image vulnerability management
- reproducible builds

Production container behavior may differ from a local Compose environment.

Do not treat local Docker configuration as proof of production architecture.

Use the actual deployment configuration as evidence.

---

## 42. Practical Checklist

Before saying that you understand the repository's Docker setup, check:

- [ ] I found the Dockerfiles.
- [ ] I found the Compose files.
- [ ] I found `.dockerignore` where applicable.
- [ ] I know the base image.
- [ ] I know what files are copied into the image.
- [ ] I know how dependencies are installed.
- [ ] I know the container entry point.
- [ ] I know which services exist.
- [ ] I know what each service does.
- [ ] I understand the Docker networks.
- [ ] I understand host vs container ports.
- [ ] I know which volumes are used.
- [ ] I know how environment variables enter containers.
- [ ] I understand service readiness.
- [ ] I know how migrations are run.
- [ ] I know how tests use Docker services.
- [ ] I understand the CI Docker setup.
- [ ] I can investigate container startup failures.
- [ ] I can investigate networking failures.
- [ ] I can investigate volume and persistence problems.

---

## 43. What You Learned

In this recipe, you learned how to investigate Docker as part of understanding a Data Engineering repository.

You learned how to:

- understand images and containers
- read Dockerfiles
- follow `FROM`, `COPY`, `RUN`, `CMD`, and `ENTRYPOINT`
- find the real container entry point
- understand Docker Compose
- map services
- understand Docker networking
- distinguish host and container ports
- investigate volumes
- trace environment variables into containers
- understand service readiness
- investigate health checks
- connect Docker to migrations and tests
- investigate CI Docker behavior
- troubleshoot networking, storage, startup, and image problems
- and build a Docker system map

The main lesson is simple:

> Docker configuration is part of the application's execution path.

When investigating a Data Engineering repository, do not stop at the application source code. Find the image, container, network, volumes, configuration, and startup command that make the application run.

---

## Recipe Preview

The repository investigation map now includes the execution environment:

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
    ↓
Docker
```

These pieces help you understand the repository from source code all the way to a running local system.

Next: **Chapter 16 — Understanding CI/CD**.