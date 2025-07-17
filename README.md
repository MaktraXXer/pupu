Ниже — «цельный» скрипт-чекер: вставляется одним куском, никакие
дополнительные переменные объявлять не нужно.

* **Параметры** заданы в одном блоке (`@Anchor`, `@Prev`, `@HorizonTo`).
* Все сравнения с датами идут через `CAST ( … AS date)` — ошибок
  преобразования типов не будет.
* Поле `conv` и остальные ключевые колонки приводятся к одному
  `VARCHAR(50)` (если в справочниках другой размер — увеличьте).

```sql
/*────────────────────  ПАРАМЕТРЫ  ────────────────────*/
DECLARE
        @Anchor     date = '2025-07-15',          -- свежий DT_REP
        @Prev       date = DATEADD(day,-1,'2025-07-15'),
        @HorizonTo  date = '2025-08-31';

/*───────────────── 1. свежие фикс-депозиты 15-го ─────────────────*/
IF OBJECT_ID('tempdb..#fresh15') IS NOT NULL DROP TABLE #fresh15;

SELECT  bg.BALANCE_GROUP,
        tg.TERM_GROUP,
        t.PROD_NAME_RES,
        t.TSEGMENTNAME,
        conv = CAST(t.conv AS varchar(50)),
        t.out_rub,
        spread = t.rate_con - fk.AVG_KEY_RATE
INTO    #fresh15
FROM    ALM.ALM.vw_balance_rest_all      t  WITH (NOLOCK)
JOIN    ALM_TEST.WORK.ForecastKey_Cache  fk
           ON fk.DT_REP = CAST(t.dt_open AS date)
          AND fk.TERM   = t.termdays
LEFT JOIN WORK.man_BalanceGroup          bg
           ON t.out_rub BETWEEN bg.BALANCE_FROM AND bg.BALANCE_TO
LEFT JOIN WORK.man_TermGroup             tg
           ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE   t.dt_rep      = @Anchor
  AND   t.is_floatrate = 0
  AND   CAST(t.dt_open AS date) = @Anchor;     -- открыты 15-го

/*────────────── 2. резерв 14-го – только непокрытые ─────────────*/
IF OBJECT_ID('tempdb..#fresh14') IS NOT NULL DROP TABLE #fresh14;

SELECT  bg.BALANCE_GROUP,
        tg.TERM_GROUP,
        t.PROD_NAME_RES,
        t.TSEGMENTNAME,
        conv = CAST(t.conv AS varchar(50)),
        t.out_rub,
        spread = t.rate_con - fk.AVG_KEY_RATE
INTO    #fresh14
FROM    ALM.ALM.vw_balance_rest_all      t  WITH (NOLOCK)
JOIN    ALM_TEST.WORK.ForecastKey_Cache  fk
           ON fk.DT_REP = CAST(t.dt_open AS date)
          AND fk.TERM   = t.termdays
LEFT JOIN WORK.man_BalanceGroup          bg
           ON t.out_rub BETWEEN bg.BALANCE_FROM AND bg.BALANCE_TO
LEFT JOIN WORK.man_TermGroup             tg
           ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
LEFT JOIN #fresh15 f
       ON  f.BALANCE_GROUP  = bg.BALANCE_GROUP
       AND f.TERM_GROUP     = tg.TERM_GROUP
       AND f.PROD_NAME_RES  = t.PROD_NAME_RES
       AND f.TSEGMENTNAME   = t.TSEGMENTNAME
       AND f.conv           = CAST(t.conv AS varchar(50))
WHERE   t.dt_rep      = @Anchor
  AND   t.is_floatrate = 0
  AND   CAST(t.dt_open AS date) = @Prev
  AND   f.BALANCE_GROUP IS NULL;

/*────────────── 3. готовим «рыночные» средневзвешенные спреды ───*/
IF OBJECT_ID('tempdb..#mkt_spread') IS NOT NULL DROP TABLE #mkt_spread;
SELECT  BALANCE_GROUP , TERM_GROUP ,
        PROD_NAME_RES , TSEGMENTNAME , conv ,
        spread_mkt = SUM(out_rub*spread) / NULLIF(SUM(out_rub),0)
INTO    #mkt_spread
FROM   (SELECT * FROM #fresh15 UNION ALL SELECT * FROM #fresh14) u
GROUP  BY BALANCE_GROUP , TERM_GROUP , PROD_NAME_RES , TSEGMENTNAME , conv;

/*────────────── 4. фикс-депозиты, которые перевернутся ──────────*/
IF OBJECT_ID('tempdb..#roll_fix') IS NOT NULL DROP TABLE #roll_fix;
SELECT  t.con_id , t.out_rub , t.rate_con ,
        bg.BALANCE_GROUP , tg.TERM_GROUP ,
        t.PROD_NAME_RES , t.TSEGMENTNAME ,
        conv = CAST(t.conv AS varchar(50)),
        t.termdays , t.dt_open , t.dt_close
INTO    #roll_fix
FROM    ALM.ALM.vw_balance_rest_all      t  WITH (NOLOCK)
LEFT JOIN WORK.man_BalanceGroup          bg
           ON t.out_rub BETWEEN bg.BALANCE_FROM AND bg.BALANCE_TO
LEFT JOIN WORK.man_TermGroup             tg
           ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE   t.dt_rep      = @Anchor
  AND   t.is_floatrate = 0
  AND   CAST(t.dt_close AS date) <= @HorizonTo;

/*────────────── 5. матчинг и оценка покрытия ────────────────────*/
IF OBJECT_ID('tempdb..#roll_match') IS NOT NULL DROP TABLE #roll_match;
SELECT  r.* ,
        m.spread_mkt
INTO    #roll_match
FROM    #roll_fix r
LEFT JOIN #mkt_spread m
  ON  m.BALANCE_GROUP  = r.BALANCE_GROUP
  AND m.TERM_GROUP     = r.TERM_GROUP
  AND m.PROD_NAME_RES  = r.PROD_NAME_RES
  AND m.TSEGMENTNAME   = r.TSEGMENTNAME
  AND m.conv           = r.conv;

/*──────────────── общая картина ────────────────*/
SELECT  total_deals   = COUNT(*) ,
        covered_deals = COUNT(spread_mkt) ,
        pct_deals     = 100.0*COUNT(spread_mkt)/COUNT(*) ,
        total_rub     = SUM(out_rub) ,
        covered_rub   = SUM(CASE WHEN spread_mkt IS NULL THEN 0 ELSE out_rub END) ,
        pct_rub       = 100.0*SUM(CASE WHEN spread_mkt IS NULL THEN 0 ELSE out_rub END)
/                            / NULLIF(SUM(out_rub),0);

/*──────────── детализация по BALANCE_GROUP × TERM_GROUP ───────────*/
SELECT  BALANCE_GROUP , TERM_GROUP ,
        deals_tot = COUNT(*) ,
        deals_ok  = COUNT(spread_mkt) ,
        pct_deals = 100.0*COUNT(spread_mkt)/COUNT(*) ,
        rub_tot   = SUM(out_rub) ,
        rub_ok    = SUM(CASE WHEN spread_mkt IS NULL THEN 0 ELSE out_rub END) ,
        pct_rub   = 100.0*SUM(CASE WHEN spread_mkt IS NULL THEN 0 ELSE out_rub END)
/                             / NULLIF(SUM(out_rub),0)
FROM    #roll_match
GROUP  BY BALANCE_GROUP , TERM_GROUP
ORDER BY pct_rub DESC;
```

**Как пользоваться**

1. Скопируйте-вставьте скрипт в SSMS / Azure Data Studio.
2. Он сам:

   * создаёт/очищает все `#temp`-таблицы;
   * берёт сначала 15-е июля, потом недостающие комбинации 14-го;
   * считает средневзвешенный «рыночный» spread;
   * совпоставляет roll-over-фиксы;
   * выводит покрытие в двух `SELECT`-ах.

Не нужно ничего дополнительно объявлять, типы данных выровнены.
