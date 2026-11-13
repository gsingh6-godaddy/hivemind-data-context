# Data Dictionary: `hivemind_mart.analysis_result_cln`

> ⚠️ **OWNER INPUT NEEDED** (OQ-AR-08): table-level definition not clear (control-cohort label conventions and scorecard template contents). Hivemind owner to fill in. See the owner table at the end of this file.

**Purpose.** This is the Redshift copy of the Hivemind experiment **scorecard**: what `get_experiment_analysis` returns, one Redshift row per metric and cohort, for every daily analysis run. It holds each cohort's metric value, and for each treatment arm the absolute and relative lift vs control, p-value, alpha, significance flag, confidence intervals, test name and variance / CUPED internals. Bayesian runs carry `metric_pbest` and `credible_interval` instead. The sample-ratio (SRM) health check is stored as `metric_category = 'OverallHealth'` rows. Analysts use it to answer "did it win?", "is it significant?", "did it flicker?", "is there SRM?" and "what is the $/user delta?".

**Grain.** One row per `experiment_id` × `report_date` × `metric_id` × `cohort_id` × segment combination (8 segment columns). Since mid-June 2026 every row is the overall row (all 8 segment columns = `'ALL'`). On SRM rows for experiments with 3+ cohorts there is an extra `cohort_id = 'Overall'` row that carries the SRM p-value.

**Primary key (logical).** `experiment_id, report_date, metric_id, cohort_id, new_existing_customer_name, market_site, device_name, ip_country_name, customer_tenure_day_segment, domain_ownership_count, customer_type, market_sub_region`. It is not enforced. Exact duplicate rows exist (for example, `agi_plg_email_inline_upgrade_erp`, report 2026-10-07, informative `GCR` rows appear twice). The same metric can also appear under two `metric_category` values on one report. Always use `SELECT DISTINCT`.

**Partition / sort.** `report_date` (STRING `'YYYY-MM-DD'`) is the partition column.

**Mandatory filters.**

| Filter | Expression | Why |
|---|---|---|
| Scope | `experiment_id = '<id>'` (or a tight IN-list) and/or one `report_date` | An unscoped scan across report dates times out (>2 min). |
| Overall rows | all 8 segment columns `= 'ALL'` | Before June 2026, rows multiply across segment values. |
| One report | `report_date = <chosen date>` | History is kept, so mixing dates double-counts. |
| De-dup | `SELECT DISTINCT` | Exact duplicate rows exist. |

**Row counts by scope.**

| Scope | Rows | Experiments | Source |
|---|---|---|---|
| Profile scope: report_date >= 2026-07-01 for 150 hash-sampled experiments (all types that bucketed since 2026-07-01) | 259,320 | 150 | profile |
| One full report, report_date = '2026-11-10' (all experiments) | 62,567 (62,387 distinct experiment\|metric_id\|cohort_id\|metric_category) | 897 | SQL 2026-11-13 |
| report_date = '2026-10-01' | 69,321 | — | SQL |
| report_date = '2026-08-22' | 63,158 | — | SQL |
| report_date = '2026-06-20' (first sampled date with no segment rows) | 56,132 | — | SQL |
| report_date = '2026-06-01' (segment era) | 374,492 (321,396 segment rows) | — | SQL |
| report_date = '2026-05-01' (segment era) | 300,946 (256,131 segment rows) | — | SQL |

Segment (non-`'ALL'`) rows were last seen on report_date 2026-06-11 (SQL over 2026-06-02 to 2026-06-19).

**Composition of one report (report_date = '2026-11-10', SQL):**

| analysis_type | analysis_method_used | metric_category | rows | experiments |
|---|---|---|---|---|
| bayesian | fixed | OverallHealth | 40 | 17 |
| bayesian | fixed | decision | 47 | 17 |
| bayesian | fixed | informative | 2,057 | 17 |
| frequentist | fixed | OverallHealth | 1,285 | 605 |
| frequentist | fixed | decision | 702 | 239 |
| frequentist | fixed | guardrail | 523 | 133 |
| frequentist | fixed | informative | 37,025 | 682 |
| frequentist | sequential | OverallHealth | 390 | 183 |
| frequentist | sequential | decision | 407 | 188 |
| frequentist | sequential | guardrail | 261 | 61 |
| frequentist | sequential | informative | 19,830 | 198 |

The 150-experiment profile sample contains **no Bayesian experiments**. That is why `metric_pbest` and `credible_interval` profile as 100% NULL, and `analysis_type` profiles as only `frequentist`. The full table does contain Bayesian rows.

---

## Column index

| # | Column | Type | Null % (scope) | Distinct (scope) | One-line meaning | Confidence |
|---|---|---|---|---|---|---|
| 1 | `metric_id` | string | 0.0 | 2,295 | Metric definition id, usually with a 32-hex hash suffix; `cohort_percentage` = SRM | Verified |
| 2 | `new_existing_customer_name` | string | 0.0 | 1 | Segment: New Prospect / Existing Customer; `'ALL'` = overall | Verified |
| 3 | `market_site` | string | 0.0 | 1 | Segment: market site; `'ALL'` = overall | Verified |
| 4 | `device_name` | string | 0.0 | 1 | Segment: device breakpoint; `'ALL'` = overall | Verified |
| 5 | `ip_country_name` | string | 0.0 | 1 | Segment: IP country; `'ALL'` = overall | Verified |
| 6 | `customer_tenure_day_segment` | string | 0.0 | 1 | Segment: customer tenure bucket; `'ALL'` = overall | Verified |
| 7 | `domain_ownership_count` | string | 0.0 | 1 | Segment: domain-count bucket; `'ALL'` = overall | Verified |
| 8 | `customer_type` | string | 0.0 | 1 | Segment: customer type; `'ALL'` = overall | Verified |
| 9 | `market_sub_region` | string | 0.0 | 1 | Segment: market sub-region; `'ALL'` = overall | Verified |
| 10 | `cohort_id` | string | 0.0 | 82 | Cohort (arm) the row describes; `'Overall'` on some SRM rows | Verified |
| 11 | `metric_description` | string | 0.0 | 963 | Metric description text (API `description`) | Verified |
| 12 | `label` | string | 0.0 | 129 | Metric display label (API `label`) | Verified |
| 13 | `bigger_is_better_flag` | boolean | 0.0 | 2 | Direction of improvement (API `biggerIsBetter`) | Verified |
| 14 | `metric_value` | real | 0.94 | 158,330 | Cohort metric value; on SRM rows, the cohort user count | Verified |
| 15 | `expected_proportion` | real | 98.12 | 72 | SRM rows: configured split share | Verified |
| 16 | `sample_proportion` | real | 98.12 | 4,280 | SRM rows: observed split share | Verified |
| 17 | `metric_delta` | real | 54.61 | 9,520 | Treatment − control absolute difference, rounded to 3 dp | Verified |
| 18 | `metric_variance` | real | 50.31 | 92,339 | Raw variance (numerator variance for ratio metrics) | Verified |
| 19 | `metric_adjusted_variance` | real | 55.01 | 86,895 | CUPED-adjusted variance (numerator for ratio metrics) | Verified |
| 20 | `metric_variance_reduction_percentage` | real | 55.01 | 71,451 | Variance-reduction share 0–1 (numerator for ratio metrics) | Verified |
| 21 | `missing_pre_experiment_data_count` | integer | 0.0 | 3,762 | Users in the cohort with no pre-experiment data | Inferred |
| 22 | `missing_pre_experiment_data_percentage` | real | 54.74 | 10,287 | Column 21 ÷ cohort sample count | Inferred |
| 23 | `metric_variance_denominator` | real | 92.83 | 14,367 | Ratio metrics: denominator variance | Verified |
| 24 | `metric_adjusted_variance_denominator` | real | 93.55 | 13,925 | Ratio metrics: adjusted denominator variance | Verified |
| 25 | `metric_variance_reduction_percentage_denominator` | real | 93.55 | 13,385 | Ratio metrics: denominator variance reduction 0–1 | Verified |
| 26 | `missing_pre_experiment_data_count_denominator` | integer | 0.0 | 2,144 | Ratio metrics: missing pre-period count for the denominator | Inferred |
| 27 | `missing_pre_experiment_data_percentage_denominator` | real | 93.36 | 6,176 | Ratio metrics: missing pre-period share for the denominator | Inferred |
| 28 | `metric_significance_flag` | boolean | 0.0 | 2 | Hivemind's significance call (API `isSignificant`) | Verified |
| 29 | `alpha` | real | 55.66 | 3 | Significance level used for this comparison | Verified |
| 30 | `metric_pvalue` | real | 55.66 | 26,784 | p-value (API `pValue`) | Verified |
| 31 | `confidence_interval` | string | 56.47 | 45,109 | Absolute-difference CI as text (API `absoluteCI`) | Verified |
| 32 | `metric_relative_delta` | real | 56.48 | 4,556 | Relative lift vs control, rounded to 3 dp (API `relativeDifference`) | Verified |
| 33 | `relative_confidence_interval` | string | 56.47 | 82,075 | Relative-lift CI as text (API `relativeCI`) | Verified |
| 34 | `metric_pbest` | real | 100.0 | 0 | Bayesian probability to be best (API `pBest`) | Verified |
| 35 | `credible_interval` | string | 100.0 | 0 | Bayesian credible interval (API `credibleInterval`) | Verified |
| 36 | `statistical_test_name` | string | 55.66 | 9 | Statistical test used for the comparison | Verified |
| 37 | `metric_warning` | string | 0.0 (73.86 empty) | 302 | Hivemind warning text (API `warnings`) | Verified |
| 38 | `decision_metric_flag` | boolean | 0.0 | 2 | TRUE iff `metric_category = 'decision'` | Verified |
| 39 | `metric_category` | string | 0.0 | 4 | Metric role: decision / guardrail / informative / OverallHealth | Verified |
| 40 | `control_cohort` | string | 0.0 | 33 | The experiment's control / baseline cohort id (API `baseline`) | Verified |
| 41 | `scorecard_template_name` | string | 0.0 | 49 | Scorecard template used (API `scorecardTemplateName`) | Verified |
| 42 | `analysis_type` | string | 0.0 | 1 | frequentist / bayesian (API `analysisType`) | Verified |
| 43 | `fixed_design_flag` | boolean | 0.0 | 1 | API `isFixedDesign` (TRUE everywhere observed) | Verified |
| 44 | `start_mst_ts` | timestamp | 1.38 | 489 | Analysis window start = first bucketing (API `healthMetric.startTime`) | Verified |
| 45 | `end_mst_ts` | timestamp | 1.38 | 1,907 | Analysis window end = data through (API `healthMetric.endTime`) | Verified |
| 46 | `insert_utc_ts` | timestamp | 0.0 | 3,105 | When the row was written (UTC); the run time | Verified |
| 47 | `analysis_method_used` | string | 0.0 | 2 | fixed / sequential (API `analysisMethodUsed`) | Verified |
| 48 | `experiment_id` | string | 0.0 | 150 | Experiment key (case-sensitive) | Verified |
| 49 | `report_date` | string | 0.0 | 134 | Analysis run label 'YYYY-MM-DD' (partition) | Verified |

---

## Cross-table value map

| This table | Other table.column | Relationship / trap |
|---|---|---|
| `experiment_id` | `experiment_metadata_snap.experiment_id`, `experiment_order.experiment_id`, `user_allocation_history.experiment_id` | Exact, case-sensitive key. Many-to-one to metadata. **Never join to `experiment_order` row-wise**, because it fans out. Aggregate each side, then compare. |
| `cohort_id` | `experiment_order.cohort_id`, `user_allocation_history.cohort_id`; `experiment_metadata_snap.cohort_name_list` (comma list) | Same cohort ids (e.g. `control_variant`/`b_variant`, `default`/`variant`, `control`/`testa`..`testd`, `on`/`off` all appear in both). `'Overall'` exists only here (SRM rows). |
| `control_cohort` | `experiment_metadata_snap.baseline_cohort_id` | Same concept. Verified equal for `agi_plg_email_inline_upgrade_erp` (`default`) and `MKT_EM_177_Failed-Billing-Manual-D28_CM-24927` (`control`). Prefer this table's value on the latest report. |
| `scorecard_template_name` | `experiment_metadata_snap.scorecard_template_name`, `experiment_order.scorecard_template_name`, `user_allocation_history.scorecard_template_name` | Same template ids (e.g. `cm-renewals-audience-recording`, `hp_core_metrics`, `CDE-PrePost-Default-Scorecard`). |
| `alpha` | `experiment_metadata_snap.alpha_value` | Same values (0.1, 0.2, 0.01 …). SRM rows always use 0.01 regardless of `alpha_value`. |
| `analysis_type` | `experiment_metadata_snap.analysis_type` | `frequentist` / `bayesian`. Metadata also has 3 one-off junk values (`revenue_impact`, `User Interaction`, `conversion_rate_analysis`) that do not appear here. |
| `analysis_method_used` | `experiment_metadata_snap.analysis_method_enum` | `fixed` / `sequential`. This table is per report and can differ from the metadata value (it can switch over the experiment's life). |
| `fixed_design_flag` | `experiment_metadata_snap.fixed_design_flag` | Same API field `isFixedDesign`. |
| `REGEXP_REPLACE(metric_id,'_[0-9a-f]{32}$','')` | `experiment_metadata_snap.decision_metric_id_list`, `guardrail_metric_id_list`, `sig_lift_decision_metric_id_list`, `sig_loss_decision_metric_id_list`, `sig_guardrail_metric_id_list`, `sig_lift/loss_informative_metric_id_list`, `annualized_metric_definition_id` | Match only after stripping the hash. The stripped id can still differ from the config id for the same metric (e.g. `gcr_new_purchases` vs `new_purchase_gcr_amt`). When ids disagree, fall back to `label` + `metric_category`. |
| `start_mst_ts` | `experiment_metadata_snap.first_user_bucket_mst_ts` | Equal on both verified experiments. |
| `end_mst_ts` | `experiment_metadata_snap.last_user_bucket_mst_ts` | Equals the last bucketing time *as of that report*. On the final report it can equal metadata's value (MKT_EM_177) or sit one run earlier (agi, report 2026-10-06 → 2026-10-05 23:59:52 vs metadata 2026-10-06 23:59:05). |
| `report_date` / `insert_utc_ts` | `experiment_metadata_snap.rollout_mst_ts` | `rollout_mst_ts` is the API `rolloutDate` stored as a **UTC** instant despite its name. Compare `DATEADD(hour,-7,insert_utc_ts)` to `DATEADD(hour,-7,rollout_mst_ts)`. |
| `metric_value` on `metric_id='cohort_percentage'` | `user_allocation_history` cohort counts | The SRM row's `metric_value` is the cohort user count as of the report's data cut. It sits below UAH counts on the final report, so use UAH for counts. |
| Segment values (pre-June 2026 only) | `experiment_order` / `user_allocation_history` `new_existing_customer_name`, `device_breakpoint_name`, `customer_tenure_day_segment`, `domain_ownership_count`, `ip_country_name`, `market_site`, `customer_type` | Same vocabularies (`New Prospect`/`Existing Customer`; `Desktop`/`Mobile`/`Other`/`Multiple`; `0-1 day`; `0`; country names). Here they are dead since June 2026 (`'ALL'` only). `market_sub_region` values (`OTHERS`, `DEM`, `ROW DM`, `ROW EM`, `India`) have no counterpart column in the other three tables. |

---

## Columns

### `metric_id` - string
- **Meaning:** Hivemind metric definition id for the row. The API equivalent is `businessMetrics[].definitionId`, which the API returns *without* the hash. Most ids carry a `_<32 hex>` suffix (e.g. `upgrade_o365_purchase_gcr_amt_40b3fdc9898c30e0e0108289bf2552d2`). Some do not (e.g. `email_resource_renewal_yield`, `renewal_rate_unit`). SRM rows use `cohort_percentage`.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 2,295 distinct; length 3–95.
- **Values:** 2,295 distinct (high cardinality). Top values:

  | Value | Count |
  |---|---|
  | cohort_percentage | 7,248 |
  | sales_qualified_dify_lead_count_60d738347627862f3a3b5d9d95598780 | 4,738 |
  | new_ox_purchase_gcr_amt_49968a743d3a9a6d411786bd07107e4d | 3,995 |
  | new_domain_purchase_gcr_amt_d2daeb2a4a80eb1a128dd4e7833cf014 | 3,995 |
  | new_web_gcr_e773d16c64ca40de8d15898aa7ba8abe | 3,995 |
  | existing_customer_new_purchase_gcr_65c5c347ed28915e134c37f972f38e73 | 3,995 |
  | ssl_new_gcr_7ee044316f19f2583110d79f65552bf6 | 3,995 |
  | gcr_amt_corporate_0641e7954b52c81ab3e5f4095a502b63 | 3,995 |
  | new_email_purchase_gcr_amt_2e91499bebcc369be8439440e33c6096 | 3,995 |
  | New_customer_new_purchase_gcr_0e0a89193e555243e5f2eceba44e0f13 | 3,995 |
  | New_MWP_Purchase_GCR_Amount_confirmed_34ca084859b2459f1d45f85cb2a3e36e | 3,995 |
  | new_purchase_gcr_amt_2beed7fb16166fdd40a2f7660e120a2a | 3,995 |
  | new_wam_gcr_3bf2a5aa4ac15f92d551a9a527e37683 | 3,995 |
  | domain_setup_complete_rate_corporate_65ca6c25144c520c53e3b65782cb7c82 | 3,918 |
  | Commerce_RAC_lead_generated_cccf97f575d3825fe3e84dc7c73f2b91 | 3,535 |
- **How to use / Traps:** Ids are case-sensitive and mixed case (`New_customer_...`, `Commerce_RAC_...`). Strip the hash with `REGEXP_REPLACE(metric_id,'_[0-9a-f]{32}$','')` before comparing to metadata id lists. Never exact-match the raw id. The suffix can change between runs, so group history by the stripped id. `metric_id = 'cohort_percentage'` together with `metric_category = 'OverallHealth'` marks the SRM rows.
- **Confidence:** Verified (profile, verified doc, SQL; MCP `definitionId` = stripped id for `upgrade_o365_purchase_gcr_amt`).

### `new_existing_customer_name` - string
- **Meaning:** Segment dimension (new vs existing customer). `'ALL'` = the overall, unsegmented row.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 1 distinct; length 3–3.
- **Values (complete in scope):**

  | Value | Count | Meaning |
  |---|---|---|
  | ALL | 259,320 | Overall row (no segment) |

  Before mid-June 2026 the column also held `New Prospect` and `Existing Customer` (SQL, report_date 2026-06-01: 1,381 / 1,369 rows).
- **How to use / Traps:** Always filter `= 'ALL'` with the other 7 segment columns. Segment rows stopped after report_date 2026-06-11. Do not promise segment breakdowns for later reports. Use the segmentation tools or `experiment_order` instead.
- **Confidence:** Verified (profile + SQL).

### `market_site` - string
- **Meaning:** Segment dimension (market site, e.g. `en-us` in sibling tables). `'ALL'` = overall.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 1 distinct; length 3–3.
- **Values (complete in scope):**

  | Value | Count | Meaning |
  |---|---|---|
  | ALL | 259,320 | Overall row |
- **How to use / Traps:** Filter `= 'ALL'`. The pre-June 2026 segment values were not enumerated for this column in the sources. Sibling tables use locale codes (`en-us`, `en-in`, `Multiple`, …).
- **Confidence:** Verified (profile).

### `device_name` - string
- **Meaning:** Segment dimension: device breakpoint. The API and sibling tables call it `device_breakpoint_name`. `'ALL'` = overall.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 1 distinct; length 3–3.
- **Values (complete in scope):**

  | Value | Count | Meaning |
  |---|---|---|
  | ALL | 259,320 | Overall row |

  Pre-June 2026 values (SQL, 2026-06-01): `Desktop` 1,385, `Mobile` 1,368, `Other` 1,326, `Multiple` 1,317.
- **How to use / Traps:** Filter `= 'ALL'`.
- **Confidence:** Verified (profile + SQL).

### `ip_country_name` - string
- **Meaning:** Segment dimension: IP-geolocated country name. `'ALL'` = overall.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 1 distinct; length 3–3.
- **Values (complete in scope):**

  | Value | Count | Meaning |
  |---|---|---|
  | ALL | 259,320 | Overall row |

  Pre-June 2026 (SQL, 2026-06-01): full country names, e.g. `United States` 1,381, `India` 1,373, `Canada` 1,369, plus `Multiple` 1,331.
- **How to use / Traps:** Filter `= 'ALL'`.
- **Confidence:** Verified (profile + SQL).

### `customer_tenure_day_segment` - string
- **Meaning:** Segment dimension: customer tenure bucket. `'ALL'` = overall.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 1 distinct; length 3–3.
- **Values (complete in scope):**

  | Value | Count | Meaning |
  |---|---|---|
  | ALL | 259,320 | Overall row |

  Pre-June 2026 (SQL, 2026-06-01): `0-1 day` 2 rows only. Sibling tables use `0-1 day`, `8-30 day`, `31-365 day`, `365+ day`.
- **How to use / Traps:** Filter `= 'ALL'`.
- **Confidence:** Verified (profile + SQL).

### `domain_ownership_count` - string
- **Meaning:** Segment dimension: domain-ownership bucket, stored as a string. `'ALL'` = overall.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 1 distinct; length 3–3.
- **Values (complete in scope):**

  | Value | Count | Meaning |
  |---|---|---|
  | ALL | 259,320 | Overall row |

  Pre-June 2026 (SQL, 2026-06-01): `0` 2 rows. Sibling tables use `0`, `1`, `2-5`, `6-10`, `10+`.
- **How to use / Traps:** Filter `= 'ALL'`. It is a string bucket, not a number.
- **Confidence:** Verified (profile + SQL).

### `customer_type` - string
- **Meaning:** Segment dimension: customer type. Sibling tables have `US Independent`, `International Independent`, `Investor`, `Partner`, `Not Evaluated`. `'ALL'` = overall.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 1 distinct; length 3–3.
- **Values (complete in scope):**

  | Value | Count | Meaning |
  |---|---|---|
  | ALL | 259,320 | Overall row |

  The column was `'ALL'` on every row even on 2026-06-01 (374,492 rows), during the segment era (SQL).
- **How to use / Traps:** Filter `= 'ALL'`. No segment values were observed in any sampled report.
- **Confidence:** Verified (profile + SQL).

### `market_sub_region` - string

> ⚠️ **OWNER INPUT NEEDED** (OQ-AR-03): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Segment dimension: market sub-region. The API calls it `market_sub_regions`. `'ALL'` = overall.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 1 distinct; length 3–3.
- **Values (complete in scope):**

  | Value | Count | Meaning |
  |---|---|---|
  | ALL | 259,320 | Overall row |

  Pre-June 2026 (SQL, 2026-06-01): `OTHERS` 1,375, `DEM` 1,365, `ROW DM` 1,356, `ROW EM` 1,348, `India` 1,346. **⚠️ OWNER INPUT NEEDED (OQ-AR-03):** the meanings of `DEM`, `ROW DM`, `ROW EM` and `OTHERS` are not clear. A Hivemind owner needs to fill them in.
- **How to use / Traps:** Filter `= 'ALL'`.
- **Confidence:** Verified (profile + SQL). Expansions of the region codes are Unknown.

### `cohort_id` - string
- **Meaning:** The cohort (arm) this row describes. The API equivalent is the keys of `summary.metricValue` / `analysis.cohorts[].id`. There is one row per cohort, including control. On SRM rows of experiments with 3+ cohorts there is an extra `cohort_id = 'Overall'` row that carries the omnibus SRM test (API `healthMetric.statisticalAnalysis.cohortComparison = 'Overall'`).
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 82 distinct; length 1–63.
- **Values:** 82 distinct (experiment-specific labels). Top values:

  | Value | Count |
  |---|---|
  | off | 35,098 |
  | on | 35,098 |
  | post | 10,330 |
  | pre | 10,330 |
  | control | 8,475 |
  | b_variant | 7,569 |
  | baseline | 6,899 |
  | control_variant | 6,877 |
  | trad-domainreactive-keepalive-banner-critical-maxsuppress | 5,949 |
  | trad-domainreactive-keepalive-inline-critical-maxsuppress | 5,949 |
  | trad-domainreactive-keepalive-bannerinline-critical-maxsuppress | 5,949 |
  | trad-domainreactive-keepalive-inline-critical | 5,949 |
  | trad-domainreactive-keepalive-banner-critical | 5,949 |
  | trad-domainreactive-keepalive-bannerinline-critical | 5,949 |
  | test | 4,847 |
- **How to use / Traps:** Treatment rows are `cohort_id <> control_cohort` (also exclude `'Overall'`). Control rows have NULL `metric_delta`, `metric_relative_delta`, `metric_pvalue`, `alpha` and `statistical_test_name`. `pre`/`post` are the PRE-POST (Causal Impact) periods, not arms. Ids are case-sensitive (`Control` and `control` both exist across experiments). With 2 cohorts the SRM p-value sits on the treatment row. With 3+ cohorts it sits on the `'Overall'` row, whose `expected_proportion` is NULL (SQL 2026-11-10: 35 `'Overall'` rows, all OverallHealth).
- **Confidence:** Verified (profile, verified doc, SQL, MCP payloads).

### `metric_description` - string
- **Meaning:** Free-text description of the metric definition (API `businessMetrics[].description`, e.g. "GCR from upgrades of o365"). On SRM rows: "configured split percentages, among the experiment population".
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 963 distinct; length 8–274.
- **Values:** 963 distinct (high cardinality). Top values:

  | Value | Count |
  |---|---|
  | configured split percentages, among the experiment population | 7,248 |
  | GCR from new purchases | 5,552 |
  | New MWP Purchase GCR for product_pnl_new_renewal_name = 'New Purchase' AND product_pnl_line_name IN ('WordPress Managed Plans', 'Wordpress Managed Plans') | 5,080 |
  | GCR from SSL New Order | 5,080 |
  | New Purchase GCR from New Customers | 5,080 |
  | GCR from new email purchases | 5,080 |
  | GCR from new OX purchases | 5,080 |
  | GCR from new domain purchases | 5,080 |
  | Corporate Metrics: GCR | 5,080 |
  | Total New Web GCR generated during an exerperiment | 5,072 |
  | websites & marketing gcr only from new purchases since the start of the experiment | 5,012 |
  | New Purchase GCR from Existing Customers | 5,004 |
  | Users who have completed all the required fields in the 'Request a Call' form and either submitted the lead for a call back or scheduled a call. | 4,890 |
  | Commerce Lead generated via submission of FoS RAC form | 4,548 |
  | Corporate Metric - setup complete rate for Domain (set up complete is the same definition as domain activation) | 3,918 |
- **How to use / Traps:** Author-written text with typos (e.g. "exerperiment"). Use it to explain a metric, never as a join or filter key. The same metric_id can carry more than one description over time (counts differ from metric_id counts).
- **Confidence:** Verified (profile + MCP payload).

### `label` - string
- **Meaning:** Human display label of the metric (API `businessMetrics[].label`). It is generic, so many metrics share a label (e.g. every GCR metric = `GCR`). SRM rows use `'Expected cohort percentage'`.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 129 distinct; length 3–99.
- **Values:** 129 distinct. Top values:

  | Value | Count |
  |---|---|
  | GCR | 79,475 |
  | Event completion rate | 51,674 |
  | Conversion rate | 26,068 |
  | Units purchased | 17,603 |
  | Expected cohort percentage | 7,248 |
  | n-day Email Receipts Renewal Rate | 6,653 |
  | n-day Email Units Renewal Rate | 5,194 |
  | DIFY Sales Qualified Leads (SQL) | 4,890 |
  | Event completion rate with property | 4,758 |
  | Setup Complete Rate | 3,978 |
  | Free trial signup | 2,594 |
  | Percentage of users who saw particular page | 2,343 |
  | Units Renewal Rate | 2,211 |
  | transaction_session_cnt | 2,030 |
  | renewal_session_cnt | 2,030 |
- **How to use / Traps:** Not unique per metric, so never use label alone to identify a metric. Use label + `metric_category` as a fallback match when the stripped `metric_id` differs from the config id.
- **Confidence:** Verified (profile, MCP payload, verified doc).

### `bigger_is_better_flag` - boolean
- **Meaning:** Direction of improvement (API `biggerIsBetter`). TRUE = an increase is good. FALSE = a decrease is good (SQL 2026-11-10: churn rates, cancellation rate, exit rate, cost-per-query, max-turns metrics). NULL: not observed.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 2 distinct.
- **Values (complete):**

  | Value | Count | Meaning |
  |---|---|---|
  | True | 257,479 | Higher is better |
  | False | 1,841 | Lower is better (e.g. `customer_churn_rate`, `two_plus_customer_churn_rate`, `wam_cancellation_rate`, `exited_*`, `cost_avg`) |
- **How to use / Traps:** Combine with the sign of `metric_relative_delta` to call improvement vs harm: significant AND `(metric_relative_delta > 0) = bigger_is_better_flag` → improvement. OverallHealth rows are TRUE, which is meaningless for SRM.
- **Confidence:** Verified (profile, SQL, MCP).

### `metric_value` - real
- **Meaning:** The cohort's value of the metric (API `summary.metricValue[cohort]`): per-user mean, rate or ratio, winsorized as Hivemind computes it. On SRM (`cohort_percentage`) rows it is the cohort's **user count** (API `healthMetric.summary.sampleCount`). For example, agi report 2026-10-07: `default` = 117,032, `variant` = 115,717, matching the MCP payload exactly.
- **Profile (scope):** 259,320 rows; null 0.94%; empty 0.0%; 158,330 distinct; min 0.0, max 5,157,370.0, avg 1,906.0749.
- **Values:** high cardinality. Top: `0.0` 45,896; NULL 2,439; `1.0` 1,166; `28.0` 875; `29.0` 145; `0.000360425` 136.
- **How to use / Traps:** For money deltas, use treatment `metric_value` − control `metric_value` on the same report, not `metric_delta` (which is rounded to 3 dp). For precise lift, use treatment/control − 1. Values are Hivemind-adjusted (winsorized), so raw SQL from `experiment_order` differs by a few percent, and Commercial OS differs again. NULL appears on some rows where analysis was skipped. `0.0` is common for skipped or no-activity metrics. Check `metric_warning`.
- **Confidence:** Verified (MCP payload match, SQL, use-cases).

### `expected_proportion` - real
- **Meaning:** SRM rows only: the configured traffic share of the cohort at report time (API `healthMetric.summary.expectedProportion[cohort]`).
- **Profile (scope):** 259,320 rows; null 98.12%; empty 0.0%; 72 distinct; min 0.0, max 1.0, avg 0.4153.
- **Values:** 72 distinct. Top: NULL 254,450; `0.5` 3,396; `0.25` 506; `0.333333` 263; `0.04` 138; `0.01` 116; `0.05` 68.
- **How to use / Traps:** It is NULL on every non-OverallHealth row and on `cohort_id = 'Overall'` rows. A value of **0 or 1 means a post-rollout rerun** (split recomputed as 100/0), so the SRM on that report is spurious. Skip whole **report dates** where any cohort has 0 or 1. Do not filter row by row, or you drop the `'Overall'` p-value row.
- **Confidence:** Verified (verified doc, SQL, MCP).

### `sample_proportion` - real
- **Meaning:** SRM rows only: the observed share of users in the cohort (API `healthMetric.summary.sampleProportion`). Example: agi 2026-10-07 `default` 0.502825 vs MCP 0.50282.
- **Profile (scope):** 259,320 rows; null 98.12%; empty 0.0%; 4,280 distinct; min 0.0217295, max 1.0, avg 0.4172.
- **Values:** Top: NULL 254,450; `0.502083` 66; `0.497917` 66; `0.188722` 33; `1.0` 18; `0.5` 14.
- **How to use / Traps:** Compare to `expected_proportion` for an SRM read. Trust the test result (`metric_pvalue` vs `alpha` = 0.01 / `metric_significance_flag` on the SRM row), not eyeballing.
- **Confidence:** Verified.

### `metric_delta` - real

> ⚠️ **OWNER INPUT NEEDED** (OQ-AR-06): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Absolute difference treatment − control (API `statisticalAnalysis[].absoluteDifference`; Bayesian `difference`), **rounded to 3 decimals** (agi: −0.641 vs API −0.64055).
- **Profile (scope):** 259,320 rows; null 54.61%; empty 0.0%; 9,520 distinct; min −34,244.0, max 411,311.0, avg −33.3096.
- **Values:** Top: NULL 141,625; `0.0` 18,155; `0.001` 6,268; `-0.001` 5,956; `0.002` 3,570; `-0.002` 3,402.
- **How to use / Traps:** NULL on control rows and on skipped comparisons. Present on Bayesian treatment rows, where `metric_relative_delta` is NULL. On SRM treatment rows it is mostly NULL (SQL: 37 of 312 fixed SRM treatment rows NULL). Its meaning there is not documented. Because of the rounding, recompute from `metric_value` for money.
- **Confidence:** Verified (MCP match, SQL).

### `metric_variance` - real
- **Meaning:** Variance of the cohort metric before variance reduction (API `summary.variance`; agi control 11,230.2 vs 11,230.24986). For **ratio** metrics it is the numerator variance (API `numeratorVariance`; MKT_EM_177 control 0.0194323 vs 0.01943).
- **Profile (scope):** 259,320 rows; null 50.31%; empty 0.0%; 92,339 distinct; min 0.0, max 48,711,300.0, avg 50,663.8593.
- **Values:** Top: NULL 130,453; `0.0` 21,864; `6563.6` 72; `7604.05` 72.
- **How to use / Traps:** NULL on SRM rows, on boolean-metric rows (tests `Z Test of Proportions`, `Binomial-WSPRT`: 100% NULL on 2026-11-10), on `Causal Impact` rows and on Bayesian rows. It is an internal stat. Not needed for verdicts.
- **Confidence:** Verified (MCP match, SQL).

### `metric_adjusted_variance` - real
- **Meaning:** Variance after Hivemind's variance reduction (CUPED) (API `summary.adjustedVariance`; agi control 10,703.8 vs 10,703.76506). For ratio metrics: `adjustedNumeratorVariance` (MKT_EM_177 0.0102713 vs 0.01027).
- **Profile (scope):** 259,320 rows; null 55.01%; empty 0.0%; 86,895 distinct; min 0.0, max 1,833,940.0, avg 8,761.4337.
- **Values:** Top: NULL 142,656; `0.0` 18,681; `6350.27` 72; `61.4985` 44.
- **How to use / Traps:** Same NULL pattern as `metric_variance` plus more (no adjustment computed). Internal.
- **Confidence:** Verified.

### `metric_variance_reduction_percentage` - real
- **Meaning:** The share of variance removed by variance reduction, as a **fraction 0–1, not a percent** despite the name (API `summary.varianceReduction`; agi control 0.046881 vs 0.04688). For ratio metrics: `numeratorVarianceReduction` (0.471431 vs 0.47143).
- **Profile (scope):** 259,320 rows; null 55.01%; empty 0.0%; 71,451 distinct; min 0.0, max 1.0, avg 0.0482.
- **Values:** Top: NULL 142,656; `0.0` 37,534; `0.0325024` 72; `1.0` 47.
- **How to use / Traps:** `0.0` = no reduction achieved. Multiply by 100 only for display. Internal.
- **Confidence:** Verified.

### `missing_pre_experiment_data_count` - integer

> ⚠️ **OWNER INPUT NEEDED** (OQ-AR-01): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** The number of the cohort's users with no pre-experiment data, presumably the CUPED covariate period. Arithmetic check: agi report 2026-10-07 `default` 10,955 / 117,032 users = 0.0936, which equals `missing_pre_experiment_data_percentage` 0.0936069. No API field in the MCP payloads.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 3,762 distinct; min 0, max 4,771,145, avg 46,861.2727.
- **Values:** Top: `0` 149,725; `4` 1,654; `1` 1,269; `6` 899; `5` 810; `15` 800.
- **How to use / Traps:** It is never NULL. `0` is written on rows with no variance reduction (SRM rows, boolean metrics), so `0` does not prove "no missing data". Internal.
- **Confidence:** Inferred. The count/percentage relationship is verified by arithmetic. The "pre-experiment covariate (CUPED)" meaning comes from the name plus co-occurrence with the variance-reduction columns, and no source defines it.

### `missing_pre_experiment_data_percentage` - real

> ⚠️ **OWNER INPUT NEEDED** (OQ-AR-01): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** `missing_pre_experiment_data_count` ÷ cohort sample count, as a fraction 0–1.
- **Profile (scope):** 259,320 rows; null 54.74%; empty 0.0%; 10,287 distinct; min 0.0, max 1.0, avg 0.302.
- **Values:** Top: NULL 141,961; `0.0` 7,764; `0.606501` 396; `0.606578` 396; `0.0740741` 367.
- **How to use / Traps:** NULL where no variance reduction was attempted. High values mean CUPED had little to work with. Internal.
- **Confidence:** Inferred (same reasoning as column 21).

### `metric_variance_denominator` - real
- **Meaning:** Ratio metrics only: variance of the denominator (API `summary.denominatorVariance`; MKT_EM_177 control 2.34346 = 2.34346).
- **Profile (scope):** 259,320 rows; null 92.83%; empty 0.0%; 14,367 distinct; min 0.0, max 1,604,380.0, avg 14,740.0302.
- **Values:** Top: NULL 240,731; `0.0` 866; `3429.74` 34; `3707.15` 34.
- **How to use / Traps:** A non-NULL value marks a ratio-type metric (API `dataclassType = RatioType`). On 2026-11-10 it was non-NULL on 100% of `Delta Method-based Z test` and `Ratio-WSPRT` rows. Internal.
- **Confidence:** Verified.

### `metric_adjusted_variance_denominator` - real
- **Meaning:** Ratio metrics: adjusted (CUPED) denominator variance (API `adjustedDenominatorVariance`; 0.609386 vs 0.60939).
- **Profile (scope):** 259,320 rows; null 93.55%; empty 0.0%; 13,925 distinct; min 0.0, max 941,008.0, avg 9,831.2779.
- **Values:** Top: NULL 242,592; `0.0` 1,463; `0.0112808` 12; `206.967` 9.
- **How to use / Traps:** Internal. Tiny values such as `4.93038e-32` are floating-point zeros.
- **Confidence:** Verified.

### `metric_variance_reduction_percentage_denominator` - real
- **Meaning:** Ratio metrics: denominator variance reduction, fraction 0–1 (API `denominatorVarianceReduction`; 0.739963 vs 0.73996).
- **Profile (scope):** 259,320 rows; null 93.55%; empty 0.0%; 13,385 distinct; min 0.0, max 1.0, avg 0.3239.
- **Values:** Top: NULL 242,592; `1.0` 1,207; `0.0` 871; `0.943596` 12. `1.0` appears more than once in the profile list (1,207; 16; 6), presumably float values that print identically.
- **How to use / Traps:** Internal.
- **Confidence:** Verified.

### `missing_pre_experiment_data_count_denominator` - integer

> ⚠️ **OWNER INPUT NEEDED** (OQ-AR-01): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Ratio metrics: count of users missing pre-experiment data for the denominator. In every sampled ratio row it equalled `missing_pre_experiment_data_count` (SQL, 6 rows on 2026-11-10 and MKT_EM_177).
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 2,144 distinct; min 0, max 78,591, avg 64.5207.
- **Values:** Top: `0` 245,322; `1` 711; `5` 305; `2` 296; `6` 287.
- **How to use / Traps:** `0` (not NULL) on non-ratio rows. Internal.
- **Confidence:** Inferred (name + equality with column 21; no source definition).

### `missing_pre_experiment_data_percentage_denominator` - real

> ⚠️ **OWNER INPUT NEEDED** (OQ-AR-01): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Ratio metrics: share of users missing pre-experiment data for the denominator. It equalled `missing_pre_experiment_data_percentage` in sampled rows.
- **Profile (scope):** 259,320 rows; null 93.36%; empty 0.0%; 6,176 distinct; min 0.0, max 1.0, avg 0.2304.
- **Values:** Top: NULL 242,100; `0.0` 3,222; `0.333333` 166; `1.0` 84; `0.5` 54.
- **How to use / Traps:** Internal.
- **Confidence:** Inferred (as column 26).

### `metric_significance_flag` - boolean

> ⚠️ **OWNER INPUT NEEDED** (OQ-AR-07): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Hivemind's significance call for the comparison at the experiment's own alpha (API `statisticalAnalysis[].isSignificant`; SRM: `healthMetric.statisticalAnalysis.isSignificant`).
  - TRUE = significant.
  - FALSE = not significant, **or** no test was run (control rows, skipped comparisons). The column is never NULL.
  - NULL = never occurs.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 2 distinct.
- **Values (complete):**

  | Value | Count | Meaning |
  |---|---|---|
  | False | 248,908 | Not significant / not tested (control row, skipped analysis) |
  | True | 10,412 | Significant at the row's `alpha` |
- **How to use / Traps:** On frequentist treatment rows it agrees exactly with `metric_pvalue < alpha` (SQL 2026-11-10: every TRUE row has p < alpha, no FALSE row does). Use it rather than a hand-picked 0.05.
  - On sequential runs it can be TRUE while `relative_confidence_interval` includes 0 (that CI is fixed-horizon). Trust the flag.
  - FALSE with NULL `metric_pvalue` means "no test", not "not significant".
  - **Bayesian rows:** the flag was FALSE on every Bayesian row on 2026-06-20, 2026-08-22, 2026-10-01 and 2026-11-10 (only 1 and 15 TRUE on 2026-05-01 / 2026-06-01), even with `metric_pbest` up to 1.0. Do not use it for Bayesian experiments. Read `metric_pbest` / `credible_interval`.
  - On SRM rows TRUE = SRM detected.
- **Confidence:** Verified (SQL, MCP, verified doc).

### `alpha` - real
- **Meaning:** Significance level used for the comparison (API `statisticalAnalysis[].alpha`). It is the experiment's configured alpha (metadata `alpha_value`) on metric rows, and 0.01 on SRM rows.
- **Profile (scope):** 259,320 rows; null 55.66%; empty 0.0%; 3 distinct; min 0.01, max 0.2, avg 0.1234.
- **Values (complete):**

  | Value | Count | Meaning |
  |---|---|---|
  | NULL | 144,346 | No test on this row: control rows, skipped comparisons, Bayesian rows |
  | 0.1 | 67,418 | 90% two-sided level (experiment alpha) |
  | 0.2 | 36,684 | 80% two-sided level (experiment alpha) |
  | 0.01 | 10,872 | SRM test alpha (OverallHealth), or experiments configured at 0.01 |
- **How to use / Traps:** Applied per treatment arm vs control. Metadata allows other alphas (0.4, 0.05, 0.3, 0.15 …) that were not in this sample. Never substitute 0.05.
- **Confidence:** Verified.

### `metric_pvalue` - real
- **Meaning:** p-value of the comparison (API `pValue`). It is an always-valid p-value on sequential (WSPRT) tests.
- **Profile (scope):** 259,320 rows; null 55.66%; empty 0.0%; 26,784 distinct; min 0.0, max 1.0, avg 0.7446.
- **Values:** Top: NULL 144,346; `1.0` 32,841; `0.0` 977; `0.757575` 364; `0.838679` 168; `0.976793` 168.
- **How to use / Traps:**
  - NULL = no test (control row, skipped analysis, Bayesian). Read `metric_warning`.
  - In sequential runs `1.0` means the boundary was not crossed. On 2026-11-10, 2,738 of 2,776 informative `Binomial-WSPRT` rows were 1.0.
  - Identical p-values across arms or decision metrics are expected (multiplicity adjustment). Report each as stored.
- **Confidence:** Verified.

### `confidence_interval` - string
- **Meaning:** Absolute-difference confidence interval as text `'(lo, hi)'`, rounded to 3 dp (API `absoluteCI`; agi `(-1.481, 0.2)` matches).
- **Profile (scope):** 259,320 rows; null 56.47%; empty 0.0%; 45,109 distinct; length 10–23.
- **Values:** Top: NULL 146,437; `(-0.0, 0.0)` 4,743; `(-0.001, 0.001)` 2,732; `(-0.001, 0.0)` 1,854; `(-0.0, 0.001)` 1,810.
- **How to use / Traps:** Text, so parse it with `SPLIT_PART`/`REGEXP_SUBSTR`. Two-sided at (1 − alpha). Rounding collapses small CIs to `(-0.0, 0.0)`. NULL on control, skipped and Bayesian rows.
- **Confidence:** Verified.

### `metric_relative_delta` - real
- **Meaning:** Relative lift vs control, (treatment / control − 1), as a ratio rounded to 3 dp (API `relativeDifference`; agi −0.019 vs −0.01857).
- **Profile (scope):** 259,320 rows; null 56.48%; empty 0.0%; 4,556 distinct; min −278.43, max 980.538, avg 0.2264.
- **Values:** Top: NULL 146,458; `0.001` 889; `-0.001` 811; `0.002` 783; `0.0` 783.
- **How to use / Traps:** NULL on control, skipped and **all Bayesian** rows (SQL). Precision is 0.1 pp. For precise lift compute it from `metric_value`. Extreme values come from near-zero control values.
- **Confidence:** Verified.

### `relative_confidence_interval` - string

> ⚠️ **OWNER INPUT NEEDED** (OQ-AR-05): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Relative-lift confidence interval as text, two-sided at (1 − alpha) (API `relativeCI`). An 80% CI when alpha = 0.2.
- **Profile (scope):** 259,320 rows; null 56.47%; empty 0.0%; 82,075 distinct; length 10–20.
- **Values:** Top: NULL 146,437; `(None, None)` 883; `(-0.191, 0.063)` 72; `(-1.0, -1.0)` 59; `(-inf, None)` 50; `(-2.644, 0.645)` 43.
- **How to use / Traps:** On sequential runs it is a fixed-horizon interval and can include 0 while the flag is TRUE. Trust `metric_significance_flag`. The literal `None` / `-inf` tokens are Python serializations (the interval is undefined). They are presumably caused by a zero control value, but sources do not state the cause. Do not cast this column to numbers without handling those tokens.
- **Confidence:** Verified (mapping, verified doc). The cause of the `None`/`inf` tokens is Inferred.

### `metric_pbest` - real
- **Meaning:** Bayesian probability that the treatment is best vs control (API `BayesianMethod.pBest`).
- **Profile (scope):** 259,320 rows; null 100.0%; empty 0.0%; 0 distinct. The scope contained no Bayesian experiments.
- **Values (complete in scope):**

  | Value | Count | Meaning |
  |---|---|---|
  | NULL | 259,320 | Frequentist row (or Bayesian control row) |

  Outside scope (SQL 2026-11-10): non-NULL on 1,063 Bayesian treatment rows, range 0.0–1.0.
- **How to use / Traps:** It is the only verdict signal on Bayesian rows (`analysis_type = 'bayesian'`). Do not call a Bayesian result "significant / not significant".
- **Confidence:** Verified (SQL, MCP).

### `credible_interval` - string
- **Meaning:** Bayesian credible interval of the absolute difference, as text (API `credibleInterval`; serp_precheck treatment1 `(0.004, 0.012)`).
- **Profile (scope):** 259,320 rows; null 100.0%; empty 0.0%; 0 distinct.
- **Values (complete in scope):**

  | Value | Count | Meaning |
  |---|---|---|
  | NULL | 259,320 | Frequentist row / Bayesian control row |
- **How to use / Traps:** Present on every Bayesian treatment row that has a test (SQL 2026-11-10: 1,063 rows).
- **Confidence:** Verified.

### `statistical_test_name` - string

> ⚠️ **OWNER INPUT NEEDED** (OQ-AR-04): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Name of the statistical test used for this comparison. There is no direct field in the MCP analysis payload, which only gives `dataclassType = FrequentistMethod` / `BayesianMethod`. The test depends on metric type (numeric / boolean / ratio) × method (fixed / sequential), as established by co-occurrence with the variance columns and `analysis_method_used` (SQL 2026-11-10).
- **Profile (scope):** 259,320 rows; null 55.66%; empty 0.0%; 9 distinct (+NULL); length 6–56.
- **Values (complete in scope):**

  | Value | Count | Meaning |
  |---|---|---|
  | NULL | 144,346 | No test: control rows, skipped comparisons, SRM control rows |
  | T Test | 33,921 | Fixed design, numeric metric (API `NumericType`) |
  | Z Test of Proportions | 28,996 | Fixed design, boolean/proportion metric (`metric_variance` NULL) |
  | 𝓉-WSPRT | 16,360 | Sequential, numeric metric (weighted sequential probability ratio t-test). Name begins with the Unicode character U+1D4C9 |
  | Binomial-WSPRT | 13,885 | Sequential, boolean metric |
  | Causal Impact | 8,781 | PRE-POST experiments (`control_cohort = 'pre'`, scorecard `CDE-PrePost-Default-Scorecard`) |
  | Delta Method-based Z test | 6,284 | Fixed design, ratio metric (denominator variance present) |
  | Ratio-WSPRT | 4,561 | Sequential, ratio metric |
  | Weighted Sequential Likelihood Ratio Test of Proportions | 2,091 | SRM test on OverallHealth rows (alpha 0.01), used under both fixed and sequential analysis |
  | Mann-Whitney U | 95 | Rank test seen on a few informative metrics in fixed analyses. Selection rule not documented. **⚠️ OWNER INPUT NEEDED (OQ-AR-04)** |

  Outside scope (SQL 2026-11-10): `Bayesian Boolean` (385 rows) and `Bayesian Numeric` (678 rows) on Bayesian experiments.
- **How to use / Traps:** The test for a metric can change between reports when the method switches. For example, agi's decision metric used `𝓉-WSPRT` on report 2026-10-06 and `T Test` on 2026-10-07. Compare like with like (same `analysis_method_used`). Match `𝓉-WSPRT` with `LIKE '%-WSPRT'` rather than typing the Unicode character.
- **Confidence:** Verified (values from profile/SQL; the type/method mapping comes from SQL co-occurrence). The Mann-Whitney selection rule is Unknown.

### `metric_warning` - string
- **Meaning:** Hivemind warning text for the metric (API `businessMetrics[].warnings` / `healthMetric.warnings`, joined into one string). `''` = no warning. Never NULL.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 73.86%; 302 distinct; length 0–470.
- **Values:** 302 distinct. Top values:

  | Value | Count |
  |---|---|
  | `''` (no warning) | 191,527 |
  | ⚠ Analysis skipped: ['off', 'on'] cohorts only have one value | 10,668 |
  | ⚠ Analysis skipped: ['off'] cohorts only have one value | 3,930 |
  | ⚠ Analysis skipped: [] cohorts only have one value | 3,446 |
  | ⚠ Analysis skipped: ['call', 'no_call'] cohorts only have one value | 3,160 |
  | ⚠ Analysis skipped: ['baseline', 'trad-domainreactive-keepalive-banner-critical', … (truncated in profile) | 2,989 |
  | ⚠ Analysis skipped: ['off_generic', 'on_pricing'] cohorts only have one value | 1,990 |
  | ⚠ Analysis skipped: ['on'] cohorts only have one value | 1,804 |
  | ⚠ Analysis skipped: ['wapaab', 'wapaabh'] cohorts only have one value | 1,290 |
  | ⚠ Analysis skipped: ['ctrl', 'testa', 'testb', 'testc'] cohorts only have one value | 1,280 |
  | ⚠ Analysis skipped: ['control', 'test'] cohorts only have one value | 1,264 |
  | ⚠ Pairwise comparison skipped for ['trad-domainreactive-keepalive-inline-critical-maxsuppress']: these cohorts only have one value | 1,232 |
  | ⚠ Analysis skipped: ['auconb', 'auconbh'] cohorts only have one value | 1,082 |
  | ⚠ Analysis skipped: ['no_call'] cohorts only have one value | 1,048 |
  | ⚠ Analysis skipped: ['control'] cohorts only have one value | 1,042 |

  Warning families seen on report 2026-11-10 (SQL, 62,567 rows):

  | Family | Rows | Meaning (from text) |
  |---|---|---|
  | (empty) | 46,761 | No warning |
  | `⚠ Analysis skipped: [...] cohorts only have one value` | 13,487 (all NULL p-value) | Metric constant in the listed cohorts, so no test was run |
  | `⚠ Pairwise comparison skipped for [...]` | 402 | Only the listed arm(s) skipped |
  | `⚠ Analysis of <metric> may be incomplete; cohorts [...] mismatch expected cohorts [...]` | hundreds (many variants) | Some cohorts are missing from the metric's data |
  | `⚠ No new users found in experiment for at least 3 days. If the experiment is still active, traffic data (#traffic) may be stale; check this report again tomorrow` | 69 | Bucketing stopped or traffic stale |
  | `⛔ Unhealthy experiment detected: sample ratio mismatch` | 38 | SRM flagged on the experiment |
- **How to use / Traps:** Filter with `metric_warning <> ''` (not `IS NOT NULL`). NULL p-value + "Analysis skipped" = no test run, not "not significant". Text begins with ⚠ / ⛔ characters, so match with `LIKE '%Analysis skipped%'`.
- **Confidence:** Verified (profile, SQL, MCP payload warnings).

### `decision_metric_flag` - boolean
- **Meaning:** TRUE when the metric is a decision metric on this report. It is exactly equivalent to `metric_category = 'decision'` (scope: 5,036 TRUE = 5,036 decision rows; SQL 2026-11-10: all decision rows TRUE, all others FALSE).
  - TRUE = decision metric.
  - FALSE = guardrail / informative / OverallHealth.
  - NULL = never occurs.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 2 distinct.
- **Values (complete):**

  | Value | Count | Meaning |
  |---|---|---|
  | False | 254,284 | Not a decision metric |
  | True | 5,036 | Decision metric |
- **How to use / Traps:** Redundant with `metric_category`. Either works.
- **Confidence:** Verified.

### `metric_category` - string
- **Meaning:** The role of the metric in this experiment's scorecard (API `businessMetrics[].metricCategory`). `OverallHealth` = the API `healthMetric` (SRM).
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 4 distinct; length 8–13.
- **Values (complete):**

  | Value | Count | Meaning |
  |---|---|---|
  | informative | 242,453 | Monitoring metrics; not used for the verdict (MCP: scorecard metrics not chosen as decision or guardrail) |
  | OverallHealth | 7,248 | SRM rows: `metric_id = 'cohort_percentage'`, label `Expected cohort percentage` |
  | decision | 5,036 | Decision (primary) metrics that drive the verdict |
  | guardrail | 4,583 | Guardrail ("do no harm") metrics |
- **How to use / Traps:** Take the verdict from the latest report_date that **has** decision rows, because some later runs carry only informative metrics. A report is complete when its decision-row count equals the previous report's. No guardrail rows + empty metadata `guardrail_metric_id_list` = no guardrails configured. Say so, and never claim "no harm". A metric can appear in two categories on one report: keep decision > guardrail > informative.
- **Confidence:** Verified.

### `control_cohort` - string
- **Meaning:** The control/baseline cohort id for the experiment on this row (API `analysis.baseline`; config `baselineCohortId`). The value is constant for an experiment on a report.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 33 distinct; length 1–57.
- **Values (complete):** Each value is the id of an experiment's control arm. The meaning of a specific label is experiment-specific and not documented beyond the label itself. Where a count equals one experiment's total row count in the profile, that experiment is noted (Inferred from count equality).

  | Value | Count | Meaning |
  |---|---|---|
  | off | 63,744 | Feature-off control (Inferred from label only; not documented — owner input needed) |
  | trad-domainreactive-keepalive-bannerinline-critical | 31,802 | Control arm of a multi-arm DCC keep-alive test (likely part of `dcc_jtbd_persona_traditional_keepalive_endgame`; Inferred from the label, not a count match: 31,802 vs that experiment's 41,738 rows) |
  | pre | 20,660 | Pre-period baseline of PRE-POST / Causal Impact experiments (= 2 × 10,330 `pre`/`post` rows) |
  | control | 19,247 | Generic control |
  | 22_Ctrl | 14,118 | Control of `MKT_EM_OCM_Domain_TransferToCompleted_GD_22LumaCTA3` (count match, inferred) |
  | control_variant | 13,754 | Generic control (paired with `b_variant`) |
  | baseline | 12,369 | Generic baseline arm |
  | off_generic | 11,662 | Control paired with `on_pricing` (from warning text) |
  | no_call | 8,998 | Control paired with `call` (from warning text) |
  | ctrl | 7,987 | Generic control (paired with `testa`..`testc`, from warning text); count equals `MKT_EM_1250_BMAT-Failed-Billing-Email1_CM-23840` (7,987), inferred |
  | control1 | 7,912 | Control of `serp_new_spinner_all_markets_mobile_mod_and_non_mod` (count match, inferred) |
  | 1046-Airo-CSPostPub-US-OGSL_CM-22982 | 6,294 | Control of `MKT_EM_1046_Airo-Coming-Soon-Publish_EN-US_CM-22959` (count match, inferred) |
  | 7619 | 5,025 | Control of `MKT_EM_OCM_Renewals_ParkedDomainAlertDay4_GD_REN-6691` (count match, inferred) |
  | wacmnth | 4,746 | Meaning not documented — owner input needed |
  | waddch | 3,642 | Meaning not documented — owner input needed |
  | cohort1 | 3,442 | Meaning not documented — owner input needed |
  | auconbh | 3,134 | Paired with `auconb` (from warning text); meaning not documented |
  | wapaab | 2,788 | Paired with `wapaabh` (from warning text); meaning not documented |
  | smsidm | 2,762 | Meaning not documented — owner input needed |
  | 873_Free-WM-Traffic-Health-Control-enUS_CM-25355 | 2,621 | Named control of a CM email/traffic test |
  | wapaabh | 2,000 | Meaning not documented — owner input needed |
  | 7817_Control | 1,844 | Named control |
  | aabf2p55h | 1,558 | Meaning not documented — owner input needed |
  | 1731_Control | 1,242 | Named control |
  | 49-Welcome-Generic_OctCtrl_CM-25602 | 1,230 | Named control |
  | 566_ctrl | 1,026 | Named control |
  | 1222_V3-Ctrl-Claim-TrustedSite-EC-English-Manage_CM-24424 | 1,008 | Named control |
  | binshown | 829 | Meaning not documented — owner input needed |
  | A | 668 | Control arm labelled `A`; meaning not documented — owner input needed |
  | 1211_DomainMatch-COMtoCA-BizEmail_Image_CM-22428 | 432 | Named control |
  | trust-base | 384 | Meaning not documented — owner input needed |
  | FreeMAT-Expiration_Control_CM-23292 | 212 | Named control |
  | default | 180 | Generic default arm (e.g. `agi_*` experiments; MCP `baseline = 'default'`) |
- **How to use / Traps:** Treatment rows = `cohort_id <> control_cohort` (and `<> 'Overall'`). Prefer this over metadata `baseline_cohort_id`. Labels are case-sensitive. For PRE-POST experiments `pre` is a time period, not a randomized arm.
- **Confidence:** Verified (column meaning). Per-value experiment attributions are Inferred where noted.

### `scorecard_template_name` - string
- **Meaning:** Hivemind scorecard template that defined the metric set for this analysis (API `analysis.scorecardTemplateName` = config `scorecardTemplate`; agi `AGI_Mon_Email_ERP` matches both).
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 49 distinct; length 4–50.
- **Values (complete):** Template contents are not documented in the sources; meanings are the template name only. Experiment attributions are noted where the count equals a single experiment's row count (Inferred).

  | Value | Count | Meaning |
  |---|---|---|
  | dcc_jtbd_endgame_scorecard | 41,738 | Used by `dcc_jtbd_persona_traditional_keepalive_endgame` (count match) |
  | cm-renewals-audience-recording | 26,170 | Customer-marketing renewals template (Inferred from name; contents not documented — owner input needed). Also in experiment_order/UAH |
  | CDE-PrePost-Default-Scorecard | 20,660 | Default PRE-POST template (Causal Impact; control `pre`) |
  | blog_experiment_PAR | 16,110 | Used by `gd-resources-ca-left-placement-ads-6cde0280f748` (count match) |
  | cm-generic-engagement | 15,694 | Template contents not documented — owner input needed |
  | Activation-CVR | 11,150 | Template contents not documented — owner input needed |
  | aab-budget-service-integration | 10,166 | Used by `airo-builder-manage-plan-aa` (count match) |
  | CDT-Test-Metrics | 8,998 | Template contents not documented — owner input needed (control `no_call`, same count) |
  | cm-website-builder-attach-specific-manage | 8,915 | Template contents not documented — owner input needed |
  | Commerce-Activation-Scorecard | 8,412 | Used by `cx_onboarding_bank_payout_ab_test` (count match) |
  | dpp-e2e | 7,912 | Used by `serp_new_spinner_all_markets_mobile_mod_and_non_mod` (count match) |
  | sso_phone_consent_collection_login_v2_scorecard | 6,761 | Used by `sso_phone_consent_collection_login_v2` (count match) |
  | Activation-Airo-AI-Builder | 6,346 | Template contents not documented — owner input needed |
  | bell_notifications | 5,866 | Used by `notifications_hide_email_expired` (count match) |
  | MHWP_SmartSite_ProtectionBundle | 5,230 | Template contents not documented — owner input needed |
  | mobile_ios_site_renewal | 4,834 | Used by `mobile_ios_sites_renewal_us` (count match) |
  | hp_core_metrics | 4,244 | Homepage core metrics (Inferred from name; contents not documented — owner input needed). Also in experiment_order/UAH |
  | commerce-order-fulfillment | 4,136 | Template contents not documented — owner input needed |
  | cart | 4,088 | Template contents not documented — owner input needed |
  | hp_intl_aab | 4,016 | Template contents not documented — owner input needed |
  | vbo_renewal | 3,442 | Template contents not documented — owner input needed (control `cohort1`, same count) |
  | Activation-Email-Investors | 3,134 | Template contents not documented — owner input needed (control `auconbh`, same count) |
  | mobile_iOS_navigation | 3,098 | Template contents not documented — owner input needed |
  | hp_us_core | 2,976 | Template contents not documented — owner input needed |
  | bell_notifications_attach | 2,664 | Template contents not documented — owner input needed |
  | fos-airo-ai-builder | 2,220 | Template contents not documented — owner input needed (also in experiment_order) |
  | Commerce_Growth | 1,714 | Template contents not documented — owner input needed |
  | fos_merch_hosting_cnp-mwp | 1,682 | Template contents not documented — owner input needed |
  | productivity_post_password_rec | 1,634 | Template contents not documented — owner input needed |
  | dcc_dmarc_copy_in_xgrade | 1,571 | Template contents not documented — owner input needed |
  | sso_login_levelup_studio_push_to_approve_scorecard | 1,540 | Template contents not documented — owner input needed |
  | Activation-Email-Onboarding-General-Updated | 1,230 | Template contents not documented — owner input needed |
  | usi_wam_editor | 1,020 | Template contents not documented — owner input needed |
  | Activation-TrustedSite | 1,008 | Template contents not documented — owner input needed |
  | usi-wam-ols-editor | 962 | Template contents not documented — owner input needed |
  | dcc_domain_connect_wizard_redesign | 862 | Template contents not documented — owner input needed |
  | MHWP_Security | 860 | Template contents not documented — owner input needed |
  | tdfs-pis-buynow | 829 | Template contents not documented — owner input needed (control `binshown`, same count) |
  | productivity_post_password_rec_v2 | 812 | Template contents not documented — owner input needed |
  | fos-airo-ai-builder-ec | 770 | Template contents not documented — owner input needed |
  | cm-auction-domain-attach-specific-manage | 692 | Template contents not documented — owner input needed |
  | aab_T2_First_Build | 682 | Template contents not documented — owner input needed |
  | domain-find-abridged | 444 | Template contents not documented — owner input needed |
  | cm-domain-attach-specific-manage | 432 | Template contents not documented — owner input needed |
  | productivity_primary_actions_redesign_scorecard | 418 | Template contents not documented — owner input needed |
  | GoDaddy-app_mobile_SSO_Scorecard | 390 | Template contents not documented — owner input needed |
  | tdfs-gd-buynow | 384 | Template contents not documented — owner input needed (control `trust-base`, same count) |
  | digital-care | 224 | Template contents not documented — owner input needed |
  | AGI_Mon_Email_Upgrade | 180 | Template contents not documented — owner input needed (control `default`, same count) |
- **How to use / Traps:** Case-sensitive, mixed naming conventions. Metadata has 1,384 templates table-wide. Use `get_scorecard_template_by_id` (MCP) for the metric list of a template.
- **Confidence:** Verified (column meaning). Template contents and the experiment attributions are Inferred / undocumented as noted.

### `analysis_type` - string
- **Meaning:** Statistical paradigm of the analysis (API `analysis.analysisType`; MCP enum `analysisType`: `frequentist`, `bayesian`).
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 1 distinct; length 11–11.
- **Values (complete in scope):**

  | Value | Count | Meaning |
  |---|---|---|
  | frequentist | 259,320 | p-value / alpha / CI logic applies |

  Outside scope (SQL 2026-11-10): `bayesian` on 2,144 rows / 17 experiments. Use `metric_pbest` + `credible_interval` there. `metric_pvalue`, `alpha`, `metric_relative_delta`, `confidence_interval` are NULL, and `metric_significance_flag` is unreliable (FALSE).
- **How to use / Traps:** Branch every verdict query on this column.
- **Confidence:** Verified.

### `fixed_design_flag` - boolean
- **Meaning:** API `isFixedDesign` (MCP: TRUE = fixed sample size design, FALSE = adaptive/sequential).
  - TRUE = fixed design per config.
  - FALSE = never observed here.
  - NULL = never observed here.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 1 distinct.
- **Values (complete):**

  | Value | Count | Meaning |
  |---|---|---|
  | True | 259,320 | Fixed design flag set |
- **How to use / Traps:** It is TRUE even on `analysis_method_used = 'sequential'` rows (SQL 2026-11-10: all 21,278 sequential rows TRUE). It does **not** tell you whether the analysis was sequential. Use `analysis_method_used`.
- **Confidence:** Verified.

### `start_mst_ts` - timestamp

> ⚠️ **OWNER INPUT NEEDED** (OQ-AR-02): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Start of the analysis window = the experiment's first user bucketing (API `healthMetric.startTime`). It equals metadata `first_user_bucket_mst_ts` on both verified experiments (agi 2026-08-26 14:32:44; MKT_EM_177 2026-09-10 09:45:46). Labelled MST.
- **Profile (scope):** 259,320 rows; null 1.38%; empty 0.0%; 489 distinct; min 2026-05-18 16:01:51, max 2026-11-09 14:01:14.
- **Values:** Top: 2026-07-22 07:22:34 26,082; 2026-07-22 00:20:17 12,816; 2026-07-09 10:15:47 7,987; 2026-08-12 00:14:36 7,976.
- **How to use / Traps:** For real experiment dates use metadata `first_user_bucket_mst_ts` / `last_user_bucket_mst_ts`. NULL on whole experiments (SQL 2026-11-10: e.g. `AIWD-2670`, `cde_app_350a20b8952b420b9c4b78b7b0aeef12_2026-10-29`, `MKT_EM_1309_Airo-AI-Builder-Free-Users-OB-CM-23062`). `end_mst_ts` is NULL on the same rows. The sources do not say why.
- **Confidence:** Verified (meaning). The reason for NULLs is Unknown.

### `end_mst_ts` - timestamp

> ⚠️ **OWNER INPUT NEEDED** (OQ-AR-02): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** End of the analysis window, meaning the latest bucketing/data timestamp the report covers (API `healthMetric.endTime`). Example: agi report 2026-10-06 → 2026-10-05 23:59:52; report 2026-10-07 → 2026-10-06 23:59:05 (MCP `endTime` 2026-10-06 23:59:05).
- **Profile (scope):** 259,320 rows; null 1.38%; empty 0.0%; 1,907 distinct; min 2026-06-28 00:00:00, max 2026-11-12 16:59:24.
- **Values:** Top: 2026-08-21 23:57:05 26,552; NULL 3,582 (as listed in profile values); 2026-07-30 11:31:37 2,652; 2026-07-28 20:49:10 944.
- **How to use / Traps:** It advances with each report while the experiment buckets and freezes after bucketing stops. A report covers data only to about the prior day. Not a substitute for metadata real end (`last_user_bucket_mst_ts`). Never use metadata `ended_at_mst_ts` / `end_mst_date` either. The profile lists 3,582 NULLs versus 1.38% (≈3,579) in null_pct. The small difference is in the profile itself.
- **Confidence:** Verified.

### `insert_utc_ts` - timestamp
- **Meaning:** UTC time the report row was written. It matches the Hivemind report time (API `createdAt`) within minutes and is the reliable run time.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 3,105 distinct; min 2026-07-01 07:24:27.837549, max 2026-11-13 07:45:20.223162.
- **Values:** Top: 2026-10-29 05:19:22.164071 421; 2026-10-27 05:11:51.691237 421; 2026-10-28 05:24:07.220889 421; … (one timestamp per experiment run).
- **How to use / Traps:**
  - MST run time = `DATEADD(hour,-7, MAX(insert_utc_ts))` per report_date.
  - For "last report before rollout", compare it to `DATEADD(hour,-7, rollout_mst_ts)` (rollout_mst_ts is UTC).
  - Detect missing runs as gaps of more than 36 h between consecutive run times, not as skipped labels.
  - One report_date can span inserts from ~07:26 UTC that day to ~07:26 UTC the next (SQL 2026-11-10).
- **Confidence:** Verified.

### `analysis_method_used` - string
- **Meaning:** Statistical method actually used on this report (API `analysis.analysisMethodUsed`).
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 2 distinct; length 5–10.
- **Values (complete):**

  | Value | Count | Meaning |
  |---|---|---|
  | fixed | 170,563 | Fixed-horizon tests (T Test, Z Test of Proportions, Delta Method-based Z test, Causal Impact, Mann-Whitney U) |
  | sequential | 88,757 | Sequential always-valid tests (𝓉-WSPRT, Binomial-WSPRT, Ratio-WSPRT). p = 1.0 until the boundary is crossed |
- **How to use / Traps:** It can switch mid-life (sequential ↔ fixed), so restrict trend and flicker analysis to the anchor report's method. Bayesian reports are `fixed`.
- **Confidence:** Verified.

### `experiment_id` - string
- **Meaning:** Hivemind experiment key (API `experimentId`). Case-sensitive. Mixed formats: snake_case (`agi_plg_email_inline_upgrade_erp`), CM email codes (`MKT_EM_177_Failed-Billing-Manual-D28_CM-24927`), hashed ids (`gd-resources-ca-left-placement-ads-6cde0280f748`), ticket-like (`AIWD-2670`).
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 150 distinct (sample of 150 experiments; 897 on report 2026-11-10); length 15–64.
- **Values:** Top in scope: `dcc_jtbd_persona_traditional_keepalive_endgame` 41,738; `gd-resources-ca-left-placement-ads-6cde0280f748` 16,110; `MKT_EM_OCM_Domain_TransferToCompleted_GD_22LumaCTA3` 14,118; `airo-builder-manage-plan-aa` 10,166; `MKT_EM_17_Email-Suspend-Cancel-Alert_Suspension_CM-24145` 9,694.
- **How to use / Traps:** Always filter on it (mandatory). Exact match, so never use `LOWER()` or `ILIKE` for joins. Joins many-to-one to `experiment_metadata_snap`.
- **Confidence:** Verified.

### `report_date` - string
- **Meaning:** Label of the Hivemind analysis run, `'YYYY-MM-DD'` STRING, used as the partition column. Runs execute around 20:30–00:30 MST with data through about the prior day. A run just after midnight carries the previous day's label.
- **Profile (scope):** 259,320 rows; null 0.0%; empty 0.0%; 134 distinct; length 10–10.
- **Values:** Top: `2026-08-07` 2,724; `2026-08-05` 2,658; `2026-08-06` 2,638; `2026-08-10` 2,602.
- **How to use / Traps:**
  - Compare as a string (`report_date = '2026-11-10'`) or `TO_DATE(report_date,'YYYY-MM-DD')`.
  - History is kept, with one method per experiment per date. Some runs are missing (0–14% in samples).
  - The newest date is often a partial load. Check its decision-row count against the previous date.
  - After rollout, reruns recompute with a 100/0 expected split, so SRM on those dates is spurious.
- **Confidence:** Verified.

---

## ⚠️ **OWNER INPUT NEEDED** — definitions only the Hivemind owners can supply

The definitions below are **not clear** from the data, Hivemind MCP or any document we could find. **A Hivemind data owner needs to fill in the "Owner answer" column.** Until then, agents must answer "definition not confirmed by the Hivemind owners" for these and must not guess.

| ID | Column / value | Question for the owner | Why it matters | What we observed | Owner answer |
|---|---|---|---|---|---|
| OQ-AR-01 | `missing_pre_experiment_data_* (4 columns)` | What do these measure (which pre-period, which covariate for CUPED)? | Variance-reduction questions. | No API field. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-AR-02 | `start_mst_ts / end_mst_ts` | Why are they NULL for whole experiments (1.38% of rows, e.g. AIWD-2670, cde_app_*)? | Analysis-window questions. | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-AR-03 | `market_sub_region codes 'DEM', 'ROW DM', 'ROW EM', 'OTHERS'` | What do these codes stand for? | Segment questions (pre-June 2026 data). | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-AR-04 | `statistical_test_name = 'Mann-Whitney U'` | When does Hivemind choose Mann-Whitney U instead of a T test or Z test? | Explaining which test produced a p-value. | 95 rows in the profile. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-AR-05 | `relative_confidence_interval tokens '(None, None)', '(-inf, …)', '(-1.0, -1.0)'` | What produces these degenerate intervals (zero control, one-valued cohort)? | Reading CIs. | Assumed degenerate inputs; unconfirmed. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-AR-06 | `metric_delta on SRM (OverallHealth) treatment rows` | What does metric_delta mean on SRM rows? | SRM explanations. | Populated, meaning unknown. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-AR-07 | `metric_significance_flag on Bayesian rows` | The flag is FALSE in Redshift where the API says isSignificant = true. Is this an ETL mapping gap? | Bayesian experiments may be reported as not significant. | Seen on cart_payment_lockicon_v4 and others. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-AR-08 | `control-cohort labels (wacmnth, waddch, smsidm, aabf2p55h, binshown, trust-base, A, …) and scorecard template contents` | Is there a reference list of cohort label conventions and template contents? | Interpreting cohort names (control must still come from baseline_cohort_id). | Use get_scorecard_template_by_id / get_experiment_by_id per experiment. | _TO BE FILLED BY HIVEMIND OWNER_ |

## Sources

1. Live column profile: `analysis_result_cln.profile.json` (author's local workspace, not committed) (scope: 259,320 rows, report_date >= 2026-07-01, 150 hash-sampled experiments).
2. Verified table doc (9/9 vs Hivemind MCP): `tables/hivemind_mart.analysis_result_cln.md`.
3. Hivemind MCP enums: Hivemind MCP tool input schemas (analysisType, isFixedDesign, alpha, healthMetric_srm).
4. MCP payloads: saved MCP payloads (author's local workspace). Field mappings were checked against `agi_plg_email_inline_upgrade_erp` (NumericType, healthMetric), `MKT_EM_177_Failed-Billing-Manual-D28_CM-24927` (RatioType) and `serp_precheck_m365_mod_and_non_mod_1col_single_only` (Bayesian).
5. Use cases: `experimentation-hm-did-it-work.md`, `experimentation-hm-whats-it-worth.md` in `use-cases/`.
6. Read-only Redshift SQL, 2026-11-13: report_date 2026-11-10 composition and null/test patterns; segment-era reports 2026-05-01 / 2026-06-01 / 2026-06-02..19; per-experiment row checks for the three experiments above; join check vs `experiment_metadata_snap`.
7. Sibling profiles (cross-table map): `experiment_metadata_snap.profile.json`, `experiment_order.profile.json`, `user_allocation_history.profile.json`.
