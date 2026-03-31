USE [ALM];
SET NOCOUNT ON;

DECLARE @StartDate date = '2026-03-23';   -- понедельник / дата стартового баланса
DECLARE @EndDate   date = '2026-03-28';   -- фактическая дата конца периода
DECLARE @FromDate  date = DATEADD(day, 1, @StartDate);  -- 2026-03-24

IF OBJECT_ID('tempdb..#bal') IS NOT NULL DROP TABLE #bal;
IF OBJECT_ID('tempdb..#clients_scope') IS NOT NULL DROP TABLE #clients_scope;
IF OBJECT_ID('tempdb..#client_segment') IS NOT NULL DROP TABLE #client_segment;

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
WHERE t.dt_rep IN (@StartDate, @EndDate)
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
    INCLUDE (con_id, out_rub, rate_con, dt_open, TSEGMENTNAME);

CREATE NONCLUSTERED INDEX IX_#bal_open
    ON #bal (dt_rep, section_name, dt_open, cli_id)
    INCLUDE (con_id, out_rub, rate_con, dt_close_plan, TSEGMENTNAME);

CREATE NONCLUSTERED INDEX IX_#bal_ns
    ON #bal (section_name, dt_rep, cli_id)
    INCLUDE (out_rub);

SELECT DISTINCT
    b.cli_id
INTO #clients_scope
FROM #bal b
WHERE b.dt_rep = @StartDate
  AND b.section_name = N'Срочные'
  AND b.dt_close_plan >= @FromDate
  AND b.dt_close_plan <= @EndDate;

CREATE UNIQUE CLUSTERED INDEX CIX_#clients_scope
    ON #clients_scope(cli_id);

CREATE TABLE #client_segment
(
      cli_id       bigint       NOT NULL
    , segment_name nvarchar(50) NOT NULL
);

CREATE UNIQUE CLUSTERED INDEX CIX_#client_segment
    ON #client_segment(cli_id);

INSERT INTO #client_segment (cli_id, segment_name)
SELECT
    c.cli_id,
    CASE
        WHEN EXISTS
        (
            SELECT 1
            FROM #bal b
            WHERE b.dt_rep = @StartDate
              AND b.cli_id = c.cli_id
              AND b.section_name = N'Срочные'
              AND b.TSEGMENTNAME = N'ДЧБО'
        )
        THEN N'ДЧБО'
        ELSE N'Розничный бизнес'
    END
FROM #clients_scope c;

;WITH seg AS
(
    SELECT N'Все клиенты' AS segment_name
    UNION ALL SELECT N'ДЧБО'
    UNION ALL SELECT N'Розничный бизнес'
),
deposits_to_exit AS
(
    SELECT
          N'Все клиенты' AS segment_name
        , b.cli_id
        , b.con_id
        , b.out_rub
        , b.rate_con
    FROM #bal b
    WHERE b.dt_rep = @StartDate
      AND b.section_name = N'Срочные'
      AND b.dt_close_plan >= @FromDate
      AND b.dt_close_plan <= @EndDate

    UNION ALL

    SELECT
          cs.segment_name
        , b.cli_id
        , b.con_id
        , b.out_rub
        , b.rate_con
    FROM #bal b
    INNER JOIN #client_segment cs
        ON b.cli_id = cs.cli_id
    WHERE b.dt_rep = @StartDate
      AND b.section_name = N'Срочные'
      AND b.dt_close_plan >= @FromDate
      AND b.dt_close_plan <= @EndDate
),
opened_deposits AS
(
    SELECT
          N'Все клиенты' AS segment_name
        , b.cli_id
        , b.con_id
        , b.out_rub
        , b.rate_con
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE b.dt_rep = @EndDate
      AND b.section_name = N'Срочные'
      AND b.dt_open >= @FromDate
      AND b.dt_open <= @EndDate

    UNION ALL

    SELECT
          cs.segment_name
        , b.cli_id
        , b.con_id
        , b.out_rub
        , b.rate_con
    FROM #bal b
    INNER JOIN #client_segment cs
        ON b.cli_id = cs.cli_id
    WHERE b.dt_rep = @EndDate
      AND b.section_name = N'Срочные'
      AND b.dt_open >= @FromDate
      AND b.dt_open <= @EndDate
),
ns_by_date AS
(
    SELECT
          N'Все клиенты' AS segment_name
        , b.dt_rep
        , b.cli_id
        , SUM(b.out_rub) AS ns_out_rub
    FROM #bal b
    INNER JOIN #clients_scope c
        ON b.cli_id = c.cli_id
    WHERE b.section_name = N'Накопительный счёт'
    GROUP BY b.dt_rep, b.cli_id

    UNION ALL

    SELECT
          cs.segment_name
        , b.dt_rep
        , b.cli_id
        , SUM(b.out_rub) AS ns_out_rub
    FROM #bal b
    INNER JOIN #client_segment cs
        ON b.cli_id = cs.cli_id
    WHERE b.section_name = N'Накопительный счёт'
    GROUP BY cs.segment_name, b.dt_rep, b.cli_id
),
agg_exit AS
(
    SELECT
          segment_name
        , COUNT(DISTINCT cli_id) AS cnt_cli_exit
        , COUNT(DISTINCT con_id) AS cnt_con_exit
        , SUM(out_rub) AS vol_exit
        , CAST(
            SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub * rate_con END)
            / NULLIF(SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END), 0)
            AS decimal(18,6)
          ) AS wavg_rate_exit
    FROM deposits_to_exit
    GROUP BY segment_name
),
agg_open AS
(
    SELECT
          segment_name
        , COUNT(DISTINCT cli_id) AS cnt_cli_open
        , COUNT(DISTINCT con_id) AS cnt_con_open
        , SUM(out_rub) AS vol_open
        , CAST(
            SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub * rate_con END)
            / NULLIF(SUM(CASE WHEN rate_con IS NOT NULL THEN out_rub END), 0)
            AS decimal(18,6)
          ) AS wavg_rate_open
    FROM opened_deposits
    GROUP BY segment_name
),
agg_ns AS
(
    SELECT
          N'Все клиенты' AS segment_name
        , COUNT(DISTINCT c.cli_id) AS cnt_cli_scope
        , SUM(CASE WHEN n.dt_rep = @StartDate THEN n.ns_out_rub ELSE 0 END) AS ns_start_vol
        , SUM(CASE WHEN n.dt_rep = @EndDate   THEN n.ns_out_rub ELSE 0 END) AS ns_end_vol
    FROM #clients_scope c
    LEFT JOIN ns_by_date n
        ON n.cli_id = c.cli_id
       AND n.segment_name = N'Все клиенты'

    UNION ALL

    SELECT
          cs.segment_name
        , COUNT(DISTINCT cs.cli_id) AS cnt_cli_scope
        , SUM(CASE WHEN n.dt_rep = @StartDate THEN n.ns_out_rub ELSE 0 END) AS ns_start_vol
        , SUM(CASE WHEN n.dt_rep = @EndDate   THEN n.ns_out_rub ELSE 0 END) AS ns_end_vol
    FROM #client_segment cs
    LEFT JOIN ns_by_date n
        ON n.cli_id = cs.cli_id
       AND n.segment_name = cs.segment_name
    GROUP BY cs.segment_name
)
SELECT
      @StartDate AS week_start_monday
    , @EndDate   AS fact_end_date
    , @FromDate  AS week_from_tuesday
    , @EndDate   AS week_to_fact_date
    , s.segment_name
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
FROM seg s
LEFT JOIN agg_exit ex
    ON s.segment_name = ex.segment_name
LEFT JOIN agg_open op
    ON s.segment_name = op.segment_name
LEFT JOIN agg_ns ns
    ON s.segment_name = ns.segment_name
ORDER BY CASE
             WHEN s.segment_name = N'Все клиенты' THEN 1
             WHEN s.segment_name = N'ДЧБО' THEN 2
             ELSE 3
         END;
