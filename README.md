DECLARE @dt date = '2026-08-31';

WITH base AS (
    SELECT
        t.con_id,
        t.cli_id,
        t.TSEGMENTNAME,
        t.out_rub
    FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
    WHERE t.dt_rep = @dt
      AND t.section_name = N'Срочные'
      AND t.block_name = N'Привлечение ФЛ'
      AND t.od_flag = 1
      AND t.cur = '810'
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0
),

cli_segment AS (
    SELECT
        cli_id,
        CASE
            WHEN MAX(CASE WHEN TSEGMENTNAME = N'ДЧБО' THEN 1 ELSE 0 END) = 1
                THEN N'ДЧБО'
            ELSE MAX(TSEGMENTNAME)
        END AS TSEGMENTNAME
    FROM base
    GROUP BY cli_id
),

cli_attr AS (
    SELECT
        a.cli_id,
        MAX(CASE WHEN a.cli_attr_code = 'IS_SALARY_CLIENT' THEN 1 ELSE 0 END) AS IS_SALARY_CLIENT,
        MAX(CASE WHEN a.cli_attr_code = 'PREMIUM_PACKAGE'   THEN 1 ELSE 0 END) AS PREMIUM_PACKAGE
    FROM [ALM].[ehd].[attr_cli_stat_sal] a WITH (NOLOCK)
    WHERE @dt BETWEEN a.dt_from AND a.dt_to
      AND a.cli_attr_code IN ('IS_SALARY_CLIENT', 'PREMIUM_PACKAGE')
    GROUP BY a.cli_id
)

SELECT
    s.TSEGMENTNAME,
    ISNULL(a.IS_SALARY_CLIENT, 0) AS IS_SALARY_CLIENT,
    ISNULL(a.PREMIUM_PACKAGE, 0)   AS PREMIUM_PACKAGE,
    SUM(b.out_rub)                 AS OUT_RUB,
    COUNT(DISTINCT b.cli_id)       AS CLIENT_CNT
FROM base b
JOIN cli_segment s
    ON s.cli_id = b.cli_id
LEFT JOIN cli_attr a
    ON a.cli_id = b.cli_id
GROUP BY
    s.TSEGMENTNAME,
    ISNULL(a.IS_SALARY_CLIENT, 0),
    ISNULL(a.PREMIUM_PACKAGE, 0)
ORDER BY
    s.TSEGMENTNAME,
    ISNULL(a.IS_SALARY_CLIENT, 0),
    ISNULL(a.PREMIUM_PACKAGE, 0);
