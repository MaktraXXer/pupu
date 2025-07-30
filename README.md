/* Данные: ALM.ALM.VW_Balance_Rest_All  ****************************/
/* Условия:
   – в выборке только BLOCK_NAME='Привлечение ФЛ' и SECTION_NAME='Срочные'
   – для каждого месяца берём САМЫЙ ПОЗДНИЙ dt_rep (конец месяца)
   – учитываем лишь те вклады, у которых DT_OPEN_fact попадает в ТОТ ЖЕ месяц
      (EOMONTH(DT_OPEN_fact) = конец месяца снимка)
   – агрегируем: dt_rep, корзинка срока → SUM(out_rub)
*/

;WITH  /* 1. маленький календарь из 13 месяцев */
months AS (
    SELECT CAST('2023-01-01' AS date) AS month_start
    UNION ALL
    SELECT DATEADD(month, 1, month_start)
    FROM   months
    WHERE  month_start < '2024-01-01'          -- последним будет янв-24
),

/* 2. выбираем «снимок» – самый поздний dt_rep в пределах месяца */
month_latest AS (
    SELECT
        EOMONTH(m.month_start)         AS month_end,   -- 31.01.2023, 28.02.2023…
        latest.dt_rep
    FROM   months m
    CROSS APPLY (
        SELECT TOP (1) dt_rep
        FROM   ALM.ALM.VW_Balance_Rest_All
        WHERE  dt_rep BETWEEN m.month_start AND EOMONTH(m.month_start)
          AND  BLOCK_NAME   = 'Привлечение ФЛ'
          AND  SECTION_NAME = 'Срочные'
        ORDER BY dt_rep DESC               -- самый поздний в месяце
    ) latest
    WHERE latest.dt_rep IS NOT NULL        -- если в месяце нет снимка – пропускаем
),

/* 3. основной набор строк по найденным dt_rep */
base AS (
    SELECT
        ml.month_end             AS dt_rep,         -- конец месяца
        /* корзинка срока */
        CASE
            WHEN w.termdays BETWEEN  28 AND  33 THEN  31
            WHEN w.termdays BETWEEN  60 AND  70 THEN  61
            WHEN w.termdays BETWEEN  85 AND 110 THEN  91
            WHEN w.termdays BETWEEN 119 AND 140 THEN 124
            WHEN w.termdays BETWEEN 175 AND 200 THEN 181
            WHEN w.termdays BETWEEN 245 AND 290 THEN 274
            WHEN w.termdays BETWEEN 340 AND 405 THEN 365
            WHEN w.termdays BETWEEN 540 AND 621 THEN 550
            WHEN w.termdays BETWEEN 720 AND 763 THEN 750
            WHEN w.termdays BETWEEN 1090 AND 1140 THEN 1100
            WHEN w.termdays BETWEEN 1450 AND 1475 THEN 1460
            WHEN w.termdays BETWEEN 1795 AND 1830 THEN 1825
            ELSE w.termdays                    -- если срок вне сетки
        END                       AS term_bucket,
        w.out_rub
    FROM   month_latest ml
    JOIN   ALM.ALM.VW_Balance_Rest_All w
           ON w.dt_rep = ml.dt_rep
          AND w.BLOCK_NAME   = 'Привлечение ФЛ'
          AND w.SECTION_NAME = 'Срочные'
    /* учитываем только депозиты, открытые в том же месяце */
    WHERE  EOMONTH(w.DT_OPEN_fact) = ml.month_end
)

/* 4. итоговая агрегация */
SELECT
       dt_rep,                      -- 31.01.2023, 28.02.2023, …, 31.01.2024
       term_bucket  AS [Срок, дн.], -- 31, 61, 91, …
       SUM(out_rub) AS sum_out_rub
FROM   base
GROUP BY dt_rep, term_bucket
ORDER  BY dt_rep, term_bucket;
