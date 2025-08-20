USE [ALM_TEST];
GO
SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO

CREATE OR ALTER VIEW [WORK].[vDepositInterestsRate_FL_markets] AS
SELECT
    v.[DT_REP],
    v.[CON_ID],
    v.[CLI_ID],
    v.[INN],
    v.[CLI_SHORT_NAME],
    v.[IsDomRF],
    v.[ACC_NO],
    v.[PROD_TYPE],
    v.[DT_OPEN],
    v.[DT_CLOSE],
    v.[DT_CLOSE_PLAN],
    v.[MATUR],
    v.[RATE],
    v.[BALANCE_CUR],
    v.[CUR],
    v.[BALANCE_RUB],
    v.[CLI_SUBTYPE],
    v.[Group],
    v.[CONVENTION],
    v.[PROD_NAME],
    v.[IS_PDR],
    v.[IS_OPTION],
    v.[IS_FINANCE],
    v.[IS_FINANCE_LCR],
    v.[SEG_NAME],
    v.[LIQ_ФОР],
    v.[ALM_ФОР],
    v.[LIQ_ФОР_Fcast],
    v.[LIQ_ССВ],
    v.[ALM_ССВ],
    v.[LIQ_ССВ_Fcast],
    v.[ALM_LiquidityPrefRate],
    v.[ALM_OptionRate],
    v.[MonthlyCONV_RoisFix],
    v.[MonthlyCONV_LIQ_TransfertRate],
    v.[MonthlyCONV_ALM_TransfertRate],
    v.[MonthlyCONV_KBD],
    v.[MonthlyCONV_KRS],
    v.[MonthlyCONV_ForecastKeyRate],

    /* Быстрый пересчёт: (AtTheEnd_RATE + PERCRATE) -> monthly */
    CAST((
        SELECT [LIQUIDITY].[liq].[fnc_IntRate](
            v.[AtTheEnd_RATE] + ap.PERCRATE,  -- at-the-end + надбавка
            'at the end', 'monthly',          -- источник: at-the-end, цель: monthly
            v.[MATUR], 1
        )
    ) AS decimal(18,8)) AS [MonthlyCONV_RATE],

    v.[AtTheEnd_RoisFix],
    v.[AtTheEnd_LIQ_TransfertRate],
    v.[AtTheEnd_ALM_TransfertRate],
    v.[AtTheEnd_KBD],
    v.[AtTheEnd_KRS],
    v.[AtTheEnd_ForecastKeyRate],
    v.[AtTheEnd_RATE],
    v.[TSEGMENTNAME],
    v.[isfloat]
FROM [WORK].[vDepositInterestsRate_For_FL_only] v WITH (NOLOCK)
CROSS APPLY (
    SELECT TOP (1) p.PERCRATE
    FROM [ALM_TEST].[markets].[prod_term_rates] p WITH (NOLOCK)
    WHERE p.PROD_NAME = v.PROD_NAME
      AND v.[MATUR]   BETWEEN p.[TERMDAYS_FROM] AND p.[TERMDAYS_TO]
      AND v.[DT_OPEN] BETWEEN p.[DT_FROM]       AND p.[DT_TO]
    ORDER BY p.[DT_FROM] DESC, p.[DT_TO] DESC, p.[TERMDAYS_FROM] DESC
) ap
WHERE v.[AtTheEnd_RATE] IS NOT NULL;  -- чтобы UDF не получал NULL
GO
