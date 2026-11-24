
USE ALM_TEST;
SET NOCOUNT ON;
SET XACT_ABORT ON;

DECLARE @DateFrom date = '2026-10-01';
DECLARE @DateTo date = '2026-11-04';

DECLARE @DaysPerRun int = 4;
DECLARE @MaxRunSeconds int = 420;
DECLARE @IncludePrivate bit = 0;

DECLARE @RunStart datetime2 = SYSDATETIME();
DECLARE @DayStart datetime2;
DECLARE @dt date;
DECLARE @Done int = 0;
DECLARE @Msg nvarchar(500);

IF OBJECT_ID('WORK.RK_Oct2026_ClientRegistry', 'U') IS NULL
    THROW 50001, 'Не найден реестр клиентов', 1;


/* ИНИЦИАЛИЗАЦИЯ.
   При повторных запусках таблица сохраняется. */

IF OBJECT_ID('tempdb..#rk_oct26_group_balances') IS NULL
BEGIN

    CREATE TABLE #rk_oct26_group_balances (
        dt_rep date NOT NULL,
        client_group nvarchar(150) NOT NULL,

        deposits_rub decimal(38,2),
        ns_rub decimal(38,2),
        total_rub decimal(38,2),

        PRIMARY KEY (dt_rep, client_group)
    );

END;


/* Определяем следующую необработанную дату */

SELECT @dt = DATEADD(day, 1, MAX(dt_rep))
FROM #rk_oct26_group_balances;

IF @dt IS NULL
    SET @dt = @DateFrom;


/* ПОДНЕВНЫЕ БАЛАНСЫ ПО КАТЕГОРИЯМ КЛИЕНТОВ.
   Используем готовый реестр.
   Маркеты и ФУ исключаем по продуктам.
   Фильтры первоначального балансового скрипта сохраняются. */

WHILE @dt <= @DateTo
  AND @Done < @DaysPerRun
BEGIN

    IF DATEDIFF(second, @RunStart, SYSDATETIME())
       >= @MaxRunSeconds
        BREAK;

    SET @DayStart = SYSDATETIME();

    SET @Msg = CONCAT(
        N'Баланс по категориям: ',
        CONVERT(varchar(10), @dt, 120)
    );

    RAISERROR(N'%s', 0, 1, @Msg) WITH NOWAIT;


    /* Один дневной срез баланса */

    ;WITH balances AS (

        SELECT
            c.client_group,

            SUM(CASE
                WHEN t.section_name = N'Срочные'
                    THEN CAST(t.out_rub AS decimal(20,2))
                ELSE 0
            END) AS deposits_rub,

            SUM(CASE
                WHEN t.section_name = N'Накопительный счёт'
                    THEN CAST(t.out_rub AS decimal(20,2))
                ELSE 0
            END) AS ns_rub,

            SUM(CAST(t.out_rub AS decimal(20,2)))
                AS total_rub

        FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

        INNER JOIN ALM_TEST.WORK.RK_Oct2026_ClientRegistry c
            ON c.cli_id = t.CLI_ID

        WHERE t.dt_rep >= @dt
          AND t.dt_rep < DATEADD(day, 1, @dt)

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

              /* НС Розница */

              (
                  t.section_name = N'Накопительный счёт'
                  AND t.acc_role = N'LIAB'
              )

              OR

              /* Стандартная линейка вкладов Розница */

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

        GROUP BY
            c.client_group
    ),

    /* Сохраняем все категории на каждую дату */

    client_groups AS (

        SELECT client_group

        FROM (VALUES
            (N'новый клиент масс'),
            (N'старый клиент масс кто принес новые деньги'),
            (N'старый клиент масс кто не принес новые деньги'),
            (N'потенциальный клиент private кто попал в статистику масс по балансу')
        ) v(client_group)

    )

    INSERT INTO #rk_oct26_group_balances (
        dt_rep,
        client_group,
        deposits_rub,
        ns_rub,
        total_rub
    )

    SELECT
        @dt,
        g.client_group,

        ISNULL(b.deposits_rub, 0),
        ISNULL(b.ns_rub, 0),
        ISNULL(b.total_rub, 0)

    FROM client_groups g

    LEFT JOIN balances b
        ON b.client_group = g.client_group

    OPTION (MAXDOP 1, RECOMPILE);


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


/* СТАТУС РАСЧЕТА */

SELECT
    CASE
        WHEN COUNT(DISTINCT dt_rep)
             = DATEDIFF(day, @DateFrom, @DateTo) + 1
            THEN N'Готово'
        ELSE N'Балансы'
    END AS stage,

    COUNT(DISTINCT dt_rep) AS days_done,

    DATEDIFF(day, @DateFrom, @DateTo) + 1 AS days_total,

    MAX(dt_rep) AS last_date,

    DATEDIFF(second, @RunStart, SYSDATETIME()) AS run_seconds

FROM #rk_oct26_group_balances;


/* ИТОГ. Подневные остатки по категориям в млрд рублей */

SELECT
    dt_rep,
    client_group,

    CAST(
        deposits_rub / 1000000000.0
        AS decimal(20,6)
    ) AS deposits_bln,

    CAST(
        ns_rub / 1000000000.0
        AS decimal(20,6)
    ) AS ns_bln,

    CAST(
        total_rub / 1000000000.0
        AS decimal(20,6)
    ) AS total_bln

FROM #rk_oct26_group_balances

WHERE @IncludePrivate = 1

   OR client_group IN (
       N'новый клиент масс',
       N'старый клиент масс кто принес новые деньги',
       N'старый клиент масс кто не принес новые деньги'
   )

ORDER BY
    dt_rep,
    CASE client_group
        WHEN N'новый клиент масс' THEN 1
        WHEN N'старый клиент масс кто принес новые деньги' THEN 2
        WHEN N'старый клиент масс кто не принес новые деньги' THEN 3
        ELSE 4
    END;
