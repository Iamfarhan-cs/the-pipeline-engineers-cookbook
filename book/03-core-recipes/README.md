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

## Learning Order

The recipes are arranged in **beginner-to-production learning order**. Each recipe introduces a problem that prepares you for the next one.

### Stage 1 — Build the Pipeline

| Recipe | Topic |
|---|---|
| [Recipe 17 — Create an Ingestion Pipeline](17-create-an-ingestion-pipeline.md) | Ingestion |
| [Recipe 18 — Create a Staging Layer](18-create-a-staging-layer.md) | Staging |
| [Recipe 19 — Validate Incoming Data](19-validate-incoming-data.md) | Validation |

### Stage 2 — Make the Pipeline Correct and Observable

| Recipe | Topic |
|---|---|
| [Recipe 20 — Add Error Handling](20-add-error-handling.md) | Error handling |
| [Recipe 21 — Add Processing Status](21-add-processing-status.md) | Processing state |
| [Recipe 22 — Add Idempotency](22-add-idempotency.md) | Idempotency |
| [Recipe 23 — Add Deduplication](23-add-deduplication.md) | Deduplication |

### Stage 3 — Recover from Failure

| Recipe | Topic |
|---|---|
| [Recipe 24 — Add Retry Logic](24-add-retry-logic.md) | Retries |
| [Recipe 25 — Quarantine Failed Data](25-quarantine-failed-data.md) | Quarantine |
| [Recipe 26 — Replay / Reprocessing](26-replay-reprocessing.md) | Replay |

### Stage 4 — Process Data Efficiently

| Recipe | Topic |
|---|---|
| [Recipe 27 — Incremental Processing](27-incremental-processing.md) | Incremental processing |
| [Recipe 28 — Checkpointing](28-checkpointing.md) | Checkpoints |
| [Recipe 29 — Backfill Historical Data](29-backfill-historical-data.md) | Backfill |

### Stage 5 — Handle Real-World Data Problems

| Recipe | Topic |
|---|---|
| [Recipe 30 — Handle Late Data](30-handle-late-data.md) | Late data |
| [Recipe 31 — Handle Missing Data](31-handle-missing-data.md) | Completeness |
| [Recipe 32 — Data Reconciliation](32-data-reconciliation.md) | Reconciliation |

### Stage 6 — Evolve and Operate the Pipeline

| Recipe | Topic |
|---|---|
| [Recipe 33 — Handle Schema Changes](33-handle-schema-changes.md) | Schema evolution |
| [Recipe 34 — Handle Backpressure](34-handle-backpressure.md) | Backpressure |
| [Recipe 35 — Dead-Letter / Data-Quality Lifecycle](35-dead-letter-dq-lifecycle.md) | DLQ / DQ lifecycle |

## The Complete Learning Path

```text
17 Ingestion
 ↓
18 Staging
 ↓
19 Validation
 ↓
20 Error Handling
 ↓
21 Processing Status
 ↓
22 Idempotency
 ↓
23 Deduplication
 ↓
24 Retry Logic
 ↓
25 Quarantine
 ↓
26 Replay / Reprocessing
 ↓
27 Incremental Processing
 ↓
28 Checkpointing
 ↓
29 Backfill
 ↓
30 Late Data
 ↓
31 Missing Data
 ↓
32 Reconciliation
 ↓
33 Schema Changes
 ↓
34 Backpressure
 ↓
35 Dead-Letter / Data-Quality Lifecycle
```

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
