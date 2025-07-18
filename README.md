/*──────────────────── 3-NEW. фактовая база 16-07-2025 ───────────*/
IF OBJECT_ID('tempdb..#base') IS NOT NULL DROP TABLE #base;
SELECT  t.con_id,
        t.out_rub,
        t.rate_con,
        t.is_floatrate,
        t.termdays,
        t.dt_open,
        t.dt_close,
        -- float: сразу факт-спред
        spread_float = CASE WHEN t.is_floatrate = 1
                                 THEN t.rate_con - ks.KEY_RATE END,
        -- fix : либо найденный spread_final, либо факт
        spread_fix   = CASE WHEN t.is_floatrate = 0
                                 THEN COALESCE(fs.spread_final,
                                               t.rate_con - fk_open.AVG_KEY_RATE) END
INTO    #base
FROM    ALM.ALM.vw_balance_rest_all t  WITH (NOLOCK)
JOIN    #key_spot ks              ON ks.DT_REP = @Anchor
LEFT    JOIN WORK.ForecastKey_Cache fk_open
           ON fk_open.DT_REP = t.dt_open
          AND fk_open.TERM   = t.termdays
LEFT    JOIN #fix_spread fs      ON fs.con_id   = t.con_id   -- ← spread_final
WHERE   t.dt_rep       = @Anchor
  AND   t.section_name = N'Срочные'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.od_flag      = 1
  AND   t.cur          = '810'
  AND   t.out_rub      IS NOT NULL;

/*──────────────────── 5-NEW. roll-over-цепочки ─────────────────*/
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;

;WITH seq AS (
    SELECT  con_id,
            out_rub,
            is_floatrate,
            termdays,
            spread_float,
            spread_fix,          -- ← уже «правильный»
            dt_open,
            n = 0
    FROM    #base
    UNION ALL
    SELECT  con_id,
            out_rub,
            is_floatrate,
            termdays,
            spread_float,
            spread_fix,          -- просто наследуем
            DATEADD(day,termdays,dt_open),
            n + 1
    FROM    seq
    WHERE   DATEADD(day,termdays,dt_open) <= @HorizonTo
)
SELECT  con_id,
        out_rub,
        is_floatrate,
        termdays,
        dt_open,
        dt_close = DATEADD(day,termdays,dt_open),
        spread_float,
        spread_fix,
        n
INTO   #rolls
FROM   seq
OPTION (MAXRECURSION 0);
