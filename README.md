
USE ALM;
SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @DateBase date = '2026-10-31';
DECLARE @DateFrom date = '2026-11-01';
DECLARE @DateTo date = '2026-11-18';
DECLARE @HistoryFrom date = DATEADD(day, -7, @DateFrom);

DECLARE @threshold decimal(18,2) = 200000000;
DECLARE @DaysPerRun int = 4;
DECLARE @MaxRunSeconds int = 420;

DECLARE @RunStart datetime2 = SYSDATETIME();
DECLARE @DayStart datetime2;
DECLARE @dt date;
DECLARE @d date;
DECLARE @Stage int;
DECLARE @Done int;
DECLARE @Msg nvarchar(500);
DECLARE @CohortClients bigint = NULL;
DECLARE @ReuseTriggers bit = 0;

IF @@TRANCOUNT <> 0
    THROW 50001, 'Open transaction', 1;

IF @DateBase >= @DateFrom OR @DateFrom > @DateTo
    THROW 50006, 'Incorrect date parameters', 1;


/* ИНИЦИАЛИЗАЦИЯ.
   При повторных запусках таблицы сохраняются.
   Старые агрегаты только по когорте сбрасываются.
   Готовые триггеры по тем же датам сохраняются. */

IF OBJECT_ID('tempdb..#r2_job_all_v1') IS NULL
BEGIN

    IF OBJECT_ID('tempdb..#r2_job') IS NOT NULL
       AND OBJECT_ID('tempdb..#r2_triggers') IS NOT NULL
       AND OBJECT_ID('tempdb..#r2_trigger_done') IS NOT NULL
    BEGIN

        IF NOT EXISTS (
            SELECT 1
            FROM #r2_job
            WHERE date_base <> @DateBase
               OR date_from <> @DateFrom
               OR date_to <> @DateTo
        )
            SET @ReuseTriggers = 1;

    END;


    DROP TABLE IF EXISTS #r2_existing;
    DROP TABLE IF EXISTS #r2_scan_clients;
    DROP TABLE IF EXISTS #r2_closed;
    DROP TABLE IF EXISTS #r2_messages;
    DROP TABLE IF EXISTS #r2_sms_windows;
    DROP TABLE IF EXISTS #r2_push_windows;
    DROP TABLE IF EXISTS #r2_clients;
    DROP TABLE IF EXISTS #r2_flags_day;
    DROP TABLE IF EXISTS #r2_pay_day;
    DROP TABLE IF EXISTS #r2_pay_sums;
    DROP TABLE IF EXISTS #r2_pay_done;


    /* Действующие клиенты по состоянию на 31.10 */

    SELECT
        t.CLI_ID AS cli_id

    INTO #r2_existing

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    WHERE t.dt_rep = @DateBase
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

    GROUP BY t.CLI_ID
    HAVING SUM(t.out_rub) > 0

    OPTION (MAXDOP 1, RECOMPILE);

    CREATE UNIQUE CLUSTERED INDEX IX_r2_existing
    ON #r2_existing (cli_id);

    IF NOT EXISTS (SELECT 1 FROM #r2_existing)
        THROW 50002, 'No balances on 31.10', 1;


    /* Сохраняем только даты и CLI_ID триггеров */

    IF @ReuseTriggers = 0
    BEGIN

        DROP TABLE IF EXISTS #r2_triggers;
        DROP TABLE IF EXISTS #r2_trigger_done;

        SELECT TOP (0)
            t.cli_id,
            CAST(t.dt_rep AS date) AS dt

        INTO #r2_triggers

        FROM ALM.ehd.VW_transfers_FL_det t;

        CREATE UNIQUE CLUSTERED INDEX IX_r2_triggers
        ON #r2_triggers (cli_id, dt);


        CREATE TABLE #r2_trigger_done (
            dt date NOT NULL PRIMARY KEY
        );

    END;


    SELECT TOP (0)
        cli_id

    INTO #r2_scan_clients

    FROM #r2_triggers;

    CREATE UNIQUE CLUSTERED INDEX IX_r2_scan_clients
    ON #r2_scan_clients (cli_id);


    /* Закрытия срочных вкладов */

    SELECT TOP (0)
        cli_id,
        CAST(NULL AS date) AS dt_close

    INTO #r2_closed

    FROM #r2_triggers;

    CREATE UNIQUE CLUSTERED INDEX IX_r2_closed
    ON #r2_closed (cli_id, dt_close);


    /* Реестр фактических уведомлений */

    SELECT TOP (0)
        cli_id,
        CAST(NULL AS date) AS dt,
        CAST(0 AS int) AS sms_sent,
        CAST(0 AS int) AS push_created,
        CAST(0 AS int) AS push_viewed,
        CAST(0 AS int) AS push_accepted

    INTO #r2_messages

    FROM #r2_triggers;

    CREATE UNIQUE CLUSTERED INDEX IX_r2_messages
    ON #r2_messages (cli_id, dt);


    /* Окна СМС и пушей */

    SELECT TOP (0)
        cli_id,
        dt AS dt_from,
        dt AS dt_to

    INTO #r2_sms_windows

    FROM #r2_triggers;

    CREATE UNIQUE CLUSTERED INDEX IX_r2_sms_windows
    ON #r2_sms_windows (cli_id, dt_from);


    SELECT TOP (0)
        cli_id,
        dt AS dt_from,
        dt AS dt_to

    INTO #r2_push_windows

    FROM #r2_triggers;

    CREATE UNIQUE CLUSTERED INDEX IX_r2_push_windows
    ON #r2_push_windows (cli_id, dt_from);


    /* Уникальные клиенты, связанные с удержанием */

    SELECT TOP (0)
        cli_id,
        CAST(NULL AS nvarchar(20)) AS client_type,
        CAST(0 AS int) AS sms_eligible_ever,
        CAST(0 AS int) AS push_eligible_ever,
        CAST(0 AS int) AS sms_fact_ever,
        CAST(0 AS int) AS push_fact_ever,
        CAST(0 AS int) AS push_viewed_ever

    INTO #r2_clients

    FROM #r2_triggers;

    CREATE UNIQUE CLUSTERED INDEX IX_r2_clients
    ON #r2_clients (cli_id);


    /* Подневные флаги клиентов.
       Считаем один раз на клиента за дату,
       а не на каждый платеж. */

    SELECT TOP (0)
        cli_id,
        sms_eligible_ever,
        push_eligible_ever,
        sms_fact_ever,
        push_fact_ever,
        push_viewed_ever,

        CAST(0 AS int) AS sms_window,
        CAST(0 AS int) AS push_window,
        CAST(0 AS int) AS sms_offer_window,
        CAST(0 AS int) AS push_view_day

    INTO #r2_flags_day

    FROM #r2_clients;

    CREATE UNIQUE CLUSTERED INDEX IX_r2_flags_day
    ON #r2_flags_day (cli_id);


    /* Агрегаты всех платежей.
       Клиенты без удержания получают нулевые флаги.
       Точность сумм сохранена decimal(38,6). */

    CREATE TABLE #r2_pay_sums (
        dt_rep date NOT NULL,

        client_type nvarchar(20),
        bank nvarchar(512),
        bank_type varchar(10),
        gr_size nvarchar(100),

        sms_eligible_ever int,
        push_eligible_ever int,
        sms_fact_ever int,
        push_fact_ever int,
        push_viewed_ever int,

        trigger_payment int,
        sms_window int,
        push_window int,
        sms_offer_window int,
        push_view_day int,

        incoming_ns decimal(38,6),
        outgoing_ns decimal(38,6),

        incoming_sr decimal(38,6),
        outgoing_sr decimal(38,6),

        incoming_dvs decimal(38,6),
        outgoing_dvs decimal(38,6)
    );


    SELECT TOP (0) *
    INTO #r2_pay_day
    FROM #r2_pay_sums;


    CREATE TABLE #r2_pay_done (
        dt date NOT NULL PRIMARY KEY
    );


    /* 1 = триггеры, 2 = когорта, 3 = платежи, 4 = готово */

    CREATE TABLE #r2_job_all_v1 (
        date_base date NOT NULL,
        date_from date NOT NULL,
        date_to date NOT NULL,
        threshold decimal(18,2) NOT NULL,
        stage int NOT NULL
    );

    INSERT INTO #r2_job_all_v1
    VALUES (
        @DateBase,
        @DateFrom,
        @DateTo,
        @threshold,
        1
    );

    DROP TABLE IF EXISTS #r2_job;

END;


/* ПРОВЕРКА СОСТОЯНИЯ ПОВТОРНОГО ЗАПУСКА */

IF OBJECT_ID('tempdb..#r2_existing') IS NULL
   OR OBJECT_ID('tempdb..#r2_triggers') IS NULL
   OR OBJECT_ID('tempdb..#r2_scan_clients') IS NULL
   OR OBJECT_ID('tempdb..#r2_trigger_done') IS NULL
   OR OBJECT_ID('tempdb..#r2_closed') IS NULL
   OR OBJECT_ID('tempdb..#r2_messages') IS NULL
   OR OBJECT_ID('tempdb..#r2_sms_windows') IS NULL
   OR OBJECT_ID('tempdb..#r2_push_windows') IS NULL
   OR OBJECT_ID('tempdb..#r2_clients') IS NULL
   OR OBJECT_ID('tempdb..#r2_flags_day') IS NULL
   OR OBJECT_ID('tempdb..#r2_pay_day') IS NULL
   OR OBJECT_ID('tempdb..#r2_pay_sums') IS NULL
   OR OBJECT_ID('tempdb..#r2_pay_done') IS NULL
    THROW 50003, 'Incomplete calculation state', 1;


IF EXISTS (
    SELECT 1
    FROM #r2_job_all_v1
    WHERE date_base <> @DateBase
       OR date_from <> @DateFrom
       OR date_to <> @DateTo
       OR threshold <> @threshold
)
    THROW 50004, 'Calculation parameters changed', 1;


SELECT @Stage = stage
FROM #r2_job_all_v1;


/* ИЩЕМ ТОЛЬКО СБП-ТРИГГЕРЫ ПО 4 ДНЯ.
   Не сохраняем все платежи, только CLI_ID и дату. */

IF @Stage = 1
BEGIN

    SELECT @dt = DATEADD(day, 1, MAX(dt))
    FROM #r2_trigger_done;

    IF @dt IS NULL
        SET @dt = @HistoryFrom;

    SET @Done = 0;


    WHILE @dt <= @DateTo
      AND @Done < @DaysPerRun
    BEGIN

        IF DATEDIFF(second, @RunStart, SYSDATETIME())
           >= @MaxRunSeconds
            BREAK;

        SET @DayStart = SYSDATETIME();

        SET @Msg = CONCAT(
            N'Триггеры: ',
            CONVERT(varchar(10), @dt, 120)
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        TRUNCATE TABLE #r2_scan_clients;


        INSERT INTO #r2_scan_clients (cli_id)

        SELECT DISTINCT
            t.cli_id

        FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

        WHERE t.dt_rep >= @dt
          AND t.dt_rep < DATEADD(day, 1, @dt)

          AND t.cli_id IS NOT NULL
          AND t.cli_biz = N'Розница'

          AND t.direction_type = N'Переводы себе'
          AND t.transaction_type = N'СБП-Перевод'
          AND t.direction = '-1'
          AND t.amount_out <= -1000000

          AND t.[целевая категория] = 1
          AND t.transit_max_id IS NULL
          AND t.bank_name_main <> N'Без перевода'

          AND (
              t.spec_cat NOT IN ('BR','SR','FU')
              OR (
                  t.spec_cat IN ('BR','SR','FU')
                  AND t.transaction_type <> N'Наличные'
              )
          )

        OPTION (MAXDOP 1, RECOMPILE);


        /* Сохраняем результат готового дня */

        BEGIN TRY

            BEGIN TRANSACTION;

            INSERT INTO #r2_triggers (
                cli_id,
                dt
            )

            SELECT
                cli_id,
                @dt

            FROM #r2_scan_clients;


            INSERT INTO #r2_trigger_done (dt)
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
        FROM #r2_trigger_done
    ) = DATEDIFF(day, @HistoryFrom, @DateTo) + 1
    BEGIN

        UPDATE #r2_job_all_v1
        SET stage = 2;

    END;

END


/* ЗАКРЫТИЯ, СМС, ПУШИ И КЛИЕНТСКИЕ ФЛАГИ.
   Отдельный запуск. */

ELSE IF @Stage = 2
BEGIN

    RAISERROR(
        N'Формирую закрытия вкладов и реестр уведомлений',
        0, 1
    ) WITH NOWAIT;


    /* При повторном запуске этап 2 пересчитывается.
       Готовые триггеры сохраняются. */

    TRUNCATE TABLE #r2_closed;
    TRUNCATE TABLE #r2_messages;
    TRUNCATE TABLE #r2_sms_windows;
    TRUNCATE TABLE #r2_push_windows;
    TRUNCATE TABLE #r2_clients;
    TRUNCATE TABLE #r2_flags_day;


    /* Закрытия срочных вкладов */

    INSERT INTO #r2_closed (
        cli_id,
        dt_close
    )

    SELECT
        a.cli_id,
        CAST(a.DT_CLOSE AS date)

    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)

    INNER JOIN (
        SELECT DISTINCT cli_id
        FROM #r2_triggers
    ) c
        ON c.cli_id = a.cli_id

    WHERE a.DT_CLOSE >= DATEADD(day, -2, @HistoryFrom)
      AND a.DT_CLOSE < DATEADD(day, 1, @DateTo)
      AND a.cli_short_name = N'ФЛ'
      AND a.cur = 'RUR'

      AND a.PROD_NAME NOT IN (
          N'Накопительный счёт',
          N'Накопительный счёт Ультра',
          N'Агентские эскроу ФЛ по ставке КС+спред',
          N'Агентские эскроу ФЛ по ставке КС/2',
          N'Спец. банк.счёт',
          N'Спец. банк. счёт',
          N'Залоговый',
          N'Эскроу',
          N'Депозит ФЛ (суррогатный договор для счета Новой Афины)',
          N'Тип договора не определен'
      )

    GROUP BY
        a.cli_id,
        CAST(a.DT_CLOSE AS date)

    OPTION (MAXDOP 1, RECOMPILE);


    /* Реестр фактических уведомлений */

    INSERT INTO #r2_messages (
        cli_id,
        dt,
        sms_sent,
        push_created,
        push_viewed,
        push_accepted
    )

    SELECT
        m.cli_id,
        v.dt,

        MAX(v.sms_sent),
        MAX(v.push_created),
        MAX(v.push_viewed),
        MAX(v.push_accepted)

    FROM ALM.ehd.VW_promocode_FL_list m WITH (NOLOCK)

    CROSS APPLY (
        VALUES
            (
                CAST(m.msgbegindate AS date),
                CASE
                    WHEN LOWER(m.message_type) = N'sms'
                        THEN 1
                    ELSE 0
                END,
                CASE
                    WHEN LOWER(m.message_type) = N'push'
                        THEN 1
                    ELSE 0
                END,
                0,
                0
            ),
            (
                CAST(m.dt_view AS date),
                0,
                0,
                CASE
                    WHEN LOWER(m.message_type) = N'push'
                        THEN 1
                    ELSE 0
                END,
                0
            ),
            (
                CAST(m.dt_accept AS date),
                0,
                0,
                0,
                CASE
                    WHEN LOWER(m.message_type) = N'push'
                        THEN 1
                    ELSE 0
                END
            )
    ) v(
        dt,
        sms_sent,
        push_created,
        push_viewed,
        push_accepted
    )

    WHERE m.cli_id IS NOT NULL
      AND m.msgbegindate >= @HistoryFrom
      AND m.msgbegindate < DATEADD(day, 1, @DateTo)
      AND v.dt BETWEEN @HistoryFrom AND @DateTo

    GROUP BY
        m.cli_id,
        v.dt

    OPTION (MAXDOP 1, RECOMPILE);


    /* СМС: окно 7 дней с первого подходящего перевода */

    SET @d = @HistoryFrom;

    WHILE @d <= @DateTo
    BEGIN

        INSERT INTO #r2_sms_windows (
            cli_id,
            dt_from,
            dt_to
        )

        SELECT
            t.cli_id,
            @d,
            DATEADD(day, 6, @d)

        FROM #r2_triggers t

        WHERE t.dt = @d

          AND NOT EXISTS (
              SELECT 1
              FROM #r2_sms_windows w
              WHERE w.cli_id = t.cli_id
                AND @d BETWEEN w.dt_from AND w.dt_to
          )

          AND NOT EXISTS (
              SELECT 1
              FROM #r2_messages m
              WHERE m.cli_id = t.cli_id
                AND m.sms_sent = 1
                AND m.dt BETWEEN DATEADD(day, -6, @d)
                             AND DATEADD(day, -1, @d)
          );

        SET @d = DATEADD(day, 1, @d);

    END;


    /* Пуш: подходящий перевод после закрытия вклада */

    INSERT INTO #r2_push_windows (
        cli_id,
        dt_from,
        dt_to
    )

    SELECT
        t.cli_id,
        t.dt,
        MAX(DATEADD(day, 2, c.dt_close))

    FROM #r2_triggers t

    INNER JOIN #r2_closed c
        ON c.cli_id = t.cli_id
       AND t.dt BETWEEN c.dt_close
                    AND DATEADD(day, 2, c.dt_close)

    GROUP BY
        t.cli_id,
        t.dt;


    /* Уникальная клиентская когорта и постоянные флаги */

    INSERT INTO #r2_clients (
        cli_id,
        client_type,
        sms_eligible_ever,
        push_eligible_ever,
        sms_fact_ever,
        push_fact_ever,
        push_viewed_ever
    )

    SELECT
        ids.cli_id,

        CASE
            WHEN e.cli_id IS NOT NULL
                THEN N'Действующий'
            ELSE N'Новый'
        END,

        CASE WHEN EXISTS (
            SELECT 1
            FROM #r2_sms_windows w
            WHERE w.cli_id = ids.cli_id
              AND w.dt_to >= @DateFrom
        ) THEN 1 ELSE 0 END,

        CASE WHEN EXISTS (
            SELECT 1
            FROM #r2_push_windows w
            WHERE w.cli_id = ids.cli_id
              AND w.dt_to >= @DateFrom
        ) THEN 1 ELSE 0 END,

        CASE WHEN EXISTS (
            SELECT 1
            FROM #r2_messages m
            WHERE m.cli_id = ids.cli_id
              AND m.sms_sent = 1
              AND m.dt >= DATEADD(day, -6, @DateFrom)
        ) THEN 1 ELSE 0 END,

        CASE WHEN EXISTS (
            SELECT 1
            FROM #r2_messages m
            WHERE m.cli_id = ids.cli_id
              AND m.push_created = 1
              AND m.dt >= DATEADD(day, -2, @DateFrom)
        ) THEN 1 ELSE 0 END,

        CASE WHEN EXISTS (
            SELECT 1
            FROM #r2_messages m
            WHERE m.cli_id = ids.cli_id
              AND m.push_viewed = 1
        ) THEN 1 ELSE 0 END

    FROM (
        SELECT cli_id
        FROM #r2_sms_windows
        WHERE dt_to >= @DateFrom

        UNION

        SELECT cli_id
        FROM #r2_push_windows
        WHERE dt_to >= @DateFrom

        UNION

        SELECT cli_id
        FROM #r2_messages
        WHERE (
            sms_sent = 1
            AND dt >= DATEADD(day, -6, @DateFrom)
        )
        OR (
            push_created = 1
            AND dt >= DATEADD(day, -2, @DateFrom)
        )
        OR (
            push_viewed = 1
            AND dt >= @DateFrom
        )
    ) ids

    LEFT JOIN #r2_existing e
        ON e.cli_id = ids.cli_id;


    UPDATE #r2_job_all_v1
    SET stage = 3;

END


/* ВСЕ ПЛАТЕЖИ ПО 4 ДНЯ ЗА ЗАПУСК.
   Читаем по одному дню.
   Сохраняем только агрегаты.
   Все исходные платежи остаются в расчете.
   Клиентские флаги присоединяются через LEFT JOIN. */

ELSE IF @Stage = 3
BEGIN

    SELECT @dt = DATEADD(day, 1, MAX(dt))
    FROM #r2_pay_done;

    IF @dt IS NULL
        SET @dt = @DateFrom;

    SET @Done = 0;


    WHILE @dt <= @DateTo
      AND @Done < @DaysPerRun
    BEGIN

        IF DATEDIFF(second, @RunStart, SYSDATETIME())
           >= @MaxRunSeconds
            BREAK;

        SET @DayStart = SYSDATETIME();

        SET @Msg = CONCAT(
            N'Платежи: ',
            CONVERT(varchar(10), @dt, 120)
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;


        /* Флаги на конкретную дату.
           Одна строка на каждого клиента когорты. */

        TRUNCATE TABLE #r2_flags_day;

        INSERT INTO #r2_flags_day (
            cli_id,

            sms_eligible_ever,
            push_eligible_ever,
            sms_fact_ever,
            push_fact_ever,
            push_viewed_ever,

            sms_window,
            push_window,
            sms_offer_window,
            push_view_day
        )

        SELECT
            c.cli_id,

            c.sms_eligible_ever,
            c.push_eligible_ever,
            c.sms_fact_ever,
            c.push_fact_ever,
            c.push_viewed_ever,

            CASE WHEN EXISTS (
                SELECT 1
                FROM #r2_sms_windows w
                WHERE w.cli_id = c.cli_id
                  AND @dt BETWEEN w.dt_from AND w.dt_to
            ) THEN 1 ELSE 0 END,

            CASE WHEN EXISTS (
                SELECT 1
                FROM #r2_push_windows w
                WHERE w.cli_id = c.cli_id
                  AND @dt BETWEEN w.dt_from AND w.dt_to
            ) THEN 1 ELSE 0 END,

            CASE WHEN EXISTS (
                SELECT 1
                FROM #r2_messages m
                WHERE m.cli_id = c.cli_id
                  AND m.sms_sent = 1
                  AND @dt BETWEEN m.dt
                              AND DATEADD(day, 6, m.dt)
            ) THEN 1 ELSE 0 END,

            CASE WHEN EXISTS (
                SELECT 1
                FROM #r2_messages m
                WHERE m.cli_id = c.cli_id
                  AND m.push_viewed = 1
                  AND m.dt = @dt
            ) THEN 1 ELSE 0 END

        FROM #r2_clients c

        OPTION (MAXDOP 1, RECOMPILE);


        /* Один дневной срез всех платежей */

        TRUNCATE TABLE #r2_pay_day;


        ;WITH base AS (
            SELECT
                @dt AS dt_rep,

                CASE
                    WHEN e.cli_id IS NOT NULL
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

                ISNULL(
                    t.gr_size,
                    N'Без gr_size'
                ) AS gr_size,

                ISNULL(f.sms_eligible_ever, 0)
                    AS sms_eligible_ever,

                ISNULL(f.push_eligible_ever, 0)
                    AS push_eligible_ever,

                ISNULL(f.sms_fact_ever, 0)
                    AS sms_fact_ever,

                ISNULL(f.push_fact_ever, 0)
                    AS push_fact_ever,

                ISNULL(f.push_viewed_ever, 0)
                    AS push_viewed_ever,

                CASE
                    WHEN t.direction_type = N'Переводы себе'
                     AND t.transaction_type = N'СБП-Перевод'
                     AND t.direction = '-1'
                     AND t.amount_out <= -1000000
                     AND t.cli_biz = N'Розница'
                        THEN 1
                    ELSE 0
                END AS trigger_payment,

                ISNULL(f.sms_window, 0) AS sms_window,
                ISNULL(f.push_window, 0) AS push_window,

                ISNULL(f.sms_offer_window, 0)
                    AS sms_offer_window,

                ISNULL(f.push_view_day, 0)
                    AS push_view_day,

                a.amt_ns,
                a.amt_sr,
                a.amt_dvs,

                CASE
                    WHEN t.direction_txt = N'входящие'
                      OR (
                          t.direction_txt = N'смешанные'
                          AND a.amt > 0
                      )
                        THEN 1
                    ELSE 0
                END AS is_incoming,

                CASE
                    WHEN t.direction_txt = N'исходящие'
                      OR (
                          t.direction_txt = N'смешанные'
                          AND a.amt < 0
                      )
                        THEN 1
                    ELSE 0
                END AS is_outgoing

            FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

            LEFT JOIN #r2_existing e
                ON e.cli_id = t.cli_id

            LEFT JOIN #r2_flags_day f
                ON f.cli_id = t.cli_id

            CROSS APPLY (
                VALUES (
                    CAST(
                        ISNULL(t.[распознано_НС], 0)
                        AS decimal(28,6)
                    ),
                    CAST(
                        ISNULL(t.[распознано_СР], 0)
                        AS decimal(28,6)
                    ),
                    CAST(
                        ISNULL(t.[Распознано_ДВС], 0)
                        AS decimal(28,6)
                    )
                )
            ) v(amt_ns, amt_sr, amt_dvs)

            CROSS APPLY (
                VALUES (
                    v.amt_ns,
                    v.amt_sr,
                    v.amt_dvs,

                    v.amt_ns
                    + v.amt_sr
                    + v.amt_dvs
                )
            ) a(amt_ns, amt_sr, amt_dvs, amt)

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
        )

        INSERT INTO #r2_pay_day (
            dt_rep,
            client_type,
            bank,
            bank_type,
            gr_size,

            sms_eligible_ever,
            push_eligible_ever,
            sms_fact_ever,
            push_fact_ever,
            push_viewed_ever,

            trigger_payment,
            sms_window,
            push_window,
            sms_offer_window,
            push_view_day,

            incoming_ns,
            outgoing_ns,

            incoming_sr,
            outgoing_sr,

            incoming_dvs,
            outgoing_dvs
        )

        SELECT
            dt_rep,
            client_type,
            bank,
            bank_type,
            gr_size,

            sms_eligible_ever,
            push_eligible_ever,
            sms_fact_ever,
            push_fact_ever,
            push_viewed_ever,

            trigger_payment,
            sms_window,
            push_window,
            sms_offer_window,
            push_view_day,

            SUM(CASE
                WHEN is_incoming = 1
                    THEN amt_ns
                ELSE 0
            END),

            SUM(CASE
                WHEN is_outgoing = 1
                    THEN amt_ns
                ELSE 0
            END),

            SUM(CASE
                WHEN is_incoming = 1
                    THEN amt_sr
                ELSE 0
            END),

            SUM(CASE
                WHEN is_outgoing = 1
                    THEN amt_sr
                ELSE 0
            END),

            SUM(CASE
                WHEN is_incoming = 1
                    THEN amt_dvs
                ELSE 0
            END),

            SUM(CASE
                WHEN is_outgoing = 1
                    THEN amt_dvs
                ELSE 0
            END)

        FROM base

        WHERE is_incoming = 1
           OR is_outgoing = 1

        GROUP BY
            dt_rep,
            client_type,
            bank,
            bank_type,
            gr_size,

            sms_eligible_ever,
            push_eligible_ever,
            sms_fact_ever,
            push_fact_ever,
            push_viewed_ever,

            trigger_payment,
            sms_window,
            push_window,
            sms_offer_window,
            push_view_day

        OPTION (MAXDOP 1, RECOMPILE);


        /* Сохраняем результат готового дня */

        BEGIN TRY

            BEGIN TRANSACTION;

            INSERT INTO #r2_pay_sums (
                dt_rep,
                client_type,
                bank,
                bank_type,
                gr_size,

                sms_eligible_ever,
                push_eligible_ever,
                sms_fact_ever,
                push_fact_ever,
                push_viewed_ever,

                trigger_payment,
                sms_window,
                push_window,
                sms_offer_window,
                push_view_day,

                incoming_ns,
                outgoing_ns,

                incoming_sr,
                outgoing_sr,

                incoming_dvs,
                outgoing_dvs
            )

            SELECT
                dt_rep,
                client_type,
                bank,
                bank_type,
                gr_size,

                sms_eligible_ever,
                push_eligible_ever,
                sms_fact_ever,
                push_fact_ever,
                push_viewed_ever,

                trigger_payment,
                sms_window,
                push_window,
                sms_offer_window,
                push_view_day,

                incoming_ns,
                outgoing_ns,

                incoming_sr,
                outgoing_sr,

                incoming_dvs,
                outgoing_dvs

            FROM #r2_pay_day;


            INSERT INTO #r2_pay_done (dt)
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
        FROM #r2_pay_done
    ) = DATEDIFF(day, @DateFrom, @DateTo) + 1
    BEGIN

        UPDATE #r2_job_all_v1
        SET stage = 4;

    END;

END;


/* СТАТУС ВЫПОЛНЕНИЯ ДЛЯ ВЫВОДА */

SELECT @Stage = stage
FROM #r2_job_all_v1;

IF @Stage >= 3
BEGIN

    SELECT @CohortClients = COUNT_BIG(*)
    FROM #r2_clients;

END;


SELECT
    @@SPID AS session_id,
    @Stage AS stage,

    (
        SELECT COUNT(*)
        FROM #r2_trigger_done
    ) AS trigger_days_done,

    (
        SELECT COUNT(*)
        FROM #r2_pay_done
    ) AS payment_days_done,

    (
        SELECT COUNT_BIG(*)
        FROM #r2_triggers
    ) AS trigger_client_days,

    @CohortClients AS cohort_clients,

    DATEDIFF(
        second,
        @RunStart,
        SYSDATETIME()
    ) AS run_seconds;


/* ИТОГ */

IF @Stage = 4
BEGIN

    /* Порог банка считаем по всем платежам.
       Все типы клиентов, все gr_size и все флаги. */

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

        FROM #r2_pay_sums

        GROUP BY
            bank,
            bank_type
    ),

    classified AS (
        SELECT
            p.*,

            CASE
                WHEN p.bank_type = 'bank'
                 AND ABS(b.bank_incoming) <= @threshold
                 AND ABS(b.bank_outgoing) <= @threshold
                    THEN N'else'

                ELSE p.bank
            END AS bank_name_main

        FROM #r2_pay_sums p

        INNER JOIN bank_totals b
            ON b.bank = p.bank
           AND b.bank_type = p.bank_type
    )

    SELECT
        p.dt_rep,
        p.client_type,
        p.bank_name_main,
        p.gr_size,
        v.raspoznano_type,

        CASE
            WHEN p.sms_eligible_ever = 1
             AND p.push_eligible_ever = 1
                THEN N'SMS + Push'

            WHEN p.sms_eligible_ever = 1
                THEN N'SMS'

            WHEN p.push_eligible_ever = 1
                THEN N'Push'

            ELSE N'Нет'
        END AS eligible_group,

        CASE
            WHEN p.sms_fact_ever = 1
             AND p.push_fact_ever = 1
                THEN N'SMS + Push'

            WHEN p.sms_fact_ever = 1
                THEN N'SMS'

            WHEN p.push_fact_ever = 1
                THEN N'Push'

            ELSE N'Нет'
        END AS fact_group,

        p.push_viewed_ever,
        p.sms_window,
        p.push_window,
        p.sms_offer_window,
        p.push_view_day,
        p.trigger_payment,

        SUM(v.incoming) / 1000000000.0 AS incoming,

        SUM(v.outgoing) / 1000000000.0 AS outgoing,

        SUM(
            v.incoming + v.outgoing
        ) / 1000000000.0 AS saldo

    FROM classified p

    CROSS APPLY (
        VALUES
            (
                N'НС',
                p.incoming_ns,
                p.outgoing_ns
            ),
            (
                N'Вклады',
                p.incoming_sr,
                p.outgoing_sr
            ),
            (
                N'ДВС',
                p.incoming_dvs,
                p.outgoing_dvs
            )
    ) v(
        raspoznano_type,
        incoming,
        outgoing
    )

    GROUP BY
        p.dt_rep,
        p.client_type,
        p.bank_name_main,
        p.gr_size,
        v.raspoznano_type,

        p.sms_eligible_ever,
        p.push_eligible_ever,
        p.sms_fact_ever,
        p.push_fact_ever,
        p.push_viewed_ever,

        p.sms_window,
        p.push_window,
        p.sms_offer_window,
        p.push_view_day,
        p.trigger_payment

    ORDER BY
        p.dt_rep,
        p.client_type,
        p.bank_name_main,
        p.gr_size,
        v.raspoznano_type;


    /* Реестр уникальных клиентов и флагов */

    SELECT
        client_type,

        COUNT(*) AS clients,

        SUM(sms_eligible_ever) AS sms_eligible_clients,
        SUM(push_eligible_ever) AS push_eligible_clients,

        SUM(sms_fact_ever) AS sms_fact_clients,
        SUM(push_fact_ever) AS push_fact_clients,

        SUM(push_viewed_ever) AS push_viewed_clients

    FROM #r2_clients

    GROUP BY client_type;

END;


/* ДОПОЛНИТЕЛЬНЫЙ ЗАПРОС НА БАЛАНСЫ

   После завершения этапа 2 доступна #r2_clients.
   Использовать в той же SQL-сессии.

   Смотрим остатки ровно выбранных клиентов,
   сохраняя их постоянные флаги.

   Клиенты вне когорты в этот запрос не включаются.

DECLARE @BalanceDate date = '2026-11-10';

SELECT
    t.dt_rep,
    c.client_type,

    c.sms_eligible_ever,
    c.push_eligible_ever,
    c.sms_fact_ever,
    c.push_fact_ever,
    c.push_viewed_ever,

    SUM(CASE
        WHEN t.section_name = N'Срочные'
            THEN t.out_rub ELSE 0
    END) / 1000000000.0 AS deposits_bln,

    SUM(CASE
        WHEN t.section_name = N'Накопительный счёт'
            THEN t.out_rub ELSE 0
    END) / 1000000000.0 AS ns_bln,

    SUM(CASE
        WHEN t.section_name = N'До востребования'
            THEN t.out_rub ELSE 0
    END) / 1000000000.0 AS dvs_bln,

    SUM(t.out_rub) / 1000000000.0 AS total_bln

FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

INNER JOIN #r2_clients c
    ON c.cli_id = t.CLI_ID

WHERE t.dt_rep = @BalanceDate
  AND t.block_name = N'Привлечение ФЛ'

  AND t.section_name IN (
      N'Срочные',
      N'Накопительный счёт',
      N'До востребования'
  )

  AND t.od_flag = 1
  AND t.cur = '810'
  AND t.out_rub >= 0

GROUP BY
    t.dt_rep,
    c.client_type,

    c.sms_eligible_ever,
    c.push_eligible_ever,
    c.sms_fact_ever,
    c.push_fact_ever,
    c.push_viewed_ever

OPTION (MAXDOP 1, RECOMPILE);

*/
