
USE ALM_TEST;
SET NOCOUNT ON;

DECLARE @Dt_start date = '2026-11-01';
DECLARE @Dt_end   date = '2026-11-07';

DECLARE @LastDtRep datetime;

SELECT @LastDtRep = MAX(DT_REP)
FROM LIQUIDITY.liq.DepositInterestsRate WITH (NOLOCK);

DROP TABLE IF EXISTS #ul_dates;
DROP TABLE IF EXISTS #ul_dep;
DROP TABLE IF EXISTS #ul_saldo;
DROP TABLE IF EXISTS #ul_report_dates;
DROP TABLE IF EXISTS #ul_balances;
DROP TABLE IF EXISTS #ul_ids;
DROP TABLE IF EXISTS #ul_meta;
DROP TABLE IF EXISTS #ul_export;


/* ============================================================
   1. КАЛЕНДАРЬ

   Новые привлечения: только рабочие дни.
   Срез: все календарные дни.
   ============================================================ */

SELECT
    CAST([Date] AS date) AS dt_rep,
    MAX(CAST(IsWorkDay AS int)) AS is_workday
INTO #ul_dates
FROM ALM.info.VW_Calendar WITH (NOLOCK)
WHERE [Date] >= @Dt_start
  AND [Date] < DATEADD(day, 1, @Dt_end)
GROUP BY CAST([Date] AS date);

CREATE UNIQUE CLUSTERED INDEX IX_ul_dates
ON #ul_dates (dt_rep);


/* ============================================================
   2. ДОГОВОРЫ ПО ФИЛЬТРАМ ПРОЦЕДУРЫ ЮЛ

   Берем только последний DT_REP.
   ЮЛ Крупн. + ЮЛ ССВ = ЮЛ.

   SEG_NAME = Розница исключаем.
   Остальные бизнес-сегменты оставляем.

   all termgroup / all PDR / all FINANCE_LCR
   не являются фильтрами отдельных сделок.
   ============================================================ */

SELECT
    d.CON_ID,
    d.CLI_ID,

    d.ISDOMRF,
    d.IS_PDR,

    CAST(d.DT_OPEN AS date) AS dt_open,
    CAST(d.DT_CLOSE AS date) AS dt_close,

    d.MATUR,
    d.RATE,
    d.ISFLOAT,
    d.CUR,

    d.MonthlyCONV_RATE,
    d.LIQ_ССВ_Fcast,
    d.MonthlyCONV_LIQ_TransfertRate,
    d.MonthlyCONV_ForecastKeyRate,
    d.ALM_OptionRate

INTO #ul_dep

FROM LIQUIDITY.liq.DepositInterestsRate d WITH (NOLOCK)

WHERE d.DT_REP = @LastDtRep

  AND d.CLI_SUBTYPE IN (
      N'ЮЛ Крупн.',
      N'ЮЛ ССВ'
  )

  AND ISNULL(d.SEG_NAME, N'Прочий_сегмент') <> N'Розница'

  AND d.CUR = 'RUR'
  AND d.LIQ_ФОР IS NOT NULL
  AND d.MonthlyCONV_RATE IS NOT NULL
  AND d.MonthlyCONV_LIQ_TransfertRate IS NOT NULL

  AND d.ISDOMRF = 0
  AND ISNULL(d.ISFLOAT, 0) = 0
  AND d.RATE > 0.01

  AND d.MonthlyCONV_RATE BETWEEN
      d.MonthlyCONV_ForecastKeyRate - 0.07
      AND d.MonthlyCONV_ForecastKeyRate + 0.07

  AND d.CLI_ID NOT IN (
      3731800,
      3939590185
  )

  AND d.DT_OPEN < DATEADD(day, 1, @Dt_end)
  AND d.DT_CLOSE > @Dt_start;

CREATE CLUSTERED INDEX IX_ul_dep
ON #ul_dep (CON_ID);


/* ============================================================
   3. SALDO

   Одно обращение к DepositContract_Saldo.

   Берем только интервалы, пересекающие нужную неделю,
   только по договорам из #ul_dep.

   Не размножаем строки на календарь при чтении SALDO.
   ============================================================ */

SELECT
    s.CON_ID,
    s.DT_FROM,
    s.DT_TO,
    CAST(s.OUT_RUB AS decimal(28,2)) AS OUT_RUB

INTO #ul_saldo

FROM LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)

INNER JOIN #ul_dep d
    ON d.CON_ID = s.CON_ID

WHERE s.DT_FROM <= @Dt_end
  AND s.DT_TO >= @Dt_start
  AND s.OUT_RUB <> 0

OPTION (MAXDOP 2, RECOMPILE);

CREATE CLUSTERED INDEX IX_ul_saldo
ON #ul_saldo (CON_ID, DT_FROM);


/* ============================================================
   4. ФОРМИРУЕМ НУЖНЫЕ ДАТЫ ПО ДОГОВОРАМ

   1) Начало день ко дню:
      только дата открытия + рабочий день.

   2) Срез:
      договор действует на отчетную дату.

   Здесь нет обращения к большим таблицам.
   ============================================================ */

SELECT
    CAST(N'Начало день ко дню' AS nvarchar(50))
        AS report_type,

    c.dt_rep,
    d.CON_ID

INTO #ul_report_dates

FROM #ul_dep d

INNER JOIN #ul_dates c
    ON c.dt_rep = d.dt_open
   AND c.is_workday = 1

UNION ALL

SELECT
    CAST(N'Срез' AS nvarchar(50)),
    c.dt_rep,
    d.CON_ID

FROM #ul_dep d

INNER JOIN #ul_dates c
    ON d.dt_open <= c.dt_rep
   AND d.dt_close > c.dt_rep;

CREATE CLUSTERED INDEX IX_ul_report_dates
ON #ul_report_dates (CON_ID, dt_rep, report_type);


/* ============================================================
   5. ПОСДЕЛОЧНЫЕ ОСТАТКИ

   Для каждой сделки и даты суммируем все подходящие
   записи SALDO, аналогично исходной процедуре.

   Сальдо уже находится во временной таблице.
   ============================================================ */

SELECT
    r.report_type,
    r.dt_rep,
    r.CON_ID,

    SUM(s.OUT_RUB) AS BALANCE_RUB

INTO #ul_balances

FROM #ul_report_dates r

INNER JOIN #ul_saldo s
    ON s.CON_ID = r.CON_ID
   AND r.dt_rep BETWEEN s.DT_FROM AND s.DT_TO

GROUP BY
    r.report_type,
    r.dt_rep,
    r.CON_ID

OPTION (MAXDOP 2, RECOMPILE);

CREATE CLUSTERED INDEX IX_ul_balances
ON #ul_balances (CON_ID, dt_rep, report_type);


/* ============================================================
   6. ДОГОВОРЫ, КОТОРЫЕ РЕАЛЬНО ПОПАЛИ В РЕЗУЛЬТАТ

   Дополнительные атрибуты читаем только для них.
   ============================================================ */

SELECT DISTINCT
    CON_ID
INTO #ul_ids
FROM #ul_balances;

CREATE UNIQUE CLUSTERED INDEX IX_ul_ids
ON #ul_ids (CON_ID);


/* ============================================================
   7. ДОПОЛНИТЕЛЬНЫЕ РЕКВИЗИТЫ

   INN, CLI_SHORT_NAME, PROD_TYPE, CONV.

   Одно соединение с витриной реквизитов.
   Если в витрине несколько строк по CON_ID,
   выбираем одну с последним DT_OPEN.

   Эти поля не участвуют в финансовом отборе.
   ============================================================ */

;WITH ranked AS (
    SELECT
        m.CON_ID,
        m.INN,
        m.CLI_SHORT_NAME,
        m.PROD_TYPE,
        m.CONV,

        ROW_NUMBER() OVER (
            PARTITION BY m.CON_ID
            ORDER BY m.DT_OPEN DESC
        ) AS rn

    FROM LIQUIDITY.rep.DepositContract_LEGAL_InterestsRate m
         WITH (NOLOCK)

    INNER JOIN #ul_ids i
        ON i.CON_ID = m.CON_ID
)

SELECT
    CON_ID,
    INN,
    CLI_SHORT_NAME,
    PROD_TYPE,
    CONV

INTO #ul_meta

FROM ranked
WHERE rn = 1

OPTION (MAXDOP 2, RECOMPILE);

CREATE UNIQUE CLUSTERED INDEX IX_ul_meta
ON #ul_meta (CON_ID);


/* ============================================================
   8. ГОТОВАЯ ПОСДЕЛОЧНАЯ ВЫГРУЗКА

   Финансовые показатели — из DepositInterestsRate.
   Остатки — из DepositContract_Saldo.

   ТС* для ЮЛ = MAX(
       MonthlyCONV_LIQ_TransfertRate,
       MonthlyCONV_RATE + LIQ_ССВ_Fcast
   )

   Spread_KeyRate — как в агрегированной вьюхе.
   ============================================================ */

SELECT
    b.dt_rep AS [Дата отчета],
    b.report_type,

    d.CON_ID,
    d.CLI_ID,
    m.INN,
    m.CLI_SHORT_NAME,

    d.ISDOMRF,
    m.PROD_TYPE,
    d.IS_PDR,

    d.dt_open,
    d.dt_close,
    d.MATUR,
    d.RATE,
    m.CONV,

    d.ISFLOAT AS IS_FLOAT,
    d.CUR,

    b.BALANCE_RUB,

    d.MonthlyCONV_RATE,
    d.LIQ_ССВ_Fcast AS SSVRate_Fcast,

    d.MonthlyCONV_RATE + d.LIQ_ССВ_Fcast
        AS [Внешняя ставка +ССВ (ежемес)],

    CASE
        WHEN d.MonthlyCONV_LIQ_TransfertRate
             >= d.MonthlyCONV_RATE + d.LIQ_ССВ_Fcast
            THEN d.MonthlyCONV_LIQ_TransfertRate

        ELSE d.MonthlyCONV_RATE + d.LIQ_ССВ_Fcast
    END AS [ТС (ежемес до вычета ССВ)],

    (
        d.MonthlyCONV_RATE
        + d.LIQ_ССВ_Fcast
        + d.ALM_OptionRate
        - d.MonthlyCONV_ForecastKeyRate
    ) AS [Спред внешней ставки к КС (ежемес)]

INTO #ul_export

FROM #ul_balances b

INNER JOIN #ul_dep d
    ON d.CON_ID = b.CON_ID

LEFT JOIN #ul_meta m
    ON m.CON_ID = b.CON_ID;

CREATE CLUSTERED INDEX IX_ul_export
ON #ul_export (report_type, [Дата отчета], CON_ID);


/* ============================================================
   РЕЗУЛЬТАТ 1. НОВЫЕ ПРИВЛЕЧЕНИЯ

   01–07.11.2026, только рабочие дни.
   Остаток на дату открытия.
   ============================================================ */

SELECT
    [Дата отчета],
    CON_ID,
    CLI_ID,
    INN,
    CLI_SHORT_NAME,
    ISDOMRF,
    PROD_TYPE,
    IS_PDR,
    dt_open,
    dt_close,
    MATUR,
    RATE,
    CONV,
    IS_FLOAT,
    CUR,
    BALANCE_RUB,
    MonthlyCONV_RATE,
    SSVRate_Fcast,
    [Внешняя ставка +ССВ (ежемес)],
    [ТС (ежемес до вычета ССВ)],
    [Спред внешней ставки к КС (ежемес)]

FROM #ul_export

WHERE report_type = N'Начало день ко дню'

ORDER BY [Дата отчета], CON_ID;


/* ============================================================
   РЕЗУЛЬТАТЫ 2–8. СЕМЬ ЕЖЕДНЕВНЫХ СРЕЗОВ

   01.11.2026
   02.11.2026
   03.11.2026
   04.11.2026
   05.11.2026
   06.11.2026
   07.11.2026

   Здесь только чтение маленькой #ul_export.
   ============================================================ */

DECLARE @Day date = @Dt_start;

WHILE @Day <= @Dt_end
BEGIN

    SELECT
        [Дата отчета],
        CON_ID,
        CLI_ID,
        INN,
        CLI_SHORT_NAME,
        ISDOMRF,
        PROD_TYPE,
        IS_PDR,
        dt_open,
        dt_close,
        MATUR,
        RATE,
        CONV,
        IS_FLOAT,
        CUR,
        BALANCE_RUB,
        MonthlyCONV_RATE,
        SSVRate_Fcast,
        [Внешняя ставка +ССВ (ежемес)],
        [ТС (ежемес до вычета ССВ)],
        [Спред внешней ставки к КС (ежемес)]

    FROM #ul_export

    WHERE report_type = N'Срез'
      AND [Дата отчета] = @Day

    ORDER BY CON_ID;

    SET @Day = DATEADD(day, 1, @Day);

END;
