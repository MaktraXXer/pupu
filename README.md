
USE ALM;
SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @DateFrom date = '2026-10-01';
DECLARE @DateTo date = '2026-11-04';

DECLARE @DaysPerRun int = 2;
DECLARE @MaxRunSeconds int = 420;

DECLARE @RunStart datetime2 = SYSDATETIME();
DECLARE @dt date;
DECLARE @Done int = 0;
DECLARE @Msg nvarchar(500);

IF @@TRANCOUNT <> 0
    THROW 50001, 'Open transaction', 1;

IF OBJECT_ID('ALM_TEST.WORK.RK_Oct2026_ClientRegistry', 'U') IS NULL
    THROW 50002, 'Не найден реестр клиентов', 1;


/* ИНИЦИАЛИЗАЦИЯ */

IF OBJECT_ID('tempdb..#rk26p_cli') IS NULL
   AND OBJECT_ID('tempdb..#rk26p_done') IS NULL
BEGIN

    CREATE TABLE #rk26p_cli (
        dt_rep date NOT NULL,
        cli_id bigint NOT NULL,
        client_group nvarchar(150),

        bank_name_main nvarchar(512),
        bank_type varchar(10),
        gr_size nvarchar(128),

        payment_type nvarchar(32),
        transaction_type nvarchar(128),
        operation_type nvarchar(128),
        payment_direction nvarchar(32),
        cli_biz nvarchar(128),

        target_flag int,
        sbp_self_1m int,

        source_rows bigint,
        count_in bigint,
        count_out bigint,

        amount_in decimal(28,2),
        amount_out decimal(28,2),

        incoming_sr decimal(28,2),
        outgoing_sr decimal(28,2),

        incoming_ns decimal(28,2),
        outgoing_ns decimal(28,2),

        incoming_dvs decimal(28,2),
        outgoing_dvs decimal(28,2),

        incoming_other decimal(28,2),
        outgoing_other decimal(28,2),

        incoming_unknown decimal(28,2),
        outgoing_unknown decimal(28,2)
    );

    CREATE CLUSTERED INDEX IX_rk26p_cli
    ON #rk26p_cli (dt_rep, cli_id);

    CREATE TABLE #rk26p_done (
        dt_rep date NOT NULL PRIMARY KEY
    );

END;

IF OBJECT_ID('tempdb..#rk26p_cli') IS NULL
   OR OBJECT_ID('tempdb..#rk26p_done') IS NULL
    THROW 50003, 'Incomplete calculation state', 1;


SELECT @dt = ISNULL(
    DATEADD(day, 1, MAX(dt_rep)),
    @DateFrom
)
FROM #rk26p_done;


/* ПЛАТЕЖИ ПО ДНЯМ */

WHILE @dt <= @DateTo
  AND @Done < @DaysPerRun
BEGIN

    IF DATEDIFF(second, @RunStart, SYSDATETIME())
       >= @MaxRunSeconds
        BREAK;

    SET @Msg = CONCAT(
        N'Платежи: ',
        CONVERT(varchar(10), @dt, 120)
    );

    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

    DELETE FROM #rk26p_cli
    WHERE dt_rep = @dt;


    ;WITH base AS (

        SELECT
            t.cli_id,
            c.client_group,

            CASE
                WHEN t.transaction_type = N'Наличные'
                    THEN N'Наличные'
                ELSE ISNULL(
                    NULLIF(t.bank_name_main, N''),
                    N'Банк не определен'
                )
            END AS bank_name_main,

            CASE
                WHEN t.transaction_type = N'Наличные'
                    THEN 'cash'
                WHEN t.bank_name_main IS NULL
                  OR t.bank_name_main IN (N'', N'Без перевода')
                    THEN 'unknown'
                ELSE 'bank'
            END AS bank_type,

            ISNULL(t.gr_size, N'Без gr_size') AS gr_size,

            CASE
                WHEN t.transaction_type = N'СБП-Перевод'
                    THEN N'СБП-Перевод'
                ELSE N'Другие платежи'
            END AS payment_type,

            ISNULL(t.transaction_type, N'Не определено')
                AS transaction_type,

            ISNULL(t.direction_type, N'Не определено')
                AS operation_type,

            ISNULL(t.direction_txt, N'Не определено')
                AS payment_direction,

            t.cli_biz,

            t.[целевая категория] AS target_flag,

            CASE
                WHEN t.transaction_type = N'СБП-Перевод'
                 AND t.direction_type = N'Переводы себе'
                 AND t.direction = -1
                 AND t.amount_out <= -1000000
                    THEN 1
                ELSE 0
            END AS sbp_self_1m,

            ISNULL(t.count_in, 0) AS count_in,
            ISNULL(t.count_out, 0) AS count_out,

            ISNULL(t.amount_in, 0) AS amount_in,
            ISNULL(t.amount_out, 0) AS amount_out,

            ISNULL(t.[Распознано_СР], 0) AS amt_sr,
            ISNULL(t.[Распознано_НС], 0) AS amt_ns,
            ISNULL(t.[Распознано_ДВС], 0) AS amt_dvs,

            ISNULL(t.[Распознано_АК], 0) AS amt_ak,
            ISNULL(t.[Распознано_ЭС], 0) AS amt_es,
            ISNULL(t.[Распознано_КР], 0) AS amt_kr,

            ISNULL(t.[Нераспознано], 0) AS amt_unknown

        FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

        INNER JOIN ALM_TEST.WORK.RK_Oct2026_ClientRegistry c
            ON c.cli_id = t.cli_id

        WHERE t.dt_rep = @dt

          AND c.client_group IN (
              N'новый клиент масс',
              N'старый клиент масс кто принес новые деньги',
              N'старый клиент масс кто не принес новые деньги'
          )

          AND t.spec_cat NOT IN ('BR','SR','FU')

          AND ISNULL(t.transaction_type, '')
              <> N'Внутренние переводы'

          AND ISNULL(t.direction_type, '')
              <> N'Ошибка'

          AND t.transit_max_id IS NULL

    )

    INSERT INTO #rk26p_cli

    SELECT
        @dt,
        cli_id,
        client_group,

        bank_name_main,
        bank_type,
        gr_size,

        payment_type,
        transaction_type,
        operation_type,
        payment_direction,
        cli_biz,

        target_flag,
        sbp_self_1m,

        COUNT_BIG(*),
        SUM(count_in),
        SUM(count_out),

        SUM(amount_in),
        SUM(amount_out),

        SUM(CASE WHEN amt_sr > 0 THEN amt_sr ELSE 0 END),
        SUM(CASE WHEN amt_sr < 0 THEN amt_sr ELSE 0 END),

        SUM(CASE WHEN amt_ns > 0 THEN amt_ns ELSE 0 END),
        SUM(CASE WHEN amt_ns < 0 THEN amt_ns ELSE 0 END),

        SUM(CASE WHEN amt_dvs > 0 THEN amt_dvs ELSE 0 END),
        SUM(CASE WHEN amt_dvs < 0 THEN amt_dvs ELSE 0 END),

        SUM(
            CASE WHEN amt_ak > 0 THEN amt_ak ELSE 0 END
            + CASE WHEN amt_es > 0 THEN amt_es ELSE 0 END
            + CASE WHEN amt_kr > 0 THEN amt_kr ELSE 0 END
        ),

        SUM(
            CASE WHEN amt_ak < 0 THEN amt_ak ELSE 0 END
            + CASE WHEN amt_es < 0 THEN amt_es ELSE 0 END
            + CASE WHEN amt_kr < 0 THEN amt_kr ELSE 0 END
        ),

        SUM(CASE
            WHEN amt_unknown > 0 THEN amt_unknown ELSE 0
        END),

        SUM(CASE
            WHEN amt_unknown < 0 THEN amt_unknown ELSE 0
        END)

    FROM base

    GROUP BY
        cli_id,
        client_group,

        bank_name_main,
        bank_type,
        gr_size,

        payment_type,
        transaction_type,
        operation_type,
        payment_direction,
        cli_biz,

        target_flag,
        sbp_self_1m

    OPTION (MAXDOP 1, RECOMPILE);


    INSERT INTO #rk26p_done VALUES (@dt);

    SET @Done = @Done + 1;

    SET @Msg = CONCAT(
        N'Готово: ',
        CONVERT(varchar(10), @dt, 120),
        N' | ',
        @Done, N'/', @DaysPerRun
    );

    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

    SET @dt = DATEADD(day, 1, @dt);

END;


/* СТАТУС */

SELECT
    @@SPID AS session_id,

    CASE
        WHEN COUNT(*) = 35 THEN N'Готово'
        ELSE N'Сбор платежей'
    END AS stage,

    COUNT(*) AS days_done,
    35 AS days_total,
    MAX(dt_rep) AS last_date,

    DATEDIFF(second, @RunStart, SYSDATETIME())
        AS run_seconds

FROM #rk26p_done;
