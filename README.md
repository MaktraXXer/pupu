
USE ALM;
SET NOCOUNT ON;

DECLARE @DateFrom date = '2026-09-01';
DECLARE @DateTo date = '2026-11-14';

DECLARE @BatchDays int = 1;

DECLARE @dt date = @DateFrom;
DECLARE @BatchEnd date;
DECLARE @BatchStartTime datetime2;
DECLARE @StartTime datetime2 = SYSDATETIME();

DECLARE @TotalDays int = DATEDIFF(day, @DateFrom, @DateTo) + 1;
DECLARE @DoneDays int;
DECLARE @Pct int;
DECLARE @NewClients int;
DECLARE @Progress nvarchar(500);

IF @BatchDays NOT BETWEEN 1 AND 7
    THROW 50001, 'BatchDays must be between 1 and 7', 1;

IF @DateFrom > @DateTo
    THROW 50002, 'DateFrom is greater than DateTo', 1;

DROP TABLE IF EXISTS #batch;
DROP TABLE IF EXISTS #clients;
DROP TABLE IF EXISTS #result;
DROP TABLE IF EXISTS #covered;


/* 1. Временная таблица для одного пакета дат.
   CLI_ID наследует тип из исходной вьюхи.
   TOP (0) не загружает балансовые данные. */

SELECT TOP (0)
    CAST(t.dt_rep AS date) AS dt_rep,
    t.CLI_ID,

    CAST(0 AS decimal(38,2)) AS deposits_rub,
    CAST(0 AS decimal(38,2)) AS ns_rub,
    CAST(0 AS decimal(38,2)) AS dvs_rub,
    CAST(0 AS decimal(38,2)) AS total_rub

INTO #batch

FROM ALM.ALM.VW_balance_rest_all t;

CREATE UNIQUE CLUSTERED INDEX IX_batch
ON #batch (CLI_ID, dt_rep);


/* 2. Справочник клиентов.
   Одна строка на CLI_ID.
   Бакет и статус фиксируются навсегда. */

SELECT TOP (0)
    CLI_ID,
    CAST(NULL AS date) AS first_date,
    CAST(NULL AS bit) AS existing_flag,
    CAST(NULL AS tinyint) AS bucket_id

INTO #clients

FROM #batch;

CREATE UNIQUE CLUSTERED INDEX IX_clients
ON #clients (CLI_ID);


/* 3. Итоговые агрегаты по дням */

CREATE TABLE #result (
    dt_rep date NOT NULL,
    existing_flag bit NOT NULL,
    bucket_id tinyint NOT NULL,

    deposits_rub decimal(38,2) NOT NULL,
    ns_rub decimal(38,2) NOT NULL,
    dvs_rub decimal(38,2) NOT NULL,
    total_rub decimal(38,2) NOT NULL
);

CREATE UNIQUE CLUSTERED INDEX IX_result
ON #result (dt_rep, existing_flag, bucket_id);


/* Даты, за которые получены положительные остатки */

CREATE TABLE #covered (
    dt_rep date NOT NULL PRIMARY KEY
);


/* 4. Основной цикл */

RAISERROR(
    N'Начало обработки. Источник: VW_balance_rest_all',
    0, 1
) WITH NOWAIT;

WHILE @dt <= @DateTo
BEGIN

    SET @BatchStartTime = SYSDATETIME();

    SET @BatchEnd = DATEADD(day, @BatchDays - 1, @dt);

    IF @BatchEnd > @DateTo
        SET @BatchEnd = @DateTo;

    TRUNCATE TABLE #batch;


    /* ЕДИНСТВЕННОЕ обращение к большой вьюхе
       за текущий пакет дат */

    INSERT INTO #batch (
        dt_rep,
        CLI_ID,
        deposits_rub,
        ns_rub,
        dvs_rub,
        total_rub
    )

    SELECT
        CAST(t.dt_rep AS date),
        t.CLI_ID,

        SUM(CASE
            WHEN t.section_name = N'Срочные'
                THEN CAST(t.out_rub AS decimal(28,2))
            ELSE 0
        END),

        SUM(CASE
            WHEN t.section_name = N'Накопительный счёт'
                THEN CAST(t.out_rub AS decimal(28,2))
            ELSE 0
        END),

        SUM(CASE
            WHEN t.section_name = N'До востребования'
                THEN CAST(t.out_rub AS decimal(28,2))
            ELSE 0
        END),

        SUM(CAST(t.out_rub AS decimal(28,2)))

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    WHERE t.dt_rep >= @dt
      AND t.dt_rep < DATEADD(day, 1, @BatchEnd)

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

      AND t.PROD_NAME_res NOT IN (
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
          N'ДОМанадёжно',
          N'Всё в ДОМ'
      )

    GROUP BY
        CAST(t.dt_rep AS date),
        t.CLI_ID

    HAVING SUM(t.out_rub) > 0

    OPTION (MAXDOP 1, RECOMPILE);


    /* Обязательная проверка исходного дня */

    IF @dt = @DateFrom
       AND NOT EXISTS (
           SELECT 1
           FROM #batch
           WHERE dt_rep = @DateFrom
       )
    BEGIN
        THROW 50003,
            'Нет положительных остатков на 01.09 во VW_balance_rest_all',
            1;
    END;


    /* Регистрируем даты с данными */

    INSERT INTO #covered (dt_rep)
    SELECT DISTINCT dt_rep
    FROM #batch;


    /* 5. Находим впервые появившихся клиентов.
       Если клиент есть в нескольких днях пакета,
       берем его самый ранний день. */

    ;WITH first_seen AS (
        SELECT
            CLI_ID,
            MIN(dt_rep) AS first_date
        FROM #batch
        GROUP BY CLI_ID
    )

    INSERT INTO #clients (
        CLI_ID,
        first_date,
        existing_flag,
        bucket_id
    )

    SELECT
        f.CLI_ID,
        f.first_date,

        CASE
            WHEN f.first_date = @DateFrom THEN 1
            ELSE 0
        END,

        CASE
            WHEN b.total_rub <= 500000 THEN 1
            WHEN b.total_rub <= 800000 THEN 2
            WHEN b.total_rub <= 1000000 THEN 3
            WHEN b.total_rub <= 1500000 THEN 4
            WHEN b.total_rub <= 3000000 THEN 5
            WHEN b.total_rub <= 5000000 THEN 6
            WHEN b.total_rub <= 10000000 THEN 7
            WHEN b.total_rub <= 30000000 THEN 8
            ELSE 9
        END

    FROM first_seen f

    INNER JOIN #batch b
        ON b.CLI_ID = f.CLI_ID
       AND b.dt_rep = f.first_date

    WHERE NOT EXISTS (
        SELECT 1
        FROM #clients c
        WHERE c.CLI_ID = f.CLI_ID
    );

    SET @NewClients = @@ROWCOUNT;


    /* 6. Подневные агрегаты по фиксированным бакетам */

    INSERT INTO #result (
        dt_rep,
        existing_flag,
        bucket_id,
        deposits_rub,
        ns_rub,
        dvs_rub,
        total_rub
    )

    SELECT
        b.dt_rep,
        c.existing_flag,
        c.bucket_id,

        SUM(b.deposits_rub),
        SUM(b.ns_rub),
        SUM(b.dvs_rub),
        SUM(b.total_rub)

    FROM #batch b

    INNER JOIN #clients c
        ON c.CLI_ID = b.CLI_ID

    GROUP BY
        b.dt_rep,
        c.existing_flag,
        c.bucket_id;


    /* 7. Прогресс выполнения */

    SET @DoneDays =
        DATEDIFF(day, @DateFrom, @BatchEnd) + 1;

    SET @Pct = @DoneDays * 100 / @TotalDays;

    SET @Progress = CONCAT(
        N'[',
        REPLICATE(N'#', @Pct / 5),
        REPLICATE(N'-', 20 - @Pct / 5),
        N'] ',
        @Pct,
        N'% | ',
        CONVERT(varchar(10), @dt, 120),
        N' - ',
        CONVERT(varchar(10), @BatchEnd, 120),
        N' | ',
        @DoneDays,
        N'/',
        @TotalDays,
        N' дней',
        N' | впервые встретились: ',
        @NewClients,
        N' | секунд за пакет: ',
        DATEDIFF(second, @BatchStartTime, SYSDATETIME())
    );

    RAISERROR(N'%s', 0, 1, @Progress) WITH NOWAIT;

    SET @dt = DATEADD(day, 1, @BatchEnd);

END;


/* 8. Итоговый результат */

;WITH calendar AS (
    SELECT @DateFrom AS dt_rep

    UNION ALL

    SELECT DATEADD(day, 1, dt_rep)
    FROM calendar
    WHERE dt_rep < @DateTo
)

SELECT
    cal.dt_rep,

    CASE s.existing_flag
        WHEN 1 THEN N'Действующий'
        ELSE N'Новый'
    END AS client_type,

    b.bucket_id,

    CASE b.bucket_id
        WHEN 1 THEN N'До 500 тыс.'
        WHEN 2 THEN N'500–800 тыс.'
        WHEN 3 THEN N'800 тыс.–1 млн'
        WHEN 4 THEN N'1–1,5 млн'
        WHEN 5 THEN N'1,5–3 млн'
        WHEN 6 THEN N'3–5 млн'
        WHEN 7 THEN N'5–10 млн'
        WHEN 8 THEN N'10–30 млн'
        WHEN 9 THEN N'Более 30 млн'
    END AS balance_bucket,

    CASE WHEN v.dt_rep IS NULL THEN NULL
         ELSE CAST(
             ISNULL(r.deposits_rub, 0) / 1000000000.0
             AS decimal(20,6)
         )
    END AS deposits_bln,

    CASE WHEN v.dt_rep IS NULL THEN NULL
         ELSE CAST(
             ISNULL(r.ns_rub, 0) / 1000000000.0
             AS decimal(20,6)
         )
    END AS ns_bln,

    CASE WHEN v.dt_rep IS NULL THEN NULL
         ELSE CAST(
             ISNULL(r.dvs_rub, 0) / 1000000000.0
             AS decimal(20,6)
         )
    END AS dvs_bln,

    CASE WHEN v.dt_rep IS NULL THEN NULL
         ELSE CAST(
             ISNULL(r.total_rub, 0) / 1000000000.0
             AS decimal(20,6)
         )
    END AS total_bln,

    CASE
        WHEN v.dt_rep IS NULL
            THEN N'Нет положительных остатков после фильтров'
        ELSE N'OK'
    END AS data_status

FROM calendar cal

CROSS JOIN (
    VALUES (1), (0)
) s(existing_flag)

CROSS JOIN (
    VALUES (1),(2),(3),(4),(5),(6),(7),(8),(9)
) b(bucket_id)

LEFT JOIN #result r
    ON r.dt_rep = cal.dt_rep
   AND r.existing_flag = s.existing_flag
   AND r.bucket_id = b.bucket_id

LEFT JOIN #covered v
    ON v.dt_rep = cal.dt_rep

ORDER BY
    cal.dt_rep,
    s.existing_flag DESC,
    b.bucket_id

OPTION (MAXRECURSION 0);


/* 9. Контроль отсутствующих дат */

;WITH calendar AS (
    SELECT @DateFrom AS dt_rep
    UNION ALL
    SELECT DATEADD(day, 1, dt_rep)
    FROM calendar
    WHERE dt_rep < @DateTo
)

SELECT
    cal.dt_rep AS missing_date
FROM calendar cal
LEFT JOIN #covered v
    ON v.dt_rep = cal.dt_rep
WHERE v.dt_rep IS NULL
ORDER BY cal.dt_rep
OPTION (MAXRECURSION 0);


/* 10. Время полного выполнения */

SELECT
    DATEDIFF(second, @StartTime, SYSDATETIME())
        AS total_execution_seconds,

    (SELECT COUNT_BIG(*) FROM #clients)
        AS unique_clients;
