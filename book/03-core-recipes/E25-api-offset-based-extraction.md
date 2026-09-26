# E25 — API Offset-Based Extraction

## 1. Problem Recognition

Some APIs paginate with an offset and limit:

```text
GET /customers?offset=0&limit=100
GET /customers?offset=100&limit=100
GET /customers?offset=200&limit=100
```

This is easy to implement, but offsets can become unreliable when the source changes while extraction is running.

Recognize this pattern when an API exposes:
- `offset` + `limit`
- `start` + `count`
- page numbers that behave as offsets
- documented numeric record positions

The important question is:

> Can the same offset still represent the same logical records when the source changes?

## 2. Concept and Reasoning

An offset means “skip N records.” A cursor means “continue from this provider-defined position.”

```text
CHOOSE STABLE ORDER
       ↓
REQUEST OFFSET + LIMIT
       ↓
VALIDATE
       ↓
PERSIST
       ↓
CHECKPOINT
       ↓
ADVANCE OFFSET
       ↓
REPEAT
```

Offset pagination is appropriate when the API provides sufficiently stable ordering and consistency semantics.

## 3. Stable Ordering

Never rely on an undocumented default ordering.

Suppose page one returns:

```text
A B C D
```

Then a new record appears at the beginning:

```text
X A B C D
```

The next offset can now point at a different logical boundary, causing duplicates or skips.

Prefer an explicit deterministic sort, ideally with a unique tie-breaker:

```text
sort=created_at,id
```

Do not change sort or filter parameters while traversing one extraction.

## 4. Basic Implementation

```python
def extract_all(client, page_size=100):
    offset = 0

    while True:
        response = client.get(
            "/customers",
            params={"offset": offset, "limit": page_size},
        )
        rows = validate(response)

        if not rows:
            break

        persist_rows(rows)
        offset += len(rows)
```

Incrementing by actual returned rows is safer than blindly adding the requested page size when short pages are possible.

The exact termination rule must follow the provider contract.

## 5. Explicit Termination

APIs may expose:
- `has_more=false`
- `next=null`
- total count
- returned count
- empty page

Prefer an explicit provider signal:

```python
if response["has_more"] is False:
    break
```

Do not assume that a short page means completion unless documented.

An empty page is also not universally terminal.

## 6. Offset Checkpointing

A useful checkpoint stores the **next safe offset**:

```json
{
  "checkpoint_version": 1,
  "resource": "customers",
  "next_offset": 5000,
  "page_size": 100,
  "status": "active"
}
```

Correct ordering:

```text
REQUEST offset 5000
      ↓
VALIDATE
      ↓
PERSIST
      ↓
COMMIT
      ↓
CHECKPOINT next_offset=5100
```

Never checkpoint an offset before the represented records are durable.

## 7. Atomic Data + Offset State

When extracted data and checkpoint state share PostgreSQL:

```python
def persist_page(conn, rows, next_offset):
    with conn.transaction():
        insert_rows(conn, rows)
        save_checkpoint(conn, next_offset)
```

A successful commit makes both states durable. A rollback prevents the checkpoint from moving beyond committed data.

When data and checkpoint state live in different systems, use an explicit durable protocol and verify the artifact before advancing the checkpoint.

## 8. Offset + Idempotency

A crash can occur after data persistence but before checkpoint persistence:

```text
offset 5000 persisted
       ↓
CRASH
       ↓
restart offset 5000
       ↓
replay
```

This is normal. Persistence should therefore be idempotent:

```sql
INSERT INTO customers (source_id, name)
VALUES (%s, %s)
ON CONFLICT (source_id)
DO UPDATE SET name = EXCLUDED.name;
```

Checkpointing and idempotency solve different recovery problems.

## 9. The Mutation Problem

Offset pagination is vulnerable when records are inserted or deleted before the current offset.

Example:

```text
Initial: A B C D E F
page 1:  A B C

New record inserted at beginning:
X A B C D E F

offset=3 → C D E
```

Now C is duplicated and the logical boundary has shifted.

Deletes can create the opposite problem and cause records to be skipped.

This is the central weakness of naive offset extraction over mutable datasets.

## 10. Snapshot Consistency

The strongest solution is a source snapshot when the API supports one:

```text
CREATE SNAPSHOT
      ↓
OFFSET 0
      ↓
OFFSET 100
      ↓
OFFSET 200
      ↓
COMPLETE
```

All offsets then refer to the same logical dataset.

If snapshots are unavailable, use the provider's documented consistency mechanism, bounded windows, stable ordering, reconciliation, or another strategy appropriate to the source.

## 11. Fixed Extraction Windows

For mutable APIs, capture an upper boundary at run start:

```text
upper_bound = run_start_time
```

Then extract:

```text
updated_at < upper_bound
```

This prevents records arriving during the run from continually moving the dataset boundary.

A robust incremental pattern can be:

```text
previous watermark - overlap
          ↓
fixed upper bound
          ↓
offset pagination inside window
          ↓
complete window
          ↓
advance watermark
```

## 12. Offset + Incremental Extraction

Incremental filtering and offset pagination solve different problems.

```text
INCREMENTAL FILTER
       ↓
BOUNDED SOURCE WINDOW
       ↓
OFFSET PAGINATION
       ↓
PERSIST
       ↓
CHECKPOINT
```

Keep the extraction window fixed while offsets advance.

Do not recalculate the time boundary on every page.

## 13. Deletes

A normal `updated_at` filter cannot discover a hard deletion if the record disappears from the API.

Possible deletion mechanisms:
- soft-delete fields
- deletion endpoints
- audit APIs
- change feeds
- periodic reconciliation
- periodic full snapshots

Offset pagination does not solve deletion detection.

## 14. Large Offsets and Performance

Some APIs become slower at deep offsets because the underlying system may need to scan or skip many records.

Conceptually:

```text
offset=0        → small work
offset=100000   → more work
offset=1000000  → potentially expensive
```

Monitor latency. If deep offsets become a bottleneck, investigate cursor or keyset pagination.

Do not assume the provider's implementation, but do measure its behavior.

## 15. Page Size

Large pages reduce request count but increase:
- response size
- memory usage
- timeout exposure
- retry cost

Small pages increase:
- request count
- rate-limit pressure
- per-request overhead

Choose page size based on provider limits, response size, latency, processing cost, and quota.

## 16. Retries and Rate Limits

A failed request should retry the **same offset**:

```text
REQUEST offset 1000
      ↓
TIMEOUT
      ↓
BACKOFF + JITTER
      ↓
REQUEST offset 1000
```

Do not advance the offset until the page is successfully persisted.

Rate limiting remains necessary:

```text
OFFSET STATE
     ↓
RATE LIMITER
     ↓
REQUEST
     ↓
PERSIST
     ↓
ADVANCE OFFSET
```

## 17. Concurrency

Naively doing this:

```text
worker 1 → offset 0
worker 2 → offset 100
worker 3 → offset 200
```

is safe only when the source provides sufficiently stable semantics.

Safer parallelization requires things such as:
- a source snapshot
- independent tenants
- stable key ranges
- provider-supported partitions

Do not assume numeric offsets make mutable data safely parallelizable.

## 18. Production-Oriented Python Example

```python
def extract_offset_pages(client, store, page_size=100):
    offset = store.load_next_offset() or 0

    while True:
        response = client.get(
            "/customers",
            params={
                "offset": offset,
                "limit": page_size,
                "sort": "id",
            },
        )

        rows = validate(response)
        next_offset = offset + len(rows)

        persist_rows_idempotently(rows)
        store.save_next_offset(next_offset)

        if response.get("has_more") is False:
            store.mark_complete()
            break

        if not rows:
            raise RuntimeError(
                "empty page without explicit termination signal"
            )

        offset = next_offset
```

The exact termination condition must match the provider contract.

## 19. Testing

### Unit Tests

Test:
- initial offset
- offset advancement
- short pages
- explicit termination
- empty pages
- checkpoint loading
- checkpoint versioning
- retrying the same offset
- deterministic sorting

Example:

```python
def test_offset_advances_by_actual_rows():
    offset = 100
    rows = [1, 2, 3]
    assert offset + len(rows) == 103
```

### Integration Test

Use a fake API:

```text
offset 0 → [1,2,3]
offset 3 → [4,5,6]
offset 6 → [7]
has_more=false
```

Verify all records are persisted and the final checkpoint represents completion.

### Mutation Test

Insert a record between two requests and verify that the pipeline's consistency strategy detects or prevents duplicate/skipped records.

## 20. Intentional Failure

### Failure Drill A — Crash Before Checkpoint

Persist a page, then crash before checkpointing.

Expected:
- restart from the same offset
- replay occurs
- idempotent persistence prevents corruption

### Failure Drill B — Moving Source

Insert records before the current offset between requests.

Observe the duplicate/skipped-record risk. Repeat using a stable snapshot or bounded-window strategy.

### Failure Drill C — Wrong Termination

Return a short page with `has_more=true`.

Expected: extraction continues.

### Failure Drill D — Empty Page

Return an empty page while the provider indicates more data.

Expected: follow the documented continuation contract.

### Failure Drill E — Deep Offset

Test progressively larger offsets and observe latency.

## 21. Observability

Track:

| Signal | Purpose |
|---|---|
| current offset | Extraction position |
| pages processed | Progress |
| records extracted | Volume |
| page size | Request behavior |
| request latency | Deep-offset degradation |
| empty pages | Unusual source behavior |
| short pages | Source behavior |
| duplicate rate | Mutation problems |
| retry count | Dependency instability |
| extraction lag | Synchronization delay |

Useful logs:

```text
run_id
resource
offset
limit
rows_returned
has_more
attempt
request_latency_ms
next_offset
```

## 22. Recovery

When an offset extraction fails:

1. Load the last trusted next offset.
2. Verify persisted data around that boundary.
3. Retry the same offset.
4. Use idempotent persistence for replay.
5. Continue only after successful persistence.
6. Reconcile if source mutation could have affected correctness.

If the source changed substantially, restarting from the last offset may not be sufficient. Use a snapshot, incremental window, or reconciliation mechanism.

## 23. Production Tools You Should Know

### Requests
A simple synchronous HTTP client for implementing offset-based API extraction.

### HTTPX
Useful when explicit timeout, connection, or asynchronous HTTP behavior is required.

### Airbyte
Provides production connector patterns for paginated and incremental API extraction.

Understand the offset mechanics before delegating them to a connector.

## 24. Production Runbook

### Before deployment

- Confirm offset/limit semantics.
- Confirm maximum page size.
- Confirm ordering guarantees.
- Confirm termination semantics.
- Determine whether records can change during extraction.
- Determine whether snapshots are available.
- Define checkpoint meaning.
- Define idempotent persistence.
- Define reconciliation.

### If records appear duplicated

1. Check source mutations.
2. Check ordering.
3. Check offset advancement.
4. Check retry behavior.
5. Check overlapping windows.
6. Reconcile against the source.

### If records appear missing

1. Check moving-source behavior.
2. Check filters.
3. Check short-page handling.
4. Check termination logic.
5. Check checkpoint state.
6. Check late visibility or hard deletes.
7. Re-extract the affected range.

### What not to do

- Do not assume offsets are stable on mutable data.
- Do not checkpoint before persistence.
- Do not stop merely because a page is short.
- Do not parallelize offsets blindly.
- Do not ignore deep-offset performance.
- Do not assume offset pagination detects deletions.

## 25. Common Mistakes

1. Relying on default API ordering.
2. Ignoring source mutations.
3. Treating a short page as automatic completion.
4. Advancing offsets before persistence.
5. Failing to make replay idempotent.
6. Using very large offsets without monitoring latency.
7. Assuming offsets detect deletions.
8. Parallelizing mutable offset ranges without a snapshot.
9. Changing filters or sort parameters during extraction.
10. Never reconciling a mutable-source extraction.

## 26. Definition of Done

- [ ] Offset and limit semantics are documented.
- [ ] Stable ordering is explicitly defined.
- [ ] Termination behavior is documented.
- [ ] Short pages are handled correctly.
- [ ] Empty-page behavior follows the provider contract.
- [ ] Offset state is durably checkpointed.
- [ ] Data persistence precedes offset advancement.
- [ ] Replay is idempotent.
- [ ] Source mutation risks are understood.
- [ ] Snapshot or bounded-window strategy is selected where needed.
- [ ] Incremental extraction interaction is defined.
- [ ] Retry and rate-limit behavior is integrated.
- [ ] Parallelism is justified by source semantics.
- [ ] Deep-offset performance is monitored.
- [ ] Mutation failure drills pass.
- [ ] Recovery is documented.
- [ ] Reconciliation exists.

## 27. What You Learned

After this recipe, you should be able to independently:

- implement offset/limit extraction
- checkpoint offset progress safely
- choose correct page termination behavior
- reason about stable ordering
- understand why mutable sources break naive offsets
- combine offsets with incremental windows
- combine offsets with retries and rate limits
- reason about offset performance at scale
- identify when cursor or keyset pagination is more appropriate
- design safe crash recovery
- test moving-source behavior
- operate offset-based extraction in production

### Core Mental Model

```text
CHOOSE STABLE ORDER
       ↓
DEFINE EXTRACTION WINDOW
       ↓
LOAD SAFE OFFSET
       ↓
REQUEST OFFSET + LIMIT
       ↓
VALIDATE
       ↓
PERSIST IDEMPOTENTLY
       ↓
CHECKPOINT NEXT OFFSET
       ↓
CONTINUE UNTIL DOCUMENTED END
       ↓
RECONCILE IF SOURCE IS MUTABLE
```

> Offset pagination is a numeric position, not a guarantee of a stable dataset. Correct extraction depends on ordering, source consistency, checkpointing, idempotency, and an explicit termination contract.
