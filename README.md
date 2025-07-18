/*───────────────────── 9. Roll-over с применением spread_final */
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;
;WITH seq AS (
    SELECT con_id, out_rub, is_floatrate, termdays,
           spread_float, spread_fix, dt_open, n = 0
    FROM   #base
    UNION ALL
    SELECT s.con_id, s.out_rub, s.is_floatrate, s.termdays,
           s.spread_float, s.spread_fix, DATEADD(day, s.termdays, s.dt_open), n + 1
    FROM   seq s
    WHERE  DATEADD(day, s.termdays, s.dt_open) <= @HorizonTo
)
SELECT s.con_id, s.out_rub, s.is_floatrate, s.termdays,
       s.dt_open,
       dt_close = DATEADD(day, s.termdays, s.dt_open),
       s.spread_float,
       spread_fix = ISNULL(fs.spread_final, s.spread_fix)
INTO   #rolls
FROM   seq s
LEFT   JOIN #fix_spread fs ON fs.con_id = s.con_id
OPTION (MAXRECURSION 0);
