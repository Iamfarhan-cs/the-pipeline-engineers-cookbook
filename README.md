# The Pipeline Engineer's Cookbook

## From Ingestion to Production

A practical Data Engineering implementation guide.

This repository contains the source for *The Pipeline Engineer's Cookbook* by Farhan Anjum.

The book is written as a practical guide for learning how to design, build, test, verify, operate, and recover reliable data pipelines.

## Book structure

The cookbook uses **one continuous recipe sequence**. Recipe numbers never restart when a new part begins.

### Part I — Foundations

Recipes 1–8 establish the mental model required to understand data pipelines.

1. How Data Pipelines Work
2. How Events Move Through a Pipeline
3. Batch vs Streaming
4. Raw, Staging, Curated and Warehouse Layers
5. Idempotency
6. Retries and Failures
7. Data Quality
8. Pipeline Observability

### Part II — Repository Investigation

Recipes 9–16 teach how to investigate an existing Data Engineering system before changing it.

9. How to Read a Data Engineering Repository
10. Finding the Entry Point
11. Finding Database Code
12. Finding Migrations
13. Finding Tests
14. Understanding Configuration
15. Understanding Docker
16. Understanding CI/CD

### Part III — Core Pipeline Recipes

Recipes 17–35 build production pipeline mechanisms in beginner-to-production order.

17. Create an Ingestion Pipeline
18. Create a Staging Layer
19. Validate Incoming Data
20. Add Error Handling
21. Add Processing Status
22. Add Idempotency
23. Add Deduplication
24. Add Retry Logic
25. Quarantine Failed Data
26. Replay / Reprocessing
27. Incremental Processing
28. Checkpointing
29. Backfill Historical Data
30. Handle Late Data
31. Handle Missing Data
32. Data Reconciliation
33. Handle Schema Changes
34. Handle Backpressure
35. Dead-Letter / Data-Quality Lifecycle

### Part IV — Data Storage

The next production recipes continue from **Recipe 36**.

36. PostgreSQL Pipeline
37. Raw Data Storage
38. Data Modeling
39. Fact Tables
40. Dimension Tables
41. Slowly Changing Dimensions
42. Indexing
43. Partitioning

### Part V — Streaming

44. Kafka Fundamentals
45. Producer
46. Consumer
47. Consumer Groups
48. Offsets
49. Replay
50. Dead-Letter Topics
51. Kafka → PostgreSQL
52. Kafka → Warehouse

### Part VI — Orchestration

53. Airflow Fundamentals
54. First DAG
55. Dependencies
56. Retries
57. Backfills
58. Scheduling
59. Sensors
60. Production DAG

### Part VII — Data Quality

61. Completeness
62. Uniqueness
63. Validity
64. Referential Integrity
65. Freshness
66. Anomaly Detection
67. Data Quality Framework

### Part VIII — Production

68. Logging
69. Metrics
70. Tracing
71. Alerting
72. Monitoring
73. Security
74. PII Handling
75. Schema Evolution
76. Data Contracts

### Part IX — Real Project Recipes

77. Payment Telemetry Pipeline
78. Sanctions Intelligence Pipeline
79. API → PostgreSQL
80. PostgreSQL → Warehouse
81. Event Replay System
82. End-to-End Production Pipeline

### Part X — Troubleshooting

83. Database Errors
84. Migration Errors
85. Duplicate Events
86. Failed Jobs
87. Missing Data
88. Late Data
89. Schema Mismatch
90. Pipeline Recovery

## Current Recipe Sequence

```text
Recipes 01–08
Foundations
        ↓
Recipes 09–16
Repository Investigation
        ↓
Recipes 17–35
Core Pipeline Engineering
        ↓
Recipe 36+
Advanced Production Data Engineering
```

## Core engineering cycle

**Understand → Investigate → Design → Implement → Test → Verify → Observe → Recover → Improve**

## Source of truth

The Markdown files in this repository are the master manuscript.

Publishing formats can be generated later from this source without making a publishing platform the primary location of the book.
