/*── 4. #roll_fix ── (оставляем как в предыдущем варианте) */

/*── 5. сопоставление + итоговый спред ─────────────────────────*/
IF OBJECT_ID('tempdb..#match') IS NOT NULL DROP TABLE #match;
CREATE TABLE #match
( con_id        bigint         NOT NULL,
  out_rub       money          NOT NULL,
  bucket        varchar(20)    NOT NULL,
  r             tinyint        NOT NULL,
  TERM_GROUP    varchar(100)   NULL,
  PROD_NAME_RES nvarchar(200)  NULL,
  TSEGMENTNAME  nvarchar(100)  NULL,
  conv_norm     varchar(50)    NOT NULL,
  spread_fix    decimal(18,6)  NULL,
  spread_mkt    decimal(18,6)  NULL,
  spread_final  decimal(18,6)  NULL,
  is_matched    bit            NOT NULL );

/* -- явный список => никакой путаницы со столбцами больше нет */
INSERT INTO #match
        (con_id,out_rub,bucket,r,TERM_GROUP,PROD_NAME_RES,
         TSEGMENTNAME,conv_norm,spread_fix,spread_mkt,
         spread_final,is_matched)
SELECT  rf.con_id,
        rf.out_rub,
        rf.bucket,
        rf.r,
        rf.TERM_GROUP,
        rf.PROD_NAME_RES,
        rf.TSEGMENTNAME,
        rf.conv_norm,
        rf.spread_fix,
        mkt.spread_mkt,
        COALESCE(mkt.spread_mkt,ma.spread_any,rf.spread_fix)       AS spread_final,
        CASE WHEN mkt.spread_mkt IS NOT NULL OR ma.spread_any IS NOT NULL
             THEN 1 ELSE 0 END                                     AS is_matched
FROM    #roll_fix rf
/* a) тот же / более крупный бакет */
OUTER  APPLY (
    SELECT TOP 1 m.spread_mkt
    FROM   #mkt m
    JOIN   #bucket_def b_m ON b_m.bucket=m.bucket AND b_m.r>=rf.r
    WHERE  m.TERM_GROUP    = rf.TERM_GROUP
      AND  m.PROD_NAME_RES = rf.PROD_NAME_RES
      AND  m.TSEGMENTNAME  = rf.TSEGMENTNAME
      AND  m.conv_norm     = rf.conv_norm
    ORDER  BY b_m.r )                        mkt
/* b) fallback только для ДЧБО */
OUTER  APPLY (
    SELECT ma.spread_any
    FROM   #mkt_any ma
    WHERE  rf.TSEGMENTNAME = N'ДЧБО'
      AND  ma.TERM_GROUP    = rf.TERM_GROUP
      AND  ma.PROD_NAME_RES = rf.PROD_NAME_RES
      AND  ma.TSEGMENTNAME  = rf.TSEGMENTNAME
      AND  ma.conv_norm     = rf.conv_norm)  ma;

/*── 6. сводка ──*/
SELECT  total_deals   = COUNT(*) ,
        covered_deals = SUM(is_matched) ,
        pct_deals     = 100.*SUM(is_matched)/COUNT(*) ,
        total_rub     = SUM(out_rub) ,
        covered_rub   = SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END) ,
        pct_rub       = 100.*SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END)
/                   NULLIF(SUM(out_rub),0)
FROM    #match;

/*── 7. детализация ──*/
SELECT  bucket , TERM_GROUP , PROD_NAME_RES , TSEGMENTNAME , conv_norm ,
        deals_tot = COUNT(*) ,
        deals_ok  = SUM(is_matched) ,
        pct_deals = 100.*SUM(is_matched)/COUNT(*) ,
        rub_tot   = SUM(out_rub) ,
        rub_ok    = SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END) ,
        pct_rub   = 100.*SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END)
/                   NULLIF(SUM(out_rub),0)
FROM    #match
GROUP BY bucket , TERM_GROUP , PROD_NAME_RES , TSEGMENTNAME , conv_norm
ORDER  BY pct_rub DESC;

/*── 8. полотно ──*/
SELECT  con_id, out_rub, bucket, TERM_GROUP, PROD_NAME_RES,
        TSEGMENTNAME, conv_norm,
        spread_fix, spread_mkt, spread_final, is_matched
FROM    #match
ORDER  BY is_matched, bucket, TERM_GROUP, PROD_NAME_RES;
