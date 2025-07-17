/*──────────────  6. сводка покрытия  ──────────────*/
IF OBJECT_ID('tempdb..#report_summary') IS NOT NULL DROP TABLE #report_summary;

SELECT
        total_deals   = COUNT(*) ,
        covered_deals = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
        pct_deals     = CAST(
                         100.0 * SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
                               / NULLIF(COUNT(*),0) AS decimal(18,4)),
        total_rub     = SUM(out_rub) ,
        covered_rub   = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
        pct_rub       = CAST(
                         100.0 * SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
                               / NULLIF(SUM(out_rub),0) AS decimal(18,4))
INTO    #report_summary
FROM    #match;

SELECT * FROM #report_summary;



/*──────────────  7. детализация  ──────────────*/
IF OBJECT_ID('tempdb..#report_detail') IS NOT NULL DROP TABLE #report_detail;

SELECT
        bucket,
        TERM_GROUP,
        PROD_NAME_RES,
        TSEGMENTNAME,
        conv_norm,
        deals_tot = COUNT(*) ,
        deals_ok  = SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END) ,
        pct_deals = CAST(
                     100.0 * SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
                           / NULLIF(COUNT(*),0) AS decimal(18,4)),
        rub_tot   = SUM(out_rub) ,
        rub_ok    = SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END) ,
        pct_rub   = CAST(
                     100.0 * SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
                           / NULLIF(SUM(out_rub),0) AS decimal(18,4))
INTO    #report_detail
FROM    #match
GROUP BY bucket, TERM_GROUP, PROD_NAME_RES, TSEGMENTNAME, conv_norm;

SELECT *
FROM   #report_detail
ORDER  BY pct_rub DESC, pct_deals DESC;
