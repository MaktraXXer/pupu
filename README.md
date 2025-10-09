DECLARE @date_from date = '2025-06-01';
DECLARE @date_to   date = '2025-08-31';  -- включительно по август

;WITH month_ends AS (
    -- Ровно концы месяцев между @date_from и @date_to
    SELECT EOMONTH(@date_from) AS dt_rep
    UNION ALL
    SELECT EOMONTH(DATEADD(month, 1, dt_rep))
    FROM month_ends
    WHERE dt_rep < EOMONTH(@date_to)
),
base AS (
    SELECT
        m.dt_rep,
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
            ELSE w.termdays
        END AS term_bucket,
        w.out_rub
    FROM month_ends m
    JOIN ALM.ALM.VW_Balance_Rest_All w WITH (NOLOCK)
      ON w.dt_rep = m.dt_rep
     AND w.BLOCK_NAME   = N'Привлечение ФЛ'
     AND w.SECTION_NAME = N'Срочные'
    CROSS APPLY (VALUES (
        DATEADD(day, 1, EOMONTH(m.dt_rep, -1)),      -- month_start
        DATEADD(day, 1, m.dt_rep)                     -- next_month_start
    )) b(month_start, next_month_start)
    WHERE w.DT_OPEN_fact >= b.month_start
      AND w.DT_OPEN_fact <  b.next_month_start       -- только открытые в этом месяце
)
SELECT
    dt_rep,                                          -- 2025-06-30, 2025-07-31, 2025-08-31
    term_bucket AS [Срок, дн.],
    SUM(out_rub) AS sum_out_rub
FROM base
GROUP BY dt_rep, term_bucket
ORDER BY dt_rep, term_bucket
OPTION (MAXRECURSION 12);
