/* ══════════════════════════════════════════════════════════════
   NS-forecast — Часть 2
   FIX-promo-roll (июль-авг-25 / prod_id 654)
   исправлена ошибка "#key.KEY_RATE … GROUP BY"
   v.2025-08-07-promo-fix
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;

/*–– проверяем, что объекты из Ч-1 есть ––*/
IF OBJECT_ID('tempdb..#bal')          IS NULL OR
   OBJECT_ID('tempdb..#cal')          IS NULL OR
   OBJECT_ID('tempdb..#key')          IS NULL OR
   OBJECT_ID('tempdb..#FLOAT_daily')  IS NULL OR
   OBJECT_ID('tempdb..#FIX_base_daily') IS NULL
BEGIN
    RAISERROR(N'Сначала выполните Часть 1!',16,1);  RETURN;
END

/*–– ключ на дату Anchor как скаляр ––*/
DECLARE @KeyAnchor decimal(9,4);
SELECT  @KeyAnchor = KEY_RATE FROM #key WHERE DT_REP=@Anchor;

/*----------------------------------------------------------------
 1. promo-SPREAD (по август-25 открытиям)  → WORK.NS_Spreads
----------------------------------------------------------------*/
DROP TABLE IF EXISTS WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads(
  TSEGMENTNAME nvarchar(40) PRIMARY KEY,
  spread       decimal(9,4)
);

DECLARE @Spread_DChbo  decimal(9,4),
        @Spread_Retail decimal(9,4);

;WITH aug AS (
    SELECT TSEGMENTNAME,
           w_rate = SUM(out_rub*rate_con)/SUM(out_rub)
    FROM   #bal
    WHERE  prod_id=654
      AND  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
    GROUP  BY TSEGMENTNAME )
SELECT @Spread_DChbo   = MAX(CASE WHEN TSEGMENTNAME=N'ДЧБО'            THEN w_rate END) - @KeyAnchor,
       @Spread_Retail = MAX(CASE WHEN TSEGMENTNAME=N'Розничный бизнес' THEN w_rate END) - @KeyAnchor
FROM   aug;

INSERT WORK.NS_Spreads VALUES
       (N'ДЧБО',            @Spread_DChbo),
       (N'Розничный бизнес',@Spread_Retail);

/*----------------------------------------------------------------
 2. сетка promo-окон (2 месяца)  → #promo_win
----------------------------------------------------------------*/
DROP TABLE IF EXISTS #promo_win;
;WITH seed AS (
        SELECT b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
               win_start = b.dt_open,
               win_end   = EOMONTH(b.dt_open,1)
        FROM   #bal b
        WHERE  b.prod_id=654
          AND  b.dt_open BETWEEN '2025-07-01' AND '2025-08-31')
, seq AS (
        SELECT *,0 AS n FROM seed
        UNION ALL
        SELECT con_id,cli_id,TSEGMENTNAME,out_rub,
               DATEADD(day,1,win_end),
               EOMONTH(DATEADD(day,1,win_end),1),
               n+1
        FROM   seq
        WHERE  DATEADD(day,1,win_end)<=@HorizonTo)
SELECT * INTO #promo_win FROM seq OPTION (MAXRECURSION 0);

/*----------------------------------------------------------------
 3. raw-дневник promo + базовый день
----------------------------------------------------------------*/
DROP TABLE IF EXISTS #FIX_promo_daily_raw;

SELECT  p.con_id,p.cli_id,p.TSEGMENTNAME,p.out_rub,
        c.d AS dt_rep,
        rate_con = CASE
            WHEN c.d <  p.win_end  -- внутри promo-периода
                 THEN CASE p.TSEGMENTNAME
                        WHEN N'ДЧБО'            THEN @Spread_DChbo  + k.KEY_RATE
                        ELSE                          @Spread_Retail+ k.KEY_RATE
                      END
            WHEN c.d = p.win_end   -- последний день 2-го мес. → база
                 THEN @BaseRate
            WHEN c.d = DATEADD(day,1,p.win_end) -- 1-е число нового окна
                 THEN CASE p.TSEGMENTNAME
                        WHEN N'ДЧБО'            THEN @Spread_DChbo  + k.KEY_RATE
                        ELSE                          @Spread_Retail+ k.KEY_RATE
                      END
        END
INTO    #FIX_promo_daily_raw
FROM    #promo_win p
JOIN    #cal c  ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
JOIN    #key k  ON k.DT_REP = c.d;

/*----------------------------------------------------------------
 4. glue перелива 1-го числа
----------------------------------------------------------------*/
DROP TABLE IF EXISTS #FIX_promo_daily_glued;

;WITH first_day AS (
      SELECT DISTINCT cli_id,
             m1 = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
      FROM   #FIX_promo_daily_raw )
SELECT  NULL con_id,
        d.cli_id,
        NULL TSEGMENTNAME,
        SUM(d.out_rub) out_rub,
        fd.m1          dt_rep,
        MAX(d.rate_con) rate_con
INTO    #FIX_promo_daily_glued
FROM   #FIX_promo_daily_raw d
JOIN   first_day fd
       ON fd.cli_id=d.cli_id AND fd.m1=d.dt_rep
GROUP  BY d.cli_id,fd.m1;

/*----------------------------------------------------------------
 5. non-first-day promo строки
----------------------------------------------------------------*/
DROP TABLE IF EXISTS #FIX_promo_daily_not1;
SELECT *
INTO   #FIX_promo_daily_not1
FROM   #FIX_promo_daily_raw
WHERE  dt_rep <> DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1);

/*----------------------------------------------------------------
 6. итоговая promo-лента
----------------------------------------------------------------*/
DROP TABLE IF EXISTS #FIX_promo_daily_final;
SELECT * INTO #FIX_promo_daily_final
FROM (
      SELECT * FROM #FIX_promo_daily_not1
      UNION ALL
      SELECT * FROM #FIX_promo_daily_glued
) z;

/*----------------------------------------------------------------
 7. агрегат promo  (WORK.Forecast_NS_Promo)
----------------------------------------------------------------*/
DROP TABLE IF EXISTS WORK.Forecast_NS_Promo;
SELECT  dt_rep,
        SUM(out_rub)                       out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) rate_avg
INTO    WORK.Forecast_NS_Promo
FROM   #FIX_promo_daily_final
GROUP  BY dt_rep;

/*----------------------------------------------------------------
 8. итог TOTAL = FLOAT + FIX-base + FIX-promo
----------------------------------------------------------------*/
DROP TABLE IF EXISTS WORK.Forecast_BalanceDaily_NS;
SELECT  f.dt_rep,
        (f.out_rub_total + b.out_rub_total + p.out_rub_total)            AS out_rub_total,
        (f.out_rub_total*f.rate_avg +
         b.out_rub_total*b.rate_avg +
         p.out_rub_total*p.rate_avg)
        /(f.out_rub_total + b.out_rub_total + p.out_rub_total)           AS rate_avg
INTO    WORK.Forecast_BalanceDaily_NS
FROM   WORK.Forecast_NS_Float    f
JOIN   WORK.Forecast_NS_FixBase  b ON b.dt_rep=f.dt_rep
JOIN   WORK.Forecast_NS_Promo    p ON p.dt_rep=f.dt_rep;

/*----------------------------------------------------------------
 9. promo-ставка по месяцам для контроля
----------------------------------------------------------------*/
DROP TABLE IF EXISTS WORK.NS_PromoRates;
CREATE TABLE WORK.NS_PromoRates(
  month_first  date,
  TSEGMENTNAME nvarchar(40),
  promo_rate   decimal(9,4),
  CONSTRAINT PK_month_Tseg PRIMARY KEY(month_first,TSEGMENTNAME)
);

;WITH month_key AS (
        SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) m1,
               MIN(KEY_RATE) key_min
        FROM   #key
        GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1))
INSERT WORK.NS_PromoRates
SELECT mk.m1, s.TSEGMENTNAME,
       mk.key_min +
       CASE s.TSEGMENTNAME
            WHEN N'ДЧБО'            THEN @Spread_DChbo
            ELSE                          @Spread_Retail
       END
FROM   month_key mk
CROSS  JOIN (VALUES (N'ДЧБО'),(N'Розничный бизнес')) s(TSEGMENTNAME);

/*----------------------------------------------------------------
 10. контрольный вывод
----------------------------------------------------------------*/
PRINT N'=== spread (модель) ===';
SELECT * FROM WORK.NS_Spreads;

PRINT N'=== promo-rate by month ===';
SELECT * FROM WORK.NS_PromoRates ORDER BY month_first,TSEGMENTNAME;

PRINT N'=== sanity 30/31/01 ===';
SELECT dt_rep,out_rub_total,rate_avg
FROM   WORK.Forecast_BalanceDaily_NS
WHERE  dt_rep IN ('2025-08-30','2025-08-31',
                  '2025-09-01','2025-09-30',
                  '2025-10-01')
ORDER BY dt_rep;

PRINT N'=== portfolio TOP-40 ===';
SELECT TOP (40) * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep;
GO
