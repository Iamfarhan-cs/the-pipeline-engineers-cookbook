# E17 — API Pagination

## 1. Problem Recognition

APIs rarely return an entire dataset in one response. Instead, they expose a pagination contract that tells the client how to request the next portion of the dataset.

Recognize this problem when an API response contains fields such as `page`, `offset`, `limit`, `next`, `next_cursor`, `next_token`, or a `Link` header, or when documentation says that results are paginated.

The production problem is not simply how to fetch page 2. It is how to move through every page **without skipping records, duplicating records, looping forever, or losing progress when a run fails**.

### What can go wrong

| Failure | Result |
|---|---|
| Unstable ordering | Records can be skipped or duplicated between pages |
| Wrong termination condition | Pipeline stops early or requests pages forever |
| Repeated cursor | Infinite loop |
| Expired continuation token | Extraction cannot continue from an old state |
| Checkpoint before persistence | Restart can lose records |
| Checkpoint after persistence but before state update | Safe duplicate processing may occur unless writes are idempotent |
| Very large page size | Slow responses, memory pressure, API rejection |
| Very small page size | Excessive request count and latency |

## 2. Concept and Reasoning

Pagination is a **state machine**. The current pagination state identifies the next portion of the source that must be requested.

Common contracts include:

### Page-number pagination

```text
page=1&per_page=100
page=2&per_page=100
...
```

### Offset/limit pagination

```text
offset=0&limit=100
offset=100&limit=100
...
```

### Cursor pagination

```text
cursor=<opaque-value>
```

The response supplies the cursor for the next request.

### Link-header pagination

```text
Link: <https://api.example.com/items?page=2>; rel="next"
```

### Continuation-token pagination

The response returns an opaque token such as `next_token`. The client sends it back on the next request.

Do not assume that one pagination mechanism can be inferred from another. **Discover the provider's actual contract.**

### The critical ordering rule

```text
REQUEST PAGE
     ↓
VALIDATE PAGE
     ↓
PERSIST RECORDS
     ↓
VERIFY PERSISTENCE
     ↓
CHECKPOINT PAGINATION STATE
     ↓
DETERMINE NEXT PAGE
     ↓
REQUEST NEXT PAGE
```

> Pagination state is progress state. Persist it only after the records represented by that page are durably persisted.

This ordering allows a failed run to repeat a page safely. Therefore the destination should also use an idempotent write or deduplication strategy.

## 3. Implementation

### Step 1 — Define the pagination contract

Before writing the loop, document:

- pagination type
- request parameter or header
- starting state
- maximum page size
- ordering requirement
- response field containing records
- next-page field/header
- termination condition
- cursor/token lifetime
- whether the source is mutable during extraction
- whether the provider offers a snapshot

Example contract:

```python
PAGINATION = {
    "type": "cursor",
    "page_size": 100,
    "records_field": "items",
    "next_cursor_field": "next_cursor",
    "max_pages": 100_000,
}
```

### Step 2 — Start with a provider-neutral paginator

```python
from dataclasses import dataclass
from typing import Any, Callable, Iterable

@dataclass
class Page:
    records: list[dict[str, Any]]
    next_state: str | None
    raw: dict[str, Any]


class PaginationError(RuntimeError):
    pass


def paginate(
    fetch_page: Callable[[str | None], Page],
    persist_page: Callable[[list[dict[str, Any]]], None],
    checkpoint: Callable[[str | None], None],
    start_state: str | None = None,
    max_pages: int = 100_000,
) -> int:
    state = start_state
    seen_states: set[str] = set()
    pages = 0

    while True:
        if pages >= max_pages:
            raise PaginationError("maximum page limit exceeded")

        if state is not None:
            if state in seen_states:
                raise PaginationError("repeated pagination state detected")
            seen_states.add(state)

        page = fetch_page(state)

        if not isinstance(page.records, list):
            raise PaginationError("page records must be a list")

        # Durable persistence happens before pagination checkpointing.
        persist_page(page.records)
        checkpoint(page.next_state)

        pages += 1

        if page.next_state is None:
            return pages

        state = page.next_state
```

This abstraction deliberately does not know whether `state` is a page number, offset, cursor, URL, or token.

### Step 3 — Page-number pagination

```python
def page_number_pages(fetch, persist, start_page=1, page_size=100):
    page = start_page

    while True:
        response = fetch(page=page, per_page=page_size)
        records = response["items"]

        persist(records)

        if not response.get("has_next", False):
            break

        page += 1
```

Do not rely only on `len(records) < page_size` unless the provider explicitly documents that as a valid termination rule.

### Step 4 — Offset/limit pagination

```python
def offset_pages(fetch, persist, start_offset=0, limit=100):
    offset = start_offset

    while True:
        response = fetch(offset=offset, limit=limit)
        records = response["items"]
        persist(records)

        if not response.get("has_more", False):
            break

        offset += len(records)
```

Using the actual number of returned records is safer than blindly adding `limit` when the API can return fewer records while still having more data.

Offset pagination can become unsafe on a changing dataset. If records are inserted or deleted before the current offset, later pages can shift.

### Step 5 — Cursor pagination

```python
def cursor_pages(fetch, persist, start_cursor=None):
    cursor = start_cursor
    seen = set()

    while True:
        if cursor is not None:
            if cursor in seen:
                raise PaginationError("cursor repeated")
            seen.add(cursor)

        response = fetch(cursor=cursor)
        records = response["items"]
        next_cursor = response.get("next_cursor")

        persist(records)

        if next_cursor is None:
            break

        cursor = next_cursor
```

Never manufacture cursor values. Treat cursors as opaque provider state.

### Step 6 — Link-header pagination

```python
from urllib.parse import parse_header_links

def next_link(link_header: str | None) -> str | None:
    if not link_header:
        return None

    links = parse_header_links(link_header.rstrip(","))
    for link in links:
        if link.get("rel") == "next":
            return link.get("url")
    return None
```

Persist the provider's next URL or enough state to reconstruct it only when that URL is safe and durable for the required recovery window.

### Step 7 — Continuation-token pagination

```python
def token_pages(fetch, persist, start_token=None):
    token = start_token
    seen = set()

    while True:
        if token is not None:
            if token in seen:
                raise PaginationError("continuation token repeated")
            seen.add(token)

        response = fetch(next_token=token)
        persist(response["items"])

        token = response.get("next_token")
        if token is None:
            break
```

Tokens can expire. An expired token is not automatically recoverable by inventing another token. The recovery strategy must follow the provider contract, such as restarting from a stable boundary or using a fresh snapshot.

### Step 8 — Persist page-level checkpoint state

A useful checkpoint contains enough information to identify the next request:

```python
checkpoint = {
    "source": "customer_api",
    "run_id": "2026-09-26T12:00:00Z",
    "pagination_type": "cursor",
    "cursor": "opaque-next-cursor",
    "pages_completed": 42,
    "records_persisted": 4187,
    "updated_at": "2026-09-26T12:42:00Z",
}
```

Do not treat `pages_completed` as the real progress boundary. The cursor, offset, page number, or next URL is the meaningful pagination state.

### Step 9 — Make the destination idempotent

A crash can happen after persistence and before checkpoint completion. The next run may therefore process the same page again.

Example PostgreSQL pattern:

```sql
INSERT INTO raw_customer (customer_id, payload)
VALUES (%s, %s)
ON CONFLICT (customer_id) DO NOTHING;
```

For mutable records, an upsert may be more appropriate than `DO NOTHING`. The important point is that replaying a page must not corrupt the destination.

## 4. Page Size Trade-Offs

| Page size | Benefit | Cost |
|---|---|---|
| Small | Lower memory and response size | More requests and overhead |
| Medium | Balanced throughput | Requires tuning |
| Large | Fewer requests | Higher latency, memory, timeout, and API-limit risk |

Start from the provider's documented maximum or recommended size, then measure.

Do not increase page size simply because the pipeline is slow. First determine whether the bottleneck is request latency, server processing, serialization, network transfer, persistence, or rate limiting.

## 5. Stable Ordering and Changing Data

Pagination over a mutable source can produce duplicates or gaps.

Example:

```text
Page 1: A B C
        ↓
New record X is inserted before C
        ↓
Page 2 now starts at a different position
```

Offset pagination is especially sensitive to this problem.

Safer approaches include:

- provider snapshot semantics
- cursor pagination designed for a stable traversal
- deterministic ordering
- immutable extraction windows
- timestamp or ID boundaries
- overlap windows followed by deduplication
- source-side snapshot identifiers

Never claim a paginated extraction is complete merely because the final page was reached. Completeness depends on the source's consistency semantics.

## 6. Pagination + Incremental Extraction

Pagination and incrementality solve different problems.

- **Incremental extraction** defines which records belong in the extraction window.
- **Pagination** defines how those records are divided into requests.

Example:

```text
updated_at >= watermark
        ↓
ORDER BY updated_at, id
        ↓
PAGE/CURSOR THROUGH RESULT
        ↓
PERSIST
        ↓
ADVANCE SAFE WATERMARK
```

Use a deterministic tie-breaker such as `id` when ordering by a non-unique timestamp.

Do not advance the high-watermark beyond data that has actually been persisted.

## 7. Concurrent Pagination

Pagination pages cannot automatically be fetched in parallel.

Parallelization may be safe when:

- the API supports independent partitions
- the provider guarantees snapshot consistency
- pages are independent and deterministic
- rate limits permit the concurrency
- destination writes are idempotent

It is usually unsafe to concurrently request offset pages from a changing dataset because page boundaries can shift.

Cursor chains are inherently sequential when each next cursor depends on the previous response.

## 8. Large Result Sets

Never build the entire API dataset in memory just because the API is paginated.

Prefer:

```text
REQUEST PAGE
     ↓
VALIDATE
     ↓
PERSIST PAGE
     ↓
RELEASE MEMORY
     ↓
REQUEST NEXT PAGE
```

Bound memory by processing one page or a controlled batch at a time.

## 9. Testing

### Unit tests

Test at least:

- first page
- multiple pages
- empty final page
- explicit `has_next=false`
- missing next cursor
- repeated cursor
- repeated continuation token
- maximum page limit
- malformed page response
- persistence failure
- checkpoint failure
- restart from checkpoint
- cursor expiry
- duplicate records across pages
- unstable ordering scenario

Example:

```python
def test_repeated_cursor_fails():
    responses = iter([
        {"items": [{"id": 1}], "next_cursor": "abc"},
        {"items": [{"id": 2}], "next_cursor": "abc"},
    ])

    def fetch(cursor):
        return next(responses)

    seen = set()
    cursor = None

    for _ in range(10):
        response = fetch(cursor)
        next_cursor = response["next_cursor"]

        if next_cursor in seen:
            break

        seen.add(next_cursor)
        cursor = next_cursor
    else:
        raise AssertionError("loop protection did not trigger")
```

### Integration tests

Use a fake or test API that can simulate:

1. normal multi-page extraction
2. empty result set
3. repeated cursor
4. expired cursor
5. page request timeout
6. persistence failure
7. restart from checkpoint
8. records duplicated at a page boundary

## 10. Observability

Track pagination-specific signals.

| Signal | Why it matters |
|---|---|
| Pages requested | Measures extraction progress |
| Records per page | Detects unusual page behavior |
| Page latency | Finds slow API pages |
| Empty pages | Helps diagnose termination behavior |
| Pagination failures | Detects provider/API problems |
| Repeated cursor count | Detects loops |
| Termination reason | Explains why extraction stopped |
| Current pagination state | Supports recovery |
| Checkpoint age | Shows stale progress |
| Duplicate records | Detects boundary/replay problems |

Useful structured log fields:

```text
run_id
source
resource
pagination_type
page_number
offset
cursor_hash
records_received
page_latency_ms
termination_reason
checkpoint_state
```

Do not log sensitive cursor/token values if they contain secrets or credentials. Hash or redact them where appropriate.

## 11. Intentional Failure

Break the paginator deliberately.

### Failure drill A — Repeated cursor

Return the same cursor forever.

Expected result:

- paginator detects repetition
- extraction stops
- no infinite request loop
- failure is visible in logs/metrics

### Failure drill B — Persistence failure

Make page 3 fail during database persistence.

Expected result:

- checkpoint does not advance past page 2
- page 3 can be retried
- already persisted records remain valid

### Failure drill C — Crash after persistence

Simulate a process crash immediately after page persistence and before checkpoint completion.

Expected result:

- restart repeats the affected page
- idempotent destination prevents corruption
- checkpoint eventually advances after successful persistence

### Failure drill D — Expired cursor

Make the API reject the saved cursor.

Expected result:

- cursor-expiry error is classified
- pipeline does not invent a cursor
- recovery follows a documented restart or boundary strategy

## 12. Recovery

When pagination fails:

1. Identify the last durable checkpoint.
2. Identify the pagination type.
3. Determine whether the saved state is still valid.
4. Check whether the previous page was persisted.
5. Resume from the last safe state.
6. Allow safe duplicate processing where necessary.
7. Verify destination counts and uniqueness.
8. Record the recovery outcome.

If pagination state is invalid or expired, restart from a known safe boundary rather than guessing.

## 13. Production Tools You Should Know

### Requests
Python HTTP client commonly used to implement explicit pagination loops and inspect raw responses.

### HTTPX
Python HTTP client useful when you need synchronous or asynchronous HTTP behavior around pagination.

### Airbyte
Production data-movement platform whose connectors demonstrate reusable pagination, checkpointing, and incremental extraction patterns.

These tools do not replace understanding the pagination contract. You must still know what the provider's page boundaries and progress semantics mean.

## 14. Production Runbook

### Before running

- Confirm pagination type.
- Confirm termination semantics.
- Confirm stable ordering.
- Confirm page-size limits.
- Confirm cursor/token lifetime.
- Confirm checkpoint storage.
- Confirm destination idempotency.
- Confirm API rate limits.

### During a run

- Monitor page latency.
- Monitor records per page.
- Watch for repeated cursors.
- Watch for unexpected empty pages.
- Watch request and rate-limit errors.
- Confirm checkpoints advance only after persistence.

### If extraction stops unexpectedly

- Inspect the last durable checkpoint.
- Inspect the last successful page.
- Check whether the pagination state expired.
- Check whether the source changed during extraction.
- Resume from the safe checkpoint.
- Reconcile duplicates and missing records.

### What not to do

- Do not increment page state before persistence.
- Do not assume `len(page) < page_size` is universally valid termination.
- Do not treat cursors as arithmetic values.
- Do not ignore repeated cursors.
- Do not parallelize cursor chains without understanding their semantics.
- Do not use huge pages without measuring the effect.
- Do not claim completeness without understanding source consistency.

## 15. Common Mistakes

1. Treating pagination as a simple `for page in range(...)` loop.
2. Advancing checkpoints before records are durable.
3. Forgetting loop detection.
4. Assuming page numbers remain stable on mutable data.
5. Ignoring provider-specific termination rules.
6. Using offset pagination when the source requires a stable cursor.
7. Loading all pages into memory.
8. Retrying a failed page without idempotent destination writes.
9. Logging opaque tokens that may contain sensitive information.
10. Treating an expired cursor as a transient network error.

## 16. Definition of Done

- [ ] Pagination contract is documented.
- [ ] Pagination type is implemented correctly.
- [ ] Termination is explicit.
- [ ] Stable ordering has been evaluated.
- [ ] Page size is intentional.
- [ ] Pagination state is durable.
- [ ] Records are persisted before checkpoint advancement.
- [ ] Repeated pagination state is detected.
- [ ] Maximum-page protection exists.
- [ ] Destination writes are safe to replay.
- [ ] Restart from checkpoint has been tested.
- [ ] Pagination failures are observable.
- [ ] Cursor/token expiry behavior is documented.
- [ ] Mutable-source consistency risks are understood.
- [ ] Intentional failure drills pass.

## 17. What You Learned

After this recipe, you should be able to independently:

- recognize the pagination contract of an unfamiliar API
- implement page, offset, cursor, link, and token pagination
- choose a reasonable page size
- define safe termination conditions
- detect repeated pagination state
- persist page-level progress safely
- resume after failure
- reason about mutable datasets and page-boundary changes
- combine pagination with incremental extraction
- decide when concurrent pagination is safe
- diagnose pagination failures using operational signals

### Core Mental Model

```text
API PAGINATION
      ↓
UNDERSTAND PAGE CONTRACT
      ↓
REQUEST ONE PAGE
      ↓
VALIDATE
      ↓
PERSIST RECORDS
      ↓
VERIFY PERSISTENCE
      ↓
CHECKPOINT PAGINATION STATE
      ↓
DETERMINE NEXT PAGE
      ↓
REPEAT SAFELY
```

> The pagination loop is not the hard part. The hard part is making pagination **complete, resumable, observable, and safe under failure**.