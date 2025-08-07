/* ─────────── 4-A. FLOAT daily (исправлено) ─────────── */
IF OBJECT_ID('tempdb..#FLOAT_daily') IS NOT NULL DROP TABLE #FLOAT_daily;
SELECT  b.con_id,
        b.cli_id,
        b.TSEGMENTNAME,
        b.out_rub,
        c.d                                      AS dt_rep,
        (b.rate_con-k0.KEY_RATE)+k1.KEY_RATE     AS rate_con
INTO    #FLOAT_daily
FROM    #bal b
JOIN    #key k0 ON k0.DT_REP = @Anchor          -- ⬅ spread фиксируем на Anchor
JOIN    #cal c  ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
JOIN    #key k1 ON k1.DT_REP = c.d
WHERE   b.prod_id = 3103;
