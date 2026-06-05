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


-- 1. Баланс на базовую дату
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


-- 2. Баланс на конечную дату
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


-- 3. Отбираем ТОЛЬКО клиентов, у которых был НС на первую дату
SELECT
      b.cli_id
    , SUM(b.out_rub) AS ns_start_sum
INTO #client_scope
FROM #bal_base b
WHERE
    b.section_name = N'Накопительный счёт'
GROUP BY
    b.cli_id;


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

-- 4. Сумма вкладов к выходу у этих клиентов
exit_sum AS (
    SELECT
          b.cli_id
        , SUM(b.out_rub) AS exit_td_sum
    FROM #bal_base b
    INNER JOIN #client_scope c
        ON c.cli_id = b.cli_id
    WHERE
        b.section_name = N'Срочные'
        AND b.dt_close_plan >= @ExitFrom
        AND b.dt_close_plan <= @ExitTo
    GROUP BY
        b.cli_id
),

-- 5. НС на конечную дату
ns_end AS (
    SELECT
          e.cli_id
        , SUM(e.out_rub) AS ns_end_sum
    FROM #bal_end e
    INNER JOIN #client_scope c
        ON c.cli_id = e.cli_id
    WHERE
        e.section_name = N'Накопительный счёт'
    GROUP BY
        e.cli_id
),

-- 6. Вклады на базовую дату
td_start AS (
    SELECT
          b.cli_id
        , SUM(b.out_rub) AS td_start_sum
    FROM #bal_base b
    INNER JOIN #client_scope c
        ON c.cli_id = b.cli_id
    WHERE
        b.section_name = N'Срочные'
    GROUP BY
        b.cli_id
),

-- 7. Вклады на конечную дату
td_end AS (
    SELECT
          e.cli_id
        , SUM(e.out_rub) AS td_end_sum
    FROM #bal_end e
    INNER JOIN #client_scope c
        ON c.cli_id = e.cli_id
    WHERE
        e.section_name = N'Срочные'
    GROUP BY
        e.cli_id
),

-- 8. Открытые вклады за период
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

-- 9. Классификация открытых вкладов: 2m / 4m / other
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

    -- НС
    , ISNULL(c.ns_start_sum, 0) AS ns_start_sum
    , ISNULL(ns2.ns_end_sum, 0) AS ns_end_sum
    , ISNULL(ns2.ns_end_sum, 0) - ISNULL(c.ns_start_sum, 0) AS ns_delta

    , CASE
          WHEN ISNULL(ns2.ns_end_sum, 0) < ISNULL(c.ns_start_sum, 0) THEN 1
          ELSE 0
      END AS ns_decrease_flag

    , CASE
          WHEN ISNULL(ns2.ns_end_sum, 0) > ISNULL(c.ns_start_sum, 0) THEN 1
          ELSE 0
      END AS ns_increase_flag

    -- срочные вклады на двух датах
    , ISNULL(td1.td_start_sum, 0) AS td_start_sum
    , ISNULL(td2.td_end_sum, 0) AS td_end_sum
    , ISNULL(td2.td_end_sum, 0) - ISNULL(td1.td_start_sum, 0) AS td_delta

    -- вклады к выходу
    , ISNULL(e.exit_td_sum, 0) AS exit_td_sum
    , CASE WHEN ISNULL(e.exit_td_sum, 0) > 0 THEN 1 ELSE 0 END AS had_exit_td_flag

    -- открытые вклады, разбитые на 3 категории
    , ISNULL(o.opened_2m, 0) AS opened_2m
    , ISNULL(o.opened_4m, 0) AS opened_4m
    , ISNULL(o.opened_other, 0) AS opened_other
    , ISNULL(o.opened_total, 0) AS opened_total

    , CASE WHEN ISNULL(o.opened_total, 0) > 0 THEN 1 ELSE 0 END AS had_opened_td_flag

    -- условная оценка перекладки НС во вклады:
    -- если НС снизился, а вклады открылись, берем минимум между снижением НС и открытиями
    , CASE
          WHEN ISNULL(ns2.ns_end_sum, 0) < ISNULL(c.ns_start_sum, 0)
           AND ISNULL(o.opened_total, 0) > 0
          THEN
              CASE
                  WHEN ABS(ISNULL(ns2.ns_end_sum, 0) - ISNULL(c.ns_start_sum, 0)) <= ISNULL(o.opened_total, 0)
                  THEN ABS(ISNULL(ns2.ns_end_sum, 0) - ISNULL(c.ns_start_sum, 0))
                  ELSE ISNULL(o.opened_total, 0)
              END
          ELSE 0
      END AS estimated_ns_to_td_sum

    -- retention-метрики относительно начального НС
    , CAST(
        ISNULL(o.opened_total, 0)
        / NULLIF(ISNULL(c.ns_start_sum, 0), 0)
      AS decimal(18,6)) AS ns_to_opened_total_ratio

    , CAST(
        ISNULL(o.opened_2m, 0)
        / NULLIF(ISNULL(c.ns_start_sum, 0), 0)
      AS decimal(18,6)) AS ns_to_opened_2m_ratio

    , CAST(
        ISNULL(o.opened_4m, 0)
        / NULLIF(ISNULL(c.ns_start_sum, 0), 0)
      AS decimal(18,6)) AS ns_to_opened_4m_ratio

INTO #client_mart
FROM #client_scope c
LEFT JOIN client_flags f
    ON c.cli_id = f.cli_id
LEFT JOIN exit_sum e
    ON c.cli_id = e.cli_id
LEFT JOIN ns_end ns2
    ON c.cli_id = ns2.cli_id
LEFT JOIN td_start td1
    ON c.cli_id = td1.cli_id
LEFT JOIN td_end td2
    ON c.cli_id = td2.cli_id
LEFT JOIN opened_agg o
    ON c.cli_id = o.cli_id;


SELECT
      cli_id
    , segment_flag
    , exit_amount_flag

    , ns_start_sum
    , ns_end_sum
    , ns_delta
    , ns_decrease_flag
    , ns_increase_flag

    , td_start_sum
    , td_end_sum
    , td_delta

    , exit_td_sum
    , had_exit_td_flag

    , opened_2m
    , opened_4m
    , opened_other
    , opened_total
    , had_opened_td_flag

    , estimated_ns_to_td_sum
    , ns_to_opened_total_ratio
    , ns_to_opened_2m_ratio
    , ns_to_opened_4m_ratio
FROM #client_mart
ORDER BY
      segment_flag
    , ns_decrease_flag DESC
    , opened_total DESC
    , ns_delta ASC
    , cli_id;
