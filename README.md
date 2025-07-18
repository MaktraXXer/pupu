/*──────── 3. фактовая база 16-07-2025  ───────*/
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;

SELECT  t.con_id,
        t.out_rub,
        t.rate_con,
        t.is_floatrate,
        t.termdays,
        t.dt_open,
        t.dt_close,
        t.TSEGMENTNAME,          -- ← добавили
        t.conv,                  -- ← добавили
        /* спреды */
        spread_float = CASE WHEN t.is_floatrate = 1
                                 THEN t.rate_con - ks.KEY_RATE END,
        spread_fix   = CASE WHEN t.is_floatrate = 0
                                 THEN COALESCE(fs.spread_final,
                                               t.rate_con - fk_open.AVG_KEY_RATE) END
INTO    #base
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
JOIN    #key_spot ks
           ON ks.DT_REP = @Anchor
LEFT    JOIN WORK.ForecastKey_Cache fk_open
           ON fk_open.DT_REP = t.dt_open
          AND fk_open.TERM   = t.termdays
LEFT    JOIN #fix_spread fs
           ON fs.con_id = t.con_id
WHERE   t.dt_rep       = @Anchor
  AND   t.section_name = N'Срочные'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.od_flag      = 1
  AND   t.cur          = '810'
  AND   t.out_rub      IS NOT NULL;
