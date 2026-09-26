# E22 — API Checkpointing

## 1. Problem Recognition

An API extraction job can process thousands or millions of records. If the process crashes after successfully processing most of them, restarting from the beginning wastes time and can create duplicates.

Checkpointing records **durable extraction progress** so the next run can resume from a known safe boundary.

Recognize this problem when:

- API jobs run for a long time
- pagination contains many pages
- the process can restart unexpectedly
- workers can be interrupted during deployment
- API quotas make restarting expensive
- duplicate extraction is costly
- the source changes while extraction is running
- operators need to resume a failed run safely

The critical question is:

> What progress has definitely been persisted, and what progress is only believed to have happened?

## 2. Concept and Reasoning

Checkpointing is a durable record of pipeline progress.

```text
API
 ↓
REQUEST PAGE
 ↓
VALIDATE
 ↓
PERSIST DATA
 ↓
VERIFY PERSISTENCE
 ↓
COMMIT CHECKPOINT
 ↓
REQUEST NEXT PAGE
```

Never checkpoint progress before the represented data is durable.

### The Critical Ordering Rule

```text
WRONG
checkpoint page 25
      ↓
save page 25
      ↓
crash

RESULT: checkpoint says page 25 is complete, but page 25 may not exist.
```

Correct:

```text
save page 25
      ↓
commit
      ↓
checkpoint page 25
      ↓
commit
```

Depending on the storage architecture, the data and checkpoint should ideally commit atomically.

## 3. What a Checkpoint Contains

A useful checkpoint normally identifies:

| Field | Purpose |
|---|---|
| pipeline_id | Identifies the extraction pipeline |
| run_id | Identifies the execution |
| source | Identifies the API/source |
| resource | Identifies the resource being extracted |
| pagination_state | Cursor, page, or continuation token |
| watermark | Incremental extraction boundary, if applicable |
| last_successful_at | Last confirmed progress time |
| records_processed | Operational progress information |
| status | Current checkpoint state |

Example:

```json
{
  "pipeline_id": "customer-api",
  "resource": "customers",
  "cursor": "eyJwYWdlIjoyNX0=",
  "records_processed": 5000,
  "last_successful_at": "2026-09-26T12:00:00Z"
}
```

Do not assume `records_processed` alone is enough to resume. The checkpoint must contain a source-specific position.

## 4. Checkpoint Granularity

Checkpoint frequency is a trade-off.

### Page-level checkpointing

```text
PAGE 1 → checkpoint
PAGE 2 → checkpoint
PAGE 3 → checkpoint
```

Good for APIs with explicit pagination boundaries.

### Record-level checkpointing

```text
record 1 → checkpoint
record 2 → checkpoint
record 3 → checkpoint
```

This provides fine-grained recovery but can create excessive checkpoint writes.

### Batch-level checkpointing

```text
1000 records
     ↓
persist
     ↓
checkpoint
```

Often useful when records are processed in batches.

Choose the smallest boundary that provides useful recovery without making checkpoint storage the bottleneck.

## 5. Page-Based Checkpointing

For page-number APIs:

```python
def next_page(current_page: int) -> int:
    return current_page + 1
```

A checkpoint can store:

```python
checkpoint = {
    "page": 25,
    "status": "complete",
}
```

On restart:

```python
start_page = checkpoint["page"] + 1
```

But this is safe only if page 25 was durably persisted before the checkpoint was committed.

## 6. Cursor Checkpointing

Cursor-based APIs require the cursor itself to be persisted.

```python
checkpoint = {
    "cursor": next_cursor,
    "status": "complete",
}
```

Important:

> A cursor is not automatically a universal resume position. Its validity depends on the provider's pagination contract.

Cursors can expire, become invalid, or depend on the state of the source query.

## 7. Link-Based Checkpointing

Some APIs return a next-page URL.

```json
{
  "data": ["..."],
  "next": "https://api.example.com/items?page=26"
}
```

Persist the provider's continuation information rather than reconstructing it from assumptions.

```python
checkpoint = {
    "next_url": response["next"],
    "status": "complete",
}
```

## 8. Checkpoint Storage

A simple PostgreSQL table:

```sql
CREATE TABLE api_checkpoints (
    pipeline_id TEXT NOT NULL,
    resource TEXT NOT NULL,
    cursor TEXT,
    page_number BIGINT,
    watermark TEXT,
    records_processed BIGINT NOT NULL DEFAULT 0,
    status TEXT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (pipeline_id, resource)
);
```

The primary key prevents multiple active checkpoints for the same logical extraction unless your design explicitly supports parallel partitions.

## 9. Atomic Data + Checkpoint Commit

The strongest pattern is to persist the data and checkpoint in the same transaction when both use the same transactional database.

```python
def persist_batch_and_checkpoint(conn, rows, checkpoint):
    with conn.transaction():
        insert_rows(conn, rows)
        save_checkpoint(conn, checkpoint)
```

Then the database provides one important guarantee:

```text
COMMIT
  ↓
data durable + checkpoint durable
```

If the transaction rolls back:

```text
data not committed
checkpoint not committed
```

This prevents the checkpoint from moving beyond durable data.

## 10. When Data and Checkpoints Use Different Systems

Sometimes data is stored in object storage while checkpoints are stored in PostgreSQL.

Then there is no simple cross-system transaction.

Use an explicit state machine:

```text
DISCOVER PAGE
     ↓
DOWNLOAD
     ↓
WRITE DURABLE ARTIFACT
     ↓
VERIFY ARTIFACT
     ↓
COMMIT CHECKPOINT
```

The checkpoint should refer to an immutable or verifiable artifact.

Example:

```text
checkpoint:
  source_position = cursor-25
  artifact = raw/customer-api/run-123/page-25.json
  checksum = abc123...
```

This creates recoverable evidence of what the checkpoint represents.

## 11. Checkpoint State Machine

Use explicit states instead of treating every checkpoint as complete.

```text
DISCOVERED
    ↓
FETCHED
    ↓
VALIDATED
    ↓
PERSISTED
    ↓
CHECKPOINTED
    ↓
COMPLETE
```

A failed page can remain:

```text
FETCHED
   ↓
VALIDATION_FAILED
```

or:

```text
VALIDATED
   ↓
PERSISTENCE_FAILED
```

This makes recovery easier because operators can distinguish where progress stopped.

## 12. Checkpoint Versioning

Checkpoint formats evolve.

Store a version:

```json
{
  "checkpoint_version": 1,
  "cursor": "abc123",
  "resource": "customers"
}
```

If the extraction implementation later changes its checkpoint structure, versioning prevents old state from being interpreted incorrectly.

## 13. Idempotency and Checkpointing

Checkpointing does not eliminate duplicates by itself.

Consider:

```text
PAGE 25 persisted
      ↓
CRASH BEFORE CHECKPOINT
      ↓
restart PAGE 25
      ↓
PAGE 25 persisted again
```

This is normal and can be safe if persistence is idempotent.

Therefore robust extraction normally combines:

```text
CHECKPOINTING
      +
IDEMPOTENT PERSISTENCE
      +
DETERMINISTIC SOURCE POSITION
```

The goal is not to guarantee that every page is attempted exactly once. The goal is to guarantee that recovery does not lose data or corrupt state.

## 14. Checkpoint + Incremental Watermark

Incremental extraction may use a timestamp watermark:

```text
last_watermark = 2026-09-25T00:00:00Z
```

A common safe pattern uses an overlap:

```text
previous watermark
       ↓
subtract overlap window
       ↓
extract
       ↓
deduplicate
       ↓
advance watermark
```

Do not advance the watermark to the current time simply because a request succeeded.

The new watermark should represent data that has actually been extracted and durably processed.

## 15. Mutable Source Records

Checkpointing becomes harder when records can change during extraction.

Example:

```text
12:00 → extract customer A
12:05 → customer A changes
12:10 → checkpoint advances
```

Depending on the source API's semantics, the changed record may be missed.

Possible mechanisms include:

- overlap windows
- source snapshots
- updated-at filters
- change-data capture
- reconciliation queries

Checkpoint design must match source consistency semantics.

## 16. Checkpoint Recovery Algorithm

```python
def extract_with_checkpoint(api, store):
    checkpoint = store.load_checkpoint()

    state = checkpoint or {
        "page": 1,
        "records_processed": 0,
    }

    while True:
        page = state["page"]
        response = api.fetch_page(page)
        rows = validate(response)

        if not rows and response.get("has_next") is False:
            store.mark_complete()
            break

        store.persist_rows(rows)

        state = {
            "page": page + 1,
            "records_processed": state["records_processed"] + len(rows),
        }
        store.save_checkpoint(state)

        if response.get("has_next") is False:
            store.mark_complete()
            break
```

The important sequence is:

```text
FETCH
 ↓
VALIDATE
 ↓
PERSIST
 ↓
CHECKPOINT
 ↓
ADVANCE
```

## 17. Testing

### Unit tests

Test:

- checkpoint creation
- checkpoint loading
- checkpoint updates
- checkpoint versioning
- page resume calculation
- cursor persistence
- duplicate checkpoint writes
- completed-state handling
- malformed checkpoint handling

Example:

```python
def test_resume_starts_after_completed_page():
    checkpoint = {"page": 25, "status": "complete"}
    assert checkpoint["page"] + 1 == 26
```

### Integration tests

Test a realistic sequence:

```text
page 1 → success
page 2 → success
page 3 → crash
restart
page 3 → success
page 4 → success
```

Verify:

- no data loss
- duplicate handling works
- checkpoint ends at page 4
- completed state is correct

### Atomicity test

Force a failure between data persistence and checkpoint update.

With a shared database transaction, verify that both changes roll back.

## 18. Intentional Failure

### Failure drill A — Crash after persistence

Simulate:

```text
persist page 10
     ↓
CRASH
     ↓
checkpoint still says page 9
```

Restart.

Expected:
- page 10 is processed again
- idempotent persistence prevents corruption
- checkpoint eventually advances

### Failure drill B — Crash before persistence

Simulate a crash after the API response but before data storage.

Expected:
- checkpoint remains unchanged
- page is retried
- no progress is falsely recorded

### Failure drill C — Corrupt checkpoint

Replace the checkpoint with invalid JSON or an invalid cursor.

Expected:
- pipeline refuses unsafe resume
- error is visible
- previous valid checkpoint or manual recovery path is available

### Failure drill D — Checkpoint ahead of data

Deliberately create:

```text
checkpoint = page 20
data = only through page 19
```

Expected:
- reconciliation detects the inconsistency
- pipeline does not silently continue
- checkpoint is repaired from durable evidence

### Failure drill E — Repeated restart

Kill the process repeatedly during extraction.

Expected:
- pipeline eventually completes
- no records are lost
- duplicate handling remains safe

## 19. Observability

Track checkpoint behavior explicitly.

| Metric / Signal | Purpose |
|---|---|
| checkpoint age | Detects stalled progress |
| checkpoint position | Shows extraction progress |
| records since checkpoint | Measures recovery exposure |
| checkpoint failures | Detects state persistence problems |
| resume count | Shows repeated restarts |
| replayed pages | Measures duplicate work |
| checkpoint lag | Measures distance from current source position |
| completed runs | Confirms successful extraction |

Useful logs:

```text
pipeline_id
run_id
resource
checkpoint_position
previous_position
records_processed
checkpoint_version
checkpoint_status
resume_reason
```

Alert when checkpoint progress stops while the extraction process appears healthy.

## 20. Recovery

When an extraction fails:

1. Load the last durable checkpoint.
2. Verify its source position.
3. Verify the data represented by the checkpoint exists.
4. Determine whether the source cursor is still valid.
5. Resume from the safe boundary.
6. Allow idempotent persistence to handle replayed work.
7. Monitor progress.
8. Reconcile source and destination after completion.

If the checkpoint is corrupted or cannot be trusted, do not guess a resume position. Recover from the last independently verified durable state.

## 21. Production Tools You Should Know

### PostgreSQL
Useful for transactional checkpoint state when extracted data is also stored in PostgreSQL.

### Redis
Can store fast-moving progress state, but durability and recovery semantics must be designed deliberately.

### Apache Airflow
Provides workflow-level task state and retry mechanisms. Understand its task state separately from application-level source checkpoints.

These tools do not remove the need to understand checkpoint semantics.

## 22. Production Runbook

### Before deployment

- Define the checkpoint key.
- Define the source position.
- Define checkpoint granularity.
- Define checkpoint durability.
- Define the data/checkpoint commit relationship.
- Define recovery behavior.
- Define idempotent persistence.
- Define checkpoint corruption handling.

### When a job restarts

1. Load checkpoint.
2. Validate checkpoint schema/version.
3. Validate source position.
4. Verify durable data boundary.
5. Resume safely.
6. Monitor replayed work.

### When a checkpoint is suspicious

- Stop automatic advancement.
- Compare checkpoint with persisted data.
- Inspect source position.
- Recover from the last trusted boundary.
- Reconcile after recovery.

### What not to do

- Do not checkpoint before persistence.
- Do not trust an unvalidated checkpoint.
- Do not assume a cursor is permanent.
- Do not treat record counts as a complete resume position.
- Do not delete checkpoints during an active recovery.
- Do not manually advance a checkpoint just to make a job green.

## 23. Common Mistakes

1. Updating the checkpoint before writing data.
2. Storing only a record count.
3. Assuming page numbers are always stable.
4. Assuming cursors never expire.
5. Failing to make persistence idempotent.
6. Ignoring source consistency semantics.
7. Keeping checkpoints without version information.
8. Having no recovery path for corrupted state.
9. Using volatile process memory as the only checkpoint.
10. Treating a successful HTTP response as proof that downstream persistence succeeded.

## 24. Definition of Done

- [ ] A durable checkpoint exists.
- [ ] The checkpoint identifies the source position.
- [ ] Checkpoint ordering is safe.
- [ ] Data persistence and checkpointing are atomic where possible.
- [ ] Cross-system durability has an explicit protocol.
- [ ] Checkpoint state is versioned.
- [ ] Resume behavior is deterministic.
- [ ] Persistence is idempotent.
- [ ] Incremental watermarks are handled safely.
- [ ] Checkpoint corruption is detectable.
- [ ] Restart tests pass.
- [ ] Intentional crash tests pass.
- [ ] Checkpoint observability exists.
- [ ] Recovery procedures are documented.
- [ ] Source-to-target reconciliation can verify final state.

## 25. What You Learned

After this recipe, you should be able to independently:

- design a durable API checkpoint
- select an appropriate checkpoint boundary
- checkpoint page, cursor, link, and watermark state
- persist data before advancing progress
- use transactions for atomic data + checkpoint updates
- reason about checkpoints across different storage systems
- combine checkpointing with idempotency
- recover safely after process crashes
- detect and repair invalid checkpoint state
- test repeated restart scenarios
- operate checkpointed API extraction in production

### Core Mental Model

```text
LOAD CHECKPOINT
      ↓
REQUEST FROM SAFE POSITION
      ↓
VALIDATE
      ↓
PERSIST DATA
      ↓
VERIFY / COMMIT
      ↓
COMMIT CHECKPOINT
      ↓
ADVANCE SOURCE POSITION
      ↓
REPEAT
```

> A checkpoint is a claim about durable progress. Never advance that claim beyond what the pipeline can prove has been safely persisted.