/* ════════════════════════════════════════
   NS forecast   version 2025-08-07-rev
════════════════════════════════════════ */

USE ALM_TEST;
GO
/* ПАРАМЕТРЫ ----------------------------------------------------*/
DECLARE
    @scen      tinyint      = 1,              -- сценарий ключа
    @Anchor    date         = '2025-08-04',   -- последний факт
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;         -- ставка после promo

/* ВРЕМЕННЫЕ НАБОРЫ --------------------------------------------*/
DROP TABLE IF EXISTS #cal, #key, #bal, #promo, #float, #base, #daily;

/* календарь горизонта */
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* ключ (TERM=1) день-в-день */
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/* факт-портфель */
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

/* ==== 1. СПРЕДЫ (promo-FIX, июль+август) =====================*/
DROP TABLE IF EXISTS #spread;
;WITH src AS (
        SELECT bf.dt_open, bf.TSEGMENTNAME,
               w_rate = SUM(bf.out_rub*bf.rate_con)/SUM(bf.out_rub)
        FROM   #bal bf
        WHERE  bf.prod_id = 654
          AND  DATEDIFF(month,bf.dt_open,@Anchor) IN (0,1)
        GROUP  BY bf.dt_open,bf.TSEGMENTNAME)
SELECT  s.dt_open,
        s.TSEGMENTNAME,
        spread = s.w_rate - k.KEY_RATE           -- KEY того же дня!
INTO    #spread
FROM   src s
JOIN   #key k ON k.DT_REP = s.dt_open;

/* ==== 2. PROMO-ОКНА (2 мес) ===================================*/
DROP TABLE IF EXISTS #promo;
;WITH seed AS (
        SELECT bf.con_id,bf.cli_id,bf.TSEGMENTNAME,bf.out_rub,
               win_start = bf.dt_open,
               win_end   = EOMONTH(bf.dt_open,1),      -- L-день promo
               bf.dt_open AS id_open
        FROM   #bal bf
        WHERE  bf.prod_id = 654
          AND  DATEDIFF(month,bf.dt_open,@Anchor) IN (0,1)
), seq AS (
        SELECT *,0 AS n FROM seed
        UNION ALL
        SELECT con_id,cli_id,TSEGMENTNAME,out_rub,
               DATEADD(day,1,win_end),
               EOMONTH(DATEADD(day,1,win_end),1),
               id_open, n+1
        FROM   seq
        WHERE  DATEADD(day,1,win_end) <= @HorizonTo )
SELECT * INTO #promo FROM seq OPTION (MAXRECURSION 0);

/* ==== 3. СТАВКИ ==============================================*/
---- 3.1 FIX-promo и Base-day
SELECT  p.con_id,p.cli_id,p.TSEGMENTNAME,p.out_rub,
        c.d                                            AS dt_rep,
        CASE WHEN c.d <= p.win_end                     -- promo
             THEN sp.spread + k.KEY_RATE
             ELSE @BaseRate END                        AS rate_con
INTO    #promo_rates
FROM   #promo p
JOIN   #spread  sp ON sp.dt_open = p.id_open
                  AND sp.TSEGMENTNAME = p.TSEGMENTNAME
JOIN   #cal     c  ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
JOIN   #key     k  ON k.DT_REP = c.d;

---- 3.2 FLOAT
SELECT  f.con_id,f.cli_id,f.TSEGMENTNAME,f.out_rub,
        c.d,
        (f.rate_con - k0.KEY_RATE) + k1.KEY_RATE       AS rate_con
INTO   #float
FROM   #bal    f
JOIN   #key    k0 ON k0.DT_REP = f.dt_open
JOIN   #cal    c  ON c.d BETWEEN f.dt_open AND ISNULL(f.dt_close,@HorizonTo)
JOIN   #key    k1 ON k1.DT_REP = c.d
WHERE  f.prod_id = 3103;

---- 3.3 FIX-base (старые)
SELECT  b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
        c.d, b.rate_con
INTO   #base
FROM   #bal b
JOIN   #cal c ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
WHERE  b.prod_id = 654
  AND  b.dt_open < DATEFROMPARTS(2025,8,1);

/* ==== 4. СЛИЯНИЕ И «ПЕРЕЛИВ» 1-ГО ЧИСЛА ======================*/
DROP TABLE IF EXISTS #daily_all;
SELECT * INTO #daily_all FROM
(SELECT * FROM #promo_rates
 UNION ALL
 SELECT * FROM #float
 UNION ALL
 SELECT * FROM #base) q;

/* promos-first-day */
DROP TABLE IF EXISTS #m1;
SELECT DISTINCT DATEFROMPARTS(YEAR(d),MONTH(d),1) d INTO #m1 FROM #cal;

;WITH promos_1 AS (
     SELECT  cli_id, d            AS m1
     FROM    #daily_all
     WHERE   d IN (SELECT d FROM #m1)
       AND   con_id IS NOT NULL   -- только реальные promo-FIX
       AND   prod_id IS NULL      -- (feat) prod_id нет ⇒ promo-FIX
     GROUP BY cli_id,d)
, best AS (
     SELECT p.cli_id,p.m1 AS dt_rep,
            SUM(out_rub)              AS out_rub,
            MAX(rate_con)             AS rate_con
     FROM   #daily_all d
     JOIN   promos_1  p
       ON   p.cli_id=d.cli_id AND p.m1=d.d
     GROUP  BY p.cli_id,p.m1)
, final AS (
     /* заменить промо-набор агрегированной строкой */
     SELECT NULL con_id,cli_id,TSEGMENTNAME=NULL,
            out_rub,dt_rep,rate_con FROM best
     UNION ALL
     SELECT con_id,cli_id,TSEGMENTNAME,out_rub,d,rate_con
     FROM   #daily_all d
     WHERE  NOT EXISTS (SELECT 1
                        FROM promos_1 p
                        WHERE p.cli_id=d.cli_id AND p.m1=d.d))
/* ==== 5. ИТОГ ПОРТФЕЛЯ =======================================*/
INSERT WORK.Forecast_BalanceDaily_NS
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   final
GROUP  BY dt_rep;

/* ==== TEST-BLOCK (оставьте, пожалуйста) =====================*/
--SELECT TOP (20) * FROM #promo ORDER BY cli_id, win_start;           -- окна
--SELECT TOP (10) * FROM #promo_rates ORDER BY cli_id,dt_rep;         -- ставки
--SELECT TOP (20) * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep; -- итог
