OUTER APPLY
(
    /* одно вычисление — потом используем дважды */
    SELECT TOP 1 WITH TIES          -- ← добавили WITH TIES
           COALESCE(m1.spread_mkt, m2.spread_any) AS spread_mkt
    FROM   -- (а) тот же или более крупный бакет
           ( SELECT m.spread_mkt,
                    b_m.r                          /* ← ранги для сортировки */
             FROM   #mkt m
             JOIN   #bucket_def b_m ON b_m.bucket = m.bucket
             WHERE  m.TERM_GROUP     = rf.TERM_GROUP
               AND  m.PROD_NAME_RES  = rf.PROD_NAME_RES
               AND  m.TSEGMENTNAME   = rf.TSEGMENTNAME
               AND  m.conv_norm      = rf.conv_norm
               AND  b_m.r            >= rf.r )             m1
           FULL OUTER JOIN
           -- (б) fallback для ДЧБО
           ( SELECT ma.spread_any
             FROM   #mkt_any ma
             WHERE  rf.TSEGMENTNAME = N'ДЧБО'
               AND  ma.TERM_GROUP    = rf.TERM_GROUP
               AND  ma.PROD_NAME_RES = rf.PROD_NAME_RES
               AND  ma.TSEGMENTNAME  = rf.TSEGMENTNAME
               AND  ma.conv_norm     = rf.conv_norm )      m2
             ON 1 = 1
    ORDER BY b_m.r                                   -- ← однозначно
) sp_mkt;
