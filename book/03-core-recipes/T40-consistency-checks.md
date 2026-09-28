# T40 — Consistency Checks

> **Goal:** Prove that data which exists is internally coherent: related fields, records, aggregates, states, units, and independent sources agree according to explicit business rules.

Completeness asks whether the expected population exists. Consistency asks whether the facts within that population agree with each other.

## 1. Problem Recognition

A pipeline can be complete, unique, typed, and referentially valid while still producing contradictory data.

Typical failures include:
- order total disagrees with the sum of order lines;
- start time occurs after end time;
- a payment marked SETTLED has no settlement timestamp;
- child and parent tenant IDs disagree;
- a daily aggregate disagrees with its detailed transactions;
- a balance does not equal opening balance plus movements;
- an event history contains an illegal state transition;
- percentages do not sum to the expected total;
- two systems report different facts for the same business transaction;
- a monetary value uses incompatible currency or unit context.

The key observation is:

> **Each individual value can be valid while the relationship between values is invalid.**

### 1.1 Consistency dimensions

| Dimension | Question | Example |
|---|---|---|
| Field-to-field | Do related columns agree? | start_at <= end_at |
| Record-to-record | Do related records agree? | Order total vs line totals |
| Parent-child | Does child state agree with parent scope? | Tenant IDs match |
| Aggregate-detail | Does summary reconcile to detail? | Daily total vs transactions |
| Temporal | Are events ordered coherently? | Settlement after authorization |
| State | Are transitions allowed? | CANCELLED → SETTLED prohibited |
| Unit | Are values expressed in compatible units? | cents vs dollars |
| Currency | Is monetary context consistent? | EUR amount with USD metadata |
| Cross-source | Do independent sources agree? | Ledger vs processor |
| Derived | Does a calculated value agree with source facts? | Tax total |
| Snapshot | Does state agree with its cutoff? | Active-user count |

### 1.2 Consistency versus other quality dimensions

- **Completeness:** Did expected data arrive?
- **Uniqueness:** Does each logical identity occur once?
- **Referential integrity:** Does a relationship point to an existing parent?
- **Consistency:** Do related facts agree?
- **Validity:** Is each value acceptable by its own type, domain, and range?

A record can pass several dimensions and still fail consistency.

### 1.3 Define the invariant

Start with an explicit invariant:

    order_total = sum(order_line.extended_amount)
    start_at <= end_at
    settled_at IS NOT NULL when status = SETTLED
    child.tenant_id = parent.tenant_id
    opening_balance + credits - debits = closing_balance

Write the invariant before writing SQL.

### 1.4 Classify the invariant

Useful categories:
- arithmetic;
- temporal;
- state-machine;
- relational;
- aggregate;
- unit;
- currency;
- cross-source;
- derived-value;
- scope/tenant;
- conditional.

### 1.5 Hard versus soft consistency

A hard invariant may require exact equality. A soft invariant may permit a documented tolerance.

Example:

    abs(reported_total - calculated_total) <= 0.01

The tolerance must be defined before the failure occurs and versioned with the rule.

## 2. Concept and Reasoning

### 2.1 Consistency is relational

Suppose:

    quantity = 2
    unit_price = 50.00
    line_total = 150.00

Every value is individually plausible, but the relationship violates:

    line_total = quantity * unit_price

That is a consistency failure.

### 2.2 Local versus global consistency

**Local consistency** can be checked within one row, such as start_at <= end_at.

**Global consistency** requires multiple records, such as invoice.total = sum(invoice_lines.amount).

Use the smallest grain that fully represents the invariant.

### 2.3 Field-to-field consistency

Examples:

    SELECT order_id, ordered_at, shipped_at
    FROM orders
    WHERE shipped_at IS NOT NULL
      AND ordered_at IS NOT NULL
      AND shipped_at < ordered_at;

Conditional example:

    SELECT payment_id, status, settled_at
    FROM payments
    WHERE status = 'SETTLED'
      AND settled_at IS NULL;

### 2.4 Aggregate-detail consistency

An invoice total can be valid by itself while disagreeing with its line items.

Always calculate detail at the intended grain before comparing it with a parent aggregate.

### 2.5 Parent-child consistency

Referential integrity proves that a parent exists. Consistency can require semantic agreement as well:

    child.tenant_id = parent.tenant_id
    child.currency = parent.currency
    child.status is compatible with parent.status

### 2.6 Temporal consistency

Typical invariants include:

    created_at <= updated_at
    authorized_at <= captured_at
    captured_at <= settled_at
    valid_from < valid_to

Define whether the comparison uses event time, processing time, effective time, or source time.

### 2.7 State-machine consistency

Business entities often have legal transitions:

    CREATED → AUTHORIZED → CAPTURED → SETTLED

An observed transition such as SETTLED → AUTHORIZED may be invalid even though both individual states are valid.

### 2.8 Aggregate reconciliation

For a balance:

    opening_balance + credits - debits = closing_balance

The equation is stronger than checking whether each component is merely inside a range.

### 2.9 Derived-value consistency

If:

    gross = net + tax

test the relationship directly.

### 2.10 Units and currency

Numeric equality does not prove semantic equality.

Examples:
- cents versus dollars;
- kilograms versus grams;
- EUR versus USD;
- transaction currency versus settlement currency.

If conversion is allowed, the conversion factor, unit, effective time, and rounding policy must be part of the invariant.

### 2.11 Cross-source consistency

Two systems may independently report the same business fact.

A mismatch can result from timing, fees, FX, rounding, missing records, duplicates, source correction, or actual corruption.

A consistency failure is therefore an investigation trigger, not automatic proof that one source is wrong.

### 2.12 Snapshot consistency

Compare snapshots only when their population definition, cutoff time, scope, and source version agree.

Example:

    reported_active_count = 10000
    detailed_active_count = 9700

This is meaningful only if both counts describe the same population.

### 2.13 NULL semantics

NULL can make predicates appear healthy.

For example, start_at > end_at does not identify rows where either value is NULL.

Decide whether NULL means unknown, not applicable, not yet available, or invalid. Then test presence separately when required.

### 2.14 Correct grain

If an invoice has many lines:

    invoice_id → SUM(line.amount)

Do not join another one-to-many table first and then aggregate. Fan-out can manufacture a false inconsistency.

## 3. Implementation

### 3.1 Rule registry

    CREATE TABLE consistency_rule (
        rule_id text PRIMARY KEY,
        rule_name text NOT NULL,
        dataset_name text NOT NULL,
        rule_type text NOT NULL,
        severity text NOT NULL,
        tolerance numeric(20,8),
        enabled boolean NOT NULL DEFAULT true,
        rule_version text NOT NULL,
        created_at timestamptz NOT NULL DEFAULT now()
    );

Useful rule types include FIELD, TEMPORAL, AGGREGATE, STATE, RELATIONAL, CROSS_SOURCE, UNIT, and DERIVED.

### 3.2 Result table

    CREATE TABLE consistency_result (
        result_id bigserial PRIMARY KEY,
        rule_id text NOT NULL REFERENCES consistency_rule(rule_id),
        run_id uuid NOT NULL,
        record_key text,
        status text NOT NULL,
        observed_value jsonb,
        expected_value jsonb,
        evaluated_at timestamptz NOT NULL DEFAULT now()
    );

Avoid storing sensitive raw payloads in diagnostics unless explicitly permitted.

### 3.3 Field-to-field SQL

    SELECT order_id, ordered_at, shipped_at
    FROM orders
    WHERE shipped_at IS NOT NULL
      AND ordered_at IS NOT NULL
      AND shipped_at < ordered_at;

### 3.4 Arithmetic consistency

    SELECT
        line_id,
        quantity,
        unit_price,
        line_total,
        quantity * unit_price AS calculated_total
    FROM order_lines
    WHERE line_total IS DISTINCT FROM quantity * unit_price;

For money, use the same decimal scale and rounding policy as the business contract.

### 3.5 Aggregate-detail consistency

Pre-aggregate detail before joining unrelated one-to-many tables.

    WITH line_totals AS (
        SELECT invoice_id, sum(line_total) AS calculated_total
        FROM invoice_lines
        GROUP BY invoice_id
    )
    SELECT i.invoice_id,
           i.total AS reported_total,
           l.calculated_total
    FROM invoices i
    JOIN line_totals l ON l.invoice_id = i.invoice_id
    WHERE i.total IS DISTINCT FROM l.calculated_total;

### 3.6 Balance consistency

    SELECT account_id,
           opening_balance,
           credits,
           debits,
           closing_balance,
           opening_balance + credits - debits AS calculated_closing
    FROM daily_account_balance
    WHERE closing_balance IS DISTINCT FROM
          opening_balance + credits - debits;

With a documented tolerance:

    WHERE abs(
        closing_balance - (opening_balance + credits - debits)
    ) > 0.01;

### 3.7 Parent-scope consistency

    SELECT c.child_id,
           c.tenant_id AS child_tenant,
           p.tenant_id AS parent_tenant
    FROM child_records c
    JOIN parent_records p ON p.parent_id = c.parent_id
    WHERE c.tenant_id IS DISTINCT FROM p.tenant_id;

### 3.8 Temporal consistency

    SELECT payment_id
    FROM payments
    WHERE authorized_at IS NOT NULL
      AND captured_at IS NOT NULL
      AND captured_at < authorized_at;

### 3.9 State-transition validation

    CREATE TABLE allowed_status_transition (
        entity_type text NOT NULL,
        from_status text NOT NULL,
        to_status text NOT NULL,
        PRIMARY KEY (entity_type, from_status, to_status)
    );

Compare observed transitions against this table:

    SELECT h.entity_id, h.from_status, h.to_status
    FROM status_history h
    LEFT JOIN allowed_status_transition a
      ON a.entity_type = h.entity_type
     AND a.from_status = h.from_status
     AND a.to_status = h.to_status
    WHERE a.entity_type IS NULL;

### 3.10 Sequence-based state validation

Use LAG over a deterministic event order:

    LAG(status) OVER (
        PARTITION BY payment_id
        ORDER BY occurred_at, event_id
    )

Then validate the previous-state to current-state transition.

Equal timestamps require a deterministic tie-breaker.

### 3.11 Unit consistency

    SELECT amount_id, amount, unit
    FROM monetary_values
    WHERE unit NOT IN ('USD_CENTS', 'EUR_CENTS');

For conversions, validate the factor, source unit, target unit, effective timestamp, and rounding policy.

### 3.12 Currency consistency

    SELECT line_id, invoice_id, line_currency, invoice_currency
    FROM invoice_lines
    JOIN invoices USING (invoice_id)
    WHERE line_currency <> invoice_currency;

Use this only when the business contract requires matching currencies. A multi-currency model should instead require explicit conversion or allocation evidence.

### 3.13 Cross-source reconciliation

    SELECT a.transaction_id,
           a.amount AS ledger_amount,
           b.amount AS processor_amount
    FROM ledger_settlements a
    JOIN processor_settlements b
      ON b.transaction_id = a.transaction_id
    WHERE abs(a.amount - b.amount) > 0.01;

Before declaring a defect, align currency, fees, FX, cutoff, source version, and rounding semantics.

### 3.14 Snapshot consistency

    WITH detail AS (
        SELECT count(*)::bigint AS active_count
        FROM customers
        WHERE status = 'ACTIVE'
          AND snapshot_date = DATE '2026-09-28'
    )
    SELECT s.reported_active_count, d.active_count
    FROM daily_snapshot s
    CROSS JOIN detail d
    WHERE s.snapshot_date = DATE '2026-09-28'
      AND s.reported_active_count <> d.active_count;

### 3.15 Python invariant checker

    from dataclasses import dataclass
    from decimal import Decimal

    @dataclass(frozen=True)
    class CheckResult:
        rule_id: str
        passed: bool
        reason: str

    def check_balance(rule_id, opening, credits, debits, closing, tolerance=Decimal('0')):
        calculated = opening + credits - debits
        difference = abs(closing - calculated)
        return CheckResult(
            rule_id=rule_id,
            passed=difference <= tolerance,
            reason='within tolerance' if difference <= tolerance else f'difference={difference}',
        )

Keep invariant evaluation pure and deterministic.

### 3.16 Separate detection from disposition

A detector should answer:

    does this record violate the invariant?

A downstream policy decides:

    reject
    quarantine
    warn
    correct
    accept with tolerance

Do not embed irreversible correction inside a generic consistency check.

### 3.17 Rule versioning and evidence

Persist rule ID and version with every evaluation.

Useful evidence includes:
- rule ID and version;
- run ID;
- dataset and scope;
- affected record or aggregate key;
- observed relationship;
- expected relationship;
- tolerance;
- evaluation timestamp;
- source lineage.

### 3.18 Privacy-safe diagnostics

Prefer rule ID, run ID, dataset, record-key hash, and summarized values over storing sensitive transaction payloads.

## 4. Testing

Consistency tests must prove both violation detection and false-positive resistance.

### 4.1 Field tests

Test:
- valid interval;
- reversed interval;
- NULL start;
- NULL end;
- both NULL;
- boundary equality.

### 4.2 Arithmetic tests

Test:
- exact equality;
- one-cent difference;
- tolerance boundary;
- tolerance exceeded;
- rounding before comparison;
- rounding after comparison;
- decimal precision.

### 4.3 Aggregate tests

Test:
- detail exactly matches parent;
- one detail record missing;
- duplicate detail record;
- unrelated join causing fan-out;
- parent with zero details;
- expected-zero aggregate.

### 4.4 Temporal tests

Test:
- valid lifecycle;
- impossible ordering;
- equal timestamps;
- NULL timestamps;
- timezone normalization;
- deterministic tie-breaking.

### 4.5 State-machine tests

Test:
- every allowed transition;
- every prohibited transition;
- repeated same-state event if allowed;
- missing previous state;
- duplicate state event;
- out-of-order event;
- unknown status.

### 4.6 Parent-scope tests

Test:
- matching tenant;
- mismatched tenant;
- NULL tenant;
- parent with different currency;
- parent state incompatible with child state.

### 4.7 Cross-source tests

Test:
- exact match;
- allowed rounding difference;
- currency mismatch;
- fee difference;
- cutoff mismatch;
- missing counterpart;
- duplicate counterpart.

### 4.8 False-positive tests

Explicitly test legitimate exceptions:
- correctly modeled multi-currency data;
- zero-value transactions;
- permitted settlement paths without capture;
- valid zero-activity aggregates;
- late corrections that supersede an earlier value.

A good consistency system rejects contradictions without rejecting legitimate business states.

### 4.9 Idempotence

Running the same rule against unchanged data should produce the same violations and should not create duplicate incidents.

## 5. Observability

Consistency checks need evidence that explains failure volume and affected relationships.

### 5.1 Core metrics

| Metric | Meaning |
|---|---|
| consistency_rules_evaluated_total | Rule evaluations |
| consistency_failures_total | Violations |
| consistency_failure_rate | Violations divided by evaluated population |
| consistency_critical_failures_total | Critical violations |
| consistency_aggregate_difference | Reconciliation difference |
| consistency_state_transition_failures_total | Invalid transitions |
| consistency_temporal_failures_total | Invalid temporal relationships |
| consistency_cross_source_mismatches_total | Cross-system mismatches |
| consistency_evaluation_lag_seconds | Delay between readiness and evaluation |

### 5.2 Dimensions

Use rule ID, dataset, environment, source, severity, rule version, data date, and tenant where appropriate.

Avoid raw customer or transaction identifiers in metric labels.

### 5.3 Failure-rate interpretation

A small inconsistency rate can be acceptable for one dataset and unacceptable for another. Interpret metrics against rule severity and the business contract.

### 5.4 Diagnostic storage

Retain enough evidence to answer:
1. Which rule failed?
2. Which run and population were affected?
3. What relationship was expected?
4. What relationship was observed?
5. What tolerance applied?
6. Which rule version evaluated it?

### 5.5 Alerts

Alert on critical invariant violations, sudden failure-rate changes, new failures after deployment, aggregate reconciliation breaks, invalid state transitions, cross-source mismatch spikes, and repeated failures in one partition.

Do not page on every low-severity discrepancy.

## 6. Intentional Failure

Break consistency deliberately.

### Failure 1 — Change an order total

Set an invoice total to 100 while its lines sum to 95.

Expected:

    aggregate_difference = 5
    status = FAILED

### Failure 2 — Reverse timestamps

Set ordered_at = 10:00 and shipped_at = 09:00.

The temporal rule must fail.

### Failure 3 — Break a state transition

Insert SETTLED → AUTHORIZED.

The state-machine check must identify the transition as invalid.

### Failure 4 — Create join fan-out

Duplicate one line in a joined detail table.

A poorly designed aggregate check may produce a false difference. The correct implementation calculates the invariant at the intended grain.

### Failure 5 — Break tenant consistency

Change a child record tenant while retaining its valid parent ID.

Referential existence still succeeds. T40 must fail the semantic scope invariant.

### Failure 6 — Introduce currency mismatch

Set a line currency to USD while the invoice requires EUR.

The currency rule must fail when the contract requires matching currencies.

### Failure 7 — Introduce duplicate compensation

Remove one transaction and duplicate another so total row count remains constant.

Count checks may pass. Identity and consistency checks should expose the resulting contradiction where applicable.

### Failure 8 — Use floating-point money

Use binary floating-point for a decimal monetary invariant.

The test should demonstrate why exact decimal or numeric arithmetic is required.

## 7. Recovery

Recovery must preserve source evidence and avoid silently correcting facts.

### 7.1 Field inconsistency

1. Identify the rule and affected records.
2. Determine the authoritative source field.
3. Check source lineage.
4. Correct upstream data if possible.
5. Replay the affected scope.
6. Re-run the invariant.

### 7.2 Aggregate mismatch

1. Compare parent and detail at the correct grain.
2. Check duplicate and fan-out effects.
3. Check missing or late detail.
4. Check rounding and currency policy.
5. Recompute from authoritative detail.
6. Reconcile before publication.

### 7.3 State inconsistency

1. Materialize event history.
2. Order events deterministically.
3. Identify the first invalid transition.
4. Determine whether an event is missing, late, duplicated, or incorrectly mapped.
5. Correct source data or replay the sequence.
6. Recompute current state.

Do not simply overwrite the final status.

### 7.4 Cross-source mismatch

1. Confirm both sources use the same transaction identity.
2. Align cutoff times.
3. Compare currencies and units.
4. Compare fee and FX treatment.
5. Determine whether one source is delayed.
6. Reconcile using authoritative evidence.
7. Record the disposition.

### 7.5 False-positive recovery

Sometimes the data is correct and the rule is wrong.

If a newly supported business case legitimately violates an old invariant:

1. confirm the business contract;
2. document the exception;
3. update the rule;
4. version the rule;
5. re-evaluate historical failures where appropriate;
6. preserve original evidence.

Do not weaken a rule silently.

### 7.6 Safe replay

Replay the smallest affected scope and preserve run ID, source identity, business key, rule version, original evidence, correction reason, and replay boundary.

Then re-run all dependent consistency checks.

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

Use PostgreSQL for SQL invariants, aggregate reconciliation, temporal checks, state-transition queries, relational comparisons, and durable rule results.

The underlying skill is translating a business invariant into a query that cannot hide the violation.

### 8.2 dbt

dbt tests can express relationship and model-level assertions alongside warehouse transformations.

Use dbt when consistency belongs to model contracts and transformation CI.

### 8.3 Great Expectations

Great Expectations can provide declarative expectations and validation reporting.

Use it when teams need reusable validation suites and standardized result reporting.

It does not determine the business invariant for you.

## 9. Production Runbook

### Alert

1. Identify rule ID and dataset.
2. Check severity.
3. Identify affected run and data window.
4. Determine whether the failure is local or widespread.

### Diagnose

5. Read the exact invariant.
6. Verify rule version.
7. Inspect affected population.
8. Check completeness and uniqueness for the same scope.
9. Check join grain and fan-out.
10. Check time, currency, unit, and cutoff semantics.
11. Compare authoritative source evidence.

### Recover

12. Classify the failure as data, transformation, timing, rule, or source-contract issue.
13. Correct the smallest safe scope.
14. Replay idempotently where required.
15. Re-run the invariant.
16. Reconcile downstream results.

### Close

17. Record root cause.
18. Preserve evidence.
19. Record rule version and disposition.
20. Confirm failure metrics return to expected levels.
21. Update the rule only when the business contract actually changed.

### Incident decision table

| Observation | Likely classification | First action |
|---|---|---|
| Field relationship violated | Transformation or source defect | Inspect source lineage |
| Aggregate differs from detail | Missing, duplicate, fan-out, or calculation issue | Reconcile at target grain |
| Invalid state transition | Event ordering or state defect | Inspect event history |
| Parent exists but scope differs | Tenant/business-scope defect | Verify relationship contract |
| Cross-source mismatch | Timing, currency, fee, or source discrepancy | Align semantics before correction |
| Failure appears only after deployment | Transformation regression | Compare release and rule version |
| Many unrelated rules fail together | Shared upstream issue | Investigate common dependency |
| Rule fails for newly valid business state | Rule drift | Review and version the contract |

## 10. Common Mistakes

### Mistake 1 — Checking fields independently

Valid fields can form an invalid combination.

### Mistake 2 — Treating referential integrity as semantic consistency

A valid parent relationship can still have the wrong tenant, currency, state, or effective period.

### Mistake 3 — Aggregating after a one-to-many join

Join fan-out can create false inconsistencies.

### Mistake 4 — Ignoring cutoff times

Two snapshots can disagree simply because they represent different moments.

### Mistake 5 — Mixing units

A numerically plausible value can still be semantically wrong.

### Mistake 6 — Using floating-point arithmetic for money

Binary floating-point can introduce comparison errors.

### Mistake 7 — Treating every mismatch as corruption

Timing, fees, FX, rounding, and source semantics can explain legitimate differences.

### Mistake 8 — Hard-coding state transitions

Business workflows evolve. Govern the transition contract.

### Mistake 9 — Ignoring NULL semantics

Unknown and not-applicable values need explicit treatment.

### Mistake 10 — Correcting data inside the validator

Detection and disposition should be separate.

### Mistake 11 — Ignoring rule versioning

Historical failures must remain explainable under the rule that generated them.

### Mistake 12 — Using one global tolerance

Different invariants require different precision and business tolerances.

## 11. Definition of Done

You are done with T40 when you can:
- define consistency independently from completeness and validity;
- write a business invariant before implementing a check;
- classify invariants by type;
- implement field-to-field checks;
- implement conditional invariants;
- reconcile parent totals with detail;
- detect temporal contradictions;
- validate state transitions;
- validate tenant and business scope;
- validate units and currencies;
- compare independent sources safely;
- protect aggregate checks from join fan-out;
- handle NULL semantics explicitly;
- apply documented tolerances;
- use exact decimal arithmetic where required;
- version consistency rules;
- persist diagnostic evidence;
- implement checks in SQL;
- implement pure invariant checks in Python;
- write boundary and false-positive tests;
- observe failure rates and affected populations;
- intentionally create consistency failures;
- diagnose whether the defect is data, timing, transformation, or rule-related;
- recover the smallest safe scope;
- replay and revalidate;
- operate the mechanism with the runbook.

## 12. What You Learned

Consistency is about relationships, not isolated values.

The core mental model is:

    DEFINE INVARIANT
          ↓
    IDENTIFY GRAIN
          ↓
    OBSERVE RELATED FACTS
          ↓
    EVALUATE RELATIONSHIP
          ↓
    CLASSIFY VIOLATION
          ↓
    PRESERVE EVIDENCE
          ↓
    RECOVER OR UPDATE CONTRACT
          ↓
    RECHECK

You learned to:
- distinguish consistency from completeness;
- express business rules as explicit invariants;
- validate relationships between fields and records;
- reconcile aggregates with detail;
- validate temporal and state-machine semantics;
- protect checks against join fan-out;
- handle currency, units, precision, and NULLs;
- compare cross-source facts without assuming identical semantics;
- separate detection from correction;
- version rules and preserve evidence;
- recover safely and revalidate downstream effects.

The production principle is:

> **A dataset can be complete and individually valid while still being internally contradictory. Consistency must be tested as explicit relationships between facts.**

### Next Recipe

**T41 — Statistical Anomaly Detection**

T41 will move from deterministic invariants to statistical signals, where the question becomes: **does this data behave unexpectedly even when it passes explicit rules?**