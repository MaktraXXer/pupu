/* ---------------------------------------------------------------
   SUM-волюм по новым сделкам ЮЛ Крупн. + ЮЛ ССВ
   (TYPE = 'Начало', i = 2)
-----------------------------------------------------------------*/
DECLARE 
    @d_from date = '2025-01-09',
    @d_to   date = '2025-01-09';

DROP TABLE IF EXISTS #calendar;
SELECT [Date]
INTO   #calendar
FROM   ALM.info.VW_Calendar WITH (NOLOCK)
WHERE  [Date] BETWEEN @d_from AND @d_to;

SELECT
        dep.DT_OPEN                           AS [Date],          -- день открытия
        SUM(saldo.OUT_RUB)                    AS SUM_OUT_RUB      -- суммарный остаток
FROM    LIQUIDITY.liq.DepositInterestsRate    dep   WITH (NOLOCK)

JOIN    #calendar                             cal   ON cal.[Date] = dep.DT_OPEN   -- i = 2
JOIN    LIQUIDITY.liq.DepositContract_Saldo   saldo WITH (NOLOCK)
          ON  saldo.CON_ID = dep.CON_ID
          AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO

/* — те же справочники и диапазон спред-бакета, что и в исходе — */
LEFT JOIN LIQUIDITY.liq.man_TermGroup         term  WITH (NOLOCK)
          ON dep.MATUR BETWEEN term.TERM_FROM AND term.TERM_TO

LEFT JOIN LIQUIDITY.liq.man_Spread_Bucket     spr_buck WITH (NOLOCK)
          ON dep.MonthlyCONV_Rate + dep.LIQ_ССВ_Fcast + dep.ALM_OptionRate
               - dep.MonthlyCONV_ForecastKeyRate  >  spr_buck.Spread_From
         AND dep.MonthlyCONV_Rate + dep.LIQ_ССВ_Fcast + dep.ALM_OptionRate
               - dep.MonthlyCONV_ForecastKeyRate  <= spr_buck.Spread_To

WHERE  dep.DT_REP = (SELECT MAX(DT_REP)
                     FROM LIQUIDITY.liq.DepositInterestsRate)

  AND  dep.CUR                 = 'RUR'
  AND  dep.CLI_SUBTYPE         IN ('ЮЛ Крупн.','ЮЛ ССВ')
  AND  dep.IsDomRF             = 0
  AND  dep.RATE                > 0.01
  AND  ISNULL(dep.isfloat,0)   = 0
  AND  dep.LIQ_ФОР             IS NOT NULL
  AND  dep.MonthlyCONV_RATE    IS NOT NULL
  AND  dep.MonthlyCONV_RATE    BETWEEN dep.MonthlyCONV_ForecastKeyRate - 0.07
                                  AND dep.MonthlyCONV_ForecastKeyRate + 0.07
  AND  saldo.OUT_RUB           <> 0
  AND  dep.CLI_ID              <> 3731800
  AND  /* not-null transfer, как в обеих SP */
       CASE 
          WHEN dep.CLI_SUBTYPE <> 'ФЛ'
               THEN dep.MonthlyCONV_LIQ_TransfertRate
          ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                      dep.MonthlyCONV_LIQ_TransfertRate)
       END IS NOT NULL

GROUP BY dep.DT_OPEN           -- если нужен итог за все даты, уберите эту строку
OPTION (RECOMPILE);
