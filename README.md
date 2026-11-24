
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



USE ALM;
SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @DateFrom date = '2026-10-01';
DECLARE @DateTo date = '2026-11-04';

DECLARE @SMSFrom date = DATEADD(day, -6, @DateFrom);
DECLARE @PushFrom date = DATEADD(day, -2, @DateFrom);

DECLARE @TriggerFrom date = @SMSFrom;
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

IF OBJECT_ID('tempdb..#rk26r_job') IS NULL
BEGIN

    DROP TABLE IF EXISTS #rk26r_msg;
    DROP TABLE IF EXISTS #rk26r_trigger;
    DROP TABLE IF EXISTS #rk26r_scan;

    CREATE TABLE #rk26r_msg (
        cli_id bigint NOT NULL,
        dt_rep date NOT NULL,
        sms_fact_day int NOT NULL,
        push_fact_day int NOT NULL,

        PRIMARY KEY (cli_id, dt_rep)
    );


    CREATE TABLE #rk26r_trigger (
        dt_rep date NOT NULL,
        cli_id bigint NOT NULL,

        trigger_rows bigint NOT NULL,

        has_app_activity int NOT NULL,
        has_old_app int NOT NULL,
        has_new_app int NOT NULL,
        has_unknown_app int NOT NULL,

        close_dt date NULL,

        sms_cooldown int NOT NULL,

        sms_eligible int NOT NULL,
        push_eligible int NOT NULL,

        sms_unknown int NOT NULL,
        push_unknown int NOT NULL,

        PRIMARY KEY (dt_rep, cli_id)
    );


    SELECT TOP (0) *
    INTO #rk26r_scan
    FROM #rk26r_trigger;

    CREATE UNIQUE CLUSTERED INDEX IX_rk26r_scan
    ON #rk26r_scan (cli_id);


    CREATE TABLE #rk26r_job (
        stage int NOT NULL,
        next_dt date NULL
    );

    INSERT INTO #rk26r_job
    VALUES (1, @MessageFrom);

END;


IF OBJECT_ID('tempdb..#rk26r_msg') IS NULL
   OR OBJECT_ID('tempdb..#rk26r_trigger') IS NULL
   OR OBJECT_ID('tempdb..#rk26r_scan') IS NULL
    THROW 50004, 'Incomplete calculation state', 1;


SELECT
    @Stage = stage,
    @dt = next_dt
FROM #rk26r_job;


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


    DELETE FROM #rk26r_msg
    WHERE dt_rep >= @dt
      AND dt_rep < @End;


    INSERT INTO #rk26r_msg

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

        UPDATE #rk26r_job
        SET stage = 2,
            next_dt = @TriggerFrom;

    ELSE

        UPDATE #rk26r_job
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
            N'Триггеры: ',
            CONVERT(varchar(10), @dt, 120)
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        TRUNCATE TABLE #rk26r_scan;


        /* Все подходящие СБП-переводы.
           Фильтры портфельных платежей не применяем. */

        INSERT INTO #rk26r_scan

        SELECT
            @dt,
            t.cli_id,

            COUNT_BIG(*),

            0, 0, 0, 0,

            CAST(NULL AS date),

            0,

            0, 0,

            0, 0

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

        OPTION (MAXDOP 1, RECOMPILE);


        IF EXISTS (SELECT 1 FROM #rk26r_scan)
        BEGIN

            /* АКТИВНОСТЬ В ПРИЛОЖЕНИЯХ.
               Проверяем все события в день перевода. */

            RAISERROR(N'Проверка версий приложений', 0, 1)
            WITH NOWAIT;


            ;WITH devices AS (

                SELECT
                    s.cli_id,

                    LOWER(LTRIM(RTRIM(d.OS))) AS os_name,

                    LTRIM(RTRIM(d.APP_VERSION)) AS app_version

                FROM ALM.ehd.ods_cli_devices d WITH (NOLOCK)

                INNER JOIN #rk26r_scan s
                    ON s.cli_id = d.CLI_ID

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

                        /* Неопределенная версия */

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

                        /* Версия от 5.7.1 */

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

            FROM #rk26r_scan s

            INNER JOIN app_flags a
                ON a.cli_id = s.cli_id

            OPTION (MAXDOP 1, RECOMPILE);


            /* ЗАКРЫТИЯ ВКЛАДОВ.
               Любая конвенция.
               Закрытие в день перевода или за два дня. */

            RAISERROR(N'Проверка закрытий вкладов', 0, 1)
            WITH NOWAIT;


            ;WITH closed AS (

                SELECT
                    s.cli_id,

                    MAX(CAST(h.DT_CLOSE AS date))
                        AS close_dt

                FROM LIQUIDITY.liq.DepositContract_all h
                    WITH (NOLOCK)

                INNER JOIN #rk26r_scan s
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

            FROM #rk26r_scan s

            INNER JOIN closed c
                ON c.cli_id = s.cli_id

            OPTION (MAXDOP 1, RECOMPILE);


            /* ОГРАНИЧЕНИЕ ПОВТОРНОЙ СМС.
               Проверяем именно фактические уведомления. */

            UPDATE s

            SET sms_cooldown = 1

            FROM #rk26r_scan s

            WHERE EXISTS (

                SELECT 1

                FROM #rk26r_msg m

                WHERE m.cli_id = s.cli_id
                  AND m.sms_fact_day = 1

                  AND m.dt_rep BETWEEN DATEADD(day, -6, @dt)
                                   AND DATEADD(day, -1, @dt)

            )

            OPTION (MAXDOP 1, RECOMPILE);


            /* ПОТЕНЦИАЛЬНЫЕ ПРАВА.
               СМС и пуш считаем независимо. */

            UPDATE #rk26r_scan

            SET sms_eligible =

                CASE
                    WHEN has_old_app = 1
                     AND sms_cooldown = 0
                        THEN 1 ELSE 0
                END,

                push_eligible =

                CASE
                    WHEN has_new_app = 1
                     AND close_dt IS NOT NULL
                        THEN 1 ELSE 0
                END,

                sms_unknown =

                CASE
                    WHEN has_old_app = 0
                     AND sms_cooldown = 0

                     AND (
                         has_app_activity = 0
                         OR has_unknown_app = 1
                     )

                        THEN 1 ELSE 0
                END,

                push_unknown =

                CASE
                    WHEN has_new_app = 0
                     AND close_dt IS NOT NULL

                     AND (
                         has_app_activity = 0
                         OR has_unknown_app = 1
                     )

                        THEN 1 ELSE 0
                END;

        END;


        /* Сохраняем готовую дату */

        DELETE FROM #rk26r_trigger
        WHERE dt_rep = @dt;

        INSERT INTO #rk26r_trigger
        SELECT *
        FROM #rk26r_scan;


        SET @Msg = CONCAT(
            N'Триггеры готовы: ',
            CONVERT(varchar(10), @dt, 120)
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        SET @Done = @Done + 1;
        SET @dt = DATEADD(day, 1, @dt);

        UPDATE #rk26r_job
        SET next_dt = @dt;

    END;


    IF @dt > @DateTo

        UPDATE #rk26r_job
        SET stage = 3,
            next_dt = @DateFrom;

END


/* =====================================================
   STAGE 3. КЛИЕНТСКИЕ И ПОДНЕВНЫЕ ФЛАГИ
   ===================================================== */

ELSE IF @Stage = 3
BEGIN

    DROP TABLE IF EXISTS #rk26r_windows;
    DROP TABLE IF EXISTS #rk26r_ever;
    DROP TABLE IF EXISTS #rk26r_dayflags;
    DROP TABLE IF EXISTS #rk26r_bank;
    DROP TABLE IF EXISTS #rk26r_agg;


    /* ОКНА УДЕРЖАНИЯ.
       Потенциальная СМС: триггер + 6 дней.
       Потенциальный пуш: до закрытия + 2 дня.
       Фактическая СМС: отправка + 6 дней. */

    RAISERROR(N'Формируем окна удержания', 0, 1)
    WITH NOWAIT;


    ;WITH nums AS (

        SELECT n
        FROM (VALUES (0),(1),(2),(3),(4),(5),(6)) v(n)

    ),

    windows AS (

        SELECT
            t.cli_id,
            DATEADD(day, n.n, t.dt_rep) AS dt_rep,

            t.sms_eligible AS sms_window,
            0 AS push_window,

            t.sms_unknown AS sms_unknown_window,
            0 AS push_unknown_window,

            0 AS sms_offer_window,
            0 AS sms_cooldown_day

        FROM #rk26r_trigger t

        CROSS JOIN nums n

        WHERE t.sms_eligible = 1
           OR t.sms_unknown = 1


        UNION ALL


        SELECT
            t.cli_id,
            DATEADD(day, n.n, t.dt_rep),

            0,
            t.push_eligible,

            0,
            t.push_unknown,

            0, 0

        FROM #rk26r_trigger t

        CROSS JOIN nums n

        WHERE n.n <= 2

          AND (
              t.push_eligible = 1
              OR t.push_unknown = 1
          )

          AND DATEADD(day, n.n, t.dt_rep)
              <= DATEADD(day, 2, t.close_dt)


        UNION ALL


        SELECT
            m.cli_id,
            DATEADD(day, n.n, m.dt_rep),

            0, 0,

            0, 0,

            1,

            CASE WHEN n.n > 0 THEN 1 ELSE 0 END

        FROM #rk26r_msg m

        CROSS JOIN nums n

        WHERE m.sms_fact_day = 1

    )

    SELECT
        cli_id,
        dt_rep,

        MAX(sms_window) AS sms_window,
        MAX(push_window) AS push_window,

        MAX(sms_unknown_window) AS sms_unknown_window,
        MAX(push_unknown_window) AS push_unknown_window,

        MAX(sms_offer_window) AS sms_offer_window,
        MAX(sms_cooldown_day) AS sms_cooldown_day

    INTO #rk26r_windows

    FROM windows

    WHERE dt_rep BETWEEN @DateFrom AND @DateTo

    GROUP BY cli_id, dt_rep

    OPTION (MAXDOP 1, RECOMPILE);


    CREATE UNIQUE CLUSTERED INDEX IX_rk26r_windows
    ON #rk26r_windows (dt_rep, cli_id);


    /* ПОСТОЯННЫЕ ФЛАГИ КЛИЕНТОВ.
       Учитываем окна, пересекающие период РК. */

    ;WITH facts AS (

        SELECT
            cli_id,

            MIN(CASE
                WHEN sms_fact_day = 1
                 AND dt_rep >= @SMSFrom
                    THEN dt_rep
            END) AS first_sms_date,

            MIN(CASE
                WHEN push_fact_day = 1
                 AND dt_rep >= @PushFrom
                    THEN dt_rep
            END) AS first_push_date

        FROM #rk26r_msg

        GROUP BY cli_id

    ),

    eligible AS (

        SELECT
            cli_id,

            MAX(sms_window) AS sms_eligible_ever,
            MAX(push_window) AS push_eligible_ever,

            MAX(sms_unknown_window) AS sms_unknown_ever,
            MAX(push_unknown_window) AS push_unknown_ever

        FROM #rk26r_windows

        GROUP BY cli_id

    )

    SELECT
        ISNULL(f.cli_id, e.cli_id) AS cli_id,

        f.first_sms_date,
        f.first_push_date,

        CASE
            WHEN f.first_sms_date IS NOT NULL
                THEN 1 ELSE 0
        END AS sms_fact_ever,

        CASE
            WHEN f.first_push_date IS NOT NULL
                THEN 1 ELSE 0
        END AS push_fact_ever,

        ISNULL(e.sms_eligible_ever, 0)
            AS sms_eligible_ever,

        ISNULL(e.push_eligible_ever, 0)
            AS push_eligible_ever,

        ISNULL(e.sms_unknown_ever, 0)
            AS sms_unknown_ever,

        ISNULL(e.push_unknown_ever, 0)
            AS push_unknown_ever

    INTO #rk26r_ever

    FROM facts f

    FULL OUTER JOIN eligible e
        ON e.cli_id = f.cli_id

    OPTION (MAXDOP 1, RECOMPILE);


    CREATE UNIQUE CLUSTERED INDEX IX_rk26r_ever
    ON #rk26r_ever (cli_id);


    /* ПОДНЕВНЫЕ ФЛАГИ.
       Одна строка на клиента и дату.
       Не размножаем весь реестр клиентов на календарь. */

    RAISERROR(N'Формируем подневные флаги', 0, 1)
    WITH NOWAIT;


    ;WITH client_days AS (

        SELECT cli_id, dt_rep
        FROM #rk26r_trigger
        WHERE dt_rep BETWEEN @DateFrom AND @DateTo

        UNION

        SELECT cli_id, dt_rep
        FROM #rk26r_msg
        WHERE dt_rep BETWEEN @DateFrom AND @DateTo

        UNION

        SELECT cli_id, dt_rep
        FROM #rk26r_windows

    )

    SELECT
        d.dt_rep,
        d.cli_id,

        CASE
            WHEN t.cli_id IS NOT NULL THEN 1 ELSE 0
        END AS trigger_day,

        ISNULL(t.has_app_activity, 0)
            AS has_app_activity,

        ISNULL(t.has_old_app, 0)
            AS has_old_app,

        ISNULL(t.has_new_app, 0)
            AS has_new_app,

        ISNULL(t.has_unknown_app, 0)
            AS has_unknown_app,

        CASE
            WHEN t.close_dt IS NOT NULL THEN 1 ELSE 0
        END AS has_closed_deposit,

        ISNULL(t.sms_eligible, 0)
            AS sms_eligible_day,

        ISNULL(t.push_eligible, 0)
            AS push_eligible_day,

        ISNULL(t.sms_unknown, 0)
            AS sms_unknown_day,

        ISNULL(t.push_unknown, 0)
            AS push_unknown_day,

        ISNULL(m.sms_fact_day, 0)
            AS sms_fact_day,

        ISNULL(m.push_fact_day, 0)
            AS push_fact_day,

        ISNULL(w.sms_window, 0)
            AS sms_window,

        ISNULL(w.push_window, 0)
            AS push_window,

        ISNULL(w.sms_unknown_window, 0)
            AS sms_unknown_window,

        ISNULL(w.push_unknown_window, 0)
            AS push_unknown_window,

        ISNULL(w.sms_offer_window, 0)
            AS sms_offer_window,

        ISNULL(w.sms_cooldown_day, 0)
            AS sms_cooldown_day

    INTO #rk26r_dayflags

    FROM client_days d

    LEFT JOIN #rk26r_trigger t
        ON t.cli_id = d.cli_id
       AND t.dt_rep = d.dt_rep

    LEFT JOIN #rk26r_msg m
        ON m.cli_id = d.cli_id
       AND m.dt_rep = d.dt_rep

    LEFT JOIN #rk26r_windows w
        ON w.cli_id = d.cli_id
       AND w.dt_rep = d.dt_rep

    OPTION (MAXDOP 1, RECOMPILE);


    CREATE UNIQUE CLUSTERED INDEX IX_rk26r_dayflags
    ON #rk26r_dayflags (dt_rep, cli_id);


    /* КЛАССИФИКАЦИЯ БАНКОВ.
       Порог 200 млн по распознанным СР, НС, ДВС
       за весь период по трем группам MASS. */

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

    INTO #rk26r_bank

    FROM #rk26p_cli

    WHERE bank_type = 'bank'

    GROUP BY bank_name_main

    OPTION (MAXDOP 1, RECOMPILE);


    /* ТАБЛИЦА ИТОГОВЫХ АГРЕГАТОВ */

    SELECT TOP (0)
        dt_rep,
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

        CAST(0 AS int) AS sms_fact_ever,
        CAST(0 AS int) AS push_fact_ever,

        CAST(0 AS int) AS sms_eligible_ever,
        CAST(0 AS int) AS push_eligible_ever,

        CAST(0 AS int) AS sms_unknown_ever,
        CAST(0 AS int) AS push_unknown_ever,

        CAST(0 AS int) AS sms_fact_to_date,
        CAST(0 AS int) AS push_fact_to_date,

        CAST(0 AS int) AS trigger_day,

        CAST(0 AS int) AS has_app_activity,
        CAST(0 AS int) AS has_old_app,
        CAST(0 AS int) AS has_new_app,
        CAST(0 AS int) AS has_unknown_app,

        CAST(0 AS int) AS has_closed_deposit,

        CAST(0 AS int) AS sms_eligible_day,
        CAST(0 AS int) AS push_eligible_day,

        CAST(0 AS int) AS sms_unknown_day,
        CAST(0 AS int) AS push_unknown_day,

        CAST(0 AS int) AS sms_fact_day,
        CAST(0 AS int) AS push_fact_day,

        CAST(0 AS int) AS sms_window,
        CAST(0 AS int) AS push_window,

        CAST(0 AS int) AS sms_unknown_window,
        CAST(0 AS int) AS push_unknown_window,

        CAST(0 AS int) AS sms_offer_window,
        CAST(0 AS int) AS sms_cooldown_day,

        source_rows,
        count_in,
        count_out,

        amount_in,
        amount_out,

        incoming_sr,
        outgoing_sr,

        incoming_ns,
        outgoing_ns,

        incoming_dvs,
        outgoing_dvs,

        incoming_other,
        outgoing_other,

        incoming_unknown,
        outgoing_unknown

    INTO #rk26r_agg

    FROM #rk26p_cli;


    CREATE CLUSTERED INDEX IX_rk26r_agg
    ON #rk26r_agg (dt_rep);


    UPDATE #rk26r_job
    SET stage = 4,
        next_dt = @DateFrom;

END


/* =====================================================
   STAGE 4. МАРКИРОВКА И АГРЕГАЦИЯ ВСЕХ ПЛАТЕЖЕЙ
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


        DELETE FROM #rk26r_agg
        WHERE dt_rep = @dt;


        ;WITH marked AS (

            SELECT
                p.*,

                ISNULL(b.bank_group, p.bank_name_main)
                    AS bank_group,

                ISNULL(e.sms_fact_ever, 0)
                    AS sms_fact_ever,

                ISNULL(e.push_fact_ever, 0)
                    AS push_fact_ever,

                ISNULL(e.sms_eligible_ever, 0)
                    AS sms_eligible_ever,

                ISNULL(e.push_eligible_ever, 0)
                    AS push_eligible_ever,

                ISNULL(e.sms_unknown_ever, 0)
                    AS sms_unknown_ever,

                ISNULL(e.push_unknown_ever, 0)
                    AS push_unknown_ever,

                CASE
                    WHEN e.first_sms_date <= @dt
                        THEN 1 ELSE 0
                END AS sms_fact_to_date,

                CASE
                    WHEN e.first_push_date <= @dt
                        THEN 1 ELSE 0
                END AS push_fact_to_date,

                ISNULL(d.trigger_day, 0)
                    AS trigger_day,

                ISNULL(d.has_app_activity, 0)
                    AS has_app_activity,

                ISNULL(d.has_old_app, 0)
                    AS has_old_app,

                ISNULL(d.has_new_app, 0)
                    AS has_new_app,

                ISNULL(d.has_unknown_app, 0)
                    AS has_unknown_app,

                ISNULL(d.has_closed_deposit, 0)
                    AS has_closed_deposit,

                ISNULL(d.sms_eligible_day, 0)
                    AS sms_eligible_day,

                ISNULL(d.push_eligible_day, 0)
                    AS push_eligible_day,

                ISNULL(d.sms_unknown_day, 0)
                    AS sms_unknown_day,

                ISNULL(d.push_unknown_day, 0)
                    AS push_unknown_day,

                ISNULL(d.sms_fact_day, 0)
                    AS sms_fact_day,

                ISNULL(d.push_fact_day, 0)
                    AS push_fact_day,

                ISNULL(d.sms_window, 0)
                    AS sms_window,

                ISNULL(d.push_window, 0)
                    AS push_window,

                ISNULL(d.sms_unknown_window, 0)
                    AS sms_unknown_window,

                ISNULL(d.push_unknown_window, 0)
                    AS push_unknown_window,

                ISNULL(d.sms_offer_window, 0)
                    AS sms_offer_window,

                ISNULL(d.sms_cooldown_day, 0)
                    AS sms_cooldown_day

            FROM #rk26p_cli p

            LEFT JOIN #rk26r_ever e
                ON e.cli_id = p.cli_id

            LEFT JOIN #rk26r_dayflags d
                ON d.cli_id = p.cli_id
               AND d.dt_rep = p.dt_rep

            LEFT JOIN #rk26r_bank b
                ON b.bank_name_main = p.bank_name_main
               AND p.bank_type = 'bank'

            WHERE p.dt_rep = @dt

        )

        INSERT INTO #rk26r_agg

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

            sms_fact_ever,
            push_fact_ever,

            sms_eligible_ever,
            push_eligible_ever,

            sms_unknown_ever,
            push_unknown_ever,

            sms_fact_to_date,
            push_fact_to_date,

            trigger_day,

            has_app_activity,
            has_old_app,
            has_new_app,
            has_unknown_app,

            has_closed_deposit,

            sms_eligible_day,
            push_eligible_day,

            sms_unknown_day,
            push_unknown_day,

            sms_fact_day,
            push_fact_day,

            sms_window,
            push_window,

            sms_unknown_window,
            push_unknown_window,

            sms_offer_window,
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

            sms_fact_ever,
            push_fact_ever,

            sms_eligible_ever,
            push_eligible_ever,

            sms_unknown_ever,
            push_unknown_ever,

            sms_fact_to_date,
            push_fact_to_date,

            trigger_day,

            has_app_activity,
            has_old_app,
            has_new_app,
            has_unknown_app,

            has_closed_deposit,

            sms_eligible_day,
            push_eligible_day,

            sms_unknown_day,
            push_unknown_day,

            sms_fact_day,
            push_fact_day,

            sms_window,
            push_window,

            sms_unknown_window,
            push_unknown_window,

            sms_offer_window,
            sms_cooldown_day

        OPTION (MAXDOP 1, RECOMPILE);


        SET @Done = @Done + 1;
        SET @dt = DATEADD(day, 1, @dt);

        UPDATE #rk26r_job
        SET next_dt = @dt;

    END;


    IF @dt > @DateTo

        UPDATE #rk26r_job
        SET stage = 5,
            next_dt = NULL;

END;


/* СТАТУС ВЫПОЛНЕНИЯ */

SELECT
    @@SPID AS session_id,

    stage,

    CASE stage
        WHEN 1 THEN N'Фактические уведомления'
        WHEN 2 THEN N'Триггеры, приложения и закрытия'
        WHEN 3 THEN N'Клиентские флаги'
        WHEN 4 THEN N'Агрегация платежей'
        WHEN 5 THEN N'Готово'
    END AS stage_name,

    next_dt,

    (SELECT COUNT_BIG(*) FROM #rk26r_msg)
        AS message_client_days,

    (SELECT COUNT_BIG(*) FROM #rk26r_trigger)
        AS trigger_client_days,

    DATEDIFF(second, @RunStart, SYSDATETIME())
        AS run_seconds

FROM #rk26r_job;



IF NOT EXISTS (
    SELECT 1
    FROM #rk26r_job
    WHERE stage = 5
)
    THROW 50001, 'Расчет не завершен', 1;


SELECT
    a.dt_rep,
    a.client_group,

    a.bank_name_main,
    a.bank_type,
    a.gr_size,

    a.payment_type,
    a.transaction_type,
    a.operation_type,
    a.payment_direction,
    a.cli_biz,

    a.target_flag,
    a.sbp_self_1m,


    /* Фактические уведомления за РК */

    CASE
        WHEN a.sms_fact_ever = 1
         AND a.push_fact_ever = 1
            THEN N'SMS + Push'

        WHEN a.sms_fact_ever = 1
            THEN N'SMS'

        WHEN a.push_fact_ever = 1
            THEN N'Push'

        ELSE N'Нет'
    END AS fact_group,


    /* Потенциальное право за РК */

    CASE
        WHEN a.sms_eligible_ever = 1
         AND a.push_eligible_ever = 1
            THEN N'SMS + Push'

        WHEN a.sms_eligible_ever = 1
            THEN N'SMS'

        WHEN a.push_eligible_ever = 1
            THEN N'Push'

        ELSE N'Нет подтвержденного права'
    END AS eligible_group,


    /* Постоянные флаги */

    a.sms_fact_ever,
    a.push_fact_ever,

    a.sms_eligible_ever,
    a.push_eligible_ever,

    a.sms_unknown_ever,
    a.push_unknown_ever,


    /* Фактические уведомления к дате платежа */

    a.sms_fact_to_date,
    a.push_fact_to_date,


    /* Триггеры и приложение */

    a.trigger_day,

    a.has_app_activity,
    a.has_old_app,
    a.has_new_app,
    a.has_unknown_app,

    a.has_closed_deposit,


    /* Подневные права и факты */

    a.sms_eligible_day,
    a.push_eligible_day,

    a.sms_unknown_day,
    a.push_unknown_day,

    a.sms_fact_day,
    a.push_fact_day,


    /* Окна удержания */

    a.sms_window,
    a.push_window,

    a.sms_unknown_window,
    a.push_unknown_window,

    a.sms_offer_window,
    a.sms_cooldown_day,


    /* Денежные показатели */

    v.amount_scope,
    v.raspoznano_type,

    CASE
        WHEN v.amount_scope = N'Полные суммы'
            THEN a.source_rows
    END AS source_rows,

    CASE
        WHEN v.amount_scope = N'Полные суммы'
            THEN a.count_in
    END AS count_in,

    CASE
        WHEN v.amount_scope = N'Полные суммы'
            THEN a.count_out
    END AS count_out,

    CAST(
        v.incoming / 1000000000.0
        AS decimal(20,6)
    ) AS incoming,

    CAST(
        v.outgoing / 1000000000.0
        AS decimal(20,6)
    ) AS outgoing,

    CAST(
        (v.incoming + v.outgoing) / 1000000000.0
        AS decimal(20,6)
    ) AS saldo

FROM #rk26r_agg a

CROSS APPLY (

    VALUES

        (
            N'Компоненты',
            N'Вклады',
            a.incoming_sr,
            a.outgoing_sr
        ),

        (
            N'Компоненты',
            N'НС',
            a.incoming_ns,
            a.outgoing_ns
        ),

        (
            N'Компоненты',
            N'ДВС',
            a.incoming_dvs,
            a.outgoing_dvs
        ),

        (
            N'Компоненты',
            N'Другие распознанные',
            a.incoming_other,
            a.outgoing_other
        ),

        (
            N'Компоненты',
            N'Нераспознанные',
            a.incoming_unknown,
            a.outgoing_unknown
        ),

        (
            N'Полные суммы',
            N'Полные переводы',
            a.amount_in,
            a.amount_out
        )

) v(amount_scope, raspoznano_type, incoming, outgoing)

WHERE v.incoming <> 0
   OR v.outgoing <> 0
   OR (
       v.amount_scope = N'Полные суммы'
       AND a.source_rows > 0
   )

ORDER BY
    a.dt_rep,
    a.client_group,
    a.bank_name_main,
    a.gr_size,
    v.amount_scope,
    v.raspoznano_type

OPTION (MAXDOP 1, RECOMPILE);
