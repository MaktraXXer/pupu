/* ============================================================
   === SELECT 1.5 ===
   ПЛАНОВЫЕ ВЫХОДЫ НЕ-МАРКЕТ ВКЛАДОВ
   У ТЕХ ЖЕ КЛИЕНТОВ (24–06 ДЕКАБРЯ)
   ============================================================ */

IF OBJECT_ID('tempdb..#bank_exits') IS NOT NULL DROP TABLE #bank_exits;

SELECT
      t.cli_id,
      t.con_id,
      CAST(t.dt_close AS date) AS dt_close_d,
      t.out_rub
INTO #bank_exits
FROM ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep       = @dt_rep1
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.acc_role     = N'LIAB'
  AND t.cur          = '810'
  AND t.out_rub > 0
  AND t.dt_close > t.dt_rep
  AND CAST(t.dt_close AS date) BETWEEN @d_from AND @d_to
  AND t.cli_id IN (SELECT cli_id FROM #cli_mp_exit)
  -- ИСКЛЮЧАЕМ МАРКЕТПЛЕЙС
  AND NOT EXISTS (
        SELECT 1 FROM @mp m WHERE m.prod_name = t.prod_name
  );

/* ------------------------------------------------------------
   Агрегация по датам закрытия
   ------------------------------------------------------------ */

SELECT
      dt_close_d AS [date],
      COUNT(*)   AS cnt_bank_exits,
      SUM(out_rub) AS sum_bank_exits
FROM #bank_exits
GROUP BY dt_close_d
ORDER BY dt_close_d;
