SELECT 
    SUM(t.OUT_RUB) AS total_closed_out_rub
FROM alm.ALM.vw_balance_rest_all t WITH (NOLOCK)
LEFT JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] saldo WITH (NOLOCK)
       ON t.con_id = saldo.con_id
      AND '2025-07-29' BETWEEN saldo.DT_FROM AND saldo.DT_TO
      AND ISNULL(saldo.OUT_RUB, 0) > 0
WHERE t.dt_rep = '2025-06-30'
  AND t.section_name = N'Срочные'
  AND t.is_floatrate = 0
  AND t.block_name = N'Привлечение ФЛ'
  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.OUT_RUB IS NOT NULL
  AND saldo.con_id IS NULL
