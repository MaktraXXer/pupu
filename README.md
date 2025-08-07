/* ════════════════════════════════════════════════════════════════════
     NS-forecast – FLOAT • FIX-base • FIX-promo-roll
     v. 2025-08-07-split-FINAL
   ═══════════════════════════════════════════════════════════════════ */

USE ALM_TEST;
GO
/*──────────────────────────── 0. параметры ──────────────────────────*/
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;

/*──────────────────────── 1. служебный календарь/KEY ───────────────*/
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

DROP TABLE IF EXISTS #key;            /* TERM = 1 → spot-KEY */
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/*──────────────────────── 2. исходный портфель на @Anchor ──────────*/
DROP TABLE IF EXISTS #bal;
SELECT  t.con_id ,
        t.cli_id ,
        t.prod_id ,
        t.out_rub ,
        t.rate_con ,
        t.is_floatrate ,
        CAST(t.dt_open  AS date) AS dt_open ,
        CAST(t.dt_close AS date) AS dt_close ,
        t.TSEGMENTNAME
INTO    #bal
FROM    ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE   t.dt_rep        = @Anchor
  AND   t.section_name  = N'Накопительный счёт'
  AND   t.block_name    = N'Привлечение ФЛ'
  AND   t.od_flag       = 1
  AND   t.cur           = '810'
  AND   t.out_rub IS NOT NULL
  AND  (t.dt_close_fact IS NULL OR t.dt_close_fact >= @Anchor);

/*====================================================================
   ❶  FLOAT  (prod_id 3103)   – спред фиксируется в dt_open
  ====================================================================*/
DROP TABLE IF EXISTS #FLOAT_daily;
SELECT  b.con_id ,
        b.cli_id ,
        b.TSEGMENTNAME ,
        b.out_rub ,
        c.d                                  AS dt_rep ,
        (b.rate_con-k0.KEY_RATE)+k1.KEY_RATE AS rate_con ,
        3103                                 AS prod_id
INTO    #FLOAT_daily
FROM    #bal b
JOIN    #key k0 ON k0.DT_REP = b.dt_open
JOIN    #cal c  ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
JOIN    #key k1 ON k1.DT_REP = c.d
WHERE   b.prod_id = 3103;

/*====================================================================
   ❷  FIX-base  (старые 654, ≤ июн-25)  – ставка константа
  ====================================================================*/
DROP TABLE IF EXISTS #FIX_base_daily;
SELECT  b.con_id ,
        b.cli_id ,
        b.TSEGMENTNAME ,
        b.out_rub ,
        c.d        AS dt_rep ,
        b.rate_con AS rate_con ,
        654        AS prod_id
INTO    #FIX_base_daily
FROM    #bal b
JOIN    #cal c ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
WHERE   b.prod_id = 654
  AND   b.dt_open < '2025-07-01';

/*====================================================================
   ❸  FIX-promo  (июл-авг-25 открытия)  – 2-месячный roll
  ====================================================================*/

/* ❸-1. единый spread по сегментам (считаем по открытым в августе-25) */
DECLARE @Spread_DChbo  decimal(9,4),
        @Spread_Retail decimal(9,4);

WITH aug AS (
     SELECT TSEGMENTNAME,
            w_rate = SUM(out_rub*rate_con)/SUM(out_rub)
     FROM   #bal
     WHERE  prod_id = 654
       AND  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
     GROUP  BY TSEGMENTNAME )
SELECT @Spread_DChbo  = MAX(CASE WHEN TSEGMENTNAME=N'ДЧБО'            THEN w_rate END) - k.KEY_RATE ,
       @Spread_Retail= MAX(CASE WHEN TSEGMENTNAME=N'Розничный бизнес' THEN w_rate END) - k.KEY_RATE
FROM   aug
CROSS  JOIN #key k
WHERE  k.DT_REP = @Anchor;

/* ❸-2. рекурсивная сетка promo-окон (2 мес) */
DROP TABLE IF EXISTS #promo_win;
;WITH seed AS (
      SELECT  b.con_id ,
              b.cli_id ,
              b.TSEGMENTNAME ,
              b.out_rub ,
              win_start = b.dt_open ,
              win_end   = EOMONTH(b.dt_open,1)
      FROM    #bal b
      WHERE   b.prod_id = 654
        AND   b.dt_open BETWEEN '2025-07-01' AND '2025-08-31')
, seq AS (
      SELECT *,0 AS n FROM seed
      UNION ALL
      SELECT  con_id ,cli_id ,TSEGMENTNAME ,out_rub ,
              DATEADD(day,1,win_end) ,
              EOMONTH(DATEADD(day,1,win_end),1) ,
              n+1
      FROM    seq
      WHERE   DATEADD(day,1,win_end) <= @HorizonTo)
SELECT * INTO #promo_win FROM seq OPTION (MAXRECURSION 0);

/* ❸-3. дневная лента promo + базовый день */
DROP TABLE IF EXISTS #FIX_promo_daily;
SELECT  p.con_id ,p.cli_id ,p.TSEGMENTNAME ,p.out_rub ,
        c.d AS dt_rep ,
        CASE
             WHEN c.d <= p.win_end
                  THEN CASE p.TSEGMENTNAME
                           WHEN N'ДЧБО'            THEN @Spread_DChbo  + k.KEY_RATE
                           ELSE                         @Spread_Retail + k.KEY_RATE
                       END
             /* базовый день = следующий за win_end */
             WHEN c.d = DATEADD(day,1,p.win_end)
                  THEN @BaseRate
        END AS rate_con ,
        654 AS prod_id
INTO    #FIX_promo_daily
FROM    #promo_win p
JOIN    #cal c  ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
JOIN    #key k  ON k.DT_REP = c.d;

/* ❸-4. перелив promo-денег 1-го числа */
--------------------------------------------------------------
/* 1-e число, где у клиента есть хотя бы один promo-FIX */
DROP TABLE IF EXISTS #p1;
SELECT DISTINCT cli_id,
       DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1) AS m1
INTO   #p1
FROM   #FIX_promo_daily
WHERE  dt_rep = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1);

/* агрегируем promo по max-ставке */
DROP TABLE IF EXISTS #p1_glue;
SELECT  CAST(NULL AS bigint)                  AS con_id ,
        p.cli_id ,
        CAST(NULL AS nvarchar(40))            AS TSEGMENTNAME ,
        SUM(f.out_rub)                        AS out_rub ,
        p.m1                                  AS dt_rep ,
        MAX(f.rate_con)                       AS rate_con ,
        654                                   AS prod_id
INTO    #p1_glue
FROM    #p1 p
JOIN    #FIX_promo_daily f
       ON  f.cli_id = p.cli_id
       AND f.dt_rep = p.m1
GROUP  BY p.cli_id ,p.m1;

/* прочие promo-дни (кроме 1-го) */
DROP TABLE IF EXISTS #p_not1;
SELECT * INTO #p_not1
FROM   #FIX_promo_daily
WHERE  dt_rep <> DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1);

/*====================================================================
   4. склейка всех daily-лент
  ====================================================================*/
DROP TABLE IF EXISTS #PORTF_ALL;
SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con
INTO   #PORTF_ALL
FROM   #FLOAT_daily
UNION  ALL
SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con
FROM   #FIX_base_daily
UNION  ALL
SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con
FROM   #p_not1
UNION  ALL
SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con
FROM   #p1_glue;

/*====================================================================
   5. итоговый агрегат портфеля
  ====================================================================*/
DROP TABLE IF EXISTS WORK.Forecast_BalanceDaily_NS;
CREATE TABLE WORK.Forecast_BalanceDaily_NS(
  dt_rep date PRIMARY KEY,
  out_rub_total decimal(20,2),
  rate_avg      decimal(9,4));

INSERT WORK.Forecast_BalanceDaily_NS
SELECT  dt_rep ,
        SUM(out_rub)                       AS out_rub_total ,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #PORTF_ALL
GROUP  BY dt_rep;

/*====================================================================
   6. контрольные данные для спреда / promo-ставок
  ====================================================================*/
IF OBJECT_ID('WORK.NS_Spreads','U') IS NULL
    CREATE TABLE WORK.NS_Spreads(
        TSEGMENTNAME nvarchar(40) PRIMARY KEY,
        spread       decimal(9,4));

IF OBJECT_ID('WORK.NS_PromoRates','U') IS NULL
    CREATE TABLE WORK.NS_PromoRates(
        month_first  date         NOT NULL,
        TSEGMENTNAME nvarchar(40) NOT NULL,
        promo_rate   decimal(9,4) NOT NULL,
        CONSTRAINT   PK_NSPromo PRIMARY KEY(month_first,TSEGMENTNAME));

TRUNCATE TABLE WORK.NS_Spreads;
INSERT  WORK.NS_Spreads
VALUES (N'ДЧБО',@Spread_DChbo),(N'Розничный бизнес',@Spread_Retail);

TRUNCATE TABLE WORK.NS_PromoRates;
WITH mkey AS (
     SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) AS m1,
            MIN(KEY_RATE) min_key
     FROM   #key
     GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1))
INSERT WORK.NS_PromoRates
SELECT m1,
       TSEGMENTNAME,
       min_key + CASE TSEGMENTNAME
                     WHEN N'ДЧБО'            THEN @Spread_DChbo
                     ELSE                         @Spread_Retail
                END
FROM   mkey
CROSS  JOIN (VALUES(N'ДЧБО'),(N'Розничный бизнес')) AS s(TSEGMENTNAME);

/*====================================================================
   7. sanity-check  (ступеньки promo-FIX + быстрая проверка)
  ====================================================================*/
PRINT N'╔═ sample (ступеньки promo-FIX) ══════════════════════════════';
SELECT dt_rep,out_rub_total,rate_avg
FROM   WORK.Forecast_BalanceDaily_NS
WHERE  dt_rep IN ('2025-08-30','2025-08-31',
                  '2025-09-01','2025-09-15',
                  '2025-09-30','2025-10-01')
ORDER  BY dt_rep;
PRINT N'╚═════════════════════════════════════════════════════════════';

PRINT N'=== spread (модель) ===';
SELECT * FROM WORK.NS_Spreads;

PRINT N'=== promo-rate by month ===';
SELECT * FROM WORK.NS_PromoRates ORDER BY month_first,TSEGMENTNAME;

PRINT N'=== portfolio TOP-40 ===';
SELECT TOP (40) *
FROM   WORK.Forecast_BalanceDaily_NS
ORDER  BY dt_rep;
GO
