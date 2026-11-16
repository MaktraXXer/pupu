
USE ALM;
SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @DateFrom date = '2026-09-01';
DECLARE @DateTo date = '2026-11-14';

DECLARE @DaysPerRun int = 15;
DECLARE @MaxRunSeconds int = 420;

DECLARE @RunStart datetime2 = SYSDATETIME();
DECLARE @DayStart datetime2;
DECLARE @dt date;
DECLARE @DayText varchar(10);

DECLARE @TotalDays int;
DECLARE @DoneDays int;
DECLARE @ThisRun int = 0;
DECLARE @RowCount int;
DECLARE @NewClients int;
DECLARE @Pct int;
DECLARE @Msg nvarchar(500);

SET @TotalDays = DATEDIFF(day, @DateFrom, @DateTo) + 1;

IF @DaysPerRun < 1 OR @MaxRunSeconds < 1
    THROW 50001, 'Incorrect run limits', 1;


/* =====================================================
   1. ИНИЦИАЛИЗАЦИЯ
   Выполняется только при первом запуске в сессии.
   При повторном запуске таблицы НЕ удаляются.
   ===================================================== */

IF OBJECT_ID('tempdb..#port_cfg') IS NULL
BEGIN

    CREATE TABLE #port_cfg (
        date_from date NOT NULL,
        date_to date NOT NULL
    );

    INSERT INTO #port_cfg
    VALUES (@DateFrom, @DateTo);


    SELECT TOP (0)
        t.CLI_ID,
        CAST(0 AS decimal(38,2)) AS deposits_rub,
        CAST(0 AS decimal(38,2)) AS ns_rub,
        CAST(0 AS decimal(38,2)) AS dvs_rub,
        CAST(0 AS decimal(38,2)) AS total_rub
    INTO #port_day
    FROM ALM.ALM.VW_balance_rest_all t;

    CREATE UNIQUE CLUSTERED INDEX IX_port_day
    ON #port_day (CLI_ID);


    SELECT TOP (0)
        CLI_ID,
        CAST(NULL AS date) AS first_date,
        CAST(0 AS bit) AS existing_flag,
        CAST(0 AS tinyint) AS bucket_id
    INTO #port_clients
    FROM #port_day;

    CREATE UNIQUE CLUSTERED INDEX IX_port_clients
    ON #port_clients (CLI_ID);


    CREATE TABLE #port_totals (
        dt_rep date NOT NULL,
        existing_flag bit NOT NULL,
        bucket_id tinyint NOT NULL,
        deposits_rub decimal(38,2) NOT NULL,
        ns_rub decimal(38,2) NOT NULL,
        dvs_rub decimal(38,2) NOT NULL,
        total_rub decimal(38,2) NOT NULL,

        PRIMARY KEY (dt_rep, existing_flag, bucket_id)
    );


    CREATE TABLE #port_done (
        dt_rep date NOT NULL PRIMARY KEY,
        data_status varchar(10) NOT NULL,
        client_count int NOT NULL,
        new_clients int NOT NULL,
        seconds_taken int NOT NULL
    );

END;


/* Проверяем, что повторный запуск идет
   с теми же датами и сохранившимися таблицами. */

IF EXISTS (
    SELECT 1 FROM #port_cfg
    WHERE date_from <> @DateFrom OR date_to <> @DateTo
)
    THROW 50002, 'Date range changed. Start a new calculation', 1;

IF OBJECT_ID('tempdb..#port_day') IS NULL
   OR OBJECT_ID('tempdb..#port_clients') IS NULL
   OR OBJECT_ID('tempdb..#port_totals') IS NULL
   OR OBJECT_ID('tempdb..#port_done') IS NULL
    THROW 50003, 'Incomplete temporary state', 1;


/* =====================================================
   2. ОПРЕДЕЛЯЕМ ДАТУ ПРОДОЛЖЕНИЯ
   ===================================================== */

SELECT
    @dt = DATEADD(day, 1, MAX(dt_rep))
FROM #port_done;

IF @dt IS NULL
    SET @dt = @DateFrom;

SET @DayText = CONVERT(varchar(10), @dt, 120);

RAISERROR(
    N'Начало/продолжение с %s. SPID=%d',
    0, 1, @DayText, @@SPID
) WITH NOWAIT;


/* =====================================================
   3. ОБРАБОТКА ОЧЕРЕДНЫХ ДНЕЙ
   ===================================================== */

WHILE @dt <= @DateTo
  AND @ThisRun < @DaysPerRun
BEGIN

    IF DATEDIFF(second, @RunStart, SYSDATETIME())
        >= @MaxRunSeconds
        BREAK;

    SET @DayStart = SYSDATETIME();
    SET @DayText = CONVERT(varchar(10), @dt, 120);

    RAISERROR(
        N'Читаю VW_balance_rest_all: %s',
        0, 1, @DayText
    ) WITH NOWAIT;

    TRUNCATE TABLE #port_day;


    /* Одно обращение к вьюхе за один день */

    INSERT INTO #port_day (
        CLI_ID,
        deposits_rub,
        ns_rub,
        dvs_rub,
        total_rub
    )

    SELECT
        t.CLI_ID,

        SUM(CASE
            WHEN t.section_name = N'Срочные'
                THEN CAST(t.out_rub AS decimal(28,2))
            ELSE 0
        END),

        SUM(CASE
            WHEN t.section_name = N'Накопительный счёт'
                THEN CAST(t.out_rub AS decimal(28,2))
            ELSE 0
        END),

        SUM(CASE
            WHEN t.section_name = N'До востребования'
                THEN CAST(t.out_rub AS decimal(28,2))
            ELSE 0
        END),

        SUM(CAST(t.out_rub AS decimal(28,2)))

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    WHERE t.dt_rep >= @dt
      AND t.dt_rep < DATEADD(day, 1, @dt)

      AND t.block_name = N'Привлечение ФЛ'

      AND t.section_name IN (
          N'Срочные',
          N'Накопительный счёт',
          N'До востребования'
      )

      AND t.od_flag = 1
      AND t.cur = '810'
      AND t.CLI_ID IS NOT NULL
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0

      AND t.PROD_NAME_res NOT IN (
          N'Надёжный прайм',
          N'Надёжный VIP',
          N'Надёжный премиум',
          N'Надёжный промо',
          N'Надёжный старт',
          N'Надёжный Т2',
          N'Надёжный Мегафон',
          N'Надёжный процент',
          N'Могучий',
          N'Надёжный',
          N'ДОМанадёжно',
          N'Всё в ДОМ'
      )

    GROUP BY t.CLI_ID

    HAVING SUM(t.out_rub) > 0

    OPTION (MAXDOP 1, RECOMPILE);

    SET @RowCount = @@ROWCOUNT;
    SET @NewClients = 0;


    IF @dt = @DateFrom AND @RowCount = 0
        THROW 50004, 'No positive balances on initial date', 1;


    /* Все изменения одного дня фиксируются
       в одной короткой транзакции. */

    BEGIN TRY

        BEGIN TRANSACTION;

        IF @RowCount > 0
        BEGIN

            /* Новые CLI_ID — только те, которых
               еще нет в справочнике. */

            INSERT INTO #port_clients (
                CLI_ID,
                first_date,
                existing_flag,
                bucket_id
            )

            SELECT
                d.CLI_ID,
                @dt,

                CASE WHEN @dt = @DateFrom
                    THEN 1 ELSE 0
                END,

                CASE
                    WHEN d.total_rub <= 500000 THEN 1
                    WHEN d.total_rub <= 800000 THEN 2
                    WHEN d.total_rub <= 1000000 THEN 3
                    WHEN d.total_rub <= 1500000 THEN 4
                    WHEN d.total_rub <= 3000000 THEN 5
                    WHEN d.total_rub <= 5000000 THEN 6
                    WHEN d.total_rub <= 10000000 THEN 7
                    WHEN d.total_rub <= 30000000 THEN 8
                    ELSE 9
                END

            FROM #port_day d

            WHERE NOT EXISTS (
                SELECT 1
                FROM #port_clients c
                WHERE c.CLI_ID = d.CLI_ID
            );

            SET @NewClients = @@ROWCOUNT;


            /* Только агрегаты — никакой истории
               по отдельным договорам не сохраняем. */

            INSERT INTO #port_totals (
                dt_rep,
                existing_flag,
                bucket_id,
                deposits_rub,
                ns_rub,
                dvs_rub,
                total_rub
            )

            SELECT
                @dt,
                c.existing_flag,
                c.bucket_id,

                SUM(d.deposits_rub),
                SUM(d.ns_rub),
                SUM(d.dvs_rub),
                SUM(d.total_rub)

            FROM #port_day d

            INNER JOIN #port_clients c
                ON c.CLI_ID = d.CLI_ID

            GROUP BY
                c.existing_flag,
                c.bucket_id;

        END;


        /* Контрольная точка записывается последней.
           Она существует только после успешного
           формирования данных за день. */

        INSERT INTO #port_done (
            dt_rep,
            data_status,
            client_count,
            new_clients,
            seconds_taken
        )

        VALUES (
            @dt,
            CASE WHEN @RowCount = 0
                THEN 'MISSING'
                ELSE 'OK'
            END,
            @RowCount,
            @NewClients,
            DATEDIFF(second, @DayStart, SYSDATETIME())
        );

        COMMIT TRANSACTION;

    END TRY

    BEGIN CATCH

        IF XACT_STATE() <> 0
            ROLLBACK TRANSACTION;

        THROW;

    END CATCH;


    /* Прогресс после успешного COMMIT */

    SET @ThisRun = @ThisRun + 1;

    SELECT @DoneDays = COUNT(*)
    FROM #port_done;

    SET @Pct = @DoneDays * 100 / @TotalDays;

    SET @Msg = CONCAT(
        N'[',
        REPLICATE(N'#', @Pct / 5),
        REPLICATE(N'-', 20 - @Pct / 5),
        N'] ',
        @Pct, N'% | ',
        @DayText,
        N' | новых клиентов: ',
        @NewClients,
        N' | секунд: ',
        DATEDIFF(second, @DayStart, SYSDATETIME())
    );

    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

    SET @dt = DATEADD(day, 1, @dt);

END;


/* Освобождаем дневную детализацию */

TRUNCATE TABLE #port_day;


/* =====================================================
   4. СТАТУС — ВЫВОДИТСЯ ПОСЛЕ КАЖДОГО ЗАПУСКА
   ===================================================== */

SELECT @DoneDays = COUNT(*)
FROM #port_done;

SELECT
    @@SPID AS session_id,

    @DoneDays AS completed_days,
    @TotalDays AS total_days,

    CAST(
        100.0 * @DoneDays / @TotalDays
        AS decimal(6,2)
    ) AS progress_pct,

    (SELECT MAX(dt_rep) FROM #port_done)
        AS last_completed_date,

    CASE
        WHEN @dt <= @DateTo THEN @dt
        ELSE NULL
    END AS next_date,

    (SELECT COUNT_BIG(*) FROM #port_clients)
        AS unique_clients,

    DATEDIFF(second, @RunStart, SYSDATETIME())
        AS current_run_seconds;


/* =====================================================
   5. ГОТОВАЯ СТАТИСТИКА
   Выводим только после завершения всех 75 дней.
   ===================================================== */

IF @DoneDays = @TotalDays
BEGIN

    SELECT
        d.dt_rep,

        CASE s.existing_flag
            WHEN 1 THEN N'Действующий'
            ELSE N'Новый'
        END AS client_type,

        b.bucket_id,

        CASE b.bucket_id
            WHEN 1 THEN N'До 500 тыс.'
            WHEN 2 THEN N'500–800 тыс.'
            WHEN 3 THEN N'800 тыс.–1 млн'
            WHEN 4 THEN N'1–1,5 млн'
            WHEN 5 THEN N'1,5–3 млн'
            WHEN 6 THEN N'3–5 млн'
            WHEN 7 THEN N'5–10 млн'
            WHEN 8 THEN N'10–30 млн'
            WHEN 9 THEN N'Более 30 млн'
        END AS balance_bucket,

        CASE WHEN d.data_status = 'MISSING'
            THEN NULL
            ELSE CAST(
                ISNULL(r.deposits_rub, 0) / 1000000000.0
                AS decimal(20,6)
            )
        END AS deposits_bln,

        CASE WHEN d.data_status = 'MISSING'
            THEN NULL
            ELSE CAST(
                ISNULL(r.ns_rub, 0) / 1000000000.0
                AS decimal(20,6)
            )
        END AS ns_bln,

        CASE WHEN d.data_status = 'MISSING'
            THEN NULL
            ELSE CAST(
                ISNULL(r.dvs_rub, 0) / 1000000000.0
                AS decimal(20,6)
            )
        END AS dvs_bln,

        CASE WHEN d.data_status = 'MISSING'
            THEN NULL
            ELSE CAST(
                ISNULL(r.total_rub, 0) / 1000000000.0
                AS decimal(20,6)
            )
        END AS total_bln,

        d.data_status

    FROM #port_done d

    CROSS JOIN (
        VALUES (1), (0)
    ) s(existing_flag)

    CROSS JOIN (
        VALUES (1),(2),(3),(4),(5),
               (6),(7),(8),(9)
    ) b(bucket_id)

    LEFT JOIN #port_totals r
        ON r.dt_rep = d.dt_rep
       AND r.existing_flag = s.existing_flag
       AND r.bucket_id = b.bucket_id

    ORDER BY
        d.dt_rep,
        s.existing_flag DESC,
        b.bucket_id;


    /* Даты, когда баланс оказался пустым */

    SELECT dt_rep AS missing_date
    FROM #port_done
    WHERE data_status = 'MISSING'
    ORDER BY dt_rep;

END;
