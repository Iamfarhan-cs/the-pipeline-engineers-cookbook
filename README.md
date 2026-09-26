# The Pipeline Engineer's Cookbook

## From Ingestion to Production

A practical Data Engineering implementation guide.

This repository contains the source for *The Pipeline Engineer's Cookbook* by Farhan Anjum.

The book is written as a practical guide for learning how to design, build, test, verify, operate, and recover reliable data pipelines.

## Book structure

The book keeps **core pipeline recipes independently numbered from the rest of the book**.

- Parts I and II use the book's chapter numbering.
- **Part III — Core Pipeline Recipes starts at Recipe 1 and continues onward.**
- Core recipe numbers do not restart or inherit the chapter numbers from Parts I and II.

### Part I — Foundations

Chapters 1–8 establish the mental model required to understand data pipelines.

1. How Data Pipelines Work
2. How Events Move Through a Pipeline
3. Batch vs Streaming
4. Raw, Staging, Curated and Warehouse Layers
5. Idempotency
6. Retries and Failures
7. Data Quality
8. Pipeline Observability

### Part II — Repository Investigation

Chapters 9–16 teach how to investigate an existing Data Engineering system before changing it.

9. How to Read a Data Engineering Repository
10. Finding the Entry Point
11. Finding Database Code
12. Finding Migrations
13. Finding Tests
14. Understanding Configuration
15. Understanding Docker
16. Understanding CI/CD

### Part III — Core Pipeline Recipes

**Independent recipe sequence: 1 onward.**

1. Create an Ingestion Pipeline
2. Create a Staging Layer
3. Validate Incoming Data
4. Add Error Handling
5. Add Processing Status
6. Add Idempotency
7. Add Deduplication
8. Add Retry Logic
9. Quarantine Failed Data
10. Replay / Reprocessing
11. Incremental Processing
12. Checkpointing
13. Backfill Historical Data
14. Handle Late Data
15. Handle Missing Data
16. Data Reconciliation
17. Handle Schema Changes
18. Handle Backpressure
19. Dead-Letter / Data-Quality Lifecycle

The Core Recipes section is intentionally expandable. The next core recipe will be **Recipe 20**, followed by 21, 22, and so on.

### Part IV — Data Storage

The rest of the book retains its own chapter numbering.

30. PostgreSQL Pipeline
31. Raw Data Storage
32. Data Modeling
33. Fact Tables
34. Dimension Tables
35. Slowly Changing Dimensions
36. Indexing
37. Partitioning

### Part V — Streaming

38. Kafka Fundamentals
39. Producer
40. Consumer
41. Consumer Groups
42. Offsets
43. Replay
44. Dead-Letter Topics
45. Kafka → PostgreSQL
46. Kafka → Warehouse

### Part VI — Orchestration

47. Airflow Fundamentals
48. First DAG
49. Dependencies
50. Retries
51. Backfills
52. Scheduling
53. Sensors
54. Production DAG

### Part VII — Data Quality

55. Completeness
56. Uniqueness
57. Validity
58. Referential Integrity
59. Freshness
60. Anomaly Detection
61. Data Quality Framework

### Part VIII — Production

62. Logging
63. Metrics
64. Tracing
65. Alerting
66. Monitoring
67. Security
68. PII Handling
69. Schema Evolution
70. Data Contracts

### Part IX — Real Project Recipes

71. Payment Telemetry Pipeline
72. Sanctions Intelligence Pipeline
73. API → PostgreSQL
74. PostgreSQL → Warehouse
75. Event Replay System
76. End-to-End Production Pipeline

### Part X — Troubleshooting

77. Database Errors
78. Migration Errors
79. Duplicate Events
80. Failed Jobs
81. Missing Data
82. Late Data
83. Schema Mismatch
84. Pipeline Recovery

## Numbering Model

```text
BOOK CHAPTERS
Part I  → Chapters 1–8
Part II → Chapters 9–16

        separate numbering boundary
                 ↓

CORE PIPELINE RECIPES
Part III → Recipe 1
           Recipe 2
           Recipe 3
           ...
           Recipe 19
           Recipe 20+
```

Core recipes are an evolving practical cookbook. Their numbering is independent from the surrounding book chapters.

## Core engineering cycle

**Understand → Investigate → Design → Implement → Test → Verify → Observe → Recover → Improve**

## Source of truth

The Markdown files in this repository are the master manuscript.

Publishing formats can be generated later from this source without making a publishing platform the primary location of the book.
