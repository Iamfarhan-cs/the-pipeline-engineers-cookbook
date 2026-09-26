# ETL Recipe Roadmap

This is the canonical taxonomy for the ETL portion of The Pipeline Engineer's Cookbook.

The goal is not to collect random tools or tutorials. Every recipe should teach a reusable Data Engineering mechanism through:

    PROBLEM
       ↓
    REASONING
       ↓
    IMPLEMENT
       ↓
    TEST
       ↓
    OBSERVE
       ↓
    BREAK
       ↓
    RECOVER
       ↓
    OPERATE

## ETL Architecture

    EXTRACT
       ↓
    TRANSFORM
       ↓
    LOAD
       ↓
    PRODUCTION ETL

The **Extract**, **Transform**, and **Load** groups teach the mechanics of moving and changing data.

The **Cross-Cutting Production** group teaches the reliability, performance, observability, security, and operational mechanisms that apply across ETL.

---

# Part I — EXTRACT RECIPES

## A. Extraction Fundamentals

| ID | Recipe |
|---|---|
| E01 | [Extract Data from REST APIs](E01-extract-data-from-rest-apis.md) |
| E02 | Extract Data from Paginated APIs |
| E03 | Extract Data from GraphQL APIs |
| E04 | Extract Data from Relational Databases |
| E05 | Extract Data from CSV Files |
| E06 | Extract Data from JSON Files |
| E07 | Extract Data from XML Files |
| E08 | Extract Data from Excel Files |
| E09 | Extract Data from Object Storage |
| E10 | Extract Data from SFTP/FTP |
| E11 | Extract Data from Webhooks |
| E12 | Extract Data from Message Queues |
| E13 | Extract Data from Kafka |
| E14 | Extract Data from Cloud Storage |
| E15 | Extract Data from SaaS Platforms |

## B. API Extraction

| ID | Recipe |
|---|---|
| E16 | API Authentication |
| E17 | API Pagination |
| E18 | API Rate Limits |
| E19 | API Retries |
| E20 | API Timeouts |
| E21 | API Backoff and Jitter |
| E22 | API Checkpointing |
| E23 | API Incremental Extraction |
| E24 | API Cursor-Based Extraction |
| E25 | API Offset-Based Extraction |
| E26 | API Token Refresh |
| E27 | API Response Validation |
| E28 | API Schema Changes |
| E29 | API Partial Failure |
| E30 | API Deduplication |
| E31 | API Extraction Resume |
| E32 | API Extraction Auditing |

## C. Database Extraction

| ID | Recipe |
|---|---|
| E33 | Full Database Extraction |
| E34 | Incremental Database Extraction |
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
| E48 | Source Schema Evolution |

## D. File Extraction

| ID | Recipe |
|---|---|
| E49 | File Discovery |
| E50 | File Naming Conventions |
| E51 | File Arrival Detection |
| E52 | File Completeness Detection |
| E53 | File Validation |
| E54 | CSV Extraction |
| E55 | JSON Extraction |
| E56 | JSONL Extraction |
| E57 | XML Extraction |
| E58 | Parquet Extraction |
| E59 | Avro Extraction |
| E60 | Compressed File Extraction |
| E61 | Large File Streaming |
| E62 | Multi-File Extraction |
| E63 | Duplicate File Detection |
| E64 | Missing File Detection |
| E65 | Late File Detection |

---

# Part II — TRANSFORM RECIPES

## A. Transformation Fundamentals

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

## B. Relational Transformations

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
| T30 | Deduplication with Window Functions |
| T31 | Pivoting |
| T32 | Unpivoting |

## C. Data Quality Transformations

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
| T44 | Quarantine During Transformation |

## D. Advanced Transformations

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
| T53 | Sessionization |
| T54 | Event Aggregation |
| T55 | Event-Time Transformation |
| T56 | Late-Event Handling |
| T57 | Stateful Transformation |
| T58 | Incremental Transformation |
| T59 | Change-Based Transformation |
| T60 | Data Enrichment from Reference Data |
| T61 | Lookup Transformations |
| T62 | Slowly Changing Reference Data |

## E. Transformation Performance

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

---

# Part III — LOAD RECIPES

## A. Loading Fundamentals

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
| L09 | Append Loading |
| L10 | Replace Loading |
| L11 | Upsert Loading |
| L12 | Merge Loading |

## B. Database Loading

| ID | Recipe |
|---|---|
| L13 | Batch Inserts |
| L14 | Bulk Inserts |
| L15 | Database COPY |
| L16 | Batch Size Selection |
| L17 | Chunked Loading |
| L18 | Transactional Loading |
| L19 | Atomic Loading |
| L20 | Upsert with ON CONFLICT |
| L21 | MERGE-Based Loading |
| L22 | Loading into Staging Tables |
| L23 | Staging-to-Target Loading |
| L24 | Temporary Tables |
| L25 | Load Ordering |
| L26 | Foreign-Key-Aware Loading |

## C. Warehouse Loading

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
| L38 | Warehouse Reconciliation |

## D. Load Reliability

| ID | Recipe |
|---|---|
| L39 | Idempotent Loading |
| L40 | Duplicate Prevention |
| L41 | Load Retry |
| L42 | Partial Load Failure |
| L43 | Failed Batch Recovery |
| L44 | Load Checkpointing |
| L45 | Load Reprocessing |
| L46 | Load Rollback |
| L47 | Load Validation |
| L48 | Source-to-Target Reconciliation |
| L49 | Exactly-Once-Like Loading |
| L50 | Ambiguous Load Outcomes |

## E. Load Performance

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

---

# Part IV — CROSS-CUTTING PRODUCTION ETL

These mechanisms apply to Extract, Transform, and Load.

| ID | Recipe |
|---|---|
| X01 | Pipeline Idempotency |
| X02 | Deduplication |
| X03 | Retry Design |
| X04 | Error Classification |
| X05 | Dead-Letter Queues |
| X06 | Quarantine |
| X07 | Replay |
| X08 | Reprocessing |
| X09 | Checkpointing |
| X10 | Incremental Processing |
| X11 | Backfills |
| X12 | Late Data |
| X13 | Missing Data |
| X14 | Schema Evolution |
| X15 | Data Contracts |
| X16 | Contract Testing |
| X17 | Transactions |
| X18 | Atomicity |
| X19 | Partial Failure |
| X20 | Rate Limiting |
| X21 | Backpressure |
| X22 | Concurrency |
| X23 | Pipeline Orchestration |
| X24 | Pipeline Run Tracking |
| X25 | Structured Logging |
| X26 | Pipeline Metrics |
| X27 | Bottleneck Detection |
| X28 | Data Lineage |
| X29 | Data Observability |
| X30 | Freshness Monitoring |
| X31 | SLA/SLO Management |
| X32 | Alerting |
| X33 | Pipeline Security |
| X34 | Secrets Management |
| X35 | Access Control |
| X36 | Auditability |
| X37 | Disaster Recovery |
| X38 | Pipeline CI/CD |
| X39 | Pipeline Deployment |
| X40 | Pipeline Rollback |

---

# Existing Recipes — Mapping

The existing sequential recipes remain valid learning material. They are now classified into the ETL taxonomy rather than discarded.

| Existing Recipe | Primary ETL Classification |
|---:|---|
| 1 — Create an Ingestion Pipeline | Extract foundation |
| 2 — Create a Staging Layer | Extract → Transform boundary |
| 3 — Validate Incoming Data | Extract / Transform quality |
| 4 — Add Error Handling | Cross-cutting |
| 5 — Add Processing Status | Cross-cutting |
| 6 — Add Idempotency | X01 |
| 7 — Add Deduplication | X02 |
| 8 — Add Retry Logic | X03 |
| 9 — Quarantine Failed Data | X06 |
| 10 — Replay / Reprocessing | X07 / X08 |
| 11 — Incremental Processing | X10 |
| 12 — Checkpointing | X09 |
| 13 — Backfill Historical Data | X11 |
| 14 — Handle Late Data | X12 |
| 15 — Handle Missing Data | X13 |
| 16 — Data Reconciliation | Load / X29 |
| 17 — Handle Schema Changes | X14 |
| 18 — Handle Backpressure | X21 |
| 19 — Dead-Letter / Data-Quality Lifecycle | X05 / data quality |
| 20 — Boundary Validation | Extract / Transform / Load boundaries |
| 21 — Transactions & Atomicity | X17 / X18 |
| 22 — Bulk Loading | Load |
| 23 — Batch Size & Chunking | Load / Transform performance |
| 24 — Partial Failure | X19 |
| 25 — Rate Limiting | X20 |
| 26 — Network Failure | Extract / Load reliability |
| 27 — Pipeline Orchestration & Scheduling | X23 |
| 28 — Pipeline Run Tracking | X24 |
| 29 — Pipeline Logging | X25 |
| 30 — Pipeline Metrics | X26 |
| 31 — Bottleneck Detection | X27 |
| 32 — Concurrency & Parallel Processing | X22 |

## Important Rule

Do **not** duplicate an existing recipe merely because the same concept appears in an ETL category.

For example:

- Existing Recipe 6 teaches general idempotency.
- A future API recipe may apply idempotency specifically to API extraction.
- A future load recipe may apply idempotency specifically to database loading.

The specialized recipe should teach the **ETL-specific application**, not repeat the entire general mechanism.

---

# Recommended Learning Order

The cookbook should now progress as:

    FOUNDATIONS
         ↓
    REPOSITORY INVESTIGATION
         ↓
    EXISTING CORE RELIABILITY RECIPES
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

The first three ETL implementation tracks are:

    EXTRACT
    E01 → E65

    TRANSFORM
    T01 → T75

    LOAD
    L01 → L60

Then:

    PRODUCTION ETL
    X01 → X40

This gives the cookbook a clear path from learning how data enters a system to transforming it, loading it, and operating the complete pipeline safely.

# Recipe Standard

Every new ETL recipe must answer:

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
