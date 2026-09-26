# E03 — Extract Data from GraphQL APIs

GraphQL changes the shape of API extraction.

Instead of requesting a fixed server-defined resource representation, the client sends a query describing the fields it needs. A production extractor therefore has to reason about query design, variables, pagination connections, partial errors, schema changes, response validation, and extraction state.

This recipe focuses specifically on GraphQL extraction mechanics. E01 covers general API extraction and E02 covers pagination mechanics; this recipe applies those reliability principles to GraphQL.

## 1. Problem Recognition

A GraphQL endpoint may return:

    {
        "data": {
            "payments": {
                "nodes": [...]
            }
        }
    }

But GraphQL responses can also contain data plus errors.

This creates an important distinction:

> An HTTP 200 response does not automatically mean the extraction succeeded.

Common production problems include:

- HTTP 200 with GraphQL errors
- requested fields missing from the response
- nullable fields appearing unexpectedly
- query shape becoming invalid after a schema change
- pagination cursors being mishandled
- partial data being persisted as complete data
- queries requesting too many fields
- nested objects multiplying the extracted data unexpectedly
- rate limits or query-cost limits
- authentication failures
- schema changes breaking extraction

The production problem is:

> How do I extract GraphQL data completely and safely while treating the GraphQL response contract—not just HTTP status—as the source of truth?

## 2. Concept and Reasoning

A GraphQL request normally contains:

    query
    variables
    operation_name

Example:

    query Payments($limit: Int!) {
        payments(limit: $limit) {
            id
            amount
            currency
        }
    }

Variables should normally be separated from the query.

    {
        "limit": 100
    }

This improves reuse, testing, safety, observability, and query management.

The extraction pipeline becomes:

    QUERY
      ↓
    VARIABLES
      ↓
    HTTP REQUEST
      ↓
    HTTP VALIDATION
      ↓
    GRAPHQL ERROR VALIDATION
      ↓
    DATA SHAPE VALIDATION
      ↓
    PERSIST
      ↓
    CHECKPOINT

## 3. HTTP Success Is Not GraphQL Success

A common mistake is checking only the HTTP layer.

A GraphQL response can be:

    HTTP 200
       ↓
    GraphQL errors
       ↓
    incomplete or unusable data

Therefore validate both layers.

    HTTP status
         +
    GraphQL errors
         +
    expected data structure
         =
    extraction success

## 4. Basic GraphQL Request

A minimal Python client can use Requests:

    import requests

    query = """
    query Payments($limit: Int!) {
        payments(limit: $limit) {
            id
            amount
            currency
        }
    }
    """

    variables = {
        "limit": 100,
    }

    response = requests.post(
        "https://api.example.com/graphql",
        json={
            "query": query,
            "variables": variables,
            "operationName": "Payments",
        },
        timeout=30,
    )

    response.raise_for_status()

    payload = response.json()

    if payload.get("errors"):
        raise RuntimeError(
            f"GraphQL errors: {payload['errors']}"
        )

    data = payload.get("data")

    if data is None:
        raise RuntimeError("GraphQL response has no data")

## 5. Query Design

Only request fields required by the pipeline.

Avoid requesting an entire object when the pipeline needs only:

    id
    amount
    currency
    status

Requesting unnecessary fields can increase:

- response size
- query execution cost
- network usage
- parsing cost
- memory usage

A production query should have a clear purpose.

## 6. Operation Names

Give extraction queries explicit operation names.

Example:

    query PaymentsForExtraction($limit: Int!) {
        ...
    }

Operation names help with:

- logs
- debugging
- server-side tracing
- query identification
- incident investigation

Log the operation name rather than storing the entire query in normal operational metrics.

## 7. Variables

Do not construct dynamic queries by concatenating runtime values into query strings.

Prefer a stable query with variables.

    query Payments($status: PaymentStatus!) {
        payments(status: $status) {
            id
        }
    }

with:

    variables = {
        "status": "COMPLETED"
    }

Variables separate query structure from runtime parameters.

## 8. Response Structure

GraphQL responses commonly contain:

    {
        "data": {...},
        "errors": [...]
    }

The data object follows the query's selection structure.

Therefore validation should check the expected path:

    data
      ↓
    payments
      ↓
    nodes

A response containing data does not prove that data.payments.nodes exists.

## 9. GraphQL Errors

Errors may include:

    {
        "message": "Field not found",
        "path": ["payments", "currency"],
        "extensions": {...}
    }

Treat errors according to their meaning.

Possible categories include:

- authentication/authorization
- validation/query errors
- resolver errors
- temporary server failures
- rate/query-cost failures
- business-domain errors

Do not automatically retry every GraphQL error.

A malformed query should not be retried indefinitely.

## 10. Partial Data

GraphQL can return both data and errors.

Whether partial data is usable depends on the extraction contract.

For a critical dataset, a safe policy may be:

    required extraction field fails
        ↓
    reject page

For an optional field:

    optional field fails
        ↓
    record warning
        ↓
    continue

Never silently discard GraphQL errors.

## 11. Required vs Optional Fields

Separate fields into:

### Required

The pipeline cannot process the record without them.

Example:

    id
    occurred_at
    amount

### Optional

The pipeline can continue without them.

Example:

    description
    metadata
    optional_reference

This distinction determines whether a response with partial errors can be accepted.

## 12. GraphQL Pagination

GraphQL frequently uses connection-style pagination.

Example:

    payments {
        edges {
            node {
                id
                amount
            }
            cursor
        }
        pageInfo {
            hasNextPage
            endCursor
        }
    }

The extraction loop is:

    request
       ↓
    process edges
       ↓
    read pageInfo
       ↓
    hasNextPage?
       ↓ yes
    use endCursor
       ↓
    request next page

This is a specialized application of the pagination principles from E02.

## 13. Cursor-Based GraphQL Extraction

Conceptually:

    cursor = null

    while True:

        payload = request(
            variables={
                "first": 100,
                "after": cursor,
            }
        )

        validate_graphql_response(payload)

        connection = payload["data"]["payments"]

        for edge in connection["edges"]:
            persist(edge["node"])

        page_info = connection["pageInfo"]

        if not page_info["hasNextPage"]:
            break

        next_cursor = page_info["endCursor"]

        if next_cursor == cursor:
            raise RuntimeError(
                "GraphQL cursor did not advance"
            )

        cursor = next_cursor

The cursor should be treated as opaque.

## 14. Edges vs Nodes

GraphQL APIs may expose:

    nodes

or:

    edges {
        node
        cursor
    }

If edge metadata is required, use edges.

If only records are required and the API supports nodes, nodes may be simpler.

Choose based on the API contract.

## 15. Nested Data

GraphQL makes nested extraction convenient.

Example:

    payment
      ↓
    customer
      ↓
    account
      ↓
    country

But nested extraction can create:

- repeated data
- very large responses
- one-to-many expansion
- difficult relational modeling
- unexpected nulls

Decide the target model before writing a large query.

Separate entities may be easier to load than deeply nested duplicated structures.

## 16. Query Complexity

GraphQL servers may enforce:

- maximum query depth
- query complexity
- response-size limits
- execution time limits
- rate limits

A query that is syntactically valid can still be operationally unacceptable.

Prefer smaller extraction queries when the dataset is large.

## 17. Schema Discovery

GraphQL has a schema describing available types and fields.

Before implementation, identify:

- query operation
- required arguments
- field types
- nullability
- pagination model
- connection fields
- error behavior
- authentication
- rate/query-cost rules
- incremental filter fields

Do not treat the schema as permanently immutable.

## 18. Schema Changes

Possible changes include:

- field removed
- field renamed
- field type changed
- field becomes nullable
- argument changes
- enum value changes
- pagination structure changes

A query can fail before returning data if a requested field no longer exists.

Protect extraction with:

- query tests
- schema compatibility checks where appropriate
- response validation
- versioned extraction definitions
- controlled deployment

## 19. Incremental GraphQL Extraction

GraphQL can support incremental extraction when the schema exposes suitable filters.

Example:

    updatedAt > watermark

The pipeline becomes:

    last_watermark
          ↓
    GraphQL filter
          ↓
    paginate
          ↓
    persist
          ↓
    reconcile
          ↓
    advance watermark

Do not advance the watermark until all required pages have been successfully persisted.

## 20. Checkpointing

A useful checkpoint may contain:

    extraction_run_id
    operation_name
    query_version
    filter_boundary
    cursor
    last_successful_page
    updated_at

The checkpoint must represent durable progress.

Correct order:

    REQUEST
       ↓
    VALIDATE
       ↓
    PERSIST
       ↓
    VERIFY
       ↓
    CHECKPOINT

## 21. Raw Response Preservation

For difficult integrations, preserve the raw response or an appropriate immutable extraction artifact before transformation.

This helps investigate:

- schema changes
- missing fields
- unexpected GraphQL errors
- pagination bugs
- source behavior

Do not retain sensitive data unnecessarily. Apply the pipeline's data-retention and privacy requirements.

## 22. Testing

Test at least:

1. Successful query.
2. HTTP 401/403.
3. HTTP 429.
4. HTTP 500.
5. Invalid JSON.
6. GraphQL query validation error.
7. GraphQL response with errors.
8. GraphQL response with partial data.
9. Missing required data path.
10. Missing optional field.
11. Single pagination page.
12. Multiple cursor pages.
13. Repeated cursor.
14. hasNextPage=false.
15. Missing endCursor.
16. Duplicate records across pages.
17. Schema/type mismatch.
18. Incremental extraction boundary.
19. Persistence failure before checkpoint.
20. Restart from a saved cursor.

## 23. Observability

Track metrics such as:

    graphql_requests_total
    graphql_request_failures_total
    graphql_errors_total
    graphql_partial_responses_total
    graphql_rate_limit_total
    graphql_request_duration_seconds
    graphql_pages_completed_total
    graphql_records_extracted_total
    graphql_pagination_failures_total
    graphql_query_validation_failures_total

Useful log fields:

    extraction_run_id
    operation_name
    endpoint
    page_number
    cursor_state
    record_count
    response_status
    graphql_error_count
    duration
    attempt

Avoid putting raw query text, secrets, tokens, or sensitive variables into normal logs.

## 24. Intentional Failure

### Failure drill 1 — HTTP 200 with GraphQL errors

Return HTTP 200 with an errors array.

Verify the extractor does not report success automatically.

### Failure drill 2 — Partial data

Return usable data plus an error affecting a required field.

Verify the configured policy rejects the affected extraction.

### Failure drill 3 — Repeated cursor

Return the same endCursor.

Verify the paginator stops safely.

### Failure drill 4 — Missing data path

Remove the expected connection from the response.

Verify response validation fails.

### Failure drill 5 — Query schema change

Remove a requested field from the test schema.

Verify the extraction test detects the incompatible query.

### Failure drill 6 — Persistence failure

Fail persistence after receiving a valid page.

Verify the cursor checkpoint does not advance.

## 25. Recovery

When GraphQL extraction fails:

1. Identify the extraction run.
2. Identify the operation.
3. Determine whether the failure occurred at HTTP, GraphQL, validation, persistence, or checkpoint level.
4. Inspect GraphQL errors.
5. Verify the last durable cursor.
6. Check whether the source schema changed.
7. Check rate/query-cost limits.
8. Resume from the last safe checkpoint where possible.
9. Reconcile extracted records.
10. Complete only after required data is verified.

Do not retry malformed queries indefinitely.

## 26. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **GraphQL Python** | Python client library for constructing and executing GraphQL operations. |
| **Ariadne** | Useful for understanding GraphQL schemas and server-side GraphQL behavior. |
| **Apollo** | Widely used GraphQL ecosystem for schemas, operations, and observability concepts. |

> These are reference tools for recognition and production vocabulary. The underlying GraphQL extraction mechanics should still be understood independently.

## 27. Production Runbook

### HTTP 200 but extraction failed

Check:

1. GraphQL errors.
2. Required data paths.
3. Partial-data policy.
4. Response validation.

### Query suddenly fails

Check:

1. Schema changes.
2. Removed or renamed fields.
3. Argument changes.
4. Query version.
5. Deployment changes.

### Pagination stops unexpectedly

Check:

1. hasNextPage.
2. endCursor.
3. Cursor advancement.
4. Connection structure.
5. API filters.

### Query is rejected or throttled

Check:

1. Rate limits.
2. Query complexity.
3. Query depth.
4. Response size.
5. Page size.
6. Request frequency.

### Duplicate records appear

Check:

1. Cursor handling.
2. Retry behavior.
3. Source mutation.
4. Stable source identifiers.
5. Downstream uniqueness constraints.

### What not to do

Do not:

- treat HTTP 200 as complete success
- ignore GraphQL errors
- retry invalid queries forever
- expose secrets or variables in logs
- assume nested data is cheap
- modify opaque cursors
- advance checkpoints before persistence
- request every field simply because GraphQL permits it

## 28. Common Mistakes

### Mistake 1 — Only checking HTTP status

GraphQL has its own error layer.

### Mistake 2 — Ignoring partial responses

Partial data can be incomplete for the pipeline's purpose.

### Mistake 3 — Building huge queries

Large nested queries increase operational risk.

### Mistake 4 — No cursor safety

A repeated cursor can cause infinite extraction.

### Mistake 5 — No schema compatibility testing

A field change can break extraction immediately.

### Mistake 6 — Logging sensitive variables

GraphQL variables can contain credentials, identifiers, filters, or other sensitive information.

### Mistake 7 — Advancing state too early

A successful response is not enough; durable persistence must happen first.

## 29. Definition of Done

You are done when you can:

- explain how GraphQL differs from REST extraction
- construct a GraphQL query with variables
- use explicit operation names
- validate both HTTP and GraphQL response layers
- distinguish complete, partial, and failed responses
- classify GraphQL errors
- identify required versus optional fields
- extract connection-based pagination
- safely advance cursors
- detect repeated cursors
- extract nested data deliberately
- reason about query complexity
- investigate a GraphQL schema
- handle schema changes
- combine GraphQL pagination with incremental extraction
- checkpoint GraphQL extraction safely
- test partial failures
- intentionally break the extractor
- recover from failed extraction
- operate GraphQL extraction using a production runbook

## 30. What You Learned

The central principle is:

> A GraphQL HTTP response is not automatically a successful data extraction.

A production GraphQL extractor validates:

    HTTP RESPONSE
          ↓
    GRAPHQL ERRORS
          ↓
    EXPECTED DATA SHAPE
          ↓
    PAGINATION STATE
          ↓
    PERSISTENCE
          ↓
    CHECKPOINT

For every GraphQL integration, ask:

> What exactly proves that this response contains the complete data required by my pipeline?

That question drives query design, error handling, pagination, validation, checkpointing, and recovery.
