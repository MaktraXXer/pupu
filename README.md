/*---------------------------------------------------------------
-- ПОД НАСТРОЙКУ: "значительно ниже" и "примерно равно"
----------------------------------------------------------------*/
DECLARE @diff_min     decimal(10,4) = 2.0 ;  -- разница ≥ 2 п.п.   (09-янв минус 31-дек)
DECLARE @equal_toller decimal(10,4) = 0.5 ;  -- |30-дек - 09-янв| ≤ 0.5 п.п.

/*---------------------------------------------------------------
-- ОСНОВНОЙ ЗАПРОС
----------------------------------------------------------------*/
;WITH src AS (  -- только нужные даты и фильтры
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
pivoted AS (   -- разворачиваем даты в колонки
    SELECT con_id,
           MAX(CASE WHEN dt_rep = '20241230' THEN rate_con END) AS rate_20241230,
           MAX(CASE WHEN dt_rep = '20241231' THEN rate_20241231 END) AS rate_20241231,
           MAX(CASE WHEN dt_rep = '20250109' THEN rate_20250109 END) AS rate_20250109,
           MAX(CASE WHEN dt_rep = '20241230' THEN out_rub END)  AS out_20241230,
           MAX(CASE WHEN dt_rep = '20241231' THEN out_rub END)  AS out_20241231
    FROM   src
    GROUP BY con_id
),
candidates AS (  -- применяем правила отбора
    SELECT *,
           rate_20250109 - rate_20241231 AS diff_31_vs_09
    FROM   pivoted
    WHERE  rate_20241231 IS NOT NULL
      AND  rate_20250109 IS NOT NULL
      AND  out_20241231  >= out_20241230          -- баланс не уменьшился
      AND  rate_20250109 - rate_20241231 >= @diff_min   -- 31-дек серьёзно ниже 09-янв
      AND  ABS(rate_20241230 - rate_20250109) <= @equal_toller -- 30-дек ≈ 09-янв
)
SELECT  con_id,
        rate_20241230,
        rate_20241231,
        rate_20250109,
        diff_31_vs_09,          -- на сколько п.п. «просела» 31-е
        out_20241230,
        out_20241231
FROM    candidates
ORDER BY diff_31_vs_09 DESC;     -- наибольший «провал» вверху
