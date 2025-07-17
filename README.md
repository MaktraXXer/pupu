/*────────── 4-NEW. roll-over + fact-spread ──────────*/
IF OBJECT_ID('tempdb..#roll_fix') IS NOT NULL DROP TABLE #roll_fix;
CREATE TABLE #roll_fix
( con_id        bigint       NOT NULL,
  out_rub       money        NOT NULL,
  bucket        varchar(20)  NOT NULL,
  r             tinyint      NOT NULL,
  TERM_GROUP    varchar(100) NULL,
  PROD_NAME_RES nvarchar(200) NULL,
  TSEGMENTNAME  nvarchar(100) NULL,
  conv_norm     varchar(50)  NOT NULL,
  spread_fix    decimal(18,6) NULL );

INSERT INTO #roll_fix
        (con_id,out_rub,bucket,r,TERM_GROUP,PROD_NAME_RES,
         TSEGMENTNAME,conv_norm,spread_fix)
SELECT  r.con_id,
        r.out_rub,
        b.bucket,
        b.r,
        tg.TERM_GROUP,
        r.PROD_NAME_RES,
        r.TSEGMENTNAME,
        CASE WHEN r.conv='AT_THE_END' THEN '1M'
             ELSE CAST(r.conv AS varchar(50)) END,
        r.rate_con - fk_open.AVG_KEY_RATE
FROM    ALM.ALM.vw_balance_rest_all r  WITH (NOLOCK)
JOIN    #bucket_def              b ON r.out_rub BETWEEN b.lo AND ISNULL(b.hi,r.out_rub)
LEFT    JOIN WORK.man_TermGroup  tg ON r.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
LEFT    JOIN ALM_TEST.WORK.ForecastKey_Cache fk_open
           ON fk_open.DT_REP = CAST(r.DT_OPEN AS date)
          AND fk_open.TERM   = r.termdays
CROSS   APPLY (SELECT TRY_CAST(r.DT_CLOSE AS date) d_close) c
WHERE   r.dt_rep = @Anchor
  AND   r.section_name = N'Срочные'
  AND   r.block_name   = N'Привлечение ФЛ'
  AND   r.od_flag      = 1
  AND   r.cur          = '810'
  AND   r.is_floatrate = 0
  AND   r.out_rub      > 0
  AND   c.d_close      <= @HorizonTo
  AND   c.d_close      IS NOT NULL;

/*────────── 5-NEW. сопоставление / итоговый спред ──────────*/
IF OBJECT_ID('tempdb..#match') IS NOT NULL DROP TABLE #match;
CREATE TABLE #match
( con_id        bigint,
  out_rub       money,
  bucket        varchar(20),
  r             tinyint,
  TERM_GROUP    varchar(100),
  PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME  nvarchar(100),
  conv_norm     varchar(50),
  spread_fix    decimal(18,6),
  spread_mkt    decimal(18,6),
  spread_final  decimal(18,6),
  is_matched    bit );

INSERT INTO #match
SELECT  rf.*,
        mkt.spread_mkt,
        COALESCE(mkt.spread_mkt, ma.spread_any, rf.spread_fix),
        IIF(mkt.spread_mkt IS NOT NULL OR ma.spread_any IS NOT NULL,1,0)
FROM    #roll_fix rf
OUTER  APPLY (   /* a) тот же / более крупный бакет */
    SELECT TOP 1 m.spread_mkt
    FROM   #mkt m
    JOIN   #bucket_def b_m ON b_m.bucket = m.bucket AND b_m.r >= rf.r
    WHERE  m.TERM_GROUP     = rf.TERM_GROUP
      AND  m.PROD_NAME_RES  = rf.PROD_NAME_RES
      AND  m.TSEGMENTNAME   = rf.TSEGMENTNAME
      AND  m.conv_norm      = rf.conv_norm
    ORDER  BY b_m.r )         mkt
OUTER  APPLY (   /* b) fallback only for ДЧБО */
    SELECT ma.spread_any
    FROM   #mkt_any ma
    WHERE  rf.TSEGMENTNAME   = N'ДЧБО'
      AND  ma.TERM_GROUP     = rf.TERM_GROUP
      AND  ma.PROD_NAME_RES  = rf.PROD_NAME_RES
      AND  ma.TSEGMENTNAME   = rf.TSEGMENTNAME
      AND  ma.conv_norm      = rf.conv_norm)  ma;

/*────────── 6. сводка ──────────*/
SELECT  total_deals   = COUNT(*) ,
        covered_deals = SUM(is_matched) ,
        pct_deals     = 100.*SUM(is_matched)/COUNT(*) ,
        total_rub     = SUM(out_rub) ,
        covered_rub   = SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END) ,
        pct_rub       = 100.*SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END)
/                   NULLIF(SUM(out_rub),0)
FROM    #match;

/*────────── 7. детализация ──────────*/
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

/*────────── 8. «полотно» ──────────*/
SELECT  con_id,
        out_rub,
        bucket,
        TERM_GROUP,
        PROD_NAME_RES,
        TSEGMENTNAME,
        conv_norm,
        spread_fix,
        spread_mkt,
        spread_final,
        is_matched
FROM    #match
ORDER BY is_matched, bucket, TERM_GROUP, PROD_NAME_RES;
