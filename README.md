/* ---------------------------------------------------------------
   Сверка «деталь» vs «что идёт в агрегацию»  (дата = 09-01-2025)
   — без CTE, только временные таблицы
-----------------------------------------------------------------*/
DECLARE @d date = '2025-01-09';   -- дата-для-отладки

/* ───── календарь для “i = 2 / Начало” */
DROP TABLE IF EXISTS #calendar;
SELECT [Date]
INTO   #calendar
FROM   ALM.info.VW_Calendar WITH (NOLOCK)
WHERE  [Date] = @d;

/*----------------------------------------------------------------
  1.  #raw_new  — пос-сделочная выборка (то, чем вы сверялись)
----------------------------------------------------------------*/
DROP TABLE IF EXISTS #raw_new;
SELECT  dep.CON_ID,
        dep.DT_OPEN            AS [Date],
        saldo.OUT_RUB
INTO    #raw_new
FROM    LIQUIDITY.liq.DepositInterestsRate dep  WITH (NOLOCK)
JOIN    #calendar                  c   ON c.[Date] = dep.DT_OPEN          -- i = 2
JOIN    LIQUIDITY.liq.DepositContract_Saldo saldo WITH (NOLOCK)
           ON  saldo.CON_ID = dep.CON_ID
           AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO
WHERE   dep.DT_REP = (SELECT MAX(DT_REP) FROM LIQUIDITY.liq.DepositInterestsRate)
  AND   dep.CUR              = 'RUR'
  AND   dep.CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ')
  AND   dep.IsDomRF          = 0
  AND   dep.RATE             > 0.01
  AND   ISNULL(dep.isfloat,0)= 0
  AND   dep.LIQ_ФОР IS NOT NULL
  AND   dep.MonthlyCONV_RATE IS NOT NULL
  AND   dep.MonthlyCONV_RATE BETWEEN dep.MonthlyCONV_ForecastKeyRate-0.07
                                AND dep.MonthlyCONV_ForecastKeyRate+0.07
  AND   saldo.OUT_RUB <> 0
  AND   dep.CLI_ID    <> 3731800
  AND   CASE WHEN dep.CLI_SUBTYPE<>'ФЛ'
             THEN dep.MonthlyCONV_LIQ_TransfertRate
             ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                         dep.MonthlyCONV_LIQ_TransfertRate)
      END IS NOT NULL;

/*----------------------------------------------------------------
  2.  #proc_base — что забирает процедура в блоке «Начало»
----------------------------------------------------------------*/
DROP TABLE IF EXISTS #proc_base;
SELECT  dep.CON_ID,
        dep.DT_OPEN            AS [Date],
        saldo.OUT_RUB
INTO    #proc_base
FROM    LIQUIDITY.liq.DepositInterestsRate dep  WITH (NOLOCK)
JOIN    #calendar                  c   ON c.[Date] = dep.DT_OPEN          -- i = 2
JOIN    LIQUIDITY.liq.DepositContract_Saldo saldo WITH (NOLOCK)
           ON  saldo.CON_ID = dep.CON_ID
           AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO
WHERE   dep.DT_REP = (SELECT MAX(DT_REP) FROM LIQUIDITY.liq.DepositInterestsRate)
  AND   dep.CUR              = 'RUR'
  AND   dep.CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ')
  AND   dep.IsDomRF          = 0
  AND   dep.RATE             > 0.01
  AND   ISNULL(dep.isfloat,0)= 0
  AND   dep.LIQ_ФОР IS NOT NULL
  AND   dep.MonthlyCONV_RATE IS NOT NULL
  AND   dep.MonthlyCONV_RATE BETWEEN dep.MonthlyCONV_ForecastKeyRate-0.07
                                AND dep.MonthlyCONV_ForecastKeyRate+0.07
  AND   saldo.OUT_RUB <> 0
  AND   dep.CLI_ID    <> 3731800
  AND   CASE WHEN dep.CLI_SUBTYPE<>'ФЛ'
             THEN dep.MonthlyCONV_LIQ_TransfertRate
             ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                         dep.MonthlyCONV_LIQ_TransfertRate)
      END IS NOT NULL;

/*----------------------------------------------------------------
  3.  Сравнение суммарного баланса
----------------------------------------------------------------*/
SELECT  COALESCE(r.[Date],p.[Date])          AS [Date],
        SUM(r.OUT_RUB)                       AS balance_raw_new,
        SUM(p.OUT_RUB)                       AS balance_proc_base,
        SUM(p.OUT_RUB)-SUM(r.OUT_RUB)        AS diff
FROM    #raw_new   r
FULL JOIN #proc_base p ON p.CON_ID = r.CON_ID
GROUP  BY COALESCE(r.[Date],p.[Date]);

/*----------------------------------------------------------------
  4a.  Сделки, которые есть ТОЛЬКО в деталях (#raw_new)
----------------------------------------------------------------*/
SELECT r.CON_ID,
       r.OUT_RUB
FROM   #raw_new r
LEFT   JOIN #proc_base p ON p.CON_ID = r.CON_ID
WHERE  p.CON_ID IS NULL;

/*----------------------------------------------------------------
  4b.  Сделки, которые есть ТОЛЬКО в выборке процедуры (#proc_base)
----------------------------------------------------------------*/
SELECT p.CON_ID,
       p.OUT_RUB
FROM   #proc_base p
LEFT   JOIN #raw_new r ON r.CON_ID = p.CON_ID
WHERE  r.CON_ID IS NULL;
