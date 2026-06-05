USE [ALM];
SET NOCOUNT ON;

DECLARE @BaseDate date = '2026-05-30'; -- дата баланса было
DECLARE @EndDate  date = '2026-06-02'; -- дата баланса стало

DECLARE @ExitFrom date = '2026-05-30'; -- выходы с
DECLARE @ExitTo   date = '2026-06-02'; -- выходы по включительно

DECLARE @OpenFrom date = '2026-05-30'; -- открытые с
DECLARE @OpenTo   date = '2026-06-02'; -- открытые по включительно

DECLARE @eps decimal(18,6) = 0.000005;

IF OBJECT_ID('tempdb..#bal_base') IS NOT NULL DROP TABLE #bal_base;
IF OBJECT_ID('tempdb..#bal_end') IS NOT NULL DROP TABLE #bal_end;
IF OBJECT_ID('tempdb..#client_scope') IS NOT NULL DROP TABLE #client_scope;
IF OBJECT_ID('tempdb..#client_mart') IS NOT NULL DROP TABLE #client_mart;


SELECT
      CAST(t.dt_rep AS date) AS dt_rep
    , CAST(t.cli_id AS bigint) AS cli_id
    , CAST(t.con_id AS bigint) AS con_id
    , CAST(t.dt_open AS date) AS dt_open
    , CAST(t.dt_close_plan AS date) AS dt_close_plan
    , t.section_name
    , CAST(t.out_rub AS decimal(38,6)) AS out_rub
    , CAST(t.rate_con AS decimal(18,6)) AS rate_con
    , CASE
          WHEN NULLIF(LTRIM(RTRIM(COALESCE(t.conv, ''))), '') IS NULL THEN 'AT_THE_END'
          ELSE UPPER(LTRIM(RTRIM(t.conv)))
      END AS conv_norm
    , t.termdays
    , t.TSEGMENTNAME
INTO #bal_base
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE
    t.dt_rep = @BaseDate
    AND t.section_name IN (N'Срочные', N'Накопительный счёт')
    AND t.block_name = N'Привлечение ФЛ'
    AND t.acc_role   = N'LIAB'
    AND t.od_flag    = 1
    AND t.cur        = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0;


SELECT
      CAST(t.dt_rep AS date) AS dt_rep
    , CAST(t.cli_id AS bigint) AS cli_id
    , CAST(t.con_id AS bigint) AS con_id
    , CAST(t.dt_open AS date) AS dt_open
    , CAST(t.dt_close_plan AS date) AS dt_close_plan
    , t.section_name
    , CAST(t.out_rub AS decimal(38,6)) AS out_rub
    , CAST(t.rate_con AS decimal(18,6)) AS rate_con
    , CASE
          WHEN NULLIF(LTRIM(RTRIM(COALESCE(t.conv, ''))), '') IS NULL THEN 'AT_THE_END'
          ELSE UPPER(LTRIM(RTRIM(t.conv)))
      END AS conv_norm
    , t.termdays
    , t.TSEGMENTNAME
INTO #bal_end
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE
    t.dt_rep = @EndDate
    AND t.section_name IN (N'Срочные', N'Накопительный счёт')
    AND t.block_name = N'Привлечение ФЛ'
    AND t.acc_role   = N'LIAB'
    AND t.od_flag    = 1
    AND t.cur        = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0;


SELECT DISTINCT
    cli_id
INTO #client_scope
FROM #bal_base
WHERE
    section_name = N'Срочные'
    AND dt_close_plan >= @ExitFrom
    AND dt_close_plan <= @ExitTo;


WITH client_flags AS (
    SELECT
          c.cli_id
        , CASE
              WHEN EXISTS (
                  SELECT 1
                  FROM #bal_base b
                  WHERE b.cli_id = c.cli_id
                    AND b.section_name IN (N'Срочные', N'Накопительный счёт')
                    AND b.TSEGMENTNAME = N'ДЧБО'
              )
              THEN N'ДЧБО'
              ELSE N'Розница'
          END AS client_segment
    FROM #client_scope c
),

exit_sum AS (
    SELECT
          cli_id
        , SUM(out_rub) AS exit_td_sum
    FROM #bal_base
    WHERE
        section_name = N'Срочные'
        AND dt_close_plan >= @ExitFrom
        AND dt_close_plan <= @ExitTo
    GROUP BY
        cli_id
),

ns_start AS (
    SELECT
          cli_id
        , SUM(out_rub) AS ns_start_sum
    FROM #bal_base
    WHERE
        section_name = N'Накопительный счёт'
    GROUP BY
        cli_id
),

ns_end AS (
    SELECT
          cli_id
        , SUM(out_rub) AS ns_end_sum
    FROM #bal_end
    WHERE
        section_name = N'Накопительный счёт'
    GROUP BY
        cli_id
),

opened_by_con AS (
    SELECT
          b.cli_id
        , b.con_id
        , SUM(b.out_rub) AS out_rub
        , MIN(b.rate_con) AS rate_con
        , MIN(b.termdays) AS termdays
        , CASE
              WHEN MIN(NULLIF(LTRIM(RTRIM(COALESCE(b.conv_norm, ''))), '')) IS NULL THEN 'AT_THE_END'
              ELSE MIN(b.conv_norm)
          END AS conv_norm
    FROM #bal_end b
    INNER JOIN #client_scope c
        ON b.cli_id = c.cli_id
    WHERE
        b.section_name = N'Срочные'
        AND b.dt_open >= @OpenFrom
        AND b.dt_open <= @OpenTo
    GROUP BY
          b.cli_id
        , b.con_id
),

opened_classified AS (
    SELECT
          o.cli_id
        , o.con_id
        , o.out_rub
        , CASE
            /* 45-79 дней */
            WHEN o.termdays BETWEEN 45 AND 79
             AND o.conv_norm = 'AT_THE_END'
             AND o.out_rub >= 1500000
             AND ABS(o.rate_con - 0.1450) <= @eps
            THEN N'2m'

            WHEN o.termdays BETWEEN 45 AND 79
             AND o.conv_norm = 'AT_THE_END'
             AND o.out_rub < 1500000
             AND ABS(o.rate_con - 0.1430) <= @eps
            THEN N'2m'

            WHEN o.termdays BETWEEN 45 AND 79
             AND o.conv_norm <> 'AT_THE_END'
             AND o.out_rub < 1500000
             AND ABS(o.rate_con - 0.1410) <= @eps
            THEN N'2m'

            WHEN o.termdays BETWEEN 45 AND 79
             AND o.conv_norm <> 'AT_THE_END'
             AND o.out_rub >= 1500000
             AND ABS(o.rate_con - 0.1430) <= @eps
            THEN N'2m'


            /* 120-150 дней */
            WHEN o.termdays BETWEEN 120 AND 150
             AND o.conv_norm = 'AT_THE_END'
             AND o.out_rub >= 1500000
             AND ABS(o.rate_con - 0.1400) <= @eps
            THEN N'4m'

            WHEN o.termdays BETWEEN 120 AND 150
             AND o.conv_norm = 'AT_THE_END'
             AND o.out_rub < 1500000
             AND ABS(o.rate_con - 0.1390) <= @eps
            THEN N'4m'

            WHEN o.termdays BETWEEN 120 AND 150
             AND o.conv_norm <> 'AT_THE_END'
             AND o.out_rub < 1500000
             AND ABS(o.rate_con - 0.1370) <= @eps
            THEN N'4m'

            WHEN o.termdays BETWEEN 120 AND 150
             AND o.conv_norm <> 'AT_THE_END'
             AND o.out_rub >= 1500000
             AND ABS(o.rate_con - 0.1380) <= @eps
            THEN N'4m'

            ELSE N'other'
          END AS open_category
    FROM opened_by_con o
),

opened_agg AS (
    SELECT
          cli_id
        , SUM(CASE WHEN open_category = N'2m'    THEN out_rub ELSE 0 END) AS opened_2m
        , SUM(CASE WHEN open_category = N'4m'    THEN out_rub ELSE 0 END) AS opened_4m
        , SUM(CASE WHEN open_category = N'other' THEN out_rub ELSE 0 END) AS opened_other
        , SUM(out_rub) AS opened_total
    FROM opened_classified
    GROUP BY
        cli_id
)

SELECT
      c.cli_id
    , f.client_segment AS segment_flag

    , CASE
          WHEN ISNULL(e.exit_td_sum, 0) < 1500000 THEN N'01. Выход < 1.5 млн'
          WHEN ISNULL(e.exit_td_sum, 0) < 5000000 THEN N'02. Выход 1.5-5 млн'
          ELSE N'03. Выход >= 5 млн'
      END AS exit_amount_flag

    , ISNULL(e.exit_td_sum, 0) AS exit_td_sum

    , ISNULL(ns1.ns_start_sum, 0) AS ns_start_sum

    -- открытые вклады, разбитые на 3 категории
    , ISNULL(o.opened_2m, 0) AS opened_2m
    , ISNULL(o.opened_4m, 0) AS opened_4m
    , ISNULL(o.opened_other, 0) AS opened_other
    , ISNULL(o.opened_total, 0) AS opened_total

    , ISNULL(ns2.ns_end_sum, 0) AS ns_end_sum
    , ISNULL(ns2.ns_end_sum, 0) - ISNULL(ns1.ns_start_sum, 0) AS ns_delta

    -- 1. Все открытые вклады / выходящие вклады
    , CAST(
        ISNULL(o.opened_total, 0)
        / NULLIF(ISNULL(e.exit_td_sum, 0), 0)
      AS decimal(18,6)) AS retention_1_td_all

    -- 2. Дельта НС + все открытые вклады / выходящие вклады
    , CAST(
        (
            ISNULL(ns2.ns_end_sum, 0) - ISNULL(ns1.ns_start_sum, 0)
            + ISNULL(o.opened_total, 0)
        )
        / NULLIF(ISNULL(e.exit_td_sum, 0), 0)
      AS decimal(18,6)) AS retention_2_td_all_plus_ns

    -- 3. Только 2m / выходящие вклады
    , CAST(
        ISNULL(o.opened_2m, 0)
        / NULLIF(ISNULL(e.exit_td_sum, 0), 0)
      AS decimal(18,6)) AS retention_3_td_2m

    -- 4. Дельта НС + 2m / выходящие вклады
    , CAST(
        (
            ISNULL(ns2.ns_end_sum, 0) - ISNULL(ns1.ns_start_sum, 0)
            + ISNULL(o.opened_2m, 0)
        )
        / NULLIF(ISNULL(e.exit_td_sum, 0), 0)
      AS decimal(18,6)) AS retention_4_td_2m_plus_ns

    -- 5. 2m + 4m / выходящие вклады
    , CAST(
        (
            ISNULL(o.opened_2m, 0)
            + ISNULL(o.opened_4m, 0)
        )
        / NULLIF(ISNULL(e.exit_td_sum, 0), 0)
      AS decimal(18,6)) AS retention_5_td_2m_plus_4m

    -- 6. Дельта НС + 2m + 4m / выходящие вклады
    , CAST(
        (
            ISNULL(ns2.ns_end_sum, 0) - ISNULL(ns1.ns_start_sum, 0)
            + ISNULL(o.opened_2m, 0)
            + ISNULL(o.opened_4m, 0)
        )
        / NULLIF(ISNULL(e.exit_td_sum, 0), 0)
      AS decimal(18,6)) AS retention_6_td_2m_plus_4m_plus_ns

INTO #client_mart
FROM #client_scope c
LEFT JOIN client_flags f
    ON c.cli_id = f.cli_id
LEFT JOIN exit_sum e
    ON c.cli_id = e.cli_id
LEFT JOIN ns_start ns1
    ON c.cli_id = ns1.cli_id
LEFT JOIN ns_end ns2
    ON c.cli_id = ns2.cli_id
LEFT JOIN opened_agg o
    ON c.cli_id = o.cli_id;


SELECT
      cli_id
    , segment_flag
    , exit_amount_flag
    , exit_td_sum
    , ns_start_sum
    , opened_2m
    , opened_4m
    , opened_other
    , opened_total
    , ns_end_sum
    , ns_delta
    , retention_1_td_all
    , retention_2_td_all_plus_ns
    , retention_3_td_2m
    , retention_4_td_2m_plus_ns
    , retention_5_td_2m_plus_4m
    , retention_6_td_2m_plus_4m_plus_ns
FROM #client_mart
ORDER BY
      segment_flag
    , exit_amount_flag
    , cli_id;
