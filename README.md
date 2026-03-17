USE [ALM];
SET NOCOUNT ON;

/* ============================================
   ПАРАМЕТРЫ
   ============================================ */
DECLARE @MondayStart date = '2026-03-09';   -- обязательно понедельник
DECLARE @FactEndDate date = '2026-03-14';   -- фактическая дата конца неполной недели

/* ============================================
   ПРОВЕРКИ
   ============================================ */
IF (DATEDIFF(day, '19000101', @MondayStart) % 7) <> 0
BEGIN
    RAISERROR(N'@MondayStart должен быть понедельником.', 16, 1);
    RETURN;
END;

DECLARE @ReliableDate date = DATEADD(day, -2, CAST(GETDATE() AS date));

IF @FactEndDate > @ReliableDate
BEGIN
    RAISERROR(N'@FactEndDate не может быть больше GETDATE()-2.', 16, 1);
    RETURN;
END;

IF @FactEndDate <= @MondayStart
BEGIN
    RAISERROR(N'@FactEndDate должен быть больше @MondayStart.', 16, 1);
    RETURN;
END;

DECLARE @WeekFrom date = DATEADD(day, 1, @MondayStart);  -- вторник
DECLARE @WeekTo   date = @FactEndDate;                   -- фактический конец

IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;
IF OBJECT_ID('tempdb..#clients_scope') IS NOT NULL DROP TABLE #clients_scope;

/* ============================================
   ЗАГРУЗКА ДВУХ СРЕЗОВ БАЛАНСА:
   стартовый понедельник + фактический конец
   ============================================ */
SELECT
      CAST(t.dt_rep AS date)            AS dt_rep
    , CAST(t.cli_id AS bigint)          AS cli_id
    , CAST(t.con_id AS bigint)          AS con_id
    , CAST(t.dt_open AS date)           AS dt_open
    , CAST(t.dt_close_plan AS date)     AS dt_close_plan
    , CAST(t.section_name AS nvarchar(50)) AS section_name
    , CAST(t.out_rub AS decimal(38,6))  AS out_rub
    , CAST(t.rate_con AS decimal(18,6)) AS rate_con
    , CAST(ISNULL(t.is_floatrate,0) AS bit) AS is_floatrate
    , CAST(t.PROD_NAME_res AS nvarchar(255)) AS PROD_NAME_res
    , CAST(t.TSEGMENTNAME AS nvarchar(255))  AS TSEGMENTNAME
INTO #bal
FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
WHERE t.dt_rep IN (@MondayStart, @FactEndDate)
  AND t.section_name IN (N'Срочные', N'Накопительный счёт')
  AND t.block_name = N'Привлечение ФЛ'
  AND t.acc_role   = N'LIAB'
  AND t.cur        = '810'
  AND t.od_flag    = 1
  AND t.out_rub IS NOT NULL
  AND t.out_rub >= 0;

CREATE CLUSTERED INDEX CIX_#bal
    ON #bal (dt_rep, section_name, cli_id, con_id);

CREATE NONCLUSTERED INDEX IX_#bal_exit
    ON #bal (dt_rep, section_name, dt_close_plan, cli_id)
    INCLUDE (con_id, out_rub, rate_con, dt_open);

CREATE NONCLUSTERED INDEX IX_#bal_open
    ON #bal (dt_rep, section_name, dt_open, cli_id)
    INCLUDE (con_id, out_rub, rate_con, dt_close_plan);

CREATE NONCLUSTERED INDEX IX_#bal_ns
    ON #bal (section_name, dt_rep, cli_id)
    INCLUDE (out_rub);

/* ============================================
   КЛИЕНТЫ В СКОУПЕ: есть выходящий вклад
   ============================================ */
SELECT DISTINCT
    b.cli_id
INTO #clients_scope
FROM #bal b
WHERE b.dt_rep = @MondayStart
  AND b.section_name = N'Срочные'
  AND b.dt_close_plan >= @WeekFrom
  AND b.dt_close_plan <= @WeekTo;

CREATE UNIQUE CLUSTERED INDEX CIX_#clients_scope
    ON #clients_scope(cli_id);

/* ============================================
   ИТОГОВЫЙ РАСЧЁТ
   ============================================ */
;WITH deposits_to_exit AS
(
    SELECT
          b.cli_id
        , b.con_id
        , b.out_rub
        , b.rate_con
    FROM #bal b
    WHERE b.dt_rep = @MondayStart
      AND b.section_name = N'Срочные'
      AND b.dt_close_plan >= @WeekFrom
      AND b.dt_close_plan <= @WeekTo
),
opened_deposits AS
(
    SELECT
          b.cli_id
        , b.con_id
        , b.out_rub
        , b.rate_con
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE b.dt_rep = @FactEndDate
      AND b.section_name = N'Срочные'
      AND b.dt_open >= @WeekFrom
      AND b.dt_open <= @WeekTo
),
ns_by_date AS
(
    SELECT
          b.dt_rep
        , b.cli_id
        , SUM(b.out_rub) AS ns_out_rub
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE b.section_name = N'Накопительный счёт'
    GROUP BY
          b.dt_rep
        , b.cli_id
),
agg_exit AS
(
    SELECT
          COUNT(DISTINCT cli_id) AS cnt_cli_exit
        , COUNT(DISTINCT con_id) AS cnt_con_exit
        , SUM(out_rub) AS vol_exit
        , CAST(
            SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub * rate_con END)
            / NULLIF(SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END), 0)
            AS decimal(18,6)
          ) AS wavg_rate_exit
    FROM deposits_to_exit
),
agg_open AS
(
    SELECT
          COUNT(DISTINCT cli_id) AS cnt_cli_open
        , COUNT(DISTINCT con_id) AS cnt_con_open
        , SUM(out_rub) AS vol_open
        , CAST(
            SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub * rate_con END)
            / NULLIF(SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END), 0)
            AS decimal(18,6)
          ) AS wavg_rate_open
    FROM opened_deposits
),
agg_ns AS
(
    SELECT
          COUNT(DISTINCT c.cli_id) AS cnt_cli_scope
        , SUM(CASE WHEN n.dt_rep = @MondayStart THEN n.ns_out_rub ELSE 0 END) AS ns_start_vol
        , SUM(CASE WHEN n.dt_rep = @FactEndDate THEN n.ns_out_rub ELSE 0 END) AS ns_end_vol
    FROM #clients_scope c
    LEFT JOIN ns_by_date n
        ON c.cli_id = n.cli_id
)
SELECT
      @MondayStart AS week_start_monday
    , @FactEndDate AS fact_end_date
    , @WeekFrom AS week_from_tuesday
    , @WeekTo   AS week_to_fact_date

    , ISNULL(ns.cnt_cli_scope, 0) AS cnt_cli_scope

    , ISNULL(ex.cnt_con_exit, 0) AS cnt_con_exit
    , ISNULL(ex.vol_exit, 0)     AS vol_exit_deposits_rub
    , ex.wavg_rate_exit          AS wavg_con_rate_exit

    , ISNULL(op.cnt_con_open, 0) AS cnt_con_open
    , ISNULL(op.vol_open, 0)     AS vol_opened_deposits_rub
    , op.wavg_rate_open          AS wavg_con_rate_open

    , ISNULL(ns.ns_start_vol, 0) AS ns_balance_start_rub
    , ISNULL(ns.ns_end_vol, 0)   AS ns_balance_end_rub
    , ISNULL(ns.ns_end_vol, 0) - ISNULL(ns.ns_start_vol, 0) AS ns_balance_delta_rub
FROM agg_exit ex
CROSS JOIN agg_open op
CROSS JOIN agg_ns ns;

/*
DROP TABLE #clients_scope;
DROP TABLE #bal;
*/
