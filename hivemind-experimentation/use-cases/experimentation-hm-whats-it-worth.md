---
type: use-case
entry_id: experimentation-hm-whats-it-worth
products: [experimentation]
pnl_lines: []
metric_family: gcr
metric_variant: igcr
intents: [hm_igcr_lookup, hm_daily_dollar_impact, hm_product_line_gcr]
status: DETERMINISTIC
historical_question_count: 0
last_updated: 2026-11-13
sources:
  governed_articles: [521, 439]
methodology:
  authority: pattern_doc
  source_chain: [hivemind_mart.experiment_metadata_snap, hivemind_mart.analysis_result_cln, hivemind_mart.experiment_order]
  refresh: "daily (~D-1)"
derived_from: []
supersedes: null
verified_on: 2026-10-22
notes:
  - "Merged 2026-11-13 from Path A of experimentation-hm-whats-it-worth, -daily-dollar-impact and -product-line-gcr (SQL unchanged)."
  - "annualization_value is the governed iGCR (Alation doc #521 / #439): discount factor and seasonality mix already applied."
---

# What is experiment X worth? — governed iGCR, $ per user / per day, product-line GCR breakdown

> Single-experiment money questions: the governed annualized iGCR and the metric it was annualized on (overall vs product-level), the dollar value of a lift per user and per day, and which product lines / new vs renewal explain the GCR gap.

## Routing — pick the path first

| Question shape | Path |
|---|---|
| "What iGCR did X deliver?", "what was annualized, is it overall or product-level?", "why is there no iGCR?" | **Path A — Governed iGCR lookup** |
| "How many dollars per user / per day is the lift worth?", "daily dollar value", "what is the GCR lift worth per day?" | **Path B — Daily $ value** (always show the Path A governed iGCR per day beside it) |
| "Where did the GCR difference come from?", "which product lines / new vs renewal explain the gap?" | **Path C — Product-line GCR decomposition** |
| Did it win / significance / readout | not here, use experimentation-hm-did-it-work |
| $ value of a conversion-rate lift, iGCR "landed" since rollout, which product line is significant, recompute iGCR, iGCR vs target | **decline** (see Out of scope) |

**Timezone rule (applies to every path):** `rollout_mst_ts` holds the Hivemind rolloutDate as a **UTC** instant despite its name (verified against `get_experiment_by_id` rolloutDate on 10 experiments, 2026-11-13). Always use `DATEADD(hour, -7, rollout_mst_ts)` as the MST rollout time, label times you show as MST, and compare report run times (`insert_utc_ts` − 7h) to the converted value. A value of exactly 00:00:00 UTC is a date-only rollout = 17:00 MST the previous day.

Shared rules: annualization_value is the governed iGCR with the discount already applied (never re-discount); Commercial OS "annualized gcr" is a different, undiscounted, overall number (never mix them); experiment_order uses exactly ONE snapshot partition, the latest for the experiment.

## Canonical Interpretations

### Path A — Governed iGCR lookup  _(Governed iGCR from experiment metadata)_
- **Source table**: `hivemind_mart.experiment_metadata_snap` (one row per experiment, refreshed daily)
- **Definition**: `annualization_value` = Hivemind's governed annualized incremental GCR for the deciding treatment vs control, computed per Alation doc #521 (new purchase) or #439 (renewal): Δ GCR/user × total bucketed users (all cohorts) × (trailing-12-month GCR ÷ prior-year GCR for the same date window) × discount factor.
- **Metric variant**: igcr (see `metric-families/gcr.md`)
- **Time anchor**: `rollout_mst_ts` (rollout decision time). Real run window = `first_user_bucket_mst_ts` → `last_user_bucket_mst_ts`.
- **Grain**: one row per experiment
- **Key fields**:
  - `annualization_value` — governed iGCR in USD (discount already applied; can be negative)
  - `annualization_type_enum` — surface bucket: site new purchase / customer marketing new purchase / site renewal / product renewal / billing agent renewal (v2) / legacy 'new purchase iGCR'. Different surfaces use different discount factors.
  - `annualized_metric_definition_id` — the metric that was annualized (may differ from the decision metric; it can be a guardrail or informative metric, or a "monetization pair" of a non-$ decision metric)
  - `annualized_metric_significance_flag` — TRUE when annualized; NULL otherwise (never FALSE in practice)
  - `deciding_cohort_id_text` / `baseline_cohort_id` — which treatment was compared to which control
  - `rollout_decision_enum`, `rollout_mst_ts` — human decision and when it was made
- **Overall vs product-level rule** (applied to `annualized_metric_definition_id`):
  - overall new purchase: `new_purchase_gcr_amt`, `gcr_new_purchases`, `total_new_gcr` (web + C3)
  - overall, Web point of purchase only: `new_web_gcr`
  - overall products, new-acquisition orders only: `New_customer_new_purchase_gcr`
  - renewal surfaces: report the metric id and cite doc #439 — do not label overall/product
  - NULL metric id with a value: legacy row, scope unknown — say so
  - anything else: product-level / scoped (e.g. o365, WAM, domains, SSL, Airo metrics)
- **SQL**:
  ```sql
  SELECT
      experiment_id,
      experiment_type_enum,
      business_unit_name,
      rollout_decision_enum,
      automated_conclusion_enum,
      DATEADD(hour, -7, rollout_mst_ts) AS rollout_mst,
      last_user_bucket_mst_ts,
      deciding_cohort_id_text AS deciding_cohort,
      baseline_cohort_id AS control_cohort,
      annualization_type_enum AS annualization_surface,
      annualized_metric_definition_id AS annualized_metric,
      ROUND(annualization_value, 0) AS annualized_igcr_usd,
      annualized_metric_significance_flag AS annualized_metric_significant,
      CASE
          WHEN annualization_value IS NULL THEN 'not annualized'
          WHEN annualized_metric_definition_id IS NULL THEN 'unknown - metric id not recorded (legacy row)'
          WHEN annualization_type_enum ILIKE '%renewal%' THEN 'renewal iGCR (doc #439) - not classified overall/product'
          WHEN annualized_metric_definition_id IN ('new_purchase_gcr_amt', 'gcr_new_purchases', 'total_new_gcr') THEN 'overall new purchase'
          WHEN annualized_metric_definition_id = 'new_web_gcr' THEN 'overall new purchase - Web point of purchase only'
          WHEN LOWER(annualized_metric_definition_id) = 'new_customer_new_purchase_gcr' THEN 'overall new purchase - new-acquisition orders only'
          ELSE 'product-level / scoped new purchase'
      END AS metric_scope,
      CASE
          WHEN annualization_value IS NOT NULL THEN NULL
          WHEN COALESCE(rollout_decision_enum, '') <> 'win' THEN 'rollout_decision_enum is not win'
          WHEN COALESCE(sig_lift_decision_metric_id_list, '') = '' THEN 'win, but no decision metric shows a significant lift'
          WHEN sig_lift_decision_metric_id_list NOT ILIKE '%gcr%' AND sig_lift_decision_metric_id_list NOT ILIKE '%receipt%'
               AND sig_lift_decision_metric_id_list NOT ILIKE '%yield%' THEN 'win, but significant decision metrics are not GCR/receipt metrics'
          ELSE 'win with a significant GCR-type decision metric, but no annualization recorded - check Hivemind UI'
      END AS null_reason_from_fields,
      CASE
          WHEN annualized_metric_definition_id IS NULL THEN NULL
          WHEN ',' || REPLACE(COALESCE(decision_metric_id_list, ''), ' ', '') || ',' LIKE '%,' || annualized_metric_definition_id || ',%' THEN 'decision metric'
          WHEN ',' || REPLACE(COALESCE(guardrail_metric_id_list, ''), ' ', '') || ',' LIKE '%,' || annualized_metric_definition_id || ',%' THEN 'guardrail metric'
          ELSE 'informative / monetization-pair metric'
      END AS annualized_metric_role,
      decision_metric_id_list,
      guardrail_metric_id_list,
      sig_lift_decision_metric_id_list,
      deciding_metrics_id_list
  FROM hivemind_mart.experiment_metadata_snap
  WHERE experiment_id = '<experiment_id>'
  LIMIT 5;
  ```
- **How to answer**:
  1. State the value, the surface (`annualization_type_enum`), the metric, its role (`annualized_metric_role`), the scope, and the deciding cohort vs control. If the role is 'guardrail metric' (especially with an empty `decision_metric_id_list`), say so explicitly: "this is a do-no-harm style test annualized on a guardrail, not on a decision metric".
  2. Say explicitly: "This is the governed iGCR; the discount factor (≈0.48 site / in-product / billing, ≈0.335 customer marketing) is already applied."
  3. If `experiment_type_enum` is not 'A/B', report the type and warn the value may not follow the A/B methodology.
  4. If NULL, say "not annualized" and give only the reason the fields support (`null_reason_from_fields`). Do not invent other reasons (sample size, duration, Finance review) — the table does not record them.
  5. If `rollout_decision_enum` is not 'win' but a value exists, report it and flag that doc #521 requires a GCR win for iGCR to be reportable.

### Path B — Daily $ value of a lift  _(Scorecard Δ per user × users per day)_
- **Source table**: `hivemind_mart.analysis_result_cln` (scorecard rows) + `hivemind_mart.experiment_metadata_snap` (duration, governed iGCR)
- **Definition**:
  - $ per user = treatment `metric_value` − control `metric_value` on the chosen $/user metric (raw difference; `metric_delta` is rounded to 3 decimals, so do not use it for money)
  - users per day = cohort user counts (`cohort_percentage.metric_value`) ÷ `duration_day_value` (days between first and last bucketing)
  - daily $ at full rollout (undiscounted) = $ per user × users per day
  - governed iGCR per day = `annualization_value` / 365 (discounted, seasonality-adjusted — doc #521)
- **Metric variant**: igcr (see `metric-families/gcr.md`)
- **Time anchor**: `report_date` = latest report that has rows for the chosen metric (R3); `duration_day_value` for the per-day divisor
- **Grain**: one row per treatment cohort vs control
- **Metric choice (in order)**: 1) the metric the user names (`'<metric_id>'`, bare id; pass `''` if none); 2) the governed annualized metric (`annualized_metric_definition_id`); 3) decision metrics whose `label` contains "GCR". Only $-per-user metrics (label "GCR", receipts) give a dollar answer.
- **Key filters**: all 8 segment columns = 'ALL' (R4), `SELECT DISTINCT` (duplicate grains), metric id compared on `REGEXP_REPLACE(metric_id, '_[0-9a-f]{32}$', '')` (R5), treatment rows = `cohort_id <> control_cohort` (R6).
- **SQL**:
  ```sql
  WITH exp AS (
      SELECT experiment_id, experiment_type_enum, business_unit_name, deciding_cohort_id_text,
             duration_day_value, annualization_value, annualization_type_enum, annualized_metric_definition_id
      FROM hivemind_mart.experiment_metadata_snap
      WHERE experiment_id = '<experiment_id>'
  ),
  overall_rows AS (
      SELECT DISTINCT
          report_date, analysis_method_used, analysis_type,
          REGEXP_REPLACE(metric_id, '_[0-9a-f]{32}$', '') AS metric_id_bare,
          metric_category, label, cohort_id, control_cohort,
          metric_value, metric_delta, metric_pvalue, metric_significance_flag, alpha, metric_pbest, credible_interval
      FROM hivemind_mart.analysis_result_cln
      WHERE experiment_id = '<experiment_id>'
        AND new_existing_customer_name = 'ALL' AND market_site = 'ALL' AND device_name = 'ALL'
        AND ip_country_name = 'ALL' AND customer_tenure_day_segment = 'ALL' AND domain_ownership_count = 'ALL'
        AND customer_type = 'ALL' AND market_sub_region = 'ALL'
  ),
  target_metric AS (
      SELECT DISTINCT o.metric_id_bare
      FROM overall_rows o
      CROSS JOIN exp e
      WHERE o.metric_id_bare = COALESCE(NULLIF('<metric_id>', ''), e.annualized_metric_definition_id)
         OR (COALESCE(NULLIF('<metric_id>', ''), e.annualized_metric_definition_id) IS NULL
             AND o.metric_category = 'decision' AND o.label ILIKE '%GCR%')
  ),
  readout_date AS (
      SELECT MAX(o.report_date) AS report_date
      FROM overall_rows o
      JOIN target_metric m ON o.metric_id_bare = m.metric_id_bare
  ),
  cohort_sample AS (
      SELECT o.cohort_id, o.metric_value AS cohort_users
      FROM overall_rows o
      JOIN readout_date r ON o.report_date = r.report_date
      WHERE o.metric_id_bare = 'cohort_percentage'
  ),
  metric_rows AS (
      SELECT o.*
      FROM overall_rows o
      JOIN readout_date r ON o.report_date = r.report_date
      JOIN target_metric m ON o.metric_id_bare = m.metric_id_bare
  ),
  paired AS (
      SELECT t.report_date, t.analysis_method_used, t.analysis_type, t.metric_id_bare, t.metric_category,
             t.cohort_id AS treatment_cohort, t.control_cohort,
             t.metric_value AS treatment_value, c.metric_value AS control_value,
             t.metric_value - c.metric_value AS delta_per_user_usd,
             t.metric_delta AS metric_delta_rounded_3dp,
             t.metric_pvalue, t.metric_significance_flag, t.alpha, t.metric_pbest, t.credible_interval
      FROM metric_rows t
      JOIN metric_rows c ON c.metric_id_bare = t.metric_id_bare AND c.cohort_id = t.control_cohort
      WHERE t.cohort_id <> t.control_cohort
  )
  SELECT
      p.report_date, p.analysis_method_used, p.analysis_type, p.metric_id_bare, p.metric_category,
      p.treatment_cohort, p.control_cohort, e.deciding_cohort_id_text AS deciding_cohort,
      p.treatment_value, p.control_value, p.delta_per_user_usd, p.metric_delta_rounded_3dp,
      p.metric_pvalue, p.metric_significance_flag, p.alpha, p.metric_pbest, p.credible_interval,
      e.duration_day_value,
      ts.cohort_users + cs.cohort_users AS pair_users,
      (SELECT SUM(cohort_users) FROM cohort_sample) AS all_cohort_users,
      p.delta_per_user_usd * (ts.cohort_users + cs.cohort_users) / NULLIF(e.duration_day_value, 0) AS daily_usd_pair_basis,
      p.delta_per_user_usd * (SELECT SUM(cohort_users) FROM cohort_sample) / NULLIF(e.duration_day_value, 0) AS daily_usd_all_cohort_basis,
      p.delta_per_user_usd / NULLIF(e.duration_day_value, 0) AS delta_per_user_per_day_usd,
      e.annualization_value / 365.0 AS governed_igcr_per_day,
      e.annualization_type_enum, e.experiment_type_enum, e.business_unit_name
  FROM paired p
  JOIN cohort_sample ts ON ts.cohort_id = p.treatment_cohort
  JOIN cohort_sample cs ON cs.cohort_id = p.control_cohort
  CROSS JOIN exp e
  ORDER BY p.treatment_cohort
  LIMIT 20;
  ```
- **How to answer**:
  1. Lead with the deciding cohort's row. Give $ per user (`delta_per_user_usd`), users per day, and daily $ (`daily_usd_pair_basis` for a 2-arm test; for multi-arm say which basis you use — pair matches Commercial OS, all-cohort matches governed iGCR).
  2. Always show `governed_igcr_per_day` beside it and explain the gap: the governed number is discounted (≈0.48 / ≈0.335) and seasonality-adjusted, the daily $ is not.
  3. State significance in the right framework: frequentist → `metric_significance_flag` at the experiment's `alpha` (not 0.05); sequential p-value = 1 means boundary not crossed; bayesian → `metric_pbest` and `credible_interval` (the significance flag and p-value are NULL/FALSE for bayesian rows — do not report "not significant" from them).
  4. If the metric is not significant, give the number as a point estimate and say so.
  5. **Always state** that `metric_delta` is stored rounded to 3 dp (quote it, e.g. 0.469) and that the dollars use the unrounded treatment − control `metric_value`.
  6. If you mention Commercial OS `metric_impact.gcr`, label it as Commercial OS overall winsorized GCR (all products) and say it is refreshed live and moves between refreshes; never present it as the same number.
- **What it measures**: the incremental $ per day if the treatment had been shown to all of the experiment's eligible traffic, at the traffic rate observed during bucketing. It is not realised revenue.

### Path C — Product-line GCR decomposition  _(Raw GCR/user decomposition from experiment_order (one partition))_
- **Source table**: `hivemind_mart.experiment_order` (one row per order line per allocated user; non-buyers appear with `order_id IS NULL`)
- **Definition**: for each cohort, GCR per user in a cell = `SUM(gcr_usd_amt)` in that cell ÷ ALL users in the cohort (buyers + non-buyers). Δ cell = treatment − control. Because every cell shares the cohort denominator, cell Δs add up exactly to the total GCR/user Δ.
- **Metric variant**: total_gcr (all orders, new + renewal; see `metric-families/gcr.md`)
- **Time anchor**: `record_create_mst_date` — cumulative snapshot partition (STRING 'YYYY-MM-DD'); the latest partition is usually the last analysis report_date / day after bucketing stopped.
- **Grain**: treatment cohort × `product_pnl_line_name` × `product_pnl_new_renewal_name` × `product_period_name`
- **Key filters**: `experiment_id = '<experiment_id>'`, `record_create_mst_date = '<latest_partition>'` (one partition, R9). Refund rows (`refund_flag = TRUE`) are kept: they carry positive gross GCR. Cell shares are reliable; absolute totals can sit below Commercial OS `gcr_raw` (partitions freeze ~1 day after bucketing) — do not present them as the COS figure.
- **Users**: `COUNT(DISTINCT CASE WHEN experiment_user_identifier = 'shopper_id' THEN shopper_id ELSE visitor_guid END)` — the id type the experiment buckets on. (`COALESCE(shopper_id, visitor_guid)` undercounts visitor-keyed experiments where logged-in visitors also carry a shopper_id.)
- **Control cohort**: `analysis_result_cln.control_cohort` on the latest report, else `experiment_metadata_snap.baseline_cohort_id`.

- **Step 0 — get the bucketing end date (fast)**:
  ```sql
  SELECT experiment_id, experiment_type_enum, traffic_type_enum, baseline_cohort_id,
         TO_CHAR(first_user_bucket_mst_ts, 'YYYY-MM-DD') AS first_bucket_date,
         TO_CHAR(last_user_bucket_mst_ts, 'YYYY-MM-DD') AS last_bucket_date
  FROM hivemind_mart.experiment_metadata_snap
  WHERE experiment_id = '<experiment_id>'
  LIMIT 1;
  ```
- **Step 1 — find the latest partition (literal lower bound so Spectrum prunes; ~5–40 s)**:
  ```sql
  SELECT MAX(record_create_mst_date) AS latest_partition, COUNT(DISTINCT record_create_mst_date) AS partitions_scanned
  FROM hivemind_mart.experiment_order
  WHERE experiment_id = '<experiment_id>'
    AND record_create_mst_date >= '<last_bucket_date>';
  ```
  If it returns NULL, rerun with `>= '<first_bucket_date>'`. Never run MAX without a literal bound (full-history scan, times out).
- **Step 2 — decomposition (~15–40 s)**:
  ```sql
  WITH orders AS (
      SELECT eo.cohort_id,
             CASE WHEN eo.experiment_user_identifier = 'shopper_id' THEN eo.shopper_id ELSE eo.visitor_guid END AS user_key,
             COALESCE(eo.gcr_usd_amt, 0) AS gcr_usd_amt,
             eo.order_id,
             COALESCE(eo.product_pnl_line_name, '(none)') AS product_pnl_line_name,
             COALESCE(eo.product_pnl_new_renewal_name, '(none)') AS new_renewal,
             COALESCE(eo.product_period_name, '(none)') AS product_period_name
      FROM hivemind_mart.experiment_order eo
      WHERE eo.experiment_id = '<experiment_id>'
        AND eo.record_create_mst_date = '<latest_partition>'
  ),
  cohort_users AS (
      SELECT cohort_id, COUNT(DISTINCT user_key) AS users, SUM(gcr_usd_amt) AS cohort_gcr_usd
      FROM orders
      GROUP BY cohort_id
  ),
  control AS (
      SELECT COALESCE(
               (SELECT MAX(control_cohort) FROM hivemind_mart.analysis_result_cln
                 WHERE experiment_id = '<experiment_id>' AND control_cohort IS NOT NULL
                   AND report_date = (SELECT MAX(report_date) FROM hivemind_mart.analysis_result_cln
                                       WHERE experiment_id = '<experiment_id>')),
               (SELECT MAX(baseline_cohort_id) FROM hivemind_mart.experiment_metadata_snap
                 WHERE experiment_id = '<experiment_id>')) AS control_cohort
  ),
  cell AS (
      SELECT o.cohort_id, o.product_pnl_line_name, o.new_renewal, o.product_period_name,
             SUM(o.gcr_usd_amt) AS gcr_usd,
             COUNT(DISTINCT o.order_id) AS orders
      FROM orders o
      WHERE o.order_id IS NOT NULL
      GROUP BY 1, 2, 3, 4
  ),
  per_user AS (
      SELECT c.cohort_id, c.product_pnl_line_name, c.new_renewal, c.product_period_name,
             c.orders, c.gcr_usd / cu.users AS gcr_per_user
      FROM cell c JOIN cohort_users cu ON cu.cohort_id = c.cohort_id
  ),
  keys AS (
      SELECT DISTINCT product_pnl_line_name, new_renewal, product_period_name FROM per_user
  ),
  treat AS (
      SELECT cu.cohort_id, cu.users FROM cohort_users cu CROSS JOIN control ct WHERE cu.cohort_id <> ct.control_cohort
  ),
  compare AS (
      SELECT t.cohort_id AS treatment_cohort, ct.control_cohort,
             t.users AS treatment_users, cc.users AS control_users,
             t_tot.cohort_gcr_usd / t.users - cc.cohort_gcr_usd / cc.users AS total_delta_gcr_per_user,
             k.product_pnl_line_name, k.new_renewal, k.product_period_name,
             COALESCE(pt.gcr_per_user, 0) AS treatment_gcr_per_user,
             COALESCE(pc.gcr_per_user, 0) AS control_gcr_per_user,
             COALESCE(pt.gcr_per_user, 0) - COALESCE(pc.gcr_per_user, 0) AS delta_gcr_per_user,
             COALESCE(pt.orders, 0) AS treatment_orders,
             COALESCE(pc.orders, 0) AS control_orders
      FROM keys k
      CROSS JOIN treat t
      CROSS JOIN control ct
      JOIN cohort_users cc ON cc.cohort_id = ct.control_cohort
      JOIN cohort_users t_tot ON t_tot.cohort_id = t.cohort_id
      LEFT JOIN per_user pt ON pt.cohort_id = t.cohort_id AND pt.product_pnl_line_name = k.product_pnl_line_name
                           AND pt.new_renewal = k.new_renewal AND pt.product_period_name = k.product_period_name
      LEFT JOIN per_user pc ON pc.cohort_id = ct.control_cohort AND pc.product_pnl_line_name = k.product_pnl_line_name
                           AND pc.new_renewal = k.new_renewal AND pc.product_period_name = k.product_period_name
  ),
  ranked AS (
      SELECT compare.*, ROW_NUMBER() OVER (PARTITION BY treatment_cohort ORDER BY ABS(delta_gcr_per_user) DESC) AS rank_abs_delta
      FROM compare
  )
  SELECT treatment_cohort, control_cohort, treatment_users, control_users,
         ROUND(total_delta_gcr_per_user, 4) AS total_delta_gcr_per_user,
         rank_abs_delta, product_pnl_line_name, new_renewal, product_period_name,
         ROUND(treatment_gcr_per_user, 4) AS treatment_gcr_per_user,
         ROUND(control_gcr_per_user, 4) AS control_gcr_per_user,
         ROUND(delta_gcr_per_user, 4) AS delta_gcr_per_user,
         ROUND(delta_gcr_per_user / NULLIF(total_delta_gcr_per_user, 0), 3) AS share_of_total_delta,
         treatment_orders, control_orders
  FROM ranked
  WHERE rank_abs_delta <= 8          -- top cells per treatment by |Δ|
  ORDER BY treatment_cohort, rank_abs_delta
  LIMIT 40;
  ```
- **Coarser views**: because cells share the cohort denominator, product-line totals = sum of that line's cells; new-vs-renewal totals = sum over lines. Roll up the Step 2 output instead of re-querying (or drop columns from the `cell` GROUP BY and `keys`).
- **How to answer**: give the total raw Δ GCR/user, then the cells that explain most of it (positive and negative), with order counts for both cohorts; then summarise by new vs renewal. Always say "raw (unwinsorized) GCR, no significance test — these cells describe where the gap sits, not which effects are real".
  - **Keep `product_period_name` (term) in the cell grain** even when the question only says product line × new/renewal. Month and year renewals of the same line often move in opposite directions; netting them hides the real driver. Give the line and new/renewal roll-ups only as a summary under the cells.
  - **Quote Δ GCR/user to 4 dp** (the query already rounds to 4); do not round to 2 dp.
  - **State the denominator**: "per user = all bucketed users in the cohort, buyers and non-buyers (e.g. 95,024 users vs ~26K buyers); dividing by buyers would give average order value, not GCR/user."

## Out of scope (decline)

Decline these even when the data looks like it could answer them: say what cannot be answered, why, and where to go instead. Never invent a number.

### For Path A (Governed iGCR lookup)
- "How much of this iGCR has actually landed / been realised since rollout?" — not in this table. Point to Commercial OS post-rollout durability (`get_commercial_os_post_rollout_metrics`, while-active / 30 / 90 / 180 / 365-day lift) and Finance for booked revenue.
- "Re-compute the iGCR with a different discount / different metric" — the governed value is authoritative; recomputation needs the prior-year GCR mix inputs (doc #521 DATAGOV query) and is an analyst task.
- "What iGCR would a partial (e.g. 50%) rollout deliver?" — doc #521 assumes 100% of the target segment; scaling requires the target-audience share, which is not stored.
- Daily $ value of the lift → Path B of experimentation-hm-whats-it-worth. Portfolio totals → a portfolio iGCR entry (not in this corpus).

### For Path B (Daily $ value of a lift)
- "What is the daily dollar value of the conversion-rate lift?" — a rate metric has no $/user; Commercial OS has a separate `cvr` impact with its own (undocumented) $ conversion. Decline the $ conversion from this table; point to Commercial OS `metric_impact.cvr` labelled as Commercial OS's figure.
- "How much has it earned since rollout?" — realised post-rollout revenue is not in the scorecard; use `get_commercial_os_post_rollout_metrics` (durability windows) or Finance.
- "Is the daily $ statistically significant?" — significance belongs to the per-user metric; no interval is computed for the daily $ here.

### For Path C (Product-line GCR decomposition)
- "Which product line's lift is statistically significant?" — no significance here. Point to the Hivemind scorecard product-specific metrics (e.g. a product GCR metric in `analysis_result_cln`) or the segmentation results tools; do not infer significance from cell size.
- "Annualize the product-line gap" — not a governed number; use Path A of experimentation-hm-whats-it-worth (governed iGCR) or Commercial OS annualized (labelled as Commercial OS, overall, undiscounted).
- "Break down by customer segment / market / device" — different dimensions; use the segmentation entries, not this one.

## Gotchas

### Path A — Governed iGCR lookup
- **Commercial OS "annualized gcr" is a different number.** `get_commercial_os_metrics` returns annualized *overall* GCR (all products, new + renewal), winsorized, undiscounted, and uses treatment+control-pair sample × 365/duration. It is usually much larger than the governed iGCR. Never present it as iGCR or mix the two.
- **Do not re-discount.** `annualization_value` already includes the discount factor and the seasonality mix (doc #521). `annualization_value_prorated` is always NULL — do not use it.
- **The annualized metric is not necessarily the decision metric.** It can be a guardrail (e.g. a "do no harm" experiment annualized on a product GCR guardrail) or an informative metric; `decision_metric_id_list` can even be empty. Always report `annualized_metric_definition_id`, not the decision metric.
- **A value can exist on a non-win.** A minority of annualized rows have `rollout_decision_enum` = inconclusive / loss / 'issue with experiment', and some have `deciding_cohort_id_text` = 'ineligible'. Report the value but flag that it does not meet the doc #521 reporting criteria.
- **Negative values exist** (a significant GCR loss on the annualized metric). Report the sign as-is.
- **`rollout_decision_enum` is the human result; `automated_conclusion_enum` is sparse and may disagree.** Values like 'killed by switchboard automatically' are not wins.
- **`rollout_mst_ts` carries the Hivemind rolloutDate epoch in UTC** despite the `_mst_` name (exact match to the config API). It can be a few hours off from an MST calendar date.
- **Legacy surface 'new purchase iGCR'** rows mostly have NULL `annualized_metric_definition_id` — scope is unknown; say so.
- Freshness: snap refreshes daily (~D-1). An experiment rolled out today may not have its value yet.

### Path B — Daily $ value of a lift
- **`metric_delta` is rounded to 3 decimals** — always recompute treatment − control from `metric_value` for money; on large samples the rounding moves the daily $ materially.
- **Hivemind `metric_value` is winsorized / CUPED-adjusted**, Commercial OS is winsorized differently and uses *overall* GCR (all products, new + renewal), and `experiment_order` is raw. Expect three different per-user deltas for the same experiment; Commercial OS `winsorization_diagnostics.gcr_raw` shows how much winsorization removed.
- **Sample basis**: Commercial OS `metric_impact` uses treatment + control sample; doc #521 iGCR uses all bucketed cohorts. Identical for 2-arm tests, different for multi-arm.
- **Per-user Δ accumulates over the observation window** (bucketing start → report_date, which continues after bucketing stops). `delta_per_user_per_day_usd` is only a rough rate; do not multiply it by users again.
- Use the latest report_date that has the metric (R3) and do not mix `analysis_method_used` values across dates.
- Undiscounted daily $ × 365 is NOT iGCR. Use Path A of experimentation-hm-whats-it-worth for the governed figure.
- `duration_day_value` ≈ last − first bucket; never use `ended_at_mst_ts` or `end_mst_date` (R2).

### Path C — Product-line GCR decomposition
- **One partition only.** Summing partitions multiplies GCR by the number of days. Never use `BETWEEN` on `record_create_mst_date`.
- **Denominator must include non-buyers** (`order_id IS NULL` rows). Dividing by buyers gives average order value, not GCR/user.
- **Raw ≠ scorecard.** Hivemind scorecard values are winsorized/CUPED and scoped to the metric's order filter (e.g. new purchase only); Commercial OS `gcr_per_user` is winsorized overall GCR. Raw cell Δs can be smaller, larger, or even opposite in sign to the winsorized line-level Δ. Commercial OS `winsorization_diagnostics.gcr_raw / sample` is the comparable raw figure.
- **Small cells swing on one order.** A cell with a handful of orders (e.g. a hosting renewal) can show a large Δ from one high-value order; always show order counts and call such cells noise-prone.
- **Renewal GCR inside an onsite test** is mostly customers who happened to renew while bucketed; a renewal Δ is usually not caused by an onsite treatment — flag before attributing.
- `experiment_user_identifier` holds the id TYPE ('shopper_id' | 'visitor_guid'), not an id.
- Multi-arm: one block per treatment vs the same control.
- Cost: always filter `experiment_id` and a literal partition date; Step 1 without a literal bound times out.
