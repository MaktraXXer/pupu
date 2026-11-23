
USE ALM;
SET NOCOUNT ON;

SELECT
    t.dt_rep,
    t.rate_con,
    t.rate_con_src,

    COUNT(DISTINCT t.CON_ID) AS contracts_count,
    COUNT(DISTINCT t.CLI_ID) AS clients_count,

    SUM(t.out_rub) / 1000000000.0 AS out_bln

FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

WHERE t.dt_rep >= '2026-11-15'
  AND t.dt_rep < '2026-11-22'
  AND t.dt_open >= '2026-11-15'
  AND t.dt_open <= t.dt_rep

  AND t.block_name = N'Привлечение ФЛ'
  AND t.section_name = N'Накопительный счёт'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.out_rub > 0

GROUP BY
    t.dt_rep,
    t.rate_con,
    t.rate_con_src

ORDER BY
    t.dt_rep,
    t.rate_con_src,
    t.rate_con

OPTION (MAXDOP 1, RECOMPILE);
