/*────────────────────  5*. каскадное сопоставление  ───────────*/
IF OBJECT_ID('tempdb..#match') IS NOT NULL DROP TABLE #match;

SELECT
        rf.* ,

        /* найденный «рыночный» спред в том же / более крупном бакете */
        mkt_up.spread_mkt                   AS spread_mkt ,

        /* итог: рынок ↦ fallback ↦ фактический */
        COALESCE(mkt_up.spread_mkt,
                 dchbo_any.spread_any,
                 rf.spread_fix)             AS spread_final ,

        /* флаг: 1 — нашли рынок / fallback, 0 — взяли фактический */
        CASE WHEN mkt_up.spread_mkt      IS NOT NULL
               OR dchbo_any.spread_any  IS NOT NULL
             THEN 1 ELSE 0 END             AS is_matched
INTO    #match
FROM    #roll_fix            AS rf

/* ── рынок того же или более «крупного» бакета */
OUTER APPLY (
        SELECT TOP (1) m.spread_mkt
        FROM   #mkt        AS m
        JOIN   #bucket_def AS b_m ON b_m.bucket = m.bucket
        WHERE  m.TERM_GROUP     = rf.TERM_GROUP
          AND  m.PROD_NAME_RES  = rf.PROD_NAME_RES
          AND  m.TSEGMENTNAME   = rf.TSEGMENTNAME
          AND  m.conv_norm      = rf.conv_norm
          AND  b_m.r            >= rf.r       -- тот же или крупнее
        ORDER  BY b_m.r                      -- ближайший ↑-бакет
) AS mkt_up

/* ── fallback «вне бакетов» только для ДЧБО */
OUTER APPLY (
        SELECT ma.spread_any
        FROM   #mkt_any AS ma
        WHERE  rf.TSEGMENTNAME  = N'ДЧБО'
          AND  ma.TERM_GROUP    = rf.TERM_GROUP
          AND  ma.PROD_NAME_RES = rf.PROD_NAME_RES
          AND  ma.TSEGMENTNAME  = rf.TSEGMENTNAME
          AND  ma.conv_norm     = rf.conv_norm
) AS dchbo_any;
