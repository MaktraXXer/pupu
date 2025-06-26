Вот минимальный «контроль-шот», который показывает:

1. **сколько физически денег лежит на новых сделках 9-го января** (сырой набор `raw_you`);
2. **какой объём на ту же дату попал в витрину** `test_UL_cube_full`
   на уровне агрегата
   *CLI\_SUBTYPE = 'ЮЛ',  все остальные измерения – `all …`,  TYPE = 'Начало'*;
3. **какие CON\_ID потерялись / дублировались**.

Код можно брать и гонять как есть – поменяешь только имя витрины, если
она у тебя другая.

```sql
/*------------------------------------------------------------------
   0. одна дата на тест
------------------------------------------------------------------*/
DECLARE @d date = '2025-01-09';

/*------------------------------------------------------------------
   1. календарь (не даём селекту соскользнуть на соседние дни)
------------------------------------------------------------------*/
DROP TABLE IF EXISTS #calendar;
SELECT [Date] INTO #calendar
FROM   ALM.info.VW_Calendar WITH (NOLOCK)
WHERE  [Date] = @d;

/*------------------------------------------------------------------
   2. POS-deal срез ― ровно то, что мы должны агрегировать
------------------------------------------------------------------*/
DROP TABLE IF EXISTS #raw_you;
SELECT  dep.CON_ID,
        dep.DT_OPEN       AS [Date],
        saldo.OUT_RUB     AS BAL_RUB          -- остаток “как есть”
INTO    #raw_you
FROM    LIQUIDITY.liq.DepositInterestsRate dep WITH (NOLOCK)
JOIN    #calendar  c           ON c.[Date] = dep.DT_OPEN          -- i = 2
JOIN    LIQUIDITY.liq.DepositContract_Saldo saldo WITH (NOLOCK)
           ON  saldo.CON_ID = dep.CON_ID
           AND dep.DT_OPEN BETWEEN saldo.DT_FROM AND saldo.DT_TO
WHERE   dep.DT_REP = (SELECT MAX(DT_REP)
                      FROM LIQUIDITY.liq.DepositInterestsRate)
  AND   dep.CUR        = 'RUR'
  AND   dep.CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ')
  AND   dep.IsDomRF    = 0
  AND   dep.RATE       > 0.01
  AND   ISNULL(dep.isfloat,0)=0
  AND   dep.LIQ_ФОР IS NOT NULL
  AND   dep.MonthlyCONV_RATE IS NOT NULL
  AND   dep.MonthlyCONV_RATE BETWEEN dep.MonthlyCONV_ForecastKeyRate-0.07
                                 AND dep.MonthlyCONV_ForecastKeyRate+0.07
  AND   saldo.OUT_RUB <> 0
  AND   dep.CLI_ID    <> 3731800
  AND   CASE
          WHEN dep.CLI_SUBTYPE<>'ФЛ' THEN dep.MonthlyCONV_LIQ_TransfertRate
          ELSE ISNULL(dep.MonthlyCONV_ALM_TransfertRate,
                      dep.MonthlyCONV_LIQ_TransfertRate)
        END IS NOT NULL;

/*------------------------------------------------------------------
   3. Агрегат из кубовой витрины  (одна строка на дату)
------------------------------------------------------------------*/
DROP TABLE IF EXISTS #raw_col;
SELECT  t.[Date],
        t.BALANCE_RUB AS BAL_RUB
INTO    #raw_col
FROM    ALM_TEST.WORK.test_UL_cube_full t    -- <-- имя своей витрины
WHERE   t.[TYPE]          = 'Начало'
  AND   t.CLI_SUBTYPE     = 'ЮЛ'             -- roll-up до уровня «все ЮЛ»
  AND   t.MARGIN_TYPE     = 'all margin'
  AND   t.TERM_GROUP      = 'all termgroup'
  AND   t.IS_OPTION       = 'all option_type'
  AND   t.SEG_NAME        = 'all segment'
  AND   t.IS_PDR          = 'all PDR'
  AND   t.IS_FINANCE_LCR  = 'all FINANCE_LCR'
  AND   t.[Date]          = @d;

/*------------------------------------------------------------------
   4. Сравниваем ПО ОБЪЁМУ
------------------------------------------------------------------*/
SELECT  @d                              AS [Date],
        (SELECT SUM(BAL_RUB) FROM #raw_you) AS bal_you,
        (SELECT SUM(BAL_RUB) FROM #raw_col) AS bal_cube,
        (SELECT SUM(BAL_RUB) FROM #raw_you)
      - (SELECT SUM(BAL_RUB) FROM #raw_col) AS diff;

/*------------------------------------------------------------------
   5. Что лишнее / что пропало
------------------------------------------------------------------*/
/* 5.1 есть в кубе, нет в pos-deal */
SELECT col.CON_ID, col.BAL_RUB
FROM (
      /* раскручиваем vitrine → CON_ID, чтобы понять, что там сидит   */
      SELECT  r.CON_ID,
              s.OUT_RUB BAL_RUB
      FROM    ALM_TEST.WORK.test_UL_cube_full t 
      JOIN    LIQUIDITY.liq.DepositInterestsRate r WITH (NOLOCK)
                 ON t.[Date]  = r.DT_OPEN
                AND r.CLI_SUBTYPE IN ('ЮЛ Крупн.','ЮЛ ССВ')
      JOIN    LIQUIDITY.liq.DepositContract_Saldo s WITH (NOLOCK)
                 ON r.CON_ID = s.CON_ID
                AND r.DT_OPEN BETWEEN s.DT_FROM AND s.DT_TO
      WHERE   t.[TYPE]         = 'Начало'
        AND   t.CLI_SUBTYPE    = 'ЮЛ'
        AND   t.[Date]         = @d
     ) col
WHERE NOT EXISTS (SELECT 1 FROM #raw_you y
                  WHERE y.CON_ID = col.CON_ID);

/* 5.2 есть в pos-deal, нет в кубе  */
SELECT y.*
FROM   #raw_you y
WHERE  NOT EXISTS (SELECT 1 FROM #raw_col c
                   WHERE c.[Date] = y.[Date]);  -- куб на этом уровне знает только дату
```

---

## Как пользоваться

* **Запусти** скрипт.
  В блоке 4 увидишь три числа:
  `bal_you` (как в POS-deal), `bal_cube` (что уехало в витрину) и `diff`.
* Если `diff = 0` → объём сошёлся; проблема где-то дальше (смешение TYPE, дубль в UNION и т.д.).
* Если `diff ≠ 0` →

  * смотри блок **5.1** – CON\_ID, которые «лишние» в витрине;
  * смотри блок **5.2** – CON\_ID, которые потерялись при агрегации.

Проверь у «потеряшек» самые частые ловушки:

| что проверить                                  | где смотреть                                      |
| ---------------------------------------------- | ------------------------------------------------- |
| в `DepositContract_Saldo` нет строки на 9 янв? | блок 5.2 → любой CON\_ID → saldo-таблица          |
| `MonthlyCONV_RATE` вылетел за ± 0.07 КС        | убери фильтр и смотри, исчезла ли сделка          |
| `RATE≤0.01` или `isfloat=1`                    | значения полей в `DepositInterestsRate`           |
| NULL-вращение (`IS_PDR`, `IS_FINANCE_LCR`)     | vitrine заменяет NULL → ‘all …’, в деталке — NULL |

После того, как выловишь конкретные CON\_ID, становится понятно, на каком именно фильтре они «умирают» или где идёт двойное суммирование.
