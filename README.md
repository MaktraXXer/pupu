
USE ALM;
SET NOCOUNT ON;

DROP TABLE IF EXISTS #nm_candidates;
DROP TABLE IF EXISTS #nm_candidate_clients;
DROP TABLE IF EXISTS #nm_mass_clients;
DROP TABLE IF EXISTS #nm_selected;
DROP TABLE IF EXISTS #nm_openings;
DROP TABLE IF EXISTS #nm_events;
DROP TABLE IF EXISTS #nm_daily;

/* ============================================================
   1. ДОГОВОРЫ, ОТКРЫТЫЕ В ТРЕХ РК
   ============================================================ */

SELECT
    o.period_id,
    o.CLI_ID,
    o.CON_ID,
    o.dt_open
INTO #nm_candidates
FROM #opened o
WHERE o.PROD_NAME NOT LIKE N'Накопительный счёт%'
  AND o.PROD_NAME NOT IN (
      N'Всё в ДОМ',
      N'Надёжныйпроцент',
      N'Надёжныйпромо',
      N'Надёжныйпрайм',
      N'Классический',
      N'Привилегия',
      N'Достояние',
      N'Достояние без опций',
      N'Достояние с пополнением и снятием',
      N'Достояние с пополнением',
      N'Достояние (перем.ст.) без опций',
      N'Достояние (перем.ст.) с пополнением и снятием'
  );

CREATE UNIQUE CLUSTERED INDEX IX_nm_candidates
ON #nm_candidates (period_id, CON_ID);

SELECT DISTINCT
    period_id,
    CLI_ID
INTO #nm_candidate_clients
FROM #nm_candidates;

CREATE UNIQUE CLUSTERED INDEX IX_nm_candidate_clients
ON #nm_candidate_clients (period_id, CLI_ID);


/* ============================================================
   2. MASS-КЛИЕНТЫ, НЕ ПОПАВШИЕ В ПЕРВУЮ ВЫГРУЗКУ

   Исключаем #mass_clients.
   Исключаем ДЧБО по истории DepositInterestsRate.
   ============================================================ */

SELECT
    c.period_id,
    c.CLI_ID
INTO #nm_mass_clients
FROM #nm_candidate_clients c

INNER JOIN #periods p
    ON p.period_id = c.period_id

LEFT JOIN #mass_clients m
    ON m.period_id = c.period_id
   AND m.CLI_ID = c.CLI_ID

WHERE m.CLI_ID IS NULL

  AND NOT EXISTS (
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

CREATE UNIQUE CLUSTERED INDEX IX_nm_mass_clients
ON #nm_mass_clients (period_id, CLI_ID);


/* ============================================================
   3. ДОГОВОРЫ С НАДБАВКАМИ НОВ / НДП / НДМ
   ============================================================ */

SELECT
    c.period_id,
    c.CLI_ID,
    c.CON_ID,
    c.dt_open,

    CASE
        WHEN ISNULL(TRY_CAST(a.[Нов] AS int), 0) = 1
            THEN N'НОВ'

        WHEN ISNULL(TRY_CAST(a.[НДП] AS int), 0) = 1
          OR ISNULL(TRY_CAST(a.[НДМ] AS int), 0) = 1
            THEN N'НДП / НДМ'
    END AS category

INTO #nm_selected
FROM #nm_candidates c

INNER JOIN #nm_mass_clients m
    ON m.period_id = c.period_id
   AND m.CLI_ID = c.CLI_ID

OUTER APPLY (
    SELECT TOP (1)
        a.[Нов],
        a.[НДП],
        a.[НДМ]
    FROM ALM.ehd.attr_DepoFLConditions a WITH (NOLOCK)
    WHERE TRY_CAST(a.CON_ID AS bigint) = c.CON_ID
    ORDER BY
        a.DT_UPDATE DESC,
        a.loaddate DESC
) a

WHERE ISNULL(TRY_CAST(a.[Нов] AS int), 0) = 1
   OR ISNULL(TRY_CAST(a.[НДП] AS int), 0) = 1
   OR ISNULL(TRY_CAST(a.[НДМ] AS int), 0) = 1;

CREATE UNIQUE CLUSTERED INDEX IX_nm_selected
ON #nm_selected (period_id, CON_ID);

CREATE INDEX IX_nm_selected_con
ON #nm_selected (CON_ID);


/* ============================================================
   4. КОЛИЧЕСТВО ОТКРЫТЫХ ДОГОВОРОВ ПО ДНЯМ
   ============================================================ */

SELECT
    period_id,
    dt_open AS dt_rep,

    SUM(CASE
        WHEN category = N'НОВ'
            THEN CAST(1 AS bigint)
        ELSE 0
    END) AS nov_count,

    SUM(CASE
        WHEN category = N'НДП / НДМ'
            THEN CAST(1 AS bigint)
        ELSE 0
    END) AS ndp_ndm_count

INTO #nm_openings
FROM #nm_selected
GROUP BY
    period_id,
    dt_open;

CREATE UNIQUE CLUSTERED INDEX IX_nm_openings
ON #nm_openings (period_id, dt_rep);


/* ============================================================
   5. ИСТОРИЯ САЛЬДО ЧЕРЕЗ СОБЫТИЯ
   ============================================================ */

;WITH saldo_clipped AS (
    SELECT
        c.period_id,
        c.category,
        p.dt_to,

        CAST(s.OUT_RUB AS decimal(38,6)) AS out_rub,

        bounds.effective_from,
        bounds.effective_to

    FROM #nm_selected c

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

saldo_events AS (
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

    SUM(CASE
        WHEN category = N'НОВ'
            THEN delta
        ELSE 0
    END) AS nov_delta,

    SUM(CASE
        WHEN category = N'НДП / НДМ'
            THEN delta
        ELSE 0
    END) AS ndp_ndm_delta

INTO #nm_events
FROM saldo_events
GROUP BY
    period_id,
    dt_rep;

CREATE UNIQUE CLUSTERED INDEX IX_nm_events
ON #nm_events (period_id, dt_rep);


/* ============================================================
   6. ПОДНЕВНОЙ НАРАСТАЮЩИЙ ИТОГ
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

    SUM(ISNULL(o.nov_count, 0)) OVER (
        PARTITION BY c.period_id
        ORDER BY c.dt_rep
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS nov_opened_cum,

    SUM(ISNULL(o.ndp_ndm_count, 0)) OVER (
        PARTITION BY c.period_id
        ORDER BY c.dt_rep
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS ndp_ndm_opened_cum,

    SUM(ISNULL(e.nov_delta, 0)) OVER (
        PARTITION BY c.period_id
        ORDER BY c.dt_rep
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS nov_balance_rub,

    SUM(ISNULL(e.ndp_ndm_delta, 0)) OVER (
        PARTITION BY c.period_id
        ORDER BY c.dt_rep
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS ndp_ndm_balance_rub

INTO #nm_daily
FROM calendar c

INNER JOIN #periods p
    ON p.period_id = c.period_id

LEFT JOIN #nm_openings o
    ON o.period_id = c.period_id
   AND o.dt_rep = c.dt_rep

LEFT JOIN #nm_events e
    ON e.period_id = c.period_id
   AND e.dt_rep = c.dt_rep

OPTION (MAXRECURSION 0, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_nm_daily
ON #nm_daily (period_id, dt_rep);


/* ============================================================
   РЕЗУЛЬТАТ 1. ОКТЯБРЬ 2026
   ============================================================ */

SELECT
    dt_rep,
    nov_opened_cum AS nov_contracts,
    ndp_ndm_opened_cum AS ndp_ndm_contracts,
    nov_opened_cum + ndp_ndm_opened_cum AS total_contracts,

    CAST(nov_balance_rub / 1000000000.0
        AS decimal(18,6)) AS nov_bn,

    CAST(ndp_ndm_balance_rub / 1000000000.0
        AS decimal(18,6)) AS ndp_ndm_bn,

    CAST((nov_balance_rub + ndp_ndm_balance_rub)
        / 1000000000.0 AS decimal(18,6)) AS total_bn

FROM #nm_daily
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

    CAST(nov_balance_rub / 1000000000.0
        AS decimal(18,6)) AS nov_bn,

    CAST(ndp_ndm_balance_rub / 1000000000.0
        AS decimal(18,6)) AS ndp_ndm_bn,

    CAST((nov_balance_rub + ndp_ndm_balance_rub)
        / 1000000000.0 AS decimal(18,6)) AS total_bn

FROM #nm_daily
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

    CAST(nov_balance_rub / 1000000000.0
        AS decimal(18,6)) AS nov_bn,

    CAST(ndp_ndm_balance_rub / 1000000000.0
        AS decimal(18,6)) AS ndp_ndm_bn,

    CAST((nov_balance_rub + ndp_ndm_balance_rub)
        / 1000000000.0 AS decimal(18,6)) AS total_bn

FROM #nm_daily
WHERE period_id = 3
ORDER BY dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ 4. ПАНЕЛЬ НА 17 ДАТ
   ============================================================ */

;WITH days AS (
    SELECT day_num
    FROM (VALUES
        (1), (4), (7), (10), (13),
        (16), (19), (22), (25), (28),
        (30), (32), (33), (34), (35),
        (36), (38)
    ) v(day_num)
)

SELECT
    CONVERT(
        varchar(10),
        DATEADD(day, d.day_num - 1, DATEFROMPARTS(2025,10,1)),
        104
    ) AS [Column1],

    MAX(CASE WHEN x.period_id = 1
        THEN x.nov_opened_cum END) AS [Окт26 НОВ шт],

    MAX(CASE WHEN x.period_id = 1
        THEN x.ndp_ndm_opened_cum END) AS [Окт26 НДП НДМ шт],

    MAX(CASE WHEN x.period_id = 1
        THEN CAST(x.nov_balance_rub / 1e9 AS decimal(18,6))
    END) AS [Окт26 НОВ млрд],

    MAX(CASE WHEN x.period_id = 1
        THEN CAST(x.ndp_ndm_balance_rub / 1e9 AS decimal(18,6))
    END) AS [Окт26 НДП НДМ млрд],

    MAX(CASE WHEN x.period_id = 1
        THEN CAST((x.nov_balance_rub + x.ndp_ndm_balance_rub)
                  / 1e9 AS decimal(18,6))
    END) AS [РК октябрь 26],

    MAX(CASE WHEN x.period_id = 2
        THEN x.nov_opened_cum END) AS [Мар26 НОВ шт],

    MAX(CASE WHEN x.period_id = 2
        THEN x.ndp_ndm_opened_cum END) AS [Мар26 НДП НДМ шт],

    MAX(CASE WHEN x.period_id = 2
        THEN CAST(x.nov_balance_rub / 1e9 AS decimal(18,6))
    END) AS [Мар26 НОВ млрд],

    MAX(CASE WHEN x.period_id = 2
        THEN CAST(x.ndp_ndm_balance_rub / 1e9 AS decimal(18,6))
    END) AS [Мар26 НДП НДМ млрд],

    MAX(CASE WHEN x.period_id = 2
        THEN CAST((x.nov_balance_rub + x.ndp_ndm_balance_rub)
                  / 1e9 AS decimal(18,6))
    END) AS [РК март 26],

    MAX(CASE WHEN x.period_id = 3
        THEN x.nov_opened_cum END) AS [Окт25 НОВ шт],

    MAX(CASE WHEN x.period_id = 3
        THEN x.ndp_ndm_opened_cum END) AS [Окт25 НДП НДМ шт],

    MAX(CASE WHEN x.period_id = 3
        THEN CAST(x.nov_balance_rub / 1e9 AS decimal(18,6))
    END) AS [Окт25 НОВ млрд],

    MAX(CASE WHEN x.period_id = 3
        THEN CAST(x.ndp_ndm_balance_rub / 1e9 AS decimal(18,6))
    END) AS [Окт25 НДП НДМ млрд],

    MAX(CASE WHEN x.period_id = 3
        THEN CAST((x.nov_balance_rub + x.ndp_ndm_balance_rub)
                  / 1e9 AS decimal(18,6))
    END) AS [РК октябрь 25]

FROM days d
LEFT JOIN #nm_daily x
    ON x.day_num = d.day_num

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

    FROM #periods p

    LEFT JOIN #nm_daily d
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
