
USE ALM;
SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @DateBase date = '2026-10-31';
DECLARE @DateFrom date = '2026-11-01';
DECLARE @DateTo date = '2026-11-18';

DECLARE @threshold decimal(18,2) = 200000000;

DECLARE @DaysPerRun int = 2;
DECLARE @MaxRunSeconds int = 360;

DECLARE @RunStart datetime2 = SYSDATETIME();
DECLARE @DayStart datetime2;
DECLARE @dt date;
DECLARE @DoneDays int;
DECLARE @TotalDays int = DATEDIFF(day, @DateFrom, @DateTo) + 1;
DECLARE @RowsDay int;
DECLARE @Msg nvarchar(400);

IF @@TRANCOUNT <> 0
    THROW 50001, 'There is an open transaction', 1;


/* =========================================================
   1. ИНИЦИАЛИЗАЦИЯ.
   Выполняется только при первом запуске.
   ========================================================= */

IF OBJECT_ID('tempdb..#pay_job') IS NULL
BEGIN

    DROP TABLE IF EXISTS #pay_clients;
    DROP TABLE IF EXISTS #pay_day;
    DROP TABLE IF EXISTS #pay_sums;
    DROP TABLE IF EXISTS #pay_done;

    /* Клиенты с положительным остатком на 31.10 */

    SELECT DISTINCT
        t.CLI_ID

    INTO #pay_clients

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    WHERE t.dt_rep >= @DateBase
      AND t.dt_rep < DATEADD(day, 1, @DateBase)

      AND t.block_name = N'Привлечение ФЛ'
      AND t.section_name IN (
          N'Срочные',
          N'Накопительный счёт',
          N'До востребования'
      )

      AND t.od_flag = 1
      AND t.cur = '810'
      AND t.CLI_ID IS NOT NULL
      AND t.out_rub > 0

    OPTION (MAXDOP 1, RECOMPILE);

    CREATE UNIQUE CLUSTERED INDEX IX_pay_clients
    ON #pay_clients (CLI_ID);

    IF NOT EXISTS (SELECT 1 FROM #pay_clients)
        THROW 50002,
            'No clients found on 2026-10-31',
            1;


    /* Промежуточный агрегат текущего дня */

    CREATE TABLE #pay_day (
        client_type nvarchar(20),
        bank nvarchar(400),
        bank_type varchar(10),
        gr_size nvarchar(100),

        incoming_ns decimal(38,6),
        outgoing_ns decimal(38,6),

        incoming_sr decimal(38,6),
        outgoing_sr decimal(38,6),

        incoming_dvs decimal(38,6),
        outgoing_dvs decimal(38,6)
    );


    /* Накопленные агрегаты всех обработанных дней */

    CREATE TABLE #pay_sums (
        dt_rep date NOT NULL,

        client_type nvarchar(20),
        bank nvarchar(400),
        bank_type varchar(10),
        gr_size nvarchar(100),

        incoming_ns decimal(38,6),
        outgoing_ns decimal(38,6),

        incoming_sr decimal(38,6),
        outgoing_sr decimal(38,6),

        incoming_dvs decimal(38,6),
        outgoing_dvs decimal(38,6)
    );


    /* Контроль завершённых дней */

    CREATE TABLE #pay_done (
        dt_rep date NOT NULL PRIMARY KEY
    );


    /* Параметры расчёта */

    CREATE TABLE #pay_job (
        date_base date,
        date_from date,
        date_to date
    );

    INSERT INTO #pay_job
    VALUES (@DateBase, @DateFrom, @DateTo);

END;


/* =========================================================
   2. ПРОВЕРКА ПОВТОРНОГО ЗАПУСКА
   ========================================================= */

IF EXISTS (
    SELECT 1
    FROM #pay_job
    WHERE date_base <> @DateBase
       OR date_from <> @DateFrom
       OR date_to <> @DateTo
)
    THROW 50003,
        'Date parameters changed',
        1;

IF OBJECT_ID('tempdb..#pay_clients') IS NULL
   OR OBJECT_ID('tempdb..#pay_day') IS NULL
   OR OBJECT_ID('tempdb..#pay_sums') IS NULL
   OR OBJECT_ID('tempdb..#pay_done') IS NULL
    THROW 50004,
        'Temporary state is incomplete',
        1;


/* Продолжаем с первой необработанной даты */

SELECT @dt = DATEADD(day, 1, MAX(dt_rep))
FROM #pay_done;

IF @dt IS NULL
    SET @dt = @DateFrom;


/* =========================================================
   3. ОБРАБОТКА ДЕТАЛИЗАЦИИ ПО ОДНОМУ ДНЮ
   ========================================================= */

DECLARE @DaysThisRun int = 0;

WHILE @dt <= @DateTo
  AND @DaysThisRun < @DaysPerRun
BEGIN

    IF DATEDIFF(second, @RunStart, SYSDATETIME())
       >= @MaxRunSeconds
        BREAK;

    SET @DayStart = SYSDATETIME();

    TRUNCATE TABLE #pay_day;

    SET @Msg = CONCAT(
        N'Читаю платежи: ',
        CONVERT(varchar(10), @dt, 120)
    );

    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;


    /* ЕДИНСТВЕННОЕ ОБРАЩЕНИЕ К ДЕТАЛЬНОЙ ВИТРИНЕ
       ЗА ТЕКУЩИЙ ДЕНЬ */

    ;WITH base AS (
        SELECT
            CASE
                WHEN c.CLI_ID IS NOT NULL
                    THEN N'Действующий'
                ELSE N'Новый'
            END AS client_type,

            CASE
                WHEN t.spec_cat IN ('BR','SR','FU')
                    THEN N'Market'
                WHEN t.transaction_type = N'Наличные'
                    THEN N'Наличные'
                ELSE t.bank_name_main
            END AS bank,

            CASE
                WHEN t.spec_cat IN ('BR','SR','FU')
                    THEN 'market'
                WHEN t.transaction_type = N'Наличные'
                    THEN 'cash'
                ELSE 'bank'
            END AS bank_type,

            ISNULL(t.gr_size, N'Без gr_size') AS gr_size,

            t.direction_txt,

            t.[распознано_НС] AS amt_ns,
            t.[распознано_СР] AS amt_sr,
            t.[Распознано_ДВС] AS amt_dvs,

            t.[распознано_НС]
              + t.[распознано_СР]
              + t.[Распознано_ДВС] AS amt

        FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

        LEFT JOIN #pay_clients c
            ON c.CLI_ID = t.cli_id

        WHERE t.dt_rep >= @dt
          AND t.dt_rep < DATEADD(day, 1, @dt)

          AND t.cli_id IS NOT NULL

          AND t.direction_type <> N'Ошибка'
          AND t.[целевая категория] = 1
          AND t.transaction_type <> N'Внутренние переводы'
          AND t.transit_max_id IS NULL
          AND t.bank_name_main <> N'Без перевода'
          AND t.cli_biz IN (N'Розница', N'ДЧБО')

          AND (
              t.spec_cat NOT IN ('BR','SR','FU')
              OR (
                  t.spec_cat IN ('BR','SR','FU')
                  AND t.transaction_type <> N'Наличные'
              )
          )
    ),

    flows AS (
        SELECT
            client_type,
            bank,
            bank_type,
            gr_size,

            amt_ns,
            amt_sr,
            amt_dvs,

            CASE
                WHEN amt IS NOT NULL
                 AND (
                    direction_txt = N'входящие'
                    OR (
                        direction_txt = N'смешанные'
                        AND amt > 0
                    )
                 )
                    THEN 1
                ELSE 0
            END AS is_incoming,

            CASE
                WHEN amt IS NOT NULL
                 AND (
                    direction_txt = N'исходящие'
                    OR (
                        direction_txt = N'смешанные'
                        AND amt < 0
                    )
                 )
                    THEN 1
                ELSE 0
            END AS is_outgoing

        FROM base
    )

    INSERT INTO #pay_day (
        client_type,
        bank,
        bank_type,
        gr_size,

        incoming_ns,
        outgoing_ns,
        incoming_sr,
        outgoing_sr,
        incoming_dvs,
        outgoing_dvs
    )

    SELECT
        client_type,
        bank,
        bank_type,
        gr_size,

        SUM(CASE WHEN is_incoming = 1
            THEN amt_ns ELSE 0 END),

        SUM(CASE WHEN is_outgoing = 1
            THEN amt_ns ELSE 0 END),

        SUM(CASE WHEN is_incoming = 1
            THEN amt_sr ELSE 0 END),

        SUM(CASE WHEN is_outgoing = 1
            THEN amt_sr ELSE 0 END),

        SUM(CASE WHEN is_incoming = 1
            THEN amt_dvs ELSE 0 END),

        SUM(CASE WHEN is_outgoing = 1
            THEN amt_dvs ELSE 0 END)

    FROM flows

    GROUP BY
        client_type,
        bank,
        bank_type,
        gr_size

    OPTION (MAXDOP 1, RECOMPILE);


    SET @RowsDay = @@ROWCOUNT;


    /* Записываем готовый день и контрольную точку
       одной транзакцией */

    BEGIN TRY

        BEGIN TRANSACTION;

        INSERT INTO #pay_sums (
            dt_rep,
            client_type,
            bank,
            bank_type,
            gr_size,

            incoming_ns,
            outgoing_ns,
            incoming_sr,
            outgoing_sr,
            incoming_dvs,
            outgoing_dvs
        )

        SELECT
            @dt,
            client_type,
            bank,
            bank_type,
            gr_size,

            incoming_ns,
            outgoing_ns,
            incoming_sr,
            outgoing_sr,
            incoming_dvs,
            outgoing_dvs

        FROM #pay_day;


        INSERT INTO #pay_done (dt_rep)
        VALUES (@dt);

        COMMIT TRANSACTION;

    END TRY

    BEGIN CATCH

        IF XACT_STATE() <> 0
            ROLLBACK TRANSACTION;

        THROW;

    END CATCH;


    SET @DaysThisRun = @DaysThisRun + 1;

    SET @Msg = CONCAT(
        N'Готово: ',
        CONVERT(varchar(10), @dt, 120),
        N' | групп: ', @RowsDay,
        N' | секунд: ',
        DATEDIFF(second, @DayStart, SYSDATETIME())
    );

    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

    SET @dt = DATEADD(day, 1, @dt);

END;


/* Очищаем детализацию последнего дня */

TRUNCATE TABLE #pay_day;


/* =========================================================
   4. СТАТУС РАСЧЁТА
   ========================================================= */

SELECT @DoneDays = COUNT(*)
FROM #pay_done;

SELECT
    @@SPID AS session_id,

    @DoneDays AS completed_days,
    @TotalDays AS total_days,

    CAST(
        100.0 * @DoneDays / @TotalDays
        AS decimal(6,2)
    ) AS progress_pct,

    (SELECT MAX(dt_rep) FROM #pay_done)
        AS last_completed_date,

    CASE
        WHEN @DoneDays = @TotalDays THEN NULL
        ELSE @dt
    END AS next_date,

    (SELECT COUNT_BIG(*) FROM #pay_clients)
        AS existing_clients_count,

    DATEDIFF(second, @RunStart, SYSDATETIME())
        AS run_seconds;


/* =========================================================
   5. ИТОГОВЫЙ ОТЧЁТ
   Только после обработки всего периода.

   Порог рассчитывается по банку по ВСЕМ:
   - клиентским группам
   - gr_size
   - видам распознано
   ========================================================= */

IF @DoneDays = @TotalDays
BEGIN

    ;WITH bank_totals AS (
        SELECT
            bank,
            bank_type,

            SUM(
                ISNULL(incoming_ns, 0)
              + ISNULL(incoming_sr, 0)
              + ISNULL(incoming_dvs, 0)
            ) AS bank_incoming,

            SUM(
                ISNULL(outgoing_ns, 0)
              + ISNULL(outgoing_sr, 0)
              + ISNULL(outgoing_dvs, 0)
            ) AS bank_outgoing

        FROM #pay_sums

        GROUP BY
            bank,
            bank_type
    ),

    classified AS (
        SELECT
            a.client_type,
            a.gr_size,

            CASE
                WHEN a.bank_type = 'bank'
                 AND ABS(b.bank_incoming) <= @threshold
                 AND ABS(b.bank_outgoing) <= @threshold
                    THEN N'else'
                ELSE a.bank
            END AS bank_name_main,

            v.raspoznano_type,
            v.incoming,
            v.outgoing

        FROM #pay_sums a

        INNER JOIN bank_totals b
            ON b.bank = a.bank
           AND b.bank_type = a.bank_type

        CROSS APPLY (
            VALUES
                (N'НС', a.incoming_ns, a.outgoing_ns),
                (N'Вклады', a.incoming_sr, a.outgoing_sr),
                (N'ДВС', a.incoming_dvs, a.outgoing_dvs)
        ) v(raspoznano_type, incoming, outgoing)
    )

    SELECT
        CONCAT(
            YEAR(@DateFrom), N' ',
            CONVERT(char(5), @DateFrom, 104),
            N'-',
            CONVERT(char(5), @DateTo, 104)
        ) AS r_w,

        client_type,
        bank_name_main,
        gr_size,
        raspoznano_type,

        SUM(incoming) / 1000000000.0 AS incoming,
        SUM(outgoing) / 1000000000.0 AS outgoing,

        SUM(incoming + outgoing) / 1000000000.0 AS saldo

    FROM classified

    GROUP BY
        client_type,
        bank_name_main,
        gr_size,
        raspoznano_type

    ORDER BY
        client_type,
        bank_name_main,
        gr_size,
        raspoznano_type;

END;
