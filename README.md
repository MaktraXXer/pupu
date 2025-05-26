/* ----------------------------------------------------------
   0. Список «месячных» дат: от 31-дек-2022 до 31-мар-2025
   (при желании замените верхнюю границу @EndDate)
---------------------------------------------------------- */
DECLARE @StartDate date = '2022-12-31';
DECLARE @EndDate   date = '2025-03-31';

;WITH MonthEnds AS (
    SELECT EOMONTH(@StartDate) AS dt_Rep
    UNION ALL
    SELECT EOMONTH(DATEADD(month, 1, dt_Rep))
    FROM   MonthEnds
    WHERE  dt_Rep < @EndDate
)
/* ----------------------------------------------------------
   1. Фактические строки баланса, только на даты-концевики
---------------------------------------------------------- */
, cte AS (
    SELECT
        bra.dt_Rep,
        bra.SECTION_NAME,
        bra.TSEGMENTNAME,
        bra.PROD_NAME_RES,
        bra.con_id,
        bra.OUT_RUB
    FROM   ALM.ALM.Balance_Rest_All bra  WITH (NOLOCK)
    JOIN   MonthEnds me ON me.dt_Rep = bra.dt_Rep     -- ← ключевой фильтр
    WHERE  bra.section_name  IN ('Срочные','До востребования','Накопительный счёт')
      AND  bra.cur           = '810'
      AND  bra.od_flag       = 1
      AND  bra.is_floatrate  = 0
      AND  bra.TSEGMENTNAME IN ('ДЧБО','Розничный бизнес')
      AND  bra.BLOCK_NAME    = 'Привлечение ФЛ'
      AND  bra.PROD_NAME_RES IS NOT NULL
      AND  bra.OUT_RUB       IS NOT NULL
)
/* ----------------------------------------------------------
   2. Итоговая агрегация
---------------------------------------------------------- */
SELECT
    dt_Rep,
    SECTION_NAME,
    TSEGMENTNAME,
    PROD_NAME_RES,
    SUM(OUT_RUB)           AS sum_OUT_RUB,
    COUNT(DISTINCT con_id) AS count_CON_ID
FROM cte
GROUP BY
    dt_Rep,
    SECTION_NAME,
    TSEGMENTNAME,
    PROD_NAME_RES
ORDER BY
    dt_Rep,
    SECTION_NAME,
    TSEGMENTNAME,
    PROD_NAME_RES
OPTION (MAXRECURSION 0);   -- рекурсивный CTE без лимита
