USE [ALM];
SET NOCOUNT ON;

DECLARE @BaseDate date = '2026-04-29'; -- баланс было
DECLARE @EndDate  date = '2026-05-13'; -- баланс стало

DECLARE @ExitFrom date = '2026-04-30'; -- вклады к выходу с
DECLARE @ExitTo   date = '2026-05-12'; -- вклады к выходу по включительно

DECLARE @OpenFrom date = '2026-04-30'; -- открытия с
DECLARE @OpenTo   date = '2026-05-12'; -- открытия по включительно

DECLARE @eps decimal(18,6) = 0.000005;

IF OBJECT_ID('tempdb..#bal_base') IS NOT NULL DROP TABLE #bal_base;
IF OBJECT_ID('tempdb..#bal_end') IS NOT NULL DROP TABLE #bal_end;
IF OBJECT_ID('tempdb..#client_scope') IS NOT NULL DROP TABLE #client_scope;
IF OBJECT_ID('tempdb..#sms_clients') IS NOT NULL DROP TABLE #sms_clients;
IF OBJECT_ID('tempdb..#client_mart') IS NOT NULL DROP TABLE #client_mart;


SELECT
      CAST(t.dt_rep AS date) AS dt_rep
    , CAST(t.cli_id AS bigint) AS cli_id
    , CAST(t.con_id AS bigint) AS con_id
    , t.acc_no
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
    , t.PROD_NAME_res
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


SELECT
      CAST(t.dt_rep AS date) AS dt_rep
    , CAST(t.cli_id AS bigint) AS cli_id
    , CAST(t.con_id AS bigint) AS con_id
    , t.acc_no
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
    , t.PROD_NAME_res
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


SELECT DISTINCT
    TRY_CAST(s.cli_id AS bigint) AS cli_id
INTO #sms_clients
FROM ALM_TEST.[TESTWORKSPACE].[sms_promo_messages] s WITH (NOLOCK)
WHERE TRY_CAST(s.cli_id AS bigint) IS NOT NULL;


WITH exit_clients AS (
    SELECT DISTINCT
        b.cli_id
    FROM #bal_base b
    WHERE
        b.section_name = N'Срочные'
        AND b.dt_close_plan BETWEEN @ExitFrom AND @ExitTo
),

opened_clients AS (
    SELECT DISTINCT
        e.cli_id
    FROM #bal_end e
    WHERE
        e.section_name = N'Срочные'
        AND e.dt_open BETWEEN @OpenFrom AND @OpenTo
),

client_scope AS (
    SELECT cli_id FROM exit_clients
    UNION
    SELECT cli_id FROM opened_clients
)

SELECT DISTINCT
    cli_id
INTO #client_scope
FROM client_scope;


WITH client_segment AS (
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
    FROM #client_scope c
),

exit_deposits AS (
    SELECT
          b.cli_id
        , COUNT(DISTINCT b.con_id) AS exit_dep_cnt
        , SUM(b.out_rub) AS exit_dep_sum
    FROM #bal_base b
    WHERE
        b.section_name = N'Срочные'
        AND b.dt_close_plan BETWEEN @ExitFrom AND @ExitTo
    GROUP BY
        b.cli_id
),

balance_base AS (
    SELECT
          b.cli_id
        , SUM(CASE WHEN b.section_name = N'Срочные' THEN b.out_rub ELSE 0 END) AS td_base_sum
        , SUM(CASE WHEN b.section_name = N'Накопительный счёт' THEN b.out_rub ELSE 0 END) AS ns_base_sum
        , SUM(b.out_rub) AS total_base_sum
    FROM #bal_base b
    GROUP BY
        b.cli_id
),

balance_end AS (
    SELECT
          e.cli_id
        , SUM(CASE WHEN e.section_name = N'Срочные' THEN e.out_rub ELSE 0 END) AS td_end_sum
        , SUM(CASE WHEN e.section_name = N'Накопительный счёт' THEN e.out_rub ELSE 0 END) AS ns_end_sum
        , SUM(e.out_rub) AS total_end_sum
    FROM #bal_end e
    GROUP BY
        e.cli_id
),

opened_by_con AS (
    SELECT
          e.cli_id
        , e.con_id
        , SUM(e.out_rub) AS out_rub
        , MIN(e.rate_con) AS rate_con
        , MIN(e.termdays) AS termdays
        , MIN(e.dt_open) AS dt_open
        , MIN(e.conv_norm) AS conv_norm
    FROM #bal_end e
    WHERE
        e.section_name = N'Срочные'
        AND e.dt_open BETWEEN @OpenFrom AND @OpenTo
    GROUP BY
          e.cli_id
        , e.con_id
),

opened_classified AS (
    SELECT
          o.cli_id
        , o.con_id
        , o.out_rub
        , o.rate_con
        , o.termdays
        , o.dt_open
        , o.conv_norm

        , CASE 
              WHEN o.termdays BETWEEN 45 AND 79
               AND o.conv_norm = 'AT_THE_END'
               AND o.out_rub >= 1500000
               AND ABS(o.rate_con - 0.1470) <= @eps THEN 1

              WHEN o.termdays BETWEEN 45 AND 79
               AND o.conv_norm = 'AT_THE_END'
               AND o.out_rub < 1500000
               AND ABS(o.rate_con - 0.1450) <= @eps THEN 1

              WHEN o.termdays BETWEEN 45 AND 79
               AND o.conv_norm <> 'AT_THE_END'
               AND o.out_rub < 1500000
               AND ABS(o.rate_con - 0.1430) <= @eps THEN 1

              WHEN o.termdays BETWEEN 45 AND 79
               AND o.conv_norm <> 'AT_THE_END'
               AND o.out_rub >= 1500000
               AND ABS(o.rate_con - 0.1450) <= @eps THEN 1

              ELSE 0
          END AS new_money_rate_flag

        , CASE 
              WHEN o.termdays BETWEEN 45 AND 79
               AND o.conv_norm = 'AT_THE_END'
               AND o.out_rub >= 1500000
               AND ABS(o.rate_con - 0.1450) <= @eps THEN 1

              WHEN o.termdays BETWEEN 45 AND 79
               AND o.conv_norm = 'AT_THE_END'
               AND o.out_rub < 1500000
               AND ABS(o.rate_con - 0.1430) <= @eps THEN 1

              WHEN o.termdays BETWEEN 45 AND 79
               AND o.conv_norm <> 'AT_THE_END'
               AND o.out_rub < 1500000
               AND ABS(o.rate_con - 0.1410) <= @eps THEN 1

              WHEN o.termdays BETWEEN 45 AND 79
               AND o.conv_norm <> 'AT_THE_END'
               AND o.out_rub >= 1500000
               AND ABS(o.rate_con - 0.1430) <= @eps THEN 1

              ELSE 0
          END AS retention_rate_flag
    FROM opened_by_con o
),

opened_agg AS (
    SELECT
          oc.cli_id

        , COUNT(DISTINCT oc.con_id) AS opened_total_cnt
        , SUM(oc.out_rub) AS opened_total_sum

        , COUNT(DISTINCT CASE WHEN oc.new_money_rate_flag = 1 THEN oc.con_id END) AS opened_new_money_cnt
        , SUM(CASE WHEN oc.new_money_rate_flag = 1 THEN oc.out_rub ELSE 0 END) AS opened_new_money_sum

        , COUNT(DISTINCT CASE WHEN oc.retention_rate_flag = 1 THEN oc.con_id END) AS opened_retention_cnt
        , SUM(CASE WHEN oc.retention_rate_flag = 1 THEN oc.out_rub ELSE 0 END) AS opened_retention_sum

        , COUNT(DISTINCT CASE 
              WHEN oc.new_money_rate_flag = 0 
               AND oc.retention_rate_flag = 0 
              THEN oc.con_id 
          END) AS opened_other_cnt

        , SUM(CASE 
              WHEN oc.new_money_rate_flag = 0 
               AND oc.retention_rate_flag = 0 
              THEN oc.out_rub 
              ELSE 0 
          END) AS opened_other_sum
    FROM opened_classified oc
    GROUP BY
        oc.cli_id
),

sms_flags AS (
    SELECT
          c.cli_id
        , CASE WHEN s.cli_id IS NOT NULL THEN 1 ELSE 0 END AS sms_promo_flag
    FROM #client_scope c
    LEFT JOIN #sms_clients s
        ON s.cli_id = c.cli_id
)

SELECT
      c.cli_id

    -- сегмент и смс
    , seg.segment_flag
    , sms.sms_promo_flag

    -- флаги наличия событий
    , CASE WHEN ISNULL(ex.exit_dep_cnt, 0) > 0 THEN 1 ELSE 0 END AS had_exit_dep_flag
    , CASE WHEN ISNULL(op.opened_total_cnt, 0) > 0 THEN 1 ELSE 0 END AS had_opened_dep_flag

    -- вклады к выходу
    , ISNULL(ex.exit_dep_cnt, 0) AS exit_dep_cnt
    , ISNULL(ex.exit_dep_sum, 0) AS exit_dep_sum
    , ISNULL(ex.exit_dep_sum, 0) / 1000000.0 AS exit_dep_sum_mln

    -- открытия всего
    , ISNULL(op.opened_total_cnt, 0) AS opened_total_cnt
    , ISNULL(op.opened_total_sum, 0) AS opened_total_sum
    , ISNULL(op.opened_total_sum, 0) / 1000000.0 AS opened_total_sum_mln

    -- открытия по ставке новых денег
    , ISNULL(op.opened_new_money_cnt, 0) AS opened_new_money_cnt
    , ISNULL(op.opened_new_money_sum, 0) AS opened_new_money_sum
    , ISNULL(op.opened_new_money_sum, 0) / 1000000.0 AS opened_new_money_sum_mln

    -- открытия по ставке удержания
    , ISNULL(op.opened_retention_cnt, 0) AS opened_retention_cnt
    , ISNULL(op.opened_retention_sum, 0) AS opened_retention_sum
    , ISNULL(op.opened_retention_sum, 0) / 1000000.0 AS opened_retention_sum_mln

    -- открытия по остальным ставкам
    , ISNULL(op.opened_other_cnt, 0) AS opened_other_cnt
    , ISNULL(op.opened_other_sum, 0) AS opened_other_sum
    , ISNULL(op.opened_other_sum, 0) / 1000000.0 AS opened_other_sum_mln

    -- балансы на 29.04 и 13.05
    , ISNULL(bb.td_base_sum, 0) AS td_base_sum
    , ISNULL(be.td_end_sum, 0) AS td_end_sum
    , ISNULL(be.td_end_sum, 0) - ISNULL(bb.td_base_sum, 0) AS td_delta

    , ISNULL(bb.ns_base_sum, 0) AS ns_base_sum
    , ISNULL(be.ns_end_sum, 0) AS ns_end_sum
    , ISNULL(be.ns_end_sum, 0) - ISNULL(bb.ns_base_sum, 0) AS ns_delta

    , ISNULL(bb.total_base_sum, 0) AS total_base_sum
    , ISNULL(be.total_end_sum, 0) AS total_end_sum
    , ISNULL(be.total_end_sum, 0) - ISNULL(bb.total_base_sum, 0) AS total_delta

    -- флаги по НС
    , CASE 
          WHEN ISNULL(be.ns_end_sum, 0) < ISNULL(bb.ns_base_sum, 0) THEN 1 
          ELSE 0 
      END AS ns_decrease_flag

    , CASE 
          WHEN ISNULL(be.ns_end_sum, 0) > ISNULL(bb.ns_base_sum, 0) THEN 1 
          ELSE 0 
      END AS ns_increase_flag

    , CASE 
          WHEN ISNULL(be.ns_end_sum, 0) = ISNULL(bb.ns_base_sum, 0) THEN 1 
          ELSE 0 
      END AS ns_unchanged_flag

    , CASE 
          WHEN ISNULL(bb.ns_base_sum, 0) = 0
           AND ISNULL(be.ns_end_sum, 0) = 0 THEN 1 
          ELSE 0 
      END AS had_no_ns_flag

INTO #client_mart
FROM #client_scope c
LEFT JOIN client_segment seg
    ON seg.cli_id = c.cli_id
LEFT JOIN exit_deposits ex
    ON ex.cli_id = c.cli_id
LEFT JOIN opened_agg op
    ON op.cli_id = c.cli_id
LEFT JOIN balance_base bb
    ON bb.cli_id = c.cli_id
LEFT JOIN balance_end be
    ON be.cli_id = c.cli_id
LEFT JOIN sms_flags sms
    ON sms.cli_id = c.cli_id;


SELECT
      cli_id
    , segment_flag
    , sms_promo_flag

    , had_exit_dep_flag
    , had_opened_dep_flag

    , exit_dep_cnt
    , exit_dep_sum
    , exit_dep_sum_mln

    , opened_total_cnt
    , opened_total_sum
    , opened_total_sum_mln

    , opened_new_money_cnt
    , opened_new_money_sum
    , opened_new_money_sum_mln

    , opened_retention_cnt
    , opened_retention_sum
    , opened_retention_sum_mln

    , opened_other_cnt
    , opened_other_sum
    , opened_other_sum_mln

    , td_base_sum
    , td_end_sum
    , td_delta

    , ns_base_sum
    , ns_end_sum
    , ns_delta

    , total_base_sum
    , total_end_sum
    , total_delta

    , ns_decrease_flag
    , ns_increase_flag
    , ns_unchanged_flag
    , had_no_ns_flag
FROM #client_mart
ORDER BY
      segment_flag
    , sms_promo_flag DESC
    , had_exit_dep_flag DESC
    , had_opened_dep_flag DESC
    , opened_total_sum DESC
    , exit_dep_sum DESC
    , cli_id;
