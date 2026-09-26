# ETL Recipe Roadmap

This is the canonical taxonomy for the ETL portion of The Pipeline Engineer's Cookbook.

The roadmap separates **general pipeline mechanisms** from **ETL-specific applications**. Existing Core Recipes teach the mechanism once. ETL recipes apply that mechanism to specific extraction, transformation, or loading problems.

## Architecture

    FOUNDATIONS
         ↓
    CORE PIPELINE MECHANISMS
         ↓
    EXTRACT
         ↓
    TRANSFORM
         ↓
    LOAD
         ↓
    CROSS-CUTTING PRODUCTION ETL
         ↓
    END-TO-END PROJECTS

### The distinction

- **Core Recipes** teach reusable engineering mechanisms.
- **ETL Recipes** teach how those mechanisms are applied to specific ETL problems.
- **Cross-Cutting Recipes** cover production capabilities that are not already adequately taught by the Core Recipes.

Do not duplicate a mechanism just because it appears in another ETL category.

---

# Part I — EXTRACT RECIPES

Each recipe remains numbered by its stable E01–E65 identifier. The category determines **what kind of extraction problem it teaches**; the ID does not determine the category.

## Category 01 — API Extraction

| ID | Recipe |
|---|---|
| E01 | [Extract Data from REST APIs](E01-extract-data-from-rest-apis.md) |
| E02 | [Extract Data from Paginated APIs](E02-extract-data-from-paginated-apis.md) |
| E03 | [Extract Data from GraphQL APIs](E03-extract-data-from-graphql-apis.md) |
| E11 | Extract Data from Webhooks |
| E15 | Extract Data from SaaS Platforms |
| E16 | API Authentication — ETL Application |
| E17 | [API Pagination](E17-api-pagination.md) |
| E18 | [API Rate Limits](E18-api-rate-limits.md) |
| E19 | [API Retries](E19-api-retries.md) |
| E20 | [API Timeouts](E20-api-timeouts.md) |
| E21 | [API Backoff and Jitter](E21-api-backoff-and-jitter.md) |
| E22 | [API Checkpointing](E22-api-checkpointing.md) |
| E23 | [API Incremental Extraction](E23-api-incremental-extraction.md) |
| E24 | [API Cursor-Based Extraction](E24-api-cursor-based-extraction.md) |
| E25 | [API Offset-Based Extraction](E25-api-offset-based-extraction.md) |
| E26 | API Token Refresh |
| E27 | API Response Validation |
| E28 | API Schema Changes — ETL Application |
| E29 | API Partial Failure |
| E30 | API Deduplication — ETL Application |
| E31 | API Extraction Resume — ETL Application |
| E32 | API Extraction Auditing |

## Category 02 — Database Extraction

| ID | Recipe |
|---|---|
| E04 | [Extract Data from Relational Databases](E04-extract-data-from-relational-databases.md) |
| E33 | Full Database Extraction |
| E34 | Incremental Database Extraction — ETL Application |
| E35 | Watermark-Based Extraction |
| E36 | Timestamp-Based Extraction |
| E37 | ID-Based Extraction |
| E38 | High-Watermark Management |
| E39 | Database CDC |
| E40 | Transaction Log CDC |
| E41 | Snapshot Extraction |
| E42 | Consistent Database Snapshots |
| E43 | Database Connection Pooling |
| E44 | Database Extraction Batching |
| E45 | Database Extraction Parallelism |
| E46 | Extracting Large Tables Safely |
| E47 | Handling Source Database Load |
| E48 | Source Schema Evolution — ETL Application |

## Category 03 — File Extraction

| ID | Recipe |
|---|---|
| E05 | [Extract Data from CSV Files](E05-extract-data-from-csv-files.md) |
| E06 | [Extract Data from JSON Files](E06-extract-data-from-json-files.md) |
| E07 | [Extract Data from XML Files](E07-extract-data-from-xml-files.md) |
| E08 | [Extract Data from Excel Files](E08-extract-data-from-excel-files.md) |
| E49 | File Discovery |
| E50 | File Naming Conventions |
| E51 | File Arrival Detection |
| E52 | File Completeness Detection |
| E53 | File Validation — ETL Application |
| E54 | CSV Extraction — ETL Application |
| E55 | JSON Extraction — ETL Application |
| E56 | JSONL Extraction |
| E57 | XML Extraction — ETL Application |
| E58 | Parquet Extraction |
| E59 | Avro Extraction |
| E60 | Compressed File Extraction |
| E61 | Large File Streaming |
| E62 | Multi-File Extraction |
| E63 | Duplicate File Detection |
| E64 | Missing File Detection |
| E65 | Late File Detection |

## Category 04 — Object and Cloud Storage Extraction

| ID | Recipe |
|---|---|
| E09 | [Extract Data from Object Storage](E09-extract-data-from-object-storage.md) |
| E14 | Extract Data from Cloud Storage |

## Category 05 — Remote File Transfer Extraction

| ID | Recipe |
|---|---|
| E10 | Extract Data from SFTP/FTP |

## Category 06 — Messaging and Event Extraction

| ID | Recipe |
|---|---|
| E12 | Extract Data from Message Queues |
| E13 | Extract Data from Kafka |

> **ETL application rule:** API, database, file, storage, and messaging recipes should reference the relevant Core Recipe when a general mechanism already exists. The ETL recipe must teach the source-specific implementation, constraints, and failure modes instead of copying the Core Recipe.


# Part II — TRANSFORM RECIPES

Each transformation recipe belongs to a specific transformation category. General mechanisms already covered by Core Recipes should be applied, not duplicated.

## Category 01 — Transformation Foundations

| ID | Recipe |
|---|---|
| T01 | Raw Data to Staging |
| T02 | Data Type Conversion |
| T03 | Null Handling |
| T04 | Default Values |
| T05 | String Normalization |
| T06 | Date and Time Transformation |
| T07 | Time Zone Conversion |
| T08 | Numeric Transformation |
| T09 | Boolean Normalization |
| T10 | Code/Status Mapping |
| T11 | Data Standardization |
| T12 | Data Cleansing |
| T13 | Record Filtering |
| T14 | Record Enrichment |
| T15 | Record Splitting |
| T16 | Record Merging |

## Category 02 — SQL and Relational Transformations

| ID | Recipe |
|---|---|
| T17 | SQL SELECT Transformations |
| T18 | WHERE Filtering |
| T19 | GROUP BY Aggregation |
| T20 | JOIN Transformations |
| T21 | INNER JOIN |
| T22 | LEFT JOIN |
| T23 | FULL OUTER JOIN |
| T24 | Anti-Joins |
| T25 | Semi-Joins |
| T26 | Window Functions |
| T27 | Ranking |
| T28 | Running Totals |
| T29 | Moving Windows |
| T30 | Deduplication with Window Functions — Transform Application |
| T31 | Pivoting |
| T32 | Unpivoting |

## Category 03 — Data Quality Transformations

| ID | Recipe |
|---|---|
| T33 | Required-Field Validation |
| T34 | Type Validation |
| T35 | Range Validation |
| T36 | Domain Validation |
| T37 | Referential Integrity |
| T38 | Uniqueness Validation |
| T39 | Completeness Checks |
| T40 | Consistency Checks |
| T41 | Statistical Anomaly Detection |
| T42 | Data Quality Scoring |
| T43 | Invalid Record Handling |
| T44 | Quarantine During Transformation — Transform Application |

## Category 04 — Dimensional and Warehouse Transformations

| ID | Recipe |
|---|---|
| T45 | Slowly Changing Dimensions |
| T46 | SCD Type 1 |
| T47 | SCD Type 2 |
| T48 | Fact Table Transformation |
| T49 | Dimension Table Transformation |
| T50 | Surrogate Keys |
| T51 | Natural Keys |
| T52 | Business Keys |

## Category 05 — Event and Stateful Transformations

| ID | Recipe |
|---|---|
| T53 | Sessionization |
| T54 | Event Aggregation |
| T55 | Event-Time Transformation |
| T56 | Late-Event Handling — Transform Application |
| T57 | Stateful Transformation |
| T58 | Incremental Transformation — Transform Application |
| T59 | Change-Based Transformation |
| T60 | Data Enrichment from Reference Data |
| T61 | Lookup Transformations |
| T62 | Slowly Changing Reference Data |

## Category 06 — Transformation Performance and Scale

| ID | Recipe |
|---|---|
| T63 | Transformation Bottlenecks |
| T64 | Vectorized Processing |
| T65 | Batch Transformation |
| T66 | Parallel Transformation |
| T67 | Memory-Efficient Transformation |
| T68 | Streaming Transformation |
| T69 | Partition-Based Transformation |
| T70 | Data Skew |
| T71 | Join Optimization |
| T72 | Aggregation Optimization |
| T73 | Predicate Pushdown |
| T74 | Column Pruning |
| T75 | Query Optimization |


# Part III — LOAD RECIPES

Load recipes are grouped by destination and loading problem. General reliability mechanisms from the Core Recipes should be applied rather than re-taught from scratch.

## Category 01 — Destination and Format Loading

| ID | Recipe |
|---|---|
| L01 | Load Data into PostgreSQL |
| L02 | Load Data into MySQL |
| L03 | Load Data into a Data Warehouse |
| L04 | Load Data into Object Storage |
| L05 | Load CSV Data |
| L06 | Load JSON Data |
| L07 | Load Parquet Data |
| L08 | Load Partitioned Data |

## Category 02 — Loading Semantics

| ID | Recipe |
|---|---|
| L09 | Append Loading |
| L10 | Replace Loading |
| L11 | Upsert Loading |
| L12 | Merge Loading |

## Category 03 — Database Loading Mechanics

| ID | Recipe |
|---|---|
| L13 | Batch Inserts |
| L14 | Bulk Inserts |
| L15 | Database COPY |
| L16 | Batch Size Selection |
| L17 | Chunked Loading |
| L18 | Transactional Loading — Load Application |
| L19 | Atomic Loading — Load Application |
| L20 | Upsert with ON CONFLICT |
| L21 | MERGE-Based Loading |
| L22 | Loading into Staging Tables |
| L23 | Staging-to-Target Loading |
| L24 | Temporary Tables |
| L25 | Load Ordering |
| L26 | Foreign-Key-Aware Loading |

## Category 04 — Warehouse Loading

| ID | Recipe |
|---|---|
| L27 | Dimension Loading |
| L28 | Fact Loading |
| L29 | Incremental Warehouse Loading |
| L30 | Full Warehouse Refresh |
| L31 | Partition Loading |
| L32 | Partition Replacement |
| L33 | SCD Dimension Loading |
| L34 | Fact Incremental Loading |
| L35 | Late-Arriving Dimensions |
| L36 | Late-Arriving Facts |
| L37 | Warehouse MERGE |
| L38 | Warehouse Reconciliation — Load Application |

## Category 05 — Load Reliability and Recovery

| ID | Recipe |
|---|---|
| L39 | Idempotent Loading — Load Application |
| L40 | Duplicate Prevention — Load Application |
| L41 | Load Retry — Load Application |
| L42 | Partial Load Failure — Load Application |
| L43 | Failed Batch Recovery — Load Application |
| L44 | Load Checkpointing — Load Application |
| L45 | Load Reprocessing — Load Application |
| L46 | Load Rollback — Load Application |
| L47 | Load Validation — Load Application |
| L48 | Source-to-Target Reconciliation — Load Application |
| L49 | Exactly-Once-Like Loading |
| L50 | Ambiguous Load Outcomes |

## Category 06 — Load Performance and Scale

| ID | Recipe |
|---|---|
| L51 | Bulk Loading Optimization |
| L52 | Batch Size Optimization |
| L53 | Parallel Loading |
| L54 | Partition-Aware Loading |
| L55 | Index-Aware Loading |
| L56 | Loading into Large Tables |
| L57 | Small File Prevention |
| L58 | File Compaction |
| L59 | Write Amplification |
| L60 | Load Throughput Measurement |


# Part IV — CROSS-CUTTING PRODUCTION ETL

Cross-cutting recipes are reserved for production capabilities that are genuinely broader than a single ETL implementation and are not already adequately covered by the Core Recipes.

## A. Production Architecture and Operations

| ID | Recipe |
|---|---|
| X01 | Pipeline Run Tracking — Production Application |
| X02 | Structured Logging — Production Application |
| X03 | Pipeline Metrics — Production Application |
| X04 | Data Lineage |
| X05 | Data Observability |
| X06 | Freshness Monitoring |
| X07 | SLA/SLO Management |
| X08 | Alerting |
| X09 | Pipeline Security |
| X10 | Secrets Management |
| X11 | Access Control |
| X12 | Auditability |
| X13 | Disaster Recovery |

## B. Data Contracts and Evolution

| ID | Recipe |
|---|---|
| X14 | Schema Evolution — Cross-Pipeline Application |
| X15 | Data Contracts |
| X16 | Contract Testing |

## C. Delivery and Lifecycle

| ID | Recipe |
|---|---|
| X17 | Pipeline CI/CD |
| X18 | Pipeline Deployment |
| X19 | Pipeline Rollback |

## Intentionally Removed as Duplicates

The following old X recipes are no longer standalone roadmap slots because the general mechanism is already taught by the Core Recipes:

| Old Recipe | Covered By |
|---|---|
| X01 Pipeline Idempotency | Core Recipe 06 |
| X02 Deduplication | Core Recipe 07 |
| X03 Retry Design | Core Recipe 08 |
| X04 Error Classification | Core Recipe 04 |
| X05 Dead-Letter Queues | Core Recipe 19 |
| X06 Quarantine | Core Recipe 09 |
| X07 Replay | Core Recipe 10 |
| X08 Reprocessing | Core Recipe 10 |
| X09 Checkpointing | Core Recipe 12 |
| X10 Incremental Processing | Core Recipe 11 |
| X11 Backfills | Core Recipe 13 |
| X12 Late Data | Core Recipe 14 |
| X13 Missing Data | Core Recipe 15 |
| X17 Transactions | Core Recipe 21 |
| X18 Atomicity | Core Recipe 21 |
| X19 Partial Failure | Core Recipe 24 |
| X20 Rate Limiting | Core Recipe 25 |
| X21 Backpressure | Core Recipe 18 |
| X22 Concurrency | Core Recipe 32 |
| X23 Pipeline Orchestration | Core Recipe 27 |
| X24 Pipeline Run Tracking | Retained as X01 for production ETL application |
| X25 Structured Logging | Retained as X02 for production ETL application |
| X26 Pipeline Metrics | Retained as X03 for production ETL application |
| X27 Bottleneck Detection | Core Recipe 31 |

## Existing Core Recipes — Classification

The existing sequential recipes remain valid learning material. They are the foundational mechanisms and are not deleted or replaced by this roadmap.

| Existing Recipe | Classification |
|---:|---|
| 01 — Create an Ingestion Pipeline | Core mechanism / Extract foundation |
| 02 — Create a Staging Layer | Core mechanism / Extract → Transform boundary |
| 03 — Validate Incoming Data | Core mechanism / Data quality boundary |
| 04 — Add Error Handling | Core mechanism / Failure handling |
| 05 — Add Processing Status | Core mechanism / State management |
| 06 — Add Idempotency | Core mechanism / Idempotency |
| 07 — Add Deduplication | Core mechanism / Deduplication |
| 08 — Add Retry Logic | Core mechanism / Retry design |
| 09 — Quarantine Failed Data | Core mechanism / Quarantine |
| 10 — Replay / Reprocessing | Core mechanism / Replay and reprocessing |
| 11 — Incremental Processing | Core mechanism / Incremental processing |
| 12 — Checkpointing | Core mechanism / Checkpointing |
| 13 — Backfill Historical Data | Core mechanism / Backfills |
| 14 — Handle Late Data | Core mechanism / Late data |
| 15 — Handle Missing Data | Core mechanism / Missing data |
| 16 — Data Reconciliation | Core mechanism / Reconciliation |
| 17 — Handle Schema Changes | Core mechanism / Schema evolution |
| 18 — Handle Backpressure | Core mechanism / Backpressure |
| 19 — Dead-Letter / Data-Quality Lifecycle | Core mechanism / DLQ and quality lifecycle |
| 20 — Boundary Validation | Core mechanism / Boundary validation |
| 21 — Transactions & Atomicity | Core mechanism / Transactions and atomicity |
| 22 — Bulk Loading | Core mechanism / Bulk loading |
| 23 — Batch Size & Chunking | Core mechanism / Batching and chunking |
| 24 — Partial Failure | Core mechanism / Partial failure |
| 25 — Rate Limiting | Core mechanism / Rate limiting |
| 26 — Network Failure | Core mechanism / Network failure |
| 27 — Pipeline Orchestration & Scheduling | Core mechanism / Orchestration |
| 28 — Pipeline Run Tracking | Core mechanism / Run tracking |
| 29 — Pipeline Logging | Core mechanism / Logging |
| 30 — Pipeline Metrics | Core mechanism / Metrics |
| 31 — Bottleneck Detection | Core mechanism / Bottleneck analysis |
| 32 — Concurrency & Parallel Processing | Core mechanism / Concurrency |

## How Specialized Recipes Should Work

Example:

    Core Recipe 06 — Add Idempotency
                ↓
    E30 — API Deduplication / identity application
                ↓
    L39 — Idempotent Loading / database application

The Core Recipe teaches the general mechanism. The specialized ETL recipe teaches the source- or destination-specific implementation, constraints, and failure modes.

Another example:

    Core Recipe 12 — Checkpointing
                ↓
    E22 — API Checkpointing
                ↓
    E38 — High-Watermark Management
                ↓
    L44 — Load Checkpointing

Specialized recipes must not reproduce the entire Core Recipe. They should reference the prerequisite mechanism and spend their detail on ETL-specific behavior.

---

# Recommended Learning Order

    FOUNDATIONS
         ↓
    REPOSITORY INVESTIGATION
         ↓
    CORE PIPELINE MECHANISMS
         ↓
    EXTRACT
         ↓
    TRANSFORM
         ↓
    LOAD
         ↓
    CROSS-CUTTING PRODUCTION ETL
         ↓
    END-TO-END ETL PROJECTS

Within each ETL section, recipes should move from simple mechanics to failure-aware and performance-aware implementations.

---

# Recipe Standard

Every ETL recipe must answer:

1. What real production problem does this solve?
2. How do I recognize that problem?
3. What mechanism solves it?
4. Can I implement the mechanism myself?
5. Can I test normal and failure paths?
6. Can I observe it?
7. Can I intentionally break it?
8. Can I diagnose the failure?
9. Can I recover safely?
10. Which production tools implement or support this mechanism?
11. Can I operate it using a runbook?
12. Can I prove that I understand it without another tutorial?

The objective is **independent implementation ability**, not memorization.