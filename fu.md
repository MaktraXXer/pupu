USE [ALM];
SET NOCOUNT ON;

DECLARE @BaseDate date = '2026-04-29';
DECLARE @EndDate  date = '2026-05-17';

DECLARE @ExitFrom date = '2026-04-30';
DECLARE @ExitTo   date = '2026-05-17';

DECLARE @OpenFrom date = '2026-04-30';
DECLARE @OpenTo   date = '2026-05-17';

DECLARE @FlatRateFrom date = '2026-05-13';
DECLARE @eps decimal(18,6) = 0.000005;

IF OBJECT_ID('tempdb..#bal_base') IS NOT NULL DROP TABLE #bal_base;
IF OBJECT_ID('tempdb..#bal_end') IS NOT NULL DROP TABLE #bal_end;
IF OBJECT_ID('tempdb..#base_clients') IS NOT NULL DROP TABLE #base_clients;
IF OBJECT_ID('tempdb..#client_scope') IS NOT NULL DROP TABLE #client_scope;
IF OBJECT_ID('tempdb..#exit_agg') IS NOT NULL DROP TABLE #exit_agg;
IF OBJECT_ID('tempdb..#ns_base') IS NOT NULL DROP TABLE #ns_base;
IF OBJECT_ID('tempdb..#ns_end') IS NOT NULL DROP TABLE #ns_end;
IF OBJECT_ID('tempdb..#opened_agg') IS NOT NULL DROP TABLE #opened_agg;
IF OBJECT_ID('tempdb..#sms_clients') IS NOT NULL DROP TABLE #sms_clients;
IF OBJECT_ID('tempdb..#client_mart') IS NOT NULL DROP TABLE #client_mart;


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


-- 3. Все клиенты, у которых на 29.04 был хотя бы один срочный вклад или НС
-- Это флаг "старого/действующего" клиента на базовую дату
SELECT DISTINCT
    b.cli_id
INTO #base_clients
FROM #bal_base b
WHERE
    b.section_name IN (N'Срочные', N'Накопительный счёт');


-- 4. НС на базовую дату
SELECT
      b.cli_id
    , SUM(b.out_rub) AS ns_base_sum
INTO #ns_base
FROM #bal_base b
WHERE
    b.section_name = N'Накопительный счёт'
GROUP BY
    b.cli_id;


-- 5. НС на конечную дату
SELECT
      e.cli_id
    , SUM(e.out_rub) AS ns_end_sum
INTO #ns_end
FROM #bal_end e
WHERE
    e.section_name = N'Накопительный счёт'
GROUP BY
    e.cli_id;


-- 6. СМС
-- Важно: создаём до #client_scope, чтобы все клиенты из СМС попали в витрину
SELECT DISTINCT
    TRY_CAST(s.cli_id AS bigint) AS cli_id
INTO #sms_clients
FROM ALM_TEST.[TESTWORKSPACE].[sms_promo_messages] s WITH (NOLOCK)
WHERE
    s.msgbegindate_dt < DATEADD(DAY, 1, @OpenTo)
    AND TRY_CAST(s.cli_id AS bigint) IS NOT NULL;


-- 7. Клиенты:
-- 1) были вклады к выходу
-- 2) были открытые вклады
-- 3) была дельта НС
-- 4) получили СМС
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

    UNION

    SELECT COALESCE(nb.cli_id, ne.cli_id) AS cli_id
    FROM #ns_base nb
    FULL JOIN #ns_end ne
        ON ne.cli_id = nb.cli_id
    WHERE
        ISNULL(ne.ns_end_sum, 0) <> ISNULL(nb.ns_base_sum, 0)

    UNION

    SELECT s.cli_id
    FROM #sms_clients s
) x;


-- 8. Сумма вкладов к выходу
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


-- 9. Открытые вклады: раскладываем на 4 категории
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
              -- с 13.05 единая ставка/сетка после изменения условий
              WHEN MIN(o.dt_open) >= @FlatRateFrom
               AND MIN(o.termdays) BETWEEN 45 AND 79
               AND MIN(o.conv_norm) = 'AT_THE_END'
               AND SUM(o.out_rub) >= 1500000
               AND ABS(MIN(o.rate_con) - 0.1450) <= @eps
              THEN N'flat_145_from_13'

              WHEN MIN(o.dt_open) >= @FlatRateFrom
               AND MIN(o.termdays) BETWEEN 45 AND 79
               AND MIN(o.conv_norm) = 'AT_THE_END'
               AND SUM(o.out_rub) < 1500000
               AND ABS(MIN(o.rate_con) - 0.1430) <= @eps
              THEN N'flat_145_from_13'

              WHEN MIN(o.dt_open) >= @FlatRateFrom
               AND MIN(o.termdays) BETWEEN 45 AND 79
               AND MIN(o.conv_norm) <> 'AT_THE_END'
               AND SUM(o.out_rub) < 1500000
               AND ABS(MIN(o.rate_con) - 0.1410) <= @eps
              THEN N'flat_145_from_13'

              WHEN MIN(o.dt_open) >= @FlatRateFrom
               AND MIN(o.termdays) BETWEEN 45 AND 79
               AND MIN(o.conv_norm) <> 'AT_THE_END'
               AND SUM(o.out_rub) >= 1500000
               AND ABS(MIN(o.rate_con) - 0.1430) <= @eps
              THEN N'flat_145_from_13'

              -- новые деньги до 13.05
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

              -- удержание до 13.05
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


-- 10. Заполнение клиентского дата-марта
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

    -- новый флаг: на 29.04 у клиента был хотя бы один срочный вклад или НС
    , CASE WHEN bc.cli_id IS NOT NULL THEN 1 ELSE 0 END AS had_any_dep_or_ns_base_flag

    -- старый СМС-клиент: получил СМС и уже имел срочный вклад/НС на 29.04
    , CASE 
          WHEN sms.cli_id IS NOT NULL
           AND bc.cli_id IS NOT NULL
          THEN 1
          ELSE 0
      END AS sms_old_client_flag

    -- новый СМС-клиент: получил СМС, но на 29.04 не имел срочного вклада/НС
    , CASE 
          WHEN sms.cli_id IS NOT NULL
           AND bc.cli_id IS NULL
          THEN 1
          ELSE 0
      END AS sms_new_client_flag

    , CASE WHEN ISNULL(ex.exit_dep_sum, 0) > 0 THEN 1 ELSE 0 END AS had_exit_dep_flag
    , CASE WHEN ISNULL(op.opened_total_sum, 0) > 0 THEN 1 ELSE 0 END AS had_opened_dep_flag

    , CASE
          WHEN ISNULL(op.opened_new_money_sum, 0)
             + ISNULL(op.opened_retention_sum, 0)
             + ISNULL(op.opened_flat_145_sum, 0) = 0
          THEN 0
          ELSE 1
      END AS only_promo_opened_flag

    , CASE
          WHEN ISNULL(ne.ns_end_sum, 0) <> ISNULL(nb.ns_base_sum, 0) THEN 1
          ELSE 0
      END AS had_ns_delta_flag

    , CASE 
          WHEN sms.cli_id IS NOT NULL
           AND ISNULL(ex.exit_dep_sum, 0) = 0
           AND ISNULL(op.opened_total_sum, 0) = 0
           AND ISNULL(ne.ns_end_sum, 0) = ISNULL(nb.ns_base_sum, 0)
          THEN 1
          ELSE 0
      END AS only_sms_client_flag

    , ISNULL(ex.exit_dep_sum, 0) AS exit_dep_sum

    , ISNULL(op.opened_total_sum, 0) AS opened_total_sum
    , ISNULL(op.opened_new_money_sum, 0) AS opened_new_money_sum
    , ISNULL(op.opened_retention_sum, 0) AS opened_retention_sum
    , ISNULL(op.opened_flat_145_sum, 0) AS opened_flat_145_sum
    , ISNULL(op.opened_other_sum, 0) AS opened_other_sum

    , ISNULL(nb.ns_base_sum, 0) AS ns_base_sum
    , ISNULL(ne.ns_end_sum, 0) AS ns_end_sum
    , ISNULL(ne.ns_end_sum, 0) - ISNULL(nb.ns_base_sum, 0) AS ns_delta

    , CASE
          WHEN ISNULL(ne.ns_end_sum, 0) < ISNULL(nb.ns_base_sum, 0) THEN 1
          ELSE 0
      END AS ns_decrease_flag

INTO #client_mart
FROM #client_scope c
LEFT JOIN #base_clients bc
    ON bc.cli_id = c.cli_id
LEFT JOIN #exit_agg ex
    ON ex.cli_id = c.cli_id
LEFT JOIN #opened_agg op
    ON op.cli_id = c.cli_id
LEFT JOIN #ns_base nb
    ON nb.cli_id = c.cli_id
LEFT JOIN #ns_end ne
    ON ne.cli_id = c.cli_id
LEFT JOIN #sms_clients sms
    ON sms.cli_id = c.cli_id;


-- 11. Проверка, что в дата-марте нет дублей по cli_id
SELECT
      cli_id
    , COUNT(*) AS row_cnt
FROM #client_mart
GROUP BY
    cli_id
HAVING COUNT(*) > 1;


-- 12. Сам клиентский дата-март
SELECT
      cli_id
    , segment_flag
    , sms_promo_flag
    , had_any_dep_or_ns_base_flag
    , sms_old_client_flag
    , sms_new_client_flag
    , had_exit_dep_flag
    , had_opened_dep_flag
    , only_promo_opened_flag
    , had_ns_delta_flag
    , only_sms_client_flag
    , exit_dep_sum
    , opened_total_sum
    , opened_new_money_sum
    , opened_retention_sum
    , opened_flat_145_sum
    , opened_other_sum
    , ns_base_sum
    , ns_end_sum
    , ns_delta
    , ns_decrease_flag
FROM #client_mart
ORDER BY
      segment_flag
    , sms_promo_flag DESC
    , sms_old_client_flag DESC
    , sms_new_client_flag DESC
    , only_sms_client_flag DESC
    , had_exit_dep_flag DESC
    , had_opened_dep_flag DESC
    , had_ns_delta_flag DESC
    , opened_total_sum DESC
    , exit_dep_sum DESC
    , ns_delta ASC
    , cli_id;


-- 13. Сводка по СМС-клиентам: старые / новые
SELECT
      CASE
          WHEN sms_old_client_flag = 1 THEN N'01. СМС: старые клиенты, был вклад/НС на 29.04'
          WHEN sms_new_client_flag = 1 THEN N'02. СМС: новые клиенты, не было вклада/НС на 29.04'
          ELSE N'03. Не СМС'
      END AS client_base_group
    , segment_flag
    , COUNT(DISTINCT cli_id) AS client_cnt
    , SUM(exit_dep_sum) AS exit_dep_sum
    , SUM(opened_total_sum) AS opened_total_sum
    , SUM(opened_new_money_sum) AS opened_new_money_sum
    , SUM(opened_retention_sum) AS opened_retention_sum
    , SUM(opened_flat_145_sum) AS opened_flat_145_sum
    , SUM(opened_other_sum) AS opened_other_sum
    , SUM(ns_delta) AS ns_delta
FROM #client_mart
WHERE
    sms_promo_flag = 1
GROUP BY
      CASE
          WHEN sms_old_client_flag = 1 THEN N'01. СМС: старые клиенты, был вклад/НС на 29.04'
          WHEN sms_new_client_flag = 1 THEN N'02. СМС: новые клиенты, не было вклада/НС на 29.04'
          ELSE N'03. Не СМС'
      END
    , segment_flag
ORDER BY
      client_base_group
    , segment_flag;


-- 14. Выгрузка платежей по всем СМС-клиентам розницы
SELECT
      N'01. Все СМС-клиенты розницы' AS client_group
    , m.cli_id
    , m.segment_flag
    , m.sms_promo_flag
    , m.had_any_dep_or_ns_base_flag
    , m.sms_old_client_flag
    , m.sms_new_client_flag
    , m.had_exit_dep_flag
    , m.had_opened_dep_flag
    , m.only_promo_opened_flag
    , m.had_ns_delta_flag
    , m.only_sms_client_flag
    , m.exit_dep_sum
    , m.opened_total_sum
    , m.opened_new_money_sum
    , m.opened_retention_sum
    , m.opened_flat_145_sum
    , m.opened_other_sum
    , m.ns_base_sum
    , m.ns_end_sum
    , m.ns_delta
    , m.ns_decrease_flag
    , tr.*
FROM #client_mart m
INNER JOIN [ALM].[ehd].[VW_transfers_FL_det] tr WITH (NOLOCK)
    ON tr.cli_id = m.cli_id
WHERE
    m.segment_flag <> N'ДЧБО'
    AND m.sms_promo_flag = 1
    AND tr.dt_rep >= @OpenFrom
    AND tr.dt_rep <= @OpenTo
    AND tr.transaction_type <> N'Внутренние переводы'
    AND tr.transit_max_id IS NULL
ORDER BY
      m.sms_old_client_flag DESC
    , m.sms_new_client_flag DESC
    , m.cli_id
    , tr.dt_rep;
