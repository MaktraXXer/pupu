/* ПАРАМЕТРЫ */
DECLARE
    @Anchor    date = '2025-07-15',
    @HorizonTo date = '2025-08-31';
GO
/* 0. справочник бакетов */
IF OBJECT_ID('tempdb..#bucket_def') IS NOT NULL DROP TABLE #bucket_def;
CREATE TABLE #bucket_def
( [bucket] varchar(20) PRIMARY KEY,
  lo money NOT NULL,
  hi money NULL,
  r  tinyint NOT NULL );
INSERT #bucket_def VALUES
('[0-1.5 млн)',0,1500000,0),
('[1.5-15 млн)',1500000,15000000,1),
('[15-100 млн)',15000000,100000000,2),
('[100 млн+]',100000000,NULL,3);
GO
/* 1. #fresh15 – рынок (фиксы, открытые 15-го) */
IF OBJECT_ID('tempdb..#fresh15') IS NOT NULL DROP TABLE #fresh15;
SELECT
        b.[bucket],
        tg.[TERM_GROUP],
        t.[PROD_NAME_RES],
        t.[TSEGMENTNAME],
        [conv] = CAST(t.[conv] AS varchar(50)),
        t.[out_rub],
        spread = t.[rate_con] - fk.[AVG_KEY_RATE]
INTO    #fresh15
FROM    ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
JOIN    #bucket_def  b ON t.[out_rub] >= b.lo
                       AND (t.[out_rub] < b.hi OR b.hi IS NULL)
CROSS APPLY (SELECT TRY_CAST(t.[DT_OPEN] AS date) AS d_open) o
JOIN    ALM_TEST.WORK.ForecastKey_Cache fk
             ON fk.[DT_REP]=o.d_open
            AND fk.[TERM]  =t.[termdays]
LEFT JOIN WORK.man_TermGroup tg
             ON t.[termdays] BETWEEN tg.[TERM_FROM] AND tg.[TERM_TO]
WHERE   t.[dt_rep]=@Anchor
  AND   t.[section_name]=N'Срочные'
  AND   t.[block_name]  =N'Привлечение ФЛ'
  AND   t.[od_flag]=1
  AND   t.[cur]='810'
  AND   t.[is_floatrate]=0
  AND   t.[out_rub]>0
  AND   o.d_open=@Anchor;
GO
/* 2. #mkt (точный) и #mkt_any (без-бакет) */
IF OBJECT_ID('tempdb..#mkt')     IS NOT NULL DROP TABLE #mkt;
IF OBJECT_ID('tempdb..#mkt_any') IS NOT NULL DROP TABLE #mkt_any;
SELECT  [bucket],[TERM_GROUP],[PROD_NAME_RES],[TSEGMENTNAME],[conv],
        spread_mkt = SUM([out_rub]*spread)/NULLIF(SUM([out_rub]),0)
INTO   #mkt
FROM   #fresh15
GROUP BY [bucket],[TERM_GROUP],[PROD_NAME_RES],[TSEGMENTNAME],[conv];

SELECT  [TERM_GROUP],[PROD_NAME_RES],[TSEGMENTNAME],[conv],
        spread_any = SUM([out_rub]*spread)/NULLIF(SUM([out_rub]),0)
INTO   #mkt_any
FROM   #fresh15
GROUP BY [TERM_GROUP],[PROD_NAME_RES],[TSEGMENTNAME],[conv];
GO
/* 3. #roll_fix – фиксы, закрывающиеся ≤ 31-08 */
IF OBJECT_ID('tempdb..#roll_fix') IS NOT NULL DROP TABLE #roll_fix;
SELECT  r.[con_id],r.[out_rub],r.[rate_con],
        b.[bucket],b.r,
        tg.[TERM_GROUP],
        r.[PROD_NAME_RES],r.[TSEGMENTNAME],
        [conv]=CAST(r.[conv] AS varchar(50))
INTO   #roll_fix
FROM   ALM.ALM.vw_balance_rest_all r WITH (NOLOCK)
JOIN   #bucket_def b ON r.[out_rub]>=b.lo
                     AND (r.[out_rub]<b.hi OR b.hi IS NULL)
CROSS APPLY (SELECT TRY_CAST(r.[DT_CLOSE] AS date) AS d_close) c
LEFT JOIN WORK.man_TermGroup tg
           ON r.[termdays] BETWEEN tg.[TERM_FROM] AND tg.[TERM_TO]
WHERE  r.[dt_rep]=@Anchor
  AND  r.[section_name]=N'Срочные'
  AND  r.[block_name]  =N'Привлечение ФЛ'
  AND  r.[od_flag]=1
  AND  r.[cur]='810'
  AND  r.[is_floatrate]=0
  AND  r.[out_rub]>0
  AND  c.d_close<=@HorizonTo;
GO
/* 4. #match – каскадное сопоставление */
IF OBJECT_ID('tempdb..#match') IS NOT NULL DROP TABLE #match;
SELECT rf.*,
       spread_mkt = COALESCE
       ( /* а) точный или ↑-бакет */
         ( SELECT TOP 1 m.spread_mkt
           FROM #mkt m
           JOIN #bucket_def b_m ON b_m.[bucket]=m.[bucket]
           WHERE m.[TERM_GROUP]=rf.[TERM_GROUP]
             AND m.[PROD_NAME_RES]=rf.[PROD_NAME_RES]
             AND m.[TSEGMENTNAME]=rf.[TSEGMENTNAME]
             AND m.[conv]=rf.[conv]
             AND b_m.r>=rf.r
           ORDER BY b_m.r ),
         /* б) fallback для ДЧБО */
         CASE WHEN rf.[TSEGMENTNAME]=N'ДЧБО'
              THEN (SELECT ma.spread_any
                    FROM #mkt_any ma
                    WHERE ma.[TERM_GROUP]=rf.[TERM_GROUP]
                      AND ma.[PROD_NAME_RES]=rf.[PROD_NAME_RES]
                      AND ma.[TSEGMENTNAME]=rf.[TSEGMENTNAME]
                      AND ma.[conv]=rf.[conv]) END )
INTO   #match
FROM   #roll_fix rf;
GO
/* 5. сводка */
SELECT total_deals=COUNT(*),
       covered_deals=SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END),
       pct_deals=100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/                COUNT(*),
       total_rub=SUM(out_rub),
       covered_rub=SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END),
       pct_rub=100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/               NULLIF(SUM(out_rub),0)
FROM #match;
GO
/* 6. детализация */
SELECT [bucket],[TERM_GROUP],[PROD_NAME_RES],[TSEGMENTNAME],[conv],
       deals_tot=COUNT(*),
       deals_ok =SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END),
       pct_deals=100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN 1 ELSE 0 END)
/                COUNT(*),
       rub_tot  =SUM(out_rub),
       rub_ok   =SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END),
       pct_rub  =100.*SUM(CASE WHEN spread_mkt IS NOT NULL THEN out_rub ELSE 0 END)
/                NULLIF(SUM(out_rub),0)
FROM #match
GROUP BY [bucket],[TERM_GROUP],[PROD_NAME_RES],[TSEGMENTNAME],[conv]
ORDER BY pct_rub DESC;
GO
