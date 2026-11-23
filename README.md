
USE ALM;
SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @DateBase date = '2026-10-31';
DECLARE @DateTo date = '2026-11-18';

DECLARE @DaysPerRun int = 4;
DECLARE @MaxRunSeconds int = 420;

DECLARE @RunStart datetime2 = SYSDATETIME();
DECLARE @DayStart datetime2;
DECLARE @dt date;
DECLARE @DaysThisRun int = 0;
DECLARE @BalDone int;
DECLARE @TotalBalDays int = DATEDIFF(day, @DateBase, @DateTo) + 1;
DECLARE @Msg nvarchar(500);

IF @@TRANCOUNT <> 0
    THROW 50001, 'Open transaction in session', 1;


/* ДОПОЛНИТЕЛЬНЫЙ ЗАПРОС НА БАЛАНСЫ.
   Используем реестр #r2_clients и готовые платежи.
   Группы клиентов фиксируются за весь период.
   Все суммы в млрд рублей. */

IF OBJECT_ID('tempdb..#r2_clients') IS NULL
   OR OBJECT_ID('tempdb..#r2_pay_sums') IS NULL
   OR OBJECT_ID('tempdb..#r2_job_all_v1') IS NULL
    THROW 50002, 'Run main script first', 1;

IF EXISTS (
    SELECT 1
    FROM #r2_job_all_v1
    WHERE stage <> 4
       OR date_base <> @DateBase
       OR date_to <> @DateTo
)
    THROW 50003, 'Main calculation not completed', 1;


/* ИНИЦИАЛИЗАЦИЯ.
   При повторных запусках таблицы сохраняются. */

IF OBJECT_ID('tempdb..#r2_balance_job_v1') IS NULL
BEGIN

    DROP TABLE IF EXISTS #r2_balance_day;
    DROP TABLE IF EXISTS #r2_balance_totals;
    DROP TABLE IF EXISTS #r2_balance_done;

    CREATE TABLE #r2_balance_totals (
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
    INTO #r2_balance_day
    FROM #r2_balance_totals;

    CREATE TABLE #r2_balance_done (
        dt_rep date NOT NULL PRIMARY KEY
    );

    CREATE TABLE #r2_balance_job_v1 (
        date_base date NOT NULL,
        date_to date NOT NULL
    );

    INSERT INTO #r2_balance_job_v1
    VALUES (@DateBase, @DateTo);

END;


/* ПРОВЕРКА СОСТОЯНИЯ ПОВТОРНОГО ЗАПУСКА */

IF OBJECT_ID('tempdb..#r2_balance_totals') IS NULL
   OR OBJECT_ID('tempdb..#r2_balance_day') IS NULL
   OR OBJECT_ID('tempdb..#r2_balance_done') IS NULL
    THROW 50004, 'Incomplete balance state', 1;

IF EXISTS (
    SELECT 1
    FROM #r2_balance_job_v1
    WHERE date_base <> @DateBase
       OR date_to <> @DateTo
)
    THROW 50005, 'Balance dates changed', 1;


/* ПОДНЕВНЫЕ БАЛАНСЫ ПО 4 ДНЯ ЗА ЗАПУСК.
   Только клиенты из #r2_clients.
   Сохраняем разбивку по eligible_group и fact_group. */

SELECT @dt = DATEADD(day, 1, MAX(dt_rep))
FROM #r2_balance_done;

IF @dt IS NULL
    SET @dt = @DateBase;


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

    TRUNCATE TABLE #r2_balance_day;


    /* Один дневной срез баланса */

    INSERT INTO #r2_balance_day (
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
        g.eligible_group,
        g.fact_group,

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

    INNER JOIN #r2_clients c
        ON c.cli_id = t.CLI_ID

    CROSS APPLY (
        VALUES (
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
        )
    ) g(eligible_group, fact_group)

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
        g.eligible_group,
        g.fact_group

    OPTION (MAXDOP 1, RECOMPILE);


    /* Сохраняем результат готового дня */

    BEGIN TRY

        BEGIN TRANSACTION;

        INSERT INTO #r2_balance_totals
        SELECT *
        FROM #r2_balance_day;

        INSERT INTO #r2_balance_done (dt_rep)
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
        N' | ',
        @DaysThisRun, N'/', @DaysPerRun,
        N' | секунд: ',
        DATEDIFF(second, @DayStart, SYSDATETIME())
    );

    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

    SET @dt = DATEADD(day, 1, @dt);

END;


/* СТАТУС РАСЧЕТА */

SELECT @BalDone = COUNT(*)
FROM #r2_balance_done;

SELECT
    @@SPID AS session_id,
    @BalDone AS balance_days_done,
    @TotalBalDays AS balance_days_total,
    (SELECT MAX(dt_rep) FROM #r2_balance_done) AS last_balance_date,
    DATEDIFF(second, @RunStart, SYSDATETIME()) AS run_seconds;


/* ИТОГ — ПОДНЕВНЫЕ БАЛАНСЫ И ПЛАТЕЖИ.
   Балансы и платежи одних и тех же клиентских когорт.
   Все суммы в млрд рублей. */

IF @BalDone = @TotalBalDays
BEGIN

    ;WITH client_groups AS (
        SELECT DISTINCT
            c.client_type,

            CASE
                WHEN c.sms_eligible_ever = 1
                 AND c.push_eligible_ever = 1
                    THEN N'SMS + Push'
                WHEN c.sms_eligible_ever = 1
                    THEN N'SMS'
                WHEN c.push_eligible_ever = 1
                    THEN N'Push'
                ELSE N'Нет'
            END AS eligible_group,

            CASE
                WHEN c.sms_fact_ever = 1
                 AND c.push_fact_ever = 1
                    THEN N'SMS + Push'
                WHEN c.sms_fact_ever = 1
                    THEN N'SMS'
                WHEN c.push_fact_ever = 1
                    THEN N'Push'
                ELSE N'Нет'
            END AS fact_group

        FROM #r2_clients c
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

        WHERE p.sms_eligible_ever = 1
           OR p.push_eligible_ever = 1
           OR p.sms_fact_ever = 1
           OR p.push_fact_ever = 1
           OR p.push_viewed_ever = 1

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

        FROM #r2_balance_done d
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

        LEFT JOIN #r2_balance_totals b
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

            SUM(incoming_rub + outgoing_rub) OVER (
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
