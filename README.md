USE [ALM_TEST];
GO
DECLARE @asof date = '2025-07-02';

;WITH snap AS (
    SELECT  dep.*,
            -- Рассчёт модифицированного ТС на уровне сделки (как в процедуре)
            CASE 
                WHEN dep.[CLI_SUBTYPE] = N'ФЛ' THEN 
                    CASE 
                        WHEN dep.[MonthlyCONV_ALM_TransfertRate] - (dep.[MonthlyCONV_RATE] + dep.[LIQ_ССВ_Fcast]) >= 0 
                             THEN dep.[MonthlyCONV_ALM_TransfertRate]
                        ELSE dep.[MonthlyCONV_RATE] + dep.[LIQ_ССВ_Fcast] + 0.001
                    END
                ELSE dep.[MonthlyCONV_ALM_TransfertRate]
            END AS [MonthlyCONV_TransfertRate_MOD_calc]
    FROM [WORK].[snap_markets] dep WITH (NOLOCK)
    WHERE dep.[DT_REP] = (SELECT MAX(DT_REP) FROM [WORK].[DepositInterestsRateSnap] WITH (NOLOCK))
      AND dep.[DT_OPEN] = @asof                      -- новые сделки на дату
      AND dep.[CLI_SUBTYPE] = N'ФЛ'
      AND dep.[MonthlyCONV_ALM_TransfertRate] IS NOT NULL
      AND dep.[CUR] = 'RUR'
      AND dep.[LIQ_ФОР] IS NOT NULL
      AND dep.[MonthlyCONV_RATE] IS NOT NULL
      AND ISNULL(dep.[IsDomRF],0) = 0
      AND dep.[RATE] > 0.01
      AND dep.[MonthlyCONV_RATE] + dep.[ALM_OptionRate]*dep.[IS_OPTION]
            BETWEEN dep.[MonthlyCONV_ForecastKeyRate] - 0.07 
                AND dep.[MonthlyCONV_ForecastKeyRate] + 0.07
      AND ISNULL(dep.[isfloat],0) = 0
      AND dep.[CLI_ID] <> 3731800
      AND dep.[DT_OPEN] <> DATEADD(day,-1,dep.[DT_CLOSE_PLAN])
      AND dep.[MATUR] BETWEEN 91 AND 184            -- 3–6 месяцев
),
snap_with_saldo AS (
    SELECT s.*,
           sal.OUT_RUB
    FROM snap s
    JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] sal WITH (NOLOCK)
      ON sal.[CON_ID] = s.[CON_ID]
     AND @asof BETWEEN sal.[DT_FROM] AND sal.[DT_TO]
    WHERE sal.[OUT_RUB] <> 0
)
SELECT TOP (50)
       w.[CON_ID],
       w.[CLI_ID],
       w.[CLI_SHORT_NAME],
       w.[PROD_NAME],
       w.[DT_OPEN],
       w.[DT_CLOSE_PLAN],
       w.[MATUR],
       w.[OUT_RUB],

       -- Базовые составляющие
       w.[MonthlyCONV_RATE]                 AS [MonthlyConv_Rate],
       w.[LIQ_ССВ_Fcast]                    AS [SSV],
       w.[ALM_OptionRate]                   AS [OptionPrem],
       w.[MonthlyCONV_ALM_TransfertRate]    AS [MonthlyConv_ALM_TR],
       w.[MonthlyCONV_ForecastKeyRate]      AS [KeyRate_Monthly],
       w.[MonthlyCONV_TransfertRate_MOD_calc] AS [MonthlyConv_TR_MOD],

       -- Спреды на уровне сделки
       (w.[MonthlyCONV_TransfertRate_MOD_calc] - w.[MonthlyCONV_ForecastKeyRate]) AS [TS_minus_KS_spread],
       (w.[MonthlyCONV_RATE] + w.[LIQ_ССВ_Fcast] + w.[ALM_OptionRate] - w.[MonthlyCONV_ForecastKeyRate]) AS [ExtRate_minus_KS_spread],

       -- Вклад сделки в агрегат (для ранжирования драйверов)
       w.[OUT_RUB] * (w.[MonthlyCONV_TransfertRate_MOD_calc] - w.[MonthlyCONV_ForecastKeyRate]) AS [contrib_TS_KS],
       w.[OUT_RUB] * (w.[MonthlyCONV_RATE] + w.[LIQ_ССВ_Fcast] + w.[ALM_OptionRate] - w.[MonthlyCONV_ForecastKeyRate]) AS [contrib_Ext_KS],

       -- Диагностика "клемпа" ТС
       (w.[MonthlyCONV_ALM_TransfertRate] - (w.[MonthlyCONV_RATE] + w.[LIQ_ССВ_Fcast])) AS [TR_minus_ExtRateNoOpt],
       CASE WHEN (w.[MonthlyCONV_ALM_TransfertRate] - (w.[MonthlyCONV_RATE] + w.[LIQ_ССВ_Fcast])) < 0 THEN 1 ELSE 0 END AS [is_TS_clamped]
FROM snap_with_saldo w
ORDER BY [contrib_Ext_KS] DESC, [contrib_TS_KS] DESC;
