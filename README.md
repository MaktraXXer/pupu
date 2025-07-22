```sql
/* =================================================================
   СКРИПТ-«ВСЁ В ОДНОМ»
   • Ищем договоры, которые:
        Накопительный счёт → До востребования → Накопительный счёт
        (середина попадает в 18-30 марта 2025 как «новые» ДВС)
   • Сохраняем:
        #seg_shift       – список договоров + даты переключений
        #seg_shift_stats – влияние «ошибочного» ДВС-отрезка на портфель
   • Сразу выводим #seg_shift (по con_id, как просили)
=================================================================*/

------------------------------------------------------------------
-- ПАРАМЕТРЫ
------------------------------------------------------------------
DECLARE @dvs_from      date = N'2025-03-18';   -- окно «ошибочных» ДВС-срезов
DECLARE @dvs_to        date = N'2025-03-30';
DECLARE @period_from   date = N'2025-03-01';   -- анализируем март-апрель
DECLARE @period_to     date = N'2025-04-01';   -- включительно

------------------------------------------------------------------
-- ШАГ 1. con_id, которые 18-30 марта попали в ДВС как «новые»
------------------------------------------------------------------
WITH dvs_new AS (
    SELECT DISTINCT con_id
    FROM alm.ALM.vw_balance_rest_all
    WHERE dt_rep BETWEEN @dvs_from AND @dvs_to
      AND section_name = N'До востребования'
      AND dt_open      = dt_rep            -- «новый» договор
      AND block_name   = N'Привлечение ФЛ'
      AND od_flag      = 1
      AND cur          = '810'
),

------------------------------------------------------------------
-- ШАГ 2. все записи за март-апрель по найденным con_id
------------------------------------------------------------------
base AS (
    SELECT t.con_id,
           t.dt_rep,
           t.section_name,
           t.out_rub,
           t.rate_con
    FROM alm.ALM.vw_balance_rest_all AS t
    JOIN dvs_new d ON d.con_id = t.con_id
    WHERE t.dt_rep BETWEEN @period_from AND @period_to
      AND t.block_name = N'Привлечение ФЛ'
      AND t.od_flag    = 1
      AND t.cur        = '810'
),

------------------------------------------------------------------
-- ШАГ 3. нумеруем «куски» одинаковых сегментов
------------------------------------------------------------------
runs AS (
    SELECT b.*,
           CASE WHEN LAG(section_name) OVER (PARTITION BY con_id ORDER BY dt_rep)
                     = section_name
                THEN 0 ELSE 1 END                                              AS seg_flip,
           SUM(CASE WHEN LAG(section_name) OVER (PARTITION BY con_id ORDER BY dt_rep)
                          = section_name
                       THEN 0 ELSE 1 END)
           OVER (PARTITION BY con_id ORDER BY dt_rep
                 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)             AS run_id
    FROM base b
),

------------------------------------------------------------------
-- ШАГ 4. превращаем «куски» в интервалы
------------------------------------------------------------------
intervals AS (
    SELECT
        con_id,
        run_id,
        MIN(dt_rep)            AS run_from,
        MAX(dt_rep)            AS run_to,
        MIN(section_name)      AS section_name   -- одинаков внутри «куска»
    FROM runs
    GROUP BY con_id, run_id
),

------------------------------------------------------------------
-- ШАГ 5. находим паттерн   НС → ДВС → НС
------------------------------------------------------------------
pattern AS (
    SELECT
        cur.con_id,
        prev.section_name  AS segment_before,      -- Накопительный счёт
        cur.section_name   AS segment_changed,     -- До востребования
        cur.run_from       AS changed_from,
        cur.run_to         AS changed_to,
        next.run_from      AS restored_from,       -- когда вернули НС
        next.run_to        AS restored_to,
        next.section_name  AS segment_after        -- снова Накопительный счёт
    FROM intervals cur
    JOIN intervals prev  ON prev.con_id = cur.con_id 
                        AND prev.run_id = cur.run_id - 1
    JOIN intervals next  ON next.con_id = cur.con_id 
                        AND next.run_id = cur.run_id + 1
    WHERE prev.section_name  = N'Накопительный счёт'
      AND cur.section_name   = N'До востребования'
      AND next.section_name  = N'Накопительный счёт'
      -- середина (ДВС-отрезок) пересекает 18-30 марта
      AND cur.run_from <= @dvs_to 
      AND cur.run_to   >= @dvs_from
)

------------------------------------------------------------------
-- ШАГ 6. сохраняем результаты в temp-таблицы
------------------------------------------------------------------
IF OBJECT_ID('tempdb..#seg_shift') IS NOT NULL DROP TABLE #seg_shift;

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

IF OBJECT_ID('tempdb..#seg_shift_stats') IS NOT NULL DROP TABLE #seg_shift_stats;

SELECT
    t.dt_rep,
    SUM(t.out_rub)                                           AS out_rub_total,
    SUM(t.rate_con * t.out_rub) / NULLIF(SUM(t.out_rub),0)   AS rate_avg
INTO #seg_shift_stats
FROM alm.ALM.vw_balance_rest_all t
JOIN #seg_shift s
      ON s.con_id = t.con_id
     AND t.dt_rep BETWEEN s.changed_from AND s.changed_to
WHERE t.section_name = N'До востребования'       -- как учёл отчёт
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
GROUP BY t.dt_rep;

------------------------------------------------------------------
-- ШАГ 7. вывод по con_id, как раньше (можно фильтровать/сортировать)
------------------------------------------------------------------
SELECT
    con_id,
    segment_before,
    segment_changed,
    changed_from,
    changed_to,
    restored_from,
    segment_after,
    /* на 1 апреля */
    (SELECT section_name FROM base b
      WHERE b.con_id = s.con_id AND b.dt_rep = N'2025-04-01') AS segment_on_1apr,
    (SELECT out_rub FROM base b
      WHERE b.con_id = s.con_id AND b.dt_rep = N'2025-04-01') AS out_rub_on_1apr
FROM #seg_shift AS s
ORDER BY con_id;

-- при необходимости:
-- SELECT * FROM #seg_shift_stats ORDER BY dt_rep;
```

**Что вы получаете**

* `#seg_shift` — таблица-верификация по каждому договору:
  «что было → на что поменяли → когда вернули» + контроль на 1 апреля.
* `#seg_shift_stats` — ежедневный вклад этих договоров в ДВС-портфель
  (остаток + средневзвешенная ставка) только за «ошибочный» период.
  Используйте её для сверки или вычитания из отчётной статистики.

Обе temp-таблицы видны до конца текущей сессии, поэтому можно выполнять дополнительные запросы, строить графики и т. д., не повторяя всю логику.
