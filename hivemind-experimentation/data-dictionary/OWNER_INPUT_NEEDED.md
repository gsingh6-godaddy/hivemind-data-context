# Hivemind owner input needed — open definitions

**For:** Hivemind data owners. **From:** Hivemind GD Data Context work, 2026-11-13.

These 43 definitions cannot be settled from Redshift, Hivemind MCP, Alation or Confluence. Please fill in the **Owner answer** column (or reply per ID). Each item is also flagged with ⚠️ OWNER INPUT NEEDED in the matching table file.

## hivemind_mart.experiment_metadata_snap (16)

| ID | Column / value | Question | Why it matters | Owner answer |
|---|---|---|---|---|
| OQ-MS-01 | `additional_deciding_metrics_id_list` | What does this list hold, how is it different from deciding_metrics_id_list, and which config field feeds it? | Agents may treat it as a decision list. | |
| OQ-MS-02 | `ended_at_mst_ts` | Which source field feeds it (end action time vs lastUpdateTime), and why was it bulk-overwritten on 2026-11-03 21:57? | It looks like an end date but is later than the last bucket on 66% of A/B rows. | |
| OQ-MS-03 | `opportunities_text` | Where do values that are not in config.opportunities come from (readout or learnings form)? | Text answers about learnings may cite the wrong source. | |
| OQ-MS-04 | `badging_mst_ts / automated_conclusion_mst_ts` | What event sets each timestamp, and are they really MST? (rollout_mst_ts is named MST but holds UTC.) | Time-of-badge and time-of-conclusion questions. | |
| OQ-MS-05 | `annualization_trigger_metric_type_enum` | What do the GCR vs CVR trigger values mean for annualization? | iGCR explanations. | |
| OQ-MS-06 | `sig_lift_informative_metric_id_list / sig_loss_informative_metric_id_list` | Confirm these list informative metrics with a significant lift or loss on the latest report, and at which alpha. | Meaning is inferred from the name only. | |
| OQ-MS-07 | `trigger_property_name_text / trigger_property_value_text` | Which Hivemind config field feeds these, and how should they be read with trigger_eid_text? | Triggered-experiment questions. | |
| OQ-MS-08 | `annualization_value_prorated, first_report_generated_mst_ts, informative_metrics_list, cln_create_utc_ts` | These are always NULL. Are they deprecated, or should they be populated? | Agents may try to use them. | |
| OQ-MS-09 | `experiment_type_enum = 'PRE-POST-STANDALONE'` | What is a PRE-POST-STANDALONE experiment and how is it different from PRE-POST? (Not in the MCP experimentType list.) | Experiment-type counts. | |
| OQ-MS-10 | `status_enum = 'released' vs 'rolled out'` | What is the difference between released and rolled out? | Lifecycle counts; released_flag does not mean shipped. | |
| OQ-MS-11 | `analysis_type values 'revenue_impact', 'User Interaction', 'conversion_rate_analysis'` | What are these analysis types? (The API only lists frequentist / bayesian.) | Choosing how to read results. | |
| OQ-MS-12 | `experiment_classification_enum legacy values: 'Protect my Business', 'Big Swing', 'One System', 'Seamless Experience', 'Airo'` | Which current classification does each legacy value map to? | Classification win-rate / portfolio counts. | |
| OQ-MS-13 | `business_unit_name values 'FORGE', 'GDI', 'Midas', 'FOS', 'Domains', 'Domain Investors', and other small BUs` | Which canonical BU does each roll up to (or is it its own BU)? | BU portfolio rollups. | |
| OQ-MS-14 | `rollout_decision_enum = 'issue with experiment'` | Confirm the meaning: the experiment was invalid (bug, SRM, setup), so no result. Should it count as decided? | Decided / conclusive-rate denominators. | |
| OQ-MS-15 | `traffic_type_enum non-web id types (care_ucid_jomax_id, care_conversation_id, resource_id, billing_cycle_id, auction_id, payment_id, subscription_lifetime_id)` | Confirm what unit each id type buckets, and which column of experiment_order / user_allocation_history holds the user key for them. | User counts for non-web experiments. | |
| OQ-MS-16 | `physical column count` | The schema has 74 columns on 2026-11-13 vs 73 on 2026-10-22. Which column was added, and what does it mean? | Doc drift. | |

## hivemind_mart.analysis_result_cln (8)

| ID | Column / value | Question | Why it matters | Owner answer |
|---|---|---|---|---|
| OQ-AR-01 | `missing_pre_experiment_data_* (4 columns)` | What do these measure (which pre-period, which covariate for CUPED)? | Variance-reduction questions. | |
| OQ-AR-02 | `start_mst_ts / end_mst_ts` | Why are they NULL for whole experiments (1.38% of rows, e.g. AIWD-2670, cde_app_*)? | Analysis-window questions. | |
| OQ-AR-03 | `market_sub_region codes 'DEM', 'ROW DM', 'ROW EM', 'OTHERS'` | What do these codes stand for? | Segment questions (pre-June 2026 data). | |
| OQ-AR-04 | `statistical_test_name = 'Mann-Whitney U'` | When does Hivemind choose Mann-Whitney U instead of a T test or Z test? | Explaining which test produced a p-value. | |
| OQ-AR-05 | `relative_confidence_interval tokens '(None, None)', '(-inf, …)', '(-1.0, -1.0)'` | What produces these degenerate intervals (zero control, one-valued cohort)? | Reading CIs. | |
| OQ-AR-06 | `metric_delta on SRM (OverallHealth) treatment rows` | What does metric_delta mean on SRM rows? | SRM explanations. | |
| OQ-AR-07 | `metric_significance_flag on Bayesian rows` | The flag is FALSE in Redshift where the API says isSignificant = true. Is this an ETL mapping gap? | Bayesian experiments may be reported as not significant. | |
| OQ-AR-08 | `control-cohort labels (wacmnth, waddch, smsidm, aabf2p55h, binshown, trust-base, A, …) and scorecard template contents` | Is there a reference list of cohort label conventions and template contents? | Interpreting cohort names (control must still come from baseline_cohort_id). | |

## hivemind_mart.experiment_order (12)

| ID | Column / value | Question | Why it matters | Owner answer |
|---|---|---|---|---|
| OQ-EO-01 | `upgraded_pf_id` | Is this the product upgraded FROM or TO? | Upgrade analyses. | |
| OQ-EO-02 | `margin_gcr_usd_amt` | What is the formula (which costs are subtracted)? | Margin questions. | |
| OQ-EO-03 | `new_acquisition_flag` | Business definition: first-ever purchase by the customer, by product, or in what window? | New vs existing customer splits. | |
| OQ-EO-04 | `virtual_order_flag` | What is a virtual order? | Whether to exclude them from GCR. | |
| OQ-EO-05 | `product_free_trial_flag / product_free_trial_conversion_flag` | Definitions, and whether trial orders carry GCR. | Free-trial and conversion analyses. | |
| OQ-EO-06 | `dpp_order_flag` | What is a DPP order (Domain Purchase Path?), and should it be excluded anywhere? | Domain analyses. | |
| OQ-EO-07 | `domain_bulk_flag` | What makes an order a bulk domain order (threshold)? | Domain analyses / outliers. | |
| OQ-EO-08 | `bundle_id vs parent_bundle_id` | How do the two relate (nesting rule)? | Bundle analyses. | |
| OQ-EO-09 | `segment values: customer_type 'Not Evaluated', product_pnl_group/category 'NA', market_site 'uk-uk', device 'Other', 'Multiple' in device / market / country` | What does each mean, and what rule produces 'Multiple'? | Segment breakdowns. | |
| OQ-EO-10 | `customer_tenure_day_segment` | What is the tenure anchor, and does a '2-7 day' bucket exist? | Tenure splits. | |
| OQ-EO-11 | `monthly_recurring_revenue_usd_amt` | Always 0 in scope. Is it ever populated? | MRR questions. | |
| OQ-EO-12 | `*_mst_* timestamp columns` | Confirm they are true MST (UTC-7) and not UTC like rollout_mst_ts. | Order-date filters. | |

## hivemind_mart.user_allocation_history (7)

| ID | Column / value | Question | Why it matters | Owner answer |
|---|---|---|---|---|
| OQ-UA-01 | `bucket_session_start_mst_ts` | Which session is chosen, and why is it often later than bucketed_mst_ts (147,663 of 225,901 rows for mobile_ios_contextual_nudges_v2)? | Session-level timing questions. | |
| OQ-UA-02 | `wam_free_trial_flag` | Exact definition of a WAM free trial and its as-of date. | WAM trial splits. | |
| OQ-UA-03 | `device_breakpoint_name, market_site, ip_country_name 'Other' / 'Multiple'` | Meaning of 'Other', and the rule that produces 'Multiple' (it occurs on single-visit users). | Segment splits. | |
| OQ-UA-04 | `customer_tenure_day_segment / domain_ownership_count` | Tenure anchor, as-of date, the boundary at exactly 10 domains, and whether a '2-7 day' bucket exists. | Segment splits. | |
| OQ-UA-05 | `customer_type / new_existing_customer_name` | Business definition of each customer type (incl. 'Not Evaluated') and the Existing vs New Prospect rule. | Customer segment splits. | |
| OQ-UA-06 | `visit_count` | Counting window and session definition. | Engagement questions. | |
| OQ-UA-07 | `user key for non-web traffic types` | For billing_cycle_id / care_* / resource_id experiments, which column holds the bucketed unit? | User counts for non-web experiments. | |

## Already resolved from data (no action needed)

- **experiment_order.`allocation_id join to user_allocation_history`** — Tested 2026-11-13 on recore_abn_gdexp_13936: 190,324 allocation_ids in both tables, all join, 0 cohort mismatches. allocation_id is a valid user-level join key.
- **experiment_metadata_snap.`duration_day_value vs bucket span`** — Tested 2026-11-13: equals last − first bucket to within 0.1 day on 100% of 8,724 A/B rows that have both timestamps (the earlier ~76% counted rows with NULL buckets).
- **experiment_metadata_snap.`annualized_metric_significance_flag`** — TRUE on 611 of 615 annualized rows; 4 annualized rows are NULL.
- **experiment_metadata_snap.`rollout_mst_ts midnight share`** — 925 of 2,478 A/B rows with a rollout (37%) are exactly 00:00:00 UTC.
- **user_allocation_history.`allocation_id hash inputs`** — Not needed for analysis; dropped from the owner list.
