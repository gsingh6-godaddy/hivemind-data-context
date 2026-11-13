---
type: table
entry_id: hivemind_mart.experiment_metadata_snap
table: hivemind_mart.experiment_metadata_snap
synonyms: [experiment_metadata_snap, hivemind experiments, hivemind experiment registry, experiment registry, experiment metadata]
status: DETERMINISTIC
lifecycle: active
intents: [schema_lookup, column_lookup, experiment_lookup]
grain: one row per experiment_id (current state, refreshed daily ~D-1)
primary_key: [experiment_id]
partition_column: null
partition_filter_idiom: no partition; full table is ~24k rows and scans in seconds
catalog_source: redshift_svv
column_count: 45
physical_column_count: 73
alation_table_id: 6987788
owner_team: not_in_alation
owner_contact: not_in_alation
schema_fetched: 2026-10-22
last_updated: 2026-10-22
verified_on: 2026-10-22
---

# `hivemind_mart.experiment_metadata_snap`

> One row per registered Hivemind experiment — configuration, owners, real bucketing dates, declared result, badge and governed annualized iGCR. The spine for every single-experiment lookup and every portfolio count.

## Dimensions

| Column | Type | Description | Synonyms | Is Enum | Sample Values |
|--------|------|-------------|----------|---------|---------------|
| `experiment_id` | STRING | Primary key. ~30 ids differ only by case — match exactly. Feature flags start `ff-`; machine-generated pre/post runs start `cde_app_`. | exp id | no | `cart_123reg_empty_new_ux` |
| `experiment_name` | STRING | Display name. | name, title | no | — |
| `experiment_type_enum` | STRING | Experiment type. Portfolio questions use `'A/B'` only. 'MAB' and 'mab' both exist; ~3.8k NULL. | type | yes | `A/B`, `PRE-POST`, `FEATURE-FLAG`, `A/A`, `ENDGAME`, `MAB` |
| `status_enum` | STRING | Lifecycle status. NULL for most pre-2026 experiments and sometimes stale (concluded tests still 'draft'). Do not use for portfolio status. | status | yes | `in progress`, `rolled out`, `rolled back`, `killed`, `released`, `draft`, `deleted` |
| `rollout_decision_enum` | STRING | **The declared result** (human decision, lowercase) — this is what "result" / "did it win" means; report `automated_conclusion_enum` only as a secondary line. ~80% filled on A/B; ~20% NULL = undecided. Can be set mid-flight. | declared result, decision, outcome | yes | `win`, `loss`, `inconclusive`, `issue with experiment`, `killed by switchboard automatically` |
| `automated_conclusion_enum` | STRING | Hivemind automatic conclusion (UPPERCASE, sparse). Can disagree with the latest stats. | auto conclusion | yes | `WIN`, `LOSS`, `INCONCLUSIVE` |
| `business_unit_name` | STRING | Owning BU, free text with drift — canonicalize (see Traps). | BU | no | `CMO`, `Customer & Site`, `AGI`, `US Independents` |
| `team_name` | STRING | Owning team, free text; 35%+ NULL, spelling variants, comma lists. Do not group by it. | team, squad | no | — |
| `experiment_owner_id_list` | STRING | Comma list of owner usernames. | owners | no | — |
| `scorecard_template_name` | STRING | Scorecard template. | scorecard | no | — |
| `factor_change_enum` | STRING | What kind of change was tested. Near-empty before 2024-Q4. | change type, factor | yes | `compellingCopy`, `compellingDesign`, `personalization`, `promotions`, `priceTransparency`, `other` |
| `experiment_classification_enum` | STRING | Strategic classification; spelling drift across years. | classification, pillar | yes | `Seamless Experience: Site & Renewals`, `Pricing & Bundling` |
| `baseline_cohort_id` | STRING | Control cohort id (free text: 'off', 'pre', 'control', 'A', …); NULL on ~3% of A/B (then use `analysis_result_cln.control_cohort`). Never infer control by name. | control, baseline | no | `off` |
| `deciding_cohort_id_text` | STRING | Treatment cohort chosen for the decision / annualization. 'ineligible' = not eligible for iGCR (exclude from iGCR sums, keep in the experiment population). | deciding cohort, winner cohort | no | `on` |
| `cohort_name_list` | STRING | Comma list of cohorts. | cohorts, arms | no | `off,on` |
| `traffic_type_enum` | STRING | Bucketing id type. | traffic type, keyed by | yes | `visitor_guid`, `shopper_id` |
| `alpha_value` | FLOAT | Significance level (0.1 ≈ 66%, 0.2 ≈ 28%, NULL ≈ 4%; rare 0.4 / 0.3 / 0.05). Never assume 0.05. Scorecard rows carry the same value in `analysis_result_cln.alpha`; SRM uses its own 0.01. | alpha | no | `0.1`, `0.2` |
| `analysis_method_enum` | STRING | Configured analysis method. | method | yes | `fixed`, `sequential` |
| `decision_metric_id_list` | STRING | Comma list of configured decision metric ids (bare, no hash; may repeat — dedupe). Empty on ~10% of A/B (do-no-harm designs decide on guardrails). The scorecard id can differ — match `analysis_result_cln` via `deciding_metrics_id_list` or `label`. | decision metrics, primary metric | no | `new_purchase_conversion`, `new_purchase_gcr_amt` |
| `deciding_metrics_id_list` | STRING | Metrics actually used to decide — a DIFFERENT list from decision_metric_id_list. It carries the scorecard / analysis metric id (e.g. `gcr_new_purchases`) where `decision_metric_id_list` carries the configured id (e.g. `new_purchase_gcr_amt`); match `analysis_result_cln` on this one. | deciding metrics | no | — |
| `guardrail_metric_id_list` | STRING | Comma list of guardrails. Empty for ~70% of A/B — means "none configured", not "no harm". When empty, say "no guardrails were configured" explicitly. | guardrails | no | — |
| `sig_lift_decision_metric_id_list` | STRING | Decision metrics with a significant lift (may repeat ids). | — | no | — |
| `sig_loss_decision_metric_id_list` | STRING | Decision metrics with a significant loss. | — | no | — |
| `sig_guardrail_metric_id_list` | STRING | Guardrails that moved significantly. | — | no | — |
| `annualization_type_enum` | STRING | iGCR surface bucket; not additive across types. Discount already inside `annualization_value`: ≈0.48 for site / in-product / billing surfaces (incl. renewal), ≈0.335 for customer marketing. | iGCR type | yes | `customer marketing new purchase iGCR`, `site new purchase iGCR`, `new purchase iGCR` (legacy), `site renewal iGCR`, `billing agent renewal iGCR (v2)`, `product renewal iGCR` |
| `annualized_metric_definition_id` | STRING | Metric that was annualized; NULL when there is no `annualization_value` (and on most legacy 'new purchase iGCR' rows). Can be a guardrail or informative metric, not only a decision metric — always state its role. | annualized metric | no | `new_purchase_gcr_amt`, `new_o365_addons_purchase_gcr_amt` |
| `annualized_metric_significance_flag` | BOOLEAN | Only ever TRUE or NULL (TRUE on 611 of the 615 rows that have an annualization value, 2026-11-13) — adds no information; use `annualization_value IS NOT NULL`. | — | no | — |
| `badge_name` | STRING | Rigor badge. | badge | yes | `Platinum`, `Gold`, `Silver`, `Bronze` |
| `unmet_badging_criteria_list` | STRING | Badge criteria not met. | — | no | — |
| `srm_flag` | BOOLEAN | SRM flag from the LATEST report — false positive when the latest report is a post-rollout rerun. | SRM | no | — |
| `hypothesis_change_text` | STRING | What changed. Full text — matches `get_experiment_by_id` hypothesis.change character-for-character (verified 2026-11-13; max length ~13k). Quote it in full; it can contain newlines. Placeholders like 'TBD'/'NA' exist. | hypothesis | no | — |
| `hypothesis_expectation_text` | STRING | Expected outcome. Full text (matches the Hivemind config). | — | no | — |
| `hypothesis_reason_text` | STRING | Why. Full text (matches the Hivemind config). | — | no | — |
| `trigger_eid_text` | STRING | Trigger EID regex (sparse). | trigger | no | — |
| `first_user_bucket_mst_ts` | TIMESTAMP | Real start: first user bucketed (true MST, equals MIN `user_allocation_history.bucketed_mst_ts`). NULL ≈ never bucketed (never launched; ~21% of A/B rows, ~4% of decided win/loss/inconclusive rows, 0% of 2026 completions). Prefer over the configured `start_mst_date`. | real start, launch date | no | — |
| `last_user_bucket_mst_ts` | TIMESTAMP | Real end: last user bucketed. Use for "completed in period". | real end, completion date | no | — |
| `start_mst_date` | DATE | Configured start (may differ from real start; a few are in the future). | configured start | no | — |
| `end_mst_date` | DATE | Configured end (18% filled on A/B, 2026-11-13). Never use for duration or completion. | configured end | no | — |
| `rollout_mst_ts` | TIMESTAMP | **UTC, not MST** — convert with `DATEADD(hour, -7, rollout_mst_ts)` before labelling or comparing to MST run times (exact match to `get_experiment_by_id` rolloutDate, verified 2026-11-13). Rollout timestamp — despite the name it is the Hivemind rolloutDate UTC epoch, usually midnight = a calendar date (compare MST report run time, insert_utc_ts − 7h, against it). ~30% of decided experiments have one; NULL for most wins before 2026-04; sometimes before start; many are midnight-only. | rollout date | no | — |

## Facts

| Column | Type | Description | Synonyms | Aggregation | Unit |
|--------|------|-------------|----------|-------------|------|
| `annualization_value` | FLOAT | Governed annualized iGCR (Alation doc #521): Δ GCR/user × total sample × seasonality × surface discount (≈0.48 site, ≈0.335 CM) — discount already applied. Frozen at decision time. | iGCR, annualized impact | SUM within one annualization_type_enum | USD |
| `actual_sample_cnt` | BIGINT | Sample count; usually all cohorts but sometimes one cohort — prefer per-cohort counts from user_allocation_history. | sample size | none | users |
| `target_sample_cnt` | BIGINT | Power-calculator sample size, PER COHORT. | target sample | none | users |
| `duration_day_value` | FLOAT | Bucketing span in days — the canonical "duration" / "how long it ran" (= last − first bucket to within 0.1 day on 100% of A/B rows that have both timestamps; NULL when the experiment never bucketed). | duration, run length | AVG / MEDIAN | days |
| `traffic_allocation_value` | INT | % of traffic allocated. | allocation | none | percent |

## Mandatory Filters

| Filter | Expression | Why | Source |
|--------|------------|-----|--------|
| A/B population (portfolio only) | `experiment_type_enum = 'A/B' AND experiment_id NOT ILIKE 'ff-%' AND experiment_id NOT ILIKE 'cde\\_app\\_%' AND COALESCE(status_enum,'') NOT IN ('deleted','draft')` | PRE-POST machine runs, feature flags and drafts inflate counts and distort rates | portfolio rule (not used by the two narrowed use cases) |

## Omitted Columns

| Column Group | Reason |
|--------------|--------|
| `ended_at_mst_ts`, `first_report_generated_mst_ts`, `cln_create_utc_ts`, `annualization_value_prorated`, `fixed_design_flag`, `informative_metrics_list` | broken: backfill-corrupted / 0% filled / always NULL / constant (fixed_design_flag is TRUE or NULL even on sequential tests — use analysis_method_enum) — never use |
| `annualization_trigger_metric_type_enum`, `sig_lift_informative_metric_id_list`, `sig_loss_informative_metric_id_list` | not needed for the committed use-cases; informative significance comes from analysis_result_cln |
| `cross_bucket_cnt`, `even_split_flag`, `released_flag`, `killed_flag` | do not reconcile or mean something other than they appear (released ≠ shipped; even_split reflects post-rollout config) |
| free-text extras (`observation_text`, `opportunities_text`, `customer_problem_text`, `target_customer_text`, `prior_experiments_list`, `insight_tool*`, `label_list`, `rtb_initiative_name`, `category_enum`, `trigger_property_*`, `badging_mst_ts`, `automated_conclusion_mst_ts`, `analysis_type`, `additional_deciding_metrics_id_list`, `deciding_metrics_category_list`, `sig_*_informative_*`) | not needed for the committed use-cases; present on the physical table |

## Related Tables

| Table | Join Keys | Cardinality | Join Type |
|-------|-----------|-------------|-----------|
| `hivemind_mart.analysis_result_cln` | `experiment_id` = `experiment_id` | one_to_many | LEFT |
| `hivemind_mart.user_allocation_history` | `experiment_id` = `experiment_id` | one_to_many | LEFT |
| `hivemind_mart.experiment_order` | `experiment_id` = `experiment_id` | one_to_many | LEFT |

## Never Join

None known.

## Traps

- Never use `ended_at_mst_ts` or `end_mst_date` for duration or "completed in period" — use `last_user_bucket_mst_ts`. (`ended_at_mst_ts` is more than 2 days after the last bucket on 66% of A/B rows that have both.)
- **`rollout_mst_ts` is UTC.** MST is a fixed UTC−7 all year (Arizona, no DST), so `DATEADD(hour, -7, rollout_mst_ts)` is always the MST rollout time. ~22% of A/B experiments have one; ~37% of A/B rows that have one are exactly midnight UTC (date-only = 17:00 MST the previous day; real time unknown).
- **iGCR reporting criteria**: `annualization_value` IS NOT NULL AND `rollout_decision_enum = 'win'` AND `deciding_cohort_id_text <> 'ineligible'`. 39 of 579 A/B rows with a value are not wins.
- **Portfolio population**: killed / rolled-back experiments ARE in the A/B population; only `deleted` / `draft` are excluded.
- BU names drift: map `Customer & site`/`Customer and Site`/`Cust & Site` → `Customer & Site`; `Airo Growth & Innovation`/`Airo Growth  Innovation` → `AGI`; `USI` → `US Independents`; NULL/''/`N/A`/`Other` → `(unassigned)`. `AGI - Investors` is a separate BU (do not merge into AGI). A 2025→2026 reorg moved many teams into Customer & Site / AGI, so BU trends across those years are not like-for-like.
- `status_enum` is not a reliable portfolio status; `released_flag` does not mean the treatment shipped.
- `srm_flag` reflects the latest report, which can be a post-rollout rerun — recheck SRM from analysis_result_cln.
- `annualization_value` is already discounted; Commercial OS "annualized GCR" is a different, undiscounted number. A few non-win rows (inconclusive / loss / 'issue with experiment', or deciding cohort 'ineligible') also carry a value — portfolio iGCR sums must filter `rollout_decision_enum = 'win'`.

## ⚠️ **OWNER INPUT NEEDED**

These definitions are **not clear** and must be filled in by a Hivemind data owner: `additional_deciding_metrics_id_list` (OQ-MS-01); `ended_at_mst_ts` (OQ-MS-02); `opportunities_text` (OQ-MS-03); `badging_mst_ts / automated_conclusion_mst_ts` (OQ-MS-04); `annualization_trigger_metric_type_enum` (OQ-MS-05); `sig_lift_informative_metric_id_list / sig_loss_informative_metric_id_list` (OQ-MS-06); `trigger_property_name_text / trigger_property_value_text` (OQ-MS-07); `annualization_value_prorated, first_report_generated_mst_ts, informative_metrics_list, cln_create_utc_ts` (OQ-MS-08); `experiment_type_enum = 'PRE-POST-STANDALONE'` (OQ-MS-09); `status_enum = 'released' vs 'rolled out'` (OQ-MS-10); `analysis_type values 'revenue_impact', 'User Interaction', 'conversion_rate_analysis'` (OQ-MS-11); `experiment_classification_enum legacy values: 'Protect my Business', 'Big Swing', 'One System', 'Seamless Experience', 'Airo'` (OQ-MS-12); `business_unit_name values 'FORGE', 'GDI', 'Midas', 'FOS', 'Domains', 'Domain Investors', and other small BUs` (OQ-MS-13); `rollout_decision_enum = 'issue with experiment'` (OQ-MS-14); `traffic_type_enum non-web id types (care_ucid_jomax_id, care_conversation_id, resource_id, billing_cycle_id, auction_id, payment_id, subscription_lifetime_id)` (OQ-MS-15); `physical column count` (OQ-MS-16). Full questions are in `data-dictionary/hivemind_mart.experiment_metadata_snap.dictionary.md`. If a question depends on one of them, answer "definition not confirmed by the Hivemind owners" — do not guess.

## Provenance & Gaps

| Field | Source | Status |
|-------|--------|--------|
| columns | redshift_svv (describe_table 2026-10-22) | populated |
| column descriptions | Alation (0 of 73; auto text marked Requires Review and partly wrong) | not_in_alation |
| enums / sample values | execute_query 2026-10-22 | populated |
| owner | — | not_in_alation |
