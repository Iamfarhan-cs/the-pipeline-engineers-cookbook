# Part III — Core Recipes

Part III is the implementation section of the book.

The earlier chapters explained how pipelines work and how to investigate an existing repository. Here we start building pipelines.

These recipes are intentionally practical. Each recipe starts with a small working system and then adds one engineering capability at a time.

## Recipes

1. [Create an Ingestion Pipeline](17-create-an-ingestion-pipeline.md)
2. Create a Staging Layer
3. Validate Incoming Data
4. Add Idempotency
5. Add Deduplication
6. Add Processing Status
7. Add Error Handling
8. Add Retry Logic
9. Replay / Reprocessing
10. Quarantine Failed Data
11. Backfill Historical Data
12. Incremental Processing
13. Checkpointing

## This Section Will Keep Evolving

Part III is not intended to be frozen.

As new pipeline patterns are implemented and investigated, new recipes can be added here.

New recipes should follow the same engineering cycle:

```text
Understand
    ↓
Investigate
    ↓
Design
    ↓
Implement
    ↓
Test
    ↓
Verify
    ↓
Observe
    ↓
Recover
    ↓
Improve
```

Later recipes can build on earlier ones instead of repeating the same implementation from scratch.

For example:

```text
Recipe 17
Basic ingestion
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
...
```

The goal is to grow a practical recipe library that can be reused when building real Data Engineering systems.