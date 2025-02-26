SELECT 
    CASE WHEN i.[i] = 2 THEN dep.[DT_OPEN] END AS [Date],
    CASE WHEN i.[i] = 2 THEN 'Начало' END     AS [TYPE],
    dep.[CLI_SUBTYPE],
    ISNULL(dep.[MARGIN_TYPE], 'Прочий_тип')   AS [MARGIN_TYPE],
    term.[TERM_GROUP],
    CAST(dep.[IS_OPTION] AS VARCHAR(255))     AS [IS_OPTION],
    ISNULL(dep.[SEG_NAME], 'Прочий_сегмент')  AS [SEG_NAME],
    spr_buck.[Spread_Bucket]                  AS [Spread_KeyRate],
    saldo.[OUT_RUB]                           AS [BALANCE_RUB],
    dep.[LIQ_ФОР]                             AS [ФОР],
    dep.[LIQ_ССВ_Fcast]                       AS [ССВ],
    dep.[ALM_OptionRate]                      AS [ALM_OptionRate],
    CASE 
        WHEN dep.[MATUR] <= 365 
        THEN dep.[MonthlyCONV_RoisFix] 
        ELSE dep.[MonthlyCONV_KBD] 
    END                                       AS [MonthlyCONV_OIS],
    CASE 
        WHEN dep.[CLI_SUBTYPE] != 'ФЛ' THEN dep.[MonthlyCONV_LIQ_TransfertRate]
        ELSE ISNULL(dep.[MonthlyCONV_ALM_TransfertRate], dep.[MonthlyCONV_LIQ_TransfertRate])
    END                                       AS [MonthlyCONV_TransfertRate],
    CASE 
        WHEN dep.[CLI_SUBTYPE] != 'ФЛ' THEN 
            CASE 
                WHEN dep.[MonthlyCONV_LIQ_TransfertRate] 
                     - (dep.[MonthlyCONV_Rate] + dep.[LIQ_ССВ_Fcast]) >= 0 
                THEN dep.[MonthlyCONV_LIQ_TransfertRate]
                ELSE dep.[MonthlyCONV_Rate] + dep.[LIQ_ССВ_Fcast]
            END
        ELSE ISNULL(dep.[MonthlyCONV_ALM_TransfertRate], dep.[MonthlyCONV_LIQ_TransfertRate])
    END                                       AS [MonthlyCONV_TransfertRate_MOD],
    dep.[MonthlyCONV_ALM_TransfertRate],
    dep.[MonthlyCONV_KBD],
    ISNULL(dep.[MonthlyCONV_ForecastKeyRate], 0) AS [MonthlyCONV_ForecastKeyRate],
    dep.[MonthlyCONV_Rate]
FROM
(
    ----------------------------------------------------------------------
    -- Внутренний подзапрос (тот же, что в коде, где вы считаете MARGIN_TYPE и Spread_KeyRate)
    ----------------------------------------------------------------------
    SELECT 
         di.*
        ,CASE
            WHEN 
                (
                  CASE 
                    WHEN di.[CLI_SUBTYPE] != 'ФЛ' 
                    THEN di.[MonthlyCONV_LIQ_TransfertRate] 
                    ELSE di.[MonthlyCONV_ALM_TransfertRate] 
                  END
                ) 
                - (di.[MonthlyCONV_Rate] + di.[LIQ_ССВ_Fcast]) < 0 
              THEN 'MINUS'
            WHEN 
                (
                  CASE 
                    WHEN di.[CLI_SUBTYPE] != 'ФЛ' 
                    THEN di.[MonthlyCONV_LIQ_TransfertRate] 
                    ELSE di.[MonthlyCONV_ALM_TransfertRate] 
                  END
                )
                - (di.[MonthlyCONV_Rate] + di.[LIQ_ССВ_Fcast]) >= 0 
              THEN 'PLUS'
         END AS [MARGIN_TYPE],

         CASE
             WHEN di.[MATUR] <= 31  THEN 'SHORT MATUR'
             WHEN di.[MATUR] BETWEEN 32 AND 181 THEN 'MEDIUM MATUR'
             WHEN di.[MATUR] >= 182 THEN 'LONG MATUR'
         END AS [MATUR_TYPE],

         (di.[MonthlyCONV_Rate] 
          + di.[LIQ_ССВ_Fcast] 
          + di.[ALM_OptionRate] 
          - di.[MonthlyCONV_ForecastKeyRate]) AS [Spread_KeyRate]

    FROM [LIQUIDITY].[liq].[DepositInterestsRate] di WITH (NOLOCK)
    WHERE di.[DT_REP] = (
            SELECT MAX([DT_REP]) 
            FROM [LIQUIDITY].[liq].[DepositInterestsRate] WITH (NOLOCK)
          )
      AND CASE 
            WHEN di.[CLI_SUBTYPE] != 'ФЛ' 
            THEN di.[MonthlyCONV_LIQ_TransfertRate] 
            ELSE ISNULL(di.[MonthlyCONV_ALM_TransfertRate], di.[MonthlyCONV_LIQ_TransfertRate]) 
          END IS NOT NULL
      AND ISNULL(di.[isfloat], 0) = 0
) AS dep
----------------------------------------------------------------------
-- Искусственная таблица (select 2 as [i]) для "Начала"
----------------------------------------------------------------------
JOIN (SELECT 2 AS [i]) i ON 1 = 1

----------------------------------------------------------------------
-- Календарь, чтобы убедиться, что dep.[DT_OPEN] совпадает с нужной датой
-- В оригинале используется #calendar, но можно использовать общий VW_Calendar
----------------------------------------------------------------------
JOIN [ALM].[info].[VW_Calendar] cal 
    ON cal.[Date] = dep.[DT_OPEN]  
    -- если нужно ограничить период, можно было бы добавить
    -- AND cal.[Date] BETWEEN @dt_Start AND @dt_End

----------------------------------------------------------------------
-- Сальдо по сделкам
----------------------------------------------------------------------
JOIN [LIQUIDITY].[liq].[DepositContract_Saldo] saldo WITH (NOLOCK)
    ON dep.[CON_ID] = saldo.[CON_ID]
    AND dep.[DT_OPEN] BETWEEN saldo.[DT_FROM] AND saldo.[DT_TO]

----------------------------------------------------------------------
-- Группа сроков
----------------------------------------------------------------------
LEFT JOIN [LIQUIDITY].liq.[man_TermGroup] term WITH (NOLOCK)
    ON dep.[MATUR] BETWEEN term.[TERM_FROM] AND term.[TERM_TO]

----------------------------------------------------------------------
-- Корзина спредов
----------------------------------------------------------------------
LEFT JOIN [LIQUIDITY].[liq].[man_Spread_Bucket] spr_buck WITH (NOLOCK)
    ON dep.[Spread_KeyRate] > spr_buck.[Spread_From]
   AND dep.[Spread_KeyRate] <= spr_buck.[Spread_To]

----------------------------------------------------------------------
-- Фильтры в точности, как в оригинале
----------------------------------------------------------------------
WHERE 1=1
  AND i.[i] = 2                         -- только "Начало"
  AND dep.[CUR] = 'RUR'
  AND dep.[LIQ_ФОР] IS NOT NULL
  AND dep.[MonthlyCONV_RATE] IS NOT NULL
  AND dep.[IsDomRF] = 0
  AND dep.[RATE] > 0.01
  AND ISNULL(dep.[isfloat], 0) = 0      -- исключаем плавающую ставку
  AND dep.[MonthlyCONV_RATE] BETWEEN dep.[MonthlyCONV_ForecastKeyRate] - 0.07 
                                  AND dep.[MonthlyCONV_ForecastKeyRate] + 0.07
  AND saldo.[OUT_RUB] != 0
  AND dep.[CLI_ID] != 3731800           -- исключаем ФК
  AND cal.[Date] IN ('2025-01-09','2025-01-30','2025-01-31')  -- нужные даты
  AND dep.[CLI_SUBTYPE] IN ('ЮЛ Крупн.', 'ЮЛ ССВ')            -- нужный тип клиента
;
