
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
DECLARE @Stage int;
DECLARE @Done int = 0;
DECLARE @Reuse bit = 0;
DECLARE @Msg nvarchar(500);

DECLARE @TotalPayDays int =
    DATEDIFF(day, @DateFrom, @DateTo) + 1;

DECLARE @TotalBalDays int =
    DATEDIFF(day, @DateBase, @DateTo) + 1;

IF @@TRANCOUNT <> 0
    THROW 50001, 'Open transaction', 1;

IF OBJECT_ID('tempdb..#r2_clients') IS NULL
   OR OBJECT_ID('tempdb..#r2_existing') IS NULL
   OR OBJECT_ID('tempdb..#r2_pay_sums') IS NULL
   OR OBJECT_ID('tempdb..#r2_job_all_v1') IS NULL
    THROW 50002, 'Run main script first', 1;

IF EXISTS (
    SELECT 1
    FROM #r2_job_all_v1
    WHERE stage <> 4
       OR date_base <> @DateBase
       OR date_from <> @DateFrom
       OR date_to <> @DateTo
)
    THROW 50003, 'Main calculation not completed', 1;


/* ИНИЦИАЛИЗАЦИЯ.
   При повторных запусках таблицы сохраняются. */

IF OBJECT_ID('tempdb..#r2b_all_job') IS NULL
BEGIN

    DROP TABLE IF EXISTS #r2b_all_participants;
    DROP TABLE IF EXISTS #r2b_all_scan_day;
    DROP TABLE IF EXISTS #r2b_all_scan_done;
    DROP TABLE IF EXISTS #r2b_all_clients;
    DROP TABLE IF EXISTS #r2b_all_day;
    DROP TABLE IF EXISTS #r2b_all_totals;
    DROP TABLE IF EXISTS #r2b_all_done;


    /* Уникальные участники всех платежей */

    SELECT TOP (0)
        t.cli_id
    INTO #r2b_all_participants
    FROM ALM.ehd.VW_transfers_FL_det t;

    CREATE UNIQUE CLUSTERED INDEX IX_r2b_all_participants
    ON #r2b_all_participants (cli_id);


    SELECT TOP (0)
        cli_id
    INTO #r2b_all_scan_day
    FROM #r2b_all_participants;

    CREATE UNIQUE CLUSTERED INDEX IX_r2b_all_scan_day
    ON #r2b_all_scan_day (cli_id);


    CREATE TABLE #r2b_all_scan_done (
        dt_rep date NOT NULL PRIMARY KEY
    );


    /* Реестр всех участников платежей с флагами */

    SELECT TOP (0)
        cli_id,
        CAST(NULL AS nvarchar(20)) AS client_type,
        CAST(NULL AS nvarchar(20)) AS eligible_group,
        CAST(NULL AS nvarchar(20)) AS fact_group
    INTO #r2b_all_clients
    FROM #r2b_all_participants;

    CREATE UNIQUE CLUSTERED INDEX IX_r2b_all_clients
    ON #r2b_all_clients (cli_id);


    /* Балансовые агрегаты */

    CREATE TABLE #r2b_all_totals (
        dt_rep date NOT NULL,
        client_type nvarchar(20) NOT NULL,
        eligible_group nvarchar(20) NOT NULL,
        fact_group nvarchar(20) NOT NULL,

        deposits_rub decimal(38,6),
        ns_rub decimal(38,6),
        dvs_rub decimal(38,6),
        total_rub decimal(38,6),

        PRIMARY KEY (
            dt_rep,
            client_type,
            eligible_group,
            fact_group
        )
    );

    SELECT TOP (0) *
    INTO #r2b_all_day
    FROM #r2b_all_totals;


    CREATE TABLE #r2b_all_done (
        dt_rep date NOT NULL PRIMARY KEY
    );


    /* Если остался готовый реестр участников
       из первоначального скрипта, используем его. */

    IF OBJECT_ID('tempdb..#pay_participants') IS NOT NULL
       AND OBJECT_ID('tempdb..#pay_done') IS NOT NULL
       AND OBJECT_ID('tempdb..#flow_job_isnull_v2') IS NOT NULL
    BEGIN

        EXEC sys.sp_executesql
            N'
            IF EXISTS (
                SELECT 1
                FROM #flow_job_isnull_v2
                WHERE date_base = @db
                  AND date_from = @df
                  AND date_to = @dto
            )
            AND (
                SELECT COUNT(*)
                FROM #pay_done
                WHERE dt_rep BETWEEN @df AND @dto
            ) = DATEDIFF(day, @df, @dto) + 1
            BEGIN

                INSERT INTO #r2b_all_participants (cli_id)
                SELECT CLI_ID
                FROM #pay_participants;

                INSERT INTO #r2b_all_scan_done (dt_rep)
                SELECT dt_rep
                FROM #pay_done
                WHERE dt_rep BETWEEN @df AND @dto;

                SET @reuse = 1;

            END;
            ',
            N'@db date, @df date, @dto date, @reuse bit OUTPUT',
            @db = @DateBase,
            @df = @DateFrom,
            @dto = @DateTo,
            @reuse = @Reuse OUTPUT;

    END;


    /* 1 = участники платежей, 2 = балансы, 3 = готово */

    CREATE TABLE #r2b_all_job (
        date_base date NOT NULL,
        date_from date NOT NULL,
        date_to date NOT NULL,
        stage int NOT NULL
    );

    INSERT INTO #r2b_all_job
    VALUES (@DateBase, @DateFrom, @DateTo, 1);

END;


IF EXISTS (
    SELECT 1
    FROM #r2b_all_job
    WHERE date_base <> @DateBase
       OR date_from <> @DateFrom
       OR date_to <> @DateTo
)
    THROW 50004, 'Date parameters changed', 1;


SELECT @Stage = stage
FROM #r2b_all_job;


/* ВСЕ УЧАСТНИКИ ПЛАТЕЖЕЙ ПО 4 ДНЯ.
   Фильтры как в оригинальном скрипте.
   Сохраняем только уникальные CLI_ID. */

IF @Stage = 1
BEGIN

    SELECT @dt = DATEADD(day, 1, MAX(dt_rep))
    FROM #r2b_all_scan_done;

    IF @dt IS NULL
        SET @dt = @DateFrom;


    WHILE @dt <= @DateTo
      AND @Done < @DaysPerRun
    BEGIN

        IF DATEDIFF(second, @RunStart, SYSDATETIME())
           >= @MaxRunSeconds
            BREAK;

        SET @DayStart = SYSDATETIME();

        SET @Msg = CONCAT(
            N'Участники платежей: ',
            CONVERT(varchar(10), @dt, 120)
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        TRUNCATE TABLE #r2b_all_scan_day;


        INSERT INTO #r2b_all_scan_day (cli_id)

        SELECT DISTINCT
            t.cli_id

        FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

        CROSS APPLY (
            VALUES (
                ISNULL(t.[распознано_НС], 0),
                ISNULL(t.[распознано_СР], 0),
                ISNULL(t.[Распознано_ДВС], 0)
            )
        ) a(amt_ns, amt_sr, amt_dvs)

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
                  AND a.amt_ns + a.amt_sr + a.amt_dvs <> 0
              )
          )

        OPTION (MAXDOP 1, RECOMPILE);


        /* Сохраняем результат готового дня */

        BEGIN TRY

            BEGIN TRANSACTION;

            INSERT INTO #r2b_all_participants (cli_id)

            SELECT s.cli_id
            FROM #r2b_all_scan_day s
            WHERE NOT EXISTS (
                SELECT 1
                FROM #r2b_all_participants p
                WHERE p.cli_id = s.cli_id
            );

            INSERT INTO #r2b_all_scan_done (dt_rep)
            VALUES (@dt);

            COMMIT TRANSACTION;

        END TRY

        BEGIN CATCH

            IF XACT_STATE() <> 0
                ROLLBACK TRANSACTION;

            THROW;

        END CATCH;


        SET @Done = @Done + 1;

        SET @Msg = CONCAT(
            N'Готово: ',
            CONVERT(varchar(10), @dt, 120),
            N' | секунд: ',
            DATEDIFF(second, @DayStart, SYSDATETIME())
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        SET @dt = DATEADD(day, 1, @dt);

    END;


    IF (
        SELECT COUNT(*)
        FROM #r2b_all_scan_done
    ) = @TotalPayDays
    BEGIN

        /* Всем участникам платежей присваиваем флаги.
           Клиенты вне удержания получают Нет/Нет. */

        TRUNCATE TABLE #r2b_all_clients;

        INSERT INTO #r2b_all_clients (
            cli_id,
            client_type,
            eligible_group,
            fact_group
        )

        SELECT
            p.cli_id,

            CASE
                WHEN e.cli_id IS NOT NULL
                    THEN N'Действующий'
                ELSE N'Новый'
            END,

            CASE
                WHEN c.sms_eligible_ever = 1
                 AND c.push_eligible_ever = 1
                    THEN N'SMS + Push'

                WHEN c.sms_eligible_ever = 1
                    THEN N'SMS'

                WHEN c.push_eligible_ever = 1
                    THEN N'Push'

                ELSE N'Нет'
            END,

            CASE
                WHEN c.sms_fact_ever = 1
                 AND c.push_fact_ever = 1
                    THEN N'SMS + Push'

                WHEN c.sms_fact_ever = 1
                    THEN N'SMS'

                WHEN c.push_fact_ever = 1
                    THEN N'Push'

                ELSE N'Нет'
            END

        FROM #r2b_all_participants p

        LEFT JOIN #r2_existing e
            ON e.cli_id = p.cli_id

        LEFT JOIN #r2_clients c
            ON c.cli_id = p.cli_id

        OPTION (MAXDOP 1, RECOMPILE);


        UPDATE #r2b_all_job
        SET stage = 2;

    END;

END


/* ПОДНЕВНЫЕ БАЛАНСЫ ПО 4 ДНЯ ЗА ЗАПУСК.
   Все участники платежей 01.11-18.11.
   Группы клиентов фиксируются за весь период. */

ELSE IF @Stage = 2
BEGIN

    SELECT @dt = DATEADD(day, 1, MAX(dt_rep))
    FROM #r2b_all_done;

    IF @dt IS NULL
        SET @dt = @DateBase;


    WHILE @dt <= @DateTo
      AND @Done < @DaysPerRun
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

        TRUNCATE TABLE #r2b_all_day;


        /* Один дневной срез баланса */

        INSERT INTO #r2b_all_day (
            dt_rep,
            client_type,
            eligible_group,
            fact_group,

            deposits_rub,
            ns_rub,
            dvs_rub,
            total_rub
        )

        SELECT
            @dt,
            c.client_type,
            c.eligible_group,
            c.fact_group,

            SUM(CASE
                WHEN t.section_name = N'Срочные'
                    THEN a.out_rub
                ELSE 0
            END),

            SUM(CASE
                WHEN t.section_name = N'Накопительный счёт'
                    THEN a.out_rub
                ELSE 0
            END),

            SUM(CASE
                WHEN t.section_name = N'До востребования'
                    THEN a.out_rub
                ELSE 0
            END),

            SUM(a.out_rub)

        FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

        INNER JOIN #r2b_all_clients c
            ON c.cli_id = t.CLI_ID

        CROSS APPLY (
            VALUES (
                CASE
                    WHEN @dt = @DateBase
                        THEN CAST(t.out_rub AS decimal(28,2))
                    ELSE CAST(t.out_rub AS decimal(28,6))
                END
            )
        ) a(out_rub)

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
            c.client_type,
            c.eligible_group,
            c.fact_group

        OPTION (MAXDOP 1, RECOMPILE);


        /* Сохраняем результат готового дня */

        BEGIN TRY

            BEGIN TRANSACTION;

            INSERT INTO #r2b_all_totals
            SELECT *
            FROM #r2b_all_day;

            INSERT INTO #r2b_all_done (dt_rep)
            VALUES (@dt);

            COMMIT TRANSACTION;

        END TRY

        BEGIN CATCH

            IF XACT_STATE() <> 0
                ROLLBACK TRANSACTION;

            THROW;

        END CATCH;


        SET @Done = @Done + 1;

        SET @Msg = CONCAT(
            N'Готово: ',
            CONVERT(varchar(10), @dt, 120),
            N' | ',
            @Done, N'/', @DaysPerRun,
            N' | секунд: ',
            DATEDIFF(second, @DayStart, SYSDATETIME())
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        SET @dt = DATEADD(day, 1, @dt);

    END;


    IF (
        SELECT COUNT(*)
        FROM #r2b_all_done
    ) = @TotalBalDays
    BEGIN

        UPDATE #r2b_all_job
        SET stage = 3;

    END;

END;


/* СТАТУС РАСЧЕТА */

SELECT @Stage = stage
FROM #r2b_all_job;

SELECT
    @@SPID AS session_id,

    CASE
        WHEN @Stage = 1 THEN N'Участники платежей'
        WHEN @Stage = 2 THEN N'Балансы'
        ELSE N'Готово'
    END AS stage,

    (SELECT COUNT(*) FROM #r2b_all_scan_done)
        AS payment_days_done,

    @TotalPayDays AS payment_days_total,

    (SELECT COUNT_BIG(*) FROM #r2b_all_participants)
        AS payment_clients,

    (SELECT COUNT(*) FROM #r2b_all_done)
        AS balance_days_done,

    @TotalBalDays AS balance_days_total,

    (SELECT MAX(dt_rep) FROM #r2b_all_done)
        AS last_balance_date,

    DATEDIFF(second, @RunStart, SYSDATETIME())
        AS run_seconds;


/* ИТОГ — ПОДНЕВНЫЕ БАЛАНСЫ И ПЛАТЕЖИ.
   Все участники платежей.
   Все суммы в млрд рублей. */

IF @Stage = 3
BEGIN

    ;WITH client_groups AS (
        SELECT DISTINCT
            client_type,
            eligible_group,
            fact_group
        FROM #r2b_all_clients
    ),

    pay_daily AS (
        SELECT
            p.dt_rep,
            p.client_type,
            g.eligible_group,
            g.fact_group,

            SUM(
                ISNULL(p.incoming_ns, 0)
              + ISNULL(p.incoming_sr, 0)
              + ISNULL(p.incoming_dvs, 0)
            ) AS incoming_rub,

            SUM(
                ISNULL(p.outgoing_ns, 0)
              + ISNULL(p.outgoing_sr, 0)
              + ISNULL(p.outgoing_dvs, 0)
            ) AS outgoing_rub

        FROM #r2_pay_sums p

        CROSS APPLY (
            VALUES (
                CASE
                    WHEN p.sms_eligible_ever = 1
                     AND p.push_eligible_ever = 1
                        THEN N'SMS + Push'

                    WHEN p.sms_eligible_ever = 1
                        THEN N'SMS'

                    WHEN p.push_eligible_ever = 1
                        THEN N'Push'

                    ELSE N'Нет'
                END,

                CASE
                    WHEN p.sms_fact_ever = 1
                     AND p.push_fact_ever = 1
                        THEN N'SMS + Push'

                    WHEN p.sms_fact_ever = 1
                        THEN N'SMS'

                    WHEN p.push_fact_ever = 1
                        THEN N'Push'

                    ELSE N'Нет'
                END
            )
        ) g(eligible_group, fact_group)

        GROUP BY
            p.dt_rep,
            p.client_type,
            g.eligible_group,
            g.fact_group
    ),

    frame AS (
        SELECT
            d.dt_rep,
            g.client_type,
            g.eligible_group,
            g.fact_group

        FROM #r2b_all_done d
        CROSS JOIN client_groups g
    ),

    joined AS (
        SELECT
            f.dt_rep,
            f.client_type,
            f.eligible_group,
            f.fact_group,

            ISNULL(b.deposits_rub, 0) AS deposits_rub,
            ISNULL(b.ns_rub, 0) AS ns_rub,
            ISNULL(b.dvs_rub, 0) AS dvs_rub,
            ISNULL(b.total_rub, 0) AS total_rub,

            ISNULL(p.incoming_rub, 0) AS incoming_rub,
            ISNULL(p.outgoing_rub, 0) AS outgoing_rub

        FROM frame f

        LEFT JOIN #r2b_all_totals b
            ON b.dt_rep = f.dt_rep
           AND b.client_type = f.client_type
           AND b.eligible_group = f.eligible_group
           AND b.fact_group = f.fact_group

        LEFT JOIN pay_daily p
            ON p.dt_rep = f.dt_rep
           AND p.client_type = f.client_type
           AND p.eligible_group = f.eligible_group
           AND p.fact_group = f.fact_group
    ),

    calculated AS (
        SELECT
            j.*,

            FIRST_VALUE(total_rub) OVER (
                PARTITION BY
                    client_type,
                    eligible_group,
                    fact_group
                ORDER BY dt_rep
            ) AS initial_rub,

            SUM(
                incoming_rub + outgoing_rub
            ) OVER (
                PARTITION BY
                    client_type,
                    eligible_group,
                    fact_group
                ORDER BY dt_rep
                ROWS UNBOUNDED PRECEDING
            ) AS cumulative_payment_rub

        FROM joined j
    )

    SELECT
        dt_rep,
        client_type,
        eligible_group,
        fact_group,

        CAST(
            deposits_rub / 1000000000.0
            AS decimal(20,6)
        ) AS deposits_bln,

        CAST(
            ns_rub / 1000000000.0
            AS decimal(20,6)
        ) AS ns_bln,

        CAST(
            dvs_rub / 1000000000.0
            AS decimal(20,6)
        ) AS dvs_bln,

        CAST(
            total_rub / 1000000000.0
            AS decimal(20,6)
        ) AS total_bln,

        CAST(
            (total_rub - initial_rub) / 1000000000.0
            AS decimal(20,6)
        ) AS balance_change_bln,

        CAST(
            incoming_rub / 1000000000.0
            AS decimal(20,6)
        ) AS incoming_bln,

        CAST(
            outgoing_rub / 1000000000.0
            AS decimal(20,6)
        ) AS outgoing_bln,

        CAST(
            (incoming_rub + outgoing_rub) / 1000000000.0
            AS decimal(20,6)
        ) AS payment_saldo_bln,

        CAST(
            cumulative_payment_rub / 1000000000.0
            AS decimal(20,6)
        ) AS cumulative_payment_bln,

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
        CASE
            WHEN client_type = N'Действующий'
                THEN 0
            ELSE 1
        END,
        eligible_group,
        fact_group;

END;
