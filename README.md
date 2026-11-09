
USE ALM_TEST;
SET NOCOUNT ON;

DECLARE @Dt_start date = '2026-10-01';
DECLARE @Dt_end   date = '2026-10-31';

DECLARE @LastDtRep datetime2(7);

SELECT @LastDtRep = MAX(DT_REP)
FROM LIQUIDITY.liq.DepositInterestsRate WITH (NOLOCK);

DROP TABLE IF EXISTS #new_deals;
DROP TABLE IF EXISTS #new_balances;


/* ============================================================
   1. НОВЫЕ ПРИВЛЕЧЕНИЯ ЮЛ ЗА ОКТЯБРЬ 2026

   Только последняя DT_REP.
   Только рабочие дни.
   Фильтры исходной процедуры ЮЛ.
   ============================================================ */

SELECT
    d.CON_ID,
    d.CLI_ID,
    d.ISDOMRF,
    d.IS_PDR,
    d.DT_OPEN,
    CAST(d.DT_OPEN AS date) AS dt_open_d,
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

INTO #new_deals

FROM LIQUIDITY.liq.DepositInterestsRate d WITH (NOLOCK)

WHERE d.DT_REP = @LastDtRep

  AND d.DT_OPEN >= @Dt_start
  AND d.DT_OPEN < DATEADD(day, 1, @Dt_end)

  AND EXISTS (
      SELECT 1
      FROM ALM.info.VW_Calendar c WITH (NOLOCK)
      WHERE c.[Date] = CAST(d.DT_OPEN AS date)
        AND c.IsWorkDay = 1
  )

  AND d.CLI_SUBTYPE IN (
      N'ЮЛ Крупн.',
      N'ЮЛ ССВ'
  )

  AND (
      d.SEG_NAME IS NULL
      OR d.SEG_NAME <> N'Розница'
  )

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

OPTION (MAXDOP 2, RECOMPILE);

CREATE CLUSTERED INDEX IX_new_deals
ON #new_deals (CON_ID);


/* ============================================================
   2. SALDO НА ДАТУ ОТКРЫТИЯ

   Сначала обращаемся только к найденным договорам.

   Суммируем остатки непосредственно по CON_ID.
   Без GROUP BY по остальным атрибутам.
   ============================================================ */

SELECT
    d.*,
    s.BALANCE_RUB

INTO #new_balances

FROM #new_deals d

CROSS APPLY (
    SELECT
        SUM(CAST(s.OUT_RUB AS decimal(28,2)))
            AS BALANCE_RUB

    FROM LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)

    WHERE s.CON_ID = d.CON_ID
      AND s.DT_FROM <= d.dt_open_d
      AND s.DT_TO >= d.dt_open_d
      AND s.OUT_RUB <> 0
) s

WHERE s.BALANCE_RUB IS NOT NULL

OPTION (MAXDOP 2, RECOMPILE);

CREATE CLUSTERED INDEX IX_new_balances
ON #new_balances (CON_ID);


/* ============================================================
   3. ИТОГОВАЯ ПОСДЕЛОЧНАЯ ВЫГРУЗКА

   Реквизиты из вспомогательной витрины.
   Ставки — из DepositInterestsRate.
   Остатки — из DepositContract_Saldo.
   ============================================================ */

SELECT
    b.dt_open_d AS [Дата отчета],

    b.CON_ID,
    b.CLI_ID,

    m.INN,
    m.CLI_SHORT_NAME,

    b.ISDOMRF,
    m.PROD_TYPE,
    b.IS_PDR,

    b.DT_OPEN,
    b.DT_CLOSE,
    b.MATUR,
    b.RATE,
    m.CONV,

    b.ISFLOAT AS IS_FLOAT,
    b.CUR,

    b.BALANCE_RUB,

    b.MonthlyCONV_RATE,
    b.LIQ_ССВ_Fcast AS SSVRate_Fcast,

    b.MonthlyCONV_RATE + b.LIQ_ССВ_Fcast
        AS [Внешняя ставка +ССВ (ежемес)],

    CASE
        WHEN b.MonthlyCONV_LIQ_TransfertRate
             >= b.MonthlyCONV_RATE + b.LIQ_ССВ_Fcast
            THEN b.MonthlyCONV_LIQ_TransfertRate

        ELSE b.MonthlyCONV_RATE + b.LIQ_ССВ_Fcast
    END AS [ТС (ежемес до вычета ССВ)],

    b.MonthlyCONV_RATE
        + b.LIQ_ССВ_Fcast
        + b.ALM_OptionRate
        - b.MonthlyCONV_ForecastKeyRate
        AS [Спред внешней ставки к КС (ежемес)]

FROM #new_balances b

OUTER APPLY (
    SELECT TOP (1)
        m.INN,
        m.CLI_SHORT_NAME,
        m.PROD_TYPE,
        m.CONVENTION AS CONV

    FROM LIQUIDITY.rep.DepositContract_LEGAL_InterestsRate m
         WITH (NOLOCK)

    WHERE m.CON_ID = b.CON_ID

    ORDER BY m.DT_OPEN DESC
) m

ORDER BY b.DT_OPEN, b.CON_ID

OPTION (MAXDOP 2, RECOMPILE);
