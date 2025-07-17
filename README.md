/*─── патч только к блоку 4  ─────────────────────────────────────*/
/* 4. roll-over фиксы */
CREATE TABLE #roll_fix
( con_id        bigint,
  out_rub       money,
  bucket        varchar(20),
  r             tinyint,
  TERM_GROUP    varchar(100),
  PROD_NAME_RES nvarchar(200),
  TSEGMENTNAME  nvarchar(100),
  conv_norm     varchar(50));

INSERT #roll_fix (con_id,out_rub,bucket,r,
                  TERM_GROUP,PROD_NAME_RES,TSEGMENTNAME,conv_norm)
SELECT r.con_id,
       r.out_rub,
       b.bucket,
       b.r,
       tg.TERM_GROUP,
       r.PROD_NAME_RES,
       r.TSEGMENTNAME,
       CASE WHEN r.conv='AT_THE_END' THEN '1M'
            ELSE CAST(r.conv AS varchar(50)) END
FROM  ALM.ALM.vw_balance_rest_all r WITH (NOLOCK)
JOIN  #bucket_def b
          ON r.out_rub>=b.lo AND (r.out_rub<b.hi OR b.hi IS NULL)
CROSS APPLY (SELECT TRY_CAST(r.DT_CLOSE AS date) d_close) c
LEFT JOIN WORK.man_TermGroup tg
          ON r.termdays BETWEEN tg.TERM_FROM AND tg.TERM_TO
WHERE r.dt_rep       = @Anchor
  AND r.section_name = N'Срочные'
  AND r.block_name   = N'Привлечение ФЛ'
  AND r.od_flag      = 1
  AND r.cur          = '810'
  AND r.is_floatrate = 0
  AND r.out_rub      > 0
  AND c.d_close      <= @HorizonTo
  AND c.d_close      IS NOT NULL;

/*─── шаг 5 остаётся без изменений (он ловит rf.*) ───────────────*/

/*────────────────────  6. сводка  ──────────────────────────────*/
SELECT
    total_deals   = COUNT(*) ,
    covered_deals = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals     = 100.0 *
                    SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/                   COUNT(*) ,
    total_rub     = SUM(out_rub) ,
    covered_rub   = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
    pct_rub       = 100.0 *
                    SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/                   NULLIF(SUM(out_rub),0);
                    
/*────────────────────  7. детализация  ─────────────────────────*/
SELECT
    bucket , TERM_GROUP , PROD_NAME_RES , TSEGMENTNAME , conv_norm ,
    deals_tot = COUNT(*) ,
    deals_ok  = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
    pct_deals = 100.0 *
                SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/               COUNT(*) ,
    rub_tot   = SUM(out_rub) ,
    rub_ok    = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
    pct_rub   = 100.0 *
                SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/               NULLIF(SUM(out_rub),0)
FROM #match
GROUP BY bucket , TERM_GROUP , PROD_NAME_RES , TSEGMENTNAME , conv_norm
ORDER BY pct_rub DESC;
