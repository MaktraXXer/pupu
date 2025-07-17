/*────────────────────────  0. параметры  ───────────────────────*/
DECLARE
    @Anchor    date = '2025-07-15',
    @HorizonTo date = '2025-12-31';

/*──────────────────  очистка TEMP  ─────────────────────────────*/
IF OBJECT_ID('tempdb..#bucket_def') IS NOT NULL DROP TABLE #bucket_def;
IF OBJECT_ID('tempdb..#fresh15')    IS NOT NULL DROP TABLE #fresh15;
IF OBJECT_ID('tempdb..#mkt')        IS NOT NULL DROP TABLE #mkt;
IF OBJECT_ID('tempdb..#mkt_any')    IS NOT NULL DROP TABLE #mkt_any;
IF OBJECT_ID('tempdb..#roll_fix')   IS NOT NULL DROP TABLE #roll_fix;
IF OBJECT_ID('tempdb..#match')      IS NOT NULL DROP TABLE #match;

SET NOCOUNT ON;

/*────────────────────  1. бакеты объёма  ──────────────────────*/
CREATE TABLE #bucket_def
( bucket varchar(20) PRIMARY KEY,
  lo     money       NOT NULL,
  hi     money       NULL,
  r      tinyint     NOT NULL);

INSERT #bucket_def (bucket,lo,hi,r) VALUES
('[0-1.5 млн)',0,1500000,0),
('[1.5-15 млн)',1500000,15000000,1),
('[15-100 млн)',15000000,100000000,2),
('[100 млн+]',100000000,NULL,3);

/*────────────────────  2. fresh-выдачи 15-го  ──────────────────*/
CREATE TABLE #fresh15
( bucket        varchar(20),
  TERM_GROUP    varchar(100),
  PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME  nvarchar(100),
  conv_norm     varchar(50),
  out_rub       money,
  spread        decimal(18,6));

/* 2-а. реальные сделки с конвенцией ≠ AT_THE_END */
INSERT #fresh15 (bucket,TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,
                 conv_norm,out_rub,spread)
SELECT b.bucket,
       tg.TERM_GROUP,
       t.PROD_NAME_RES,
       t.TSEGMENTNAME,
       CAST(t.conv AS varchar(50))           AS conv_norm,
       t.out_rub,
       t.rate_con - fk.AVG_KEY_RATE          AS spread
FROM  ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
JOIN  #bucket_def b
          ON t.out_rub>=b.lo AND (t.out_rub<b.hi OR b.hi IS NULL)
CROSS APPLY (SELECT TRY_CAST(t.DT_OPEN AS date) o(d_open)
JOIN  ALM_TEST.WORK.ForecastKey_Cache fk
          ON fk.DT_REP=o.d_open AND fk.TERM=t.termdays
LEFT JOIN WORK.man_TermGroup tg
          ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE t.dt_rep       = @Anchor
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.is_floatrate = 0
  AND t.out_rub      > 0
  AND t.conv        <> 'AT_THE_END'
  AND o.d_open       = @Anchor;

/* 2-б. «виртуальная» 1M ставка для AT_THE_END */
INSERT #fresh15 (bucket,TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,
                 conv_norm,out_rub,spread)
SELECT b.bucket,
       tg.TERM_GROUP,
       t.PROD_NAME_RES,
       t.TSEGMENTNAME,
       '1M'                                   AS conv_norm,
       t.out_rub,
       CAST([LIQUIDITY].[liq].[fnc_IntRate]
              (t.rate_con,'at the end','monthly',t.termdays,1)
            AS decimal(18,6)) - fk.AVG_KEY_RATE AS spread
FROM  ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
JOIN  #bucket_def b
          ON t.out_rub>=b.lo AND (t.out_rub<b.hi OR b.hi IS NULL)
CROSS APPLY (SELECT TRY_CAST(t.DT_OPEN AS date) o(d_open)
JOIN  ALM_TEST.WORK.ForecastKey_Cache fk
          ON fk.DT_REP=o.d_open AND fk.TERM=t.termdays
LEFT JOIN WORK.man_TermGroup tg
          ON t.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE t.dt_rep       = @Anchor
  AND t.section_name = N'Срочные'
  AND t.block_name   = N'Привлечение ФЛ'
  AND t.od_flag      = 1
  AND t.cur          = '810'
  AND t.is_floatrate = 0
  AND t.out_rub      > 0
  AND t.conv         = 'AT_THE_END'
  AND o.d_open       = @Anchor;

/*────────────────────  3. справочники спредов  ─────────────────*/
SELECT bucket,TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm,
       spread_mkt = SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
INTO #mkt
FROM #fresh15
GROUP BY bucket,TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm;

SELECT TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm,
       spread_any = SUM(out_rub*spread)/NULLIF(SUM(out_rub),0)
INTO #mkt_any
FROM #fresh15
GROUP BY TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm;

/*──────────────  4. roll-over фиксы (+ фактический спред)  ─────────────*/
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
        spread_fix = r.rate_con - fk_open.AVG_KEY_RATE
INTO    #roll_fix
FROM    ALM.ALM.vw_balance_rest_all       r  WITH (NOLOCK)
JOIN    #bucket_def                       b
           ON r.out_rub>=b.lo AND (r.out_rub<b.hi OR b.hi IS NULL)
LEFT JOIN WORK.man_TermGroup              tg
           ON r.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
LEFT JOIN ALM_TEST.WORK.ForecastKey_Cache fk_open
           ON fk_open.DT_REP = TRY_CAST(r.DT_OPEN AS date)
          AND fk_open.TERM   = r.termdays
CROSS  APPLY (SELECT TRY_CAST(r.DT_CLOSE AS date) c(d_close)
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
SELECT 
    rf.*,
    spread_mkt = COALESCE(mkt_r.spread_mkt, mkt_any.spread_any, rf.spread_fix),
    spread_final = COALESCE(mkt_r.spread_mkt, mkt_any.spread_any, rf.spread_fix),
    is_matched = CASE 
                    WHEN mkt_r.spread_mkt IS NOT NULL OR mkt_any.spread_any IS NOT NULL 
                    THEN 1 
                    ELSE 0 
                 END
INTO #match
FROM #roll_fix rf
LEFT JOIN (
    SELECT m.*, b_m.r 
    FROM #mkt m
    JOIN #bucket_def b_m ON m.bucket = b_m.bucket
) mkt_r 
    ON mkt_r.TERM_GROUP = rf.TERM_GROUP
    AND mkt_r.PROD_NAME_RES = rf.PROD_NAME_RES
    AND mkt_r.TSEGMENTNAME = rf.TSEGMENTNAME
    AND mkt_r.conv_norm = rf.conv_norm
    AND mkt_r.r >= rf.r
LEFT JOIN #mkt_any mkt_any 
    ON mkt_any.TERM_GROUP = rf.TERM_GROUP
    AND mkt_any.PROD_NAME_RES = rf.PROD_NAME_RES
    AND mkt_any.TSEGMENTNAME = rf.TSEGMENTNAME
    AND mkt_any.conv_norm = rf.conv_norm
    AND rf.TSEGMENTNAME = N'ДЧБО';

/*──────────────  6. сводка покрытия  ───────────────────────────*/
SELECT
    total_deals   = COUNT(*) ,
    covered_deals = SUM(is_matched) ,
    pct_deals     = 100.0 * SUM(is_matched) / COUNT(*) ,
    total_rub     = SUM(out_rub) ,
    covered_rub   = SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END) ,
    pct_rub       = 100.0 * SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END)
                   / NULLIF(SUM(out_rub),0)
FROM #match;

/*──────────────  7. детализация  ───────────────────────────────*/
SELECT bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm,
       deals_tot = COUNT(*) ,
       deals_ok  = SUM(is_matched) ,
       pct_deals = 100.0 * SUM(is_matched) / COUNT(*) ,
       rub_tot   = SUM(out_rub) ,
       rub_ok    = SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END) ,
       pct_rub   = 100.0 * SUM(CASE WHEN is_matched=1 THEN out_rub ELSE 0 END)
                   / NULLIF(SUM(out_rub),0)
FROM #match
GROUP BY bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm
ORDER BY pct_rub DESC;

/*──────────────  8. «полотно» по roll-over-сделкам  ────────────*/
SELECT con_id, out_rub, bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm,
       spread_fix, spread_mkt, spread_final, is_matched
FROM #match
ORDER BY is_matched, bucket, TERM_GROUP, PROD_NAME_RES;
