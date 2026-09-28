# Part III — Core Pipeline Recipes

> **ETL taxonomy:** See [ETL-ROADMAP.md](ETL-ROADMAP.md) for the complete Extract → Transform → Load recipe map and the cross-cutting production ETL taxonomy.

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
| [Recipe 1 — Create an Ingestion Pipeline](01-create-an-ingestion-pipeline.md) | Ingestion |
| [Recipe 2 — Create a Staging Layer](02-create-a-staging-layer.md) | Staging |
| [Recipe 3 — Validate Incoming Data](03-validate-incoming-data.md) | Boundary validation |
| [Recipe 4 — Add Error Handling](04-add-error-handling.md) | Error classification |
| [Recipe 5 — Add Processing Status](05-add-processing-status.md) | Processing state |

### Stage 2 — Make Execution Safe

| Recipe | Topic |
|---:|---|
| [Recipe 6 — Add Idempotency](06-add-idempotency.md) | Safe repeated execution |
| [Recipe 7 — Add Deduplication](07-add-deduplication.md) | Duplicate records |
| [Recipe 8 — Add Retry Logic](08-add-retry-logic.md) | Bounded retries |
| [Recipe 9 — Quarantine Failed Data](09-quarantine-failed-data.md) | Failure isolation |
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
| [Recipe 20 — Boundary Validation](20-boundary-validation.md) | Validate at trust boundaries |
| [Recipe 21 — Transactions & Atomicity](21-transactions-atomicity.md) | Atomic database operations |
| [Recipe 22 — Bulk Loading](22-bulk-loading.md) | High-volume loading |
| [Recipe 23 — Batch Size & Chunking](23-batch-size-and-chunking.md) | Workload chunking |
| [Recipe 24 — Partial Failure](24-partial-failure.md) | Failure isolation |
| [Recipe 25 — Rate Limiting](25-rate-limiting.md) | Dependency traffic control |
| [Recipe 26 — Network Failure](26-network-failure.md) | Network reliability and recovery |


### Stage 6 — File-Based ETL

| Recipe | Topic |
|---:|---|
| [E49 — File Discovery](E49-file-discovery.md) | Discover, identify, register, and safely hand off files |
| [E50 — File Naming Conventions](E50-file-naming-conventions.md) | Deterministic file identity and naming contracts |
| [E51 — File Arrival Detection](E51-file-arrival-detection.md) | Detect expected deliveries and classify on-time, late, or unexpected arrivals |
| [E52 — File Completeness Detection](E52-file-completeness-detection.md) | Prove all required members of a file delivery are present |
| [E53 — File Validation — ETL Application](E53-file-validation-etl-application.md) | Validate file integrity, structure, format, and extraction readiness |
| [E54 — CSV Extraction — ETL Application](E54-csv-extraction-etl-application.md) | Stream and extract CSV records safely into staging |
| [E55 — JSON Extraction — ETL Application](E55-json-extraction-etl-application.md) | Parse, validate, stream, and stage JSON documents safely |
| [E56 — JSONL Extraction](E56-jsonl-extraction.md) | Stream, validate, checkpoint, and stage newline-delimited JSON safely |
| [E57 — XML Extraction — ETL Application](E57-xml-extraction-etl-application.md) | Extract, validate, stream, and stage XML safely |
| [E58 — Parquet Extraction](E58-parquet-extraction.md) | Efficiently read, filter, project, validate, and stage Parquet datasets |
| [E59 — Avro Extraction](E59-avro-extraction.md) | Stream schema-governed binary records, handle schema evolution, and stage Avro safely |
| [E60 — Compressed File Extraction](E60-compressed-file-extraction.md) | Safely decompress, inspect, stream, validate, and stage compressed deliveries |
| [E61 — Large File Streaming](E61-large-file-streaming.md) | Process files larger than memory using bounded streaming, batching, backpressure, and restart-safe checkpoints |
| [E62 — Multi-File Extraction](E62-multi-file-extraction.md) | Group, process, reconcile, retry, and recover multi-file deliveries with durable file and delivery state |
| [E63 — Duplicate File Detection](E63-duplicate-file-detection.md) | Detect duplicate observations, duplicate content, replay, and replacements without losing valid corrections |
| [E64 — Missing File Detection](E64-missing-file-detection.md) | Determine when required files are truly missing using durable expectations, deadlines, calendars, dependencies, and idempotent incidents |
| [E65 — Late File Detection](E65-late-file-detection.md) | Measure delivery lateness, SLA breaches, timing thresholds, escalation, recovery, and late-file operational metrics |

### Stage 7 — Transform Recipes

| Recipe | Topic |
|---:|---|
| [T01 — Raw Data to Staging](T01-raw-data-to-staging.md) | Establish a traceable, idempotent staging boundary between raw evidence and downstream transformation |
| [T02 — Data Type Conversion](T02-data-type-conversion.md) | Convert source representations into explicit, validated, precision-safe target types |
| [T03 — Null Handling](T03-null-handling.md) | Define, normalize, validate, observe, and safely recover NULL and missing-value semantics |
| [T04 — Default Values](T04-default-values.md) | Apply explicit, validated, observable defaults without corrupting missing-value semantics |
| [T05 — String Normalization](T05-string-normalization.md) | Canonicalize text representations safely while preserving field semantics and Unicode |
| [T06 — Date and Time Transformation](T06-date-and-time-transformation.md) | Parse, normalize, validate, observe, and safely recover date/time values without changing their meaning |
| [T07 — Time Zone Conversion](T07-time-zone-conversion.md) | Convert local times and instants safely across named time zones, including DST edge cases and explicit ambiguity policies |
| [T08 — Numeric Transformation](T08-numeric-transformation.md) | Convert, validate, round, and persist numeric values without silently changing precision, scale, units, or business meaning |
| [T09 — Boolean Normalization](T09-boolean-normalization.md) | Normalize source boolean representations into explicit TRUE, FALSE, and NULL semantics without hiding invalid or unknown values |
| [T10 — Code/Status Mapping](T10-code-status-mapping.md) | Translate source codes and statuses into governed canonical values with explicit unknown, effective-date, and versioning policies |
| [T11 — Data Standardization](T11-data-standardization.md) | Align cross-source representations, units, categories, and reference data to governed canonical standards without losing provenance |
| [T12 — Data Cleansing](T12-data-cleansing.md) | Detect, correct, quarantine, or reject data-quality defects using explicit rules, evidence, accounting, and safe recovery |
| [T13 — Record Filtering](T13-record-filtering.md) | Select records for downstream purposes using explicit predicates, explainable dispositions, population accounting, and safe replay |
| [T14 — Record Enrichment](T14-record-enrichment.md) | Add trusted context using controlled joins, cardinality checks, temporal correctness, freshness, provenance, and safe recovery |
| [T15 — Record Splitting](T15-record-splitting.md) | Expand one record into accountable child records with deterministic identity, lineage, cardinality, reconciliation, and replay safety |
| [T16 — Record Merging](T16-record-merging.md) | Consolidate records into one logical output using deterministic identity, field-level survivorship, conflict handling, provenance, and replay safety |
| [T17 — SQL SELECT Transformations](T17-sql-select-transformations.md) | Build controlled SQL projections with explicit columns, derived expressions, types, NULL semantics, units, and preserved grain |
| [T18 — WHERE Filtering](T18-where-filtering.md) | Select rows using explicit predicates, NULL semantics, time boundaries, security scope, population accounting, and safe replay |
| [T19 — GROUP BY Aggregation](T19-group-by-aggregation.md) | Reduce detailed records into reconciled summary grains with explicit grouping keys, aggregate semantics, NULL handling, and double-counting protection |
| [T20 — JOIN Transformations](T20-join-transformations.md) | Combine relational datasets safely with explicit grain, keys, cardinality, matching semantics, multiplication detection, temporal correctness, and reconciliation |
| [T21 — INNER JOIN](T21-inner-join.md) | Retain only valid matches while explicitly accounting for exclusions, cardinality, NULLs, duplicate references, and population loss |
| [T22 — LEFT JOIN](T22-left-join.md) | Preserve the left population while controlling unmatched semantics, right-side cardinality, NULLs, filter placement, and fan-out |
| [T23 — FULL OUTER JOIN](T23-full-outer-join.md) | Reconcile two populations by preserving matched, left-only, and right-only records with explicit grain, identity, comparison, and difference accounting |
| [T24 — Anti-Joins](T24-anti-joins.md) | Find left records with no qualifying relationship using explicit existence semantics, NULL-safe predicates, composite keys, temporal rules, and reconciliation |
| [T25 — Semi-Joins](T25-semi-joins.md) | Keep left records with at least one qualifying relationship using EXISTS, duplicate-safe membership logic, complete keys, temporal rules, and population reconciliation |
| [T26 — Window Functions](T26-window-functions.md) | Calculate row-aware analytics without collapsing grain using partitions, deterministic ordering, frames, ranking, running metrics, and late-data handling |
| [T27 — Ranking](T27-ranking.md) | Rank records globally or within partitions with explicit metrics, tie policies, deterministic ordering, NULL handling, top-N semantics, and snapshot-aware recomputation |
| [T28 — Running Totals](T28-running-totals.md) | Calculate cumulative state over deterministic event sequences with partitioning, opening balances, resets, late-data handling, and independent reconciliation |
| [T29 — Moving Windows](T29-moving-windows.md) | Calculate bounded rolling metrics with explicit row/time semantics, boundaries, sparse-data handling, timezone policy, late-data replay, and coverage monitoring |
| [T30 — Deduplication with Window Functions](T30-deduplication-with-window-functions.md) | Deduplicate logical records with deterministic survivor selection, composite identities, conflict classification, auditability, and target-grain enforcement |
| [T31 — Pivoting](T31-pivoting.md) | Transform categorical row values into stable analytical columns with explicit grain, aggregation, NULL semantics, category contracts, and reconciliation |
| [T32 — Unpivoting](T32-unpivoting.md) | Transform wide columns into governed long-form records with explicit mappings, target grain, NULL semantics, expansion controls, lineage, and reconciliation |
| [T33 — Required-Field Validation](T33-required-field-validation.md) | Enforce required-field contracts with explicit presence semantics, conditional rules, field-level diagnostics, quarantine, replay, and quality monitoring |
| [T34 — Type Validation](T34-type-validation.md) | Validate source representations against typed contracts with safe parsing, precision and range controls, timezone rules, schema-drift detection, and quarantine |
| [T35 — Range Validation](T35-range-validation.md) | Validate typed values against explicit numeric, temporal, cross-field, and reference-driven bounds with boundary testing, quarantine, and rule monitoring |
| [T36 — Domain Validation](T36-domain-validation.md) | Validate typed values against governed business domains, canonical mappings, effective-dated codes, scoped combinations, unknown values, and domain evolution |
| [T37 — Referential Integrity](T37-referential-integrity.md) | Verify child-to-parent relationships across keys, tenant scope, parent state, effective time, late arrivals, mappings, and recovery workflows |
| [T38 — Uniqueness Validation](T38-uniqueness-validation.md) | Validate business-key uniqueness at the correct grain, detect exact and conflicting duplicates, enforce deterministic survivor rules, and protect against replay and fan-out |
| [T39 — Completeness Checks](T39-completeness-checks.md) | Prove expected populations, partitions, keys, files, and time coverage are present at the correct grain with explicit deadlines, tolerances, evidence, and recovery |
| [T40 — Consistency Checks](T40-consistency-checks.md) | Validate relationships among fields, records, aggregates, states, units, currencies, and independent sources using explicit invariants and tolerances |
| [T41 — Statistical Anomaly Detection](T41-statistical-anomaly-detection.md) | Detect statistically unusual volume, distributions, NULL rates, cardinality, category shares, and time-series behavior using contextual baselines |
| [T42 — Data Quality Scoring](T42-data-quality-scoring.md) | Combine normalized quality dimensions into an auditable score with coverage, critical gates, policy versions, and component-level explanations |
| [T43 — Invalid Record Handling](T43-invalid-record-handling.md) | Classify, quarantine, retry, correct, replay, and reconcile invalid records without silently dropping data |
| [T44 — Quarantine During Transformation](T44-quarantine-during-transformation.md) | Apply quarantine during transformation with explicit side outputs, reconciliation, atomic publication, and replay |
| [T45 — Slowly Changing Dimensions](T45-slowly-changing-dimensions.md) | Preserve governed dimension history with change detection, effective dating, current-row semantics, temporal correctness, and safe recovery |
| [T46 — SCD Type 1](T46-scd-type-1.md) | Apply current-state overwrite semantics with deterministic upserts, source authority, idempotency, and safe recovery |
| [T47 — SCD Type 2](T47-scd-type-2.md) | Preserve full dimension history with effective intervals, versioned surrogate keys, temporal joins, late-change handling, and safe recovery |

### Stage 5 — Orchestration & Pipeline Operations

| Recipe | Topic |
|---:|---|
| [Recipe 27 — Pipeline Orchestration & Scheduling](27-pipeline-orchestration-and-scheduling.md) | Workflow execution, dependencies and scheduling |
| [Recipe 28 — Pipeline Run Tracking](28-pipeline-run-tracking.md) | Durable execution identity, state and history |
| [Recipe 29 — Pipeline Logging](29-pipeline-logging.md) | Structured execution evidence and correlation |
| [Recipe 30 — Pipeline Metrics](30-pipeline-metrics.md) | Measurable pipeline health, performance and data signals |
| [Recipe 31 — Bottleneck Detection](31-bottleneck-detection.md) | Finding limiting pipeline constraints |
| [Recipe 32 — Concurrency & Parallel Processing](32-concurrency-and-parallel-processing.md) | Controlled parallel execution and capacity management |

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
20 Boundary Validation
   ↓
21 Transactions & Atomicity
   ↓
22 Bulk Loading
   ↓
23 Batch Size & Chunking
   ↓
24 Partial Failure
   ↓
25 Rate Limiting
   ↓
26 Network Failure
   ↓
27 Pipeline Orchestration & Scheduling
   ↓
28 Pipeline Run Tracking
   ↓
29 Pipeline Logging
   ↓
30 Pipeline Metrics
   ↓
31 Bottleneck Detection
   ↓
32 Concurrency & Parallel Processing
   ↓
33+
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

## ETL Learning Model

The Core Recipes are now organized conceptually into four ETL domains:

1. **Extract** — obtain data reliably from APIs, databases, files, queues, and SaaS systems.
2. **Transform** — clean, validate, join, aggregate, enrich, model, and optimize data.
3. **Load** — write data safely and efficiently into databases, warehouses, and storage systems.
4. **Cross-Cutting Production ETL** — apply reliability, concurrency, observability, security, recovery, and operational controls across all three stages.

The existing numbered recipes remain the implementation history and foundational reliability sequence. New ETL-specific recipes should follow the canonical taxonomy in `ETL-ROADMAP.md`.
