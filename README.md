
USE ALM_TEST;
SET NOCOUNT ON;

DECLARE @Dt_start date = '2026-10-01';
DECLARE @Dt_end   date = '2026-10-31';

DECLARE @LastDtRep datetime2(7);

SELECT @LastDtRep = MAX(DT_REP)
FROM LIQUIDITY.liq.DepositInterestsRate WITH (NOLOCK);


/* ============================================================
   НОВЫЕ ПРИВЛЕЧЕНИЯ ЮЛ ЗА ОКТЯБРЬ 2026
   Только последняя DT_REP, только рабочие дни.
   ============================================================ */

;WITH contracts AS (
    SELECT
        d.CON_ID,
        d.CLI_ID,
        d.CLI_SHORT_NAME,
        d.CLI_SUBTYPE,
        d.SEG_NAME,
        d.PROD_TYPE,
        d.CONVENTION AS CONV,

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
)

SELECT
    CAST(d.DT_OPEN AS date) AS [Дата отчета],

    d.CON_ID,
    d.CLI_ID,
    d.CLI_SHORT_NAME,
    d.CLI_SUBTYPE,
    d.SEG_NAME,
    d.PROD_TYPE,
    d.CONV,

    d.ISDOMRF,
    d.IS_PDR,
    d.DT_OPEN,
    d.DT_CLOSE,
    d.MATUR,
    d.RATE,
    d.ISFLOAT AS IS_FLOAT,
    d.CUR,

    s.BALANCE_RUB,

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

FROM contracts d

CROSS APPLY (
    SELECT
        SUM(CAST(s.OUT_RUB AS decimal(28,2)))
            AS BALANCE_RUB
    FROM LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)
    WHERE s.CON_ID = d.CON_ID
      AND d.DT_OPEN BETWEEN s.DT_FROM AND s.DT_TO
      AND s.OUT_RUB <> 0
) s

WHERE s.BALANCE_RUB IS NOT NULL

ORDER BY
    d.DT_OPEN,
    d.CON_ID

OPTION (MAXDOP 2, RECOMPILE);
