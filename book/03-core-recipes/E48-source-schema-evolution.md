# E48 — Source Schema Evolution

## 1. Problem Recognition

### The production problem

A database extractor reads a source table whose structure changes while the pipeline is still running or between pipeline runs.

Examples:

- a new column is added;
- a column is removed;
- a column is renamed;
- a numeric column changes from `integer` to `bigint` or `numeric`;
- a nullable column becomes `NOT NULL`;
- a view changes its selected columns;
- an extraction index is removed or replaced;
- a migration is deployed while one worker is reading and another worker starts later;
- a read replica temporarily exposes a different schema state from the primary;
- a long-running extraction observes a schema that differs from the schema used when the run started.

The dangerous assumption is:

> If the SQL query still runs, the schema change is safe.

That is false. A query can continue to run while the meaning, types, ordering, nullability, performance, or downstream contract has changed.

### How to recognize the problem

Investigate source schema evolution when you see:

- `column does not exist` errors;
- `undefined column` or `unknown field` errors;
- type conversion failures;
- unexpected `NULL` values;
- new columns appearing unexpectedly;
- missing columns in extracted records;
- changed query plans after a migration;
- a view returning a different shape;
- different schema fingerprints across workers;
- extraction success but destination validation failures;
- different results before and after a deployment;
- replica workers behaving differently from primary workers;
- a long-running run that spans a source migration window.

### Why this is an ETL-specific problem

Core Recipe 17 teaches the general schema-change mechanism. E28 teaches schema evolution for APIs. This recipe focuses specifically on **database source schemas during extraction**.

The extractor must answer two separate questions:

1. **What schema does the source have?**
2. **Can this extraction run safely against that schema?**

Detection answers only the first question.

---

## 2. Concept and Reasoning

### The core mental model

Treat the source database schema as an explicit input contract.

```text
SOURCE DATABASE
      ↓
SCHEMA DISCOVERY
      ↓
SCHEMA FINGERPRINT
      ↓
COMPARE WITH EXPECTED CONTRACT
      ↓
CLASSIFY CHANGE
      ↓
CHECK EXTRACTION COMPATIBILITY
      ↓
EXTRACT
      ↓
VALIDATE
      ↓
ADAPT / QUARANTINE / STOP
      ↓
RECONCILE
```

### Schema contract

At minimum, an extraction contract should describe:

| Property | Example | Why it matters |
|---|---|---|
| table | `payments` | Identifies the source relation |
| column | `amount` | Identifies the field |
| data type | `numeric(18,2)` | Controls interpretation |
| nullable | `false` | Controls validity |
| ordinal position | `4` | Useful for diagnostics, not identity |
| extraction role | `watermark` | Controls progress semantics |
| required | `true` | Controls extraction validity |
| semantic version | `12` | Identifies the expected contract |

Do not use ordinal position as column identity. A column can move without changing its meaning.

### Additive versus breaking changes

Common changes can be classified like this:

| Change | Typical classification | Main risk |
|---|---|---|
| Add nullable column | Usually compatible | Destination may not expect it |
| Add column with safe default | Usually compatible | Default semantics |
| Add required column without compatible default | Potentially breaking | Existing rows may not satisfy contract |
| Remove unused column | Potentially compatible | Query/model may still reference it |
| Remove required column | Breaking | Extractor cannot produce contract |
| Rename column | Breaking unless mapped | Identity changes |
| Widen numeric type safely | Often compatible | Precision/overflow assumptions |
| Narrow numeric type | Breaking | Data loss/overflow |
| Increase string length | Usually compatible | Destination constraints |
| Change string to numeric | Breaking | Interpretation changes |
| Make nullable column `NOT NULL` | Usually source-safe, contract-sensitive | Existing extraction assumptions may differ |
| Make `NOT NULL` column nullable | Potentially breaking | New nulls can enter pipeline |
| Change view definition | Contract-dependent | Shape or meaning can change |
| Remove extraction index | Semantically compatible, operationally dangerous | Query performance |

These classifications are starting points. The extractor must evaluate the actual contract and downstream requirements.

### Compatibility directions

Think about compatibility in three directions:

- **Extractor backward compatibility:** can the new source still be read by the existing extractor?
- **Extractor forward compatibility:** can the extractor tolerate source fields introduced by a newer schema?
- **Destination compatibility:** can the extracted data still be represented and validated downstream?

An additive source column may be harmless to a `SELECT` that names columns explicitly, but dangerous to `SELECT *` or to a destination expecting an exact record shape.

### Why `SELECT *` is dangerous

Prefer:

```sql
SELECT
    payment_id,
    account_id,
    amount,
    currency,
    updated_at
FROM payments;
```

over:

```sql
SELECT *
FROM payments;
```

Explicit projection makes the extraction contract visible and reduces accidental coupling to additive source columns.

---

## 3. Implementation

### Step 1 — Discover the source schema

For PostgreSQL, inspect `information_schema` or PostgreSQL catalog tables.

```sql
SELECT
    table_schema,
    table_name,
    column_name,
    ordinal_position,
    data_type,
    is_nullable
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name = 'payments'
ORDER BY ordinal_position;
```

For deeper PostgreSQL-specific information, inspect `pg_catalog`.

```sql
SELECT
    n.nspname AS schema_name,
    c.relname AS table_name,
    a.attname AS column_name,
    pg_catalog.format_type(a.atttypid, a.atttypmod) AS data_type,
    a.attnotnull AS not_null
FROM pg_catalog.pg_attribute a
JOIN pg_catalog.pg_class c
  ON c.oid = a.attrelid
JOIN pg_catalog.pg_namespace n
  ON n.oid = c.relnamespace
WHERE n.nspname = 'public'
  AND c.relname = 'payments'
  AND a.attnum > 0
  AND NOT a.attisdropped
ORDER BY a.attnum;
```

### Step 2 — Represent the expected contract

Keep the extraction contract in code or configuration rather than hiding it inside SQL.

```python
EXPECTED_SCHEMA = {
    "payment_id": {"type": "bigint", "nullable": False},
    "account_id": {"type": "bigint", "nullable": False},
    "amount": {"type": "numeric", "nullable": False},
    "currency": {"type": "text", "nullable": False},
    "updated_at": {"type": "timestamp with time zone", "nullable": False},
}
```

The exact type representation should be normalized before comparison because database-specific type names can have aliases.

### Step 3 — Normalize schema metadata

Do not compare raw catalog output blindly.

Normalize:

- schema name;
- table name;
- column name;
- normalized data type;
- nullability;
- relevant constraints;
- extraction role;
- optionally relevant index information.

Example:

```python
def normalize_column(row):
    return {
        "name": row["column_name"],
        "type": normalize_type(row["data_type"]),
        "nullable": row["is_nullable"] == "YES",
    }
```

### Step 4 — Fingerprint the expected schema

A deterministic fingerprint makes schema comparison cheap and auditable.

```python
import hashlib
import json


def schema_fingerprint(schema: dict) -> str:
    canonical = json.dumps(
        schema,
        sort_keys=True,
        separators=(",", ":"),
    )
    return hashlib.sha256(canonical.encode("utf-8")).hexdigest()
```

Store the fingerprint with extraction-run metadata.

Example:

```text
run_id: 2026-09-26T10:00:00Z
source: postgres-primary
table: public.payments
schema_fingerprint: 7b2...
schema_version: 14
```

### Step 5 — Compare schemas

```python
def compare_schema(expected: dict, actual: dict) -> dict:
    expected_names = set(expected)
    actual_names = set(actual)

    return {
        "added": sorted(actual_names - expected_names),
        "removed": sorted(expected_names - actual_names),
        "changed": sorted(
            name
            for name in expected_names & actual_names
            if expected[name] != actual[name]
        ),
    }
```

This is intentionally simple. Production code should return detailed differences, including old and new definitions.

### Step 6 — Classify the change

```python
def classify_change(diff: dict) -> str:
    if diff["removed"] or diff["changed"]:
        return "BREAKING_OR_REVIEW_REQUIRED"
    if diff["added"]:
        return "ADDITIVE"
    return "UNCHANGED"
```

Do not automatically label every additive change safe. Check the query projection and destination contract first.

### Step 7 — Pin the schema for a run

For an important extraction, record the schema state used by the run.

```sql
CREATE TABLE IF NOT EXISTS extraction_schema_snapshot (
    run_id UUID PRIMARY KEY,
    source_name TEXT NOT NULL,
    relation_name TEXT NOT NULL,
    schema_version TEXT,
    schema_fingerprint TEXT NOT NULL,
    captured_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Now the run can answer:

> Which source schema did this extraction process?

### Step 8 — Decide what happens when the schema changes

Use explicit policy:

| Change | Example action |
|---|---|
| Safe additive field not selected | Continue |
| Safe additive field selected and destination supports it | Continue after contract update |
| Required field removed | Stop before checkpoint advancement |
| Type narrowed | Stop |
| Semantics changed | Stop or require explicit migration |
| Unknown schema version | Stop or quarantine according to policy |
| Index changed only | Continue, but monitor performance |
| View changed | Revalidate contract |

Never silently reinterpret a field because the new database type happens to be convertible.

### Step 9 — Extract with explicit projection

Build SQL from a controlled column list.

```python
EXTRACT_COLUMNS = [
    "payment_id",
    "account_id",
    "amount",
    "currency",
    "updated_at",
]

column_sql = ", ".join(EXTRACT_COLUMNS)
query = f"SELECT {column_sql} FROM public.payments ORDER BY payment_id"
```

In production, validate every requested identifier against the known contract rather than accepting arbitrary user input.

### Step 10 — Validate the result

Schema validation must happen before the extraction checkpoint moves.

```python
def validate_record(record):
    required = [
        "payment_id",
        "account_id",
        "amount",
        "currency",
        "updated_at",
    ]
    missing = [name for name in required if name not in record]
    if missing:
        raise ValueError(f"Missing required fields: {missing}")
```

Also validate types, nullability, ranges, and semantic constraints appropriate to the source contract.

---

## 4. Important Database-Specific Cases

### Column added during extraction

Suppose a worker begins with:

```text
payment_id
amount
updated_at
```

and a migration adds:

```text
currency
```

If the query explicitly selects the original columns, the worker may continue safely.

If it uses `SELECT *`, the result shape changes and downstream processing may fail.

### Column renamed

Renames should be treated as breaking unless the extraction layer has an explicit mapping.

```text
customer_id  →  account_id
```

Do not infer that these are equivalent merely because both are integers.

### Data type change

Consider:

```sql
ALTER TABLE payments
ALTER COLUMN amount TYPE numeric(20,4);
```

Even when PostgreSQL performs the migration safely, the extraction contract may change because precision and scale changed.

Compare semantic requirements, not only whether the database accepts the DDL.

### Nullability change

Changing:

```text
currency NOT NULL
```

to nullable changes the data-quality contract even though the query may continue to run.

An extractor that assumes non-null currency must detect this change or validate the resulting records.

### Index changes

Index changes normally do not alter row meaning, but they can radically change extraction performance.

An extraction query should therefore monitor its query plan and execution time after source migrations.

### View changes

Views are especially important when the extractor reads from a view instead of a base table.

A view can keep the same name while changing:

- selected columns;
- column aliases;
- joins;
- filters;
- data types;
- row semantics.

Treat the view definition as part of the extraction contract.

---

## 5. Schema Evolution During Long-Running Extraction

A large extraction can span a deployment window.

That creates a difficult case:

```text
10:00  extractor starts
10:20  worker A reads schema version 14
10:25  migration deploys schema version 15
10:30  worker B starts
10:35  workers process different schema assumptions
```

Do not allow workers to independently discover and accept incompatible schemas.

Use a run-level contract:

```text
RUN START
   ↓
CAPTURE SCHEMA VERSION / FINGERPRINT
   ↓
WORKERS USE SAME CONTRACT
   ↓
DETECT DRIFT
   ↓
STOP / ADAPT ACCORDING TO POLICY
```

### Snapshot interaction

A consistent database snapshot controls row visibility. It does not automatically mean that every aspect of your application-level schema contract is permanently safe for the whole run.

For PostgreSQL, understand how DDL, transaction snapshots, and catalog visibility interact with your extraction design. Test the exact migration behavior instead of assuming that a long-running transaction provides an application-level schema freeze.

### Parallel extraction

Parallel workers must agree on:

- schema version;
- selected columns;
- type expectations;
- extraction boundaries;
- source endpoint or replica policy.

Otherwise, two workers can produce records that have different interpretations under one run ID.

---

## 6. Source and Replica Schema Consistency

If extraction reads from replicas, verify the schema state you are actually querying.

Useful checks include:

```sql
SELECT current_database(), current_user;
```

and the same schema fingerprint query used for the primary.

Record:

- source endpoint;
- schema fingerprint;
- schema version if available;
- capture time;
- replica identity.

A replica can be behind in data and can also be in a different operational state during deployments. Do not treat `same database name` as proof of identical extraction conditions.

---

## 7. Testing

### Unit tests

Test schema comparison independently.

```python
def test_added_nullable_column_is_detected():
    expected = {
        "id": {"type": "bigint", "nullable": False},
    }
    actual = {
        **expected,
        "currency": {"type": "text", "nullable": True},
    }

    diff = compare_schema(expected, actual)

    assert diff["added"] == ["currency"]
    assert diff["removed"] == []
```

Test breaking changes:

- required column removed;
- type changed;
- nullability changed;
- renamed column;
- unexpected view column;
- unknown schema version.

### Integration tests

Use a disposable PostgreSQL database.

Test this sequence:

```text
CREATE SOURCE TABLE
        ↓
CAPTURE SCHEMA
        ↓
EXTRACT
        ↓
ALTER TABLE
        ↓
DISCOVER NEW SCHEMA
        ↓
COMPARE
        ↓
CLASSIFY
        ↓
VERIFY POLICY
```

### Representative migration tests

At minimum test:

1. add nullable column;
2. add required column;
3. remove unused column;
4. remove required column;
5. rename column;
6. widen a numeric type;
7. narrow a numeric type;
8. change nullability;
9. alter a view;
10. drop an extraction index.

### Mixed-worker test

Start two workers with the same run contract. Apply a migration between their startup times. Verify that the second worker cannot silently use a different schema contract.

---

## 8. Observability

Log structured schema information, never raw sensitive data.

Recommended fields:

```text
run_id
worker_id
source_name
relation_name
schema_version
schema_fingerprint
expected_schema_fingerprint
schema_change_class
added_columns
removed_columns
changed_columns
action
records_processed
checkpoint
duration_ms
```

Useful metrics:

| Metric | Purpose |
|---|---|
| `schema_fingerprint_changes_total` | Detect schema changes |
| `schema_validation_failures_total` | Detect contract failures |
| `schema_breaking_changes_total` | Detect blocking migrations |
| `schema_drift_detected_total` | Detect unexpected drift |
| `extraction_runs_blocked_by_schema_total` | Measure operational impact |
| `extraction_duration_seconds` | Detect performance regressions |

Alert when a breaking schema change blocks an important extraction or when different workers report different schema fingerprints for the same run.

---

## 9. Intentional Failure

Do not trust this recipe until you can break it.

### Failure Drill 1 — Remove a required column

1. Start an extraction using `payment_id`, `amount`, and `updated_at`.
2. Remove `amount` in a controlled test database.
3. Run schema comparison.
4. Verify the change is classified as breaking.
5. Verify extraction does not advance its checkpoint.
6. Verify the run is marked failed or blocked.

Expected result:

```text
SCHEMA DRIFT
    ↓
BREAKING CHANGE
    ↓
STOP BEFORE CHECKPOINT ADVANCEMENT
```

### Failure Drill 2 — Add a column

1. Add a nullable column.
2. Recalculate the schema fingerprint.
3. Verify the change is detected.
4. Verify the extractor follows the configured additive-change policy.

### Failure Drill 3 — Change a type

1. Change a test field to an incompatible type.
2. Run schema comparison.
3. Verify the extractor refuses to reinterpret it silently.

### Failure Drill 4 — Mixed workers

1. Start worker A.
2. Apply a source migration.
3. Start worker B.
4. Verify both workers are forced to honor the run-level schema contract.

---

## 10. Recovery

### Recovery sequence

```text
DETECT DRIFT
    ↓
IDENTIFY EXACT CHANGE
    ↓
STOP UNSAFE PROGRESS
    ↓
CHECK SOURCE MIGRATION STATUS
    ↓
CLASSIFY COMPATIBILITY
    ↓
UPDATE CONTRACT OR ROLL FORWARD SOURCE/EXTRACTOR
    ↓
RESTART FROM LAST SAFE BOUNDARY
    ↓
RECONCILE
```

### Safe recovery rules

- Never advance a checkpoint past records whose schema interpretation is uncertain.
- Preserve the old run's schema fingerprint.
- Keep the source migration and extractor change auditable.
- Prefer a forward fix when the migration is valid but the extractor is outdated.
- If the migration must be rolled back, verify both source and replica schema state before restarting.
- Reprocess from a known safe boundary when mixed-schema output may have been produced.
- Reconcile record counts and key ranges after recovery.

### Rollback versus forward-fix

Use a rollback only when the source migration itself is unsafe or must be reversed operationally.

A forward fix is often cleaner when:

- the source migration is correct;
- the extractor contract is outdated;
- the new schema can be handled deterministically;
- historical records remain interpretable.

Never make the choice based only on whether the SQL query can be made to run.

---

## 11. Production Runbook

### Before deployment

- [ ] Identify tables, views, and columns used by extraction.
- [ ] Record the expected schema contract.
- [ ] Define compatible and breaking changes.
- [ ] Avoid `SELECT *` for stable extraction contracts.
- [ ] Test representative migrations.
- [ ] Check destination compatibility.
- [ ] Verify source and replica deployment behavior.
- [ ] Confirm schema fingerprinting is observable.

### When drift is detected

1. Stop unsafe checkpoint advancement.
2. Capture the current schema fingerprint.
3. Compare it with the run contract.
4. Identify exact DDL or migration responsible.
5. Classify the change.
6. Check whether records were already processed under different schemas.
7. Choose continue, adapt, quarantine, or stop.
8. Restart from the last safe boundary if required.
9. Reconcile the recovered output.

### Do not

- silently accept a removed required column;
- silently cast incompatible types;
- use `SELECT *` as a substitute for a schema contract;
- assume a successful SQL query means semantic compatibility;
- let workers independently choose schema versions within one run;
- advance checkpoints after an unresolved schema validation failure;
- ignore index changes that materially affect extraction performance.

---

## 12. Production Tools You Should Know

### 1. PostgreSQL

PostgreSQL provides catalog metadata, transactional DDL behavior, views, constraints, indexes, and schema inspection mechanisms. It is the concrete database used to practice this recipe.

### 2. Flyway

Flyway is a database migration tool. Know how migration versioning and deployment sequencing affect an extractor that depends on a source schema.

### 3. Debezium

Debezium captures database changes and carries schema information alongside change events. It is useful when schema evolution must be coordinated with CDC pipelines.

---

## 13. Common Mistakes

### Mistake 1 — Treating schema discovery as schema management

Seeing a changed schema is not enough. You need a compatibility decision.

### Mistake 2 — Using `SELECT *`

An additive source column can silently change the extracted record shape.

### Mistake 3 — Comparing only column names

Type, nullability, constraints, and semantics can change without the name changing.

### Mistake 4 — Treating every additive change as safe

A new required field or a changed view can break downstream assumptions.

### Mistake 5 — Ignoring schema state across workers

One run must not contain silently mixed schema contracts.

### Mistake 6 — Ignoring indexes

Index changes may not change correctness, but they can change source load and extraction duration dramatically.

### Mistake 7 — Advancing checkpoints after schema failure

A checkpoint represents safe progress. It must not move when the interpretation of processed data is uncertain.

---

## 14. Definition of Done

Before considering this recipe complete, you should be able to:

- [ ] Discover a database table schema programmatically.
- [ ] Define an explicit extraction schema contract.
- [ ] Normalize database type metadata.
- [ ] Generate a deterministic schema fingerprint.
- [ ] Detect added, removed, renamed, and changed columns.
- [ ] Classify schema changes according to an explicit policy.
- [ ] Explain additive versus breaking evolution.
- [ ] Explain extractor backward and forward compatibility.
- [ ] Avoid accidental `SELECT *` contracts.
- [ ] Record the schema used by an extraction run.
- [ ] Detect mixed schema versions across workers.
- [ ] Account for view and index changes.
- [ ] Test schema migrations against a real database.
- [ ] Stop safely before checkpoint advancement on an incompatible change.
- [ ] Recover from a schema change from the last safe boundary.
- [ ] Reconcile the recovered extraction.
- [ ] Observe schema drift in logs and metrics.

---

## 15. What You Learned

Source schema evolution is not just a database migration problem. It is an extraction-contract problem.

You learned to:

1. discover the actual source schema;
2. represent the extractor's expected contract explicitly;
3. fingerprint and compare schemas;
4. classify additive and breaking changes;
5. protect extraction from silent semantic changes;
6. coordinate schema state across workers;
7. account for views, indexes, replicas, and long-running runs;
8. stop before unsafe checkpoint advancement;
9. recover from a known safe boundary;
10. reconcile after recovery.

### The key principle

> A schema change is safe only when the extractor can prove that the new source structure remains compatible with the extraction contract and downstream interpretation. Detection alone is not schema management.