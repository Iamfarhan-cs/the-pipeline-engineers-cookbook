# E23 — API Incremental Extraction

## 1. Problem Recognition

Many APIs contain far more records than a pipeline needs to process on every run. A daily pipeline that downloads the entire dataset repeatedly wastes API quota, network bandwidth, compute, storage, and time.

Incremental extraction means:

> Extract only the source data that has changed or arrived since a previously established extraction boundary.

Recognize the problem when:

- the source dataset grows continuously
- full extraction is becoming slow or expensive
- the API exposes `updated_at`, `created_at`, IDs, cursors, or change tokens
- the provider documents incremental synchronization
- API quotas make full extraction impractical
- only new or changed records are needed
- the pipeline needs frequent synchronization

Typical architecture:

```text
PREVIOUS SAFE WATERMARK
        ↓
BUILD INCREMENTAL REQUEST
        ↓
API
        ↓
VALIDATE
        ↓
PERSIST
        ↓
DEDUPLICATE / MERGE
        ↓
ADVANCE WATERMARK
        ↓
NEXT RUN
```

## 2. Concept and Reasoning

Full extraction asks:

```text
Give me everything.
```

Incremental extraction asks:

```text
Give me what changed after my last safe extraction boundary.
```

The difficult part is not adding a filter such as:

```text
updated_at > last_run_time
```

The difficult part is defining a boundary that cannot silently lose records.

## 3. The Incremental Boundary

An incremental boundary is the source position that separates already processed data from data that still needs processing.

Common boundaries:

| Boundary | Example |
|---|---|
| Timestamp | `updated_at > 2026-09-25T00:00:00Z` |
| ID | `id > 500000` |
| Cursor | provider continuation token |
| Sequence | `sequence > 92831` |
| Change token | provider synchronization token |

Each has different correctness properties.

## 4. Timestamp-Based Incremental Extraction

A common API contract is:

```text
GET /customers?updated_after=2026-09-25T00:00:00Z
```

Store the last successfully processed timestamp.

```python
last_watermark = "2026-09-25T00:00:00Z"
```

Then request records after that point.

However, using the exact previous maximum timestamp can lose records when multiple records share the same timestamp or when source updates arrive late.

## 5. The Overlap Window

A safer pattern is to intentionally move the next extraction boundary backward.

```text
Previous watermark: 12:00
Overlap:            5 minutes

Next extraction starts at:
11:55
```

Then deduplicate or upsert the overlapping records.

```text
RUN 1
───────────────→ watermark 12:00

RUN 2
      ← overlap →
───────────────→ watermark 13:00
```

The overlap trades duplicate work for protection against boundary races and late visibility.

## 6. Why `>` vs `>=` Matters

Suppose the previous watermark is:

```text
2026-09-26T10:00:00Z
```

Two records have exactly that timestamp.

Using:

```sql
updated_at > :watermark
```

can exclude them.

Using:

```sql
updated_at >= :watermark
```

allows them to be re-read, after which idempotent persistence removes the duplicate effect.

In practice, a robust design often uses an overlap plus deduplication rather than relying on timestamp precision alone.

## 7. Watermark Precision

Do not assume timestamps have unlimited precision.

A source may store timestamps to:

- seconds
- milliseconds
- microseconds

If the pipeline stores less precision than the source, different source records can collapse onto the same boundary.

Normalize timestamps explicitly and preserve the source precision required by the extraction contract.

## 8. High-Watermark Extraction

A high-watermark algorithm tracks the greatest source position successfully processed.

```python
def update_high_watermark(current, observed_values):
    if not observed_values:
        return current
    return max(current, max(observed_values))
```

Important:

> Advance the high watermark only after all records represented by it are durably persisted.

## 9. Safe Watermark Advancement

Incorrect:

```text
REQUEST
  ↓
READ MAX(updated_at)
  ↓
ADVANCE WATERMARK
  ↓
PERSIST RECORDS
  ↓
CRASH
```

The watermark now claims progress that may not exist.

Correct:

```text
REQUEST
  ↓
VALIDATE
  ↓
PERSIST ALL RECORDS
  ↓
VERIFY / COMMIT
  ↓
ADVANCE WATERMARK
```

This is the same durable-progress principle used by checkpointing.

## 10. Choose the Next Boundary Before the Request

An incremental run should define a finite extraction window.

Example:

```text
lower_bound = previous_safe_watermark - overlap
upper_bound = current_run_start_time
```

Then:

```text
lower_bound ≤ record.updated_at < upper_bound
```

Why capture `upper_bound` before extraction?

Because records can continue changing while the run is executing.

Without a fixed upper boundary:

```text
extract begins
     ↓
new records arrive
     ↓
extract continues
     ↓
boundary keeps moving
     ↓
run may never represent a stable window
```

## 11. Python Incremental Extraction Example

```python
from datetime import datetime, timedelta, timezone


def build_window(previous_watermark: datetime, overlap_minutes: int):
    run_upper_bound = datetime.now(timezone.utc)
    lower_bound = previous_watermark - timedelta(minutes=overlap_minutes)
    return lower_bound, run_upper_bound


def build_params(lower_bound, upper_bound):
    return {
        "updated_after": lower_bound.isoformat(),
        "updated_before": upper_bound.isoformat(),
    }
```

Extraction:

```python
lower, upper = build_window(previous_watermark, overlap_minutes=5)

for page in api.iter_pages(
    updated_after=lower.isoformat(),
    updated_before=upper.isoformat(),
):
    rows = validate(page)
    persist_idempotently(rows)

save_watermark(upper)
```

The real implementation must save the watermark only after all pages in the window are successfully persisted.

## 12. Incremental Extraction with Pagination

Incremental extraction and pagination must work together.

```text
DEFINE WINDOW
    ↓
REQUEST PAGE 1
    ↓
PERSIST
    ↓
CHECKPOINT PAGE 1
    ↓
REQUEST PAGE 2
    ↓
PERSIST
    ↓
CHECKPOINT PAGE 2
    ↓
COMPLETE WINDOW
    ↓
ADVANCE WATERMARK
```

Do not advance the global watermark after page 1 merely because page 1 succeeded.

The watermark represents completion of the extraction window, not completion of an individual page.

## 13. Incremental Extraction with Mutable Records

Suppose customer 100 changes repeatedly:

```text
10:00 → status = pending
10:05 → status = approved
10:10 → status = suspended
```

If the API returns the latest representation only, the pipeline may not receive every historical state.

Incremental extraction is therefore usually a synchronization mechanism, not automatically a change-history mechanism.

If historical changes are required, use a source that exposes change events, audit history, CDC, or another historical mechanism.

## 14. Inserts, Updates, and Deletes

Incremental extraction must define what happens to each mutation type.

| Mutation | Possible API representation | Pipeline action |
|---|---|---|
| Insert | new record | insert |
| Update | changed record | update/upsert |
| Soft delete | deleted flag | update/delete according to contract |
| Hard delete | record disappears | requires delete detection/reconciliation |

A filter on `updated_at` cannot discover a hard deletion if the deleted record no longer appears in the API.

Deletion handling therefore needs a source-specific strategy.

## 15. ID-Based Incremental Extraction

If IDs are monotonically increasing, a pipeline may use:

```text
id > last_processed_id
```

This works well for append-only sources.

It is unsafe as a general replacement for timestamps when existing records can be updated.

Example:

```text
id=100 created
id=101 created

id=100 updated later
```

An ID-only extractor that has already passed 100 will not discover the update.

Choose the boundary according to the source's mutation model.

## 16. Cursor-Based Incremental Synchronization

Some APIs provide a synchronization cursor:

```json
{
  "data": [...],
  "next_cursor": "abc123"
}
```

The cursor can represent provider-specific change state that cannot safely be reconstructed from timestamps.

Use the provider's documented cursor semantics.

Persist the cursor only after the corresponding records are durable.

## 17. Change Tokens

Some APIs expose a token representing a synchronization point:

```text
initial token → abc
changes → records 1–500
next token → def
```

On the next run:

```text
token def
   ↓
request changes
   ↓
persist
   ↓
commit next token
```

Change tokens can be more precise than timestamp filtering because the provider controls the underlying change boundary.

## 18. Empty Incremental Windows

An incremental run may return zero records.

That does not automatically mean the pipeline should leave its watermark unchanged.

If the finite extraction window was successfully processed and the API contract guarantees completeness, the upper boundary can advance.

```text
window: 10:00 → 11:00
records: 0
result: successful empty window
watermark: 11:00
```

Distinguish:

- successful empty extraction
- failed extraction
- incomplete extraction

Never advance the watermark for an incomplete window merely because no records were returned.

## 19. Late Visibility

A record may have:

```text
updated_at = 10:00
```

but become visible through the API at 10:07.

If the pipeline already moved beyond 10:00, the record can be missed.

Possible protections:

- overlap windows
- source-side change tokens
- source snapshots
- delayed watermarks
- reconciliation

The correct choice depends on the source's consistency and visibility guarantees.

## 20. Incremental Extraction and Idempotent Loading

Overlap means duplicate source records are expected.

Therefore the destination needs deterministic identity.

```sql
CREATE UNIQUE INDEX uq_customer_source_id
ON customers(source_id);
```

Then:

```sql
INSERT INTO customers (source_id, name, updated_at)
VALUES (%s, %s, %s)
ON CONFLICT (source_id)
DO UPDATE SET
    name = EXCLUDED.name,
    updated_at = EXCLUDED.updated_at;
```

Incremental extraction becomes safer when replayed records have the same final effect.

## 21. Multi-Tenant Incremental Extraction

For SaaS APIs, each tenant can have independent progress.

Do not use one global watermark if tenant data has independent extraction state.

```text
tenant A → watermark A
tenant B → watermark B
tenant C → watermark C
```

A checkpoint key might be:

```text
(tenant_id, resource)
```

This allows one tenant to fail without incorrectly advancing another tenant's progress.

## 22. API Rate Limits

Incremental extraction usually reduces request volume, but it can still create spikes.

Use:

```text
incremental boundary
       ↓
pagination
       ↓
rate limiter
       ↓
retry + backoff
       ↓
checkpoint
```

Do not remove rate limiting just because the extraction is incremental.

## 23. Reconciliation

Incremental pipelines need periodic verification because incremental boundaries can fail silently if the source contract is misunderstood.

Example reconciliation:

```text
incremental state
      ↓
compare against source aggregate
      ↓
detect count / timestamp / checksum differences
      ↓
targeted repair or backfill
```

Periodic full or sampled reconciliation provides evidence that incremental extraction remains correct.

## 24. Testing

### Unit tests

Test:

- watermark calculation
- overlap calculation
- upper-bound capture
- timestamp precision
- empty windows
- duplicate timestamps
- ID-based boundaries
- cursor persistence
- delete handling
- tenant-specific state

Example:

```python
def test_overlap_moves_lower_boundary_back():
    from datetime import datetime, timedelta, timezone

    watermark = datetime(2026, 9, 26, 12, 0, tzinfo=timezone.utc)
    lower = watermark - timedelta(minutes=5)

    assert lower.hour == 11
    assert lower.minute == 55
```

### Integration tests

Simulate:

```text
RUN 1
records 1–100
watermark = T1

RUN 2
records 90–120
overlap = 10 records

destination
records 1–120
```

Verify that overlapping records do not create incorrect duplicates.

### Failure tests

Test:

```text
page 1 → persisted
page 2 → persisted
page 3 → failure
restart
page 3 → retry
complete window
advance watermark
```

Verify that the watermark does not move beyond page 3 until the window is complete.

## 25. Intentional Failure

### Failure drill A — Remove the overlap

Create a source record near the extraction boundary and make it visible slightly later.

Observe whether the record can be missed.

Restore the overlap strategy and verify recovery.

### Failure drill B — Advance watermark too early

Move the watermark immediately after the first successful page.

Fail a later page.

Expected diagnosis:

```text
watermark says complete
data is incomplete
```

Restore correct end-of-window advancement.

### Failure drill C — Duplicate timestamp

Create many records with the same timestamp.

Test the extraction with both `>` and `>=`.

Verify that the selected boundary strategy does not lose records.

### Failure drill D — Late visibility

Create a record whose source update time falls inside an already processed window but make it visible after that window completes.

Verify that overlap or reconciliation detects it.

### Failure drill E — Hard deletion

Delete a source record after the initial extraction.

Verify whether the API contract exposes the deletion.

If it does not, confirm that reconciliation or another deletion strategy is required.

## 26. Observability

Track incremental state explicitly.

| Metric / Signal | Purpose |
|---|---|
| current watermark | Shows extraction boundary |
| watermark age | Detects stale extraction |
| window duration | Measures run size |
| records extracted | Measures incremental volume |
| records replayed | Measures overlap cost |
| duplicate rate | Measures overlap/deduplication behavior |
| empty windows | Detects normal or suspicious zero-change periods |
| late records | Measures source visibility delay |
| extraction lag | Measures distance behind source |
| reconciliation failures | Detects silent correctness problems |

Useful structured logs:

```text
pipeline_id
run_id
resource
lower_bound
upper_bound
previous_watermark
new_watermark
records_extracted
records_upserted
records_deduplicated
pages_processed
```

Do not log sensitive record contents.

## 27. Recovery

When an incremental run fails:

1. Keep the previous safe watermark.
2. Identify the incomplete extraction window.
3. Resume from the last safe checkpoint.
4. Re-extract the affected window.
5. Allow idempotent loading to handle overlap.
6. Verify the entire window completed.
7. Advance the watermark.
8. Reconcile if the failure could have affected correctness.

Do not manually move the watermark forward merely to avoid repeated processing.

## 28. Production Tools You Should Know

### Airbyte
Provides connector-based incremental synchronization patterns and state management.

### Debezium
Useful when API-style incremental extraction is insufficient and the source exposes database changes through CDC.

### Singer
Uses tap/target concepts and supports stateful incremental extraction through connector state.

These tools provide implementations, but the underlying concepts remain watermarks, source positions, state, idempotency, and recovery.

## 29. Production Runbook

### Before deployment

- Identify the source's supported incremental mechanism.
- Document timestamp precision.
- Document mutation behavior.
- Determine whether deletions are visible.
- Define overlap policy.
- Define upper-bound semantics.
- Define watermark storage.
- Define checkpoint ordering.
- Define reconciliation.

### When extraction falls behind

1. Check watermark age.
2. Check incremental window size.
3. Check API rate limits.
4. Check retry volume.
5. Check source latency.
6. Check pagination volume.
7. Increase capacity only after identifying the bottleneck.

### When records appear missing

1. Inspect previous watermark.
2. Inspect overlap window.
3. Inspect source visibility delay.
4. Check timestamp precision.
5. Check pagination completeness.
6. Run reconciliation.
7. Re-extract the affected interval.

### What not to do

- Do not advance watermarks before durable persistence.
- Do not assume timestamps are unique.
- Do not assume IDs represent updates.
- Do not ignore deletions.
- Do not remove overlap solely to reduce duplicate work.
- Do not use one global watermark for independently progressing tenants.
- Do not treat an empty response as proof that the source has no data unless the API contract supports that conclusion.

## 30. Common Mistakes

1. Using `updated_at > watermark` without considering equal timestamps.
2. Not using an overlap window where source visibility requires one.
3. Advancing the watermark after only the first successful page.
4. Using an ID watermark for mutable records.
5. Ignoring hard deletions.
6. Failing to capture a finite upper bound.
7. Assuming API timestamps exactly represent visibility time.
8. Not making overlapping writes idempotent.
9. Sharing one watermark across independent tenants.
10. Never reconciling the incremental pipeline.

## 31. Definition of Done

- [ ] The source's incremental mechanism is documented.
- [ ] A durable watermark or source position exists.
- [ ] Lower and upper extraction boundaries are defined.
- [ ] Timestamp precision is understood.
- [ ] Overlap behavior is intentional.
- [ ] Pagination works inside the incremental window.
- [ ] Watermark advancement occurs only after successful persistence.
- [ ] Overlapping records are handled idempotently.
- [ ] Inserts and updates are handled.
- [ ] Deletion behavior is documented.
- [ ] Multi-tenant state is isolated where necessary.
- [ ] Rate limiting and retry behavior are integrated.
- [ ] Reconciliation exists.
- [ ] Restart tests pass.
- [ ] Late-visibility tests pass.
- [ ] Duplicate-boundary tests pass.
- [ ] Observability exposes incremental progress.
- [ ] Recovery procedures are documented.

## 32. What You Learned

After this recipe, you should be able to independently:

- explain why full extraction does not scale indefinitely
- choose an incremental source boundary
- implement timestamp-based incremental extraction
- use overlap windows safely
- reason about `>` versus `>=`
- use high-watermarks correctly
- combine incremental extraction with pagination
- handle mutable records and deletions
- distinguish IDs from true change positions
- use cursors and change tokens
- isolate state across tenants
- combine incremental extraction with idempotent loading
- reconcile incremental pipelines
- recover failed incremental windows without losing data

### Core Mental Model

```text
LOAD LAST SAFE POSITION
        ↓
DEFINE FINITE WINDOW
        ↓
EXTRACT CHANGES
        ↓
PAGINATE
        ↓
VALIDATE
        ↓
PERSIST IDEMPOTENTLY
        ↓
COMPLETE ENTIRE WINDOW
        ↓
ADVANCE WATERMARK
        ↓
RECONCILE
        ↓
NEXT RUN
```

> Incremental extraction is not simply filtering by a timestamp. It is a stateful synchronization process where the extraction boundary must advance only when the pipeline can prove that the corresponding source window has been safely processed.