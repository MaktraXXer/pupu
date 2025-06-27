/*---------------------------------------------------------------
-- ПАРАМЕТРЫ ПОДБОРА
----------------------------------------------------------------*/
DECLARE @gap_min    decimal(10,4) = 2.0 ;  -- «просадка» 31-го ≥ 2 п.п.
DECLARE @equal_tol  decimal(10,4) = 0.5 ;  -- 30-е и 09-е отличаются ≤ 0.5 п.п.

/*---------------------------------------------------------------
-- ОСНОВНЫЕ ДАТЫ
----------------------------------------------------------------*/
;WITH src AS (  -- забираем три даты
    SELECT con_id,
           dt_rep,
           rate_con,
           out_rub
    FROM   alm.[ALM].[vw_balance_rest_all] WITH (NOLOCK)
    WHERE  dt_rep IN ('20241230','20241231','20250109')
      AND  section_name  = N'Накопительный счёт'
      AND  block_name    = N'Привлечение ФЛ'
      AND  od_flag       = 1
      AND  out_rub      IS NOT NULL
      AND  cur           = '810'
      AND  prod_name_res = N'Накопительный счёт'
),
pvt AS (         -- разворачиваем в колонки
    SELECT con_id,
           MAX(CASE WHEN dt_rep = '20241230' THEN rate_con END) AS rate_30,
           MAX(CASE WHEN dt_rep = '20241231' THEN rate_con END) AS rate_31,
           MAX(CASE WHEN dt_rep = '20250109' THEN rate_con END) AS rate_09,
           MAX(CASE WHEN dt_rep = '20241231' THEN out_rub  END) AS out_31,
           MAX(CASE WHEN dt_rep = '20250109' THEN out_rub  END) AS out_09
    FROM   src
    GROUP BY con_id
),
candidates AS (  -- правила отбора
    SELECT *,
           ABS(rate_31 - rate_09)              AS gap_31_vs_09,   -- величина аномалии
           ABS(rate_30 - rate_09)              AS diff_30_vs_09
    FROM   pvt
    WHERE  rate_30 IS NOT NULL
      AND  rate_31 IS NOT NULL
      AND  rate_09 IS NOT NULL
      AND  out_09  >=  out_31                    -- деньги не ушли
      AND  ABS(rate_31 - rate_09) >= @gap_min    -- 31-е сильно отличается
      AND  ABS(rate_30 - rate_09) <= @equal_tol  -- 30-е ≈ 09-е
)
SELECT
       con_id,
       rate_30       AS rate_20241230,
       rate_31       AS rate_20241231,
       rate_09       AS rate_20250109,
       gap_31_vs_09,                       -- «просадка» 31-го (п.п.)
       out_31        AS out_rub_20241231,
       out_09        AS out_rub_20250109
FROM   candidates
ORDER BY out_09 DESC, gap_31_vs_09 DESC;    -- самый «дорогой» счёт с аномалией вверху
