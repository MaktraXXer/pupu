
USE ALM;
SET NOCOUNT ON;

DROP TABLE IF EXISTS #pr_periods;
DROP TABLE IF EXISTS #pr_opened;
DROP TABLE IF EXISTS #pr_new_clients;
DROP TABLE IF EXISTS #private_clients;
DROP TABLE IF EXISTS #pr_candidates;
DROP TABLE IF EXISTS #pr_flagged;
DROP TABLE IF EXISTS #pr_flagged_clients;
DROP TABLE IF EXISTS #pr_other_clients;
DROP TABLE IF EXISTS #pr_flows;
DROP TABLE IF EXISTS #pr_openings;
DROP TABLE IF EXISTS #pr_events;
DROP TABLE IF EXISTS #pr_daily;


/* ============================================================
   1. ПЕРИОДЫ ТРЕХ РК
   ============================================================ */

CREATE TABLE #pr_periods (
    period_id int PRIMARY KEY,
    period_name nvarchar(50),
    dt_from date,
    dt_to date
);

INSERT INTO #pr_periods VALUES
    (1, N'Октябрь 2026',
        DATEFROMPARTS(2026,10,1),
        DATEFROMPARTS(2026,11,4)),

    (2, N'Март 2026',
        DATEFROMPARTS(2026,3,1),
        DATEFROMPARTS(2026,4,7)),

    (3, N'Октябрь 2025',
        DATEFROMPARTS(2025,10,1),
        DATEFROMPARTS(2025,11,5));


/* ============================================================
   2. ДОГОВОРЫ, ОТКРЫТЫЕ В ПЕРИОД РК
   ============================================================ */

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

    INNER JOIN #pr_periods p
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

INTO #pr_opened

FROM ranked

WHERE rn = 1
  AND CON_ID IS NOT NULL;

CREATE UNIQUE CLUSTERED INDEX IX_pr_opened
ON #pr_opened (period_id, CON_ID);

CREATE INDEX IX_pr_opened_cli
ON #pr_opened (period_id, CLI_ID);


/* ============================================================
   3. НОВЫЕ КЛИЕНТЫ ПО ИСТОРИИ ДОГОВОРОВ
   ============================================================ */

;WITH candidates AS (
    SELECT DISTINCT
        period_id,
        CLI_ID
    FROM #pr_opened
)

SELECT
    c.period_id,
    c.CLI_ID

INTO #pr_new_clients

FROM candidates c

INNER JOIN #pr_periods p
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

CREATE UNIQUE CLUSTERED INDEX IX_pr_new_clients
ON #pr_new_clients (period_id, CLI_ID);


/* ============================================================
   4. НОВЫЕ КЛИЕНТЫ PRIVATE

   Хотя бы один договор с историческим ДЧБО
   до окончания соответствующей РК.
   ============================================================ */

SELECT
    n.period_id,
    n.CLI_ID

INTO #private_clients

FROM #pr_new_clients n

INNER JOIN #pr_periods p
    ON p.period_id = n.period_id

WHERE EXISTS (
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

CREATE UNIQUE CLUSTERED INDEX IX_private_clients
ON #private_clients (period_id, CLI_ID);


/* ============================================================
   5. ДОГОВОРЫ ОСТАЛЬНЫХ КЛИЕНТОВ
   ============================================================ */

SELECT
    o.period_id,
    o.CLI_ID,
    o.CON_ID,
    o.dt_open

INTO #pr_candidates

FROM #pr_opened o

WHERE o.PROD_NAME NOT LIKE N'Накопительный счёт%'

  AND NOT EXISTS (
      SELECT 1
      FROM #private_clients p
      WHERE p.period_id = o.period_id
        AND p.CLI_ID = o.CLI_ID
  );

CREATE UNIQUE CLUSTERED INDEX IX_pr_candidates
ON #pr_candidates (period_id, CON_ID);


/* ============================================================
   6. ОТБОР НОВ / НДП / НДМ
   ============================================================ */

SELECT
    c.period_id,
    c.CLI_ID,
    c.CON_ID,
    c.dt_open,

    CASE
        WHEN ISNULL(TRY_CAST(a.[Нов] AS int), 0) = 1
            THEN N'НОВ'
        ELSE N'НДП / НДМ'
    END AS category

INTO #pr_flagged

FROM #pr_candidates c

CROSS APPLY (
    SELECT TOP (1)
        a.[Нов],
        a.[НДП],
        a.[НДМ]

    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)

    WHERE a.CON_ID = CONVERT(varchar(30), c.CON_ID)

    ORDER BY
        a.DT_UPDATE DESC,
        a.loaddate DESC
) a

WHERE ISNULL(TRY_CAST(a.[Нов] AS int), 0) = 1
   OR ISNULL(TRY_CAST(a.[НДП] AS int), 0) = 1
   OR ISNULL(TRY_CAST(a.[НДМ] AS int), 0) = 1;

CREATE UNIQUE CLUSTERED INDEX IX_pr_flagged
ON #pr_flagged (period_id, CON_ID);


/* ============================================================
   7. УНИКАЛЬНЫЕ КЛИЕНТЫ С НАДБАВКАМИ
   ============================================================ */

SELECT DISTINCT
    period_id,
    CLI_ID

INTO #pr_flagged_clients

FROM #pr_flagged;

CREATE UNIQUE CLUSTERED INDEX IX_pr_flagged_clients
ON #pr_flagged_clients (period_id, CLI_ID);


/* ============================================================
   8. ОСТАЛЬНЫЕ КЛИЕНТЫ PRIVATE

   Исторический ДЧБО проверяем только для клиентов
   с нужными надбавками.
   ============================================================ */

SELECT
    c.period_id,
    c.CLI_ID

INTO #pr_other_clients

FROM #pr_flagged_clients c

INNER JOIN #pr_periods p
    ON p.period_id = c.period_id

WHERE EXISTS (
    SELECT 1

    FROM LIQUIDITY.liq.DepositContract_all h WITH (NOLOCK)

    WHERE h.CLI_ID = c.CLI_ID
      AND h.DT_OPEN < DATEADD(day, 1, p.dt_to)
      AND h.CON_ID IS NOT NULL

      AND EXISTS (
          SELECT 1

          FROM LIQUIDITY.liq.DepositInterestsRate r WITH (NOLOCK)

          WHERE r.CON_ID = h.CON_ID
            AND r.seg_name = N'ДЧБО'
      )
);

CREATE UNIQUE CLUSTERED INDEX IX_pr_other_clients
ON #pr_other_clients (period_id, CLI_ID);


/* ============================================================
   9. ОБЪЕДИНЯЕМ ДВА НЕПЕРЕСЕКАЮЩИХСЯ ПОРТФЕЛЯ

   Новые Private: все открытые вклады и НС.
   Остальные Private: только НОВ / НДП / НДМ.
   ============================================================ */

SELECT
    x.period_id,
    x.CLI_ID,
    x.CON_ID,
    x.dt_open,
    x.flow_type

INTO #pr_flows

FROM (
    SELECT
        o.period_id,
        o.CLI_ID,
        o.CON_ID,
        o.dt_open,

        CASE
            WHEN o.PROD_NAME LIKE N'Накопительный счёт%'
                THEN N'Новые НС'
            ELSE N'Новые вклады'
        END AS flow_type

    FROM #pr_opened o

    INNER JOIN #private_clients n
        ON n.period_id = o.period_id
       AND n.CLI_ID = o.CLI_ID

    UNION ALL

    SELECT
        f.period_id,
        f.CLI_ID,
        f.CON_ID,
        f.dt_open,
        f.category AS flow_type

    FROM #pr_flagged f

    INNER JOIN #pr_other_clients c
        ON c.period_id = f.period_id
       AND c.CLI_ID = f.CLI_ID
) x;

CREATE UNIQUE CLUSTERED INDEX IX_pr_flows
ON #pr_flows (period_id, CON_ID);

CREATE INDEX IX_pr_flows_con
ON #pr_flows (CON_ID);


/* ============================================================
   10. КОЛИЧЕСТВО ОТКРЫТЫХ ВКЛАДОВ ОСТАЛЬШИХСЯ
   ============================================================ */

SELECT
    period_id,
    dt_open AS dt_rep,

    SUM(CASE
        WHEN flow_type = N'НОВ'
            THEN CAST(1 AS bigint)
        ELSE 0
    END) AS nov_count,

    SUM(CASE
        WHEN flow_type = N'НДП / НДМ'
            THEN CAST(1 AS bigint)
        ELSE 0
    END) AS ndp_ndm_count

INTO #pr_openings

FROM #pr_flows

WHERE flow_type IN (
    N'НОВ',
    N'НДП / НДМ'
)

GROUP BY
    period_id,
    dt_open;

CREATE UNIQUE CLUSTERED INDEX IX_pr_openings
ON #pr_openings (period_id, dt_rep);


/* ============================================================
   11. ИСТОРИЯ САЛЬДО

   Общий расчёт для обеих групп.
   Одна запись сальдо = максимум два события.
   ============================================================ */

;WITH saldo_clipped AS (
    SELECT
        c.period_id,
        c.flow_type,
        p.dt_to,

        CAST(s.OUT_RUB AS decimal(38,6)) AS out_rub,

        bounds.effective_from,
        bounds.effective_to

    FROM #pr_flows c

    INNER JOIN #pr_periods p
        ON p.period_id = c.period_id

    INNER JOIN LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)
        ON s.CON_ID = c.CON_ID

    CROSS APPLY (
        SELECT
            CASE
                WHEN CAST(s.DT_FROM AS date) > c.dt_open
                    THEN CAST(s.DT_FROM AS date)
                ELSE c.dt_open
            END AS effective_from,

            CASE
                WHEN s.DT_TO IS NULL
                  OR CAST(s.DT_TO AS date) > p.dt_to
                    THEN p.dt_to
                ELSE CAST(s.DT_TO AS date)
            END AS effective_to
    ) bounds

    WHERE s.DT_FROM < DATEADD(day, 1, p.dt_to)
      AND (s.DT_TO IS NULL OR s.DT_TO >= c.dt_open)
      AND s.OUT_RUB IS NOT NULL
      AND bounds.effective_from <= bounds.effective_to
),

saldo_events AS (
    SELECT
        s.period_id,
        s.flow_type,
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
        WHEN flow_type = N'Новые вклады'
            THEN delta
        ELSE 0
    END) AS delta_new_deposits,

    SUM(CASE
        WHEN flow_type = N'Новые НС'
            THEN delta
        ELSE 0
    END) AS delta_new_ns,

    SUM(CASE
        WHEN flow_type = N'НОВ'
            THEN delta
        ELSE 0
    END) AS delta_nov,

    SUM(CASE
        WHEN flow_type = N'НДП / НДМ'
            THEN delta
        ELSE 0
    END) AS delta_ndp_ndm

INTO #pr_events

FROM saldo_events

GROUP BY
    period_id,
    dt_rep;

CREATE UNIQUE CLUSTERED INDEX IX_pr_events
ON #pr_events (period_id, dt_rep);


/* ============================================================
   12. ПОДНЕВНЫЕ ОСТАТКИ НАРАСТАЮЩИМ ИТОГОМ
   ============================================================ */

;WITH calendar AS (
    SELECT
        period_id,
        dt_from AS dt_rep,
        dt_to
    FROM #pr_periods

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

    SUM(ISNULL(e.delta_new_deposits, 0)) OVER (
        PARTITION BY c.period_id
        ORDER BY c.dt_rep
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS balance_deposits,

    SUM(ISNULL(e.delta_new_ns, 0)) OVER (
        PARTITION BY c.period_id
        ORDER BY c.dt_rep
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS balance_ns,

    SUM(ISNULL(e.delta_nov, 0)) OVER (
        PARTITION BY c.period_id
        ORDER BY c.dt_rep
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS nov_balance_rub,

    SUM(ISNULL(e.delta_ndp_ndm, 0)) OVER (
        PARTITION BY c.period_id
        ORDER BY c.dt_rep
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS ndp_ndm_balance_rub,

    SUM(ISNULL(o.nov_count, 0)) OVER (
        PARTITION BY c.period_id
        ORDER BY c.dt_rep
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS nov_opened_cum,

    SUM(ISNULL(o.ndp_ndm_count, 0)) OVER (
        PARTITION BY c.period_id
        ORDER BY c.dt_rep
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS ndp_ndm_opened_cum

INTO #pr_daily

FROM calendar c

INNER JOIN #pr_periods p
    ON p.period_id = c.period_id

LEFT JOIN #pr_events e
    ON e.period_id = c.period_id
   AND e.dt_rep = c.dt_rep

LEFT JOIN #pr_openings o
    ON o.period_id = c.period_id
   AND o.dt_rep = c.dt_rep

OPTION (MAXRECURSION 0, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_pr_daily
ON #pr_daily (period_id, dt_rep);


/* ============================================================
   ЧАСТЬ 1. НОВЫЕ КЛИЕНТЫ PRIVATE

   РЕЗУЛЬТАТ 1. ОКТЯБРЬ 2026
   ============================================================ */

SELECT
    dt_rep,

    CAST(balance_deposits / 1e9
        AS decimal(18,6)) AS deposits_bn,

    CAST(balance_ns / 1e9
        AS decimal(18,6)) AS ns_bn,

    CAST((balance_deposits + balance_ns) / 1e9
        AS decimal(18,6)) AS total_bn

FROM #pr_daily
WHERE period_id = 1
ORDER BY dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ 2. МАРТ 2026
   ============================================================ */

SELECT
    dt_rep,

    CAST(balance_deposits / 1e9
        AS decimal(18,6)) AS deposits_bn,

    CAST(balance_ns / 1e9
        AS decimal(18,6)) AS ns_bn,

    CAST((balance_deposits + balance_ns) / 1e9
        AS decimal(18,6)) AS total_bn

FROM #pr_daily
WHERE period_id = 2
ORDER BY dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ 3. ОКТЯБРЬ 2025
   ============================================================ */

SELECT
    dt_rep,

    CAST(balance_deposits / 1e9
        AS decimal(18,6)) AS deposits_bn,

    CAST(balance_ns / 1e9
        AS decimal(18,6)) AS ns_bn,

    CAST((balance_deposits + balance_ns) / 1e9
        AS decimal(18,6)) AS total_bn

FROM #pr_daily
WHERE period_id = 3
ORDER BY dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ 4. ПАНЕЛЬ НОВЫХ КЛИЕНТОВ PRIVATE
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

balances AS (
    SELECT
        period_id,
        day_num,

        CAST(
            (balance_deposits + balance_ns) / 1000000000.0
            AS decimal(18,6)
        ) AS total_bn

    FROM #pr_daily
)

SELECT
    CONVERT(
        varchar(10),
        DATEADD(
            day,
            d.day_num - 1,
            DATEFROMPARTS(2025,10,1)
        ),
        104
    ) AS [Column1],

    MAX(CASE WHEN b.period_id = 1
        THEN b.total_bn END) AS [РК октябрь 26],

    MAX(CASE WHEN b.period_id = 2
        THEN b.total_bn END) AS [РК март 26],

    MAX(CASE WHEN b.period_id = 3
        THEN b.total_bn END) AS [РК октябрь 25]

FROM days d

LEFT JOIN balances b
    ON b.day_num = d.day_num

GROUP BY d.day_num
ORDER BY d.day_num;


/* ============================================================
   ЧАСТЬ 2. НОВ / НДП / НДМ ОСТАЛЬНЫХ КЛИЕНТОВ PRIVATE

   РЕЗУЛЬТАТ 1. ОКТЯБРЬ 2026
   ============================================================ */

SELECT
    dt_rep,

    nov_opened_cum AS nov_contracts,
    ndp_ndm_opened_cum AS ndp_ndm_contracts,

    nov_opened_cum + ndp_ndm_opened_cum AS total_contracts,

    CAST(nov_balance_rub / 1e9
        AS decimal(18,6)) AS nov_bn,

    CAST(ndp_ndm_balance_rub / 1e9
        AS decimal(18,6)) AS ndp_ndm_bn,

    CAST(
        (nov_balance_rub + ndp_ndm_balance_rub) / 1e9
        AS decimal(18,6)
    ) AS total_bn

FROM #pr_daily
WHERE period_id = 1
ORDER BY dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ 2. МАРТ 2026
   ============================================================ */

SELECT
    dt_rep,

    nov_opened_cum AS nov_contracts,
    ndp_ndm_opened_cum AS ndp_ndm_contracts,

    nov_opened_cum + ndp_ndm_opened_cum AS total_contracts,

    CAST(nov_balance_rub / 1e9
        AS decimal(18,6)) AS nov_bn,

    CAST(ndp_ndm_balance_rub / 1e9
        AS decimal(18,6)) AS ndp_ndm_bn,

    CAST(
        (nov_balance_rub + ndp_ndm_balance_rub) / 1e9
        AS decimal(18,6)
    ) AS total_bn

FROM #pr_daily
WHERE period_id = 2
ORDER BY dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ 3. ОКТЯБРЬ 2025
   ============================================================ */

SELECT
    dt_rep,

    nov_opened_cum AS nov_contracts,
    ndp_ndm_opened_cum AS ndp_ndm_contracts,

    nov_opened_cum + ndp_ndm_opened_cum AS total_contracts,

    CAST(nov_balance_rub / 1e9
        AS decimal(18,6)) AS nov_bn,

    CAST(ndp_ndm_balance_rub / 1e9
        AS decimal(18,6)) AS ndp_ndm_bn,

    CAST(
        (nov_balance_rub + ndp_ndm_balance_rub) / 1e9
        AS decimal(18,6)
    ) AS total_bn

FROM #pr_daily
WHERE period_id = 3
ORDER BY dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ 4. ПАНЕЛЬ НОВ / НДП / НДМ PRIVATE
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

balances AS (
    SELECT
        period_id,
        day_num,

        CAST(
            (nov_balance_rub + ndp_ndm_balance_rub)
            / 1000000000.0
            AS decimal(18,6)
        ) AS total_bn

    FROM #pr_daily
)

SELECT
    CONVERT(
        varchar(10),
        DATEADD(
            day,
            d.day_num - 1,
            DATEFROMPARTS(2025,10,1)
        ),
        104
    ) AS [Column1],

    MAX(CASE WHEN b.period_id = 1
        THEN b.total_bn END) AS [РК октябрь 26],

    MAX(CASE WHEN b.period_id = 2
        THEN b.total_bn END) AS [РК март 26],

    MAX(CASE WHEN b.period_id = 3
        THEN b.total_bn END) AS [РК октябрь 25]

FROM days d

LEFT JOIN balances b
    ON b.day_num = d.day_num

GROUP BY d.day_num
ORDER BY d.day_num;


/* ============================================================
   РЕЗУЛЬТАТ 5. КОНТРОЛЬ НОВ В РУБЛЯХ
   ============================================================ */

;WITH control AS (
    SELECT
        p.period_id,
        p.period_name,
        p.dt_to,

        ISNULL(d.nov_opened_cum, 0) AS nov_contracts,
        ISNULL(d.nov_balance_rub, 0) AS nov_balance_rub

    FROM #pr_periods p

    LEFT JOIN #pr_daily d
        ON d.period_id = p.period_id
       AND d.dt_rep = p.dt_to
),

result AS (
    SELECT
        period_id AS sort_order,
        period_name AS [РК],
        dt_to AS [Дата контроля],
        nov_contracts AS [Открыто НОВ договоров],

        CAST(nov_balance_rub AS decimal(28,2))
            AS [Остаток НОВ, руб]

    FROM control

    UNION ALL

    SELECT
        4,
        N'ИТОГО',
        CAST(NULL AS date),
        SUM(nov_contracts),
        CAST(SUM(nov_balance_rub) AS decimal(28,2))

    FROM control
)

SELECT
    [РК],
    [Дата контроля],
    [Открыто НОВ договоров],
    [Остаток НОВ, руб]

FROM result

ORDER BY sort_order;
