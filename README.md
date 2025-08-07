/* ════════════════════════════════════════
     NS forecast: FIX-promo (2 мес), FIX-base, FLOAT
     version 2025-08-07 m
════════════════════════════════════════ */

USE ALM_TEST;
GO
/* 0. Параметры */
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;

/* 1. Чистим / создаём итоговые таблицы */
DROP TABLE IF EXISTS WORK.Forecast_BalanceDaily_NS;
CREATE TABLE WORK.Forecast_BalanceDaily_NS(
  dt_rep date PRIMARY KEY,
  out_rub_total decimal(20,2),
  rate_avg      decimal(9,4));

DROP TABLE IF EXISTS WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads(            -- фиксируем на dt_open!
  dt_open      date         NOT NULL,
  TSEGMENTNAME nvarchar(40) NOT NULL,
  spread       decimal(9,4) NOT NULL,
  CONSTRAINT PK_NSSpreads PRIMARY KEY(dt_open,TSEGMENTNAME));

DROP TABLE IF EXISTS WORK.NS_PromoRates;
CREATE TABLE WORK.NS_PromoRates(
  month_first  date         NOT NULL,
  TSEGMENTNAME nvarchar(40) NOT NULL,
  promo_rate   decimal(9,4) NOT NULL,
  CONSTRAINT PK_NSPromo PRIMARY KEY(month_first,TSEGMENTNAME));

/* 2. Календарь горизонта */
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* 3. Ключевая ставка */
DROP TABLE IF EXISTS #key_spot;
SELECT fc.DT_REP,fc.KEY_RATE
INTO   #key_spot
FROM   WORK.ForecastKey_Cache_Scen fc
JOIN   #cal c ON c.d = fc.DT_REP
WHERE  fc.Scenario=@scen AND fc.TERM=1;

/* 4. Факт-портфель НС */
DROP TABLE IF EXISTS #bal_fact;
SELECT  t.con_id,t.cli_id,t.prod_id,
        t.out_rub,t.rate_con,t.is_floatrate,
        CAST(t.dt_open  AS date) AS dt_open,
        CAST(t.dt_close AS date) AS dt_close,
        t.TSEGMENTNAME,
        t.conv
INTO    #bal_fact
FROM    ALM.ALM.vw_balance_rest_all t WITH (NOLOCK)
WHERE   t.dt_rep = @Anchor
  AND   t.section_name = N'Накопительный счёт'
  AND   t.block_name   = N'Привлечение ФЛ'
  AND   t.od_flag      = 1
  AND   t.cur          = '810'
  AND   t.out_rub IS NOT NULL
  AND  (t.dt_close_fact IS NULL OR t.dt_close_fact >= @Anchor);

/* 5. Spread promo-FIX (только август-2025 открытие!) */
;WITH src AS (
     SELECT bf.dt_open, bf.TSEGMENTNAME,
            w_rate = SUM(bf.out_rub*bf.rate_con)/SUM(bf.out_rub)
     FROM   #bal_fact bf
     WHERE  bf.prod_id = 654
       AND  bf.dt_open BETWEEN '2025-08-01' AND '2025-08-31'
     GROUP  BY bf.dt_open,bf.TSEGMENTNAME)
INSERT INTO WORK.NS_Spreads
SELECT s.dt_open,
       s.TSEGMENTNAME,
       s.w_rate - ks.KEY_RATE
FROM   src   s
JOIN   #key_spot ks ON ks.DT_REP = s.dt_open;     -- ключ на день открытия

/* 6. Promo-цепочки (2 полных мес. + rollover) */
DROP TABLE IF EXISTS #promo_seq;
;WITH seed AS (
     SELECT bf.con_id,bf.cli_id,bf.TSEGMENTNAME,bf.out_rub,
            win_start = bf.dt_open,
            win_end   = DATEADD(day,-1,EOMONTH(DATEADD(month,1,bf.dt_open))),
            bf.dt_open AS id_open
     FROM   #bal_fact bf
     WHERE  bf.prod_id = 654
       AND  bf.dt_open BETWEEN '2025-08-01' AND '2025-08-31'
), seq AS (
     SELECT *,0 AS n FROM seed
     UNION ALL
     SELECT s.con_id,s.cli_id,s.TSEGMENTNAME,s.out_rub,
            DATEADD(day,1,s.win_end),
            DATEADD(day,-1,EOMONTH(DATEADD(month,1,DATEADD(day,1,s.win_end)))),
            s.id_open, n+1
     FROM   seq s
     WHERE  DATEADD(day,1,s.win_end) <= @HorizonTo)
SELECT * INTO #promo_seq FROM seq OPTION (MAXRECURSION 0);

/* 7. Ставки */
---- FIX-promo (+ базовый день)
DROP TABLE IF EXISTS #rate_fixed;
SELECT p.con_id,p.cli_id,p.TSEGMENTNAME,p.out_rub,
       c.d AS dt_rep,
       rate_con = CASE
                     WHEN c.d <= p.win_end
                          THEN sp.spread + ks.KEY_RATE
                     ELSE @BaseRate
                  END
INTO   #rate_fixed
FROM   #promo_seq p
JOIN   WORK.NS_Spreads sp
          ON sp.dt_open = p.id_open AND sp.TSEGMENTNAME = p.TSEGMENTNAME
JOIN   #cal c  ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
JOIN   #key_spot ks ON ks.DT_REP = c.d;

---- FLOAT
DROP TABLE IF EXISTS #rate_float;
SELECT f.con_id,f.cli_id,f.TSEGMENTNAME,f.out_rub,
       c.d AS dt_rep,
       rate_con = (f.rate_con - ks.KEY_RATE) + ks2.KEY_RATE
INTO   #rate_float
FROM   #bal_fact f
JOIN   #key_spot ks  ON ks.DT_REP = f.dt_open
JOIN   #cal      c   ON c.d BETWEEN f.dt_open AND ISNULL(f.dt_close,@HorizonTo)
JOIN   #key_spot ks2 ON ks2.DT_REP = c.d
WHERE  f.prod_id = 3103;

---- FIX-base
DROP TABLE IF EXISTS #rate_base;
SELECT b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
       c.d AS dt_rep,
       rate_con = b.rate_con
INTO   #rate_base
FROM   #bal_fact b
JOIN   #cal c ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
WHERE  b.prod_id = 654
  AND  b.dt_open <  '2025-08-01';      -- старые FIX

/* 8. Promo-rate месяца */
WITH min_key AS (
     SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) AS m1,
            MIN(KEY_RATE) AS min_key
     FROM   #key_spot
     GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1))
INSERT INTO WORK.NS_PromoRates
SELECT k.m1, s.TSEGMENTNAME,
       k.min_key + s.spread
FROM   min_key k
JOIN   (SELECT DISTINCT TSEGMENTNAME,spread FROM WORK.NS_Spreads) s
     ON 1=1;

/* 9. День-в-день: убираем двойной учёт promo */
DROP TABLE IF EXISTS #all_rates;
SELECT * INTO #all_rates FROM (
     SELECT * FROM #rate_fixed
     UNION ALL SELECT * FROM #rate_float
     UNION ALL SELECT * FROM #rate_base) q;

;WITH ranked AS (
     SELECT *, ROW_NUMBER() OVER
                (PARTITION BY cli_id,dt_rep ORDER BY rate_con DESC) AS rn
     FROM   #all_rates)
INSERT INTO WORK.Forecast_BalanceDaily_NS
SELECT  dt_rep,
        SUM(out_rub)                           AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub)     AS rate_avg
FROM   ranked
WHERE  rn = 1                                  -- «перелили» всё на макс. ставку
GROUP  BY dt_rep;

/* 10. Проверка */
SELECT * FROM WORK.NS_Spreads           ORDER BY dt_open,TSEGMENTNAME;
SELECT * FROM WORK.NS_PromoRates        ORDER BY month_first,TSEGMENTNAME;
SELECT * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep;
