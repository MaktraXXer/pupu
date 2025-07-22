/* ===============================================================
   ВСЁ-В-ОДНОМ: поиск НС → ДВС → НС + вклад «ошибки» в портфель
=============================================================== */

--------------------------- 0. Чистим temp ----------------------
IF OBJECT_ID('tempdb..#seg_shift')       IS NOT NULL DROP TABLE #seg_shift;
IF OBJECT_ID('tempdb..#seg_shift_stats') IS NOT NULL DROP TABLE #seg_shift_stats;

--------------------------- 1. Параметры ------------------------
DECLARE @dvs_from    date = N'2025-03-18';
DECLARE @dvs_to      date = N'2025-03-30';
DECLARE @period_from date = N'2025-03-01';
DECLARE @period_to   date = N'2025-04-01';   -- включительно

;WITH ----------------------------------------------------------- -- начало цепочки CTE
dvs_new AS (      -- договор впервые «новый» в ДВС-портфеле 18-30 марта
    SELECT DISTINCT con_id
    FROM alm.ALM.vw_balance_rest_all
    WHERE dt_rep BETWEEN @dvs_from AND @dvs_to
      AND section_name = N'До востребования'
      AND dt_open      = dt_rep
      AND block_name   = N'Привлечение ФЛ'
      AND od_flag      = 1
      AND cur          = '810'
),
base AS (         -- все записи за март-апрель по таким con_id
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
lag_mark AS (     -- Шаг A: флаг «изменилась/не изменилась» относительно вчера
    SELECT b.*,
           CASE
               WHEN LAG(section_name) OVER (PARTITION BY con_id ORDER BY dt_rep)
                    = section_name THEN 0 ELSE 1
           END AS seg_flip
    FROM base b
),
runs AS (         -- Шаг B: кумулятивная сумма флагов → ID «куска»
    SELECT  l.*,
            SUM(seg_flip) OVER (PARTITION BY con_id ORDER BY dt_rep
                                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS run_id
    FROM lag_mark l
),
intervals AS (    -- превращаем в интервалы одинаковых сегментов
    SELECT con_id,
           run_id,
           MIN(dt_rep)        AS run_from,
           MAX(dt_rep)        AS run_to,
           MIN(section_name)  AS section_name
    FROM runs
    GROUP BY con_id, run_id
),
pattern AS (      -- ищем НС → ДВС → НС, где «ДВС» пересекает 18-30 марта
    SELECT cur.con_id,
           prev.section_name AS segment_before,   -- Накопительный счёт
           cur.section_name  AS segment_changed,  -- До востребования
           cur.run_from      AS changed_from,
           cur.run_to        AS changed_to,
           next.run_from     AS restored_from,
           next.run_to       AS restored_to,
           next.section_name AS segment_after     -- Накопительный счёт
    FROM intervals cur
    JOIN intervals prev  ON prev.con_id = cur.con_id AND prev.run_id = cur.run_id - 1
    JOIN intervals next  ON next.con_id = cur.con_id AND next.run_id = cur.run_id + 1
    WHERE prev.section_name  = N'Накопительный счёт'
      AND cur.section_name   = N'До востребования'
      AND next.section_name  = N'Накопительный счёт'
      AND cur.run_from <= @dvs_to
      AND cur.run_to   >= @dvs_from
)
/* ---------------- 2. сохраняем результаты проверки ---------------- */
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

/* ---------------- 3. суточный вклад «ошибки» в ДВС-портфель ------- */
SELECT
    t.dt_rep,
    SUM(t.out_rub)                                         AS out_rub_total,
    SUM(t.rate_con * t.out_rub) /
        NULLIF(SUM(t.out_rub),0)                           AS rate_avg
INTO #seg_shift_stats
FROM alm.ALM.vw_balance_rest_all t
JOIN #seg_shift s
      ON s.con_id = t.con_id
     AND t.dt_rep BETWEEN s.changed_from AND s.changed_to
WHERE t.section_name = N'До востребования'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
GROUP BY t.dt_rep;

/* ---------------- 4. итоговый вывод по con_id --------------------- */
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

/* --- при необходимости:   SELECT * FROM #seg_shift_stats ORDER BY dt_rep; --- */



```sql
/* ================================================================
   «БЫСТРО И КОНКРЕТНО»
   — показываем, для каждого договора:
        1) первый за март-апрель срез (дата + сегмент);
        2) дату/сегмент, когда он впервые стал «До востребования»;
        3) дату/сегмент, когда снова появился «Накопительный счёт».
   Основано только на тех con_id, что ВПЕРВЫЕ пришли в ДВС-портфель
   как «новые» между 18-30 марта 2025.
================================================================ */

--------------------------- 0. чистим temp ------------------------
IF OBJECT_ID('tempdb..#base') IS NOT NULL        DROP TABLE #base;
IF OBJECT_ID('tempdb..#seg_change_quick') IS NOT NULL DROP TABLE #seg_change_quick;

--------------------------- 1. параметры --------------------------
DECLARE @dvs_from    date = N'2025-03-18';
DECLARE @dvs_to      date = N'2025-03-30';
DECLARE @period_from date = N'2025-03-01';
DECLARE @period_to   date = N'2025-04-01';      -- включительно

--------------------------- 2. con_id «новые в ДВС» 18-30 марта ---
;WITH dvs_new AS (
    SELECT DISTINCT con_id
    FROM alm.ALM.vw_balance_rest_all
    WHERE dt_rep BETWEEN @dvs_from AND @dvs_to
      AND section_name = N'До востребования'
      AND dt_open      = dt_rep            -- «новый»
      AND block_name   = N'Привлечение ФЛ'
      AND od_flag      = 1
      AND cur          = '810'
)
SELECT                                          -- записи март–апрель
        t.con_id,
        t.dt_rep,
        t.section_name,
        t.out_rub,
        t.rate_con
INTO    #base
FROM    alm.ALM.vw_balance_rest_all AS t
JOIN    dvs_new d  ON d.con_id = t.con_id
WHERE   t.dt_rep BETWEEN @period_from AND @period_to
  AND   t.block_name = N'Привлечение ФЛ'
  AND   t.od_flag    = 1
  AND   t.cur        = '810';

-- ускоряем поиск по con_id/дате
CREATE CLUSTERED INDEX IX_base ON #base(con_id, dt_rep);

--------------------------- 3. «три ключевые точки» ---------------
SELECT
    c.con_id,

    /* --- 1. первый срез (ещё до любых изменений) --- */
    initial.dt_rep           AS dt_initial,
    initial.section_name     AS segment_initial,

    /* --- 2. когда ВПЕРВЫЕ стал ДВС --- */
    first_dvs.dt_rep         AS dt_first_dvs,
    first_dvs.section_name   AS segment_first_dvs,

    /* --- 3. когда ВПЕРВЫЕ после этого вернулся в НС --- */
    first_ns.dt_rep          AS dt_first_ns,
    first_ns.section_name    AS segment_first_ns

INTO #seg_change_quick
FROM   (SELECT DISTINCT con_id FROM #base) AS c

/*  --- первая запись в периоде --- */
CROSS APPLY (
        SELECT TOP (1) dt_rep, section_name
        FROM   #base
        WHERE  con_id = c.con_id
        ORDER  BY dt_rep
) AS initial

/*  --- первое появление ДВС после initial --- */
OUTER APPLY (
        SELECT TOP (1) dt_rep, section_name
        FROM   #base
        WHERE  con_id = c.con_id
          AND  dt_rep > initial.dt_rep
          AND  section_name = N'До востребования'
          AND  section_name <> initial.section_name
        ORDER  BY dt_rep
) AS first_dvs

/*  --- первое возвращение НС после того, как был ДВС --- */
OUTER APPLY (
        SELECT TOP (1) dt_rep, section_name
        FROM   #base
        WHERE  con_id = c.con_id
          AND  first_dvs.dt_rep IS NOT NULL        -- ДВС реально был
          AND  dt_rep > first_dvs.dt_rep
          AND  section_name = N'Накопительный счёт'
        ORDER  BY dt_rep
) AS first_ns;

--------------------------- 4. вывод ------------------------------
SELECT  *
FROM    #seg_change_quick
ORDER BY con_id;
```

### Что делает запрос

* **#base** — все записи за 1 марта – 1 апреля по тем договорам, которые впервые появились в ДВС-портфеле 18-30 марта как «новые».
  Индекс `(con_id, dt_rep)` ускоряет дальнейшие выборки.

* **#seg\_change\_quick** — по каждому `con_id` ровно одна строка с тремя датами:

  1. `dt_initial` / `segment_initial` — какой сегмент был изначально в начале периода.
  2. `dt_first_dvs` / `segment_first_dvs` — первая дата, когда договор стал «До востребования».
  3. `dt_first_ns` / `segment_first_ns` — первая дата, когда после этого снова появился «Накопительный счёт».

Если договор так и не вернулся в НС, колонки `dt_first_ns` и `segment_first_ns` будут **NULL**; если он с самого начала был ДВС, то `dt_first_dvs` совпадёт с `dt_initial`, а `segment_initial = 'До востребования'`.

Временная таблица остаётся доступной в сессии — можно строить дополнительную аналитику или фильтровать выборку по нужным `con_id`.
