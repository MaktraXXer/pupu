Ниже — дополнение к уже рабочему скрипту.
Меняем **только** шаг 4 + 5 и добавляем шаг 8, который выводит «полотно» по
переворачивающимся фиксам:

* `spread_fix` — фактический спред сделки
* `spread_mkt` — спред, найденный алгоритмом
* `spread_final` — что будет использовано в прогнозе
* `is_matched` = 1, если удалось сопоставить (используется `spread_mkt`)

---

```sql
/*────────────────────  4-NEW. roll-over фиксы  ──────────────────*/
IF OBJECT_ID('tempdb..#roll_fix') IS NOT NULL DROP TABLE #roll_fix;
CREATE TABLE #roll_fix
( con_id        bigint,
  out_rub       money,
  bucket        varchar(20),
  r             tinyint,
  TERM_GROUP    varchar(100),
  PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME  nvarchar(100),
  conv_norm     varchar(50),
  spread_fix    decimal(18,6) );           -- ← фактический спред сделки

INSERT #roll_fix (con_id,out_rub,bucket,r,
                  TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm,spread_fix)
SELECT  r.con_id,
        r.out_rub,
        b.bucket,
        b.r,
        tg.TERM_GROUP,
        r.PROD_NAME_RES,
        r.TSEGMENTNAME,
        CASE WHEN r.conv = 'AT_THE_END' THEN '1M'
             ELSE CAST(r.conv AS varchar(50)) END,
        /* фактический спред: ставка –- key-rate на дату открытия */
        r.rate_con - fk_open.AVG_KEY_RATE
FROM    ALM.ALM.vw_balance_rest_all      r  WITH (NOLOCK)
JOIN    #bucket_def                      b
           ON r.out_rub BETWEEN b.lo AND ISNULL(b.hi, r.out_rub)
LEFT JOIN WORK.man_TermGroup             tg
           ON r.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache fk_open        -- 🔑 на дату открытия
           ON fk_open.DT_REP = CAST(r.DT_OPEN AS date)
          AND fk_open.TERM   = r.termdays
CROSS  APPLY (SELECT TRY_CAST(r.DT_CLOSE AS date) d_close) c
WHERE   r.dt_rep       = @Anchor
  AND   r.section_name = N'Срочные'
  AND   r.block_name   = N'Привлечение ФЛ'
  AND   r.od_flag      = 1
  AND   r.cur          = '810'
  AND   r.is_floatrate = 0
  AND   r.out_rub      > 0
  AND   c.d_close      <= @HorizonTo
  AND   c.d_close IS NOT NULL;

/*────────────────────  5-NEW. каскадное сопоставление  ─────────*/
IF OBJECT_ID('tempdb..#match') IS NOT NULL DROP TABLE #match;
SELECT rf.*,
       /* найденный «рыночный» спред — или NULL, если ничего не подошло */
       spread_mkt = COALESCE
       (
         /* (а) тот же/крупнее бакет */
         ( SELECT TOP 1 m.spread_mkt
             FROM #mkt m
             JOIN #bucket_def b_m ON b_m.bucket = m.bucket
            WHERE m.TERM_GROUP     = rf.TERM_GROUP
              AND m.PROD_NAME_RES  = rf.PROD_NAME_RES
              AND m.TSEGMENTNAME   = rf.TSEGMENTNAME
              AND m.conv_norm      = rf.conv_norm
              AND b_m.r            >= rf.r
            ORDER BY b_m.r ),
         /* (б) fallback для ДЧБО */
         CASE WHEN rf.TSEGMENTNAME = N'ДЧБО'
              THEN ( SELECT ma.spread_any
                       FROM #mkt_any ma
                      WHERE ma.TERM_GROUP    = rf.TERM_GROUP
                        AND ma.PROD_NAME_RES = rf.PROD_NAME_RES
                        AND ma.TSEGMENTNAME  = rf.TSEGMENTNAME
                        AND ma.conv_norm     = rf.conv_norm )
         END
       ),
       /* итоговый спред и флаг */
       spread_final = COALESCE(
                        /* есть сопоставление? */ 
                        ( SELECT TOP 1 m.spread_mkt
                            FROM #mkt m
                            JOIN #bucket_def b_m ON b_m.bucket = m.bucket
                           WHERE m.TERM_GROUP     = rf.TERM_GROUP
                             AND m.PROD_NAME_RES  = rf.PROD_NAME_RES
                             AND m.TSEGMENTNAME   = rf.TSEGMENTNAME
                             AND m.conv_norm      = rf.conv_norm
                             AND b_m.r            >= rf.r
                           ORDER BY b_m.r ),
                        CASE WHEN rf.TSEGMENTNAME = N'ДЧБО'
                             THEN ( SELECT ma.spread_any
                                      FROM #mkt_any ma
                                     WHERE ma.TERM_GROUP    = rf.TERM_GROUP
                                       AND ma.PROD_NAME_RES = rf.PROD_NAME_RES
                                       AND ma.TSEGMENTNAME  = rf.TSEGMENTNAME
                                       AND ma.conv_norm     = rf.conv_norm )
                        END,
                        /* ничего не нашли → берём фактический */
                        rf.spread_fix ),
       is_matched = CASE WHEN
                         ( SELECT TOP 1 1
                           FROM #mkt m
                           JOIN #bucket_def b_m ON b_m.bucket = m.bucket
                          WHERE m.TERM_GROUP     = rf.TERM_GROUP
                            AND m.PROD_NAME_RES  = rf.PROD_NAME_RES
                            AND m.TSEGMENTNAME   = rf.TSEGMENTNAME
                            AND m.conv_norm      = rf.conv_norm
                            AND b_m.r            >= rf.r ) IS NOT NULL
                       OR ( rf.TSEGMENTNAME = N'ДЧБО'
                            AND EXISTS (
                                   SELECT 1
                                     FROM #mkt_any ma
                                    WHERE ma.TERM_GROUP    = rf.TERM_GROUP
                                      AND ma.PROD_NAME_RES = rf.PROD_NAME_RES
                                      AND ma.TSEGMENTNAME  = rf.TSEGMENTNAME
                                      AND ma.conv_norm     = rf.conv_norm ) )
                     THEN 1 ELSE 0 END
INTO  #match
FROM  #roll_fix rf;

/*────────────────────  6. сводка (как раньше)  ─────────────────*/
SELECT
    total_deals   = COUNT(*) ,
    covered_deals = SUM(is_matched) ,
    pct_deals     = 100.0 * SUM(is_matched) / COUNT(*) ,
    total_rub     = SUM(out_rub) ,
    covered_rub   = SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END) ,
    pct_rub       = 100.0 * SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END)
/                   NULLIF(SUM(out_rub),0)
FROM #match;

/*────────────────────  7. детализация (как раньше) ─────────────*/
SELECT bucket , TERM_GROUP , PROD_NAME_RES , TSEGMENTNAME , conv_norm ,
       deals_tot = COUNT(*) ,
       deals_ok  = SUM(is_matched) ,
       pct_deals = 100.0*SUM(is_matched)/COUNT(*) ,
       rub_tot   = SUM(out_rub) ,
       rub_ok    = SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END) ,
       pct_rub   = 100.0*SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END)
/                  NULLIF(SUM(out_rub),0)
FROM #match
GROUP BY bucket , TERM_GROUP , PROD_NAME_RES , TSEGMENTNAME , conv_norm
ORDER BY pct_rub DESC;

/*────────────────────  8. полотно по сделкам  ──────────────────*/
SELECT con_id ,
       out_rub ,
       bucket ,
       TERM_GROUP ,
       PROD_NAME_RES ,
       TSEGMENTNAME ,
       conv_norm ,
       spread_fix     ,       -- фактический
       spread_mkt     ,       -- найденный алгоритмом (NULL, если нет)
       spread_final   ,       -- что попадёт в расчёты
       is_matched     -- 1 = нашли; 0 = взяли фактический
FROM   #match
ORDER  BY is_matched , bucket , TERM_GROUP , PROD_NAME_RES;
```

**Как работает теперь**

| Шаг | Что добавили/исправили                                                                                                                                       |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 4   | в `#roll_fix` вычисляем `spread_fix` сразу при выборке из витрины.                                                                                           |
| 5   | в `#match` считаем<br>• `spread_mkt` — как раньше;<br>• `spread_final` = `COALESCE(spread_mkt, spread_fix)`;<br>• `is_matched` флаг успешного сопоставления. |
| 6-7 | сводка/деталка считают покрытие по `is_matched`.                                                                                                             |
| 8   | выдаём «полотно» — полный список roll-over-сделок с обоими спредами и флагом.                                                                                |

Скопируйте **только** блоки 4-8 вместо прежних — первую половину
скрипта (шаги 0-3) оставляем без изменений.
Выполняется одной вставкой, без ошибок имён, и сразу показывает
результаты по каждому переворачивающемуся депозиту.
