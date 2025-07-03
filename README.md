/* ----------- единый подзапрос с корректной ставкой ---------------- */
;WITH t_ext AS (
    SELECT
          t.dt_rep
        , t.dt_open
        , t.out_rub
        , t.con_id

        /* ─── НОВАЯ логика выбора ставки ───────────────────────────
           r.rate  – из LIQUIDITY.liq.DepositContract_Rate
           t.rate_con – ставка из баланса
           
           1) r < 0,  t > 0  → берём t.rate_con      (положит.)
           2) r < 0,  t < 0  → обе <0: max(|r|,|t|) (положит.)
           3) r ≥0,  t ≥0   → обычное правило: r.rate
           4) r > 0,  t < 0 → берём r.rate          (положит.)
           5) r IS NULL      → t.rate_con (как раньше)
        ---------------------------------------------------------------- */
        , CASE
              WHEN r.rate IS NULL                      THEN t.rate_con          -- r нет
              WHEN r.rate < 0  AND t.rate_con > 0      THEN t.rate_con          -- (1)
              WHEN r.rate < 0  AND t.rate_con < 0      THEN                    -- (2)
                   CASE WHEN ABS(r.rate) >= ABS(t.rate_con)
                        THEN ABS(r.rate) ELSE ABS(t.rate_con) END
              WHEN r.rate >= 0 AND t.rate_con >= 0     THEN r.rate             -- (3)
              WHEN r.rate > 0  AND t.rate_con < 0      THEN r.rate             -- (4)
              ELSE r.rate                                                     -- fallback
          END                                         AS rate_use
    FROM alm.[ALM].[vw_balance_rest_all] t
    LEFT JOIN LIQUIDITY.liq.DepositContract_Rate r
           ON  r.con_id = t.con_id
           AND CASE WHEN t.dt_open = t.dt_rep
                    THEN DATEADD(day,1,t.dt_rep)
                    ELSE t.dt_rep END
              BETWEEN r.dt_from AND r.dt_to
    WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
      AND t.section_name = N'Накопительный счёт'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
)
