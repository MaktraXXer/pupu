WITH month_latest AS (
    SELECT
        EOMONTH(dt_rep) AS month_end,
        MAX(dt_rep)     AS dt_rep
    FROM ALM.ALM.VW_Balance_Rest_All WITH (NOLOCK)
    WHERE BLOCK_NAME   = N'Привлечение ФЛ'
      AND SECTION_NAME = N'Срочные'
      AND dt_rep BETWEEN '2025-06-01' AND '2025-09-30'
    GROUP BY EOMONTH(dt_rep)
),
base AS (
    SELECT
        ml.month_end             AS dt_rep,
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
    FROM month_latest ml
    JOIN ALM.ALM.VW_Balance_Rest_All w WITH (NOLOCK)
         ON w.dt_rep = ml.dt_rep
        AND w.BLOCK_NAME   = N'Привлечение ФЛ'
        AND w.SECTION_NAME = N'Срочные'
    WHERE EOMONTH(w.DT_OPEN_fact) = ml.month_end   -- только открытые в этом месяце
)
SELECT
       dt_rep,
       term_bucket AS [Срок, дн.],
       SUM(out_rub) AS sum_out_rub
FROM base
GROUP BY dt_rep, term_bucket
ORDER BY dt_rep, term_bucket;
