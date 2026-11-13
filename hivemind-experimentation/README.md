# Hivemind experimentation context: 2 use cases (proposal for review)

This folder holds GD Data Context files for two Hivemind experimentation use cases. It is for leader review before it is proposed to the GD Data Context bounded-context team in `gdcorp-dna/gd-data-context`. **Nothing here is live**: no agent reads this repo.

## The two use cases

| Use case | Questions it answers | File |
|---|---|---|
| **Did it work?** | Did experiment X win? Was it still significant on the last report before rollout, and did the result flicker? Give me the full readout. | [use-cases/experimentation-hm-did-it-work.md](use-cases/experimentation-hm-did-it-work.md) |
| **What is it worth?** | What governed iGCR did X deliver, and on which metric? What is the lift worth per user and per day? Which product lines explain the GCR gap? | [use-cases/experimentation-hm-whats-it-worth.md](use-cases/experimentation-hm-whats-it-worth.md) |

Each file routes a question to a path (A, B or C) and gives the SQL, the interpretation rules, the gotchas and an "Out of scope (decline)" list.

## What's in the folder

| Folder | Contents | Role |
|---|---|---|
| `use-cases/` | 2 use-case files | How to answer each type of question |
| `tables/` | 4 `hivemind_mart` table files: `experiment_metadata_snap`, `analysis_result_cln`, `experiment_order`, `user_allocation_history` | What the columns mean, the grain, the join keys and the traps |
| `data-dictionary/` | A full-depth dictionary per table, plus `OWNER_INPUT_NEEDED.md` | Reference layer: every column (200/200), every value of the 93 enum columns, and a Verified / Inferred / Unknown label on each column |
| `registration/` | Proposed entries for `index/experimentation.yaml` and `vocabulary/intents/experimentation.yaml` | Lets agents route to the two use cases once they go live |

## How it was tested

All tests were blind. An agent could read only the files under test and run read-only Redshift SQL. A separate grader then compared every value against Hivemind MCP, the system of record, on 2026-11-13.

The full list of questions, the verified answer for each and the per-question results are in **[EVAL_QUESTIONS.md](EVAL_QUESTIONS.md)**.

| Test | Result |
|---|---|
| 24 golden questions for the 2 use cases (12 each, including 6 that the agent should decline), using the use-case and table files | **24 / 24 match Hivemind MCP** |
| The same 24 questions using the 4 table files only | 10 match, 10 partly match, 4 don't match |
| Table lookup questions (owners, cohorts, lift, SRM, users per cohort, raw GCR), using the table files only | 9 / 9 match |
| 10 column and value meaning questions: table files only vs table files plus the data dictionary | 3 / 10 → 9 / 10 |

**Why both file types are needed.** With the table files only, almost every number still matched MCP. What went wrong was the method:
- which report to headline;
- how to read a Bayesian experiment;
- when to decline;
- giving both the pair-basis and the all-cohort $ for 3+ arms;
- splitting product-line GCR by term.

Those rules live in the use-case files. The table files say what the data means; the use-case files say how to answer a type of question.

## Findings that affect existing docs

- **`rollout_mst_ts` holds UTC, not MST.** It equals `get_experiment_by_id.rolloutDate` exactly. Of A/B rollouts, 37% are exactly 00:00:00 UTC, which means a date-only rollout at 17:00 MST the day before. Converting with −7h changes which report counts as "the last report before rollout".
- **Find gaps from run times, not `report_date` labels.** Runs that start after midnight carry the previous day's label. A gap is runs more than 36 hours apart.
- **`experiment_order` partitions are cumulative.** Read one partition, the latest. Summing across partitions double-counts.
- **`annualization_value` is the governed iGCR with the discount already applied.** Do not apply the surface discount again.
- **Golden set errata:**
  - HM-S-1 and HM-S-2 anchor on a run that is after rollout once UTC is converted. The true S-1 is not significant at rollout (p 0.33).
  - Commercial OS figures for `recore_abn_gdexp_13936` have drifted since 2026-10-22.

## Owner input needed

43 definitions can't be settled from Redshift, Hivemind MCP, Alation or Confluence. Examples:
- the `margin_gcr_usd_amt` formula;
- what `PRE-POST-STANDALONE` means;
- the `DEM` / `ROW DM` / `ROW EM` codes;
- when Mann-Whitney U is used.

Each is flagged **⚠️ OWNER INPUT NEEDED (ID)** where it appears in the files, and all 43 are collected in [data-dictionary/OWNER_INPUT_NEEDED.md](data-dictionary/OWNER_INPUT_NEEDED.md). Until an owner answers, agents must reply "definition not confirmed by the Hivemind owners" and not guess.

## Questions for reviewers

1. **Overlap with what's live.** `gd-data-context`'s `knowledge_wiki/global/tables/` already has all 4 `hivemind_mart` table files (from PR #194 and later edits, including the dify and domains_aftermarket additions). PR #260 also changes them. Should these versions replace, merge with, or sit beside those files?
2. **Intent overlap.** The existing `scorecard` and `rollout_decision` intents share aliases such as "did the experiment win". Which entry should own them?
3. **Where the dictionary lives.** Should the full data dictionary go into `knowledge_wiki/`, or stay as reference documentation outside the agent's routing?
4. **Ownership.** Who answers the 43 owner questions, and who maintains these files after go-live?

## Path to go-live

1. Leaders review this PR.
2. A PR to `gdcorp-dna/gd-data-context` brings the files to the bounded-context team, who decide on questions 1 to 3.
3. The files move into `knowledge_wiki/global/` (`use-cases/experimentation/` and `tables/`) and the `registration/` entries are added.
4. The 24-question eval is re-run against the merged corpus.
