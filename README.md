
USE ALM_TEST;
SET NOCOUNT ON;

DECLARE @Dt_start date = '2026-11-01';
DECLARE @Dt_end   date = '2026-11-07';

DECLARE @LastDtRep datetime2(7);

SELECT @LastDtRep = MAX(DT_REP)
FROM LIQUIDITY.liq.DepositInterestsRate WITH (NOLOCK);

DROP TABLE IF EXISTS #deals;
DROP TABLE IF EXISTS #balances;
DROP TABLE IF EXISTS #meta;
DROP TABLE IF EXISTS #result;


/* ============================================================
   1. ДОГОВОРЫ И СТАВКИ

   DepositInterestsRate — только последний DT_REP.
   Фильтры исходной процедуры ЮЛ.
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

INTO #deals

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

  AND (
      d.DT_CLOSE > @Dt_start
      OR d.DT_OPEN >= @Dt_start
  );

CREATE CLUSTERED INDEX IX_deals
ON #deals (CON_ID);


/* ============================================================
   2. ПРИСОЕДИНЯЕМ SALDO НА КОНКРЕТНЫЕ ДАТЫ

   Начало день ко дню:
       дата открытия, только IsWorkDay = 1.

   Срез:
       договор действует на дату.

   Не выгружаем историю сальдо отдельно.
   ============================================================ */

;WITH calendar AS (
    SELECT
        CAST(c.[Date] AS date) AS dt_rep,
        c.IsWorkDay

    FROM ALM.info.VW_Calendar c WITH (NOLOCK)

    WHERE c.[Date] >= @Dt_start
      AND c.[Date] < DATEADD(day, 1, @Dt_end)
),

report_dates AS (
    SELECT
        d.CON_ID,
        c.dt_rep,
        CAST(N'Начало день ко дню' AS nvarchar(50))
            AS report_type

    FROM #deals d

    INNER JOIN calendar c
        ON c.dt_rep = d.dt_open
       AND c.IsWorkDay = 1

    UNION ALL

    SELECT
        d.CON_ID,
        c.dt_rep,
        CAST(N'Срез' AS nvarchar(50))

    FROM #deals d

    INNER JOIN calendar c
        ON d.dt_open <= c.dt_rep
       AND d.dt_close > c.dt_rep
)

SELECT
    r.report_type,
    r.dt_rep,
    r.CON_ID,

    SUM(CAST(s.OUT_RUB AS decimal(28,2)))
        AS BALANCE_RUB

INTO #balances

FROM report_dates r

INNER JOIN LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)
    ON s.CON_ID = r.CON_ID
   AND r.dt_rep BETWEEN s.DT_FROM AND s.DT_TO

WHERE s.OUT_RUB <> 0

GROUP BY
    r.report_type,
    r.dt_rep,
    r.CON_ID

OPTION (MAXDOP 2, RECOMPILE);

CREATE CLUSTERED INDEX IX_balances
ON #balances (CON_ID, dt_rep, report_type);


/* ============================================================
   3. ДОПОЛНИТЕЛЬНЫЕ РЕКВИЗИТЫ

   Только для договоров, попавших в #balances.

   Вспомогательная витрина без DT_REP.
   ============================================================ */

SELECT
    i.CON_ID,
    m.INN,
    m.CLI_SHORT_NAME,
    m.PROD_TYPE,
    m.CONV

INTO #meta

FROM (
    SELECT DISTINCT CON_ID
    FROM #balances
) i

OUTER APPLY (
    SELECT TOP (1)
        x.INN,
        x.CLI_SHORT_NAME,
        x.PROD_TYPE,
        x.CONVENTION AS CONV

    FROM LIQUIDITY.rep.DepositContract_LEGAL_InterestsRate x
         WITH (NOLOCK)

    WHERE x.CON_ID = i.CON_ID

    ORDER BY x.DT_OPEN DESC
) m;

CREATE UNIQUE CLUSTERED INDEX IX_meta
ON #meta (CON_ID);


/* ============================================================
   4. ГОТОВАЯ ПОСДЕЛОЧНАЯ ВЫГРУЗКА

   Spread_KeyRate — формула из агрегированной вьюхи.
   ТС* = MAX(ТС, внешняя ставка + ССВ).
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

    d.MonthlyCONV_RATE
        + d.LIQ_ССВ_Fcast
        + d.ALM_OptionRate
        - d.MonthlyCONV_ForecastKeyRate
        AS [Спред внешней ставки к КС (ежемес)]

INTO #result

FROM #balances b

INNER JOIN #deals d
    ON d.CON_ID = b.CON_ID

LEFT JOIN #meta m
    ON m.CON_ID = b.CON_ID;

CREATE CLUSTERED INDEX IX_result
ON #result (report_type, [Дата отчета], CON_ID);


/* ============================================================
   РЕЗУЛЬТАТ 1. НОВЫЕ ПРИВЛЕЧЕНИЯ

   01–07.11.2026, только рабочие дни.
   ============================================================ */

SELECT *
FROM #result
WHERE report_type = N'Начало день ко дню'
ORDER BY [Дата отчета], CON_ID;


/* ============================================================
   РЕЗУЛЬТАТЫ 2–8. СЕМЬ СРЕЗОВ

   01.11.2026 — 07.11.2026.
   ============================================================ */

DECLARE @Day date = @Dt_start;

WHILE @Day <= @Dt_end
BEGIN

    SELECT *
    FROM #result
    WHERE report_type = N'Срез'
      AND [Дата отчета] = @Day
    ORDER BY CON_ID;

    SET @Day = DATEADD(day, 1, @Day);

END;
