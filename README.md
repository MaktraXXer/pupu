
USE ALM;
SET NOCOUNT ON;

DECLARE @Segment nvarchar(10) = N'Масс';
DECLARE @ControlPeriod nvarchar(50) = N'Октябрь 2026';
DECLARE @ControlDate date = '2026-11-04';

IF @Segment NOT IN (N'Масс', N'УЧК')
    THROW 50001, 'Segment must be Масс or УЧК', 1;

DROP TABLE IF EXISTS #periods;
DROP TABLE IF EXISTS #contracts;
DROP TABLE IF EXISTS #campaign_clients;
DROP TABLE IF EXISTS #new_clients;
DROP TABLE IF EXISTS #tracked;
DROP TABLE IF EXISTS #snap;
DROP TABLE IF EXISTS #daily_agg;
DROP TABLE IF EXISTS #daily;

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
   1. ДОГОВОРЫ, ОТКРЫТЫЕ В ПЕРИОДЫ РК
   ============================================================ */

;WITH ranked AS (
    SELECT
        p.period_id,
        TRY_CAST(a.CON_ID AS bigint) AS con_id,
        a.CLI_ID AS cli_id,
        CAST(a.DT_OPEN AS date) AS dt_open,

        ROW_NUMBER() OVER (
            PARTITION BY p.period_id, a.CON_ID
            ORDER BY a.DT_OPEN, a.PROD_NAME
        ) AS rn

    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)

    INNER JOIN #periods p
        ON a.DT_OPEN >= p.dt_from
       AND a.DT_OPEN < DATEADD(day, 1, p.dt_to)

    WHERE a.cli_short_name = N'ФЛ'
      AND a.cur = 'RUR'
      AND a.CLI_ID IS NOT NULL
      AND TRY_CAST(a.CON_ID AS bigint) IS NOT NULL

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
    dt_open
INTO #contracts
FROM ranked
WHERE rn = 1;

CREATE UNIQUE CLUSTERED INDEX IX_contracts
ON #contracts (period_id, con_id);

CREATE INDEX IX_contracts_client
ON #contracts (period_id, cli_id);


/* ============================================================
   2. НОВЫЕ КЛИЕНТЫ КАЖДОЙ РК
   ============================================================ */

SELECT DISTINCT
    period_id,
    cli_id
INTO #campaign_clients
FROM #contracts;

CREATE UNIQUE CLUSTERED INDEX IX_campaign_clients
ON #campaign_clients (period_id, cli_id);

SELECT
    c.period_id,
    c.cli_id
INTO #new_clients
FROM #campaign_clients c

INNER JOIN #periods p
    ON p.period_id = c.period_id

WHERE NOT EXISTS (
    SELECT 1
    FROM LIQUIDITY.liq.DepositContract_all h WITH (NOLOCK)
    WHERE h.CLI_ID = c.cli_id
      AND h.DT_OPEN < p.dt_from
      AND h.cli_short_name = N'ФЛ'
      AND h.cur = 'RUR'
      AND h.DT_OPEN IS NOT NULL

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
   3. ДОГОВОРЫ НОВЫХ И ОСТАЛЬНЫХ КЛИЕНТОВ

   Для новых клиентов — все договоры периода.
   Для остальных — только НОВ / НДП / НДМ.
   ============================================================ */

;WITH classified AS (
    SELECT
        c.period_id,
        c.con_id,
        c.cli_id,
        c.dt_open,

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

    FROM #contracts c

    LEFT JOIN #new_clients n
        ON n.period_id = c.period_id
       AND n.cli_id = c.cli_id

    OUTER APPLY (
        SELECT TOP (1)
            a.[Нов],
            a.[НДП],
            a.[НДМ]
        FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)
        WHERE a.CON_ID = c.con_id
          AND n.cli_id IS NULL
        ORDER BY
            a.DT_UPDATE DESC,
            a.loaddate DESC
    ) a
)
SELECT
    period_id,
    con_id,
    cli_id,
    dt_open,
    flow_category
INTO #tracked
FROM classified
WHERE flow_category IS NOT NULL;

CREATE UNIQUE CLUSTERED INDEX IX_tracked
ON #tracked (period_id, con_id);


/* ============================================================
   4. ПОДНЕВНЫЕ ОСТАТКИ ПО БАЛАНСОВОЙ КЛАССИФИКАЦИИ
   ============================================================ */

SELECT
    t.period_id,
    CAST(v.dt_rep AS date) AS dt_rep,
    t.flow_category,
    t.cli_id,
    t.con_id,
    SUM(CAST(v.out_rub AS decimal(28,2))) AS volume

INTO #snap

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
    v.dt_rep,
    t.flow_category,
    t.cli_id,
    t.con_id

OPTION (RECOMPILE);

CREATE CLUSTERED INDEX IX_snap
ON #snap (period_id, dt_rep, flow_category, cli_id);


/* ============================================================
   РЕЗУЛЬТАТ 1. НОВЫЕ КЛИЕНТЫ ПО АРХИВУ
   ============================================================ */

SELECT
    p.period_name,
    COUNT(n.cli_id) AS new_clients_archive
FROM #periods p
LEFT JOIN #new_clients n
    ON n.period_id = p.period_id
GROUP BY p.period_id, p.period_name
ORDER BY p.period_id;


/* ============================================================
   РЕЗУЛЬТАТ 2. КОНТРОЛЬ НА КОНЕЦ КАЖДОЙ РК
   ============================================================ */

SELECT
    p.period_name,
    p.dt_to AS control_date,
    g.flow_category,
    COUNT(DISTINCT s.cli_id) AS clients,
    COUNT(DISTINCT s.con_id) AS contracts,
    CAST(
        SUM(ISNULL(s.volume, 0)) / 1000000000.0
        AS decimal(18,6)
    ) AS balance_bn

FROM #periods p

CROSS JOIN (VALUES
    (N'Новые клиенты'),
    (N'НОВ остальных'),
    (N'НДП/НДМ остальных')
) g(flow_category)

LEFT JOIN #snap s
    ON s.period_id = p.period_id
   AND s.dt_rep = p.dt_to
   AND s.flow_category = g.flow_category

GROUP BY
    p.period_id,
    p.period_name,
    p.dt_to,
    g.flow_category

ORDER BY
    p.period_id,
    g.flow_category;


/* ============================================================
   РЕЗУЛЬТАТ 3. ПОКЛИЕНТНЫЙ КОНТРОЛЬ НА ВЫБРАННУЮ ДАТУ
   ============================================================ */

SELECT
    p.period_name,
    s.dt_rep,
    s.flow_category,
    s.cli_id,
    COUNT(DISTINCT s.con_id) AS contracts,
    CAST(
        SUM(s.volume) / 1000000.0
        AS decimal(18,4)
    ) AS balance_mn

FROM #snap s
INNER JOIN #periods p
    ON p.period_id = s.period_id

WHERE p.period_name = @ControlPeriod
  AND s.dt_rep = @ControlDate

GROUP BY
    p.period_name,
    s.dt_rep,
    s.flow_category,
    s.cli_id

ORDER BY
    s.flow_category,
    balance_mn DESC;


/* ============================================================
   РЕЗУЛЬТАТ 4. ПРОВЕРКА НОВ СРЕДИ ОСТАЛЬНЫХ КЛИЕНТОВ
   ============================================================ */

SELECT
    p.period_name,

    COUNT(DISTINCT t.cli_id) AS nov_clients,

    COUNT(DISTINCT t.con_id) AS nov_contracts,

    CAST(
        SUM(ISNULL(s.volume, 0)) / 1000000000.0
        AS decimal(18,6)
    ) AS nov_balance_bn

FROM #periods p

LEFT JOIN #tracked t
    ON t.period_id = p.period_id
   AND t.flow_category = N'НОВ остальных'

LEFT JOIN #snap s
    ON s.period_id = t.period_id
   AND s.con_id = t.con_id
   AND s.dt_rep = p.dt_to

GROUP BY
    p.period_id,
    p.period_name

ORDER BY p.period_id;


/* ============================================================
   5. ПОДНЕВНАЯ АГРЕГАЦИЯ
   ============================================================ */

SELECT
    period_id,
    dt_rep,

    SUM(CASE
        WHEN flow_category = N'Новые клиенты'
        THEN volume ELSE 0
    END) AS new_clients,

    SUM(CASE
        WHEN flow_category = N'НОВ остальных'
        THEN volume ELSE 0
    END) AS nov_others,

    SUM(CASE
        WHEN flow_category = N'НДП/НДМ остальных'
        THEN volume ELSE 0
    END) AS ndp_ndm_others

INTO #daily_agg

FROM #snap

GROUP BY
    period_id,
    dt_rep;

CREATE UNIQUE CLUSTERED INDEX IX_daily_agg
ON #daily_agg (period_id, dt_rep);


/* ============================================================
   6. КАЛЕНДАРЬ И ПОДНЕВНАЯ ДИНАМИКА
   ============================================================ */

;WITH calendar AS
(
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
        ISNULL(a.new_clients, 0) / 1000000000.0
        AS decimal(18,6)
    ) AS new_clients_bn,

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
    ) AS existing_clients_bn,

    CAST(
        (
            ISNULL(a.new_clients, 0)
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

OPTION (MAXRECURSION 0, RECOMPILE);


/* ============================================================
   РЕЗУЛЬТАТ 5. ПОЛНАЯ ПОДНЕВНАЯ ДИНАМИКА
   ============================================================ */

SELECT
    period_name,
    dt_rep,
    day_num,
    new_clients_bn,
    nov_others_bn,
    ndp_ndm_others_bn,
    existing_clients_bn,
    total_bn
FROM #daily
ORDER BY
    period_id,
    dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ 6. ИТОГОВАЯ ПАНЕЛЬ ДЛЯ EXCEL
   ============================================================ */

;WITH days AS
(
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
        DATEADD(day, d.day_num - 1, '2025-10-01'),
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
