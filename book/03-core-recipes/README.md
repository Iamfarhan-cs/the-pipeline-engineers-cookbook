# Part III — Core Recipes

Part III is the practical implementation section of the book.

Parts I and II explain how Data Engineering pipelines work and how to investigate an existing repository. Part III turns those concepts into hands-on **recipes** that can be implemented, tested, observed, broken, recovered, and operated.

## Recipe Philosophy

These are not theory-only chapters. Each recipe is designed around a real production pipeline problem.

By the end of every recipe, you should be able to:

- Recognize the problem in a real pipeline.
- Explain why the problem happens.
- Choose an appropriate engineering solution.
- Implement the mechanism from scratch.
- Test the implementation and important edge cases.
- Observe the behavior through logs, metrics, and state.
- Break the implementation intentionally.
- Diagnose the failure.
- Recover the pipeline safely.
- Understand the production tools commonly used for the same problem.
- Operate the mechanism using a practical runbook.
- Prove that the implementation is complete.

The core learning loop is:

```text
REAL DE PROBLEM
      ↓
UNDERSTAND THE MECHANISM
      ↓
IMPLEMENT IT YOURSELF
      ↓
TEST IT
      ↓
OBSERVE IT
      ↓
BREAK IT INTENTIONALLY
      ↓
DIAGNOSE THE FAILURE
      ↓
RECOVER IT
      ↓
LEARN PRODUCTION TOOLS
      ↓
OPERATE IT
      ↓
PROVE YOU ARE DONE
```

## Recipe Collection

### Ingestion and Data Boundaries

| Recipe | Topic | What you learn |
|---|---|---|
| [Recipe 17 — Create an Ingestion Pipeline](17-create-an-ingestion-pipeline.md) | Ingestion | Build a complete source-to-database pipeline. |
| [Recipe 18 — Create a Staging Layer](18-create-a-staging-layer.md) | Staging | Create a durable boundary between ingestion and processing. |
| [Recipe 19 — Validate Incoming Data](19-validate-incoming-data.md) | Validation | Detect invalid data before it reaches downstream processing. |

### Reliability and Correctness

| Recipe | Topic | What you learn |
|---|---|---|
| [Recipe 20 — Add Idempotency](20-add-idempotency.md) | Idempotency | Make repeated delivery safe. |
| [Recipe 21 — Add Deduplication](21-add-deduplication.md) | Deduplication | Detect and control duplicate logical records. |
| [Recipe 22 — Add Processing Status](22-add-processing-status.md) | Processing state | Track what happened to each record. |
| [Recipe 23 — Add Error Handling](23-add-error-handling.md) | Error handling | Classify failures and respond safely. |
| [Recipe 24 — Add Retry Logic](24-add-retry-logic.md) | Retries | Retry transient failures without creating retry storms. |
| [Recipe 25 — Replay / Reprocessing](25-replay-reprocessing.md) | Replay | Reprocess selected data safely and repeatedly. |
| [Recipe 26 — Quarantine Failed Data](26-quarantine-failed-data.md) | Quarantine | Isolate failed records while preserving evidence for recovery. |

### Historical and Incremental Processing

| Recipe | Topic | What you learn |
|---|---|---|
| [Recipe 27 — Backfill Historical Data](27-backfill-historical-data.md) | Backfill | Process historical ranges safely without overwhelming the pipeline. |
| [Recipe 28 — Incremental Processing](28-incremental-processing.md) | Incremental loads | Process only new or changed data using a durable progress boundary. |
| [Recipe 29 — Checkpointing](29-checkpointing.md) | Checkpoints | Persist safe processing progress and recover after crashes. |
| [Recipe 30 — Handle Late Data](30-handle-late-data.md) | Late data | Handle events that arrive after their expected processing window. |
| [Recipe 31 — Handle Missing Data](31-handle-missing-data.md) | Completeness | Detect missing records, windows, files, or partitions and recover safely. |
| [Recipe 32 — Data Reconciliation](32-data-reconciliation.md) | Reconciliation | Prove that source and target states agree according to defined controls. |

### Data Quality and Pipeline Evolution

| Recipe | Topic | What you learn |
|---|---|---|
| [Recipe 33 — Dead-Letter / Data-Quality Lifecycle](33-dead-letter-dq-lifecycle.md) | DLQ / DQ lifecycle | Move failed data through detection, investigation, repair, replay, and final state. |
| [Recipe 34 — Handle Schema Changes](34-handle-schema-changes.md) | Schema evolution | Detect schema changes and apply safe compatibility and migration patterns. |
| [Recipe 35 — Handle Backpressure](35-handle-backpressure.md) | Backpressure | Detect downstream pressure, bound work, and recover without uncontrolled overload. |

## Standard Recipe Structure

Every core recipe should follow the same engineering structure:

1. **Problem Recognition**
2. **Concept and Reasoning**
3. **Implementation**
4. **Testing**
5. **Observability**
6. **Intentional Failure**
7. **Recovery**
8. **Production Tools You Should Know**
9. **Production Runbook**
10. **Common Mistakes**
11. **Definition of Done**
12. **What You Learned**

The implementation should be practical enough that you can reproduce the mechanism without following another tutorial.

## How the Recipes Build on Each Other

The recipes form an evolving pipeline rather than isolated examples:

```text
Recipe 17
Ingestion
   ↓
Recipe 18
Staging
   ↓
Recipe 19
Validation
   ↓
Recipe 20
Idempotency
   ↓
Recipe 21
Deduplication
   ↓
Recipe 22
Processing Status
   ↓
Recipe 23
Error Handling
   ↓
Recipe 24
Retries
   ↓
Recipe 25
Replay / Reprocessing
   ↓
Recipe 26
Quarantine
   ↓
Recipe 27
Backfill
   ↓
Recipe 28
Incremental Processing
   ↓
Recipe 29
Checkpointing
   ↓
Recipe 30
Late Data
   ↓
Recipe 31
Missing Data
   ↓
Recipe 32
Reconciliation
   ↓
Recipe 33
Dead-Letter / Data-Quality Lifecycle
   ↓
Recipe 34
Schema Changes
   ↓
Recipe 35
Backpressure
```

Each later recipe can build on mechanisms established earlier instead of treating every problem as a completely new system.

## Production Tools Rule

Recipes teach the underlying engineering mechanism first.

The **Production Tools You Should Know** section introduces a maximum of three relevant production tools per recipe. These tools provide vocabulary and recognition of how the same problem is handled in real systems.

They do not replace understanding the underlying mechanism.

## Definition of a Complete Recipe

A recipe is considered complete when you can independently:

- Explain the production problem.
- Identify the failure mode.
- Design a safe solution.
- Implement it from scratch.
- Write tests for normal and failure paths.
- Observe the important operational signals.
- Intentionally break it.
- Diagnose the failure from evidence.
- Recover without creating additional corruption or loss.
- Explain the relevant production tools.
- Follow a runbook to operate the mechanism.

The goal is not to memorize recipes.

The goal is to develop the ability to recognize and solve recurring Data Engineering pipeline problems independently.
