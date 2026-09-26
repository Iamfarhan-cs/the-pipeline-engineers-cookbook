# Chapter 10 — Finding the Entry Point

Before you can understand what a Data Engineering repository does, you need to know where execution starts.

This sounds simple.

In a small script, it may be obvious:

```text
python pipeline.py
        |
        v
     main()
```

In a real repository, it may not be obvious at all.

The system may start through:

- A command-line command
- A Python module
- A shell script
- A Docker entrypoint
- Docker Compose
- A scheduler
- An Airflow DAG
- A web server
- A background worker
- A Kafka consumer
- A queue consumer
- A CI job
- A cloud job

There may also be several entry points for different processes.

This chapter teaches you how to find them and trace what happens after execution begins.

---

# What Is an Entry Point?

An entry point is the place where a particular execution path begins.

For a command-line application, it may be the function called by the command.

For a web application, it may be the server configuration that starts the application.

For a worker, it may be the process that starts the worker loop.

For an Airflow pipeline, the DAG definition may be the entry point into scheduled orchestration.

For a container, the entry point may be defined by the container's startup command.

The important question is:

> What causes this piece of code to start running?

That question is more useful than simply asking:

> Which file looks like the main file?

---

# Why Finding the Entry Point Matters

Suppose a ticket says:

> Add retry handling to the ingestion pipeline.

You find a file called `ingestion.py`.

You could edit it immediately.

But you do not yet know:

- Who calls it?
- When is it called?
- Is it called once or repeatedly?
- Does a scheduler trigger it?
- Does a worker call it?
- Does another service call it?
- Is there already retry logic above it?
- What configuration reaches it?
- How are failures handled by the caller?

Without knowing the execution path, you may add retry logic at the wrong level.

Finding the entry point gives you the beginning of the path.

---

# Entry Point vs Business Logic

An entry point does not necessarily contain the main business logic.

Consider:

```text
Command
  |
  v
main()
  |
  v
run_pipeline()
  |
  v
fetch_data()
  |
  v
validate_data()
  |
  v
store_data()
```

`main()` starts the execution.

`run_pipeline()` coordinates the work.

The lower-level functions perform specific operations.

This distinction is important.

Finding the entry point is only the beginning of repository investigation.

---

# There Can Be More Than One Entry Point

An existing repository may contain several processes.

For example:

```text
Repository
   |
   +-- API service
   |      |
   |      +-- API entry point
   |
   +-- Worker
   |      |
   |      +-- Worker entry point
   |
   +-- Scheduler
   |      |
   |      +-- Scheduler entry point
   |
   +-- Migration command
          |
          +-- Migration entry point
```

**GENERIC EXAMPLE**

Do not assume the repository has one universal starting point.

First identify which process is relevant to the task.

---

# Start With the Command

One of the fastest ways to find an entry point is to start from the command that runs the system.

Ask:

> What command does the developer or deployment system execute?

Look in:

- README files
- Makefiles
- `pyproject.toml`
- `package.json`
- Dockerfiles
- Docker Compose files
- Shell scripts
- CI workflows
- Deployment configuration

For example, a README might contain a command such as:

```text
python -m app.worker
```

That command already tells you something important.

The process starts from a Python module.

---

# Command → Module → Function

Suppose the command is:

```text
python -m app.worker
```

Your investigation becomes:

```text
python -m app.worker
        |
        v
app/worker.py
        |
        v
startup function
        |
        v
worker loop
```

**GENERIC EXAMPLE**

The exact module and function names must be verified from the repository.

The goal is to turn a command into a concrete execution path.

---

# Python Entry Points

Python repositories can start in several ways.

Common patterns include:

### Direct script execution

```text
python pipeline.py
```

### Module execution

```text
python -m package.module
```

### Console scripts

A Python project can define command-line entry points through its packaging configuration.

### Application server

A web server may import an application object and start it.

### Worker process

A worker command may import a worker module and start a processing loop.

These are common patterns.

Do not assume a repository uses any particular one until you inspect its configuration.

---

# Look for `__main__`

A common Python pattern is:

```python
if __name__ == "__main__":
    main()
```

**GENERIC EXAMPLE**

This means the code inside the condition runs when the file is executed as a script.

It does not prove that this is the repository's main production entry point.

The file may also be imported by another process.

Always check how the file is actually invoked.

---

# Package Entry Points

Some Python applications are started through package metadata rather than direct script execution.

A project configuration may define a command that maps to a Python function.

Conceptually:

```text
command
   |
   v
package configuration
   |
   v
module:function
   |
   v
application code
```

**GENERIC EXAMPLE**

When investigating a Python repository, inspect project metadata if the startup command is not obvious.

---

# Shell Script Entry Points

Sometimes the visible command is a shell script.

For example:

```text
./scripts/start-worker.sh
```

The shell script may:

- Set environment variables
- Run migrations
- Wait for dependencies
- Start the application
- Pass command-line arguments
- Configure logging
- Execute another command

Therefore, do not stop at the shell script.

Open it and continue following the command.

---

# Docker Entry Points

Docker adds another layer.

A Dockerfile can define how the container starts.

Conceptually:

```text
Docker image
    |
    v
Container starts
    |
    v
ENTRYPOINT / CMD
    |
    v
Application command
    |
    v
Application code
```

**GENERIC EXAMPLE**

If you are investigating a containerized pipeline, inspect the Dockerfile.

Look for:

- `ENTRYPOINT`
- `CMD`
- Startup scripts
- Working directory
- Environment variables

---

# Docker Compose Entry Points

Docker Compose can override or provide commands for services.

A Compose file may conceptually define:

```yaml
services:
  worker:
    command: ...
```

**GENERIC EXAMPLE**

The exact command must be read from the actual Compose configuration.

This is why repository investigation should include infrastructure files.

The Python file alone may not tell you how the production-like environment starts it.

---

# Environment Variables Can Affect Startup

The entry point may depend on configuration.

For example:

```text
ENVIRONMENT=production
DATABASE_URL=...
WORKER_MODE=...
```

These values may change:

- Which service starts
- Which database is used
- Which configuration file is loaded
- Which feature is enabled
- Which logging level is active

Never copy real secrets into notes or documentation.

Record configuration names and behavior, not sensitive values.

---

# Web Application Entry Points

A web service has a different kind of entry point.

The operating system starts a server process.

The server loads the application.

Requests then enter through routes or handlers.

Conceptually:

```text
Server process
     |
     v
Application object
     |
     v
Router
     |
     v
Handler
     |
     v
Service logic
```

**GENERIC EXAMPLE**

If a Data Engineering repository exposes ingestion through an API, the API handler may be the entry point for each request, while the server startup is the process-level entry point.

These are different levels of entry.

---

# Worker Entry Points

Background workers often run continuously.

Conceptually:

```text
Worker process starts
        |
        v
Connect to dependencies
        |
        v
Start worker loop
        |
        v
Wait for work
        |
        v
Receive item
        |
        v
Process item
        |
        +----> repeat
```

The worker's startup function and the processing function are not necessarily the same.

When investigating a worker, trace both.

---

# Scheduler Entry Points

A scheduled pipeline may start because a scheduler triggers it.

Conceptually:

```text
Scheduler
   |
   v
Schedule reached
   |
   v
Job starts
   |
   v
Pipeline function
   |
   v
Processing
```

**GENERIC EXAMPLE**

The scheduler configuration may be outside the pipeline code.

This is why you should inspect orchestration configuration when tracing scheduled jobs.

---

# Airflow Entry Points

In an Airflow-based system, a DAG defines an orchestration workflow.

Conceptually:

```text
Airflow scheduler
        |
        v
       DAG
        |
        v
       Task
        |
        v
Pipeline code
```

**GENERIC EXAMPLE**

The DAG is an orchestration entry point for the scheduled workflow.

The actual business logic may live in another module called by the task.

Do not assume the DAG file contains all processing logic.

---

# Kafka Consumer Entry Points

A Kafka-based pipeline may start by launching a consumer.

Conceptually:

```text
Consumer process
      |
      v
Consumer loop
      |
      v
Poll message
      |
      v
Process message
      |
      v
Commit / acknowledge
```

**GENERIC EXAMPLE**

When investigating a consumer, identify both:

1. What starts the consumer process?
2. What function handles each message?

These are separate investigation points.

---

# Queue Consumer Entry Points

Other queue systems follow a similar pattern.

Conceptually:

```text
Worker startup
     |
     v
Connect to queue
     |
     v
Receive message
     |
     v
Handler
     |
     v
Processing
```

The queue connection setup and message handler may live in different files.

Follow the call chain.

---

# CI Entry Points

CI pipelines also have entry points.

A workflow may start because:

- Code was pushed
- A pull request was opened
- A schedule ran
- A manual action was triggered

Then it may execute:

```text
Workflow trigger
    |
    v
Job
    |
    v
Step
    |
    v
Command
    |
    v
Application / tests
```

**GENERIC EXAMPLE**

If you are investigating why tests run differently in CI, tracing this path can reveal the actual command and environment.

---

# Find the First Real Function

Once you find the startup command, keep following it until you reach meaningful application logic.

For example:

```text
docker compose up
      |
      v
container command
      |
      v
startup script
      |
      v
python -m app.worker
      |
      v
worker.start()
      |
      v
run_pipeline()
```

At this point, you have reached the first meaningful pipeline function.

This is often the point where deeper code investigation becomes useful.

---

# Do Not Stop at the First Function

Finding `main()` is not enough.

Suppose:

```text
main()
  |
  v
run()
  |
  v
execute_pipeline()
  |
  v
process_batch()
  |
  v
write_records()
```

If the task is about database writes, `main()` is not the relevant code.

If the task is about scheduling, `main()` may also not be the relevant code.

Continue tracing until you reach the operation related to the task.

---

# Find the Caller

When you find an interesting function, ask:

> Who calls this function?

This is often more important than asking what the function calls.

Suppose you find:

```text
write_records()
```

Search for callers.

You may discover:

```text
worker_loop()
   |
   v
process_batch()
   |
   v
write_records()
```

Now you know where the database write fits into the execution path.

---

# Find the Callees

Then ask:

> What does this function call?

For example:

```text
process_batch()
     |
     +-- validate()
     +-- transform()
     +-- write_records()
     +-- update_status()
```

This reveals the next layer of the execution path.

Together, callers and callees build the call graph you need for the investigation.

---

# Call Graph

A call graph shows which functions invoke which other functions.

Conceptually:

```text
entry_point()
     |
     v
run_pipeline()
     |
     +---- fetch_data()
     |
     +---- validate_data()
     |
     +---- store_data()
              |
              +---- insert_records()
```

**GENERIC EXAMPLE**

You do not need a formal graph tool for every investigation.

A simple hand-written map can be enough.

---

# Entry Point vs Trigger

These terms can be related but are not identical.

A **trigger** is the event that causes execution to start.

An **entry point** is the code or command where that execution enters the application.

For example:

```text
Trigger:
Scheduled time reached
        |
        v
Entry point:
DAG task starts
        |
        v
Pipeline function
```

Another example:

```text
Trigger:
Message arrives
        |
        v
Entry point:
Consumer callback
        |
        v
Processing function
```

This distinction becomes useful when debugging scheduled and event-driven systems.

---

# Entry Point vs Pipeline Stage

A pipeline stage is a piece of processing.

An entry point is where execution begins for the relevant path.

For example:

```text
Entry point
     |
     v
Acquire
     |
     v
Validate
     |
     v
Transform
     |
     v
Store
```

Finding the entry point helps you discover the stages that follow.

---

# Use Tests to Confirm the Entry Path

Tests can help verify what you discovered.

Suppose you believe:

```text
worker.start()
      |
      v
run_pipeline()
```

Look for tests that call or mock these functions.

Tests may show:

- How the pipeline is started
- Which dependencies are injected
- Which function is expected to be called
- What happens when startup fails

Tests are evidence that can confirm or challenge your initial understanding.

---

# Use Docker to Confirm the Entry Path

If the repository is containerized, compare your understanding with the container configuration.

Trace:

```text
Docker Compose service
       |
       v
command / entrypoint
       |
       v
application startup
       |
       v
pipeline code
```

If your code investigation says one function starts the worker but Docker starts another command, investigate the difference.

Do not ignore conflicting evidence.

---

# Use CI to Confirm the Entry Path

CI can also reveal how the repository expects code to run.

For example, a workflow may execute a command that differs from the README.

That can reveal:

- An additional startup script
- Required environment setup
- Test-specific entry points
- Different commands for different components

Again, the goal is to verify behavior from multiple sources when necessary.

---

# A Practical Investigation Example

Imagine you receive a task:

> The ingestion worker is not processing new records.

You begin at the repository root.

### Step 1 — Find how the worker starts

Search the README and Docker configuration.

### Step 2 — Find the worker command

Follow the command into the application.

### Step 3 — Find the worker loop

Identify where it waits for new work.

### Step 4 — Find the message or batch handler

Follow the function called when work arrives.

### Step 5 — Find the processing function

Follow validation, transformation, and storage.

### Step 6 — Find failure handling

Check whether errors are retried, logged, or discarded.

### Step 7 — Find tests

Look for tests covering worker startup and processing.

### Step 8 — Compare with runtime configuration

Verify that the deployment actually starts the same process you investigated.

Now you have an evidence-based execution path.

---

# Finding an Entry Point in an Unfamiliar Python Repository

Here is a practical search order.

### 1. README

Look for commands such as:

```text
python ...
python -m ...
make ...
docker compose ...
```

### 2. Project configuration

Inspect:

```text
pyproject.toml
setup configuration
package metadata
```

**GENERIC EXAMPLE**

### 3. Scripts

Look for startup scripts.

### 4. Docker

Inspect `Dockerfile` and Compose configuration.

### 5. CI

Find commands used by automated workflows.

### 6. Code

Search for `__main__`, application objects, worker loops, and command functions.

### 7. Tests

Use tests to verify the discovered path.

---

# Finding an Entry Point in an Unfamiliar Repository

Do not limit this technique to Python.

The general process is:

```text
How is the system started?
        |
        v
What command/configuration starts it?
        |
        v
What module/class/function receives control?
        |
        v
What does it call?
        |
        v
Where does the actual pipeline work begin?
```

The language changes.

The investigation method remains similar.

---

# Common Mistakes

## Mistake 1: Assuming the obvious file is the entry point

A file named `main.py` may not be used by production.

Verify how the application is started.

## Mistake 2: Finding the entry point and stopping

The entry point may only initialize the system.

Continue into the relevant processing path.

## Mistake 3: Ignoring Docker

The container command may change how the application starts.

## Mistake 4: Ignoring schedulers

A scheduled pipeline may be triggered outside the application code.

## Mistake 5: Ignoring workers

Background processing often starts through a separate process.

## Mistake 6: Ignoring tests

Tests can reveal the intended execution path.

## Mistake 7: Assuming one repository has one entry point

Large repositories may contain multiple services and jobs.

## Mistake 8: Guessing from filenames

File names are clues, not proof.

---

# Production Considerations

When identifying an entry point for production work, also determine:

- How the process is started
- Which user runs it
- Which environment it uses
- Which configuration it loads
- Which dependencies it requires
- Whether it runs once or continuously
- Whether a scheduler triggers it
- Whether multiple instances can run
- How it shuts down
- What happens if startup fails
- What happens if a worker crashes
- How it is restarted

These questions become increasingly important as the system becomes more distributed.

---

# Practical Checklist

When investigating a pipeline entry point, ask:

- [ ] What command starts the relevant process?
- [ ] Is there a shell script?
- [ ] Is there a Docker entrypoint?
- [ ] Is Docker Compose involved?
- [ ] Is a scheduler involved?
- [ ] Is a worker involved?
- [ ] Is there an API server?
- [ ] Is there a queue or Kafka consumer?
- [ ] Where does execution enter the application?
- [ ] What function receives control first?
- [ ] What does that function call?
- [ ] Where does the actual pipeline work begin?
- [ ] What configuration affects startup?
- [ ] What tests confirm the execution path?
- [ ] Does CI use the same or a different command?
- [ ] Does the runtime configuration match the code I investigated?
- [ ] Are there multiple entry points?

---

# What You Learned

In this recipe, you learned:

- An entry point is where a particular execution path begins.
- A repository can have multiple entry points.
- The fastest way to find one is often to start from the command that launches the system.
- Python projects can use scripts, modules, package entry points, servers, and workers.
- Docker and Docker Compose can define or modify startup behavior.
- Schedulers and message consumers introduce different kinds of triggers and entry points.
- Finding `main()` is not always enough.
- You should follow callers and callees to understand the relevant execution path.
- Tests, Docker configuration, and CI can help confirm your understanding.
- Entry point investigation is about tracing real execution, not guessing from filenames.

The main lesson is:

**Do not start by asking which file to edit. Start by asking what starts the system and trace that execution path until you reach the part of the pipeline that actually matters to your task.**

---
