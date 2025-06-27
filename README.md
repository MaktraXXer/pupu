;WITH base AS (                       -- отбираем только нужные даты
    SELECT con_id, dt_rep, rate_con
    FROM   alm.[ALM].[vw_balance_rest_all] WITH (NOLOCK)
    WHERE  dt_rep IN ('2024-12-30','2024-12-31')
      AND  section_name  = N'Накопительный счёт'
      AND  block_name    = N'Привлечение ФЛ'
      AND  od_flag       = 1
      AND  out_rub      IS NOT NULL
      AND  cur           = '810'
      AND  prod_name_res = N'Накопительный счёт'
)
SELECT
       con_id,
       MAX(CASE WHEN dt_rep = '2024-12-30' THEN rate_con END) AS rate_20241230,
       MAX(CASE WHEN dt_rep = '2024-12-31' THEN rate_con END) AS rate_20241231
FROM   base
GROUP BY con_id
HAVING MAX(CASE WHEN dt_rep = '2024-12-30' THEN rate_con END) <>
       MAX(CASE WHEN dt_rep = '2024-12-31' THEN rate_con END);
