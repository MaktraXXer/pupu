/*=======================================================================
  3-4.  roll-over-цепочка           (ФАКТ ≠ ПРОГНОЗНЫЙ)
        ─────────────────────────────────────────────────────────
        • n = 0 → spread_fix_fact   (как в балансе)
        • n ≥ 1 → spread_final      (TO-BE  |  fact-fallback)
=======================================================================*/

/* 3-а. база без подмены: spread_fix_fact остаётся как есть */
IF OBJECT_ID('tempdb..#base2') IS NOT NULL DROP TABLE #base2;
SELECT  b.con_id,
        b.out_rub,
        b.is_floatrate,
        b.termdays,
        b.dt_open,
        b.spread_float,
        b.spread_fix_fact          -- ← только факт!
INTO    #base2
FROM    #base b;

/* 3-б. рекурсивная цепочка  */
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;

;WITH seq AS (
        /* n = 0 : фактическая запись */
        SELECT  con_id, out_rub, is_floatrate, termdays,
                dt_open,
                spread_float,
                spread_fix = spread_fix_fact,
                n = 0
        FROM    #base2
        UNION ALL
        /* n ≥ 1 : rollover со spread_final (или факт-fallback) */
        SELECT  s.con_id,
                s.out_rub,
                s.is_floatrate,
                s.termdays,
                DATEADD(day,s.termdays,s.dt_open),          -- новый open
                s.spread_float,
                spread_fix = COALESCE(fs.spread_final, s.spread_fix),  -- ← здесь
                n + 1
        FROM    seq            s
        LEFT    JOIN #fix_spread fs ON fs.con_id = s.con_id
        WHERE   DATEADD(day,s.termdays,s.dt_open) <= @HorizonTo
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
INTO    #rolls
FROM    seq
OPTION (MAXRECURSION 0);
