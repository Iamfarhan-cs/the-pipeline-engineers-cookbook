# T06 — Date and Time Transformation

> **Goal:** Convert date/time representations into explicit, correct, testable values without silently changing the meaning of an event.

Date and time transformation is not mainly a formatting problem. It is a **semantic conversion problem**.

A pipeline can successfully parse a timestamp and still corrupt data if it:

- treats local time as UTC
- confuses an event timestamp with a processing timestamp
- interprets epoch milliseconds as seconds
- converts a date-only value into a timestamp
- silently invents a missing business time
- truncates precision that downstream systems require
- applies a timezone conversion where none was requested
- accepts ambiguous or invalid source values without a contract.

By the end of this recipe, you should be able to recognize these failures, implement deterministic transformations, test boundary cases, observe transformation quality, intentionally break the implementation, and recover safely.

---

## 1. Problem Recognition

### 1.1 Typical source representations

A pipeline may receive the same business concept in many representations:

    2026-09-28
    2026-09-28T10:30:00
    2026-09-28T10:30:00Z
    2026-09-28T10:30:00+05:00
    28/09/2026 10:30
    2026-09-28 10:30:00.123456
    1727519400
    1727519400123

These values are not automatically interchangeable.

### 1.2 First question: what does the field mean?

Before transforming a value, classify it:

| Semantic type | Example | Correct interpretation |
|---|---|---|
| Calendar date | `2026-09-28` | A date, not an instant |
| Local time | `10:30:00` | A clock reading without a date/zone |
| Local datetime | `2026-09-28 10:30:00` | Date + clock time, timezone may be missing |
| Instant | `2026-09-28T05:30:00Z` | A point on the global timeline |
| Offset datetime | `2026-09-28T10:30:00+05:00` | An instant plus its stated offset |
| Epoch timestamp | `1727519400` | An instant encoded as a numeric count, if the unit is known |
| Processing timestamp | `2026-09-28T05:31:12Z` | When the pipeline processed the record |

The first engineering decision is therefore **semantic classification**, not parsing syntax.

### 1.3 Red flags

Investigate immediately when you see:

- multiple timestamp formats in one source
- timestamps with and without offsets
- integer timestamps whose unit is undocumented
- sudden changes in timestamp precision
- dates that move backward unexpectedly
- event times far in the future
- records shifted by exactly a few hours
- all records occurring at midnight
- a large number of parse failures after a source release
- code calling `datetime.now()` to fill a missing business timestamp.

---

## 2. Concept and Reasoning

## 2.1 Date, time, datetime, and instant are different concepts

A **date** answers: `Which calendar day?`

A **time** answers: `What clock time?`

A **datetime** combines a calendar date and clock time.

An **instant** identifies one point on a timeline.

A timezone or UTC offset provides the information needed to relate a local clock reading to an instant.

Do not add missing semantic information merely because the target schema expects a timestamp.

### 2.2 Naive versus timezone-aware datetimes

In Python, a datetime without timezone information is commonly called **naive**. A datetime carrying timezone information is **aware**.

    from datetime import datetime, timezone

    naive = datetime(2026, 9, 28, 10, 30)
    aware = datetime(2026, 9, 28, 10, 30, tzinfo=timezone.utc)

A naive datetime does not tell you which instant it represents.

That missing information must come from the source contract. Do not silently assume UTC unless the source contract explicitly says so.

### 2.3 Instant versus local business time

Consider a store opening time:

    09:00

That may be a local business rule rather than an instant.

Consider a payment authorization event:

    2026-09-28T04:00:00Z

That is an instant and can be compared globally.

The transformation strategy depends on the business meaning.

### 2.4 Event time versus processing time

Never confuse:

    event_time = when the source event happened
    ingestion_time = when your system received it
    processing_time = when your pipeline processed it

These timestamps answer different operational questions.

Keep them as separate fields when all three are useful.

### 2.5 Canonical storage

For events that represent instants, a common design is:

    source representation
          ↓
    parse
          ↓
    validate semantic contract
          ↓
    convert to canonical UTC instant
          ↓
    store

UTC normalization is appropriate when the business meaning is an instant. It is not a universal rule for every date/time field.

### 2.6 Preserve source evidence

When auditability matters, consider storing both:

- the canonical timestamp used for computation
- the original source representation.

Example:

    event_time_utc = 2026-09-28T05:30:00Z
    source_event_time = 2026-09-28T10:30:00+05:00

The canonical value supports computation; the source value preserves evidence.

### 2.7 ISO-style representations

Prefer an explicit machine-readable contract rather than accepting arbitrary human-formatted dates.

Good examples include:

    2026-09-28
    2026-09-28T05:30:00Z
    2026-09-28T10:30:00+05:00

Do not make a parser permissive simply because it can be.

### 2.8 Epoch units

Numeric timestamps require a documented unit.

Common units include:

| Unit | Example shape |
|---|---:|
| seconds | `1727519400` |
| milliseconds | `1727519400000` |
| microseconds | `1727519400000000` |
| nanoseconds | `1727519400000000000` |

Never infer the unit only from the number's size in production when the source contract can define it.

### 2.9 Date arithmetic

Date arithmetic must respect the semantic type.

Adding a fixed duration such as 24 hours is not always equivalent to moving to the same local clock time on the next calendar day when timezone transitions are involved.

That distinction becomes especially important when working with timezone-aware business schedules. Detailed timezone conversion belongs in T07.

### 2.10 Precision

Do not silently reduce precision.

    2026-09-28T05:30:00.123456Z

contains microsecond precision.

If the target only supports seconds, truncation is a schema decision and should be explicit.

---

## 3. Implementation

## 3.1 Establish a field contract

Before coding, define:

| Contract item | Example |
|---|---|
| Semantic type | event instant |
| Accepted input | RFC-style timestamp with offset |
| Canonical type | UTC-aware datetime |
| Precision | microseconds |
| Invalid input | reject/quarantine |
| Missing input | preserve NULL |
| Fallback | none |
| Output format | ISO 8601/RFC-style string at API boundary |

This prevents transformation code from becoming a collection of guesses.

## 3.2 Safe Python parser

    from datetime import datetime, timezone

    def parse_event_timestamp(value: str) -> datetime:
        if value is None:
            raise ValueError("event timestamp is required")

        text = value.strip()
        if not text:
            raise ValueError("event timestamp cannot be empty")

        # Accept a trailing Z as UTC.
        if text.endswith("Z"):
            text = text[:-1] + "+00:00"

        parsed = datetime.fromisoformat(text)

        if parsed.tzinfo is None:
            raise ValueError("event timestamp must include timezone information")

        return parsed.astimezone(timezone.utc)

    event_time = parse_event_timestamp("2026-09-28T10:30:00+05:00")
    print(event_time.isoformat())

The important behavior is not the parser call. It is the explicit rejection of an ambiguous timezone-less event timestamp.

## 3.3 Parse a date-only value as a date

    from datetime import date

    def parse_business_date(value: str) -> date:
        text = value.strip()
        if not text:
            raise ValueError("business date cannot be empty")

        return date.fromisoformat(text)

Do not do this merely to satisfy a timestamp column:

    datetime.fromisoformat("2026-09-28T00:00:00")

unless midnight is genuinely part of the business meaning.

## 3.4 Explicit epoch conversion

Keep the unit in the function interface.

    from datetime import datetime, timezone

    def epoch_seconds_to_utc(value: int | float) -> datetime:
        return datetime.fromtimestamp(value, tz=timezone.utc)

    def epoch_milliseconds_to_utc(value: int | float) -> datetime:
        return datetime.fromtimestamp(value / 1000, tz=timezone.utc)

This is safer than a generic function that silently guesses the unit.

## 3.5 Normalize a timestamp column

    def transform_record(record: dict) -> dict:
        output = dict(record)
        output["event_time"] = parse_event_timestamp(record["event_time"])
        return output

Keep transformation functions deterministic. A record transformation should normally produce the same result every time for the same input and configuration.

## 3.6 Separate current time from source time

Do not replace a missing source event time with the current clock:

    # Dangerous
    record["event_time"] = datetime.now(timezone.utc)

Instead, keep source absence explicit and add processing metadata separately:

    record["event_time"] = None
    record["processed_at"] = processing_clock.now()

## 3.7 Inject the clock

Code that depends on current time is easier to test when the clock is injectable.

    from datetime import datetime, timezone

    def process_record(record: dict, clock) -> dict:
        output = dict(record)
        output["processed_at"] = clock.now()
        return output

    class FixedClock:
        def __init__(self, value: datetime):
            self.value = value

        def now(self) -> datetime:
            return self.value

Test code can now use a fixed timestamp instead of depending on the machine clock.

## 3.8 PostgreSQL representation

Use PostgreSQL types according to semantics.

    CREATE TABLE events (
        event_id text PRIMARY KEY,
        event_time timestamptz,
        business_date date,
        processed_at timestamptz NOT NULL
    );

`date` is appropriate for calendar dates. `timestamptz` is appropriate for values representing instants.

## 3.9 Normalize at the boundary

A robust ETL flow is:

    source
      ↓
    raw representation
      ↓
    parse
      ↓
    semantic validation
      ↓
    canonical representation
      ↓
    downstream transformation
      ↓
    load

Do not allow every downstream consumer to implement its own timestamp interpretation.

---

## 4. Testing

Date/time transformations require boundary-oriented tests.

### 4.1 Basic tests

    def test_parse_utc_timestamp():
        result = parse_event_timestamp("2026-09-28T05:30:00Z")
        assert result.isoformat() == "2026-09-28T05:30:00+00:00"

    def test_parse_offset_timestamp():
        result = parse_event_timestamp("2026-09-28T10:30:00+05:00")
        assert result.isoformat() == "2026-09-28T05:30:00+00:00"

    def test_reject_naive_event_timestamp():
        try:
            parse_event_timestamp("2026-09-28T10:30:00")
            assert False
        except ValueError:
            pass

### 4.2 Date tests

    def test_business_date_is_not_datetime():
        result = parse_business_date("2026-09-28")
        assert result.isoformat() == "2026-09-28"

### 4.3 Epoch tests

    def test_epoch_seconds():
        result = epoch_seconds_to_utc(0)
        assert result.isoformat() == "1970-01-01T00:00:00+00:00"

    def test_epoch_milliseconds():
        result = epoch_milliseconds_to_utc(0)
        assert result.isoformat() == "1970-01-01T00:00:00+00:00"

### 4.4 Invalid dates

Test values such as:

    2026-02-30
    2026-13-01
    2026-00-10
    empty string
    whitespace-only string
    malformed timezone offset

These should fail according to the field contract rather than being silently repaired.

### 4.5 Leap-year boundaries

Test:

    2024-02-29
    2025-02-29
    2028-02-29

One is valid only in a leap year; another must be rejected.

### 4.6 Month and year boundaries

Test transformations around:

    2026-01-01
    2026-01-31
    2026-02-01
    2026-12-31
    2027-01-01

### 4.7 Precision

Verify whether your target contract preserves:

    2026-09-28T05:30:00.123456Z

Do not accidentally convert microseconds to seconds.

### 4.8 Idempotence

Canonicalization should be stable:

    canonicalize(canonicalize(value)) == canonicalize(value)

An already canonical UTC timestamp should remain unchanged.

### 4.9 Determinism

A transformation that depends only on source values and configuration should return the same result on replay.

If processing time is required, isolate it in a separate metadata field and inject the clock during tests.

---

## 5. Observability

Date/time transformations need operational evidence.

Track at least:

| Metric/signal | Why it matters |
|---|---|
| parse failures | Detect malformed source data |
| rejected naive timestamps | Detect missing timezone contracts |
| input format distribution | Detect source format changes |
| epoch-unit failures | Detect unit mismatches |
| future event count | Detect clock or timezone errors |
| unusually old event count | Detect stale/replayed data |
| timestamp conversion count | Measure transformation volume |
| precision-loss count | Detect schema-driven truncation |

Example structured log:

    {
      "event": "datetime_transform_failed",
      "field": "event_time",
      "reason": "missing_timezone",
      "source_system": "payments-api",
      "record_id": "evt_123"
    }

Do not log sensitive payloads merely to debug timestamp failures. Log identifiers and classification metadata that are safe under your data policy.

### 5.1 Format-distribution monitoring

If yesterday every source value contained `Z` and today many values lack an offset, the format distribution itself is an operational signal.

Monitor changes rather than assuming the source contract will remain stable forever.

---

## 6. Intentional Failure

The purpose of this section is to deliberately create the mistakes that commonly occur in production.

### Failure 1 — Treat local time as UTC

Take:

    2026-09-28T10:30:00+05:00

and incorrectly remove the offset while labeling the value UTC.

Expected symptom:

    The event shifts by five hours.

Diagnosis:

- compare source offset with canonical UTC
- inspect a known reference event
- compare event distributions before and after deployment.

### Failure 2 — Seconds versus milliseconds

Pass a millisecond value into a seconds-based parser.

Expected symptom:

- dates far outside the expected operating period
- overflow or range errors
- absurd future timestamps.

Diagnosis:

- inspect source documentation
- inspect representative raw values
- verify the configured unit explicitly.

### Failure 3 — Fill missing event time with `now()`

Replace a NULL source event time with the processing clock.

Expected symptom:

- events appear to occur at ingestion time
- event-time analytics become distorted
- replay changes historical event times.

Recovery:

- restore the source event timestamp
- keep processing time separate
- reprocess affected records from trusted raw data.

### Failure 4 — Convert every date to midnight

Convert `2026-09-28` to `2026-09-28T00:00:00Z` without a business rule.

Expected symptom:

- downstream users interpret a calendar date as an event instant
- timezone shifts can move the represented calendar day.

---

## 7. Recovery

### 7.1 Wrong timezone assumption

1. Stop or isolate the affected transformation version.
2. Identify the deployment window.
3. Determine the source timezone contract.
4. Compare raw source values with transformed values.
5. Calculate the affected record range.
6. Restore from raw evidence.
7. Apply the corrected transformation.
8. Reconcile counts and timestamp distributions.
9. Record the incident and prevention change.

### 7.2 Wrong epoch unit

1. Identify when the unit configuration changed.
2. Locate affected raw records.
3. Verify the intended unit from the source contract.
4. Recompute canonical timestamps from raw values.
5. Replace incorrect derived values.
6. Re-run downstream transformations if necessary.
7. Add a regression test for the unit.

### 7.3 Incorrect fallback timestamp

If missing event times were replaced with processing time:

1. Do not attempt to infer the original event time from the incorrect fallback.
2. Recover the trusted source records.
3. Reprocess with missing values preserved.
4. Keep processing metadata separate.
5. Reconcile affected aggregates and reports.

### 7.4 Precision loss

If a migration reduced timestamp precision:

1. Determine whether the source/raw layer still preserves the original value.
2. Restore from the highest-fidelity source available.
3. Recompute derived timestamps.
4. Verify downstream uniqueness and ordering assumptions.
5. Treat precision as part of the schema contract going forward.

---

## 8. Production Tools You Should Know

Only learn the tools after understanding the mechanism.

### 8.1 Python `datetime` and `zoneinfo`

Know how to use:

- `date`
- `time`
- `datetime`
- `timezone`
- `timedelta`
- `zoneinfo.ZoneInfo`
- `fromisoformat()`
- `timestamp()` / `fromtimestamp()`.

The important skill is understanding the semantics of the resulting object, not memorizing methods.

### 8.2 PostgreSQL

Know:

- `date`
- `timestamp`
- `timestamptz`
- timestamp precision
- `AT TIME ZONE`
- date/time arithmetic.

Understand what conversion means before writing SQL.

### 8.3 Pandas

For tabular transformations, know:

- `pd.to_datetime()`
- `errors="raise"` versus permissive error handling
- timezone-aware series
- explicit `unit=` for epoch values
- explicit format policies where appropriate.

Do not allow a dataframe convenience function to hide an ambiguous source contract.

---

## 9. Production Runbook

### Symptom: timestamps shifted by a fixed number of hours

Check:

1. Source timezone/offset.
2. Parser behavior.
3. UTC conversion.
4. Database type.
5. Any application-layer serialization.

### Symptom: timestamps are absurdly far in the future

Check:

1. Epoch unit.
2. Integer overflow/conversion.
3. Source release changes.
4. Seconds versus milliseconds.

### Symptom: all event times equal processing time

Check:

1. Missing-value defaults.
2. Fallback logic.
3. Transformation deployment.
4. Whether raw event time was overwritten.

### Symptom: calendar dates move backward or forward

Check:

1. Whether the field is actually date-only.
2. Whether midnight was introduced artificially.
3. Whether timezone conversion was applied.
4. Whether the target representation changed the calendar-day meaning.

### Symptom: parse failures suddenly increase

Check:

1. Source format distribution.
2. Recent upstream deployment.
3. Offset/precision changes.
4. New null or empty values.
5. Parser contract.

---

## 10. Common Mistakes

### Mistake 1 — Assuming every timestamp is UTC

UTC is a useful canonical representation for instants, but a timezone-less source value is not automatically UTC.

### Mistake 2 — Using the current time as a business fallback

Current processing time is not evidence of when an event happened.

### Mistake 3 — Guessing epoch units

Make the unit explicit in the contract and implementation.

### Mistake 4 — Converting dates to timestamps

A calendar date can lose its meaning when represented as midnight in an arbitrary timezone.

### Mistake 5 — Accepting every parser format

Permissive parsing can hide upstream contract violations.

### Mistake 6 — Ignoring precision

Milliseconds and microseconds can matter for ordering, deduplication, and event correlation.

### Mistake 7 — Mixing event and processing timestamps

Keep operational metadata separate from source business facts.

### Mistake 8 — Testing only ordinary dates

Most serious defects appear around boundaries, invalid inputs, precision changes, and timezone assumptions.

### Mistake 9 — Making transformations non-deterministic

Replay should not change historical event semantics.

### Mistake 10 — Fixing bad timestamps by guessing

Recover from trusted raw evidence instead of inventing a replacement value.

---

## 11. Definition of Done

T06 is complete when you can:

- distinguish date, local time, datetime, and instant
- explain naive versus timezone-aware datetime semantics
- define a field-level date/time contract
- parse explicit ISO-style timestamps safely
- reject ambiguous event timestamps when the contract requires an offset
- convert known epoch units explicitly
- preserve date-only semantics
- separate event time from ingestion/processing time
- preserve precision intentionally
- implement deterministic transformations
- test leap days and calendar boundaries
- test invalid and malformed values
- observe parsing and transformation failures
- intentionally reproduce timezone and epoch-unit failures
- recover incorrect derived timestamps from trusted raw data
- use PostgreSQL date/time types correctly
- use Python and dataframe tooling without hiding semantic ambiguity.

---

## 12. What You Learned

The central lesson is:

> **Date/time transformation is semantic normalization, not string formatting.**

A production Data Engineer should be able to answer before transforming a value:

1. What does this field mean?
2. Is it a date, local clock value, or instant?
3. If it is an instant, where is the timezone/offset information?
4. What precision does the contract require?
5. What happens when the value is missing or invalid?
6. Can the transformation be replayed deterministically?
7. Can I recover the original value from trusted source evidence?

If those answers are explicit, date/time transformations become predictable engineering rather than hidden data corruption.

---

## Next Recipe

**T07 — Time Zone Conversion**

T06 establishes correct date/time semantics. T07 will focus specifically on converting between time zones, including DST behavior, ambiguous/nonexistent local times, IANA timezone identifiers, and production-safe timezone policies.