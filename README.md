/* ═══════════════════════════════════════════════════════════════════
   NS-forecast :  Ч А С Т Ь   2
   Только FIX-promo-roll (июль-авг-2025, код 654)
   — 2-месячное окно
   — базовый день (последний день 2-го месяца)
   — 1-е число → новая promo + перелив “на лучший счёт”
   Итог: #FIX_promo_daily_final
         + объединение с #FLOAT_daily, #FIX_base_daily
   v.2025-08-07 promo-part
═══════════════════════════════════════════════════════════════════*/

USE ALM_TEST;
GO
/* — те же параметры, что и в ч-1 — */
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;

/*------------------------------------------------------------------
  0. проверяем, что ключевые темп-таблицы из ч-1 в памяти
-------------------------------------------------------------------*/
IF OBJECT_ID('tempdb..#bal')          IS NULL OR
   OBJECT_ID('tempdb..#cal')          IS NULL OR
   OBJECT_ID('tempdb..#key')          IS NULL OR
   OBJECT_ID('tempdb..#FLOAT_daily')  IS NULL OR
   OBJECT_ID('tempdb..#FIX_base_daily') IS NULL
BEGIN
    RAISERROR(N'Не найдены объекты ч-1. Сначала выполните первую часть!',
              16, 1);
    RETURN;
END

/*------------------------------------------------------------------
  1. promo-SPREAD (по август-25 открытиям)   → переменные и таблица
-------------------------------------------------------------------*/
DROP TABLE IF EXISTS WORK.NS_Spreads;         -- перезаписываем
CREATE TABLE WORK.NS_Spreads(
  TSEGMENTNAME nvarchar(40) PRIMARY KEY,
  spread       decimal(9,4) NOT NULL
);

DECLARE @Spread_DChbo  decimal(9,4),
        @Spread_Retail decimal(9,4);

;WITH aug AS (
    SELECT TSEGMENTNAME,
           w_rate = SUM(out_rub*rate_con)/SUM(out_rub)
    FROM   #bal
    WHERE  prod_id = 654
      AND  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
    GROUP  BY TSEGMENTNAME)
SELECT @Spread_DChbo  = MAX(CASE WHEN TSEGMENTNAME=N'ДЧБО'            THEN w_rate END) - k.KEY_RATE,
       @Spread_Retail= MAX(CASE WHEN TSEGMENTNAME=N'Розничный бизнес' THEN w_rate END) - k.KEY_RATE
FROM   aug
CROSS  JOIN #key k
WHERE  k.DT_REP = @Anchor;

INSERT WORK.NS_Spreads VALUES
        (N'ДЧБО',            @Spread_DChbo),
        (N'Розничный бизнес',@Spread_Retail);

/*------------------------------------------------------------------
  2. сетка promo-окон  (2 месяца)  → #promo_win
-------------------------------------------------------------------*/
DROP TABLE IF EXISTS #promo_win;

;WITH seed AS (
      SELECT b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
             win_start = b.dt_open,
             win_end   = EOMONTH(b.dt_open,1)          -- конец 2-го мес.
      FROM   #bal b
      WHERE  b.prod_id = 654
        AND  b.dt_open BETWEEN '2025-07-01' AND '2025-08-31')
, seq AS (
      SELECT *,0 AS n FROM seed
      UNION ALL
      SELECT con_id,cli_id,TSEGMENTNAME,out_rub,
             DATEADD(day,1,win_end),                     -- новый старт
             EOMONTH(DATEADD(day,1,win_end),1),         -- плюс два мес.
             n+1
      FROM   seq
      WHERE  DATEADD(day,1,win_end) <= @HorizonTo)
SELECT * INTO #promo_win FROM seq OPTION (MAXRECURSION 0);

/*------------------------------------------------------------------
  3. raw-дневник promo + базовый день
-------------------------------------------------------------------*/
DROP TABLE IF EXISTS #FIX_promo_daily_raw;

SELECT  p.con_id,p.cli_id,p.TSEGMENTNAME,p.out_rub,
        c.d AS dt_rep,
        rate_con = CASE
            WHEN c.d <  p.win_end          -- первый месяц + второй (до конца)
                 THEN CASE p.TSEGMENTNAME
                          WHEN N'ДЧБО'            THEN @Spread_DChbo  + k.KEY_RATE
                          ELSE                          @Spread_Retail+ k.KEY_RATE
                      END
            WHEN c.d = p.win_end            -- ПОСЛЕДНИЙ день окна ⇒ база
                 THEN @BaseRate
            WHEN c.d = DATEADD(day,1,p.win_end) -- 1-е число нового окна
                 THEN CASE p.TSEGMENTNAME
                          WHEN N'ДЧБО'            THEN @Spread_DChbo  + k.KEY_RATE
                          ELSE                          @Spread_Retail+ k.KEY_RATE
                      END
            ELSE NULL   -- не должно случиться
        END
INTO    #FIX_promo_daily_raw
FROM    #promo_win p
JOIN    #cal c ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
JOIN    #key k ON k.DT_REP = CASE
                              WHEN c.d = DATEADD(day,1,p.win_end)
                                   THEN c.d                 -- KEY 1-го числа
                              ELSE p.win_start              -- KEY при открытии
                             END;

/*------------------------------------------------------------------
  4. перелив promo 1-го числа: glue-агрегация
-------------------------------------------------------------------*/
DROP TABLE IF EXISTS #FIX_promo_daily_glued;

;WITH first_day AS (
        SELECT DISTINCT cli_id,
               m1 = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
        FROM   #FIX_promo_daily_raw )
SELECT  NULL AS con_id,
        d.cli_id,
        NULL AS TSEGMENTNAME,
        SUM(d.out_rub)                     AS out_rub,
        fd.m1                              AS dt_rep,
        MAX(d.rate_con)                    AS rate_con
INTO    #FIX_promo_daily_glued
FROM   #FIX_promo_daily_raw d
JOIN   first_day            fd
       ON fd.cli_id = d.cli_id
      AND fd.m1     = d.dt_rep
GROUP  BY d.cli_id, fd.m1;

/*------------------------------------------------------------------
  5. non-first-day строки promo
-------------------------------------------------------------------*/
DROP TABLE IF EXISTS #FIX_promo_daily_not1;

SELECT *
INTO   #FIX_promo_daily_not1
FROM   #FIX_promo_daily_raw d
WHERE  d.dt_rep <> DATEFROMPARTS(YEAR(d.dt_rep),MONTH(d.dt_rep),1);

/*------------------------------------------------------------------
  6. финальная лента promo  (#FIX_promo_daily_final)
-------------------------------------------------------------------*/
DROP TABLE IF EXISTS #FIX_promo_daily_final;

SELECT * INTO #FIX_promo_daily_final
FROM (
      SELECT * FROM #FIX_promo_daily_not1
      UNION ALL
      SELECT * FROM #FIX_promo_daily_glued
) z;

/*------------------------------------------------------------------
  7. склейка всех трёх лент и агрегат портфеля
-------------------------------------------------------------------*/
DROP TABLE IF EXISTS #PORTF_ALL;

SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con
INTO   #PORTF_ALL
FROM   #FLOAT_daily
UNION ALL
SELECT * FROM #FIX_base_daily
UNION ALL
SELECT * FROM #FIX_promo_daily_final;

TRUNCATE TABLE WORK.Forecast_BalanceDaily_NS;
INSERT  WORK.Forecast_BalanceDaily_NS
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #PORTF_ALL
GROUP  BY dt_rep;

/*------------------------------------------------------------------
  8. promo-ставка по месяцам  (для контроля)  → WORK.NS_PromoRates
-------------------------------------------------------------------*/
DROP TABLE IF EXISTS WORK.NS_PromoRates;
CREATE TABLE WORK.NS_PromoRates(
  month_first  date PRIMARY KEY,
  TSEGMENTNAME nvarchar(40),
  promo_rate   decimal(9,4)
);

;WITH month_key AS (
        SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) AS m1,
               MIN(KEY_RATE) AS key_min
        FROM   #key
        GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1))
INSERT WORK.NS_PromoRates
SELECT mk.m1, s.TSEGMENTNAME,
       mk.key_min +
       CASE s.TSEGMENTNAME
            WHEN N'ДЧБО'            THEN @Spread_DChbo
            ELSE                          @Spread_Retail
       END AS promo_rate
FROM   month_key mk
CROSS  JOIN (VALUES (N'ДЧБО'),(N'Розничный бизнес')) s(TSEGMENTNAME);

/*------------------------------------------------------------------
  9. контрольный вывод
-------------------------------------------------------------------*/
PRINT N'=== spread (модель) ===';
SELECT * FROM WORK.NS_Spreads;

PRINT N'=== promo-rate by month ===';
SELECT * FROM WORK.NS_PromoRates ORDER BY month_first,TSEGMENTNAME;

PRINT N'=== sanity-check (30/31/01) ===';
SELECT dt_rep,out_rub_total,rate_avg
FROM   WORK.Forecast_BalanceDaily_NS
WHERE  dt_rep IN ('2025-08-30','2025-08-31',
                  '2025-09-01','2025-09-30',
                  '2025-10-01')
ORDER BY dt_rep;

PRINT N'=== portfolio TOP-40 ===';
SELECT TOP (40) * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep;
GO
