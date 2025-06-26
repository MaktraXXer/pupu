/*------------------------------------------------------------------
   0. параметры периода
------------------------------------------------------------------*/
DECLARE 
    @d_from date = '2025-05-01',
    @d_to   date = '2025-06-24';

/*------------------------------------------------------------------
   1. календарь, чтобы не тянуть лишние даты
------------------------------------------------------------------*/
DROP TABLE IF EXISTS #calendar;
SELECT [Date]
INTO   #calendar
FROM   ALM.info.VW_Calendar WITH (NOLOCK)
WHERE  [Date] BETWEEN @d_from AND @d_to;

/*------------------------------------------------------------------
   2. «Твоя» пос-сделочная выборка (i = 2  ⇒  TYPE = 'Начало')
------------------------------------------------------------------*/
DROP TABLE IF EXISTS #raw_you;

SELECT  dep.CON_ID,
        dep.DT_OPEN                  AS [Date],
        saldo.OUT_RUB                AS BAL_RUB
INTO    #raw_you
FROM    LIQUIDITY.liq.DepositInterestsRate dep WITH (NOLOCK)
JOIN    #calendar                cal   ON cal.[Date] = dep.DT_OPEN
JOIN    LIQUIDITY.liq.DepositContract_Saldo saldo WITH (NOLOCK)
           ON saldo.CON_ID = dep.CON_ID
           AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO
WHERE   dep.DT_REP   = (SELECT MAX(DT_REP) 
                        FROM LIQUIDITY.liq.DepositInterestsRate)
  AND   dep.CUR       = 'RUR'
  AND   dep.CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ')
  AND   dep.IsDomRF   = 0
  AND   dep.RATE      > 0.01
  AND   ISNULL(dep.isfloat,0) = 0
  AND   dep.LIQ_ФОР   IS NOT NULL
  AND   dep.MonthlyCONV_RATE IS NOT NULL
  AND   dep.MonthlyCONV_RATE BETWEEN dep.MonthlyCONV_ForecastKeyRate-0.07
                                 AND dep.MonthlyCONV_ForecastKeyRate+0.07
  AND   saldo.OUT_RUB <> 0
  AND   dep.CLI_ID    <> 3731800
  AND   /* not-null transfer, как в обоих SP */
        CASE WHEN dep.CLI_SUBTYPE<>'ФЛ'
             THEN dep.MonthlyCONV_LIQ_TransfertRate
             ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                         dep.MonthlyCONV_LIQ_TransfertRate)
        END IS NOT NULL;

/*------------------------------------------------------------------
   3. «Коллеги» / агрегатная витрина после твоей процедуры
       (TYPE='Начало',   агрегаты 'all …'  => CLI_SUBTYPE='ЮЛ')
------------------------------------------------------------------*/
DROP TABLE IF EXISTS #raw_col;

SELECT  t.[Date],
        t.BALANCE_RUB               AS BAL_RUB
INTO    #raw_col
FROM    ALM_TEST.WORK.test_UL_cube_full t      -- ← витрина, которую мы собрали
WHERE   t.[TYPE]        = 'Начало'
  AND   t.CLI_SUBTYPE   = 'ЮЛ'                 -- уровень агрегирования
  AND   t.MARGIN_TYPE   = 'all margin'
  AND   t.TERM_GROUP    = 'all termgroup'
  AND   t.IS_OPTION     = 'all option_type'
  AND   t.SEG_NAME      = 'all segment'
  AND   t.IS_PDR        = 'all PDR'
  AND   t.IS_FINANCE_LCR= 'all FINANCE_LCR'
  AND   t.[Date] BETWEEN @d_from AND @d_to;

/*------------------------------------------------------------------
   4. Сводка по датам: видно, где объёмы «разошлись»
------------------------------------------------------------------*/
SELECT  c.[Date],
        ISNULL(col.BAL_RUB,0)  AS bal_colleague,
        ISNULL(you.BAL_RUB,0)  AS bal_you,
        ISNULL(you.BAL_RUB,0) - ISNULL(col.BAL_RUB,0) AS diff
FROM    #calendar c
LEFT JOIN (SELECT [Date],SUM(BAL_RUB) BAL_RUB
           FROM #raw_you GROUP BY [Date]) you ON you.[Date]=c.[Date]
LEFT JOIN #raw_col col ON col.[Date]=c.[Date]
ORDER  BY c.[Date];

/*------------------------------------------------------------------
   5. Какие именно CON_ID «потерялись» / «лишние»
------------------------------------------------------------------*/
/* а) есть у коллег, нет у тебя */
SELECT col.CON_ID, col.[Date], col.BAL_RUB
FROM  (SELECT DISTINCT r.CON_ID, r.[Date], s.OUT_RUB BAL_RUB
       FROM ALM_TEST.WORK.test_UL_cube_full t
       JOIN LIQUIDITY.liq.DepositInterestsRate r WITH (NOLOCK)
              ON t.[Date]=r.DT_OPEN AND r.CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ')
       JOIN LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)
              ON r.CON_ID=s.CON_ID AND r.DT_OPEN BETWEEN s.DT_FROM AND s.DT_TO
       WHERE t.[TYPE]='Начало'
         AND t.CLI_SUBTYPE='ЮЛ'
         AND t.[Date] BETWEEN @d_from AND @d_to) col
WHERE NOT EXISTS (SELECT 1 FROM #raw_you y 
                  WHERE y.CON_ID=col.CON_ID AND y.[Date]=col.[Date]);

/* б) есть у тебя, нет у коллег */
SELECT y.*
FROM   #raw_you y
WHERE  NOT EXISTS (SELECT 1 
                   FROM #raw_col c 
                   WHERE c.[Date]=y.[Date]);   -- rolled-up, поэтому только дата
