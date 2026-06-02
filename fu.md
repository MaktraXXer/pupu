USE [ALM];
SET NOCOUNT ON;

DECLARE @dt_rep       date = '2026-05-31';
DECLARE @OpenFrom     date = '2026-02-28';
DECLARE @FlatRateFrom date = '2026-05-13';
DECLARE @eps          decimal(18,6) = 0.000005;


WITH opened_deposits AS (
    SELECT
        t.cli_id,
        t.con_id,
        t.out_rub,
        t.dt_open,
        t.dt_close_plan,
        t.termdays,
        t.rate_con,
        t.conv,
        CASE 
            WHEN t.conv = 'AT_THE_END' THEN 'AT_THE_END'
            ELSE 'NOT_AT_THE_END'
        END AS conv_norm,
        t.PROD_NAME_res
    FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
    WHERE
        t.dt_rep = @dt_rep
        AND t.dt_open >= @OpenFrom
        AND t.section_name = N'Срочные'
        AND t.block_name = N'Привлечение ФЛ'
        AND t.acc_role = N'LIAB'
        AND t.od_flag = 1
        AND t.cur = '810'
        AND t.out_rub IS NOT NULL
        AND t.out_rub >= 0
        AND t.PROD_NAME_res = N'Надёжный'
        AND (
            t.termdays BETWEEN 45 AND 79
            OR t.termdays BETWEEN 120 AND 150
        )
),
classified AS (
    SELECT
        o.*,

        CASE
            /* 45-79 дней */

            WHEN o.termdays BETWEEN 45 AND 79
             AND o.conv_norm = 'AT_THE_END'
             AND o.out_rub >= 1500000
             AND ABS(o.rate_con - 0.1450) <= @eps
            THEN N'2m_45_79_>=1.5_AT_END_14.5'

            WHEN o.termdays BETWEEN 45 AND 79
             AND o.conv_norm = 'AT_THE_END'
             AND o.out_rub < 1500000
             AND ABS(o.rate_con - 0.1430) <= @eps
            THEN N'2m_45_79_<1.5_AT_END_14.3'

            WHEN o.termdays BETWEEN 45 AND 79
             AND o.conv_norm <> 'AT_THE_END'
             AND o.out_rub < 1500000
             AND ABS(o.rate_con - 0.1410) <= @eps
            THEN N'2m_45_79_<1.5_NOT_AT_END_14.1'

            WHEN o.termdays BETWEEN 45 AND 79
             AND o.conv_norm <> 'AT_THE_END'
             AND o.out_rub >= 1500000
             AND ABS(o.rate_con - 0.1430) <= @eps
            THEN N'2m_45_79_>=1.5_NOT_AT_END_14.3'


            /* 120-150 дней */

            WHEN o.termdays BETWEEN 120 AND 150
             AND o.conv_norm = 'AT_THE_END'
             AND o.out_rub >= 1500000
             AND ABS(o.rate_con - 0.1400) <= @eps
            THEN N'4m_120_150_>=1.5_AT_END_14.0'

            WHEN o.termdays BETWEEN 120 AND 150
             AND o.conv_norm = 'AT_THE_END'
             AND o.out_rub < 1500000
             AND ABS(o.rate_con - 0.1390) <= @eps
            THEN N'4m_120_150_<1.5_AT_END_13.9'

            WHEN o.termdays BETWEEN 120 AND 150
             AND o.conv_norm <> 'AT_THE_END'
             AND o.out_rub < 1500000
             AND ABS(o.rate_con - 0.1370) <= @eps
            THEN N'4m_120_150_<1.5_NOT_AT_END_13.7'

            WHEN o.termdays BETWEEN 120 AND 150
             AND o.conv_norm <> 'AT_THE_END'
             AND o.out_rub >= 1500000
             AND ABS(o.rate_con - 0.1380) <= @eps
            THEN N'4m_120_150_>=1.5_NOT_AT_END_13.8'

        END AS rate_group,

        CASE
            WHEN o.termdays BETWEEN 45 AND 79 THEN N'2 месяца'
            WHEN o.termdays BETWEEN 120 AND 150 THEN N'4 месяца'
        END AS term_group

    FROM opened_deposits o
),
filtered AS (
    SELECT *
    FROM classified
    WHERE rate_group IS NOT NULL
)

SELECT
    term_group,
    rate_group,
    COUNT(DISTINCT cli_id) AS cli_cnt,
    COUNT(DISTINCT con_id) AS con_cnt,
    SUM(out_rub) AS out_rub_sum,
    AVG(rate_con) AS avg_rate_con,
    MIN(dt_open) AS min_dt_open,
    MAX(dt_open) AS max_dt_open
FROM filtered
GROUP BY
    term_group,
    rate_group
ORDER BY
    term_group,
    rate_group;
