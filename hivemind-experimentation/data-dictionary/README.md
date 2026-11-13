# Hivemind data dictionary (full depth)

Full metadata for the 4 `hivemind_mart` tables. Built 2026-11-13 from a live Redshift profile of every column, Hivemind MCP enum definitions and payloads, and the verified lean table files.

| File | Columns | Enum columns fully listed |
|---|---|---|
| hivemind_mart.experiment_metadata_snap.dictionary.md | 74 / 74 | 30 / 30 |
| hivemind_mart.analysis_result_cln.dictionary.md | 49 / 49 | 17 / 17 |
| hivemind_mart.experiment_order.dictionary.md | 57 / 57 | 33 / 33 |
| hivemind_mart.user_allocation_history.dictionary.md | 20 / 20 | 13 / 13 |

Each column has its meaning, its Hivemind API equivalent, a profile (null %, empty %, distinct values, range), every value with a count and meaning (for columns with 60 or fewer values), traps, and a confidence label: Verified, Inferred or Unknown.

**How to use:** the agent's main files are still `tables/` + `use-cases/` (2 use cases and 4 lean tables). Add this folder as a reference layer for questions about columns or values the lean files don't cover. Definitions that only the Hivemind owners can supply are flagged **⚠️ OWNER INPUT NEEDED (ID)**. The flag appears under the column heading and on the value row, and each file ends with an owner table whose "Owner answer" column reads `_TO BE FILLED BY HIVEMIND OWNER_`. **[OWNER_INPUT_NEEDED.md](OWNER_INPUT_NEEDED.md)** collects all 43 questions in one list to send to the owners. Until an owner answers, agents must reply "definition not confirmed by the Hivemind owners" and not guess.

**Tested:** on 10 questions that depend on these columns, matches with Hivemind MCP went from 3/10 with the lean files only to 9/10 with the dictionary added. Correct value meanings went from 3/10 to 9/10. No meanings were invented in either run.

Profile scopes: experiment_metadata_snap = full table; analysis_result_cln = 259K rows over 150 experiments (Jul–Nov 2026); experiment_order = latest partition of 5 experiments; user_allocation_history = 5 experiments. Counts are a point-in-time snapshot. Re-profile quarterly (profiling script available from the author).
