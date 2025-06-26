/* ---------------------------------------------------------------
   POS-DEAL (CON_ID-LEVEL) ВЫБОРКА «НОВЫХ» ДЕПОЗИТОВ ЮЛ КРУПН./ССВ
   ─ опирается строго на логику коллеги-процедуры
   ─ TYPE = 'Начало'  (i = 2)
   ─ CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ')
   ─ отдаёт все расчётные поля + CON_ID без группировки
-----------------------------------------------------------------*/

DECLARE 
    @d_from date = '2025-05-01',       -- диапазон, который хотите смотреть
    @d_to   date = '2025-06-24';

/* ─── календарь, чтобы отсечь лишние даты */
DROP TABLE IF EXISTS #calendar;
SELECT  [Date]
INTO    #calendar
FROM    ALM.info.VW_Calendar  WITH (NOLOCK)
WHERE   [Date] BETWEEN @d_from AND @d_to;


/* ─── итоговый запрос (i = 2 «Начало») */
SELECT
        dep.CON_ID,
        dep.DT_OPEN                              AS [Date],      -- «камень» открытия
        'Начало'                                 AS [TYPE],
        dep.CLI_SUBTYPE,
        ISNULL(dep.MARGIN_TYPE,'Прочий_тип')     AS MARGIN_TYPE,
        term.TERM_GROUP,
        CAST(dep.IS_OPTION AS varchar(255))      AS IS_OPTION,
        ISNULL(dep.SEG_NAME,'Прочий_сегмент')    AS SEG_NAME,

        /* bucket для спреда к ключевой ставке */
        spr_buck.Spread_Bucket                   AS Spread_KeyRate,

        /* абсолютное сальдо на дату открытия */
        saldo.OUT_RUB                            AS BALANCE_RUB,

        /* все «веса» в первозданном виде, как в логике коллеги */
        dep.LIQ_ФОР                              AS ФОР,
        dep.LIQ_ССВ_Fcast                        AS ССВ,
        dep.ALM_OptionRate                       AS ALM_OptionRate,

        CASE WHEN dep.MATUR <= 365
             THEN dep.MonthlyCONV_RoisFix
             ELSE dep.MonthlyCONV_KBD
        END                                      AS MonthlyCONV_OIS,

        /* базовый трансферт */
        CASE 
            WHEN dep.CLI_SUBTYPE <> 'ФЛ' 
                 THEN dep.MonthlyCONV_LIQ_TransfertRate
            ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                       dep.MonthlyCONV_LIQ_TransfertRate)
        END                                      AS MonthlyCONV_TransfertRate,

        /* скорректированный трансферт */
        CASE 
            WHEN dep.CLI_SUBTYPE <> 'ФЛ' THEN 
                CASE WHEN dep.MonthlyCONV_LIQ_TransfertRate 
                          - (dep.MonthlyCONV_Rate + dep.LIQ_ССВ_Fcast) >= 0
                     THEN dep.MonthlyCONV_LIQ_TransfertRate
                     ELSE dep.MonthlyCONV_Rate + dep.LIQ_ССВ_Fcast
                END
            ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                       dep.MonthlyCONV_LIQ_TransfertRate)
        END                                      AS MonthlyCONV_TransfertRate_MOD,

        dep.MonthlyCONV_ALM_TransfertRate,
        dep.MonthlyCONV_KBD,
        dep.MonthlyCONV_ForecastKeyRate,
        dep.MonthlyCONV_Rate
FROM    LIQUIDITY.liq.DepositInterestsRate      dep   WITH (NOLOCK)

JOIN    #calendar                               cal   ON cal.[Date] = dep.DT_OPEN   -- i = 2
JOIN    LIQUIDITY.liq.DepositContract_Saldo     saldo WITH (NOLOCK)
           ON  saldo.CON_ID = dep.CON_ID
           AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO

LEFT JOIN LIQUIDITY.liq.man_TermGroup           term  WITH (NOLOCK)
           ON  dep.MATUR BETWEEN term.TERM_FROM AND term.TERM_TO

LEFT JOIN LIQUIDITY.liq.man_Spread_Bucket       spr_buck WITH (NOLOCK)
           ON  dep.MonthlyCONV_Rate + dep.LIQ_ССВ_Fcast + dep.ALM_OptionRate 
               - dep.MonthlyCONV_ForecastKeyRate  >  spr_buck.Spread_From
           AND dep.MonthlyCONV_Rate + dep.LIQ_ССВ_Fcast + dep.ALM_OptionRate 
               - dep.MonthlyCONV_ForecastKeyRate  <= spr_buck.Spread_To

WHERE   dep.DT_REP = (SELECT MAX(DT_REP) 
                      FROM LIQUIDITY.liq.DepositInterestsRate)

  AND   dep.CUR                 = 'RUR'
  AND   dep.CLI_SUBTYPE         IN ('ЮЛ Крупн.','ЮЛ ССВ')
  AND   dep.IsDomRF             = 0
  AND   dep.RATE                > 0.01
  AND   ISNULL(dep.isfloat,0)   = 0
  AND   dep.LIQ_ФОР             IS NOT NULL
  AND   dep.MonthlyCONV_RATE    IS NOT NULL
  AND   dep.MonthlyCONV_RATE    BETWEEN dep.MonthlyCONV_ForecastKeyRate - 0.07
                                   AND dep.MonthlyCONV_ForecastKeyRate + 0.07
  AND   saldo.OUT_RUB           <> 0
  AND   dep.CLI_ID              <> 3731800
  /* обязательный not-null по трансферту (как в обеих SP) */
  AND   CASE 
           WHEN dep.CLI_SUBTYPE <> 'ФЛ' 
                THEN dep.MonthlyCONV_LIQ_TransfertRate
           ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                       dep.MonthlyCONV_LIQ_TransfertRate)
        END IS NOT NULL

OPTION (RECOMPILE);   -- убирает план-кеш, полезно при отладке
