
USE ALM;
SET NOCOUNT ON;

DROP TABLE IF EXISTS #cmp_periods;
DROP TABLE IF EXISTS #cmp_dates;
DROP TABLE IF EXISTS #cmp_contracts;
DROP TABLE IF EXISTS #cmp_clients;
DROP TABLE IF EXISTS #cmp_uchk;
DROP TABLE IF EXISTS #cmp_vw;
DROP TABLE IF EXISTS #cmp_saldo;

/* ============================================================
   1. ДВЕ РК И ЧЕТЫРЕ КОНТРОЛЬНЫЕ ДАТЫ
   ============================================================ */

CREATE TABLE #cmp_periods (
    period_id int PRIMARY KEY,
    period_name nvarchar(50),
    dt_from date,
    dt_to date
);

INSERT INTO #cmp_periods VALUES
    (1, N'Март 2026',    DATEFROMPARTS(2026,3,1),
                         DATEFROMPARTS(2026,4,7)),
    (2, N'Октябрь 2026', DATEFROMPARTS(2026,10,1),
                         DATEFROMPARTS(2026,11,4));

SELECT
    period_id,
    DATEADD(day, -1, dt_from) AS dt_rep,
    N'До РК' AS point_name
INTO #cmp_dates
FROM #cmp_periods

UNION ALL

SELECT
    period_id,
    dt_to,
    N'Конец РК'
FROM #cmp_periods;

CREATE UNIQUE CLUSTERED INDEX IX_cmp_dates
ON #cmp_dates (period_id, dt_rep);


/* ============================================================
   2. СПОСОБ №1: БАЛАНСОВАЯ ВИТРИНА

   Сегментация по TSEGMENTNAME:
   ДЧБО = УЧК, остальные = Розница.

   Исключаем маркетплейсы и ФУ.
   ============================================================ */

SELECT
    d.period_id,
    d.dt_rep,

    SUM(CASE
        WHEN t.section_name = N'Накопительный счёт'
         AND t.TSEGMENTNAME = N'ДЧБО'
            THEN CAST(t.out_rub AS decimal(28,2))
        ELSE 0
    END) AS ns_uchk,

    SUM(CASE
        WHEN t.section_name = N'Срочные'
         AND t.TSEGMENTNAME = N'ДЧБО'
            THEN CAST(t.out_rub AS decimal(28,2))
        ELSE 0
    END) AS dep_uchk,

    SUM(CASE
        WHEN t.section_name = N'Накопительный счёт'
         AND (t.TSEGMENTNAME <> N'ДЧБО'
              OR t.TSEGMENTNAME IS NULL)
            THEN CAST(t.out_rub AS decimal(28,2))
        ELSE 0
    END) AS ns_retail,

    SUM(CASE
        WHEN t.section_name = N'Срочные'
         AND (t.TSEGMENTNAME <> N'ДЧБО'
              OR t.TSEGMENTNAME IS NULL)
            THEN CAST(t.out_rub AS decimal(28,2))
        ELSE 0
    END) AS dep_retail

INTO #cmp_vw

FROM #cmp_dates d

LEFT JOIN ALM.ALM.VW_balance_rest_all t WITH (NOLOCK)
    ON t.dt_rep = d.dt_rep

   AND t.section_name IN (
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

   AND (
       t.section_name <> N'Срочные'
       OR t.PROD_NAME_res IS NULL
       OR t.PROD_NAME_res NOT IN (
           N'ДОМа надёжно',
           N'Всё в ДОМ',
           N'Надёжный',
           N'Надёжный прайм',
           N'Надёжныйпрайм',
           N'Надёжный VIP',
           N'НадёжныйVIP',
           N'Надёжный премиум',
           N'Надёжный промо',
           N'Надёжныйпромо',
           N'Надёжный старт',
           N'Надёжный Т2',
           N'Надёжный T2',
           N'Надёжный Мегафон',
           N'Надёжный процент',
           N'Надёжныйпроцент',
           N'Могучий'
       )
   )

GROUP BY
    d.period_id,
    d.dt_rep;

CREATE UNIQUE CLUSTERED INDEX IX_cmp_vw
ON #cmp_vw (period_id, dt_rep);


/* ============================================================
   3. СПОСОБ №2: РЕЕСТР ДОГОВОРОВ ДЛЯ SALDO

   Используем те же продуктовые исключения.
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
            N'Надёжный прайм',
            N'Надёжныйпрайм',
            N'Надёжный VIP',
            N'НадёжныйVIP',
            N'Надёжный премиум',
            N'Надёжный промо',
            N'Надёжныйпромо',
            N'Надёжный старт',
            N'Надёжный Т2',
            N'Надёжный T2',
            N'Надёжный Мегафон',
            N'Надёжный процент',
            N'Надёжныйпроцент',
            N'Могучий'
        )
            THEN 1
        ELSE 0
    END) AS is_excluded

INTO #cmp_contracts

FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)

WHERE a.DT_OPEN IS NOT NULL
  AND a.DT_OPEN < DATEFROMPARTS(2026,11,5)
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

CREATE UNIQUE CLUSTERED INDEX IX_cmp_contracts
ON #cmp_contracts (CON_ID);

CREATE INDEX IX_cmp_contracts_cli
ON #cmp_contracts (CLI_ID);


/* ============================================================
   4. УНИКАЛЬНЫЕ КЛИЕНТЫ И ИСТОРИЧЕСКИЙ ПРИЗНАК УЧК

   Если был хотя бы один договор с seg_name = ДЧБО,
   клиент УЧК с даты открытия первого такого договора.
   ============================================================ */

SELECT DISTINCT
    CLI_ID
INTO #cmp_clients
FROM #cmp_contracts
WHERE is_excluded = 0;

CREATE UNIQUE CLUSTERED INDEX IX_cmp_clients
ON #cmp_clients (CLI_ID);


SELECT
    h.CLI_ID,
    MIN(CAST(h.DT_OPEN AS date)) AS first_uchk_date

INTO #cmp_uchk

FROM LIQUIDITY.liq.DepositContract_all h WITH (NOLOCK)

INNER JOIN #cmp_clients c
    ON c.CLI_ID = h.CLI_ID

WHERE h.CON_ID IS NOT NULL
  AND h.DT_OPEN IS NOT NULL
  AND h.DT_OPEN < DATEFROMPARTS(2026,11,5)

  AND EXISTS (
      SELECT 1
      FROM LIQUIDITY.liq.DepositInterestsRate r WITH (NOLOCK)
      WHERE r.CON_ID = h.CON_ID
        AND r.seg_name = N'ДЧБО'
  )

GROUP BY h.CLI_ID;

CREATE UNIQUE CLUSTERED INDEX IX_cmp_uchk
ON #cmp_uchk (CLI_ID);


/* ============================================================
   5. ОСТАТКИ ИЗ DepositContract_Saldo НА ТЕ ЖЕ 4 ДАТЫ

   Без календаря и накопительных событий:
   берем срезы непосредственно на нужные даты.

   При пересечении интервалов одного CON_ID
   выбираем запись с наиболее поздним DT_FROM.
   ============================================================ */

;WITH snapshots AS (
    SELECT
        d.period_id,
        d.dt_rep,
        c.CON_ID,
        c.is_ns,

        CASE
            WHEN u.first_uchk_date <= p.dt_to
                THEN 1
            ELSE 0
        END AS is_uchk,

        CAST(s.OUT_RUB AS decimal(28,2)) AS out_rub,

        ROW_NUMBER() OVER (
            PARTITION BY d.period_id, d.dt_rep, c.CON_ID
            ORDER BY s.DT_FROM DESC, s.DT_TO DESC
        ) AS rn

    FROM #cmp_dates d

    INNER JOIN #cmp_periods p
        ON p.period_id = d.period_id

    INNER JOIN #cmp_contracts c
        ON c.dt_open <= d.dt_rep
       AND c.is_excluded = 0

    LEFT JOIN #cmp_uchk u
        ON u.CLI_ID = c.CLI_ID

    INNER JOIN LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)
        ON s.CON_ID = c.CON_ID
       AND s.DT_FROM < DATEADD(day, 1, d.dt_rep)
       AND (
           s.DT_TO IS NULL
           OR s.DT_TO >= d.dt_rep
       )

    WHERE s.OUT_RUB IS NOT NULL
      AND s.OUT_RUB >= 0
)

SELECT
    d.period_id,
    d.dt_rep,

    SUM(CASE
        WHEN s.is_ns = 1
         AND s.is_uchk = 1
            THEN s.out_rub
        ELSE 0
    END) AS ns_uchk,

    SUM(CASE
        WHEN s.is_ns = 0
         AND s.is_uchk = 1
            THEN s.out_rub
        ELSE 0
    END) AS dep_uchk,

    SUM(CASE
        WHEN s.is_ns = 1
         AND s.is_uchk = 0
            THEN s.out_rub
        ELSE 0
    END) AS ns_retail,

    SUM(CASE
        WHEN s.is_ns = 0
         AND s.is_uchk = 0
            THEN s.out_rub
        ELSE 0
    END) AS dep_retail

INTO #cmp_saldo

FROM #cmp_dates d

LEFT JOIN snapshots s
    ON s.period_id = d.period_id
   AND s.dt_rep = d.dt_rep
   AND s.rn = 1

GROUP BY
    d.period_id,
    d.dt_rep;

CREATE UNIQUE CLUSTERED INDEX IX_cmp_saldo
ON #cmp_saldo (period_id, dt_rep);


/* ============================================================
   6. ИТОГОВОЕ СРАВНЕНИЕ

   4 СТРОКИ:
   28.02.2026, 07.04.2026,
   30.09.2026, 04.11.2026.

   Все объемы и приросты в млрд руб.
   ============================================================ */

;WITH comparison AS (
    SELECT
        d.period_id,
        p.period_name,
        d.dt_rep,
        d.point_name,

        ISNULL(v.ns_uchk, 0) AS vw_ns_uchk,
        ISNULL(v.dep_uchk, 0) AS vw_dep_uchk,
        ISNULL(v.ns_retail, 0) AS vw_ns_retail,
        ISNULL(v.dep_retail, 0) AS vw_dep_retail,

        ISNULL(s.ns_uchk, 0) AS saldo_ns_uchk,
        ISNULL(s.dep_uchk, 0) AS saldo_dep_uchk,
        ISNULL(s.ns_retail, 0) AS saldo_ns_retail,
        ISNULL(s.dep_retail, 0) AS saldo_dep_retail

    FROM #cmp_dates d

    INNER JOIN #cmp_periods p
        ON p.period_id = d.period_id

    LEFT JOIN #cmp_vw v
        ON v.period_id = d.period_id
       AND v.dt_rep = d.dt_rep

    LEFT JOIN #cmp_saldo s
        ON s.period_id = d.period_id
       AND s.dt_rep = d.dt_rep
),

totals AS (
    SELECT
        *,

        vw_ns_uchk + vw_dep_uchk AS vw_uchk,
        saldo_ns_uchk + saldo_dep_uchk AS saldo_uchk,

        vw_ns_retail + vw_dep_retail AS vw_retail,
        saldo_ns_retail + saldo_dep_retail AS saldo_retail

    FROM comparison
)

SELECT
    period_name AS [РК],
    CONVERT(varchar(10), dt_rep, 104) AS [Дата],
    point_name AS [Срез],

    CAST(vw_ns_uchk / 1e9 AS decimal(18,4))
        AS [Баланс НС УЧК],

    CAST(vw_dep_uchk / 1e9 AS decimal(18,4))
        AS [Баланс вклады УЧК],

    CAST(vw_uchk / 1e9 AS decimal(18,4))
        AS [Баланс УЧК всего],

    CAST(saldo_ns_uchk / 1e9 AS decimal(18,4))
        AS [Saldo НС УЧК],

    CAST(saldo_dep_uchk / 1e9 AS decimal(18,4))
        AS [Saldo вклады УЧК],

    CAST(saldo_uchk / 1e9 AS decimal(18,4))
        AS [Saldo УЧК всего],

    CAST((saldo_uchk - vw_uchk) / 1e9
        AS decimal(18,4))
        AS [Разница УЧК],

    CAST(vw_retail / 1e9 AS decimal(18,4))
        AS [Баланс Розница всего],

    CAST(saldo_retail / 1e9 AS decimal(18,4))
        AS [Saldo Розница всего],

    CAST(
        (saldo_uchk + saldo_retail
         - vw_uchk - vw_retail) / 1e9
        AS decimal(18,4))
        AS [Разница всего портфеля],

    CAST(
        (
            vw_uchk
            - FIRST_VALUE(vw_uchk) OVER (
                PARTITION BY period_id
                ORDER BY dt_rep
            )
        ) / 1e9 AS decimal(18,4))
        AS [Прирост УЧК баланс],

    CAST(
        (
            saldo_uchk
            - FIRST_VALUE(saldo_uchk) OVER (
                PARTITION BY period_id
                ORDER BY dt_rep
            )
        ) / 1e9 AS decimal(18,4))
        AS [Прирост УЧК Saldo]

FROM totals
ORDER BY period_id, dt_rep;
