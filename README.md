
USE ALM_TEST;
SET NOCOUNT ON;

DECLARE @Dt_start date = '2026-10-01';
DECLARE @Dt_end   date = '2026-10-31';

DECLARE @LastDtRep datetime2(7);

SELECT @LastDtRep = MAX(DT_REP)
FROM LIQUIDITY.liq.DepositInterestsRate WITH (NOLOCK);

DROP TABLE IF EXISTS #deals;
DROP TABLE IF EXISTS #balances;


/* ============================================================
   1. НОВЫЕ ПРИВЛЕЧЕНИЯ ЮЛ ЗА ОКТЯБРЬ

   Только последний DT_REP.
   Только рабочие дни.
   Фильтры исходной процедуры ЮЛ.
   ============================================================ */

SELECT
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

INTO #deals

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

  AND ISNULL(d.SEG_NAME, N'Прочий_сегмент')
      <> N'Розница'

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
  );

CREATE INDEX IX_deals_con
ON #deals (CON_ID);


/* ============================================================
   2. SALDO НА ДАТУ ОТКРЫТИЯ

   Только подходящие договоры.
   Не читаем сальдо за остальные дни.
   ============================================================ */

SELECT
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
    d.ALM_OptionRate,

    SUM(CAST(s.OUT_RUB AS decimal(28,2)))
        AS BALANCE_RUB

INTO #balances

FROM #deals d

INNER JOIN LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)
    ON s.CON_ID = d.CON_ID
   AND CAST(d.DT_OPEN AS date)
       BETWEEN s.DT_FROM AND s.DT_TO

WHERE s.OUT_RUB <> 0

GROUP BY
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

OPTION (MAXDOP 2, RECOMPILE);

CREATE INDEX IX_balances_con
ON #balances (CON_ID);


/* ============================================================
   3. ИТОГОВАЯ ПОСДЕЛОЧНАЯ ВЫГРУЗКА

   Один SELECT.
   Дополнительные реквизиты — без DT_REP.

   Спред к КС — по формуле агрегированной вьюхи.
   ============================================================ */

SELECT
    CAST(b.DT_OPEN AS date) AS [Дата отчета],

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

FROM #balances b

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

ORDER BY
    b.DT_OPEN,
    b.CON_ID

OPTION (MAXDOP 2, RECOMPILE);

