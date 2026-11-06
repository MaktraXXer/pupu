
USE ALM;
SET NOCOUNT ON;

DROP TABLE IF EXISTS #port_periods;
DROP TABLE IF EXISTS #port_contracts;
DROP TABLE IF EXISTS #port_clients;
DROP TABLE IF EXISTS #port_uchk;
DROP TABLE IF EXISTS #port_selected;
DROP TABLE IF EXISTS #port_events;
DROP TABLE IF EXISTS #port_daily;

CREATE TABLE #port_periods (
    period_id int PRIMARY KEY,
    period_name nvarchar(50),
    dt_from date,
    dt_to date
);

INSERT INTO #port_periods VALUES
    (1, N'Октябрь 2026', '2026-10-01', '2026-11-04'),
    (2, N'Март 2026',    '2026-03-01', '2026-04-07'),
    (3, N'Октябрь 2025', '2025-10-01', '2025-11-05');

DECLARE @MaxTo date = (
    SELECT MAX(dt_to) FROM #port_periods
);

/* ============================================================
   1. РЕЕСТР ВСЕХ РУБЛЕВЫХ ДОГОВОРОВ ФЛ
   ============================================================ */

SELECT
    a.CON_ID,
    MIN(a.CLI_ID) AS CLI_ID,
    MIN(CAST(a.DT_OPEN AS date)) AS dt_open,

    MAX(CASE
        WHEN a.PROD_NAME LIKE N'Накопительный счёт%'
            THEN 1
        ELSE 0
    END) AS is_ns,

    MAX(CASE
        WHEN a.PROD_NAME IN (
            N'ДОМа надёжно',
            N'Всё в ДОМ',
            N'Надёжный',
            N'Надёжный VIP',
            N'НадёжныйVIP',
            N'Надёжный премиум',
            N'Надёжный промо',
            N'Надёжныйпромо',
            N'Надёжный старт',
            N'Надёжный Т2',
            N'Надёжный T2',
            N'Надёжный Мегафон',
            N'Надёжный прайм',
            N'Надёжныйпрайм',
            N'Надёжный процент',
            N'Надёжныйпроцент',
            N'Могучий'
        ) THEN 1
        ELSE 0
    END) AS is_mp

INTO #port_contracts

FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)

WHERE a.DT_OPEN IS NOT NULL
  AND a.DT_OPEN < DATEADD(day, 1, @MaxTo)
  AND a.CON_ID IS NOT NULL
  AND a.CLI_ID IS NOT NULL
  AND a.cli_short_name = N'ФЛ'
  AND a.cur = 'RUR'

  AND a.PROD_NAME NOT IN (
      N'Агентские эскроу ФЛ по ставке КС+спред',
      N'Спец. банк.счёт',
      N'Залоговый',
      N'Агентские эскроу ФЛ по ставке КС/2',
      N'Эскроу',
      N'Депозит ФЛ (суррогатный договор для счета Новой Афины)'
  )

GROUP BY a.CON_ID;

CREATE UNIQUE CLUSTERED INDEX IX_port_contracts
ON #port_contracts (CON_ID);

CREATE INDEX IX_port_contracts_cli
ON #port_contracts (CLI_ID);


/* ============================================================
   2. КЛИЕНТЫ ИЗ НУЖНОГО ПОРТФЕЛЯ
   ============================================================ */

SELECT DISTINCT
    CLI_ID
INTO #port_clients
FROM #port_contracts
WHERE is_mp = 0;

CREATE UNIQUE CLUSTERED INDEX IX_port_clients
ON #port_clients (CLI_ID);


/* ============================================================
   3. ИСТОРИЧЕСКОЕ ОПРЕДЕЛЕНИЕ УЧК

   Если хотя бы один вклад клиента имел seg_name = ДЧБО,
   определяем дату открытия первого такого договора.
   ============================================================ */

SELECT
    h.CLI_ID,
    MIN(CAST(h.DT_OPEN AS date)) AS first_uchk_date

INTO #port_uchk

FROM LIQUIDITY.liq.DepositContract_all h WITH (NOLOCK)

INNER JOIN #port_clients c
    ON c.CLI_ID = h.CLI_ID

WHERE h.CON_ID IS NOT NULL
  AND h.DT_OPEN IS NOT NULL
  AND h.DT_OPEN < DATEADD(day, 1, @MaxTo)

  AND EXISTS (
      SELECT 1
      FROM LIQUIDITY.liq.DepositInterestsRate r WITH (NOLOCK)
      WHERE r.CON_ID = h.CON_ID
        AND r.seg_name = N'ДЧБО'
  )

GROUP BY h.CLI_ID;

CREATE UNIQUE CLUSTERED INDEX IX_port_uchk
ON #port_uchk (CLI_ID);


/* ============================================================
   4. КЛАССИФИКАЦИЯ ДОГОВОРОВ ПО КАЖДОЙ РК
   ============================================================ */

SELECT
    p.period_id,
    c.CON_ID,
    c.CLI_ID,
    c.dt_open,

    CASE
        WHEN c.is_ns = 1
         AND u.first_uchk_date <= p.dt_to
            THEN N'НС УЧК'

        WHEN c.is_ns = 1
            THEN N'НС Розница'

        WHEN u.first_uchk_date <= p.dt_to
            THEN N'Стандартная линейка УЧК'

        ELSE N'Стандартная линейка Розница'
    END AS category

INTO #port_selected

FROM #port_contracts c

INNER JOIN #port_periods p
    ON c.dt_open <= p.dt_to

LEFT JOIN #port_uchk u
    ON u.CLI_ID = c.CLI_ID

WHERE c.is_mp = 0;

CREATE CLUSTERED INDEX IX_port_selected
ON #port_selected (CON_ID, period_id);


/* ============================================================
   5. ИСТОРИЯ САЛЬДО

   Берем период с предыдущего дня до конца каждой РК.
   Каждый интервал превращаем в два события.
   ============================================================ */

;WITH saldo_clipped AS (
    SELECT
        c.period_id,
        c.category,
        p.dt_to,

        CAST(s.OUT_RUB AS decimal(28,6)) AS out_rub,

        bounds.effective_from,
        bounds.effective_to

    FROM #port_selected c

    INNER JOIN #port_periods p
        ON p.period_id = c.period_id

    INNER JOIN LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)
        ON s.CON_ID = c.CON_ID
       AND s.DT_FROM < DATEADD(day, 1, p.dt_to)
       AND (
           s.DT_TO IS NULL
           OR s.DT_TO >= DATEADD(day, -1, p.dt_from)
       )

    CROSS APPLY (
        SELECT
            CASE
                WHEN CAST(s.DT_FROM AS date) > c.dt_open
                    THEN CAST(s.DT_FROM AS date)
                ELSE c.dt_open
            END AS start_date
    ) f

    CROSS APPLY (
        SELECT
            CASE
                WHEN f.start_date > DATEADD(day, -1, p.dt_from)
                    THEN f.start_date
                ELSE DATEADD(day, -1, p.dt_from)
            END AS effective_from,

            CASE
                WHEN s.DT_TO IS NULL
                  OR CAST(s.DT_TO AS date) > p.dt_to
                    THEN p.dt_to
                ELSE CAST(s.DT_TO AS date)
            END AS effective_to
    ) bounds

    WHERE s.OUT_RUB IS NOT NULL
      AND bounds.effective_from <= bounds.effective_to
),

events AS (
    SELECT
        s.period_id,
        s.category,
        e.dt_rep,
        e.delta

    FROM saldo_clipped s

    CROSS APPLY (VALUES
        (s.effective_from, s.out_rub),

        (
            CASE
                WHEN s.effective_to < s.dt_to
                    THEN DATEADD(day, 1, s.effective_to)
            END,
            -s.out_rub
        )
    ) e(dt_rep, delta)

    WHERE e.dt_rep IS NOT NULL
)

SELECT
    period_id,
    dt_rep,

    SUM(CASE WHEN category = N'НС Розница'
        THEN delta ELSE 0 END) AS delta_ns_retail,

    SUM(CASE WHEN category = N'Стандартная линейка Розница'
        THEN delta ELSE 0 END) AS delta_dep_retail,

    SUM(CASE WHEN category = N'НС УЧК'
        THEN delta ELSE 0 END) AS delta_ns_uchk,

    SUM(CASE WHEN category = N'Стандартная линейка УЧК'
        THEN delta ELSE 0 END) AS delta_dep_uchk

INTO #port_events

FROM events
GROUP BY period_id, dt_rep;

CREATE UNIQUE CLUSTERED INDEX IX_port_events
ON #port_events (period_id, dt_rep);


/* ============================================================
   6. ЕЖЕДНЕВНЫЕ ОСТАТКИ И НАКОПЛЕННЫЙ ПРИРОСТ
   ============================================================ */

;WITH calendar AS (
    SELECT
        period_id,
        DATEADD(day, -1, dt_from) AS dt_rep,
        dt_to
    FROM #port_periods

    UNION ALL

    SELECT
        period_id,
        DATEADD(day, 1, dt_rep),
        dt_to
    FROM calendar
    WHERE dt_rep < dt_to
),

daily_delta AS (
    SELECT
        c.period_id,
        c.dt_rep,

        ISNULL(e.delta_ns_retail, 0) AS delta_ns_retail,
        ISNULL(e.delta_dep_retail, 0) AS delta_dep_retail,
        ISNULL(e.delta_ns_uchk, 0) AS delta_ns_uchk,
        ISNULL(e.delta_dep_uchk, 0) AS delta_dep_uchk

    FROM calendar c

    LEFT JOIN #port_events e
        ON e.period_id = c.period_id
       AND e.dt_rep = c.dt_rep
),

balances AS (
    SELECT
        period_id,
        dt_rep,

        SUM(delta_ns_retail) OVER (
            PARTITION BY period_id
            ORDER BY dt_rep
            ROWS UNBOUNDED PRECEDING
        ) AS balance_ns_retail,

        SUM(delta_dep_retail) OVER (
            PARTITION BY period_id
            ORDER BY dt_rep
            ROWS UNBOUNDED PRECEDING
        ) AS balance_dep_retail,

        SUM(delta_ns_uchk) OVER (
            PARTITION BY period_id
            ORDER BY dt_rep
            ROWS UNBOUNDED PRECEDING
        ) AS balance_ns_uchk,

        SUM(delta_dep_uchk) OVER (
            PARTITION BY period_id
            ORDER BY dt_rep
            ROWS UNBOUNDED PRECEDING
        ) AS balance_dep_uchk

    FROM daily_delta
)

SELECT
    b.period_id,
    b.dt_rep,

    b.balance_ns_retail,
    b.balance_dep_retail,
    b.balance_ns_uchk,
    b.balance_dep_uchk,

    b.balance_ns_retail + b.balance_dep_uchk
        AS balance_selected,

    b.balance_ns_retail + b.balance_dep_uchk
    - FIRST_VALUE(
        b.balance_ns_retail + b.balance_dep_uchk
      ) OVER (
        PARTITION BY b.period_id
        ORDER BY b.dt_rep
      ) AS growth_selected

INTO #port_daily
FROM balances b
OPTION (MAXRECURSION 0, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_port_daily
ON #port_daily (period_id, dt_rep);


/* ============================================================
   РЕЗУЛЬТАТ 1. ОКТЯБРЬ 2026
   ============================================================ */

SELECT
    d.dt_rep,

    CAST(d.balance_ns_retail / 1e9
        AS decimal(18,6)) AS ns_retail_bn,

    CAST(d.balance_dep_uchk / 1e9
        AS decimal(18,6)) AS deposits_uchk_bn,

    CAST(d.balance_selected / 1e9
        AS decimal(18,6)) AS balance_bn,

    CAST(d.growth_selected / 1e9
        AS decimal(18,6)) AS growth_bn

FROM #port_daily d
INNER JOIN #port_periods p
    ON p.period_id = d.period_id

WHERE d.period_id = 1
  AND d.dt_rep >= p.dt_from

ORDER BY d.dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ 2. МАРТ 2026
   ============================================================ */

SELECT
    d.dt_rep,

    CAST(d.balance_ns_retail / 1e9
        AS decimal(18,6)) AS ns_retail_bn,

    CAST(d.balance_dep_uchk / 1e9
        AS decimal(18,6)) AS deposits_uchk_bn,

    CAST(d.balance_selected / 1e9
        AS decimal(18,6)) AS balance_bn,

    CAST(d.growth_selected / 1e9
        AS decimal(18,6)) AS growth_bn

FROM #port_daily d
INNER JOIN #port_periods p
    ON p.period_id = d.period_id

WHERE d.period_id = 2
  AND d.dt_rep >= p.dt_from

ORDER BY d.dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ 3. ОКТЯБРЬ 2025
   ============================================================ */

SELECT
    d.dt_rep,

    CAST(d.balance_ns_retail / 1e9
        AS decimal(18,6)) AS ns_retail_bn,

    CAST(d.balance_dep_uchk / 1e9
        AS decimal(18,6)) AS deposits_uchk_bn,

    CAST(d.balance_selected / 1e9
        AS decimal(18,6)) AS balance_bn,

    CAST(d.growth_selected / 1e9
        AS decimal(18,6)) AS growth_bn

FROM #port_daily d
INNER JOIN #port_periods p
    ON p.period_id = d.period_id

WHERE d.period_id = 3
  AND d.dt_rep >= p.dt_from

ORDER BY d.dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ 4. ПАНЕЛЬ НА 17 ДАТ
   НАКОПЛЕННЫЙ ПРИРОСТ, МЛРД РУБ.
   ============================================================ */

;WITH days AS (
    SELECT day_num
    FROM (VALUES
        (1), (4), (7), (10), (13),
        (16), (19), (22), (25), (28),
        (30), (32), (33), (34), (35),
        (36), (38)
    ) v(day_num)
),

panel AS (
    SELECT
        d.period_id,
        DATEDIFF(day, p.dt_from, d.dt_rep) + 1 AS day_num,

        CAST(d.growth_selected / 1000000000.0
            AS decimal(18,6)) AS growth_bn

    FROM #port_daily d

    INNER JOIN #port_periods p
        ON p.period_id = d.period_id

    WHERE d.dt_rep >= p.dt_from
)

SELECT
    CONVERT(
        varchar(10),
        DATEADD(day, x.day_num - 1, DATEFROMPARTS(2025,10,1)),
        104
    ) AS [Column1],

    MAX(CASE WHEN p.period_id = 1
        THEN p.growth_bn END) AS [РК октябрь 26],

    MAX(CASE WHEN p.period_id = 2
        THEN p.growth_bn END) AS [РК март 26],

    MAX(CASE WHEN p.period_id = 3
        THEN p.growth_bn END) AS [РК октябрь 25]

FROM days x

LEFT JOIN panel p
    ON p.day_num = x.day_num

GROUP BY x.day_num
ORDER BY x.day_num;
