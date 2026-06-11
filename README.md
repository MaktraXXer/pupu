WITH base AS (
    SELECT
        a.out_rub,
        a.[Срок, дн],
        b.AGG_PROD_NAME,
        CASE
            WHEN a.[Срок, дн] >= 0        AND a.[Срок, дн] <  15 * 365 THEN N'0-15 лет'
            WHEN a.[Срок, дн] >= 15 * 365 AND a.[Срок, дн] <  20 * 365 THEN N'15-20 лет'
            WHEN a.[Срок, дн] >= 20 * 365 AND a.[Срок, дн] <  25 * 365 THEN N'20-25 лет'
            WHEN a.[Срок, дн] >= 25 * 365 AND a.[Срок, дн] <= 30 * 365 THEN N'25-30 лет'
            WHEN a.[Срок, дн] >  30 * 365 THEN N'Больше 30 лет'
            ELSE N'Вне бакетов'
        END AS term_bucket,
        CASE
            WHEN a.[Срок, дн] >= 0        AND a.[Срок, дн] <  15 * 365 THEN 1
            WHEN a.[Срок, дн] >= 15 * 365 AND a.[Срок, дн] <  20 * 365 THEN 2
            WHEN a.[Срок, дн] >= 20 * 365 AND a.[Срок, дн] <  25 * 365 THEN 3
            WHEN a.[Срок, дн] >= 25 * 365 AND a.[Срок, дн] <= 30 * 365 THEN 4
            WHEN a.[Срок, дн] >  30 * 365 THEN 5
            ELSE 99
        END AS bucket_sort
    FROM [ALM].[ALM].[VW_FL_mortgage_TERM_WEEK] a WITH (NOLOCK)
    LEFT JOIN ALM_TEST.mort.man_mort_mapping b WITH (NOLOCK)
        ON a.PROD_NAME_RES = b.prod_name
    WHERE 
        a.dt_rep = '2026-06-10'
        AND b.AGG_PROD_NAME IN (N'Семейная ипотека')
        AND a.out_rub IS NOT NULL
        AND a.[Срок, дн] IS NOT NULL
),
agg AS (
    SELECT
        AGG_PROD_NAME,
        term_bucket,
        bucket_sort,
        SUM(out_rub) AS out_rub,
        SUM([Срок, дн] * out_rub) / NULLIF(SUM(out_rub), 0) AS avg_term_days
    FROM base
    WHERE term_bucket <> N'Вне бакетов'
    GROUP BY
        AGG_PROD_NAME,
        term_bucket,
        bucket_sort
),
total AS (
    SELECT
        SUM(out_rub) AS total_out_rub
    FROM agg
)
SELECT
    a.AGG_PROD_NAME,
    a.term_bucket AS [Бакет],
    a.avg_term_days AS [Срочность бакета, дн],
    a.avg_term_days / 365.0 AS [Срочность бакета, лет],
    a.out_rub AS [Объем],
    a.out_rub / NULLIF(t.total_out_rub, 0) AS [Доля объема]
FROM agg a
CROSS JOIN total t
ORDER BY
    a.AGG_PROD_NAME,
    a.bucket_sort;
