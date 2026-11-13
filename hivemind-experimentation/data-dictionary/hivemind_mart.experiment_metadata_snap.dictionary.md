# hivemind_mart.experiment_metadata_snap - Data Dictionary

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-16): table-level definition not clear (the physical column count changed from 73 to 74; which column was added and what it means). Hivemind owner to fill in. See the owner table at the end of this file.

**Purpose.** One row per registered Hivemind experiment: its configuration (type, cohorts, baseline, metrics, alpha, traffic), ownership (BU, team, owners), documentation (hypothesis, observation, customer problem), real bucketing window and sample, health (SRM, cross-bucketing), the human declared result (`rollout_decision_enum`), the automatic conclusion, the rigor badge, and the governed annualized iGCR (`annualization_value`). It is the spine for every single-experiment lookup ("what was X, who owns it, did it win, what was it worth") and for every portfolio count. Most columns are a flattened copy of the Hivemind `get_experiment_by_id` config plus the `healthMetric` block of the latest `get_experiment_analysis` report (mappings below were checked on 17 prod experiments, 2026-11-13).

| Item | Value |
|---|---|
| Grain | one row per `experiment_id` (current state; refreshed daily, about D-1) |
| Primary key | `experiment_id` (25,109 rows = 25,109 distinct ids; case-sensitive - only 25,078 distinct after `LOWER()`, so 31 rows collide by case) |
| Partition / sort keys | none; full table scans in seconds |
| Physical columns | 74 (this document covers all 74) |
| Mandatory filter (portfolio questions only) | `experiment_type_enum = 'A/B' AND experiment_id NOT ILIKE 'ff-%' AND experiment_id NOT ILIKE 'cde\\_app\\_%' AND COALESCE(status_enum,'') NOT IN ('deleted','draft')` - killed / rolled-back experiments stay IN the population |
| Mandatory filter (single-experiment questions) | `experiment_id = '<exact id>'` (exact case) |
| iGCR reporting filter | `annualization_value IS NOT NULL AND rollout_decision_enum = 'win' AND deciding_cohort_id_text <> 'ineligible'` |
| Timezone rule | `*_mst_ts` / `*_mst_date` are MST (fixed UTC-7) EXCEPT `rollout_mst_ts`, which holds a UTC instant - use `DATEADD(hour, -7, rollout_mst_ts)` |

**Row counts by scope (profile + SQL, 2026-11-13)**

| Scope | Rows |
|---|---|
| Full table (all types) | 25,109 |
| `experiment_type_enum = 'A/B'` | 12,142 |
| A/B portfolio population (mandatory filter above) | 11,462 |
| A/B population with a declared win / loss / inconclusive | 7,646 |
| A/B population never bucketed (`first_user_bucket_mst_ts` NULL) | 2,356 |
| A/B population with `annualization_value` | 579 (505 meet the iGCR reporting filter) |

**A/B population fill rate by real start year (SQL)** - many config fields were added to Hivemind over time:

| Real start | Rows | status_enum NULL | factor_change NULL | classification NULL | analysis_method NULL | has rollout_mst_ts | badge NULL | decision NULL | has automated_conclusion |
|---|---|---|---|---|---|---|---|---|---|
| before 2024 | 376 | 376 | 374 | 350 | 374 | 0 | 29 | 44 | 19 |
| 2024 | 2,384 | 2,377 | 2,126 | 1,809 | 2,357 | 182 | 8 | 121 | 326 |
| 2025 | 2,828 | 2,812 | 422 | 421 | 138 | 384 | 2 | 205 | 282 |
| 2026 | 3,518 | 1,484 | 9 | 13 | 0 | 1,872 | 0 | 568 | 969 |
| never bucketed | 2,356 | 2,108 | 1,269 | 1,276 | 2,355 | 40 | 1,026 | 1,351 | 1 |

## Column index

Null % is over the full table (25,109 rows). Confidence = confidence in the stated meaning.

| Column | Type | Null % | Distinct | One-line meaning | Confidence |
|---|---|---|---|---|---|
| `experiment_id` | string | 0.00 | 25,109 | Hivemind experiment id (PK, case-sensitive) | Verified |
| `actual_sample_cnt` | bigint | 0.00 | 11,355 | Users bucketed (sum of cohort sample counts in latest report); PRE-POST = days | Verified |
| `additional_deciding_metrics_id_list` | string | 80.78 | 94 | Free-text extra deciding metrics; legacy, junk-laden | Unknown |
| `alpha_value` | real | 16.96 | 9 | Configured significance level | Verified |
| `analysis_method_enum` | string | 49.61 | 2 | fixed / sequential method used | Verified |
| `analysis_type` | string | 16.85 | 5 | frequentist / bayesian | Verified |
| `automated_conclusion_enum` | string | 93.29 | 3 | Hivemind automatic conclusion WIN/LOSS/INCONCLUSIVE | Verified |
| `automated_conclusion_mst_ts` | timestamp | 93.29 | 1,684 | When the automatic conclusion was set | Inferred |
| `annualization_type_enum` | string | 97.55 | 6 | iGCR surface bucket | Verified |
| `annualization_trigger_metric_type_enum` | string | 99.98 | 2 | GCR / CVR type of the metric that triggered annualization | Inferred |
| `annualization_value` | real | 97.55 | 615 | Governed annualized iGCR, USD, discount applied | Verified |
| `annualization_value_prorated` | real | 100.00 | 0 | Always NULL - unused | Inferred |
| `annualized_metric_definition_id` | string | 97.70 | 45 | Metric that was annualized | Verified |
| `badge_name` | string | 54.57 | 4 | Rigor badge Platinum/Gold/Silver/Bronze | Verified |
| `badging_mst_ts` | timestamp | 54.57 | 11,402 | When the badge was computed | Inferred |
| `unmet_badging_criteria_list` | string | 56.73 | 15 | Badge criteria not met | Verified |
| `baseline_cohort_id` | string | 12.31 | 2,285 | Control cohort id | Verified |
| `business_unit_name` | string | 35.55 | 40 | Owning BU (free text, drift) | Verified |
| `category_enum` | string | 17.26 | 5 | onsite / offsite | Verified |
| `cohort_name_list` | string | 0.00 | 3,122 | Comma list of cohort names | Verified |
| `cross_bucket_cnt` | bigint | 31.28 | 1,112 | Users bucketed into more than one cohort | Verified |
| `customer_problem_text` | string | 43.11 | 7,223 | Customer problem statement | Verified |
| `deciding_cohort_id_text` | string | 35.41 | 1,584 | Treatment chosen for decision / annualization ('ineligible' = not iGCR-eligible) | Verified |
| `deciding_metrics_category_list` | string | 12.51 | 230 | Category of each deciding metric (parallel list) | Verified |
| `deciding_metrics_id_list` | string | 12.51 | 2,725 | Deciding metrics (scorecard/analysis ids) | Verified |
| `decision_metric_id_list` | string | 25.25 | 2,243 | Configured decision metrics | Verified |
| `duration_day_value` | real | 33.15 | 4,379 | Bucketing span in days (canonical duration) | Verified |
| `end_mst_date` | date | 65.00 | 970 | Configured end date | Verified |
| `ended_at_mst_ts` | timestamp | 9.64 | 22,661 | Time of last end/update action (UTC-like, backfill-corrupted) - never use | Inferred |
| `even_split_flag` | boolean | 0.00 | 2 | Current config cohort splits are all equal | Verified |
| `experiment_name` | string | 34.82 | 14,904 | Display name | Verified |
| `experiment_owner_id_list` | string | 9.99 | 3,839 | Comma list of owner usernames | Verified |
| `experiment_type_enum` | string | 15.23 | 13 | Experiment methodology type | Verified |
| `factor_change_enum` | string | 59.93 | 15 | Kind of change tested | Verified |
| `experiment_classification_enum` | string | 62.24 | 19 | Strategic classification (with legacy spellings) | Verified |
| `rtb_initiative_name` | string | 99.60 | 8 | Legacy RTB initiative name | Verified |
| `first_user_bucket_mst_ts` | timestamp | 32.98 | 11,596 | Real start: first user bucketed (MST) | Verified |
| `first_report_generated_mst_ts` | timestamp | 100.00 | 0 | Always NULL - unused | Inferred |
| `fixed_design_flag` | boolean | 41.50 | 2 | API isFixedDesign; TRUE even on sequential tests - do not use | Verified |
| `guardrail_metric_id_list` | string | 31.55 | 1,168 | Configured guardrail metrics | Verified |
| `hypothesis_change_text` | string | 17.53 | 13,840 | Hypothesis: what changes | Verified |
| `hypothesis_expectation_text` | string | 17.80 | 7,721 | Hypothesis: expected outcome | Verified |
| `hypothesis_reason_text` | string | 18.12 | 7,949 | Hypothesis: why | Verified |
| `informative_metrics_list` | string | 100.00 | 0 | Always NULL - unused | Inferred |
| `insight_tool` | string | 52.70 | 9 | Source of the insight behind the test | Verified |
| `insight_tool_description` | string | 93.85 | 906 | Free-text detail on the insight source | Verified |
| `annualized_metric_significance_flag` | boolean | 97.57 | 1 | TRUE or NULL only - no information | Verified |
| `killed_flag` | boolean | 21.60 | 2 | API `killed` flag | Verified |
| `label_list` | string | 23.07 | 1,197 | Comma list of config labels | Verified |
| `last_user_bucket_mst_ts` | timestamp | 32.98 | 11,718 | Real end: last user bucketed (MST) | Verified |
| `observation_text` | string | 38.55 | 7,470 | Observation that motivated the test | Verified |
| `opportunities_text` | string | 93.06 | 1,525 | Free-text opportunities / follow-ups | Inferred |
| `prior_experiments_list` | string | 87.58 | 2,272 | Prior related experiment ids | Verified |
| `target_customer_text` | string | 84.44 | 2,022 | Free-text target customer | Verified |
| `released_flag` | boolean | 33.88 | 2 | API `released` flag (not "treatment shipped") | Verified |
| `rollout_decision_enum` | string | 34.42 | 6 | THE declared result (human decision) | Verified |
| `rollout_mst_ts` | timestamp | 65.17 | 7,203 | Rollout date - stored as UTC despite the name | Verified |
| `scorecard_template_name` | string | 18.48 | 1,384 | Scorecard template id | Verified |
| `sig_guardrail_metric_id_list` | string | 0.00 | 227 | Guardrails that moved significantly | Verified |
| `sig_lift_informative_metric_id_list` | string | 0.00 | 2,692 | Informative metrics with significant lift | Inferred |
| `sig_loss_informative_metric_id_list` | string | 0.00 | 2,749 | Informative metrics with significant loss | Inferred |
| `sig_lift_decision_metric_id_list` | string | 0.00 | 744 | Decision metrics with significant lift | Verified |
| `sig_loss_decision_metric_id_list` | string | 0.00 | 442 | Decision metrics with significant loss | Verified |
| `srm_flag` | boolean | 0.00 | 2 | SRM detected in the latest report | Verified |
| `start_mst_date` | date | 22.82 | 1,210 | Configured start date | Verified |
| `status_enum` | string | 74.36 | 7 | Lifecycle status (sparse before 2026) | Verified |
| `target_sample_cnt` | bigint | 80.13 | 3,420 | Power-calculator sample size per cohort | Verified |
| `team_name` | string | 18.61 | 928 | Owning team(s), free text | Verified |
| `traffic_allocation_value` | integer | 9.78 | 24 | Current % of traffic allocated | Verified |
| `traffic_type_enum` | string | 9.67 | 9 | Bucketing identifier type | Verified |
| `trigger_eid_text` | string | 85.83 | 2,259 | Trigger EID / regex | Verified |
| `trigger_property_name_text` | string | 99.03 | 81 | Trigger property name filter | Inferred |
| `trigger_property_value_text` | string | 99.03 | 119 | Trigger property value / regex filter | Inferred |
| `cln_create_utc_ts` | timestamp | 100.00 | 0 | Always NULL - unused | Inferred |

## Cross-table value map

Values in this table that join to, or must be compared with, values in the other `hivemind_mart` tables.

| This table | Other table.column | Relationship / rule |
|---|---|---|
| `experiment_id` | `analysis_result_cln.experiment_id`, `user_allocation_history.experiment_id`, `experiment_order.experiment_id` | Join key, exact case, one-to-many |
| `baseline_cohort_id` | `analysis_result_cln.control_cohort` | Same value (e.g. `off`, `pre`, `control`, `control_variant`); when NULL use `control_cohort` |
| `baseline_cohort_id`, `deciding_cohort_id_text`, items of `cohort_name_list` | `cohort_id` in `analysis_result_cln`, `user_allocation_history`, `experiment_order` | Same cohort names (e.g. `on`/`off`, `control_variant`/`b_variant`, `default`/`variant`). `deciding_cohort_id_text = 'ineligible'` never appears as a cohort |
| `scorecard_template_name` | `scorecard_template_name` in all three tables | Same id (e.g. `cm-renewals-audience-recording`, `AGI_Mon_Email_ERP`, `hp_core_metrics`, `CDE-PrePost-Default-Scorecard`) |
| `alpha_value` (0.1, 0.2, 0.01 ...) | `analysis_result_cln.alpha` (0.1, 0.2, 0.01 in sample) | Same value on scorecard rows; SRM health rows use 0.01 |
| `analysis_type` (`frequentist`, `bayesian`) | `analysis_result_cln.analysis_type` | Same vocabulary |
| `analysis_method_enum` (`fixed`, `sequential`) | `analysis_result_cln.analysis_method_used` | Same vocabulary |
| `fixed_design_flag` | `analysis_result_cln.fixed_design_flag` | Same API flag (TRUE on all sampled rows, even sequential) |
| ids in `decision_metric_id_list`, `deciding_metrics_id_list`, `guardrail_metric_id_list`, `sig_*_list`, `annualized_metric_definition_id` | `REGEXP_REPLACE(analysis_result_cln.metric_id, '_[0-9a-f]{32}$', '')` | Bare ids here; `metric_id` carries a 32-hex hash suffix. Match the scorecard on `deciding_metrics_id_list` (it carries the analysis id); `decision_metric_id_list` can carry a different configured id |
| `deciding_metrics_category_list` (`DECISION METRICS`, `INFORMATIVE METRICS`, `GUARDRAIL METRICS`) | `analysis_result_cln.metric_category` (`decision`, `informative`, `guardrail`, `OverallHealth`) | Same concept, different spelling / case |
| `traffic_type_enum` (`shopper_id`, `visitor_guid`) | `experiment_user_identifier` in `user_allocation_history` / `experiment_order` | Same values for those two id types |
| `first_user_bucket_mst_ts` / `last_user_bucket_mst_ts` | MIN / MAX `user_allocation_history.bucketed_mst_ts` | Verified doc: first = MIN bucketed_mst_ts (true MST) |
| `actual_sample_cnt` | COUNT of `user_allocation_history` rows per experiment | e.g. agi_mon_email_banner_multi_surface 686,464 both; MKT_EM_177 277,217 both; mobile_ios_contextual_nudges_v2 225,901 both |
| `rollout_mst_ts` (UTC) | `analysis_result_cln.insert_utc_ts` | Compare `insert_utc_ts - 7h` (MST run time) with `DATEADD(hour,-7,rollout_mst_ts)` |

---

## Column details

### `experiment_id` - STRING
- **Meaning:** Hivemind experiment identifier, the primary key. API: `get_experiment_by_id.id` (also `pk = '#EXP_<id>'`).
- **Profile (full table):** 25,109 rows, 0.00% null, 0.00% empty, 25,109 distinct, length 2-101.
- **Values:** N distinct (high cardinality). Examples: `abn_gdexp_4775`, `cart_use_order_shim`, `DC_7170_ChatYieldThreshold_General`, `ml_v4_m365_ee_x_year_usd_auto_x_Jul25`. Prefixes seen in data: `ff-` = feature flags (923 of 925 FEATURE-FLAG rows), `cde_app_` = machine-generated pre/post runs (5,313 of 6,534 PRE-POST rows), `abn_gdexp_<n>` = ids tied to GDEXP tickets, `MKT_EM_` / `MKT_SMS_` / `MKT_WA_` = customer-marketing email / SMS / WhatsApp sends (from examples).
- **How to use / Traps:** Match exactly (`=`), never `ILIKE` for lookups: 31 rows collide with another id by case only (e.g. `MyP_flyout_TLD_prompt` vs `myp_flyout_tld_prompt`, `serp_domain_pricing_test_123Reg` vs `serp_domain_pricing_test_123reg`, `agi-mwp-entry-to-GDMVisibilty` (A/B) vs `agi-mwp-entry-to-GDMvisibilty` (PRE-POST-STANDALONE)). When a user gives a name, not an id, search `experiment_name` / `experiment_id` and ask if several match. Exclude `ff-%` and `cde\_app\_%` from portfolio counts.
- **Confidence:** Verified (profile, SQL, MCP config).

### `actual_sample_cnt` - BIGINT
- **Meaning:** Number of bucketed users in the latest analysis report = sum of `healthMetric.summary.sampleCount` across cohorts (agi_plg_email_inline_upgrade_erp: 117,032 + 115,717 = 232,749 in both).
- **Profile (full table):** 25,109 rows, 0.00% null, 0 empty, 11,355 distinct, min 0, max 23,324,095, avg 224,594.18.
- **Values:** not complete. Top: `0` 8,281; `42` 2,573; `43` 1,455; `41` 485; `1` 57; `35` 47; `36` 44; `2` 42; `30` 40; `40` 34.
- **How to use / Traps:** `0` = never bucketed (8,280 of the 8,281 zero rows also have NULL `first_user_bucket_mst_ts`). For **PRE-POST** rows the value is a day count, not users: of 4,755 PRE-POST rows with 30-45, 4,753 equal `duration_day_value` within 1 day (SQL) - that is why 41/42/43 dominate. Verified doc: usually all cohorts but sometimes one cohort - prefer per-cohort counts from `user_allocation_history` / `analysis_result_cln`. Not additive across experiments as "users" (same user can be in many experiments).
- **Confidence:** Verified (MCP payloads, SQL).

### `additional_deciding_metrics_id_list` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-01): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Free-text list of extra deciding metrics. No equivalent field found in the MCP payloads.
- **Profile (full table):** 25,109 rows, 80.78% null, 18.75% empty, 94 distinct, length 0-908.
- **Values:** not complete. Top: NULL 20,282; `''` 4,707; `email_quality_score` 14; `EQS` 11; `email_quality_score,email_click_through_rate` 3; `gcr_new_purchases` 2; `email_click_through_rate,email_quality_score` 2. Contains non-ids such as a Slack URL, `GCR/visit`, `EQS:78.92%,DDCNewPurchases:0`, quoted lists.
- **How to use / Traps:** Do not parse as metric ids; quote as entered only. Under 0.5% of rows have a non-empty value.
- **Confidence:** Unknown - meaning not documented — owner input needed; would be resolved by the Hivemind config schema field it is sourced from.

### `alpha_value` - REAL
- **Meaning:** Configured significance level used by the frequentist tests. API: `config.alpha` (MCP: "Significance level for statistical tests").
- **Profile (full table):** 25,109 rows, 16.96% null, 9 distinct, min 0.01, max 0.4, avg 0.0971.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| 0.1 | 11,615 | 90% confidence (default for most A/B; 7,917 A/B rows) |
| 0.01 | 5,311 | 99% confidence; 5,305 of these are PRE-POST machine runs |
| NULL | 4,259 | not configured (512 in A/B population) |
| 0.2 | 3,719 | 80% confidence (3,469 A/B rows) |
| 0.4 | 137 | 60% confidence |
| 0.05 | 33 | 95% confidence |
| 0.3 | 29 | 70% confidence |
| 0.15 | 4 | 85% confidence |
| 0.16 | 1 | 84% confidence |
| 0.12 | 1 | 88% confidence |

- **How to use / Traps:** Never assume 0.05. Use the experiment's own alpha (or `analysis_result_cln.alpha` / `metric_significance_flag`). SRM uses its own 0.01 (MCP healthMetric). Badges depend on it ("has alpha <= 0.1", "has alpha <= 0.2").
- **Confidence:** Verified.

### `analysis_method_enum` - STRING
- **Meaning:** Analysis method used. API: `analysis.analysisMethodUsed`.
- **Profile (full table):** 25,109 rows, 49.61% null, 2 distinct, length 5-10.
- **Values (complete):**

| Value | Count | A/B pop | Meaning |
|---|---|---|---|
| NULL | 12,456 | 5,224 | not recorded (nearly all experiments that started before 2025 or never bucketed) |
| fixed | 10,808 | 4,663 | fixed-horizon test (T test / Z test etc.) |
| sequential | 1,845 | 1,575 | sequential test (WSPRT family in analysis_result_cln) |

- **How to use / Traps:** Use this, not `fixed_design_flag`, to tell fixed from sequential.
- **Confidence:** Verified (MCP analysis payloads).

### `analysis_type` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-11): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Statistical framework. API: `config.analysisType` / `customJsonConfig.analysisType` (MCP enum `frequentist`, `bayesian`).
- **Profile (full table):** 25,109 rows, 16.85% null, 5 distinct, length 8-24.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| frequentist | 18,886 | p-values / CIs at `alpha_value` |
| NULL | 4,232 | not configured |
| bayesian | 1,988 | Bayesian (report pbest / credible interval; no p-value) - 1,494 A/B rows |
| revenue_impact | 1 | non-standard value on test experiment `test_abn_gdexp_11286`; meaning not documented **⚠️ OWNER INPUT NEEDED (OQ-MS-11)** |
| User Interaction | 1 | non-standard value on test experiment `ml_10463_exp_pat-test-1`; meaning not documented **⚠️ OWNER INPUT NEEDED (OQ-MS-11)** |
| conversion_rate_analysis | 1 | non-standard value on `abn_gdexp_1128645645`; meaning not documented **⚠️ OWNER INPUT NEEDED (OQ-MS-11)** |
- **How to use / Traps:** If `bayesian`, do not speak of "significant / not significant". Only 614 of 1,467 bayesian A/B-population rows have an `analysis_method_enum`.
- **Confidence:** Verified (MCP enum + payloads); the 3 odd values Unknown.

### `automated_conclusion_enum` - STRING
- **Meaning:** Hivemind automatic conclusion of the experiment (UPPERCASE). MCP catalog `decisionResult` enum: "Statistical outcome of the experiment's own analysis metrics; never a lifecycle value".
- **Profile (full table):** 25,109 rows, 93.29% null, 3 distinct.
- **Values (complete):**

| Value | Count | A/B pop | Meaning |
|---|---|---|---|
| NULL | 23,425 | 9,865 | no automatic conclusion |
| INCONCLUSIVE | 1,079 | 1,013 | no significant decision-metric result |
| WIN | 414 | 401 | significant positive result |
| LOSS | 191 | 183 | significant negative result |

- **How to use / Traps:** Secondary to `rollout_decision_enum` (the human decision). In the A/B population 1,597 rows have it; on 205 of them it disagrees with a declared win/loss/inconclusive. Can disagree with the latest stats. Only A/B (plus 2 other) rows ever carry it.
- **Confidence:** Verified.

### `automated_conclusion_mst_ts` - TIMESTAMP

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-04): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** When the automatic conclusion was recorded (inferred from name; filled exactly when `automated_conclusion_enum` is filled - same 93.29% null, 0 mismatches by SQL).
- **Profile (full table):** 25,109 rows, 93.29% null, 1,684 distinct, 2023-11-08 18:56:43 to 2026-11-12 09:28:07.
- **Values:** high cardinality, microsecond timestamps (e.g. `2026-09-29 16:05:18.396734`).
- **How to use / Traps:** Timezone not verified (name says MST). Use only for "when did Hivemind auto-conclude".
- **Confidence:** Inferred (name + co-fill with the enum).

### `annualization_type_enum` - STRING
- **Meaning:** iGCR surface bucket for `annualization_value`; sets the discount factor. MCP: `annualizableMetrics` lists surfaces `site new purchase iGCR`, `customer marketing new purchase iGCR` per metric.
- **Profile (full table):** 25,109 rows, 97.55% null, 6 distinct, length 17-36.
- **Values (complete):**

| Value | Count | A/B pop | Meaning |
|---|---|---|---|
| NULL | 24,493 | 10,882 | not annualized |
| customer marketing new purchase iGCR | 319 | 312 | new-purchase iGCR from customer-marketing (email/SMS) surface; discount about 0.335 |
| site new purchase iGCR | 232 | 203 | new-purchase iGCR from site / in-product surface; discount about 0.48 |
| new purchase iGCR | 39 | 39 | legacy new-purchase label (pre surface split); usually no `annualized_metric_definition_id` |
| site renewal iGCR | 16 | 16 | renewal iGCR on site (Alation doc #439); discount about 0.48 |
| billing agent renewal iGCR (v2) | 7 | 7 | renewal iGCR from billing agent (v2); discount about 0.48 |
| product renewal iGCR | 3 | 3 | renewal iGCR from product surface |

- **How to use / Traps:** Sum `annualization_value` only within one type, or show by type; types use different discounts. 1 row has a type but no value.
- **Confidence:** Verified (verified doc, MCP).

### `annualization_trigger_metric_type_enum` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-05): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Type of metric that triggered annualization - GCR or CVR (inferred from name; all 6 filled rows are wins with `site`/`customer marketing new purchase iGCR` annualized on `new_purchase_gcr_amt`).
- **Profile (full table):** 25,109 rows, 99.98% null, 2 distinct.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| NULL | 25,103 | not recorded |
| GCR | 5 | annualization triggered by a GCR-type metric (inferred) |
| CVR | 1 | triggered by a conversion-type metric (`cvr-test-all-pass-2`, looks like a test) (inferred) |

- **How to use / Traps:** Too sparse to use. Do not infer the annualized metric from it - use `annualized_metric_definition_id`.
- **Confidence:** Inferred (name + 6 rows).

### `annualization_value` - REAL
- **Meaning:** Governed annualized incremental GCR (iGCR) in USD for the deciding cohort vs control (Alation doc #521 new purchase / #439 renewal): delta GCR per user x total sample (all cohorts) x seasonality x surface discount - discount already applied. Frozen at decision time.
- **Profile (full table):** 25,109 rows, 97.55% null, 615 distinct, min -2,024,600, max 7,536,650, avg 571,590.71.
- **Values:** high cardinality (615 non-null values, each about unique, e.g. 755575.0, 86583.7). Can be negative.
- **How to use / Traps:** Reporting filter: value NOT NULL AND `rollout_decision_enum='win'` AND `deciding_cohort_id_text <> 'ineligible'` (A/B population: 579 with a value, 505 pass). Never re-discount. Never mix with Commercial OS "annualized gcr" (undiscounted, winsorized, overall, live - e.g. agi_plg_email_inline_upgrade_erp COS annualized gcr -2,229,551 while this column is NULL). SUM only within one `annualization_type_enum`.
- **Confidence:** Verified.

### `annualization_value_prorated` - REAL

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-08): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Presumably a prorated iGCR (name); never populated.
- **Profile (full table):** 25,109 rows, 100.00% null, 0 distinct.
- **Values (complete):** NULL 25,109.
- **How to use / Traps:** Never use.
- **Confidence:** Inferred (name only; emptiness Verified).

### `annualized_metric_definition_id` - STRING
- **Meaning:** Metric id that was annualized into `annualization_value` (bare id, no hash). Must be one of the experiment's `annualizableMetrics` in the API config.
- **Profile (full table):** 25,109 rows, 97.70% null, 45 distinct, length 11-44.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| NULL | 24,531 | not annualized, or legacy row without metric id |
| new_purchase_gcr_amt | 298 | overall new-purchase GCR |
| gcr_new_purchases | 119 | overall new-purchase GCR (scorecard id variant) |
| new_domain_purchase_gcr_amt | 18 | product-level: domains |
| new_wam_gcr | 15 | product-level: WAM (Websites and Marketing) |
| new_o365_purchase_gcr_amt | 13 | product-level: Microsoft 365 |
| new_domain_name_reg_gcr | 13 | product-level: domain registration |
| new_nondom_purchase_gcr_amt | 10 | product-level: non-domain products |
| dop_lite_dop_clone_dop_standalone_order_gcr | 7 | product-level: Domain Ownership Protection orders |
| renewal_rate_receipt | 7 | renewal metric |
| new_wam_receipts | 7 | product-level: WAM receipts |
| new_airo_builder_gcr | 6 | product-level: Airo builder |
| renewal_yield_31d | 6 | renewal metric |
| new_o365_addons_purchase_gcr_amt | 5 | product-level: M365 add-ons |
| new_airo_builder_paid_plan_gcr | 4 | product-level |
| email_renewal_yield_fb_manual_1d | 3 | renewal metric |
| domain_renewal_yield_receipts | 3 | renewal metric |
| domains_renewal_yield_31d | 3 | renewal metric |
| new_ddc_order_gcr | 3 | product-level |
| New_customer_new_purchase_gcr | 2 | overall new purchase, new-acquisition orders only |
| auctions_expired_gcr | 2 | product-level: auctions |
| domain_transfer_in_gcr | 2 | product-level |
| total_new_gcr | 2 | overall new purchase |
| auctions_expired_and_third_party_expired_gcr | 2 | product-level |
| new_web_gcr | 2 | overall new purchase - Web point of purchase only |
| TrustedSite_gcr_amt | 2 | product-level |
| new_presence_builder_paid_subscription_gcr | 2 | product-level |
| new_hosting_purchase_gcr_amt | 2 | product-level |
| new_conversations_deluxe_gcr | 2 | product-level |
| new_email_purchase_gcr_amt | 2 | product-level |
| email_renewal_yield_hosting_cancel_15d | 1 | renewal metric |
| new_addons_purchase_gcr_amt | 1 | product-level |
| new_airo_plus_gcr | 1 | product-level |
| ssl_new_gcr | 1 | product-level: SSL |
| wam_renewal_yield_31d | 1 | renewal metric |
| marketing_standalone_new_paid_gcr | 1 | product-level |
| renewal_yield_post_expiry_20d | 1 | renewal metric |
| new_o365_backup_addons_purchase_gcr_amt | 1 | product-level |
| new_o365_archiving_addons_purchase_gcr_amt | 1 | product-level |
| new_ox_titan_purchase_gcr_amt | 1 | product-level |
| combined_ssl_new_gcr | 1 | product-level |
| mssl_new_gcr | 1 | product-level |
| email_renewal_yield_domain_manual_15d | 1 | renewal metric |
| o365_email_plus_gcr_amt | 1 | product-level |
| new_titan_only_version_purchase_gcr_amt | 1 | product-level |
| conversations_new_paid_gcr | 1 | product-level |

  (Overall vs product-level classification is the rule from use case experimentation-hm-whats-it-worth; per-metric product glosses beyond that rule are read from the id name.)
- **How to use / Traps:** Can be a guardrail or informative metric, not only a decision metric - always state its role by checking it against `decision_metric_id_list` / `guardrail_metric_id_list`. Case-sensitive (`New_customer_new_purchase_gcr`).
- **Confidence:** Verified.

### `badge_name` - STRING
- **Meaning:** Hivemind experiment rigor badge. Tier is set by which badging criteria are unmet (see `unmet_badging_criteria_list`).
- **Profile (full table):** 25,109 rows, 54.57% null, 4 distinct.
- **Values (complete):**

| Value | Count | A/B pop | Meaning (from SQL against unmet criteria) |
|---|---|---|---|
| NULL | 13,701 | 1,065 | no badge (non-A/B types, never-bucketed tests) |
| Silver | 3,796 | 3,472 | misses "has alpha <= 0.1" and/or "is a frequentist experiment" |
| Gold | 3,687 | 3,482 | misses "has attached experiment duration calculation" (3,404) and/or "has alpha <= 0.1" |
| Platinum | 2,747 | 2,618 | all criteria met (unmet list empty) |
| Bronze | 1,178 | 825 | misses hypothesis / decision-or-guardrail metric / alpha <= 0.2 / metric-count criteria |

- **How to use / Traps:** Order Platinum > Gold > Silver > Bronze is inferred from the growing unmet-criteria sets. A badge measures design rigor, not result.
- **Confidence:** Verified (values, criteria link by SQL); tier order Inferred.

### `badging_mst_ts` - TIMESTAMP

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-04): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** When the badge was assigned/recomputed (inferred from name; null exactly when `badge_name` is null - 0 mismatches).
- **Profile (full table):** 25,109 rows, 54.57% null, 11,402 distinct, 2023-12-07 06:45:39 to 2026-11-12 18:01:40.
- **Values:** high cardinality; millisecond timestamps (e.g. `2026-07-15 19:37:07.333`).
- **How to use / Traps:** Examples are weeks after rollout (e.g. agi_plg_email_inline_upgrade_erp badged 2026-11-03 18:06 vs rollout 2026-10-08), so it is likely a recompute time, not the end of the test. Timezone not verified.
- **Confidence:** Inferred.

### `unmet_badging_criteria_list` - STRING
- **Meaning:** Comma list of badge criteria the experiment did not meet.
- **Profile (full table):** 25,109 rows, 56.73% null, 10.88% empty, 15 distinct (14 non-empty strings + ''; NULL not counted), length 0-122.
- **Values (complete):**

| Value | Count | Badge it occurs with (SQL) | Meaning |
|---|---|---|---|
| NULL | 14,245 | NULL badge, or 544 badged rows (334 Bronze, 194 Silver, 15 Platinum, 1 Gold) | not recorded |
| has attached experiment duration calculation | 3,404 | Gold | no power/duration calculation attached |
| '' (empty) | 2,732 | Platinum | all criteria met |
| has alpha <= 0.1 | 2,454 | Silver 2,303, Gold 151 | alpha above 0.1 |
| is a frequentist experiment | 776 | Silver | Bayesian or non-frequentist |
| is a frequentist experiment,has alpha <= 0.1 | 523 | Silver | both |
| has documented hypothesis (reason, change, and expectation) | 384 | Bronze | hypothesis fields incomplete |
| has at least one decision or guardrail metric | 213 | Bronze | no decision/guardrail metric |
| has documented hypothesis (reason, change, and expectation),has at least one decision or guardrail metric | 133 | Bronze | both |
| has alpha <= 0.1,has attached experiment duration calculation | 131 | Gold | both |
| has documented hypothesis (reason, change, and expectation),has at least one decision or guardrail metric,has alpha <= 0.2 | 64 | Bronze | three criteria |
| has alpha <= 0.2 | 39 | Bronze | alpha above 0.2 |
| has documented hypothesis (reason, change, and expectation),has alpha <= 0.2 | 7 | Bronze | both |
| has at least one decision or guardrail metric,has alpha <= 0.2 | 2 | Bronze | both |
| has no more than 3 decision metrics | 1 | Bronze | more than 3 decision metrics |
| has no more than 3 guardrail metrics | 1 | Bronze | more than 3 guardrails |

- **How to use / Traps:** Distinguish NULL (not recorded) from '' (nothing unmet = Platinum).
- **Confidence:** Verified.

### `baseline_cohort_id` - STRING
- **Meaning:** Control cohort id. API: `config.baselineCohortId` / `analysis.baseline`.
- **Profile (full table):** 25,109 rows, 12.31% null, 1.51% empty, 2,285 distinct, length 0-1,909.
- **Values:** not complete. Top: `pre` 6,139; `off` 5,848; NULL 3,092; `control_variant` 2,030; `control1` 747; `A` 697; `cohort_value_false` 652; `default` 507; `''` 378; `baseline` 259; `control` 247; `on` 217; `no_call` 160; `cohort_value_true` 138; `Control` 114.
- **How to use / Traps:** Free text, case-sensitive (`control` vs `Control`). Never infer control from a name (`on` is the baseline on 217 rows). NULL on 234 A/B-population rows - then use `analysis_result_cln.control_cohort`. `deciding_cohort_id_text = baseline_cohort_id` on 1,442 A/B-population rows (control was chosen = treatment not shipped).
- **Confidence:** Verified.

### `business_unit_name` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-13): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Owning business unit. API: `config.businessUnit`. Free text.
- **Profile (full table):** 25,109 rows, 35.55% null, 0.02% empty, 40 distinct, length 0-31.
- **Values (complete):** canonical mapping from the verified doc; "-" = keep as is.

| Value | Count | A/B pop | Canonical / meaning |
|---|---|---|---|
| NULL | 8,926 | 74 | (unassigned) |
| CMO | 5,367 | 4,055 | - (expansion not documented — owner input needed) |
| Customer & Site | 3,055 | 2,534 | - |
| US Independents | 2,208 | 1,261 | - |
| CTO | 1,208 | 939 | - |
| AGI | 1,184 | 733 | - (Airo Growth & Innovation) |
| Domain Registrars and Investors | 630 | 418 | - |
| International Independents | 468 | 319 | - |
| Partners | 393 | 243 | - |
| Commerce | 391 | 277 | - |
| Care | 380 | 251 | - |
| People Ops | 303 | 45 | - |
| AGI - Investors | 182 | 117 | separate BU - do NOT merge into AGI |
| USI | 90 | 59 | -> US Independents |
| Finance | 73 | 4 | - |
| Paid Marketing | 59 | 48 | - |
| Brand | 53 | 14 | - |
| Airo Growth & Innovation | 37 | 25 | -> AGI |
| Legal | 19 | 7 | - |
| FORGE | 18 | 6 | meaning not documented — owner input needed **⚠️ OWNER INPUT NEEDED (OQ-MS-13)** |
| CPO | 12 | 5 | - |
| GDI | 11 | 9 | meaning not documented — owner input needed **⚠️ OWNER INPUT NEEDED (OQ-MS-13)** |
| '' (empty) | 5 | 4 | (unassigned) |
| Other | 5 | 3 | (unassigned) |
| FOS | 4 | 0 | meaning not documented; `fos` is a case duplicate **⚠️ OWNER INPUT NEEDED (OQ-MS-13)** |
| Marketing | 4 | 1 | meaning not documented (not mapped) |
| Midas | 4 | 2 | meaning not documented **⚠️ OWNER INPUT NEEDED (OQ-MS-13)** |
| corporate marketing | 3 | 1 | case variant of Corporate Marketing |
| Airo Growth  Innovation | 2 | 1 | -> AGI (typo, double space) |
| Corporate Marketing | 2 | 2 | - |
| N/A | 2 | 1 | (unassigned) |
| Domains | 2 | 1 | meaning not documented (not mapped) **⚠️ OWNER INPUT NEEDED (OQ-MS-13)** |
| AGI Growth | 1 | 1 | probably AGI - not in verified mapping (Inferred) |
| Domain Investors | 1 | 1 | not mapped; do not merge without confirmation **⚠️ OWNER INPUT NEEDED (OQ-MS-13)** |
| Web Services | 1 | 0 | meaning not documented |
| GoDaddy | 1 | 0 | meaning not documented |
| Customer and Site | 1 | 0 | -> Customer & Site |
| fos | 1 | 0 | case variant of FOS |
| Customer & site | 1 | 1 | -> Customer & Site |
| Domain Registration | 1 | 0 | not mapped |
| Cust & Site | 1 | 0 | -> Customer & Site |

- **How to use / Traps:** Canonicalize with a CASE before grouping. A 2025 -> 2026 reorg moved many teams into Customer & Site / AGI, so BU trends across those years are not like-for-like.
- **Confidence:** Verified (values, mapping from verified doc); unmapped values noted.

### `category_enum` - STRING
- **Meaning:** Onsite vs offsite experiment. API: `config.category` (MCP enum `onsite`, `offsite`).
- **Profile (full table):** 25,109 rows, 17.26% null, 5 distinct.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| onsite | 10,876 | on GoDaddy web/app surfaces |
| offsite | 9,892 | off-site channels; examples are email/SMS sends (`MKT_EM_*`, label `sfmc`) and PRE-POST runs (5,159 PRE-POST rows) |
| NULL | 4,334 | not set |
| Offsite | 5 | case variant of `offsite` (`MKT_EM_112_*`, `MKT_SMS_361_*`, `MKT_EM_1_*`) |
| UI/UX | 1 | non-standard, test experiment `ml_10463_exp_1756140647347` |
| User Interface | 1 | non-standard, test experiment `ml_10463_exp_pat-test-1` |

- **How to use / Traps:** Use `LOWER(category_enum)`.
- **Confidence:** Verified.

### `cohort_name_list` - STRING
- **Meaning:** Comma list of cohort names in config order (API `config.cohorts[].name`; agi_plg_email_inline_upgrade_erp `variant,default` matches).
- **Profile (full table):** 25,109 rows, 0.00% null, 10.79% empty, 3,122 distinct, length 0-4,231.
- **Values:** not complete. Top: `on,off` 6,232; `pre,post` 5,543; `''` 2,709; `control_variant,b_variant` 1,745; `cohort_value_false,cohort_value_true` 639; `post,pre` 602; `A,B` 509; `treatment1,control1` 465; `variant,default` 245; `cohort_0,cohort_1` 215.
- **How to use / Traps:** Order is not control-first. Count arms with `REGEXP_COUNT(cohort_name_list, ',') + 1` when not empty. Empty = no cohorts recorded.
- **Confidence:** Verified.

### `cross_bucket_cnt` - BIGINT
- **Meaning:** Users bucketed into more than one cohort, from the latest report. API: `healthMetric.crossBucketCount` (ISC_Legacy_Vs_AWS_DoNoHarm 1 = 1).
- **Profile (full table):** 25,109 rows, 31.28% null, 1,112 distinct, min 0, max 627,971, avg 1,115.31.
- **Values:** not complete. Top: `0` 15,284; NULL 7,854; `1` 165; `2` 74; `3` 38; `4` 32; `6` 26; `5` 26.
- **How to use / Traps:** NULL = no analysis report (never bucketed). Verified doc: does not reconcile with other sources - treat as indicative. A/B population: 601 rows > 0, of which 176 also have `srm_flag` TRUE.
- **Confidence:** Verified (mapping); reconciliation caveat from verified doc.

### `customer_problem_text` - STRING
- **Meaning:** Customer problem statement. API: `config.customerProblem`.
- **Profile (full table):** 25,109 rows, 43.11% null, 0.14% empty, 7,223 distinct, length 1-3,631.
- **Values:** high cardinality. Placeholders dominate the top: `n/a` 328; `TBD` 302; `N/A` 273; `.` 244; `tbd` 241; `-` 230; `TBA` 228; `NA` 222; real text e.g. "Customers may not be aware of the availability for relevant domains to them." 127; "Help me find a domain" 100.
- **How to use / Traps:** Treat placeholders (`n/a`, `TBD`, `.`, `-`, `NA`, `None`, `test`, `See GDEXP`) as missing. Quote in full; may contain newlines and mojibake (UTF-8 decoded as Latin-1).
- **Confidence:** Verified.

### `deciding_cohort_id_text` - STRING
- **Meaning:** Treatment cohort chosen for the decision / annualization - the cohort that was shipped. API: `config.defaultTreatment` (all 17 checked match, including `ineligible`).
- **Profile (full table):** 25,109 rows, 35.41% null, 1,584 distinct, length 1-64.
- **Values:** not complete. Top: NULL 8,890; `post` 5,979 (PRE-POST runs); `ineligible` 5,709; `on` 1,132; `off` 399; `b_variant` 338; `control_variant` 205; `variant` 85; `call` 68; `treatment1` 61.
- **How to use / Traps:** `ineligible` = not eligible for iGCR - exclude from iGCR sums, keep in the experiment population (A/B population: 583 killed, 879 wins, 2,450 losses/inconclusive carry it; 49 carry an annualization_value). Equal to `baseline_cohort_id` = control was kept. 5 A/B-population rows have a value not found in `cohort_name_list`.
- **Confidence:** Verified.

### `deciding_metrics_category_list` - STRING
- **Meaning:** Category of each entry in `deciding_metrics_id_list`, same order. API: `config.decidingMetrics[].category`.
- **Profile (full table):** 25,109 rows, 12.51% null, 57.41% empty, 230 distinct, length 0-556.
- **Values:** not complete. Top: `''` 14,416; `DECISION METRICS` 3,708; NULL 3,140; `DECISION METRICS,DECISION METRICS` 756; `INFORMATIVE METRICS` 389; `decision` 365; `DECISION METRICS,INFORMATIVE METRICS` 281; `decision,decision` 214; `DECISION METRICS,GUARDRAIL METRICS` 143. Element vocabulary: `DECISION METRICS`, `INFORMATIVE METRICS`, `GUARDRAIL METRICS`, legacy lowercase `decision`.
- **How to use / Traps:** Zip by position with `deciding_metrics_id_list`. Same null/empty rate as that column.
- **Confidence:** Verified.

### `deciding_metrics_id_list` - STRING
- **Meaning:** Metrics actually used to decide (scorecard / analysis ids). API: `config.decidingMetrics[].name` (abn_gdexp_14418: `airo_builder_conversion,airo_builder_freemium_signup,...` matches).
- **Profile (full table):** 25,109 rows, 12.51% null, 57.41% empty, 2,725 distinct, length 0-833.
- **Values:** not complete. Top: `''` 14,416; NULL 3,140; `new_purchase_conversion` 780; `new_purchase_gcr_amt` 536; `new_purchase_conversion,new_purchase_gcr_amt` 322; `email_resource_renewal_rate` 151; `gcr_amt` 129; `email_resource_bill_date_renewal_rate` 127; `gcr_new_purchases` 113.
- **How to use / Traps:** A different list from `decision_metric_id_list`: it carries the analysis id (e.g. `gcr_new_purchases`) where the configured list can carry `new_purchase_gcr_amt`; match `analysis_result_cln` on this one. It can include INFORMATIVE entries - filter by the parallel category list. Order may differ from the configured list (ISC: `conversion,gcr_amt` vs `gcr_amt,conversion`).
- **Confidence:** Verified.

### `decision_metric_id_list` - STRING
- **Meaning:** Configured decision metrics. API: `config.decisionMetrics`.
- **Profile (full table):** 25,109 rows, 25.25% null, 23.69% empty, 2,243 distinct, length 0-177.
- **Values:** not complete. Top: NULL 6,341; `''` 5,949; `new_purchase_conversion` 1,693; `new_purchase_gcr_amt` 1,317; `gcr_amt_corporate` 922; `new_purchase_conversion,new_purchase_gcr_amt` 816; `email_resource_renewal_rate` 341; `email_resource_bill_date_renewal_rate` 235; `gcr_amt` 222.
- **How to use / Traps:** Bare ids, may repeat - dedupe. NULL or empty on 1,130 A/B-population rows (do-no-harm designs decide on guardrails). `gcr_amt_corporate` (922) appears with FEATURE-FLAG / PRE-POST style rows. When empty, say there were no decision metrics; do not substitute informative metrics.
- **Confidence:** Verified.

### `duration_day_value` - REAL
- **Meaning:** Bucketing span in days - the canonical "how long it ran". API: `healthMetric.duration` (41.39, 69.71, 0.01 ... match).
- **Profile (full table):** 25,109 rows, 33.15% null, 4,379 distinct, min 0.01, max 126.61, avg 30.39.
- **Values:** not complete. Top: NULL 8,323; `42.0` 2,596; `43.0` 1,471; `41.0` 549; `1.0` 450; `0.01` 180; `69.0` 125.
- **How to use / Traps:** Equals `last_user_bucket_mst_ts - first_user_bucket_mst_ts` to within 0.1 day on 100% of the 8,724 A/B rows that have both timestamps (SQL, 2026-11-13); use it as the canonical run length. `0.01` = single-send (e.g. an email blast bucketed in minutes). The 41/42/43 spikes are PRE-POST runs. NULL = no report.
- **Confidence:** Verified.

### `end_mst_date` - DATE
- **Meaning:** Configured (planned) end date. API: `config.endDate`.
- **Profile (full table):** 25,109 rows, 65.00% null, 970 distinct, 2022-10-03 to 2040-07-01.
- **Values:** not complete. Top: NULL 16,322; `2026-10-07` 88; `2026-06-30` 81; `2026-10-08` 73; `2026-04-01` 62.
- **How to use / Traps:** Filled on only 2,080 of 11,462 A/B-population rows. Never use for duration or "completed in period" - use `last_user_bucket_mst_ts`. Values can be before the start (abn_gdexp_14418: end 2026-09-22, start 2026-09-23) or far future (2040). One checked row (abn_gdexp_14418) has a value here while the current API endDate is '' - it can be stale.
- **Confidence:** Verified.

### `ended_at_mst_ts` - TIMESTAMP

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-02): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Timestamp of the last end / status-change update to the config. In the 17 checked experiments it equals the API `lastUpdateTime` (as UTC) or lands seconds after the rollout action; on 900 A/B-population rows it is within 60 s of `rollout_mst_ts` (which is UTC). A cluster at 2026-11-03 21:57-21:58 appears on several experiments regardless of their real end (bulk backfill).
- **Profile (full table):** 25,109 rows, 9.64% null, 22,661 distinct, 2022-10-17 19:47:49 to 2026-11-12 17:58:35.
- **Values:** high cardinality, millisecond timestamps.
- **How to use / Traps:** Broken - never use for end, duration or completion (verified doc: more than 2 days after the last bucket on 66% of A/B rows that have both). Despite the name it appears to be UTC.
- **Confidence:** Inferred (from 17 payload comparisons + SQL; exact source field not documented).

### `even_split_flag` - BOOLEAN
- **Meaning:** Whether the current config gives all cohorts the same split. Derived from `config.cohorts[].split` (killed 25/25/25/25 -> TRUE; rolled out 0/100 -> FALSE).
- **Profile (full table):** 25,109 rows, 0.00% null, 2 distinct.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| FALSE | 12,905 | splits unequal now (typically 0/100 after rollout or rollback) |
| TRUE | 12,204 | splits equal now (still running, killed, never launched) |

  NULL never occurs.
- **How to use / Traps:** Reflects post-rollout config, not the design during the test (0 of 734 rolled-out A/B rows are TRUE). Do not use to judge test design; use `analysis_result_cln.expected_proportion` or healthMetric `expectedProportion`.
- **Confidence:** Verified.

### `experiment_name` - STRING
- **Meaning:** Display name. API: `config.name`.
- **Profile (full table):** 25,109 rows, 34.82% null, 0.82% empty, 14,904 distinct, length 0-563.
- **Values:** high cardinality. Top: NULL 8,743; `''` 206; "Promo Pricing by experiment" 24; `test` 9; `Test` 7; "E-Ranker is optimized towards revenue..." 6.
- **How to use / Traps:** Not unique; NULL on 5,359 PRE-POST and 685 A/B rows. Contains mojibake. Use for fuzzy search, then confirm with `experiment_id`.
- **Confidence:** Verified.

### `experiment_owner_id_list` - STRING
- **Meaning:** Comma list of owner usernames (GoDaddy short ids). API: `config.owners`.
- **Profile (full table):** 25,109 rows, 9.99% null, 1.43% empty, 3,839 distinct, length 0-210.
- **Values:** high cardinality. Top: NULL 2,508; `cde` 1,930; `cpappala` 480; `agelinas` 393; `kcheung` 379; `''` 360; `hivemind,cde` 323.
- **How to use / Traps:** `cde` / `hivemind` = system owners on machine-generated runs (inferred from co-occurrence with PRE-POST). Match a person with `',' || experiment_owner_id_list || ',' LIKE '%,<user>,%'`. Lowercase usernames, but compare case-insensitively.
- **Confidence:** Verified (mapping); system-owner reading Inferred.

### `experiment_type_enum` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-09): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Experiment methodology. API: `config.experimentType` (MCP enum: A/A, A/B, MAB, PRE-POST, GEO-BASED, DID, PLATFORM-LEARNING, BENCHMARK, FEATURE-FLAG, OTHER, ENDGAME).
- **Profile (full table):** 25,109 rows, 15.23% null, 13 distinct.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| A/B | 12,142 | randomized A/B test - the only type in portfolio questions |
| PRE-POST | 6,534 | pre/post comparison; 5,313 are `cde_app_` machine runs (alpha 0.01, `CDE-PrePost-Default-Scorecard`, cohorts pre/post, sample = days) |
| NULL | 3,823 | not set (older / incomplete configs) |
| FEATURE-FLAG | 925 | feature flag created from switchboard (`ff-` ids, hypothesis "feature flag") |
| A/A | 585 | A/A validation test |
| BENCHMARK | 238 | benchmark; all 238 never bucketed |
| MAB | 204 | multi-armed bandit (MCP: epsilon-greedy / thompson-sampling) |
| PRE-POST-STANDALONE | 169 | not in MCP enum; all 169 never bucketed; meaning beyond name not documented **⚠️ OWNER INPUT NEEDED (OQ-MS-09)** |
| OTHER | 145 | other; all 145 never bucketed |
| ENDGAME | 144 | ENDGAME experiment (MCP: specialized business-impact / uplift analysis pipeline) |
| PLATFORM-LEARNING | 124 | platform learning; all never bucketed |
| DID | 54 | difference-in-differences; 52 never bucketed |
| GEO-BASED | 21 | geo-based test; all never bucketed |
| mab | 1 | lowercase duplicate of MAB |

- **How to use / Traps:** Portfolio = `'A/B'` only plus the mandatory filter. Treat `mab` as `MAB`.
- **Confidence:** Verified.

### `factor_change_enum` - STRING
- **Meaning:** What kind of change was tested. API: `config.factorChange` (MCP enum, 14 values).
- **Profile (full table):** 25,109 rows, 59.93% null, 15 distinct.
- **Values (complete):** meanings are the MCP enum names; no longer descriptions exist in sources.

| Value | Count | A/B pop | Meaning |
|---|---|---|---|
| NULL | 15,048 | 4,200 | not set (near-empty before 2024-Q4) |
| other | 3,144 | 1,601 | other |
| compellingDesign | 1,647 | 1,322 | visual design change |
| compellingCopy | 1,424 | 1,145 | copy / messaging change |
| personalization | 994 | 874 | personalization |
| buttonUIOrCTA | 760 | 586 | button UI or call-to-action |
| navigation | 527 | 433 | navigation |
| marketingCampaign | 452 | 391 | marketing campaign |
| subscriptions | 331 | 263 | subscriptions |
| promotions | 279 | 236 | promotions |
| domainSearchResult | 194 | 172 | domain search result |
| priceTransparency | 161 | 126 | price transparency |
| leadGeneration | 86 | 64 | lead generation |
| socialProofing | 50 | 43 | social proofing |
| buyNowAndPayLater | 7 | 6 | buy now and pay later |
| Hero Section Layout | 5 | 0 | non-enum free text (not in A/B population) |

- **Confidence:** Verified.

### `experiment_classification_enum` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-12): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Strategic classification / pillar. API: `config.classification` (MCP enum of 11 + legacy `RTB Initiative`).
- **Profile (full table):** 25,109 rows, 62.24% null, 19 distinct.
- **Values (complete):**

| Value | Count | A/B pop | Meaning / canonical |
|---|---|---|---|
| NULL | 15,629 | 3,869 | not set |
| Seamless Experience: Site & Renewals | 4,599 | 4,085 | current enum |
| Other | 1,856 | 1,283 | current enum |
| Site & Renewals | 1,016 | 794 | legacy -> Seamless Experience: Site & Renewals (Inferred: same suffix) |
| Airo & Attach | 608 | 403 | current enum (sub-field airoAttachInitiative not in this table) |
| Pricing & Bundling | 385 | 317 | current enum |
| Commerce | 314 | 207 | current enum |
| Airo | 294 | 198 | legacy; probably -> Airo & Attach (Inferred) **⚠️ OWNER INPUT NEEDED (OQ-MS-12)** |
| RTB Initiative | 70 | 50 | legacy, update-only in API; see `rtb_initiative_name` |
| Seamless Experience: Payments & Billing Agent | 64 | 56 | current enum |
| AI Transformation | 61 | 37 | current enum |
| Payments & Billing Agent | 50 | 50 | legacy -> Seamless Experience: Payments & Billing Agent (Inferred) |
| Strategic Partnerships | 46 | 28 | current enum |
| One Platform | 41 | 34 | current enum |
| Protect my Business | 27 | 20 | legacy, not in enum; meaning not documented **⚠️ OWNER INPUT NEEDED (OQ-MS-12)** |
| Cost Optimization | 21 | 14 | current enum |
| Big Swing | 11 | 7 | legacy, not in enum; meaning not documented **⚠️ OWNER INPUT NEEDED (OQ-MS-12)** |
| One System | 10 | 7 | legacy, not in enum; meaning not documented (possibly One Platform - unconfirmed) **⚠️ OWNER INPUT NEEDED (OQ-MS-12)** |
| Seamless Experience | 5 | 3 | legacy, ambiguous between the two Seamless Experience values **⚠️ OWNER INPUT NEEDED (OQ-MS-12)** |
| Fund AI | 2 | 0 | current enum |

- **How to use / Traps:** Spelling drift across years; map legacy values only where stated, otherwise show raw.
- **Confidence:** Verified (enum); legacy mappings Inferred.

### `rtb_initiative_name` - STRING
- **Meaning:** Name of the legacy RTB initiative. API: `rtbInitiativeName` (MCP: only for legacy RTB Initiative experiments; new experiments cannot use it).
- **Profile (full table):** 25,109 rows, 99.60% null, 8 distinct.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| NULL | 25,009 | not an RTB initiative |
| 2025 Independents | 58 | 2025 RTB initiative name |
| 2025 MWP Growth Target | 20 | 2025 RTB initiative name |
| 2025 BU Mktg Experimentation | 10 | 2025 RTB initiative name |
| 2025 SSL Growth Target | 5 | 2025 RTB initiative name |
| 2025 DIFY MRR Model & Increase Leads | 2 | 2025 RTB initiative name |
| 2025 DIFY MWP Growth | 2 | 2025 RTB initiative name |
| 2025 Backup and Archiving Add-ons | 2 | 2025 RTB initiative name |
| 2025 Refund Opportunity | 1 | 2025 RTB initiative name |

- **How to use / Traps:** In the A/B population 80 rows have a name but only 50 have classification `RTB Initiative` - the name can outlive a reclassification.
- **Confidence:** Verified.

### `first_user_bucket_mst_ts` - TIMESTAMP
- **Meaning:** Real start - first user bucketed, true MST. API: `healthMetric.startTime`; equals MIN `user_allocation_history.bucketed_mst_ts`.
- **Profile (full table):** 25,109 rows, 32.98% null, 11,596 distinct, 2022-10-11 14:08:36 to 2026-11-11 12:54:21.
- **Values:** high cardinality. Top: NULL 8,280; `2025-01-31 18:58:00` 125 (bulk value); midnight values like `2026-08-26 00:00:00` 75 (date-only starts).
- **How to use / Traps:** Prefer over `start_mst_date`. NULL = never bucketed (2,356 of the A/B population; about 4% of decided rows per verified doc; 0% of 2026 completions). Exactly-midnight values are date-grain starts.
- **Confidence:** Verified.

### `first_report_generated_mst_ts` - TIMESTAMP

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-08): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Presumably the time of the first analysis report (name); never populated.
- **Profile (full table):** 25,109 rows, 100.00% null, 0 distinct.
- **Values (complete):** NULL 25,109.
- **How to use / Traps:** Never use; first report date comes from MIN `analysis_result_cln.report_date`.
- **Confidence:** Inferred.

### `fixed_design_flag` - BOOLEAN
- **Meaning:** API `isFixedDesign` ("true = fixed sample size, false = adaptive/sequential").
- **Profile (full table):** 25,109 rows, 41.50% null, 2 distinct.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| TRUE | 14,688 | API says fixed design - but TRUE on 1,826 rows whose `analysis_method_enum` is sequential |
| NULL | 10,419 | not recorded |
| FALSE | 2 | API says adaptive/sequential |

- **How to use / Traps:** Constant in practice (the API itself returns isFixedDesign=true with analysisMethodUsed=sequential, e.g. MKT_WA_1249). Never use; use `analysis_method_enum`.
- **Confidence:** Verified.

### `guardrail_metric_id_list` - STRING
- **Meaning:** Configured guardrail metrics. API: `config.guardrailMetrics`.
- **Profile (full table):** 25,109 rows, 31.55% null, 53.46% empty, 1,168 distinct, length 0-283.
- **Values:** not complete. Top: `''` 13,423; NULL 7,923; `new_purchase_gcr_amt` 335; `email_resource_renewal_yield` 180; `new_purchase_conversion` 171; `email_resource_bill_date_renewal_yield` 85.
- **How to use / Traps:** NULL or empty on 8,070 of 11,462 A/B-population rows = "no guardrails were configured", not "no harm". Say so explicitly; informative metrics are not guardrails.
- **Confidence:** Verified.

### `hypothesis_change_text` - STRING
- **Meaning:** Hypothesis - what is changed. API: `config.hypothesis.change` (character-for-character, verified doc).
- **Profile (full table):** 25,109 rows, 17.53% null, 1.00% empty, 13,840 distinct, length 0-13,404.
- **Values:** high cardinality. Top: NULL 4,402; `feature flag` 922 (all FEATURE-FLAG); `TBD` 253; `' '` 251; `tbd` 214; `NA` 172; `N/A` 169.
- **How to use / Traps:** Quote in full (can contain newlines). Treat placeholders (`TBD`, `NA`, `-`, `.`, `' '`, `See GDEXP`, `test`) as missing. Badge criterion "has documented hypothesis" depends on these three fields.
- **Confidence:** Verified.

### `hypothesis_expectation_text` - STRING
- **Meaning:** Hypothesis - expected outcome. API: `config.hypothesis.expectation`.
- **Profile (full table):** 25,109 rows, 17.80% null, 1.00% empty, 7,721 distinct, length 0-2,012.
- **Values:** high cardinality. Top: "Measure impact of deployment on key metrics" 5,067 (all PRE-POST machine runs); NULL 4,469; `feature flag` 922; `TBD` 272; `5%` 147; `10%` 123.
- **Confidence:** Verified.

### `hypothesis_reason_text` - STRING
- **Meaning:** Hypothesis - reason. API: `config.hypothesis.reason`.
- **Profile (full table):** 25,109 rows, 18.12% null, 1.00% empty, 7,949 distinct, length 0-3,085.
- **Values:** high cardinality. Top: "Track and analyze the effects of this deployment on business metrics" 5,067 (PRE-POST template); NULL 4,551; `feature flag` 922; `TBD` 285.
- **Confidence:** Verified.

### `informative_metrics_list` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-08): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Presumably configured informative metrics (name); never populated. (MCP: informative metrics are derived as scorecard metrics not used as decision/guardrail.)
- **Profile (full table):** 25,109 rows, 100.00% null.
- **Values (complete):** NULL 25,109.
- **How to use / Traps:** Never use; informative metrics come from `analysis_result_cln.metric_category = 'informative'`.
- **Confidence:** Inferred.

### `insight_tool` - STRING
- **Meaning:** Tool/source of the insight that motivated the test. API: `config.insightTool` (MCP enum: Hivemind Experiment, Tableau Dashboard, Full Story, Customer Research, None, Other).
- **Profile (full table):** 25,109 rows, 52.70% null, 9 distinct.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| NULL | 13,232 | not set |
| None | 7,496 | explicitly no insight tool |
| Hivemind Experiment | 3,020 | insight from a prior Hivemind experiment |
| Other | 1,015 | other source |
| Full Story | 131 | FullStory session analytics |
| Tableau Dashboard | 66 | Tableau dashboard |
| Insights from Business Analytics | 57 | legacy, not in enum |
| Customer Research | 54 | customer research |
| Dashboard | 29 | legacy, not in enum |
| Tableau | 9 | legacy; probably = Tableau Dashboard (Inferred) |

- **Confidence:** Verified.

### `insight_tool_description` - STRING
- **Meaning:** Free-text detail on the insight source. API: `insightToolDescription` (MCP create_experiment parameter).
- **Profile (full table):** 25,109 rows, 93.85% null, 0.01% empty, 906 distinct, length 1-1,928.
- **Values:** high cardinality. Top: NULL 23,566; `none` 118; `tbd` 86; `TBD` 35; `Other` 29; `N/A` 29; `Search Mart` 18; Confluence / Tableau URLs.
- **Confidence:** Verified.

### `annualized_metric_significance_flag` - BOOLEAN
- **Meaning:** Whether the annualized metric was significant.
- **Profile (full table):** 25,109 rows, 97.57% null, 1 distinct.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| NULL | 24,498 | not annualized (and 4 annualized rows) |
| TRUE | 611 | annualized; set on 611 of the 615 rows with `annualization_value` |

  FALSE never occurs.
- **How to use / Traps:** Adds no information; do not report as a significance test. Significance comes from `analysis_result_cln`.
- **Confidence:** Verified.

### `killed_flag` - BOOLEAN
- **Meaning:** API `config.killed` (afternic_tdfs_lander_bin_30979 / agi_mon_email_banner_multi_surface TRUE, both status `killed`, traffic 0).
- **Profile (full table):** 25,109 rows, 21.60% null, 2 distinct.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| TRUE | 11,870 | experiment was killed (stopped); 4,684 of 4,720 A/B-population TRUE rows have traffic 0 |
| FALSE | 7,816 | not killed |
| NULL | 5,423 | not recorded |

- **How to use / Traps:** Verified doc: does not reconcile with other fields - TRUE on 3,766 A/B rows with NULL status and on 347 with `released_flag` TRUE; 5,706 PRE-POST rows are TRUE. Not a result: killed experiments still carry win/loss decisions. Use `rollout_decision_enum` for outcome and `status_enum` (2026+) for lifecycle.
- **Confidence:** Verified (mapping); semantics caveat from verified doc.

### `label_list` - STRING
- **Meaning:** Comma list of config labels/tags. API: `config.labels`.
- **Profile (full table):** 25,109 rows, 23.07% null, 16.92% empty, 1,197 distinct, length 0-231.
- **Values:** not complete. Top: NULL 5,792; `''` 4,248; `sfmc` 2,244; `katana,wingcorepx` 871; `cart` 837; `RealtimeDataHydratorEnabled` 774; `katana,gxcoreservices` 451.
- **How to use / Traps:** Free tags; `sfmc` appears to mark Salesforce Marketing Cloud sends (Inferred from customer-marketing examples and the MCP `bucketingSdkType` value `salesforceMarketing`; not documented — owner input needed). Match with comma-delimited LIKE.
- **Confidence:** Verified.

### `last_user_bucket_mst_ts` - TIMESTAMP
- **Meaning:** Real end - last user bucketed, MST. API: `healthMetric.endTime`.
- **Profile (full table):** 25,109 rows, 32.98% null, 11,718 distinct, 2022-10-23 18:20:57 to 2026-11-11 18:07:43.
- **Values:** high cardinality. Top: NULL 8,280; `2026-11-10 00:00:00` 297 (still-running tests updated to the latest day); `2025-01-31 19:19:34` 136 (bulk value).
- **How to use / Traps:** Use for "completed in period" and real end. For a running experiment it moves each day. Same NULL set as `first_user_bucket_mst_ts`.
- **Confidence:** Verified.

### `observation_text` - STRING
- **Meaning:** Observation that motivated the experiment. API: `config.observation`.
- **Profile (full table):** 25,109 rows, 38.55% null, 0.49% empty, 7,470 distinct, length 0-3,455.
- **Values:** high cardinality. Top: NULL 9,680; `NA` 933; "Created automatically from switchboard" 923 (all FEATURE-FLAG); `N/A` 857; `n/a` 449; `TBD` 310.
- **Confidence:** Verified.

### `opportunities_text` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-03): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Free-text opportunities / next tests. Matches API `config.opportunities` where that field exists (MKT_WA_1249: "Retest alternative messaging"), but other rows carry text not found in the current config (mobile_ios_contextual_nudges_v2, productivity_email_users_page, serp_precheck_m365_..., afternic_tdfs_lander_bin_30979 with a Hivemind URL) - source for those not identified.
- **Profile (full table):** 25,109 rows, 93.06% null, 0 empty, 1,525 distinct, length 1-1,355.
- **Values:** high cardinality. Top: NULL 23,366; "Kill Experiment" 29; "Test in other regions" 12; "Test alternative messaging" 10; "Test in other channels" 10; `asdf` 9.
- **How to use / Traps:** Quote only; do not treat as a decision.
- **Confidence:** Inferred.

### `prior_experiments_list` - STRING
- **Meaning:** Prior related experiment ids. API: `config.priorExperiments`.
- **Profile (full table):** 25,109 rows, 87.58% null, 0.20% empty, 2,272 distinct, length 0-476.
- **Values:** high cardinality. Top: NULL 21,990; `''` 49; `-gdp_onboarding_find_or_create_backward_compatibility` 11; `www-find-ai-domain-search-pgen` 10.
- **How to use / Traps:** Ids are free text (GDEXP keys or experiment ids, sometimes with a leading `-`); join to `experiment_id` only after trimming and exact-case match.
- **Confidence:** Verified.

### `target_customer_text` - STRING
- **Meaning:** Target customer description. API: `config.targetCustomer`.
- **Profile (full table):** 25,109 rows, 84.44% null, 1.96% empty, 2,022 distinct, length 0-659.
- **Values:** high cardinality. Top: NULL 21,202; `''` 491; `US` 145; `Global` 130; `Global/All Devices` 73; `All Markets` 68.
- **How to use / Traps:** Free text; API '' can surface as NULL.
- **Confidence:** Verified.

### `released_flag` - BOOLEAN
- **Meaning:** API `config.released`. TRUE on rolled-out AND rolled-back experiments in the checked payloads.
- **Profile (full table):** 25,109 rows, 33.88% null, 2 distinct.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| FALSE | 10,845 | not released (killed / never ended) |
| NULL | 8,508 | not recorded |
| TRUE | 5,756 | experiment was ended via release (rollout or rollback) |

- **How to use / Traps:** Does NOT mean the treatment shipped - rolled-back losses are TRUE (agi_plg_email_inline_upgrade_erp, abn_gdexp_14418). What shipped = `deciding_cohort_id_text` vs `baseline_cohort_id`.
- **Confidence:** Verified (mapping + examples); "ended via release" wording Inferred.

### `rollout_decision_enum` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-14): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** THE declared result - the human decision recorded in Hivemind (lowercase). API: `config.result` (all 17 checked match). "Did it win" means this column.
- **Profile (full table):** 25,109 rows, 34.42% null, 0.13% empty, 6 distinct.
- **Values (complete):**

| Value | Count | A/B pop | Meaning |
|---|---|---|---|
| inconclusive | 9,102 | 3,647 | declared no clear winner (4,948 are PRE-POST) |
| NULL | 8,643 | 2,289 | undecided / no decision recorded |
| win | 4,034 | 2,810 | declared win |
| issue with experiment | 1,811 | 1,495 | declared invalid (setup/data problem) - not a result **⚠️ OWNER INPUT NEEDED (OQ-MS-14)** |
| loss | 1,456 | 1,189 | declared loss |
| '' (empty) | 33 | 32 | blank; treat as undecided |
| killed by switchboard automatically | 30 | 0 | system value, all 30 are FEATURE-FLAG rows |

- **How to use / Traps:** Can be set mid-flight. Report `automated_conclusion_enum` only as a secondary line. Win rate denominators must state whether `issue with experiment` and NULL are included. iGCR only on `win`.
- **Confidence:** Verified.

### `rollout_mst_ts` - TIMESTAMP
- **Meaning:** Rollout date/time - the API `config.rolloutDate` epoch stored as a **UTC** instant despite the name (exact match on 10+ experiments).
- **Profile (full table):** 25,109 rows, 65.17% null, 7,203 distinct, 2014-04-15 00:00:00 to 2026-11-12 17:27:46.
- **Values:** high cardinality. Top: NULL 16,363; then midnight dates like `2026-06-15 00:00:00` 16, `2026-05-19 00:00:00` 15.
- **How to use / Traps:** Always `DATEADD(hour, -7, rollout_mst_ts)` for MST (MST is fixed UTC-7). Exactly 00:00:00 UTC = a date-only rollout = 17:00 MST the previous day. A/B population: 2,478 rows have it (21.6%); 925 of those are exactly midnight (SQL; the verified doc's "~8%" figure is not reproduced by this SQL); NULL for most wins before 2026-04; 10 are before the real start (e.g. serp_sherpa_display_when_taken... rollout 2026-04-25 vs start 2026-09-25 - matches the API). 2014 minimum is a data-entry artifact.
- **Confidence:** Verified.

### `scorecard_template_name` - STRING
- **Meaning:** Scorecard template id that defines the metrics analyzed. API: `config.scorecardTemplate` / `analysis.scorecardTemplateName`.
- **Profile (full table):** 25,109 rows, 18.48% null, 0.40% empty, 1,384 distinct, length 0-57.
- **Values:** not complete. Top: `CDE-PrePost-Default-Scorecard` 5,299 (PRE-POST); NULL 4,640; `feature-flag-shopper-scorecard` 901; `cart` 833; `cm-renewals-audience-recording` 722; `dpp-e2e` 704; `hp_core_metrics` 395.
- **How to use / Traps:** Case-sensitive id; joins to `scorecard_template_name` in the other mart tables and to `get_scorecard_template_by_id`.
- **Confidence:** Verified.

### `sig_guardrail_metric_id_list` - STRING
- **Meaning:** Guardrail metrics that moved significantly (Hivemind snapshot).
- **Profile (full table):** 25,109 rows, 0.00% null, 98.47% empty, 227 distinct, length 0-791.
- **Values:** not complete. Top: `''` 24,724; `new_purchase_conversion` 22; `resource_renewal_receipts_per_expiry_year_net_of_refund` 16; `new_purchase_gcr_amt` 15; `updated_added_payment` 11.
- **How to use / Traps:** Empty = none significant OR none configured OR not analyzed - check `guardrail_metric_id_list` first. Direction (harm vs improvement) is not in this column; get it from `analysis_result_cln`. Snapshot can lag or lead the latest report.
- **Confidence:** Verified (verified doc).

### `sig_lift_informative_metric_id_list` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-06): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Informative metrics with a significant lift (inferred by analogy with the decision-metric columns).
- **Profile (full table):** 25,109 rows, 0.00% null, 80.66% empty, 2,692 distinct, length 0-4,974.
- **Values:** not complete. Top: `''` 20,252; `new_purchase_gcr_per_session,new_purchase_session_rate,transaction_session_rate,free_trial_session_rate,saw_cart_session_rate` 214; `renewal_gcr_per_session,renewal_session_rate` 121; `marketing_actions` 100; `gcr_amt` 63.
- **How to use / Traps:** Informative only - never a verdict. Use `analysis_result_cln` for informative significance.
- **Confidence:** Inferred (name).

### `sig_loss_informative_metric_id_list` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-06): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Informative metrics with a significant loss (inferred).
- **Profile (full table):** 25,109 rows, 0.00% null, 75.80% empty, 2,749 distinct, length 0-4,076.
- **Values:** not complete. Top: `''` 19,032; a long `new_purchase_gcr_per_session,renewal_gcr_per_session,...` list 410; `free_trial_session_rate` 281; `gcr_amt,converted_cnt` 212; `converted_cnt` 194.
- **Confidence:** Inferred (name).

### `sig_lift_decision_metric_id_list` - STRING
- **Meaning:** Decision metrics with a significant lift.
- **Profile (full table):** 25,109 rows, 0.00% null, 91.91% empty, 744 distinct, length 0-209.
- **Values:** not complete. Top: `''` 23,077; `new_purchase_gcr_amt` 235; `new_purchase_conversion` 183; `gcr_new_purchases` 85; `wam_conversion` 32.
- **How to use / Traps:** Bare ids, may repeat - dedupe. Hivemind snapshot; can lag or lead the latest report.
- **Confidence:** Verified (verified doc).

### `sig_loss_decision_metric_id_list` - STRING
- **Meaning:** Decision metrics with a significant loss.
- **Profile (full table):** 25,109 rows, 0.00% null, 95.52% empty, 442 distinct, length 0-503.
- **Values:** not complete. Top: `''` 23,983; `new_purchase_conversion` 126; `new_purchase_gcr_amt` 103; `gcr_amt_corporate` 49; `gcr_new_purchases` 36; repeats like `new_purchase_conversion,new_purchase_conversion` 12.
- **How to use / Traps:** Dedupe repeated ids.
- **Confidence:** Verified.

### `srm_flag` - BOOLEAN
- **Meaning:** Sample ratio mismatch detected in the LATEST report. API: `healthMetric.statisticalAnalysis.isSignificant` (SRM test, alpha 0.01, cohortComparison 'Overall'; mobile_ios_contextual_nudges_v2 and MKT_WA_1249 TRUE with warning "Unhealthy experiment detected: sample ratio mismatch").
- **Profile (full table):** 25,109 rows, 0.00% null, 2 distinct.
- **Values (complete):**

| Value | Count | Meaning |
|---|---|---|
| FALSE | 24,109 | no SRM in latest report, or no report |
| TRUE | 1,000 | SRM detected (728 A/B) |

- **How to use / Traps:** False positive when the latest report is a post-rollout rerun: mobile_ios_contextual_nudges_v2 latest report expects off 0 / on 1 after rollout. Recheck SRM from the pre-rollout `analysis_result_cln` OverallHealth rows. FALSE also on never-bucketed rows (no test ran).
- **Confidence:** Verified.

### `start_mst_date` - DATE
- **Meaning:** Configured start date. API: `config.startDate`.
- **Profile (full table):** 25,109 rows, 22.82% null, 1,210 distinct, 2021-07-16 to 2029-12-19.
- **Values:** not complete. Top: NULL 5,729; `2026-08-26` 94; `2026-02-18` 90; `2026-08-27` 89.
- **How to use / Traps:** May differ from the real start; a few are in the future. Show separately from `first_user_bucket_mst_ts`.
- **Confidence:** Verified.

### `status_enum` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-10): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Lifecycle status. API: `config.status` (MCP enum: draft, in progress, killed, released, rolled out, rolled back, deleted; "not the same field as decisionResult").
- **Profile (full table):** 25,109 rows, 74.36% null, 7 distinct.
- **Values (complete):**

| Value | Count | A/B pop | Meaning |
|---|---|---|---|
| NULL | 18,670 | 9,157 | not recorded (most pre-2026 experiments) |
| killed | 3,248 | 585 | killed (MCP enum name; further meaning not documented — owner input needed) |
| draft | 1,092 | excluded | draft (MCP enum name; excluded from the population per verified doc) |
| in progress | 940 | 606 | in progress / running (MCP enum name) |
| rolled out | 740 | 734 | rolled out (MCP enum name; "treatment released to all" is Inferred, not documented — owner input needed) |
| rolled back | 381 | 370 | rolled back (MCP enum name; "reverted to control" is Inferred, not documented — owner input needed) |
| released | 36 | 10 | released (distinct from rolled out; meaning beyond the enum name not documented) **⚠️ OWNER INPUT NEEDED (OQ-MS-10)** |
| deleted | 2 | excluded | deleted |

- **How to use / Traps:** Do not use for portfolio status (sparse and sometimes stale - concluded tests still `draft`). Only `draft` and `deleted` are excluded from the population.
- **Confidence:** Verified.

### `target_sample_cnt` - BIGINT
- **Meaning:** Power-calculator sample size, PER COHORT. API: `config.powerCalculatorInfo.sampleSize` (113,230; 667; 4,880; 89,155 match).
- **Profile (full table):** 25,109 rows, 80.13% null, 3,420 distinct, min 10, max 460,294,729, avg 256,871.39.
- **Values:** not complete. Top: NULL 20,120; `89155` 82; `63839` 38; `48221` 38; `122408` 28.
- **How to use / Traps:** Compare to per-cohort actual samples, not `actual_sample_cnt` (total). NULL = no power calculation attached (badge criterion). Filled on 4,396 A/B-population rows. The API also has a `systemPowerCalculatorInfo` sample size that is not in this table.
- **Confidence:** Verified.

### `team_name` - STRING
- **Meaning:** Owning team(s). API: `config.team` (array, comma-joined; `[]` -> '').
- **Profile (full table):** 25,109 rows, 18.61% null, 37.88% empty, 928 distinct, length 0-101.
- **Values:** not complete. Top: `''` 9,511; NULL 4,674; `cart` 739; `SERPents` 682; `Customer Marketing` 418; `Retention` 330; `Digital Care` 277; `Homepage` 269; `FIND` 176 / `Find` 140 (case variants).
- **How to use / Traps:** NULL or '' on 3,981 A/B-population rows (34.7%); spelling and case variants; comma lists. Show as entered; do not group or rank by it.
- **Confidence:** Verified.

### `traffic_allocation_value` - INTEGER
- **Meaning:** CURRENT % of traffic allocated to the experiment. API: `config.trafficAllocation`.
- **Profile (full table):** 25,109 rows, 9.78% null, 24 distinct, min 0, max 100, avg 31.43.
- **Values (complete):**

| Value | Count | Value | Count | Value | Count |
|---|---|---|---|---|---|
| 0 | 15,482 | 30 | 5 | 51 | 2 |
| 100 | 7,098 | 20 | 5 | 41 | 1 |
| NULL | 2,455 | 25 | 3 | 99 | 1 |
| 10 | 15 | 5 | 3 | 64 | 1 |
| 50 | 13 | 2 | 2 | 70 | 1 |
| 1 | 9 | 60 | 2 | 53 | 1 |
| | | 38 | 2 | 55 | 1 |
| | | 34 | 2 | 28 | 1 |
| | | 40 | 2 | 35 | 1 |
| | | | | 97 | 1 |

  Meaning of each value: that percent of eligible traffic is allocated now; 0 = no traffic (killed, ended or not started); NULL = not recorded.
- **How to use / Traps:** Post-decision state, not the allocation during the test (MKT_EM_177 rolled out with 0; rolled-out tests usually 100). Never use to describe how the test ran.
- **Confidence:** Verified.

### `traffic_type_enum` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-15): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Identifier used for bucketing. API: `config.trafficType` (MCP enum, 10 values).
- **Profile (full table):** 25,109 rows, 9.67% null, 9 distinct.
- **Values (complete):**

| Value | Count | A/B pop | Meaning |
|---|---|---|---|
| visitor_guid | 11,604 | 4,349 | visitor GUID (enum name; finer meaning not documented — owner input needed) |
| shopper_id | 10,950 | 7,034 | shopper id (enum name; finer meaning not documented — owner input needed) |
| NULL | 2,428 | 0 | not set |
| care_ucid_jomax_id | 38 | 12 | Care UCID / jomax id (enum name; finer meaning not documented — owner input needed) |
| care_conversation_id | 33 | 20 | Care conversation id |
| resource_id | 23 | 17 | product resource id |
| billing_cycle_id | 23 | 22 | billing cycle id |
| auction_id | 8 | 6 | auction id |
| payment_id | 1 | 1 | payment id |
| subscription_lifetime_id | 1 | 1 | subscription lifetime id |

  (`customer_id` is in the MCP enum but absent from data.) Glosses are from the enum names.
- **How to use / Traps:** Determines which id column is populated in `user_allocation_history` / `experiment_order` (`experiment_user_identifier`). Some metrics do not support the configured user type (API warning `invalid_metric_user_type`).
- **Confidence:** Verified.

### `trigger_eid_text` - STRING
- **Meaning:** Trigger event id (EID) or regex that marks a bucketed user as triggered. API: `config.triggerEvent.eid` (agi_plg_email_inline_upgrade_erp matches).
- **Profile (full table):** 25,109 rows, 85.83% null, 2,259 distinct, length 1-1,962.
- **Values:** high cardinality. Top: NULL 21,552; `gce.cart.purchase.page.impression` 36; `gce.cart.basket.page.impression` 35; pipe-separated alternatives (`find.sales.button...|...`); anchored regexes (`^commerce.ch.page_view.impression$`). Format: dot-separated event names, `|` = OR, `^...$` regex anchors.
- **How to use / Traps:** NULL = no trigger (all bucketed users analyzed). 2,320 A/B-population rows have one. Case-sensitive.
- **Confidence:** Verified.

### `trigger_property_name_text` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-07): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Name of an event property the trigger must match (inferred; filled only with `trigger_eid_text` - 0 rows without it).
- **Profile (full table):** 25,109 rows, 99.03% null, 81 distinct, length 2-27.
- **Values:** not complete. Top: NULL 24,865; `view` 30; `cohort` 21; `iframeType` 13; `context` 10; `cardId` 8; `currentTab` 8; `flowKey` 8.
- **Confidence:** Inferred (name + co-fill; not present in sampled API configs).

### `trigger_property_value_text` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-07): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Value or regex the trigger property must match (inferred; pairs with `trigger_property_name_text`, identical null rate).
- **Profile (full table):** 25,109 rows, 99.03% null, 119 distinct, length 1-128.
- **Values:** not complete. Top: NULL 24,865; `true` 20; `^false$` 15; `^invoices-create$` 11; `presentation` 9; `^true$` 8.
- **Confidence:** Inferred.

### `cln_create_utc_ts` - TIMESTAMP

> ⚠️ **OWNER INPUT NEEDED** (OQ-MS-08): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Presumably the ETL row-creation time (name; same column is populated in `user_allocation_history`); never populated here.
- **Profile (full table):** 25,109 rows, 100.00% null.
- **Values (complete):** NULL 25,109.
- **How to use / Traps:** Never use; table freshness cannot be read from this column.
- **Confidence:** Inferred.

---

## Fields only in Hivemind MCP (not in this table)

These experiment attributes exist in Hivemind (`get_experiment_by_id`, env prod) but have **no column** in `experiment_metadata_snap`. If a question asks for one, say it is not in Redshift and fetch it from the MCP. Do not answer "not defined".

| MCP field | Allowed values (MCP schema) | Meaning |
|---|---|---|
| `configType` | `default`, `parent`, `child`, `gocaas`, `sticky-bucketing` | Configuration type: parent/child (nested experiments), GoCaaS, or sticky bucketing (a unit keeps its cohort across sessions, e.g. billing-cycle dunning tests). Example: ecomm_nes_dunning_m365_daily_in_first_week_EP95240_billing_cycle = `sticky-bucketing`. |
| `bucketingSdkType` | `appconfig`, `salesforceMarketing`, `nonHivemind` | Which SDK buckets the traffic. |
| `category` | `onsite`, `offsite` | Onsite (web/app) vs offsite (email, SMS, care). The table's `category_enum` carries the same values where filled. |
| `isFixedDesign` | true / false | Fixed sample size (true) vs adaptive / sequential (false). It can be true even when `analysisMethodUsed` = sequential on a report. |
| `customerReportFilter` | `Existing Customer`, `New Prospect` | Customer report filter. |
| `algorithmType`, `phase`, `epsilon`, `trials` | `epsilon-greedy` / `thompson-sampling`; `Exploration` / `Exploitation` | Multi-armed bandit (MAB) settings only. |
| `airoAttachInitiative` | `Offer-Led`, `Marketing-Led`, `Product-Led`, `Other` | Sub-initiative when classification = Airo & Attach. |
| `rules`, `geosForcedToControl`, `geosForcedToTreatment`, `excludedExperiments` | lists | Targeting and exclusion rules. |
| `rerunInfo` | object | Last analysis rerun: reason, requester, time. Explains post-rollout reruns. |
| `annualizableMetrics` | metric → surfaces | Metrics eligible for annualization and the iGCR surfaces they may use. The value itself is only in `annualization_value`. |

Source: Hivemind MCP tool schemas and `get_experiment_by_id` payloads, read 2026-11-13.

## ⚠️ **OWNER INPUT NEEDED** — definitions only the Hivemind owners can supply

The definitions below are **not clear** from the data, Hivemind MCP or any document we could find. **A Hivemind data owner needs to fill in the "Owner answer" column.** Until then, agents must answer "definition not confirmed by the Hivemind owners" for these and must not guess.

| ID | Column / value | Question for the owner | Why it matters | What we observed | Owner answer |
|---|---|---|---|---|---|
| OQ-MS-01 | `additional_deciding_metrics_id_list` | What does this list hold, how is it different from deciding_metrics_id_list, and which config field feeds it? | Agents may treat it as a decision list. | Populated on some rows; no matching field seen in get_experiment_by_id payloads. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-02 | `ended_at_mst_ts` | Which source field feeds it (end action time vs lastUpdateTime), and why was it bulk-overwritten on 2026-11-03 21:57? | It looks like an end date but is later than the last bucket on 66% of A/B rows. | Do not use for duration; documented as unreliable. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-03 | `opportunities_text` | Where do values that are not in config.opportunities come from (readout or learnings form)? | Text answers about learnings may cite the wrong source. | Some values don't appear in the API config. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-04 | `badging_mst_ts / automated_conclusion_mst_ts` | What event sets each timestamp, and are they really MST? (rollout_mst_ts is named MST but holds UTC.) | Time-of-badge and time-of-conclusion questions. | Timezone unverified. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-05 | `annualization_trigger_metric_type_enum` | What do the GCR vs CVR trigger values mean for annualization? | iGCR explanations. | Only 6 rows populated. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-06 | `sig_lift_informative_metric_id_list / sig_loss_informative_metric_id_list` | Confirm these list informative metrics with a significant lift or loss on the latest report, and at which alpha. | Meaning is inferred from the name only. | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-07 | `trigger_property_name_text / trigger_property_value_text` | Which Hivemind config field feeds these, and how should they be read with trigger_eid_text? | Triggered-experiment questions. | Field not seen in sampled API payloads. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-08 | `annualization_value_prorated, first_report_generated_mst_ts, informative_metrics_list, cln_create_utc_ts` | These are always NULL. Are they deprecated, or should they be populated? | Agents may try to use them. | 100% NULL on the full table. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-09 | `experiment_type_enum = 'PRE-POST-STANDALONE'` | What is a PRE-POST-STANDALONE experiment and how is it different from PRE-POST? (Not in the MCP experimentType list.) | Experiment-type counts. | Present in data, absent from the API enum. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-10 | `status_enum = 'released' vs 'rolled out'` | What is the difference between released and rolled out? | Lifecycle counts; released_flag does not mean shipped. | Both values present. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-11 | `analysis_type values 'revenue_impact', 'User Interaction', 'conversion_rate_analysis'` | What are these analysis types? (The API only lists frequentist / bayesian.) | Choosing how to read results. | Present in data, absent from the API enum. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-12 | `experiment_classification_enum legacy values: 'Protect my Business', 'Big Swing', 'One System', 'Seamless Experience', 'Airo'` | Which current classification does each legacy value map to? | Classification win-rate / portfolio counts. | Not in the current MCP classification list. 'Site & Renewals' → 'Seamless Experience: Site & Renewals' is the assumed mapping. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-13 | `business_unit_name values 'FORGE', 'GDI', 'Midas', 'FOS', 'Domains', 'Domain Investors', and other small BUs` | Which canonical BU does each roll up to (or is it its own BU)? | BU portfolio rollups. | Only the drift mappings for Customer & Site, AGI, USI and Corporate Marketing are confirmed. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-14 | `rollout_decision_enum = 'issue with experiment'` | Confirm the meaning: the experiment was invalid (bug, SRM, setup), so no result. Should it count as decided? | Decided / conclusive-rate denominators. | 1,495 A/B rows. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-15 | `traffic_type_enum non-web id types (care_ucid_jomax_id, care_conversation_id, resource_id, billing_cycle_id, auction_id, payment_id, subscription_lifetime_id)` | Confirm what unit each id type buckets, and which column of experiment_order / user_allocation_history holds the user key for them. | User counts for non-web experiments. | 79 A/B experiments use these; the lean docs only cover shopper_id / visitor_guid. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-MS-16 | `physical column count` | The schema has 74 columns on 2026-11-13 vs 73 on 2026-10-22. Which column was added, and what does it mean? | Doc drift. | — | _TO BE FILLED BY HIVEMIND OWNER_ |

**Resolved from data (no owner input needed):**

- `duration_day_value vs bucket span` — Tested 2026-11-13: equals last − first bucket to within 0.1 day on 100% of 8,724 A/B rows that have both timestamps (the earlier ~76% counted rows with NULL buckets).
- `annualized_metric_significance_flag` — TRUE on 611 of 615 annualized rows; 4 annualized rows are NULL.
- `rollout_mst_ts midnight share` — 925 of 2,478 A/B rows with a rollout (37%) are exactly 00:00:00 UTC.

## Sources

1. Live column profile: `experiment_metadata_snap.profile.json` (author's local workspace, not committed) (scope: full table, all experiments, all types; `values_ab_population` = A/B portfolio population).
2. Verified table doc: `tables/hivemind_mart.experiment_metadata_snap.md` (traps, mandatory filters, BU mapping, iGCR rules).
3. Hivemind MCP enums: Hivemind MCP tool input schemas.
4. Hivemind MCP payloads (`get_experiment_by_id`, `get_experiment_analysis`, `get_commercial_os_metrics`) for 17 prod experiments: saved MCP payloads (author's local workspace).
5. Use cases: `use-cases/experimentation-hm-did-it-work.md`, `experimentation-hm-whats-it-worth.md`.
6. Read-only Redshift SQL against `hivemind_mart.experiment_metadata_snap`, 2026-11-13 (population counts, fill by start year, cross-column checks, case-collision ids, API-vs-table comparisons). Other-table vocabularies from the sibling column profiles (author's local workspace).
