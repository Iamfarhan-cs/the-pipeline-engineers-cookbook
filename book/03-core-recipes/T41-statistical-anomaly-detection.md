# T41 — Statistical Anomaly Detection

> **Goal:** Detect data behavior that is statistically unusual even when deterministic validation rules pass.

Deterministic checks answer questions such as whether a value is valid, a required field exists, or an aggregate reconciles. Statistical anomaly detection asks:

> **Is the observed behavior materially different from the behavior this dataset normally exhibits?**

This recipe focuses on production data-quality anomaly detection, not generic machine-learning anomaly detection.

## 1. Problem Recognition

A pipeline can pass completeness, uniqueness, type, range, domain, referential-integrity, and consistency checks while still being wrong.

Typical examples:

- hourly volume falls from 20,000 rows to 2,000;
- a NULL rate rises from 0.2% to 18%;
- a normally diverse identifier column becomes nearly constant;
- one payment status suddenly represents 80% of traffic;
- the mean payment amount doubles while every individual amount remains within range;
- a source version changes the value distribution without violating any hard rule;
- the row count remains correct while the population composition changes;
- a pipeline produces a long flatline because an upstream process is stuck.

### 1.1 Common anomaly dimensions

| Dimension | Example signal |
|---|---|
| Volume | 80,000 rows instead of a normal 20,000 |
| Mean | Average amount doubles |
| Median | Typical value shifts |
| Quantiles | P95 or P99 changes sharply |
| Variance | Distribution becomes much wider |
| NULL rate | NULL rate rises from 0.2% to 18% |
| Cardinality | Distinct customer count collapses |
| Category share | One status dominates unexpectedly |
| Frequency | One event type becomes unusually common |
| Time gaps | Expected observations stop arriving |
| Magnitude | Aggregate moves far beyond baseline |
| Correlation | Related metrics stop moving together |

### 1.2 Anomaly does not mean invalid

An anomaly may be:

- a real business event;
- a source change;
- a seasonal event;
- a deployment effect;
- a data-quality defect;
- a delayed batch;
- a legitimate new customer population.

Never automatically delete or reject anomalous data.

### 1.3 Statistical detection versus hard validation

A deterministic rule might say:

    amount >= 0

A statistical rule might say:

    amount is far above its historical baseline

The first establishes a contract. The second identifies unusual behavior.

### 1.4 Define the observation population

Before calculating a baseline, define:

- dataset;
- metric;
- grain;
- time window;
- population filter;
- tenant or business scope;
- timezone;
- baseline history;
- minimum sample size.

Averages calculated over changing populations are often misleading.

### 1.5 Define the action

An anomaly score without an operational response is not a production control.

Possible actions:

- observe only;
- create a warning;
- investigate;
- quarantine a batch;
- pause publication;
- trigger a source check;
- request manual approval.

Severity must be tied to business impact.

## 2. Concept and Reasoning

### 2.1 Baseline first

Anomaly detection requires an expectation.

Common baselines:

- rolling mean;
- rolling median;
- historical percentile;
- same hour of previous days;
- same weekday;
- exponentially weighted baseline;
- reference distribution.

The baseline should represent comparable observations.

### 2.2 Mean and standard deviation

For approximately stable and reasonably symmetric data:

    z = (x - mean) / standard_deviation

Large absolute z-scores indicate unusual observations.

Standard deviation is sensitive to extreme values, so do not blindly use z-scores for every dataset.

### 2.3 Robust statistics

Median and median absolute deviation are less sensitive to extreme values.

Define:

    MAD = median(|x - median(x)|)

A robust standardized score can then be constructed using the chosen scaling convention.

The important production principle is that the baseline itself should not be dominated by the anomalies it is supposed to detect.

### 2.4 Percentile baselines

Instead of assuming a distribution shape, compare an observation with historical quantiles.

Example:

    historical_p99 = 9500
    current_value = 17000

The current observation is above the historical P99.

Percentiles are often easier to explain operationally than a complex model.

### 2.5 Volume anomalies

Volume is one of the highest-value pipeline signals.

Example:

    normal hourly volume = 18000 to 24000
    current hour = 2100

Possible causes include:

- source outage;
- missing partition;
- delayed delivery;
- filter regression;
- upstream business change.

Volume anomalies should be interpreted alongside completeness.

### 2.6 Distribution anomalies

Two batches can contain the same number of rows while having radically different distributions.

Compare:

- mean;
- median;
- standard deviation;
- quantiles;
- NULL rate;
- cardinality;
- category frequencies.

### 2.7 NULL-rate anomalies

A column may permit NULL while its population quality changes dramatically.

Example:

    historical_null_rate = 0.3%
    current_null_rate = 12.0%

This is statistically abnormal even if NULL is technically allowed.

### 2.8 Cardinality anomalies

Track distinct-value counts and useful ratios.

Examples:

- unique customer count collapses;
- status cardinality changes unexpectedly;
- merchant cardinality doubles;
- an identifier becomes constant.

Cardinality anomalies can expose upstream collapse or accidental filtering.

### 2.9 Category-share anomalies

Suppose:

    APPROVED = 92%
    DECLINED = 6%
    PENDING = 2%

A new batch:

    APPROVED = 20%
    DECLINED = 78%
    PENDING = 2%

may indicate a source or business change even though every status is valid.

### 2.10 Seasonality

Many datasets are seasonal:

- weekday versus weekend;
- business hours versus overnight;
- month-end accounting;
- holidays;
- payroll dates.

Comparing Monday 09:00 with Sunday 03:00 can create false anomalies.

Use comparable periods.

### 2.11 Trend-aware baselines

A steadily changing metric can repeatedly alert against a static baseline.

Example:

    10000 → 11000 → 12000 → 13000 → 14000

Rolling or trend-aware baselines can better represent an evolving process.

### 2.12 Minimum sample size

Small samples produce unstable estimates.

Do not treat one extreme observation as statistically equivalent to a large historical population.

Define a minimum baseline size and return an explicit NOT_ENOUGH_HISTORY state when it is not satisfied.

### 2.13 Baseline contamination

If an outage enters the baseline, future anomalies may become harder to detect.

Baseline history should therefore have a controlled eligibility policy.

Potential exclusions:

- known incidents;
- backfills;
- migrations;
- test runs;
- partial batches.

### 2.14 Statistical versus business significance

A tiny change can be statistically detectable in a huge dataset but operationally irrelevant.

A large business-impacting change can occur in a small population.

Combine statistical deviation with business effect size where appropriate.

### 2.15 Multiple testing

Monitoring hundreds of metrics creates alert noise.

Control it with:

- severity tiers;
- effect-size thresholds;
- grouped incidents;
- alert budgets;
- correlated-signal suppression.

### 2.16 Anomaly score is not probability

A z-score or distance metric does not automatically mean a given probability that the data is wrong.

A statistical score measures deviation under a model. It does not establish root cause.

### 2.17 Point versus contextual anomalies

A point anomaly is unusual by itself.

A contextual anomaly is unusual given context.

Example:

    100 events per minute

may be normal during business hours but unusual at 03:00.

Context must therefore be part of the baseline.

### 2.18 Collective anomalies

A sequence can be anomalous even when every individual observation is normal.

Example:

    10, 10, 10, 10, 10, 10

may be an unexpected six-hour plateau caused by a stuck process.

Monitor time-series behavior, not only individual values.

## 3. Implementation

### 3.1 Store statistical observations

Create a durable metric table.

    CREATE TABLE quality_metric_observation (
        observation_id bigserial PRIMARY KEY,
        dataset_name text NOT NULL,
        metric_name text NOT NULL,
        observation_time timestamptz NOT NULL,
        population_count bigint NOT NULL,
        metric_value numeric,
        null_rate numeric,
        distinct_count bigint,
        metadata jsonb NOT NULL DEFAULT '{}'::jsonb
    );

Keep source data separate from monitoring state.

### 3.2 Store anomaly results

    CREATE TABLE anomaly_result (
        anomaly_id bigserial PRIMARY KEY,
        dataset_name text NOT NULL,
        metric_name text NOT NULL,
        observation_time timestamptz NOT NULL,
        baseline_value numeric,
        observed_value numeric,
        deviation numeric,
        score numeric,
        severity text NOT NULL,
        status text NOT NULL,
        rule_version text NOT NULL,
        created_at timestamptz NOT NULL DEFAULT now()
    );

Persisting results makes anomaly decisions auditable.

### 3.3 Rolling baseline

For hourly row counts:

    SELECT
        observation_time,
        metric_value,
        avg(metric_value) OVER (
            ORDER BY observation_time
            ROWS BETWEEN 24 PRECEDING AND 1 PRECEDING
        ) AS baseline_mean
    FROM quality_metric_observation
    WHERE dataset_name = 'payments'
      AND metric_name = 'row_count';

The current observation is excluded from its own baseline.

### 3.4 Rolling standard deviation

    SELECT
        observation_time,
        metric_value,
        avg(metric_value) OVER w AS baseline_mean,
        stddev_samp(metric_value) OVER w AS baseline_stddev
    FROM quality_metric_observation
    WHERE dataset_name = 'payments'
      AND metric_name = 'row_count'
    WINDOW w AS (
        ORDER BY observation_time
        ROWS BETWEEN 24 PRECEDING AND 1 PRECEDING
    );

Calculate the score in a second query layer so the baseline remains explicit.

### 3.5 Handle zero variance

If:

    standard_deviation = 0

do not divide by zero.

A simple policy can be:

    observed = baseline → NORMAL
    observed != baseline → ANOMALY

The exact policy should be documented.

### 3.6 Percentile baseline

PostgreSQL can calculate historical quantiles.

    SELECT
        percentile_cont(0.50) WITHIN GROUP (ORDER BY metric_value) AS p50,
        percentile_cont(0.95) WITHIN GROUP (ORDER BY metric_value) AS p95,
        percentile_cont(0.99) WITHIN GROUP (ORDER BY metric_value) AS p99
    FROM quality_metric_observation
    WHERE dataset_name = 'payments'
      AND metric_name = 'amount'
      AND observation_time >= now() - interval '30 days';

Use a comparison population that matches the current context.

### 3.7 Same-context baseline

For hourly systems, compare the current 14:00 observation with previous comparable 14:00 observations rather than every hour.

Useful grouping dimensions can include:

    weekday
    hour_of_day
    tenant_scope
    dataset
    metric

### 3.8 NULL-rate monitoring

    SELECT
        count(*) AS total_rows,
        count(*) FILTER (WHERE merchant_id IS NULL) AS null_rows,
        count(*) FILTER (WHERE merchant_id IS NULL)::numeric
            / NULLIF(count(*), 0) AS null_rate
    FROM staging_payments
    WHERE event_date = DATE '2026-09-28';

Store the resulting rate as a quality metric.

### 3.9 Cardinality monitoring

    SELECT
        count(*) AS row_count,
        count(DISTINCT customer_id) AS distinct_customers,
        count(DISTINCT merchant_id) AS distinct_merchants
    FROM staging_payments
    WHERE event_date = DATE '2026-09-28';

Monitor both absolute cardinality and ratios such as:

    distinct_customers / row_count

### 3.10 Category distribution

    SELECT
        status,
        count(*) AS rows,
        count(*)::numeric / sum(count(*)) OVER () AS share
    FROM staging_payments
    WHERE event_date = DATE '2026-09-28'
    GROUP BY status;

Persist category shares for historical comparison.

### 3.11 Effect size

Do not use statistical deviation alone.

Example policy:

    alert if z_score > 4
    AND absolute_change_pct > 20%

The thresholds are examples. They must be selected and documented for the dataset.

### 3.12 Robust baseline in Python

    from statistics import median

    def median_absolute_deviation(values):
        center = median(values)
        deviations = [abs(value - center) for value in values]
        return median(deviations)

Keep baseline calculation separate from alert policy.

### 3.13 Python z-score

    from math import sqrt

    def z_score(value, values):
        mean = sum(values) / len(values)
        variance = sum((x - mean) ** 2 for x in values) / len(values)
        stddev = sqrt(variance)

        if stddev == 0:
            return 0.0 if value == mean else float('inf')

        return (value - mean) / stddev

Define the sample/population variance convention explicitly.

### 3.14 Minimum-history guard

    def enough_history(values, minimum=20):
        return len(values) >= minimum

Do not silently produce a score from insufficient history.

### 3.15 Baseline eligibility

Mark known incident observations explicitly:

    is_baseline_eligible = false

Exclude them according to the documented baseline policy.

Do not delete historical observations just to make future alerts quieter.

### 3.16 Severity mapping

A simple policy can be:

    score < threshold_1 → NORMAL
    score >= threshold_1 → WARNING
    score >= threshold_2 → CRITICAL

Add minimum sample and business-effect conditions where appropriate.

Thresholds are operational policy, not universal statistical truth.

### 3.17 Detector workflow

A production detector can follow:

    anomaly detected
        ↓
    enrich with context
        ↓
    compare deterministic checks
        ↓
    evaluate severity
        ↓
    warn / investigate / quarantine according to policy

Do not automatically quarantine every anomaly.

## 4. Testing

Statistical checks require controlled test data in addition to ordinary unit tests.

### 4.1 Stable baseline

Provide a stable historical series.

Expected:

    no anomaly

### 4.2 Single spike

Add one extreme observation.

Expected:

    anomaly detected

Verify that the spike does not immediately redefine its own baseline.

### 4.3 Step change

Shift all observations to a new stable level.

Define whether the detector should classify the first transition as anomalous and how the new level becomes normal.

### 4.4 Seasonal test

Create different normal values for business hours and overnight.

Ensure the detector does not compare unlike contexts.

### 4.5 Small-sample test

Provide fewer observations than the minimum history requirement.

Expected:

    NOT_ENOUGH_HISTORY

not a fabricated score.

### 4.6 Constant-baseline test

Test zero variance and ensure the result is deterministic.

### 4.7 Baseline-contamination test

Inject a known incident into historical data.

Verify that baseline eligibility can exclude it when policy requires.

### 4.8 NULL-rate anomaly

Create:

    historical_null_rate = 0.2%
    current_null_rate = 15%

Expected:

    anomaly detected

### 4.9 Cardinality collapse

Replace diverse identifiers with one repeated identifier.

Row count stays stable, but cardinality should become anomalous.

### 4.10 Category shift

Change a category's historical share materially while keeping every category value valid.

The detector should identify the distribution shift.

### 4.11 Duplicate compensation

Remove unique entities and duplicate existing entities while preserving row count.

Volume remains normal while uniqueness or cardinality changes.

### 4.12 False-positive tests

Test legitimate:

- holiday traffic;
- month-end spikes;
- product launches;
- planned migrations;
- known backfills;
- new tenant onboarding.

Represent known business events as context rather than silently disabling checks.

### 4.13 Determinism

Run the detector twice over identical observations.

Results should be identical.

### 4.14 Idempotent result handling

Reprocessing the same metric observation must not create duplicate active incidents.

Use stable identifiers or appropriate unique constraints.

## 5. Observability

Anomaly detection must itself be observable.

### 5.1 Detector metrics

| Metric | Meaning |
|---|---|
| anomaly_rules_evaluated_total | Number of evaluations |
| anomalies_detected_total | Detected anomalies |
| anomaly_warning_total | Warning-level anomalies |
| anomaly_critical_total | Critical anomalies |
| anomaly_suppressed_total | Suppressed or deduplicated alerts |
| anomaly_insufficient_history_total | Evaluations without enough baseline |
| anomaly_baseline_size | Number of baseline observations |
| anomaly_score | Current deviation score |
| anomaly_effect_size | Magnitude of business change |

### 5.2 Baseline health

Monitor:

- baseline sample count;
- baseline age;
- baseline eligibility rate;
- baseline variance;
- baseline freshness;
- seasonal-group coverage.

A broken baseline can produce a broken detector.

### 5.3 Alert quality

Track:

- alert rate;
- acknowledged alerts;
- confirmed incidents;
- false positives;
- suppressed alerts;
- repeated alerts for the same root cause.

Alert volume is itself an operational quality metric.

### 5.4 Dashboards

Useful panels include:

- current value versus baseline;
- historical percentile band;
- anomaly score;
- sample size;
- recent alerts;
- affected partitions;
- deterministic quality results;
- deployment timeline.

### 5.5 Correlate signals

An anomaly becomes more actionable when combined with deterministic evidence.

Example:

    row_count anomaly
    + missing partition
    + source lag
    = likely delivery issue

Do not collapse every signal into one opaque score.

### 5.6 Alert deduplication

A sustained anomaly can produce many identical alerts.

Group incidents by:

- dataset;
- metric;
- anomaly episode;
- rule version;
- relevant partition/window.

Keep underlying observations available for diagnosis.

## 6. Intentional Failure

Break the monitoring system deliberately.

### Failure 1 — Cut volume by 80%

Expected:

    volume anomaly

Then verify whether completeness also identifies the missing population.

### Failure 2 — Preserve volume but change distribution

Swap ordinary values for extreme values while preserving row count.

Expected:

    distribution anomaly

### Failure 3 — Collapse cardinality

Make nearly every customer ID identical.

Expected:

    cardinality anomaly

### Failure 4 — Inflate NULL rate

Increase NULLs far above historical baseline.

Expected:

    null-rate anomaly

### Failure 5 — Shift category share

Change a normally dominant category into a minority while keeping all category values valid.

Expected:

    category-distribution anomaly

### Failure 6 — Poison the baseline

Insert a large known incident into baseline history.

Expected:

    baseline eligibility prevents the incident from silently becoming normal

### Failure 7 — Remove historical context

Reduce baseline history below the minimum sample size.

Expected:

    NOT_ENOUGH_HISTORY

### Failure 8 — Create a seasonal false positive

Compare overnight traffic against business-hour history.

Expected:

    contextual baseline prevents a false critical alert

## 7. Recovery

Anomaly recovery begins with classification, not automatic data modification.

### 7.1 Source-delivery anomaly

1. Check completeness and source arrival state.
2. Identify missing or delayed partitions.
3. Confirm source delivery status.
4. Recover the missing scope.
5. Recompute affected metrics.
6. Re-evaluate the anomaly.

### 7.2 Transformation anomaly

1. Compare the anomaly with recent deployments.
2. Inspect filters and joins.
3. Compare pre- and post-transformation distributions.
4. Identify the first stage where the distribution changes.
5. Correct the transformation.
6. Replay the smallest affected range.
7. Recompute downstream metrics.

### 7.3 Legitimate business anomaly

For a legitimate event:

1. confirm the business event;
2. record its effective period;
3. retain anomaly evidence;
4. annotate the event for operational context;
5. decide whether future baseline policy should include comparable events.

Do not delete the anomaly from history.

### 7.4 Baseline recovery

If the baseline becomes contaminated:

1. identify contaminated observations;
2. mark them according to policy;
3. recompute the baseline;
4. re-evaluate affected anomaly decisions;
5. preserve original observations and decisions.

### 7.5 Threshold changes

If business behavior changes:

1. measure the new distribution;
2. compare it with the business contract;
3. determine whether the process or threshold changed;
4. version any threshold adjustment;
5. backtest before production rollout.

### 7.6 Safe replay

Replay only affected partitions or observation windows.

Preserve:

- source batch ID;
- metric observation time;
- detector rule version;
- baseline policy/version;
- original anomaly result;
- replay reason.

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

PostgreSQL can calculate rolling statistics, percentiles, counts, cardinality, category shares, and time-window comparisons close to the data.

The important skill is designing a statistically meaningful comparison population.

### 8.2 dbt

dbt can materialize quality metrics and schedule tests around warehouse models.

Use it for repeatable transformation-layer monitoring and metric generation.

Statistical anomaly logic still requires an explicit baseline and alert policy.

### 8.3 Great Expectations

Great Expectations can provide declarative expectations and validation reporting.

Use it when teams need reusable validation suites and standardized quality reporting.

Do not treat a tool's default threshold as a universal anomaly definition.

## 9. Production Runbook

### Alert

1. Identify dataset, metric, and observation window.
2. Record anomaly score and effect size.
3. Check sample size and baseline freshness.
4. Determine severity.

### Diagnose

5. Compare against the correct historical context.
6. Check seasonality.
7. Check completeness and delivery state.
8. Check deterministic quality failures.
9. Check recent deployments.
10. Compare upstream and downstream metrics.
11. Determine whether the anomaly is real, contextual, or caused by the detector.

### Recover

12. Correct source or transformation defects when confirmed.
13. Replay the smallest affected scope.
14. Recompute metrics.
15. Re-evaluate the anomaly.
16. Reconcile downstream outputs.

### Close

17. Record root cause or business explanation.
18. Preserve anomaly evidence.
19. Record detector and baseline versions.
20. Confirm alert behavior returns to normal.
21. Update baseline policy only when justified.

### Incident decision table

| Observation | Classification | Action |
|---|---|---|
| Large volume drop + missing partition | Delivery issue | Recover missing partition |
| Large volume drop + complete source | Business or filter change | Investigate source and transformation |
| Normal row count + cardinality collapse | Identity/data issue | Inspect keys and deduplication |
| Normal row count + distribution shift | Value transformation or business change | Compare upstream/downstream distributions |
| High score with tiny sample | Insufficient evidence | Wait for more history |
| Seasonal spike | Contextual anomaly | Compare with seasonal baseline |
| Repeated alerts from one incident | Alert duplication | Group anomaly episode |
| Baseline contaminated by known incident | Baseline defect | Exclude according to policy and recompute |
| Legitimate business event | Real anomaly, not necessarily defect | Document and contextualize |

## 10. Common Mistakes

### Mistake 1 — Treating anomalies as invalid records

Unusual does not automatically mean wrong.

### Mistake 2 — Using a global average for seasonal data

Compare like with like.

### Mistake 3 — Letting anomalies contaminate the baseline

Known incidents can become the new normal if eligibility is uncontrolled.

### Mistake 4 — Ignoring sample size

Small populations produce unstable estimates.

### Mistake 5 — Using z-score everywhere

Heavy-tailed and skewed distributions may require robust or percentile-based methods.

### Mistake 6 — Alerting on statistical significance alone

Add meaningful effect-size and business-impact criteria.

### Mistake 7 — Comparing the wrong population

A tenant-specific metric should not automatically use a global baseline.

### Mistake 8 — Including the current observation in its own baseline

This can dilute the anomaly being detected.

### Mistake 9 — Ignoring detector health

A stale baseline can generate both false positives and false negatives.

### Mistake 10 — Creating one alert per metric

High-dimensional monitoring requires grouping, suppression, and severity policy.

### Mistake 11 — Changing thresholds without backtesting

Threshold changes can hide real defects.

### Mistake 12 — Replacing deterministic checks with statistics

Statistical monitoring complements contracts; it does not replace required-field, type, range, completeness, or consistency rules.

## 11. Definition of Done

You are done with T41 when you can:

- explain why statistical anomaly detection complements deterministic validation;
- define an observation population and grain;
- build an appropriate baseline;
- calculate rolling means and standard deviations;
- use robust statistics when appropriate;
- use percentile-based thresholds;
- detect volume anomalies;
- detect distribution shifts;
- monitor NULL rates;
- monitor cardinality;
- monitor category shares;
- account for seasonality;
- enforce minimum sample sizes;
- prevent baseline contamination;
- separate statistical score from business impact;
- version detector rules and baseline policies;
- persist anomaly evidence;
- implement core calculations in SQL;
- implement core statistical helpers in Python;
- test spikes, shifts, seasonality, small samples, and false positives;
- intentionally create anomalies;
- correlate anomaly signals with deterministic quality checks;
- recover source or transformation defects safely;
- backtest threshold changes;
- operate anomaly detection with the production runbook.

## 12. What You Learned

Statistical anomaly detection answers a different question from deterministic validation.

The core mental model is:

    DEFINE POPULATION
          ↓
    BUILD COMPARABLE BASELINE
          ↓
    MEASURE CURRENT OBSERVATION
          ↓
    CALCULATE DEVIATION
          ↓
    APPLY EFFECT + BUSINESS CONTEXT
          ↓
    CLASSIFY
          ↓
    INVESTIGATE / RECOVER
          ↓
    RECHECK

You learned to:

- distinguish unusual from invalid;
- choose baselines that match time, population, and business context;
- use robust and percentile-based statistics when appropriate;
- monitor volume, distributions, NULL rates, cardinality, and category shares;
- account for seasonality and trend;
- protect baselines from known incidents;
- avoid overreacting to tiny samples;
- combine statistical signals with deterministic quality checks;
- preserve anomaly evidence and detector versions;
- recover the underlying pipeline problem without destroying historical evidence.

The production principle is:

> **A dataset can satisfy every deterministic rule and still behave abnormally. Statistical monitoring detects changes in behavior that explicit invariants cannot describe.**

### Next Recipe

**T42 — Data Quality Scoring**

T42 will combine multiple quality signals into an explicit, auditable data-quality score without hiding the individual checks underneath one number.
