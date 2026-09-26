# E24 — API Cursor-Based Extraction

## 1. Problem Recognition

Some APIs do not expose reliable page numbers or offsets. Instead, the API returns a cursor or continuation token that identifies where the next request should continue.

Typical response:

```json
{
  "data": [
    {"id": 101},
    {"id": 102}
  ],
  "next_cursor": "eyJvZmZzZXQiOjEwMn0="
  }
```

The next request sends that cursor back to the API.

```text
REQUEST
   ↓
DATA + NEXT CURSOR
   ↓
PERSIST DATA
   ↓
CHECKPOINT CURSOR
   ↓
REQUEST WITH CURSOR
   ↓
REPEAT
```

Recognize this problem when:

- the API documents cursor-based pagination
- offsets become slow or unstable
- records can be inserted while extraction is running
- the API returns `next_cursor`, `continuation_token`, or similar state
- the provider explicitly says cursors are required for pagination

Cursor extraction is not just pagination syntax. The cursor is **source-controlled progress state**.

## 2. Concept and Reasoning

A cursor represents a position in a provider-defined result stream.

Unlike an offset:

```text
offset = "skip N records"
cursor = "continue from this provider-defined position"
```

This distinction matters when the source changes while extraction is running.

### Core Rule

> Persist the data represented by a cursor before advancing the checkpoint to the next cursor.

Correct sequence:

```text
REQUEST cursor A
       ↓
RECEIVE records + cursor B
       ↓
VALIDATE records
       ↓
PERSIST records
       ↓
VERIFY COMMIT
       ↓
CHECKPOINT cursor B
       ↓
REQUEST cursor B
```

## 3. Cursor Lifecycle

A typical lifecycle is:

```text
NO CURSOR
    ↓
FIRST REQUEST
    ↓
CURSOR A
    ↓
REQUEST WITH A
    ↓
CURSOR B
    ↓
REQUEST WITH B
    ↓
END
```

At every transition, the pipeline must know which cursor is:

- currently being processed
- safely persisted
- next to request

Confusing these states can cause skipped pages or repeated work.

## 4. Initial Extraction

The first request usually has no cursor.

```python
def fetch_first_page(client):
    return client.get("/customers")
```

Response:

```json
{
  "data": [
    {"id": 1},
    {"id": 2}
  ],
  "next_cursor": "abc123"
}
```

Persist the records first.

Then checkpoint:

```text
next_cursor = abc123
```

## 5. Subsequent Requests

```python
def fetch_page(client, cursor):
    return client.get(
        "/customers",
        params={"cursor": cursor},
    )
```

Do not reconstruct cursor values from record IDs unless the provider explicitly documents that behavior.

A cursor is usually opaque.

## 6. Treat Cursors as Opaque

A cursor might look like:

```text
eyJvZmZzZXQiOjEwMjQsInRzIjoiMjAyNi0wOS0yNiJ9
```

Do not assume it contains an offset or timestamp that your pipeline should decode.

Bad:

```python
offset = decode_cursor(cursor)["offset"]
```

Better:

```python
params = {"cursor": cursor}
```

Use the provider's documented cursor contract.

## 7. Cursor Checkpointing

A checkpoint can contain:

```json
{
  "checkpoint_version": 1,
  "resource": "customers",
  "cursor": "abc123",
  "status": "active"
}
```

However, define precisely what the stored cursor means.

Two common meanings are:

### A — Cursor to process next

```text
checkpoint.cursor = next unprocessed cursor
```

On restart, request that cursor.

### B — Cursor just processed

```text
checkpoint.cursor = cursor used by previous request
```

On restart, the implementation must derive the next cursor from additional state.

The first model is usually easier to reason about because the checkpoint directly represents the next safe source position.

## 8. Safe Cursor Algorithm

```python
def extract(client, store):
    cursor = store.load_next_cursor()

    while True:
        params = {} if cursor is None else {"cursor": cursor}
        response = client.get("/customers", params=params)

        rows = validate(response)
        next_cursor = response.get("next_cursor")

        persist_rows_idempotently(rows)

        store.save_cursor(next_cursor)

        if next_cursor is None:
            store.mark_complete()
            break

        cursor = next_cursor
```

The storage implementation should make data persistence and cursor checkpointing atomic whenever possible.

## 9. Atomic Data + Cursor State

If both records and checkpoint are stored in PostgreSQL:

```python
def persist_page(conn, rows, next_cursor):
    with conn.transaction():
        insert_rows(conn, rows)
        save_cursor(conn, next_cursor)
```

This gives:

```text
COMMIT
  ↓
rows durable + cursor durable
```

If the transaction fails:

```text
rows not committed
cursor not committed
```

This prevents the cursor from moving past data that was not safely persisted.

## 10. Cursor Expiration

Some providers issue cursors with a limited lifetime.

Example:

```text
cursor created: 10:00
cursor expires: 10:30
job resumes: 11:15
```

The cursor may no longer be valid.

Possible recovery strategies:

- restart from a supported synchronization boundary
- request a new cursor
- use a timestamp overlap
- use provider-specific resume functionality
- run reconciliation after recovery

Never assume a cursor is permanent.

## 11. Invalid Cursor Handling

An API may return:

```text
400 invalid cursor
410 cursor expired
401/403 authorization failure
429 rate limited
5xx provider failure
```

These failures have different meanings.

```text
INVALID CURSOR
      ↓
DO NOT BLINDLY RETRY FOREVER
      ↓
RECOVER USING SOURCE CONTRACT
```

A permanently invalid cursor should not enter an infinite retry loop.

## 12. Cursor Loop Detection

A broken API or client can accidentally return the same cursor repeatedly.

Example:

```text
cursor A
   ↓
cursor B
   ↓
cursor B
   ↓
cursor B
   ↓
cursor B
```

Detect repeated cursor state:

```python
seen_cursors = set()

while cursor is not None:
    if cursor in seen_cursors:
        raise RuntimeError("cursor loop detected")

    seen_cursors.add(cursor)
    response = fetch(cursor)
    cursor = response.get("next_cursor")
```

For long-running jobs, avoid unbounded memory growth by using a more targeted loop-detection strategy or bounded recent-history tracking when appropriate.

## 13. Empty Pages

An empty page does not necessarily mean extraction is complete.

Possible response:

```json
{
  "data": [],
  "next_cursor": "abc456"
}
```

The provider may still have more data.

Termination should be based on the documented pagination contract, not merely:

```python
if not rows:
    break
```

Correct:

```python
if next_cursor is None:
    break
```

unless the provider explicitly defines another termination condition.

## 14. Cursor + Deterministic Query Parameters

A cursor is often valid only for the same query context.

For example:

```text
resource = customers
filter = active
sort = updated_at
cursor = abc123
```

Changing the filter while reusing the cursor may produce invalid or incorrect results.

Store or reconstruct the extraction contract consistently.

```text
CURSOR
+
RESOURCE
+
FILTERS
+
SORT
=
CURSOR CONTEXT
```

## 15. Cursor + Stable Ordering

Some cursor systems depend on a stable sort order.

Example:

```text
sort = created_at ASC
```

Do not change the sort order halfway through extraction.

Otherwise the cursor may no longer represent the same logical result stream.

## 16. Cursor + Incremental Extraction

Cursor pagination can be combined with incremental filtering.

```text
updated_after = T1
updated_before = T2
       ↓
cursor = A
       ↓
page 1
       ↓
cursor = B
       ↓
page 2
```

The extraction window should remain fixed for the entire cursor traversal.

Do not rebuild the time window after every page.

Correct:

```text
RUN START
   ↓
capture lower + upper bounds
   ↓
cursor traversal
   ↓
complete window
   ↓
advance watermark
```

## 17. Cursor + Checkpoint + Watermark

When both cursor and watermark exist, store both when necessary.

Example:

```json
{
  "resource": "customers",
  "lower_bound": "2026-09-26T10:00:00Z",
  "upper_bound": "2026-09-26T11:00:00Z",
  "next_cursor": "abc123",
  "status": "in_progress"
}
```

During the run:

```text
fixed window
     ↓
cursor A
     ↓
cursor B
     ↓
cursor C
     ↓
window complete
     ↓
watermark advances to upper bound
```

Do not advance the global watermark merely because one cursor page succeeded.

## 18. Cursor + Retries

Retries should repeat the same logical cursor request.

```text
REQUEST cursor B
     ↓
timeout
     ↓
backoff
     ↓
REQUEST cursor B
```

Do not advance to cursor C after a failed request for cursor B.

Request-level retry state and pagination state are separate:

```text
pagination state = cursor B
retry state = attempt 3
```

## 19. Cursor + Rate Limits

Cursor traversal can still generate many requests.

Use:

```text
CURSOR
  ↓
RATE LIMIT
  ↓
REQUEST
  ↓
FAILURE?
  ↓
BACKOFF + JITTER
  ↓
RETRY
```

Never treat cursor pagination as permission to bypass API quotas.

## 20. Large Cursor-Based Extractions

Do not load every page into memory.

Bad:

```python
all_rows = []
while cursor:
    all_rows.extend(fetch(cursor))
```

Better:

```python
while cursor:
    rows, cursor = fetch(cursor)
    persist_rows(rows)
    save_cursor(cursor)
```

This keeps memory bounded by the current page or batch.

## 21. Parallel Cursor Traversal

Cursor chains are often inherently sequential:

```text
A → B → C → D
```

Because B is unknown until A completes, simple parallelization is impossible.

Do not create multiple workers that independently invent cursor positions.

Parallelism may be possible only when the provider exposes independent partitions or cursor streams.

Example:

```text
tenant A → cursor chain A1 → A2 → A3
tenant B → cursor chain B1 → B2 → B3
tenant C → cursor chain C1 → C2 → C3
```

Those independent streams can potentially run concurrently.

## 22. Cursor Token Security

Cursors may contain encoded source state or sensitive query information.

Treat them as operational state.

Do not:

- print full cursors in logs
- expose them unnecessarily to users
- store them in URLs that are publicly observable unless required
- assume encoding means encryption

Prefer a redacted representation in logs:

```python
def redact_cursor(cursor):
    if not cursor:
        return None
    return cursor[:4] + "..."
```

## 23. Testing

### Unit tests

Test:

- initial request without cursor
- next cursor extraction
- cursor persistence
- `None` termination
- empty page with next cursor
- repeated cursor detection
- invalid cursor classification
- cursor expiration
- checkpoint versioning
- fixed query context

Example:

```python
def test_empty_page_with_next_cursor_continues():
    response = {
        "data": [],
        "next_cursor": "abc123",
    }

    assert response["next_cursor"] is not None
```

### Integration tests

Create a fake API:

```text
request 1 → rows [1, 2], cursor A
request 2 → rows [3, 4], cursor B
request 3 → rows [5], cursor null
```

Verify:

- all five records are persisted
- three requests are made
- final cursor is null
- pipeline marks the extraction complete

### Recovery test

Simulate:

```text
cursor A → persisted
cursor B → API success
cursor B data → persistence failure
restart
cursor B → retry
```

Verify that the checkpoint remains at the last safe boundary.

## 24. Intentional Failure

### Failure drill A — Crash before cursor checkpoint

Persist page data and crash before saving the next cursor.

Expected:

```text
restart from previous cursor
replay page
idempotent persistence handles duplicate work
```

### Failure drill B — Corrupt cursor

Replace the checkpoint with an invalid cursor.

Expected:
- API rejects it
- retry policy does not loop forever
- recovery path is invoked

### Failure drill C — Cursor loop

Make the fake API return:

```text
A → B → B → B
```

Expected:
- loop detected
- extraction stops safely
- no unbounded API traffic

### Failure drill D — Expired cursor

Make the provider reject a previously valid cursor.

Expected:
- cursor-expiration handling runs
- pipeline does not silently skip the affected range
- recovery/reconciliation is triggered

### Failure drill E — Empty page with continuation

Return an empty page with a valid next cursor.

Expected:
- pipeline continues
- extraction does not terminate prematurely

## 25. Observability

Track cursor behavior without exposing full token values.

| Metric / Signal | Purpose |
|---|---|
| pages processed | Measures extraction progress |
| cursor advances | Detects normal traversal |
| repeated cursor count | Detects cursor loops |
| cursor errors | Detects invalid/expired state |
| cursor age | Detects stale extraction state |
| records per page | Measures source response behavior |
| empty pages with continuation | Detects unusual provider behavior |
| retries per cursor | Detects unstable requests |
| extraction lag | Measures source synchronization delay |

Useful logs:

```text
run_id
resource
cursor_hash
previous_cursor_hash
page_number
records_count
attempt
error_class
next_cursor_present
```

Hash or redact cursor values rather than logging sensitive tokens in full.

## 26. Recovery

When cursor extraction fails:

1. Identify the last trusted cursor checkpoint.
2. Verify the data associated with that checkpoint.
3. Determine whether the cursor is still valid.
4. Retry transient request failures with bounded backoff.
5. If the cursor expired, use the provider's documented recovery method.
6. Replay from a safe boundary when necessary.
7. Deduplicate replayed records.
8. Reconcile the affected extraction range.

If the cursor cannot be trusted, do not invent a replacement cursor.

## 27. Production Tools You Should Know

### HTTPX
Useful for implementing cursor-aware API clients with explicit request and timeout control.

### Requests
A common synchronous HTTP client for implementing provider-specific cursor traversal.

### Airbyte
Provides production connector patterns that include cursor/state-based incremental synchronization.

Learn the provider's cursor contract before relying on a connector abstraction.

## 28. Production Runbook

### Before deployment

- Document cursor semantics.
- Document cursor expiration.
- Document termination conditions.
- Document query parameters that define cursor context.
- Define checkpoint meaning.
- Define recovery for invalid cursors.
- Define loop detection.
- Define idempotent persistence.
- Define cursor-safe logging.

### If extraction stops unexpectedly

1. Inspect the last trusted cursor checkpoint.
2. Check whether the cursor is expired.
3. Check provider errors.
4. Check retry history.
5. Verify persisted data.
6. Resume from the safe cursor.
7. Reconcile after completion.

### If a cursor loop occurs

1. Stop the extraction.
2. Record the cursor hash.
3. Confirm provider behavior.
4. Do not keep retrying the same cursor.
5. Escalate or switch to the documented recovery mechanism.

### What not to do

- Do not decode opaque cursors without a documented reason.
- Do not assume an empty page means completion.
- Do not advance the cursor before persistence.
- Do not reuse a cursor with a different query context.
- Do not retry invalid cursors forever.
- Do not log full cursor tokens unnecessarily.
- Do not parallelize a single cursor chain without provider support.

## 29. Common Mistakes

1. Treating a cursor like an offset.
2. Advancing the checkpoint before persistence.
3. Breaking on the first empty page.
4. Ignoring cursor expiration.
5. Failing to detect repeated cursors.
6. Reusing cursors with changed filters or sort order.
7. Logging sensitive cursor values.
8. Loading the entire cursor stream into memory.
9. Advancing an incremental watermark before the cursor traversal completes.
10. Assuming a cursor can always be reconstructed.

## 30. Definition of Done

- [ ] Cursor semantics are documented.
- [ ] Initial and subsequent requests are implemented.
- [ ] Cursors are treated as opaque.
- [ ] The checkpoint meaning is explicit.
- [ ] Data persistence precedes cursor advancement.
- [ ] Cursor state is durable.
- [ ] Cursor expiration is handled.
- [ ] Invalid cursors are classified.
- [ ] Cursor loops are detected.
- [ ] Empty-page continuation is handled.
- [ ] Query context remains stable.
- [ ] Cursor traversal works with retries and rate limits.
- [ ] Memory remains bounded.
- [ ] Recovery is tested.
- [ ] Cursor values are safely represented in logs.
- [ ] Reconciliation exists for recovery scenarios.

## 31. What You Learned

After this recipe, you should be able to independently:

- explain why cursor pagination differs from offset pagination
- implement cursor traversal
- checkpoint the next safe cursor
- treat cursors as opaque provider state
- detect cursor loops
- handle empty pages correctly
- handle cursor expiration
- preserve cursor query context
- combine cursor extraction with incremental windows
- combine cursors with retries, backoff, and rate limits
- recover safely after crashes
- test cursor extraction independently
- operate cursor-based extraction in production

### Core Mental Model

```text
LOAD SAFE CURSOR
      ↓
REQUEST CURSOR
      ↓
VALIDATE RESPONSE
      ↓
PERSIST DATA
      ↓
VERIFY / COMMIT
      ↓
CHECKPOINT NEXT CURSOR
      ↓
NEXT CURSOR
      ↓
REPEAT UNTIL PROVIDER SAYS STOP
```

> A cursor is source-controlled progress state. Treat it as opaque, durable, and valid only within its documented query context.