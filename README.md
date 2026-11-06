
USE [ALM];
SET NOCOUNT ON;

DROP TABLE IF EXISTS #periods;
DROP TABLE IF EXISTS #mass_grouped;
DROP TABLE IF EXISTS #mass_daily;
DROP TABLE IF EXISTS #mass_clients;

CREATE TABLE #periods (
    period_id int PRIMARY KEY,
    period_name nvarchar(50),
    dt_from date,
    dt_to date
);

INSERT INTO #periods VALUES
    (1, N'Октябрь 2026', '2026-10-01', '2026-11-04'),
    (2, N'Март 2026',    '2026-03-01', '2026-04-07'),
    (3, N'Октябрь 2025', '2025-10-01', '2025-11-05');


/* ============================================================
   1. БАЛАНС MASS ПО ТРЕМ РК
   КЛАССИФИКАЦИЯ КАК В ИСХОДНОМ СКРИПТЕ
   ============================================================ */

;WITH base AS (
    SELECT
        p.period_id,
        CAST(t.dt_rep AS date) AS dt_rep,
        t.cli_id,
        t.section_name,
        t.TSEGMENTNAME,
        t.PROD_NAME_res,
        CAST(t.out_rub AS decimal(28,2)) AS out_rub

    FROM ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)

    INNER JOIN #periods p
        ON t.dt_rep BETWEEN p.dt_from AND p.dt_to

    WHERE t.section_name IN (
        N'Срочные',
        N'Накопительный счёт'
    )
      AND t.block_name = N'Привлечение ФЛ'
      AND t.od_flag = 1
      AND t.cur = '810'
      AND t.out_rub IS NOT NULL
      AND t.out_rub >= 0
      AND (
          t.section_name <> N'Накопительный счёт'
          OR t.acc_role = N'LIAB'
      )
),

labeled AS (
    SELECT
        period_id,
        dt_rep,
        cli_id,

        CASE
            WHEN section_name = N'Накопительный счёт'
             AND TSEGMENTNAME = N'ДЧБО'
                THEN N'НС УЧК'

            WHEN section_name = N'Накопительный счёт'
                THEN N'НС Розница'

            WHEN section_name = N'Срочные'
             AND PROD_NAME_res = N'ДОМа надёжно'
                THEN N'Банки.ру'

            WHEN section_name = N'Срочные'
             AND PROD_NAME_res = N'Всё в ДОМ'
                THEN N'Сравни.ру'

            WHEN section_name = N'Срочные'
             AND PROD_NAME_res IN (
                 N'Надёжный прайм',
                 N'НадёжныйVIP',
                 N'Надёжный премиум',
                 N'Надёжный промо',
                 N'Надёжный старт',
                 N'Надёжный Т2',
                 N'Надёжный Мегафон',
                 N'Надёжный процент',
                 N'Могучий'
             )
                THEN N'Промо ФУ'

            WHEN section_name = N'Срочные'
             AND PROD_NAME_res = N'Надёжный'
                THEN N'ФУ'

            WHEN section_name = N'Срочные'
             AND TSEGMENTNAME = N'ДЧБО'
                THEN N'Стандартная линейка УЧК'

            WHEN section_name = N'Срочные'
                THEN N'Стандартная линейка Розница'
        END AS category,

        out_rub
    FROM base
)

SELECT
    period_id,
    dt_rep,
    category,
    cli_id,
    SUM(out_rub) AS volume,
    GROUPING(dt_rep) AS is_client_group

INTO #mass_grouped

FROM labeled

WHERE category IN (
    N'НС Розница',
    N'Стандартная линейка Розница'
)

GROUP BY GROUPING SETS (
    (period_id, dt_rep, category),
    (period_id, cli_id)
)

OPTION (RECOMPILE);


/* ============================================================
   2. ПОДНЕВНЫЕ ОСТАТКИ
   ============================================================ */

SELECT
    period_id,
    dt_rep,
    category,
    volume AS sum_out_rub
INTO #mass_daily
FROM #mass_grouped
WHERE is_client_group = 0;

CREATE CLUSTERED INDEX IX_mass_daily
ON #mass_daily (period_id, dt_rep);


/* ============================================================
   РЕЗУЛЬТАТ 1. ОКТЯБРЬ 2026
   ============================================================ */

SELECT
    dt_rep AS dtrep,
    category,
    sum_out_rub,
    CAST(sum_out_rub / 1000000000.0
         AS decimal(18,6)) AS volume_bn
FROM #mass_daily
WHERE period_id = 1
ORDER BY
    dt_rep,
    CASE category
        WHEN N'Стандартная линейка Розница' THEN 1
        WHEN N'НС Розница' THEN 2
    END;


/* ============================================================
   РЕЗУЛЬТАТ 2. МАРТ 2026
   ============================================================ */

SELECT
    dt_rep AS dtrep,
    category,
    sum_out_rub,
    CAST(sum_out_rub / 1000000000.0
         AS decimal(18,6)) AS volume_bn
FROM #mass_daily
WHERE period_id = 2
ORDER BY
    dt_rep,
    CASE category
        WHEN N'Стандартная линейка Розница' THEN 1
        WHEN N'НС Розница' THEN 2
    END;


/* ============================================================
   РЕЗУЛЬТАТ 3. ОКТЯБРЬ 2025
   ============================================================ */

SELECT
    dt_rep AS dtrep,
    category,
    sum_out_rub,
    CAST(sum_out_rub / 1000000000.0
         AS decimal(18,6)) AS volume_bn
FROM #mass_daily
WHERE period_id = 3
ORDER BY
    dt_rep,
    CASE category
        WHEN N'Стандартная линейка Розница' THEN 1
        WHEN N'НС Розница' THEN 2
    END;


/* ============================================================
   РЕЗУЛЬТАТ 4. ПАНЕЛЬ ТРЕХ РК
   ============================================================ */

;WITH days AS (
    SELECT day_num
    FROM (VALUES
        (1), (4), (7), (10), (13),
        (16), (19), (22), (25), (28),
        (32), (33), (35), (36), (38)
    ) v(day_num)
),

dates AS (
    SELECT
        p.period_id,
        d.day_num,
        DATEADD(day, d.day_num - 1, p.dt_from) AS dt_rep
    FROM #periods p
    CROSS JOIN days d
    WHERE DATEADD(day, d.day_num - 1, p.dt_from) <= p.dt_to
),

balances AS (
    SELECT
        d.period_id,
        d.day_num,
        d.dt_rep,
        SUM(m.sum_out_rub) AS mass_volume
    FROM dates d

    LEFT JOIN #mass_daily m
        ON m.period_id = d.period_id
       AND m.dt_rep = d.dt_rep

    GROUP BY
        d.period_id,
        d.day_num,
        d.dt_rep
)

SELECT
    CONVERT(
        varchar(10),
        DATEADD(day, d.day_num - 1, '2025-10-01'),
        104
    ) AS [Column1],

    MAX(CASE WHEN b.period_id = 1
        THEN CAST(b.mass_volume / 1000000000.0
                  AS decimal(18,6))
    END) AS [РК октябрь 26],

    MAX(CASE WHEN b.period_id = 2
        THEN CAST(b.mass_volume / 1000000000.0
                  AS decimal(18,6))
    END) AS [РК март 26],

    MAX(CASE WHEN b.period_id = 3
        THEN CAST(b.mass_volume / 1000000000.0
                  AS decimal(18,6))
    END) AS [РК октябрь 25]

FROM days d
LEFT JOIN balances b
    ON b.day_num = d.day_num

GROUP BY d.day_num
ORDER BY d.day_num;


/* ============================================================
   3. СОХРАНЕНИЕ ВСЕХ CLI_ID MASS ПО КАЖДОЙ РК
   ============================================================ */

SELECT
    period_id,
    cli_id

INTO #mass_clients

FROM #mass_grouped

WHERE is_client_group = 1
  AND cli_id IS NOT NULL;

CREATE UNIQUE CLUSTERED INDEX IX_mass_clients
ON #mass_clients (period_id, cli_id);


/* ============================================================
   КОНТРОЛЬ СОХРАНЕННЫХ CLI_ID
   ============================================================ */

SELECT
    p.period_name,
    COUNT(m.cli_id) AS clients_count

FROM #periods p

LEFT JOIN #mass_clients m
    ON m.period_id = p.period_id

GROUP BY
    p.period_id,
    p.period_name

ORDER BY p.period_id;
