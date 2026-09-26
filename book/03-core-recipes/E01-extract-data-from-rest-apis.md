# E01 — Extract Data from REST APIs

REST APIs are one of the most common external data sources in Data Engineering.

A production API extractor must do more than send an HTTP request. It must handle authentication, timeouts, response validation, pagination, rate limits, retries, partial failure, checkpointing, raw-data preservation, and safe reruns.

This recipe builds the extraction mechanism from scratch so you can recognize and implement REST API ingestion without depending on a framework.

## 1. Problem Recognition

A simple script looks like:

    response = requests.get(url)
    data = response.json()

That is not a production extractor.

Real APIs can:

- time out
- return 429 rate limits
- return 500-series errors
- return malformed JSON
- change response schemas
- paginate results
- expire authentication tokens
- return partial results
- return duplicate records
- disconnect during a long extraction
- change data between requests

The production problem is:

> **How do I reliably extract a complete and reproducible dataset from an external REST API without losing data or repeatedly creating duplicates?**

## 2. Concept and Reasoning

A basic extraction lifecycle is:

    API
      ↓
    Authenticate
      ↓
    Request
      ↓
    Validate response
      ↓
    Parse
      ↓
    Persist raw response
      ↓
    Record extraction state
      ↓
    Continue
      ↓
    Complete

The extractor should separate:

1. HTTP communication
2. response validation
3. parsing
4. raw persistence
5. extraction state
6. downstream transformation

Do not mix all of these into one large function.

## 3. Understand the API Contract

Before writing the extractor, determine:

| Question | Why it matters |
|---|---|
| Authentication method? | Determines request headers/token handling |
| Pagination method? | Determines how all records are discovered |
| Maximum page size? | Controls request volume |
| Rate limit? | Controls request frequency |
| Timeout behavior? | Determines failure handling |
| Response schema? | Determines validation |
| Incremental field? | Enables incremental extraction |
| Ordering guarantee? | Affects checkpointing |
| Error codes? | Determines retry classification |
| Deletion semantics? | Determines whether extraction can detect deletes |

Do not assume the API behaves like another API.

The API contract is the source of truth.

## 4. Implementation Architecture

Build these components:

    APIClient
       │
       ├── authenticate()
       ├── request()
       ├── validate_response()
       └── retry()
              │
              ▼
        APIExtractor
              │
              ├── paginate()
              ├── parse()
              ├── persist_raw()
              └── checkpoint()
              │
              ▼
        Extraction State

The HTTP client should not know how pipeline state is stored.

The extractor should coordinate the workflow.

## 5. HTTP Client

Use explicit timeouts.

    import requests

    class APIClient:
        def __init__(self, base_url, token, timeout=30):
            self.base_url = base_url
            self.session = requests.Session()
            self.session.headers.update({
                "Authorization": f"Bearer {token}",
                "Accept": "application/json",
            })
            self.timeout = timeout

        def get(self, path, params=None):
            response = self.session.get(
                f"{self.base_url}{path}",
                params=params,
                timeout=self.timeout,
            )

            return response

Never rely on an implicit infinite network wait.

## 6. Response Classification

Not every HTTP error should be retried.

A useful classification:

| Response | Typical action |
|---|---|
| 200 | Process |
| 201 | Process if applicable |
| 204 | Process as empty response if contract allows |
| 400 | Usually permanent |
| 401 | Refresh/re-authenticate |
| 403 | Usually configuration/permission issue |
| 404 | Usually permanent or endpoint-specific |
| 408 | Retry |
| 409 | Contract-dependent |
| 429 | Retry after rate limit |
| 500 | Retry |
| 502 | Retry |
| 503 | Retry |
| 504 | Retry |

The exact policy must follow the API contract.

## 7. Authentication

Never hard-code credentials.

Use environment/configuration:

    API_BASE_URL
    API_TOKEN

Example:

    import os

    token = os.environ["API_TOKEN"]

Do not log:

    Authorization
    API tokens
    passwords
    refresh tokens
    secret headers

If the API uses OAuth, authentication should be treated as a separate lifecycle:

    obtain token
        ↓
    use token
        ↓
    detect expiry
        ↓
    refresh
        ↓
    continue

## 8. Response Validation

Never assume a successful HTTP response means valid data.

Example:

    response.raise_for_status()

    payload = response.json()

Then validate the expected structure:

    if not isinstance(payload, dict):
        raise ValueError("Expected JSON object")

    if "data" not in payload:
        raise ValueError("Missing data field")

The validator should distinguish:

    HTTP failure
    malformed JSON
    unexpected schema
    valid empty response
    valid response

An empty result is not necessarily an error.

## 9. Pagination

Many APIs return only a portion of the dataset.

Common pagination models:

### Offset pagination

    ?offset=0&limit=100
    ?offset=100&limit=100
    ?offset=200&limit=100

### Page pagination

    ?page=1&limit=100
    ?page=2&limit=100

### Cursor pagination

    ?cursor=abc
    ?cursor=def

Cursor pagination is often preferable for changing datasets because offsets can become unstable when records are inserted or removed.

Never assume pagination semantics.

## 10. Build Pagination

A simple page-based extractor:

    def extract_pages(client, path, page_size=100):
        page = 1

        while True:
            response = client.get(
                path,
                params={
                    "page": page,
                    "limit": page_size,
                },
            )

            response.raise_for_status()
            payload = response.json()

            records = payload["data"]

            if not records:
                break

            yield records

            page += 1

This is only correct if the API contract defines page/limit semantics.

## 11. Cursor Pagination

A cursor-based API may return:

    {
        "data": [...],
        "next_cursor": "abc123"
    }

Implementation:

    cursor = None

    while True:
        params = {"limit": 100}

        if cursor:
            params["cursor"] = cursor

        payload = client.get("/payments", params=params).json()

        yield payload["data"]

        cursor = payload.get("next_cursor")

        if not cursor:
            break

The cursor should be treated as opaque.

Do not attempt to interpret or modify it unless the API contract explicitly says to do so.

## 12. Raw Response Preservation

For important pipelines, preserve the raw API response before transformation.

Conceptually:

    API
      ↓
    RAW
      ↓
    STAGING
      ↓
    TRANSFORM
      ↓
    TARGET

A raw artifact can contain:

    source
    endpoint
    extraction_time
    request identifier
    response status
    response body
    schema/version metadata

This gives you a recovery point.

If transformation logic later contains a bug, you can reprocess the raw extraction without calling the external API again.

## 13. Extraction State

A production extractor needs state.

Example:

    extraction_state

    source
    endpoint
    cursor
    last_successful_page
    started_at
    completed_at
    status

For incremental APIs, state might instead contain:

    last_successful_timestamp

or:

    last_successful_id

The state must be updated only after the corresponding data has been safely persisted.

This ordering is critical.

Bad:

    update checkpoint
    ↓
    persist data

If persistence fails, the checkpoint can move past data that was never stored.

Safer:

    extract
       ↓
    persist
       ↓
    verify persistence
       ↓
    checkpoint

## 14. Checkpointing

Suppose the extractor processes:

    page 1 ✓
    page 2 ✓
    page 3 ✓
    page 4 ✗

The extractor should resume according to its pagination contract rather than blindly restarting everything.

For example:

    checkpoint = page 3

After recovery:

    resume from page 4

But checkpointing alone does not guarantee correctness.

You still need:

- idempotency
- duplicate handling
- correct pagination semantics
- safe persistence ordering

## 15. Retries

Retry only failures that are plausibly transient.

Use:

    bounded retries
    exponential backoff
    jitter

Conceptually:

    delay = base * 2^attempt + jitter

Example:

    attempt 1 → short delay
    attempt 2 → longer delay
    attempt 3 → longer delay
    exhausted → fail/quarantine

Do not retry permanent errors indefinitely.

## 16. Rate Limits

An API may return:

    HTTP 429

The response may contain:

    Retry-After: 10

Respect the server's instruction when provided.

The extraction flow becomes:

    request
      ↓
    429
      ↓
    wait
      ↓
    retry

Rate limiting is different from retrying a server error.

The extractor must also avoid creating a retry storm.

## 17. Timeouts

Always configure:

- connection timeout
- read timeout

Depending on the HTTP client, this may be represented as one combined timeout or separate values.

A timeout means the client does not know whether the server completed the operation.

For GET extraction requests, repeating the request is generally safer than for non-idempotent writes, but the extractor should still consider duplicate pages and changing datasets.

## 18. Deduplication

API extraction can produce duplicates because of:

- retries
- overlapping pages
- unstable pagination
- repeated source records
- extraction restarts

Use a stable source identifier where available:

    source_record_id

Then enforce uniqueness in the raw/staging layer where appropriate.

Do not use an arbitrary generated ID as the only duplicate-prevention mechanism.

## 19. Incremental Extraction

A full extraction may process:

    100 million records

every day.

If the API provides:

    updated_at

you may extract:

    updated_at > last_watermark

The flow becomes:

    last watermark
          ↓
    API filter
          ↓
    extract changes
          ↓
    persist
          ↓
    verify
          ↓
    advance watermark

Watermark advancement must happen only after successful persistence.

## 20. Complete Extractor Skeleton

A practical learning skeleton:

    def extract_api(client, path, state_store, raw_store):
        state = state_store.load(path)

        for page in paginate(
            client,
            path,
            state=state,
        ):
            validate_page(page)

            raw_id = raw_store.write(
                source=path,
                payload=page,
            )

            raw_store.verify(raw_id)

            state_store.advance(path, page)

    The key ordering is:

    REQUEST
       ↓
    VALIDATE
       ↓
    PERSIST RAW
       ↓
    VERIFY
       ↓
    CHECKPOINT

## 21. Testing

### Test 1 — Successful request

Verify a valid response becomes records.

### Test 2 — Timeout

Simulate a timeout and verify bounded retry behavior.

### Test 3 — 500

Verify retryable server errors are retried.

### Test 4 — 400

Verify permanent client errors are not retried indefinitely.

### Test 5 — 429

Verify Retry-After/rate-limit handling.

### Test 6 — Invalid JSON

Verify malformed responses fail safely.

### Test 7 — Missing field

Verify schema validation catches unexpected responses.

### Test 8 — Pagination

Verify all pages are extracted exactly once.

### Test 9 — Cursor pagination

Verify the cursor is carried forward correctly.

### Test 10 — Empty result

Verify a valid empty response completes successfully.

### Test 11 — Restart

Fail extraction after several pages and resume.

### Test 12 — Duplicate page

Verify duplicate records do not create duplicate target records when the pipeline contract requires uniqueness.

### Test 13 — Checkpoint ordering

Simulate persistence failure and verify the checkpoint does not advance.

### Test 14 — Authentication failure

Verify expired/invalid authentication follows the configured authentication policy.

## 22. Observability

Measure at minimum:

    api_requests_total
    api_request_failures_total
    api_request_duration_seconds
    api_429_total
    api_timeouts_total
    api_retries_total
    records_extracted_total
    pages_extracted_total
    extraction_failures_total
    extraction_duration_seconds

Log structured context such as:

    source
    endpoint
    extraction_run_id
    page/cursor
    attempt
    HTTP status
    duration
    record count

Never log secrets or sensitive response bodies indiscriminately.

## 23. Intentional Failure

### Failure drill 1 — Timeout

Force the API client to time out.

Verify:

    timeout
      ↓
    retry
      ↓
    success

### Failure drill 2 — 500 sequence

Return:

    500
    500
    200

Verify bounded retry behavior.

### Failure drill 3 — 429

Return:

    429 + Retry-After

Verify the extractor waits before continuing.

### Failure drill 4 — Malformed response

Return invalid JSON.

Verify the extraction fails without advancing the checkpoint.

### Failure drill 5 — Persistence failure

Allow the API request to succeed but make raw storage fail.

Verify:

    checkpoint does NOT advance

### Failure drill 6 — Worker/process interruption

Stop the extractor after several successful pages.

Restart it and verify safe recovery.

## 24. Recovery

When an API extraction fails:

1. Identify the extraction run.
2. Identify the endpoint and page/cursor.
3. Determine the HTTP failure class.
4. Check whether the error is transient.
5. Check rate-limit state.
6. Inspect the last durable checkpoint.
7. Verify which raw pages were persisted.
8. Resume from the correct state.
9. Check for duplicate records.
10. Reconcile expected and extracted counts.
11. Mark the extraction complete only after verification.

Never manually advance a checkpoint simply to make the pipeline appear successful.

## 25. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Requests** | Python HTTP client fundamentals: sessions, headers, timeouts and responses. |
| **HTTPX** | Modern Python HTTP client supporting synchronous and asynchronous workflows. |
| **Airbyte** | Production-oriented connector framework for extracting data from many external sources. |

> These tools provide production capabilities, but the extraction mechanism should be understood independently of them.

## 26. Production Runbook

### API extraction is failing

Check:

1. API availability.
2. HTTP status codes.
3. Authentication.
4. Timeout rate.
5. Rate-limit responses.
6. Recent API changes.
7. Last successful checkpoint.

### Extraction is incomplete

Check:

1. Pagination mechanism.
2. Last cursor/page.
3. Raw artifacts.
4. Source record counts.
5. API filtering.
6. Extraction state.

### Duplicate records appear

Check:

1. Retry behavior.
2. Pagination stability.
3. Source IDs.
4. Restart behavior.
5. Checkpoint ordering.

### API rate limits are reached

Check:

1. Request rate.
2. Worker concurrency.
3. Retry storm.
4. Retry-After handling.
5. Other clients using the same API credentials.

### What not to do

Do not:

- use infinite retries
- omit timeouts
- hard-code credentials
- advance checkpoints before persistence
- assume HTTP 200 means valid data
- assume every API uses the same pagination model
- log access tokens
- ignore 429 responses
- discard raw responses when recovery requirements need them

## 27. Common Mistakes

### Mistake 1 — Treating extraction as a single HTTP request

Real datasets are usually paginated or incremental.

### Mistake 2 — No timeout

A stuck request can hold a pipeline indefinitely.

### Mistake 3 — Infinite retries

Transient recovery becomes an outage amplifier.

### Mistake 4 — Advancing state too early

This can permanently skip data.

### Mistake 5 — No raw layer

Transformation failures become much harder to recover.

### Mistake 6 — Ignoring API rate limits

The API may throttle or block the extractor.

### Mistake 7 — No source identifier

Duplicate detection becomes unreliable.

### Mistake 8 — Treating API behavior as static

External APIs change.

## 28. Definition of Done

You are done when you can:

- explain the REST extraction lifecycle
- inspect an API contract before implementation
- authenticate safely
- configure HTTP timeouts
- classify HTTP responses
- implement pagination
- implement cursor pagination
- validate response structure
- preserve raw responses
- maintain extraction state
- implement safe checkpointing
- implement bounded retries
- handle 429 responses
- handle malformed responses
- deduplicate source records
- design incremental extraction
- test restart/recovery behavior
- prove checkpoint safety
- observe API extraction metrics
- intentionally break the extractor
- recover it without skipping data
- explain Requests, HTTPX and Airbyte
- operate an API extractor using a runbook

## 29. What You Learned

The central principle is:

> **API extraction is a stateful reliability problem, not an HTTP request problem.**

A production extraction flow is:

    API
      ↓
    REQUEST
      ↓
    RESPONSE VALIDATION
      ↓
    PAGINATION
      ↓
    RAW PERSISTENCE
      ↓
    VERIFICATION
      ↓
    CHECKPOINT
      ↓
    NEXT PAGE
      ↓
    COMPLETE

And the critical safety rule is:

    PERSIST SUCCESSFULLY
            ↓
       THEN ADVANCE
        CHECKPOINT

If you understand this mechanism, you can adapt it to many REST APIs even when the authentication, pagination, schema, and rate-limit rules differ.
