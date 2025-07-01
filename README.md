/*==========================================================
--  Пример запрос-витрина для контроля расчётов НС
--  15.01.2025‒25.01.2025
==========================================================*/
DECLARE @DateFrom date = '2025-01-15',
        @DateTo   date = '2025-01-25';

SELECT
      t.dt_rep                                  AS [date]          -- дата среза
    , t.con_id                                                  -- договор
    , ISNULL(r.rate, t.rate_con)               AS rate_liq        -- ставка из LIQUIDITY
    , t.out_rub                                AS out_rub         -- объём, ₽
FROM alm.[ALM].[vw_balance_rest_all]            t
LEFT JOIN LIQUIDITY.liq.DepositContract_Rate    r
       ON  r.con_id = t.con_id
       AND CASE WHEN t.dt_open = t.dt_rep       -- если счёт открыт сегодня
                 THEN DATEADD(day,1,t.dt_rep)   -- → берём ставку «на завтра»
                 ELSE t.dt_rep                  -- иначе ставку «на сегодня»
           END BETWEEN r.dt_from AND r.dt_to
WHERE t.dt_rep BETWEEN @DateFrom AND @DateTo
  AND t.section_name = N'Накопительный счёт'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
ORDER BY t.dt_rep, t.con_id;
