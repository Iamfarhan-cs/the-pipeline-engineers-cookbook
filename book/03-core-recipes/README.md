# Part III — Core Pipeline Recipes

Part III is the practical implementation section of the book.

> **Important numbering rule:** Part III has its **own independent recipe sequence**. Core recipes start at **Recipe 1** and continue onward. Their numbers are separate from the chapter numbering used by the rest of the book.

## Recipe Philosophy

These are not theory-only chapters. Each recipe is built around a real production Data Engineering problem.

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
- Understand the relevant production tools.
- Operate the mechanism using a practical runbook.
- Prove that the implementation is complete.

The learning loop is:

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

## Core Recipe Sequence

### Stage 1 — Build the Pipeline

| Recipe | Topic |
|---:|---|
| [Recipe 1 — Create an Ingestion Pipeline](1-create-an-ingestion-pipeline.md) | Ingestion |
| [Recipe 2 — Create a Staging Layer](2-create-a-staging-layer.md) | Staging |
| [Recipe 3 — Validate Incoming Data](3-validate-incoming-data.md) | Boundary validation |
| [Recipe 4 — Add Error Handling](4-add-error-handling.md) | Error classification |
| [Recipe 5 — Add Processing Status](5-add-processing-status.md) | Processing state |

### Stage 2 — Make Execution Safe

| Recipe | Topic |
|---:|---|
| [Recipe 6 — Add Idempotency](6-add-idempotency.md) | Safe repeated execution |
| [Recipe 7 — Add Deduplication](7-add-deduplication.md) | Duplicate records |
| [Recipe 8 — Add Retry Logic](8-add-retry-logic.md) | Bounded retries |
| [Recipe 9 — Quarantine Failed Data](9-quarantine-failed-data.md) | Failure isolation |
| [Recipe 10 — Replay / Reprocessing](10-replay-reprocessing.md) | Controlled recovery |

### Stage 3 — Process Data Correctly and Efficiently

| Recipe | Topic |
|---:|---|
| [Recipe 11 — Incremental Processing](11-incremental-processing.md) | Incremental state |
| [Recipe 12 — Checkpointing](12-checkpointing.md) | Durable progress |
| [Recipe 13 — Backfill Historical Data](13-backfill-historical-data.md) | Historical recovery |
| [Recipe 14 — Handle Late Data](14-handle-late-data.md) | Event-time correctness |
| [Recipe 15 — Handle Missing Data](15-handle-missing-data.md) | Completeness |
| [Recipe 16 — Data Reconciliation](16-data-reconciliation.md) | Source-to-target correctness |

### Stage 4 — Handle Production-Scale Pipeline Problems

| Recipe | Topic |
|---:|---|
| [Recipe 17 — Handle Schema Changes](17-handle-schema-changes.md) | Schema evolution |
| [Recipe 18 — Handle Backpressure](18-handle-backpressure.md) | Flow control |
| [Recipe 19 — Dead-Letter / Data-Quality Lifecycle](19-dead-letter-dq-lifecycle.md) | Failure and DQ lifecycle |

## Learning Order

```text
1  Ingestion
   ↓
2  Staging
   ↓
3  Validation
   ↓
4  Error Handling
   ↓
5  Processing Status
   ↓
6  Idempotency
   ↓
7  Deduplication
   ↓
8  Retry
   ↓
9  Quarantine
   ↓
10 Replay / Reprocessing
   ↓
11 Incremental Processing
   ↓
12 Checkpointing
   ↓
13 Backfill
   ↓
14 Late Data
   ↓
15 Missing Data
   ↓
16 Reconciliation
   ↓
17 Schema Changes
   ↓
18 Backpressure
   ↓
19 Dead-Letter / Data-Quality Lifecycle
   ↓
20+
Future Core Recipes
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

## Production Tools Rule

Recipes teach the underlying engineering mechanism first.

The **Production Tools You Should Know** section introduces a maximum of three relevant production tools per recipe. These tools provide vocabulary and recognition of how the same problem is handled in real systems.

They do not replace understanding the underlying mechanism.

## Definition of a Complete Recipe

A recipe is complete when you can independently:

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

The goal is to develop the ability to recognize and solve recurring Data Engineering problems independently.
