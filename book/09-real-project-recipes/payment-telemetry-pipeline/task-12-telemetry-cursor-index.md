# Recipe 12 — Telemetry Cursor Index

> **Real project:** Payment Telemetry Pipeline  
> **Stage:** 12  
> **Source implementation commit:** <code>01218b951</code> — <code>feat: add telemetry cursor index</code>  
> **Source repository:** <code>svc</code>

## 1. What This Recipe Teaches

This recipe adds the physical database access path required by the incremental keyset extraction introduced in Stage 11.

By completing it, you should understand how to:
- identify the query pattern that needs an index;
- design a composite B-tree index for a row-value cursor;
- match index column order and direction to the extractor;
- keep database migrations in the repository that owns the source table;
- register a migration and regenerate the schema snapshot;
- verify an index with PostgreSQL EXPLAIN;
- understand migration locking and rollback behavior.

The Stage 12 implementation adds <code>frontend_telemetry_event_received_event_idx</code> on <code>(received_at ASC, event_id ASC)</code> to <code>public.frontend_telemetry_event</code>, bumps migration version to 294, and regenerates the schema snapshot. fileciteturn21file0L3-L15

## 2. The Problem

Stage 11 introduced incremental extraction using the composite cursor:

~~~sql
WHERE (received_at, event_id) > (%s, %s)
ORDER BY received_at ASC, event_id ASC
LIMIT %s
~~~

The logical pagination design is correct, but without a matching index PostgreSQL may need to scan a large portion of the source table for every incremental batch.

As telemetry volume grows, repeatedly scanning <code>public.frontend_telemetry_event</code> becomes increasingly expensive. Stage 12 adds a composite B-tree index that directly supports the Stage 11 access pattern. fileciteturn21file0L8-L15

## 3. The Core Design

The pipeline cursor is:

~~~text
(received_at, event_id)
~~~

The database index uses exactly the same logical ordering:

~~~text
(received_at ASC, event_id ASC)
~~~

The relationship is:

~~~text
Stage 11 cursor
      |
      v
(received_at, event_id)
      |
      v
Stage 12 composite B-tree index
      |
      v
efficient seek to the cursor position
~~~

The source explicitly requires exact column and direction matching. fileciteturn21file0L40-L43

## 4. Step 1 — Start With the Real Query

Design the index from the query that the pipeline actually executes:

~~~sql
SELECT ...
FROM public.frontend_telemetry_event
WHERE (received_at, event_id) > (%s, %s)
ORDER BY received_at ASC, event_id ASC
LIMIT 1000;
~~~

Three details matter:

1. The filter uses both cursor columns.
2. The ordering uses the same two columns.
3. Both columns are ascending.

The index should mirror those requirements rather than being chosen generically.

## 5. Step 2 — Create the Composite Index

Create:

~~~sql
CREATE INDEX frontend_telemetry_event_received_event_idx
    ON public.frontend_telemetry_event (received_at ASC, event_id ASC);
~~~

This is the exact index recorded by the Stage 12 source. fileciteturn21file0L30-L36

## 6. Why Column Order Matters

These are different indexes:

~~~text
(received_at, event_id)
(event_id, received_at)
~~~

Stage 11's cursor and ORDER BY are based on:

~~~text
(received_at, event_id)
~~~

Therefore Stage 12 uses that exact sequence.

A composite index is ordered by its defined column sequence. The source explicitly identifies exact column and direction matching as a design decision. fileciteturn21file0L40-L43

## 7. Why the Event ID Is the Second Column

Telemetry events can share the same <code>received_at</code> timestamp.

For example:

~~~text
received_at          event_id
-------------------  --------
10:00:00.000         A
10:00:00.000         B
10:00:00.000         C
10:00:01.000         D
~~~

A timestamp-only cursor cannot identify the exact position between A, B, and C.

The composite cursor does:

~~~text
(10:00:00.000, A)
(10:00:00.000, B)
(10:00:00.000, C)
~~~

The event ID provides the deterministic tie-breaker. This is the same ordering established in Stage 11. fileciteturn21file0L11-L15

## 8. Step 3 — Put the Migration in svc

The indexed table is:

~~~text
public.frontend_telemetry_event
~~~

The table is created and owned by <code>svc</code> migrations.

Therefore the index migration belongs in <code>svc</code>, not the <code>devops</code> pipeline repository.

The source explicitly identifies this ownership boundary. fileciteturn21file0L40-L43

## 9. Step 4 — Create the Up Migration

Create:

~~~text
db/migrations/000294_frontend_telemetry_received_event_cursor_index.up.sql
~~~

Contents:

~~~sql
CREATE INDEX frontend_telemetry_event_received_event_idx
    ON public.frontend_telemetry_event (received_at ASC, event_id ASC);
~~~

The source documents this migration as migration 294. fileciteturn21file0L19-L23

## 10. Step 5 — Create the Down Migration

Create the matching rollback migration:

~~~text
db/migrations/000294_frontend_telemetry_received_event_cursor_index.down.sql
~~~

Use:

~~~sql
DROP INDEX IF EXISTS frontend_telemetry_event_received_event_idx;
~~~

The source explicitly records <code>DROP INDEX IF EXISTS</code> for clean rollback behavior. fileciteturn21file0L19-L23 fileciteturn21file0L71-L73

## 11. Step 6 — Register Migration 294

Update:

~~~text
db/migrate.go
~~~

Bump:

~~~text
migrationVersion = 294
~~~

The source explicitly records the version change from 293 to 294. fileciteturn21file0L21-L23

Migration registration is part of the database change. Creating the SQL file without updating migration state leaves the repository inconsistent.

## 12. Step 7 — Regenerate the Schema Snapshot

Regenerate:

~~~text
db/schema/0-schema.sql
~~~

The source records regeneration of the schema snapshot through version 294. fileciteturn21file0L21-L23

The intended state is:

~~~text
migration history = schema snapshot
~~~

Do not leave the snapshot representing version 293 after introducing migration 294.

## 13. Step 8 — Verify Migration Execution

Verify that the migration:
- executes cleanly;
- creates the expected index;
- uses the correct columns;
- uses the correct direction;
- can be rolled back;
- can be reapplied according to the project's migration process.

The source records verification of index creation syntax and compatibility with the keyset pagination plans. fileciteturn21file0L19-L24

## 14. Step 9 — Verify the Actual Query Plan

Do not stop at checking whether the index exists.

Run EXPLAIN against the actual Stage 11 extraction pattern:

~~~sql
EXPLAIN
SELECT ...
FROM public.frontend_telemetry_event
WHERE (received_at, event_id) > (%s, %s)
ORDER BY received_at ASC, event_id ASC
LIMIT 1000;
~~~

The source records that PostgreSQL EXPLAIN plans were verified to use the composite index for row-value comparisons. fileciteturn21file0L93-L95

Conceptually, the intended access path is:

~~~text
Index Scan
    |
    v
frontend_telemetry_event_received_event_idx
    |
    v
seek to cursor position
    |
    v
read subsequent ordered events
~~~

The source describes the intended cursor seek as O(log N). fileciteturn21file0L49-L60

## 15. Architecture and Data Flow

~~~text
Payment Telemetry Pipeline
          |
          v
    extract_events()
          |
          v
public.frontend_telemetry_event
          |
          +--> WHERE (received_at, event_id) > cursor
          |
          +--> ORDER BY received_at ASC, event_id ASC
          |
          v
frontend_telemetry_event_received_event_idx
          |
          v
      ordered batch
~~~

The source explicitly documents this query-to-index relationship. fileciteturn21file0L47-L60

## 16. Transaction Safety

The DDL migration runs inside standard transaction boundaries:

~~~text
BEGIN
  |
  +--> CREATE INDEX
  |
COMMIT
~~~

The source also records an important operational characteristic: non-concurrent index creation takes a table lock during creation. fileciteturn21file0L65-L67

Do not silently rewrite the documented migration as <code>CREATE INDEX CONCURRENTLY</code>. That would be a different migration strategy.

## 17. Idempotency and Rollback

Stage 12's idempotency concern is schema rollback rather than event processing.

The down migration uses:

~~~sql
DROP INDEX IF EXISTS frontend_telemetry_event_received_event_idx;
~~~

This allows rollback to remain safe when the index is already absent. The source explicitly documents this behavior. fileciteturn21file0L71-L73

## 18. Database Schema

Stage 12 does not introduce a new table.

It adds this index to the existing source table:

~~~sql
CREATE INDEX frontend_telemetry_event_received_event_idx
    ON public.frontend_telemetry_event (received_at ASC, event_id ASC);
~~~

The source records this exact definition. fileciteturn21file0L83-L88

## 19. What Stage 12 Does Not Change

Stage 12 does not change:
- telemetry event payload structure;
- validation rules;
- staging behavior;
- checkpoint semantics;
- pipeline orchestration;
- quarantine behavior.

Its purpose is physical query optimization for the existing Stage 11 extraction pattern. The source explicitly describes it as directly accelerating Stage 11 extraction. fileciteturn21file0L77-L79

## 20. PostgreSQL Validation

The source records validation on PostgreSQL 16.

It specifically verifies B-tree support for the composite types:

~~~text
(timestamptz, uuid)
~~~

The cursor therefore combines:

~~~text
received_at -> TIMESTAMPTZ
event_id    -> UUID
~~~

The source explicitly records this PostgreSQL 16 validation. fileciteturn21file0L99-L101

## 21. Tests and Verification

The source records two primary verification outcomes:

1. The migration executes cleanly.
2. PostgreSQL EXPLAIN plans use the composite index for row-value comparisons.

These are stronger checks than merely confirming that an index object exists. fileciteturn21file0L93-L95

Recommended verification sequence:

~~~text
Apply migration
      |
      v
Inspect index
      |
      v
Run EXPLAIN on Stage 11 query
      |
      v
Confirm composite index is available/used
      |
      v
Rollback if required
~~~

## 22. Common Implementation Mistakes

### Mistake 1 — Index the columns in the wrong order

Do not use <code>(event_id, received_at)</code> when the pipeline cursor is <code>(received_at, event_id)</code>.

### Mistake 2 — Index only received_at

That does not represent the complete composite cursor used by Stage 11.

### Mistake 3 — Ignore direction

The documented index uses ascending order for both columns.

### Mistake 4 — Put the migration in devops

The source table belongs to <code>svc</code>, so its index lifecycle belongs there.

### Mistake 5 — Forget migration registration

The SQL file alone is not enough. Migration version 294 must be registered.

### Mistake 6 — Forget the schema snapshot

The Stage 12 implementation regenerates <code>db/schema/0-schema.sql</code>.

### Mistake 7 — Verify only index existence

Use EXPLAIN against the real Stage 11 query.

### Mistake 8 — Silently switch to concurrent index creation

The supplied implementation uses normal <code>CREATE INDEX</code> and documents its locking behavior.

### Mistake 9 — Add unrelated indexes

Stage 12 exists specifically to support the telemetry cursor query.

## 23. Independent Implementation Exercise

Take an append-oriented table with an incremental query shaped like:

~~~sql
WHERE (created_at, event_id) > (%s, %s)
ORDER BY created_at ASC, event_id ASC
LIMIT %s
~~~

Implement the corresponding database optimization.

Requirements:
1. Identify the exact cursor columns.
2. Match their order in the index.
3. Match their documented sort direction.
4. Create a migration in the repository that owns the table.
5. Add a rollback migration.
6. Register the migration version.
7. Regenerate the schema snapshot if the project uses one.
8. Apply the migration.
9. Inspect the resulting index.
10. Run EXPLAIN against the real incremental query.
11. Verify that the composite index can support the row-value comparison.
12. Document locking implications of the chosen index-creation method.

Then compare the query plan before and after the index and explain what access path changed.

## 24. Validation Checklist

### Query alignment
- [ ] Stage 11 uses <code>(received_at, event_id)</code> as its cursor.
- [ ] The index uses <code>(received_at, event_id)</code>.
- [ ] Both index columns use ascending order.
- [ ] The index supports the row-value comparison.
- [ ] The index supports the ORDER BY sequence.

### Migration
- [ ] Migration <code>000294</code> exists.
- [ ] Up migration creates the expected index.
- [ ] Down migration uses <code>DROP INDEX IF EXISTS</code>.
- [ ] Migration version is 294.
- [ ] Schema snapshot is regenerated.
- [ ] Migration executes successfully.

### Ownership
- [ ] Index migration lives in <code>svc</code>.
- [ ] The indexed table remains owned by <code>svc</code>.
- [ ] Migration ownership is not duplicated in <code>devops</code>.

### Query-plan verification
- [ ] EXPLAIN was run against the Stage 11 extraction query.
- [ ] Composite row-value comparison was tested.
- [ ] PostgreSQL can use the new composite index.
- [ ] PostgreSQL 16 validation passes.

### Operations
- [ ] Index-creation locking behavior is understood.
- [ ] Rollback is available.
- [ ] No unrelated indexes were added.

### Privacy
- [ ] The index contains no telemetry payload fields.
- [ ] No sensitive payload data is introduced by the migration.

## 25. Scope Boundaries

### Implemented in Stage 12
- composite B-tree cursor index;
- migration 294;
- migration registration;
- schema snapshot synchronization;
- query-plan verification;
- PostgreSQL 16 validation.

### Previously implemented in Stage 11
- composite keyset extraction;
- checkpointing;
- batch orchestration.

### Next stage — Stage 13
- persistent quarantine storage in <code>devops</code>.

The source explicitly defines the Stage 12 implementation and Stage 13 boundary. fileciteturn21file0L111-L114

## 26. What You Should Understand Before the Next Recipe

Stage 12 completes the physical database support for incremental extraction:

~~~text
Stage 11
Logical cursor
(received_at, event_id)
        |
        v
Stage 12
Matching B-tree index
        |
        v
Efficient PostgreSQL seek
        |
        v
Scalable incremental extraction
~~~

The central lessons are:
- database performance must be designed around the actual query pattern;
- composite cursor columns must be indexed in the same logical order used by extraction;
- index ownership follows table and migration ownership;
- migration registration and schema snapshots are part of a complete database change;
- EXPLAIN is stronger evidence than simply checking that an index exists;
- physical indexing complements keyset pagination;
- schema changes have operational locking characteristics that must be documented;
- Stage 12 optimizes extraction without changing the pipeline's logical processing behavior.

The supplied Stage 12 implementation records successful migration execution, EXPLAIN verification, and PostgreSQL 16 validation. fileciteturn21file0L93-L101

## Source Traceability

This recipe is derived from the supplied Stage 12 implementation record:

- Source commit: <code>01218b951</code>
- Subject: <code>feat: add telemetry cursor index</code>
- Repository: <code>svc</code>
- Overview and technical objective: fileciteturn21file0L3-L15
- Implementation sequence: fileciteturn21file0L19-L24
- Technical changes: fileciteturn21file0L28-L36
- Design decisions: fileciteturn21file0L40-L43
- Architecture and data flow: fileciteturn21file0L47-L60
- Transaction safety and idempotency: fileciteturn21file0L65-L73
- Integration and schema: fileciteturn21file0L77-L88
- Verification and PostgreSQL validation: fileciteturn21file0L93-L101
- Privacy and scope boundaries: fileciteturn21file0L105-L114