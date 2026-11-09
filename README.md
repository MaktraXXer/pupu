
USE ALM_TEST;
SET NOCOUNT ON;

DECLARE @Dt_start date = '2026-11-01';
DECLARE @Dt_end   date = '2026-11-07';

DECLARE @LastDtRep datetime2(7);

SELECT @LastDtRep = MAX(DT_REP)
FROM LIQUIDITY.liq.DepositInterestsRate WITH (NOLOCK);

DROP TABLE IF EXISTS #ul_dates;
DROP TABLE IF EXISTS #ul_dep;
DROP TABLE IF EXISTS #ul_deals;
DROP TABLE IF EXISTS #ul_meta;
DROP TABLE IF EXISTS #ul_export;


/* ============================================================
   1. КАЛЕНДАРЬ

   Новые привлечения — только рабочие дни.
   Срезы — все календарные дни.
   ============================================================ */

SELECT
    CAST(c.[Date] AS date) AS dt_rep,
    MAX(CAST(c.IsWorkDay AS int)) AS IsWorkDay
INTO #ul_dates
FROM ALM.info.VW_Calendar c WITH (NOLOCK)
WHERE c.[Date] >= @Dt_start
  AND c.[Date] < DATEADD(day, 1, @Dt_end)
GROUP BY CAST(c.[Date] AS date);

CREATE UNIQUE CLUSTERED INDEX IX_ul_dates
ON #ul_dates (dt_rep);


/* ============================================================
   2. ДОГОВОРЫ ИЗ ПОСЛЕДНЕГО СРЕЗА DepositInterestsRate

   Повторяем фильтры исходной процедуры ЮЛ.

   Включаем:
   ЮЛ Крупн. + ЮЛ ССВ

   Исключаем:
   SEG_NAME = Розница

   Не ограничиваем:
   TERM_GROUP, IS_PDR, IS_FINANCE_LCR, IS_OPTION.
   ============================================================ */

SELECT
    dep.CON_ID,
    dep.CLI_ID,
    dep.CLI_SUBTYPE,
    dep.SEG_NAME,

    dep.ISDOMRF,
    dep.IS_PDR,

    dep.DT_OPEN,
    dep.DT_CLOSE,
    dep.MATUR,
    dep.RATE,
    dep.ISFLOAT,
    dep.CUR,

    dep.MonthlyCONV_RATE,
    dep.LIQ_ССВ_Fcast,
    dep.MonthlyCONV_LIQ_TransfertRate,
    dep.MonthlyCONV_ForecastKeyRate,
    dep.ALM_OptionRate

INTO #ul_dep

FROM LIQUIDITY.liq.DepositInterestsRate dep WITH (NOLOCK)

WHERE dep.DT_REP = @LastDtRep

  AND dep.CLI_SUBTYPE IN (
      N'ЮЛ Крупн.',
      N'ЮЛ ССВ'
  )

  AND ISNULL(dep.SEG_NAME, N'Прочий_сегмент')
      <> N'Розница'

  AND dep.CUR = 'RUR'
  AND dep.LIQ_ФОР IS NOT NULL
  AND dep.MonthlyCONV_RATE IS NOT NULL
  AND dep.MonthlyCONV_LIQ_TransfertRate IS NOT NULL

  AND dep.ISDOMRF = 0
  AND ISNULL(dep.ISFLOAT, 0) = 0
  AND dep.RATE > 0.01

  AND dep.MonthlyCONV_RATE BETWEEN
      dep.MonthlyCONV_ForecastKeyRate - 0.07
      AND dep.MonthlyCONV_ForecastKeyRate + 0.07

  AND dep.CLI_ID NOT IN (
      3731800,
      3939590185
  )

  AND dep.DT_OPEN < DATEADD(day, 1, @Dt_end)

  AND (
      dep.DT_CLOSE > @Dt_start
      OR dep.DT_OPEN >= @Dt_start
  );

CREATE INDEX IX_ul_dep
ON #ul_dep (CON_ID);


/* ============================================================
   3. ПОСДЕЛОЧНЫЕ ОСТАТКИ

   Начало день ко дню:
   DT_OPEN = дата отчета, рабочий день.

   Срез:
   DT_OPEN <= дата отчета < DT_CLOSE.

   Сальдо берём на соответствующую дату,
   как в исходной процедуре.
   ============================================================ */

;WITH report_dates AS (
    SELECT
        dt_rep,
        N'Срез' AS report_type
    FROM #ul_dates

    UNION ALL

    SELECT
        dt_rep,
        N'Начало день ко дню'
    FROM #ul_dates
    WHERE IsWorkDay = 1
)

SELECT
    r.dt_rep,
    r.report_type,

    d.CON_ID,
    d.CLI_ID,
    d.ISDOMRF,
    d.IS_PDR,

    d.DT_OPEN,
    d.DT_CLOSE,
    d.MATUR,
    d.RATE,
    d.ISFLOAT,
    d.CUR,

    SUM(CAST(s.OUT_RUB AS decimal(38,6)))
        AS BALANCE_RUB,

    d.MonthlyCONV_RATE,
    d.LIQ_ССВ_Fcast,
    d.MonthlyCONV_LIQ_TransfertRate,
    d.MonthlyCONV_ForecastKeyRate,
    d.ALM_OptionRate

INTO #ul_deals

FROM report_dates r

INNER JOIN #ul_dep d
    ON (
        r.report_type = N'Срез'
        AND d.DT_OPEN <= r.dt_rep
        AND d.DT_CLOSE > r.dt_rep
    )
    OR (
        r.report_type = N'Начало день ко дню'
        AND d.DT_OPEN = r.dt_rep
    )

INNER JOIN LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)
    ON s.CON_ID = d.CON_ID
   AND r.dt_rep BETWEEN s.DT_FROM AND s.DT_TO

WHERE s.OUT_RUB <> 0

GROUP BY
    r.dt_rep,
    r.report_type,

    d.CON_ID,
    d.CLI_ID,
    d.ISDOMRF,
    d.IS_PDR,

    d.DT_OPEN,
    d.DT_CLOSE,
    d.MATUR,
    d.RATE,
    d.ISFLOAT,
    d.CUR,

    d.MonthlyCONV_RATE,
    d.LIQ_ССВ_Fcast,
    d.MonthlyCONV_LIQ_TransfertRate,
    d.MonthlyCONV_ForecastKeyRate,
    d.ALM_OptionRate

OPTION (RECOMPILE);

CREATE INDEX IX_ul_deals
ON #ul_deals (report_type, dt_rep, CON_ID);


/* ============================================================
   4. ДОПОЛНИТЕЛЬНЫЕ АТРИБУТЫ

   INN, CLI_SHORT_NAME, PROD_TYPE, CONV.

   Подтягиваем из посделочной витрины.
   На отбор и финансовые расчеты не влияют.
   ============================================================ */

SELECT DISTINCT
    CON_ID,

    CAST(NULL AS nvarchar(100)) AS INN,
    CAST(NULL AS nvarchar(255)) AS CLI_SHORT_NAME,
    CAST(NULL AS nvarchar(255)) AS PROD_TYPE,
    CAST(NULL AS nvarchar(100)) AS CONV

INTO #ul_meta

FROM #ul_deals;

CREATE UNIQUE CLUSTERED INDEX IX_ul_meta
ON #ul_meta (CON_ID);


/* CONV может отсутствовать в старой версии витрины */

DECLARE @ConvExpr nvarchar(300);

IF COL_LENGTH(
    'LIQUIDITY.rep.DepositContract_LEGAL_InterestsRate',
    'CONV'
) IS NOT NULL
    SET @ConvExpr = N'CAST(m.CONV AS nvarchar(100))';
ELSE
    SET @ConvExpr = N'CAST(NULL AS nvarchar(100))';


DECLARE @sql nvarchar(max) = N'
UPDATE t

SET
    t.INN = x.INN,
    t.CLI_SHORT_NAME = x.CLI_SHORT_NAME,
    t.PROD_TYPE = x.PROD_TYPE,
    t.CONV = x.CONV

FROM #ul_meta t

OUTER APPLY (
    SELECT TOP (1)

        CAST(m.INN AS nvarchar(100)) AS INN,

        CAST(m.CLI_SHORT_NAME AS nvarchar(255))
            AS CLI_SHORT_NAME,

        CAST(m.PROD_TYPE AS nvarchar(255))
            AS PROD_TYPE,

        ' + @ConvExpr + N' AS CONV

    FROM LIQUIDITY.rep.DepositContract_LEGAL_InterestsRate m
         WITH (NOLOCK)

    WHERE m.CON_ID = t.CON_ID

    ORDER BY m.DT_OPEN DESC
) x;
';

EXEC sys.sp_executesql @sql;


/* ============================================================
   5. ИТОГОВАЯ ПОСДЕЛОЧНАЯ ВЫГРУЗКА

   Только запрошенные атрибуты и ставки.

   ТС* = MAX(ТС, Внешняя ставка + ССВ)

   Спред к КС повторяет расчет итоговой вьюхи,
   включая ALM_OptionRate.
   ============================================================ */

SELECT
    d.dt_rep AS [Дата отчета],

    d.CON_ID,
    d.CLI_ID,

    m.INN,
    m.CLI_SHORT_NAME,

    d.ISDOMRF,
    m.PROD_TYPE,
    d.IS_PDR,

    d.DT_OPEN,
    d.DT_CLOSE,
    d.MATUR,
    d.RATE,
    m.CONV,

    d.ISFLOAT AS IS_FLOAT,
    d.CUR,

    CAST(d.BALANCE_RUB AS decimal(28,2))
        AS BALANCE_RUB,

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
    ) AS [Спред внешней ставки к КС (ежемес)],

    d.report_type

INTO #ul_export

FROM #ul_deals d

LEFT JOIN #ul_meta m
    ON m.CON_ID = d.CON_ID;

CREATE INDEX IX_ul_export
ON #ul_export (report_type, [Дата отчета], CON_ID);


/* ============================================================
   РЕЗУЛЬТАТ 1.
   НОВЫЕ ПРИВЛЕЧЕНИЯ 01–07.11.2026

   Только рабочие дни по календарю.
   BALANCE_RUB — остаток на дату открытия.
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
    DT_OPEN,
    DT_CLOSE,
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
   РЕЗУЛЬТАТЫ 2–8.
   СЕМЬ БАЛАНСОВЫХ СРЕЗОВ

   01.11.2026
   02.11.2026
   03.11.2026
   04.11.2026
   05.11.2026
   06.11.2026
   07.11.2026
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
        DT_OPEN,
        DT_CLOSE,
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
