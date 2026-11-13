---
type: table
entry_id: hivemind_mart.analysis_result_cln
table: hivemind_mart.analysis_result_cln
synonyms: [analysis_result_cln, hivemind scorecard, experiment results, experiment scorecard table, hivemind analysis results]
status: DETERMINISTIC
lifecycle: active
intents: [schema_lookup, column_lookup, partition_guidance]
grain: one row per experiment_id per report_date per metric_id per cohort_id per segment combination (overall rows have every segment column = 'ALL')
primary_key: [experiment_id, report_date, metric_id, cohort_id, new_existing_customer_name, market_site, device_name, ip_country_name, customer_tenure_day_segment, domain_ownership_count, customer_type, market_sub_region]
partition_column: report_date
partition_filter_idiom: filter experiment_id (or a tight IN-list) and/or a single report_date string 'YYYY-MM-DD'; an unscoped scan across report dates times out
catalog_source: redshift_svv
column_count: 31
physical_column_count: 49
alation_table_id: not_in_alation
owner_team: not_in_alation
owner_contact: not_in_alation
schema_fetched: 2026-10-22
last_updated: 2026-10-22
verified_on: 2026-10-22
---

# `hivemind_mart.analysis_result_cln`

> One row per experiment, report date, metric, cohort and segment: the Hivemind scorecard numbers (cohort value, lift, p-value, significance, CI) that analysts use to answer "did it win?", "is it significant?" and "is there SRM?".

## Dimensions

| Column | Type | Description | Synonyms | Is Enum | Sample Values |
|--------|------|-------------|----------|---------|---------------|
| `experiment_id` | STRING | Experiment key; joins `experiment_metadata_snap.experiment_id`. Always filter on it. | exp id, experiment key | no | `cart_123reg_empty_new_ux` |
| `report_date` | STRING | Scorecard RUN label as 'YYYY-MM-DD' string (runs ~20:30–00:30 MST, data through ~the prior day). History is kept (one method per experiment per date) but some runs are missing (0–14% in samples). The newest date is often a partial load. For "before rollout" use run time (`insert_utc_ts` − 7h ≤ `rollout_mst_ts`), since many rollout timestamps are midnight-only. | scorecard date, run date | no | `2026-10-19` |
| `metric_id` | STRING | Metric definition id WITH a 32-hex hash suffix (e.g. `new_purchase_gcr_amt_<32hex>`); ~20% have no suffix (e.g. SRM rows use `cohort_percentage`). The hash-stripped id can differ from the config id for the same metric (e.g. scorecard `gcr_new_purchases` vs config `new_purchase_gcr_amt`) — match on `label` + category when ids disagree and the suffix can change between runs. Strip with `REGEXP_REPLACE(metric_id,'_[0-9a-f]{32}$','')` before comparing to metadata lists. | metric, definition id | no | `new_purchase_conversion_<hash>` |
| `label` | STRING | Human label of the metric. SRM rows use label `'Expected cohort percentage'`. | metric name | no | `Expected cohort percentage` |
| `metric_category` | STRING | Role of the metric in this experiment. `OverallHealth` = sample-ratio (SRM) rows: `metric_id = 'cohort_percentage'`, label `'Expected cohort percentage'`. No `guardrail` rows + empty metadata `guardrail_metric_id_list` = no guardrails configured (say so; never "no harm"). | category, metric role | yes | `decision`, `guardrail`, `informative`, `OverallHealth` |
| `cohort_id` | STRING | Cohort (arm). One row per cohort including control. | arm, variant, cohort | no | `on`, `off`, `treatment`, `control` |
| `control_cohort` | STRING | The control cohort id for the experiment on that row. Treatment rows = `cohort_id <> control_cohort`. | baseline, control | no | `off` |
| `analysis_method_used` | STRING | Statistical method on that report. Can switch mid-life (sequential ↔ fixed); compare like with like. | method | yes | `fixed`, `sequential` |
| `analysis_type` | STRING | Frequentist or Bayesian. Bayesian rows use `metric_pbest` / `credible_interval`; p-value logic does not apply. | stats type | yes | `frequentist`, `bayesian` |
| `decision_metric_flag` | BOOLEAN | TRUE for decision metrics. | is decision metric | no | `true` |
| `bigger_is_better_flag` | BOOLEAN | Direction of improvement for the metric. | direction | no | `true` |
| `metric_significance_flag` | BOOLEAN | Hivemind's significance call at the experiment's own alpha. Agrees with p < alpha on frequentist rows. Use this, not a hand-picked 0.05. | significant, stat sig | no | `true` |
| `metric_warning` | STRING | Free-text warnings (e.g. sample ratio mismatch, no new users, "Analysis skipped" — a NULL p-value with this warning means no test was run, not "not significant"). | warning | no | — |
| `insert_utc_ts` | TIMESTAMP | When the report row was written; matches Hivemind report time within minutes. Use for "last report before rollout". | run time | no | — |
| `statistical_test_name` | STRING | Test used. | test | no | — |
| `fixed_design_flag` | BOOLEAN | Fixed-horizon design flag. | fixed design | no | — |
| `scorecard_template_name` | STRING | Scorecard template used. | scorecard | no | — |
| `new_existing_customer_name` | STRING | Segment. `'ALL'` = overall. | customer type segment | yes | `ALL`, `New Prospect`, `Existing Customer` |
| `market_site` | STRING | Segment. `'ALL'` = overall. | market | no | `ALL` |
| `device_name` | STRING | Segment. `'ALL'` = overall (the API calls this device_breakpoint_name). | device | yes | `ALL` |
| `ip_country_name` | STRING | Segment. `'ALL'` = overall. | country | no | `ALL` |
| `customer_tenure_day_segment` | STRING | Segment. `'ALL'` = overall. | tenure | no | `ALL` |
| `domain_ownership_count` | STRING | Segment. `'ALL'` = overall. | domain count | no | `ALL` |
| `customer_type` | STRING | Segment. `'ALL'` = overall. | — | no | `ALL` |
| `market_sub_region` | STRING | Segment. `'ALL'` = overall (the API calls this market_sub_regions). | sub region | no | `ALL` |

## Facts

| Column | Type | Description | Synonyms | Aggregation | Unit |
|--------|------|-------------|----------|-------------|------|
| `metric_value` | FLOAT | Cohort value of the metric (per-user value, rate, or for SRM rows the cohort user count). Winsorized / variance-reduced by Hivemind. | value, cohort value | none (read per row) | metric unit |
| `metric_delta` | FLOAT | Treatment − control (absolute). NULL on the control row. Rounded to 3 decimals. | absolute lift, diff | none | metric unit |
| `metric_relative_delta` | FLOAT | Relative lift vs control. NULL on the control row. Rounded to 3 decimals (0.1pp) — for a more precise lift compute treatment `metric_value` / control `metric_value` − 1 on the same report. | lift, relative lift, % change | none | ratio |
| `metric_pvalue` | FLOAT | p-value (frequentist). In sequential runs 1.0 means the boundary was not crossed. | p value | none | probability |
| `alpha` | FLOAT | Significance level used for this experiment (0.1 or 0.2 typically). NULL on baseline rows. Applied per treatment arm vs control. SRM rows use their own alpha (0.01). | alpha, significance level | none | probability |
| `confidence_interval` | STRING | Absolute CI as text. Sometimes NULL. | CI | none | text |
| `relative_confidence_interval` | STRING | Relative CI as text, two-sided at (1 − alpha) — an 80% CI when alpha = 0.2. On sequential runs it is a fixed-horizon interval and can include 0 while `metric_significance_flag` is TRUE (always-valid p-value). Trust the flag. | relative CI | none | text |
| `metric_pbest` | FLOAT | Probability to be best (Bayesian rows only). | pbest | none | probability |
| `credible_interval` | STRING | Bayesian credible interval. | credible interval | none | text |
| `expected_proportion` | FLOAT | SRM rows: configured split share at report time. 0 or 1 means a post-rollout rerun — ignore those reports for SRM. | expected split | none | ratio |
| `sample_proportion` | FLOAT | SRM rows: observed share. | observed split | none | ratio |

## Mandatory Filters

| Filter | Expression | Why | Source |
|--------|------------|-----|--------|
| Scope | `experiment_id = '<id>'` (or IN-list) | unscoped scans over report dates time out (>2 min) | domain_doc |
| Overall rows | all 8 segment columns `= 'ALL'` | otherwise rows multiply across segment combinations | use_case:experimentation-hm-did-it-work |
| One report | `report_date = <chosen date>` | history is kept; mixing dates double-counts | use_case:experimentation-hm-did-it-work |

## Omitted Columns

| Column Group | Reason |
|--------------|--------|
| variance internals (`metric_variance`, `metric_adjusted_variance`, `*_denominator`, `metric_variance_reduction_percentage*`, `missing_pre_experiment_data_*`) | not needed for any committed use-case; present on the physical table |
| `metric_description` | long text, not needed for answers |
| `start_mst_ts`, `end_mst_ts` | analysis window stamps; use metadata bucket timestamps for real dates (`insert_utc_ts` is kept — it is the reliable run time) |

## Related Tables

| Table | Join Keys | Cardinality | Join Type |
|-------|-----------|-------------|-----------|
| `hivemind_mart.experiment_metadata_snap` | `experiment_id` = `experiment_id` | many_to_one | INNER |

## Never Join

| Table | Why Not | What To Do Instead |
|-------|---------|--------------------|
| `hivemind_mart.experiment_order` | different grain (orders vs metric rows); joining fans out | aggregate each side separately, then compare per cohort |

## Traps

- Segment rows stopped landing in June 2026 — since then every row is the overall `'ALL'` row. Do not promise segment breakdowns from this table.
- The latest verdict must come from the latest report_date that actually HAS `metric_category='decision'` rows; some later runs carry only informative metrics.
- **Identical p-values across arms or across decision metrics are expected** — Hivemind applies a multiplicity adjustment; report each row as stored, do not treat it as an error.
- **Partial newest load test**: a report_date is complete when it has the same number of `metric_category = 'decision'` rows as the previous report_date. If the newest date has fewer, use the previous one and say why.
- **Run time**: `report_date` is a label; a run executed just after midnight carries the previous day's label. Use `DATEADD(hour, -7, MAX(insert_utc_ts))` per report_date as the MST run time, and find missing runs as gaps of more than 36 h between consecutive run times, not as skipped labels.
- Post-rollout reruns recompute against the new 100/0 split and produce false SRM. For health checks use the last report on or before the real end and skip whole REPORT DATES where any cohort has `expected_proportion` 0 or 1 — filter per report date, not per row: with 3+ cohorts the SRM p-value sits on a `cohort_id = 'Overall'` row whose `expected_proportion` is NULL, and a row filter silently drops it. With 2 cohorts the p-value is on the treatment row. The SRM test uses its own alpha (0.01) and is a sequential likelihood-ratio test.
- The report dated on the real end covers data only to about the prior day, so its cohort counts sit below `user_allocation_history`; use UAH for counts and the report for the SRM verdict.
- `metric_id` hash suffix — never exact-match against metadata lists; strip the suffix.
- Exact duplicate rows exist for some grains — use `SELECT DISTINCT`.
- Values are Hivemind-adjusted (winsorization / variance reduction); raw SQL from `experiment_order` will differ by a few percent.

## ⚠️ **OWNER INPUT NEEDED**

These definitions are **not clear** and must be filled in by a Hivemind data owner: `missing_pre_experiment_data_* (4 columns)` (OQ-AR-01); `start_mst_ts / end_mst_ts` (OQ-AR-02); `market_sub_region codes 'DEM', 'ROW DM', 'ROW EM', 'OTHERS'` (OQ-AR-03); `statistical_test_name = 'Mann-Whitney U'` (OQ-AR-04); `relative_confidence_interval tokens '(None, None)', '(-inf, …)', '(-1.0, -1.0)'` (OQ-AR-05); `metric_delta on SRM (OverallHealth) treatment rows` (OQ-AR-06); `metric_significance_flag on Bayesian rows` (OQ-AR-07); `control-cohort labels (wacmnth, waddch, smsidm, aabf2p55h, binshown, trust-base, A, …) and scorecard template contents` (OQ-AR-08). Full questions are in `data-dictionary/hivemind_mart.analysis_result_cln.dictionary.md`. If a question depends on one of them, answer "definition not confirmed by the Hivemind owners" — do not guess.

## Provenance & Gaps

| Field | Source | Status |
|-------|--------|--------|
| columns | redshift_svv (describe_table 2026-10-22) | populated |
| column descriptions | Alation (0 of 49 described) | not_in_alation |
| mandatory filters | use-case SQL + live timing tests | populated |
| enums / sample values | execute_query 2026-10-22 | populated |
| owner | — | not_in_alation |
