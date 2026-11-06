
USE ALM;
SET NOCOUNT ON;

DROP TABLE IF EXISTS #periods;
DROP TABLE IF EXISTS #opened;
DROP TABLE IF EXISTS #new_clients;
DROP TABLE IF EXISTS #mass_clients;
DROP TABLE IF EXISTS #contracts;
DROP TABLE IF EXISTS #daily_delta;
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


/* 1. ДОГОВОРЫ, ОТКРЫТЫЕ В ПЕРИОД РК */

;WITH ranked AS (
    SELECT
        p.period_id,
        a.CLI_ID,
        TRY_CAST(a.CON_ID AS bigint) AS CON_ID,
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
    CLI_ID,
    CON_ID,
    PROD_NAME,
    dt_open
INTO #opened
FROM ranked
WHERE rn = 1
  AND CON_ID IS NOT NULL;

CREATE UNIQUE CLUSTERED INDEX IX_opened
ON #opened (period_id, CON_ID);

CREATE INDEX IX_opened_cli
ON #opened (period_id, CLI_ID);


/* 2. НОВЫЕ КЛИЕНТЫ ПО ИСТОРИИ ДОГОВОРОВ */

;WITH candidates AS (
    SELECT DISTINCT
        period_id,
        CLI_ID
    FROM #opened
)

SELECT
    c.period_id,
    c.CLI_ID
INTO #new_clients
FROM candidates c

INNER JOIN #periods p
    ON p.period_id = c.period_id

WHERE NOT EXISTS (
    SELECT 1
    FROM LIQUIDITY.liq.DepositContract_all h WITH (NOLOCK)

    WHERE h.CLI_ID = c.CLI_ID
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
ON #new_clients (period_id, CLI_ID);


/* 3. ОСТАВЛЯЕМ ТОЛЬКО НОВЫХ КЛИЕНТОВ MASS */

SELECT
    n.period_id,
    n.CLI_ID
INTO #mass_clients

FROM #new_clients n

INNER JOIN #periods p
    ON p.period_id = n.period_id

WHERE NOT EXISTS (
    SELECT 1

    FROM LIQUIDITY.liq.DepositContract_all h WITH (NOLOCK)

    WHERE h.CLI_ID = n.CLI_ID
      AND h.DT_OPEN < DATEADD(day, 1, p.dt_to)
      AND h.CON_ID IS NOT NULL

      AND EXISTS (
          SELECT 1
          FROM LIQUIDITY.liq.DepositInterestsRate r WITH (NOLOCK)

          WHERE r.CON_ID = h.CON_ID
            AND r.seg_name = N'ДЧБО'
      )
);

CREATE UNIQUE CLUSTERED INDEX IX_mass_clients
ON #mass_clients (period_id, CLI_ID);


/* 4. ДОГОВОРЫ НОВЫХ MASS-КЛИЕНТОВ */

SELECT
    o.period_id,
    o.CLI_ID,
    o.CON_ID,
    o.dt_open,

    CASE
        WHEN o.PROD_NAME LIKE N'Накопительный счёт%'
            THEN N'НС'
        ELSE N'Вклады'
    END AS product_type

INTO #contracts

FROM #opened o

INNER JOIN #mass_clients m
    ON m.period_id = o.period_id
   AND m.CLI_ID = o.CLI_ID;

CREATE UNIQUE CLUSTERED INDEX IX_contracts
ON #contracts (period_id, CON_ID);

CREATE INDEX IX_contracts_con
ON #contracts (CON_ID);


/* 5. ИСТОРИЯ САЛЬДО */

;WITH saldo_clipped AS (
    SELECT
        c.period_id,
        c.product_type,
        p.dt_to,

        CAST(s.OUT_RUB AS decimal(38,6)) AS out_rub,

        bounds.effective_from,
        bounds.effective_to

    FROM #contracts c

    INNER JOIN #periods p
        ON p.period_id = c.period_id

    INNER JOIN LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)
        ON s.CON_ID = c.CON_ID

    CROSS APPLY (
        SELECT
            (
                SELECT MAX(v.dt)
                FROM (VALUES
                    (CAST(s.DT_FROM AS date)),
                    (c.dt_open),
                    (p.dt_from)
                ) v(dt)
            ) AS effective_from,

            (
                SELECT MIN(v.dt)
                FROM (VALUES
                    (ISNULL(CAST(s.DT_TO AS date), p.dt_to)),
                    (p.dt_to)
                ) v(dt)
            ) AS effective_to
    ) bounds

    WHERE s.DT_FROM < DATEADD(day, 1, p.dt_to)
      AND (s.DT_TO IS NULL OR s.DT_TO >= p.dt_from)
      AND s.OUT_RUB IS NOT NULL
      AND bounds.effective_from <= bounds.effective_to
),

events AS (
    SELECT
        s.period_id,
        s.product_type,
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

    SUM(CASE
        WHEN product_type = N'Вклады'
            THEN delta
        ELSE 0
    END) AS delta_deposits,

    SUM(CASE
        WHEN product_type = N'НС'
            THEN delta
        ELSE 0
    END) AS delta_ns

INTO #daily_delta

FROM events
GROUP BY period_id, dt_rep;

CREATE UNIQUE CLUSTERED INDEX IX_daily_delta
ON #daily_delta (period_id, dt_rep);


/* 6. ПОДНЕВНЫЕ ОСТАТКИ НАРАСТАЮЩИМ ИТОГОМ */

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
    c.period_id,
    c.dt_rep,

    SUM(ISNULL(d.delta_deposits, 0)) OVER (
        PARTITION BY c.period_id
        ORDER BY c.dt_rep
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND CURRENT ROW
    ) AS balance_deposits,

    SUM(ISNULL(d.delta_ns, 0)) OVER (
        PARTITION BY c.period_id
        ORDER BY c.dt_rep
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND CURRENT ROW
    ) AS balance_ns

INTO #daily

FROM calendar c

LEFT JOIN #daily_delta d
    ON d.period_id = c.period_id
   AND d.dt_rep = c.dt_rep

OPTION (MAXRECURSION 0, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_daily
ON #daily (period_id, dt_rep);


/* РЕЗУЛЬТАТ 1. ОКТЯБРЬ 2026 */

SELECT
    dt_rep,
    CAST(balance_deposits / 1e9 AS decimal(18,6)) AS deposits_bn,
    CAST(balance_ns / 1e9 AS decimal(18,6)) AS ns_bn,
    CAST(
        (balance_deposits + balance_ns) / 1e9
        AS decimal(18,6)
    ) AS total_bn
FROM #daily
WHERE period_id = 1
ORDER BY dt_rep;


/* РЕЗУЛЬТАТ 2. МАРТ 2026 */

SELECT
    dt_rep,
    CAST(balance_deposits / 1e9 AS decimal(18,6)) AS deposits_bn,
    CAST(balance_ns / 1e9 AS decimal(18,6)) AS ns_bn,
    CAST(
        (balance_deposits + balance_ns) / 1e9
        AS decimal(18,6)
    ) AS total_bn
FROM #daily
WHERE period_id = 2
ORDER BY dt_rep;


/* РЕЗУЛЬТАТ 3. ОКТЯБРЬ 2025 */

SELECT
    dt_rep,
    CAST(balance_deposits / 1e9 AS decimal(18,6)) AS deposits_bn,
    CAST(balance_ns / 1e9 AS decimal(18,6)) AS ns_bn,
    CAST(
        (balance_deposits + balance_ns) / 1e9
        AS decimal(18,6)
    ) AS total_bn
FROM #daily
WHERE period_id = 3
ORDER BY dt_rep;
