Ниже полный скрипт-«одной вставкой», реализующий новую каскадную
логику (только 15 июля как “рынок” + поиск вверх по бакетам, а для ДЧБО — дополнительный
fallback на “средний по категории без бакета”).

```sql
/*────────────────────  ПАРАМЕТРЫ  ───────────────────*/
DECLARE
    @Anchor    date = '2025-07-15',   -- фактический dt_rep
    @HorizonTo date = '2025-08-31';   -- закрытие ≤

/*────────── temp-cleanup ─────────*/
IF OBJECT_ID('tempdb..#fresh15')     IS NOT NULL DROP TABLE #fresh15;
IF OBJECT_ID('tempdb..#mkt')         IS NOT NULL DROP TABLE #mkt;
IF OBJECT_ID('tempdb..#mkt_any')     IS NOT NULL DROP TABLE #mkt_any;
IF OBJECT_ID('tempdb..#bucket_rank') IS NOT NULL DROP TABLE #bucket_rank;
IF OBJECT_ID('tempdb..#roll_fix')    IS NOT NULL DROP TABLE #roll_fix;
IF OBJECT_ID('tempdb..#match')       IS NOT NULL DROP TABLE #match;

/* ── 0. шкала бакетов (от малого к большому) ─────────────────────*/
CREATE TABLE #bucket_rank(bucket varchar(20) PRIMARY KEY, r int);
INSERT #bucket_rank VALUES
('[0-1.5 млн)',0),('[1.5-15 млн)',1),('[15-100 млн)',2),('[100 млн+]',3);

/* ── 1. «рынок» = фиксы, открытые 15-го ──────────────────────────*/
SELECT
        bucket =
             CASE WHEN t.out_rub < 1500000      THEN '[0-1.5 млн)'
                  WHEN t.out_rub < 15000000     THEN '[1.5-15 млн)'
                  WHEN t.out_rub < 100000000    THEN '[15-100 млн)'
                  ELSE                              '[100 млн+]'
             END,
        tg.TERM_GROUP,
        t.PROD_NAME_RES,
        t.TSEGMENTNAME,
        conv = CAST(t.conv AS varchar(50)),
        t.out_rub,
        spread = t.rate_con - fk.AVG_KEY_RATE
INTO    #fresh15
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
CROSS  APPLY (SELECT TRY_CAST(t.dt_open AS date) d_open) o
JOIN    ALM_TEST.WORK.ForecastKey_Cache fk
           ON fk.DT_REP = o.d_open
          AND fk.TERM   = t.termdays
LEFT JOIN WORK.man_TermGroup tg
           ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE   t.dt_rep       = @Anchor
  AND   t.section_name = N'Срочные'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.od_flag      = 1
  AND   t.cur          = '810'
  AND   t.is_floatrate = 0
  AND   t.out_rub      > 0
  AND   o.d_open       = @Anchor
  AND   o.d_open IS NOT NULL;

/* ── 2. рынок-спред по БАКЕТУ (точный) ───────────────────────────*/
SELECT
        bucket, TERM_GROUP,
        PROD_NAME_RES, TSEGMENTNAME, conv,
        spread_mkt = SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
INTO    #mkt
FROM    #fresh15
GROUP  BY bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv;

/* ── 3. рынок-спред «без-бакетный» (для ДЧБО-fallback) ───────────*/
SELECT
        TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv,
        spread_any = SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
INTO    #mkt_any
FROM    #fresh15
GROUP  BY TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv;

/* ── 4. фиксы, закрывающиеся ≤ 31-08 (roll-over) ─────────────────*/
SELECT
        r.con_id,
        r.out_rub,
        r.rate_con,
        bucket =
             CASE WHEN r.out_rub < 1500000      THEN '[0-1.5 млн)'
                  WHEN r.out_rub < 15000000     THEN '[1.5-15 млн)'
                  WHEN r.out_rub < 100000000    THEN '[15-100 млн)'
                  ELSE                              '[100 млн+]'
             END,
        tg.TERM_GROUP,
        r.PROD_NAME_RES,
        r.TSEGMENTNAME,
        conv = CAST(r.conv AS varchar(50))
INTO    #roll_fix
FROM    ALM.ALM.vw_balance_rest_all r  WITH (NOLOCK)
CROSS APPLY (SELECT TRY_CAST(r.dt_close AS date) d_close) c
LEFT  JOIN WORK.man_TermGroup tg
           ON r.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE   r.dt_rep       = @Anchor
  AND   r.section_name = N'Срочные'
  AND   r.block_name   = N'Привлечение ФЛ'
  AND   r.od_flag      = 1
  AND   r.cur          = '810'
  AND   r.is_floatrate = 0
  AND   r.out_rub      > 0
  AND   c.d_close      <= @HorizonTo
  AND   c.d_close IS NOT NULL;

/* ── 5. каскадный поиск спреда ───────────────────────────────────*/
SELECT  rf.*,
        spread_mkt =
        COALESCE
        (   /* 5-a. точный или ↑-бакетный поиск */
            (
             SELECT TOP 1 m.spread_mkt
             FROM   #mkt              m
             JOIN   #bucket_rank      b_m ON b_m.bucket = m.bucket
             JOIN   #bucket_rank      b_r ON b_r.bucket = rf.bucket
             WHERE  m.TERM_GROUP      = rf.TERM_GROUP
               AND  m.PROD_NAME_RES   = rf.PROD_NAME_RES
               AND  m.TSEGMENTNAME    = rf.TSEGMENTNAME
               AND  m.conv            = rf.conv
               AND  b_m.r             >= b_r.r          -- тот же или БОЛЬШИЙ bucket
             ORDER  BY b_m.r          -- closest larger
            ),

            /* 5-b. fallback для ДЧБО: без-бакетный */
            CASE WHEN rf.TSEGMENTNAME = N'ДЧБО'
                 THEN ( SELECT ma.spread_any
                        FROM   #mkt_any ma
                        WHERE  ma.TERM_GROUP    = rf.TERM_GROUP
                          AND  ma.PROD_NAME_RES = rf.PROD_NAME_RES
                          AND  ma.TSEGMENTNAME  = rf.TSEGMENTNAME
                          AND  ma.conv          = rf.conv )
                 END
        )
INTO    #match
FROM    #roll_fix rf;

/* ── 6. сводка покрытия ─────────────────────────────────────────*/
SELECT
    total_deals   = COUNT(*) ,
    covered_deals = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals     = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)/COUNT(*) ,
    total_rub     = SUM(out_rub) ,
    covered_rub   = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
    pct_rub       = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/                   NULLIF(SUM(out_rub),0);
                    
/* ── 7. детализация (пять мер + conv) ───────────────────────────*/
SELECT
    bucket,
    TERM_GROUP,
    PROD_NAME_RES,
    TSEGMENTNAME,
    conv,
    deals_tot = COUNT(*) ,
    deals_ok  = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)/COUNT(*) ,
    rub_tot   = SUM(out_rub) ,
    rub_ok    = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
    pct_rub   = 100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/                NULLIF(SUM(out_rub),0)
FROM  #match
GROUP BY
    bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv
ORDER BY pct_rub DESC;
```

---

## Как работает новая каскадная логика

| Этап                         | Кому применяется | Что делаем                                                                                                                              |
| ---------------------------- | ---------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **1.** точный ключ           | все              | ищем спред в своём объём-бакете                                                                                                         |
| **2.** «ищем выше»           | Retail и ДЧБО    | если нет — идём в ближайший **бóльший** бакет (0→1→2→3)                                                                                 |
| **3.** категорийный fallback | **только ДЧБО**  | если и там пусто → берём средневзвешенный спред 15-июля по тем же `TERM_GROUP × PROD_NAME_RES × conv`, но уже **без** деления на бакеты |

> Одним `CROSS APPLY` не обошлось: каскад оформлен через подзапрос-TOP 1
> (ищущий ближайший подходящий бакет) + `COALESCE()` с ДЧБО-fallback.

---

### Оценка результата

* **Гарантированно** не смотрим старые даты (только 15 июля).
* Retail получает дополнительное покрытие за счёт “шага вверх” по объёму.
  Если в 0-1.5 млн нет аналога — смотрим 1.5-15 млн и т.д.
* ДЧБО после такого шага, если всё ещё «дырка», берёт усреднённый
  спред 15-июля по своей категории — поэтому должна пропасть
  львиная доля крупных непокрытых депозитов.

Запустите скрипт и сравните новый `pct_rub`: он должен стать
существенно выше (ориентир — ≥ 90 % по объёму).
