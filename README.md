DECLARE @dt_start date = '20260101';
DECLARE @dt_end   date = '20260503';

DROP TABLE IF EXISTS #keyrate;

SELECT 
      [date]
    , [rate] / 100.0 AS [rate]
INTO #keyrate
FROM ALM.info.VW_CBKEY_everyday
WHERE [Date] BETWEEN @dt_start AND @dt_end;


DROP TABLE IF EXISTS #ruonia;

SELECT 
      [date]
    , [rate] / 100.0 AS [rate]
INTO #ruonia
FROM ALM.info.VW_Ruonia_everyday
WHERE [Date] BETWEEN @dt_start AND @dt_end;


DROP TABLE IF EXISTS #calendar;

SELECT *
INTO #calendar
FROM ALM.info.[VW_calendar]
WHERE [Date] BETWEEN @dt_start AND @dt_end;


DROP TABLE IF EXISTS #deposit;

SELECT 
      cal.[Date]
    , dep.[CON_ID]
    , dep.[ACC_NO]
    , dep.[CLI_ID]
    , dep.[INN]

    , CASE
          WHEN dep.CLI_ID IN ('3731800', '3939590185') THEN 'Фед.Казна'
          ELSE dep.[CLI_SHORT_NAME]
      END AS [CLI_SHORT_NAME]

    -- старый баланс можно оставить только для сверки
    , dep.[BALANCE_CUR]
    , dep.[BALANCE_RUB] AS [DEP_BALANCE_RUB]

    -- основной вес для статистики — только сальдо на дату
    , saldo.[OUT_RUB] AS [BALANCE_RUB]

    , dep.[CUR]
    , dep.[DT_OPEN]
    , dep.[DT_CLOSE]
    , dep.[DT_CLOSE_PLAN]
    , dep.[MATUR]
    , DATEDIFF(day, cal.[Date], dep.[DT_CLOSE_PLAN]) AS [DURATION]
    , dep.[SEG_NAME]
    , dep.[PROD_NAME]
    , dep.[IS_PDR]

    , CASE 
          WHEN ISNULL(man.[basis], man2.[basis]) IS NULL THEN 0 
          ELSE 1 
      END AS [IS_FLOAT]

    , dep.[OKVED_CODE]
    , dep.[OKVED_NAME]
    , dep.[SSVRate_Fcast]

    , CASE     
          WHEN ISNULL(man.[basis], man2.[basis]) IS NOT NULL THEN 
              kr.[rate] / (1 - 0.0450) - kr.[rate]
          ELSE dep.[RFRate_Fcast_Int]
      END AS [FOR_Rate]

    , ISNULL(man.[basis], man2.[basis]) AS [BASIS]

    , CASE 
          WHEN ISNULL(man.[basis], man2.[basis]) IS NOT NULL THEN rn.[rate] 
      END AS [BASIS_RATE]

    , CASE 
          WHEN ISNULL(man.[basis], man2.[basis]) IS NOT NULL 
              THEN ISNULL(man.[con_spread], man2.correction) / 100.0 
      END AS [BASIS_SPREAD]

    , CASE     
          WHEN ISNULL(man.[basis], man2.[basis]) IS NOT NULL THEN 
              rn.[rate] 
              + ISNULL(man.[con_spread], man2.correction) / 100.0
              - (kr.[rate] / (1 - 0.0450) - kr.[rate])
          ELSE dep.[RATE]
      END AS [RATE]

    , CASE     
          WHEN ISNULL(man.[basis], man2.[basis]) IS NOT NULL THEN 
              LIQUIDITY.liq.fnc_IntRate(
                    rn.[rate] 
                    + ISNULL(man.[con_spread], man2.correction) / 100.0
                    - (kr.[rate] / (1 - 0.0450) - kr.[rate])
                  , 'at the end'
                  , 'monthly'
                  , dep.[MATUR]
                  , 1
              )
          ELSE dep.[MonthlyConv_RATE]
      END AS [MonthlyConv_Rate]

    , dep.[KeyRate]
    , dep.[MonthlyConv_OISRate]

    , CASE     
          WHEN ISNULL(man.[basis], man2.[basis]) IS NOT NULL THEN 
              LIQUIDITY.liq.fnc_IntRate(
                    rn.[rate] 
                    + ISNULL(man.[con_spread], man2.correction) / 100.0
                    - (kr.[rate] / (1 - 0.0450) - kr.[rate])
                  , 'at the end'
                  , 'monthly'
                  , dep.[MATUR]
                  , 1
              )
          ELSE dep.[MonthlyConv_RATE] + dep.[SSVRate_Fcast]
      END - dep.[KeyRate] AS [Spread_KeyRate]

    , CASE     
          WHEN ISNULL(man.[basis], man2.[basis]) IS NOT NULL THEN 
              LIQUIDITY.liq.fnc_IntRate(
                    rn.[rate] 
                    + ISNULL(man.[con_spread], man2.correction) / 100.0
                    - (kr.[rate] / (1 - 0.0450) - kr.[rate])
                  , 'at the end'
                  , 'monthly'
                  , dep.[MATUR]
                  , 1
              )
          ELSE dep.[MonthlyConv_RATE] + dep.[SSVRate_Fcast]
      END - dep.[MonthlyConv_OISRate] AS [Spread_OIS]

    , CASE
          WHEN 
              dep.TransfertRate * (1 - dep.[RFRate_Controlling]) 
              + ISNULL(lr.[Value], 0)
              - (
                    CASE     
                        WHEN ISNULL(man.[basis], man2.[basis]) IS NOT NULL THEN 
                            rn.[rate] 
                            + ISNULL(man.[con_spread], man2.correction) / 100.0
                            - (kr.[rate] / (1 - 0.0450) - kr.[rate])
                        ELSE dep.[RATE]
                    END
                    + dep.SSVRate_Fcast
                ) <= 0 
          THEN 0 
          ELSE 
              dep.TransfertRate * (1 - dep.[RFRate_Controlling]) 
              + ISNULL(lr.[Value], 0)
              - (
                    CASE     
                        WHEN ISNULL(man.[basis], man2.[basis]) IS NOT NULL THEN 
                            rn.[rate] 
                            + ISNULL(man.[con_spread], man2.correction) / 100.0
                            - (kr.[rate] / (1 - 0.0450) - kr.[rate])
                        ELSE dep.[RATE]
                    END
                    + dep.SSVRate_Fcast
                )
      END AS [Margin]

    , CASE
          WHEN intr.[RFRate_TRF] IS NOT NULL THEN 
              intr.[BaseRate_TRF] 
              + ISNULL(intr.[LiquidityPrefRate_TRF], 0) 
              + intr.[RFRate_TRF] 
              + ISNULL(intr.OptionRate_TRF, 0) 
              + COALESCE(intr.[liqrate_TRF], lr.[Value], 0)
          ELSE 
              (intr.[BaseRate_TRF] + ISNULL(intr.[LiquidityPrefRate_TRF], 0)) 
              * (1 - dep.[RFRate_Controlling]) 
              + ISNULL(intr.OptionRate_TRF, 0) 
              + COALESCE(intr.[liqrate_TRF], lr.[Value], 0)
      END AS [TransfertRate_Controlling]

    , dep.TransfertRate * (1 - dep.[RFRate_Controlling]) 
      + ISNULL(lr.[Value], 0) AS [TransfertRate]

    , CASE
          WHEN 
              CASE
                  WHEN intr.[RFRate_TRF] IS NOT NULL THEN 
                      intr.[BaseRate_TRF] 
                      + ISNULL(intr.[LiquidityPrefRate_TRF], 0) 
                      + intr.[RFRate_TRF] 
                      + ISNULL(intr.OptionRate_TRF, 0) 
                      + COALESCE(intr.[liqrate_TRF], lr.[Value], 0)
                  ELSE 
                      (intr.[BaseRate_TRF] + ISNULL(intr.[LiquidityPrefRate_TRF], 0)) 
                      * (1 - dep.[RFRate_Controlling]) 
                      + ISNULL(intr.OptionRate_TRF, 0) 
                      + COALESCE(intr.[liqrate_TRF], lr.[Value], 0)
              END
              - (
                    CASE     
                        WHEN ISNULL(man.[basis], man2.[basis]) IS NOT NULL THEN 
                            rn.[rate] 
                            + ISNULL(man.[con_spread], man2.correction) / 100.0
                            - (kr.[rate] / (1 - 0.0450) - kr.[rate])
                        ELSE dep.[RATE]
                    END
                    + dep.SSVRate_Fcast
                ) <= 0 
          THEN 0 
          ELSE
              CASE
                  WHEN intr.[RFRate_TRF] IS NOT NULL THEN 
                      intr.[BaseRate_TRF] 
                      + ISNULL(intr.[LiquidityPrefRate_TRF], 0) 
                      + intr.[RFRate_TRF] 
                      + ISNULL(intr.OptionRate_TRF, 0) 
                      + COALESCE(intr.[liqrate_TRF], lr.[Value], 0)
                  ELSE 
                      (intr.[BaseRate_TRF] + ISNULL(intr.[LiquidityPrefRate_TRF], 0)) 
                      * (1 - dep.[RFRate_Controlling]) 
                      + ISNULL(intr.OptionRate_TRF, 0) 
                      + COALESCE(intr.[liqrate_TRF], lr.[Value], 0)
              END
              - (
                    CASE     
                        WHEN ISNULL(man.[basis], man2.[basis]) IS NOT NULL THEN 
                            rn.[rate] 
                            + ISNULL(man.[con_spread], man2.correction) / 100.0
                            - (kr.[rate] / (1 - 0.0450) - kr.[rate])
                        ELSE dep.[RATE]
                    END
                    + dep.SSVRate_Fcast
                )
      END AS [Margin_Controlling]

INTO #deposit

FROM [LIQUIDITY].rep.DepositContract_LEGAL_InterestsRate dep WITH (NOLOCK)

    JOIN #calendar cal
        ON dep.[DT_OPEN] <= cal.[Date]
       AND dep.[DT_CLOSE] > cal.[Date]

    -- ключевая замена: берём остаток по договору из сальдо на конкретную дату календаря
    JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] saldo WITH (NOLOCK)
        ON dep.[CON_ID] = saldo.[CON_ID]
       AND cal.[Date] BETWEEN saldo.[DT_FROM] AND saldo.[DT_TO]

    LEFT JOIN LIQUIDITY.liq.floatrate_deals man 
        ON dep.CON_ID = CAST(man.[con_id] AS varchar(255))

    LEFT JOIN LIQUIDITY.liq.man_FloatContracts man2 
        ON CAST(man2.[con_id] AS varchar(255)) = dep.[con_id]

    LEFT JOIN #keyrate kr 
        ON cal.[Date] = kr.[date]

    LEFT JOIN #ruonia rn 
        ON cal.[Date] = rn.[date]

    LEFT JOIN alm.[info].[VW_liquidity_rates_interpolated] lr WITH (NOLOCK)
        ON dep.DT_OPEN BETWEEN lr.[dt_from] AND lr.[dt_to]
       AND lr.[Cur] = 810
       AND dep.[CUR] = 'RUR'
       AND ISNULL(dep.[IS_PDR], 0) = lr.[IS_PDR]
       AND ISNULL(dep.[IS_FINANCE_LCR], 0) = lr.[IS_FINANCE_LCR]
       AND dep.[MATUR] = lr.[Term]

    LEFT JOIN LIQUIDITY.liq.InterestsRateForDeposit intr WITH (NOLOCK)
        ON dep.[CON_ID] = intr.[CON_ID]

WHERE dep.DT_CLOSE > @dt_start
  AND dep.ISDOMRF = 0
  AND dep.[CUR] = 'RUR'
  AND saldo.[OUT_RUB] IS NOT NULL
  AND saldo.[OUT_RUB] <> 0;
