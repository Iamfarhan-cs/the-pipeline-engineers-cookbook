# E31 — API Extraction Resume

## 1. Problem Recognition

API extraction jobs can stop after processing only part of the source.

Common causes:

- worker crash
- deployment or restart
- network failure
- provider outage
- rate-limit exhaustion
- process timeout
- validation failure
- planned shutdown
- checkpoint corruption

Recognize this problem when restarting would either repeat a large amount of work or risk skipping data.

The production objective is:

> Resume API extraction from the last safe position without losing records, corrupting progress, or unnecessarily replaying the entire source.

## 2. Concept and Reasoning

Resume is a state-recovery problem.

\`\`\`text
EXTRACTION
    ↓
WORK PROGRESSES
    ↓
FAILURE
    ↓
LOAD LAST SAFE STATE
    ↓
RESUME
    ↓
VERIFY
\`\`\`

A resume point is not simply the last request that was sent.

It is the last source position for which the pipeline can prove that the represented data was safely persisted.

> A resume checkpoint is a claim about durable progress.

## 3. What Can Be Resumed?

| API mechanism | Resume state |
|---|---|
| page number | next page |
| offset | next offset |
| cursor | next cursor |
| link pagination | next URL/state |
| timestamp | high-watermark/window |
| ID range | last processed ID |
| continuation token | next token |

Treat provider-controlled cursors and tokens as opaque values.

## 4. Safe Resume Boundary

Correct sequence:

\`\`\`text
REQUEST
   ↓
VALIDATE
   ↓
PERSIST
   ↓
VERIFY / COMMIT
   ↓
CHECKPOINT
   ↓
NEXT REQUEST
\`\`\`

Unsafe sequence:

\`\`\`text
REQUEST
   ↓
CHECKPOINT
   ↓
PERSIST
\`\`\`

The second sequence can skip data after a crash.

## 5. Durable Resume State

Use durable storage for production resume state.

Example PostgreSQL table:

\`\`\`sql
CREATE TABLE extraction_checkpoints (
    pipeline_name TEXT NOT NULL,
    source_name TEXT NOT NULL,
    partition_key TEXT NOT NULL DEFAULT '',
    position JSONB NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (pipeline_name, source_name, partition_key)
);
\`\`\`

Do not keep production resume state only in process memory.

## 6. Checkpoint Contents

A useful checkpoint may contain:

\`\`\`json
{
  "pagination_type": "cursor",
  "cursor": "opaque-provider-value",
  "source_window_start": "2026-09-26T00:00:00Z",
  "source_window_end": "2026-09-26T01:00:00Z",
  "schema_version": "v2"
}
\`\`\`

Only store state needed to reproduce the next safe extraction position.

## 7. Page-Level Resume

For page-based APIs:

\`\`\`text
page 1 → persisted → checkpoint page 2
page 2 → persisted → checkpoint page 3
page 3 → failure
\`\`\`

Restart from page 3.

Earlier pages do not need to be downloaded again when the checkpoint is trustworthy.

## 8. Offset Resume

Suppose:

\`\`\`text
offset=0   → success
offset=100 → success
offset=200 → failure
\`\`\`

Resume from offset 200.

However, offset pagination is sensitive to source mutation. Stable ordering and bounded extraction windows remain necessary.

## 9. Cursor Resume

Store the last safely committed cursor.

\`\`\`text
cursor A
   ↓
persist page
   ↓
checkpoint cursor B
\`\`\`

On restart, request using cursor B.

Do not inspect, modify, or generate provider cursors unless the API contract explicitly requires it.

## 10. Cursor Expiration

A stored cursor may expire.

\`\`\`text
LOAD CURSOR
    ↓
CURSOR VALID?
 /          \
YES          NO
 ↓            ↓
RESUME     SAFE FALLBACK
\`\`\`

The fallback may be a timestamp overlap, source snapshot, or fresh extraction window depending on the provider.

Do not silently skip forward when a cursor becomes invalid.

## 11. Resume with Incremental Extraction

Incremental extraction often needs both a watermark and a recovery overlap.

Example:

\`\`\`text
last_safe_watermark = 10:00
overlap = 5 minutes
resume window = 09:55 → bounded end
\`\`\`

Re-read the overlap and deduplicate using stable identity.

This is often safer than assuming timestamps provide exact recovery.

## 12. Checkpoint Granularity

Choose the smallest useful recovery unit.

\`\`\`text
RUN
 ├── PAGE 1
 ├── PAGE 2
 ├── PAGE 3
 └── PAGE 4
\`\`\`

Page-level checkpoints provide finer recovery than run-level checkpoints, but increase metadata writes and operational complexity.

## 13. Resume and Idempotency

Even correct checkpoints can leave ambiguous outcomes.

Example:

\`\`\`text
PERSIST DATA
   ↓
CRASH BEFORE CHECKPOINT
\`\`\`

On restart, the pipeline may process the same page again.

Therefore:

\`\`\`text
SAFE CHECKPOINTING + IDEMPOTENT PERSISTENCE
                         ↓
                  SAFE RESTART
\`\`\`

Do not design resume as if crashes occur only between requests.

## 14. Ambiguous Outcomes

An ambiguous outcome occurs when the pipeline cannot prove whether persistence completed.

For example, a database commit may complete while the client loses the response.

The safest recovery is normally to retry the source work and rely on idempotent destination semantics.

## 15. Resume After Worker Crash

Recovery sequence:

1. Load the last durable checkpoint.
2. Inspect destination state if needed.
3. Determine the source position represented by the checkpoint.
4. Resume from that position.
5. Reprocess ambiguous work safely.
6. Persist new data.
7. Advance the checkpoint only after persistence.

## 16. Resume After Deployment

Deployments should not require full re-extraction.

Before shutdown:

- stop accepting new work
- finish or safely abandon the current unit
- persist durable progress
- close connections

After deployment:

- load checkpoint
- validate checkpoint compatibility
- resume

## 17. Checkpoint Compatibility

A checkpoint can become invalid when the extraction implementation changes.

Example:

\`\`\`text
old parser → old pagination contract
new parser → new pagination contract
\`\`\`

Store enough metadata to determine whether a checkpoint belongs to the current extraction contract.

Useful fields:

\`\`\`text
pipeline_version
schema_version
api_version
checkpoint_version
\`\`\`

## 18. Checkpoint Versioning

Version checkpoint structures explicitly.

\`\`\`json
{
  "checkpoint_version": 2,
  "cursor": "abc",
  "schema_version": "v2"
}
\`\`\`

Do not silently reinterpret an old checkpoint using new semantics.

## 19. Checkpoint Corruption

Validate:

- required fields
- data types
- pagination mechanism
- source identity
- version compatibility
- valid state transitions

If the checkpoint is invalid, stop safely and recover from a known source position rather than guessing.

## 20. Multiple Extraction Partitions

A pipeline may have independent checkpoints:

\`\`\`text
pipeline
 ├── tenant A → cursor A
 ├── tenant B → cursor B
 └── tenant C → cursor C
\`\`\`

Do not let one partition overwrite another partition's state.

Use a durable partition key.

## 21. Parallel Resume

Independent partitions can resume independently.

\`\`\`text
worker A → tenant A checkpoint
worker B → tenant B checkpoint
worker C → tenant C checkpoint
\`\`\`

Shared checkpoints require concurrency control.

Never allow workers to overwrite progress with stale state.

## 22. Compare-and-Set Checkpoints

For concurrent workers, conditional updates can prevent stale checkpoint writes.

Conceptually:

\`\`\`sql
UPDATE extraction_checkpoints
SET position = %s, updated_at = now()
WHERE pipeline_name = %s
  AND source_name = %s
  AND position = %s;
\`\`\`

If zero rows are updated, another process may have advanced the state.

Handle this as a concurrency event rather than overwriting newer progress.

## 23. Resume and Pagination Safety

A resumed page is not necessarily equivalent to the original page.

Source mutation can cause:

- records to move between pages
- records to disappear
- records to appear again

Stable ordering, overlap windows, snapshots, or reconciliation may be required.

## 24. Resume and Rate Limits

A restarted pipeline can immediately generate a burst of requests.

Resume logic must still respect:

- provider rate limits
- local request pacing
- retry budgets
- concurrency limits

Restarting from a checkpoint does not justify ignoring API traffic policy.

## 25. Resume and Schema Changes

A restart may happen after the provider changes its schema.

Before continuing:

\`\`\`text
LOAD CHECKPOINT
      ↓
CHECK CONTRACT COMPATIBILITY
      ↓
COMPATIBLE?
 /          \
YES          NO
 ↓            ↓
RESUME      MIGRATE / STOP
\`\`\`

Do not resume old state through an incompatible parser without a deliberate migration strategy.

## 26. Resume and Backfills

Backfills should use isolated checkpoint namespaces.

Do not let a historical backfill overwrite the production incremental checkpoint.

\`\`\`text
production → checkpoint A
backfill-2025 → checkpoint B
\`\`\`

Each extraction mode needs its own state.

## 27. Resume vs Reprocessing

Resume means continue from the last safe position.

Reprocessing means intentionally run source work again.

They can work together:

\`\`\`text
CHECKPOINT
    ↓
RESUME
    ↓
DISCOVER AMBIGUOUS WORK
    ↓
REPROCESS SAFELY
    ↓
CONTINUE
\`\`\`

Do not confuse ordinary restart with deliberate historical replay.

## 28. Testing

### Unit Tests

Test:

- checkpoint creation
- checkpoint update
- checkpoint loading
- restart from page
- restart from offset
- restart from cursor
- expired cursor
- invalid checkpoint
- checkpoint version mismatch
- duplicate resume
- partition isolation
- stale concurrent checkpoint update

### Integration Tests

Simulate crashes at:

\`\`\`text
before request
after request
after validation
after persistence
before checkpoint
after checkpoint
\`\`\`

Verify that the final destination state remains correct.

## 29. Intentional Failure

### Failure Drill A — Crash Before Checkpoint

Persist a page and terminate before checkpointing.

Expected:

- restart reprocesses the page
- destination does not duplicate data
- checkpoint advances safely afterward

### Failure Drill B — Crash Before Persistence

Terminate before data persistence.

Expected:

- checkpoint does not move
- page is retried

### Failure Drill C — Corrupt Checkpoint

Replace the checkpoint with invalid state.

Expected:

- pipeline refuses unsafe resume
- recovery identifies a safe fallback

### Failure Drill D — Expired Cursor

Make the stored cursor invalid.

Expected:

- cursor failure is detected
- fallback extraction strategy is invoked
- data is not skipped silently

### Failure Drill E — Stale Worker Checkpoint

Have two workers attempt conflicting checkpoint updates.

Expected:

- stale state cannot overwrite newer progress
- conflict is observable

### Failure Drill F — Incompatible Checkpoint

Use a checkpoint generated by an incompatible parser or schema version.

Expected:

- pipeline refuses unsafe continuation
- migration or controlled restart is required

## 30. Observability

Track:

| Metric | Purpose |
|---|---|
| resume count | restart frequency |
| resumed source position | recovery visibility |
| checkpoint age | freshness of progress state |
| checkpoint write failures | state durability |
| checkpoint conflicts | concurrency issues |
| ambiguous replay count | crash/recovery behavior |
| resume duration | recovery performance |
| fallback resume count | cursor/checkpoint failures |

Useful logs:

\`\`\`text
run_id
pipeline_name
source_name
partition_key
checkpoint_version
previous_position
new_position
resume_reason
worker_id
\`\`\`

## 31. Recovery

When a pipeline fails:

1. Identify the last durable checkpoint.
2. Verify checkpoint integrity and compatibility.
3. Determine whether the checkpoint represents persisted data.
4. Identify ambiguous work after that point.
5. Resume from the safe position.
6. Let idempotent writes absorb repeated records.
7. Reconcile source and destination state.
8. Continue normal extraction.

If the checkpoint cannot be trusted, do not guess. Use a documented fallback such as a bounded overlap or controlled re-extraction.

## 32. Production Tools You Should Know

### PostgreSQL

Useful for durable checkpoint state, transactional updates, uniqueness, and conditional state changes.

### Redis

Useful for short-lived coordination or distributed locks when required; durable extraction progress should have an appropriate persistent store.

### Apache Airflow

Useful for orchestration-level task state, retries, and rerunning incomplete tasks.

These tools implement pieces of production recovery, but the core mechanism is durable source-position management.

## 33. Production Runbook

### When an extraction restarts

1. Identify the pipeline and partition.
2. Load the last safe checkpoint.
3. Validate checkpoint version and schema compatibility.
4. Confirm source pagination state is still valid.
5. Apply rate limits before resuming.
6. Resume from the safe position.
7. Watch for duplicate/replayed records.
8. Verify checkpoint advancement.
9. Reconcile after recovery.

### When the checkpoint is invalid

- stop automatic progression
- preserve the invalid checkpoint as evidence
- identify the last known safe source position
- choose a documented fallback
- reprocess idempotently
- create a new valid checkpoint

### What not to do

- Do not delete checkpoints blindly.
- Do not advance a checkpoint before persistence.
- Do not assume the last request was successful.
- Do not resume with an incompatible checkpoint.
- Do not ignore duplicate replay after crashes.
- Do not let one partition overwrite another partition's state.

## 34. Common Mistakes

1. Storing resume state only in memory.
2. Treating the last request as the last safe position.
3. Checkpointing before persistence.
4. Forgetting ambiguous outcomes.
5. Resuming an expired cursor without a fallback.
6. Reusing production checkpoints for backfills.
7. Allowing stale workers to overwrite newer checkpoints.
8. Ignoring API rate limits after restart.
9. Ignoring schema compatibility.
10. Replaying without idempotent persistence.
11. Guessing when checkpoint state is corrupt.
12. Failing to reconcile after recovery.

## 35. Definition of Done

- [ ] A durable resume state exists.
- [ ] The source position is explicitly defined.
- [ ] Checkpoint updates occur after safe persistence.
- [ ] Ambiguous outcomes are handled.
- [ ] Idempotent persistence protects replay.
- [ ] Checkpoint versions are defined.
- [ ] Checkpoint compatibility is validated.
- [ ] Partition-specific checkpoints are isolated.
- [ ] Concurrent checkpoint updates are protected.
- [ ] Cursor expiration has a documented fallback.
- [ ] Backfills use isolated state.
- [ ] Crash scenarios are tested.
- [ ] Checkpoint corruption is tested.
- [ ] Resume behavior is observable.
- [ ] Recovery and reconciliation are documented.

## 36. What You Learned

After this recipe, you should be able to independently:

- define a safe API resume point
- store durable extraction progress
- resume page, offset, cursor, and watermark extraction
- handle ambiguous persistence outcomes
- combine checkpoints with idempotency
- recover from worker and deployment failures
- detect invalid or incompatible checkpoints
- resume independent partitions safely
- protect concurrent checkpoint updates
- handle expired cursors
- isolate backfill checkpoints
- test crash and restart behavior
- reconcile after recovery
- operate API extraction resumes in production

### Core Mental Model

\`\`\`text
LOAD LAST SAFE CHECKPOINT
          ↓
VALIDATE CHECKPOINT
          ↓
REQUEST FROM SAFE POSITION
          ↓
VALIDATE RESPONSE
          ↓
PERSIST IDEMPOTENTLY
          ↓
VERIFY / COMMIT
          ↓
ADVANCE CHECKPOINT
          ↓
CONTINUE
          ↓
RECONCILE
\`\`\`

> Resume is not restarting the script. It is recovering from a durable statement about what the pipeline has already processed safely.
