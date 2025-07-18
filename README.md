/*=======================================================================
  3.  база n = 0  (факт)  +  заранее вычисленный spread_final
      ----------------------------------------------------------
      • spread_fix_fact  – всегда факт-спред
      • spread_final     – TO-BE-спред (или тот же факт, если пары нет)
=======================================================================*/
IF OBJECT_ID('tempdb..#base2') IS NOT NULL DROP TABLE #base2;

SELECT  b.con_id,
        b.out_rub,
        b.is_floatrate,
        b.termdays,
        b.dt_open,
        b.spread_float,
        -- факт-спред:
        b.spread_fix                 AS spread_fix_fact,
        -- что возьмём при rollover (n≥1):
        COALESCE(fs.spread_final, b.spread_fix) AS spread_final
INTO    #base2
FROM    #base  b               -- из шага «фактовая база»
LEFT    JOIN #fix_spread fs    -- готовый справочник TO-BE
           ON fs.con_id = b.con_id;

/*=======================================================================
  4.  roll-over-цепочка без OUTER JOIN внутри рекурсии
=======================================================================*/
IF OBJECT_ID('tempdb..#rolls') IS NOT NULL DROP TABLE #rolls;

;WITH seq AS (
        /* ---- n = 0 : факт (spread_fix_fact) ---- */
        SELECT  con_id,
                out_rub,
                is_floatrate,
                termdays,
                dt_open,
                spread_float,
                spread_fix = spread_fix_fact,      -- ← только факт!
                spread_final,                      -- запасём для n≥1
                n = 0
        FROM    #base2

        UNION ALL

        /* ---- n ≥ 1 : следующий период (spread_final) ---- */
        SELECT  s.con_id,
                s.out_rub,
                s.is_floatrate,
                s.termdays,
                DATEADD(day,s.termdays,s.dt_open)      AS dt_open,
                s.spread_float,
                s.spread_final                         AS spread_fix,
                s.spread_final,                        -- передаём дальше
                n + 1
        FROM    seq s
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
INTO   #rolls
FROM   seq
OPTION (MAXRECURSION 0);
