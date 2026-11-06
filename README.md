
USE ALM;
SET NOCOUNT ON;

DECLARE @Segment nvarchar(10) = N'Масс';

DROP TABLE IF EXISTS #periods;
DROP TABLE IF EXISTS #opened;
DROP TABLE IF EXISTS #new_clients;
DROP TABLE IF EXISTS #tracked;
DROP TABLE IF EXISTS #balances;
DROP TABLE IF EXISTS #daily_agg;
DROP TABLE IF EXISTS #daily;

IF @Segment NOT IN (N'Масс', N'УЧК')
    THROW 50001, 'Invalid segment', 1;


/* ============================================================
   1. ПЕРИОДЫ ТРЕХ РК
   ============================================================ */

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
   2. ДОГОВОРЫ, ОТКРЫТЫЕ В ПЕРИОД РК
   ============================================================ */

;WITH ranked AS (
    SELECT
        p.period_id,
        TRY_CAST(a.CON_ID AS bigint) AS con_id,
        a.CLI_ID AS cli_id,
        a.PROD_NAME,
        CAST(a.DT_OPEN AS date) AS dt_open,

        ROW_NUMBER() OVER (
            PARTITION BY p.period_id, a.CON_ID
            ORDER BY a.DT_OPEN, a.PROD_NAME
        ) AS rn

    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)

    INNER JOIN #periods p
        ON a.DT_OPEN >= p.dt_from
       AND a.DT_OPEN < DATEADD(day, 1, p.dt_to)

    WHERE a.DT_OPEN IS NOT NULL
      AND a.CLI_ID IS NOT NULL
      AND a.CON_ID IS NOT NULL
      AND a.cli_short_name = N'ФЛ'
      AND a.cur = 'RUR'

      AND a.PROD_NAME NOT IN (
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
)

SELECT
    period_id,
    con_id,
    cli_id,
    PROD_NAME,
    dt_open
INTO #opened
FROM ranked
WHERE rn = 1
  AND con_id IS NOT NULL;

CREATE UNIQUE CLUSTERED INDEX IX_opened
ON #opened (period_id, con_id);

CREATE INDEX IX_opened_client
ON #opened (period_id, cli_id);


/* ============================================================
   3. НОВЫЕ КЛИЕНТЫ

   По каждой РК отдельно проверяем историю до начала РК.
   ============================================================ */

;WITH candidates AS (
    SELECT DISTINCT
        period_id,
        cli_id
    FROM #opened
)

SELECT
    c.period_id,
    c.cli_id
INTO #new_clients

FROM candidates c

INNER JOIN #periods p
    ON p.period_id = c.period_id

WHERE NOT EXISTS (
    SELECT 1
    FROM LIQUIDITY.liq.DepositContract_all h WITH (NOLOCK)

    WHERE h.CLI_ID = c.cli_id
      AND h.DT_OPEN < p.dt_from
      AND h.cli_short_name = N'ФЛ'
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
);

CREATE UNIQUE CLUSTERED INDEX IX_new_clients
ON #new_clients (period_id, cli_id);


/* ============================================================
   4. ДВА НЕЗАВИСИМЫХ ПОТОКА ДОГОВОРОВ

   А. Новые клиенты: все открытые договоры
   Б. Остальные: только вклады с НОВ / НДП / НДМ
   ============================================================ */

SELECT
    o.period_id,
    o.con_id,
    o.cli_id,
    o.dt_open,

    CASE
        WHEN n.cli_id IS NOT NULL
            THEN N'Новые клиенты'

        WHEN ISNULL(TRY_CAST(a.[Нов] AS int), 0) = 1
            THEN N'НОВ остальных'

        WHEN ISNULL(TRY_CAST(a.[НДП] AS int), 0) = 1
          OR ISNULL(TRY_CAST(a.[НДМ] AS int), 0) = 1
            THEN N'НДП/НДМ остальных'

        ELSE NULL
    END AS flow_category

INTO #tracked

FROM #opened o

LEFT JOIN #new_clients n
    ON n.period_id = o.period_id
   AND n.cli_id = o.cli_id

OUTER APPLY (
    SELECT TOP (1)
        a.[Нов],
        a.[НДП],
        a.[НДМ]

    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)

    WHERE TRY_CAST(a.CON_ID AS bigint) = o.con_id
      AND n.cli_id IS NULL

    ORDER BY
        a.DT_UPDATE DESC,
        a.loaddate DESC
) a

WHERE n.cli_id IS NOT NULL
   OR ISNULL(TRY_CAST(a.[Нов] AS int), 0) = 1
   OR ISNULL(TRY_CAST(a.[НДП] AS int), 0) = 1
   OR ISNULL(TRY_CAST(a.[НДМ] AS int), 0) = 1;

CREATE UNIQUE CLUSTERED INDEX IX_tracked
ON #tracked (period_id, con_id);


/* ============================================================
   5. ОСТАТКИ

   Строго повторяем классификацию исходного
   балансового скрипта MASS / УЧК.
   ============================================================ */

SELECT
    t.period_id,
    CAST(v.dt_rep AS date) AS dt_rep,
    t.flow_category,
    t.cli_id,
    t.con_id,
    v.section_name,

    SUM(CAST(v.out_rub AS decimal(28,2))) AS volume

INTO #balances

FROM #tracked t

INNER JOIN #periods p
    ON p.period_id = t.period_id

INNER JOIN ALM.ALM.VW_balance_rest_all v WITH (NOLOCK)
    ON v.con_id = t.con_id
   AND v.dt_rep BETWEEN p.dt_from AND p.dt_to

CROSS APPLY (
    SELECT
        CASE
            WHEN v.section_name = N'Накопительный счёт'
             AND v.TSEGMENTNAME = N'ДЧБО'
                THEN N'НС УЧК'

            WHEN v.section_name = N'Накопительный счёт'
                THEN N'НС Розница'

            WHEN v.section_name = N'Срочные'
             AND v.PROD_NAME_res = N'ДОМа надёжно'
                THEN N'Банки.ру'

            WHEN v.section_name = N'Срочные'
             AND v.PROD_NAME_res = N'Всё в ДОМ'
                THEN N'Сравни.ру'

            WHEN v.section_name = N'Срочные'
             AND v.PROD_NAME_res IN (
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

            WHEN v.section_name = N'Срочные'
             AND v.PROD_NAME_res = N'Надёжный'
                THEN N'ФУ'

            WHEN v.section_name = N'Срочные'
             AND v.TSEGMENTNAME = N'ДЧБО'
                THEN N'Стандартная линейка УЧК'

            WHEN v.section_name = N'Срочные'
                THEN N'Стандартная линейка Розница'

        END AS balance_category
) bc

WHERE v.section_name IN (
    N'Срочные',
    N'Накопительный счёт'
)
  AND v.block_name = N'Привлечение ФЛ'
  AND v.od_flag = 1
  AND v.cur = '810'
  AND v.out_rub IS NOT NULL
  AND v.out_rub >= 0

  AND (
      v.section_name <> N'Накопительный счёт'
      OR v.acc_role = N'LIAB'
  )

  AND (
      (
          @Segment = N'Масс'
          AND bc.balance_category IN (
              N'НС Розница',
              N'Стандартная линейка Розница'
          )
      )
      OR
      (
          @Segment = N'УЧК'
          AND bc.balance_category IN (
              N'НС УЧК',
              N'Стандартная линейка УЧК'
          )
      )
  )

  AND (
      t.flow_category = N'Новые клиенты'
      OR v.section_name = N'Срочные'
  )

GROUP BY
    t.period_id,
    CAST(v.dt_rep AS date),
    t.flow_category,
    t.cli_id,
    t.con_id,
    v.section_name

OPTION (RECOMPILE);

CREATE CLUSTERED INDEX IX_balances
ON #balances (period_id, dt_rep, flow_category);


/* ============================================================
   КОНТРОЛЬ 1.
   КОЛИЧЕСТВО НОВЫХ КЛИЕНТОВ ПО АРХИВУ
   ============================================================ */

SELECT
    p.period_name,
    COUNT(n.cli_id) AS new_clients_archive

FROM #periods p

LEFT JOIN #new_clients n
    ON n.period_id = p.period_id

GROUP BY
    p.period_id,
    p.period_name

ORDER BY p.period_id;


/* ============================================================
   КОНТРОЛЬ 2.
   НОВЫЕ КЛИЕНТЫ: ОСТАТКИ НА КОНЕЦ КАЖДОЙ РК
   ============================================================ */

SELECT
    p.period_name,
    p.dt_to AS control_date,

    COUNT(DISTINCT b.cli_id) AS clients,
    COUNT(DISTINCT b.con_id) AS contracts,

    CAST(
        SUM(CASE
            WHEN b.section_name = N'Срочные'
            THEN b.volume ELSE 0
        END) / 1000000000.0
        AS decimal(18,6)
    ) AS deposits_bn,

    CAST(
        SUM(CASE
            WHEN b.section_name = N'Накопительный счёт'
            THEN b.volume ELSE 0
        END) / 1000000000.0
        AS decimal(18,6)
    ) AS ns_bn,

    CAST(
        ISNULL(SUM(b.volume), 0) / 1000000000.0
        AS decimal(18,6)
    ) AS total_bn

FROM #periods p

LEFT JOIN #balances b
    ON b.period_id = p.period_id
   AND b.dt_rep = p.dt_to
   AND b.flow_category = N'Новые клиенты'

GROUP BY
    p.period_id,
    p.period_name,
    p.dt_to

ORDER BY p.period_id;


/* ============================================================
   КОНТРОЛЬ 3.
   ОСТАЛЬНЫЕ КЛИЕНТЫ: НОВ / НДП / НДМ НА КОНЕЦ РК
   ============================================================ */

SELECT
    p.period_name,
    p.dt_to AS control_date,
    g.flow_category,

    COUNT(DISTINCT b.cli_id) AS clients,
    COUNT(DISTINCT b.con_id) AS contracts,

    CAST(
        ISNULL(SUM(b.volume), 0) / 1000000000.0
        AS decimal(18,6)
    ) AS balance_bn

FROM #periods p

CROSS JOIN (VALUES
    (N'НОВ остальных'),
    (N'НДП/НДМ остальных')
) g(flow_category)

LEFT JOIN #balances b
    ON b.period_id = p.period_id
   AND b.dt_rep = p.dt_to
   AND b.flow_category = g.flow_category

GROUP BY
    p.period_id,
    p.period_name,
    p.dt_to,
    g.flow_category

ORDER BY
    p.period_id,
    g.flow_category;


/* ============================================================
   КОНТРОЛЬ 4.
   СКОЛЬКО ДОГОВОРОВ НОВ ОТКРЫТО ОСТАЛЬНЫМИ КЛИЕНТАМИ
   ============================================================ */

SELECT
    p.period_name,

    COUNT(DISTINCT t.cli_id) AS nov_clients,
    COUNT(DISTINCT t.con_id) AS nov_contracts

FROM #periods p

LEFT JOIN #tracked t
    ON t.period_id = p.period_id
   AND t.flow_category = N'НОВ остальных'

GROUP BY
    p.period_id,
    p.period_name

ORDER BY p.period_id;


/* ============================================================
   6. ПОДНЕВНАЯ АГРЕГАЦИЯ
   ============================================================ */

SELECT
    period_id,
    dt_rep,

    SUM(CASE
        WHEN flow_category = N'Новые клиенты'
         AND section_name = N'Срочные'
        THEN volume ELSE 0
    END) AS new_deposits,

    SUM(CASE
        WHEN flow_category = N'Новые клиенты'
         AND section_name = N'Накопительный счёт'
        THEN volume ELSE 0
    END) AS new_ns,

    SUM(CASE
        WHEN flow_category = N'НОВ остальных'
        THEN volume ELSE 0
    END) AS nov_others,

    SUM(CASE
        WHEN flow_category = N'НДП/НДМ остальных'
        THEN volume ELSE 0
    END) AS ndp_ndm_others

INTO #daily_agg

FROM #balances

GROUP BY
    period_id,
    dt_rep;

CREATE UNIQUE CLUSTERED INDEX IX_daily_agg
ON #daily_agg (period_id, dt_rep);


/* ============================================================
   7. КАЛЕНДАРЬ: ВСЕ ДНИ КАЖДОЙ РК
   ============================================================ */

;WITH calendar AS (
    SELECT
        period_id,
        dt_from AS dt_rep,
        dt_to
    FROM #periods

    UNION ALL

    SELECT
        period_id,
        DATEADD(day, 1, dt_rep),
        dt_to
    FROM calendar
    WHERE dt_rep < dt_to
)

SELECT
    p.period_id,
    p.period_name,
    c.dt_rep,

    DATEDIFF(day, p.dt_from, c.dt_rep) + 1 AS day_num,

    CAST(
        ISNULL(a.new_deposits, 0) / 1000000000.0
        AS decimal(18,6)
    ) AS new_deposits_bn,

    CAST(
        ISNULL(a.new_ns, 0) / 1000000000.0
        AS decimal(18,6)
    ) AS new_ns_bn,

    CAST(
        (
            ISNULL(a.new_deposits, 0)
            + ISNULL(a.new_ns, 0)
        ) / 1000000000.0
        AS decimal(18,6)
    ) AS new_clients_total_bn,

    CAST(
        ISNULL(a.nov_others, 0) / 1000000000.0
        AS decimal(18,6)
    ) AS nov_others_bn,

    CAST(
        ISNULL(a.ndp_ndm_others, 0) / 1000000000.0
        AS decimal(18,6)
    ) AS ndp_ndm_others_bn,

    CAST(
        (
            ISNULL(a.nov_others, 0)
            + ISNULL(a.ndp_ndm_others, 0)
        ) / 1000000000.0
        AS decimal(18,6)
    ) AS other_clients_total_bn,

    CAST(
        (
            ISNULL(a.new_deposits, 0)
            + ISNULL(a.new_ns, 0)
            + ISNULL(a.nov_others, 0)
            + ISNULL(a.ndp_ndm_others, 0)
        ) / 1000000000.0
        AS decimal(18,6)
    ) AS total_bn

INTO #daily

FROM calendar c

INNER JOIN #periods p
    ON p.period_id = c.period_id

LEFT JOIN #daily_agg a
    ON a.period_id = c.period_id
   AND a.dt_rep = c.dt_rep

OPTION (MAXRECURSION 0);


/* ============================================================
   РЕЗУЛЬТАТ 5.
   ПОДНЕВНАЯ ДИНАМИКА ДВУХ ПОТОКОВ
   ============================================================ */

SELECT
    period_name,
    dt_rep,
    day_num,

    new_deposits_bn,
    new_ns_bn,
    new_clients_total_bn,

    nov_others_bn,
    ndp_ndm_others_bn,
    other_clients_total_bn,

    total_bn

FROM #daily

ORDER BY
    period_id,
    dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ 6.
   ИТОГОВАЯ ПАНЕЛЬ ТРЕХ РК
   ============================================================ */

;WITH days AS (
    SELECT day_num
    FROM (VALUES
        (1), (4), (7), (10), (13),
        (16), (19), (22), (25), (28),
        (32), (33), (35), (36), (38)
    ) v(day_num)
)

SELECT
    CONVERT(
        varchar(10),
        DATEADD(day, d.day_num - 1, CONVERT(date, '2025-10-01')),
        104
    ) AS [Column1],

    MAX(CASE
        WHEN x.period_id = 1
        THEN x.total_bn
    END) AS [РК октябрь 26],

    MAX(CASE
        WHEN x.period_id = 2
        THEN x.total_bn
    END) AS [РК март 26],

    MAX(CASE
        WHEN x.period_id = 3
        THEN x.total_bn
    END) AS [РК октябрь 25]

FROM days d

LEFT JOIN #daily x
    ON x.day_num = d.day_num

GROUP BY d.day_num
ORDER BY d.day_num;
