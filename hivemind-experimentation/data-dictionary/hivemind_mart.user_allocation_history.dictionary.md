# `hivemind_mart.user_allocation_history` - Data Dictionary (full depth)

> ⚠️ **OWNER INPUT NEEDED** (OQ-UA-07): table-level definition not clear (which column holds the bucketed unit for non-web traffic types). Hivemind owner to fill in. See the owner table at the end of this file.

**Purpose.** One row per user that Hivemind actually analyzed in an experiment: which cohort (arm) the user was assigned to, when the user was first bucketed, which id type keys the user, plus a frozen copy of the user's segment attributes (device, market site, new/existing, IP country, tenure, domain-ownership bucket, customer type, WAM free-trial flag, visit count). It is the source for exact per-cohort analyzed-user counts (they equal Hivemind's `healthMetric.summary.sampleCount`) and for the real experiment start/end (MIN/MAX `bucketed_mst_ts`). It is NOT the raw assignment log: it is already deduplicated (so it cannot measure cross-bucketing), and for triggered experiments it is already the triggered/analyzed population (smaller than raw assignments).

| Property | Value |
|---|---|
| Grain | one row per analyzed user per `experiment_id` (deduplicated; `bucketed_mst_date` = first bucket date) |
| Primary key | (`experiment_id`, `allocation_id`); `allocation_id` was also unique across the whole profiled scope (1,397,235 distinct of 1,397,235 rows) |
| Sort / partition key | `bucketed_mst_date` |
| Mandatory filters | `experiment_id = '<exact id>'` (case-sensitive) AND `bucketed_mst_date BETWEEN DATE(first_user_bucket_mst_ts) - 1 AND DATE(last_user_bucket_mst_ts) + 1`, bounds read from `hivemind_mart.experiment_metadata_snap` for the same experiment. Query ONE experiment per statement: unbounded scans are ~20x slower and hit the 2-minute timeout (verified doc: 50-100 s each); a 5-experiment statement in this session was terminated by the server. |
| User count rule | `COUNT(*)` per `cohort_id` = Hivemind cohort sample. No DISTINCT needed (`COUNT(*) = COUNT(DISTINCT allocation_id)`; on agi_mon_email_banner_multi_surface also `= COUNT(DISTINCT shopper_id)` = 686,464, SQL). |
| Control cohort | NOT stored here. Always read `experiment_metadata_snap.baseline_cohort_id`; never infer from names like `default`, `off`, `ctrl`. |
| Physical columns | 20 |

### Row counts by scope (profile scope: 5 experiments, 1,397,235 rows)

| experiment_id | Rows | Id type | Scorecard | Cohorts (rows) | Control (metadata `baseline_cohort_id`) | MIN / MAX `bucketed_mst_ts` | Status (MCP config) |
|---|---|---|---|---|---|---|---|
| `agi_mon_email_banner_multi_surface` | 686,464 | shopper_id | mon_banner_baseline | default 171,122; variant1 171,652; variant2 172,321; variant3 171,369 | default | 2026-09-22 00:00:00.021 / 2026-10-10 23:59:42.247 | killed; triggered (eid `mon.banner.experiment.agi_mon_email_banner_multi_surface.impression`) |
| `MKT_EM_177_Failed-Billing-Manual-D28_CM-24927` | 277,217 | shopper_id | cm-renewals-audience-recording | control 55,502; testa 55,336; testb 55,481; testc 55,622; testd 55,276 | control | 2026-09-10 09:45:46.457 / 2026-10-06 09:52:35.814 | rolled out |
| `mobile_ios_contextual_nudges_v2` | 225,901 | shopper_id | iOS_contextual_nudges | off 112,816; on 113,085 | off | 2026-09-23 05:02:35.073 / 2026-10-07 23:59:56.122 | rolled out |
| `recore_abn_gdexp_13936` | 190,324 | visitor_guid | hp_core_metrics | control_variant 95,300; b_variant 95,024 | control_variant | 2026-10-12 14:39:23.737 / 2026-10-20 00:32:22.641 | rolled out |
| `MKT_SMS_516-UPM-D5_CM-24664` | 17,329 | shopper_id | cm-renewals-audience-recording | ctrl 5,924; testa 5,682; testb 5,723 | ctrl | 2026-08-27 11:49:51.959 / 2026-10-22 10:43:55.785 | rolled back |

Every cohort count above equals `get_experiment_analysis.healthMetric.summary.sampleCount` in the MCP payloads, and MIN/MAX `bucketed_mst_ts` equal `healthMetric.startTime/endTime` and `experiment_metadata_snap.first/last_user_bucket_mst_ts` (to the second) for all 5 experiments (SQL + MCP, 2026-11-13). `crossBucketCount` = 0 for all 5 in MCP.

### Two population shapes (important for every segment column)

- **Offsite / CRM-audience experiments** (`MKT_EM_177...`, `MKT_SMS_516...`; 294,546 rows): `device_breakpoint_name`, `market_site`, `new_existing_customer_name`, `ip_country_name`, `wam_free_trial_flag` are NULL on every row; `visit_count` = 1 on every row; `bucket_session_start_mst_ts` = `bucketed_mst_ts` on every row. `customer_type`, tenure and domain-ownership ARE filled. (SQL verified.)
- **Visitor-keyed onsite experiment** (`recore_abn_gdexp_13936`; 190,324 rows): `shopper_id` and `customer_type` are NULL on every row; `customer_tenure_day_segment` = `0-1 day` and `domain_ownership_count` = `0` on every row (defaults, not real tenure/portfolio); `wam_free_trial_flag` = FALSE on every row. (SQL verified.)
- **Shopper-keyed onsite experiments** (`agi_mon...`, `mobile_ios...`): all segment columns filled on >99.8% of rows (SQL).
- Segment cuts from this table have no significance test (verified doc) - descriptive only.

## Column index

| Column | Type | Null % | Distinct | One-line meaning | Confidence |
|---|---|---|---|---|---|
| `allocation_id` | string | 0.0 | 1,397,235 | 64-char hex id of one analyzed user in one experiment (PK with experiment_id) | Verified |
| `shopper_id` | string | 13.62 | 1,175,769 | GoDaddy shopper (customer account) id; NULL on visitor-keyed experiments | Verified |
| `visitor_guid` | string | 86.38 | 190,324 | Web visitor GUID; filled only on visitor-keyed experiments | Verified |
| `cohort_id` | string | 0.0 | 14 | Cohort / arm the user was bucketed into | Verified |
| `bucketed_mst_date` | date | 0.0 | 57 | Date (MST) of first bucketing; sort key, always filter | Verified |
| `bucketed_mst_ts` | timestamp | 0.0 | 1,343,963 | Timestamp (MST) of first bucketing; MIN/MAX = real start/end | Verified |
| `bucket_session_start_mst_ts` | timestamp | 0.0 | 1,339,325 | Start of the session associated with the bucketing (MST) | Inferred |
| `experiment_user_identifier` | string | 0.0 | 2 | Which id type keys the user (= traffic_type_enum) | Verified |
| `wam_free_trial_flag` | boolean | 21.08 | 2 | User had a WAM free trial | Inferred |
| `device_breakpoint_name` | string | 21.08 | 4 | Device class of the user (Desktop/Mobile/Other/Multiple) | Inferred |
| `market_site` | string | 21.19 | 53 | GoDaddy market site locale (e.g. en-us) or Multiple | Inferred |
| `new_existing_customer_name` | string | 21.08 | 2 | New Prospect vs Existing Customer segment | Verified |
| `ip_country_name` | string | 21.12 | 228 | Country from IP geolocation, or Multiple | Inferred |
| `customer_tenure_day_segment` | string | 0.0 | 4 | Customer tenure bucket in days | Inferred |
| `domain_ownership_count` | string | 0.0 | 5 | Bucket of domains owned by the customer | Inferred |
| `customer_type` | string | 13.64 | 5 | Customer type segment (Independent / Partner / Investor ...) | Inferred |
| `scorecard_template_name` | string | 0.0 | 4 | Scorecard template the users were analyzed under | Verified |
| `visit_count` | integer | 0.0 | 382 | Number of visits by the user in the analysis window | Inferred |
| `cln_create_utc_ts` | timestamp | 0.0 | 5 | UTC timestamp the rows of this experiment were built | Inferred |
| `experiment_id` | string | 0.0 | 5 | Experiment key; joins experiment_metadata_snap | Verified |

## Cross-table value map

| Column here | Shared with | Relationship |
|---|---|---|
| `experiment_id` | `experiment_metadata_snap.experiment_id`, `analysis_result_cln.experiment_id`, `experiment_order.experiment_id`; MCP `experimentId` | Same exact, case-sensitive key. Join to metadata many-to-one (INNER). |
| `cohort_id` | `analysis_result_cln.cohort_id` / `control_cohort`, `experiment_order.cohort_id`, `experiment_metadata_snap.baseline_cohort_id` / `deciding_cohort_id_text` / `cohort_name_list`; MCP `config.cohorts[].name`, `baselineCohortId` | Same cohort strings (e.g. `control_variant`, `b_variant`, `off`, `on`, `ctrl`, `control`, `default`). Cohort names are NOT unique across experiments (`testa`/`testb` occur in 2 experiments here) - always pair with `experiment_id`. |
| `experiment_user_identifier` | `experiment_metadata_snap.traffic_type_enum`; MCP `config.trafficType` | Same value per experiment (verified on all 5). MCP enum has 10 values (visitor_guid, shopper_id, care_conversation_id, care_ucid_jomax_id, customer_id, resource_id, subscription_lifetime_id, billing_cycle_id, auction_id, payment_id); only 2 occur here. |
| `scorecard_template_name` | `experiment_metadata_snap.scorecard_template_name`, `analysis_result_cln.scorecard_template_name`, `experiment_order.scorecard_template_name`; MCP `config.scorecardTemplate` | Same value per experiment (verified on all 5 vs metadata and MCP config). |
| `bucketed_mst_ts` (MIN/MAX) | `experiment_metadata_snap.first_user_bucket_mst_ts` / `last_user_bucket_mst_ts`; MCP `healthMetric.startTime/endTime` | Equal to the second (verified on all 5). |
| `COUNT(*)` by cohort | MCP `healthMetric.summary.sampleCount` | Equal (verified on all 5). |
| `bucketed_mst_date`, `bucketed_mst_ts`, `shopper_id`, `visitor_guid`, `experiment_user_identifier` | `experiment_order` | Same column names and formats (experiment_order carries the bucketing context onto orders). |
| `device_breakpoint_name`, `market_site`, `new_existing_customer_name`, `ip_country_name`, `customer_tenure_day_segment`, `domain_ownership_count`, `customer_type` | `experiment_order` (same names, same value sets: device 4, market 53, tenure 4, domain 5, customer_type 5, new/existing 2); `analysis_result_cln` (same names, except `device_name` there) | In the `analysis_result_cln` profile every row of these columns = `ALL` (overall, no segment rows), so per-segment statistical results are not available from it in that scope. |
| `new_existing_customer_name` | MCP enum `customerReportFilter` = [`Existing Customer`, `New Prospect`] | Identical value set. |

## Columns

### `allocation_id` - STRING
- **Meaning:** Identifier of one analyzed user's allocation in one experiment. Primary key together with `experiment_id`. No equivalent field in the Hivemind MCP payloads.
- **Profile (scope):** rows 1,397,235, null 0.0%, empty 0.0%, distinct 1,397,235, length 64-64.
- **Values:** 64-character lowercase hexadecimal string. Example `dff46561662ad9d4f8ddffe03b72fbb0733937b091b832d9a468a628e652250b`. Every value occurs once: 1,397,235 distinct (high cardinality, unique). The format is consistent with a SHA-256 hash; what is hashed is not documented — owner input needed.
- **How to use / Traps:** Use `COUNT(*)` (equal to `COUNT(DISTINCT allocation_id)`) for user counts. No other hivemind_mart table in the sources carries it, so it is not a join key. Compare as exact lowercase string.
- **Confidence:** Verified (uniqueness and format from profile; PK from verified doc). Hash construction: Unknown (ETL code would resolve).

### `shopper_id` - STRING
- **Meaning:** GoDaddy shopper (customer account) id of the user. The user key when `experiment_user_identifier = 'shopper_id'`.
- **Profile (scope):** rows 1,397,235, null 13.62%, empty 0.0%, distinct 1,175,769, length 6-9.
- **Values:** Numeric string, 6-9 characters. Top values: `NULL` (190,324), `290109905` (3), `676008286` (3), `85045643` (3), `638348284` (3), `285502720` (3). 1,175,769 distinct (high cardinality). Max repetition 3 = the same shopper in up to 3 of the 5 profiled experiments (one row per experiment).
- **How to use / Traps:** NULL (190,324 rows = exactly the visitor-keyed `recore_abn_gdexp_13936` rows) means the experiment is keyed by visitor, not that the user is anonymous. On shopper-keyed experiments it is filled on every row and unique within the experiment. Stored as text - compare as text, do not cast for joins. Identifier: do not paste values into tickets/chats.
- **Confidence:** Verified (SQL: NULL only on visitor-keyed rows; filled on all shopper-keyed rows).

### `visitor_guid` - STRING
- **Meaning:** Web visitor GUID (cookie-level visitor id). The user key when `experiment_user_identifier = 'visitor_guid'`.
- **Profile (scope):** rows 1,397,235, null 86.38%, empty 0.0%, distinct 190,324, length 36-36.
- **Values:** Lowercase UUID, 36 characters (8-4-4-4-12). Example `d39311a4-2980-4ba1-a4cd-09a1230b19bf`. 190,324 distinct non-null values, each once (high cardinality).
- **How to use / Traps:** Filled on every row of visitor-keyed experiments (here only `recore_abn_gdexp_13936`, 190,324 rows) and NULL on every row of shopper-keyed experiments (1,206,911 rows). NULL means 'experiment is shopper-keyed'. Do not join users across experiments with different id types. Identifier: treat as PII.
- **Confidence:** Verified (SQL).

### `cohort_id` - STRING
- **Meaning:** Cohort (arm) the user was bucketed into. Hivemind API: cohort `name` in `config.cohorts[]`; keys of `healthMetric.summary.sampleCount`.
- **Profile (scope):** rows 1,397,235, null 0.0%, empty 0.0%, distinct 14, length 2-15.
- **Values:** values_complete (14 values; experiment per value from SQL, CONTROL label from `experiment_metadata_snap.baseline_cohort_id` = MCP `baselineCohortId`):

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `variant2` | 172,321 | 12.33% | Treatment arm of `agi_mon_email_banner_multi_surface` (control there is `default`) |
| `variant1` | 171,652 | 12.29% | Treatment arm of `agi_mon_email_banner_multi_surface` |
| `variant3` | 171,369 | 12.26% | Treatment arm of `agi_mon_email_banner_multi_surface` |
| `default` | 171,122 | 12.25% | CONTROL (baseline_cohort_id) of `agi_mon_email_banner_multi_surface` |
| `on` | 113,085 | 8.09% | Treatment arm of `mobile_ios_contextual_nudges_v2`; rolled out (MCP config split now on=100) |
| `off` | 112,816 | 8.07% | CONTROL (baseline_cohort_id) of `mobile_ios_contextual_nudges_v2` |
| `control_variant` | 95,300 | 6.82% | CONTROL (baseline_cohort_id) of `recore_abn_gdexp_13936` |
| `b_variant` | 95,024 | 6.8% | Treatment arm of `recore_abn_gdexp_13936`; rolled out (MCP config split now b_variant=100) |
| `testb` | 61,204 | 4.38% | Treatment arm in TWO experiments: `MKT_EM_177...CM-24927` (55,481) + `MKT_SMS_516...CM-24664` (5,723) |
| `testa` | 61,018 | 4.37% | Treatment arm in TWO experiments: `MKT_EM_177...CM-24927` (55,336; rolled-out arm, config split now 100) + `MKT_SMS_516...CM-24664` (5,682) |
| `testc` | 55,622 | 3.98% | Treatment arm of `MKT_EM_177_Failed-Billing-Manual-D28_CM-24927` |
| `control` | 55,502 | 3.97% | CONTROL (baseline_cohort_id) of `MKT_EM_177_Failed-Billing-Manual-D28_CM-24927` |
| `testd` | 55,276 | 3.96% | Treatment arm of `MKT_EM_177_Failed-Billing-Manual-D28_CM-24927` |
| `ctrl` | 5,924 | 0.42% | CONTROL (baseline_cohort_id) of `MKT_SMS_516-UPM-D5_CM-24664` (rolled back; MCP config split now ctrl=100) |
- **How to use / Traps:** Always group with `experiment_id` - names repeat across experiments (`testa`, `testb`). Control is never flagged here: join `experiment_metadata_snap.baseline_cohort_id`. Never infer control from names. Current MCP `config.cohorts[].split` reflects post-rollout/rollback state (e.g. b_variant 100 / control_variant 0); the near-equal row counts here are the experiment-time split. Cohorts with 0 split in config still have rows. MCP analysis warnings like "Analysis skipped: ['b_variant', 'control_variant'] cohorts only have one value" refer to these cohort names for a given metric, not to missing rows here.
- **Confidence:** Verified (SQL + MCP config + analysis sampleCount).

### `bucketed_mst_date` - DATE
- **Meaning:** Calendar date (MST) of the user's first bucketing into the experiment. Sort key. Equals `DATE(bucketed_mst_ts)` on every row (0 mismatches in all 5 experiments, SQL).
- **Profile (scope):** rows 1,397,235, null 0.0%, empty 0.0%, distinct 57, min 2026-08-27, max 2026-10-22.
- **Values:** values_complete: 57 dates, 2026-08-27 to 2026-10-22. Every value means 'users first bucketed on that MST date'; counts are summed across the 5 experiments, so the shape reflects which experiments were live (MKT_SMS_516 runs across the whole range at a few hundred/day and is the only experiment on 2026-08-27..09-09, 10-11 and 10-21..10-22; MKT_EM_177 09-10..10-06; agi_mon 09-22..10-10; mobile_ios 09-23..10-07; recore 10-12..10-20 00:32, so 10-20 is mostly MKT_SMS_516 - ranges from the MIN/MAX `bucketed_mst_ts` table above).

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `2026-09-23` | 104,848 | 7.5% | users first bucketed on this date |
| `2026-09-24` | 80,184 | 5.74% | users first bucketed on this date |
| `2026-09-28` | 69,716 | 4.99% | users first bucketed on this date |
| `2026-09-29` | 66,231 | 4.74% | users first bucketed on this date |
| `2026-09-30` | 65,948 | 4.72% | users first bucketed on this date |
| `2026-09-25` | 63,793 | 4.57% | users first bucketed on this date |
| `2026-10-05` | 61,277 | 4.39% | users first bucketed on this date |
| `2026-09-22` | 59,975 | 4.29% | users first bucketed on this date |
| `2026-10-06` | 58,789 | 4.21% | users first bucketed on this date |
| `2026-10-02` | 54,958 | 3.93% | users first bucketed on this date |
| `2026-10-01` | 52,686 | 3.77% | users first bucketed on this date |
| `2026-09-27` | 49,885 | 3.57% | users first bucketed on this date |
| `2026-10-07` | 48,335 | 3.46% | users first bucketed on this date |
| `2026-09-26` | 47,518 | 3.4% | users first bucketed on this date |
| `2026-10-04` | 43,476 | 3.11% | users first bucketed on this date |
| `2026-10-03` | 41,622 | 2.98% | users first bucketed on this date |
| `2026-10-13` | 38,300 | 2.74% | users first bucketed on this date |
| `2026-10-08` | 36,408 | 2.61% | users first bucketed on this date |
| `2026-10-14` | 31,656 | 2.27% | users first bucketed on this date |
| `2026-10-09` | 30,623 | 2.19% | users first bucketed on this date |
| `2026-10-15` | 29,028 | 2.08% | users first bucketed on this date |
| `2026-10-19` | 28,379 | 2.03% | users first bucketed on this date |
| `2026-10-16` | 23,100 | 1.65% | users first bucketed on this date |
| `2026-10-10` | 20,582 | 1.47% | users first bucketed on this date |
| `2026-10-18` | 15,185 | 1.09% | users first bucketed on this date |
| `2026-10-17` | 14,434 | 1.03% | users first bucketed on this date |
| `2026-09-10` | 14,175 | 1.01% | users first bucketed on this date |
| `2026-09-11` | 13,033 | 0.93% | users first bucketed on this date |
| `2026-10-12` | 12,680 | 0.91% | users first bucketed on this date |
| `2026-09-12` | 12,504 | 0.89% | users first bucketed on this date |
| `2026-09-13` | 11,841 | 0.85% | users first bucketed on this date |
| `2026-09-17` | 11,832 | 0.85% | users first bucketed on this date |
| `2026-09-19` | 11,672 | 0.84% | users first bucketed on this date |
| `2026-09-18` | 11,628 | 0.83% | users first bucketed on this date |
| `2026-09-16` | 11,492 | 0.82% | users first bucketed on this date |
| `2026-09-14` | 11,321 | 0.81% | users first bucketed on this date |
| `2026-09-20` | 11,076 | 0.79% | users first bucketed on this date |
| `2026-09-15` | 10,815 | 0.77% | users first bucketed on this date |
| `2026-09-21` | 10,655 | 0.76% | users first bucketed on this date |
| `2026-09-07` | 356 | 0.03% | users first bucketed on this date |
| `2026-08-27` | 348 | 0.02% | users first bucketed on this date |
| `2026-10-20` | 336 | 0.02% | users first bucketed on this date |
| `2026-09-01` | 332 | 0.02% | users first bucketed on this date |
| `2026-09-06` | 326 | 0.02% | users first bucketed on this date |
| `2026-08-31` | 324 | 0.02% | users first bucketed on this date |
| `2026-08-30` | 314 | 0.02% | users first bucketed on this date |
| `2026-09-03` | 313 | 0.02% | users first bucketed on this date |
| `2026-10-22` | 312 | 0.02% | users first bucketed on this date |
| `2026-09-09` | 310 | 0.02% | users first bucketed on this date |
| `2026-09-04` | 308 | 0.02% | users first bucketed on this date |
| `2026-10-11` | 304 | 0.02% | users first bucketed on this date |
| `2026-10-21` | 296 | 0.02% | users first bucketed on this date |
| `2026-09-08` | 287 | 0.02% | users first bucketed on this date |
| `2026-08-28` | 286 | 0.02% | users first bucketed on this date |
| `2026-09-02` | 281 | 0.02% | users first bucketed on this date |
| `2026-09-05` | 280 | 0.02% | users first bucketed on this date |
| `2026-08-29` | 262 | 0.02% | users first bucketed on this date |
- **How to use / Traps:** MANDATORY filter on every query, with `experiment_id`: `BETWEEN DATE(first_user_bucket_mst_ts) - 1 AND DATE(last_user_bucket_mst_ts) + 1` from experiment_metadata_snap. Rows are deduplicated to first bucket, so counting by this date gives NEW users per day, not daily exposure; cumulative sample by day = running sum.
- **Confidence:** Verified.

### `bucketed_mst_ts` - TIMESTAMP
- **Meaning:** Timestamp (true MST, millisecond precision) of the user's first bucketing. MIN/MAX per experiment = real start / end of bucketing = `experiment_metadata_snap.first/last_user_bucket_mst_ts` = MCP `healthMetric.startTime/endTime`.
- **Profile (scope):** rows 1,397,235, null 0.0%, empty 0.0%, distinct 1,343,963, min 2026-08-27 11:49:51.959000, max 2026-10-22 10:43:55.785000.
- **Values:** Range 2026-08-27 11:49:51.959 to 2026-10-22 10:43:55.785. 1,343,963 distinct (high cardinality). Top values: `2026-09-10 09:45:49.488000` (11), `2026-09-10 09:45:49.191000` (9), `2026-09-10 09:45:49.619000` (9), `2026-09-10 09:45:49.850000` (9), `2026-09-10 09:45:48.947000` (8) - all in the 2026-09-10 09:45 MKT_EM_177 send burst (several users bucketed in the same millisecond).
- **How to use / Traps:** Use MIN/MAX for real start/end and duration, not `config.startDate/endDate` (config endDate is NULL for 4 of the 5; recore has no config startDate at all). Millisecond ties are normal for batch email/SMS sends. MST, not UTC. MCP `config.rolloutDate` is epoch-ms UTC and marks rollout, a different event.
- **Confidence:** Verified.

### `bucket_session_start_mst_ts` - TIMESTAMP

> ⚠️ **OWNER INPUT NEEDED** (OQ-UA-01): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Start timestamp (MST) of the session associated with the user's bucketing. No MCP equivalent found in sources.
- **Profile (scope):** rows 1,397,235, null 0.0%, empty 0.0%, distinct 1,339,325, min 2026-08-09 23:38:50.542000, max 2026-10-22 10:43:55.785000.
- **Values:** Range 2026-08-09 23:38:50.542000 to 2026-10-22 10:43:55.785000; 1,339,325 distinct (high cardinality). Never NULL. Relationship to `bucketed_mst_ts` (SQL): offsite experiments (MKT_EM_177, MKT_SMS_516) equal on every row; agi_mon earlier than bucket on 678,574 rows, later on 7,883, up to 60 days earlier; mobile_ios later than bucket on 147,663 of 225,901 rows; recore earlier on 163,968, later on 26,324. Its minimum (2026-08-09) predates every experiment start.
- **How to use / Traps:** Do NOT use for experiment start/end or day counts - use `bucketed_mst_ts`. Which session is chosen and why it can be later than the bucket time is not documented; treat as context only. For offsite sends it copies the bucket time.
- **Confidence:** Inferred (from name and the SQL relationships above). Session-selection rule Unknown - ETL code / pipeline owner would resolve.

### `experiment_user_identifier` - STRING
- **Meaning:** The id TYPE that keys users in this experiment (not an id). Hivemind API: `config.trafficType`; equals `experiment_metadata_snap.traffic_type_enum`.
- **Profile (scope):** rows 1,397,235, null 0.0%, empty 0.0%, distinct 2, length 10-12.
- **Values:** values_complete:

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `shopper_id` | 1,206,911 | 86.38% | User keyed by `shopper_id` (agi_mon, MKT_EM_177, mobile_ios, MKT_SMS_516); `visitor_guid` NULL |
| `visitor_guid` | 190,324 | 13.62% | User keyed by `visitor_guid` (recore_abn_gdexp_13936); `shopper_id` NULL |

Other MCP `trafficType` values possible for other experiments (not observed here): care_conversation_id, care_ucid_jomax_id, customer_id, resource_id, subscription_lifetime_id, billing_cycle_id, auction_id, payment_id. Which column would hold those ids is not documented — owner input needed.
- **How to use / Traps:** Exactly one value per experiment, never NULL. Use it to choose the id column to count/join on. It is a type label, not an id.
- **Confidence:** Verified (SQL vs metadata and MCP config).

### `wam_free_trial_flag` - BOOLEAN

> ⚠️ **OWNER INPUT NEEDED** (OQ-UA-02): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Flag that the user had a WAM free trial. WAM is GoDaddy's Websites + Marketing product (expansion inferred from GoDaddy product naming; the flag's definition is not in sources).
- **Profile (scope):** rows 1,397,235, null 21.08%, empty 0.0%, distinct 2.
- **Values:** values_complete:

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `False` | 915,359 | 65.51% | Not flagged as WAM free trial (also the value on every row of visitor-keyed recore) |
| `NULL` | 294,546 | 21.08% | Not computed: exactly the 294,546 offsite rows (MKT_EM_177 + MKT_SMS_516) |
| `True` | 187,330 | 13.41% | Flagged as WAM free trial; occurs only in shopper-keyed onsite experiments (agi_mon, mobile_ios) |
- **How to use / Traps:** TRUE = flagged free-trial user; FALSE = not flagged (on visitor-keyed tests FALSE is on 100% of rows, so it does not prove 'not a trial user'); NULL = not computed (offsite experiment). Exclude NULL when computing a share. As-of time of the flag is not documented.
- **Confidence:** Inferred (meaning from name; NULL/FALSE patterns verified by SQL).

### `device_breakpoint_name` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-UA-03): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Device class (responsive breakpoint) of the user's sessions. `analysis_result_cln` names the same segment `device_name`.
- **Profile (scope):** rows 1,397,235, null 21.08%, empty 0.0%, distinct 4, length 5-8.
- **Values:** values_complete:

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `Desktop` | 897,529 | 64.24% | Desktop-breakpoint sessions |
| `NULL` | 294,546 | 21.08% | Not computed: exactly the 294,546 offsite rows |
| `Multiple` | 110,519 | 7.91% | More than one device class across the user sessions (inferred from label; 11,608 agi_mon rows are Multiple with visit_count = 1, so the rule is not purely multi-visit) **⚠️ OWNER INPUT NEEDED (OQ-UA-03)** |
| `Mobile` | 92,628 | 6.63% | Mobile-breakpoint sessions |
| `Other` | 2,013 | 0.14% | meaning not documented — owner input needed **⚠️ OWNER INPUT NEEDED (OQ-UA-03)** |
- **How to use / Traps:** Descriptive segment only (no significance test). NULL = offsite experiment, not unknown device. Report `Multiple` separately.
- **Confidence:** Inferred (value set and NULL pattern verified; Multiple/Other meanings from labels).

### `market_site` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-UA-03): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** GoDaddy market site (locale) the user was on, as a `language-region` code, or `Multiple`.
- **Profile (scope):** rows 1,397,235, null 21.19%, empty 0.0%, distinct 53, length 5-8.
- **Values:** values_complete (53 non-null values + NULL):

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `en-us` | 568,717 | 40.7% | English / United States site (decoded from `language-region` locale code; inferred) |
| `NULL` | 296,086 | 21.19% | No site attributed: all 294,546 offsite rows (MKT_EM_177, MKT_SMS_516) plus 1,540 onsite rows (agi_mon 894, recore 646) with no site (SQL) |
| `Multiple` | 107,045 | 7.66% | User seen on more than one market site (inferred from label) **⚠️ OWNER INPUT NEEDED (OQ-UA-03)** |
| `en-in` | 88,957 | 6.37% | English / India site (decoded from `language-region` locale code; inferred) |
| `en-gb` | 76,437 | 5.47% | English / United Kingdom site (decoded from `language-region` locale code; inferred) |
| `en-ca` | 46,882 | 3.36% | English / Canada site (decoded from `language-region` locale code; inferred) |
| `en-au` | 33,293 | 2.38% | English / Australia site (decoded from `language-region` locale code; inferred) |
| `en-ph` | 23,916 | 1.71% | English / Philippines site (decoded from `language-region` locale code; inferred) |
| `en-ie` | 22,468 | 1.61% | English / Ireland site (decoded from `language-region` locale code; inferred) |
| `pt-br` | 20,199 | 1.45% | Portuguese / Brazil site (decoded from `language-region` locale code; inferred) |
| `en-ae` | 14,836 | 1.06% | English / United Arab Emirates site (decoded from `language-region` locale code; inferred) |
| `es-mx` | 12,036 | 0.86% | Spanish / Mexico site (decoded from `language-region` locale code; inferred) |
| `tr-tr` | 6,829 | 0.49% | Turkish / Turkey site (decoded from `language-region` locale code; inferred) |
| `de-de` | 6,492 | 0.46% | German / Germany site (decoded from `language-region` locale code; inferred) |
| `es-es` | 6,398 | 0.46% | Spanish / Spain site (decoded from `language-region` locale code; inferred) |
| `en-pk` | 5,425 | 0.39% | English / Pakistan site (decoded from `language-region` locale code; inferred) |
| `es-co` | 5,398 | 0.39% | Spanish / Colombia site (decoded from `language-region` locale code; inferred) |
| `es-us` | 4,560 | 0.33% | Spanish / United States site (decoded from `language-region` locale code; inferred) |
| `en-za` | 4,398 | 0.31% | English / South Africa site (decoded from `language-region` locale code; inferred) |
| `fr-fr` | 3,885 | 0.28% | French / France site (decoded from `language-region` locale code; inferred) |
| `it-it` | 3,355 | 0.24% | Italian / Italy site (decoded from `language-region` locale code; inferred) |
| `en-sg` | 3,262 | 0.23% | English / Singapore site (decoded from `language-region` locale code; inferred) |
| `en-il` | 2,954 | 0.21% | English / Israel site (decoded from `language-region` locale code; inferred) |
| `zh-tw` | 2,506 | 0.18% | Chinese / Taiwan site (decoded from `language-region` locale code; inferred) |
| `nl-nl` | 2,456 | 0.18% | Dutch / Netherlands site (decoded from `language-region` locale code; inferred) |
| `zh-sg` | 2,407 | 0.17% | Chinese / Singapore site (decoded from `language-region` locale code; inferred) |
| `es-ar` | 2,234 | 0.16% | Spanish / Argentina site (decoded from `language-region` locale code; inferred) |
| `en-my` | 2,201 | 0.16% | English / Malaysia site (decoded from `language-region` locale code; inferred) |
| `es-pe` | 2,148 | 0.15% | Spanish / Peru site (decoded from `language-region` locale code; inferred) |
| `en-nz` | 1,510 | 0.11% | English / New Zealand site (decoded from `language-region` locale code; inferred) |
| `th-th` | 1,364 | 0.1% | Thai / Thailand site (decoded from `language-region` locale code; inferred) |
| `es-cl` | 1,346 | 0.1% | Spanish / Chile site (decoded from `language-region` locale code; inferred) |
| `sv-se` | 1,332 | 0.1% | Swedish / Sweden site (decoded from `language-region` locale code; inferred) |
| `pl-pl` | 1,284 | 0.09% | Polish / Poland site (decoded from `language-region` locale code; inferred) |
| `fr-ca` | 1,266 | 0.09% | French / Canada site (decoded from `language-region` locale code; inferred) |
| `uk-ua` | 1,203 | 0.09% | Ukrainian / Ukraine site (decoded from `language-region` locale code; inferred) |
| `zh-hk` | 1,111 | 0.08% | Chinese / Hong Kong site (decoded from `language-region` locale code; inferred) |
| `ko-kr` | 951 | 0.07% | Korean / South Korea site (decoded from `language-region` locale code; inferred) |
| `pt-pt` | 935 | 0.07% | Portuguese / Portugal site (decoded from `language-region` locale code; inferred) |
| `de-ch` | 930 | 0.07% | German / Switzerland site (decoded from `language-region` locale code; inferred) |
| `id-id` | 822 | 0.06% | Indonesian / Indonesia site (decoded from `language-region` locale code; inferred) |
| `vi-vn` | 734 | 0.05% | Vietnamese / Vietnam site (decoded from `language-region` locale code; inferred) |
| `de-at` | 722 | 0.05% | German / Austria site (decoded from `language-region` locale code; inferred) |
| `ja-jp` | 684 | 0.05% | Japanese / Japan site (decoded from `language-region` locale code; inferred) |
| `nl-be` | 657 | 0.05% | Dutch / Belgium site (decoded from `language-region` locale code; inferred) |
| `en-hk` | 642 | 0.05% | English / Hong Kong site (decoded from `language-region` locale code; inferred) |
| `nb-no` | 548 | 0.04% | Norwegian Bokmal / Norway site (decoded from `language-region` locale code; inferred) |
| `da-dk` | 514 | 0.04% | Danish / Denmark site (decoded from `language-region` locale code; inferred) |
| `hi-in` | 331 | 0.02% | Hindi / India site (decoded from `language-region` locale code; inferred) |
| `ar-ae` | 229 | 0.02% | Arabic / United Arab Emirates site (decoded from `language-region` locale code; inferred) |
| `fr-be` | 174 | 0.01% | French / Belgium site (decoded from `language-region` locale code; inferred) |
| `fr-ch` | 128 | 0.01% | French / Switzerland site (decoded from `language-region` locale code; inferred) |
| `it-ch` | 37 | 0.0% | Italian / Switzerland site (decoded from `language-region` locale code; inferred) |
| `es-ve` | 1 | 0.0% | Spanish / Venezuela site (decoded from `language-region` locale code; inferred) |
- **How to use / Traps:** Lowercase `ll-rr` codes; compare exactly. `Multiple` = more than one site; exclude or report separately. Site locale is not the user's country - use `ip_country_name` for geography. NULL on offsite experiments.
- **Confidence:** Inferred (locale decoding from the code pattern; NULL breakdown verified by SQL).

### `new_existing_customer_name` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-UA-05): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** New vs existing customer segment. Value set equals the MCP enum `customerReportFilter` (Existing Customer, New Prospect).
- **Profile (scope):** rows 1,397,235, null 21.08%, empty 0.0%, distinct 2, length 12-17.
- **Values:** values_complete:

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `Existing Customer` | 978,940 | 70.06% | Existing GoDaddy customer (MCP customerReportFilter value) |
| `NULL` | 294,546 | 21.08% | Not computed: exactly the 294,546 offsite rows |
| `New Prospect` | 123,749 | 8.86% | New prospect, not yet a customer (MCP customerReportFilter value) |
- **How to use / Traps:** NULL = offsite experiment. Visitor-keyed recore is also evaluated (188,329 Existing Customer, 1,995 New Prospect, SQL). The exact cut-off rule is not documented. Do not confuse with `customer_tenure_day_segment`.
- **Confidence:** Verified (value set from MCP enum; NULL pattern by SQL). Exact rule Unknown.

### `ip_country_name` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-UA-03): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Country resolved from the user's IP address (English country name), or `Multiple`.
- **Profile (scope):** rows 1,397,235, null 21.12%, empty 0.0%, distinct 228, length 4-32.
- **Values:** 228 distinct (high cardinality; not fully listed). Top values: `United States` (492,137), `NULL` (295,091), `India` (127,770), `United Kingdom` (71,306), `Canada` (58,696), `Multiple` (40,832), `Australia` (39,335), `Brazil` (25,517), `Mexico` (16,996), `United Arab Emirates` (11,292), `Spain` (10,743), `Turkey` (10,741), `Germany` (10,319), `Pakistan` (9,817), `Colombia` (9,103). Strings 4-32 characters.
- **How to use / Traps:** NULL (295,091) = 294,546 offsite rows + 545 onsite rows without geolocation (SQL). `Multiple` = more than one country across the user's sessions (inferred from label; 33,995 rows on agi_mon). **⚠️ OWNER INPUT NEEDED (OQ-UA-03):** the rule that produces `Multiple` is not confirmed. Use for geography rather than `market_site`. Match names exactly.
- **Confidence:** Inferred (meaning from name; NULL composition by SQL).

### `customer_tenure_day_segment` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-UA-04): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Bucket of customer tenure in days (anchor - account creation vs first purchase - inferred, not documented).
- **Profile (scope):** rows 1,397,235, null 0.0%, empty 0.0%, distinct 4, length 7-10.
- **Values:** values_complete:

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `365+ day` | 877,781 | 62.82% | Tenure 365 days or more |
| `0-1 day` | 347,369 | 24.86% | Tenure 0-1 day; ALSO the default on every visitor-keyed row (all 190,324 recore rows), not real tenure there |
| `31-365 day` | 143,041 | 10.24% | Tenure 31-365 days |
| `8-30 day` | 29,044 | 2.08% | Tenure 8-30 days |

No `2-7 day` bucket occurs in scope; whether one exists is not documented — owner input needed.
- **How to use / Traps:** Never NULL, but on visitor-keyed experiments the value is a default - exclude them from tenure cuts. As-of date not documented.
- **Confidence:** Inferred (labels; visitor default verified by SQL).

### `domain_ownership_count` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-UA-04): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Bucketed number of domains owned by the user's account. STRING labels, not a number.
- **Profile (scope):** rows 1,397,235, null 0.0%, empty 0.0%, distinct 5, length 1-4.
- **Values:** values_complete:

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `1` | 443,918 | 31.77% | Owns 1 domain |
| `2-5` | 369,142 | 26.42% | Owns 2-5 domains |
| `0` | 291,222 | 20.84% | Owns 0 domains; ALSO the default on every visitor-keyed row (all 190,324 recore rows) |
| `10+` | 166,699 | 11.93% | Owns 10+ domains (boundary vs 6-10 at exactly 10 not documented) |
| `6-10` | 126,254 | 9.04% | Owns 6-10 domains |
- **How to use / Traps:** Do not SUM/AVG - it is a label. Sort with an explicit CASE (string sort gives 0,1,10+,2-5,6-10). `0` on visitor-keyed experiments is a default. Never NULL.
- **Confidence:** Inferred (name/labels; visitor default verified by SQL).

### `customer_type` - STRING

> ⚠️ **OWNER INPUT NEEDED** (OQ-UA-05): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Customer type segment of the shopper.
- **Profile (scope):** rows 1,397,235, null 13.64%, empty 0.0%, distinct 5, length 7-25.
- **Values:** values_complete:

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `International Independent` | 597,334 | 42.75% | Independent customer outside the US (inferred from label) |
| `US Independent` | 410,561 | 29.38% | Independent customer in the US (inferred from label) |
| `NULL` | 190,622 | 13.64% | Not available: 190,324 visitor-keyed recore rows (no shopper) + 298 shopper rows without a type (agi_mon 213, mobile_ios 83, MKT_EM_177 2; SQL) |
| `Partner` | 70,396 | 5.04% | Partner customer (inferred from label; definition not documented) |
| `Not Evaluated` | 64,695 | 4.63% | Shopper not classified by the source (inferred from label) **⚠️ OWNER INPUT NEEDED (OQ-UA-05)** |
| `Investor` | 63,627 | 4.55% | Domain investor customer (inferred from label) |
- **How to use / Traps:** Filled for offsite experiments too (unlike device/market). NULL mostly means visitor-keyed experiment. Keep `Not Evaluated` as its own bucket, distinct from NULL.
- **Confidence:** Inferred (labels; NULL composition verified by SQL).

### `scorecard_template_name` - STRING
- **Meaning:** Scorecard template the users were analyzed under. MCP `config.scorecardTemplate`; same as `experiment_metadata_snap.scorecard_template_name` and `analysis_result_cln.scorecard_template_name`.
- **Profile (scope):** rows 1,397,235, null 0.0%, empty 0.0%, distinct 4, length 15-30.
- **Values:** values_complete:

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `mon_banner_baseline` | 686,464 | 49.13% | Scorecard of agi_mon_email_banner_multi_surface |
| `cm-renewals-audience-recording` | 294,546 | 21.08% | Shared by MKT_EM_177 (277,217) and MKT_SMS_516 (17,329) |
| `iOS_contextual_nudges` | 225,901 | 16.17% | Scorecard of mobile_ios_contextual_nudges_v2 |
| `hp_core_metrics` | 190,324 | 13.62% | Scorecard of recore_abn_gdexp_13936 |
- **How to use / Traps:** One value per experiment. Context only - do not filter on it to select experiments (one template is shared by several experiments); filter by `experiment_id`.
- **Confidence:** Verified (SQL vs metadata and MCP config).

### `visit_count` - INTEGER

> ⚠️ **OWNER INPUT NEEDED** (OQ-UA-06): definition not clear — Hivemind owner to fill in. See the owner table at the end of this file.

- **Meaning:** Number of visits (sessions) by the user (inferred from name; counting window not documented).
- **Profile (scope):** rows 1,397,235, null 0.0%, empty 0.0%, distinct 382, min 1, max 1311, avg 2.7744.
- **Values:** Integer 1-1,311, avg 2.7744; 382 distinct (not fully listed). Top values: `1` (865,874), `2` (210,866), `3` (106,351), `4` (60,400), `5` (37,604), `6` (24,905), `7` (17,169), `8` (12,291), `9` (9,212), `10` (7,199). Never NULL, never 0.
- **How to use / Traps:** Offsite experiments have `1` on every row (default - no web visits counted, SQL). Heavy right tail (max 1,311 on mobile_ios) - use medians/buckets. Not a Hivemind-tested metric.
- **Confidence:** Inferred (name + distribution; offsite = 1 verified by SQL).

### `cln_create_utc_ts` - TIMESTAMP
- **Meaning:** UTC timestamp when this experiment's rows were built in the table (inferred from the `cln_create` name).
- **Profile (scope):** rows 1,397,235, null 0.0%, empty 0.0%, distinct 5, min 2026-10-07 01:44:10.103470, max 2026-10-23 01:44:13.518123.
- **Values:** values_complete (one value per experiment; SQL confirms a single value on all agi_mon rows and the counts match each experiment's rows):

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `2026-10-12 01:30:53.691363` | 686,464 | 49.13% | Build time of agi_mon_email_banner_multi_surface (686,464 rows) |
| `2026-10-07 01:44:10.103470` | 277,217 | 19.84% | Build time of MKT_EM_177 (277,217 rows) |
| `2026-10-13 13:44:10.582817` | 225,901 | 16.17% | Build time of mobile_ios_contextual_nudges_v2 (225,901 rows) |
| `2026-10-21 01:59:21.838167` | 190,324 | 13.62% | Build time of recore_abn_gdexp_13936 (190,324 rows) |
| `2026-10-23 01:44:13.518123` | 17,329 | 1.24% | Build time of MKT_SMS_516 (17,329 rows) |
- **How to use / Traps:** UTC, not MST. Each build lands shortly after the experiment's last bucketing (e.g. MKT_EM_177 last bucket 2026-10-06, built 2026-10-07), so it indicates freshness. Do not use as an experiment date.
- **Confidence:** Inferred (meaning from name; one value per experiment from counts/SQL).

### `experiment_id` - STRING
- **Meaning:** Experiment key; MCP `experimentId` / `config.id`. Joins `experiment_metadata_snap`, `analysis_result_cln`, `experiment_order`.
- **Profile (scope):** rows 1,397,235, null 0.0%, empty 0.0%, distinct 5, length 22-45.
- **Values:** values_complete (profile scope only; the full table holds more experiments):

| Value | Rows | % of scope | Meaning |
|---|---|---|---|
| `agi_mon_email_banner_multi_surface` | 686,464 | 49.13% | 4 cohorts, triggered, status killed (MCP) |
| `MKT_EM_177_Failed-Billing-Manual-D28_CM-24927` | 277,217 | 19.84% | 5 cohorts, offsite, rolled out (MCP) |
| `mobile_ios_contextual_nudges_v2` | 225,901 | 16.17% | on/off, rolled out (MCP) |
| `recore_abn_gdexp_13936` | 190,324 | 13.62% | visitor-keyed, 2 cohorts, rolled out (MCP) |
| `MKT_SMS_516-UPM-D5_CM-24664` | 17,329 | 1.24% | 3 cohorts, offsite, rolled back (MCP) |
- **How to use / Traps:** Exact, case-sensitive match (mixed case, hyphens and underscores; 22-45 chars in scope). Always filter one experiment per statement plus a `bucketed_mst_date` range. Never NULL.
- **Confidence:** Verified.

## ⚠️ **OWNER INPUT NEEDED** — definitions only the Hivemind owners can supply

The definitions below are **not clear** from the data, Hivemind MCP or any document we could find. **A Hivemind data owner needs to fill in the "Owner answer" column.** Until then, agents must answer "definition not confirmed by the Hivemind owners" for these and must not guess.

| ID | Column / value | Question for the owner | Why it matters | What we observed | Owner answer |
|---|---|---|---|---|---|
| OQ-UA-01 | `bucket_session_start_mst_ts` | Which session is chosen, and why is it often later than bucketed_mst_ts (147,663 of 225,901 rows for mobile_ios_contextual_nudges_v2)? | Session-level timing questions. | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-UA-02 | `wam_free_trial_flag` | Exact definition of a WAM free trial and its as-of date. | WAM trial splits. | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-UA-03 | `device_breakpoint_name, market_site, ip_country_name 'Other' / 'Multiple'` | Meaning of 'Other', and the rule that produces 'Multiple' (it occurs on single-visit users). | Segment splits. | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-UA-04 | `customer_tenure_day_segment / domain_ownership_count` | Tenure anchor, as-of date, the boundary at exactly 10 domains, and whether a '2-7 day' bucket exists. | Segment splits. | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-UA-05 | `customer_type / new_existing_customer_name` | Business definition of each customer type (incl. 'Not Evaluated') and the Existing vs New Prospect rule. | Customer segment splits. | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-UA-06 | `visit_count` | Counting window and session definition. | Engagement questions. | — | _TO BE FILLED BY HIVEMIND OWNER_ |
| OQ-UA-07 | `user key for non-web traffic types` | For billing_cycle_id / care_* / resource_id experiments, which column holds the bucketed unit? | User counts for non-web experiments. | shopper_id / visitor_guid only are documented. | _TO BE FILLED BY HIVEMIND OWNER_ |

**Resolved from data (no owner input needed):**

- `allocation_id hash inputs` — Not needed for analysis; dropped from the owner list.

## Sources

- Live column profile: `user_allocation_history.profile.json` (author's local workspace, not committed) (5 experiments, 1,397,235 rows).
- Verified table doc: `tables/hivemind_mart.user_allocation_history.md` (grain, PK, mandatory filters, control-cohort rule, triggered-population and dedup traps).
- Hivemind MCP enums: Hivemind MCP tool input schemas (trafficType, customerReportFilter).
- MCP payloads: saved MCP payloads (author's local workspace) (config trafficType / baselineCohortId / scorecardTemplate / cohorts / status / triggerEvent / rolloutDate; analysis healthMetric.summary.sampleCount / startTime / endTime / crossBucketCount; 'Analysis skipped' warnings).
- Other profiles for cross-table values: `experiment_metadata_snap`, `analysis_result_cln`, `experiment_order` profile.json in the same folder.
- Use-cases: `experimentation-hm-did-it-work.md`, `experimentation-hm-whats-it-worth.md` (bucketed users as per-user denominator; doc #521 iGCR uses all bucketed cohorts).
- Read-only Redshift SQL, 2026-11-13, on `hivemind_mart.user_allocation_history` and `hivemind_mart.experiment_metadata_snap`, one experiment per statement with `bucketed_mst_date` bounds (per-cohort counts, null patterns, timestamp relationships, date = DATE(ts) check).
