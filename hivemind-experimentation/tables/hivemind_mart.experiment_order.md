---
type: table
entry_id: hivemind_mart.experiment_order
table: hivemind_mart.experiment_order
synonyms: [experiment_order, experiment orders, hivemind orders, experiment gcr by order]
status: DETERMINISTIC
lifecycle: active
intents: [schema_lookup, column_lookup, partition_guidance]
grain: one row per allocated user × order line per experiment_id per record_create_mst_date snapshot (non-buyers appear with order_id NULL)
primary_key: []
partition_column: record_create_mst_date
partition_filter_idiom: experiment_id = '<id>' AND record_create_mst_date = the single latest partition for that experiment (string 'YYYY-MM-DD'); never a range
catalog_source: redshift_svv
column_count: 20
physical_column_count: 57
alation_table_id: not_in_alation
owner_team: not_in_alation
owner_contact: not_in_alation
schema_fetched: 2026-10-22
last_updated: 2026-10-22
verified_on: 2026-10-22
---

# `hivemind_mart.experiment_order`

> Allocated users and their orders for each experiment, with product-line GCR. Analysts reach for it to see which product lines, new vs renewal, and term lengths made up each cohort's GCR.

## Dimensions

| Column | Type | Description | Synonyms | Is Enum | Sample Values |
|--------|------|-------------|----------|---------|---------------|
| `experiment_id` | STRING | Experiment key; joins `experiment_metadata_snap`. Always filter (exact match, case-sensitive) together with a single literal `record_create_mst_date` — never scan without both. Never NULL. | exp id | no | `recore_abn_gdexp_13936` |
| `record_create_mst_date` | STRING | Snapshot partition 'YYYY-MM-DD'. CUMULATIVE: each partition re-contains the full history to date. Partitions stop ~1 day after the last bucketing (`experiment_metadata_snap.last_user_bucket_mst_ts`); orders placed after that are never added. | snapshot date, scorecard date | no | `2026-08-25` |
| `cohort_id` | STRING | Cohort (arm) the user was bucketed into; present on every row incl. non-buyers. Free-text names — the control is `experiment_metadata_snap.baseline_cohort_id`, never inferred from the name. | arm, variant | no | `control_variant`, `b_variant`, `off`, `on` |
| `shopper_id` | STRING | Shopper id. The user key only when `experiment_user_identifier = 'shopper_id'`; 100% NULL on visitor-keyed experiments (expected, not a data gap). | shopper | no | — |
| `visitor_guid` | STRING | Visitor id — the user key when `experiment_user_identifier = 'visitor_guid'` (most onsite tests). Filled on every row of visitor-keyed experiments. Do not COALESCE with shopper_id: logged-in visitors carry both and you would undercount. | visitor, visitor guid | no | — |
| `experiment_user_identifier` | STRING | The id TYPE (which column is the user key), not an id. One value per experiment, filled on every row. User key = `CASE WHEN experiment_user_identifier = 'shopper_id' THEN shopper_id ELSE visitor_guid END`. | id type, keyed by | yes | `shopper_id`, `visitor_guid` |
| `order_id` | STRING | Order id; NULL = allocated user with no order (~64% of rows on a typical onsite test). Keep those rows for the per-user denominator; count orders with `COUNT(DISTINCT order_id)`. | order | no | — |
| `bucketed_mst_date` | DATE | Date the user was bucketed. | — | no | — |
| `order_mst_date` | DATE | Order date. | — | no | — |
| `product_pnl_line_name` | STRING | PnL product line (populated on all order rows; NULL on non-buyer rows — `COALESCE(..., '(none)')` when grouping). Cells with a handful of orders swing on one order; always show order counts. | product line | yes | `Domain Name Registration`, `MS Office 365`, `Domain Ownership Protection`, `Websites and Marketing`, `Commerce Enabled`, `Airo`, `Conversations`, `AI Credit`, `CnP Hosting`, `WordPress Managed Plans` (fine-grained — ~40 lines) |
| `product_pnl_subline_name` | STRING | PnL subline. | subline | no | — |
| `product_pnl_new_renewal_name` | STRING | New vs renewal. Exact values are `'New Purchase'` and `'Renewal'` (NOT 'New' — filtering on 'New' silently returns zero). NULL on non-buyer rows. | new/renewal | yes | `New Purchase`, `Renewal` |
| `product_period_name` | STRING | Term unit. | term, billing period | yes | `year`, `month`, `onetime`, `quarter`, `6-month` |
| `refund_flag` | BOOLEAN | Refund row. `gcr_usd_amt` on these rows is POSITIVE (gross), not a negative reversal — include them for gross raw GCR, exclude them for net-of-refund, and say which you did. | refund | no | — |

## Facts

| Column | Type | Description | Synonyms | Aggregation | Unit |
|--------|------|-------------|----------|-------------|------|
| `gcr_usd_amt` | DECIMAL(18,2) | GCR of the order line (raw, unwinsorized, gross — refund rows are positive). NULL on non-buyer rows: use `COALESCE(gcr_usd_amt, 0)`. Totals can sit below Commercial OS `gcr_raw` (see Traps). | GCR, gross cash receipts | SUM within ONE partition | USD |
| `receipt_price_usd_amt` | DECIMAL(18,2) | Receipt price. | price | SUM | USD |
| `product_unit_qty` | INT | Units. | units | SUM | count |
| `product_month_qty` | DECIMAL(18,2) | Term length in months. | term months | AVG | months |

## Mandatory Filters

| Filter | Expression | Why | Source |
|--------|------------|-----|--------|
| Experiment | `experiment_id = '<id>'` | 2.3B rows; anything wider times out | use_case:experimentation-hm-whats-it-worth |
| One snapshot | `record_create_mst_date = '<latest_partition>'` (a literal), where `<latest_partition>` = `SELECT MAX(record_create_mst_date) FROM hivemind_mart.experiment_order WHERE experiment_id = '<id>' AND record_create_mst_date >= '<last_bucket_date>'` | partitions are cumulative; summing a range inflates GCR ~10–13×. The MAX lookup needs the literal lower bound (Spectrum partition pruning) or it times out | use_case:experimentation-hm-whats-it-worth |

## Metric Expressions

| Metric | Expression | Synonyms | Source |
|--------|------------|----------|--------|
| Users per cohort | `COUNT(DISTINCT CASE WHEN experiment_user_identifier = 'shopper_id' THEN shopper_id ELSE visitor_guid END)` over all rows incl. `order_id IS NULL` | sample, n | use_case:experimentation-hm-whats-it-worth |
| Raw GCR per user | `SUM(gcr_usd_amt) / users` | GCR per user | use_case:experimentation-hm-whats-it-worth |

## Omitted Columns

| Column Group | Reason |
|--------------|--------|
| segment copies (`device_breakpoint_name`, `market_site`, `new_existing_customer_name`, `ip_country_name`, `customer_tenure_day_segment`, `domain_ownership_count`, `customer_type`) | segment cuts not in scope for this context (no significance, Hivemind adjustments missing) |
| order detail (`row_id`, `allocation_id`, `pf_id`, `upgraded_pf_id`, `bundle_id`, `parent_bundle_id`, `item_tracking_code`, `purchase_path_name`, `point_of_purchase_name`, `crm_portfolio_type_name`, `order_region_2_name`, `order_visit_guid`, `order_mst_ts`, `bucketed_mst_ts`, `web_pro_*`, `c3_rep_department_name`, `dpp_order_flag`, `virtual_order_flag`, `domain_bulk_flag`, `new_acquisition_flag`, free-trial flags, `margin_gcr_usd_amt`, `monthly_recurring_revenue_usd_amt`, `original_product_pnl_new_renewal_name`, `product_pnl_group_name`, `product_pnl_category_name`, `product_pnl_version_name`, `product_name`, `scorecard_template_name`, `cln_create_utc_ts`) | not needed for committed use-cases; present on the physical table |

## Related Tables

| Table | Join Keys | Cardinality | Join Type |
|-------|-----------|-------------|-----------|
| `hivemind_mart.experiment_metadata_snap` | `experiment_id` = `experiment_id` | many_to_one | INNER |

## Never Join

| Table | Why Not | What To Do Instead |
|-------|---------|--------------------|
| `hivemind_mart.analysis_result_cln` | different grain; fans out | aggregate each separately by cohort |
| `hivemind_mart.experiment_order` (other experiments) for portfolio totals | the same order sits in ~17 concurrent experiments on median | never add GCR differences across experiments |

## Traps

- Cumulative partitions: use exactly one partition (the latest for the experiment). `BETWEEN` / `>=` on `record_create_mst_date` multiplies GCR.
- Values are raw and there is no significance here. Hivemind scorecard per-user values are winsorized / variance-reduced; the gap is often a few % but can be an order of magnitude on low-conversion experiments (abn_gdexp_14418: winsorized control GCR $64.5k vs raw $610k in Commercial OS). Never compare raw per-user values from this table to scorecard values.
- **Does NOT reconcile to Commercial OS `gcr_raw`.** User counts match the Commercial OS / Hivemind samples exactly, but GCR totals can sit below COS `winsorization_diagnostics.gcr_raw`: 0.3% low on recore_abn_gdexp_13936, 1.4% low on serp_sherpa_display_when_taken_2x, and 31% low on abn_gdexp_14418 (a rolled-back test: partitions stopped 2026-10-15 with orders to 2026-10-13, while COS kept attributing orders through 2026-11-03). Expect a close match for experiments analysed soon after they end and a growing gap when COS keeps accruing orders after the partitions freeze. Use this table for WHERE the gap sits (product line / new-renewal / term shares), not for totals; for a COS-comparable total quote `get_commercial_os_metrics` and label it Commercial OS.
- `<last_bucket_date>` for the partition lookup = `TO_CHAR(last_user_bucket_mst_ts, 'YYYY-MM-DD')` from `experiment_metadata_snap`. Control cohort = `experiment_metadata_snap.baseline_cohort_id` (never infer from the name).
- `experiment_user_identifier` is the id type, not an id. Count users on the id it names; `COALESCE(shopper_id, visitor_guid)` undercounts visitor-keyed experiments because logged-in visitors also carry a shopper_id.
- Scans take 15–100 s per experiment even when filtered. This is a Spectrum table partitioned only by `record_create_mst_date`: any query without a literal date bound on it scans every partition and times out. Resolve the latest partition in its own step (bounded by the last bucketing date), then query that one literal partition.

## ⚠️ **OWNER INPUT NEEDED**

These definitions are **not clear** and must be filled in by a Hivemind data owner: `upgraded_pf_id` (OQ-EO-01); `margin_gcr_usd_amt` (OQ-EO-02); `new_acquisition_flag` (OQ-EO-03); `virtual_order_flag` (OQ-EO-04); `product_free_trial_flag / product_free_trial_conversion_flag` (OQ-EO-05); `dpp_order_flag` (OQ-EO-06); `domain_bulk_flag` (OQ-EO-07); `bundle_id vs parent_bundle_id` (OQ-EO-08); `segment values: customer_type 'Not Evaluated', product_pnl_group/category 'NA', market_site 'uk-uk', device 'Other', 'Multiple' in device / market / country` (OQ-EO-09); `customer_tenure_day_segment` (OQ-EO-10); `monthly_recurring_revenue_usd_amt` (OQ-EO-11); `*_mst_* timestamp columns` (OQ-EO-12). Full questions are in `data-dictionary/hivemind_mart.experiment_order.dictionary.md`. If a question depends on one of them, answer "definition not confirmed by the Hivemind owners" — do not guess.

## Provenance & Gaps

| Field | Source | Status |
|-------|--------|--------|
| columns | redshift_svv (describe_table 2026-10-22) | populated |
| cumulative partition behaviour | execute_query 2026-10-22 (GCR rises monotonically by partition) | populated |
| winsorization bounds | Commercial OS API only | not_in_alation |
| owner | — | not_in_alation |
