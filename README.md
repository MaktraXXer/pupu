
USE ALM_TEST;
SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @DateFrom date = '2026-10-01';
DECLARE @DateTo date = '2026-11-04';

DECLARE @ClientsPerBatch int = 10000;
DECLARE @BatchesPerRun int = 1;
DECLARE @MaxRunSeconds int = 420;

DECLARE @RunStart datetime2 = SYSDATETIME();
DECLARE @StepStart datetime2;
DECLARE @Done int = 0;
DECLARE @Rows int;
DECLARE @Msg nvarchar(500);

IF @@TRANCOUNT <> 0
    THROW 50001, 'Open transaction', 1;

IF OBJECT_ID('tempdb..#rk_oct26_ids') IS NULL
   OR OBJECT_ID('tempdb..#rk_oct26_registry') IS NULL
   OR OBJECT_ID('tempdb..#rk_oct26_history') IS NULL
   OR OBJECT_ID('tempdb..#rk_oct26_batch') IS NULL
   OR OBJECT_ID('tempdb..#rk_oct26_job') IS NULL
    THROW 50002, 'Не найдены временные таблицы', 1;

IF NOT EXISTS (
    SELECT 1
    FROM #rk_oct26_job
    WHERE stage = 2
)
    THROW 50003, 'Расчет не находится на этапе категоризации', 1;


/* КАТЕГОРИЗАЦИЯ ОСТАВШИХСЯ КЛИЕНТОВ.
   Уже готовые CLI_ID не пересчитываем. */

WHILE @Done < @BatchesPerRun
BEGIN

    IF DATEDIFF(second, @RunStart, SYSDATETIME())
       >= @MaxRunSeconds
        BREAK;

    DROP TABLE IF EXISTS #rk_fast_con;
    DROP TABLE IF EXISTS #rk_fast_private;
    DROP TABLE IF EXISTS #rk_fast_old;
    DROP TABLE IF EXISTS #rk_fast_opened;
    DROP TABLE IF EXISTS #rk_fast_new;
    DROP TABLE IF EXISTS #rk_fast_nm_candidates;
    DROP TABLE IF EXISTS #rk_fast_nm;

    TRUNCATE TABLE #rk_oct26_batch;
    TRUNCATE TABLE #rk_oct26_history;


    /* Следующая партия клиентов */

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

    SET @Rows = @@ROWCOUNT;

    IF @Rows = 0
        BREAK;


    /* История договоров */

    SET @StepStart = SYSDATETIME();

    SET @Msg = CONCAT(N'История договоров. Клиентов: ', @Rows);
    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

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

    SET @Msg = CONCAT(
        N'История готова | секунд: ',
        DATEDIFF(second, @StepStart, SYSDATETIME())
    );

    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;


    /* Private по истории сегмента.
       Проверяем каждый уникальный договор один раз. */

    SET @StepStart = SYSDATETIME();

    RAISERROR(N'Проверка Private', 0, 1) WITH NOWAIT;

    SELECT DISTINCT
        cli_id,
        CON_ID

    INTO #rk_fast_con

    FROM #rk_oct26_history

    WHERE CON_ID IS NOT NULL;

    CREATE UNIQUE CLUSTERED INDEX IX_rk_fast_con
    ON #rk_fast_con (CON_ID, cli_id);


    SELECT DISTINCT
        h.cli_id

    INTO #rk_fast_private

    FROM #rk_fast_con h

    WHERE EXISTS (
        SELECT 1
        FROM LIQUIDITY.liq.DepositInterestsRate r WITH (NOLOCK)
        WHERE r.CON_ID = h.CON_ID
          AND r.seg_name = N'ДЧБО'
    )

    OPTION (MAXDOP 1, RECOMPILE);

    CREATE UNIQUE CLUSTERED INDEX IX_rk_fast_private
    ON #rk_fast_private (cli_id);

    SET @Msg = CONCAT(
        N'Private готов | секунд: ',
        DATEDIFF(second, @StepStart, SYSDATETIME())
    );

    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;


    /* Клиенты с историей договоров до РК */

    SELECT DISTINCT
        cli_id

    INTO #rk_fast_old

    FROM #rk_oct26_history

    WHERE eligible_product = 1
      AND DT_OPEN < @DateFrom;

    CREATE UNIQUE CLUSTERED INDEX IX_rk_fast_old
    ON #rk_fast_old (cli_id);


    /* Договоры, открытые в период РК */

    ;WITH ranked AS (
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
    )

    SELECT
        cli_id,
        con_id,
        PROD_NAME

    INTO #rk_fast_opened

    FROM ranked

    WHERE rn = 1
      AND con_id IS NOT NULL;

    CREATE INDEX IX_rk_fast_opened
    ON #rk_fast_opened (cli_id);


    /* Новые клиенты масс */

    SELECT DISTINCT
        o.cli_id

    INTO #rk_fast_new

    FROM #rk_fast_opened o

    LEFT JOIN #rk_fast_old h
        ON h.cli_id = o.cli_id

    LEFT JOIN #rk_fast_private p
        ON p.cli_id = o.cli_id

    WHERE h.cli_id IS NULL
      AND p.cli_id IS NULL;

    CREATE UNIQUE CLUSTERED INDEX IX_rk_fast_new
    ON #rk_fast_new (cli_id);


    /* Старые MASS-клиенты с открытыми вкладами.
       Новых клиентов и Private сразу исключаем. */

    SELECT DISTINCT
        o.cli_id,
        CONVERT(varchar(30), o.con_id) AS con_id

    INTO #rk_fast_nm_candidates

    FROM #rk_fast_opened o

    LEFT JOIN #rk_fast_new n
        ON n.cli_id = o.cli_id

    LEFT JOIN #rk_fast_private p
        ON p.cli_id = o.cli_id

    WHERE o.PROD_NAME NOT LIKE N'Накопительный счёт%'
      AND n.cli_id IS NULL
      AND p.cli_id IS NULL;

    CREATE UNIQUE CLUSTERED INDEX IX_rk_fast_nm_candidates
    ON #rk_fast_nm_candidates (con_id, cli_id);


    /* НОВ / НДП / НДМ */

    SET @StepStart = SYSDATETIME();

    RAISERROR(N'Проверка новых денег', 0, 1) WITH NOWAIT;

    SELECT DISTINCT
        c.cli_id

    INTO #rk_fast_nm

    FROM #rk_fast_nm_candidates c

    CROSS APPLY (
        SELECT TOP (1)
            a.[Нов],
            a.[НДП],
            a.[НДМ]

        FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)

        WHERE a.CON_ID = c.con_id

        ORDER BY
            a.DT_UPDATE DESC,
            a.loaddate DESC
    ) a

    WHERE ISNULL(TRY_CAST(a.[Нов] AS int), 0) = 1
       OR ISNULL(TRY_CAST(a.[НДП] AS int), 0) = 1
       OR ISNULL(TRY_CAST(a.[НДМ] AS int), 0) = 1

    OPTION (MAXDOP 1, RECOMPILE);

    CREATE UNIQUE CLUSTERED INDEX IX_rk_fast_nm
    ON #rk_fast_nm (cli_id);

    SET @Msg = CONCAT(
        N'Новые деньги готовы | секунд: ',
        DATEDIFF(second, @StepStart, SYSDATETIME())
    );

    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;


    /* Сохраняем готовую партию клиентов */

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

    LEFT JOIN #rk_fast_private p
        ON p.cli_id = b.cli_id

    LEFT JOIN #rk_fast_new n
        ON n.cli_id = b.cli_id

    LEFT JOIN #rk_fast_nm nm
        ON nm.cli_id = b.cli_id

    OPTION (MAXDOP 1, RECOMPILE);


    SET @Done = @Done + 1;

    SET @Msg = CONCAT(
        N'Готово партий: ',
        @Done, N'/', @BatchesPerRun
    );

    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;

END;


/* ЗАГРУЖАЕМ ГОТОВЫЙ РЕЕСТР В ALM_TEST.WORK */

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


/* СТАТУС РАСЧЕТА */

SELECT
    (SELECT stage FROM #rk_oct26_job) AS stage,

    (SELECT COUNT_BIG(*) FROM #rk_oct26_ids)
        AS total_clients,

    (SELECT COUNT_BIG(*) FROM #rk_oct26_registry)
        AS classified_clients,

    (SELECT COUNT_BIG(*) FROM #rk_oct26_ids)
    - (SELECT COUNT_BIG(*) FROM #rk_oct26_registry)
        AS remaining_clients,

    DATEDIFF(second, @RunStart, SYSDATETIME())
        AS run_seconds;


/* ИТОГ */

SELECT
    client_group,
    COUNT_BIG(*) AS clients

FROM #rk_oct26_registry

GROUP BY client_group;
