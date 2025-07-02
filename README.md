/* --- был ли ХОТЬ ОДИН вклад ранее first_dt_main ? --- */
IIF( COALESCE( MAX(CASE WHEN dt_rep < first_dt_main THEN 1 END)
                   OVER (PARTITION BY cli_id), 0) = 1,
     1, 0)   AS had_deposit_before,

/* --- только Main --- */
IIF( COALESCE( SUM(CASE WHEN target_main = 0 THEN 1 END)
                     OVER (PARTITION BY cli_id), 0) = 0,
     1, 0)   AS fu_only_overall,

IIF( COALESCE( SUM(CASE WHEN dt_rep = first_dt_main AND target_main = 0 THEN 1 END)
                     OVER (PARTITION BY cli_id), 0) = 0,
     1, 0)   AS fu_only_at_generation,

/* --- только Aux --- */
IIF( COALESCE( SUM(CASE WHEN target_aux = 0 THEN 1 END)
                     OVER (PARTITION BY cli_id), 0) = 0,
     1, 0)   AS other_only_overall,

IIF( COALESCE( SUM(CASE WHEN dt_rep = first_dt_main AND target_aux = 0 THEN 1 END)
                     OVER (PARTITION BY cli_id), 0) = 0,
     1, 0)   AS other_only_at_generation,

/* --- Main ∪ Aux --- */
IIF( COALESCE( SUM(CASE WHEN target_main = 0 AND target_aux = 0 THEN 1 END)
                     OVER (PARTITION BY cli_id), 0) = 0,
     1, 0)   AS all_only_overall,

IIF( COALESCE( SUM(CASE WHEN dt_rep = first_dt_main
                          AND target_main = 0 AND target_aux = 0 THEN 1 END)
                     OVER (PARTITION BY cli_id), 0) = 0,
     1, 0)   AS all_only_at_generation,

