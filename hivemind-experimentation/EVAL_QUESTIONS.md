# The 24 test questions

These are the 24 questions used to test this folder: 12 per use case, taken from the Hivemind golden question set. 18 come from the "Did it work" and "What it's worth" tabs. The other 6 are **decline** questions. For those, the right answer is to refuse the part the data can't support, say why, and point to the right source, without making up a number.

The questions cover 10 experiments.

**How each answer was checked.** An agent answered each question blind: it could read only these files and run read-only Redshift SQL. A separate grader then compared every number in the answer with Hivemind MCP, the system of record, on 2026-11-13. Where MCP has no field for a value (governed iGCR, product-line cells), the grader used the golden expected answer instead.

**Columns**
- **Verified answer:** the key values in the answer that matched MCP. Times are MST.
- **All files:** result with the use-case files and table files (the version in this PR).
- **Tables only:** result with the 4 table files alone.

Results are ✅ match, 🟡 partial match, ❌ mismatch.

The path codes map to the use-case files:
- **V** verdict, **S** significance before rollout, **R** readout (`experimentation-hm-did-it-work.md`)
- **IL** iGCR lookup, **DD** daily dollars, **PL** product-line breakdown (`experimentation-hm-whats-it-worth.md`)

## Did it work? (12)

| ID | Question | Verified answer (key values) | All files | Tables only |
|---|---|---|---|---|
| HM-V-1 | Did mobile_ios_contextual_nudges_v2 win? What's the lift on the decision metric and is it significant? | Declared win. `Notification_opt-in_rate_iOS` 9.68% → 9.90%, **+2.3%**, p 0.077, significant at alpha 0.1. The latest report (2026-10-13) is a post-rollout rerun, so its SRM check is not reliable. | ✅ | 🟡 |
| HM-V-2 | Did cart_consolidated_billing_long_form_v2 win? Give me the lift and significance for each variant. | Not a win (declared inconclusive). variant1 **+0.4%**, variant2 **−0.6%**, both p 0.532, not significant (alpha 0.1). No guardrails configured. | ✅ | ✅ |
| HM-V-3 | ISC_Legacy_Vs_AWS_DoNoHarm is marked a win — was the lift significant? | **No.** Conversion +0.1% and gcr_amt +0.2%, both p 0.320. Declared a win because it is a do-no-harm test, and the answer flags the gap between the declared result and the stats. | ✅ | ✅ |
| HM-S-1 | agi_plg_email_inline_upgrade_erp was called a loss — was the loss still significant on the last report before rollout, or did it flicker? | **Not significant** at rollout. The last pre-rollout run (2026-10-06) was −1.6%, p 0.330, alpha 0.2. It was significant only on 10-03 and 10-04, so it flickered. *The golden answer is wrong here: it treated the UTC rollout time as MST.* | ✅ | ✅ |
| HM-S-2 | MKT_WA_1249_Msg-Industry-Domain-Match-CM-24029 is a declared win. Was the result still significant right before rollout? | **No.** The 2026-10-06 run was +1.0%, p 0.366, alpha 0.2. It was last significant on 08-13. | ✅ | 🟡 |
| HM-S-3 | Was the mobile_ios_contextual_nudges_v2 win still significant on the last report before rollout, and did it flicker? | **Significant.** The 2026-10-07 run was +3.1%, p 0.025. It **flickered**: Hivemind MCP has a 09-29 run that was not significant (p 0.133), and Redshift is missing it. The answer has to spot the 48-hour gap between runs. | ✅ | ❌ |
| HM-R-1 | Give me the full readout for MKT_EM_177_Failed-Billing-Manual-D28_CM-24927: what was tested, who owns it, when it ran, and every scorecard metric. | A/B, BU Customer & Site, team Retention, scorecard cm-renewals-audience-recording, alpha 0.1, 5 cohorts (control + testa–testd). **Inconclusive**: testc significantly worse (−21.6%, p 0.045), other arms not significant, no guardrail significant. Read from the 2026-10-06 pre-rollout run. | ✅ | 🟡 |
| HM-R-2 | Full readout for mobile_ios_contextual_nudges_v2 please — hypothesis, owners, dates and all metrics. | AGI / Nexus, 2 owners, full hypothesis. Ran 2026-09-23 → 10-07 (14.8 days), rollout 10-08 17:00. Win, Gold badge. All scorecard metrics come from the latest report (10-13). | ✅ | 🟡 |
| HM-R-3 | What was ISC_Legacy_Vs_AWS_DoNoHarm, who ran it, how long did it run, and what did the scorecard show? | Do-no-harm migration from ISC legacy to AWS. BU CTO, team Payments. Ran 69.7 days, rollout 2026-10-06 01:37. Conversion +0.1% and gcr_amt +0.2%, both p 0.320, so no harm. | ✅ | ✅ |
| HM-V-4 *(decline)* | Should we roll out variant1 of cart_consolidated_billing_long_form_v2, and did it hurt any guardrails? | **Declines the rollout call**, which is the owner's decision. Gives the facts: inconclusive, variant1 +0.4% p 0.532, and **no guardrails were configured**. | ✅ | ❌ |
| HM-S-4 *(decline)* | Did the mobile_ios_contextual_nudges_v2 lift hold up after we rolled it out? | **Declines.** Post-rollout reports are reruns of the experiment window, not proof the lift lasted. Redirects to Commercial OS post-rollout metrics and Finance. | ✅ | ❌ |
| HM-R-4 *(decline)* | In the MKT_EM_177_Failed-Billing-Manual-D28_CM-24927 readout, how much money did it make, and break the results down by team? | **Declines both parts.** There is no governed iGCR, and the agent won't turn lifts into dollars. The team field is a single free-text label, not a breakdown. | ✅ | ✅ |

## What is it worth? (12)

| ID | Question | Verified answer (key values) | All files | Tables only |
|---|---|---|---|---|
| HM-IL-1 | What annualized iGCR did recore_abn_gdexp_13936 deliver, on which metric, and is that overall or product-level? | **$1,790,751/yr**, site new purchase iGCR, on `new_purchase_gcr_amt`, an **overall** metric. The discount is already applied. Commercial OS "annualized" is a different number. | ✅ | ✅ |
| HM-IL-2 | productivity_email_users_page shows iGCR — what was annualized and is it overall? | **$355,164/yr**, on `new_o365_addons_purchase_gcr_amt`, which is **product-level** and a guardrail metric. The test has no decision metric. | ✅ | ✅ |
| HM-IL-3 | Why does airo-app-builder-multi-tier-credit-pack have no iGCR even though it was a win? | It was never annualized. The decision metric `airo_builder_gcr` was +2.0%, p 0.170, **not significant**, so nothing qualified. No reasons or $ invented. | ✅ | ✅ |
| HM-DD-1 | The GCR lift on recore_abn_gdexp_13936 is significant — how many dollars per user and per day is it worth? | **+$0.469/user** (p 0.015), 190,324 users over 7.41 days, so **~$12,044/day** undiscounted. The governed iGCR is ~$4,906/day; the answer explains the gap (discount and seasonality). | ✅ | 🟡 |
| HM-DD-2 | What is the daily dollar value of the email users page test (productivity_email_users_page)? | **+$0.329/user**. **$1,428/day** on a pair basis (treatment vs control) and $2,149/day across all cohorts. This test has 3 arms, so both are needed. Governed: $973/day. | ✅ | 🟡 |
| HM-DD-3 | serp_precheck_m365_mod_and_non_mod_1col_single_only — what is the GCR lift worth per day? | **Bayesian** test: +$1.302/user, P(best) 0.995. **$6,889/day** pair basis, $10,368/day all cohorts. Governed: $4,483/day. | ✅ | ❌ |
| HM-PL-1 | Where did the GCR difference in recore_abn_gdexp_13936 come from — which product lines, new vs renewal? | Raw total **+$0.307/user**. Domain registration (new, yearly) **+$0.390** is more than the whole gap. M365 renewal −$0.156. Virtual Hosting renewal +$0.131 is on only 5 vs 3 orders, so it is noise. Raw, with no significance test. | ✅ | 🟡 |
| HM-PL-2 | For serp_precheck_m365_mod_and_non_mod_1col_single_only, which products explain the GCR gap for each treatment? | T1 **+$0.69**, top driver domain registration (new) +$0.61. T2 **+$0.67**, top driver domain renewal −$0.39 (a drag). One partition, raw. | ✅ | 🟡 |
| HM-PL-3 | Break down the GCR difference by product line and new vs renewal for productivity_email_users_page. | T2 **+$3.71**: M365 renewal (yearly) +$3.36, which is **~91% of the gap**. Caution: renewals come from customers who happened to renew while bucketed, so don't credit them to the page change. | ✅ | 🟡 |
| HM-IL-4 *(decline)* | How much of recore_abn_gdexp_13936's iGCR has actually landed since we rolled it out? | **Declines.** Governed iGCR is a forward 12-month estimate, not realised revenue, so it can't be prorated. Redirects to Commercial OS post-rollout metrics and Finance. | ✅ | 🟡 |
| HM-DD-4 *(decline)* | recore_abn_gdexp_13936 lifted new purchase conversion — what is that conversion lift worth in dollars per day? | **Declines.** Conversion is a rate, so lift × users gives extra buyers, not dollars. Offers the GCR metric instead (~$12K/day). | ✅ | ✅ |
| HM-PL-4 *(decline)* | Which product line's lift in recore_abn_gdexp_13936 is statistically significant? | **Declines.** The product-line breakdown is raw and has no significance test. Redirects to the product-specific scorecard metrics, which have their own p-values. | ✅ | ✅ |

## Totals

| | Match | Partial | Mismatch |
|---|---|---|---|
| **All files** (use-case + table files, this PR) | **24** | 0 | 0 |
| **Tables only** | 10 | 10 | 4 |

The tables-only answers got almost every number right. They failed on method: which report to headline, how to read a Bayesian test, when to decline, giving both the pair-basis and all-cohort $ for 3+ arms, and splitting product-line GCR by term. Those rules are in the use-case files.

## Golden set corrections found during testing

- **HM-S-1 and HM-S-2.** The expected answers anchor on a run that is actually after rollout, because the golden set read the UTC `rollout_mst_ts` as MST. The values above are the corrected ones.
- **Rollout times in R-1, R-2, R-3, S-1, S-2 and S-3** are labelled MST in the golden set but are UTC. MST is 7 hours earlier.
- **HM-DD-1.** The Commercial OS figures for `recore_abn_gdexp_13936` have changed since the golden set was written on 2026-10-22.
