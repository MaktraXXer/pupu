USE [ALM_TEST]
GO
/* если витрина уже была — удаляем */
IF OBJECT_ID('[WORK].[VW_FL_DEPO_TERM_AGGR_MEND]', 'V') IS NOT NULL
    DROP VIEW [WORK].[VW_FL_DEPO_TERM_AGGR_MEND];
GO
SET ANSI_NULLS ON
SET QUOTED_IDENTIFIER ON
GO
CREATE VIEW [WORK].[VW_FL_DEPO_TERM_AGGR_MEND] AS
/* ───────── 1. календарь из 18 месяцев ───────── */
WITH months AS (
    SELECT CAST('2024-01-01' AS date) AS month_start
    UNION ALL
    SELECT DATEADD(month, 1, month_start)
    FROM   months
    WHERE  month_start < '2025-06-01'          -- июнь-25 — последний
),
/* ───────── 2. для каждого месяца ищем «снимок» ───────── */
month_latest AS (
    SELECT
        EOMONTH(month_start)             AS month_end,        -- 31.01.2024 …
        latest.dt_rep
    FROM   months
    CROSS APPLY (
        SELECT TOP (1) dt_rep                                 -- быстрый seek
        FROM   WORK.VW_FL_DEPO_TERM_WEEK
        WHERE  dt_rep <= EOMONTH(month_start)
          AND  BLOCK_NAME   = 'Привлечение ФЛ'
          AND  SECTION_NAME = 'Срочные'
        ORDER BY dt_rep DESC                                  -- «самый поздний»
    ) latest
    WHERE  latest.dt_rep IS NOT NULL                          -- на случай пустых месяцев
),
/* ───────── 3. агрегируем сумму сразу же ───────── */
aggr AS (
    SELECT
        ml.month_end                    AS dt_rep,            -- 31.01.2024 …
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
        END                               AS term_bucket,
        w.out_rub
    FROM   month_latest           ml
    JOIN   WORK.VW_FL_DEPO_TERM_WEEK w
           ON w.dt_rep = ml.dt_rep
          AND w.BLOCK_NAME   = 'Привлечение ФЛ'
          AND w.SECTION_NAME = 'Срочные'
    WHERE  EOMONTH(TRY_CONVERT(date, w.generation, 23)) = ml.month_end
)
/* ───────── 4. финальный результат ───────── */
SELECT
       dt_rep,                       -- конец месяца
       term_bucket AS [Срок, дн.],   -- нормированный срок
       SUM(out_rub) AS sum_out_rub   -- сумма
FROM   aggr
GROUP BY dt_rep, term_bucket;
GO
