
USE ALM;
SET NOCOUNT ON;

DECLARE @DateFrom date = '2026-09-01';
DECLARE @DateTo   date = '2026-11-14';
DECLARE @dt       date = @DateFrom;
DECLARE @dtText   varchar(10);
DECLARE @err      nvarchar(4000);

DROP TABLE IF EXISTS #day;
DROP TABLE IF EXISTS #clients;
DROP TABLE IF EXISTS #result;
DROP TABLE IF EXISTS #missing;


/* 1. Временные таблицы */

SELECT TOP (0)
    t.CLI_ID,
    CAST(0 AS decimal(38,2)) AS deposits_rub,
    CAST(0 AS decimal(38,2)) AS ns_rub,
    CAST(0 AS decimal(38,2)) AS dvs_rub
INTO #day
FROM ALM.ALM.balance_rest_all t;

CREATE UNIQUE CLUSTERED INDEX IX_day
ON #day (CLI_ID);


SELECT TOP (0)
    CLI_ID,
    CAST(NULL AS date) AS first_date,
    CAST(NULL AS bit) AS existing_flag,
    CAST(NULL AS tinyint) AS bucket_id
INTO #clients
FROM #day;

CREATE UNIQUE CLUSTERED INDEX IX_clients
ON #clients (CLI_ID);


CREATE TABLE #result (
    dt_rep date,
    existing_flag bit,
    bucket_id tinyint,
    deposits_rub decimal(38,2),
    ns_rub decimal(38,2),
    dvs_rub decimal(38,2)
);

CREATE TABLE #missing (
    dt_rep date PRIMARY KEY
);


/* 2. Последовательная обработка по дням */

WHILE @dt <= @DateTo
BEGIN

    SET @dtText = CONVERT(varchar(10), @dt, 120);

    RAISERROR(
        N'Читаю баланс за %s',
        0, 1, @dtText
    ) WITH NOWAIT;

    TRUNCATE TABLE #day;

    INSERT INTO #day (
        CLI_ID,
        deposits_rub,
        ns_rub,
        dvs_rub
    )
    SELECT
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
        END)

    FROM ALM.ALM.balance_rest_all t WITH (NOLOCK)

    WHERE t.dt_rep = @dt

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

    GROUP BY t.CLI_ID

    HAVING SUM(t.out_rub) > 0

    OPTION (MAXDOP 1, RECOMPILE);


    /* 3. Контроль отсутствующих срезов */

    IF NOT EXISTS (SELECT 1 FROM #day)
    BEGIN

        IF @dt = @DateFrom
        BEGIN
            SET @err =
                N'Нет положительных остатков на исходную дату '
                + CONVERT(nvarchar(10), @dt, 120)
                + N'. Классификация клиентов невозможна.';

            THROW 50001, @err, 1;
        END;

        INSERT INTO #missing VALUES (@dt);

        RAISERROR(
            N'ВНИМАНИЕ: за %s нет положительных остатков после фильтров. Дата пропущена.',
            0, 1, @dtText
        ) WITH NOWAIT;

        SET @dt = DATEADD(day, 1, @dt);
        CONTINUE;

    END;


    /* 4. Определяем новых клиентов и фиксируем бакет */

    INSERT INTO #clients (
        CLI_ID,
        first_date,
        existing_flag,
        bucket_id
    )

    SELECT
        d.CLI_ID,
        @dt,

        CASE WHEN @dt = @DateFrom
            THEN 1 ELSE 0
        END,

        CASE
            WHEN d.deposits_rub + d.ns_rub + d.dvs_rub
                <= 500000 THEN 1

            WHEN d.deposits_rub + d.ns_rub + d.dvs_rub
                <= 800000 THEN 2

            WHEN d.deposits_rub + d.ns_rub + d.dvs_rub
                <= 1000000 THEN 3

            WHEN d.deposits_rub + d.ns_rub + d.dvs_rub
                <= 1500000 THEN 4

            WHEN d.deposits_rub + d.ns_rub + d.dvs_rub
                <= 3000000 THEN 5

            WHEN d.deposits_rub + d.ns_rub + d.dvs_rub
                <= 5000000 THEN 6

            WHEN d.deposits_rub + d.ns_rub + d.dvs_rub
                <= 10000000 THEN 7

            WHEN d.deposits_rub + d.ns_rub + d.dvs_rub
                <= 30000000 THEN 8

            ELSE 9
        END

    FROM #day d

    WHERE NOT EXISTS (
        SELECT 1
        FROM #clients c
        WHERE c.CLI_ID = d.CLI_ID
    );


    /* 5. Агрегируем текущий день */

    INSERT INTO #result (
        dt_rep,
        existing_flag,
        bucket_id,
        deposits_rub,
        ns_rub,
        dvs_rub
    )

    SELECT
        @dt,
        c.existing_flag,
        c.bucket_id,

        SUM(d.deposits_rub),
        SUM(d.ns_rub),
        SUM(d.dvs_rub)

    FROM #day d

    INNER JOIN #clients c
        ON c.CLI_ID = d.CLI_ID

    GROUP BY
        c.existing_flag,
        c.bucket_id;


    SET @dt = DATEADD(day, 1, @dt);

END;


/* 6. Итоговая подневная динамика */

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

    CASE WHEN m.dt_rep IS NOT NULL
        THEN NULL
        ELSE CAST(
            ISNULL(r.deposits_rub, 0) / 1000000000.0
            AS decimal(20,6)
        )
    END AS deposits_bln,

    CASE WHEN m.dt_rep IS NOT NULL
        THEN NULL
        ELSE CAST(
            ISNULL(r.ns_rub, 0) / 1000000000.0
            AS decimal(20,6)
        )
    END AS ns_bln,

    CASE WHEN m.dt_rep IS NOT NULL
        THEN NULL
        ELSE CAST(
            ISNULL(r.dvs_rub, 0) / 1000000000.0
            AS decimal(20,6)
        )
    END AS dvs_bln,

    CASE WHEN m.dt_rep IS NOT NULL
        THEN NULL
        ELSE CAST(
            (
                ISNULL(r.deposits_rub, 0)
              + ISNULL(r.ns_rub, 0)
              + ISNULL(r.dvs_rub, 0)
            ) / 1000000000.0
            AS decimal(20,6)
        )
    END AS total_bln,

    CASE WHEN m.dt_rep IS NOT NULL
        THEN N'Нет среза после фильтров'
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

LEFT JOIN #missing m
    ON m.dt_rep = cal.dt_rep

ORDER BY
    cal.dt_rep,
    s.existing_flag DESC,
    b.bucket_id

OPTION (MAXRECURSION 0);


/* 7. Отдельно выводим проблемные даты */

SELECT
    dt_rep AS missing_date
FROM #missing
ORDER BY dt_rep;
