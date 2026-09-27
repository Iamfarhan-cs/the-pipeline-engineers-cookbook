# Recipe 10 — Telemetry Event Validation

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 10  
> **Source implementation commit:** <code>3d6f45a</code> — <code>feat: add telemetry event validation</code>  
> **Source repository:** <code>devops</code>

## 1. What This Recipe Teaches

This recipe introduces the data-quality validation boundary of the Payment Telemetry Pipeline.

By completing it, you should understand how to:

- validate extracted telemetry events entirely in memory;
- mirror important database constraints inside application code;
- validate UUIDs, strings, versions, timestamps, and JSON properties;
- collect all validation errors instead of stopping at the first failure;
- represent invalid events without losing the original event;
- partition a batch into valid and invalid collections;
- keep validation deterministic and side-effect-free;
- place validation before staging/database mutation;
- design unit tests around every validation failure mode;
- understand how invalid events later flow toward quarantine.

The Stage 10 implementation creates <code>src/payment_telemetry/validation.py</code> with <code>validate_event()</code> and <code>validate_events()</code>, and establishes validation as the pipeline's core quality gate. fileciteturn18file0L3-L15

## 2. The Problem

Stage 6 introduced extraction of telemetry from the operational database, and Stage 9 introduced the pipeline-owned staging layer.

However, data read from a database is not automatically guaranteed to be suitable for downstream processing.

Malformed records can exist because of:

- database maintenance;
- out-of-band updates;
- legacy bugs;
- historical data that predates current constraints;
- operational mistakes.

If malformed events enter downstream analytics, machine-learning pipelines, or audit reporting, failures occur later and become harder to diagnose.

Stage 10 adds an explicit quality boundary:

~~~text
Operational source
      |
      v
Extraction
      |
      v
TelemetryEvent batch
      |
      v
Validation
      |
      +------ valid ------> downstream/staging
      |
      +------ invalid ----> quarantine
~~~

The source describes validation as a defense-in-depth boundary that catches and isolates bad data before it proceeds further. fileciteturn18file0L11-L15

## 3. Why Validate Inside the Pipeline?

Database constraints are important, but they are not sufficient as the only quality mechanism.

The pipeline should detect invalid data before attempting database mutations.

Without an explicit validation layer:

~~~text
Extract
  |
  v
Insert
  |
  X database constraint failure
  |
  v
harder batch recovery
~~~

With validation:

~~~text
Extract
  |
  v
Validate
  |
  +--> valid events
  |
  +--> invalid events + diagnostics
~~~

This gives the pipeline a controlled place to:

- inspect malformed events;
- preserve the original invalid event;
- collect deterministic error messages;
- route invalid data to quarantine later.

## 4. Target Architecture

~~~text
list[TelemetryEvent]
        |
        v
 validate_events()
        |
        +--> validate_event(event_1) --> Valid
        |
        +--> validate_event(event_2) --> InvalidEvent
        |
        +--> validate_event(event_3) --> Valid
        |
        v
BatchValidationResult
        |
        +--> valid_events
        |       |
        |       v
        |   downstream/staging
        |
        +--> invalid_events
                |
                v
             quarantine
~~~

The supplied Stage 10 implementation explicitly partitions batches into <code>valid_events</code> and <code>invalid_events</code>. fileciteturn18file0L59-L72

## 5. Step 1 — Define the Validation Contract

Before writing validation code, define exactly what a valid <code>TelemetryEvent</code> means.

The Stage 10 contract checks:

| Field | Validation |
|---|---|
| <code>event_id</code> | Must be a valid UUID |
| <code>user_id</code> | Must be a valid UUID |
| <code>event_name</code> | Must not be empty and must not exceed 100 characters |
| <code>event_version</code> | Must be greater than zero |
| <code>occurred_at</code> | Must be a <code>datetime</code> |
| <code>route</code> | Must not be empty and must not exceed 2048 characters |
| <code>properties</code> | Must be a JSON object represented by a <code>dict</code> |
| <code>received_at</code> | Must be a <code>datetime</code> |
| <code>app_version</code> | Must not be empty |

These rules are explicitly documented in the source implementation. fileciteturn18file0L30-L44

## 6. Step 2 — Mirror Database Constraints

The validation layer intentionally mirrors important database constraints.

For example:

~~~text
Database constraint
event_name <= 100
        |
        v
Pipeline validation
event_name <= 100
~~~

Likewise:

~~~text
route <= 2048
event_version > 0
properties must be an object
~~~

This prevents an event that is known to violate a staging constraint from reaching the database in the first place.

The source explicitly describes this as defensive mirroring of database constraints. fileciteturn18file0L49-L53

### Important Principle

Do not make validation rules broader or narrower accidentally.

If the pipeline validates a different contract from the database, the two quality boundaries can disagree.

## 7. Step 3 — Create the Validation Module

Create:

~~~text
src/payment_telemetry/validation.py
~~~

The module defines four core data structures/functions:

~~~text
ValidationResult
InvalidEvent
BatchValidationResult
validate_event()
validate_events()
~~~

The supplied implementation defines the first three as dataclasses/results and the final two as the validation API. fileciteturn18file0L30-L44

Keep this module focused on validation.

It should not:

- connect to PostgreSQL;
- write files;
- send network requests;
- mutate database state;
- perform quarantine writes.

## 8. Step 4 — Define ValidationResult

The event-level validation result contains:

~~~text
valid: bool
errors: list[str]
~~~

Conceptually:

~~~python
ValidationResult(
    valid=True,
    errors=[],
)
~~~

or:

~~~python
ValidationResult(
    valid=False,
    errors=[
        "event_id must be a valid UUID",
        "route must not be empty",
    ],
)
~~~

The source explicitly defines <code>ValidationResult</code> as a dataclass containing <code>valid</code> and <code>errors</code>. fileciteturn18file0L30-L34

## 9. Step 5 — Preserve Invalid Events

Invalid data should not simply disappear.

Define:

~~~text
InvalidEvent
    |
    +--> event
    +--> errors
~~~

The object contains the original <code>TelemetryEvent</code> plus its validation errors. fileciteturn18file0L31-L33

This is important because a later quarantine stage needs both:

1. the original event;
2. the reason it was rejected.

The validation layer therefore reports quality failures without destroying the evidence needed for downstream handling.

## 10. Step 6 — Define the Batch Result

A batch can contain both valid and invalid events.

Represent that explicitly:

~~~text
BatchValidationResult
    |
    +--> valid_events: list[TelemetryEvent]
    |
    +--> invalid_events: list[InvalidEvent]
~~~

The source defines exactly this structure. fileciteturn18file0L32-L44

This is better than failing the entire batch when one event is malformed.

For example:

~~~text
100 events extracted
        |
        v
95 valid
5 invalid
        |
        +--> 95 continue
        |
        +--> 5 receive diagnostics
~~~

## 11. Step 7 — Implement validate_event()

Implement:

~~~python
validate_event(event)
~~~

The function should evaluate the event against the validation contract.

The Stage 10 implementation evaluates rules in deterministic order:

1. <code>event_id</code> must be a valid UUID.
2. <code>user_id</code> must be a valid UUID.
3. <code>event_name</code> must not be empty.
4. <code>event_name</code> must not exceed 100 characters.
5. <code>event_version</code> must be greater than zero.
6. <code>occurred_at</code> must be a datetime.
7. <code>route</code> must not be empty.
8. <code>route</code> must not exceed 2048 characters.
9. <code>properties</code> must be a dictionary/JSON object.
10. <code>received_at</code> must be a datetime.
11. <code>app_version</code> must not be empty.

These exact rules and their order are documented by the source. fileciteturn18file0L34-L44

## 12. Step 8 — Validate UUIDs

Both event identity fields require UUID validation:

~~~text
event_id
user_id
~~~

The rule is not merely "value exists."

The value must be interpretable as a valid UUID.

This catches malformed identifiers before they move downstream.

Test cases should include invalid UUID strings for both fields.

The source explicitly includes a test for invalid UUIDs. fileciteturn18file0L108-L113

## 13. Step 9 — Validate Event Names

The event name must satisfy two rules:

~~~text
event_name != ""
len(event_name) <= 100
~~~

These are separate validation conditions.

That matters because an event could be:

- empty;
- non-empty but too long.

The source explicitly tests both empty and overlong event names. fileciteturn18file0L109-L113

## 14. Step 10 — Validate Event Version

The event version must be positive:

~~~text
event_version > 0
~~~

Therefore:

~~~text
1       -> valid
2       -> valid
0       -> invalid
-1      -> invalid
~~~

The source explicitly includes a test for non-positive event versions. fileciteturn18file0L111-L113

This establishes a simple invariant for versioned event contracts.

## 15. Step 11 — Validate Timestamps

Two fields must contain datetime instances:

~~~text
occurred_at
received_at
~~~

These are different timestamps with different meanings:

- <code>occurred_at</code> represents when the event occurred;
- <code>received_at</code> represents when the backend received it.

The Stage 10 validator checks that both are datetime values. fileciteturn18file0L39-L42

The source explicitly includes tests for invalid timestamps. fileciteturn18file0L114-L116

## 16. Step 12 — Validate Routes

The route must satisfy:

~~~text
route != ""
len(route) <= 2048
~~~

The pipeline is dealing with a frontend pathname, so the validation boundary checks that the route exists and stays within the documented size limit.

The source explicitly records both route checks and corresponding tests. fileciteturn18file0L40-L41 fileciteturn18file0L113-L114

## 17. Step 13 — Validate Properties as a JSON Object

The telemetry properties payload must be a dictionary:

~~~text
properties -> dict
~~~

A list/array or another non-object value is invalid.

This matches the expected JSON object contract.

The source explicitly defines the rule as "properties must be a JSON object" and documents a test for non-object properties. fileciteturn18file0L41-L42 fileciteturn18file0L113-L115

This is intentionally a structural check at this stage.

Do not expand Stage 10 into unrelated semantic validation that the source does not define.

## 18. Step 14 — Validate Application Version

The application version must not be empty:

~~~text
app_version != ""
~~~

Stage 8 introduced backend application-version persistence.

Stage 10 protects the downstream pipeline from records that contain no release/version value.

The source explicitly includes this rule and its corresponding test. fileciteturn18file0L42-L43 fileciteturn18file0L115-L117

## 19. Step 15 — Aggregate All Errors

Do not stop at the first invalid field.

For example:

~~~text
event_id = invalid
route = ""
app_version = ""
~~~

should produce multiple diagnostics:

~~~text
[
    "event_id must be a valid UUID",
    "route must not be empty",
    "app_version must not be empty"
]
~~~

The Stage 10 design explicitly requires accumulative error reporting. fileciteturn18file0L49-L52

### Why?

First-error-only validation creates an inefficient loop:

~~~text
Run 1 -> fix error A
Run 2 -> fix error B
Run 3 -> fix error C
~~~

Accumulation gives the operator the complete diagnostic set in one validation pass.

The source explicitly includes a multiple-error aggregation test. fileciteturn18file0L117-L118

## 20. Step 16 — Keep Error Ordering Deterministic

The validator evaluates rules in a strict order.

That means identical invalid input produces the same error sequence.

This matters for:

- reproducible tests;
- predictable quarantine reasons;
- debugging;
- operational diagnostics.

Do not use an unordered mechanism that makes error ordering nondeterministic.

The source explicitly describes deterministic validation ordering. fileciteturn18file0L34-L44

## 21. Step 17 — Implement validate_events()

Implement:

~~~python
validate_events(events)
~~~

The function processes the entire batch and returns:

~~~text
BatchValidationResult
~~~

Conceptually:

~~~text
for event in events:
    result = validate_event(event)

    if result.valid:
        add to valid_events
    else:
        add InvalidEvent(event, errors)
~~~

The source explicitly defines <code>validate_events()</code> as the batch-level partitioning function. fileciteturn18file0L44-L44

## 22. Empty Batch Behavior

An empty input is a valid pipeline condition.

The batch validator should produce empty collections rather than failing.

Conceptually:

~~~text
validate_events([])
        |
        v
BatchValidationResult(
    valid_events=[],
    invalid_events=[],
)
~~~

The Stage 10 test suite explicitly includes empty-input handling. fileciteturn18file0L118-L121

This keeps orchestration code simple because it can safely pass batches through the validator.

## 23. Architecture and Data Flow

The complete Stage 10 quality flow is:

~~~text
Stage 6 extractor
      |
      v
list[TelemetryEvent]
      |
      v
validate_events()
      |
      +----------------------+
      |                      |
      v                      v
valid_events          invalid_events
      |                      |
      v                      v
staging / downstream   quarantine later
~~~

The source explicitly identifies Stage 6 as upstream, Stage 9 staging as the valid-event destination, and Stage 13 quarantine as the later invalid-event destination. fileciteturn18file0L89-L92

## 24. Transaction Safety

Validation has no database transaction of its own.

It is computational work performed before database mutation.

The source explicitly states that validation runs prior to initiating database mutation transactions. fileciteturn18file0L77-L79

The boundary is:

~~~text
Extract
  |
  v
Validate
  |
  v
Begin database mutation work
~~~

This prevents invalid data from entering the database transaction path unnecessarily.

## 25. Why Pure Functions Matter

The Stage 10 validation module is deliberately side-effect-free.

It performs:

~~~text
input event
    |
    v
CPU/in-memory checks
    |
    v
result
~~~

It does not perform:

~~~text
database I/O
network I/O
file I/O
external service calls
~~~

The source explicitly describes the functions as pure, CPU-bound, deterministic, and safe for concurrent execution. fileciteturn18file0L49-L52

This makes validation:

- easy to unit test;
- easy to reason about;
- safe to rerun;
- independent of database availability.

## 26. Idempotency

Validation is naturally idempotent.

The same input event produces the same validation result.

For example:

~~~text
validate_event(event_X)
        |
        v
ValidationResult_A

validate_event(event_X)
        |
        v
ValidationResult_A
~~~

The source explicitly identifies this property and notes that validation does not mutate input objects. fileciteturn18file0L83-L85

This is valuable when batches are retried.

## 27. Database Schema

There is no new database schema in Stage 10.

The source explicitly records:

~~~text
Database schema: N/A
~~~

because validation is an in-memory engine. fileciteturn18file0L96-L99

Do not add a migration merely to implement this validation stage.

Persistent quarantine storage belongs to a later stage.

## 28. Tests

The Stage 10 implementation adds 13 validation tests and verifies the full suite with:

~~~text
21 passed in 0.12s
~~~

The source records this regression result and lists the validation tests. fileciteturn18file0L102-L121

### Validation test coverage

~~~text
test_validate_event_accepts_valid_event
test_validate_event_rejects_invalid_uuids
test_validate_event_rejects_empty_and_overlong_event_name
test_validate_event_rejects_non_positive_event_version
test_validate_event_rejects_empty_and_overlong_route
test_validate_event_rejects_non_object_properties
test_validate_event_rejects_invalid_timestamps
test_validate_event_rejects_empty_app_version
test_validate_event_collects_multiple_errors
test_validate_events_separates_valid_and_invalid_events
test_validate_events_accepts_all_valid_events
test_validate_events_collects_all_invalid_events
test_validate_events_handles_empty_input
~~~

These test names are directly documented in the supplied Stage 10 source. fileciteturn18file0L108-L121

## 29. Test Strategy

A good validation test suite should test both:

### Individual event behavior

~~~text
valid event
invalid UUID
invalid version
empty name
overlong name
empty route
overlong route
invalid properties
invalid timestamps
empty app_version
multiple simultaneous errors
~~~

### Batch behavior

~~~text
mixed valid + invalid batch
all valid batch
all invalid batch
empty batch
~~~

This mirrors the separation between <code>validate_event()</code> and <code>validate_events()</code>.

## 30. Environment Validation

The source records verification that Python's standard-library:

~~~text
uuid.UUID
datetime.datetime
~~~

type/format behavior is consistent across supported Python 3.13 runtimes. fileciteturn18file0L125-L127

This is relevant because the validator relies on standard Python types rather than database-specific validation.

## 31. Privacy and Data Protection

Validation errors should explain the structural problem without copying sensitive payload content into diagnostics.

For example, prefer:

~~~text
properties must be a JSON object
~~~

rather than:

~~~text
properties contained: <raw payload>
~~~

The source explicitly states that validation error strings are static descriptive messages and intentionally do not echo raw properties or user inputs. fileciteturn18file0L131-L133

This reduces the risk that sensitive data appears in logs, error records, or diagnostic output.

## 32. Common Implementation Mistakes

### Mistake 1 — Fail on the first error

This loses useful diagnostics.

Collect all detected errors for the event.

### Mistake 2 — Validate only database insert failures

Database constraints are a last line of defense, not the primary diagnostic mechanism.

Validate before mutation.

### Mistake 3 — Mix validation with database access

That makes tests slower and creates unnecessary coupling.

Keep validation pure and in memory.

### Mistake 4 — Drop invalid events

An invalid event should become an <code>InvalidEvent</code> containing the original event and errors.

Quarantine is a later responsibility.

### Mistake 5 — Return only a boolean

A boolean cannot explain why the event failed.

Return structured diagnostics.

### Mistake 6 — Make error ordering nondeterministic

Stable rule ordering makes tests and operations reproducible.

### Mistake 7 — Echo raw properties in errors

Error messages should remain static and descriptive.

### Mistake 8 — Add database tables in Stage 10

The supplied Stage 10 implementation is entirely in memory.

Persistent quarantine belongs to Stage 13.

### Mistake 9 — Expand the contract without evidence

Do not invent extra validation rules merely because they sound useful.

Implement the documented Stage 10 contract first.

## 33. Independent Implementation Exercise

Build an in-memory validation engine for another event pipeline.

Requirements:

1. Define an event-level validation result.
2. Define an invalid-event structure containing the original event and errors.
3. Define a batch result separating valid and invalid events.
4. Validate event identifiers.
5. Validate required strings and their maximum lengths.
6. Validate positive event versions.
7. Validate timestamp types.
8. Validate structured properties as JSON objects.
9. Validate application/release version presence.
10. Aggregate every error found for an event.
11. Keep validation order deterministic.
12. Keep validation free of I/O.
13. Keep validation idempotent.
14. Handle empty batches.
15. Write unit tests for every failure mode.
16. Verify that mixed batches partition correctly.

Then construct a batch containing:

~~~text
2 valid events
2 invalid events
~~~

Verify that:

~~~text
valid_events   == 2
invalid_events == 2
~~~

and that every invalid event retains its original event plus its complete error list.

## 34. Validation Checklist

### Validation contract

- [ ] <code>event_id</code> is validated as a UUID.
- [ ] <code>user_id</code> is validated as a UUID.
- [ ] <code>event_name</code> cannot be empty.
- [ ] <code>event_name</code> cannot exceed 100 characters.
- [ ] <code>event_version</code> must be greater than zero.
- [ ] <code>occurred_at</code> must be a datetime.
- [ ] <code>route</code> cannot be empty.
- [ ] <code>route</code> cannot exceed 2048 characters.
- [ ] <code>properties</code> must be a dictionary/JSON object.
- [ ] <code>received_at</code> must be a datetime.
- [ ] <code>app_version</code> cannot be empty.

### Validation behavior

- [ ] Validation is deterministic.
- [ ] All errors for an event are collected.
- [ ] Invalid events retain their original event.
- [ ] Batches are split into valid and invalid collections.
- [ ] Empty input is handled safely.
- [ ] Validation performs no I/O.
- [ ] Validation does not mutate events.

### Tests

- [ ] Valid event test passes.
- [ ] Invalid UUID test passes.
- [ ] Event-name boundary tests pass.
- [ ] Event-version test passes.
- [ ] Route boundary tests pass.
- [ ] Properties-type test passes.
- [ ] Timestamp tests pass.
- [ ] App-version test passes.
- [ ] Multiple-error test passes.
- [ ] Mixed-batch test passes.
- [ ] All-valid batch test passes.
- [ ] All-invalid batch test passes.
- [ ] Empty-batch test passes.
- [ ] Full regression suite passes.

### Privacy

- [ ] Error messages do not echo raw properties.
- [ ] Error messages do not echo user input.
- [ ] Diagnostic output contains only static descriptive messages.

## 35. Scope Boundaries

### Implemented in Stage 10

- in-memory event validation;
- deterministic validation order;
- multi-error aggregation;
- event-level validation results;
- batch partitioning;
- unit test coverage.

### Deferred to Stage 11

- integration into batch orchestration;
- checkpointing.

### Deferred to Stage 13

- persistent quarantine storage for invalid events.

The supplied source explicitly defines these stage boundaries. fileciteturn18file0L137-L143

## 36. What You Should Understand Before the Next Recipe

Stage 10 adds the pipeline's quality gate:

~~~text
Extract
  |
  v
Validate
  |
  +--> valid events
  |
  +--> invalid events + diagnostics
~~~

The key lessons are:

- database constraints should be mirrored where they provide useful early validation;
- validation should happen before database mutation;
- pure functions make quality checks deterministic and testable;
- invalid events should be preserved rather than discarded;
- collecting all errors is more operationally useful than first-error failure;
- batch validation should partition valid and invalid events explicitly;
- validation itself does not require a database table;
- quarantine and orchestration remain separate pipeline stages.

The supplied Stage 10 implementation reports 21 passing tests and 13 dedicated validation tests. fileciteturn18file0L102-L121

## Source Traceability

This recipe is derived from the supplied Stage 10 implementation record:

- Source commit: <code>3d6f45a</code>
- Subject: <code>feat: add telemetry event validation</code>
- Repository: <code>devops</code>
- Overview and technical objective: fileciteturn18file0L3-L15
- Implementation sequence: fileciteturn18file0L19-L24
- Validation structures and rules: fileciteturn18file0L28-L44
- Design decisions: fileciteturn18file0L49-L53
- Architecture and data flow: fileciteturn18file0L57-L72
- Transaction safety and idempotency: fileciteturn18file0L77-L85
- Pipeline integration: fileciteturn18file0L89-L92
- Tests and environment validation: fileciteturn18file0L102-L127
- Privacy and scope boundaries: fileciteturn18file0L131-L143
