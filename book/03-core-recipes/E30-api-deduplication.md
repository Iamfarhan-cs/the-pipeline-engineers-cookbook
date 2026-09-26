# E30 — API Deduplication

## 1. Problem Recognition

APIs can return the same logical record more than once because of pagination overlap, retries, provider replay, mutable sources, overlapping incremental windows, duplicated records inside a page, or concurrent workers.

Recognize this problem when record counts are unexpectedly high, unique identities are lower than extracted rows, retries create duplicate writes, or incremental runs repeatedly encounter the same records.

> Make repeated API delivery harmless by defining record identity, detecting duplicates, and ensuring persistence remains idempotent.

## 2. Concept and Reasoning

Deduplication is not simply removing identical JSON objects. Two records can have different payloads while representing the same business entity.

Identity answers: Which logical record is this? Duplicate detection answers: Have I already processed this identity? Versioning answers: Is this a newer representation of the same identity?

## 3. Stable API Identity

Prefer provider-defined stable identifiers such as customer_id, transaction_id, order_id, or event_id.

Do not use mutable fields such as email or display name as identity unless the provider explicitly defines them as stable.

## 4. Composite Identity

Some APIs require composite identity such as tenant_id + external_id or account_id + transaction_reference.

Composite identity must be deterministic, documented, and enforced consistently.

## 5. Where Duplicates Come From

Typical path:

    API → PAGINATION → RETRY → WORKERS → PERSISTENCE

Therefore, use multiple defenses where appropriate:
1. extraction-level duplicate detection
2. durable destination uniqueness
3. idempotent writes
4. reconciliation

## 6. Duplicate Records Inside One Response

Example:

    record 1: id=1
    record 2: id=2
    record 3: id=2

Detect duplicates using stable identity:

    def find_duplicate_ids(records):
        seen = set()
        duplicates = set()
        for record in records:
            record_id = record["id"]
            if record_id in seen:
                duplicates.add(record_id)
            seen.add(record_id)
        return duplicates

## 7. Duplicate Pages

A provider may return the same page twice because of retries, cursor reuse, unstable pagination, or provider-side replay.

Page-level duplication is safe only when downstream persistence is idempotent.

## 8. Pagination Overlap

Incremental extraction may intentionally overlap windows:

    Run 1: 10:00 → 11:00
    Run 2: 10:55 → 12:00

Records from 10:55 to 11:00 may appear twice. Overlap improves late-data capture, but requires deduplication or idempotent upsert behavior.

## 9. Retry-Induced Duplicates

A timeout can create an ambiguous outcome: the provider may have processed the request even though the client did not receive the response.

A retry can therefore encounter the same logical record again.

Stable record identity protects extraction destinations. APIs that support writes may additionally require provider idempotency keys.

## 10. Deduplication vs Idempotency

These are related but different.

**Deduplication:** Have I seen this logical record more than once?

**Idempotency:** If I process the same logical record again, will the final state remain correct?

Strong pipelines commonly use both.

## 11. Database Uniqueness

Let the database enforce the final identity boundary.

    CREATE TABLE customers (
        source_id TEXT PRIMARY KEY,
        name TEXT NOT NULL,
        updated_at TIMESTAMPTZ NOT NULL
    );

Repeated delivery can then be rejected or merged instead of creating another logical entity.

## 12. PostgreSQL ON CONFLICT

Use DO NOTHING when the first durable representation should win:

    INSERT INTO customers (source_id, name, updated_at)
    VALUES (%s, %s, %s)
    ON CONFLICT (source_id) DO NOTHING;

Use DO UPDATE when later source versions should replace older versions according to an explicit rule.

## 13. Upsert for Mutable Records

Example:

    INSERT INTO customers (source_id, name, updated_at)
    VALUES (%s, %s, %s)
    ON CONFLICT (source_id)
    DO UPDATE SET
        name = EXCLUDED.name,
        updated_at = EXCLUDED.updated_at
    WHERE customers.updated_at < EXCLUDED.updated_at;

The comparison rule matters. Do not replace newer destination data with an older duplicate.

## 14. Duplicate vs Older Version

Consider the same ID observed at 10:00, 10:05, and 10:03. These are observations of one logical entity, not necessarily three entities.

Define whether the target stores every observation, only the latest version, the first version, or a full history.

## 15. Event Identity vs Entity Identity

Do not confuse event identity with entity identity.

    event e1 → customer 123 updated
    event e2 → customer 123 updated

Both events can be unique while referring to the same entity. Choose the identity according to the target model.

## 16. Tenant-Aware Deduplication

The same external ID may exist in multiple tenants.

    tenant A + customer 123
    tenant B + customer 123

These may be different logical records.

Example PostgreSQL constraint:

    CREATE UNIQUE INDEX customers_tenant_source_uidx
    ON customers (tenant_id, source_id);

## 17. Deduplication Before Persistence

Small batches can be deduplicated in memory:

    def unique_records(records):
        seen = set()
        result = []
        for record in records:
            key = record["id"]
            if key not in seen:
                seen.add(key)
                result.append(record)
        return result

This reduces unnecessary writes but cannot replace database uniqueness because duplicates can arrive across workers and runs.

## 18. Database as the Final Guard

An in-memory set exists only inside one process.

    Worker A → ID 123 ┐
                      ├→ shared database uniqueness
    Worker B → ID 123 ┘

Worker coordination improves efficiency, but correctness should have a durable shared boundary.

## 19. Deduplication and Checkpointing

Correct flow:

    READ PAGE
       ↓
    IDENTIFY RECORDS
       ↓
    DEDUPLICATE / IDEMPOTENTLY PERSIST
       ↓
    VERIFY SUCCESS
       ↓
    CHECKPOINT PAGE

Even when every record already exists, the pipeline must establish that the page was safely handled before advancing.

## 20. Incremental Extraction

Overlap windows intentionally produce duplicates. Deduplicate using stable identity rather than timestamp equality.

Two different entities can legitimately share the same timestamp.

## 21. Payload Hashes

A canonical payload hash can detect identical payloads:

    import hashlib
    import json

    def payload_hash(record):
        canonical = json.dumps(record, sort_keys=True, separators=(",", ":"))
        return hashlib.sha256(canonical.encode("utf-8")).hexdigest()

A payload hash is not necessarily business identity. Different versions of one entity can have different hashes.

## 22. Duplicate Resolution Rules

When duplicate identities disagree, define which representation wins.

Possible rules:
- latest source timestamp
- highest source version
- latest observed record
- first durable record
- explicit source sequence

Do not choose arrival order unless arrival order is part of the source contract.

## 23. Out-of-Order Duplicates

APIs may return older data after newer data.

    10:05 → version B
    10:03 → version A

If the pipeline blindly upserts arrival order, it can move the destination backward.

Use a source version or timestamp comparison only when the source contract makes it meaningful.

## 24. Deletes

Deletion events require explicit identity handling.

    id=123, state=active
    id=123, state=deleted

A deduplication rule that keeps the first observation could incorrectly preserve an entity that should be deleted.

Model deletes explicitly when the API supports them.

## 25. API Record vs File Identity

API record deduplication asks whether a logical record has already been processed.

File extraction may additionally ask whether an exact file has already been processed.

These are different identities and should not be conflated.

## 26. Multi-Worker Deduplication

Concurrent workers can receive overlapping data.

Use durable uniqueness rather than relying on perfect worker coordination.

    Worker A ─┐
              ├→ shared destination uniqueness
    Worker B ─┘

## 27. Raw Storage and Duplicates

If raw responses are preserved, do not automatically discard duplicate raw responses.

Raw duplicates can provide evidence that the provider repeated delivery, pagination overlapped, retries occurred, or source behavior changed.

Deduplicate curated logical data while retaining appropriate acquisition evidence according to retention policy.

## 28. Testing

### Unit Tests

Test duplicate IDs in one page, across pages, and across runs; same IDs with newer or older data; duplicate payloads; different payloads with the same identity; tenant-scoped IDs; deletes; and concurrent duplicate writes.

### Database Tests

Verify unique constraints and upsert rules under repeated writes.

### Integration Tests

Run the same page twice and verify that the destination remains correct.

## 29. Intentional Failure

### Failure Drill A — Duplicate Page

Process the exact same page twice. Expected: no duplicate destination entities and safe repeated processing.

### Failure Drill B — Overlapping Pages

Return one record in two consecutive pages. Expected: stable identity produces one logical entity.

### Failure Drill C — Concurrent Duplicate Writes

Make two workers persist the same record simultaneously. Expected: database uniqueness protects the destination.

### Failure Drill D — Older Version Arrives Last

Persist version 10:05, then deliver version 10:03. Expected: the newer state is preserved when source ordering semantics permit comparison.

### Failure Drill E — Same Identity, Different Payload

Send two different payloads with the same source ID. Expected: an explicit conflict/version rule is applied.

## 30. Observability

| Metric | Purpose |
|---|---|
| duplicate records detected | source/pagination quality |
| duplicate pages detected | extraction behavior |
| duplicate writes prevented | destination protection |
| unique records processed | actual logical work |
| conflict count | same identity with different payload |
| stale update count | older versions rejected |
| deduplication latency | performance |

Useful logs:

    run_id
    request_id
    source_id
    tenant_id
    dedup_reason
    existing_version
    incoming_version
    action

Do not log complete sensitive payloads merely to diagnose duplicate behavior.

## 31. Recovery

When duplicate problems are discovered:
1. Determine the identity key.
2. Determine when duplication started.
3. Identify affected runs and source positions.
4. Check whether the destination has uniqueness protection.
5. Reconcile logical counts against source counts.
6. Repair duplicate destination rows only through a controlled migration.
7. Correct extraction or persistence logic.
8. Replay affected source ranges idempotently.
9. Verify future runs remain duplicate-safe.

Do not manually delete duplicates before determining which record version is authoritative.

## 32. Production Tools You Should Know

### PostgreSQL
Use unique constraints, primary keys, indexes, and ON CONFLICT as durable deduplication mechanisms.

### Redis
Useful for distributed short-lived deduplication or coordination when appropriate, but it should not replace the authoritative destination constraint for correctness.

### Bloom Filters
A probabilistic technique for memory-efficient approximate duplicate detection when exact in-memory sets are too large.

## 33. Production Runbook

### When duplicate counts increase

1. Check pagination overlap.
2. Check retry behavior.
3. Check incremental extraction windows.
4. Check concurrent workers.
5. Check provider replay behavior.
6. Verify source identity fields.
7. Inspect destination uniqueness constraints.
8. Compare duplicate identities and versions.

### When the same identity has different payloads

- determine the authoritative version rule
- compare source timestamps or versions
- verify whether updates or deletes are involved
- preserve evidence
- correct the upsert/conflict rule
- replay affected data if necessary

### What not to do

- Do not deduplicate solely by complete JSON equality.
- Do not use mutable business fields as identity without a contract.
- Do not rely only on an in-memory set.
- Do not discard newer records because they arrived later.
- Do not assume duplicate records are identical versions.
- Do not remove duplicates manually without an authority rule.

## 34. Common Mistakes

1. Confusing identity with payload equality.
2. Using email or name as an unstable primary identity.
3. Relying only on in-memory deduplication.
4. Omitting database uniqueness.
5. Ignoring tenant scope.
6. Treating older records as newer because they arrived later.
7. Ignoring deletes.
8. Advancing checkpoints without durable duplicate-safe persistence.
9. Deduplicating raw evidence that should remain auditable.
10. Treating event identity and entity identity as the same thing.
11. Using payload hashes as business keys.
12. Failing to test concurrent duplicate writes.

## 35. Definition of Done

- [ ] Stable source identity is defined.
- [ ] Composite identity is defined where necessary.
- [ ] Duplicate sources are understood.
- [ ] In-memory duplicate detection exists where useful.
- [ ] Destination uniqueness protects final correctness.
- [ ] Upsert/conflict behavior is explicit.
- [ ] Older and newer versions are handled deliberately.
- [ ] Tenant scoping is correct.
- [ ] Deletes are represented correctly.
- [ ] Pagination overlap is duplicate-safe.
- [ ] Retries are duplicate-safe.
- [ ] Concurrent writes are duplicate-safe.
- [ ] Checkpoints advance only after safe persistence.
- [ ] Duplicate metrics exist.
- [ ] Duplicate recovery and reconciliation are documented.

## 36. What You Learned

After this recipe, you should be able to independently:
- define stable API record identity
- distinguish identity from payload equality
- detect duplicates within and across pages
- handle overlap windows
- make retries duplicate-safe
- enforce database uniqueness
- implement correct upsert rules
- handle newer and older versions
- scope identity by tenant
- model deletes correctly
- protect concurrent writes
- use hashes appropriately
- reconcile duplicate data
- recover from duplicate ingestion in production

### Core Mental Model

    RECEIVE RECORD
          ↓
    IDENTIFY LOGICAL ENTITY
          ↓
    ALREADY SEEN?
       /       \
     NO         YES
     ↓           ↓
  PERSIST    COMPARE VERSION
     ↓        /          \
    NEWER    SAME/OLDER   CONFLICT
     ↓          ↓           ↓
   UPSERT     IGNORE     APPLY RULE
      \          |           /
       └────── VERIFY ──────┘
                 ↓
            CHECKPOINT

> Deduplication is an identity problem first and a data-cleaning problem second. Define the logical identity, enforce it durably, and make repeated delivery safe.