Ниже даю целиком.

1) Переписывание таблицы

Если таблица уже существует в старом виде, добавляем поле сегмента и меняем PK.

USE [ALM_TEST];
GO

IF COL_LENGTH('alm_report.DepositWeekRollStats', 'segment_name') IS NULL
BEGIN
    ALTER TABLE alm_report.DepositWeekRollStats
    ADD segment_name nvarchar(50) NOT NULL
        CONSTRAINT DF_DepositWeekRollStats_segment_name DEFAULT (N'Все клиенты');
END
GO

IF EXISTS (
    SELECT 1
    FROM sys.key_constraints
    WHERE name = N'PK_DepositWeekRollStats'
      AND parent_object_id = OBJECT_ID(N'alm_report.DepositWeekRollStats')
)
BEGIN
    ALTER TABLE alm_report.DepositWeekRollStats
    DROP CONSTRAINT PK_DepositWeekRollStats;
END
GO

ALTER TABLE alm_report.DepositWeekRollStats
ADD CONSTRAINT PK_DepositWeekRollStats
PRIMARY KEY CLUSTERED
(
      week_start_monday
    , week_end_monday
    , segment_name
);
GO

Если хочешь пересоздать таблицу с нуля, то вот полный DDL:

USE [ALM_TEST];
GO

IF OBJECT_ID(N'alm_report.DepositWeekRollStats', N'U') IS NOT NULL
    DROP TABLE alm_report.DepositWeekRollStats;
GO

CREATE TABLE [alm_report].[DepositWeekRollStats](
      [week_start_monday]       [date] NOT NULL
    , [week_end_monday]         [date] NOT NULL
    , [week_from_tuesday]       [date] NOT NULL
    , [week_to_monday]          [date] NOT NULL
    ,  NOT NULL
    , [cnt_cli_scope]           [bigint] NULL
    , [cnt_con_exit]            [bigint] NULL
    , [vol_exit_deposits_rub]   [decimal](38, 6) NULL
    , [wavg_con_rate_exit]      [decimal](18, 6) NULL
    , [cnt_con_open]            [bigint] NULL
    , [vol_opened_deposits_rub] [decimal](38, 6) NULL
    , [wavg_con_rate_open]      [decimal](18, 6) NULL
    , [ns_balance_start_rub]    [decimal](38, 6) NULL
    , [ns_balance_end_rub]      [decimal](38, 6) NULL
    , [ns_balance_delta_rub]    [decimal](38, 6) NULL
    , CONSTRAINT [PK_DepositWeekRollStats] PRIMARY KEY CLUSTERED
      (
            [week_start_monday] ASC
          , [week_end_monday]   ASC
          , [segment_name]      ASC
      )
);
GO


⸻

2) Переписывание процедуры

Логика сегментации тут такая:
	•	сначала находим cli_id из плановых выходов недели;
	•	дальше для этих клиентов смотрим все срочные вклады на стартовый понедельник;
	•	если есть хотя бы один TSEGMENTNAME = N'ДЧБО', клиент = ДЧБО;
	•	иначе Розничный бизнес;
	•	НС не придумываем заново: считаем по тем же клиентам из соответствующего сегмента.

USE [ALM_TEST];
GO

CREATE OR ALTER PROCEDURE [alm_report].[prc_DepositWeekRollStats_Sliding]
(
      @DateFrom   date
    , @DateTo     date
    , @SaveResult bit = 0
)
AS
BEGIN
    SET NOCOUNT ON;

    IF (DATEDIFF(day, '19000101', @DateFrom) % 7) <> 0
    BEGIN
        RAISERROR(N'@DateFrom должен быть понедельником.', 16, 1);
        RETURN;
    END;

    IF (DATEDIFF(day, '19000101', @DateTo) % 7) <> 0
    BEGIN
        RAISERROR(N'@DateTo должен быть понедельником.', 16, 1);
        RETURN;
    END;

    DECLARE @ReliableDate date = DATEADD(day, -2, CAST(GETDATE() AS date));

    DECLARE @MaxValidMonday date =
        DATEADD(day, -(DATEDIFF(day, '19000101', @ReliableDate) % 7), @ReliableDate);

    DECLARE @EffectiveDateTo date =
        CASE WHEN @DateTo > @MaxValidMonday THEN @MaxValidMonday ELSE @DateTo END;

    IF @DateFrom >= @EffectiveDateTo
    BEGIN
        RAISERROR(N'После применения правила GETDATE()-2 диапазон пустой.', 16, 1);
        RETURN;
    END;

    IF OBJECT_ID('tempdb..#bal_prev')         IS NOT NULL DROP TABLE #bal_prev;
    IF OBJECT_ID('tempdb..#bal_curr')         IS NOT NULL DROP TABLE #bal_curr;
    IF OBJECT_ID('tempdb..#bal_swap')         IS NOT NULL DROP TABLE #bal_swap;
    IF OBJECT_ID('tempdb..#clients_scope')    IS NOT NULL DROP TABLE #clients_scope;
    IF OBJECT_ID('tempdb..#client_segment')   IS NOT NULL DROP TABLE #client_segment;
    IF OBJECT_ID('tempdb..#result')           IS NOT NULL DROP TABLE #result;

    CREATE TABLE #bal_prev
    (
          dt_rep         date             NOT NULL
        , cli_id         bigint           NOT NULL
        , con_id         bigint           NOT NULL
        , dt_open        date             NULL
        , dt_close_plan  date             NULL
        , section_name   nvarchar(50)     NOT NULL
        , out_rub        decimal(38, 6)   NOT NULL
        , rate_con       decimal(18, 6)   NULL
        , is_floatrate   bit              NOT NULL
        , PROD_NAME_res  nvarchar(255)    NULL
        , TSEGMENTNAME   nvarchar(255)    NULL
    );

    CREATE TABLE #bal_curr
    (
          dt_rep         date             NOT NULL
        , cli_id         bigint           NOT NULL
        , con_id         bigint           NOT NULL
        , dt_open        date             NULL
        , dt_close_plan  date             NULL
        , section_name   nvarchar(50)     NOT NULL
        , out_rub        decimal(38, 6)   NOT NULL
        , rate_con       decimal(18, 6)   NULL
        , is_floatrate   bit              NOT NULL
        , PROD_NAME_res  nvarchar(255)    NULL
        , TSEGMENTNAME   nvarchar(255)    NULL
    );

    CREATE TABLE #bal_swap
    (
          dt_rep         date             NOT NULL
        , cli_id         bigint           NOT NULL
        , con_id         bigint           NOT NULL
        , dt_open        date             NULL
        , dt_close_plan  date             NULL
        , section_name   nvarchar(50)     NOT NULL
        , out_rub        decimal(38, 6)   NOT NULL
        , rate_con       decimal(18, 6)   NULL
        , is_floatrate   bit              NOT NULL
        , PROD_NAME_res  nvarchar(255)    NULL
        , TSEGMENTNAME   nvarchar(255)    NULL
    );

    CREATE CLUSTERED INDEX CIX_#bal_prev
        ON #bal_prev (dt_rep, section_name, cli_id, con_id);

    CREATE NONCLUSTERED INDEX IX_#bal_prev_exit
        ON #bal_prev (dt_rep, section_name, dt_close_plan, cli_id)
        INCLUDE (con_id, out_rub, rate_con, dt_open, TSEGMENTNAME);

    CREATE NONCLUSTERED INDEX IX_#bal_prev_open
        ON #bal_prev (dt_rep, section_name, dt_open, cli_id)
        INCLUDE (con_id, out_rub, rate_con, dt_close_plan, TSEGMENTNAME);

    CREATE NONCLUSTERED INDEX IX_#bal_prev_ns
        ON #bal_prev (section_name, dt_rep, cli_id)
        INCLUDE (out_rub);

    CREATE CLUSTERED INDEX CIX_#bal_curr
        ON #bal_curr (dt_rep, section_name, cli_id, con_id);

    CREATE NONCLUSTERED INDEX IX_#bal_curr_exit
        ON #bal_curr (dt_rep, section_name, dt_close_plan, cli_id)
        INCLUDE (con_id, out_rub, rate_con, dt_open, TSEGMENTNAME);

    CREATE NONCLUSTERED INDEX IX_#bal_curr_open
        ON #bal_curr (dt_rep, section_name, dt_open, cli_id)
        INCLUDE (con_id, out_rub, rate_con, dt_close_plan, TSEGMENTNAME);

    CREATE NONCLUSTERED INDEX IX_#bal_curr_ns
        ON #bal_curr (section_name, dt_rep, cli_id)
        INCLUDE (out_rub);

    CREATE TABLE #clients_scope
    (
        cli_id bigint NOT NULL
    );
    CREATE UNIQUE CLUSTERED INDEX CIX_#clients_scope
        ON #clients_scope(cli_id);

    CREATE TABLE #client_segment
    (
          cli_id       bigint       NOT NULL
        , segment_name nvarchar(50) NOT NULL
    );
    CREATE UNIQUE CLUSTERED INDEX CIX_#client_segment
        ON #client_segment(cli_id);

    CREATE TABLE #result
    (
          week_start_monday       date            NOT NULL
        , week_end_monday         date            NOT NULL
        , week_from_tuesday       date            NOT NULL
        , week_to_monday          date            NOT NULL
        , segment_name            nvarchar(50)    NOT NULL
        , cnt_cli_scope           bigint          NULL
        , cnt_con_exit            bigint          NULL
        , vol_exit_deposits_rub   decimal(38, 6) NULL
        , wavg_con_rate_exit      decimal(18, 6) NULL
        , cnt_con_open            bigint          NULL
        , vol_opened_deposits_rub decimal(38, 6) NULL
        , wavg_con_rate_open      decimal(18, 6) NULL
        , ns_balance_start_rub    decimal(38, 6) NULL
        , ns_balance_end_rub      decimal(38, 6) NULL
        , ns_balance_delta_rub    decimal(38, 6) NULL
    );

    DECLARE @CurMondayStart date = @DateFrom;
    DECLARE @CurMondayEnd   date = DATEADD(day, 7, @DateFrom);

    IF @CurMondayEnd > @EffectiveDateTo
    BEGIN
        RAISERROR(N'В диапазоне нет ни одной полной пары понедельников.', 16, 1);
        RETURN;
    END;

    INSERT INTO #bal_prev
    (
          dt_rep, cli_id, con_id, dt_open, dt_close_plan, section_name,
          out_rub, rate_con, is_floatrate, PROD_NAME_res, TSEGMENTNAME
    )
    SELECT
          CAST(t.dt_rep AS date)
        , CAST(t.cli_id AS bigint)
        , CAST(t.con_id AS bigint)
        , CAST(t.dt_open AS date)
        , CAST(t.dt_close_plan AS date)
        , CAST(t.section_name AS nvarchar(50))
        , CAST(t.out_rub AS decimal(38, 6))
        , CAST(t.rate_con AS decimal(18, 6))
        , CAST(ISNULL(t.is_floatrate, 0) AS bit)
        , CAST(t.PROD_NAME_res AS nvarchar(255))
        , CAST(t.TSEGMENTNAME AS nvarchar(255))
    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
    WHERE t.dt_rep = @CurMondayStart
      AND t.section_name IN (N'Срочные', N'Накопительный счёт')
      AND t.block_name = N'Привлечение ФЛ'
      AND t.acc_role   = N'LIAB'
      AND t.cur        = '810'
      AND t.od_flag    = 1
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0;

    INSERT INTO #bal_curr
    (
          dt_rep, cli_id, con_id, dt_open, dt_close_plan, section_name,
          out_rub, rate_con, is_floatrate, PROD_NAME_res, TSEGMENTNAME
    )
    SELECT
          CAST(t.dt_rep AS date)
        , CAST(t.cli_id AS bigint)
        , CAST(t.con_id AS bigint)
        , CAST(t.dt_open AS date)
        , CAST(t.dt_close_plan AS date)
        , CAST(t.section_name AS nvarchar(50))
        , CAST(t.out_rub AS decimal(38, 6))
        , CAST(t.rate_con AS decimal(18, 6))
        , CAST(ISNULL(t.is_floatrate, 0) AS bit)
        , CAST(t.PROD_NAME_res AS nvarchar(255))
        , CAST(t.TSEGMENTNAME AS nvarchar(255))
    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
    WHERE t.dt_rep = @CurMondayEnd
      AND t.section_name IN (N'Срочные', N'Накопительный счёт')
      AND t.block_name = N'Привлечение ФЛ'
      AND t.acc_role   = N'LIAB'
      AND t.cur        = '810'
      AND t.od_flag    = 1
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0;

    WHILE @CurMondayEnd <= @EffectiveDateTo
    BEGIN
        DECLARE @WeekFrom date = DATEADD(day, 1, @CurMondayStart);
        DECLARE @WeekTo   date = @CurMondayEnd;

        TRUNCATE TABLE #clients_scope;
        TRUNCATE TABLE #client_segment;

        INSERT INTO #clients_scope (cli_id)
        SELECT DISTINCT
            b.cli_id
        FROM #bal_prev b
        WHERE b.section_name = N'Срочные'
          AND b.dt_close_plan >= @WeekFrom
          AND b.dt_close_plan <= @WeekTo;

        INSERT INTO #client_segment (cli_id, segment_name)
        SELECT
            c.cli_id,
            CASE
                WHEN EXISTS
                (
                    SELECT 1
                    FROM #bal_prev bp
                    WHERE bp.cli_id = c.cli_id
                      AND bp.section_name = N'Срочные'
                      AND bp.TSEGMENTNAME = N'ДЧБО'
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
            FROM #bal_prev b
            WHERE b.section_name = N'Срочные'
              AND b.dt_close_plan >= @WeekFrom
              AND b.dt_close_plan <= @WeekTo

            UNION ALL

            SELECT
                  cs.segment_name
                , b.cli_id
                , b.con_id
                , b.out_rub
                , b.rate_con
            FROM #bal_prev b
            INNER JOIN #client_segment cs
                ON b.cli_id = cs.cli_id
            WHERE b.section_name = N'Срочные'
              AND b.dt_close_plan >= @WeekFrom
              AND b.dt_close_plan <= @WeekTo
        ),
        opened_deposits AS
        (
            SELECT
                  N'Все клиенты' AS segment_name
                , b.cli_id
                , b.con_id
                , b.out_rub
                , b.rate_con
            FROM #bal_curr b
            INNER JOIN #clients_scope c
                ON b.cli_id = c.cli_id
            WHERE b.section_name = N'Срочные'
              AND b.dt_open >= @WeekFrom
              AND b.dt_open <= @WeekTo

            UNION ALL

            SELECT
                  cs.segment_name
                , b.cli_id
                , b.con_id
                , b.out_rub
                , b.rate_con
            FROM #bal_curr b
            INNER JOIN #client_segment cs
                ON b.cli_id = cs.cli_id
            WHERE b.section_name = N'Срочные'
              AND b.dt_open >= @WeekFrom
              AND b.dt_open <= @WeekTo
        ),
        ns_union AS
        (
            SELECT dt_rep, cli_id, out_rub
            FROM #bal_prev
            WHERE section_name = N'Накопительный счёт'

            UNION ALL

            SELECT dt_rep, cli_id, out_rub
            FROM #bal_curr
            WHERE section_name = N'Накопительный счёт'
        ),
        ns_by_date AS
        (
            SELECT
                  N'Все клиенты' AS segment_name
                , n.dt_rep
                , n.cli_id
                , SUM(n.out_rub) AS ns_out_rub
            FROM ns_union n
            INNER JOIN #clients_scope c
                ON n.cli_id = c.cli_id
            GROUP BY n.dt_rep, n.cli_id

            UNION ALL

            SELECT
                  cs.segment_name
                , n.dt_rep
                , n.cli_id
                , SUM(n.out_rub) AS ns_out_rub
            FROM ns_union n
            INNER JOIN #client_segment cs
                ON n.cli_id = cs.cli_id
            GROUP BY cs.segment_name, n.dt_rep, n.cli_id
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
                    AS decimal(18, 6)
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
                    AS decimal(18, 6)
                  ) AS wavg_rate_open
            FROM opened_deposits
            GROUP BY segment_name
        ),
        agg_ns AS
        (
            SELECT
                  N'Все клиенты' AS segment_name
                , COUNT(DISTINCT c.cli_id) AS cnt_cli_scope
                , SUM(CASE WHEN n.dt_rep = @CurMondayStart THEN n.ns_out_rub ELSE 0 END) AS ns_start_vol
                , SUM(CASE WHEN n.dt_rep = @CurMondayEnd   THEN n.ns_out_rub ELSE 0 END) AS ns_end_vol
            FROM #clients_scope c
            LEFT JOIN ns_by_date n
                ON n.cli_id = c.cli_id
               AND n.segment_name = N'Все клиенты'

            UNION ALL

            SELECT
                  cs.segment_name
                , COUNT(DISTINCT cs.cli_id) AS cnt_cli_scope
                , SUM(CASE WHEN n.dt_rep = @CurMondayStart THEN n.ns_out_rub ELSE 0 END) AS ns_start_vol
                , SUM(CASE WHEN n.dt_rep = @CurMondayEnd   THEN n.ns_out_rub ELSE 0 END) AS ns_end_vol
            FROM #client_segment cs
            LEFT JOIN ns_by_date n
                ON n.cli_id = cs.cli_id
               AND n.segment_name = cs.segment_name
            GROUP BY cs.segment_name
        )
        INSERT INTO #result
        (
              week_start_monday
            , week_end_monday
            , week_from_tuesday
            , week_to_monday
            , segment_name
            , cnt_cli_scope
            , cnt_con_exit
            , vol_exit_deposits_rub
            , wavg_con_rate_exit
            , cnt_con_open
            , vol_opened_deposits_rub
            , wavg_con_rate_open
            , ns_balance_start_rub
            , ns_balance_end_rub
            , ns_balance_delta_rub
        )
        SELECT
              @CurMondayStart
            , @CurMondayEnd
            , @WeekFrom
            , @WeekTo
            , s.segment_name
            , ISNULL(ns.cnt_cli_scope, 0)
            , ISNULL(ex.cnt_con_exit, 0)
            , ISNULL(ex.vol_exit, 0)
            , ex.wavg_rate_exit
            , ISNULL(op.cnt_con_open, 0)
            , ISNULL(op.vol_open, 0)
            , op.wavg_rate_open
            , ISNULL(ns.ns_start_vol, 0)
            , ISNULL(ns.ns_end_vol, 0)
            , ISNULL(ns.ns_end_vol, 0) - ISNULL(ns.ns_start_vol, 0)
        FROM seg s
        LEFT JOIN agg_exit ex
            ON s.segment_name = ex.segment_name
        LEFT JOIN agg_open op
            ON s.segment_name = op.segment_name
        LEFT JOIN agg_ns ns
            ON s.segment_name = ns.segment_name;

        IF DATEADD(day, 7, @CurMondayEnd) <= @EffectiveDateTo
        BEGIN
            DECLARE @NextMonday date = DATEADD(day, 7, @CurMondayEnd);

            TRUNCATE TABLE #bal_swap;

            INSERT INTO #bal_swap
            (
                  dt_rep, cli_id, con_id, dt_open, dt_close_plan, section_name,
                  out_rub, rate_con, is_floatrate, PROD_NAME_res, TSEGMENTNAME
            )
            SELECT
                  dt_rep, cli_id, con_id, dt_open, dt_close_plan, section_name,
                  out_rub, rate_con, is_floatrate, PROD_NAME_res, TSEGMENTNAME
            FROM #bal_curr;

            TRUNCATE TABLE #bal_prev;

            INSERT INTO #bal_prev
            (
                  dt_rep, cli_id, con_id, dt_open, dt_close_plan, section_name,
                  out_rub, rate_con, is_floatrate, PROD_NAME_res, TSEGMENTNAME
            )
            SELECT
                  dt_rep, cli_id, con_id, dt_open, dt_close_plan, section_name,
                  out_rub, rate_con, is_floatrate, PROD_NAME_res, TSEGMENTNAME
            FROM #bal_swap;

            TRUNCATE TABLE #bal_curr;

            INSERT INTO #bal_curr
            (
                  dt_rep, cli_id, con_id, dt_open, dt_close_plan, section_name,
                  out_rub, rate_con, is_floatrate, PROD_NAME_res, TSEGMENTNAME
            )
            SELECT
                  CAST(t.dt_rep AS date)
                , CAST(t.cli_id AS bigint)
                , CAST(t.con_id AS bigint)
                , CAST(t.dt_open AS date)
                , CAST(t.dt_close_plan AS date)
                , CAST(t.section_name AS nvarchar(50))
                , CAST(t.out_rub AS decimal(38, 6))
                , CAST(t.rate_con AS decimal(18, 6))
                , CAST(ISNULL(t.is_floatrate, 0) AS bit)
                , CAST(t.PROD_NAME_res AS nvarchar(255))
                , CAST(t.TSEGMENTNAME AS nvarchar(255))
            FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
            WHERE t.dt_rep = @NextMonday
              AND t.section_name IN (N'Срочные', N'Накопительный счёт')
              AND t.block_name = N'Привлечение ФЛ'
              AND t.acc_role   = N'LIAB'
              AND t.cur        = '810'
              AND t.od_flag    = 1
              AND t.out_rub IS NOT NULL
              AND t.out_rub >= 0;
        END;

        SET @CurMondayStart = DATEADD(day, 7, @CurMondayStart);
        SET @CurMondayEnd   = DATEADD(day, 7, @CurMondayEnd);
    END;

    IF @SaveResult = 1
    BEGIN
        DELETE
        FROM ALM_REPORT.DepositWeekRollStats
        WHERE week_start_monday >= @DateFrom
          AND week_end_monday   <= @EffectiveDateTo;

        INSERT INTO ALM_REPORT.DepositWeekRollStats
        (
              week_start_monday
            , week_end_monday
            , week_from_tuesday
            , week_to_monday
            , segment_name
            , cnt_cli_scope
            , cnt_con_exit
            , vol_exit_deposits_rub
            , wavg_con_rate_exit
            , cnt_con_open
            , vol_opened_deposits_rub
            , wavg_con_rate_open
            , ns_balance_start_rub
            , ns_balance_end_rub
            , ns_balance_delta_rub
        )
        SELECT
              week_start_monday
            , week_end_monday
            , week_from_tuesday
            , week_to_monday
            , segment_name
            , cnt_cli_scope
            , cnt_con_exit
            , vol_exit_deposits_rub
            , wavg_con_rate_exit
            , cnt_con_open
            , vol_opened_deposits_rub
            , wavg_con_rate_open
            , ns_balance_start_rub
            , ns_balance_end_rub
            , ns_balance_delta_rub
        FROM #result;
    END;

    SELECT
          week_start_monday
        , week_end_monday
        , week_from_tuesday
        , week_to_monday
        , segment_name
        , cnt_cli_scope
        , cnt_con_exit
        , vol_exit_deposits_rub
        , wavg_con_rate_exit
        , cnt_con_open
        , vol_opened_deposits_rub
        , wavg_con_rate_open
        , ns_balance_start_rub
        , ns_balance_end_rub
        , ns_balance_delta_rub
    FROM #result
    ORDER BY week_start_monday,
             week_end_monday,
             CASE
                 WHEN segment_name = N'Все клиенты' THEN 1
                 WHEN segment_name = N'ДЧБО' THEN 2
                 ELSE 3
             END;
END
GO

Как запускать

EXEC ALM_REPORT.prc_DepositWeekRollStats_Sliding
      @DateFrom   = '2026-03-02',
      @DateTo     = '2026-03-16',
      @SaveResult = 1;

Что получится

На каждую неделю будет 3 строки:
	•	Все клиенты
	•	ДЧБО
	•	Розничный бизнес

Да, именно с этой логикой:
	•	есть хотя бы один срочный продукт клиента с TSEGMENTNAME = N'ДЧБО' на стартовый понедельник недели → весь клиент ДЧБО;
	•	иначе весь клиент Розничный бизнес.
