/* ════════════════════════════════════════════════════════════════════
     NS forecast  |  FLOAT  •  FIX-base  •  FIX-promo (2 мес roll)
     v.2025-08-07-split
     – генерирует три daily-ленты:
         ❶  #FLOAT_daily   – спред фиксируется в dt_open
         ❷  #FIX_base_daily – ставка константа
         ❸  #FIX_promo_daily – 2-месячный promo-цикл + «базовый день»
     – затем склеивает их → WORK.Forecast_BalanceDaily_NS
   ═══════════════════════════════════════════════════════════════════ */

USE ALM_TEST;
GO
/* — параметры — */
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;

/* ───────────────────────────────  общие служебные объекты  ── */
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

DROP TABLE IF EXISTS #key;            /* spot-KEY (TERM=1) */
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/* фактический портфель на @Anchor */
DROP TABLE IF EXISTS #bal;
SELECT  t.con_id,t.cli_id,t.prod_id,
        t.out_rub,t.rate_con,t.is_floatrate,
        CAST(t.dt_open  AS date) AS dt_open,
        CAST(t.dt_close AS date) AS dt_close,
        t.TSEGMENTNAME
INTO    #bal
FROM    ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
WHERE   t.dt_rep = @Anchor
  AND   t.section_name = N'Накопительный счёт'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.od_flag      = 1
  AND   t.cur          = '810'
  AND   t.out_rub IS NOT NULL
  AND  (t.dt_close_fact IS NULL OR t.dt_close_fact >= @Anchor);

/* ─────────────────────────────── ❶  FLOAT  (3103)  ────────── */
DROP TABLE IF EXISTS #FLOAT_daily;
SELECT  b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
        c.d          AS dt_rep,
        (b.rate_con-k0.KEY_RATE) + k1.KEY_RATE AS rate_con
INTO    #FLOAT_daily
FROM    #bal b
JOIN    #key k0 ON k0.DT_REP = b.dt_open       -- фиксируем спред
JOIN    #cal c  ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
JOIN    #key k1 ON k1.DT_REP = c.d
WHERE   b.prod_id = 3103;

/* ─────────────────────────────── ❷  FIX-base (старые) ─────── */
DROP TABLE IF EXISTS #FIX_base_daily;
SELECT  b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
        c.d        AS dt_rep,
        b.rate_con AS rate_con
INTO    #FIX_base_daily
FROM    #bal b
JOIN    #cal c ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
WHERE   b.prod_id = 654
  AND   b.dt_open < '2025-07-01';              -- ≤ июнь 2025

/* ─────────────────────────────── ❸  FIX-promo (июл-авг) ───── */
-----------------------------------------------------------------
/* ❸-1   общий promo-спред (считаем по август-25 открытиям) */
DECLARE @Spread_DChbo decimal(9,4),
        @Spread_Retail decimal(9,4);

WITH aug AS (
    SELECT TSEGMENTNAME,
           w_rate = SUM(out_rub*rate_con)/SUM(out_rub)
    FROM   #bal
    WHERE  prod_id = 654
      AND  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
    GROUP  BY TSEGMENTNAME)
SELECT @Spread_DChbo  = MAX(CASE WHEN TSEGMENTNAME=N'ДЧБО' THEN w_rate END) - k.KEY_RATE,
       @Spread_Retail= MAX(CASE WHEN TSEGMENTNAME=N'Розничный бизнес' THEN w_rate END) - k.KEY_RATE
FROM   aug
CROSS  JOIN #key k
WHERE  k.DT_REP = @Anchor;

/* ❸-2   рекурсивная сетка promo-окон (2 мес) */
DROP TABLE IF EXISTS #promo_win;
;WITH seed AS (
      SELECT b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
             win_start = b.dt_open,
             win_end   = EOMONTH(b.dt_open,1)
      FROM   #bal b
      WHERE  b.prod_id = 654
        AND  b.dt_open BETWEEN '2025-07-01' AND '2025-08-31')
, seq AS (
      SELECT *,0 AS n FROM seed
      UNION ALL
      SELECT con_id,cli_id,TSEGMENTNAME,out_rub,
             DATEADD(day,1,win_end),
             EOMONTH(DATEADD(day,1,win_end),1),
             n+1
      FROM   seq
      WHERE  DATEADD(day,1,win_end) <= @HorizonTo)
SELECT * INTO #promo_win FROM seq OPTION (MAXRECURSION 0);

/* ❸-3   дневная лента promo + базовый день + ставка по спреду */
DROP TABLE IF EXISTS #FIX_promo_daily;
SELECT  p.con_id,p.cli_id,p.TSEGMENTNAME,p.out_rub,
        c.d AS dt_rep,
        rate_con = CASE
              WHEN c.d <= p.win_end               -- promo-период
                   THEN CASE p.TSEGMENTNAME
                          WHEN N'ДЧБО'            THEN @Spread_DChbo  + k.KEY_RATE
                          ELSE                          @Spread_Retail+ k.KEY_RATE
                        END
              WHEN c.d = DATEADD(day,1,p.win_end) -- «базовый» день
                   THEN @BaseRate
              ELSE NULL                            -- это не может случиться
        END
INTO    #FIX_promo_daily
FROM    #promo_win p
JOIN    #cal c ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
JOIN    #key k ON k.DT_REP = c.d;

/* ❸-4   перелив promo 1-го числа внутри одного клиента */
----------------------------------------------------------
/* promo-FIX именно в 1-е число */
DROP TABLE IF EXISTS #p1;
SELECT DISTINCT cli_id,
       m1 = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
INTO   #p1
FROM   #FIX_promo_daily
WHERE  dt_rep = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1);

DROP TABLE IF EXISTS #p1_glue;
SELECT  NULL      AS con_id,
        p.cli_id,
        NULL      AS TSEGMENTNAME,
        SUM(f.out_rub)            AS out_rub,
        p.m1                      AS dt_rep,
        MAX(f.rate_con)           AS rate_con
INTO    #p1_glue
FROM   #p1 p
JOIN   #FIX_promo_daily f
       ON  f.cli_id=p.cli_id
       AND f.dt_rep=p.m1
GROUP  BY p.cli_id,p.m1;

/* promo-FIX в остальные дни (кроме 1-го) */
DROP TABLE IF EXISTS #p_not1;
SELECT *
INTO   #p_not1
FROM   #FIX_promo_daily
WHERE  dt_rep <> DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1);

/* ───────────────────────────────  склейка всех лент  ───────── */
DROP TABLE IF EXISTS #PORTF_ALL;
SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con
INTO   #PORTF_ALL
FROM   #FLOAT_daily
UNION ALL
SELECT * FROM #FIX_base_daily
UNION ALL
SELECT * FROM #p_not1
UNION ALL
SELECT * FROM #p1_glue;

/* ───────────────────────────────  агрегат портфеля  ───────── */
TRUNCATE TABLE WORK.Forecast_BalanceDaily_NS;
INSERT  WORK.Forecast_BalanceDaily_NS
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #PORTF_ALL
GROUP  BY dt_rep;

/* ─────────────  sanity-check: «ступеньки» promo-FIX  ───────── */
PRINT N'╔═ sample ══════════════════════════════════════════════';
SELECT dt_rep,out_rub_total,rate_avg
FROM   WORK.Forecast_BalanceDaily_NS
WHERE  dt_rep IN ('2025-08-30','2025-08-31',
                  '2025-09-01','2025-09-15',
                  '2025-09-30','2025-10-01')
ORDER BY dt_rep;
PRINT N'╚═══════════════════════════════════════════════════════';
