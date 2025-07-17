/* ───── 5. каскадное сопоставление ▀▀ */

IF OBJECT_ID('tempdb..#match') IS NOT NULL DROP TABLE #match;

SELECT rf.*,
       spread_mkt =
       COALESCE
       ( /* а) точный бакет или «выше» */
         ( SELECT TOP (1) m.spread_mkt
           FROM   #mkt         AS m
           JOIN   #bucket_def  AS b_m ON b_m.bucket = m.bucket
           WHERE  m.TERM_GROUP      = rf.TERM_GROUP
             AND  m.PROD_NAME_RES   = rf.PROD_NAME_RES
             AND  m.TSEGMENTNAME    = rf.TSEGMENTNAME
             AND  m.conv            = rf.conv
             AND  b_m.r             >= rf.r          -- тот же или крупнее
           ORDER  BY b_m.r ),                        -- ближайший «крупнее»
         /* б) fallback только для ДЧБО */
         CASE WHEN rf.TSEGMENTNAME = N'ДЧБО'
              THEN ( SELECT ma.spread_any
                     FROM   #mkt_any AS ma
                     WHERE  ma.TERM_GROUP     = rf.TERM_GROUP
                       AND  ma.PROD_NAME_RES  = rf.PROD_NAME_RES
                       AND  ma.TSEGMENTNAME   = rf.TSEGMENTNAME
                       AND  ma.conv           = rf.conv )
         END )
INTO   #match        -- ← правильный способ создать temp-таблицу «на лету»
FROM   #roll_fix AS rf;
