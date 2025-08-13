;WITH months AS (
    SELECT CAST('2023-01-01' AS date) AS month_start
    UNION ALL
    SELECT DATEADD(month, 1, month_start)
    FROM   months
    WHERE  month_start < '2025-08-01'          -- последним будет Июль 2025
),
month_latest AS (
    SELECT
        EOMONTH(m.month_start) AS month_end,   -- 31.01.2023, 29.02.2024, …
        latest.dt_rep
    FROM months m
    CROSS APPLY (
        SELECT TOP (1) dt_rep
        FROM   ALM.ALM.VW_Balance_Rest_All
        WHERE  dt_rep BETWEEN m.month_start AND EOMONTH(m.month_start)
          AND  BLOCK_NAME   = 'Привлечение ФЛ'
          AND  SECTION_NAME = 'Срочные'
          AND  CUR = '810'
        ORDER BY dt_rep DESC
    ) latest
    WHERE latest.dt_rep IS NOT NULL
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
        END                       AS term_bucket,
        w.out_rub,
        w.rate_con                -- добавили для взвешивания
    FROM   month_latest ml
    JOIN   ALM.ALM.VW_Balance_Rest_All w
           ON  w.dt_rep       = ml.dt_rep
           AND w.BLOCK_NAME   = 'Привлечение ФЛ'
           AND w.SECTION_NAME = 'Срочные'
           AND w.CUR          = '810'
    -- учитываем только депозиты, открытые в том же месяце
    WHERE  EOMONTH(w.DT_OPEN_fact) = ml.month_end
)
SELECT
    dt_rep,
    term_bucket                                        AS [Срок, дн.],
    SUM(out_rub)                                       AS sum_out_rub,          -- все объёмы (в т.ч. без ставки)
    CAST(
        SUM(CASE WHEN rate_con IS NOT NULL AND out_rub IS NOT NULL
                 THEN rate_con * out_rub END)
        / NULLIF(SUM(CASE WHEN rate_con IS NOT NULL AND out_rub IS NOT NULL
                          THEN out_rub END), 0)
        AS decimal(12,6)
    )                                                  AS wavg_rate_con          -- средневзвешенная ставка по «не-NULL»
FROM base
GROUP BY dt_rep, term_bucket
ORDER BY dt_rep, term_bucket;
