DECLARE @ChkDate date = '2025-07-10';        -- дата, которую проверяем

;WITH t_ext AS (      -- копируем логику из процедуры
    SELECT
          t.con_id
        , t.cli_id                    -- если нужен клиент
        , t.out_rub
        , t.rate_con                  AS rate_balance      -- ставка из баланса (t)
        , r.rate                      AS rate_liq          -- ставка из DepositContract_Rate (r)
        , CASE                        -- та же CASE-логика выбора
              WHEN r.rate IS NULL                     THEN t.rate_con              -- (5)
              WHEN r.rate < 0  AND t.rate_con > 0     THEN t.rate_con              -- (1)
              WHEN r.rate < 0  AND t.rate_con < 0     THEN
                   CASE WHEN ABS(r.rate) >= ABS(t.rate_con)
                        THEN ABS(r.rate) ELSE ABS(t.rate_con) END                  -- (2)
              WHEN r.rate >= 0 AND t.rate_con >= 0    THEN r.rate                  -- (3)
              WHEN r.rate > 0  AND t.rate_con < 0     THEN r.rate                  -- (4)
              ELSE r.rate
          END                              AS rate_use
    FROM alm.ALM.vw_balance_rest_all      t
    LEFT JOIN LIQUIDITY.liq.DepositContract_Rate r
           ON  r.con_id = t.con_id
           AND CASE WHEN t.dt_open = t.dt_rep
                    THEN DATEADD(day,1,t.dt_rep)
                    ELSE t.dt_rep END
               BETWEEN r.dt_from AND r.dt_to
    WHERE t.dt_rep  = @ChkDate             -- ← день отчёта
      AND t.dt_open = @ChkDate             -- ← «новые» (открылись в тот же день)
      AND t.section_name = N'Накопительный счёт'
      AND t.block_name   = N'Привлечение ФЛ'
      AND t.od_flag      = 1
      AND t.cur          = '810'
)
SELECT  con_id,
        cli_id,
        rate_balance,          -- ставка из баланса (t.rate_con)
        rate_liq,              -- ставка из DepositContract_Rate
        rate_use,              -- какая реально пойдёт в расчёт
        out_rub
FROM    t_ext
ORDER BY con_id;
