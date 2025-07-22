/* =================================================================
   СКРИПТ-«ВСЁ В ОДНОМ»  (исправленный)
=================================================================*/

------------------------------------------------------------------
-- 0. Сносим temp-таблицы, если остались с прошлого запуска
------------------------------------------------------------------
IF OBJECT_ID('tempdb..#seg_shift') IS NOT NULL        DROP TABLE #seg_shift;
IF OBJECT_ID('tempdb..#seg_shift_stats') IS NOT NULL  DROP TABLE #seg_shift_stats;
GO   -- ← новый батч, чтобы дальше можно было начинать цепочку WITH

------------------------------------------------------------------
-- 1. ПАРАМЕТРЫ
------------------------------------------------------------------
DECLARE @dvs_from      date = N'2025-03-18';
DECLARE @dvs_to        date = N'2025-03-30';
DECLARE @period_from   date = N'2025-03-01';
DECLARE @period_to     date = N'2025-04-01';   -- включительно
GO

/* -----------------------------------------------------------------
   2-4. CTE-цепочка  (НЕ ставим ничего перед ней!)
-----------------------------------------------------------------*/
;WITH -------------------------------------------------------------  -- leading «;» на всякий случай
dvs_new AS (                     -- договор впервые «новый» в ДВС
    SELECT DISTINCT con_id
    FROM alm.ALM.vw_balance_rest_all
    WHERE dt_rep BETWEEN @dvs_from AND @dvs_to
      AND section_name = N'До востребования'
      AND dt_open      = dt_rep
      AND block_name   = N'Привлечение ФЛ'
      AND od_flag      = 1
      AND cur          = '810'
),
base AS (                        -- все записи март-апрель по ним
    SELECT  t.con_id, t.dt_rep, t.section_name,
            t.out_rub, t.rate_con
    FROM alm.ALM.vw_balance_rest_all AS t
    JOIN dvs_new d ON d.con_id = t.con_id
    WHERE t.dt_rep BETWEEN @period_from AND @period_to
      AND t.block_name = N'Привлечение ФЛ'
      AND t.od_flag    = 1
      AND t.cur        = '810'
),
runs AS (                        -- нумеруем блоки одинаковых сегментов
    SELECT b.*,
           CASE WHEN LAG(section_name) OVER (PARTITION BY con_id ORDER BY dt_rep)
                     = section_name THEN 0 ELSE 1 END                         AS seg_flip,
           SUM(CASE WHEN LAG(section_name) OVER (PARTITION BY con_id ORDER BY dt_rep)
                          = section_name THEN 0 ELSE 1 END)
           OVER (PARTITION BY con_id ORDER BY dt_rep
                 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)            AS run_id
    FROM base b
),
intervals AS (                   -- превращаем в интервалы
    SELECT con_id, run_id,
           MIN(dt_rep) AS run_from,
           MAX(dt_rep) AS run_to,
           MIN(section_name) AS section_name
    FROM runs
    GROUP BY con_id, run_id
),
pattern AS (                     -- ищем НС → ДВС → НС,
    SELECT cur.con_id,
           prev.section_name AS segment_before,
           cur.section_name  AS segment_changed,
           cur.run_from      AS changed_from,
           cur.run_to        AS changed_to,
           next.run_from     AS restored_from,
           next.run_to       AS restored_to,
           next.section_name AS segment_after
    FROM intervals cur
    JOIN intervals prev  ON prev.con_id = cur.con_id AND prev.run_id = cur.run_id - 1
    JOIN intervals next  ON next.con_id = cur.con_id AND next.run_id = cur.run_id + 1
    WHERE prev.section_name  = N'Накопительный счёт'
      AND cur.section_name   = N'До востребования'
      AND next.section_name  = N'Накопительный счёт'
      AND cur.run_from <= @dvs_to     -- середина пересекает 18-30 марта
      AND cur.run_to   >= @dvs_from
)
/* -----------------------------------------------------------------
   5. Сразу же ИСПОЛЬЗУЕМ последний CTE → создаём #seg_shift
-----------------------------------------------------------------*/
SELECT
    con_id,
    segment_before,
    segment_changed,
    changed_from,
    changed_to,
    segment_after,
    restored_from,
    restored_to
INTO #seg_shift
FROM pattern;
GO   -- закончился батч с CTE; можем писать что угодно

------------------------------------------------------------------
-- 6. #seg_shift_stats – вклад «ошибочного» отрезка в портфель
------------------------------------------------------------------
INSERT INTO #seg_shift_stats
SELECT
    t.dt_rep,
    SUM(t.out_rub)                                           AS out_rub_total,
    SUM(t.rate_con * t.out_rub) / NULLIF(SUM(t.out_rub),0)   AS rate_avg
FROM alm.ALM.vw_balance_rest_all t
JOIN #seg_shift s
      ON s.con_id = t.con_id
     AND t.dt_rep BETWEEN s.changed_from AND s.changed_to
WHERE t.section_name = N'До востребования'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
GROUP BY t.dt_rep;
GO

------------------------------------------------------------------
-- 7. Итоговый вывод по con_id (как раньше)
------------------------------------------------------------------
SELECT
    s.con_id,
    s.segment_before,
    s.segment_changed,
    s.changed_from,
    s.changed_to,
    s.restored_from,
    s.segment_after,
    b.section_name  AS segment_on_1apr,
    b.out_rub       AS out_rub_on_1apr
FROM #seg_shift           AS s
LEFT JOIN alm.ALM.vw_balance_rest_all b
       ON b.con_id = s.con_id
      AND b.dt_rep = N'2025-04-01'
ORDER BY s.con_id;

-- при необходимости статистика портфеля:
-- SELECT * FROM #seg_shift_stats ORDER BY dt_rep;
