USE [ALM_TEST];
SET NOCOUNT ON;
GO

IF NOT EXISTS (
    SELECT 1
    FROM sys.schemas
    WHERE name = N'ALM_REPORT'
)
BEGIN
    EXEC('CREATE SCHEMA ALM_REPORT');
END
GO

IF OBJECT_ID(N'ALM_REPORT.DepositWeekRollStats', N'U') IS NULL
BEGIN
    CREATE TABLE ALM_REPORT.DepositWeekRollStats
    (
        week_start_monday       date            NOT NULL,
        week_end_monday         date            NOT NULL,
        week_from_tuesday       date            NOT NULL,
        week_to_monday          date            NOT NULL,

        cnt_cli_scope           bigint          NULL,

        cnt_con_exit            bigint          NULL,
        vol_exit_deposits_rub   decimal(38, 6) NULL,
        wavg_con_rate_exit      decimal(18, 6) NULL,

        cnt_con_open            bigint          NULL,
        vol_opened_deposits_rub decimal(38, 6) NULL,
        wavg_con_rate_open      decimal(18, 6) NULL,

        ns_balance_start_rub    decimal(38, 6) NULL,
        ns_balance_end_rub      decimal(38, 6) NULL,
        ns_balance_delta_rub    decimal(38, 6) NULL,

        CONSTRAINT PK_DepositWeekRollStats
            PRIMARY KEY CLUSTERED (week_start_monday, week_end_monday)
    );
END
GO

CREATE OR ALTER PROCEDURE ALM_REPORT.prc_DepositWeekRollStats_Sliding
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

    IF OBJECT_ID('tempdb..#bal_prev')      IS NOT NULL DROP TABLE #bal_prev;
    IF OBJECT_ID('tempdb..#bal_curr')      IS NOT NULL DROP TABLE #bal_curr;
    IF OBJECT_ID('tempdb..#bal_swap')      IS NOT NULL DROP TABLE #bal_swap;
    IF OBJECT_ID('tempdb..#clients_scope') IS NOT NULL DROP TABLE #clients_scope;
    IF OBJECT_ID('tempdb..#result')        IS NOT NULL DROP TABLE #result;

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
        INCLUDE (con_id, out_rub, rate_con, dt_open);
    CREATE NONCLUSTERED INDEX IX_#bal_prev_open
        ON #bal_prev (dt_rep, section_name, dt_open, cli_id)
        INCLUDE (con_id, out_rub, rate_con, dt_close_plan);
    CREATE NONCLUSTERED INDEX IX_#bal_prev_ns
        ON #bal_prev (section_name, dt_rep, cli_id)
        INCLUDE (out_rub);

    CREATE CLUSTERED INDEX CIX_#bal_curr
        ON #bal_curr (dt_rep, section_name, cli_id, con_id);
    CREATE NONCLUSTERED INDEX IX_#bal_curr_exit
        ON #bal_curr (dt_rep, section_name, dt_close_plan, cli_id)
        INCLUDE (con_id, out_rub, rate_con, dt_open);
    CREATE NONCLUSTERED INDEX IX_#bal_curr_open
        ON #bal_curr (dt_rep, section_name, dt_open, cli_id)
        INCLUDE (con_id, out_rub, rate_con, dt_close_plan);
    CREATE NONCLUSTERED INDEX IX_#bal_curr_ns
        ON #bal_curr (section_name, dt_rep, cli_id)
        INCLUDE (out_rub);

    CREATE TABLE #clients_scope
    (
        cli_id bigint NOT NULL
    );
    CREATE UNIQUE CLUSTERED INDEX CIX_#clients_scope
        ON #clients_scope(cli_id);

    CREATE TABLE #result
    (
        week_start_monday       date            NOT NULL,
        week_end_monday         date            NOT NULL,
        week_from_tuesday       date            NOT NULL,
        week_to_monday          date            NOT NULL,

        cnt_cli_scope           bigint          NULL,

        cnt_con_exit            bigint          NULL,
        vol_exit_deposits_rub   decimal(38, 6) NULL,
        wavg_con_rate_exit      decimal(18, 6) NULL,

        cnt_con_open            bigint          NULL,
        vol_opened_deposits_rub decimal(38, 6) NULL,
        wavg_con_rate_open      decimal(18, 6) NULL,

        ns_balance_start_rub    decimal(38, 6) NULL,
        ns_balance_end_rub      decimal(38, 6) NULL,
        ns_balance_delta_rub    decimal(38, 6) NULL
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

        INSERT INTO #clients_scope (cli_id)
        SELECT DISTINCT
            b.cli_id
        FROM #bal_prev b
        WHERE b.section_name = N'Срочные'
          AND b.dt_close_plan >= @WeekFrom
          AND b.dt_close_plan <= @WeekTo;

        ;WITH deposits_to_exit AS
        (
            SELECT
                  b.cli_id
                , b.con_id
                , b.out_rub
                , b.rate_con
            FROM #bal_prev b
            WHERE b.section_name = N'Срочные'
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
            FROM #bal_curr b
            INNER JOIN #clients_scope c
                ON b.cli_id = c.cli_id
            WHERE b.section_name = N'Срочные'
              AND b.dt_open >= @WeekFrom
              AND b.dt_open <= @WeekTo
        ),
        ns_by_date AS
        (
            SELECT
                  x.dt_rep
                , x.cli_id
                , SUM(x.out_rub) AS ns_out_rub
            FROM
            (
                SELECT dt_rep, cli_id, out_rub
                FROM #bal_prev
                WHERE section_name = N'Накопительный счёт'

                UNION ALL

                SELECT dt_rep, cli_id, out_rub
                FROM #bal_curr
                WHERE section_name = N'Накопительный счёт'
            ) x
            INNER JOIN #clients_scope c
                ON x.cli_id = c.cli_id
            GROUP BY
                  x.dt_rep
                , x.cli_id
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
                    AS decimal(18, 6)
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
                    AS decimal(18, 6)
                  ) AS wavg_rate_open
            FROM opened_deposits
        ),
        agg_ns AS
        (
            SELECT
                  COUNT(DISTINCT c.cli_id) AS cnt_cli_scope
                , SUM(CASE WHEN n.dt_rep = @CurMondayStart THEN n.ns_out_rub ELSE 0 END) AS ns_start_vol
                , SUM(CASE WHEN n.dt_rep = @CurMondayEnd   THEN n.ns_out_rub ELSE 0 END) AS ns_end_vol
            FROM #clients_scope c
            LEFT JOIN ns_by_date n
                ON c.cli_id = n.cli_id
        )
        INSERT INTO #result
        (
              week_start_monday
            , week_end_monday
            , week_from_tuesday
            , week_to_monday
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
            , ns.cnt_cli_scope
            , ex.cnt_con_exit
            , ex.vol_exit
            , ex.wavg_rate_exit
            , op.cnt_con_open
            , op.vol_open
            , op.wavg_rate_open
            , ns.ns_start_vol
            , ns.ns_end_vol
            , ns.ns_end_vol - ns.ns_start_vol
        FROM agg_exit ex
        CROSS JOIN agg_open op
        CROSS JOIN agg_ns ns;

        IF DATEADD(day, 7, @CurMondayEnd) <= @EffectiveDateTo
        BEGIN
            DECLARE @NextMonday date = DATEADD(day, 7, @CurMondayEnd);

            TRUNCATE TABLE #bal_swap;

            INSERT INTO #bal_swap
            SELECT * FROM #bal_curr;

            TRUNCATE TABLE #bal_prev;
            INSERT INTO #bal_prev
            SELECT * FROM #bal_swap;

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
    ORDER BY week_start_monday, week_end_monday;
END
GO
