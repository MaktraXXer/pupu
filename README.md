Два шага — одним скриптом.

* **cte\_raw\_new** — тот самый «i = 2 / Начало» запрос, только сразу суммируем `OUT_RUB`, чтобы получить «детальный» баланс.
* **cte\_proc\_base** — точно та же выборка, что в процедуре *prc\_test\_UL\_cube\_full* внутри блока «Начало» (все фильтры идентичны), тоже без всякой группировки по измерениям.
* потом:

1. Сравниваем суммы;
2. показываем «потеряшки» — где `CON_ID` есть только в одной из выборок.

```sql
/* -----------------------------------------------------------------
   Сверка «деталь» vs «что идёт в агрегацию»    (дата = 09-01-2025)
------------------------------------------------------------------*/
DECLARE @d date = '2025-01-09';      -- одна дата – удобно для отладки

/* ───── календарь для “i = 2 / Начало” */
DROP TABLE IF EXISTS #calendar;
SELECT [Date] INTO #calendar
FROM   ALM.info.VW_Calendar WITH (NOLOCK)
WHERE  [Date] = @d;

/*------------------------------------------------------------------
  1.  Детальная выборка (то, чем вы уже сверяли – i = 2)
------------------------------------------------------------------*/
;WITH cte_raw_new AS (
    SELECT  dep.CON_ID,
            dep.DT_OPEN            AS [Date],
            saldo.OUT_RUB
    FROM    LIQUIDITY.liq.DepositInterestsRate dep  WITH (NOLOCK)
    JOIN    #calendar               c   ON c.[Date] = dep.DT_OPEN          -- i = 2
    JOIN    LIQUIDITY.liq.DepositContract_Saldo  saldo WITH (NOLOCK)
               ON  saldo.CON_ID = dep.CON_ID
               AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO
    /* --- абсолютно те же фильтры --- */
    WHERE   dep.DT_REP = (SELECT MAX(DT_REP)
                          FROM LIQUIDITY.liq.DepositInterestsRate)
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
          END IS NOT NULL
)
/*------------------------------------------------------------------
  2.  То, что процедура берёт в агрегирование для «Начало»
      (тот же набор фильтров, те же поля)
------------------------------------------------------------------*/
, cte_proc_base AS (
    SELECT  dep.CON_ID,
            dep.DT_OPEN            AS [Date],
            saldo.OUT_RUB
    FROM    LIQUIDITY.liq.DepositInterestsRate dep  WITH (NOLOCK)
    JOIN    #calendar               c   ON c.[Date] = dep.DT_OPEN          -- i = 2
    JOIN    LIQUIDITY.liq.DepositContract_Saldo  saldo WITH (NOLOCK)
               ON  saldo.CON_ID = dep.CON_ID
               AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO
    /* --- фильтры 1-в-1, как в блоке “Начало” процедуры --- */
    WHERE   dep.DT_REP = (SELECT MAX(DT_REP)
                          FROM LIQUIDITY.liq.DepositInterestsRate)
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
          END IS NOT NULL
)

/*------------------------------------------------------------------
  3.  Сравнение суммарного баланса
------------------------------------------------------------------*/
SELECT  n.[Date],
        SUM(n.OUT_RUB)  AS balance_raw_new,
        SUM(p.OUT_RUB)  AS balance_proc_base,
        SUM(p.OUT_RUB) - SUM(n.OUT_RUB) AS diff
FROM    cte_raw_new  n
FULL JOIN cte_proc_base p ON p.CON_ID = n.CON_ID   -- FULL для честной суммы
GROUP  BY n.[Date];

/*------------------------------------------------------------------
  4a.  Какие сделки есть ТОЛЬКО в «детали»
------------------------------------------------------------------*/
SELECT n.CON_ID, n.OUT_RUB
FROM   cte_raw_new n
LEFT   JOIN cte_proc_base p ON p.CON_ID = n.CON_ID
WHERE  p.CON_ID IS NULL;

/*------------------------------------------------------------------
  4b.  Какие сделки есть ТОЛЬКО в выборке процедуры
------------------------------------------------------------------*/
SELECT p.CON_ID, p.OUT_RUB
FROM   cte_proc_base p
LEFT   JOIN cte_raw_new n ON n.CON_ID = p.CON_ID
WHERE  n.CON_ID IS NULL;
```

**Как использовать**

1. Запустите этот скрипт — первая выборка покажет, совпадают ли балансы за 9 января.
2. Две последние выборки сразу укажут, какие `CON_ID` «теряются» (если разница не нулевая) и где именно: в детальной выборке или в наборе, который идёт дальше в агрегирование/CUBE процедуры.
