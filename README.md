/* Параметры периода — возьмите ваш рабочий диапазон */
DECLARE @d_from date = '2025-05-01',
        @d_to   date = '2025-06-24';

/* -------------------- 1. Сырьё из вашей процедуры -------------------- */
DROP TABLE IF EXISTS #raw_you;
SELECT dep.CON_ID,
       dep.DT_OPEN                  AS dt,
       saldo.OUT_RUB,
       'YOU'                         AS src
INTO   #raw_you
FROM   LIQUIDITY.liq.DepositInterestsRate   dep
JOIN   LIQUIDITY.liq.DepositContract_Saldo  saldo
       ON  saldo.CON_ID = dep.CON_ID
       AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO
WHERE  dep.DT_REP = (SELECT MAX(DT_REP) FROM LIQUIDITY.liq.DepositInterestsRate)
  AND  dep.CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ')
  AND  dep.RATE > 0.01
  AND  dep.MonthlyCONV_RATE BETWEEN dep.MonthlyCONV_ForecastKeyRate-0.07
                                AND dep.MonthlyCONV_ForecastKeyRate+0.07
  AND  ISNULL(dep.isfloat,0)=0
  AND  dep.DT_OPEN BETWEEN @d_from AND @d_to;

/* -------------------- 2. Сырьё из процедуры коллег ------------------- */
DROP TABLE IF EXISTS #raw_colleague;
SELECT dep.CON_ID,
       dep.DT_OPEN                  AS dt,
       saldo.OUT_RUB,
       'COL'                         AS src
INTO   #raw_colleague
FROM   LIQUIDITY.liq.DepositInterestsRate   dep
JOIN   LIQUIDITY.liq.DepositContract_Saldo  saldo
       ON  saldo.CON_ID = dep.CON_ID
       AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO
WHERE  dep.DT_REP = (SELECT MAX(DT_REP) FROM LIQUIDITY.liq.DepositInterestsRate)
  AND  dep.CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ')
  AND  dep.RATE > 0.01
  AND  dep.MonthlyCONV_RATE BETWEEN dep.MonthlyCONV_ForecastKeyRate-0.07
                                AND dep.MonthlyCONV_ForecastKeyRate+0.07
  AND  ISNULL(dep.isfloat,0)=0
  AND  dep.DT_OPEN BETWEEN @d_from AND @d_to;

/* -------------------- 3. Сводка по объёмам --------------------------- */
SELECT src,
       SUM(OUT_RUB) AS balance_rub
FROM (
      SELECT * FROM #raw_you
      UNION ALL
      SELECT * FROM #raw_colleague
     ) x
GROUP BY src;

/* -------------------- 4. Найдём «лишние» или «недостающие» сделки ---- */
-- только у коллег
SELECT c.*
FROM   #raw_colleague c
WHERE  NOT EXISTS (SELECT 1 FROM #raw_you y
                   WHERE  y.CON_ID = c.CON_ID
                     AND  y.dt     = c.dt);

-- только у вас
SELECT y.*
FROM   #raw_you y
WHERE  NOT EXISTS (SELECT 1 FROM #raw_colleague c
                   WHERE  c.CON_ID = y.CON_ID
                     AND  c.dt     = y.dt);
