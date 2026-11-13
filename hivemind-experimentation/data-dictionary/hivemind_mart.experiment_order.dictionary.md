# `hivemind_mart.experiment_order` - Data Dictionary

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-12): table-level definition not clear (whether the *_mst_* timestamps are true MST or UTC). Hivemind owner to fill in. See the owner table at the end of this file.

**Purpose.** One row per allocated Hivemind experiment user per order line, with product-line GCR, so an analyst can see which product lines, new-vs-renewal splits and term lengths make up each cohort's raw GCR. Non-buyers are kept as one row with `order_id IS NULL`, so per-user denominators can be built from this table alone. Values are raw (no winsorization, no CUPED, no significance) and the table is a cumulative daily snapshot - read exactly one partition.

- **Grain:** one row per (experiment_id, record_create_mst_date, allocated user, order_id, row_id); non-buyers = one row per user with order_id/row_id NULL. SQL: (order_id,row_id) unique within a partition; allocation_id 1:1 with user.
- **Primary key:** none declared. Effective key within one partition: `allocation_id` + `order_id` + `row_id` (Verified on recore_abn_gdexp_13936 and MKT_SMS_516).
- **Partition / sort:** Spectrum table partitioned by `record_create_mst_date` (STRING). No sort key documented.
- **Mandatory filters:** `experiment_id = '<id>'` (exact, case-sensitive) AND `record_create_mst_date = '<latest partition literal>'`. Never a range. Scans take 15-100 s even when filtered.
- **Row counts:** profile scope = 1,826,832 rows: latest partition of 5 experiments (recore_abn_gdexp_13936, abn_gdexp_14418, MKT_SMS_516-UPM-D5_CM-24664, MKT_EM_177_Failed-Billing-Manual-D28_CM-24927, agi_plg_email_inline_upgrade_erp) — visitor- and shopper-keyed, onsite + email + SMS. Full table ~2.3B rows (verified doc). Per experiment in scope: MKT_EM_177_Failed-Billing-Manual-D28_CM-24927 588,001, agi_plg_email_inline_upgrade_erp 526,476, abn_gdexp_14418 444,211, recore_abn_gdexp_13936 241,640, MKT_SMS_516-UPM-D5_CM-24664 26,504.

## Column index

| # | Column | Type | Null % | Distinct | Meaning | Confidence |
|---|---|---|---|---|---|---|
| 1 | `order_id` | string | 49.54 | 564,425 | Order number of the purchase attributed to the allocated user | Verified |
| 2 | `row_id` | integer | 49.54 | 418 | Line-item number within the order, 0-based | Verified |
| 3 | `allocation_id` | string | 0.0 | 1,142,172 | Identifier of the user's allocation (bucketing) record in the experiment | Inferred |
| 4 | `shopper_id` | string | 37.54 | 522,744 | GoDaddy shopper (customer account) id of the allocated user | Verified |
| 5 | `visitor_guid` | string | 62.46 | 612,956 | Website visitor GUID of the allocated user | Verified |
| 6 | `experiment_id` | string | 0.0 | 5 | Hivemind experiment key (API: config `id` / `expId`) | Verified |
| 7 | `cohort_id` | string | 0.0 | 10 | Cohort (arm) the user was bucketed into; present on every row incl. non-buyers | Verified |
| 8 | `bucketed_mst_date` | date | 0.0 | 58 | MST calendar date on which the user was bucketed (allocated) into the experiment; date part of `bucketed_mst_ts` | Verified |
| 9 | `experiment_user_identifier` | string | 0.0 | 2 | The id TYPE that keys the experiment (which column holds the user key), not an id | Verified |
| 10 | `device_breakpoint_name` | string | 33.64 | 4 | Device class of the user (Hivemind segment dimension copied onto each row; analysis_result_cln calls it `device_name`) | Verified |
| 11 | `market_site` | string | 33.69 | 53 | GoDaddy market/site locale of the user (segment dimension), language-country code like `en-us` | Verified |
| 12 | `new_existing_customer_name` | string | 33.64 | 2 | New vs existing customer segment of the allocated user | Verified |
| 13 | `ip_country_name` | string | 33.67 | 233 | Country from the user's IP geolocation (segment dimension) | Verified |
| 14 | `customer_tenure_day_segment` | string | 0.0 | 4 | Customer tenure bucket (days since the shopper became a customer), segment dimension | Verified |
| 15 | `domain_ownership_count` | string | 0.0 | 5 | Bucketed count of domains the shopper owns (segment dimension) | Verified |
| 16 | `customer_type` | string | 37.54 | 5 | Customer type segment of the shopper | Verified |
| 17 | `scorecard_template_name` | string | 0.0 | 4 | Hivemind scorecard template of the experiment, copied onto every row | Verified |
| 18 | `bucketed_mst_ts` | timestamp | 0.0 | 1,089,171 | Timestamp (MST, millisecond precision) at which the user was bucketed | Verified |
| 19 | `order_visit_guid` | string | 68.76 | 266,400 | GUID of the web visit (session) in which the order was placed | Inferred |
| 20 | `order_mst_date` | date | 49.54 | 57 | MST date of the order; date part of `order_mst_ts` | Verified |
| 21 | `order_mst_ts` | timestamp | 49.54 | 517,407 | Order timestamp (MST) | Verified |
| 22 | `gcr_usd_amt` | numeric | 49.54 | 22,587 | Gross cash receipts (GCR) of the order line in USD - raw, unwinsorized, gross | Verified |
| 23 | `product_free_trial_flag` | boolean | 49.54 | 2 | TRUE when the order line is a free-trial product line | Inferred |
| 24 | `receipt_price_usd_amt` | numeric | 49.54 | 17,879 | Receipt price of the order line in USD | Inferred |
| 25 | `pf_id` | integer | 49.54 | 4,576 | Product (catalog item) id of the order line | Inferred |
| 26 | `upgraded_pf_id` | integer | 99.37 | 240 | Product id involved in an upgrade on this line (filled only on upgrade lines) | Inferred |
| 27 | `bundle_id` | integer | 99.86 | 2,463 | Bundle id when the line was sold as part of a bundle | Inferred |
| 28 | `parent_bundle_id` | integer | 99.64 | 2,463 | Parent bundle id of the line (the bundle this line belongs to) | Inferred |
| 29 | `product_pnl_line_name` | string | 49.54 | 47 | PnL product line of the order line | Verified |
| 30 | `product_unit_qty` | integer | 49.54 | 354 | Units on the order line | Verified |
| 31 | `product_month_qty` | numeric | 49.54 | 2,926 | Term length in months of the line | Verified |
| 32 | `product_pnl_new_renewal_name` | string | 49.54 | 2 | New purchase vs renewal classification of the line | Verified |
| 33 | `purchase_path_name` | string | 49.54 | 29 | Purchase path (site flow) through which the line was bought | Inferred |
| 34 | `item_tracking_code` | string | 68.78 | 672 | Item tracking code (ITC) - code of the page/placement that added the item to cart | Inferred |
| 35 | `new_acquisition_flag` | boolean | 49.54 | 2 | TRUE when the line is flagged as a new customer acquisition | Inferred |
| 36 | `domain_bulk_flag` | boolean | 49.54 | 2 | TRUE when the line was part of a bulk domain purchase | Inferred |
| 37 | `crm_portfolio_type_name` | string | 93.06 | 20 | CRM portfolio (account-management program) the shopper belongs to | Inferred |
| 38 | `order_region_2_name` | string | 49.54 | 8 | Region rollup of the order (level-2 region) | Inferred |
| 39 | `point_of_purchase_name` | string | 49.54 | 3 | Channel through which the order was placed | Inferred |
| 40 | `margin_gcr_usd_amt` | numeric | 49.54 | 22,139 | Margin GCR of the line in USD (GCR net of some cost component) | Inferred |
| 41 | `dpp_order_flag` | boolean | 49.54 | 2 | TRUE when the order came through the Domain Purchase Path (DPP) | Inferred |
| 42 | `original_product_pnl_new_renewal_name` | string | 49.54 | 2 | New/renewal classification before any reclassification (original value) | Inferred |
| 43 | `monthly_recurring_revenue_usd_amt` | numeric | 0.0 | 1 | Monthly recurring revenue of the line - UNPOPULATED in scope | Verified |
| 44 | `product_free_trial_conversion_flag` | boolean | 49.54 | 2 | TRUE when the line is a conversion of a free trial to paid | Inferred |
| 45 | `product_pnl_subline_name` | string | 49.54 | 723 | PnL subline under product_pnl_line_name | Verified |
| 46 | `product_pnl_group_name` | string | 49.54 | 5 | Top-level PnL group of the line | Inferred |
| 47 | `product_pnl_category_name` | string | 49.54 | 15 | PnL category (between group and line) | Inferred |
| 48 | `product_pnl_version_name` | string | 49.54 | 112 | PnL version (below subline), e.g. TLD group or product version | Inferred |
| 49 | `product_name` | string | 49.54 | 4,419 | Catalog product name of the line | Verified |
| 50 | `product_period_name` | string | 49.54 | 5 | Term unit of the line | Verified |
| 51 | `virtual_order_flag` | boolean | 49.54 | 2 | TRUE for virtual (non-cash) order lines | Inferred |
| 52 | `refund_flag` | boolean | 49.54 | 2 | TRUE when the line is a refund row | Verified |
| 53 | `web_pro_shopper_id` | string | 96.72 | 8,572 | Shopper id of the GoDaddy Pro (web professional) linked to the purchase | Inferred |
| 54 | `web_pro_participant_type_name` | string | 89.21 | 2 | Role of the purchaser in a GoDaddy Pro relationship | Inferred |
| 55 | `c3_rep_department_name` | string | 49.54 | 110 | Department of the rep (C3) credited with the order, or the channel name when no rep | Inferred |
| 56 | `cln_create_utc_ts` | timestamp | 0.0 | 5 | ETL load timestamp (UTC) of the partition | Verified |
| 57 | `record_create_mst_date` | string | 0.0 | 5 | Snapshot partition date, STRING 'YYYY-MM-DD' | Verified |

## Cross-table value map

| Column here | Other table.column | Shared values / relationship | Confidence |
|---|---|---|---|
| `experiment_id` | experiment_metadata_snap / analysis_result_cln / user_allocation_history `.experiment_id` | same key; many_to_one to experiment_metadata_snap | Verified |
| `cohort_id` | experiment_metadata_snap.baseline_cohort_id, .cohort_name_list; analysis_result_cln.cohort_id / control_cohort; user_allocation_history.cohort_id; Commercial OS cohort_id_dim | same free-text arm names (e.g. control_variant, b_variant, control, testa-testd, default, variant, ctrl) | Verified |
| `experiment_user_identifier` | experiment_metadata_snap.traffic_type_enum; user_allocation_history.experiment_user_identifier | values `shopper_id`, `visitor_guid` (subset of MCP trafficType) | Verified |
| `scorecard_template_name` | experiment_metadata_snap / analysis_result_cln / user_allocation_history `.scorecard_template_name` | e.g. cm-renewals-audience-recording, hp_core_metrics (both seen in metadata and ARC profiles) | Verified |
| `device_breakpoint_name`, `market_site`, `new_existing_customer_name`, `ip_country_name`, `customer_tenure_day_segment`, `domain_ownership_count`, `customer_type` | user_allocation_history (same column names, same value sets incl. `Multiple`, `Not Evaluated`); analysis_result_cln (`device_name`, `market_site`, ...; profiled as constant `ALL` there = overall, no segment) | same labels | Verified |
| `new_existing_customer_name` | MCP customerReportFilter enum | `Existing Customer`, `New Prospect` | Verified |
| `allocation_id`, `shopper_id`, `visitor_guid`, `bucketed_mst_ts`, `bucketed_mst_date` | user_allocation_history (same names) | same allocation-level fields; join on allocation_id not tested | Inferred |
| `record_create_mst_date` | experiment_metadata_snap.last_user_bucket_mst_ts (lower bound for the latest-partition lookup); user_allocation_history has no such column | partition logic | Verified |
| `gcr_usd_amt` | Commercial OS winsorization_diagnostics.gcr_raw / metrics.gcr; analysis_result_cln.metric_value | similar quantity but does not reconcile (raw vs winsorized/CUPED, partition freeze) | Verified |

## Columns

### `order_id` - STRING
- **Meaning:** Order number of the purchase attributed to the allocated user. NULL = the user was allocated but placed no attributed order (non-buyer row). Hivemind API: no per-order field; Commercial OS `paid_users` counts users with orders.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 564,425 distinct; length 10-10; values_complete=false
- **Values (not complete; 564,425 distinct - high cardinality):** top values: NULL (905,031), `4186662028` (418), `4182993795` (316), `4186733868` (264), `4182007662` (246), `4191638534` (228). Format: 10-digit numeric string (all lengths = 10), e.g. `4186662028`. Not unique per row: one order has one row per line item (`row_id`).
- **How to use / Traps:** Count orders with `COUNT(DISTINCT order_id)`; keep NULL rows for the per-user denominator (dividing by buyers gives average order value, not GCR/user). SQL (MKT_SMS_516 latest partition): 8,335 orders, each in exactly one cohort, one shopper, one order_visit_guid and one order_mst_ts. The same order can sit in many concurrent experiments (verified doc: ~17 on median) - never add GCR across experiments. Compare as string.
- **Confidence:** Verified (profile + SQL + verified doc).

### `row_id` - INTEGER
- **Meaning:** Line-item number within the order, 0-based. NULL on non-buyer rows.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 418 distinct; min 0; max 417; avg 3.294; values_complete=false
- **Values (not complete; 418 distinct - high cardinality):** top values: NULL (905,031), `0` (570,306), `1` (126,980), `2` (52,659), `3` (32,592), `4` (20,600), `5` (15,733), `6` (11,833). Integer 0-417.
- **How to use / Traps:** (order_id, row_id) is unique within one experiment partition (SQL: recore 86,680 order rows = 86,680 distinct pairs; MKT_SMS_516 13,258 = 13,258). row_id is dense: for all 8,335 MKT_SMS_516 orders MAX(row_id)+1 = line count. Use `row_id = 0` to get one row per order; do not use it as a quantity.
- **Confidence:** Verified (SQL).

### `allocation_id` - STRING
- **Meaning:** Identifier of the user's allocation (bucketing) record in the experiment. Present on every row incl. non-buyers.
- **Profile (scope):** 1,826,832 rows; null 0.0%; empty 0.0%; 1,142,172 distinct; length 64-64; values_complete=false
- **Values (not complete; 1,142,172 distinct - high cardinality):** top values: `0a76913cc59f7b8b13e4e4bb6c8db880b12339cc5ffade0b4246013a76e1a88e` (9,192), `2a2927f9d48a574042016e7a8f617780182da56f33324c389b5116a55bfea902` (4,007), `acd5419b292b180d9b62c2856c8e2233f28a401313e677050f125a1b535fe9a0` (2,950), `f7d32665cd62c90b87398e84cb17301a4ba63d5e87cde60ca276a9afb34abaa4` (2,538). Format: 64-char lowercase hex (looks like a SHA-256 hash).
- **How to use / Traps:** SQL: within one experiment allocation_id is 1:1 with the user key (recore: 35,364 buyer allocations = 35,364 buyer visitors; MKT_SMS_516: 17,329 allocations = 17,329 shoppers). The top value (9,192 rows) has the same count as top shopper_id 13515917, i.e. one user with 9,192 order lines. Also a column of `user_allocation_history`; the join between the two tables was not tested. Case-sensitive string.
- **Confidence:** Inferred (hash format and allocation meaning inferred from name/pattern; 1:1 with user verified by SQL).

### `shopper_id` - STRING
- **Meaning:** GoDaddy shopper (customer account) id of the allocated user. It is the user key only when `experiment_user_identifier = 'shopper_id'`.
- **Profile (scope):** 1,826,832 rows; null 37.54%; empty 0.0%; 522,744 distinct; length 6-9; values_complete=false
- **Values (not complete; 522,744 distinct - high cardinality):** top values: NULL (685,851), `13515917` (9,192), `23080542` (4,007), `652627397` (2,950), `25739626` (2,538). Format: numeric string, 6-9 chars.
- **How to use / Traps:** NULL count 685,851 = exactly the visitor-keyed rows (abn_gdexp_14418 + recore_abn_gdexp_13936 = 685,851); SQL confirmed recore has 0 shopper_id filled. NULL therefore means 'experiment keyed on visitor', not a data gap. User key = `CASE WHEN experiment_user_identifier = 'shopper_id' THEN shopper_id ELSE visitor_guid END` (verified doc rule: never COALESCE the two ids). Compare as string (leading digits matter, no numeric cast needed).
- **Confidence:** Verified (profile + SQL + verified doc).

### `visitor_guid` - STRING
- **Meaning:** Website visitor GUID of the allocated user. The user key when `experiment_user_identifier = 'visitor_guid'` (onsite tests). API: config.trafficType = 'visitor_guid'.
- **Profile (scope):** 1,826,832 rows; null 62.46%; empty 0.0%; 612,956 distinct; length 36-36; values_complete=false
- **Values (not complete; 612,956 distinct - high cardinality):** top values: NULL (1,140,981), `82e18555-ec08-464a-8289-613b2c80f4dc` (581), `35f5be4f-da67-4197-9bbd-7e2b651e6ed5` (407), `c1c21316-6b40-4c39-9c50-e59cb362070c` (256). Format: 36-char lowercase UUID, e.g. `82e18555-ec08-464a-8289-613b2c80f4dc`.
- **How to use / Traps:** NULL count 1,140,981 = exactly the shopper-keyed rows (588,001 + 526,476 + 26,504); SQL: 0 visitor_guid filled on MKT_SMS_516. Filled on every row of visitor-keyed experiments. Not the same as `order_visit_guid` (SQL on recore: most order rows have order_visit_guid <> visitor_guid). Case-sensitive string.
- **Confidence:** Verified (profile + SQL + verified doc).

### `experiment_id` - STRING
- **Meaning:** Hivemind experiment key (API: config `id` / `expId`). Joins `experiment_metadata_snap.experiment_id`, `analysis_result_cln.experiment_id`, `user_allocation_history.experiment_id`.
- **Profile (scope):** 1,826,832 rows; null 0.0%; empty 0.0%; 5 distinct; length 15-45; values_complete=true
- **Values (complete, all 5 distinct non-null values):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `MKT_EM_177_Failed-Billing-Manual-D28_CM-24927` | 588,001 | 32.19% | experiment display name not captured here (reading of the id as a Customer Marketing failed-billing email is Inferred from the id text only). shopper-keyed, offsite, status rolled out, scorecard cm-renewals-audience-recording, baseline `control`; latest partition 2026-10-07 |
| `agi_plg_email_inline_upgrade_erp` | 526,476 | 28.82% | 'Inline Upgrade CTA in ERP'. shopper-keyed, onsite, status rolled back, scorecard AGI_Mon_Email_ERP, baseline `default`; latest partition 2026-10-08 |
| `abn_gdexp_14418` | 444,211 | 24.32% | 'AAB: SLP - Split Test - Tailored Copy for Signed in AB Customers'. visitor-keyed, onsite, status rolled back, scorecard fos-airo-ai-builder, baseline `control_variant`; latest partition 2026-10-15 |
| `recore_abn_gdexp_13936` | 241,640 | 13.23% | 'US + DEM Recore - Enable DOP Recommendation in Next Best Product model'. visitor-keyed, onsite, status rolled out, scorecard hp_core_metrics, baseline `control_variant`; latest partition 2026-10-21 |
| `MKT_SMS_516-UPM-D5_CM-24664` | 26,504 | 1.45% | 'SMS UPM, 5D - Static AI Test'. shopper-keyed, offsite (SMS), status rolled back, scorecard cm-renewals-audience-recording, baseline `ctrl`; latest partition 2026-10-23 |

- **How to use / Traps:** MANDATORY filter, exact and case-sensitive, always together with one literal `record_create_mst_date` (full table is ~2.3B rows per verified doc). Never NULL. The profile scope contains only these 5 experiments; any other experiment exists in the full table but was not profiled.
- **Confidence:** Verified (profile + MCP get_experiment_by_id).

### `cohort_id` - STRING
- **Meaning:** Cohort (arm) the user was bucketed into; present on every row incl. non-buyers. API: config.cohorts[].name; Commercial OS `cohort_id_dim`.
- **Profile (scope):** 1,826,832 rows; null 0.0%; empty 0.0%; 10 distinct; length 4-15; values_complete=true
- **Values (complete, all 10 distinct non-null values):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `control_variant` | 343,297 | 18.79% | baseline (config.baselineCohortId) of recore_abn_gdexp_13936 and abn_gdexp_14418 |
| `b_variant` | 342,554 | 18.75% | treatment arm of recore_abn_gdexp_13936 and abn_gdexp_14418 |
| `default` | 265,365 | 14.53% | baseline of agi_plg_email_inline_upgrade_erp (config.baselineCohortId = 'default') |
| `variant` | 261,111 | 14.29% | treatment arm of agi_plg_email_inline_upgrade_erp |
| `testb` | 131,573 | 7.20% | treatment arm of MKT_EM_177 and MKT_SMS_516 |
| `control` | 122,955 | 6.73% | baseline of MKT_EM_177_Failed-Billing-Manual-D28_CM-24927 |
| `testa` | 122,213 | 6.69% | treatment arm of MKT_EM_177 and MKT_SMS_516 |
| `testd` | 117,276 | 6.42% | treatment arm of MKT_EM_177 only |
| `testc` | 111,400 | 6.10% | treatment arm of MKT_EM_177 only |
| `ctrl` | 9,088 | 0.50% | baseline of MKT_SMS_516-UPM-D5_CM-24664 (config.baselineCohortId = 'ctrl') |

- **How to use / Traps:** Free-text names chosen per experiment, not a global enum. The control is `experiment_metadata_snap.baseline_cohort_id` (= config.baselineCohortId) - never infer it from the name (`default` is the control on agi_plg_email_inline_upgrade_erp; `variant` is treatment). Always group by `experiment_id, cohort_id`, because the same name (`control_variant`, `testa`) appears in several experiments. Case-sensitive.
- **Confidence:** Verified (profile + MCP config baselineCohortId / cohorts).

### `bucketed_mst_date` - DATE
- **Meaning:** MST calendar date on which the user was bucketed (allocated) into the experiment; date part of `bucketed_mst_ts`.
- **Profile (scope):** 1,826,832 rows; null 0.0%; empty 0.0%; 58 distinct; min 2026-08-26; max 2026-10-22; values_complete=true
- **Values (complete, all 58 distinct non-null values):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `2026-09-10` | 139,930 | 7.66% | users/rows bucketed on this MST date |
| `2026-10-13` | 67,827 | 3.71% | users/rows bucketed on this MST date |
| `2026-09-11` | 60,603 | 3.32% | users/rows bucketed on this MST date |
| `2026-09-29` | 53,814 | 2.95% | users/rows bucketed on this MST date |
| `2026-10-02` | 51,000 | 2.79% | users/rows bucketed on this MST date |
| `2026-09-24` | 50,814 | 2.78% | users/rows bucketed on this MST date |
| `2026-09-23` | 50,514 | 2.77% | users/rows bucketed on this MST date |
| `2026-09-30` | 47,238 | 2.59% | users/rows bucketed on this MST date |
| `2026-09-28` | 47,229 | 2.59% | users/rows bucketed on this MST date |
| `2026-09-25` | 45,424 | 2.49% | users/rows bucketed on this MST date |
| `2026-09-14` | 44,349 | 2.43% | users/rows bucketed on this MST date |
| `2026-09-12` | 43,719 | 2.39% | users/rows bucketed on this MST date |
| `2026-09-26` | 42,519 | 2.33% | users/rows bucketed on this MST date |
| `2026-09-27` | 41,536 | 2.27% | users/rows bucketed on this MST date |
| `2026-10-05` | 41,335 | 2.26% | users/rows bucketed on this MST date |
| `2026-10-03` | 41,041 | 2.25% | users/rows bucketed on this MST date |
| `2026-10-14` | 40,623 | 2.22% | users/rows bucketed on this MST date |
| `2026-10-01` | 40,053 | 2.19% | users/rows bucketed on this MST date |
| `2026-09-13` | 39,487 | 2.16% | users/rows bucketed on this MST date |
| `2026-09-15` | 39,320 | 2.15% | users/rows bucketed on this MST date |
| `2026-10-04` | 38,670 | 2.12% | users/rows bucketed on this MST date |
| `2026-09-16` | 37,666 | 2.06% | users/rows bucketed on this MST date |
| `2026-10-15` | 36,420 | 1.99% | users/rows bucketed on this MST date |
| `2026-10-06` | 35,919 | 1.97% | users/rows bucketed on this MST date |
| `2026-10-12` | 35,675 | 1.95% | users/rows bucketed on this MST date |
| `2026-09-17` | 33,311 | 1.82% | users/rows bucketed on this MST date |
| `2026-10-19` | 32,818 | 1.80% | users/rows bucketed on this MST date |
| `2026-08-31` | 32,477 | 1.78% | users/rows bucketed on this MST date |
| `2026-09-18` | 32,044 | 1.75% | users/rows bucketed on this MST date |
| `2026-09-21` | 31,127 | 1.70% | users/rows bucketed on this MST date |
| `2026-08-27` | 31,121 | 1.70% | users/rows bucketed on this MST date |
| `2026-10-16` | 28,628 | 1.57% | users/rows bucketed on this MST date |
| `2026-09-01` | 26,095 | 1.43% | users/rows bucketed on this MST date |
| `2026-09-19` | 25,851 | 1.42% | users/rows bucketed on this MST date |
| `2026-09-20` | 24,752 | 1.35% | users/rows bucketed on this MST date |
| `2026-09-02` | 23,079 | 1.26% | users/rows bucketed on this MST date |
| `2026-09-03` | 21,321 | 1.17% | users/rows bucketed on this MST date |
| `2026-08-28` | 21,239 | 1.16% | users/rows bucketed on this MST date |
| `2026-10-18` | 18,207 | 1.00% | users/rows bucketed on this MST date |
| `2026-09-08` | 17,943 | 0.98% | users/rows bucketed on this MST date |
| `2026-10-08` | 17,836 | 0.98% | users/rows bucketed on this MST date |
| `2026-10-17` | 17,698 | 0.97% | users/rows bucketed on this MST date |
| `2026-10-07` | 17,152 | 0.94% | users/rows bucketed on this MST date |
| `2026-09-09` | 17,130 | 0.94% | users/rows bucketed on this MST date |
| `2026-09-04` | 16,455 | 0.90% | users/rows bucketed on this MST date |
| `2026-09-07` | 15,605 | 0.85% | users/rows bucketed on this MST date |
| `2026-10-09` | 15,483 | 0.85% | users/rows bucketed on this MST date |
| `2026-10-11` | 15,052 | 0.82% | users/rows bucketed on this MST date |
| `2026-10-10` | 14,884 | 0.81% | users/rows bucketed on this MST date |
| `2026-08-30` | 14,270 | 0.78% | users/rows bucketed on this MST date |
| `2026-09-22` | 11,967 | 0.66% | users/rows bucketed on this MST date |
| `2026-08-29` | 11,963 | 0.65% | users/rows bucketed on this MST date |
| `2026-09-06` | 9,457 | 0.52% | users/rows bucketed on this MST date |
| `2026-08-26` | 9,272 | 0.51% | users/rows bucketed on this MST date |
| `2026-09-05` | 8,918 | 0.49% | users/rows bucketed on this MST date |
| `2026-10-20` | 342 | 0.02% | users/rows bucketed on this MST date |
| `2026-10-22` | 312 | 0.02% | users/rows bucketed on this MST date |
| `2026-10-21` | 298 | 0.02% | users/rows bucketed on this MST date |

- **How to use / Traps:** Never NULL. Counts are rows (users x order lines), not users. Daily counts differ by experiment start/stop; do not treat as traffic. SQL: no order row has `order_mst_ts < bucketed_mst_ts` (recore and MKT_SMS_516), i.e. only post-bucketing orders are attributed.
- **Confidence:** Verified (profile + SQL).

### `experiment_user_identifier` - STRING
- **Meaning:** The id TYPE that keys the experiment (which column holds the user key), not an id. One value per experiment. API: config.trafficType.
- **Profile (scope):** 1,826,832 rows; null 0.0%; empty 0.0%; 2 distinct; length 10-12; values_complete=true
- **Values (complete, all 2 distinct non-null values):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `shopper_id` | 1,140,981 | 62.46% | user key is `shopper_id` (MKT_EM_177, agi_plg_email_inline_upgrade_erp, MKT_SMS_516); visitor_guid is NULL on these rows |
| `visitor_guid` | 685,851 | 37.54% | user key is `visitor_guid` (abn_gdexp_14418, recore_abn_gdexp_13936); shopper_id is NULL on these rows |

- **How to use / Traps:** Users = `COUNT(DISTINCT CASE WHEN experiment_user_identifier = 'shopper_id' THEN shopper_id ELSE visitor_guid END)` over ALL rows incl. `order_id IS NULL`. Only 2 of the 10 MCP trafficType values (visitor_guid, shopper_id) appear in the profile; the other trafficType values (care_conversation_id, resource_id, ...) are not observed here.
- **Confidence:** Verified (profile + MCP trafficType enum + verified doc).

### `device_breakpoint_name` - STRING
- **Meaning:** Device class of the user (Hivemind segment dimension copied onto each row; analysis_result_cln calls it `device_name`).
- **Profile (scope):** 1,826,832 rows; null 33.64%; empty 0.0%; 4 distinct; length 5-8; values_complete=true
- **Values (complete, all 4 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `Desktop` | 797,543 | 43.66% | desktop device |
| NULL | 614,505 | 33.64% | no web session segment: exactly the two offsite (email/SMS) experiments MKT_EM_177 + MKT_SMS_516 (588,001 + 26,504 = 614,505); SQL: 0 filled on MKT_SMS_516 |
| `Mobile` | 337,039 | 18.45% | mobile device |
| `Multiple` | 71,962 | 3.94% | user seen on more than one device class (Inferred from the value name) **⚠️ OWNER INPUT NEEDED (OQ-EO-09)** |
| `Other` | 5,783 | 0.32% | other device class (meaning not documented — owner input needed) **⚠️ OWNER INPUT NEEDED (OQ-EO-09)** |
- **How to use / Traps:** Segment cut only. Verified doc: segment cuts here carry no significance and no Hivemind adjustments - do not present per-segment deltas as results. NULL is structural for offsite experiments, not missing data.
- **Confidence:** Verified (profile + SQL; 'Multiple' semantics Inferred).

### `market_site` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-09): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** GoDaddy market/site locale of the user (segment dimension), language-country code like `en-us`.
- **Profile (scope):** 1,826,832 rows; null 33.69%; empty 0.0%; 53 distinct; length 5-8; values_complete=true
- **Values (complete, all 53 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 615,541 | 33.69% | no web segment (offsite experiments, 614,505 rows) plus 1,036 other rows |
| `en-us` | 582,462 | 31.88% | site locale `en-us` (language-country; Inferred from pattern) |
| `en-in` | 138,840 | 7.60% | site locale `en-in` (language-country; Inferred from pattern) |
| `en-ph` | 93,686 | 5.13% | site locale `en-ph` (language-country; Inferred from pattern) |
| `Multiple` | 86,223 | 4.72% | user seen on more than one market site (Inferred from value name) **⚠️ OWNER INPUT NEEDED (OQ-EO-09)** |
| `en-gb` | 73,225 | 4.01% | site locale `en-gb` (language-country; Inferred from pattern) |
| `en-ca` | 49,267 | 2.70% | site locale `en-ca` (language-country; Inferred from pattern) |
| `en-au` | 36,515 | 2.00% | site locale `en-au` (language-country; Inferred from pattern) |
| `en-ie` | 25,318 | 1.39% | site locale `en-ie` (language-country; Inferred from pattern) |
| `en-ae` | 21,692 | 1.19% | site locale `en-ae` (language-country; Inferred from pattern) |
| `en-pk` | 17,643 | 0.97% | site locale `en-pk` (language-country; Inferred from pattern) |
| `es-mx` | 11,088 | 0.61% | site locale `es-mx` (language-country; Inferred from pattern) |
| `pt-br` | 10,859 | 0.59% | site locale `pt-br` (language-country; Inferred from pattern) |
| `en-za` | 8,179 | 0.45% | site locale `en-za` (language-country; Inferred from pattern) |
| `es-us` | 6,096 | 0.33% | site locale `es-us` (language-country; Inferred from pattern) |
| `tr-tr` | 5,036 | 0.28% | site locale `tr-tr` (language-country; Inferred from pattern) |
| `de-de` | 4,278 | 0.23% | site locale `de-de` (language-country; Inferred from pattern) |
| `fr-fr` | 3,893 | 0.21% | site locale `fr-fr` (language-country; Inferred from pattern) |
| `es-es` | 3,219 | 0.18% | site locale `es-es` (language-country; Inferred from pattern) |
| `es-co` | 3,165 | 0.17% | site locale `es-co` (language-country; Inferred from pattern) |
| `en-sg` | 2,341 | 0.13% | site locale `en-sg` (language-country; Inferred from pattern) |
| `nl-nl` | 1,991 | 0.11% | site locale `nl-nl` (language-country; Inferred from pattern) |
| `en-my` | 1,987 | 0.11% | site locale `en-my` (language-country; Inferred from pattern) |
| `it-it` | 1,837 | 0.10% | site locale `it-it` (language-country; Inferred from pattern) |
| `en-il` | 1,607 | 0.09% | site locale `en-il` (language-country; Inferred from pattern) |
| `id-id` | 1,486 | 0.08% | site locale `id-id` (language-country; Inferred from pattern) |
| `zh-sg` | 1,437 | 0.08% | site locale `zh-sg` (language-country; Inferred from pattern) |
| `es-pe` | 1,423 | 0.08% | site locale `es-pe` (language-country; Inferred from pattern) |
| `zh-hk` | 1,264 | 0.07% | site locale `zh-hk` (language-country; Inferred from pattern) |
| `en-nz` | 1,153 | 0.06% | site locale `en-nz` (language-country; Inferred from pattern) |
| `fr-ca` | 1,116 | 0.06% | site locale `fr-ca` (language-country; Inferred from pattern) |
| `es-ar` | 1,114 | 0.06% | site locale `es-ar` (language-country; Inferred from pattern) |
| `zh-tw` | 1,001 | 0.05% | site locale `zh-tw` (language-country; Inferred from pattern) |
| `vi-vn` | 979 | 0.05% | site locale `vi-vn` (language-country; Inferred from pattern) |
| `pl-pl` | 876 | 0.05% | site locale `pl-pl` (language-country; Inferred from pattern) |
| `th-th` | 844 | 0.05% | site locale `th-th` (language-country; Inferred from pattern) |
| `ja-jp` | 833 | 0.05% | site locale `ja-jp` (language-country; Inferred from pattern) |
| `es-cl` | 785 | 0.04% | site locale `es-cl` (language-country; Inferred from pattern) |
| `sv-se` | 782 | 0.04% | site locale `sv-se` (language-country; Inferred from pattern) |
| `de-ch` | 758 | 0.04% | site locale `de-ch` (language-country; Inferred from pattern) |
| `ko-kr` | 717 | 0.04% | site locale `ko-kr` (language-country; Inferred from pattern) |
| `ar-ae` | 583 | 0.03% | site locale `ar-ae` (language-country; Inferred from pattern) |
| `de-at` | 573 | 0.03% | site locale `de-at` (language-country; Inferred from pattern) |
| `nl-be` | 515 | 0.03% | site locale `nl-be` (language-country; Inferred from pattern) |
| `uk-ua` | 473 | 0.03% | site locale `uk-ua` (language-country; Inferred from pattern) |
| `pt-pt` | 472 | 0.03% | site locale `pt-pt` (language-country; Inferred from pattern) |
| `hi-in` | 450 | 0.02% | site locale `hi-in` (language-country; Inferred from pattern) |
| `en-hk` | 441 | 0.02% | site locale `en-hk` (language-country; Inferred from pattern) |
| `nb-no` | 304 | 0.02% | site locale `nb-no` (language-country; Inferred from pattern) |
| `da-dk` | 278 | 0.02% | site locale `da-dk` (language-country; Inferred from pattern) |
| `fr-be` | 76 | 0.00% | site locale `fr-be` (language-country; Inferred from pattern) |
| `fr-ch` | 75 | 0.00% | site locale `fr-ch` (language-country; Inferred from pattern) |
| `it-ch` | 35 | 0.00% | site locale `it-ch` (language-country; Inferred from pattern) |
| `uk-uk` | 1 | 0.00% | single row; not a standard locale code (anomaly, meaning not documented — owner input needed) **⚠️ OWNER INPUT NEEDED (OQ-EO-09)** |
- **How to use / Traps:** Segment cut only (no significance). Lowercase `xx-yy` except `Multiple`. NULL 615,541 rows: 614,505 offsite rows plus 1,036 onsite rows without a market. Not the same as `ip_country_name` (geo-IP) or `order_region_2_name` (order region).
- **Confidence:** Verified (profile; per-locale meaning Inferred from the code pattern).

### `new_existing_customer_name` - STRING
- **Meaning:** New vs existing customer segment of the allocated user. Same values as MCP enum `customerReportFilter`.
- **Profile (scope):** 1,826,832 rows; null 33.64%; empty 0.0%; 2 distinct; length 12-17; values_complete=true
- **Values (complete, all 2 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `Existing Customer` | 782,624 | 42.84% | MCP customerReportFilter value 'Existing Customer' |
| NULL | 614,505 | 33.64% | offsite experiments (MKT_EM_177 + MKT_SMS_516), structural |
| `New Prospect` | 429,703 | 23.52% | MCP customerReportFilter value 'New Prospect' |

- **How to use / Traps:** Not the same as order-level `product_pnl_new_renewal_name` (new purchase vs renewal of a product) or `new_acquisition_flag`. SQL on visitor-keyed abn_gdexp_14418: 392,453 rows New Prospect / 51,758 Existing Customer, while tenure/domain columns there are defaults - so this column is populated for visitor-keyed tests even when shopper attributes are not.
- **Confidence:** Verified (profile + MCP enum + SQL).

### `ip_country_name` - STRING
- **Meaning:** Country from the user's IP geolocation (segment dimension).
- **Profile (scope):** 1,826,832 rows; null 33.67%; empty 0.0%; 233 distinct; length 4-32; values_complete=false
- **Values (not complete; 233 distinct - high cardinality):** top values: NULL (615,151), `United States` (566,934), `India` (152,621), `United Kingdom` (73,224), `Canada` (54,449), `Australia` (38,214), `Multiple` (37,760), `Pakistan` (20,656), `Mexico` (14,428), `Bangladesh` (13,751), `Brazil` (12,188), `United Arab Emirates` (11,837), `South Africa` (10,310), `Philippines` (8,560), `Turkey` (7,492). Full English country names; `Multiple` = more than one country (Inferred). **⚠️ OWNER INPUT NEEDED (OQ-EO-09):** the rule that produces `Multiple` is not confirmed.
- **How to use / Traps:** NULL 615,151 rows, mostly the offsite experiments. Segment only, no significance. Use `market_site` for site, `order_region_2_name` for order region.
- **Confidence:** Verified (profile; 'Multiple' Inferred).

### `customer_tenure_day_segment` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-10): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Customer tenure bucket (days since the shopper became a customer), segment dimension.
- **Profile (scope):** 1,826,832 rows; null 0.0%; empty 0.0%; 4 distinct; length 7-10; values_complete=true
- **Values (complete, all 4 distinct non-null values):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `365+ day` | 1,015,708 | 55.60% | tenure 365 days or more |
| `0-1 day` | 713,491 | 39.06% | tenure 0-1 day - ALSO the fill value on every row of visitor-keyed experiments (see traps) |
| `31-365 day` | 74,596 | 4.08% | tenure 31-365 days |
| `8-30 day` | 23,037 | 1.26% | tenure 8-30 days |

- **How to use / Traps:** Never NULL, but NOT trustworthy on visitor-keyed experiments: SQL shows every row of recore_abn_gdexp_13936 and abn_gdexp_14418 is '0-1 day' (685,851 of the 713,491 '0-1 day' rows), even for 'Existing Customer' visitors - a default, not real tenure. Only use on shopper-keyed experiments. No '2-7 day' bucket appears in this scope (meaning of the gap not documented — owner input needed).
- **Confidence:** Verified (profile + SQL).

### `domain_ownership_count` - STRING
- **Meaning:** Bucketed count of domains the shopper owns (segment dimension). STRING bucket labels, not a number.
- **Profile (scope):** 1,826,832 rows; null 0.0%; empty 0.0%; 5 distinct; length 1-4; values_complete=true
- **Values (complete, all 5 distinct non-null values):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `0` | 695,103 | 38.05% | owns 0 domains - ALSO the fill value on every row of visitor-keyed experiments |
| `10+` | 480,724 | 26.31% | owns more than 10 domains (label 10+; boundary overlap with 6-10 at 10 not documented) |
| `2-5` | 322,082 | 17.63% | owns 2-5 domains |
| `1` | 191,773 | 10.50% | owns 1 domain |
| `6-10` | 137,150 | 7.51% | owns 6-10 domains |

- **How to use / Traps:** Never NULL, but on visitor-keyed experiments every row is '0' (SQL, recore and abn_gdexp_14418: 685,851 of 695,103 '0' rows). Do not cast to int. Sort with a CASE, not alphabetically.
- **Confidence:** Verified (profile + SQL).

### `customer_type` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-09): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Customer type segment of the shopper.
- **Profile (scope):** 1,826,832 rows; null 37.54%; empty 0.0%; 5 distinct; length 7-25; values_complete=true
- **Values (complete, all 5 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 685,856 | 37.54% | visitor-keyed experiments (no shopper resolved; SQL: 100% NULL on recore and abn_gdexp_14418) plus 5 shopper-keyed rows |
| `International Independent` | 387,720 | 21.22% | non-US independent customer (name only; definition not in sources) |
| `US Independent` | 337,111 | 18.45% | US independent customer (name only; definition not in sources) |
| `Investor` | 318,831 | 17.45% | investor customer type (name only; definition not documented — owner input needed) |
| `Partner` | 97,240 | 5.32% | partner customer type (name only; definition not documented — owner input needed) |
| `Not Evaluated` | 74 | 0.00% | customer type not evaluated (meaning not documented — owner input needed) **⚠️ OWNER INPUT NEEDED (OQ-EO-09)** |
- **How to use / Traps:** Segment only. NULL count 685,856 vs shopper_id NULL 685,851.
- **Confidence:** Verified (profile + SQL; value definitions not in sources).

### `scorecard_template_name` - STRING
- **Meaning:** Hivemind scorecard template of the experiment, copied onto every row. API: config.scorecardTemplate.
- **Profile (scope):** 1,826,832 rows; null 0.0%; empty 0.0%; 4 distinct; length 15-30; values_complete=true
- **Values (complete, all 4 distinct non-null values):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `cm-renewals-audience-recording` | 614,505 | 33.64% | MKT_EM_177 + MKT_SMS_516 (588,001 + 26,504) |
| `AGI_Mon_Email_ERP` | 526,476 | 28.82% | agi_plg_email_inline_upgrade_erp |
| `fos-airo-ai-builder` | 444,211 | 24.32% | abn_gdexp_14418 |
| `hp_core_metrics` | 241,640 | 13.23% | recore_abn_gdexp_13936 |

- **How to use / Traps:** One value per experiment; constant within an experiment, so it is never a useful GROUP BY within one experiment. Same field as `experiment_metadata_snap.scorecard_template_name`.
- **Confidence:** Verified (profile + MCP config).

### `bucketed_mst_ts` - TIMESTAMP
- **Meaning:** Timestamp (MST, millisecond precision) at which the user was bucketed.
- **Profile (scope):** 1,826,832 rows; null 0.0%; empty 0.0%; 1,089,171 distinct; min 2026-08-26 14:32:44.729000; max 2026-10-22 10:43:55.785000; values_complete=false
- **Values (not complete; 1,089,171 distinct - high cardinality):** top values: `2026-09-10 09:45:48.503000` (9,201), `2026-09-10 09:45:47.471000` (4,007), `2026-09-10 09:46:15.440000` (2,950), `2026-09-14 09:49:49.955000` (2,540). Range 2026-08-26 14:32:44.729 to 2026-10-22 10:43:55.785.
- **How to use / Traps:** Never NULL. Repeated across all order lines of a user (top value 9,201 rows = one heavy user). SQL: order_mst_ts is never earlier than bucketed_mst_ts. 'MST' = Mountain Standard Time per the column-name convention (Inferred from naming; not documented — owner input needed).
- **Confidence:** Verified (profile + SQL; timezone Inferred).

### `order_visit_guid` - STRING
- **Meaning:** GUID of the web visit (session) in which the order was placed. NULL on non-buyer rows and on orders without a web visit.
- **Profile (scope):** 1,826,832 rows; null 68.76%; empty 0.0%; 266,400 distinct; length 36-36; values_complete=false
- **Values (not complete; 266,400 distinct - high cardinality):** top values: NULL (1,256,138), `8e6511c5-69a2-414c-977a-3069cf4a3fd9` (683), `43f86a32-88c1-42b6-be38-9870fb609492` (322), `39e5a4dc-b662-4369-b1ff-1c8b99f5b009` (267). Format: 36-char UUID.
- **How to use / Traps:** Null 68.76% (vs 49.54% for order_id): about 351k order rows (1,256,138 - 905,031 = 351,107) have no visit (plausibly Billing Agent orders - Inferred, not verified; SQL MKT_SMS_516: 8,149 of 13,258 order rows filled). Not equal to visitor_guid (SQL recore: most order rows differ). Use only to tie an order to a session; not a user key.
- **Confidence:** Inferred (session meaning from name/UUID pattern; fill rates from SQL).

### `order_mst_date` - DATE
- **Meaning:** MST date of the order; date part of `order_mst_ts`. NULL on non-buyer rows.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 57 distinct; min 2026-08-26; max 2026-10-21; values_complete=true
- **Values (complete, all 57 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `2026-09-28` | 46,342 | 2.54% | order lines placed on this MST date |
| `2026-10-05` | 40,932 | 2.24% | order lines placed on this MST date |
| `2026-10-01` | 40,209 | 2.20% | order lines placed on this MST date |
| `2026-09-29` | 37,608 | 2.06% | order lines placed on this MST date |
| `2026-10-02` | 36,999 | 2.03% | order lines placed on this MST date |
| `2026-09-30` | 36,856 | 2.02% | order lines placed on this MST date |
| `2026-10-04` | 31,570 | 1.73% | order lines placed on this MST date |
| `2026-10-03` | 31,175 | 1.71% | order lines placed on this MST date |
| `2026-09-24` | 31,162 | 1.71% | order lines placed on this MST date |
| `2026-09-21` | 30,517 | 1.67% | order lines placed on this MST date |
| `2026-09-25` | 29,595 | 1.62% | order lines placed on this MST date |
| `2026-09-23` | 28,461 | 1.56% | order lines placed on this MST date |
| `2026-09-22` | 28,123 | 1.54% | order lines placed on this MST date |
| `2026-09-27` | 26,221 | 1.44% | order lines placed on this MST date |
| `2026-09-26` | 24,691 | 1.35% | order lines placed on this MST date |
| `2026-09-17` | 23,749 | 1.30% | order lines placed on this MST date |
| `2026-09-18` | 22,415 | 1.23% | order lines placed on this MST date |
| `2026-09-16` | 22,338 | 1.22% | order lines placed on this MST date |
| `2026-09-15` | 22,186 | 1.21% | order lines placed on this MST date |
| `2026-09-14` | 20,926 | 1.15% | order lines placed on this MST date |
| `2026-09-20` | 18,689 | 1.02% | order lines placed on this MST date |
| `2026-10-06` | 18,518 | 1.01% | order lines placed on this MST date |
| `2026-09-19` | 17,826 | 0.98% | order lines placed on this MST date |
| `2026-10-13` | 15,737 | 0.86% | order lines placed on this MST date |
| `2026-10-19` | 15,719 | 0.86% | order lines placed on this MST date |
| `2026-09-11` | 14,807 | 0.81% | order lines placed on this MST date |
| `2026-09-13` | 13,852 | 0.76% | order lines placed on this MST date |
| `2026-10-14` | 13,353 | 0.73% | order lines placed on this MST date |
| `2026-10-15` | 13,285 | 0.73% | order lines placed on this MST date |
| `2026-10-16` | 12,404 | 0.68% | order lines placed on this MST date |
| `2026-09-10` | 12,368 | 0.68% | order lines placed on this MST date |
| `2026-09-12` | 11,838 | 0.65% | order lines placed on this MST date |
| `2026-09-09` | 9,963 | 0.55% | order lines placed on this MST date |
| `2026-09-08` | 9,513 | 0.52% | order lines placed on this MST date |
| `2026-10-18` | 9,080 | 0.50% | order lines placed on this MST date |
| `2026-10-17` | 8,487 | 0.46% | order lines placed on this MST date |
| `2026-09-03` | 8,244 | 0.45% | order lines placed on this MST date |
| `2026-09-07` | 7,970 | 0.44% | order lines placed on this MST date |
| `2026-09-02` | 7,667 | 0.42% | order lines placed on this MST date |
| `2026-09-04` | 7,476 | 0.41% | order lines placed on this MST date |
| `2026-09-01` | 7,378 | 0.40% | order lines placed on this MST date |
| `2026-10-12` | 6,680 | 0.37% | order lines placed on this MST date |
| `2026-08-31` | 6,570 | 0.36% | order lines placed on this MST date |
| `2026-09-06` | 6,049 | 0.33% | order lines placed on this MST date |
| `2026-09-05` | 5,969 | 0.33% | order lines placed on this MST date |
| `2026-08-27` | 4,334 | 0.24% | order lines placed on this MST date |
| `2026-08-28` | 4,306 | 0.24% | order lines placed on this MST date |
| `2026-08-30` | 3,477 | 0.19% | order lines placed on this MST date |
| `2026-10-07` | 3,063 | 0.17% | order lines placed on this MST date |
| `2026-08-29` | 3,019 | 0.17% | order lines placed on this MST date |
| `2026-10-08` | 2,936 | 0.16% | order lines placed on this MST date |
| `2026-10-09` | 2,678 | 0.15% | order lines placed on this MST date |
| `2026-10-11` | 2,354 | 0.13% | order lines placed on this MST date |
| `2026-10-10` | 2,172 | 0.12% | order lines placed on this MST date |
| `2026-08-26` | 1,212 | 0.07% | order lines placed on this MST date |
| `2026-10-20` | 426 | 0.02% | order lines placed on this MST date |
| `2026-10-21` | 307 | 0.02% | order lines placed on this MST date |

- **How to use / Traps:** Max 2026-10-21 in scope; orders after the experiment's last partition are never added (verified doc: partitions freeze ~1 day after last bucketing while Commercial OS keeps accruing).
- **Confidence:** Verified (profile + verified doc).

### `order_mst_ts` - TIMESTAMP
- **Meaning:** Order timestamp (MST). NULL on non-buyer rows.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 517,407 distinct; min 2026-08-26 14:33:17.923000; max 2026-10-21 23:23:23.747000; values_complete=false
- **Values (not complete; 517,407 distinct - high cardinality):** top values: NULL (905,031), `2026-09-17 09:07:32` (418), `2026-09-13 21:04:08` (316), `2026-09-17 10:10:01` (264). Range 2026-08-26 14:33:17.923 to 2026-10-21 23:23:23.747.
- **How to use / Traps:** SQL: never earlier than bucketed_mst_ts; one timestamp per order. Top values are timestamps of large multi-line orders.
- **Confidence:** Verified (profile + SQL).

### `gcr_usd_amt` - NUMERIC
- **Meaning:** Gross cash receipts (GCR) of the order line in USD - raw, unwinsorized, gross. NULL on non-buyer rows. Hivemind scorecard metric_value / Commercial OS `gcr` are winsorized versions of a similar quantity.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 22,587 distinct; min 0.00; max 56000.00; avg 33.7887; values_complete=false
- **Values (not complete; 22,587 distinct - high cardinality):** top values: NULL (905,031), `0.00` (196,439), `11.19` (47,035), `9.99` (46,740), `23.19` (36,126), `13.19` (21,173), `12.19` (17,401), `107.88` (14,239). Range 0.00-56,000.00, avg 33.7887 per order row. No negative values.
- **How to use / Traps:** `SUM(COALESCE(gcr_usd_amt,0))` within ONE partition. Refund rows are POSITIVE (SQL: recore min refund GCR 0.00; MKT_SMS_516 449 of 453 refund rows > 0) - include for gross, exclude `refund_flag = TRUE` for net-of-refund and say which. Does NOT reconcile to Commercial OS: e.g. Commercial OS `winsorization_diagnostics.gcr_raw` for abn_gdexp_14418 control_variant 609,994.61 vs winsorized `gcr` 64,544.73; per verified doc this table sits 0.3% (recore) to 31% (abn_gdexp_14418) below COS gcr_raw. Use for decomposition shares, not totals.
- **Confidence:** Verified (profile + SQL + MCP Commercial OS + verified doc).

### `product_free_trial_flag` - BOOLEAN

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-05): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** TRUE when the order line is a free-trial product line.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 2 distinct; values_complete=true
- **Values (complete, all 2 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `False` | 815,019 | 44.61% | not a free-trial line |
| `True` | 106,782 | 5.85% | free-trial line (Inferred from name); SQL MKT_SMS_516: 566 of 578 such lines have gcr_usd_amt = 0 |

- **How to use / Traps:** SQL agi_plg_email_inline_upgrade_erp: 54,992 TRUE lines are month/1.0 term and most `product_month_qty = 0.23` lines are TRUE. Exclude when counting paid units. Related: product_free_trial_conversion_flag.
- **Confidence:** Inferred (name; zero-GCR association verified by SQL).

### `receipt_price_usd_amt` - NUMERIC
- **Meaning:** Receipt price of the order line in USD. NULL on non-buyer rows.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 17,879 distinct; min 0.00; max 56000.00; avg 33.7887; values_complete=false
- **Values (not complete; 17,879 distinct - high cardinality):** top values: NULL (905,031), `0.00` (204,473), `9.99` (57,811), `11.19` (51,014), `23.19` (38,400), `13.19` (35,957), `12.19` (18,990), `107.88` (14,898). Range 0.00-56,000.00, avg 33.7887 (identical avg to gcr_usd_amt).
- **How to use / Traps:** At ORDER level it matches gcr_usd_amt (SQL MKT_SMS_516: 8,334 of 8,335 orders equal), but it is split across lines differently (2,669 of 13,258 lines differ). Use gcr_usd_amt for GCR; only use this for line-level price questions.
- **Confidence:** Inferred (meaning from name; relation to GCR verified by SQL).

### `pf_id` - INTEGER
- **Meaning:** Product (catalog item) id of the order line. NULL on non-buyer rows.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 4,576 distinct; min 101; max 2031908; avg 871447.1608; values_complete=false
- **Values (not complete; 4,576 distinct - high cardinality):** top values: NULL (905,031), `10101` (121,026), `101` (77,873), `2014475` (37,279), `1317174` (33,309), `1797699` (27,249), `1329767` (22,288), `1677554` (20,146). Integer 101-2,031,908.
- **How to use / Traps:** Join key to the product catalog (not in this table set). pf_id 101 has 77,873 rows, the same as product_name '.COM Domain Name Registration - 1 Year (recurring)' (77,873) - consistent with 101 = that product (Inferred). Prefer product_name / PnL columns for reporting.
- **Confidence:** Inferred (name + count match).

### `upgraded_pf_id` - INTEGER

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-01): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Product id involved in an upgrade on this line (filled only on upgrade lines).
- **Profile (scope):** 1,826,832 rows; null 99.37%; empty 0.0%; 240 distinct; min 875; max 2022676; avg 1365957.1737; values_complete=false
- **Values (not complete; 240 distinct - high cardinality):** top values: NULL (1,815,305), `867688` (1,912), `1556330` (1,766), `1798083` (734), `1556350` (594), `1556298` (394). Integer 875-2,022,676.
- **How to use / Traps:** 99.37% NULL. NULL = not an upgrade line (or non-buyer). Whether this is the product upgraded FROM or TO is not documented — owner input needed.
- **Confidence:** Inferred (name only; direction Unknown).

### `bundle_id` - INTEGER

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-08): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Bundle id when the line was sold as part of a bundle.
- **Profile (scope):** 1,826,832 rows; null 99.86%; empty 0.0%; 2,463 distinct; min 7334904; max 773442276; avg 195744813.6936; values_complete=false
- **Values (not complete; 2,463 distinct - high cardinality):** top values: NULL (1,824,244), `30396643` (3), `639801795` (2), `29997382` (2), `30546047` (2), `30034250` (2). Integer 7,334,904-773,442,276.
- **How to use / Traps:** 99.86% NULL = not bundled. SQL MKT_SMS_516: bundle_id never equals parent_bundle_id on the same row (69 bundle rows, 142 parent rows). Same distinct count (2,463) as parent_bundle_id.
- **Confidence:** Inferred (name; fill pattern from profile/SQL).

### `parent_bundle_id` - INTEGER

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-08): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Parent bundle id of the line (the bundle this line belongs to).
- **Profile (scope):** 1,826,832 rows; null 99.64%; empty 0.0%; 2,463 distinct; min 7334904; max 773442276; avg 159114562.9234; values_complete=false
- **Values (not complete; 2,463 distinct - high cardinality):** top values: NULL (1,820,183), `30396643` (12), `30546047` (8), `30319507` (8), `30604282` (8), `30606258` (8). Integer 7,334,904-773,442,276.
- **How to use / Traps:** 99.64% NULL. Filled on more rows than bundle_id (component lines carry the parent). Relationship semantics not documented — owner input needed.
- **Confidence:** Inferred (name; fill pattern from profile/SQL).

### `product_pnl_line_name` - STRING
- **Meaning:** PnL product line of the order line. NULL on non-buyer rows.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 47 distinct; length 3-28; values_complete=true
- **Values (complete, all 47 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `Domain Name Registration` | 344,712 | 18.87% | PnL product line (name only; definition not in sources) |
| `MS Office 365` | 165,880 | 9.08% | PnL product line (name only; definition not in sources) |
| `Websites and Marketing` | 66,951 | 3.66% | PnL product line (name only; definition not in sources) |
| `Domain Ownership Protection` | 50,585 | 2.77% | PnL product line (name only; definition not in sources) |
| `Airo` | 47,868 | 2.62% | PnL product line (name only; definition not in sources) |
| `Commerce Enabled` | 41,601 | 2.28% | PnL product line (name only; definition not in sources) |
| `Conversations` | 33,379 | 1.83% | PnL product line (name only; definition not in sources) |
| `Vendor Email` | 32,996 | 1.81% | PnL product line (name only; definition not in sources) |
| `Value Adds` | 26,980 | 1.48% | PnL product line (name only; definition not in sources) |
| `Aftermarket Non-Retail` | 26,979 | 1.48% | PnL product line (name only; definition not in sources) |
| `Domain Name Transfer` | 18,490 | 1.01% | PnL product line (name only; definition not in sources) |
| `Domain Name Auction` | 12,014 | 0.66% | PnL product line (name only; definition not in sources) |
| `AI Credit` | 10,313 | 0.56% | PnL product line (name only; definition not in sources) |
| `Redemption` | 9,741 | 0.53% | PnL product line (name only; definition not in sources) |
| `CnP Hosting` | 8,711 | 0.48% | PnL product line (name only; definition not in sources) |
| `WordPress Managed Plans` | 6,171 | 0.34% | PnL product line (name only; definition not in sources) |
| `SSL` | 3,621 | 0.20% | PnL product line (name only; definition not in sources) |
| `Website Protection` | 3,214 | 0.18% | PnL product line (name only; definition not in sources) |
| `Domain Marketplace` | 2,497 | 0.14% | PnL product line (name only; definition not in sources) |
| `Virtual Hosting` | 1,340 | 0.07% | PnL product line (name only; definition not in sources) |
| `Cashparking` | 968 | 0.05% | PnL product line (name only; definition not in sources) |
| `cPanel Business Hosting` | 862 | 0.05% | PnL product line (name only; definition not in sources) |
| `Paid IT` | 643 | 0.04% | PnL product line (name only; definition not in sources) |
| `Domain Buyers Club` | 601 | 0.03% | PnL product line (name only; definition not in sources) |
| `Miscellaneous Fees and Other` | 571 | 0.03% | PnL product line (name only; definition not in sources) |
| `Strategic Partnerships AC` | 535 | 0.03% | PnL product line (name only; definition not in sources) |
| `InstantPage` | 502 | 0.03% | PnL product line (name only; definition not in sources) |
| `Express Email Marketing` | 460 | 0.03% | PnL product line (name only; definition not in sources) |
| `Website Builder` | 402 | 0.02% | PnL product line (name only; definition not in sources) |
| `Domain Name Premium` | 382 | 0.02% | PnL product line (name only; definition not in sources) |
| `Paid Support` | 364 | 0.02% | PnL product line (name only; definition not in sources) |
| `Commerce Hardware` | 299 | 0.02% | PnL product line (name only; definition not in sources) |
| `Professional Web Services` | 297 | 0.02% | PnL product line (name only; definition not in sources) |
| `Premium DNS` | 177 | 0.01% | PnL product line (name only; definition not in sources) |
| `Open XChange` | 139 | 0.01% | PnL product line (name only; definition not in sources) |
| `Search Engine Visibility` | 127 | 0.01% | PnL product line (name only; definition not in sources) |
| `Consolidation` | 89 | 0.00% | PnL product line (name only; definition not in sources) |
| `DIFY Social` | 87 | 0.00% | PnL product line (name only; definition not in sources) |
| `Dedicated IP` | 85 | 0.00% | PnL product line (name only; definition not in sources) |
| `WooSaas` | 39 | 0.00% | PnL product line (name only; definition not in sources) |
| `Reseller` | 34 | 0.00% | PnL product line (name only; definition not in sources) |
| `Grid` | 34 | 0.00% | PnL product line (name only; definition not in sources) |
| `Domains by Proxy` | 31 | 0.00% | PnL product line (name only; definition not in sources) |
| `Hardware Rental` | 22 | 0.00% | PnL product line (name only; definition not in sources) |
| `Email` | 4 | 0.00% | PnL product line (name only; definition not in sources) |
| `ISC` | 3 | 0.00% | PnL product line (name only; definition not in sources) |
| `Dedicated Hosting` | 1 | 0.00% | PnL product line (name only; definition not in sources) |

- **How to use / Traps:** Main grouping for product-line GCR decomposition (use-case Path C): `COALESCE(product_pnl_line_name,'(none)')`. Many lines have few rows - always show order counts next to GCR. Hierarchy: group > category > line > subline > version (Inferred from names).
- **Confidence:** Verified (profile + verified doc).

### `product_unit_qty` - INTEGER
- **Meaning:** Units on the order line. NULL on non-buyer rows.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 354 distinct; min 1; max 4000; avg 1.357; values_complete=false
- **Values (not complete; 354 distinct - high cardinality):** top values: `1` (909,137), NULL (905,031), `2` (5,118), `3` (2,060), `4` (799), `5` (655), `6` (336), `7` (236). Integer 1-4,000, avg 1.357.
- **How to use / Traps:** SUM for units. Raw (Commercial OS units are winsorized; COS bounds e.g. units upper bound 12-14 on recore). Never 0 on order rows.
- **Confidence:** Verified (profile + verified doc).

### `product_month_qty` - NUMERIC
- **Meaning:** Term length in months of the line. NULL on non-buyer rows.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 2,926 distinct; min 0.00; max 48000.00; avg 13.4352; values_complete=false
- **Values (not complete; 2,926 distinct - high cardinality):** top values: NULL (905,031), `12.00` (475,574), `1.00` (323,621), `24.00` (36,590), `36.00` (28,255), `3.00` (7,534), `0.23` (5,158), `60.00` (5,075), `48.00` (1,308), `120.00` (1,004). Range 0.00-48,000.00, avg 13.4352.
- **How to use / Traps:** Pairs with product_period_name (SQL agi: year->12/24/36/48/60, month->1). 0.23 (~1 week) is mostly free-trial lines. Outliers exist (max 48,000; 'month' lines with 12.00). AVG it, never SUM across lines.
- **Confidence:** Verified (profile + SQL + verified doc).

### `product_pnl_new_renewal_name` - STRING
- **Meaning:** New purchase vs renewal classification of the line. NULL on non-buyer rows.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 2 distinct; length 7-12; values_complete=true
- **Values (complete, all 2 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `New Purchase` | 530,091 | 29.02% | new purchase (exact value; 'New' returns zero rows) |
| `Renewal` | 391,710 | 21.44% | renewal |

- **How to use / Traps:** Filter on exact `'New Purchase'` / `'Renewal'`. Not the same as new_existing_customer_name. Differs from original_product_pnl_new_renewal_name on a small number of rows (see that column).
- **Confidence:** Verified (profile + verified doc).

### `purchase_path_name` - STRING
- **Meaning:** Purchase path (site flow) through which the line was bought.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 29 distinct; length 4-27; values_complete=true
- **Values (complete, all 29 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `Unknown` | 575,400 | 31.50% | purchase path not attributed (575,400 rows - the largest bucket; includes Billing Agent auto-renewals per SQL on agi) |
| `Domain Purchase Path` | 156,376 | 8.56% | purchase path (name only; definition not in sources) |
| `Single Landing Page` | 34,323 | 1.88% | purchase path (name only; definition not in sources) |
| `Cart` | 30,998 | 1.70% | purchase path (name only; definition not in sources) |
| `Renewals` | 29,988 | 1.64% | purchase path (name only; definition not in sources) |
| `DNA Purchase Path` | 26,942 | 1.47% | purchase path (name only; definition not in sources) |
| `Productivity - O365` | 16,475 | 0.90% | purchase path (name only; definition not in sources) |
| `My Account` | 12,998 | 0.71% | purchase path (name only; definition not in sources) |
| `Applications` | 11,322 | 0.62% | purchase path (name only; definition not in sources) |
| `C3 Sales Site` | 8,253 | 0.45% | purchase path (name only; definition not in sources) |
| `Domain Control Center` | 6,548 | 0.36% | purchase path (name only; definition not in sources) |
| `P & C - vNext` | 2,558 | 0.14% | purchase path (name only; definition not in sources) |
| `Domains - DCC 6` | 2,157 | 0.12% | purchase path (name only; definition not in sources) |
| `Mobile Apps - ios` | 1,996 | 0.11% | purchase path (name only; definition not in sources) |
| `Dynamic Landing Page` | 1,555 | 0.09% | purchase path (name only; definition not in sources) |
| `Migration` | 1,264 | 0.07% | purchase path (name only; definition not in sources) |
| `Mobile Apps - Android` | 1,093 | 0.06% | purchase path (name only; definition not in sources) |
| `After Checkout Special` | 442 | 0.02% | purchase path (name only; definition not in sources) |
| `Aftermarket - Cash Parking` | 322 | 0.02% | purchase path (name only; definition not in sources) |
| `Marketplace` | 316 | 0.02% | purchase path (name only; definition not in sources) |
| `Hosting UI` | 180 | 0.01% | purchase path (name only; definition not in sources) |
| `Homepage Quickbuy` | 139 | 0.01% | purchase path (name only; definition not in sources) |
| `GoDaddy Pro - Pro Dashboard` | 62 | 0.00% | purchase path (name only; definition not in sources) |
| `Hosting - Gateway` | 54 | 0.00% | purchase path (name only; definition not in sources) |
| `Hosting - Shared - cPanel` | 14 | 0.00% | purchase path (name only; definition not in sources) |
| `Web Pro Microsite` | 12 | 0.00% | purchase path (name only; definition not in sources) |
| `MYA Home page` | 8 | 0.00% | purchase path (name only; definition not in sources) |
| `Security - Malware` | 5 | 0.00% | purchase path (name only; definition not in sources) |
| `P & C - WSB 7` | 1 | 0.00% | purchase path (name only; definition not in sources) |

- **How to use / Traps:** 'Unknown' is a real value, not NULL. SQL recore: dpp_order_flag = TRUE mostly on 'Domain Purchase Path'.
- **Confidence:** Inferred (name; values from profile).

### `item_tracking_code` - STRING
- **Meaning:** Item tracking code (ITC) - code of the page/placement that added the item to cart.
- **Profile (scope):** 1,826,832 rows; null 68.78%; empty 0.0%; 672 distinct; length 6-50; values_complete=false
- **Values (not complete; 672 distinct - high cardinality):** top values: NULL (1,256,493), `dpp_absol1.primary_exact` (57,401), `onboarding` (41,601), `dcc_portfolio_renew_domain` (38,984), `bulk_search` (21,629), `slp_airoaibuilder` (19,708), `mya_dom_srch.primary_exact` (19,180), `dcc_portfolio_renewal_dop_add_cart` (17,490), `dpp_precheck` (15,156), `upp_d2p_venture_email_buy_more` (14,342). Lowercase snake/dot codes up to 50 chars.
- **How to use / Traps:** 68.78% NULL (non-buyers plus lines without an ITC, e.g. auto-renewals). Prefixes look like surfaces (`dpp_` domain purchase path, `dcc_` domain control center, `mya_` my account, `slp_` single landing page) - Inferred. Case-sensitive.
- **Confidence:** Inferred (name/pattern).

### `new_acquisition_flag` - BOOLEAN

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-03): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** TRUE when the line is flagged as a new customer acquisition.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 2 distinct; values_complete=true
- **Values (complete, all 2 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `False` | 916,218 | 50.15% | not a new acquisition |
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `True` | 5,583 | 0.31% | new-acquisition line (Inferred from name) |

- **How to use / Traps:** Only 5,583 TRUE rows. Exact business definition not in sources; do not use as a substitute for new_existing_customer_name.
- **Confidence:** Inferred (name only).

### `domain_bulk_flag` - BOOLEAN

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-07): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** TRUE when the line was part of a bulk domain purchase.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 2 distinct; values_complete=true
- **Values (complete, all 2 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `False` | 899,872 | 49.26% | not bulk |
| `True` | 21,929 | 1.20% | bulk domain line (Inferred; ITC `bulk_search` exists) |

- **How to use / Traps:** SQL recore: 310 TRUE rows (29 orders) on 'Domain Purchase Path' - bulk orders have many lines per order, so count orders not rows.
- **Confidence:** Inferred (name + SQL pattern).

### `crm_portfolio_type_name` - STRING
- **Meaning:** CRM portfolio (account-management program) the shopper belongs to.
- **Profile (scope):** 1,826,832 rows; null 93.06%; empty 0.0%; 20 distinct; length 2-35; values_complete=true
- **Values (complete, all 20 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 1,700,141 | 93.06% | no CRM portfolio (or non-buyer row) |
| `Premier Services - Platinum` | 40,076 | 2.19% | CRM portfolio type (name only; definition not in sources) |
| `Business Care Team` | 28,810 | 1.58% | CRM portfolio type (name only; definition not in sources) |
| `Premier Services - Gold` | 24,555 | 1.34% | CRM portfolio type (name only; definition not in sources) |
| `Premier Services - China` | 8,638 | 0.47% | CRM portfolio type (name only; definition not in sources) |
| `Business Care Team Leads` | 4,953 | 0.27% | CRM portfolio type (name only; definition not in sources) |
| `Premier Services - APAC` | 4,773 | 0.26% | CRM portfolio type (name only; definition not in sources) |
| `Premier Services - India` | 3,863 | 0.21% | CRM portfolio type (name only; definition not in sources) |
| `NameFind` | 2,950 | 0.16% | CRM portfolio type (name only; definition not in sources) |
| `Premier Services - EMEA` | 2,546 | 0.14% | CRM portfolio type (name only; definition not in sources) |
| `Business Care Team - India` | 2,278 | 0.12% | CRM portfolio type (name only; definition not in sources) |
| `Business Care Elite` | 1,301 | 0.07% | CRM portfolio type (name only; definition not in sources) |
| `Top Tier WebPro` | 856 | 0.05% | CRM portfolio type (name only; definition not in sources) |
| `Premier Services - Diamond` | 372 | 0.02% | CRM portfolio type (name only; definition not in sources) |
| `Email Sales` | 266 | 0.01% | CRM portfolio type (name only; definition not in sources) |
| `CN` | 213 | 0.01% | CRM portfolio type (name only; definition not in sources) |
| `Managed Care - Commerce PowerSeller` | 88 | 0.00% | CRM portfolio type (name only; definition not in sources) |
| `Premier Services - Ruby` | 67 | 0.00% | CRM portfolio type (name only; definition not in sources) |
| `Global Commerce Merchant` | 48 | 0.00% | CRM portfolio type (name only; definition not in sources) |
| `Managed Care - Advanced Hosting` | 35 | 0.00% | CRM portfolio type (name only; definition not in sources) |
| `Super User` | 3 | 0.00% | CRM portfolio type (name only; definition not in sources) |

- **How to use / Traps:** 93.06% NULL. Use only to flag managed/premier accounts.
- **Confidence:** Inferred (name).

### `order_region_2_name` - STRING
- **Meaning:** Region rollup of the order (level-2 region).
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 8 distinct; length 5-19; values_complete=true
- **Values (complete, all 8 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `United States` | 434,861 | 23.80% | order region United States |
| `Rest of World (RoW)` | 331,906 | 18.17% | countries not broken out as their own region (Inferred from the label; definition not documented — owner input needed) |
| `India` | 44,265 | 2.42% | order region India |
| `United Kingdom` | 41,798 | 2.29% | order region United Kingdom |
| `Canada` | 34,034 | 1.86% | order region Canada |
| `Australia` | 24,689 | 1.35% | order region Australia |
| `China` | 7,404 | 0.41% | order region China |
| `Germany` | 2,844 | 0.16% | order region Germany |

- **How to use / Traps:** Order-level geography; differs from ip_country_name (user IP) and market_site.
- **Confidence:** Inferred (name; values from profile).

### `point_of_purchase_name` - STRING
- **Meaning:** Channel through which the order was placed.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 3 distinct; length 2-13; values_complete=true
- **Values (complete, all 3 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `Web` | 593,528 | 32.49% | web/self-serve purchase |
| `Billing Agent` | 286,389 | 15.68% | automated billing purchase (Inferred from name; auto-renewal reading not documented — owner input needed) |
| `C3` | 41,884 | 2.29% | rep-assisted (C3) purchase (Inferred from name and c3_rep_department_name; not documented — owner input needed) |

- **How to use / Traps:** SQL agi: 'Billing Agent' rows always have purchase_path_name 'Unknown' and c3_rep_department_name 'Billing Agent'. Exclude 'Billing Agent' to isolate customer-initiated purchases. Value meanings Inferred from names.
- **Confidence:** Inferred (names; co-occurrence verified by SQL).

### `margin_gcr_usd_amt` - NUMERIC

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-02): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Margin GCR of the line in USD (GCR net of some cost component).
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 22,139 distinct; min -10750.00; max 56000.00; avg 24.9446; values_complete=false
- **Values (not complete; 22,139 distinct - high cardinality):** top values: NULL (905,031), `0.00` (196,405), `0.73` (46,986), `12.73` (35,836), `9.99` (24,170), `-0.47` (21,456), `2.73` (18,005), `1.73` (14,904). Range -10,750.00 to 56,000.00, avg 24.9446.
- **How to use / Traps:** Can be negative (SQL recore: 8,494 order rows < 0). Formula not documented — owner input needed; do not use as GCR.
- **Confidence:** Inferred (name; formula Unknown).

### `dpp_order_flag` - BOOLEAN

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-06): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** TRUE when the order came through the Domain Purchase Path (DPP).
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 2 distinct; values_complete=true
- **Values (complete, all 2 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `False` | 745,919 | 40.83% | not a DPP order |
| `True` | 175,882 | 9.63% | DPP order (Inferred; SQL recore: mostly purchase_path_name = Domain Purchase Path) |

- **How to use / Traps:** Order-level flag repeated on all lines. 175,882 TRUE rows.
- **Confidence:** Inferred (name + SQL).

### `original_product_pnl_new_renewal_name` - STRING
- **Meaning:** New/renewal classification before any reclassification (original value).
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 2 distinct; length 7-12; values_complete=true
- **Values (complete, all 2 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `New Purchase` | 518,767 | 28.40% | originally new purchase |
| `Renewal` | 403,034 | 22.06% | originally renewal |

- **How to use / Traps:** SQL MKT_SMS_516: differs from product_pnl_new_renewal_name on 72 rows, all original 'Renewal' -> current 'New Purchase'. Use product_pnl_new_renewal_name (the verified doc's column) for reporting.
- **Confidence:** Inferred (name; difference verified by SQL).

### `monthly_recurring_revenue_usd_amt` - NUMERIC

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-11): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Monthly recurring revenue of the line - UNPOPULATED in scope.
- **Profile (scope):** 1,826,832 rows; null 0.0%; empty 0.0%; 1 distinct; min 0.000000; max 0.000000; avg 0.0; values_complete=true
- **Values (complete, all 1 distinct non-null values):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `0.000000` | 1,826,832 | 100.00% | constant zero on every row, including non-buyer rows (never NULL) |

- **How to use / Traps:** Do not use: 100% = 0. Never NULL even on non-buyer rows.
- **Confidence:** Verified (profile).

### `product_free_trial_conversion_flag` - BOOLEAN

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-05): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** TRUE when the line is a conversion of a free trial to paid.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 2 distinct; values_complete=true
- **Values (complete, all 2 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `False` | 910,416 | 49.84% | not a conversion |
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `True` | 11,385 | 0.62% | free-trial conversion line (Inferred from name) |

- **How to use / Traps:** 11,385 TRUE rows. SQL agi: TRUE mainly on paid month/year lines, rarely on free-trial lines.
- **Confidence:** Inferred (name).

### `product_pnl_subline_name` - STRING
- **Meaning:** PnL subline under product_pnl_line_name.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 723 distinct; length 2-53; values_complete=false
- **Values (not complete; 723 distinct - high cardinality):** top values: NULL (905,031), `COM` (256,686), `Free` (66,253), `Email Essentials - EE` (46,810), `PMail` (43,448), `e-store` (41,601), `Domain Ownership Protection` (39,202), `Freemium` (33,530), `Business Premium` (32,053), `Light` (27,148).
- **How to use / Traps:** Fine-grained (723 distinct); cells are small - show order counts. NULL = non-buyer.
- **Confidence:** Verified (profile + verified doc).

### `product_pnl_group_name` - STRING
- **Meaning:** Top-level PnL group of the line.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 5 distinct; length 2-31; values_complete=true
- **Values (complete, all 5 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `Domains` | 467,266 | 25.58% | PnL group (name only) |
| `Business Applications and Other` | 261,621 | 14.32% | PnL group (name only) |
| `Presence and Commerce` | 168,469 | 9.22% | PnL group (name only) |
| `Hosting` | 24,442 | 1.34% | PnL group (name only) |
| `NA` | 3 | 0.00% | not assigned (3 rows; meaning not documented — owner input needed) **⚠️ OWNER INPUT NEEDED (OQ-EO-09)** |
- **How to use / Traps:** Coarsest product rollup. 'NA' is a string value, not NULL.
- **Confidence:** Inferred (name; hierarchy inferred).

### `product_pnl_category_name` - STRING
- **Meaning:** PnL category (between group and line).
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 15 distinct; length 2-28; values_complete=true
- **Values (complete, all 15 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `Domain Registration` | 385,339 | 21.09% | PnL category (name only) |
| `Workspace` | 225,999 | 12.37% | PnL category (name only) |
| `Presence` | 126,036 | 6.90% | PnL category (name only) |
| `Domain Add Ons` | 52,451 | 2.87% | PnL category (name only) |
| `Commerce` | 41,922 | 2.29% | PnL category (name only) |
| `Business Tools` | 34,374 | 1.88% | PnL category (name only) |
| `Aftermarket` | 29,476 | 1.61% | PnL category (name only) |
| `Web Hosting` | 15,253 | 0.83% | PnL category (name only) |
| `Security` | 6,835 | 0.37% | PnL category (name only) |
| `Other Hosting` | 2,354 | 0.13% | PnL category (name only) |
| `IT Services` | 643 | 0.04% | PnL category (name only) |
| `Miscellaneous Fees and Other` | 605 | 0.03% | PnL category (name only) |
| `Site Building` | 297 | 0.02% | PnL category (name only) |
| `Engage Customers` | 214 | 0.01% | PnL category (name only) |
| `NA` | 3 | 0.00% | not assigned (3 rows) **⚠️ OWNER INPUT NEEDED (OQ-EO-09)** |
- **How to use / Traps:** 'NA' is a string value, not NULL.
- **Confidence:** Inferred (name; hierarchy inferred).

### `product_pnl_version_name` - STRING
- **Meaning:** PnL version (below subline), e.g. TLD group or product version.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 112 distinct; length 3-29; values_complete=false
- **Values (not complete; 112 distinct - high cardinality):** top values: NULL (905,031), `COM` (256,722), `MS Office 365` (160,103), `vNext` (58,938), `AI Builder` (57,884), `Domain Ownership Protection` (50,563), `Geo ccTLDs` (43,254), `New gTLDs` (36,883), `Non-COM Legacy gTLDs` (33,267), `Titan Email` (32,996).
- **How to use / Traps:** 112 distinct; NULL = non-buyer.
- **Confidence:** Inferred (name).

### `product_name` - STRING
- **Meaning:** Catalog product name of the line.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 4,419 distinct; length 3-122; values_complete=false
- **Values (not complete; 4,419 distinct - high cardinality):** top values: NULL (905,031), `.COM Domain Name Renewal - 1 Year (recurring)` (131,866), `.COM Domain Name Registration - 1 Year (recurring)` (77,873), `GoDaddy Airo AI Builder Free Plan - 1 month(s)` (37,279), `Websites + Marketing Free` (33,530), `Conversations Free` (27,706), `Lite DOP EP-39049 Lite DOP Creation - 1 year(s)` (22,288), `GoDaddy Payments` (22,047). Free text up to 122 chars, e.g. `.COM Domain Name Renewal - 1 Year (recurring)`.
- **How to use / Traps:** 4,419 distinct; includes term and campaign text. Use PnL columns for grouping; product_name for drill-down. Case-sensitive.
- **Confidence:** Verified (profile).

### `product_period_name` - STRING
- **Meaning:** Term unit of the line.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 5 distinct; length 4-7; values_complete=true
- **Values (complete, all 5 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `year` | 567,479 | 31.06% | annual term(s) |
| `month` | 273,527 | 14.97% | monthly term |
| `onetime` | 73,618 | 4.03% | one-time purchase |
| `quarter` | 6,878 | 0.38% | quarterly term |
| `6-month` | 299 | 0.02% | semi-annual term |

- **How to use / Traps:** Lowercase exact values. Pair with product_month_qty for term length.
- **Confidence:** Verified (profile + verified doc).

### `virtual_order_flag` - BOOLEAN

> ⚠️ **OWNER INPUT NEEDED** (OQ-EO-04): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** TRUE for virtual (non-cash) order lines.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 2 distinct; values_complete=true
- **Values (complete, all 2 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `False` | 740,065 | 40.51% | regular line |
| `True` | 181,736 | 9.95% | virtual line; SQL agi: every TRUE line has gcr_usd_amt = 0 |

- **How to use / Traps:** Exact business meaning not in sources; zero-GCR behaviour verified. Exclude when counting paid orders.
- **Confidence:** Inferred (name; zero-GCR verified by SQL).

### `refund_flag` - BOOLEAN
- **Meaning:** TRUE when the line is a refund row.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 2 distinct; values_complete=true
- **Values (complete, all 2 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `False` | 909,792 | 49.80% | not refunded |
| NULL | 905,031 | 49.54% | non-buyer row (allocated user with no order, `order_id IS NULL`); not FALSE |
| `True` | 12,009 | 0.66% | refund line - gcr_usd_amt is POSITIVE (gross), not a reversal |

- **How to use / Traps:** Include for gross raw GCR, exclude (`refund_flag = FALSE`) for net-of-refund, and state which (verified doc). SQL MKT_SMS_516: 453 refund rows carry 36,234.89 GCR.
- **Confidence:** Verified (profile + SQL + verified doc).

### `web_pro_shopper_id` - STRING
- **Meaning:** Shopper id of the GoDaddy Pro (web professional) linked to the purchase.
- **Profile (scope):** 1,826,832 rows; null 96.72%; empty 0.0%; 8,572 distinct; length 3-9; values_complete=false
- **Values (not complete; 8,572 distinct - high cardinality):** top values: NULL (1,766,987), `435486675` (565), `225044794` (546), `130077561` (485), `41645564` (441), `108043610` (352). Numeric string 3-9 chars.
- **How to use / Traps:** 96.72% NULL = no Pro involved. Not a user key for the experiment.
- **Confidence:** Inferred (name).

### `web_pro_participant_type_name` - STRING
- **Meaning:** Role of the purchaser in a GoDaddy Pro relationship.
- **Profile (scope):** 1,826,832 rows; null 89.21%; empty 0.0%; 2 distinct; length 3-6; values_complete=true
- **Values (complete, all 2 distinct non-null values + NULL):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| NULL | 1,629,641 | 89.21% | no Pro relationship (or non-buyer) |
| `Pro` | 136,486 | 7.47% | purchaser is the web pro (Inferred) |
| `Client` | 60,705 | 3.32% | purchaser is a Pro's client (Inferred) |

- **How to use / Traps:** 89.21% NULL. Filled on more rows than web_pro_shopper_id.
- **Confidence:** Inferred (name).

### `c3_rep_department_name` - STRING
- **Meaning:** Department of the rep (C3) credited with the order, or the channel name when no rep.
- **Profile (scope):** 1,826,832 rows; null 49.54%; empty 0.0%; 110 distinct; length 3-37; values_complete=false
- **Values (not complete; 110 distinct - high cardinality):** top values: NULL (905,031), `Web` (530,609), `Billing Agent` (286,389), `Premier Services - Platinum` (31,114), `Premier Services - Gold` (25,681), `GD Enterprise - BCT Premier Services` (7,293), `Inbound Sales III` (4,343), `Messaging - Sales` (3,536), `Messaging - India General` (3,493), `Inbound Sales II` (2,587).
- **How to use / Traps:** 'Web' (530,609) and 'Billing Agent' (286,389) mirror point_of_purchase_name; the remaining ~104,803 order rows carry rep departments (more than the 41,884 'C3' rows, so some 'Web' orders also have a rep department). NULL = non-buyer.
- **Confidence:** Inferred (name; counts from profile).

### `cln_create_utc_ts` - TIMESTAMP
- **Meaning:** ETL load timestamp (UTC) of the partition.
- **Profile (scope):** 1,826,832 rows; null 0.0%; empty 0.0%; 5 distinct; min 2026-10-07 01:47:40.774949; max 2026-10-23 01:47:11.985502; values_complete=true
- **Values (complete, all 5 distinct non-null values):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `2026-10-07 01:47:40.774949` | 588,001 | 32.19% | load of partition 2026-10-07 (MKT_EM_177) |
| `2026-10-08 01:33:52.560775` | 526,476 | 28.82% | load of partition 2026-10-08 (agi_plg_email_inline_upgrade_erp) |
| `2026-10-15 01:34:55.011011` | 444,211 | 24.32% | load of partition 2026-10-15 (abn_gdexp_14418) |
| `2026-10-21 01:45:45.304691` | 241,640 | 13.23% | load of partition 2026-10-21 (recore_abn_gdexp_13936) |
| `2026-10-23 01:47:11.985502` | 26,504 | 1.45% | load of partition 2026-10-23 (MKT_SMS_516) |

- **How to use / Traps:** One value per partition, ~01:30-01:50 UTC on the partition date. Audit only; never filter or join on it.
- **Confidence:** Verified (profile; counts match each experiment).

### `record_create_mst_date` - STRING
- **Meaning:** Snapshot partition date, STRING 'YYYY-MM-DD'. CUMULATIVE: each partition re-contains the full history to date.
- **Profile (scope):** 1,826,832 rows; null 0.0%; empty 0.0%; 5 distinct; length 10-10; values_complete=true
- **Values (complete, all 5 distinct non-null values):**

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `2026-10-07` | 588,001 | 32.19% | latest partition of MKT_EM_177_Failed-Billing-Manual-D28_CM-24927 |
| `2026-10-08` | 526,476 | 28.82% | latest partition of agi_plg_email_inline_upgrade_erp |
| `2026-10-15` | 444,211 | 24.32% | latest partition of abn_gdexp_14418 |
| `2026-10-21` | 241,640 | 13.23% | latest partition of recore_abn_gdexp_13936 |
| `2026-10-23` | 26,504 | 1.45% | latest partition of MKT_SMS_516-UPM-D5_CM-24664 |

- **How to use / Traps:** MANDATORY: exactly one literal partition, the latest for the experiment, resolved with `SELECT MAX(record_create_mst_date) FROM hivemind_mart.experiment_order WHERE experiment_id = '<id>' AND record_create_mst_date >= '<last_bucket_date>'` (last_bucket_date = TO_CHAR(experiment_metadata_snap.last_user_bucket_mst_ts,'YYYY-MM-DD')). Summing a range inflates GCR ~10-13x. SQL MKT_SMS_516: rows grow every day (22,226 on 2026-10-15 to 26,504 on 2026-10-23). Compare as a string literal. `user_allocation_history` has NO such column (SQL error).
- **Confidence:** Verified (profile + SQL + verified doc).

## ⚠️ **OWNER INPUT NEEDED** — definitions only the Hivemind owners can supply

The definitions below are **not clear** from the data, Hivemind MCP or any document we could find. **A Hivemind data owner needs to fill in the "Owner answer" column.** Until then, agents must answer "definition not confirmed by the Hivemind owners" for these and must not guess.

| ID | Column / value | Question for the owner | Why it matters | What we observed | Owner answer |
|---|---|---|---|---|---|
| OQ-EO-01 | `upgraded_pf_id` | Is this the product upgraded FROM or TO? | Upgrade analyses. | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-EO-02 | `margin_gcr_usd_amt` | What is the formula (which costs are subtracted)? | Margin questions. | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-EO-03 | `new_acquisition_flag` | Business definition: first-ever purchase by the customer, by product, or in what window? | New vs existing customer splits. | Meaning inferred from the name. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-EO-04 | `virtual_order_flag` | What is a virtual order? | Whether to exclude them from GCR. | Meaning inferred from the name. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-EO-05 | `product_free_trial_flag / product_free_trial_conversion_flag` | Definitions, and whether trial orders carry GCR. | Free-trial and conversion analyses. | Meaning inferred from the name. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-EO-06 | `dpp_order_flag` | What is a DPP order (Domain Purchase Path?), and should it be excluded anywhere? | Domain analyses. | Meaning inferred from the name. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-EO-07 | `domain_bulk_flag` | What makes an order a bulk domain order (threshold)? | Domain analyses / outliers. | Meaning inferred from the name. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-EO-08 | `bundle_id vs parent_bundle_id` | How do the two relate (nesting rule)? | Bundle analyses. | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-EO-09 | `segment values: customer_type 'Not Evaluated', product_pnl_group/category 'NA', market_site 'uk-uk', device 'Other', 'Multiple' in device / market / country` | What does each mean, and what rule produces 'Multiple'? | Segment breakdowns. | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-EO-10 | `customer_tenure_day_segment` | What is the tenure anchor, and does a '2-7 day' bucket exist? | Tenure splits. | Not observed in scope. | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-EO-11 | `monthly_recurring_revenue_usd_amt` | Always 0 in scope. Is it ever populated? | MRR questions. | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-EO-12 | `*_mst_* timestamp columns` | Confirm they are true MST (UTC-7) and not UTC like rollout_mst_ts. | Order-date filters. | Assumed from naming. | _TO BE FILLED BY HIVEMIND OWNER_ |

**Resolved from data (no owner input needed):**

- `allocation_id join to user_allocation_history` — Tested 2026-11-13 on recore_abn_gdexp_13936: 190,324 allocation_ids in both tables, all join, 0 cohort mismatches. allocation_id is a valid user-level join key.

## Sources

- Live column profile: `experiment_order.profile.json` (author's local workspace, not committed) (scope above). Other-table profiles in the same folder for the cross-table map.
- Verified table doc: hivemind_gdc\tables/hivemind_mart.experiment_order.md (traps, mandatory filters, reconciliation figures).
- MCP enums: Hivemind MCP tool input schemas (trafficType, customerReportFilter).
- MCP payloads: saved MCP payloads (author's local workspace) - get_experiment_by_id config (trafficType, baselineCohortId, cohorts, scorecardTemplate, category, status) and get_commercial_os_metrics (cohort_id_dim, sample, gcr, winsorization_diagnostics.gcr_raw) for the 5 profiled experiments.
- Use case: hivemind_gdc\use-cases/experimentation-hm-whats-it-worth.md (Path C product-line decomposition).
- Read-only Redshift SQL run 2026-11-13 on single latest partitions of MKT_SMS_516-UPM-D5_CM-24664, recore_abn_gdexp_13936, abn_gdexp_14418, agi_plg_email_inline_upgrade_erp (fill rates, key uniqueness, refund sign, gcr vs receipt price, segment defaults, partition growth).
