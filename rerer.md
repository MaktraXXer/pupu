USE [ALM];
SET NOCOUNT ON;

DECLARE @BaseDate date = '2026-04-29';
DECLARE @EndDate  date = '2026-05-13';

DECLARE @ExitFrom date = '2026-04-30';
DECLARE @ExitTo   date = '2026-05-13'; -- теперь по 13 число включительно

DECLARE @OpenFrom date = '2026-04-30';
DECLARE @OpenTo   date = '2026-05-13'; -- теперь по 13 число включительно

DECLARE @FlatRateFrom date = '2026-05-13'; -- дата, с которой ставка стала единой
DECLARE @eps decimal(18,6) = 0.000005;

IF OBJECT_ID('tempdb..#bal_base') IS NOT NULL DROP TABLE #bal_base;
IF OBJECT_ID('tempdb..#bal_end') IS NOT NULL DROP TABLE #bal_end;
IF OBJECT_ID('tempdb..#client_scope') IS NOT NULL DROP TABLE #client_scope;
IF OBJECT_ID('tempdb..#exit_agg') IS NOT NULL DROP TABLE #exit_agg;
IF OBJECT_ID('tempdb..#ns_base') IS NOT NULL DROP TABLE #ns_base;
IF OBJECT_ID('tempdb..#ns_end') IS NOT NULL DROP TABLE #ns_end;
IF OBJECT_ID('tempdb..#opened_agg') IS NOT NULL DROP TABLE #opened_agg;
IF OBJECT_ID('tempdb..#sms_clients') IS NOT NULL DROP TABLE #sms_clients;


-- 1. Баланс на базовую дату
SELECT
      CAST(t.cli_id AS bigint) AS cli_id
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
FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
WHERE
    t.dt_rep = @BaseDate
    AND t.section_name IN (N'Срочные', N'Накопительный счёт')
    AND t.block_name = N'Привлечение ФЛ'
    AND t.acc_role = N'LIAB'
    AND t.od_flag = 1
    AND t.cur = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0;


-- 2. Баланс на конечную дату
SELECT
      CAST(t.cli_id AS bigint) AS cli_id
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
FROM [ALM].[ALM].[VW_balance_rest_all] t WITH (NOLOCK)
WHERE
    t.dt_rep = @EndDate
    AND t.section_name IN (N'Срочные', N'Накопительный счёт')
    AND t.block_name = N'Привлечение ФЛ'
    AND t.acc_role = N'LIAB'
    AND t.od_flag = 1
    AND t.cur = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0;


-- 3. Клиенты: либо были вклады к выходу, либо были открытия
SELECT DISTINCT cli_id
INTO #client_scope
FROM (
    SELECT b.cli_id
    FROM #bal_base b
    WHERE
        b.section_name = N'Срочные'
        AND b.dt_close_plan BETWEEN @ExitFrom AND @ExitTo

    UNION

    SELECT e.cli_id
    FROM #bal_end e
    WHERE
        e.section_name = N'Срочные'
        AND e.dt_open BETWEEN @OpenFrom AND @OpenTo
) x;


-- 4. Сумма вкладов к выходу
SELECT
      b.cli_id
    , SUM(b.out_rub) AS exit_dep_sum
INTO #exit_agg
FROM #bal_base b
WHERE
    b.section_name = N'Срочные'
    AND b.dt_close_plan BETWEEN @ExitFrom AND @ExitTo
GROUP BY
    b.cli_id;


-- 5. НС на базовую дату
SELECT
      b.cli_id
    , SUM(b.out_rub) AS ns_base_sum
INTO #ns_base
FROM #bal_base b
WHERE
    b.section_name = N'Накопительный счёт'
GROUP BY
    b.cli_id;


-- 6. НС на конечную дату
SELECT
      e.cli_id
    , SUM(e.out_rub) AS ns_end_sum
INTO #ns_end
FROM #bal_end e
WHERE
    e.section_name = N'Накопительный счёт'
GROUP BY
    e.cli_id;


-- 7. Список клиентов из СМС
SELECT DISTINCT
    TRY_CAST(s.cli_id AS bigint) AS cli_id
INTO #sms_clients
FROM ALM_TEST.[TESTWORKSPACE].[sms_promo_messages] s WITH (NOLOCK)
WHERE TRY_CAST(s.cli_id AS bigint) IS NOT NULL;


-- 8. Открытые вклады: раскладываем на 4 категории
SELECT
      q.cli_id
    , SUM(q.out_rub) AS opened_total_sum

    , SUM(CASE WHEN q.open_category = N'new_money' THEN q.out_rub ELSE 0 END) AS opened_new_money_sum
    , SUM(CASE WHEN q.open_category = N'retention' THEN q.out_rub ELSE 0 END) AS opened_retention_sum
    , SUM(CASE WHEN q.open_category = N'flat_145_from_13' THEN q.out_rub ELSE 0 END) AS opened_flat_145_sum
    , SUM(CASE WHEN q.open_category = N'other' THEN q.out_rub ELSE 0 END) AS opened_other_sum
INTO #opened_agg
FROM (
    SELECT
          o.cli_id
        , o.con_id
        , SUM(o.out_rub) AS out_rub
        , CASE
              -- новая отдельная группа с 13.05: единая ставка 14.5%
              WHEN MIN(o.dt_open) >= @FlatRateFrom
               AND MIN(o.termdays) BETWEEN 45 AND 79
               AND ABS(MIN(o.rate_con) - 0.1450) <= @eps
              THEN N'flat_145_from_13'

              -- новые деньги, старая сетка до 13.05
              WHEN MIN(o.dt_open) < @FlatRateFrom
               AND MIN(o.termdays) BETWEEN 45 AND 79
               AND MIN(o.conv_norm) = 'AT_THE_END'
               AND SUM(o.out_rub) >= 1500000
               AND ABS(MIN(o.rate_con) - 0.1470) <= @eps
              THEN N'new_money'

              WHEN MIN(o.dt_open) < @FlatRateFrom
               AND MIN(o.termdays) BETWEEN 45 AND 79
               AND MIN(o.conv_norm) = 'AT_THE_END'
               AND SUM(o.out_rub) < 1500000
               AND ABS(MIN(o.rate_con) - 0.1450) <= @eps
              THEN N'new_money'

              WHEN MIN(o.dt_open) < @FlatRateFrom
               AND MIN(o.termdays) BETWEEN 45 AND 79
               AND MIN(o.conv_norm) <> 'AT_THE_END'
               AND SUM(o.out_rub) < 1500000
               AND ABS(MIN(o.rate_con) - 0.1430) <= @eps
              THEN N'new_money'

              WHEN MIN(o.dt_open) < @FlatRateFrom
               AND MIN(o.termdays) BETWEEN 45 AND 79
               AND MIN(o.conv_norm) <> 'AT_THE_END'
               AND SUM(o.out_rub) >= 1500000
               AND ABS(MIN(o.rate_con) - 0.1450) <= @eps
              THEN N'new_money'

              -- удержание, старая сетка до 13.05
              WHEN MIN(o.dt_open) < @FlatRateFrom
               AND MIN(o.termdays) BETWEEN 45 AND 79
               AND MIN(o.conv_norm) = 'AT_THE_END'
               AND SUM(o.out_rub) >= 1500000
               AND ABS(MIN(o.rate_con) - 0.1450) <= @eps
              THEN N'retention'

              WHEN MIN(o.dt_open) < @FlatRateFrom
               AND MIN(o.termdays) BETWEEN 45 AND 79
               AND MIN(o.conv_norm) = 'AT_THE_END'
               AND SUM(o.out_rub) < 1500000
               AND ABS(MIN(o.rate_con) - 0.1430) <= @eps
              THEN N'retention'

              WHEN MIN(o.dt_open) < @FlatRateFrom
               AND MIN(o.termdays) BETWEEN 45 AND 79
               AND MIN(o.conv_norm) <> 'AT_THE_END'
               AND SUM(o.out_rub) < 1500000
               AND ABS(MIN(o.rate_con) - 0.1410) <= @eps
              THEN N'retention'

              WHEN MIN(o.dt_open) < @FlatRateFrom
               AND MIN(o.termdays) BETWEEN 45 AND 79
               AND MIN(o.conv_norm) <> 'AT_THE_END'
               AND SUM(o.out_rub) >= 1500000
               AND ABS(MIN(o.rate_con) - 0.1430) <= @eps
              THEN N'retention'

              ELSE N'other'
          END AS open_category
    FROM #bal_end o
    WHERE
        o.section_name = N'Срочные'
        AND o.dt_open BETWEEN @OpenFrom AND @OpenTo
    GROUP BY
          o.cli_id
        , o.con_id
) q
GROUP BY
    q.cli_id;


-- 9. Финальная клиентская витрина
SELECT
      c.cli_id

    , CASE
          WHEN EXISTS (
              SELECT 1
              FROM #bal_base b
              WHERE b.cli_id = c.cli_id
                AND b.TSEGMENTNAME = N'ДЧБО'
          )
          OR EXISTS (
              SELECT 1
              FROM #bal_end e
              WHERE e.cli_id = c.cli_id
                AND e.TSEGMENTNAME = N'ДЧБО'
          )
          THEN N'ДЧБО'
          ELSE N'Розница'
      END AS segment_flag

    , CASE WHEN sms.cli_id IS NOT NULL THEN 1 ELSE 0 END AS sms_promo_flag

    , CASE WHEN ISNULL(ex.exit_dep_sum, 0) > 0 THEN 1 ELSE 0 END AS had_exit_dep_flag
    , CASE WHEN ISNULL(op.opened_total_sum, 0) > 0 THEN 1 ELSE 0 END AS had_opened_dep_flag

    , ISNULL(ex.exit_dep_sum, 0) AS exit_dep_sum

    , ISNULL(op.opened_total_sum, 0) AS opened_total_sum
    , ISNULL(op.opened_new_money_sum, 0) AS opened_new_money_sum
    , ISNULL(op.opened_retention_sum, 0) AS opened_retention_sum
    , ISNULL(op.opened_flat_145_sum, 0) AS opened_flat_145_sum
    , ISNULL(op.opened_other_sum, 0) AS opened_other_sum

    , ISNULL(ne.ns_end_sum, 0) - ISNULL(nb.ns_base_sum, 0) AS ns_delta

    , CASE
          WHEN ISNULL(ne.ns_end_sum, 0) < ISNULL(nb.ns_base_sum, 0) THEN 1
          ELSE 0
      END AS ns_decrease_flag

FROM #client_scope c
LEFT JOIN #exit_agg ex
    ON ex.cli_id = c.cli_id
LEFT JOIN #opened_agg op
    ON op.cli_id = c.cli_id
LEFT JOIN #ns_base nb
    ON nb.cli_id = c.cli_id
LEFT JOIN #ns_end ne
    ON ne.cli_id = c.cli_id
LEFT JOIN #sms_clients sms
    ON sms.cli_id = c.cli_id
ORDER BY
      segment_flag
    , sms_promo_flag DESC
    , had_exit_dep_flag DESC
    , had_opened_dep_flag DESC
    , opened_total_sum DESC
    , exit_dep_sum DESC
    , c.cli_id;
