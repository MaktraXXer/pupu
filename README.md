
USE ALM;
SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @DateBase date = '2026-10-31';
DECLARE @DateFrom date = '2026-11-01';
DECLARE @DateTo date = '2026-11-18';

DECLARE @DaysPerRun int = 4;
DECLARE @MaxRunSeconds int = 420;

DECLARE @RunStart datetime2 = SYSDATETIME();
DECLARE @DayStart datetime2;
DECLARE @dt date;
DECLARE @DaysThisRun int = 0;
DECLARE @PayDone int;
DECLARE @BalDone int;
DECLARE @TotalPayDays int = DATEDIFF(day, @DateFrom, @DateTo) + 1;
DECLARE @TotalBalDays int = DATEDIFF(day, @DateBase, @DateTo) + 1;
DECLARE @Stage int;
DECLARE @Msg nvarchar(500);

IF @@TRANCOUNT <> 0
    THROW 50001, 'Open SQL transaction', 1;


/* ==========================================================
   1. ИНИЦИАЛИЗАЦИЯ НОВОГО РАСЧЕТА

   Старые временные таблицы не нужны.
   Повторный запуск сохраняет готовые результаты.
   ========================================================== */

IF OBJECT_ID('tempdb..#cb_job_v1') IS NULL
BEGIN

    DROP TABLE IF EXISTS #cb_participants;
    DROP TABLE IF EXISTS #cb_day_clients;
    DROP TABLE IF EXISTS #cb_existing;
    DROP TABLE IF EXISTS #cb_channels;
    DROP TABLE IF EXISTS #cb_pay_done;
    DROP TABLE IF EXISTS #cb_bal_day;
    DROP TABLE IF EXISTS #cb_bal_totals;
    DROP TABLE IF EXISTS #cb_bal_done;

    CREATE TABLE #cb_job_v1 (
        date_base date NOT NULL,
        date_from date NOT NULL,
        date_to date NOT NULL,
        stage int NOT NULL
    );

    INSERT INTO #cb_job_v1
    VALUES (@DateBase, @DateFrom, @DateTo, 1);


    /* Участники платежей */

    SELECT TOP (0)
        t.cli_id AS CLI_ID
    INTO #cb_participants
    FROM ALM.ehd.VW_transfers_FL_det t;

    CREATE UNIQUE CLUSTERED INDEX IX_cb_participants
    ON #cb_participants (CLI_ID);


    /* Клиенты текущего дня */

    SELECT TOP (0)
        CLI_ID
    INTO #cb_day_clients
    FROM #cb_participants;

    CREATE UNIQUE CLUSTERED INDEX IX_cb_day_clients
    ON #cb_day_clients (CLI_ID);


    /* Действующие клиенты на 31 октября */

    SELECT TOP (0)
        CLI_ID
    INTO #cb_existing
    FROM #cb_participants;

    CREATE UNIQUE CLUSTERED INDEX IX_cb_existing
    ON #cb_existing (CLI_ID);


    /* Каналы договоров */

    SELECT TOP (0)
        v.CON_ID,
        CAST(NULL AS nvarchar(30)) AS channel_type,
        CAST(0 AS int) AS channel_variants,
        CAST(0 AS int) AS invalid_channels
    INTO #cb_channels
    FROM ALM.ehd.VW_FL_DEPO_channels v;

    CREATE UNIQUE CLUSTERED INDEX IX_cb_channels
    ON #cb_channels (CON_ID);


    CREATE TABLE #cb_pay_done (
        dt_rep date NOT NULL PRIMARY KEY
    );


    /* Агрегаты остатков текущего дня */

    CREATE TABLE #cb_bal_day (
        client_type nvarchar(20) NOT NULL,
        channel_type nvarchar(30) NOT NULL,
        deposits_rub decimal(38,2) NOT NULL,
        ns_rub decimal(38,2) NOT NULL,
        dvs_rub decimal(38,2) NOT NULL,
        total_rub decimal(38,2) NOT NULL
    );


    /* Накопленные дневные агрегаты */

    CREATE TABLE #cb_bal_totals (
        dt_rep date NOT NULL,
        client_type nvarchar(20) NOT NULL,
        channel_type nvarchar(30) NOT NULL,
        deposits_rub decimal(38,2) NOT NULL,
        ns_rub decimal(38,2) NOT NULL,
        dvs_rub decimal(38,2) NOT NULL,
        total_rub decimal(38,2) NOT NULL,

        PRIMARY KEY (
            dt_rep,
            client_type,
            channel_type
        )
    );


    CREATE TABLE #cb_bal_done (
        dt_rep date NOT NULL PRIMARY KEY
    );

END;


/* ==========================================================
   2. ПРОВЕРКА СОСТОЯНИЯ
   ========================================================== */

IF OBJECT_ID('tempdb..#cb_participants') IS NULL
   OR OBJECT_ID('tempdb..#cb_day_clients') IS NULL
   OR OBJECT_ID('tempdb..#cb_existing') IS NULL
   OR OBJECT_ID('tempdb..#cb_channels') IS NULL
   OR OBJECT_ID('tempdb..#cb_pay_done') IS NULL
   OR OBJECT_ID('tempdb..#cb_bal_day') IS NULL
   OR OBJECT_ID('tempdb..#cb_bal_totals') IS NULL
   OR OBJECT_ID('tempdb..#cb_bal_done') IS NULL
    THROW 50002, 'Incomplete calculation state', 1;

IF EXISTS (
    SELECT 1
    FROM #cb_job_v1
    WHERE date_base <> @DateBase
       OR date_from <> @DateFrom
       OR date_to <> @DateTo
)
    THROW 50003, 'Date parameters changed', 1;

SELECT @Stage = stage
FROM #cb_job_v1;


/* ==========================================================
   ЭТАП 1. СОБИРАЕМ ТОЛЬКО CLI_ID УЧАСТНИКОВ ПЛАТЕЖЕЙ

   Суммы платежей здесь не рассчитываются.
   Сохраняем все фильтры основного отчета.

   Обрабатываем по четыре дня за запуск.
   ========================================================== */

IF @Stage = 1
BEGIN

    SELECT @dt = DATEADD(day, 1, MAX(dt_rep))
    FROM #cb_pay_done;

    IF @dt IS NULL SET @dt = @DateFrom;


    WHILE @dt <= @DateTo
      AND @DaysThisRun < @DaysPerRun
    BEGIN

        IF DATEDIFF(second, @RunStart, SYSDATETIME())
            >= @MaxRunSeconds BREAK;

        SET @DayStart = SYSDATETIME();

        SET @Msg = CONCAT(
            N'ПЛАТЕЖИ CLI_ID: ',
            CONVERT(varchar(10), @dt, 120)
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        TRUNCATE TABLE #cb_day_clients;


        INSERT INTO #cb_day_clients (CLI_ID)

        SELECT DISTINCT
            t.cli_id

        FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

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

          AND (
              t.direction_txt IN (
                  N'входящие',
                  N'исходящие'
              )

              OR (
                  t.direction_txt = N'смешанные'
                  AND (
                      ISNULL(t.[распознано_НС], 0)
                    + ISNULL(t.[распознано_СР], 0)
                    + ISNULL(t.[Распознано_ДВС], 0)
                  ) <> 0
              )
          )

        OPTION (MAXDOP 1, RECOMPILE);


        IF NOT EXISTS (SELECT 1 FROM #cb_day_clients)
        BEGIN
            SET @Msg = CONCAT(
                N'Нет отобранных платежей за ',
                CONVERT(varchar(10), @dt, 120)
            );
            THROW 50004, @Msg, 1;
        END;


        BEGIN TRY
            BEGIN TRANSACTION;

            INSERT INTO #cb_participants (CLI_ID)

            SELECT d.CLI_ID
            FROM #cb_day_clients d

            WHERE NOT EXISTS (
                SELECT 1
                FROM #cb_participants p
                WHERE p.CLI_ID = d.CLI_ID
            );

            INSERT INTO #cb_pay_done
            VALUES (@dt);

            COMMIT TRANSACTION;
        END TRY

        BEGIN CATCH
            IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
            THROW;
        END CATCH;


        SET @DaysThisRun = @DaysThisRun + 1;

        SET @Msg = CONCAT(
            N'Готово ', CONVERT(varchar(10), @dt, 120),
            N' | секунд: ',
            DATEDIFF(second, @DayStart, SYSDATETIME())
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        SET @dt = DATEADD(day, 1, @dt);

    END;


    IF (SELECT COUNT(*) FROM #cb_pay_done) = @TotalPayDays
        UPDATE #cb_job_v1 SET stage = 2;

END;


/* ==========================================================
   ЭТАП 2. ОПРЕДЕЛЯЕМ ДЕЙСТВУЮЩИХ КЛИЕНТОВ

   Только CLI_ID из платежной выборки.
   Контрольная дата 31 октября.
   ========================================================== */

ELSE IF @Stage = 2
BEGIN

    RAISERROR(
        N'Определяю действующих клиентов на 31.10',
        0, 1
    ) WITH NOWAIT;

    TRUNCATE TABLE #cb_existing;


    INSERT INTO #cb_existing (CLI_ID)

    SELECT
        t.CLI_ID

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    INNER JOIN #cb_participants p
        ON p.CLI_ID = t.CLI_ID

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
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0

    GROUP BY t.CLI_ID

    HAVING SUM(t.out_rub) > 0

    OPTION (MAXDOP 1, RECOMPILE);


    IF NOT EXISTS (SELECT 1 FROM #cb_existing)
        THROW 50005, 'No existing clients on 31.10', 1;


    UPDATE #cb_job_v1
    SET stage = 3;

END;


/* ==========================================================
   ЭТАП 3. ЗАГРУЖАЕМ СПРАВОЧНИК КАНАЛОВ

   Только для участников платежей.
   Одна запись на CON_ID.

   Неизвестные коды каналов не подменяем
   категорией "Не найдена".
   ========================================================== */

ELSE IF @Stage = 3
BEGIN

    RAISERROR(
        N'Загружаю каналы договоров участников платежей',
        0, 1
    ) WITH NOWAIT;

    TRUNCATE TABLE #cb_channels;


    INSERT INTO #cb_channels (
        CON_ID,
        channel_type,
        channel_variants,
        invalid_channels
    )

    SELECT
        v.CON_ID,

        MIN(ch.channel_type),

        COUNT(DISTINCT ch.channel_type),

        SUM(CASE
            WHEN ch.channel_type IS NULL THEN 1
            ELSE 0
        END)

    FROM ALM.ehd.VW_FL_DEPO_channels v WITH (NOLOCK)

    INNER JOIN #cb_participants p
        ON p.CLI_ID = v.CLI_ID

    CROSS APPLY (
        VALUES (
            CASE
                WHEN UPPER(LTRIM(RTRIM(v.CON_CHANNEL)))
                     IN (
                         N'MOEX',
                         N'BANKIRU',
                         N'SRAVNRU',
                         N'SRAVNIRU'
                     )
                    THEN N'Маркетплейсы'

                WHEN UPPER(LTRIM(RTRIM(v.CON_CHANNEL))) = N'OFFICE'
                    THEN N'Офис'

                WHEN UPPER(LTRIM(RTRIM(v.CON_CHANNEL))) = N'DBO'
                    THEN N'DBO'

                ELSE NULL
            END
        )
    ) ch(channel_type)

    WHERE v.CON_ID IS NOT NULL

    GROUP BY v.CON_ID

    OPTION (MAXDOP 1, RECOMPILE);


    /* Не допускаем неоднозначного канала договора */

    IF EXISTS (
        SELECT 1
        FROM #cb_channels
        WHERE channel_variants <> 1
           OR invalid_channels > 0
    )
    BEGIN

        SELECT TOP (30) *
        FROM #cb_channels
        WHERE channel_variants <> 1
           OR invalid_channels > 0;

        THROW 50006,
            'Unknown or conflicting CON_CHANNEL',
            1;
    END;


    UPDATE #cb_job_v1
    SET stage = 4;

END;


/* ==========================================================
   ЭТАП 4. ПОДНЕВНЫЕ ОСТАТКИ ПО КАНАЛАМ

   Только участники платежей.
   Статус на 31.10 неизменный.
   Канал определяется по CON_ID.
   ========================================================== */

ELSE IF @Stage = 4
BEGIN

    SELECT @dt = DATEADD(day, 1, MAX(dt_rep))
    FROM #cb_bal_done;

    IF @dt IS NULL SET @dt = @DateBase;


    WHILE @dt <= @DateTo
      AND @DaysThisRun < @DaysPerRun
    BEGIN

        IF DATEDIFF(second, @RunStart, SYSDATETIME())
            >= @MaxRunSeconds BREAK;

        SET @DayStart = SYSDATETIME();

        SET @Msg = CONCAT(
            N'БАЛАНС: ',
            CONVERT(varchar(10), @dt, 120)
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        TRUNCATE TABLE #cb_bal_day;


        INSERT INTO #cb_bal_day (
            client_type,
            channel_type,
            deposits_rub,
            ns_rub,
            dvs_rub,
            total_rub
        )

        SELECT
            CASE
                WHEN e.CLI_ID IS NOT NULL
                    THEN N'Действующий'
                ELSE N'Новый'
            END AS client_type,

            ISNULL(
                ch.channel_type,
                N'Не найдена'
            ) AS channel_type,

            SUM(CASE
                WHEN t.section_name = N'Срочные'
                    THEN CAST(t.out_rub AS decimal(28,2))
                ELSE 0
            END) AS deposits_rub,

            SUM(CASE
                WHEN t.section_name = N'Накопительный счёт'
                    THEN CAST(t.out_rub AS decimal(28,2))
                ELSE 0
            END) AS ns_rub,

            SUM(CASE
                WHEN t.section_name = N'До востребования'
                    THEN CAST(t.out_rub AS decimal(28,2))
                ELSE 0
            END) AS dvs_rub,

            SUM(CAST(t.out_rub AS decimal(28,2))) AS total_rub

        FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

        INNER JOIN #cb_participants p
            ON p.CLI_ID = t.CLI_ID

        LEFT JOIN #cb_existing e
            ON e.CLI_ID = t.CLI_ID

        LEFT JOIN #cb_channels ch
            ON ch.CON_ID = t.CON_ID

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
          AND t.out_rub IS NOT NULL
          AND t.out_rub >= 0

        GROUP BY
            CASE
                WHEN e.CLI_ID IS NOT NULL
                    THEN N'Действующий'
                ELSE N'Новый'
            END,

            ISNULL(
                ch.channel_type,
                N'Не найдена'
            )

        OPTION (MAXDOP 1, RECOMPILE);


        IF NOT EXISTS (SELECT 1 FROM #cb_bal_day)
        BEGIN
            SET @Msg = CONCAT(
                N'Пустой баланс за ',
                CONVERT(varchar(10), @dt, 120)
            );
            THROW 50007, @Msg, 1;
        END;


        BEGIN TRY
            BEGIN TRANSACTION;

            INSERT INTO #cb_bal_totals (
                dt_rep,
                client_type,
                channel_type,
                deposits_rub,
                ns_rub,
                dvs_rub,
                total_rub
            )

            SELECT
                @dt,
                client_type,
                channel_type,
                deposits_rub,
                ns_rub,
                dvs_rub,
                total_rub

            FROM #cb_bal_day;


            INSERT INTO #cb_bal_done
            VALUES (@dt);

            COMMIT TRANSACTION;
        END TRY

        BEGIN CATCH
            IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
            THROW;
        END CATCH;


        SET @DaysThisRun = @DaysThisRun + 1;

        SET @Msg = CONCAT(
            N'Готово ', CONVERT(varchar(10), @dt, 120),
            N' | секунд: ',
            DATEDIFF(second, @DayStart, SYSDATETIME())
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        SET @dt = DATEADD(day, 1, @dt);

    END;


    IF (SELECT COUNT(*) FROM #cb_bal_done) = @TotalBalDays
        UPDATE #cb_job_v1 SET stage = 5;

END;


/* ==========================================================
   5. СТАТУС ПОСЛЕ КАЖДОГО ЗАПУСКА
   ========================================================== */

SELECT @Stage = stage
FROM #cb_job_v1;

SELECT @PayDone = COUNT(*)
FROM #cb_pay_done;

SELECT @BalDone = COUNT(*)
FROM #cb_bal_done;


SELECT
    @@SPID AS session_id,

    CASE @Stage
        WHEN 1 THEN N'Сбор CLI_ID платежей'
        WHEN 2 THEN N'Следующий этап: действующие клиенты'
        WHEN 3 THEN N'Следующий этап: каналы'
        WHEN 4 THEN N'Подневные балансы'
        WHEN 5 THEN N'Готово'
    END AS stage,

    @PayDone AS payment_days_done,
    @TotalPayDays AS payment_days_total,

    @BalDone AS balance_days_done,
    @TotalBalDays AS balance_days_total,

    (SELECT COUNT_BIG(*) FROM #cb_participants)
        AS payment_clients,

    (SELECT COUNT_BIG(*) FROM #cb_existing)
        AS existing_clients,

    (SELECT COUNT_BIG(*) FROM #cb_channels)
        AS mapped_contracts,

    DATEDIFF(second, @RunStart, SYSDATETIME())
        AS run_seconds;


/* ==========================================================
   6. ИТОГОВАЯ ВЫГРУЗКА

   19 дней × 2 группы клиентов × 4 канала.
   Суммы в млрд рублей.
   ========================================================== */

IF @Stage = 5
BEGIN

    SELECT
        d.dt_rep,
        ct.client_type,
        ch.channel_type,

        CAST(
            ISNULL(b.deposits_rub, 0) / 1000000000.0
            AS decimal(20,6)
        ) AS deposits_bln,

        CAST(
            ISNULL(b.ns_rub, 0) / 1000000000.0
            AS decimal(20,6)
        ) AS ns_bln,

        CAST(
            ISNULL(b.dvs_rub, 0) / 1000000000.0
            AS decimal(20,6)
        ) AS dvs_bln,

        CAST(
            ISNULL(b.total_rub, 0) / 1000000000.0
            AS decimal(20,6)
        ) AS total_bln

    FROM #cb_bal_done d

    CROSS JOIN (
        VALUES
            (N'Действующий'),
            (N'Новый')
    ) ct(client_type)

    CROSS JOIN (
        VALUES
            (N'Маркетплейсы'),
            (N'Офис'),
            (N'DBO'),
            (N'Не найдена')
    ) ch(channel_type)

    LEFT JOIN #cb_bal_totals b
        ON b.dt_rep = d.dt_rep
       AND b.client_type = ct.client_type
       AND b.channel_type = ch.channel_type

    ORDER BY
        d.dt_rep,
        ct.client_type,
        ch.channel_type;

END;
