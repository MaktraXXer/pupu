Да, тут лучше после финальной витрины сохранить результат в #client_mart, а потом сделать 3 отдельных SELECT по платежам через join к нужной группе клиентов.

Ниже блок, который нужно поставить вместо твоего финального SELECT. Он:

1. сохраняет витрину в #client_mart;
2. выводит итоговую витрину;
3. делает 3 выгрузки платежей:
    * все розничные клиенты с СМС;
    * розничные клиенты с СМС и вкладами к выходу;
    * розничные клиенты с СМС без вкладов к выходу.

IF OBJECT_ID('tempdb..#client_mart') IS NOT NULL DROP TABLE #client_mart;
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
-- 10. Сама клиентская витрина
SELECT
      cli_id
    , segment_flag
    , sms_promo_flag
    , had_exit_dep_flag
    , had_opened_dep_flag
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
    , only_sms_client_flag DESC
    , had_exit_dep_flag DESC
    , had_opened_dep_flag DESC
    , had_ns_delta_flag DESC
    , opened_total_sum DESC
    , exit_dep_sum DESC
    , ns_delta ASC
    , cli_id;
-- 11. Проверка разбиения СМС-клиентов розницы
SELECT
      CASE
          WHEN had_exit_dep_flag = 1 THEN N'02. СМС + были вклады к выходу'
          WHEN had_exit_dep_flag = 0 THEN N'03. СМС + не было вкладов к выходу'
      END AS sms_client_group
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
    segment_flag <> N'ДЧБО'
    AND sms_promo_flag = 1
GROUP BY
    CASE
        WHEN had_exit_dep_flag = 1 THEN N'02. СМС + были вклады к выходу'
        WHEN had_exit_dep_flag = 0 THEN N'03. СМС + не было вкладов к выходу'
    END
UNION ALL
SELECT
      N'01. Все СМС-клиенты розницы' AS sms_client_group
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
    segment_flag <> N'ДЧБО'
    AND sms_promo_flag = 1
ORDER BY
    sms_client_group;
-- 12. Платежи: все розничные клиенты, получившие СМС
SELECT
      N'01. Все СМС-клиенты розницы' AS client_group
    , m.cli_id
    , m.had_exit_dep_flag
    , m.had_opened_dep_flag
    , m.had_ns_delta_flag
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
      m.cli_id
    , tr.dt_rep;
-- 13. Платежи: СМС-клиенты розницы, у кого были вклады к выходу
SELECT
      N'02. СМС + были вклады к выходу' AS client_group
    , m.cli_id
    , m.had_exit_dep_flag
    , m.had_opened_dep_flag
    , m.had_ns_delta_flag
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
    AND m.had_exit_dep_flag = 1
    AND tr.dt_rep >= @OpenFrom
    AND tr.dt_rep <= @OpenTo
    AND tr.transaction_type <> N'Внутренние переводы'
    AND tr.transit_max_id IS NULL
ORDER BY
      m.cli_id
    , tr.dt_rep;
-- 14. Платежи: СМС-клиенты розницы, у кого НЕ было вкладов к выходу
SELECT
      N'03. СМС + не было вкладов к выходу' AS client_group
    , m.cli_id
    , m.had_exit_dep_flag
    , m.had_opened_dep_flag
    , m.had_ns_delta_flag
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
    AND m.had_exit_dep_flag = 0
    AND tr.dt_rep >= @OpenFrom
    AND tr.dt_rep <= @OpenTo
    AND tr.transaction_type <> N'Внутренние переводы'
    AND tr.transit_max_id IS NULL
ORDER BY
      m.cli_id
    , tr.dt_rep;

Смысл групп:

01. Все СМС-клиенты розницы
= segment_flag <> 'ДЧБО' AND sms_promo_flag = 1
02. СМС + были вклады к выходу
= segment_flag <> 'ДЧБО' AND sms_promo_flag = 1 AND had_exit_dep_flag = 1
03. СМС + не было вкладов к выходу
= segment_flag <> 'ДЧБО' AND sms_promo_flag = 1 AND had_exit_dep_flag = 0

Для платежей я оставил период @OpenFrom–@OpenTo, то есть 2026-04-30–2026-05-17, и фильтры:

transaction_type <> N'Внутренние переводы'
AND transit_max_id IS NULL

как в твоём примере.
