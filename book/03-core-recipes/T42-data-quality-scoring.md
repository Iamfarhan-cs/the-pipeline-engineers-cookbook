# T42 — Data Quality Scoring

> **Goal:** Combine multiple data-quality signals into an explicit, auditable score without hiding the individual checks underneath one number.

A production data-quality system often produces many independent signals:

- completeness;
- validity;
- uniqueness;
- referential integrity;
- consistency;
- statistical anomalies;
- freshness;
- volume;
- schema compliance.

A score can summarize these signals for dashboards, prioritization, and release gates.

But a score is a summary, not the evidence.

The production principle is:

> **A data-quality score must never replace the underlying checks that explain why the score changed.**

## 1. Problem Recognition

A pipeline may produce dozens of quality measurements per dataset.

Without a common scoring model, stakeholders may see:

    completeness = 99.2%
    uniqueness = 97.0%
    validity = 100%
    consistency = 91.0%
    anomaly status = WARNING

and still have no single operational summary.

A quality score can provide that summary.

### 1.1 What a score should answer

A useful score should help answer:

- How healthy is this dataset overall?
- Which dimensions are causing the degradation?
- Is the score comparable across runs?
- Did quality improve or decline?
- Is the dataset safe for a particular consumer?
- Which checks require investigation?
- What changed from the previous run?

### 1.2 What a score should not answer

A score should not hide:

- which rule failed;
- which records failed;
- which partition failed;
- how much data was affected;
- whether the failure is critical;
- whether the score is based on missing measurements.

For example:

    quality_score = 82

is operationally weak without:

    completeness = 100
    uniqueness = 99.8
    consistency = 71
    critical_failures = 2

### 1.3 Score versus validation

Validation asks:

    Does this rule pass?

Scoring asks:

    How should multiple measured outcomes be summarized?

Scoring comes after measurement.

### 1.4 Score versus anomaly detection

T41 detects unusual behavior.

T42 summarizes quality signals.

An anomaly may reduce a score, but an anomaly should remain separately visible.

### 1.5 Define the scored object

Before scoring, define the entity being scored:

- dataset;
- table;
- partition;
- batch;
- source;
- pipeline stage;
- tenant;
- metric group.

Do not mix incompatible grains.

### 1.6 Define the consumer

A single score may serve:

- engineering dashboards;
- data-product owners;
- downstream pipelines;
- release gates;
- executive reporting.

These consumers may need different thresholds.

A score should therefore be accompanied by its policy version and scope.

### 1.7 Define score semantics

Choose what a score means.

Example:

    100 = all required quality dimensions meet policy
      0 = no scored quality requirement is satisfied

The exact semantics must be documented.

## 2. Concept and Reasoning

### 2.1 Score components, not raw observations

A score should usually aggregate normalized quality components.

Example:

    completeness_score = 0.99
    uniqueness_score = 0.98
    validity_score = 1.00
    consistency_score = 0.92

Then combine them according to a documented policy.

Do not directly average unrelated raw counts.

### 2.2 Normalize first

Different quality dimensions use different units:

    missing_rows = 240
    null_rate = 0.04
    duplicate_rate = 0.01
    invalid_rate = 0.002

Convert them into a common scale before aggregation.

A simple convention is:

    1.0 = fully acceptable
    0.0 = completely unacceptable

### 2.3 Binary component

For a hard rule:

    passed = 1
    failed = 0

This is appropriate when a requirement is strictly mandatory.

Example:

    primary_key_exists

### 2.4 Proportional component

For a rate-based metric:

    quality = 1 - defect_rate

Example:

    invalid_rate = 0.03
    quality = 0.97

Do not use this blindly when 3% failure should trigger a hard business block.

### 2.5 Threshold-based component

Some dimensions have a tolerance.

Example policy:

    defect_rate <= 1%     → score 1.0
    defect_rate = 2%       → score 0.75
    defect_rate >= 5%      → score 0.0

The function should be documented rather than invented ad hoc.

### 2.6 Weighted score

A common model is:

    overall_score = sum(weight_i * component_score_i)

with:

    sum(weights) = 1

Example:

    completeness  = 0.25
    validity      = 0.25
    consistency   = 0.20
    uniqueness    = 0.15
    referential   = 0.15

Weights are policy.

They are not universal truths.

### 2.7 Equal weighting

Equal weights are simple:

    overall = (c1 + c2 + c3 + c4 + c5) / 5

This is useful when all dimensions have similar business importance.

Do not use equal weighting merely because it is convenient.

### 2.8 Critical-rule override

Some failures should not be diluted by averaging.

Example:

    overall_score = 96
    but primary key is missing

A policy can therefore define:

    if critical_rule_failed:
        status = BLOCKED

The score and release decision remain separate outputs.

### 2.9 Hard gates versus soft score

A production system often needs both:

    quality_score = 96.2
    quality_status = BLOCKED

because a critical control failed.

This is better than forcing every operational decision into one numeric value.

### 2.10 Missing component measurements

Suppose five dimensions are expected but only three produced results.

Do not silently treat missing dimensions as zero or one.

Possible policies:

- score only evaluated dimensions and report coverage;
- block scoring until required dimensions are available;
- assign explicit UNKNOWN state.

A score without measurement coverage can be misleading.

### 2.11 Score coverage

Track:

    measured_components / expected_components

Example:

    4 / 5 = 80% coverage

A high score with low coverage should not look healthy.

### 2.12 Unknown is not pass

If a quality check did not execute:

    UNKNOWN

is not equivalent to:

    PASS

This distinction is essential for trustworthy dashboards and gates.

### 2.13 Weighted versus unweighted coverage

When weights exist, also calculate measured weight:

    measured_weight = sum(weight_i for measured components)

Example:

    expected weight = 1.00
    measured weight = 0.75

The score should expose that only 75% of policy weight was evaluated.

### 2.14 Penalty model

Another approach starts at 100 and subtracts penalties:

    score = 100 - completeness_penalty - validity_penalty - ...

Penalty systems can be useful but become difficult to reason about when penalties overlap.

Prefer a normalized component model when possible.

### 2.15 Multiplicative scores

A multiplicative model can be:

    score = c1 * c2 * c3

This heavily penalizes multiple moderate defects.

It is mathematically different from a weighted average and should not be substituted without documenting the semantics.

### 2.16 Worst-component score

For critical datasets, the score can be:

    score = min(component_scores)

This is conservative but can hide the relative severity of non-worst components.

### 2.17 Score granularity

A dataset-level score can hide partition-level failures.

For large datasets, calculate at appropriate levels:

    row / partition / batch / dataset

Then roll up using explicit rules.

### 2.18 Roll-up weighting

If partition A contains 10 million rows and partition B contains 100 rows, equal partition weighting may be misleading.

Possible roll-up bases:

- row count;
- business value;
- transaction count;
- partition weight;
- explicit business policy.

### 2.19 Business-critical weighting

Weights should reflect documented business importance.

For example, a payment dataset may treat:

- monetary reconciliation;
- required identifiers;
- referential integrity

as more critical than low-impact descriptive attributes.

Document why each weight exists.

### 2.20 Score comparability

A score is comparable only when:

- component definitions are stable;
- weights are stable;
- thresholds are stable;
- scope is comparable;
- rule versions are known.

If policy changes, version the score.

### 2.21 Score versioning

Store:

    score_policy_version = 'v1'

When weights or normalization rules change:

    v1 → v2

Do not silently compare historical scores produced under different policies.

### 2.22 Explainability

Every score should be decomposable.

Example:

    overall = 91.5

    completeness = 100
    validity = 98
    uniqueness = 94
    consistency = 78
    referential = 100

The operator should immediately see what caused the degradation.

### 2.23 Avoid double counting

Two checks may measure the same defect.

Example:

- null-rate check;
- completeness check;
- required-field validation.

If all three penalize the same missing population independently, the overall score can exaggerate the problem.

Group related checks or document intentional overlap.

### 2.24 Evidence hierarchy

A useful model is:

    raw observations
          ↓
    quality rule results
          ↓
    normalized components
          ↓
    weighted score
          ↓
    status / decision

Do not skip the intermediate evidence.

## 3. Implementation

### 3.1 Quality rule result table

Store individual rule outcomes.

    CREATE TABLE quality_rule_result (
        result_id bigserial PRIMARY KEY,
        dataset_name text NOT NULL,
        batch_id text NOT NULL,
        dimension text NOT NULL,
        rule_name text NOT NULL,
        rule_version text NOT NULL,
        status text NOT NULL,
        observed_value numeric,
        expected_value numeric,
        defect_rate numeric,
        measured_at timestamptz NOT NULL DEFAULT now(),
        metadata jsonb NOT NULL DEFAULT '{}'::jsonb
    );

This table is the evidence layer.

### 3.2 Scoring policy table

Store scoring configuration separately.

    CREATE TABLE quality_score_policy (
        policy_version text NOT NULL,
        dimension text NOT NULL,
        weight numeric NOT NULL,
        min_score numeric NOT NULL,
        critical boolean NOT NULL DEFAULT false,
        enabled boolean NOT NULL DEFAULT true,
        PRIMARY KEY (policy_version, dimension)
    );

Policy should be data, not hidden in application code.

### 3.3 Score result table

    CREATE TABLE quality_score_result (
        score_id bigserial PRIMARY KEY,
        dataset_name text NOT NULL,
        batch_id text NOT NULL,
        policy_version text NOT NULL,
        score numeric,
        coverage numeric NOT NULL,
        status text NOT NULL,
        critical_failure_count integer NOT NULL,
        calculated_at timestamptz NOT NULL DEFAULT now()
    );

### 3.4 Normalize a defect rate

For a simple proportional metric:

    component_score = GREATEST(
        0,
        LEAST(
            1,
            1 - COALESCE(defect_rate, 1)
        )
    )

This maps:

    0% defect → 1.0
    10% defect → 0.9
    100% defect → 0.0

Use threshold-aware scoring when business tolerance is not linear.

### 3.5 Threshold-based normalization

A simple piecewise policy can be:

    defect_rate <= acceptable_rate → 1.0

    defect_rate >= unacceptable_rate → 0.0

    otherwise:
        1 - (
            defect_rate - acceptable_rate
        ) / (
            unacceptable_rate - acceptable_rate
        )

The acceptable and unacceptable thresholds must be explicitly configured.

### 3.6 Binary rule scoring

For strict rules:

    CASE
        WHEN status = 'PASS' THEN 1.0
        WHEN status = 'FAIL' THEN 0.0
        ELSE NULL
    END AS component_score

UNKNOWN should remain unknown.

### 3.7 Calculate weighted score

    SELECT
        dataset_name,
        batch_id,
        SUM(component_score * weight)
            / NULLIF(SUM(weight), 0) AS quality_score
    FROM normalized_quality_components
    WHERE measured = true
    GROUP BY dataset_name, batch_id;

If incomplete coverage is allowed, return measured weight separately.

### 3.8 Calculate coverage

    SELECT
        dataset_name,
        batch_id,
        SUM(CASE WHEN measured THEN weight ELSE 0 END)
            / NULLIF(SUM(weight), 0) AS weighted_coverage
    FROM scoring_inputs
    GROUP BY dataset_name, batch_id;

Do not call a 60% measured score fully evaluated.

### 3.9 Critical failures

    SELECT
        dataset_name,
        batch_id,
        count(*) FILTER (
            WHERE critical = true
              AND status = 'FAIL'
        ) AS critical_failures
    FROM scoring_inputs
    GROUP BY dataset_name, batch_id;

Then apply the decision policy separately:

    if critical_failures > 0:
        status = 'BLOCKED'

### 3.10 Preserve the component breakdown

Create a detailed result view:

    SELECT
        dataset_name,
        batch_id,
        dimension,
        component_score,
        weight,
        component_score * weight AS weighted_contribution,
        status
    FROM scoring_inputs;

This makes the score explainable.

### 3.11 Example end-to-end SQL

    WITH components AS (
        SELECT
            dataset_name,
            batch_id,
            dimension,
            weight,
            critical,
            CASE
                WHEN status = 'PASS' THEN 1.0
                WHEN status = 'FAIL' THEN 0.0
                ELSE NULL
            END AS component_score,
            status
        FROM scoring_inputs
    ),
    aggregate AS (
        SELECT
            dataset_name,
            batch_id,
            SUM(component_score * weight)
                / NULLIF(
                    SUM(weight) FILTER (
                        WHERE component_score IS NOT NULL
                    ),
                    0
                ) AS score,
            SUM(weight) FILTER (
                WHERE component_score IS NOT NULL
            ) AS measured_weight,
            SUM(weight) AS expected_weight,
            count(*) FILTER (
                WHERE critical = true
                  AND status = 'FAIL'
            ) AS critical_failures
        FROM components
        GROUP BY dataset_name, batch_id
    )
    SELECT
        *,
        measured_weight
            / NULLIF(expected_weight, 0) AS coverage,
        CASE
            WHEN critical_failures > 0 THEN 'BLOCKED'
            WHEN measured_weight < expected_weight THEN 'INCOMPLETE'
            ELSE 'EVALUATED'
        END AS status
    FROM aggregate;

The exact policy should be versioned and tested.

### 3.12 Python scoring function

    def weighted_score(components):
        measured = [
            item for item in components
            if item["score"] is not None
        ]

        if not measured:
            return None

        total_weight = sum(item["weight"] for item in measured)

        if total_weight == 0:
            return None

        return sum(
            item["score"] * item["weight"]
            for item in measured
        ) / total_weight

This function deliberately does not decide whether missing components are acceptable.

### 3.13 Python coverage

    def weighted_coverage(components):
        expected = sum(item["weight"] for item in components)

        measured = sum(
            item["weight"]
            for item in components
            if item["score"] is not None
        )

        if expected == 0:
            return 0.0

        return measured / expected

### 3.14 Python critical gate

    def critical_failures(components):
        return [
            item for item in components
            if item["critical"] and item["status"] == "FAIL"
        ]

Keep score calculation and gate decisions separate.

### 3.15 Score bands

Example policy:

    score >= 0.95 → HEALTHY
    score >= 0.85 → DEGRADED
    score <  0.85 → POOR

These values are examples only.

A score band should never replace the component breakdown.

### 3.16 Score plus status

A final result can be:

    score = 0.92
    coverage = 1.00
    status = DEGRADED

or:

    score = 0.98
    coverage = 1.00
    status = BLOCKED
    critical_failures = 1

This preserves both summary and operational decision.

### 3.17 Partition roll-up

For partition-level results:

    partition_score
    partition_weight

Then:

    dataset_score =
        SUM(partition_score * partition_weight)
        / SUM(partition_weight)

Choose the partition weight explicitly.

### 3.18 Business-weighted roll-up

A financial dataset may weight partitions by transaction value rather than row count.

Example:

    weight = absolute_transaction_value

This must be normalized carefully and protected from a single extreme partition dominating the entire score.

### 3.19 Rule dependency handling

Some checks depend on others.

Example:

    type validation
        ↓
    range validation
        ↓
    consistency validation

If type parsing fails, downstream numeric checks may be UNKNOWN rather than FAIL.

Avoid scoring cascading failures as independent defects unless policy explicitly requires it.

### 3.20 Duplicate evidence handling

If five rules all identify the same underlying defect, decide whether they represent:

- five independent quality problems;
- one root cause with five symptoms.

A scoring policy should document aggregation behavior.

### 3.21 Score history

Persist scores over time.

Useful dimensions:

    dataset
    batch
    partition
    policy_version
    score
    coverage
    status
    calculated_at

This enables trend analysis.

### 3.22 Score deltas

Calculate:

    current_score - previous_score

Also expose:

- largest component decline;
- newly failed critical rules;
- coverage changes;
- policy-version changes.

A score without its delta loses operational context.

## 4. Testing

Scoring requires both mathematical tests and policy tests.

### 4.1 Perfect score

All components:

    score = 1.0

Expected:

    overall = 1.0
    coverage = 1.0

### 4.2 Zero score

All measured components:

    score = 0.0

Expected:

    overall = 0.0

### 4.3 Weighted score

Use:

    component A = 1.0, weight = 0.75
    component B = 0.0, weight = 0.25

Expected:

    overall = 0.75

### 4.4 Equal weights

Verify that equal weights produce the arithmetic mean.

### 4.5 Missing component

Provide five expected components but only four measurements.

Expected behavior must match policy:

    coverage < 1.0

Never silently convert missing to PASS.

### 4.6 Unknown component

Set one result to UNKNOWN.

Verify it does not become an implicit zero or one.

### 4.7 Critical failure

Create:

    score = 0.99
    critical_failures = 1

Expected:

    score remains 0.99
    status = BLOCKED

This proves score and gate semantics are separate.

### 4.8 Threshold normalization

Test:

- exactly at acceptable threshold;
- just above acceptable threshold;
- midpoint;
- exactly at unacceptable threshold;
- beyond unacceptable threshold.

Boundary behavior must be deterministic.

### 4.9 Zero total weight

Provide no enabled policy weight.

Expected:

    explicit configuration error or undefined score

Never divide by zero.

### 4.10 Double-counting test

Create two rules that measure the same defect.

Verify the documented aggregation behavior.

### 4.11 Dependency test

Make an upstream type check fail.

Verify dependent checks become UNKNOWN where appropriate instead of producing misleading failures.

### 4.12 Policy-version test

Calculate a score under v1 and v2.

Verify both records retain their policy versions and are not silently treated as directly comparable.

### 4.13 Partition roll-up

Use partitions with known weights and verify the weighted aggregate manually.

### 4.14 Score explainability

For every overall score, verify that component contributions sum to the documented score.

### 4.15 Idempotence

Recalculate the same batch under the same policy.

Expected:

    same component results
    same score
    same status

Duplicate score records should be prevented or explicitly versioned.

### 4.16 Regression test

Maintain representative quality scenarios:

- healthy batch;
- minor degradation;
- severe degradation;
- missing measurement;
- critical failure;
- dependency failure;
- policy change;
- partial execution.

## 5. Observability

A score should be observable at both summary and component levels.

### 5.1 Core metrics

| Metric | Meaning |
|---|---|
| data_quality_score | Overall normalized score |
| data_quality_coverage | Fraction of expected quality policy measured |
| data_quality_critical_failures | Number of critical failed rules |
| data_quality_component_score | Individual dimension score |
| data_quality_score_delta | Change from previous comparable result |
| data_quality_rules_evaluated_total | Rules evaluated |
| data_quality_rules_failed_total | Rules that failed |
| data_quality_unknown_total | Rules without usable measurements |
| data_quality_policy_version | Active scoring policy |

### 5.2 Score distribution

Track score over time:

    dataset → score → time

Look for:

- gradual degradation;
- sudden drops;
- persistent low scores;
- unexplained score recovery.

### 5.3 Component contribution

For each score, expose:

    component_score
    weight
    weighted_contribution

This identifies what actually moved the result.

### 5.4 Coverage monitoring

A dangerous state is:

    score = 99%
    coverage = 40%

The score looks excellent only because most policy checks did not run.

Alert on insufficient coverage independently.

### 5.5 Critical failures

Monitor critical failures separately from score.

A critical failure must remain visible even when other healthy dimensions dilute the overall score.

### 5.6 Policy changes

Dashboard policy-version changes.

A score trend that crosses:

    v1 → v2

may not be directly comparable.

### 5.7 Alerting

Useful alerts include:

- score below policy threshold;
- coverage below minimum;
- critical failure count > 0;
- score drops sharply from prior comparable batch;
- one component degrades repeatedly;
- score is high but coverage is low.

### 5.8 Explainable dashboard

A practical dashboard can show:

    Overall Score
    Coverage
    Status
    Critical Failures

then:

    Completeness   100%
    Validity        98%
    Uniqueness      94%
    Consistency     78%
    Referential    100%

and finally:

    Rule-level evidence

The operator should not need to query raw logs to understand the score.

### 5.9 Score freshness

A stale score can be mistaken for a current score.

Track:

    score_calculated_at
    source_batch_time
    score_age

Do not present an old score as current health.

## 6. Intentional Failure

Break the scoring system deliberately.

### Failure 1 — One component collapses

Set:

    completeness = 0.0

Expected:

    overall score decreases according to its weight.

### Failure 2 — Critical rule fails

Set a critical rule to FAIL while keeping other components healthy.

Expected:

    high score may remain possible
    status = BLOCKED

### Failure 3 — Missing measurement

Remove one expected component.

Expected:

    coverage decreases
    status follows incomplete-coverage policy

### Failure 4 — Unknown result

Make a quality rule UNKNOWN.

Expected:

    it is not silently converted to PASS or FAIL.

### Failure 5 — Weight misconfiguration

Set all weights to zero.

Expected:

    scoring fails safely with an explicit configuration error.

### Failure 6 — Double-counted defect

Make one root defect cause multiple related rule failures.

Expected:

    documented aggregation policy remains deterministic.

### Failure 7 — Dependency failure

Break type validation before range validation.

Expected:

    dependent rule becomes UNKNOWN when its input is unavailable.

### Failure 8 — Policy version change

Change weights from v1 to v2.

Expected:

    score records retain their policy versions
    historical results are not silently rewritten.

### Failure 9 — High score, low coverage

Allow only one easy check to run.

Expected:

    coverage exposes that the score is incomplete.

## 7. Recovery

Score recovery means restoring the underlying quality evidence, not manually editing the score.

### 7.1 Failed source data

1. Identify failed quality dimensions.
2. Trace each failure to source, transformation, or reference data.
3. Recover the underlying defect.
4. Replay the affected scope.
5. Re-run quality rules.
6. Recalculate the score.
7. Compare with the previous score.

### 7.2 Missing quality measurements

If checks did not execute:

1. identify missing rules;
2. determine why execution stopped;
3. restore the quality-check stage;
4. rerun the missing measurements;
5. recalculate coverage;
6. recalculate the score.

Do not mark missing checks as PASS.

### 7.3 Scoring-policy defect

If the scoring implementation is wrong:

1. identify affected policy version;
2. stop using the defective policy for new decisions;
3. create a corrected policy version;
4. test the new policy;
5. recalculate affected batches;
6. preserve old results for audit;
7. document the policy change.

### 7.4 Weight misconfiguration

If weights are wrong:

1. validate intended business policy;
2. correct configuration;
3. version the policy;
4. backtest representative historical batches;
5. deploy;
6. recalculate only where required.

### 7.5 False score degradation

If a score drops because one detector is broken:

1. inspect component evidence;
2. determine whether the underlying data is actually defective;
3. repair the detector;
4. rerun the affected component;
5. recalculate;
6. preserve the original score and correction reason.

### 7.6 Score recovery is not data repair

Never change source data just to make a score higher.

The sequence is:

    identify defect
        ↓
    correct real cause
        ↓
    replay
        ↓
    remeasure
        ↓
    rescore

### 7.7 Safe replay

Preserve:

- dataset;
- batch ID;
- policy version;
- rule versions;
- original score;
- corrected score;
- replay reason;
- correction timestamp.

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

PostgreSQL is useful for:

- storing rule results;
- joining scoring inputs to policies;
- normalizing components;
- calculating weighted scores;
- preserving score history;
- generating explainable breakdowns.

The important skill is designing an auditable scoring model, not writing one aggregate query.

### 8.2 dbt

dbt can materialize quality metrics and score inputs close to warehouse models.

It is useful for:

- repeatable quality models;
- rule-result tables;
- score component generation;
- historical score snapshots.

Keep scoring policy explicit and versioned.

### 8.3 Great Expectations

Great Expectations can produce structured expectation results that become inputs to a broader quality-scoring system.

Use it for reusable expectation suites and validation evidence.

Do not delegate the scoring semantics to a framework default.

## 9. Production Runbook

### Alert

1. Identify dataset, batch, partition, and policy version.
2. Record overall score and coverage.
3. Check critical failure count.
4. Inspect component scores and weighted contributions.

### Diagnose

5. Identify the largest score contributors.
6. Inspect failed rules.
7. Distinguish root causes from symptoms.
8. Check rule dependencies.
9. Check whether any measurements are UNKNOWN.
10. Check policy-version changes.
11. Compare against previous comparable batches.

### Recover

12. Fix source or transformation defects.
13. Restore missing quality measurements.
14. Replay the smallest affected scope.
15. Re-run quality rules.
16. Recalculate the score.
17. Verify critical gates separately.

### Close

18. Record root cause.
19. Preserve original and corrected results.
20. Confirm coverage is complete.
21. Confirm policy and rule versions.
22. Record the final score and status.

### Incident decision table

| Observation | Classification | Action |
|---|---|---|
| Low score + low completeness | Population/data delivery issue | Investigate source and replay missing scope |
| High score + low coverage | Incomplete evaluation | Restore missing checks |
| High score + critical failure | Gate violation | Block consumer despite score |
| Score drop + one component decline | Local quality defect | Investigate that dimension |
| Many components decline together | Shared upstream issue | Find common root cause |
| Score changes after policy update | Policy-version effect | Compare within policy versions |
| Unknown components | Measurement failure | Restore quality-check execution |
| Duplicate rule failures from one defect | Correlated symptoms | Apply documented aggregation policy |
| Score recovers without data correction | Detector/scoring issue | Audit scoring implementation |

## 10. Common Mistakes

### Mistake 1 — Treating the score as truth

A score is a summary of evidence, not the evidence itself.

### Mistake 2 — Converting UNKNOWN to zero

This turns measurement failure into data failure.

### Mistake 3 — Converting UNKNOWN to one

This makes missing checks look healthy.

### Mistake 4 — Hiding critical failures inside an average

A mandatory control may need an independent gate.

### Mistake 5 — Double counting the same defect

Related checks can exaggerate one root cause.

### Mistake 6 — Changing weights without versioning

Historical scores become difficult or impossible to compare.

### Mistake 7 — Using equal weights by default

Importance differs by dataset and business process.

### Mistake 8 — Scoring incompatible grains together

A row-level metric and a dataset-level metric require explicit roll-up semantics.

### Mistake 9 — Ignoring coverage

A high score with incomplete measurement can be dangerously misleading.

### Mistake 10 — Using raw counts directly

A count of 100 defects means different things in 1,000 rows versus 100 million rows.

Normalize appropriately.

### Mistake 11 — Making score thresholds universal

Different datasets have different risk profiles.

### Mistake 12 — Rewriting historical scores after policy changes

Preserve the original policy version and result.

### Mistake 13 — Manually editing scores during incidents

Repair evidence and recompute.

### Mistake 14 — Replacing rule-level diagnostics with one number

Operators need to know why quality changed.

## 11. Definition of Done

You are done with T42 when you can:

- explain why scoring comes after quality measurement;
- distinguish score, status, and evidence;
- define a scored object and grain;
- normalize different quality dimensions;
- calculate binary and proportional components;
- implement threshold-based normalization;
- calculate weighted scores;
- calculate score coverage;
- handle UNKNOWN measurements safely;
- implement critical-rule overrides;
- separate score from operational gates;
- prevent double counting;
- handle rule dependencies;
- roll up partition scores correctly;
- version scoring policies;
- preserve score history;
- calculate score deltas;
- expose component contributions;
- implement scoring in SQL;
- implement scoring helpers in Python;
- test mathematical boundaries;
- test missing measurements;
- test critical failures;
- test policy changes;
- intentionally break the scoring system;
- recover scoring and underlying data defects safely;
- operate the score with a production runbook.

## 12. What You Learned

Data-quality scoring is an aggregation layer over evidence.

The production mental model is:

    RAW DATA
       ↓
    QUALITY RULES
       ↓
    RULE RESULTS
       ↓
    NORMALIZED COMPONENTS
       ↓
    WEIGHTS + POLICY
       ↓
    SCORE + COVERAGE
       ↓
    STATUS / GATE
       ↓
    EXPLAINABLE EVIDENCE

You learned to:

- normalize heterogeneous quality signals;
- calculate weighted scores;
- preserve measurement coverage;
- distinguish UNKNOWN from PASS and FAIL;
- apply critical-rule overrides;
- prevent double counting;
- version scoring policies;
- preserve component-level explanations;
- roll up partition-level results safely;
- test score mathematics and operational policy;
- recover without manually changing scores;
- use scores for summary while keeping rule-level evidence authoritative.

The production principle is:

> **A quality score should compress evidence for humans and systems, never destroy the evidence underneath it.**

### Next Recipe

**T43 — Invalid Record Handling**

T43 will define how invalid records are classified, isolated, quarantined, audited, replayed, and safely reconciled without silently dropping data.
