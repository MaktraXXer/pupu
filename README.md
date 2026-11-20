
USE ALM;
SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @DateBase date = '2026-10-31';
DECLARE @DateFrom date = '2026-11-01';
DECLARE @DateTo   date = '2026-11-18';

DECLARE @threshold decimal(18,2) = 200000000;

DECLARE @DaysPerRun int = 2;
DECLARE @MaxRunSeconds int = 420;

DECLARE @RunStart datetime2 = SYSDATETIME();
DECLARE @DayStart datetime2;
DECLARE @dt date;
DECLARE @DaysThisRun int = 0;
DECLARE @PayDone int;
DECLARE @BalDone int;
DECLARE @TotalPayDays int = DATEDIFF(day, @DateFrom, @DateTo) + 1;
DECLARE @TotalBalDays int = DATEDIFF(day, @DateBase, @DateTo) + 1;
DECLARE @Msg nvarchar(500);

IF @@TRANCOUNT <> 0
    THROW 50001, 'Open transaction in session', 1;


/* ==========================================================
   1. ИНИЦИАЛИЗАЦИЯ — ТОЛЬКО ПРИ ПЕРВОМ ЗАПУСКЕ
   ========================================================== */

IF OBJECT_ID('tempdb..#flow_job') IS NULL
BEGIN

    DROP TABLE IF EXISTS #existing_clients;
    DROP TABLE IF EXISTS #pay_participants;
    DROP TABLE IF EXISTS #pay_stage;
    DROP TABLE IF EXISTS #pay_sums;
    DROP TABLE IF EXISTS #pay_done;
    DROP TABLE IF EXISTS #bal_stage;
    DROP TABLE IF EXISTS #bal_totals;
    DROP TABLE IF EXISTS #bal_done;

    /* Положительные остатки на 31.10.
       Сохраняем также остатки по видам продуктов,
       чтобы повторно не читать баланс на 31.10. */

    SELECT
        t.CLI_ID,

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
        END) AS dvs_rub

    INTO #existing_clients

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
      AND t.out_rub >= 0

    GROUP BY t.CLI_ID
    HAVING SUM(t.out_rub) > 0

    OPTION (MAXDOP 1, RECOMPILE);

    CREATE UNIQUE CLUSTERED INDEX IX_existing_clients
    ON #existing_clients (CLI_ID);

    IF NOT EXISTS (SELECT 1 FROM #existing_clients)
        THROW 50002, 'No existing clients on 31.10', 1;


    /* Участники платежей за весь период.
       Наполняется постепенно при чтении детализации. */

    SELECT TOP (0)
        t.cli_id AS CLI_ID
    INTO #pay_participants
    FROM ALM.ehd.VW_transfers_FL_det t;

    CREATE UNIQUE CLUSTERED INDEX IX_pay_participants
    ON #pay_participants (CLI_ID);


    /* Временный результат текущего дня.
       row_kind = 0: уникальный клиент.
       row_kind = 1: агрегат платежей.

       GROUPING SETS позволяет получить оба результата
       за одно чтение платёжной витрины. */

    SELECT TOP (0)
        CAST(0 AS tinyint) AS row_kind,
        t.cli_id AS CLI_ID,
        CAST(NULL AS nvarchar(20)) AS client_type,
        CAST(NULL AS nvarchar(512)) AS bank,
        CAST(NULL AS varchar(10)) AS bank_type,
        CAST(NULL AS nvarchar(100)) AS gr_size,
        CAST(0 AS decimal(38,6)) AS incoming_ns,
        CAST(0 AS decimal(38,6)) AS outgoing_ns,
        CAST(0 AS decimal(38,6)) AS incoming_sr,
        CAST(0 AS decimal(38,6)) AS outgoing_sr,
        CAST(0 AS decimal(38,6)) AS incoming_dvs,
        CAST(0 AS decimal(38,6)) AS outgoing_dvs
    INTO #pay_stage
    FROM ALM.ehd.VW_transfers_FL_det t;


    CREATE TABLE #pay_sums (
        dt_rep date,
        client_type nvarchar(20),
        bank nvarchar(512),
        bank_type varchar(10),
        gr_size nvarchar(100),
        incoming_ns decimal(38,6),
        outgoing_ns decimal(38,6),
        incoming_sr decimal(38,6),
        outgoing_sr decimal(38,6),
        incoming_dvs decimal(38,6),
        outgoing_dvs decimal(38,6)
    );

    CREATE TABLE #pay_done (
        dt_rep date NOT NULL PRIMARY KEY
    );


    /* Промежуточные и итоговые балансы.
       Здесь всего две строки на дату. */

    CREATE TABLE #bal_stage (
        client_type nvarchar(20),
        deposits_rub decimal(38,6),
        ns_rub decimal(38,6),
        dvs_rub decimal(38,6),
        total_rub decimal(38,6)
    );

    CREATE TABLE #bal_totals (
        dt_rep date NOT NULL,
        client_type nvarchar(20) NOT NULL,
        deposits_rub decimal(38,6),
        ns_rub decimal(38,6),
        dvs_rub decimal(38,6),
        total_rub decimal(38,6),
        PRIMARY KEY (dt_rep, client_type)
    );

    CREATE TABLE #bal_done (
        dt_rep date NOT NULL PRIMARY KEY,
        matched_groups int NOT NULL
    );

    CREATE TABLE #flow_job (
        date_base date,
        date_from date,
        date_to date
    );

    INSERT INTO #flow_job
    VALUES (@DateBase, @DateFrom, @DateTo);

END;


/* ==========================================================
   2. КОНТРОЛЬ СОХРАНЁННОГО СОСТОЯНИЯ
   ========================================================== */

IF OBJECT_ID('tempdb..#existing_clients') IS NULL
   OR OBJECT_ID('tempdb..#pay_participants') IS NULL
   OR OBJECT_ID('tempdb..#pay_stage') IS NULL
   OR OBJECT_ID('tempdb..#pay_sums') IS NULL
   OR OBJECT_ID('tempdb..#pay_done') IS NULL
   OR OBJECT_ID('tempdb..#bal_stage') IS NULL
   OR OBJECT_ID('tempdb..#bal_totals') IS NULL
   OR OBJECT_ID('tempdb..#bal_done') IS NULL
    THROW 50003, 'Incomplete temporary state', 1;

IF EXISTS (
    SELECT 1
    FROM #flow_job
    WHERE date_base <> @DateBase
       OR date_from <> @DateFrom
       OR date_to <> @DateTo
)
    THROW 50004, 'Date parameters changed', 1;

SELECT @PayDone = COUNT(*) FROM #pay_done;
SELECT @BalDone = COUNT(*) FROM #bal_done;


/* ==========================================================
   3. ПЕРВЫЙ ЭТАП — ПЛАТЕЖИ

   Определяем тех же участников платежей
   и сразу агрегируем их потоки по группам.
   ========================================================== */

IF @PayDone < @TotalPayDays
BEGIN

    SELECT @dt = DATEADD(day, 1, MAX(dt_rep))
    FROM #pay_done;

    IF @dt IS NULL SET @dt = @DateFrom;

    WHILE @dt <= @DateTo
      AND @DaysThisRun < @DaysPerRun
    BEGIN

        IF DATEDIFF(second, @RunStart, SYSDATETIME())
           >= @MaxRunSeconds
            BREAK;

        SET @DayStart = SYSDATETIME();

        SET @Msg = CONCAT(
            N'ПЛАТЕЖИ: ',
            CONVERT(varchar(10), @dt, 120)
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        TRUNCATE TABLE #pay_stage;

        ;WITH base AS (
            SELECT
                t.cli_id AS CLI_ID,

                CASE
                    WHEN e.CLI_ID IS NOT NULL
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

            LEFT JOIN #existing_clients e
                ON e.CLI_ID = t.cli_id

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
                b.*,

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

            FROM base b
        )

        INSERT INTO #pay_stage (
            row_kind,
            CLI_ID,
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
            CAST(GROUPING(CLI_ID) AS tinyint),

            CLI_ID,
            client_type,
            bank,
            bank_type,
            gr_size,

            SUM(CASE WHEN is_incoming = 1
                THEN ISNULL(amt_ns,0) ELSE 0 END),

            SUM(CASE WHEN is_outgoing = 1
                THEN ISNULL(amt_ns,0) ELSE 0 END),

            SUM(CASE WHEN is_incoming = 1
                THEN ISNULL(amt_sr,0) ELSE 0 END),

            SUM(CASE WHEN is_outgoing = 1
                THEN ISNULL(amt_sr,0) ELSE 0 END),

            SUM(CASE WHEN is_incoming = 1
                THEN ISNULL(amt_dvs,0) ELSE 0 END),

            SUM(CASE WHEN is_outgoing = 1
                THEN ISNULL(amt_dvs,0) ELSE 0 END)

        FROM flows

        WHERE is_incoming = 1 OR is_outgoing = 1

        GROUP BY GROUPING SETS (
            (CLI_ID),
            (client_type, bank, bank_type, gr_size)
        )

        OPTION (MAXDOP 1, RECOMPILE);


        /* Не записываем пустой платёжный день
           как успешно обработанный. */

        IF NOT EXISTS (
            SELECT 1
            FROM #pay_stage
            WHERE row_kind = 1
        )
        BEGIN
            SET @Msg = CONCAT(
                N'Нет платежей после фильтров за ',
                CONVERT(varchar(10), @dt, 120)
            );
            THROW 50005, @Msg, 1;
        END;


        BEGIN TRY
            BEGIN TRANSACTION;

            /* Участники — по одной записи на CLI_ID */

            INSERT INTO #pay_participants (CLI_ID)
            SELECT s.CLI_ID
            FROM #pay_stage s
            WHERE s.row_kind = 0
              AND s.CLI_ID IS NOT NULL
              AND NOT EXISTS (
                  SELECT 1
                  FROM #pay_participants p
                  WHERE p.CLI_ID = s.CLI_ID
              );


            /* Только агрегированные денежные потоки */

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
            FROM #pay_stage
            WHERE row_kind = 1;


            INSERT INTO #pay_done VALUES (@dt);

            COMMIT TRANSACTION;
        END TRY
        BEGIN CATCH
            IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
            THROW;
        END CATCH;

        SET @DaysThisRun = @DaysThisRun + 1;

        SET @Msg = CONCAT(
            N'Готово: ', CONVERT(varchar(10), @dt, 120),
            N' | секунд: ',
            DATEDIFF(second, @DayStart, SYSDATETIME())
        );
        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        SET @dt = DATEADD(day, 1, @dt);

    END;

    TRUNCATE TABLE #pay_stage;

END;


/* ==========================================================
   4. ВТОРОЙ ЭТАП — БАЛАНСЫ

   Запускается после завершения платежей.
   Баланс только клиентов из #pay_participants.
   ========================================================== */

ELSE IF @BalDone < @TotalBalDays
BEGIN

    SELECT @dt = DATEADD(day, 1, MAX(dt_rep))
    FROM #bal_done;

    IF @dt IS NULL SET @dt = @DateBase;

    WHILE @dt <= @DateTo
      AND @DaysThisRun < @DaysPerRun
    BEGIN

        IF DATEDIFF(second, @RunStart, SYSDATETIME())
           >= @MaxRunSeconds
            BREAK;

        SET @DayStart = SYSDATETIME();

        SET @Msg = CONCAT(
            N'БАЛАНС: ',
            CONVERT(varchar(10), @dt, 120)
        );
        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        TRUNCATE TABLE #bal_stage;


        /* За 31 октября уже есть сохранённые данные.
           Повторно не обращаемся к VW_balance_rest_all. */

        IF @dt = @DateBase
        BEGIN

            INSERT INTO #bal_stage (
                client_type,
                deposits_rub,
                ns_rub,
                dvs_rub,
                total_rub
            )
            SELECT
                N'Действующий',
                SUM(e.deposits_rub),
                SUM(e.ns_rub),
                SUM(e.dvs_rub),
                SUM(
                    e.deposits_rub
                  + e.ns_rub
                  + e.dvs_rub
                )

            FROM #existing_clients e
            INNER JOIN #pay_participants p
                ON p.CLI_ID = e.CLI_ID

            HAVING COUNT_BIG(*) > 0;

        END

        ELSE
        BEGIN

            /* Баланс очередного дня только для
               участников платежей 01.11–18.11. */

            INSERT INTO #bal_stage (
                client_type,
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
                END,

                SUM(CASE
                    WHEN t.section_name = N'Срочные'
                        THEN CAST(t.out_rub AS decimal(28,6))
                    ELSE 0
                END),

                SUM(CASE
                    WHEN t.section_name = N'Накопительный счёт'
                        THEN CAST(t.out_rub AS decimal(28,6))
                    ELSE 0
                END),

                SUM(CASE
                    WHEN t.section_name = N'До востребования'
                        THEN CAST(t.out_rub AS decimal(28,6))
                    ELSE 0
                END),

                SUM(CAST(t.out_rub AS decimal(28,6)))

            FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

            INNER JOIN #pay_participants p
                ON p.CLI_ID = t.CLI_ID

            LEFT JOIN #existing_clients e
                ON e.CLI_ID = t.CLI_ID

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
                END

            OPTION (MAXDOP 1, RECOMPILE);

        END;


        /* Записываем оба клиентских сегмента,
           даже если у одного нет остатков. */

        BEGIN TRY
            BEGIN TRANSACTION;

            INSERT INTO #bal_totals (
                dt_rep,
                client_type,
                deposits_rub,
                ns_rub,
                dvs_rub,
                total_rub
            )
            SELECT
                @dt,
                v.client_type,
                ISNULL(s.deposits_rub, 0),
                ISNULL(s.ns_rub, 0),
                ISNULL(s.dvs_rub, 0),
                ISNULL(s.total_rub, 0)

            FROM (
                VALUES
                    (N'Действующий'),
                    (N'Новый')
            ) v(client_type)

            LEFT JOIN #bal_stage s
                ON s.client_type = v.client_type;


            INSERT INTO #bal_done (
                dt_rep,
                matched_groups
            )
            SELECT @dt, COUNT(*)
            FROM #bal_stage;

            COMMIT TRANSACTION;
        END TRY
        BEGIN CATCH
            IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
            THROW;
        END CATCH;


        SET @DaysThisRun = @DaysThisRun + 1;

        SET @Msg = CONCAT(
            N'Готово: ', CONVERT(varchar(10), @dt, 120),
            N' | секунд: ',
            DATEDIFF(second, @DayStart, SYSDATETIME())
        );
        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        SET @dt = DATEADD(day, 1, @dt);

    END;

    TRUNCATE TABLE #bal_stage;

END;


/* ==========================================================
   5. СТАТУС ПОСЛЕ КАЖДОГО ЗАПУСКА
   ========================================================== */

SELECT @PayDone = COUNT(*) FROM #pay_done;
SELECT @BalDone = COUNT(*) FROM #bal_done;

SELECT
    @@SPID AS session_id,

    CASE
        WHEN @PayDone < @TotalPayDays THEN N'Платежи'
        WHEN @BalDone < @TotalBalDays THEN N'Балансы'
        ELSE N'Готово'
    END AS stage,

    @PayDone AS payment_days_done,
    @TotalPayDays AS payment_days_total,

    @BalDone AS balance_days_done,
    @TotalBalDays AS balance_days_total,

    (SELECT COUNT_BIG(*) FROM #pay_participants)
        AS payment_clients,

    (SELECT MAX(dt_rep) FROM #pay_done)
        AS last_payment_date,

    (SELECT MAX(dt_rep) FROM #bal_done)
        AS last_balance_date,

    DATEDIFF(second, @RunStart, SYSDATETIME())
        AS run_seconds;


/* ==========================================================
   6. РЕЗУЛЬТАТ №1 — ПЛАТЕЖИ ПО БАНКАМ

   Классификация банка по всем потокам сразу,
   без зависимости от client_type, gr_size и продукта.
   ========================================================== */

IF @PayDone = @TotalPayDays
   AND @BalDone = @TotalBalDays
BEGIN

    ;WITH bank_totals AS (
        SELECT
            bank,
            bank_type,

            SUM(
                ISNULL(incoming_ns,0)
              + ISNULL(incoming_sr,0)
              + ISNULL(incoming_dvs,0)
            ) AS bank_incoming,

            SUM(
                ISNULL(outgoing_ns,0)
              + ISNULL(outgoing_sr,0)
              + ISNULL(outgoing_dvs,0)
            ) AS bank_outgoing

        FROM #pay_sums

        GROUP BY bank, bank_type
    ),

    classified AS (
        SELECT
            s.client_type,
            s.gr_size,

            CASE
                WHEN s.bank_type = 'bank'
                 AND ABS(b.bank_incoming) <= @threshold
                 AND ABS(b.bank_outgoing) <= @threshold
                    THEN N'else'
                ELSE s.bank
            END AS bank_name_main,

            v.raspoznano_type,
            v.incoming,
            v.outgoing

        FROM #pay_sums s

        INNER JOIN bank_totals b
            ON b.bank = s.bank
           AND b.bank_type = s.bank_type

        CROSS APPLY (
            VALUES
                (N'НС', s.incoming_ns, s.outgoing_ns),
                (N'Вклады', s.incoming_sr, s.outgoing_sr),
                (N'ДВС', s.incoming_dvs, s.outgoing_dvs)
        ) v(raspoznano_type, incoming, outgoing)
    )

    SELECT
        N'2026 01.11-18.11' AS r_w,
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


    /* ======================================================
       7. РЕЗУЛЬТАТ №2 — ПОДНЕВНАЯ ДИНАМИКА БАЛАНСОВ

       И рядом ежедневные и накопленные платежные потоки.
       Все суммы в млрд рублей.
       ====================================================== */

    ;WITH pay_daily AS (
        SELECT
            dt_rep,
            client_type,

            SUM(
                ISNULL(incoming_ns,0)
              + ISNULL(incoming_sr,0)
              + ISNULL(incoming_dvs,0)
            ) AS incoming_rub,

            SUM(
                ISNULL(outgoing_ns,0)
              + ISNULL(outgoing_sr,0)
              + ISNULL(outgoing_dvs,0)
            ) AS outgoing_rub

        FROM #pay_sums

        GROUP BY dt_rep, client_type
    ),

    joined AS (
        SELECT
            b.dt_rep,
            b.client_type,
            b.deposits_rub,
            b.ns_rub,
            b.dvs_rub,
            b.total_rub,

            start_bal.total_rub AS initial_rub,

            ISNULL(p.incoming_rub,0) AS incoming_rub,
            ISNULL(p.outgoing_rub,0) AS outgoing_rub

        FROM #bal_totals b

        INNER JOIN #bal_totals start_bal
            ON start_bal.dt_rep = @DateBase
           AND start_bal.client_type = b.client_type

        LEFT JOIN pay_daily p
            ON p.dt_rep = b.dt_rep
           AND p.client_type = b.client_type
    ),

    calculated AS (
        SELECT
            j.*,

            SUM(incoming_rub + outgoing_rub) OVER (
                PARTITION BY client_type
                ORDER BY dt_rep
                ROWS UNBOUNDED PRECEDING
            ) AS cumulative_payment_rub

        FROM joined j
    )

    SELECT
        dt_rep,
        client_type,

        CAST(deposits_rub / 1000000000.0
            AS decimal(20,6)) AS deposits_bln,

        CAST(ns_rub / 1000000000.0
            AS decimal(20,6)) AS ns_bln,

        CAST(dvs_rub / 1000000000.0
            AS decimal(20,6)) AS dvs_bln,

        CAST(total_rub / 1000000000.0
            AS decimal(20,6)) AS total_bln,

        CAST((total_rub - initial_rub) / 1000000000.0
            AS decimal(20,6)) AS balance_change_bln,

        CAST(incoming_rub / 1000000000.0
            AS decimal(20,6)) AS incoming_bln,

        CAST(outgoing_rub / 1000000000.0
            AS decimal(20,6)) AS outgoing_bln,

        CAST((incoming_rub + outgoing_rub) / 1000000000.0
            AS decimal(20,6)) AS payment_saldo_bln,

        CAST(cumulative_payment_rub / 1000000000.0
            AS decimal(20,6)) AS cumulative_payment_bln,

        CAST(
            (
                total_rub
              - initial_rub
              - cumulative_payment_rub
            ) / 1000000000.0
            AS decimal(20,6)
        ) AS unexplained_change_bln

    FROM calculated

    ORDER BY
        dt_rep,
        CASE WHEN client_type = N'Действующий'
            THEN 0 ELSE 1 END;

END;
