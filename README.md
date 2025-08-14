/* --- Финал: считаем отдельно размер пути и клиентов с балансом на датах --- */
SELECT
  initial_state,                  -- 'A0' / 'B0'
  final_state,                    -- 'A1' / 'B1' / 'C1' / 'N1'

  SUM(vol_total_0731) AS vol_total_start,             -- объём (все вклады) на 31.07
  SUM(vol_total_0812) AS vol_total_end,               -- объём (все вклады) на 12.08

  /* 1) Размер множества цепочки (те же cli_id) — для контроля пути */
  COUNT(*) AS path_clients_cnt,

  /* 2) Реальные «клиенты на дате» — только с положительным остатком */
  COUNT(DISTINCT CASE WHEN vol_total_0731 > 0 THEN cli_id END) AS clients_start,  -- ≤ path_clients_cnt
  COUNT(DISTINCT CASE WHEN vol_total_0812 > 0 THEN cli_id END) AS clients_end,    -- ≤ clients_start

  /* Санити-чек: end ≤ start */
  CASE WHEN
    COUNT(DISTINCT CASE WHEN vol_total_0812 > 0 THEN cli_id END)
    <= COUNT(DISTINCT CASE WHEN vol_total_0731 > 0 THEN cli_id END)
  THEN 1 ELSE 0 END AS end_leq_start_check
FROM classified
GROUP BY initial_state, final_state
ORDER BY initial_state, final_state
OPTION (RECOMPILE);
