
USE ALM_TEST;
SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @DateBase date = '2026-09-30';
DECLARE @DateFrom date = '2026-10-01';
DECLARE @DateTo date = '2026-11-04';

DECLARE @DaysPerRun int = 4;
DECLARE @ClientsPerBatch int = 50000;
DECLARE @BatchesPerRun int = 4;
DECLARE @MaxRunSeconds int = 420;

DECLARE @RunStart datetime2 = SYSDATETIME();
DECLARE @dt date;
DECLARE @Stage int;
DECLARE @Done int = 0;
DECLARE @Msg nvarchar(500);


/* ИНИЦИАЛИЗАЦИЯ.
   При повторных запусках таблицы сохраняются. */

IF OBJECT_ID('tempdb..#rk_oct26_job') IS NULL
BEGIN

    DROP TABLE IF EXISTS #rk_oct26_ids;
    DROP TABLE IF EXISTS #rk_oct26_days;
    DROP TABLE IF EXISTS #rk_oct26_batch;
    DROP TABLE IF EXISTS #rk_oct26_history;
    DROP TABLE IF EXISTS #rk_oct26_registry;

    CREATE TABLE #rk_oct26_ids (
        cli_id bigint NOT NULL PRIMARY KEY
    );

    CREATE TABLE #rk_oct26_days (
        dt_rep date NOT NULL PRIMARY KEY
    );

    CREATE TABLE #rk_oct26_batch (
        cli_id bigint NOT NULL PRIMARY KEY
    );

    CREATE TABLE #rk_oct26_registry (
        cli_id bigint NOT NULL PRIMARY KEY,
        client_group nvarchar(150) NOT NULL
    );

    /* История договоров одной партии клиентов */

    SELECT TOP (0)
        CAST(h.CLI_ID AS bigint) AS cli_id,
        h.CON_ID,
        h.DT_OPEN,
        h.PROD_NAME,
        CAST(0 AS int) AS eligible_product
    INTO #rk_oct26_history
    FROM LIQUIDITY.liq.DepositContract_all h;

    CREATE INDEX IX_rk_oct26_history
    ON #rk_oct26_history (cli_id);

    /* 1 = клиенты баланса, 2 = категоризация, 3 = готово */

    CREATE TABLE #rk_oct26_job (
        stage int NOT NULL
    );

    INSERT INTO #rk_oct26_job VALUES (1);

END;

SELECT @Stage = stage
FROM #rk_oct26_job;


/* СОБИРАЕМ ВСЕХ КЛИЕНТОВ ИСХОДНОЙ СТАТИСТИКИ РОЗНИЦЫ.
   Стандартная линейка Розница + НС Розница.
   Фильтры как в первоначальном балансовом скрипте. */

IF @Stage = 1
BEGIN

    SELECT @dt = DATEADD(day, 1, MAX(dt_rep))
    FROM #rk_oct26_days;

    IF @dt IS NULL
        SET @dt = @DateBase;

    WHILE @dt <= @DateTo
      AND @Done < @DaysPerRun
    BEGIN

        IF DATEDIFF(second, @RunStart, SYSDATETIME())
           >= @MaxRunSeconds
            BREAK;

        SET @Msg = CONCAT(
            N'Клиенты баланса: ',
            CONVERT(varchar(10), @dt, 120)
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

        INSERT INTO #rk_oct26_ids (cli_id)

        SELECT DISTINCT
            CAST(t.CLI_ID AS bigint)

        FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

        WHERE t.dt_rep >= @dt
          AND t.dt_rep < DATEADD(day, 1, @dt)

          AND t.CLI_ID IS NOT NULL
          AND t.block_name = N'Привлечение ФЛ'
          AND t.od_flag = 1
          AND t.cur = '810'
          AND t.out_rub IS NOT NULL
          AND t.out_rub >= 0

          AND (
              t.TSEGMENTNAME <> N'ДЧБО'
              OR t.TSEGMENTNAME IS NULL
          )

          AND (
              (
                  t.section_name = N'Накопительный счёт'
                  AND t.acc_role = N'LIAB'
              )
              OR
              (
                  t.section_name = N'Срочные'
                  AND (
                      t.PROD_NAME_res IS NULL
                      OR t.PROD_NAME_res NOT IN (
                          N'ДОМа надёжно',
                          N'Всё в ДОМ',
                          N'Надёжный прайм',
                          N'НадёжныйVIP',
                          N'Надёжный премиум',
                          N'Надёжный промо',
                          N'Надёжный старт',
                          N'Надёжный Т2',
                          N'Надёжный Мегафон',
                          N'Надёжный процент',
                          N'Могучий',
                          N'Надёжный'
                      )
                  )
              )
          )

          AND NOT EXISTS (
              SELECT 1
              FROM #rk_oct26_ids c
              WHERE c.cli_id = t.CLI_ID
          )

        OPTION (MAXDOP 1, RECOMPILE);

        INSERT INTO #rk_oct26_days VALUES (@dt);

        SET @Done = @Done + 1;
        SET @dt = DATEADD(day, 1, @dt);

    END;

    IF (SELECT COUNT(*) FROM #rk_oct26_days)
       = DATEDIFF(day, @DateBase, @DateTo) + 1
    BEGIN

        UPDATE #rk_oct26_job
        SET stage = 2;

    END;

END


/* КАТЕГОРИЗАЦИЯ КЛИЕНТОВ.
   Историю договоров читаем партиями.
   Сначала Private, затем новые клиенты, затем новые деньги. */

ELSE IF @Stage = 2
BEGIN

    WHILE @Done < @BatchesPerRun
    BEGIN

        IF DATEDIFF(second, @RunStart, SYSDATETIME())
           >= @MaxRunSeconds
            BREAK;

        TRUNCATE TABLE #rk_oct26_batch;
        TRUNCATE TABLE #rk_oct26_history;

        INSERT INTO #rk_oct26_batch (cli_id)

        SELECT TOP (@ClientsPerBatch)
            c.cli_id

        FROM #rk_oct26_ids c

        WHERE NOT EXISTS (
            SELECT 1
            FROM #rk_oct26_registry r
            WHERE r.cli_id = c.cli_id
        )

        ORDER BY c.cli_id;

        IF @@ROWCOUNT = 0
            BREAK;


        /* История всех договоров клиента.
           Отдельно отмечаем продукты, подходящие
           под прежнюю методологию новых клиентов. */

        INSERT INTO #rk_oct26_history (
            cli_id,
            CON_ID,
            DT_OPEN,
            PROD_NAME,
            eligible_product
        )

        SELECT
            b.cli_id,
            h.CON_ID,
            h.DT_OPEN,
            h.PROD_NAME,

            CASE
                WHEN h.cli_short_name = N'ФЛ'
                 AND h.cur = 'RUR'
                 AND h.PROD_NAME NOT IN (
                     N'Агентские эскроу ФЛ по ставке КС+спред',
                     N'Спец. банк.счёт',
                     N'Залоговый',
                     N'Агентские эскроу ФЛ по ставке КС/2',
                     N'Эскроу',
                     N'Депозит ФЛ (суррогатный договор для счета Новой Афины)',
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
                     N'ДОМа надёжно'
                 )
                    THEN 1
                ELSE 0
            END

        FROM LIQUIDITY.liq.DepositContract_all h WITH (NOLOCK)

        INNER JOIN #rk_oct26_batch b
            ON b.cli_id = h.CLI_ID

        WHERE h.DT_OPEN < DATEADD(day, 1, @DateTo)

        OPTION (MAXDOP 1, RECOMPILE);


        /* Private по истории сегмента.
           Проверка как в прежнем скрипте MASS. */

        ;WITH private_clients AS (

            SELECT DISTINCT
                h.cli_id

            FROM #rk_oct26_history h

            WHERE h.CON_ID IS NOT NULL

              AND EXISTS (
                  SELECT 1

                  FROM LIQUIDITY.liq.DepositInterestsRate r
                       WITH (NOLOCK)

                  WHERE r.CON_ID = h.CON_ID
                    AND r.seg_name = N'ДЧБО'
              )

        ),

        /* Договоры, открытые в период РК */

        opened_ranked AS (

            SELECT
                h.cli_id,
                TRY_CAST(h.CON_ID AS bigint) AS con_id,
                h.PROD_NAME,

                ROW_NUMBER() OVER (
                    PARTITION BY h.CON_ID
                    ORDER BY h.DT_OPEN, h.PROD_NAME
                ) AS rn

            FROM #rk_oct26_history h

            WHERE h.eligible_product = 1
              AND h.DT_OPEN >= @DateFrom
              AND h.CON_ID IS NOT NULL

        ),

        opened AS (

            SELECT
                cli_id,
                con_id,
                PROD_NAME

            FROM opened_ranked

            WHERE rn = 1
              AND con_id IS NOT NULL

        ),

        /* Новые клиенты по истории договоров */

        new_clients AS (

            SELECT DISTINCT
                o.cli_id

            FROM opened o

            WHERE NOT EXISTS (
                SELECT 1

                FROM #rk_oct26_history h

                WHERE h.cli_id = o.cli_id
                  AND h.eligible_product = 1
                  AND h.DT_OPEN < @DateFrom
            )

            AND NOT EXISTS (
                SELECT 1
                FROM private_clients p
                WHERE p.cli_id = o.cli_id
            )

        ),

        /* Старые клиенты с надбавками НОВ / НДП / НДМ */

        new_money_clients AS (

            SELECT DISTINCT
                o.cli_id

            FROM opened o

            CROSS APPLY (
                SELECT TOP (1)
                    a.[Нов],
                    a.[НДП],
                    a.[НДМ]

                FROM ALM.ehd.attr_DepoFLConditions a
                     WITH (NOLOCK)

                WHERE a.CON_ID = CONVERT(varchar(30), o.con_id)

                ORDER BY
                    a.DT_UPDATE DESC,
                    a.loaddate DESC
            ) a

            WHERE o.PROD_NAME NOT LIKE N'Накопительный счёт%'

              AND NOT EXISTS (
                  SELECT 1
                  FROM new_clients n
                  WHERE n.cli_id = o.cli_id
              )

              AND NOT EXISTS (
                  SELECT 1
                  FROM private_clients p
                  WHERE p.cli_id = o.cli_id
              )

              AND (
                  ISNULL(TRY_CAST(a.[Нов] AS int), 0) = 1
                  OR ISNULL(TRY_CAST(a.[НДП] AS int), 0) = 1
                  OR ISNULL(TRY_CAST(a.[НДМ] AS int), 0) = 1
              )

        )

        /* Каждому клиенту присваиваем ровно одну категорию */

        INSERT INTO #rk_oct26_registry (
            cli_id,
            client_group
        )

        SELECT
            b.cli_id,

            CASE
                WHEN p.cli_id IS NOT NULL
                    THEN N'потенциальный клиент private кто попал в статистику масс по балансу'

                WHEN n.cli_id IS NOT NULL
                    THEN N'новый клиент масс'

                WHEN nm.cli_id IS NOT NULL
                    THEN N'старый клиент масс кто принес новые деньги'

                ELSE N'старый клиент масс кто не принес новые деньги'
            END

        FROM #rk_oct26_batch b

        LEFT JOIN private_clients p
            ON p.cli_id = b.cli_id

        LEFT JOIN new_clients n
            ON n.cli_id = b.cli_id

        LEFT JOIN new_money_clients nm
            ON nm.cli_id = b.cli_id

        OPTION (MAXDOP 1, RECOMPILE);


        SET @Done = @Done + 1;

        SET @Msg = CONCAT(
            N'Готово партий: ',
            @Done, N'/', @BatchesPerRun
        );

        RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

    END;


    /* ЗАГРУЖАЕМ ГОТОВЫЙ РЕЕСТР В ALM_TEST.WORK.
       Постоянная таблица обновляется только после
       завершения категоризации всех клиентов. */

    IF NOT EXISTS (
        SELECT 1
        FROM #rk_oct26_ids c
        WHERE NOT EXISTS (
            SELECT 1
            FROM #rk_oct26_registry r
            WHERE r.cli_id = c.cli_id
        )
    )
    BEGIN

        BEGIN TRY

            BEGIN TRANSACTION;

            IF OBJECT_ID(
                'WORK.RK_Oct2026_ClientRegistry', 'U'
            ) IS NULL
            BEGIN

                CREATE TABLE WORK.RK_Oct2026_ClientRegistry (
                    cli_id bigint NOT NULL PRIMARY KEY,
                    client_group nvarchar(150) NOT NULL
                );

            END;

            TRUNCATE TABLE WORK.RK_Oct2026_ClientRegistry;

            INSERT INTO WORK.RK_Oct2026_ClientRegistry (
                cli_id,
                client_group
            )
            SELECT
                cli_id,
                client_group
            FROM #rk_oct26_registry;

            UPDATE #rk_oct26_job
            SET stage = 3;

            COMMIT TRANSACTION;

        END TRY

        BEGIN CATCH

            IF XACT_STATE() <> 0
                ROLLBACK TRANSACTION;

            THROW;

        END CATCH;

    END;

END;


/* СТАТУС РАСЧЕТА */

SELECT @Stage = stage
FROM #rk_oct26_job;

SELECT
    CASE
        WHEN @Stage = 1 THEN N'Сбор клиентов баланса'
        WHEN @Stage = 2 THEN N'Категоризация клиентов'
        ELSE N'Готово'
    END AS stage,

    (SELECT COUNT(*) FROM #rk_oct26_days)
        AS days_done,

    DATEDIFF(day, @DateBase, @DateTo) + 1
        AS days_total,

    (SELECT COUNT_BIG(*) FROM #rk_oct26_ids)
        AS total_clients,

    (SELECT COUNT_BIG(*) FROM #rk_oct26_registry)
        AS classified_clients;


/* ИТОГ */

IF @Stage = 3
BEGIN

    SELECT
        client_group,
        COUNT_BIG(*) AS clients

    FROM WORK.RK_Oct2026_ClientRegistry

    GROUP BY client_group;

END;





USE ALM_TEST;
SET NOCOUNT ON;

DECLARE @DateBase date = '2026-09-30';
DECLARE @DateTo date = '2026-11-04';

DECLARE @DaysPerRun int = 4;
DECLARE @MaxRunSeconds int = 420;

DECLARE @RunStart datetime2 = SYSDATETIME();
DECLARE @dt date;
DECLARE @Done int = 0;
DECLARE @Msg nvarchar(500);

IF OBJECT_ID(
    'WORK.RK_Oct2026_ClientRegistry', 'U'
) IS NULL
    THROW 50001, 'Сначала сформируйте реестр клиентов', 1;


/* ИНИЦИАЛИЗАЦИЯ.
   Подневные остатки сохраняются между запусками. */

IF OBJECT_ID('tempdb..#rk_oct26_balances') IS NULL
BEGIN

    CREATE TABLE #rk_oct26_balances (
        dt_rep date NOT NULL PRIMARY KEY,
        deposits_rub decimal(38,2),
        ns_rub decimal(38,2),
        total_rub decimal(38,2)
    );

END;


SELECT @dt = DATEADD(day, 1, MAX(dt_rep))
FROM #rk_oct26_balances;

IF @dt IS NULL
    SET @dt = @DateBase;


/* БАЛАНСЫ ТОЛЬКО MASS-КЛИЕНТОВ.
   Фильтры исходного балансового скрипта сохраняются.
   Private по истории сегмента исключаем. */

WHILE @dt <= @DateTo
  AND @Done < @DaysPerRun
BEGIN

    IF DATEDIFF(second, @RunStart, SYSDATETIME())
       >= @MaxRunSeconds
        BREAK;

    SET @Msg = CONCAT(
        N'Баланс MASS: ',
        CONVERT(varchar(10), @dt, 120)
    );

    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;


    INSERT INTO #rk_oct26_balances (
        dt_rep,
        deposits_rub,
        ns_rub,
        total_rub
    )

    SELECT
        @dt,

        ISNULL(SUM(CASE
            WHEN t.section_name = N'Срочные'
                THEN CAST(t.out_rub AS decimal(20,2))
            ELSE 0
        END), 0),

        ISNULL(SUM(CASE
            WHEN t.section_name = N'Накопительный счёт'
                THEN CAST(t.out_rub AS decimal(20,2))
            ELSE 0
        END), 0),

        ISNULL(SUM(CAST(t.out_rub AS decimal(20,2))), 0)

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    INNER JOIN ALM_TEST.WORK.RK_Oct2026_ClientRegistry c
        ON c.cli_id = t.CLI_ID

    WHERE t.dt_rep >= @dt
      AND t.dt_rep < DATEADD(day, 1, @dt)

      AND c.client_group IN (
          N'новый клиент масс',
          N'старый клиент масс кто принес новые деньги',
          N'старый клиент масс кто не принес новые деньги'
      )

      AND t.block_name = N'Привлечение ФЛ'
      AND t.od_flag = 1
      AND t.cur = '810'
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0

      AND (
          t.TSEGMENTNAME <> N'ДЧБО'
          OR t.TSEGMENTNAME IS NULL
      )

      AND (
          (
              t.section_name = N'Накопительный счёт'
              AND t.acc_role = N'LIAB'
          )
          OR
          (
              t.section_name = N'Срочные'

              AND (
                  t.PROD_NAME_res IS NULL
                  OR t.PROD_NAME_res NOT IN (
                      N'ДОМа надёжно',
                      N'Всё в ДОМ',
                      N'Надёжный прайм',
                      N'НадёжныйVIP',
                      N'Надёжный премиум',
                      N'Надёжный промо',
                      N'Надёжный старт',
                      N'Надёжный Т2',
                      N'Надёжный Мегафон',
                      N'Надёжный процент',
                      N'Могучий',
                      N'Надёжный'
                  )
              )
          )
      )

    OPTION (MAXDOP 1, RECOMPILE);

    SET @Done = @Done + 1;
    SET @dt = DATEADD(day, 1, @dt);

END;


/* СТАТУС РАСЧЕТА */

SELECT
    COUNT(*) AS days_done,
    DATEDIFF(day, @DateBase, @DateTo) + 1 AS days_total,
    MAX(dt_rep) AS last_date
FROM #rk_oct26_balances;


/* ИТОГ. Остатки в млрд рублей */

IF (SELECT COUNT(*) FROM #rk_oct26_balances)
   = DATEDIFF(day, @DateBase, @DateTo) + 1
BEGIN

    SELECT
        dt_rep,

        CAST(deposits_rub / 1000000000.0
            AS decimal(20,6)) AS deposits_bln,

        CAST(ns_rub / 1000000000.0
            AS decimal(20,6)) AS ns_bln,

        CAST(total_rub / 1000000000.0
            AS decimal(20,6)) AS total_bln

    FROM #rk_oct26_balances

    ORDER BY dt_rep;

END;
