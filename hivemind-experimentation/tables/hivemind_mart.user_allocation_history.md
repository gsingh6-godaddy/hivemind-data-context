---
type: table
entry_id: hivemind_mart.user_allocation_history
table: hivemind_mart.user_allocation_history
synonyms: [user_allocation_history, UAH, experiment allocations, bucketing history, cohort assignments]
status: DETERMINISTIC
lifecycle: active
intents: [schema_lookup, column_lookup, partition_guidance]
grain: one row per analyzed user per experiment_id (deduplicated; bucketed_mst_date = first bucket date)
primary_key: [experiment_id, allocation_id]
partition_column: bucketed_mst_date
partition_filter_idiom: bucketed_mst_date BETWEEN real start − 1 day AND real end + 1 day, plus experiment_id
catalog_source: redshift_svv
column_count: 9
physical_column_count: 20
alation_table_id: not_in_alation
owner_team: not_in_alation
owner_contact: not_in_alation
schema_fetched: 2026-10-22
last_updated: 2026-10-22
verified_on: 2026-10-22
---

# `hivemind_mart.user_allocation_history`

> One row per user that Hivemind analyzed in an experiment, with cohort and bucket date. Used for true per-cohort user counts and real start/end dates.

## Dimensions

| Column | Type | Description | Synonyms | Is Enum | Sample Values |
|--------|------|-------------|----------|---------|---------------|
| `experiment_id` | STRING | Experiment key; joins `experiment_metadata_snap`. Always filter (exact, case-sensitive) together with a `bucketed_mst_date` range. Never NULL. | exp id | no | `recore_abn_gdexp_13936` |
| `cohort_id` | STRING | Cohort (arm). `COUNT(*)` per cohort = the Hivemind cohort sample exactly. Control is NOT flagged here — read `experiment_metadata_snap.baseline_cohort_id`; never infer from names like 'default' or 'off'. | arm, variant | no | `control_variant`, `b_variant`, `default`, `variant1` |
| `bucketed_mst_date` | DATE | Sort key; first bucket date. Always filter. | bucket date | no | — |
| `bucketed_mst_ts` | TIMESTAMP | Bucket timestamp, true MST (MIN / MAX = real start / end, equal to `experiment_metadata_snap.first/last_user_bucket_mst_ts`). | real start, real end | no | — |
| `allocation_id` | STRING | One id per analysed user per experiment (primary key with experiment_id). `COUNT(*) = COUNT(DISTINCT allocation_id)` — the table is already deduplicated, so no DISTINCT is needed for counts. | allocation | no | — |
| `shopper_id` | STRING | Shopper id (NULL on visitor-keyed experiments). | shopper | no | — |
| `visitor_guid` | STRING | Visitor id; the user key on visitor-keyed experiments (filled on every row there). | visitor | no | — |
| `experiment_user_identifier` | STRING | Id TYPE (which column is the user key), not an id. One value per experiment, never NULL; matches `experiment_metadata_snap.traffic_type_enum`. | id type, keyed by | yes | `shopper_id`, `visitor_guid` |
| `scorecard_template_name` | STRING | Scorecard template the users were analysed under; same value as `analysis_result_cln.scorecard_template_name`. Context only — do not filter on it. | scorecard | no | `hp_core_metrics` |

## Facts

| Column | Type | Description | Synonyms | Aggregation | Unit |
|--------|------|-------------|----------|-------------|------|
| (row count) | — | Analyzed users per cohort = `COUNT(*)` per cohort; matches Hivemind cohort sample sizes exactly (verified on 6 experiments, 2026-11-13). One row per user: `COUNT(*) = COUNT(DISTINCT allocation_id)`. | users, sample | COUNT | users |

## Mandatory Filters

| Filter | Expression | Why | Source |
|--------|------------|-----|--------|
| Date bound | `bucketed_mst_date BETWEEN DATE(first_user_bucket_mst_ts) - 1 AND DATE(last_user_bucket_mst_ts) + 1`, dates taken from `experiment_metadata_snap` for the same experiment | sort key; unbounded scans are ~20× slower and time out. Query one experiment per statement (50–100 s each, close to the 2-min timeout) | user_allocation_history performance rule |
| Experiment | `experiment_id = '<id>'` | scope | user_allocation_history performance rule |

## Omitted Columns

| Column Group | Reason |
|--------------|--------|
| segment copies (`device_breakpoint_name`, `market_site`, `new_existing_customer_name`, `ip_country_name`, `customer_tenure_day_segment`, `domain_ownership_count`, `customer_type`), `visit_count`, `wam_free_trial_flag`, `bucket_session_start_mst_ts`, `cln_create_utc_ts` | not needed for committed use-cases; segment cuts from this table have no significance test |

## Related Tables

| Table | Join Keys | Cardinality | Join Type |
|-------|-----------|-------------|-----------|
| `hivemind_mart.experiment_metadata_snap` | `experiment_id` = `experiment_id` | many_to_one | INNER — source of the control cohort (`baseline_cohort_id`; never infer control from names like 'default', 'off') and of the date bounds |

## Never Join

None known.

## Traps

- For triggered experiments this table already IS the triggered/analyzed population (smaller than raw assignments).
- Already deduplicated — cannot be used to measure cross-bucketing.

## ⚠️ **OWNER INPUT NEEDED**

These definitions are **not clear** and must be filled in by a Hivemind data owner: `bucket_session_start_mst_ts` (OQ-UA-01); `wam_free_trial_flag` (OQ-UA-02); `device_breakpoint_name, market_site, ip_country_name 'Other' / 'Multiple'` (OQ-UA-03); `customer_tenure_day_segment / domain_ownership_count` (OQ-UA-04); `customer_type / new_existing_customer_name` (OQ-UA-05); `visit_count` (OQ-UA-06); `user key for non-web traffic types` (OQ-UA-07). Full questions are in `data-dictionary/hivemind_mart.user_allocation_history.dictionary.md`. If a question depends on one of them, answer "definition not confirmed by the Hivemind owners" — do not guess.

## Provenance & Gaps

| Field | Source | Status |
|-------|--------|--------|
| columns | redshift_svv (describe_table 2026-10-22) | populated |
| owner | — | not_in_alation |
