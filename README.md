USE ALM;
SET NOCOUNT ON;

DROP TABLE IF EXISTS #periods;
DROP TABLE IF EXISTS #contracts;
DROP TABLE IF EXISTS #attr;
DROP TABLE IF EXISTS #selected;
DROP TABLE IF EXISTS #events;

CREATE TABLE #periods (
    period_name nvarchar(50),
    dt_from date,
    dt_to date
);

INSERT INTO #periods VALUES
    (N'Октябрь 2026', '2026-10-01', '2026-11-02'),
    (N'Март 2026',    '2026-03-01', '2026-04-07'),
    (N'Октябрь 2025', '2025-10-01', '2025-11-05');

;WITH ranked AS (
    SELECT
        p.period_name,
        p.dt_from,
        p.dt_to,
        TRY_CAST(a.CON_ID AS bigint) AS con_id,
        CAST(a.DT_OPEN AS date) AS dt_open,
        a.PROD_NAME,
        ROW_NUMBER() OVER (
            PARTITION BY p.period_name, a.CON_ID
            ORDER BY a.DT_OPEN DESC
        ) AS rn
    FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)
    INNER JOIN #periods p
        ON a.DT_OPEN >= p.dt_from
       AND a.DT_OPEN < DATEADD(day, 1, p.dt_to)
    WHERE a.cli_short_name = N'ФЛ'
      AND a.cur = 'RUR'
      AND a.seg_name = N'Розничный бизнес'
      AND a.CON_ID IS NOT NULL
      AND a.PROD_NAME NOT IN (
          N'Агентские эскроу ФЛ по ставке КС+спред',
          N'Спец. банк.счёт',
          N'Залоговый',
          N'Агентские эскроу ФЛ по ставке КС/2',
          N'Эскроу',
          N'Депозит ФЛ (суррогатный договор для счета Новой Афины)',
          N'Накопительный счёт',
          N'Накопительный счётУльтра',
          N'Надёжный прайм',
          N'Надёжный VIP',
          N'Надёжный премиум',
          N'Надёжный промо',
          N'Надёжный старт',
          N'Надёжный Т2',
          N'Надёжный Мегафон',
          N'Надёжный процент',
          N'Надёжныйпроцент',
          N'Могучий',
          N'Надёжный',
          N'ДОМа надёжно',
          N'Всё в ДОМ'
      )
)
SELECT
    period_name,
    dt_from,
    dt_to,
    con_id,
    dt_open,
    PROD_NAME
INTO #contracts
FROM ranked
WHERE rn = 1
  AND con_id IS NOT NULL;

CREATE UNIQUE CLUSTERED INDEX IX_contracts
ON #contracts (con_id);

;WITH ranked AS (
    SELECT
        TRY_CAST(a.CON_ID AS bigint) AS con_id,
        TRY_CAST(a.[Нов] AS int) AS nov,
        TRY_CAST(a.[НДП] AS int) AS ndp,
        TRY_CAST(a.[НДМ] AS int) AS ndm,
        ROW_NUMBER() OVER (
            PARTITION BY a.CON_ID
            ORDER BY a.DT_UPDATE DESC, a.loaddate DESC
        ) AS rn
    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)
    INNER JOIN #contracts c
        ON TRY_CAST(a.CON_ID AS bigint) = c.con_id
)
SELECT
    con_id,
    nov,
    ndp,
    ndm
INTO #attr
FROM ranked
WHERE rn = 1;

CREATE UNIQUE CLUSTERED INDEX IX_attr
ON #attr (con_id);

SELECT
    c.period_name,
    c.dt_from,
    c.dt_to,
    c.con_id,
    c.dt_open,
    CASE
        WHEN ISNULL(a.nov, 0) = 1
            THEN N'НОВ'
        WHEN ISNULL(a.ndp, 0) = 1
          OR ISNULL(a.ndm, 0) = 1
            THEN N'НДП / НДМ'
    END AS category
INTO #selected
FROM #contracts c
INNER JOIN #attr a
    ON a.con_id = c.con_id
WHERE ISNULL(a.nov, 0) = 1
   OR ISNULL(a.ndp, 0) = 1
   OR ISNULL(a.ndm, 0) = 1;

CREATE UNIQUE CLUSTERED INDEX IX_selected
ON #selected (con_id);

;WITH saldo_clipped AS (
    SELECT
        c.period_name,
        c.category,
        c.dt_to,
        CAST(s.OUT_RUB AS decimal(38,6)) AS out_rub,
        bounds.effective_from,
        bounds.effective_to
    FROM LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)
    INNER JOIN #selected c
        ON c.con_id = s.CON_ID
    CROSS APPLY (
        SELECT
            (
                SELECT MAX(v.dt)
                FROM (VALUES
                    (CAST(s.DT_FROM AS date)),
                    (c.dt_open),
                    (c.dt_from)
                ) v(dt)
            ) AS effective_from,
            (
                SELECT MIN(v.dt)
                FROM (VALUES
                    (ISNULL(CAST(s.DT_TO AS date), c.dt_to)),
                    (c.dt_to)
                ) v(dt)
            ) AS effective_to
    ) bounds
    WHERE s.DT_FROM < DATEADD(day, 1, c.dt_to)
      AND (s.DT_TO IS NULL OR s.DT_TO >= c.dt_from)
      AND s.OUT_RUB IS NOT NULL
      AND bounds.effective_from <= bounds.effective_to
),
saldo_events AS (
    SELECT
        s.period_name,
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
    period_name,
    category,
    dt_rep,
    SUM(delta) AS delta
INTO #events
FROM saldo_events
GROUP BY period_name, category, dt_rep;

;WITH calendar AS (
    SELECT
        period_name,
        dt_from AS dt_rep,
        dt_to
    FROM #periods

    UNION ALL

    SELECT
        period_name,
        DATEADD(day, 1, dt_rep),
        dt_to
    FROM calendar
    WHERE dt_rep < dt_to
),
categories AS (
    SELECT N'НОВ' AS category
    UNION ALL
    SELECT N'НДП / НДМ'
),
daily AS (
    SELECT
        c.period_name,
        c.dt_rep,
        k.category,
        ISNULL(e.delta, 0) AS delta
    FROM calendar c
    CROSS JOIN categories k
    LEFT JOIN #events e
        ON e.period_name = c.period_name
       AND e.dt_rep = c.dt_rep
       AND e.category = k.category
),
balances AS (
    SELECT
        period_name,
        dt_rep,
        category,
        delta,
        SUM(delta) OVER (
            PARTITION BY period_name, category
            ORDER BY dt_rep
            ROWS BETWEEN UNBOUNDED PRECEDING
                     AND CURRENT ROW
        ) AS balance
    FROM daily
)
SELECT
    period_name,
    dt_rep,
    category,
    CAST(balance / 1000000000.0 AS decimal(18,6)) AS balance_bn,
    CAST(delta / 1000000000.0 AS decimal(18,6)) AS daily_change_bn
FROM balances
ORDER BY
    CASE period_name
        WHEN N'Октябрь 2025' THEN 1
        WHEN N'Март 2026' THEN 2
        WHEN N'Октябрь 2026' THEN 3
    END,
    dt_rep,
    category
OPTION (MAXRECURSION 0);
