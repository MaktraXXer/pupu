/*──────────────  4. roll-over фиксы (+ фактический спред)  ─────────────*/
IF OBJECT_ID('tempdb..#roll_fix') IS NOT NULL DROP TABLE #roll_fix;

SELECT
        r.con_id,
        r.out_rub,
        b.bucket,
        b.r,
        tg.TERM_GROUP,
        r.PROD_NAME_RES,
        r.TSEGMENTNAME,
        conv_norm = CASE WHEN r.conv = 'AT_THE_END' THEN '1M'
                         ELSE CAST(r.conv AS varchar(50)) END,
        -- фактический спред = ставка - key-rate на дату открытия
        spread_fix = r.rate_con - fk_open.AVG_KEY_RATE
INTO    #roll_fix
FROM    ALM.ALM.vw_balance_rest_all       r  WITH (NOLOCK)
JOIN    #bucket_def                       b
           ON r.out_rub BETWEEN b.lo AND ISNULL(b.hi,r.out_rub)
LEFT JOIN WORK.man_TermGroup              tg
           ON r.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache fk_open     -- ключ на дату открытия
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

/*──────────────  5. каскадное сопоставление + итоговый спред  ──────────*/
IF OBJECT_ID('tempdb..#match') IS NOT NULL DROP TABLE #match;

SELECT rf.* ,
       /* найденный «рыночный» спред (точный бакет → вверх) */
       spread_mkt =
           COALESCE(
               (SELECT TOP 1 m.spread_mkt
                  FROM #mkt m
                  JOIN #bucket_def b_m ON b_m.bucket = m.bucket
                 WHERE m.TERM_GROUP     = rf.TERM_GROUP
                   AND m.PROD_NAME_RES  = rf.PROD_NAME_RES
                   AND m.TSEGMENTNAME   = rf.TSEGMENTNAME
                   AND m.conv_norm      = rf.conv_norm
                   AND b_m.r            >= rf.r
                 ORDER BY b_m.r),
               CASE WHEN rf.TSEGMENTNAME = N'ДЧБО'
                    THEN (SELECT ma.spread_any
                            FROM #mkt_any ma
                           WHERE ma.TERM_GROUP    = rf.TERM_GROUP
                             AND ma.PROD_NAME_RES = rf.PROD_NAME_RES
                             AND ma.TSEGMENTNAME  = rf.TSEGMENTNAME
                             AND ma.conv_norm     = rf.conv_norm)
               END),
       /* что пойдёт в прогноз: mkt-спред или, если нет, фактический */
       spread_final = COALESCE(
                          (SELECT TOP 1 m.spread_mkt
                             FROM #mkt m
                             JOIN #bucket_def b_m ON b_m.bucket = m.bucket
                            WHERE m.TERM_GROUP     = rf.TERM_GROUP
                              AND m.PROD_NAME_RES  = rf.PROD_NAME_RES
                              AND m.TSEGMENTNAME   = rf.TSEGMENTNAME
                              AND m.conv_norm      = rf.conv_norm
                              AND b_m.r            >= rf.r
                            ORDER BY b_m.r),
                          CASE WHEN rf.TSEGMENTNAME = N'ДЧБО'
                               THEN (SELECT ma.spread_any
                                       FROM #mkt_any ma
                                      WHERE ma.TERM_GROUP    = rf.TERM_GROUP
                                        AND ma.PROD_NAME_RES = rf.PROD_NAME_RES
                                        AND ma.TSEGMENTNAME  = rf.TSEGMENTNAME
                                        AND ma.conv_norm     = rf.conv_norm)
                          END,
                          rf.spread_fix),
       /* флаг успешного сопоставления */
       is_matched = CASE WHEN
                        EXISTS (
                           SELECT 1
                             FROM #mkt m
                             JOIN #bucket_def b_m ON b_m.bucket = m.bucket
                            WHERE m.TERM_GROUP     = rf.TERM_GROUP
                              AND m.PROD_NAME_RES  = rf.PROD_NAME_RES
                              AND m.TSEGMENTNAME   = rf.TSEGMENTNAME
                              AND m.conv_norm      = rf.conv_norm
                              AND b_m.r            >= rf.r)
                     OR ( rf.TSEGMENTNAME = N'ДЧБО'
                          AND EXISTS (
                             SELECT 1
                               FROM #mkt_any ma
                              WHERE ma.TERM_GROUP    = rf.TERM_GROUP
                                AND ma.PROD_NAME_RES = rf.PROD_NAME_RES
                                AND ma.TSEGMENTNAME  = rf.TSEGMENTNAME
                                AND ma.conv_norm     = rf.conv_norm) )
                     THEN 1 ELSE 0 END
INTO   #match
FROM   #roll_fix rf;

/*──────────────  6. сводка покрытия  ───────────────────────────*/
SELECT
    total_deals   = COUNT(*) ,
    covered_deals = SUM(is_matched) ,
    pct_deals     = 100.0 * SUM(is_matched) / COUNT(*) ,
    total_rub     = SUM(out_rub) ,
    covered_rub   = SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END) ,
    pct_rub       = 100.0 *
                    SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END)
/                   NULLIF(SUM(out_rub),0)
FROM #match;

/*──────────────  7. детализация  ───────────────────────────────*/
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

/*──────────────  8. «полотно» по roll-over-сделкам  ────────────*/
SELECT con_id ,
       out_rub ,
       bucket ,
       TERM_GROUP ,
       PROD_NAME_RES ,
       TSEGMENTNAME ,
       conv_norm ,
       spread_fix ,      -- фактический спред сделки
       spread_mkt ,      -- найденный «рыночный» спред (NULL, если нет)
       spread_final ,    -- что уйдёт в прогноз
       is_matched        -- 1 = сопоставлено, 0 = взят фактический
FROM   #match
ORDER  BY is_matched , bucket , TERM_GROUP , PROD_NAME_RES;
