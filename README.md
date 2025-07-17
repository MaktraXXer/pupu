Ошибка «varchar → datetime» означает, что в `vw_balance_rest_all`
поля `dt_open` / `dt_close` хранятся строкой и часть строк
невозможно привести к `date`.  Делать `CAST()` «в лоб» нельзя —
нужно использовать **`TRY_CAST`** и отфильтровать проблемные записи
(`TRY_CAST(.. AS date) IS NOT NULL`).

Ниже полный скрипт-чекер с этой защитой. Копируйте целиком — он
сам создаёт/чистит все `#temp`-таблицы и выводит охват.

```sql
/* ───────── ПАРАМЕТРЫ ───────── */
DECLARE
    @Anchor     date = '2025-07-15',           -- свежий DT_REP
    @Prev       date = '2025-07-14',           -- резерв
    @HorizonTo  date = '2025-08-31';           -- конец roll-over

/* util: безопасно приводим дату */
DECLARE @fmt varchar(10) = 'yyyy-MM-dd';       -- для читаемости

/* ───── 1. фикс-депозиты, открытые 15-го ───── */
IF OBJECT_ID('tempdb..#fresh15') IS NOT NULL DROP TABLE #fresh15;

SELECT  bg.BALANCE_GROUP ,
        tg.TERM_GROUP ,
        t.PROD_NAME_RES ,
        t.TSEGMENTNAME ,
        conv = CAST(t.conv AS varchar(50)) ,
        t.out_rub ,
        spread = t.rate_con - fk.AVG_KEY_RATE
INTO    #fresh15
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
CROSS  APPLY (SELECT TRY_CAST(t.dt_open AS date) AS d_open) o
JOIN    ALM_TEST.WORK.ForecastKey_Cache fk
           ON fk.DT_REP = o.d_open
          AND fk.TERM   = t.termdays
LEFT JOIN WORK.man_BalanceGroup bg
           ON t.out_rub BETWEEN bg.BALANCE_FROM AND bg.BALANCE_TO
LEFT JOIN WORK.man_TermGroup    tg
           ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE   t.dt_rep        = @Anchor
  AND   t.is_floatrate  = 0
  AND   o.d_open        = @Anchor        -- только 15-е
  AND   o.d_open IS NOT NULL;

/* ───── 2. фикс-депозиты, открытые 14-го, если нет пары ───── */
IF OBJECT_ID('tempdb..#fresh14') IS NOT NULL DROP TABLE #fresh14;

SELECT  bg.BALANCE_GROUP ,
        tg.TERM_GROUP ,
        t.PROD_NAME_RES ,
        t.TSEGMENTNAME ,
        conv = CAST(t.conv AS varchar(50)) ,
        t.out_rub ,
        spread = t.rate_con - fk.AVG_KEY_RATE
INTO    #fresh14
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
CROSS  APPLY (SELECT TRY_CAST(t.dt_open AS date) AS d_open) o
JOIN    ALM_TEST.WORK.ForecastKey_Cache fk
           ON fk.DT_REP = o.d_open
          AND fk.TERM   = t.termdays
LEFT JOIN WORK.man_BalanceGroup bg
           ON t.out_rub BETWEEN bg.BALANCE_FROM AND bg.BALANCE_TO
LEFT JOIN WORK.man_TermGroup    tg
           ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
LEFT JOIN #fresh15 f
       ON  f.BALANCE_GROUP  = bg.BALANCE_GROUP
       AND f.TERM_GROUP     = tg.TERM_GROUP
       AND f.PROD_NAME_RES  = t.PROD_NAME_RES
       AND f.TSEGMENTNAME   = t.TSEGMENTNAME
       AND f.conv           = CAST(t.conv AS varchar(50))
WHERE   t.dt_rep        = @Anchor
  AND   t.is_floatrate  = 0
  AND   o.d_open        = @Prev
  AND   o.d_open IS NOT NULL
  AND   f.BALANCE_GROUP IS NULL;         -- нет аналога из 15-го

/* ───── 3. рыночные средневзвешенные спреды ───── */
IF OBJECT_ID('tempdb..#mkt_spread') IS NOT NULL DROP TABLE #mkt_spread;

SELECT  BALANCE_GROUP , TERM_GROUP ,
        PROD_NAME_RES , TSEGMENTNAME , conv ,
        spread_mkt = SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
INTO    #mkt_spread
FROM   (SELECT * FROM #fresh15
        UNION ALL
        SELECT * FROM #fresh14) s
GROUP BY BALANCE_GROUP , TERM_GROUP ,
         PROD_NAME_RES , TSEGMENTNAME , conv;

/* ───── 4. фиксы, закрывающиеся до 31-08 ───── */
IF OBJECT_ID('tempdb..#roll_fix') IS NOT NULL DROP TABLE #roll_fix;

SELECT  t.con_id , t.out_rub , t.rate_con ,
        bg.BALANCE_GROUP , tg.TERM_GROUP ,
        t.PROD_NAME_RES , t.TSEGMENTNAME ,
        conv = CAST(t.conv AS varchar(50)) ,
        t.termdays
INTO    #roll_fix
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
CROSS  APPLY (SELECT TRY_CAST(t.dt_close AS date) AS d_close) c
LEFT JOIN WORK.man_BalanceGroup bg
           ON t.out_rub BETWEEN bg.BALANCE_FROM AND bg.BALANCE_TO
LEFT JOIN WORK.man_TermGroup    tg
           ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE   t.dt_rep       = @Anchor
  AND   t.is_floatrate = 0
  AND   c.d_close      <= @HorizonTo
  AND   c.d_close IS NOT NULL;

/* ───── 5. матч + покрытие ───── */
IF OBJECT_ID('tempdb..#roll_match') IS NOT NULL DROP TABLE #roll_match;

SELECT r.* , m.spread_mkt
INTO   #roll_match
FROM   #roll_fix    r
LEFT  JOIN #mkt_spread m
         ON m.BALANCE_GROUP = r.BALANCE_GROUP
        AND m.TERM_GROUP    = r.TERM_GROUP
        AND m.PROD_NAME_RES = r.PROD_NAME_RES
        AND m.TSEGMENTNAME  = r.TSEGMENTNAME
        AND m.conv          = r.conv;

/* 6. общие цифры */
SELECT
    total_deals   = COUNT(*) ,
    covered_deals = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals     = 100.0*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/                    COUNT(*) ,
    total_rub     = SUM(out_rub) ,
    covered_rub   = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
    pct_rub       = 100.0*SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/                    NULLIF(SUM(out_rub),0)
FROM #roll_match;

/* 7. детализация по BALANCE_GROUP × TERM_GROUP */
SELECT
    BALANCE_GROUP , TERM_GROUP ,
    deals_tot = COUNT(*) ,
    deals_ok  = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals = 100.0*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/                COUNT(*) ,
    rub_tot   = SUM(out_rub) ,
    rub_ok    = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
    pct_rub   = 100.0*SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/                NULLIF(SUM(out_rub),0)
FROM #roll_match
GROUP BY BALANCE_GROUP , TERM_GROUP
ORDER BY pct_rub DESC;
```

**Что изменил**

* Везде `TRY_CAST(... AS date)` вместо `CAST` — строки-«мусор» просто
  отфильтруются, ошибок конвертации не будет.
* Фильтры `… IS NOT NULL` гарантируют, что в расчёт попадают только
  корректные даты.
* Все арифметики без лишних символов, названия колонок коррек­тны.

Запускайте скрипт одним куском — он покажет,
какую долю договоров/объёма удаётся «покрыть» рыночным спредом.
