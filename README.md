USE [ALM];
SET NOCOUNT ON;

DECLARE @BaseDate date = '2026-05-30'; -- дата баланса было
DECLARE @EndDate  date = '2026-06-03'; -- дата баланса стало

DECLARE @ExitFrom date = '2026-05-30'; -- выходы с
DECLARE @ExitTo   date = '2026-06-03'; -- выходы по включительно

DECLARE @OpenFrom date = '2026-05-30'; -- открытые с
DECLARE @OpenTo   date = '2026-06-03'; -- открытые по включительно

DECLARE @eps decimal(18,6) = 0.000005;

IF OBJECT_ID('tempdb..#bal_base') IS NOT NULL DROP TABLE #bal_base;
IF OBJECT_ID('tempdb..#bal_end') IS NOT NULL DROP TABLE #bal_end;
IF OBJECT_ID('tempdb..#client_scope') IS NOT NULL DROP TABLE #client_scope;
IF OBJECT_ID('tempdb..#opened_by_con') IS NOT NULL DROP TABLE #opened_by_con;
IF OBJECT_ID('tempdb..#opened_classified') IS NOT NULL DROP TABLE #opened_classified;
IF OBJECT_ID('tempdb..#opened_agg') IS NOT NULL DROP TABLE #opened_agg;
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


-- 3. Клиенты для анализа:
-- был НС на первую дату
-- и НЕ было срочных вкладов к выходу в периоде
SELECT
      ns.cli_id
    , SUM(ns.out_rub) AS ns_start_sum
INTO #client_scope
FROM #bal_base ns
WHERE
    ns.section_name = N'Накопительный счёт'
    AND NOT EXISTS (
        SELECT 1
        FROM #bal_base td
        WHERE
            td.cli_id = ns.cli_id
            AND td.section_name = N'Срочные'
            AND td.dt_close_plan >= @ExitFrom
            AND td.dt_close_plan <= @ExitTo
    )
GROUP BY
    ns.cli_id;


-- 4. Открытые срочные вклады по договорам
SELECT
      b.cli_id
    , b.con_id
    , SUM(b.out_rub) AS out_rub
    , MIN(b.rate_con) AS rate_con
    , MIN(b.termdays) AS termdays
    , MIN(b.conv_norm) AS conv_norm
INTO #opened_by_con
FROM #bal_end b
INNER JOIN #client_scope c
    ON b.cli_id = c.cli_id
WHERE
    b.section_name = N'Срочные'
    AND b.dt_open >= @OpenFrom
    AND b.dt_open <= @OpenTo
GROUP BY
      b.cli_id
    , b.con_id;


-- 5. Классификация открытых вкладов: 2m / 4m / other
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
INTO #opened_classified
FROM #opened_by_con o;


-- 6. Агрегация открытых вкладов по клиенту
SELECT
      cli_id
    , SUM(CASE WHEN open_category = N'2m'    THEN out_rub ELSE 0 END) AS opened_2m
    , SUM(CASE WHEN open_category = N'4m'    THEN out_rub ELSE 0 END) AS opened_4m
    , SUM(CASE WHEN open_category = N'other' THEN out_rub ELSE 0 END) AS opened_other
    , SUM(out_rub) AS opened_total
INTO #opened_agg
FROM #opened_classified
GROUP BY
    cli_id;


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

-- контрольная сумма вкладов к выходу
-- по логике отбора должна быть 0, но оставляем поле для проверки
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

-- новый флаг: есть ли у клиента другие рублевые срочные вклады на дату начала,
-- которые НЕ являются вкладами к выходу в заданном диапазоне
other_td_flag AS (
    SELECT
          c.cli_id
        , CASE
              WHEN EXISTS (
                  SELECT 1
                  FROM #bal_base b
                  WHERE b.cli_id = c.cli_id
                    AND b.section_name = N'Срочные'
                    AND NOT (
                            b.dt_close_plan >= @ExitFrom
                        AND b.dt_close_plan <= @ExitTo
                    )
              )
              THEN 1
              ELSE 0
          END AS has_other_rub_td_flag
    FROM #client_scope c
),

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
)

SELECT
      c.cli_id
    , f.client_segment AS segment_flag

    -- категория по объёму НС на первую дату
    , CASE
          WHEN ISNULL(c.ns_start_sum, 0) < 1500000 THEN N'01. НС < 1.5 млн'
          WHEN ISNULL(c.ns_start_sum, 0) < 5000000 THEN N'02. НС 1.5-5 млн'
          ELSE N'03. НС >= 5 млн'
      END AS ns_amount_flag

    -- контроль: вкладов к выходу быть не должно
    , ISNULL(e.exit_td_sum, 0) AS exit_td_sum

    -- 0 = нет других рублевых срочных вкладов на @BaseDate
    -- 1 = есть хотя бы один срочный рублевый вклад на @BaseDate,
    --     который НЕ попал в диапазон выходов @ExitFrom-@ExitTo
    , ISNULL(ot.has_other_rub_td_flag, 0) AS has_other_rub_td_flag

    -- НС
    , ISNULL(c.ns_start_sum, 0) AS ns_start_sum
    , ISNULL(ns2.ns_end_sum, 0) AS ns_end_sum
    , ISNULL(ns2.ns_end_sum, 0) - ISNULL(c.ns_start_sum, 0) AS ns_delta

    , CASE
          WHEN ISNULL(ns2.ns_end_sum, 0) < ISNULL(c.ns_start_sum, 0) THEN 1
          ELSE 0
      END AS ns_decrease_flag

    -- открытые вклады, разбитые на 3 категории
    , ISNULL(o.opened_2m, 0) AS opened_2m
    , ISNULL(o.opened_4m, 0) AS opened_4m
    , ISNULL(o.opened_other, 0) AS opened_other
    , ISNULL(o.opened_total, 0) AS opened_total

    -- 1. Все открытые вклады / НС на первую дату
    , CAST(
        ISNULL(o.opened_total, 0)
        / NULLIF(ISNULL(c.ns_start_sum, 0), 0)
      AS decimal(18,6)) AS retention_1_ns_all

    -- 2. Дельта НС + все открытые вклады / НС на первую дату
    , CAST(
        (
            ISNULL(ns2.ns_end_sum, 0) - ISNULL(c.ns_start_sum, 0)
            + ISNULL(o.opened_total, 0)
        )
        / NULLIF(ISNULL(c.ns_start_sum, 0), 0)
      AS decimal(18,6)) AS retention_2_ns_all_plus_ns

    -- 3. Только 2m / НС на первую дату
    , CAST(
        ISNULL(o.opened_2m, 0)
        / NULLIF(ISNULL(c.ns_start_sum, 0), 0)
      AS decimal(18,6)) AS retention_3_ns_2m

    -- 4. Дельта НС + 2m / НС на первую дату
    , CAST(
        (
            ISNULL(ns2.ns_end_sum, 0) - ISNULL(c.ns_start_sum, 0)
            + ISNULL(o.opened_2m, 0)
        )
        / NULLIF(ISNULL(c.ns_start_sum, 0), 0)
      AS decimal(18,6)) AS retention_4_ns_2m_plus_ns

    -- 5. 2m + 4m / НС на первую дату
    , CAST(
        (
            ISNULL(o.opened_2m, 0)
            + ISNULL(o.opened_4m, 0)
        )
        / NULLIF(ISNULL(c.ns_start_sum, 0), 0)
      AS decimal(18,6)) AS retention_5_ns_2m_plus_4m

    -- 6. Дельта НС + 2m + 4m / НС на первую дату
    , CAST(
        (
            ISNULL(ns2.ns_end_sum, 0) - ISNULL(c.ns_start_sum, 0)
            + ISNULL(o.opened_2m, 0)
            + ISNULL(o.opened_4m, 0)
        )
        / NULLIF(ISNULL(c.ns_start_sum, 0), 0)
      AS decimal(18,6)) AS retention_6_ns_2m_plus_4m_plus_ns

INTO #client_mart
FROM #client_scope c
LEFT JOIN client_flags f
    ON c.cli_id = f.cli_id
LEFT JOIN exit_sum e
    ON c.cli_id = e.cli_id
LEFT JOIN other_td_flag ot
    ON c.cli_id = ot.cli_id
LEFT JOIN ns_end ns2
    ON c.cli_id = ns2.cli_id
LEFT JOIN #opened_agg o
    ON c.cli_id = o.cli_id;


SELECT
      cli_id
    , segment_flag
    , ns_amount_flag
    , exit_td_sum
    , has_other_rub_td_flag

    , ns_start_sum
    , ns_end_sum
    , ns_delta
    , ns_decrease_flag

    , opened_2m
    , opened_4m
    , opened_other
    , opened_total

    , retention_1_ns_all
    , retention_2_ns_all_plus_ns
    , retention_3_ns_2m
    , retention_4_ns_2m_plus_ns
    , retention_5_ns_2m_plus_4m
    , retention_6_ns_2m_plus_4m_plus_ns
FROM #client_mart
ORDER BY
      segment_flag
    , ns_amount_flag
    , has_other_rub_td_flag DESC
    , ns_decrease_flag DESC
    , opened_total DESC
    , ns_delta ASC
    , cli_id;


-- Проверка: таких строк быть не должно, иначе где-то нарушена логика отбора
SELECT *
FROM #client_mart
WHERE exit_td_sum <> 0;
