WITH base AS (
    SELECT
          t.cli_id,
          t.con_id,
          t.dt_open,
          t.out_rub,
          t.rate_con           AS rate_balance,
          r.rate               AS rate_liq
    FROM   alm.ALM.vw_balance_rest_all AS t
    LEFT   JOIN LIQUIDITY.liq.DepositContract_Rate r
           ON  r.con_id = t.con_id
           AND CASE WHEN t.dt_open = t.dt_rep
                    THEN DATEADD(day,1,t.dt_rep)
                    ELSE t.dt_rep
               END BETWEEN r.dt_from AND r.dt_to
    WHERE  t.dt_rep       = '2025-07-16'
      AND  t.section_name = N'Накопительный счёт'
      AND  t.block_name   = N'Привлечение ФЛ'
      AND  t.od_flag      = 1
      AND  t.is_floatrate = 0
      AND  t.cur          = '810'
      AND  t.out_rub     >= 0
      AND  t.out_rub IS NOT NULL
),
flag_july_ns AS (
    SELECT DISTINCT cli_id, 1 AS has_july_ns
    FROM   alm.ALM.vw_balance_rest_all
    WHERE  section_name = N'Накопительный счёт'
      AND  block_name   = N'Привлечение ФЛ'
      AND  cur          = '810'
      AND  dt_open BETWEEN '2025-07-01' AND '2025-07-31'
)

SELECT 
    b.cli_id,
    b.con_id,
    b.dt_open,
    b.rate_balance,
    b.rate_liq,
    b.out_rub,
    ISNULL(f.has_july_ns, 0) AS has_july_ns
FROM base b
LEFT JOIN flag_july_ns f
       ON b.cli_id = f.cli_id
ORDER BY b.cli_id, b.dt_open;
