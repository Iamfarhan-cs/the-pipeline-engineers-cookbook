# Part III — Core Recipes

Part III is the practical implementation section of the book.

The cookbook uses **one continuous recipe sequence**. Recipes 1–16 establish the foundations and repository-investigation skills required to work safely in an existing Data Engineering system. Recipes 17 onward build production pipeline mechanisms.

## Recipe Philosophy

These are not theory-only chapters. Each recipe is designed around a real production Data Engineering problem.

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

## Continuous Recipe Sequence

### Stage 1 — Foundations

| Recipe | Topic |
|---|---|
| [Recipe 1 — How Data Pipelines Work](../01-foundations/01-how-data-pipelines-work.md) | Pipeline fundamentals |
| [Recipe 2 — How Events Move Through a Pipeline](../01-foundations/02-how-events-move-through-a-pipeline.md) | Event flow |
| [Recipe 3 — Batch vs Streaming](../01-foundations/03-batch-vs-streaming.md) | Processing models |
| [Recipe 4 — Raw, Staging, Curated and Warehouse Layers](../01-foundations/04-raw-staging-curated-and-warehouse-layers.md) | Data layers |
| [Recipe 5 — Idempotency](../01-foundations/05-idempotency.md) | Safe repeated execution |
| [Recipe 6 — Retries and Failures](../01-foundations/06-retries-and-failures.md) | Failure handling |
| [Recipe 7 — Data Quality](../01-foundations/07-data-quality.md) | Data correctness |
| [Recipe 8 — Pipeline Observability](../01-foundations/08-pipeline-observability.md) | Operational visibility |

### Stage 2 — Repository Investigation

| Recipe | Topic |
|---|---|
| [Recipe 9 — How to Read a Data Engineering Repository](../02-repository-investigation/09-how-to-read-a-data-engineering-repository.md) | Repository structure |
| [Recipe 10 — Finding the Entry Point](../02-repository-investigation/10-finding-the-entry-point.md) | Execution path |
| [Recipe 11 — Finding Database Code](../02-repository-investigation/11-finding-database-code.md) | Database path |
| [Recipe 12 — Finding Migrations](../02-repository-investigation/12-finding-migrations.md) | Schema history |
| [Recipe 13 — Finding Tests](../02-repository-investigation/13-finding-tests.md) | Test discovery |
| [Recipe 14 — Understanding Configuration](../02-repository-investigation/14-understanding-configuration.md) | Runtime configuration |
| [Recipe 15 — Understanding Docker](../02-repository-investigation/15-understanding-docker.md) | Container environment |
| [Recipe 16 — Understanding CI/CD](../02-repository-investigation/16-understanding-ci-cd.md) | Delivery pipeline |

### Stage 3 — Build the Pipeline

| Recipe | Topic |
|---|---|
| [Recipe 17 — Create an Ingestion Pipeline](17-create-an-ingestion-pipeline.md) | Ingestion |
| [Recipe 18 — Create a Staging Layer](18-create-a-staging-layer.md) | Staging |
| [Recipe 19 — Validate Incoming Data](19-validate-incoming-data.md) | Boundary validation at ingestion |

### Stage 4 — Make the Pipeline Correct and Observable

| Recipe | Topic |
|---|---|
| [Recipe 20 — Add Error Handling](20-add-error-handling.md) | Error classification |
| [Recipe 21 — Add Processing Status](21-add-processing-status.md) | Processing state |
| [Recipe 22 — Add Idempotency](22-add-idempotency.md) | Safe repeated execution |
| [Recipe 23 — Add Deduplication](23-add-deduplication.md) | Duplicate records |

### Stage 5 — Recover from Failure

| Recipe | Topic |
|---|---|
| [Recipe 24 — Add Retry Logic](24-add-retry-logic.md) | Bounded retries |
| [Recipe 25 — Quarantine Failed Data](25-quarantine-failed-data.md) | Failure isolation |
| [Recipe 26 — Replay / Reprocessing](26-replay-reprocessing.md) | Controlled recovery |

### Stage 6 — Process Data Efficiently

| Recipe | Topic |
|---|---|
| [Recipe 27 — Incremental Processing](27-incremental-processing.md) | Incremental state |
| [Recipe 28 — Checkpointing](28-checkpointing.md) | Durable progress |
| [Recipe 29 — Backfill Historical Data](29-backfill-historical-data.md) | Historical recovery |

### Stage 7 — Handle Real-World Data Problems

| Recipe | Topic |
|---|---|
| [Recipe 30 — Handle Late Data](30-handle-late-data.md) | Event-time correctness |
| [Recipe 31 — Handle Missing Data](31-handle-missing-data.md) | Completeness |
| [Recipe 32 — Data Reconciliation](32-data-reconciliation.md) | Source-to-target correctness |

### Stage 8 — Evolve and Operate the Pipeline

| Recipe | Topic |
|---|---|
| [Recipe 33 — Handle Schema Changes](33-handle-schema-changes.md) | Schema evolution |
| [Recipe 34 — Handle Backpressure](34-handle-backpressure.md) | Flow control |
| [Recipe 35 — Dead-Letter / Data-Quality Lifecycle](35-dead-letter-dq-lifecycle.md) | Failure and DQ lifecycle |

## Complete Recipe Flow

```text
01 Foundations
   ↓
02 Events
   ↓
03 Batch vs Streaming
   ↓
04 Data Layers
   ↓
05 Idempotency
   ↓
06 Failures
   ↓
07 Data Quality
   ↓
08 Observability
   ↓
09 Repository Investigation
   ↓
10 Entry Point
   ↓
11 Database Code
   ↓
12 Migrations
   ↓
13 Tests
   ↓
14 Configuration
   ↓
15 Docker
   ↓
16 CI/CD
   ↓
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
   ↓
36+
Production Data Engineering
```

## Standard Recipe Structure

Every recipe should follow the same engineering structure:

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
