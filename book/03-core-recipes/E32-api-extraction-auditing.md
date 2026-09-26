# E32 — API Extraction Auditing

## 1. Problem Recognition

A production API extractor needs durable evidence of what it processed. Logs, metrics, and traces are useful but do not by themselves provide a structured extraction history.

An audit should answer: which run, source, resource, tenant, trigger, application version, schema contract, requests, counts, checkpoints, failures, and final outcome were involved?

### Audit vs logging

| Mechanism | Purpose |
|---|---|
| Logs | Troubleshooting |
| Metrics | Measurement |
| Traces | Distributed execution |
| Extraction audit | Durable extraction evidence |

## 2. Core Pattern

```text
START EXTRACTION
      ↓
CREATE RUN AUDIT RECORD
      ↓
RECORD REQUEST METADATA
      ↓
RECORD RESPONSE METADATA
      ↓
RECORD DATA COUNTS
      ↓
RECORD CHECKPOINT MOVEMENT
      ↓
RECORD OUTCOME
      ↓
RECONCILE
      ↓
CLOSE AUDIT RUN
```

> Logs tell you what the process said happened; an audit trail records the durable evidence of what the extraction did.

Audit data should contain metadata and evidence, not become a raw API payload store.

## 3. Audit Model

Use two levels:

```text
extraction_audit_run
        │
        ├── request 001
        ├── request 002
        └── request ...
```

Run-level fields should include `run_id`, pipeline, extraction, source, resource, tenant, partition, trigger type, initiator, environment, application/deployment/configuration versions, schema/contract fingerprint, timestamps, status, aggregate counts, checkpoint-before/after, and error information.

Request-level fields should include request sequence, endpoint, page/offset/cursor fingerprint, correlation ID, timestamps, HTTP status, counts, checksum, outcome, and classified error.

Use explicit outcomes such as `RUNNING`, `SUCCEEDED`, `PARTIAL`, `FAILED`, and `CANCELLED`.

## 4. PostgreSQL Implementation

### Run table

```sql
CREATE TABLE extraction_audit_run (
    run_id UUID PRIMARY KEY,
    pipeline_name TEXT NOT NULL,
    extraction_name TEXT NOT NULL,
    source_system TEXT NOT NULL,
    resource_name TEXT NOT NULL,
    tenant_id TEXT,
    partition_key TEXT,
    trigger_type TEXT NOT NULL CHECK (trigger_type IN ('scheduled','manual','replay','backfill')),
    initiated_by TEXT,
    environment TEXT NOT NULL,
    application_version TEXT,
    deployment_version TEXT,
    configuration_version TEXT,
    schema_version TEXT,
    contract_fingerprint TEXT,
    started_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    status TEXT NOT NULL CHECK (status IN ('RUNNING','SUCCEEDED','PARTIAL','FAILED','CANCELLED')),
    records_requested BIGINT NOT NULL DEFAULT 0,
    records_received BIGINT NOT NULL DEFAULT 0,
    records_accepted BIGINT NOT NULL DEFAULT 0,
    records_rejected BIGINT NOT NULL DEFAULT 0,
    records_duplicated BIGINT NOT NULL DEFAULT 0,
    records_persisted BIGINT NOT NULL DEFAULT 0,
    records_failed BIGINT NOT NULL DEFAULT 0,
    records_quarantined BIGINT NOT NULL DEFAULT 0,
    checkpoint_before JSONB,
    checkpoint_after JSONB,
    error_code TEXT,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_run_started ON extraction_audit_run (started_at);
CREATE INDEX idx_audit_run_status ON extraction_audit_run (status);
CREATE INDEX idx_audit_run_source_resource ON extraction_audit_run (source_system, resource_name);
```

### Request table

```sql
CREATE TABLE extraction_audit_request (
    request_audit_id BIGSERIAL PRIMARY KEY,
    run_id UUID NOT NULL REFERENCES extraction_audit_run(run_id),
    request_sequence BIGINT NOT NULL,
    endpoint TEXT NOT NULL,
    http_method TEXT NOT NULL,
    page_number BIGINT,
    offset_value BIGINT,
    cursor_fingerprint TEXT,
    correlation_id TEXT,
    requested_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    http_status INTEGER,
    records_requested BIGINT NOT NULL DEFAULT 0,
    records_received BIGINT NOT NULL DEFAULT 0,
    records_accepted BIGINT NOT NULL DEFAULT 0,
    records_rejected BIGINT NOT NULL DEFAULT 0,
    records_duplicated BIGINT NOT NULL DEFAULT 0,
    records_persisted BIGINT NOT NULL DEFAULT 0,
    response_checksum TEXT,
    outcome TEXT NOT NULL CHECK (outcome IN ('SUCCEEDED','FAILED','RETRIED','RATE_LIMITED','TIMEOUT','VALIDATION_FAILED')),
    error_code TEXT,
    error_message TEXT,
    UNIQUE (run_id, request_sequence)
);

CREATE INDEX idx_audit_request_run ON extraction_audit_request (run_id, request_sequence);
```

## 5. Python Audit Writer

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from uuid import UUID, uuid4

import psycopg


@dataclass
class RunAudit:
    run_id: UUID
    started_at: datetime


class AuditWriter:
    def __init__(self, connection: psycopg.Connection):
        self.connection = connection

    def start_run(self, *, pipeline_name, extraction_name, source_system,
                  resource_name, trigger_type, environment,
                  application_version, deployment_version,
                  configuration_version, schema_version,
                  contract_fingerprint, initiated_by,
                  checkpoint_before):
        run_id = uuid4()
        started_at = datetime.now(timezone.utc)

        with self.connection.cursor() as cur:
            cur.execute(
                """
                INSERT INTO extraction_audit_run (
                    run_id, pipeline_name, extraction_name, source_system,
                    resource_name, trigger_type, initiated_by, environment,
                    application_version, deployment_version,
                    configuration_version, schema_version,
                    contract_fingerprint, started_at, status,
                    checkpoint_before
                ) VALUES (
                    %s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,'RUNNING',%s
                )
                """,
                (run_id, pipeline_name, extraction_name, source_system,
                 resource_name, trigger_type, initiated_by, environment,
                 application_version, deployment_version,
                 configuration_version, schema_version,
                 contract_fingerprint, started_at, checkpoint_before)
            )
        self.connection.commit()
        return RunAudit(run_id=run_id, started_at=started_at)
```

Create the run audit before extraction continues. If the process crashes immediately afterward, the durable `RUNNING` record remains.

## 6. Request Auditing

```python
from datetime import datetime, timezone


def record_request(connection, *, run_id, request_sequence, endpoint,
                   http_method, page_number, offset_value,
                   cursor_fingerprint, correlation_id, http_status,
                   records_requested, records_received, records_accepted,
                   records_rejected, records_duplicated, records_persisted,
                   response_checksum, outcome, error_code=None,
                   error_message=None):
    completed_at = datetime.now(timezone.utc)

    with connection.cursor() as cur:
        cur.execute(
            """
            INSERT INTO extraction_audit_request (
                run_id, request_sequence, endpoint, http_method,
                page_number, offset_value, cursor_fingerprint,
                correlation_id, requested_at, completed_at, http_status,
                records_requested, records_received, records_accepted,
                records_rejected, records_duplicated, records_persisted,
                response_checksum, outcome, error_code, error_message
            ) VALUES (
                %s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s
            )
            """,
            (run_id, request_sequence, endpoint, http_method,
             page_number, offset_value, cursor_fingerprint,
             correlation_id, completed_at, completed_at, http_status,
             records_requested, records_received, records_accepted,
             records_rejected, records_duplicated, records_persisted,
             response_checksum, outcome, error_code, error_message)
        )
    connection.commit()
```

HTTP 200 alone does not mean extraction success. Validate, process, persist, verify, then record the meaningful outcome.

## 7. Cursor and Response Fingerprints

```python
import hashlib


def fingerprint(value: str | None) -> str | None:
    if value is None:
        return None
    return hashlib.sha256(value.encode("utf-8")).hexdigest()


def response_checksum(body: bytes) -> str:
    return hashlib.sha256(body).hexdigest()
```

Fingerprints allow comparison without unnecessarily retaining sensitive or opaque provider state.

## 8. Run Counts and Completion

Prefer durable request records over process-memory counters.

```sql
UPDATE extraction_audit_run AS r
SET
    records_requested = x.records_requested,
    records_received = x.records_received,
    records_accepted = x.records_accepted,
    records_rejected = x.records_rejected,
    records_duplicated = x.records_duplicated,
    records_persisted = x.records_persisted
FROM (
    SELECT run_id,
           SUM(records_requested) records_requested,
           SUM(records_received) records_received,
           SUM(records_accepted) records_accepted,
           SUM(records_rejected) records_rejected,
           SUM(records_duplicated) records_duplicated,
           SUM(records_persisted) records_persisted
    FROM extraction_audit_request
    WHERE run_id = %s
    GROUP BY run_id
) x
WHERE r.run_id = x.run_id;
```

Close successful work explicitly:

```sql
UPDATE extraction_audit_run
SET status = 'SUCCEEDED', completed_at = now(),
    checkpoint_after = %s, updated_at = now()
WHERE run_id = %s AND status = 'RUNNING';
```

Close failures explicitly:

```sql
UPDATE extraction_audit_run
SET status = 'FAILED', completed_at = now(),
    error_code = %s, error_message = %s, updated_at = now()
WHERE run_id = %s AND status = 'RUNNING';
```

A process exit is not proof of extraction success.

## 9. Append-Only Audit Events

For high-fidelity lifecycle reconstruction:

```sql
CREATE TABLE extraction_audit_event (
    audit_event_id BIGSERIAL PRIMARY KEY,
    run_id UUID NOT NULL REFERENCES extraction_audit_run(run_id),
    event_type TEXT NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    sequence_number BIGINT NOT NULL,
    actor TEXT,
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    UNIQUE (run_id, sequence_number)
);
```

Useful events include `RUN_STARTED`, `REQUEST_STARTED`, `REQUEST_COMPLETED`, `CHECKPOINT_ADVANCED`, `RECORDS_QUARANTINED`, `RUN_PARTIAL`, `RUN_COMPLETED`, and `RUN_FAILED`.

Make audit delivery idempotent:

```sql
INSERT INTO extraction_audit_event (
    run_id, event_type, occurred_at, sequence_number, actor, metadata
)
VALUES (%s,%s,%s,%s,%s,%s)
ON CONFLICT (run_id, sequence_number) DO NOTHING;
```

## 10. Checkpoint and Audit Ordering

The safe relationship is:

```text
REQUEST
  ↓
VALIDATE
  ↓
PERSIST DATA
  ↓
VERIFY / COMMIT
  ↓
ADVANCE CHECKPOINT
  ↓
AUDIT THE PROGRESS
```

The audit record must never be used as evidence that data is safe merely because an audit row exists.

## 11. Replay, Backfill, and Multi-Tenant Runs

Record `trigger_type`, `parent_run_id`, `initiated_by`, and an optional reason.

```text
Run A → scheduled
Run B → replay → parent Run A
Run C → backfill
```

For multi-tenant extraction, preserve tenant-level state:

```text
Run 100
 ├── Tenant A → SUCCESS
 ├── Tenant B → FAILED
 └── Tenant C → SUCCESS
```

An aggregate run may therefore be `PARTIAL`.

## 12. Security and Privacy

Never store passwords, API keys, bearer tokens, refresh tokens, cookies, authorization headers, or unnecessary personal/sensitive data.

Prefer:

```text
credential_identity
tenant_id
resource
correlation_id
contract fingerprint
response checksum
counts
status
```

Use automated tests to prove secrets never reach audit storage.

## 13. Time Handling

Use UTC-aware timestamps:

```python
from datetime import datetime, timezone
now = datetime.now(timezone.utc)
```

Use monotonic time for local duration measurement:

```python
import time
started = time.monotonic()
# extraction
elapsed = time.monotonic() - started
```

Use UTC timestamps for cross-system chronology and monotonic clocks for elapsed duration.

## 14. Testing

### Unit tests

Test that:

- run creation generates a unique ID;
- source/resource/trigger are recorded;
- checkpoint-before is recorded;
- request sequence is unique;
- HTTP outcomes and counts are stored;
- failed, rate-limited, timeout, and validation outcomes are distinct;
- success/failure/partial completion states are explicit;
- duplicate audit events are ignored;
- secret values never appear in audit records.

### Integration tests

Verify:

- PostgreSQL constraints reject invalid status values;
- duplicate request sequence cannot be inserted twice;
- request totals can be aggregated into run totals;
- stale `RUNNING` runs can be found;
- replay runs link to their parent;
- checkpoint state is unchanged when persistence fails.

### Crash tests

Test both:

```text
request succeeds → crash before persistence
```

and:

```text
persist succeeds → crash before final audit update
```

The first must not advance the checkpoint. The second must leave evidence that the run needs investigation rather than silently becoming a success.

## 15. Intentional Failure Drills

### Drill 1 — Kill the process after run creation

Expected: durable `RUNNING` state and a detectable stale run.

### Drill 2 — Force HTTP 500

Expected: request failure evidence and a correct run outcome.

### Drill 3 — Force HTTP 429

Expected: rate-limit evidence and visible retry behavior.

### Drill 4 — Corrupt a response

Expected: validation failure and unchanged checkpoint.

### Drill 5 — Duplicate an audit event

Expected: one logical event.

### Drill 6 — Disconnect the audit database

Observe whether the system fails closed, buffers, or continues in explicitly degraded mode. This policy must be intentional.

## 16. Reconciliation

Useful query:

```sql
SELECT
    run_id,
    records_received,
    records_accepted,
    records_rejected,
    records_duplicated,
    records_persisted,
    records_failed,
    records_quarantined
FROM extraction_audit_run
WHERE run_id = %s;
```

Then compare audit evidence with destination state.

Do not blindly assume `received - rejected - duplicates = persisted`. Define the count semantics for the pipeline.

## 17. Observability

Track:

```text
extraction_runs_started_total
extraction_runs_succeeded_total
extraction_runs_failed_total
extraction_runs_partial_total
records_received_total
records_accepted_total
records_rejected_total
records_duplicated_total
records_persisted_total
records_quarantined_total
audit_write_failures_total
stale_running_extractions
audit_event_duplicates_total
audit_write_latency
```

Failed/partial runs:

```sql
SELECT run_id, source_system, resource_name,
       started_at, completed_at, error_code, error_message
FROM extraction_audit_run
WHERE status IN ('FAILED','PARTIAL')
ORDER BY started_at DESC;
```

Stale runs:

```sql
SELECT run_id, pipeline_name, extraction_name, started_at
FROM extraction_audit_run
WHERE status = 'RUNNING'
  AND started_at < now() - INTERVAL '2 hours';
```

Request history:

```sql
SELECT request_sequence, http_status, records_received,
       records_persisted, outcome, error_code
FROM extraction_audit_request
WHERE run_id = %s
ORDER BY request_sequence;
```

## 18. Retention and Scale

Detailed request audit can grow rapidly. Define retention explicitly and consider time-based PostgreSQL partitioning for high-volume systems.

Example structure:

```text
extraction_audit_request
    ├── 2026-09
    ├── 2026-10
    └── 2026-11
```

The correct retention period comes from operational, contractual, and regulatory requirements.

## 19. Production Tools You Should Know

### PostgreSQL
Durable audit records, constraints, uniqueness, reconciliation, retention, and partitioning.

### OpenTelemetry
Trace IDs and distributed correlation. It complements rather than replaces extraction auditing.

### DataHub
Metadata and lineage capabilities. It complements run auditing rather than replacing it.

## 20. Production Runbook

### Before deployment

- [ ] Apply audit schema migrations.
- [ ] Verify indexes and uniqueness constraints.
- [ ] Verify retention policy.
- [ ] Verify secret redaction.
- [ ] Verify UTC timestamps.
- [ ] Verify trigger types.
- [ ] Verify application/configuration version capture.
- [ ] Verify database permissions.

### During extraction

Check:

1. Was a run created?
2. Are request records appearing?
3. Are counts progressing?
4. Is checkpoint movement after durable persistence?
5. Are failures classified?
6. Are audit writes succeeding?

### If a run is stuck

```sql
SELECT * FROM extraction_audit_run WHERE run_id = %s;

SELECT *
FROM extraction_audit_request
WHERE run_id = %s
ORDER BY request_sequence;
```

Identify the last successful request, last safe checkpoint, destination state, and whether resume/replay is safe.

### If counts do not reconcile

Check request counts, validation failures, duplicates, persistence failures, quarantine records, destination counts, and checkpoint state.

Do not manually alter historical evidence merely to make the numbers match.

### If audit storage fails

Follow the predefined policy: fail extraction, buffer audit events, continue in explicitly degraded mode, or stop checkpoint advancement. Do not invent this policy during an incident.

### What not to do

- Do not store credentials.
- Do not treat logs as the only durable history.
- Do not call a run successful because the process exited normally.
- Do not advance checkpoints because an audit row was written.
- Do not mix replay/backfill with scheduled runs.
- Do not destructively rewrite historical evidence.

## 21. Common Mistakes

1. Treating logs as an audit trail.
2. Recording only final counts.
3. Storing complete payloads in the audit database.
4. Storing credentials.
5. Missing failed-run records.
6. Having no stable run identity.
7. Having no trigger identity.
8. Trusting in-memory counters.
9. Treating audit success as data success.
10. Ignoring audit-system failure.
11. Advancing checkpoints from audit code.

## 22. Definition of Done

- [ ] Explain logs, metrics, traces, and audit records.
- [ ] Create a durable run record.
- [ ] Assign a unique `run_id`.
- [ ] Record source/resource/tenant/partition identity.
- [ ] Distinguish scheduled, manual, replay, and backfill runs.
- [ ] Record application and configuration versions.
- [ ] Record schema/contract identity.
- [ ] Record request-level evidence.
- [ ] Record response status and counts.
- [ ] Record checkpoint-before and checkpoint-after.
- [ ] Record explicit terminal states.
- [ ] Prevent duplicate audit events.
- [ ] Prove secrets are not stored.
- [ ] Reconcile request and run counts.
- [ ] Detect stale `RUNNING` runs.
- [ ] Test crash scenarios.
- [ ] Operate the audit system using runbook queries.

## 23. What You Learned

You learned to model API extraction as a provable operational process rather than only a script that makes HTTP requests.

You should now be able to:

1. Create durable extraction run and request records.
2. Preserve source, execution, contract, and progress metadata.
3. Track counts with explicit semantics.
4. Distinguish scheduled, manual, replay, and backfill work.
5. Make audit writes idempotent.
6. Keep secrets and raw payloads out of audit records.
7. Reconcile audit evidence with persisted data.
8. Detect incomplete and stale runs.
9. Test failures around persistence, checkpoints, and audit storage.
10. Operate extraction history during incidents.

Final mental model:

```text
EXTRACTION
    ↓
DURABLE DATA
    ↓
CHECKPOINT
    ↓
AUDIT EVIDENCE
    ↓
RECONCILIATION
    ↓
PROVABLE OUTCOME
```

A production Data Engineer should be able to answer not only "Did the pipeline run?" but "What source did it process, what did it receive, what did it persist, what progress did it make, what failed, and what evidence proves the final state?"