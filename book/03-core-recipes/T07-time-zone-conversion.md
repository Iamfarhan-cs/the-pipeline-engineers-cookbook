# T07 — Time Zone Conversion

> **Goal:** Convert local times and instants between time zones without creating ambiguous, nonexistent, or silently shifted business timestamps.

Time zone conversion is where a seemingly correct timestamp transformation can become a production data correctness problem.

T06 established the distinction between dates, local datetimes, and instants. This recipe goes deeper into the specific problem of **mapping those values across time zones**.

A production pipeline must distinguish:

- an instant from its display timezone
- a UTC offset from a named timezone
- a local wall-clock time from a globally unique instant
- a fixed offset from an IANA timezone
- a valid local time from an ambiguous or nonexistent local time.

---

## 1. Problem Recognition

### 1.1 Common production symptoms

Investigate timezone handling when you see:

- records shifted by exactly one or more hours
- daily reports containing 23 or 25 hours
- duplicate local times around daylight-saving transitions
- local times that never occurred
- jobs firing at the wrong local business hour
- dates changing after UTC conversion
- different services disagreeing about the same event time
- one region behaving differently from another despite identical code.

### 1.2 The key distinction

These are not equivalent:

    UTC+05:00
    Asia/Karachi

`UTC+05:00` is a fixed offset.

`Asia/Karachi` is a named timezone whose rules are defined by timezone data and may encode historical or future rule changes.

For business locations, prefer the named IANA timezone when the source contract identifies a location rather than merely an offset.

### 1.3 Example

Suppose a source sends:

    2026-09-28 10:00

Without timezone information, this does not identify a unique instant.

The same wall-clock value could represent different instants in different locations.

Before conversion, you need a contract such as:

    local time + Asia/Karachi

or:

    local time + Europe/Berlin

Only then can the local value be interpreted as an instant.

---

## 2. Concept and Reasoning

## 2.1 Instant, offset, and timezone

### Instant

An instant is one point on a global timeline.

Example:

    2026-09-28T05:00:00Z

### Offset

An offset describes the difference between local clock time and UTC at a particular moment.

Example:

    +05:00

### Named timezone

A named timezone identifies a set of regional rules.

Example:

    Europe/Berlin

The timezone rules determine the applicable offset for a given local date/time.

### 2.2 Conversion versus localization

These operations must not be confused.

**Convert an aware instant:**

    UTC instant
        ↓
    display in Europe/Berlin

**Interpret a naive local time:**

    local wall-clock value
        ↓
    apply a known timezone
        ↓
    obtain an instant

The second operation is more dangerous because some local times are ambiguous or nonexistent.

### 2.3 Fixed offsets are not timezone identities

Do not replace a named timezone with today's observed offset.

Bad abstraction:

    user_timezone = "+01:00"

Better when the user has a regional timezone:

    user_timezone = "Europe/Paris"

A fixed offset may be correct for a particular event, but it does not preserve the rules of a location.

### 2.4 DST creates two special cases

Daylight-saving transitions can create:

**Ambiguous local times** — the same wall-clock time occurs twice.

**Nonexistent local times** — a range of wall-clock times is skipped.

For example, when clocks move backward, a local hour can occur twice. When clocks move forward, a local hour can disappear.

Your pipeline must define what to do instead of silently guessing.

### 2.5 Timezone conversion should normally operate on instants

For event data, a robust pattern is:

    source timestamp with offset
              ↓
       parse as aware datetime
              ↓
          canonical UTC
              ↓
      convert for consumer

Do not repeatedly convert already-converted local strings across pipeline stages.

Store canonical event semantics once and derive presentation values later.

### 2.6 Calendar date is not automatically portable

Consider:

    2026-09-28 00:30 Europe/Berlin

Converting that instant to another timezone can produce the previous calendar date.

That is not necessarily an error.

It is an expected consequence of representing one instant in another local calendar.

If the business meaning is specifically **the Berlin business date**, do not replace it with a timezone-converted instant and assume the calendar date remains unchanged.

### 2.7 Business timezone versus user timezone

Define ownership explicitly.

Examples:

- transaction settlement date → settlement location/business rule
- user notification → recipient's configured timezone
- warehouse ingestion timestamp → system/UTC convention
- store opening schedule → store's business timezone.

The timezone should come from the domain contract, not from whatever timezone happens to be configured on the worker machine.

---

## 3. Implementation

## 3.1 Use IANA timezone identifiers

Python provides timezone rules through `zoneinfo`.

    from datetime import datetime, timezone
    from zoneinfo import ZoneInfo

    event_time = datetime(2026, 9, 28, 5, 0, tzinfo=timezone.utc)
    berlin_time = event_time.astimezone(ZoneInfo("Europe/Berlin"))

    print(berlin_time.isoformat())

`astimezone()` converts an already-aware instant to another timezone.

## 3.2 Convert UTC to a business timezone

    from datetime import datetime, timezone
    from zoneinfo import ZoneInfo

    utc_event = datetime(2026, 9, 28, 12, 0, tzinfo=timezone.utc)
    karachi_event = utc_event.astimezone(ZoneInfo("Asia/Karachi"))

The instant remains the same. Only its local representation changes.

## 3.3 Verify the invariant

A timezone conversion should preserve the instant.

    original = datetime(2026, 9, 28, 12, 0, tzinfo=timezone.utc)
    converted = original.astimezone(ZoneInfo("Europe/Berlin"))

    assert converted.astimezone(timezone.utc) == original

This is one of the most useful correctness tests for timezone conversion.

## 3.4 Interpret a local datetime

A source may provide:

    2026-09-28 10:00

plus a separately configured timezone:

    Asia/Karachi

You can construct an aware datetime:

    local_time = datetime(
        2026, 9, 28, 10, 0,
        tzinfo=ZoneInfo("Asia/Karachi")
    )

    instant = local_time.astimezone(timezone.utc)

This is only safe when the source contract establishes that the local value belongs to that timezone.

## 3.5 Do not attach UTC to arbitrary local time

This is dangerous:

    naive = datetime(2026, 9, 28, 10, 0)
    wrong = naive.replace(tzinfo=timezone.utc)

`replace(tzinfo=...)` does not convert the clock reading. It attaches metadata.

If the original value represented 10:00 in another timezone, this operation changes its meaning.

## 3.6 Explicit timezone policy

Define a policy object or configuration:

    TIMEZONE_POLICY = {
        "settlement": "Europe/Berlin",
        "notifications": "Asia/Karachi",
        "warehouse": "UTC",
    }

Keep timezone policy outside transformation code when possible.

That makes changes reviewable and testable.

## 3.7 Validate timezone identifiers

    from zoneinfo import ZoneInfo, ZoneInfoNotFoundError

    def load_timezone(name: str) -> ZoneInfo:
        try:
            return ZoneInfo(name)
        except ZoneInfoNotFoundError as exc:
            raise ValueError(f"unknown timezone: {name}") from exc

Do not silently fall back to the server timezone when configuration is invalid.

## 3.8 Store canonical event time

A common architecture is:

    source local/offset timestamp
              ↓
          parse safely
              ↓
        resolve timezone
              ↓
        canonical UTC instant
              ↓
        persist canonical value
              ↓
      derive local representations

Store the canonical instant for event ordering and computation.

If the original local representation is important for audit or business semantics, preserve it separately.

## 3.9 PostgreSQL conversion

PostgreSQL can convert timestamp values using timezone expressions.

Example:

    SELECT
        event_time AT TIME ZONE 'Europe/Berlin'
    FROM events;

The exact result and type depend on the input type, so always test the expression with representative values.

For event instants, keep the stored representation semantically clear and perform timezone conversion at the reporting or application boundary when practical.

## 3.10 Application boundary versus storage boundary

A useful rule is:

    storage → canonical instant
    API → explicit timezone/offset
    UI → user's display timezone

Do not make every downstream consumer repeat the same timezone interpretation logic.

---

## 4. Testing

Timezone tests must include normal dates and transition boundaries.

## 4.1 Conversion invariant

    def test_timezone_conversion_preserves_instant():
        original = datetime(2026, 9, 28, 12, 0, tzinfo=timezone.utc)
        converted = original.astimezone(ZoneInfo("Europe/Berlin"))

        assert converted.astimezone(timezone.utc) == original

## 4.2 Multiple zones

Test representative zones relevant to the product:

    Asia/Karachi
    Europe/Berlin
    America/New_York
    UTC

Do not test only the developer's local timezone.

## 4.3 DST transition tests

For a DST-observing timezone, select known transition dates from the timezone database used by your runtime and test:

- the last valid local time before the transition
- the first valid local time after the transition
- the repeated hour during a backward transition
- the skipped interval during a forward transition.

Do not hard-code assumptions that DST changes on the same calendar date every year.

## 4.4 Ambiguous local times

When a local wall-clock time occurs twice, the test must specify which occurrence is intended.

In Python, `fold` can distinguish the two interpretations when supported by the timezone implementation:

    first = datetime(
        2026, 11, 1, 1, 30,
        tzinfo=ZoneInfo("America/New_York"),
        fold=0,
    )

    second = datetime(
        2026, 11, 1, 1, 30,
        tzinfo=ZoneInfo("America/New_York"),
        fold=1,
    )

The specific transition date in tests must match the timezone database version used by the environment. Prefer generating transition fixtures from known timezone rules rather than assuming a permanent date.

## 4.5 Nonexistent local times

When clocks move forward, some local wall-clock values do not occur.

Your source contract must define whether to:

- reject the record
- shift according to an explicit business rule
- preserve the raw value and quarantine it.

Do not silently choose a correction.

## 4.6 Round-trip tests

For an unambiguous aware datetime:

    local = datetime(2026, 9, 28, 10, 0, tzinfo=ZoneInfo("Europe/Berlin"))
    utc = local.astimezone(timezone.utc)
    round_trip = utc.astimezone(ZoneInfo("Europe/Berlin"))

    assert round_trip == local

Round-trip testing catches accidental offset changes.

## 4.7 Date-boundary tests

Test instants near midnight:

    23:59:59
    00:00:00
    00:00:01

across at least two different timezones.

Verify whether the resulting local calendar date changes as expected.

## 4.8 Invalid timezone configuration

Test:

    "Not/A/Timezone"
    "Europe/Unknown"
    ""
    None

The pipeline should fail clearly or quarantine according to its configuration contract.

---

## 5. Observability

Track timezone-related signals separately from generic parsing failures.

| Signal | Why it matters |
|---|---|
| invalid timezone configuration count | Detect deployment/configuration errors |
| local-time ambiguity count | Detect ambiguous business timestamps |
| nonexistent-time count | Detect invalid local wall-clock values |
| conversion volume by timezone | Detect unexpected distribution changes |
| source offset distribution | Detect source behavior changes |
| date-boundary shifts | Detect unexpected calendar-day movement |
| timezone conversion failures | Detect transformation regressions |

Example structured event:

    {
      "event": "timezone_conversion_failed",
      "timezone": "Europe/Berlin",
      "reason": "invalid_timezone",
      "source_system": "orders-api",
      "record_id": "order_123"
    }

Do not log full source payloads merely because a timezone conversion failed.

### 5.1 Monitor timezone distribution

A sudden change from:

    Europe/Berlin → 92%
    UTC → 8%

to a radically different distribution can indicate an upstream configuration or ingestion problem.

The expected distribution is domain-specific; monitor deviations from your established baseline.

---

## 6. Intentional Failure

### Failure 1 — Attach UTC to local time

Take:

    2026-09-28 10:00 Europe/Berlin

and incorrectly execute:

    naive.replace(tzinfo=timezone.utc)

Expected symptom:

- the event shifts to the wrong instant
- downstream UTC ordering becomes incorrect.

Diagnosis:

- compare the raw local contract
- reconstruct the expected instant
- compare with the transformed value.

### Failure 2 — Replace a named timezone with a fixed offset

Use a hard-coded offset for a timezone whose rules change.

Expected symptom:

- values are correct during one part of the year
- values become shifted around timezone-rule transitions.

Diagnosis:

- compare the configured timezone identifier
- inspect the applicable timezone rules.

### Failure 3 — Ignore DST ambiguity

Process a repeated local time without identifying which occurrence it represents.

Expected symptom:

- duplicate local timestamps map to different instants
- aggregations or ordering become inconsistent.

Recovery:

- recover the source event's offset or occurrence metadata if available
- otherwise quarantine records whose intended occurrence cannot be determined.

### Failure 4 — Accept nonexistent local time silently

Create a local time during a skipped DST interval and assume it existed.

Expected symptom:

- records may be shifted according to an implicit library/application rule
- source and target semantics diverge.

Recovery:

- preserve the raw source value
- apply the documented business policy
- reprocess affected records.

---

## 7. Recovery

### 7.1 Wrong timezone configuration

1. Identify the affected timezone configuration.
2. Determine the exact deployment/configuration window.
3. Compare source timezone metadata with transformed values.
4. Recover from raw source evidence.
5. Re-run the conversion with the corrected timezone.
6. Reconcile affected counts and time distributions.
7. Check downstream reports and partitions.
8. Add a regression test and configuration validation.

### 7.2 Wrong fixed offset

1. Identify whether the source represented a named location or a fixed offset.
2. Replace the fixed offset with the correct named timezone when appropriate.
3. Recompute affected instants.
4. Verify dates around timezone-rule transitions.
5. Reconcile downstream aggregates.

### 7.3 DST ambiguity

If the original local time is ambiguous:

1. Check whether the source also supplied an offset.
2. Check event identifiers or sequence information that can disambiguate the occurrence.
3. Check trusted raw data and upstream documentation.
4. If the intended occurrence remains unknowable, quarantine rather than inventing one.

### 7.4 Wrong timezone database

Timezone rules can change over time.

If a runtime or container uses an unexpected timezone database:

1. identify the runtime/database version
2. compare it with the approved environment
3. determine affected conversion windows
4. re-run conversions using the approved rule set
5. record the timezone data dependency in deployment controls.

---

## 8. Production Tools You Should Know

### 8.1 Python `zoneinfo`

Know:

- `ZoneInfo`
- `astimezone()`
- `fold`
- aware versus naive datetime behavior
- IANA timezone identifiers.

Prefer the standard timezone model over handwritten offset tables.

### 8.2 PostgreSQL

Know:

- `timestamptz`
- `AT TIME ZONE`
- timezone-aware comparisons
- session timezone configuration.

Always test SQL expressions with representative timestamps because the input type affects the result.

### 8.3 IANA Time Zone Database

Understand that named timezone behavior comes from timezone rule data.

Know why:

    Europe/Berlin

is more expressive than:

    UTC+01:00

when the domain means a regional location.

Timezone data is an operational dependency and should be treated accordingly.

---

## 9. Production Runbook

### Symptom: timestamps are shifted by one hour

Check:

1. Named timezone versus fixed offset.
2. DST transition.
3. Runtime timezone data.
4. Source offset.
5. Conversion direction.

### Symptom: duplicate local times appear

Check:

1. Whether a backward DST transition occurred.
2. Whether the source includes offsets.
3. Whether local time was used as an event identifier.
4. Whether the UTC instant remains unique.

### Symptom: a local timestamp never occurred

Check:

1. Forward DST transition.
2. Source timezone.
3. Application/library behavior for nonexistent local times.
4. Business rule for invalid local timestamps.

### Symptom: reports cross the wrong calendar date

Check:

1. Whether the report is based on an instant or a business date.
2. Report timezone configuration.
3. Midnight boundaries.
4. `AT TIME ZONE` usage.

### Symptom: only one region is affected

Check:

1. Region-specific timezone configuration.
2. IANA timezone identifier.
3. Source metadata.
4. Timezone database version.

---

## 10. Common Mistakes

### Mistake 1 — Treating a fixed offset as a timezone

An offset identifies a relationship to UTC at a point in time. A named timezone represents regional rules.

### Mistake 2 — Using the server timezone

A pipeline worker's local timezone must not define business semantics accidentally.

### Mistake 3 — Using `replace(tzinfo=...)` as conversion

Attaching timezone metadata is not the same operation as converting an instant.

### Mistake 4 — Ignoring DST

Most ordinary timestamps do not expose the problem. Transition boundaries do.

### Mistake 5 — Converting business dates as if they were instants

Some dates belong to a business calendar and should remain dates.

### Mistake 6 — Repeatedly converting strings

Parse once, establish canonical semantics, and derive presentation representations later.

### Mistake 7 — Hard-coding timezone rules

Timezone rules are maintained externally. Handwritten offset tables become stale.

### Mistake 8 — Assuming timezone rules are permanent

Timezone databases can change as regional rules change.

### Mistake 9 — Silently resolving ambiguity

If the source does not provide enough information, quarantine or apply an explicit documented policy.

### Mistake 10 — Testing only one timezone

Always test multiple representative zones and transition boundaries relevant to the system.

---

## 11. Definition of Done

T07 is complete when you can:

- distinguish an instant, offset, and named timezone
- explain conversion versus interpretation of local time
- use IANA timezone identifiers
- convert aware datetimes without changing the underlying instant
- explain why fixed offsets are not always equivalent to named timezones
- recognize ambiguous local times
- recognize nonexistent local times
- define an explicit policy for ambiguous/nonexistent values
- use Python `zoneinfo` correctly
- avoid accidental use of the server timezone
- use PostgreSQL timezone operations safely
- test DST transition boundaries
- test midnight and calendar-date boundaries
- observe timezone-specific failures
- intentionally reproduce timezone conversion errors
- recover bad conversions from trusted raw evidence
- account for timezone database versions in production operations.

---

## 12. What You Learned

The central lesson is:

> **Timezone conversion is a semantic operation on time, not a string-formatting operation.**

For every timezone transformation, you should be able to answer:

1. Is the input an instant or a local wall-clock value?
2. Where does the timezone information come from?
3. Is the timezone a fixed offset or a named IANA region?
4. Could the local time be ambiguous or nonexistent?
5. What business rule resolves that condition?
6. What canonical representation is stored?
7. Can the conversion be reproduced from trusted source evidence?

If those answers are explicit, timezone handling becomes a controlled transformation instead of a hidden source of data corruption.

---

## Next Recipe

**T08 — Numeric Transformation**

T08 will cover numeric parsing, precision, rounding, scale, overflow, decimal arithmetic, numeric normalization, and safe transformations for financial and analytical data.