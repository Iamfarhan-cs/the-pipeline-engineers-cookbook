# E02 — Extract Data from Paginated APIs

Many REST APIs do not return an entire dataset in one response. They split results across pages.

A production extractor must understand the API pagination contract, retrieve every required page, stop at the correct boundary, survive failures, and avoid silently skipping or duplicating records.

This recipe focuses specifically on pagination mechanics. E01 introduced REST API extraction generally; this recipe builds deep pagination handling from scratch.

## 1. Problem Recognition

An API may return:

    {
        "data": [...],
        "page": 1,
        "total": 5000
    }

If the extractor reads only the first response, it processes only part of the dataset.

Common symptoms:

- record counts are unexpectedly low
- downstream tables contain only the first page
- extraction volume suddenly drops
- pages are skipped
- records appear twice
- extraction loops forever
- the API changes results between requests
- a failed page causes the whole extraction to restart unnecessarily

The production problem is:

> How do I reliably traverse an API's complete pagination space without skipping, duplicating, or endlessly requesting data?

## 2. Concept and Reasoning

Pagination is an API protocol.

Do not invent pagination behavior.

The API documentation determines:

- page numbering
- page size
- maximum page size
- next-page representation
- cursor semantics
- total-count semantics
- ordering guarantees
- termination condition

The extractor should model pagination explicitly:

    API
      ↓
    PAGE 1
      ↓
    PAGE 2
      ↓
    PAGE 3
      ↓
    ...
      ↓
    END

Each transition must be based on evidence from the API response.

## 3. Common Pagination Models

### Page number

    ?page=1
    ?page=2
    ?page=3

### Offset

    ?offset=0&limit=100
    ?offset=100&limit=100
    ?offset=200&limit=100

### Cursor

    ?cursor=abc
    ?cursor=def
    ?cursor=ghi

### Link-based

The response may provide a next URL.

### Token-based

The API may return a next token.

Each model has different correctness properties.

## 4. Page-Based Pagination

A simple implementation:

    def paginate_pages(client, path, page_size=100):
        page = 1

        while True:
            payload = client.get(
                path,
                params={"page": page, "limit": page_size},
            ).json()

            records = payload["data"]

            if not records:
                break

            yield page, records
            page += 1

This is appropriate only when the API contract defines this behavior.

## 5. Stop Conditions

A paginator needs an explicit termination rule.

Possible rules:

    empty data page
    current_page >= total_pages
    offset >= total_count
    next == null
    next_cursor == null
    has_more == false

Do not mix termination rules casually.

If the API says has_more is authoritative, use that contract rather than guessing that an empty page means completion.

## 6. Page Size

Larger pages reduce request count but can increase:

- response size
- memory usage
- API processing time
- timeout probability
- retry cost

Smaller pages increase request count but reduce individual response size.

Choose page size according to the API contract and measured behavior.

## 7. Maximum Page Size

Many APIs silently cap page size.

You request:

    limit=10000

The API returns:

    1000 records

Determine whether the API capped the request and continue according to its pagination metadata.

Never assume the requested page size was honored.

## 8. Offset Pagination

Offset pagination is conceptually:

    offset = 0

    while True:
        records = fetch(offset, limit)

        if not records:
            break

        process(records)
        offset += len(records)

This can become unstable when the source dataset changes during extraction.

Example:

    Initial:
    A B C D E

    Page 1:
    A B

    New record inserted:
    X A B C D E

    Page 2 with offset=2:
    B C

B may now be extracted twice.

The exact behavior depends on ordering and consistency guarantees.

## 9. Cursor Pagination

Cursor pagination can avoid many offset instability problems.

Response:

    {
        "data": [...],
        "next_cursor": "abc123"
    }

Next request:

    ?cursor=abc123

Treat the cursor as opaque.

Do not decode, increment, construct, or modify it unless the API explicitly documents that behavior.

## 10. Cursor Safety

A good cursor loop checks that the cursor advances:

    cursor = None

    while True:
        params = {"limit": 100}

        if cursor is not None:
            params["cursor"] = cursor

        payload = request(params)
        yield payload["data"]

        next_cursor = payload.get("next_cursor")

        if not next_cursor:
            break

        if next_cursor == cursor:
            raise RuntimeError("Pagination cursor did not advance")

        cursor = next_cursor

This prevents a broken API from causing an infinite loop.

## 11. Link-Based Pagination

Some APIs return a server-generated next URL.

Use the returned link when the API contract defines it as authoritative.

Do not reconstruct the URL unnecessarily.

A server-generated link may contain:

- opaque cursor state
- encoded filters
- signed tokens
- version information

## 12. Page Integrity

Each page should produce extraction evidence:

    page_number
    request_started_at
    response_status
    record_count
    request_duration
    extraction_run_id

This makes it possible to answer:

> Which page failed?

rather than only:

> The extraction failed.

## 13. Persist Before Advancing

Consider:

    page 1 ✓
    page 2 ✓
    page 3 ✓
    page 4 extracted
    page 4 persistence ✗

The paginator must not mark page 4 complete.

Correct sequence:

    REQUEST PAGE
       ↓
    VALIDATE
       ↓
    PERSIST
       ↓
    VERIFY
       ↓
    CHECKPOINT PAGE
       ↓
    REQUEST NEXT PAGE

This is one of the most important pagination safety rules.

## 14. Resume After Failure

Suppose:

    page 1 ✓
    page 2 ✓
    page 3 ✓
    page 4 ✗

After restart, resume according to the API pagination contract.

For cursor APIs, store the last successfully processed cursor.

For offset APIs, store enough state to reproduce the correct next request.

Do not assume that page number alone is safe for a changing source.

## 15. Duplicate Prevention

Duplicates can occur because of:

- retries
- source changes
- unstable ordering
- overlapping extraction windows
- restart behavior

Use a stable source identifier where available:

    source_record_id

Then enforce the appropriate uniqueness rule downstream.

A pagination system should be designed with the expectation that duplicates are possible.

## 16. Ordering

Pagination correctness often depends on ordering.

If the API supports deterministic ordering, use it.

For example:

    sort=created_at

A stronger ordering may use:

    created_at + stable_id

when the API supports compound ordering.

The exact strategy must follow the API's capabilities.

## 17. Snapshot Consistency

Some APIs provide a snapshot or consistent query boundary.

If available, use it for large extractions.

Conceptually:

    SNAPSHOT
       ↓
    PAGE 1
       ↓
    PAGE 2
       ↓
    PAGE 3
       ↓
    ...
       ↓
    COMPLETE

Without snapshot consistency, records may be inserted, updated, or deleted during extraction.

That can make exact page-by-page completeness difficult.

## 18. Pagination and Incremental Extraction

Pagination and incremental extraction solve different problems.

Pagination asks:

> How do I retrieve all results in this request window?

Incremental extraction asks:

> Which records do I need to retrieve?

They often work together:

    updated_at > watermark
             ↓
       API result set
             ↓
          paginate
             ↓
      persist all pages
             ↓
       advance watermark

Do not confuse page number with extraction progress across time.

## 19. Large Result Sets

For millions of records, do not accumulate the entire dataset in memory.

Avoid:

    all_records = []

    for page in pages:
        all_records.extend(page)

Prefer:

    for page in pages:
        persist(page)

The extractor should process pages incrementally:

    PAGE
      ↓
    VALIDATE
      ↓
    PERSIST
      ↓
    RELEASE MEMORY
      ↓
    NEXT PAGE

## 20. Bounded Pagination

A production paginator should have safety limits such as:

    maximum_pages
    maximum_records
    maximum_duration

These are guardrails, not normal termination conditions.

Example:

    if page_count > max_pages:
        raise RuntimeError("Pagination safety limit exceeded")

This prevents unexpected API behavior from creating unbounded extraction.

## 21. Testing

Test at least:

1. Single page — one page completes correctly.
2. Multiple pages — every page is requested correctly.
3. Empty final page — the documented termination rule works.
4. Next link — the returned link is followed.
5. Cursor advancement — each cursor is carried forward.
6. Cursor repetition — repeated cursor causes safe termination.
7. Failed page — checkpoint remains at the previous successful page.
8. Restart — extraction resumes safely.
9. Duplicate page — downstream duplicate handling works.
10. Large result set — pages are processed incrementally.
11. Maximum pages — safety limits prevent infinite extraction.
12. Changing source — behavior matches the documented consistency strategy.

## 22. Observability

Track:

    pages_requested_total
    pages_completed_total
    records_extracted_total
    pagination_failures_total
    pagination_retries_total
    api_request_duration_seconds
    extraction_duration_seconds

Log structured context such as:

    extraction_run_id
    endpoint
    page_number
    offset
    cursor
    record_count
    response_status
    attempt

Do not put high-cardinality cursor values into metric labels.

## 23. Intentional Failure

### Failure drill 1 — Skip a page

Simulate a broken pagination implementation.

Verify reconciliation detects missing records.

### Failure drill 2 — Repeat a cursor

Return the same cursor twice.

Verify the extractor stops instead of looping forever.

### Failure drill 3 — Persistence failure

Make page persistence fail.

Verify the checkpoint does not advance.

### Failure drill 4 — API changes page size

Return fewer records than requested.

Verify the extractor uses actual response metadata and the documented termination contract.

### Failure drill 5 — Source mutation

Insert records while extraction is running.

Observe offset pagination behavior and compare it with cursor/snapshot strategies.

### Failure drill 6 — Endless pagination

Make the API continuously return a next cursor.

Verify safety limits stop the extraction.

## 24. Recovery

When pagination fails:

1. Identify the extraction run.
2. Identify the last successful page/cursor.
3. Determine the pagination model.
4. Inspect persisted raw pages.
5. Verify the checkpoint.
6. Check whether the source changed during extraction.
7. Resume using documented pagination semantics.
8. Deduplicate where required.
9. Reconcile expected and actual records.
10. Complete only after the extraction boundary is verified.

Do not manually skip a failed page without determining whether its records were persisted.

## 25. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Requests** | Low-level HTTP pagination and response handling in Python. |
| **HTTPX** | Synchronous/asynchronous HTTP extraction and connection management. |
| **Airbyte** | Connector framework that handles many source-specific pagination patterns. |

> These tools provide implementation capabilities. The pagination mechanisms and correctness rules should still be understood independently.

## 26. Production Runbook

### Record count is lower than expected

Check:

1. Pagination model.
2. Number of completed pages.
3. Last page/cursor.
4. API filters.
5. Page size.
6. Termination condition.
7. Source mutation.

### Extraction loops forever

Check:

1. Cursor advancement.
2. Next-page link.
3. Page counter.
4. Safety limits.
5. API behavior.

### Duplicate records appear

Check:

1. Offset pagination.
2. Source ordering.
3. Retries.
4. Restart behavior.
5. Overlapping pages.
6. Source mutation.

### Extraction stops after a failure

Check:

1. Last durable checkpoint.
2. Persisted pages.
3. Failed page.
4. Retry classification.
5. Resume semantics.

### What not to do

Do not:

- assume every API uses page numbers
- assume requested page size is honored
- reconstruct opaque cursors
- use unlimited pagination
- advance checkpoints before persistence
- hold millions of records in memory
- assume page number alone is durable extraction state
- ignore source mutation

## 27. Common Mistakes

### Mistake 1 — Hard-coded page assumptions

Different APIs use different pagination contracts.

### Mistake 2 — No termination guard

A malformed API can cause an infinite loop.

### Mistake 3 — Offset pagination without understanding source mutation

Records can move between pages.

### Mistake 4 — Treating cursor as an ID

A cursor is usually opaque.

### Mistake 5 — Accumulating the entire dataset

Large extractions can exhaust memory.

### Mistake 6 — No page-level checkpoint

Recovery becomes unnecessarily expensive.

### Mistake 7 — No reconciliation

A pipeline can report success after silently missing pages.

## 28. Definition of Done

You are done when you can:

- explain why APIs paginate
- identify page, offset, cursor, link, and token pagination
- inspect an API's pagination contract
- implement page-based pagination
- implement offset pagination
- implement cursor pagination
- follow link-based pagination
- implement explicit termination rules
- detect cursor loops
- enforce pagination safety limits
- process large datasets incrementally
- persist pages before advancing state
- resume after a failed page
- reason about source mutation
- use deterministic ordering where supported
- understand snapshot consistency
- combine pagination with incremental extraction
- test duplicate and missing-page scenarios
- intentionally break pagination
- recover without silently skipping data
- operate pagination using a production runbook

## 29. What You Learned

The central principle is:

> Pagination is a correctness problem before it is a performance problem.

A safe paginator follows:

    REQUEST PAGE
        ↓
    VALIDATE
        ↓
    PERSIST
        ↓
    VERIFY
        ↓
    CHECKPOINT
        ↓
    ADVANCE PAGINATION STATE
        ↓
    REQUEST NEXT PAGE

For mutable APIs, always ask:

> What guarantees that page N today represents the correct continuation of page N-1?

That question determines whether simple page numbers are sufficient or whether you need cursors, deterministic ordering, snapshots, or another consistency strategy.
