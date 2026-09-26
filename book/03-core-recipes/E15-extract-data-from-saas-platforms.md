# E15 — Extract Data from SaaS Platforms

SaaS platforms are common sources for operational data: CRM systems, payment platforms, ticketing systems, marketing systems, accounting tools, HR platforms, and many others.

SaaS extraction looks like API extraction, but production pipelines must also handle vendor-specific contracts, object relationships, authentication scopes, rate limits, pagination, incremental synchronization, deleted records, mutable records, and API-version changes.

The central question is:

> How do I reliably synchronize data from a SaaS platform without missing records, duplicating records, or silently drifting from the vendor's data model?

## 1. Problem Recognition

Typical SaaS extraction problems include:

- the API exposes many related resources
- authentication tokens expire
- the vendor limits request volume
- records are updated after their original creation
- records can be deleted
- pagination changes between endpoints
- API responses contain nested objects
- custom fields differ between accounts
- the vendor changes its schema
- incremental timestamps have ambiguous precision
- records arrive out of order
- historical records need backfilling
- the vendor API is temporarily unavailable
- an endpoint returns partial data
- different resources use different synchronization rules

The production boundary is:

```text
SAAS PLATFORM
      ↓
AUTHENTICATE
      ↓
DISCOVER CONTRACT
      ↓
EXTRACT RESOURCE
      ↓
VALIDATE
      ↓
PERSIST RAW / STAGING DATA
      ↓
ADVANCE SYNC STATE
      ↓
NEXT RESOURCE / PAGE
```

## 2. SaaS Is Not One Dataset

A SaaS platform usually exposes multiple related resources.

Example:

```text
CUSTOMERS
   ├── ORDERS
   ├── PAYMENTS
   └── ADDRESSES

PRODUCTS
   └── ORDER ITEMS
```

Do not assume one API endpoint represents the complete source.

Build a resource inventory first:

| Resource | Primary ID | Incremental Field | Deleted Records | Relationship |
|---|---|---|---|---|
| customers | customer_id | updated_at | yes/no | standalone |
| orders | order_id | updated_at | yes/no | customer_id |
| payments | payment_id | updated_at | yes/no | order_id |

The exact fields depend on the vendor.

## 3. Discover the Contract

Before implementing extraction, determine:

- authentication mechanism
- available scopes
- API version
- resource endpoints
- pagination method
- page-size limits
- rate limits
- timestamp semantics
- deletion semantics
- filtering capabilities
- sorting guarantees
- retryable status codes
- error response structure
- schema/versioning policy
- webhook or export alternatives

Do not assume every endpoint behaves like another endpoint on the same platform.

## 4. Authentication

SaaS APIs commonly use:

- API keys
- OAuth 2.0
- short-lived access tokens
- refresh tokens
- service accounts
- signed requests

Keep credentials outside source code.

```python
import os

CLIENT_ID = os.environ["SAAS_CLIENT_ID"]
CLIENT_SECRET = os.environ["SAAS_CLIENT_SECRET"]
REFRESH_TOKEN = os.environ["SAAS_REFRESH_TOKEN"]
```

Never log access tokens or refresh tokens.

## 5. OAuth Token Refresh

Short-lived tokens require refresh handling.

```text
ACCESS TOKEN
     ↓
EXPIRED
     ↓
REFRESH TOKEN
     ↓
NEW ACCESS TOKEN
     ↓
CONTINUE EXTRACTION
```

A token refresh failure is different from a transient API timeout. Classify authentication failures explicitly.

## 6. Resource-Specific Extraction

Make the resource being synchronized explicit.

```python
def extract_resource(client, resource, params):
    response = client.get(
        f"/{resource}",
        params=params,
    )

    response.raise_for_status()
    return response.json()
```

A production implementation should additionally validate the response contract, retry transient failures, and preserve synchronization state.

## 7. Pagination

SaaS APIs can use:

- page numbers
- offsets
- cursors
- continuation tokens
- next links
- time windows

Do not assume one pagination mechanism applies to the entire platform.

Generic loop:

```python
def paginate(fetch_page):
    cursor = None

    while True:
        page = fetch_page(cursor)

        for record in page["records"]:
            yield record

        cursor = page.get("next_cursor")

        if cursor is None:
            break
```

Detailed pagination mechanics are covered by E02. This recipe focuses on applying those mechanics to SaaS synchronization.

## 8. Incremental Synchronization

Full extraction is often expensive.

A common pattern is:

```text
FIRST RUN
   ↓
FULL HISTORY
   ↓
CHECKPOINT
   ↓
INCREMENTAL RUNS
   ↓
UPDATED RECORDS
```

Possible incremental fields include:

- updated_at
- modified_time
- sequence number
- cursor
- vendor-specific change token

Prefer a vendor-provided change cursor or sequence when it is reliable and supported.

## 9. Timestamp Watermarks

Suppose the last successful watermark is:

```text
2026-09-26T10:00:00Z
```

Do not blindly request:

```text
updated_at > 10:00:00Z
```

If timestamps have limited precision, records sharing the boundary timestamp may be skipped.

Use an overlap window when appropriate:

```text
LAST WATERMARK
      ↓
OVERLAP WINDOW
      ↓
RE-READ RECENT RECORDS
      ↓
DEDUPLICATE
      ↓
ADVANCE WATERMARK
```

The overlap size should be based on the vendor's timestamp precision and consistency behavior.

## 10. High-Watermark Safety

Do not advance the watermark merely because the API returned a page.

Safe sequence:

```text
REQUEST DATA
    ↓
VALIDATE
    ↓
PERSIST
    ↓
VERIFY
    ↓
ADVANCE WATERMARK
```

If persistence fails, the watermark must not move past unpersisted data.

## 11. Mutable Records

SaaS records are usually mutable.

Example:

```text
DAY 1
order status = pending

DAY 2
order status = completed
```

Extracting only newly created records can produce stale downstream state.

Incremental extraction should normally capture records that changed, not only records that were created.

## 12. Deleted Records

Deletion semantics vary widely.

Possible models:

- deleted flag
- deleted_at timestamp
- tombstone endpoint
- separate deleted-records endpoint
- webhook event
- no historical deletion information

Do not assume that absence from the current API means a record was deleted.

If deletion is important, explicitly document how the vendor exposes it.

## 13. Soft Deletes

Example:

```json
{
  "id": "customer-123",
  "deleted": true,
  "deleted_at": "2026-09-26T12:00:00Z"
}
```

Persist deletion state rather than silently dropping the record.

## 14. Hard Deletes

If the vendor permanently removes records and provides no deletion feed, incremental synchronization becomes more difficult.

Possible approaches include:

- periodic full reconciliation
- vendor change logs
- webhooks
- snapshot comparison
- separate deletion endpoints

Choose based on vendor capabilities and business requirements.

## 15. Raw Preservation

Preserve the vendor response before aggressive transformation when practical.

```text
SAAS API
   ↓
RAW RESPONSE
   ↓
STAGING
   ↓
CURATED MODEL
```

Raw preservation helps with debugging, replay, schema evolution, vendor disputes, and historical reconstruction.

Respect retention, privacy, and contractual requirements.

## 16. Custom Fields

Many SaaS platforms allow customer-defined fields.

Example:

```json
{
  "id": "customer-123",
  "name": "Example Ltd",
  "custom_fields": {
    "region_code": "EU",
    "account_segment": "enterprise"
  }
}
```

Custom fields may differ between tenants or accounts.

Do not hard-code a custom-field schema without understanding how the source manages it.

## 17. Multi-Tenant SaaS Extraction

A SaaS integration may serve multiple accounts:

```text
TENANT A → API
TENANT B → API
TENANT C → API
```

Each tenant may have:

- different credentials
- different rate limits
- different enabled resources
- different custom fields
- different data volumes

Keep tenant-specific synchronization state separate.

## 18. Tenant-Aware Checkpointing

Do not use one global watermark for unrelated tenants.

Prefer:

```text
(tenant_id, resource) → sync state
```

Example:

```sql
CREATE TABLE saas_sync_state (
    tenant_id TEXT NOT NULL,
    resource_name TEXT NOT NULL,
    cursor_value TEXT,
    updated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (tenant_id, resource_name)
);
```

## 19. Rate Limits

SaaS vendors frequently enforce request limits.

Your extractor needs:

- request pacing
- concurrency limits
- retry handling
- 429 handling
- provider-specific quota awareness

Do not assume every tenant or endpoint has the same limit.

## 20. Rate-Limit Response

A provider may return:

```text
HTTP 429
Retry-After: 30
```

When the provider supplies Retry-After, respect it when appropriate.

Generic strategy:

```text
REQUEST
  ↓
429
  ↓
WAIT
  ↓
RETRY
```

## 21. API Version Changes

SaaS vendors can deprecate API versions.

Track:

- API version
- endpoint version
- extraction code version
- source schema version where available

An API upgrade should be treated as a controlled pipeline change, not a casual configuration edit.

## 22. Schema Drift

Possible changes include:

- new fields
- removed fields
- renamed fields
- type changes
- enum changes
- nested-object changes

Validate required fields while allowing documented additive fields where appropriate.

## 23. Relationship Extraction

When resources reference one another, extraction order may matter.

Example:

```text
CUSTOMERS
   ↓
ORDERS
   ↓
PAYMENTS
```

But independent resources can often be extracted concurrently.

Use the actual relationship model rather than imposing unnecessary serial execution.

## 24. Webhooks Plus API Synchronization

Many SaaS systems provide both webhooks and APIs.

A useful architecture is:

```text
WEBHOOKS
   ↓
NEAR-REAL-TIME CHANGES
   ↓
RAW / STAGING

API
   ↓
INITIAL LOAD + RECONCILIATION
```

Webhooks alone may not provide complete historical coverage. API extraction can provide reconciliation and backfill.

## 25. API Extraction and Reconciliation

Periodic reconciliation can detect missing records:

```text
SAAS SOURCE COUNT / STATE
          ↓
COMPARE
          ↓
DESTINATION COUNT / STATE
          ↓
INVESTIGATE DIFFERENCES
```

Reconciliation is especially useful when the vendor does not provide strong deletion semantics.

## 26. Checkpoint Design

Checkpoint dimensions may include:

```text
tenant
resource
cursor
watermark
page state
API version
```

Example:

```sql
CREATE TABLE saas_checkpoint (
    tenant_id TEXT NOT NULL,
    resource_name TEXT NOT NULL,
    cursor_value TEXT,
    watermark TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (tenant_id, resource_name)
);
```

Only advance the checkpoint after the corresponding data is safely persisted.

## 27. Testing

Test at least:

1. First full synchronization.
2. Incremental synchronization.
3. Pagination.
4. Cursor expiration.
5. Access-token expiration.
6. Refresh-token failure.
7. Rate limiting.
8. API timeout.
9. Temporary vendor outage.
10. Permanent API error.
11. Duplicate record.
12. Mutable record.
13. Deleted record.
14. Missing deletion event.
15. Timestamp boundary.
16. Overlap-window deduplication.
17. Schema addition.
18. Schema type change.
19. Custom fields.
20. Multi-tenant extraction.
21. Tenant-specific rate limit.
22. Worker crash.
23. Checkpoint recovery.
24. API version change.
25. Reconciliation mismatch.

## 28. Example Unit Tests

```python
def test_overlap_window_allows_boundary_reread():
    previous = "2026-09-26T10:00:00Z"
    request_from = "2026-09-26T09:59:00Z"

    assert request_from < previous


def test_checkpoint_is_scoped_to_tenant_and_resource():
    state = {
        ("tenant-a", "customers"): "cursor-10",
        ("tenant-b", "customers"): "cursor-42",
    }

    assert (
        state[("tenant-a", "customers")]
        != state[("tenant-b", "customers")]
    )


def test_updated_record_is_not_treated_as_new_record():
    created_at = "2026-09-01T10:00:00Z"
    updated_at = "2026-09-26T10:00:00Z"

    assert updated_at > created_at
```

## 29. Observability

Track:

```text
saas_requests_total
saas_requests_failed_total
saas_records_received_total
saas_records_persisted_total
saas_records_duplicate_total
saas_records_deleted_total
saas_api_429_total
saas_auth_refresh_total
saas_auth_failures_total
saas_request_duration_seconds
saas_processing_duration_seconds
saas_checkpoint_advances_total
saas_reconciliation_mismatch_total
saas_cursor_expired_total
```

Useful log fields:

```text
tenant_id
resource
api_version
request_id
page_or_cursor
record_id
attempt
http_status
processing_status
error_type
```

Do not log access tokens or sensitive SaaS payloads by default.

## 30. Intentional Failure

### Failure 1 — Advance watermark too early

Advance the watermark before persistence and simulate a database failure. Observe the missed-data risk. Fix the ordering.

### Failure 2 — Token expiration

Expire the access token during extraction. Verify refresh and retry behavior.

### Failure 3 — Rate limiting

Force a 429 response. Verify the extractor respects the provider's retry guidance.

### Failure 4 — Duplicate record

Return the same record in two overlapping windows. Verify durable deduplication.

### Failure 5 — Mutable record

Change a record after its initial extraction. Verify incremental synchronization captures the update.

### Failure 6 — Deleted record

Delete a source record according to the vendor's deletion mechanism. Verify downstream deletion state is handled correctly.

### Failure 7 — Schema change

Add a field or change a documented field type. Verify contract validation detects the change appropriately.

### Failure 8 — Tenant isolation

Cause tenant A's synchronization to fail. Verify tenant B can continue independently when the architecture permits it.

## 31. Recovery

1. Identify tenant and resource.
2. Inspect API health and authentication state.
3. Check rate-limit status.
4. Inspect the last successful checkpoint.
5. Determine whether the affected page/window was persisted.
6. Re-run from the last safe checkpoint.
7. Use overlap windows where timestamp boundaries require them.
8. Deduplicate re-read records.
9. Reconcile source and destination state.
10. Investigate deleted records separately.
11. Verify the checkpoint after recovery.
12. Confirm subsequent incremental runs are healthy.

## 32. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **Airbyte** | Data integration platform with many SaaS connectors and incremental-sync patterns. |
| **Fivetran** | Managed integration platform focused heavily on SaaS and database connectors. |
| **Singer** | Open ecosystem and specification for tap/target-style extraction and loading. |

The objective is not to replace engineering understanding with a connector. Know what the connector is doing: authentication, pagination, incremental state, rate limiting, schema handling, and retries.

## 33. Production Runbook

### Incremental data is missing

Check:

1. Last successful checkpoint.
2. Timestamp precision.
3. Overlap window.
4. Vendor filtering semantics.
5. Pagination termination.
6. API-side ordering.
7. Deletion behavior.

### API requests are failing

Check:

1. Authentication.
2. Token expiration.
3. Permissions and scopes.
4. API version.
5. Vendor status.
6. Rate limits.
7. Request timeout.

### Duplicate records are increasing

Check:

1. Overlap-window size.
2. Deduplication key.
3. Checkpoint advancement.
4. Pagination cursor behavior.
5. Concurrent workers.

### One tenant is failing

Check:

1. Tenant credentials.
2. Tenant-specific scopes.
3. Tenant rate limit.
4. Tenant custom fields.
5. Tenant resource availability.

### What not to do

Do not use one global checkpoint for unrelated tenants. Do not assume every resource uses the same pagination. Do not advance watermarks before durable persistence. Do not assume absence means deletion. Do not ignore API-version deprecations. Do not log credentials.

## 34. Definition of Done

You are done when you can:

- discover a SaaS API contract
- identify resources and relationships
- authenticate safely
- handle token refresh
- paginate resources
- perform full synchronization
- perform incremental synchronization
- design safe timestamp watermarks
- handle mutable records
- understand deletion semantics
- handle custom fields
- isolate tenant synchronization state
- respect SaaS rate limits
- handle API version changes
- detect schema drift
- combine API extraction with webhooks when appropriate
- reconcile source and destination state
- design durable checkpoints
- intentionally break synchronization
- diagnose the failure
- recover without silently losing records

## 35. What You Learned

SaaS extraction is fundamentally about **synchronizing a mutable external system under an API contract**.

The core pattern is:

```text
DISCOVER CONTRACT
      ↓
AUTHENTICATE
      ↓
EXTRACT RESOURCE
      ↓
PAGINATE
      ↓
VALIDATE
      ↓
PERSIST
      ↓
ADVANCE CHECKPOINT
      ↓
RECONCILE
      ↓
REPEAT
```

The most important rule is:

> Treat the SaaS platform as a mutable external system, not as a static table exposed over HTTP.

Once that principle is understood, incremental synchronization, deletion handling, tenant isolation, rate limits, schema drift, reconciliation, and recovery become parts of the same extraction design.
