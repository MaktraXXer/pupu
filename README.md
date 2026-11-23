
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
DECLARE @dt date;
DECLARE @d date;
DECLARE @Stage int;
DECLARE @Done int;
DECLARE @Msg nvarchar(500);

IF @@TRANCOUNT <> 0
    THROW 50001, 'Open transaction', 1;


/* ======================================================
   1. ИНИЦИАЛИЗАЦИЯ.
   При повторных запусках таблицы сохраняются.
   ====================================================== */

IF OBJECT_ID('tempdb..#r2_job') IS NULL
BEGIN

    DROP TABLE IF EXISTS #r2_existing;
    DROP TABLE IF EXISTS #r2_triggers;
    DROP TABLE IF EXISTS #r2_scan_clients;
    DROP TABLE IF EXISTS #r2_trigger_done;
    DROP TABLE IF EXISTS #r2_closed;
    DROP TABLE IF EXISTS #r2_messages;
    DROP TABLE IF EXISTS #r2_sms_windows;
    DROP TABLE IF EXISTS #r2_push_windows;
    DROP TABLE IF EXISTS #r2_clients;
    DROP TABLE IF EXISTS #r2_pay_day;
    DROP TABLE IF EXISTS #r2_pay_sums;
    DROP TABLE IF EXISTS #r2_pay_done;

    /* Действующие клиенты по состоянию на 31.10 */

    SELECT t.CLI_ID AS cli_id
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
      AND t.out_rub >= 0
    GROUP BY t.CLI_ID
    HAVING SUM(t.out_rub) > 0
    OPTION (MAXDOP 1, RECOMPILE);

    CREATE UNIQUE CLUSTERED INDEX IX_r2_existing
    ON #r2_existing (cli_id);

    IF NOT EXISTS (SELECT 1 FROM #r2_existing)
        THROW 50002, 'No balances on 31.10', 1;

    /* Сохраняем только даты и CLI_ID триггеров */

    SELECT TOP (0)
        t.cli_id,
        CAST(t.dt_rep AS date) AS dt
    INTO #r2_triggers
    FROM ALM.ehd.VW_transfers_FL_det t;

    CREATE UNIQUE CLUSTERED INDEX IX_r2_triggers
    ON #r2_triggers (cli_id, dt);

    SELECT TOP (0) cli_id
    INTO #r2_scan_clients
    FROM #r2_triggers;

    CREATE UNIQUE CLUSTERED INDEX IX_r2_scan_clients
    ON #r2_scan_clients (cli_id);

    CREATE TABLE #r2_trigger_done (
        dt date NOT NULL PRIMARY KEY
    );

    /* Агрегаты всех платежей выбранной когорты */

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

        incoming_ns decimal(38,2),
        outgoing_ns decimal(38,2),
        incoming_sr decimal(38,2),
        outgoing_sr decimal(38,2),
        incoming_dvs decimal(38,2),
        outgoing_dvs decimal(38,2)
    );

    SELECT TOP (0) *
    INTO #r2_pay_day
    FROM #r2_pay_sums;

    CREATE TABLE #r2_pay_done (
        dt date NOT NULL PRIMARY KEY
    );

    /* 1 = триггеры, 2 = когорта, 3 = платежи, 4 = готово */

    CREATE TABLE #r2_job (
        date_base date NOT NULL,
        date_from date NOT NULL,
        date_to date NOT NULL,
        stage int NOT NULL
    );

    INSERT INTO #r2_job
    VALUES (@DateBase, @DateFrom, @DateTo, 1);

END;

IF EXISTS (
    SELECT 1 FROM #r2_job
    WHERE date_base <> @DateBase
       OR date_from <> @DateFrom
       OR date_to <> @DateTo
)
    THROW 50003, 'Date parameters changed', 1;

SELECT @Stage = stage FROM #r2_job;


/* ======================================================
   ЭТАП 1. ИЩЕМ ТОЛЬКО СБП-ТРИГГЕРЫ ПО 4 ДНЯ

   Не сохраняем все платежи, только CLI_ID и дату.
   ====================================================== */

IF @Stage = 1
BEGIN

    SELECT @dt = DATEADD(day, 1, MAX(dt))
    FROM #r2_trigger_done;

    IF @dt IS NULL SET @dt = @HistoryFrom;

    SET @Done = 0;

    WHILE @dt <= @DateTo AND @Done < @DaysPerRun
    BEGIN
        IF DATEDIFF(second, @RunStart, SYSDATETIME())
           >= @MaxRunSeconds BREAK;

        SET @Msg = CONCAT(
            N'Триггеры: ',
            CONVERT(varchar(10), @dt, 120)
        );
        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        TRUNCATE TABLE #r2_scan_clients;

        INSERT INTO #r2_scan_clients
        SELECT DISTINCT t.cli_id
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

        BEGIN TRY
            BEGIN TRANSACTION;

            INSERT INTO #r2_triggers
            SELECT cli_id, @dt FROM #r2_scan_clients;

            INSERT INTO #r2_trigger_done VALUES (@dt);

            COMMIT TRANSACTION;
        END TRY
        BEGIN CATCH
            IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
            THROW;
        END CATCH;

        SET @Done = @Done + 1;
        SET @dt = DATEADD(day, 1, @dt);
    END;

    IF (SELECT COUNT(*) FROM #r2_trigger_done)
       = DATEDIFF(day, @HistoryFrom, @DateTo) + 1
        UPDATE #r2_job SET stage = 2;

END


/* ======================================================
   ЭТАП 2. ЗАКРЫТИЯ, СМС, ПУШИ И КЛИЕНТСКИЕ ФЛАГИ

   Отдельный запуск. Платежную детализацию не читаем.
   ====================================================== */

ELSE IF @Stage = 2
BEGIN

    DROP TABLE IF EXISTS #r2_closed;
    DROP TABLE IF EXISTS #r2_messages;
    DROP TABLE IF EXISTS #r2_sms_windows;
    DROP TABLE IF EXISTS #r2_push_windows;
    DROP TABLE IF EXISTS #r2_clients;

    RAISERROR(
        N'Формирую закрытия вкладов и реестр уведомлений',
        0, 1
    ) WITH NOWAIT;

    SELECT
        a.cli_id,
        CAST(a.DT_CLOSE AS date) AS dt_close
    INTO #r2_closed
    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
    INNER JOIN (
        SELECT DISTINCT cli_id FROM #r2_triggers
    ) c ON c.cli_id = a.cli_id
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
    GROUP BY a.cli_id, CAST(a.DT_CLOSE AS date)
    OPTION (MAXDOP 1, RECOMPILE);

    CREATE UNIQUE CLUSTERED INDEX IX_r2_closed
    ON #r2_closed (cli_id, dt_close);

    /* Реестр фактических уведомлений */

    SELECT
        m.cli_id,
        v.dt,
        MAX(v.sms_sent) AS sms_sent,
        MAX(v.push_created) AS push_created,
        MAX(v.push_viewed) AS push_viewed,
        MAX(v.push_accepted) AS push_accepted
    INTO #r2_messages
    FROM ALM.ehd.VW_promocode_FL_list m WITH (NOLOCK)
    CROSS APPLY (
        VALUES
          (
            CAST(m.msgbegindate AS date),
            CASE WHEN LOWER(m.message_type) = N'sms' THEN 1 ELSE 0 END,
            CASE WHEN LOWER(m.message_type) = N'push' THEN 1 ELSE 0 END,
            0, 0
          ),
          (
            CAST(m.dt_view AS date),
            0, 0,
            CASE WHEN LOWER(m.message_type) = N'push' THEN 1 ELSE 0 END,
            0
          ),
          (
            CAST(m.dt_accept AS date),
            0, 0, 0,
            CASE WHEN LOWER(m.message_type) = N'push' THEN 1 ELSE 0 END
          )
    ) v(dt, sms_sent, push_created, push_viewed, push_accepted)
    WHERE m.cli_id IS NOT NULL
      AND m.msgbegindate >= @HistoryFrom
      AND m.msgbegindate < DATEADD(day, 1, @DateTo)
      AND v.dt BETWEEN @HistoryFrom AND @DateTo
    GROUP BY m.cli_id, v.dt
    OPTION (MAXDOP 1, RECOMPILE);

    CREATE UNIQUE CLUSTERED INDEX IX_r2_messages
    ON #r2_messages (cli_id, dt);

    /* СМС: окно 7 дней с первого подходящего перевода */

    SELECT TOP (0)
        cli_id,
        dt AS dt_from,
        dt AS dt_to
    INTO #r2_sms_windows
    FROM #r2_triggers;

    CREATE UNIQUE CLUSTERED INDEX IX_r2_sms_windows
    ON #r2_sms_windows (cli_id, dt_from);

    SET @d = @HistoryFrom;

    WHILE @d <= @DateTo
    BEGIN
        INSERT INTO #r2_sms_windows
        SELECT t.cli_id, @d, DATEADD(day, 6, @d)
        FROM #r2_triggers t
        WHERE t.dt = @d
          AND NOT EXISTS (
              SELECT 1 FROM #r2_sms_windows w
              WHERE w.cli_id = t.cli_id
                AND @d BETWEEN w.dt_from AND w.dt_to
          )
          AND NOT EXISTS (
              SELECT 1 FROM #r2_messages m
              WHERE m.cli_id = t.cli_id
                AND m.sms_sent = 1
                AND m.dt BETWEEN DATEADD(day,-6,@d)
                             AND DATEADD(day,-1,@d)
          );

        SET @d = DATEADD(day, 1, @d);
    END;

    /* Пуш: подходящий перевод после закрытия вклада */

    SELECT
        t.cli_id,
        t.dt AS dt_from,
        MAX(DATEADD(day, 2, c.dt_close)) AS dt_to
    INTO #r2_push_windows
    FROM #r2_triggers t
    INNER JOIN #r2_closed c
        ON c.cli_id = t.cli_id
       AND t.dt BETWEEN c.dt_close
                    AND DATEADD(day, 2, c.dt_close)
    GROUP BY t.cli_id, t.dt;

    CREATE UNIQUE CLUSTERED INDEX IX_r2_push_windows
    ON #r2_push_windows (cli_id, dt_from);

    /* Уникальная клиентская когорта и постоянные флаги */

    SELECT
        ids.cli_id,

        CASE WHEN e.cli_id IS NOT NULL
            THEN N'Действующий' ELSE N'Новый'
        END AS client_type,

        CAST(CASE WHEN EXISTS (
            SELECT 1 FROM #r2_sms_windows w
            WHERE w.cli_id = ids.cli_id
              AND w.dt_to >= @DateFrom
        ) THEN 1 ELSE 0 END AS int) AS sms_eligible_ever,

        CAST(CASE WHEN EXISTS (
            SELECT 1 FROM #r2_push_windows w
            WHERE w.cli_id = ids.cli_id
              AND w.dt_to >= @DateFrom
        ) THEN 1 ELSE 0 END AS int) AS push_eligible_ever,

        CAST(CASE WHEN EXISTS (
            SELECT 1 FROM #r2_messages m
            WHERE m.cli_id = ids.cli_id
              AND m.sms_sent = 1
              AND m.dt >= DATEADD(day,-6,@DateFrom)
        ) THEN 1 ELSE 0 END AS int) AS sms_fact_ever,

        CAST(CASE WHEN EXISTS (
            SELECT 1 FROM #r2_messages m
            WHERE m.cli_id = ids.cli_id
              AND m.push_created = 1
              AND m.dt >= DATEADD(day,-2,@DateFrom)
        ) THEN 1 ELSE 0 END AS int) AS push_fact_ever,

        CAST(CASE WHEN EXISTS (
            SELECT 1 FROM #r2_messages m
            WHERE m.cli_id = ids.cli_id
              AND m.push_viewed = 1
        ) THEN 1 ELSE 0 END AS int) AS push_viewed_ever

    INTO #r2_clients

    FROM (
        SELECT cli_id FROM #r2_sms_windows
        WHERE dt_to >= @DateFrom

        UNION
        SELECT cli_id FROM #r2_push_windows
        WHERE dt_to >= @DateFrom

        UNION
        SELECT cli_id FROM #r2_messages
        WHERE (sms_sent = 1 AND dt >= DATEADD(day,-6,@DateFrom))
           OR (push_created = 1 AND dt >= DATEADD(day,-2,@DateFrom))
           OR (push_viewed = 1 AND dt >= @DateFrom)
    ) ids

    LEFT JOIN #r2_existing e
        ON e.cli_id = ids.cli_id;

    CREATE UNIQUE CLUSTERED INDEX IX_r2_clients
    ON #r2_clients (cli_id);

    IF NOT EXISTS (SELECT 1 FROM #r2_clients)
        THROW 50004, 'No clients in retention cohort', 1;

    UPDATE #r2_job SET stage = 3;

END


/* ======================================================
   ЭТАП 3. ВСЕ ПЛАТЕЖИ КОГОРТЫ, ПО 4 ДНЯ ЗА ЗАПУСК

   Читаем по одному дню.
   Сохраняем только агрегаты, не сырые платежи.
   ====================================================== */

ELSE IF @Stage = 3
BEGIN

    SELECT @dt = DATEADD(day, 1, MAX(dt))
    FROM #r2_pay_done;

    IF @dt IS NULL SET @dt = @DateFrom;

    SET @Done = 0;

    WHILE @dt <= @DateTo AND @Done < @DaysPerRun
    BEGIN

        IF DATEDIFF(second, @RunStart, SYSDATETIME())
           >= @MaxRunSeconds BREAK;

        SET @Msg = CONCAT(
            N'Платежи когорты: ',
            CONVERT(varchar(10), @dt, 120)
        );
        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        TRUNCATE TABLE #r2_pay_day;

        ;WITH base AS (
            SELECT
                CAST(t.dt_rep AS date) AS dt_rep,
                c.client_type,

                CASE
                    WHEN t.spec_cat IN ('BR','SR','FU') THEN N'Market'
                    WHEN t.transaction_type = N'Наличные' THEN N'Наличные'
                    ELSE t.bank_name_main
                END AS bank,

                CASE
                    WHEN t.spec_cat IN ('BR','SR','FU') THEN 'market'
                    WHEN t.transaction_type = N'Наличные' THEN 'cash'
                    ELSE 'bank'
                END AS bank_type,

                ISNULL(t.gr_size, N'Без gr_size') AS gr_size,

                c.sms_eligible_ever,
                c.push_eligible_ever,
                c.sms_fact_ever,
                c.push_fact_ever,
                c.push_viewed_ever,

                CASE
                    WHEN t.direction_type = N'Переводы себе'
                     AND t.transaction_type = N'СБП-Перевод'
                     AND t.direction = '-1'
                     AND t.amount_out <= -1000000
                     AND t.cli_biz = N'Розница'
                        THEN 1 ELSE 0
                END AS trigger_payment,

                CASE WHEN EXISTS (
                    SELECT 1 FROM #r2_sms_windows w
                    WHERE w.cli_id = t.cli_id
                      AND @dt BETWEEN w.dt_from AND w.dt_to
                ) THEN 1 ELSE 0 END AS sms_window,

                CASE WHEN EXISTS (
                    SELECT 1 FROM #r2_push_windows w
                    WHERE w.cli_id = t.cli_id
                      AND @dt BETWEEN w.dt_from AND w.dt_to
                ) THEN 1 ELSE 0 END AS push_window,

                CASE WHEN EXISTS (
                    SELECT 1 FROM #r2_messages m
                    WHERE m.cli_id = t.cli_id
                      AND m.sms_sent = 1
                      AND @dt BETWEEN m.dt
                                  AND DATEADD(day, 6, m.dt)
                ) THEN 1 ELSE 0 END AS sms_offer_window,

                CASE WHEN EXISTS (
                    SELECT 1 FROM #r2_messages m
                    WHERE m.cli_id = t.cli_id
                      AND m.push_viewed = 1
                      AND m.dt = @dt
                ) THEN 1 ELSE 0 END AS push_view_day,

                a.amt_ns,
                a.amt_sr,
                a.amt_dvs,

                CASE WHEN t.direction_txt = N'входящие'
                       OR (t.direction_txt = N'смешанные' AND a.amt > 0)
                     THEN 1 ELSE 0 END AS is_incoming,

                CASE WHEN t.direction_txt = N'исходящие'
                       OR (t.direction_txt = N'смешанные' AND a.amt < 0)
                     THEN 1 ELSE 0 END AS is_outgoing

            FROM ALM.ehd.VW_transfers_FL_det t WITH (NOLOCK)

            INNER JOIN #r2_clients c
                ON c.cli_id = t.cli_id

            CROSS APPLY (
                SELECT
                    CAST(ISNULL(t.[распознано_НС],0) AS decimal(28,2)) AS amt_ns,
                    CAST(ISNULL(t.[распознано_СР],0) AS decimal(28,2)) AS amt_sr,
                    CAST(ISNULL(t.[Распознано_ДВС],0) AS decimal(28,2)) AS amt_dvs
            ) v

            CROSS APPLY (
                SELECT v.amt_ns, v.amt_sr, v.amt_dvs,
                       v.amt_ns + v.amt_sr + v.amt_dvs AS amt
            ) a

            WHERE t.dt_rep >= @dt
              AND t.dt_rep < DATEADD(day,1,@dt)
              AND t.direction_type <> N'Ошибка'
              AND t.[целевая категория] = 1
              AND t.transaction_type <> N'Внутренние переводы'
              AND t.transit_max_id IS NULL
              AND t.bank_name_main <> N'Без перевода'
              AND t.cli_biz IN (N'Розница',N'ДЧБО')
              AND (
                  t.spec_cat NOT IN ('BR','SR','FU')
                  OR (
                      t.spec_cat IN ('BR','SR','FU')
                      AND t.transaction_type <> N'Наличные'
                  )
              )
        )

        INSERT INTO #r2_pay_day
        SELECT
            dt_rep, client_type, bank, bank_type, gr_size,
            sms_eligible_ever, push_eligible_ever,
            sms_fact_ever, push_fact_ever, push_viewed_ever,
            trigger_payment, sms_window, push_window,
            sms_offer_window, push_view_day,

            SUM(CASE WHEN is_incoming = 1 THEN amt_ns ELSE 0 END),
            SUM(CASE WHEN is_outgoing = 1 THEN amt_ns ELSE 0 END),
            SUM(CASE WHEN is_incoming = 1 THEN amt_sr ELSE 0 END),
            SUM(CASE WHEN is_outgoing = 1 THEN amt_sr ELSE 0 END),
            SUM(CASE WHEN is_incoming = 1 THEN amt_dvs ELSE 0 END),
            SUM(CASE WHEN is_outgoing = 1 THEN amt_dvs ELSE 0 END)

        FROM base
        WHERE is_incoming = 1 OR is_outgoing = 1
        GROUP BY
            dt_rep, client_type, bank, bank_type, gr_size,
            sms_eligible_ever, push_eligible_ever,
            sms_fact_ever, push_fact_ever, push_viewed_ever,
            trigger_payment, sms_window, push_window,
            sms_offer_window, push_view_day
        OPTION (MAXDOP 1, RECOMPILE);

        /* Атомарно фиксируем результат готового дня */

        BEGIN TRY
            BEGIN TRANSACTION;

            INSERT INTO #r2_pay_sums
            SELECT * FROM #r2_pay_day;

            INSERT INTO #r2_pay_done VALUES (@dt);

            COMMIT TRANSACTION;
        END TRY
        BEGIN CATCH
            IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
            THROW;
        END CATCH;

        SET @Done = @Done + 1;
        SET @dt = DATEADD(day, 1, @dt);

    END;

    IF (SELECT COUNT(*) FROM #r2_pay_done)
       = DATEDIFF(day,@DateFrom,@DateTo) + 1
        UPDATE #r2_job SET stage = 4;

END;


/* ======================================================
   СТАТУС ВЫПОЛНЕНИЯ
   ====================================================== */

SELECT @Stage = stage FROM #r2_job;

SELECT
    @@SPID AS session_id,
    @Stage AS stage,

    (SELECT COUNT(*) FROM #r2_trigger_done)
        AS trigger_days_done,

    (SELECT COUNT(*) FROM #r2_pay_done)
        AS payment_days_done,

    (SELECT COUNT_BIG(*) FROM #r2_triggers)
        AS trigger_client_days,

    CASE WHEN @Stage >= 3
        THEN (SELECT COUNT_BIG(*) FROM #r2_clients)
        ELSE NULL
    END AS cohort_clients,

    DATEDIFF(second,@RunStart,SYSDATETIME())
        AS run_seconds;


/* ======================================================
   ИТОГОВАЯ ВЫГРУЗКА
   ====================================================== */

IF @Stage = 4
BEGIN

    ;WITH bank_totals AS (
        SELECT
            bank, bank_type,
            SUM(incoming_ns + incoming_sr + incoming_dvs) AS bank_incoming,
            SUM(outgoing_ns + outgoing_sr + outgoing_dvs) AS bank_outgoing
        FROM #r2_pay_sums
        GROUP BY bank, bank_type
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
            WHEN sms_eligible_ever = 1 AND push_eligible_ever = 1
                THEN N'SMS + Push'
            WHEN sms_eligible_ever = 1 THEN N'SMS'
            WHEN push_eligible_ever = 1 THEN N'Push'
            ELSE N'Нет'
        END AS eligible_group,

        CASE
            WHEN sms_fact_ever = 1 AND push_fact_ever = 1
                THEN N'SMS + Push'
            WHEN sms_fact_ever = 1 THEN N'SMS'
            WHEN push_fact_ever = 1 THEN N'Push'
            ELSE N'Нет'
        END AS fact_group,

        p.sms_window,
        p.push_window,
        p.sms_offer_window,
        p.push_view_day,
        p.trigger_payment,

        SUM(v.incoming) / 1000000000.0 AS incoming,
        SUM(v.outgoing) / 1000000000.0 AS outgoing,
        SUM(v.incoming + v.outgoing) / 1000000000.0 AS saldo

    FROM classified p

    CROSS APPLY (
        VALUES
            (N'НС', p.incoming_ns, p.outgoing_ns),
            (N'Вклады', p.incoming_sr, p.outgoing_sr),
            (N'ДВС', p.incoming_dvs, p.outgoing_dvs)
    ) v(raspoznano_type, incoming, outgoing)

    GROUP BY
        p.dt_rep, p.client_type, p.bank_name_main,
        p.gr_size, v.raspoznano_type,
        sms_eligible_ever, push_eligible_ever,
        sms_fact_ever, push_fact_ever,
        p.sms_window, p.push_window,
        p.sms_offer_window, p.push_view_day,
        p.trigger_payment

    ORDER BY
        p.dt_rep, p.client_type,
        p.bank_name_main, p.gr_size, v.raspoznano_type;


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


/* ======================================================
   ДОПОЛНИТЕЛЬНЫЙ ЗАПРОС НА БАЛАНСЫ

   После завершения этапа 2 доступна #r2_clients.

   Использовать в той же SQL-сессии.

   Смотрим остатки РОВНО выбранных клиентов,
   сохраняя их постоянные флаги.
   ======================================================

DECLARE @BalanceDate date = '2026-11-10';

SELECT
    t.dt_rep,
    c.client_type,
    c.sms_eligible_ever,
    c.push_eligible_ever,
    c.sms_fact_ever,
    c.push_fact_ever,

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
    c.push_fact_ever

OPTION (MAXDOP 1, RECOMPILE);

====================================================== */
