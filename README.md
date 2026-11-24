
USE ALM;
SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @DateFrom date = '2026-10-01';
DECLARE @DateTo date = '2026-11-04';

DECLARE @TriggerFrom date = DATEADD(day, -6, @DateFrom);
DECLARE @MessageFrom date = DATEADD(day, -6, @TriggerFrom);

DECLARE @threshold decimal(18,2) = 200000000;

DECLARE @DaysPerRun int = 2;
DECLARE @MessageDaysPerRun int = 7;
DECLARE @MaxRunSeconds int = 420;

DECLARE @RunStart datetime2 = SYSDATETIME();
DECLARE @Stage int;
DECLARE @dt date;
DECLARE @End date;
DECLARE @Done int = 0;
DECLARE @Msg nvarchar(500);


IF @@TRANCOUNT <> 0
    THROW 50001, 'Open transaction', 1;

IF OBJECT_ID('tempdb..#rk26p_cli') IS NULL
   OR OBJECT_ID('tempdb..#rk26p_done') IS NULL
    THROW 50002, 'Сначала собрать платежи', 1;

IF (
    SELECT COUNT(*)
    FROM #rk26p_done
    WHERE dt_rep BETWEEN @DateFrom AND @DateTo
) <> 35
    THROW 50003, 'Сбор платежей не завершен', 1;


/* ИНИЦИАЛИЗАЦИЯ */

IF OBJECT_ID('tempdb..#rk26w_job') IS NULL
BEGIN

    DROP TABLE IF EXISTS #rk26w_msg;
    DROP TABLE IF EXISTS #rk26w_event;
    DROP TABLE IF EXISTS #rk26w_scan;

    CREATE TABLE #rk26w_msg (
        cli_id bigint NOT NULL,
        dt_rep date NOT NULL,
        sms_fact_day int NOT NULL,
        push_fact_day int NOT NULL,

        PRIMARY KEY (cli_id, dt_rep)
    );


    CREATE TABLE #rk26w_event (
        dt_rep date NOT NULL,
        cli_id bigint NOT NULL,

        trigger_rows bigint NOT NULL,

        has_app_activity int NOT NULL,
        has_old_app int NOT NULL,
        has_new_app int NOT NULL,
        has_unknown_app int NOT NULL,

        close_dt date NULL,

        sms_cooldown int NOT NULL,

        sms_candidate int NOT NULL,
        push_eligible int NOT NULL,

        sms_unknown int NOT NULL,
        push_unknown int NOT NULL,

        PRIMARY KEY (dt_rep, cli_id)
    );


    SELECT TOP (0) *
    INTO #rk26w_scan
    FROM #rk26w_event;

    CREATE UNIQUE CLUSTERED INDEX IX_rk26w_scan
    ON #rk26w_scan (cli_id);


    CREATE TABLE #rk26w_job (
        stage int NOT NULL,
        next_dt date NULL
    );

    INSERT INTO #rk26w_job
    VALUES (1, @MessageFrom);

END;


IF OBJECT_ID('tempdb..#rk26w_msg') IS NULL
   OR OBJECT_ID('tempdb..#rk26w_event') IS NULL
   OR OBJECT_ID('tempdb..#rk26w_scan') IS NULL
    THROW 50004, 'Incomplete calculation state', 1;


SELECT
    @Stage = stage,
    @dt = next_dt
FROM #rk26w_job;


/* =====================================================
   STAGE 1. ФАКТИЧЕСКИЕ СМС И ПУШИ
   ===================================================== */

IF @Stage = 1
BEGIN

    SET @End = DATEADD(day, @MessageDaysPerRun, @dt);

    IF @End > DATEADD(day, 1, @DateTo)
        SET @End = DATEADD(day, 1, @DateTo);

    SET @Msg = CONCAT(
        N'Уведомления с ',
        CONVERT(varchar(10), @dt, 120),
        N' по ',
        CONVERT(varchar(10), DATEADD(day, -1, @End), 120)
    );

    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;


    DELETE FROM #rk26w_msg
    WHERE dt_rep >= @dt
      AND dt_rep < @End;


    INSERT INTO #rk26w_msg

    SELECT
        p.cli_id,
        CAST(p.msgbegindate AS date),

        MAX(CASE
            WHEN LOWER(p.message_type) = 'sms'
                THEN 1 ELSE 0
        END),

        MAX(CASE
            WHEN LOWER(p.message_type) = 'push'
                THEN 1 ELSE 0
        END)

    FROM ALM.ehd.VW_promocode_contracts_short p WITH (NOLOCK)

    INNER JOIN ALM_TEST.WORK.RK_Oct2026_ClientRegistry c
        ON c.cli_id = p.cli_id

    WHERE p.msgbegindate >= @dt
      AND p.msgbegindate < @End

      AND LOWER(p.message_type) IN ('sms','push')

      AND c.client_group IN (
          N'новый клиент масс',
          N'старый клиент масс кто принес новые деньги',
          N'старый клиент масс кто не принес новые деньги'
      )

    GROUP BY
        p.cli_id,
        CAST(p.msgbegindate AS date)

    OPTION (MAXDOP 1, RECOMPILE);


    IF @End > @DateTo

        UPDATE #rk26w_job
        SET stage = 2,
            next_dt = @TriggerFrom;

    ELSE

        UPDATE #rk26w_job
        SET next_dt = @End;

END


/* =====================================================
   STAGE 2. ТРИГГЕРЫ, ПРИЛОЖЕНИЯ И ЗАКРЫТИЯ ВКЛАДОВ
   ===================================================== */

ELSE IF @Stage = 2
BEGIN

    WHILE @dt <= @DateTo
      AND @Done < @DaysPerRun
    BEGIN

        IF DATEDIFF(second, @RunStart, SYSDATETIME())
           >= @MaxRunSeconds
            BREAK;

        SET @Msg = CONCAT(
            N'Триггеры и пуши: ',
            CONVERT(varchar(10), @dt, 120)
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        TRUNCATE TABLE #rk26w_scan;


        /* Все триггерные платежи.
           Дополнительно включаем получателей фактических
           пушей для поиска закрытий вкладов. */

        ;WITH source_events AS (

            SELECT
                t.cli_id,
                COUNT_BIG(*) AS trigger_rows

            FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

            INNER JOIN ALM_TEST.WORK.RK_Oct2026_ClientRegistry c
                ON c.cli_id = t.cli_id

            WHERE t.dt_rep = @dt

              AND t.direction_type = N'Переводы себе'
              AND t.transaction_type = N'СБП-Перевод'
              AND t.direction = -1
              AND t.amount_out <= -1000000

              AND c.client_group IN (
                  N'новый клиент масс',
                  N'старый клиент масс кто принес новые деньги',
                  N'старый клиент масс кто не принес новые деньги'
              )

            GROUP BY t.cli_id


            UNION ALL


            SELECT
                m.cli_id,
                CAST(0 AS bigint)

            FROM #rk26w_msg m

            WHERE m.dt_rep = @dt
              AND m.push_fact_day = 1

        )

        INSERT INTO #rk26w_scan

        SELECT
            @dt,
            cli_id,
            SUM(trigger_rows),

            0, 0, 0, 0,

            CAST(NULL AS date),

            0,

            0, 0,

            0, 0

        FROM source_events

        GROUP BY cli_id

        OPTION (MAXDOP 1, RECOMPILE);


        IF EXISTS (SELECT 1 FROM #rk26w_scan)
        BEGIN

            /* Версии приложений проверяем
               только в даты триггерных платежей */

            RAISERROR(N'Проверка приложений', 0, 1)
            WITH NOWAIT;


            ;WITH devices AS (

                SELECT
                    s.cli_id,

                    LOWER(LTRIM(RTRIM(d.OS))) AS os_name,

                    LTRIM(RTRIM(d.APP_VERSION)) AS app_version

                FROM ALM.ehd.ods_cli_devices d WITH (NOLOCK)

                INNER JOIN #rk26w_scan s
                    ON s.cli_id = d.CLI_ID
                   AND s.trigger_rows > 0

                WHERE d.LOG_TIMESTAMP >= @dt
                  AND d.LOG_TIMESTAMP < DATEADD(day, 1, @dt)

            ),

            normalized AS (

                SELECT
                    cli_id,
                    os_name,

                    CASE
                        WHEN LEN(app_version)
                           - LEN(REPLACE(app_version, '.', '')) = 2
                            THEN app_version + '.0'

                        WHEN LEN(app_version)
                           - LEN(REPLACE(app_version, '.', '')) = 3
                            THEN app_version

                    END AS ver4

                FROM devices

            ),

            parsed AS (

                SELECT
                    cli_id,
                    os_name,

                    TRY_CONVERT(int, PARSENAME(ver4, 4))
                        AS major_ver,

                    TRY_CONVERT(int, PARSENAME(ver4, 3))
                        AS minor_ver,

                    TRY_CONVERT(int, PARSENAME(ver4, 2))
                        AS patch_ver,

                    TRY_CONVERT(int, PARSENAME(ver4, 1))
                        AS build_ver

                FROM normalized

            ),

            classified AS (

                SELECT
                    cli_id,

                    CASE

                        /* ОС NULL считаем шторкой */

                        WHEN os_name IS NULL
                            THEN 2

                        WHEN os_name NOT IN ('android','ios')
                          OR major_ver IS NULL
                          OR minor_ver IS NULL
                          OR patch_ver IS NULL
                          OR build_ver IS NULL
                          OR major_ver < 0
                          OR minor_ver < 0
                          OR patch_ver < 0
                          OR build_ver < 0
                            THEN 0

                        /* Новая версия от 5.7.1 */

                        WHEN major_ver > 5

                          OR (
                              major_ver = 5
                              AND minor_ver > 7
                          )

                          OR (
                              major_ver = 5
                              AND minor_ver = 7
                              AND patch_ver >= 1
                          )

                            THEN 2

                        /* Старая версия */

                        ELSE 1

                    END AS app_class

                FROM parsed

            ),

            app_flags AS (

                SELECT
                    cli_id,

                    MAX(CASE
                        WHEN app_class = 1 THEN 1 ELSE 0
                    END) AS has_old_app,

                    MAX(CASE
                        WHEN app_class = 2 THEN 1 ELSE 0
                    END) AS has_new_app,

                    MAX(CASE
                        WHEN app_class = 0 THEN 1 ELSE 0
                    END) AS has_unknown_app

                FROM classified

                GROUP BY cli_id

            )

            UPDATE s

            SET s.has_app_activity = 1,
                s.has_old_app = a.has_old_app,
                s.has_new_app = a.has_new_app,
                s.has_unknown_app = a.has_unknown_app

            FROM #rk26w_scan s

            INNER JOIN app_flags a
                ON a.cli_id = s.cli_id

            OPTION (MAXDOP 1, RECOMPILE);


            /* Закрытия вкладов для триггеров
               и фактически зарегистрированных пушей.
               Любая конвенция. */

            RAISERROR(N'Проверка закрытий вкладов', 0, 1)
            WITH NOWAIT;


            ;WITH closed AS (

                SELECT
                    s.cli_id,

                    MAX(CAST(h.DT_CLOSE AS date))
                        AS close_dt

                FROM LIQUIDITY.liq.DepositContract_all h
                    WITH (NOLOCK)

                INNER JOIN #rk26w_scan s
                    ON s.cli_id = h.CLI_ID

                WHERE h.DT_CLOSE >= DATEADD(day, -2, @dt)
                  AND h.DT_CLOSE < DATEADD(day, 1, @dt)

                  AND h.cli_short_name = N'ФЛ'
                  AND h.cur = 'RUR'

                  AND h.PROD_NAME NOT LIKE N'Накопительный счёт%'

                  AND h.PROD_NAME NOT IN (
                      N'Агентские эскроу ФЛ по ставке КС+спред',
                      N'Агентские эскроу ФЛ по ставке КС/2',
                      N'Спец. банк.счёт',
                      N'Спец. банк. счёт',
                      N'Залоговый',
                      N'Эскроу',
                      N'Депозит ФЛ (суррогатный договор для счета Новой Афины)',
                      N'Тип договора не определен'
                  )

                GROUP BY s.cli_id

            )

            UPDATE s

            SET s.close_dt = c.close_dt

            FROM #rk26w_scan s

            INNER JOIN closed c
                ON c.cli_id = s.cli_id

            OPTION (MAXDOP 1, RECOMPILE);


            /* Ограничение повторной фактической СМС */

            UPDATE s

            SET sms_cooldown = 1

            FROM #rk26w_scan s

            WHERE s.trigger_rows > 0

              AND EXISTS (

                  SELECT 1

                  FROM #rk26w_msg m

                  WHERE m.cli_id = s.cli_id
                    AND m.sms_fact_day = 1

                    AND m.dt_rep BETWEEN DATEADD(day, -6, @dt)
                                     AND DATEADD(day, -1, @dt)

              )

            OPTION (MAXDOP 1, RECOMPILE);


            /* Предварительные потенциальные права.
               Повторные окна СМС отсекаются на этапе 3. */

            UPDATE #rk26w_scan

            SET sms_candidate =

                CASE
                    WHEN trigger_rows > 0
                     AND has_old_app = 1
                     AND sms_cooldown = 0
                        THEN 1 ELSE 0
                END,

                push_eligible =

                CASE
                    WHEN trigger_rows > 0
                     AND has_new_app = 1
                     AND close_dt IS NOT NULL
                        THEN 1 ELSE 0
                END,

                sms_unknown =

                CASE
                    WHEN trigger_rows > 0
                     AND has_old_app = 0
                     AND sms_cooldown = 0

                     AND (
                         has_app_activity = 0
                         OR has_unknown_app = 1
                     )

                        THEN 1 ELSE 0
                END,

                push_unknown =

                CASE
                    WHEN trigger_rows > 0
                     AND has_new_app = 0
                     AND close_dt IS NOT NULL

                     AND (
                         has_app_activity = 0
                         OR has_unknown_app = 1
                     )

                        THEN 1 ELSE 0
                END;

        END;


        DELETE FROM #rk26w_event
        WHERE dt_rep = @dt;

        INSERT INTO #rk26w_event
        SELECT *
        FROM #rk26w_scan;


        SET @Msg = CONCAT(
            N'Готово: ',
            CONVERT(varchar(10), @dt, 120)
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;


        SET @Done = @Done + 1;
        SET @dt = DATEADD(day, 1, @dt);

        UPDATE #rk26w_job
        SET next_dt = @dt;

    END;


    IF @dt > @DateTo

        UPDATE #rk26w_job
        SET stage = 3,
            next_dt = @DateFrom;

END


/* =====================================================
   STAGE 3. ДИНАМИЧЕСКИЕ ОКНА И ПОДНЕВНЫЕ ФЛАГИ
   ===================================================== */

ELSE IF @Stage = 3
BEGIN

    DROP TABLE IF EXISTS #rk26w_sms_start;
    DROP TABLE IF EXISTS #rk26w_windows;
    DROP TABLE IF EXISTS #rk26w_dayflags;
    DROP TABLE IF EXISTS #rk26w_bank;
    DROP TABLE IF EXISTS #rk26w_agg;


    /* Начала потенциальных окон СМС.
       1 = подтвержденное право.
       0 = право невозможно определить. */

    CREATE TABLE #rk26w_sms_start (
        cli_id bigint NOT NULL,
        dt_rep date NOT NULL,
        is_confirmed int NOT NULL,

        PRIMARY KEY (cli_id, dt_rep, is_confirmed)
    );


    RAISERROR(N'Формируем непересекающиеся окна СМС', 0, 1)
    WITH NOWAIT;


    SET @dt = @TriggerFrom;

    WHILE @dt <= @DateTo
    BEGIN

        /* Подтвержденные потенциальные окна.
           Повторные триггеры внутри окна игнорируем. */

        INSERT INTO #rk26w_sms_start

        SELECT
            t.cli_id,
            @dt,
            1

        FROM #rk26w_event t

        WHERE t.dt_rep = @dt
          AND t.sms_candidate = 1

          AND NOT EXISTS (

              SELECT 1

              FROM #rk26w_sms_start s

              WHERE s.cli_id = t.cli_id
                AND s.is_confirmed = 1

                AND s.dt_rep BETWEEN DATEADD(day, -6, @dt)
                                 AND DATEADD(day, -1, @dt)

          )

        OPTION (MAXDOP 1, RECOMPILE);


        /* Неопределенные окна.
           Не продлеваем повторными триггерами.
           Не блокируем ими подтвержденное право. */

        INSERT INTO #rk26w_sms_start

        SELECT
            t.cli_id,
            @dt,
            0

        FROM #rk26w_event t

        WHERE t.dt_rep = @dt
          AND t.sms_unknown = 1

          AND NOT EXISTS (

              SELECT 1

              FROM #rk26w_sms_start s

              WHERE s.cli_id = t.cli_id

                AND s.dt_rep BETWEEN DATEADD(day, -6, @dt)
                                 AND @dt

          )

        OPTION (MAXDOP 1, RECOMPILE);


        SET @dt = DATEADD(day, 1, @dt);

    END;


    /* Все потенциальные и фактические окна */

    RAISERROR(N'Формируем динамические окна', 0, 1)
    WITH NOWAIT;


    ;WITH nums AS (

        SELECT n

        FROM (
            VALUES (0),(1),(2),(3),(4),(5),(6)
        ) v(n)

    ),

    windows AS (

        /* Потенциальная СМС.
           Триггер + 6 дней. */

        SELECT
            s.cli_id,
            DATEADD(day, n.n, s.dt_rep) AS dt_rep,

            s.is_confirmed AS sms_window,
            0 AS push_window,

            1 - s.is_confirmed AS sms_unknown_window,
            0 AS push_unknown_window,

            0 AS sms_offer_window,
            0 AS push_offer_window,

            0 AS sms_cooldown_day

        FROM #rk26w_sms_start s

        CROSS JOIN nums n


        UNION ALL


        /* Потенциальный пуш.
           От триггера до закрытия + 2 дня. */

        SELECT
            t.cli_id,
            DATEADD(day, n.n, t.dt_rep),

            0,
            t.push_eligible,

            0,
            t.push_unknown,

            0, 0,

            0

        FROM #rk26w_event t

        CROSS JOIN nums n

        WHERE n.n <= 2

          AND (
              t.push_eligible = 1
              OR t.push_unknown = 1
          )

          AND DATEADD(day, n.n, t.dt_rep)
              <= DATEADD(day, 2, t.close_dt)


        UNION ALL


        /* Фактическая СМС.
           Дата отправки + 6 дней. */

        SELECT
            m.cli_id,
            DATEADD(day, n.n, m.dt_rep),

            0, 0,
            0, 0,

            1, 0,

            CASE WHEN n.n > 0 THEN 1 ELSE 0 END

        FROM #rk26w_msg m

        CROSS JOIN nums n

        WHERE m.sms_fact_day = 1


        UNION ALL


        /* Фактический пуш.
           От регистрации до закрытия + 2 дня.
           Если закрытие не найдено - только дата пуша. */

        SELECT
            m.cli_id,
            DATEADD(day, n.n, m.dt_rep),

            0, 0,
            0, 0,

            0, 1,

            0

        FROM #rk26w_msg m

        LEFT JOIN #rk26w_event e
            ON e.cli_id = m.cli_id
           AND e.dt_rep = m.dt_rep

        CROSS JOIN nums n

        WHERE m.push_fact_day = 1
          AND n.n <= 2

          AND DATEADD(day, n.n, m.dt_rep)
              <= ISNULL(
                  DATEADD(day, 2, e.close_dt),
                  m.dt_rep
              )

    )

    SELECT
        cli_id,
        dt_rep,

        MAX(sms_window) AS sms_window,
        MAX(push_window) AS push_window,

        CASE
            WHEN MAX(sms_window) = 1 THEN 0
            ELSE MAX(sms_unknown_window)
        END AS sms_unknown_window,

        CASE
            WHEN MAX(push_window) = 1 THEN 0
            ELSE MAX(push_unknown_window)
        END AS push_unknown_window,

        MAX(sms_offer_window) AS sms_offer_window,
        MAX(push_offer_window) AS push_offer_window,

        MAX(sms_cooldown_day) AS sms_cooldown_day

    INTO #rk26w_windows

    FROM windows

    WHERE dt_rep BETWEEN @DateFrom AND @DateTo

    GROUP BY cli_id, dt_rep

    OPTION (MAXDOP 1, RECOMPILE);


    CREATE UNIQUE CLUSTERED INDEX IX_rk26w_windows
    ON #rk26w_windows (dt_rep, cli_id);


    /* Одна строка на клиента и календарную дату */

    RAISERROR(N'Формируем подневные флаги', 0, 1)
    WITH NOWAIT;


    ;WITH client_days AS (

        SELECT cli_id, dt_rep
        FROM #rk26w_event
        WHERE dt_rep BETWEEN @DateFrom AND @DateTo

        UNION

        SELECT cli_id, dt_rep
        FROM #rk26w_msg
        WHERE dt_rep BETWEEN @DateFrom AND @DateTo

        UNION

        SELECT cli_id, dt_rep
        FROM #rk26w_windows

    )

    SELECT
        d.dt_rep,
        d.cli_id,

        CASE WHEN e.trigger_rows > 0
            THEN 1 ELSE 0
        END AS trigger_day,

        ISNULL(e.has_app_activity, 0) AS has_app_activity,
        ISNULL(e.has_old_app, 0) AS has_old_app,
        ISNULL(e.has_new_app, 0) AS has_new_app,
        ISNULL(e.has_unknown_app, 0) AS has_unknown_app,

        CASE WHEN e.close_dt IS NOT NULL
            THEN 1 ELSE 0
        END AS has_closed_deposit,

        ISNULL(e.sms_candidate, 0) AS sms_candidate_day,

        CASE
            WHEN e.sms_candidate = 1
             AND s.cli_id IS NULL
                THEN 1 ELSE 0
        END AS sms_repeat_blocked_day,

        CASE WHEN s.cli_id IS NOT NULL
            THEN 1 ELSE 0
        END AS sms_eligible_day,

        ISNULL(e.push_eligible, 0) AS push_eligible_day,

        ISNULL(e.sms_unknown, 0) AS sms_unknown_day,
        ISNULL(e.push_unknown, 0) AS push_unknown_day,

        ISNULL(m.sms_fact_day, 0) AS sms_fact_day,
        ISNULL(m.push_fact_day, 0) AS push_fact_day,

        CASE
            WHEN m.push_fact_day = 1
             AND e.close_dt IS NULL
                THEN 1 ELSE 0
        END AS push_fact_unmatched_day,

        ISNULL(w.sms_window, 0) AS sms_window,
        ISNULL(w.push_window, 0) AS push_window,

        ISNULL(w.sms_unknown_window, 0) AS sms_unknown_window,
        ISNULL(w.push_unknown_window, 0) AS push_unknown_window,

        ISNULL(w.sms_offer_window, 0) AS sms_offer_window,
        ISNULL(w.push_offer_window, 0) AS push_offer_window,

        ISNULL(w.sms_cooldown_day, 0) AS sms_cooldown_day

    INTO #rk26w_dayflags

    FROM client_days d

    LEFT JOIN #rk26w_event e
        ON e.cli_id = d.cli_id
       AND e.dt_rep = d.dt_rep

    LEFT JOIN #rk26w_sms_start s
        ON s.cli_id = d.cli_id
       AND s.dt_rep = d.dt_rep
       AND s.is_confirmed = 1

    LEFT JOIN #rk26w_msg m
        ON m.cli_id = d.cli_id
       AND m.dt_rep = d.dt_rep

    LEFT JOIN #rk26w_windows w
        ON w.cli_id = d.cli_id
       AND w.dt_rep = d.dt_rep

    OPTION (MAXDOP 1, RECOMPILE);


    CREATE UNIQUE CLUSTERED INDEX IX_rk26w_dayflags
    ON #rk26w_dayflags (dt_rep, cli_id);


    /* Классификация банков.
       Порог 200 млн по распознанным СР, НС, ДВС
       за весь период РК. */

    RAISERROR(N'Классифицируем банки', 0, 1)
    WITH NOWAIT;


    SELECT
        bank_name_main,

        CASE
            WHEN ABS(SUM(
                incoming_sr + incoming_ns + incoming_dvs
            )) <= @threshold

             AND ABS(SUM(
                outgoing_sr + outgoing_ns + outgoing_dvs
            )) <= @threshold

                THEN N'else'

            ELSE bank_name_main

        END AS bank_group

    INTO #rk26w_bank

    FROM #rk26p_cli

    WHERE bank_type = 'bank'

    GROUP BY bank_name_main

    OPTION (MAXDOP 1, RECOMPILE);


    /* Таблица итоговых агрегатов */

    SELECT TOP (0)
        p.dt_rep,
        p.client_group,

        p.bank_name_main,
        p.bank_type,
        p.gr_size,

        p.payment_type,
        p.transaction_type,
        p.operation_type,
        p.payment_direction,
        p.cli_biz,

        p.target_flag,
        p.sbp_self_1m,

        CAST(0 AS int) AS trigger_day,

        CAST(0 AS int) AS has_app_activity,
        CAST(0 AS int) AS has_old_app,
        CAST(0 AS int) AS has_new_app,
        CAST(0 AS int) AS has_unknown_app,

        CAST(0 AS int) AS has_closed_deposit,

        CAST(0 AS int) AS sms_candidate_day,
        CAST(0 AS int) AS sms_repeat_blocked_day,

        CAST(0 AS int) AS sms_eligible_day,
        CAST(0 AS int) AS push_eligible_day,

        CAST(0 AS int) AS sms_unknown_day,
        CAST(0 AS int) AS push_unknown_day,

        CAST(0 AS int) AS sms_fact_day,
        CAST(0 AS int) AS push_fact_day,

        CAST(0 AS int) AS push_fact_unmatched_day,

        CAST(0 AS int) AS sms_window,
        CAST(0 AS int) AS push_window,

        CAST(0 AS int) AS sms_unknown_window,
        CAST(0 AS int) AS push_unknown_window,

        CAST(0 AS int) AS sms_offer_window,
        CAST(0 AS int) AS push_offer_window,

        CAST(0 AS int) AS sms_cooldown_day,

        p.source_rows,
        p.count_in,
        p.count_out,

        p.amount_in,
        p.amount_out,

        p.incoming_sr,
        p.outgoing_sr,

        p.incoming_ns,
        p.outgoing_ns,

        p.incoming_dvs,
        p.outgoing_dvs,

        p.incoming_other,
        p.outgoing_other,

        p.incoming_unknown,
        p.outgoing_unknown

    INTO #rk26w_agg

    FROM #rk26p_cli p;


    CREATE CLUSTERED INDEX IX_rk26w_agg
    ON #rk26w_agg (dt_rep);


    UPDATE #rk26w_job
    SET stage = 4,
        next_dt = @DateFrom;

END


/* =====================================================
   STAGE 4. МАРКИРОВКА И АГРЕГАЦИЯ ПЛАТЕЖЕЙ
   ===================================================== */

ELSE IF @Stage = 4
BEGIN

    WHILE @dt <= @DateTo
      AND @Done < @DaysPerRun
    BEGIN

        IF DATEDIFF(second, @RunStart, SYSDATETIME())
           >= @MaxRunSeconds
            BREAK;

        SET @Msg = CONCAT(
            N'Агрегация: ',
            CONVERT(varchar(10), @dt, 120)
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;


        DELETE FROM #rk26w_agg
        WHERE dt_rep = @dt;


        ;WITH marked AS (

            SELECT
                p.*,

                ISNULL(b.bank_group, p.bank_name_main)
                    AS bank_group,

                ISNULL(f.trigger_day, 0) AS trigger_day,

                ISNULL(f.has_app_activity, 0) AS has_app_activity,
                ISNULL(f.has_old_app, 0) AS has_old_app,
                ISNULL(f.has_new_app, 0) AS has_new_app,
                ISNULL(f.has_unknown_app, 0) AS has_unknown_app,

                ISNULL(f.has_closed_deposit, 0) AS has_closed_deposit,

                ISNULL(f.sms_candidate_day, 0) AS sms_candidate_day,

                ISNULL(f.sms_repeat_blocked_day, 0)
                    AS sms_repeat_blocked_day,

                ISNULL(f.sms_eligible_day, 0) AS sms_eligible_day,
                ISNULL(f.push_eligible_day, 0) AS push_eligible_day,

                ISNULL(f.sms_unknown_day, 0) AS sms_unknown_day,
                ISNULL(f.push_unknown_day, 0) AS push_unknown_day,

                ISNULL(f.sms_fact_day, 0) AS sms_fact_day,
                ISNULL(f.push_fact_day, 0) AS push_fact_day,

                ISNULL(f.push_fact_unmatched_day, 0)
                    AS push_fact_unmatched_day,

                ISNULL(f.sms_window, 0) AS sms_window,
                ISNULL(f.push_window, 0) AS push_window,

                ISNULL(f.sms_unknown_window, 0) AS sms_unknown_window,
                ISNULL(f.push_unknown_window, 0) AS push_unknown_window,

                ISNULL(f.sms_offer_window, 0) AS sms_offer_window,
                ISNULL(f.push_offer_window, 0) AS push_offer_window,

                ISNULL(f.sms_cooldown_day, 0) AS sms_cooldown_day

            FROM #rk26p_cli p

            LEFT JOIN #rk26w_dayflags f
                ON f.cli_id = p.cli_id
               AND f.dt_rep = p.dt_rep

            LEFT JOIN #rk26w_bank b
                ON b.bank_name_main = p.bank_name_main
               AND p.bank_type = 'bank'

            WHERE p.dt_rep = @dt

        )

        INSERT INTO #rk26w_agg

        SELECT
            dt_rep,
            client_group,

            bank_group,
            bank_type,
            gr_size,

            payment_type,
            transaction_type,
            operation_type,
            payment_direction,
            cli_biz,

            target_flag,
            sbp_self_1m,

            trigger_day,

            has_app_activity,
            has_old_app,
            has_new_app,
            has_unknown_app,

            has_closed_deposit,

            sms_candidate_day,
            sms_repeat_blocked_day,

            sms_eligible_day,
            push_eligible_day,

            sms_unknown_day,
            push_unknown_day,

            sms_fact_day,
            push_fact_day,

            push_fact_unmatched_day,

            sms_window,
            push_window,

            sms_unknown_window,
            push_unknown_window,

            sms_offer_window,
            push_offer_window,

            sms_cooldown_day,

            SUM(source_rows),
            SUM(count_in),
            SUM(count_out),

            SUM(amount_in),
            SUM(amount_out),

            SUM(incoming_sr),
            SUM(outgoing_sr),

            SUM(incoming_ns),
            SUM(outgoing_ns),

            SUM(incoming_dvs),
            SUM(outgoing_dvs),

            SUM(incoming_other),
            SUM(outgoing_other),

            SUM(incoming_unknown),
            SUM(outgoing_unknown)

        FROM marked

        GROUP BY
            dt_rep,
            client_group,

            bank_group,
            bank_type,
            gr_size,

            payment_type,
            transaction_type,
            operation_type,
            payment_direction,
            cli_biz,

            target_flag,
            sbp_self_1m,

            trigger_day,

            has_app_activity,
            has_old_app,
            has_new_app,
            has_unknown_app,

            has_closed_deposit,

            sms_candidate_day,
            sms_repeat_blocked_day,

            sms_eligible_day,
            push_eligible_day,

            sms_unknown_day,
            push_unknown_day,

            sms_fact_day,
            push_fact_day,

            push_fact_unmatched_day,

            sms_window,
            push_window,

            sms_unknown_window,
            push_unknown_window,

            sms_offer_window,
            push_offer_window,

            sms_cooldown_day

        OPTION (MAXDOP 1, RECOMPILE);


        SET @Done = @Done + 1;

        SET @Msg = CONCAT(
            N'Агрегация готова: ',
            CONVERT(varchar(10), @dt, 120)
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;


        SET @dt = DATEADD(day, 1, @dt);

        UPDATE #rk26w_job
        SET next_dt = @dt;

    END;


    IF @dt > @DateTo

        UPDATE #rk26w_job
        SET stage = 5,
            next_dt = NULL;

END;


/* СТАТУС */

SELECT
    @@SPID AS session_id,

    stage,

    CASE stage
        WHEN 1 THEN N'Фактические уведомления'
        WHEN 2 THEN N'Триггеры, приложения и закрытия'
        WHEN 3 THEN N'Динамические окна'
        WHEN 4 THEN N'Агрегация платежей'
        WHEN 5 THEN N'Готово'
    END AS stage_name,

    next_dt,

    (SELECT COUNT_BIG(*) FROM #rk26w_msg)
        AS message_client_days,

    (SELECT COUNT_BIG(*) FROM #rk26w_event)
        AS event_client_days,

    DATEDIFF(second, @RunStart, SYSDATETIME())
        AS run_seconds

FROM #rk26w_job;
