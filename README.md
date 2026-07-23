USE [ALM];
SET NOCOUNT ON;

DECLARE @DtRep date = '2026-06-30';

IF OBJECT_ID('tempdb..#dchbo_clients') IS NOT NULL
    DROP TABLE #dchbo_clients;

IF OBJECT_ID('tempdb..#dchbo_deposits') IS NOT NULL
    DROP TABLE #dchbo_deposits;


/* ============================================================
   1. Клиенты, у которых хотя бы один срочный вклад
      на дату отчёта отмечен как ДЧБО
   ============================================================ */
SELECT DISTINCT
      CAST(t.cli_id AS bigint) AS cli_id
INTO #dchbo_clients
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE
    t.dt_rep       = @DtRep
    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.acc_role     = N'LIAB'
    AND t.od_flag      = 1
    AND t.cur          = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0
    AND t.TSEGMENTNAME = N'ДЧБО';


/* ============================================================
   2. Все срочные вклады найденных ДЧБО-клиентов

      Здесь TSEGMENTNAME — маркировка конкретного вклада.
      Поэтому у ДЧБО-клиента отдельный вклад может быть отмечен
      как "Розничный бизнес".
   ============================================================ */
SELECT
      CAST(t.cli_id AS bigint) AS cli_id
    , CAST(t.con_id AS bigint) AS con_id
    , CAST(t.dt_open AS date) AS dt_open
    , CAST(t.dt_close_plan AS date) AS dt_close_plan
    , t.is_floatrate
    , t.TSEGMENTNAME
    , t.PROD_NAME_res
    , CAST(t.out_rub AS decimal(38,6)) AS out_rub
    , CAST(1 AS int) AS is_dchbo_client
INTO #dchbo_deposits
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
INNER JOIN #dchbo_clients c
    ON c.cli_id = CAST(t.cli_id AS bigint)
WHERE
    t.dt_rep       = @DtRep
    AND t.section_name = N'Срочные'
    AND t.block_name   = N'Привлечение ФЛ'
    AND t.acc_role     = N'LIAB'
    AND t.od_flag      = 1
    AND t.cur          = '810'
    AND t.out_rub IS NOT NULL
    AND t.out_rub >= 0;


/* ============================================================
   3. Детальная витрина вкладов ДЧБО-клиентов
   ============================================================ */
SELECT
      cli_id
    , con_id
    , dt_open
    , dt_close_plan
    , is_floatrate
    , TSEGMENTNAME
    , PROD_NAME_res
    , out_rub
    , is_dchbo_client
FROM #dchbo_deposits
ORDER BY
      cli_id
    , dt_open
    , con_id;


/* ============================================================
   4. Сколько вкладов ДЧБО-клиентов отмечены
      как "Розничный бизнес"
   ============================================================ */
SELECT
      COUNT(*) AS retail_deposit_count
    , COUNT(DISTINCT cli_id) AS clients_with_retail_deposits_count
    , SUM(out_rub) AS retail_deposit_volume
FROM #dchbo_deposits
WHERE TSEGMENTNAME = N'Розничный бизнес';


/* ============================================================
   5. Полное распределение вкладов ДЧБО-клиентов
      по фактической маркировке TSEGMENTNAME
   ============================================================ */
SELECT
      ISNULL(TSEGMENTNAME, N'Не указано') AS TSEGMENTNAME
    , COUNT(*) AS deposit_count
    , COUNT(DISTINCT cli_id) AS client_count
    , SUM(out_rub) AS deposit_volume
FROM #dchbo_deposits
GROUP BY
    ISNULL(TSEGMENTNAME, N'Не указано')
ORDER BY
    deposit_volume DESC;
