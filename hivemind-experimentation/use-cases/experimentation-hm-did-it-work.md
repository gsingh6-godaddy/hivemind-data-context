---
type: use-case
entry_id: experimentation-hm-did-it-work
products: [experimentation]
pnl_lines: []
intents: [hm_verdict, hm_significance_history, hm_readout]
status: DETERMINISTIC
historical_question_count: 0
last_updated: 2026-11-13
sources:
  governed_articles: [521, 439]
methodology:
  authority: pattern_doc
  source_chain: [hivemind_mart.experiment_metadata_snap, hivemind_mart.analysis_result_cln]
  refresh: "daily (~D-1)"
derived_from: []
supersedes: null
verified_on: 2026-10-22
notes:
  - "Merged 2026-11-13 from Path A of experimentation-hm-did-it-work, -significance-history and -readout (SQL unchanged)."
  - "Single-experiment lookups: do not filter experiment_type_enum, but report it and warn if it is not A/B."
---

# Did experiment X work? — verdict, significance before rollout, full readout

> Single-experiment questions about whether a Hivemind A/B test won: the decision-metric verdict on the latest report, whether the result was still significant right before rollout (and whether it flickered), and the full readout (what, who, when, every scorecard metric).

## Routing — pick the path first

| Question shape | Path |
|---|---|
| "Did X win?", "what's the lift / is it significant?", "lift per variant" | **Path A — Verdict** |
| "Was it still significant before rollout?", "did it flicker?", "was the loss/win significant on the last report before rollout?" | **Path B — Significance history** |
| "Full readout", "what was tested, who owns it, when did it run, what did the scorecard show", "hypothesis, owners, dates, all metrics" | **Path C — Readout** (includes the Path A verdict rows) |
| Money, iGCR, $ per day, product-line GCR | not here, use experimentation-hm-whats-it-worth |
| "Should we roll out?", guardrail harm when none are configured, post-rollout durability, segment winners | **decline** (see Out of scope) |

**Timezone rule (applies to every path):** `rollout_mst_ts` holds the Hivemind rolloutDate as a **UTC** instant despite its name (verified against `get_experiment_by_id` rolloutDate on 10 experiments, 2026-11-13). Always use `DATEADD(hour, -7, rollout_mst_ts)` as the MST rollout time, label times you show as MST, and compare report run times (`insert_utc_ts` − 7h) to the converted value. A value of exactly 00:00:00 UTC is a date-only rollout = 17:00 MST the previous day.

Shared rules for all paths: overall rows only (all 8 segment columns = 'ALL'), SELECT DISTINCT, strip the 32-hex hash from metric_id, treatment rows are cohort_id <> control_cohort, use the experiment's own alpha and metric_significance_flag (never 0.05), real dates come from first_user_bucket_mst_ts / last_user_bucket_mst_ts (never ended_at_mst_ts / end_mst_date).

## Canonical Interpretations

### Path A — Verdict  _(Latest decision-metric stats + declared result)_
- **Source tables**: `hivemind_mart.analysis_result_cln` (stats), `hivemind_mart.experiment_metadata_snap` (declared result, automatic conclusion, configured metric lists)
- **Definition**: For each treatment cohort (`cohort_id <> control_cohort`) and each decision metric, the relative lift vs control (`metric_relative_delta`), p-value (`metric_pvalue`), significance at the experiment's own alpha (`metric_significance_flag`, `alpha`) and the relative CI, from the latest `report_date` that has decision rows.
- **Time anchor**: `report_date` (STRING 'YYYY-MM-DD') — label of the Hivemind analysis run. Latest one with decision rows.
- **Grain**: one row per (decision metric, treatment cohort) on that report.
- **Key filters**: `metric_category = 'decision'`; all 8 segment columns = `'ALL'`; `SELECT DISTINCT`; metric id hash suffix stripped with `REGEXP_REPLACE(metric_id, '_[0-9a-f]{32}$', '')`.
- **What it measures**: whether the change moved the metric(s) the experiment was set up to be decided on, and in which direction.
- **How to phrase the answer**:
  1. State the declared result (`rollout_decision_enum`) and, if present, `automated_conclusion_enum`.
  2. For each arm × decision metric: control value, treatment value, lift (as %), p-value, alpha, significant yes/no, analysis method (`fixed` / `sequential`) and analysis type.
  3. Compare `latest_stats_call` with the declared result. If they disagree, say so explicitly and give the likely reason (see Gotchas: do-no-harm tests, post-rollout reruns, multi-arm tests where only one arm moved, sequential boundary not crossed, decision metric changed, analysis skipped).
  4. If `bigger_is_better_flag` makes a direction look odd (e.g. an opt-out rate configured as bigger-is-better), report the raw direction and the flag; do not reinterpret.
- **Bayesian branch**: if `analysis_type = 'bayesian'`, p-value, `metric_relative_delta` and `alpha` are NULL. Report `metric_pbest` (probability treatment is best) and `credible_interval` instead and do not call it "significant" / "not significant" in the frequentist sense.
- **SQL**:
  ```sql
  WITH meta AS (
    SELECT experiment_id, experiment_name, experiment_type_enum, business_unit_name, rollout_decision_enum, automated_conclusion_enum,
           sig_lift_decision_metric_id_list, sig_loss_decision_metric_id_list, decision_metric_id_list, guardrail_metric_id_list,
           baseline_cohort_id, alpha_value, first_user_bucket_mst_ts, last_user_bucket_mst_ts, rollout_mst_ts
    FROM hivemind_mart.experiment_metadata_snap
    WHERE experiment_id = '<experiment_id>'
  ),
  dec AS (
    SELECT DISTINCT report_date, REGEXP_REPLACE(metric_id, '_[0-9a-f]{32}$', '') AS metric_base, cohort_id, control_cohort,
           analysis_type, analysis_method_used, alpha, bigger_is_better_flag, metric_value, metric_relative_delta,
           relative_confidence_interval, metric_pvalue, metric_significance_flag, metric_pbest, credible_interval, metric_warning
    FROM hivemind_mart.analysis_result_cln
    WHERE experiment_id = '<experiment_id>'
      AND metric_category = 'decision'
      AND new_existing_customer_name = 'ALL' AND market_site = 'ALL' AND device_name = 'ALL' AND ip_country_name = 'ALL'
      AND customer_tenure_day_segment = 'ALL' AND domain_ownership_count = 'ALL' AND customer_type = 'ALL' AND market_sub_region = 'ALL'
  ),
  latest AS (SELECT MAX(report_date) AS report_date FROM dec),
  ctrl AS (
    SELECT d.metric_base, d.metric_value AS control_value
    FROM dec d JOIN latest l ON d.report_date = l.report_date
    WHERE d.cohort_id = d.control_cohort
  ),
  trt AS (
    SELECT d.*, c.control_value,
           CASE
             WHEN d.analysis_type = 'bayesian' THEN 'bayesian - read metric_pbest and credible_interval'
             WHEN d.metric_pvalue IS NULL THEN 'no test computed - see metric_warning'
             WHEN d.metric_significance_flag AND ((d.metric_relative_delta > 0) = COALESCE(d.bigger_is_better_flag, TRUE)) THEN 'significant improvement'
             WHEN d.metric_significance_flag THEN 'significant harm'
             ELSE 'not significant'
           END AS stat_read
    FROM dec d
    JOIN latest l ON d.report_date = l.report_date
    LEFT JOIN ctrl c ON c.metric_base = d.metric_base
    WHERE d.cohort_id <> d.control_cohort
  )
  SELECT m.experiment_id, m.experiment_type_enum, m.business_unit_name,
         m.rollout_decision_enum AS declared_result, m.automated_conclusion_enum AS automated_conclusion,
         m.decision_metric_id_list, m.sig_lift_decision_metric_id_list, m.sig_loss_decision_metric_id_list, m.guardrail_metric_id_list,
         DATEADD(hour, -7, m.rollout_mst_ts) AS rollout_mst,
         CASE WHEN m.rollout_mst_ts IS NOT NULL AND t.report_date >= TO_CHAR(DATEADD(hour, -7, m.rollout_mst_ts), 'YYYY-MM-DD') THEN 'Y' ELSE 'N' END AS latest_report_after_rollout,
         t.report_date, t.metric_base, t.cohort_id AS treatment_cohort, t.control_cohort, t.analysis_type, t.analysis_method_used,
         t.alpha, t.bigger_is_better_flag, t.control_value, t.metric_value AS treatment_value, t.metric_relative_delta,
         t.relative_confidence_interval, t.metric_pvalue, t.metric_significance_flag, t.metric_pbest, t.credible_interval,
         t.stat_read,
         CASE
           WHEN t.report_date IS NULL THEN 'no decision-metric rows in analysis_result_cln'
           WHEN MAX(CASE WHEN t.stat_read = 'significant harm' THEN 1 ELSE 0 END) OVER () = 1
            AND MAX(CASE WHEN t.stat_read = 'significant improvement' THEN 1 ELSE 0 END) OVER () = 1 THEN 'mixed - one arm significantly better, another significantly worse'
           WHEN MAX(CASE WHEN t.stat_read = 'significant improvement' THEN 1 ELSE 0 END) OVER () = 1 THEN 'at least one arm significantly better'
           WHEN MAX(CASE WHEN t.stat_read = 'significant harm' THEN 1 ELSE 0 END) OVER () = 1 THEN 'at least one arm significantly worse'
           WHEN MAX(CASE WHEN t.analysis_type = 'bayesian' THEN 1 ELSE 0 END) OVER () = 1 THEN 'bayesian - no p-value call'
           WHEN MIN(CASE WHEN t.stat_read LIKE 'no test%' THEN 1 ELSE 0 END) OVER () = 1 THEN 'no test computed on any decision metric'
           ELSE 'no arm significant'
         END AS latest_stats_call,
         t.metric_warning
  FROM meta m
  LEFT JOIN trt t ON 1 = 1
  ORDER BY t.metric_base, t.cohort_id
  ```
  If the query returns one row with NULL `report_date`, the experiment has no decision-metric rows (empty `decision_metric_id_list`, or analysis never ran) — say so; do not fall back to informative metrics as if they were the verdict.

### Path B — Significance history  _(Decision-metric report history anchored to rollout)_
- **Source tables**: `hivemind_mart.analysis_result_cln` (daily reports), `hivemind_mart.experiment_metadata_snap` (`rollout_mst_ts`, real start/end, declared result)
- **Definition**: For each decision metric × treatment arm, the sequence of (`metric_relative_delta`, `metric_pvalue`, `metric_significance_flag`) across `report_date`s up to the anchor report, restricted to the anchor report's `analysis_type` and `analysis_method_used`.
- **Time anchor**:
  - `rollout_mst_ts` present and on/after `first_user_bucket_mst_ts` → last report whose run time (`insert_utc_ts` − 7h, MST) is ≤ the MST rollout time `DATEADD(hour, -7, rollout_mst_ts)` (the column is UTC — see the timezone rule).
  - otherwise → last `report_date` ≤ `last_user_bucket_mst_ts` + 1 day (real end, R2). Never `ended_at_mst_ts` or `end_mst_date`.
- **Grain**: Step 1 one row per (report_date, decision metric, treatment arm); Step 2 one row per (decision metric, treatment arm).
- **Key filters**: `metric_category = 'decision'`; all 8 segment columns = `'ALL'`; `SELECT DISTINCT`; hash-stripped metric id; `cohort_id <> control_cohort`.
- **How to phrase the answer**:
  1. Anchor: which report, when it ran, what it was anchored to (rollout vs real end), method and alpha.
  2. At the anchor: lift, p-value, significant yes/no — and whether it agrees with the declared result.
  3. History (same method): reports available vs calendar days spanned, number of significant reports, first significant report, significance flips, sign flips, last time it lost significance.
  4. If `n_methods_in_life > 1`, say the method switched and summarise the other-method history separately from the detail query (never as one series). When the anchor method has only one or two reports (a switch on the final report is common), the Step 2 summary cannot show flicker on its own — always run Step 1 and base the flicker answer on the longer other-method series. Step 1 can return more rows than the results preview shows; aggregate it (count significant reports, first/last significant date, flips) rather than reading rows by eye.
  5. Always qualify with "on the reports available" — gaps exist.
  6. **Flicker needs a complete run series.** Count report-date gaps between the first report and the anchor (`missing_days_before` in Step 1, or `calendar_days_spanned − n_reports_available`). If there are gaps, never answer "it did not flicker": say "no flicker on the N reports in Redshift; D days have no report in Redshift (some may never have run), so flicker cannot be ruled out — confirm with Hivemind `get_experiment_analysis_history`". Do not assert the gap days were real runs, and do not assert they were not. Report flips from Redshift only as a lower bound ("at least N significance flips"). Never write "never lost significance", "stayed significant throughout" or "the missing run may not have happened" when there are gaps. Find gaps from **run times, not report_date labels**: a gap is a row with `run_gap_flag = 'RUN GAP'` (more than 36 h since the previous run). Labels can hide or invent gaps — a run that executes just after midnight carries the previous day's label (e.g. label 2026-09-29 holding a 2026-09-30 00:26 MST run), so a skipped label is not necessarily a missing run, and a missing run can sit under consecutive labels. Name each gap as "no run in Redshift between <prev_run_mst_ts> and <run_mst_ts> MST" and say a run in that window may have changed significance.
  8. **Narrate direction from the full Step 1 series.** When you describe when the lift was positive or negative, take the date spans from the full Step 1 daily rows (first and last report of each same-sign stretch), not from a filtered or significance-only query.
  7. **Say which rollout time you anchored on**, in MST. If the UTC value is exactly 00:00:00 (date-only), the real rollout time is unknown within that MST day: name the anchor run and the run before it, and say the anchor would move one run earlier if rollout happened before the last run.
- **SQL — Step 1, daily detail** (all methods, flagged):
  ```sql
  WITH meta AS (
    SELECT experiment_id, rollout_decision_enum, rollout_mst_ts, first_user_bucket_mst_ts, last_user_bucket_mst_ts,
           CASE WHEN rollout_mst_ts IS NULL OR rollout_mst_ts < first_user_bucket_mst_ts THEN 'real_end' ELSE 'rollout' END AS anchor_source
    FROM hivemind_mart.experiment_metadata_snap
    WHERE experiment_id = '<experiment_id>'
  ),
  dec AS (
    SELECT DISTINCT report_date, REGEXP_REPLACE(metric_id, '_[0-9a-f]{32}$', '') AS metric_base, cohort_id, control_cohort,
           analysis_type, analysis_method_used, alpha, metric_relative_delta, metric_pvalue, metric_significance_flag, metric_pbest, metric_warning
    FROM hivemind_mart.analysis_result_cln
    WHERE experiment_id = '<experiment_id>'
      AND metric_category = 'decision'
      AND new_existing_customer_name = 'ALL' AND market_site = 'ALL' AND device_name = 'ALL' AND ip_country_name = 'ALL'
      AND customer_tenure_day_segment = 'ALL' AND domain_ownership_count = 'ALL' AND customer_type = 'ALL' AND market_sub_region = 'ALL'
  ),
  report_runs AS (
    SELECT report_date, COALESCE(DATEADD(hour, -7, MAX(insert_utc_ts)), TO_DATE(report_date, 'YYYY-MM-DD')) AS run_mst_ts
    FROM hivemind_mart.analysis_result_cln
    WHERE experiment_id = '<experiment_id>' AND metric_category = 'decision'
    GROUP BY report_date
  ),
  anchor_report AS (
    SELECT MAX(r.report_date) AS anchor_report_date
    FROM report_runs r CROSS JOIN meta m
    WHERE (m.anchor_source = 'rollout' AND r.run_mst_ts <= DATEADD(hour, -7, m.rollout_mst_ts))
       OR (m.anchor_source = 'real_end' AND r.report_date <= TO_CHAR(DATEADD(day, 1, m.last_user_bucket_mst_ts), 'YYYY-MM-DD'))
  ),
  anchor_method AS (
    SELECT MIN(d.analysis_type) AS anchor_analysis_type, MIN(d.analysis_method_used) AS anchor_method
    FROM dec d JOIN anchor_report a ON d.report_date = a.anchor_report_date
  ),
  hist AS (
    SELECT d.report_date, r.run_mst_ts, d.metric_base, d.cohort_id, d.analysis_type, d.analysis_method_used, d.alpha,
           d.metric_relative_delta, d.metric_pvalue, d.metric_significance_flag, d.metric_pbest, d.metric_warning,
           LAG(SIGN(d.metric_relative_delta)) OVER (PARTITION BY d.metric_base, d.cohort_id, d.analysis_type, d.analysis_method_used ORDER BY d.report_date) AS prev_sign,
           LAG(d.metric_significance_flag) OVER (PARTITION BY d.metric_base, d.cohort_id, d.analysis_type, d.analysis_method_used ORDER BY d.report_date) AS prev_sig,
           LAG(d.report_date) OVER (PARTITION BY d.metric_base, d.cohort_id, d.analysis_type, d.analysis_method_used ORDER BY d.report_date) AS prev_report_date,
           LAG(r.run_mst_ts) OVER (PARTITION BY d.metric_base, d.cohort_id, d.analysis_type, d.analysis_method_used ORDER BY d.report_date) AS prev_run_mst_ts
    FROM dec d LEFT JOIN report_runs r ON r.report_date = d.report_date
    WHERE d.cohort_id <> d.control_cohort
  )
  SELECT h.report_date, h.run_mst_ts, h.metric_base, h.cohort_id, h.analysis_type, h.analysis_method_used,
         CASE WHEN h.analysis_type = am.anchor_analysis_type AND h.analysis_method_used = am.anchor_method THEN 'Y' ELSE 'N' END AS same_method_as_anchor,
         CASE WHEN h.report_date = a.anchor_report_date THEN 'ANCHOR' WHEN h.report_date > a.anchor_report_date THEN 'after anchor' ELSE '' END AS anchor_flag,
         h.alpha, h.metric_relative_delta, h.metric_pvalue, h.metric_significance_flag,
         DATEDIFF(day, TO_DATE(h.prev_report_date, 'YYYY-MM-DD'), TO_DATE(h.report_date, 'YYYY-MM-DD')) - 1 AS missing_days_before,
         h.prev_run_mst_ts,
         ROUND(DATEDIFF(minute, h.prev_run_mst_ts, h.run_mst_ts) / 60.0, 1) AS hours_since_prev_run,
         CASE WHEN DATEDIFF(minute, h.prev_run_mst_ts, h.run_mst_ts) > 36 * 60 THEN 'RUN GAP' ELSE '' END AS run_gap_flag,
         CASE WHEN h.prev_sign IS NOT NULL AND h.prev_sign <> 0 AND SIGN(h.metric_relative_delta) <> 0 AND SIGN(h.metric_relative_delta) <> h.prev_sign THEN 'SIGN FLIP' ELSE '' END AS sign_flip,
         CASE WHEN h.prev_sig IS NOT NULL AND h.metric_significance_flag <> h.prev_sig THEN CASE WHEN h.metric_significance_flag THEN 'became significant' ELSE 'lost significance' END ELSE '' END AS significance_flip,
         h.metric_warning
  FROM hist h CROSS JOIN anchor_report a CROSS JOIN anchor_method am
  ORDER BY h.metric_base, h.cohort_id, h.report_date
  LIMIT 500
  ```
- **SQL — Step 2, summary per metric × arm** (anchor method only, up to the anchor):
  ```sql
  WITH meta AS (
    SELECT experiment_id, rollout_decision_enum, automated_conclusion_enum, rollout_mst_ts, first_user_bucket_mst_ts, last_user_bucket_mst_ts,
           CASE WHEN rollout_mst_ts IS NULL OR rollout_mst_ts < first_user_bucket_mst_ts THEN 'real_end' ELSE 'rollout' END AS anchor_source
    FROM hivemind_mart.experiment_metadata_snap
    WHERE experiment_id = '<experiment_id>'
  ),
  dec AS (
    SELECT DISTINCT report_date, REGEXP_REPLACE(metric_id, '_[0-9a-f]{32}$', '') AS metric_base, cohort_id, control_cohort,
           analysis_type, analysis_method_used, alpha, metric_relative_delta, metric_pvalue, metric_significance_flag
    FROM hivemind_mart.analysis_result_cln
    WHERE experiment_id = '<experiment_id>'
      AND metric_category = 'decision'
      AND new_existing_customer_name = 'ALL' AND market_site = 'ALL' AND device_name = 'ALL' AND ip_country_name = 'ALL'
      AND customer_tenure_day_segment = 'ALL' AND domain_ownership_count = 'ALL' AND customer_type = 'ALL' AND market_sub_region = 'ALL'
  ),
  report_runs AS (
    SELECT report_date, COALESCE(DATEADD(hour, -7, MAX(insert_utc_ts)), TO_DATE(report_date, 'YYYY-MM-DD')) AS run_mst_ts
    FROM hivemind_mart.analysis_result_cln
    WHERE experiment_id = '<experiment_id>' AND metric_category = 'decision'
    GROUP BY report_date
  ),
  anchor_report AS (
    SELECT MAX(r.report_date) AS anchor_report_date
    FROM report_runs r CROSS JOIN meta m
    WHERE (m.anchor_source = 'rollout' AND r.run_mst_ts <= DATEADD(hour, -7, m.rollout_mst_ts))
       OR (m.anchor_source = 'real_end' AND r.report_date <= TO_CHAR(DATEADD(day, 1, m.last_user_bucket_mst_ts), 'YYYY-MM-DD'))
  ),
  anchor_method AS (
    SELECT MIN(d.analysis_type) AS anchor_analysis_type, MIN(d.analysis_method_used) AS anchor_method
    FROM dec d JOIN anchor_report a ON d.report_date = a.anchor_report_date
  ),
  hist AS (
    SELECT d.*,
           LAG(SIGN(d.metric_relative_delta)) OVER (PARTITION BY d.metric_base, d.cohort_id ORDER BY d.report_date) AS prev_sign,
           LAG(d.metric_significance_flag) OVER (PARTITION BY d.metric_base, d.cohort_id ORDER BY d.report_date) AS prev_sig
    FROM dec d
    JOIN anchor_method am ON d.analysis_type = am.anchor_analysis_type AND d.analysis_method_used = am.anchor_method
    CROSS JOIN anchor_report a
    WHERE d.cohort_id <> d.control_cohort AND d.report_date <= a.anchor_report_date
  )
  SELECT m.experiment_id, m.rollout_decision_enum AS declared_result, m.automated_conclusion_enum, m.anchor_source, DATEADD(hour, -7, m.rollout_mst_ts) AS rollout_mst, m.last_user_bucket_mst_ts,
         a.anchor_report_date, am.anchor_analysis_type, am.anchor_method,
         (SELECT COUNT(DISTINCT analysis_method_used) FROM dec) AS n_methods_in_life,
         h.metric_base, h.cohort_id, MAX(h.alpha) AS alpha,
         COUNT(*) AS n_reports_available,
         MIN(h.report_date) AS first_report_date,
         DATEDIFF(day, TO_DATE(MIN(h.report_date), 'YYYY-MM-DD'), TO_DATE(a.anchor_report_date, 'YYYY-MM-DD')) + 1 AS calendar_days_spanned,
         SUM(CASE WHEN h.metric_significance_flag THEN 1 ELSE 0 END) AS n_reports_significant,
         SUM(CASE WHEN h.prev_sign IS NOT NULL AND h.prev_sign <> 0 AND SIGN(h.metric_relative_delta) <> 0 AND SIGN(h.metric_relative_delta) <> h.prev_sign THEN 1 ELSE 0 END) AS n_sign_flips,
         SUM(CASE WHEN h.prev_sig IS NOT NULL AND h.metric_significance_flag <> h.prev_sig THEN 1 ELSE 0 END) AS n_significance_flips,
         MIN(CASE WHEN h.metric_significance_flag THEN h.report_date END) AS first_significant_report,
         MAX(CASE WHEN h.prev_sig AND NOT h.metric_significance_flag THEN h.report_date END) AS last_lost_significance,
         MAX(CASE WHEN h.report_date = a.anchor_report_date THEN h.metric_relative_delta END) AS anchor_relative_delta,
         MAX(CASE WHEN h.report_date = a.anchor_report_date THEN h.metric_pvalue END) AS anchor_pvalue,
         MAX(CASE WHEN h.report_date = a.anchor_report_date THEN CASE WHEN h.metric_significance_flag THEN 1 ELSE 0 END END) AS anchor_significant
  FROM hist h CROSS JOIN meta m CROSS JOIN anchor_report a CROSS JOIN anchor_method am
  GROUP BY m.experiment_id, m.rollout_decision_enum, m.automated_conclusion_enum, m.anchor_source, m.rollout_mst_ts, m.last_user_bucket_mst_ts,
           a.anchor_report_date, am.anchor_analysis_type, am.anchor_method, h.metric_base, h.cohort_id
  ORDER BY h.metric_base, h.cohort_id
  ```
  `calendar_days_spanned − n_reports_available` = days with no report in Redshift.

### Path C — Readout  _(Metadata snapshot + full latest scorecard)_
- **Source tables**: `hivemind_mart.experiment_metadata_snap` (one row per experiment), `hivemind_mart.analysis_result_cln` (scorecard)
- **Definition**: Step 1 returns the experiment's configuration and outcome fields; Step 2 returns, for the latest `report_date` with scorecard rows, one row per (metric, treatment arm) with control value, treatment value, lift, p-value, significance and warnings, grouped decision → guardrail → informative.
- **Time anchor**: real start `first_user_bucket_mst_ts`, real end `last_user_bucket_mst_ts`; configured start `start_mst_date` shown separately. Scorecard = latest `report_date` with non-health rows.
- **Grain**: Step 1 one row; Step 2 one row per (hash-stripped metric id, treatment cohort).
- **Key filters**: `metric_category IN ('decision','guardrail','informative')` (excludes `OverallHealth`); all 8 segment columns = `'ALL'`; `SELECT DISTINCT`; if a metric appears in more than one category on the same report, keep decision > guardrail > informative (`n_dup_rows` shows if this happened).
- **How to phrase the answer**:
  1. **What**: name, type (warn if not `A/B`), hypothesis change / expectation / reason (as entered; may be blank).
  2. **Who**: owners (`experiment_owner_id_list`, comma-separated usernames), BU (canonical), team as entered.
  3. **When**: configured start, real start, real end, real duration in days, rollout time in MST (`rollout_mst`; the raw column is UTC).
  4. **How**: scorecard template, analysis type, method, alpha, cohorts and baseline, traffic type.
  5. **Outcome**: declared result, automated conclusion, badge (and unmet badging criteria if any).
  6. **Scorecard**: decision metrics first, then guardrails, then informative — per arm. Summarise informative metrics rather than dumping all rows unless asked, and always give a **count line per category** computed from the Step 2 rows, not examples: `<category>: N metrics × A arms = R arm-rows; S significant; U not tested (NULL p-value)`. Count with `SELECT metric_category, COUNT(DISTINCT metric_base), COUNT(*), SUM(CASE WHEN metric_significance_flag THEN 1 ELSE 0 END), SUM(CASE WHEN metric_pvalue IS NULL THEN 1 ELSE 0 END)` over the Step 2 result grouped by `metric_category`.
  7. **Direction as configured.** Quote `stat_read` for every significant decision/guardrail row. If the configured direction looks counter-intuitive (e.g. an opt-out rate configured bigger-is-better, so a significant drop reads as harm), say exactly that: "Hivemind scores this as significant harm because the metric is configured bigger-is-better; a lower opt-out rate is usually desirable — check the metric config." Never silently relabel it good/bad.
- **SQL — Step 1, metadata**:
  ```sql
  SELECT experiment_id, experiment_name, experiment_type_enum, status_enum,
         CASE
           WHEN business_unit_name IS NULL OR TRIM(business_unit_name) IN ('', 'N/A', 'Other') THEN '(unassigned)'
           WHEN LOWER(TRIM(business_unit_name)) IN ('customer & site','customer and site','cust & site') THEN 'Customer & Site'
           WHEN business_unit_name IN ('Airo Growth & Innovation','Airo Growth  Innovation') THEN 'AGI'
           WHEN business_unit_name = 'USI' THEN 'US Independents'
           WHEN LOWER(TRIM(business_unit_name)) = 'corporate marketing' THEN 'Corporate Marketing'
           ELSE TRIM(business_unit_name)
         END AS business_unit,
         team_name AS team_name_as_entered,
         experiment_owner_id_list,
         hypothesis_change_text, hypothesis_expectation_text, hypothesis_reason_text,
         scorecard_template_name, analysis_type, analysis_method_enum, fixed_design_flag, alpha_value,
         cohort_name_list, baseline_cohort_id, traffic_type_enum,
         start_mst_date AS configured_start_date,
         first_user_bucket_mst_ts AS real_start_mst_ts,
         last_user_bucket_mst_ts AS real_end_mst_ts,
         ROUND(DATEDIFF(minute, first_user_bucket_mst_ts, last_user_bucket_mst_ts) / 1440.0, 1) AS real_duration_days,
         DATEADD(hour, -7, rollout_mst_ts) AS rollout_mst, rollout_mst_ts AS rollout_utc, rollout_decision_enum, automated_conclusion_enum, badge_name, unmet_badging_criteria_list,
         decision_metric_id_list, guardrail_metric_id_list, informative_metrics_list
  FROM hivemind_mart.experiment_metadata_snap
  WHERE experiment_id = '<experiment_id>'
  ```
- **SQL — Step 2, full scorecard at latest report**:
  ```sql
  WITH res AS (
    SELECT DISTINCT report_date, REGEXP_REPLACE(metric_id, '_[0-9a-f]{32}$', '') AS metric_base, metric_category, label, cohort_id, control_cohort,
           analysis_type, analysis_method_used, alpha, bigger_is_better_flag, metric_value, metric_relative_delta, relative_confidence_interval,
           metric_pvalue, metric_significance_flag, metric_pbest, credible_interval, metric_warning
    FROM hivemind_mart.analysis_result_cln
    WHERE experiment_id = '<experiment_id>'
      AND metric_category IN ('decision', 'guardrail', 'informative')
      AND new_existing_customer_name = 'ALL' AND market_site = 'ALL' AND device_name = 'ALL' AND ip_country_name = 'ALL'
      AND customer_tenure_day_segment = 'ALL' AND domain_ownership_count = 'ALL' AND customer_type = 'ALL' AND market_sub_region = 'ALL'
  ),
  latest AS (SELECT MAX(report_date) AS report_date FROM res),
  ranked AS (
    SELECT r.*,
           ROW_NUMBER() OVER (PARTITION BY r.metric_base, r.cohort_id
                              ORDER BY CASE r.metric_category WHEN 'decision' THEN 1 WHEN 'guardrail' THEN 2 ELSE 3 END) AS category_rank,
           COUNT(*) OVER (PARTITION BY r.metric_base, r.cohort_id) AS n_dup_rows
    FROM res r JOIN latest l ON r.report_date = l.report_date
  ),
  ctrl AS (
    SELECT metric_base, metric_value AS control_value FROM ranked WHERE cohort_id = control_cohort AND category_rank = 1
  )
  SELECT t.report_date, t.metric_category, t.metric_base, t.label, t.cohort_id AS treatment_cohort, t.control_cohort,
         t.analysis_type, t.analysis_method_used, t.alpha, c.control_value, t.metric_value AS treatment_value,
         t.metric_relative_delta, t.relative_confidence_interval, t.metric_pvalue, t.metric_significance_flag,
         t.metric_pbest, t.credible_interval, t.bigger_is_better_flag,
         CASE
           WHEN t.analysis_type = 'bayesian' THEN 'bayesian - read metric_pbest'
           WHEN t.metric_pvalue IS NULL THEN 'not tested - see metric_warning'
           WHEN t.metric_significance_flag AND ((t.metric_relative_delta > 0) = COALESCE(t.bigger_is_better_flag, TRUE)) THEN 'significant improvement (per bigger_is_better_flag)'
           WHEN t.metric_significance_flag THEN 'significant harm (per bigger_is_better_flag)'
           ELSE 'not significant'
         END AS stat_read,
         t.n_dup_rows, t.metric_warning
  FROM ranked t LEFT JOIN ctrl c ON c.metric_base = t.metric_base
  WHERE t.category_rank = 1 AND t.cohort_id <> t.control_cohort
  ORDER BY CASE t.metric_category WHEN 'decision' THEN 1 WHEN 'guardrail' THEN 2 ELSE 3 END, t.metric_base, t.cohort_id
  LIMIT 500
  ```
  `metric_base` is already hash-stripped; show it (and `label`) rather than raw `metric_id`.

## Out of scope (decline)

Decline these even when the data looks like it could answer them: say what cannot be answered, why, and where to go instead. Never invent a number.

### For Path A (Verdict)
- **"Should we roll out / ship it?"** — a recommendation is a judgment (business context, guardrails, cost, strategy). Report the stats and the declared result; the decision belongs to the experiment owner / rollout process.
- **"Did it hurt any guardrails?" when `guardrail_metric_id_list` is empty** — no guardrails were configured, so there is no guardrail verdict. Say so; informative metrics may be listed via Path C of experimentation-hm-did-it-work, but they are not guardrails and should not be presented as a harm/no-harm check.
- **Did the lift hold after rollout / durability** — Commercial OS post-rollout tooling (`get_commercial_os_post_rollout_metrics`), not this table.
- **Dollar impact / iGCR** — `annualization_value` (governed iGCR, Alation doc #521) lives in a separate entry; do not convert the lift into money here.
- **Was it significant before rollout / did it flicker** — Path B of experimentation-hm-did-it-work.
- **Segment-level winners** — segment rows exist but segment deltas are not tested for treatment-effect differences; out of scope here.

### For Path B (Significance history)
- **Did the lift hold after rollout / durability** — Commercial OS post-rollout metrics (`get_commercial_os_post_rollout_metrics`). Post-rollout reports in `analysis_result_cln` are reruns of the experiment window, not post-rollout measurement.
- **Was it significant on the day of the readout / decision meeting** — the meeting date is not in Redshift; only rollout time and real end are.
- **"Would it have become significant if we ran longer?"** — counterfactual; no data answers it.
- **Every Hivemind run including missing ones** — the Hivemind API (`get_experiment_analysis_history`) holds runs not loaded into Redshift; use it if a complete run list is required.
- **Should we roll out** — judgment, see Path A of experimentation-hm-did-it-work.

### For Path C (Readout)
- **Money / iGCR / "how much did it make"** — governed iGCR is `annualization_value` (Alation doc #521, discount already applied, per `annualization_type_enum`) and belongs to the iGCR entry; never convert scorecard lifts to dollars here and never re-discount.
- **Team breakdowns or team rankings** — `team_name` is one unreliable free-text label per experiment (35%+ NULL, spelling drift). Show it as entered; do not group by it.
- **Should we roll out** — judgment; see Path A of experimentation-hm-did-it-work for the factual verdict.
- **Significance over time / before rollout** — Path B of experimentation-hm-did-it-work.
- **SRM / sample health** — health uses a different report rule (last report ≤ real end, ignoring 100/0 reruns); not this entry.
- **Readout slides, Slack announcement, approver, who clicked rollout** — Hivemind UI / API (`get_experiment_by_id`), not in Redshift.

## Gotchas

### Path A — Verdict
- **Declared vs stats can legitimately disagree.** Seen on validation: a "DoNoHarm" infra migration declared `win` with no significant decision metric (win = no harm detected); a 4-arm test declared `inconclusive` while one arm was significantly worse (the shipped arm was not that one); a test declared `win` whose sequential p-value never crossed the boundary at the end. Always show both, never overwrite `rollout_decision_enum` with the stats call.
- **`automated_conclusion_enum` is sparse and can differ from the human call** (e.g. WIN vs win, INCONCLUSIVE vs win). It is Hivemind's automatic call, not the owner's.
- **`sig_lift_decision_metric_id_list` / `sig_loss_decision_metric_id_list`** are comma-separated bare ids and can repeat ids — dedupe before counting. They reflect Hivemind's snapshot, which can lag or lead the latest report.
- **Latest report can be a post-rollout rerun.** After rollout Hivemind re-runs the analysis with expected split 100/0 (`expected_proportion` 0/1 on OverallHealth rows). Decision stats are still computed on the original bucketed users and may include the final days of data, so they are valid for the verdict, but the SRM / health on that report is spurious. `latest_report_after_rollout = 'Y'` flags this.
- **Multi-treatment**: one row per arm; p-values can be multiplicity-adjusted (identical p-values across arms are expected). Name the arm, never collapse to one "treatment".
- **Sequential analysis**: `metric_pvalue = 1` (or very high) means the boundary was not crossed, not "no effect". `analysis_method_used` can switch from `sequential` to `fixed` on the final report — state which method the verdict report used.
- **Analysis skipped**: `metric_pvalue` NULL with a frequentist report and `metric_warning` like "Analysis skipped: ['off'] cohorts only have one value" means no test was computed (e.g. the event only fires in treatment). Do not call this "not significant".
- **Alpha is per experiment** (0.1 or 0.2 is common; control rows have NULL alpha). Never assume 0.05; use `metric_significance_flag`.
- **`relative_confidence_interval` can be degenerate** (e.g. ±600% on a renewal-rate metric while p < alpha). Do not judge significance from the CI string; use the flag.
- **`metric_relative_delta` is rounded to 3 decimals** in Redshift; the Hivemind API has 5. Quote lifts to 0.1pp.
- **`bigger_is_better_flag` is metric configuration and can be wrong** (an opt-out rate configured as bigger-is-better). `stat_read` follows the flag; mention the raw direction when it looks odd.
- **`report_date` is a run label, not the data end date.** The run usually happens late evening MST of `report_date` (or just after midnight) and covers data through about the prior day; `insert_utc_ts` is the run time.
- **Hivemind `get_experiment_analysis` returns the same latest report**; if the API errors, that does not prove the warehouse has no data — run this SQL.

### Path B — Significance history
- **Gaps are real**: some Hivemind runs never land in Redshift and some days were never run at all. On validation, Redshift was missing 0–5 run-days per experiment versus the API, and a missing run hid a real flicker (significant → not significant → significant). Report counts "on the reports available" and do not claim "never flickered" without that caveat.
- **Report the run time, not just the label.** `report_date` is a run label; the run usually happens late evening MST that day (or just after midnight) and covers data through about the prior day. `insert_utc_ts` is the run time (UTC). Rollout timestamps are a mix of midnight-only dates and full timestamps, so compare run time to `rollout_mst_ts`, not dates.
- **Method switches** (`sequential` → `fixed`, typically on the final report) break the series. Never compute flips across methods; the anchor's method may have only one report.
- **Sequential p-values** are always-valid and stay high until the boundary is crossed; a `sequential` p of ~0.9 at the end means no conclusion, and early "significant" sequential reports that later lose significance are a genuine flicker.
- **Decision metric can change mid-life** (e.g. `new_purchase_conversion` for the first reports, then `new_purchase_gcr_amt`). The history is per metric; the current decision list is `decision_metric_id_list` in the snapshot.
- **Post-rollout reruns** appear after the anchor with expected split 100/0. They are flagged `after anchor`; never use one as the "before rollout" answer.
- **`rollout_mst_ts` earlier than the first bucket** (bad data, rare) → anchor falls back to real end; `anchor_source` shows which.
- **Bayesian experiments**: p-value logic does not apply; the detail query still lists `metric_pbest` per report — describe the pbest trajectory instead of significance flips.
- **Multi-arm experiments**: history and flips are per arm.
- Never use `ended_at_mst_ts` or `end_mst_date` as the end (R2).

### Path C — Readout
- The same metric can appear as a guardrail and as an informative metric with the same lift but a different p-value (different test setup per category). Report each row in its own category and do not reconcile or re-label them.
- When summarising the scorecard in extra queries, Redshift rejects `LISTAGG` alongside other `DISTINCT` aggregates — use counts per category instead.
- **Dates**: `ended_at_mst_ts` is backfill-corrupted (it can be days after the real end, or set for live experiments) and `end_mst_date` is mostly NULL. Use bucket timestamps. Configured `start_mst_date` can differ from the real first bucket — show both.
- **Owners** is a comma-separated username list; the API may show the same list as an array. Don't truncate to the first owner.
- **Hypothesis text** may contain trailing newlines or be blank; quote as entered, don't invent one. The `hypothesis_*_text` columns hold the **full text** (they match Hivemind `get_experiment_by_id` exactly) — quote all of it; do not shorten it or label it truncated. If you summarise for length, say you summarised and give the full text below.
- **Latest report may be a post-rollout rerun** (expected split 100/0). Scorecard stats remain on the original bucketed users, but health/SRM on that report is spurious.
- **Multi-arm**: Step 2 returns one row per arm per metric — 4 arms × 59 metrics is 236 rows. Summarise by category and call out significant arm-rows.
- **Untested metrics**: `metric_pvalue` NULL with a warning like "Analysis skipped: cohorts only have one value" — list as not tested, not as "not significant".
- **`bigger_is_better_flag`** is configuration and can be counter-intuitive (an opt-out rate configured bigger-is-better). Report direction and the flag as configured; mention the oddity.
- **Alpha per experiment** (control rows have NULL alpha). `metric_relative_delta` is rounded to 3 decimals. `relative_confidence_interval` can be degenerate — rely on `metric_significance_flag`.
- **Metadata metric lists** (`decision_metric_id_list`, `guardrail_metric_id_list`, `informative_metrics_list`) are comma-separated bare ids, can repeat, and `informative_metrics_list` is often NULL even when informative rows exist — the scorecard rows are the source of truth for what was reported.
- **Bayesian experiments**: lift/p-value/alpha are NULL; use `metric_pbest` and `credible_interval`.
- **`status_enum`** is fine for a single recent experiment but NULL for most pre-2026 rows; `released_flag` does not mean the treatment shipped.
