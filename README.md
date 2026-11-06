
USE ALM;
SET NOCOUNT ON;

DROP TABLE IF EXISTS #cmp_periods;
DROP TABLE IF EXISTS #cmp_dates;
DROP TABLE IF EXISTS #cmp_excluded;
DROP TABLE IF EXISTS #cmp_contracts;
DROP TABLE IF EXISTS #cmp_clients;
DROP TABLE IF EXISTS #cmp_uchk;
DROP TABLE IF EXISTS #cmp_vw;
DROP TABLE IF EXISTS #cmp_saldo;
DROP TABLE IF EXISTS #cmp_all;


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
    (1, N'Март 2026',
        DATEFROMPARTS(2026,3,1),
        DATEFROMPARTS(2026,4,7)),

    (2, N'Октябрь 2026',
        DATEFROMPARTS(2026,10,1),
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
   2. ЕДИНЫЕ ИСКЛЮЧЕНИЯ ДЛЯ ДВУХ ИСТОЧНИКОВ

   Технические продукты, маркетплейсы, ФУ.
   ============================================================ */

CREATE TABLE #cmp_excluded (
    PROD_NAME nvarchar(200) PRIMARY KEY
);

INSERT INTO #cmp_excluded VALUES
    (N'Агентские эскроу ФЛ по ставке КС+спред'),
    (N'Спец. банк.счёт'),
    (N'Залоговый'),
    (N'Агентские эскроу ФЛ по ставке КС/2'),
    (N'Эскроу'),
    (N'Депозит ФЛ (суррогатный договор для счета Новой Афины)'),
    (N'ДОМа надёжно'),
    (N'Всё в ДОМ'),
    (N'Надёжный'),
    (N'Надёжный прайм'),
    (N'Надёжныйпрайм'),
    (N'Надёжный VIP'),
    (N'НадёжныйVIP'),
    (N'Надёжный премиум'),
    (N'Надёжный промо'),
    (N'Надёжныйпромо'),
    (N'Надёжный старт'),
    (N'Надёжный Т2'),
    (N'Надёжный T2'),
    (N'Надёжный Мегафон'),
    (N'Надёжный процент'),
    (N'Надёжныйпроцент'),
    (N'Могучий');


/* ============================================================
   3. СПОСОБ №1. БАЛАНСОВАЯ ВИТРИНА

   TSEGMENTNAME = ДЧБО -> УЧК
   Остальные -> MASS
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
         AND (
             t.TSEGMENTNAME <> N'ДЧБО'
             OR t.TSEGMENTNAME IS NULL
         )
            THEN CAST(t.out_rub AS decimal(28,2))
        ELSE 0
    END) AS ns_mass,

    SUM(CASE
        WHEN t.section_name = N'Срочные'
         AND (
             t.TSEGMENTNAME <> N'ДЧБО'
             OR t.TSEGMENTNAME IS NULL
         )
            THEN CAST(t.out_rub AS decimal(28,2))
        ELSE 0
    END) AS dep_mass

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

   AND NOT EXISTS (
       SELECT 1
       FROM #cmp_excluded x
       WHERE x.PROD_NAME = t.PROD_NAME_res
   )

GROUP BY
    d.period_id,
    d.dt_rep;

CREATE UNIQUE CLUSTERED INDEX IX_cmp_vw
ON #cmp_vw (period_id, dt_rep);


/* ============================================================
   4. СПОСОБ №2. РЕЕСТР ДОГОВОРОВ ДЛЯ SALDO
   ============================================================ */

SELECT
    a.CON_ID,
    MIN(a.CLI_ID) AS CLI_ID,
    MIN(CAST(a.DT_OPEN AS date)) AS dt_open,

    MAX(CASE
        WHEN a.PROD_NAME LIKE N'Накопительный счёт%'
            THEN 1
        ELSE 0
    END) AS is_ns

INTO #cmp_contracts

FROM LIQUIDITY.liq.DepositContract_all a WITH (NOLOCK)

WHERE a.DT_OPEN IS NOT NULL
  AND a.DT_OPEN < DATEFROMPARTS(2026,11,5)
  AND a.CON_ID IS NOT NULL
  AND a.CLI_ID IS NOT NULL
  AND a.PROD_NAME IS NOT NULL
  AND a.cli_short_name = N'ФЛ'
  AND a.cur = 'RUR'

  AND NOT EXISTS (
      SELECT 1
      FROM #cmp_excluded x
      WHERE x.PROD_NAME = a.PROD_NAME
  )

GROUP BY a.CON_ID;

CREATE UNIQUE CLUSTERED INDEX IX_cmp_contracts
ON #cmp_contracts (CON_ID);

CREATE INDEX IX_cmp_contracts_cli
ON #cmp_contracts (CLI_ID);


/* ============================================================
   5. ИСТОРИЧЕСКОЕ ОПРЕДЕЛЕНИЕ УЧК

   Проверяем все договоры клиента, включая исключенные
   продукты, по признаку ДЧБО в DepositInterestsRate.

   Сегмент фиксируется на конец каждой РК.
   ============================================================ */

SELECT DISTINCT
    CLI_ID
INTO #cmp_clients
FROM #cmp_contracts;

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
   6. SALDO НА ЧЕТЫРЕ КОНТРОЛЬНЫЕ ДАТЫ

   Берем остаток договора на дату среза.
   Если интервалы пересеклись, берем последний DT_FROM.
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
        WHEN s.is_ns = 1 AND s.is_uchk = 1
            THEN s.out_rub ELSE 0
    END) AS ns_uchk,

    SUM(CASE
        WHEN s.is_ns = 0 AND s.is_uchk = 1
            THEN s.out_rub ELSE 0
    END) AS dep_uchk,

    SUM(CASE
        WHEN s.is_ns = 1 AND s.is_uchk = 0
            THEN s.out_rub ELSE 0
    END) AS ns_mass,

    SUM(CASE
        WHEN s.is_ns = 0 AND s.is_uchk = 0
            THEN s.out_rub ELSE 0
    END) AS dep_mass

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
   7. СОБИРАЕМ ДВА ИСТОЧНИКА В ОДНУ ТАБЛИЦУ
   ============================================================ */

SELECT
    d.period_id,
    p.period_name,
    d.dt_rep,
    d.point_name,

    ISNULL(v.ns_uchk, 0)  AS bal_ns_uchk,
    ISNULL(s.ns_uchk, 0)  AS sal_ns_uchk,

    ISNULL(v.dep_uchk, 0) AS bal_dep_uchk,
    ISNULL(s.dep_uchk, 0) AS sal_dep_uchk,

    ISNULL(v.ns_mass, 0)  AS bal_ns_mass,
    ISNULL(s.ns_mass, 0)  AS sal_ns_mass,

    ISNULL(v.dep_mass, 0) AS bal_dep_mass,
    ISNULL(s.dep_mass, 0) AS sal_dep_mass

INTO #cmp_all

FROM #cmp_dates d

INNER JOIN #cmp_periods p
    ON p.period_id = d.period_id

LEFT JOIN #cmp_vw v
    ON v.period_id = d.period_id
   AND v.dt_rep = d.dt_rep

LEFT JOIN #cmp_saldo s
    ON s.period_id = d.period_id
   AND s.dt_rep = d.dt_rep;

CREATE UNIQUE CLUSTERED INDEX IX_cmp_all
ON #cmp_all (period_id, dt_rep);


/* ============================================================
   РЕЗУЛЬТАТ 1. СРАВНЕНИЕ ОСТАТКОВ

   Ровно 4 даты, все объемы в млрд руб.
   Разница = SALDO - БАЛАНС.
   ============================================================ */

SELECT
    period_name AS [РК],
    CONVERT(varchar(10), dt_rep, 104) AS [Дата],

    CAST(bal_ns_uchk / 1e9 AS decimal(18,4))
        AS [НС УЧК баланс],

    CAST(sal_ns_uchk / 1e9 AS decimal(18,4))
        AS [НС УЧК сальдо],

    CAST(bal_dep_uchk / 1e9 AS decimal(18,4))
        AS [Вклады УЧК баланс],

    CAST(sal_dep_uchk / 1e9 AS decimal(18,4))
        AS [Вклады УЧК сальдо],

    CAST((bal_ns_uchk + bal_dep_uchk) / 1e9
        AS decimal(18,4)) AS [Итого УЧК баланс],

    CAST((sal_ns_uchk + sal_dep_uchk) / 1e9
        AS decimal(18,4)) AS [Итого УЧК сальдо],

    CAST((
        sal_ns_uchk + sal_dep_uchk
        - bal_ns_uchk - bal_dep_uchk
    ) / 1e9 AS decimal(18,4)) AS [Разница УЧК],


    CAST(bal_ns_mass / 1e9 AS decimal(18,4))
        AS [НС MASS баланс],

    CAST(sal_ns_mass / 1e9 AS decimal(18,4))
        AS [НС MASS сальдо],

    CAST(bal_dep_mass / 1e9 AS decimal(18,4))
        AS [Вклады MASS баланс],

    CAST(sal_dep_mass / 1e9 AS decimal(18,4))
        AS [Вклады MASS сальдо],

    CAST((bal_ns_mass + bal_dep_mass) / 1e9
        AS decimal(18,4)) AS [Итого MASS баланс],

    CAST((sal_ns_mass + sal_dep_mass) / 1e9
        AS decimal(18,4)) AS [Итого MASS сальдо],

    CAST((
        sal_ns_mass + sal_dep_mass
        - bal_ns_mass - bal_dep_mass
    ) / 1e9 AS decimal(18,4)) AS [Разница MASS]

FROM #cmp_all
ORDER BY period_id, dt_rep;


/* ============================================================
   РЕЗУЛЬТАТ 2. СРАВНЕНИЕ ПРИРОСТОВ ЗА РК

   Прирост = остаток на конец РК
           - остаток на день до РК.

   Всего, MASS, УЧК — млрд руб.
   ============================================================ */

;WITH growth AS (
    SELECT
        p.period_id,
        p.period_name,

        (
            e.bal_ns_uchk + e.bal_dep_uchk
            - b.bal_ns_uchk - b.bal_dep_uchk
        ) AS uchk_balance,

        (
            e.sal_ns_uchk + e.sal_dep_uchk
            - b.sal_ns_uchk - b.sal_dep_uchk
        ) AS uchk_saldo,

        (
            e.bal_ns_mass + e.bal_dep_mass
            - b.bal_ns_mass - b.bal_dep_mass
        ) AS mass_balance,

        (
            e.sal_ns_mass + e.sal_dep_mass
            - b.sal_ns_mass - b.sal_dep_mass
        ) AS mass_saldo

    FROM #cmp_periods p

    INNER JOIN #cmp_all b
        ON b.period_id = p.period_id
       AND b.dt_rep = DATEADD(day, -1, p.dt_from)

    INNER JOIN #cmp_all e
        ON e.period_id = p.period_id
       AND e.dt_rep = p.dt_to
)

SELECT
    period_name AS [РК],

    CAST((uchk_balance + mass_balance) / 1e9
        AS decimal(18,4)) AS [Весь портфель баланс],

    CAST((uchk_saldo + mass_saldo) / 1e9
        AS decimal(18,4)) AS [Весь портфель сальдо],

    CAST((
        uchk_saldo + mass_saldo
        - uchk_balance - mass_balance
    ) / 1e9 AS decimal(18,4)) AS [Разница всего],


    CAST(mass_balance / 1e9
        AS decimal(18,4)) AS [MASS баланс],

    CAST(mass_saldo / 1e9
        AS decimal(18,4)) AS [MASS сальдо],

    CAST((mass_saldo - mass_balance) / 1e9
        AS decimal(18,4)) AS [Разница MASS],


    CAST(uchk_balance / 1e9
        AS decimal(18,4)) AS [УЧК баланс],

    CAST(uchk_saldo / 1e9
        AS decimal(18,4)) AS [УЧК сальдо],

    CAST((uchk_saldo - uchk_balance) / 1e9
        AS decimal(18,4)) AS [Разница УЧК]

FROM growth
ORDER BY period_id;
