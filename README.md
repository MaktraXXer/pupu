/* ══════════════════════════════════════════════════════════════
   NS-forecast — ЧАСТЬ 2
   FIX-promo-roll (июль–авг-25, prod_id 654)
   – promo-окно = два прожитых календарных месяца
   – базовый день = последний день 2-го месяца (ставка = @BaseRate)
   – 1-е число: новая promo-ставка + glue-перелив на max-ставку
   сохраняет:
       WORK.NS_Spreads          – постоянные спреды
       WORK.NS_PromoRates       – promo-rate каждого месяца
       WORK.Forecast_NS_Promo   – дневная лента promo
       WORK.Forecast_BalanceDaily_NS – объединённый портфель
   v.2025-08-07 – fixed (нет «KEY_RATE» в агрегатах)
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;

/*–– проверяем, что Часть 1 загружена ––*/
IF OBJECT_ID('tempdb..#bal') IS NULL
 OR OBJECT_ID('tempdb..#cal') IS NULL
 OR OBJECT_ID('tempdb..#key') IS NULL
 OR OBJECT_ID('tempdb..#FLOAT_daily') IS NULL
 OR OBJECT_ID('tempdb..#FIX_base_daily') IS NULL
BEGIN
    RAISERROR(N'Сначала выполните Часть 1 (FLOAT + FIX-base).',16,1);
    RETURN;
END

/*–– ключ на Anchor как скаляр ––*/
DECLARE @KeyAnchor decimal(9,4);
SELECT @KeyAnchor = KEY_RATE FROM #key WHERE DT_REP = @Anchor;

/*----------------------------------------------------------------
 1. promo-SPREAD (по открытиям August-25) → WORK.NS_Spreads
----------------------------------------------------------------*/
DROP TABLE IF EXISTS WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads(
  TSEGMENTNAME nvarchar(40) PRIMARY KEY,
  spread       decimal(9,4));

DECLARE @Spread_DChbo  decimal(9,4),
        @Spread_Retail decimal(9,4);

;WITH aug AS (
    SELECT TSEGMENTNAME,
           w_rate = SUM(out_rub*rate_con)/SUM(out_rub)
    FROM   #bal
    WHERE  prod_id = 654
      AND  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
    GROUP  BY TSEGMENTNAME)
SELECT @Spread_DChbo  = MAX(CASE WHEN TSEGMENTNAME=N'ДЧБО'            THEN w_rate END) - @KeyAnchor,
       @Spread_Retail= MAX(CASE WHEN TSEGMENTNAME=N'Розничный бизнес' THEN w_rate END) - @KeyAnchor
FROM   aug;

INSERT WORK.NS_Spreads VALUES
      (N'ДЧБО',            @Spread_DChbo),
      (N'Розничный бизнес',@Spread_Retail);

/*----------------------------------------------------------------
 2. сетка promo-окон («два месяца») → #promo_win
----------------------------------------------------------------*/
DROP TABLE IF EXISTS #promo_win;
;WITH seed AS (
      SELECT b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
             win_start = b.dt_open,
             win_end   = DATEADD(day,-1,EOMONTH(DATEADD(month,1,b.dt_open))),
             n = 0
      FROM   #bal b
      WHERE  b.prod_id = 654
        AND  b.dt_open BETWEEN '2025-07-01' AND '2025-08-31')
, seq AS (
      SELECT * FROM seed
      UNION ALL
      SELECT  s.con_id, s.cli_id, s.TSEGMENTNAME, s.out_rub,
              DATEADD(day,1,s.win_end),
              DATEADD(day,-1,EOMONTH(DATEADD(month,1,DATEADD(day,1,s.win_end)))),
              n+1
      FROM   seq s
      WHERE  DATEADD(day,1,s.win_end) <= @HorizonTo)
SELECT * INTO #promo_win FROM seq OPTION (MAXRECURSION 0);

/*----------------------------------------------------------------
 3. дневник promo (promo-дни + базовый день + старт нового окна)
----------------------------------------------------------------*/
DROP TABLE IF EXISTS #FIX_promo_daily_raw;

SELECT  p.con_id,p.cli_id,p.TSEGMENTNAME,p.out_rub,
        c.d AS dt_rep,
        rate_con = CASE
          WHEN c.d <  p.win_end                 /* promo-период */
               THEN CASE p.TSEGMENTNAME
                      WHEN N'ДЧБО' THEN @Spread_DChbo  + k.KEY_RATE
                      ELSE              @Spread_Retail+ k.KEY_RATE END
          WHEN c.d = p.win_end                  /* базовый день */
               THEN @BaseRate
          WHEN c.d = DATEADD(day,1,p.win_end)   /* 1-е число нового окна */
               THEN CASE p.TSEGMENTNAME
                      WHEN N'ДЧБО' THEN @Spread_DChbo  + k.KEY_RATE
                      ELSE              @Spread_Retail+ k.KEY_RATE END
        END
INTO    #FIX_promo_daily_raw
FROM    #promo_win p
JOIN    #cal c  ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
JOIN    #key k  ON k.DT_REP =
        CASE WHEN c.d = DATEADD(day,1,p.win_end) THEN c.d ELSE p.win_start END;

/*----------------------------------------------------------------
 4. glue 1-го числа (перелив на max-ставку внутри cli_id)
----------------------------------------------------------------*/
DROP TABLE IF EXISTS #FIX_promo_glue;
;WITH m1 AS (
      SELECT DISTINCT cli_id,
             m = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
      FROM   #FIX_promo_daily_raw)
SELECT  CAST(NULL AS bigint)       AS con_id,
        d.cli_id,
        CAST(NULL AS nvarchar(40)) AS TSEGMENTNAME,
        SUM(d.out_rub)             AS out_rub,
        m.m                        AS dt_rep,
        MAX(d.rate_con)            AS rate_con
INTO   #FIX_promo_glue
FROM   #FIX_promo_daily_raw d
JOIN   m1                        m
       ON m.cli_id = d.cli_id AND m.m = d.dt_rep
GROUP  BY d.cli_id,m.m;

/*----------------------------------------------------------------
 5. остальные дни promo
----------------------------------------------------------------*/
DROP TABLE IF EXISTS #FIX_promo_not1;
SELECT * INTO #FIX_promo_not1
FROM   #FIX_promo_daily_raw
WHERE  dt_rep <> DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1);

/*----------------------------------------------------------------
 6. дневная лента promo (финал) → WORK.Forecast_NS_Promo
----------------------------------------------------------------*/
DROP TABLE IF EXISTS WORK.Forecast_NS_Promo;
SELECT dt_rep,
       SUM(out_rub)                       out_rub_total,
       SUM(out_rub*rate_con)/SUM(out_rub) rate_avg
INTO  WORK.Forecast_NS_Promo
FROM (
      SELECT * FROM #FIX_promo_not1
      UNION ALL
      SELECT * FROM #FIX_promo_glue
     ) z
GROUP BY dt_rep;

/*----------------------------------------------------------------
 7. promo-rate каждого месяца → WORK.NS_PromoRates
----------------------------------------------------------------*/
DROP TABLE IF EXISTS WORK.NS_PromoRates;
CREATE TABLE WORK.NS_PromoRates(
  month_first date,
  TSEGMENTNAME nvarchar(40),
  promo_rate   decimal(9,4),
  PRIMARY KEY(month_first,TSEGMENTNAME));

;WITH month_key AS (
      SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) m1,
             MIN(KEY_RATE) key_min
      FROM   #key
      GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1))
INSERT WORK.NS_PromoRates
SELECT mk.m1, s.TSEGMENTNAME,
       mk.key_min + s.spread
FROM   month_key mk
JOIN   WORK.NS_Spreads s ON 1=1;

/*----------------------------------------------------------------
 8. TOTAL-портфель = FLOAT + FIX_base + FIX_promo
----------------------------------------------------------------*/
TRUNCATE TABLE WORK.Forecast_BalanceDaily_NS;
INSERT WORK.Forecast_BalanceDaily_NS
SELECT  dt_rep,
        SUM(out_rub_total)                                        out_rub_total,
        SUM(out_rub_total*rate_avg)/SUM(out_rub_total)            rate_avg
FROM (
      SELECT * FROM WORK.Forecast_NS_Float
      UNION ALL
      SELECT * FROM WORK.Forecast_NS_FixBase
      UNION ALL
      SELECT * FROM WORK.Forecast_NS_Promo
     ) q
GROUP BY dt_rep;

/*----------------------------------------------------------------
 9. контрольный вывод
----------------------------------------------------------------*/
PRINT N'=== spread (модель) ===';          SELECT * FROM WORK.NS_Spreads;
PRINT N'=== promo-rate by month ===';      SELECT * FROM WORK.NS_PromoRates ORDER BY month_first,TSEGMENTNAME;

PRINT N'=== sanity 30/31/01 ===';
SELECT dt_rep,out_rub_total,rate_avg
FROM   WORK.Forecast_BalanceDaily_NS
WHERE  dt_rep IN ('2025-08-30','2025-08-31','2025-09-01',
                  '2025-09-30','2025-10-01')
ORDER BY dt_rep;

PRINT N'=== portfolio TOP-40 ===';
SELECT TOP (40) * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep;
GO
