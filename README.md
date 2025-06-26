/* период, который сравниваем ---------------------------------------------------*/
DECLARE 
    @d_from date = '2025-05-01',
    @d_to   date = '2025-06-24';

/* календарь, чтобы не тянуть лишние даты ---------------------------------------*/
DROP TABLE IF EXISTS #calendar;
SELECT  [Date]
INTO    #calendar
FROM    ALM.info.VW_Calendar WITH (NOLOCK)
WHERE   [Date] BETWEEN @d_from AND @d_to;



/* --------------------------- 1. «Как у коллеги» (dtStartDeal) -----------------*/
DROP TABLE IF EXISTS #raw_dtstart;
SELECT  dep.CON_ID,
        dep.DT_OPEN              AS [Date],   -- точка «Начало»
        dep.CLI_SUBTYPE,
        saldo.OUT_RUB,
        'COL'                    AS SRC
INTO    #raw_dtstart
FROM    LIQUIDITY.liq.DepositInterestsRate      dep   WITH (NOLOCK)
JOIN    #calendar                               cal   ON cal.[Date] = dep.DT_OPEN
JOIN    LIQUIDITY.liq.DepositContract_Saldo     saldo WITH (NOLOCK)
           ON  saldo.CON_ID = dep.CON_ID
           AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO
WHERE   dep.DT_REP = (SELECT MAX(DT_REP) 
                      FROM LIQUIDITY.liq.DepositInterestsRate)
  AND   dep.CUR               = 'RUR'
  AND   dep.CLI_SUBTYPE       IN ('ЮЛ Крупн.','ЮЛ ССВ')
  AND   dep.IsDomRF           = 0
  AND   dep.RATE              > 0.01
  AND   ISNULL(dep.isfloat,0) = 0
  AND   dep.LIQ_ФОР           IS NOT NULL
  AND   dep.MonthlyCONV_RATE  IS NOT NULL
  AND   dep.MonthlyCONV_RATE  BETWEEN dep.MonthlyCONV_ForecastKeyRate - 0.07
                                  AND dep.MonthlyCONV_ForecastKeyRate + 0.07
  AND   saldo.OUT_RUB         <> 0
  AND   dep.CLI_ID            <> 3731800
  AND   -- «трансферт не NULL» — как в обеих SP
        CASE 
           WHEN dep.CLI_SUBTYPE <> 'ФЛ' THEN dep.MonthlyCONV_LIQ_TransfertRate
           ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                       dep.MonthlyCONV_LIQ_TransfertRate)
        END IS NOT NULL;



/* ------------------------------ 2. «Как у тебя» (UL_matur_pdr_fo) -------------*/
DROP TABLE IF EXISTS #raw_ul;
SELECT  dep.CON_ID,
        dep.DT_OPEN              AS [Date],
        dep.CLI_SUBTYPE,
        saldo.OUT_RUB,
        'YOU'                    AS SRC,
        dep.IS_PDR,
        dep.IS_FINANCE_LCR
INTO    #raw_ul
FROM    LIQUIDITY.liq.DepositInterestsRate      dep   WITH (NOLOCK)
JOIN    #calendar                               cal   ON cal.[Date] = dep.DT_OPEN
JOIN    LIQUIDITY.liq.DepositContract_Saldo     saldo WITH (NOLOCK)
           ON  saldo.CON_ID = dep.CON_ID
           AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO
WHERE   dep.DT_REP = (SELECT MAX(DT_REP) 
                      FROM LIQUIDITY.liq.DepositInterestsRate)
  AND   dep.CUR               = 'RUR'
  AND   dep.CLI_SUBTYPE       IN ('ЮЛ Крупн.','ЮЛ ССВ')
  AND   dep.IsDomRF           = 0
  AND   dep.RATE              > 0.01
  AND   ISNULL(dep.isfloat,0) = 0
  AND   dep.LIQ_ФОР           IS NOT NULL
  AND   dep.MonthlyCONV_RATE  IS NOT NULL
  AND   dep.MonthlyCONV_RATE  BETWEEN dep.MonthlyCONV_ForecastKeyRate - 0.07
                                  AND dep.MonthlyCONV_ForecastKeyRate + 0.07
  AND   saldo.OUT_RUB         <> 0
  AND   dep.CLI_ID            <> 3731800
  AND   -- тот же not-null по трансферту
        CASE 
           WHEN dep.CLI_SUBTYPE <> 'ФЛ' THEN dep.MonthlyCONV_LIQ_TransfertRate
           ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                       dep.MonthlyCONV_LIQ_TransfertRate)
        END IS NOT NULL;



/* ----------------------------- 3. Сводим объёмы по датам -----------------------*/
SELECT  d.[Date],
        ISNULL(SUM(d.OUT_RUB),0) AS bal_colleague,
        ISNULL(SUM(u.OUT_RUB),0) AS bal_yours,
        ISNULL(SUM(u.OUT_RUB),0) - ISNULL(SUM(d.OUT_RUB),0) AS diff
FROM    #calendar c
LEFT JOIN #raw_dtstart d ON d.[Date] = c.[Date]
LEFT JOIN #raw_ul      u ON u.[Date] = c.[Date]
GROUP  BY c.[Date]
ORDER  BY c.[Date];



/* ----------------------------- 4. Что есть только у коллег --------------------*/
SELECT * 
FROM   #raw_dtstart d
WHERE  NOT EXISTS (SELECT 1 FROM #raw_ul u
                   WHERE u.CON_ID = d.CON_ID
                     AND u.[Date] = d.[Date]);

/* ----------------------------- 5. Что есть только у тебя ----------------------*/
SELECT * 
FROM   #raw_ul u
WHERE  NOT EXISTS (SELECT 1 FROM #raw_dtstart d
                   WHERE d.CON_ID = u.CON_ID
                     AND d.[Date] = u.[Date]);
