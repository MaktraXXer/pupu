ниже ― полный, заново-собранный скрипт (можно копировать “как есть”).
— таблицы WORK.NS_Spreads / NS_PromoRates / Forecast_BalanceDaily_NS целиком пересоздаются → гарантируется столбец TSEGMENTNAME;
— ошибка q.spread not in GROUP BY устранена: promo-ставки считаются через отдельный CTE min_key без лишних столбцов;
— спред рассчитывается строго как взвешенная ставка promo-счётов – spot-KEY на @Anchor (одна строка на каждый TSEGMENTNAME).

/*═════════════════════════════════════════════════════════════════════
   НАКОПИТЕЛЬНЫЕ СЧЁТА (NS)  –  FIX-promo (2 мес), FIX-base, FLOAT
   roll-over + “перелив” promo
   version 2025-08-07 h  — правка TSEGMENTNAME и promo-GROUP-BY
═════════════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
/*–– 0. параметры ––––––––––––––––––––––––––––––––––––––––––––––*/
DECLARE
    @scen      tinyint      = 1,             -- сценарий KEY
    @Anchor    date         = '2025-08-04',  -- последний факт-день
    @HorizonTo date         = '2025-12-31',  -- горизонт
    @BaseRate  decimal(9,4) = 0.0650;        -- базовая ставка

/*–– 1. итоговые таблицы (drop-create) ––––––––––––––––––––––––*/
IF OBJECT_ID('WORK.Forecast_BalanceDaily_NS','U') IS NOT NULL
    DROP TABLE WORK.Forecast_BalanceDaily_NS;
CREATE TABLE WORK.Forecast_BalanceDaily_NS(
  dt_rep date PRIMARY KEY,
  out_rub_total decimal(20,2),
  rate_avg      decimal(9,4)
);

IF OBJECT_ID('WORK.NS_Spreads','U') IS NOT NULL
    DROP TABLE WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads(
  anchor        date         NOT NULL,
  TSEGMENTNAME  nvarchar(40) NOT NULL,
  spread        decimal(9,4) NOT NULL,
  CONSTRAINT PK_NSSpreads PRIMARY KEY(anchor,TSEGMENTNAME)
);

IF OBJECT_ID('WORK.NS_PromoRates','U') IS NOT NULL
    DROP TABLE WORK.NS_PromoRates;
CREATE TABLE WORK.NS_PromoRates(
  month_first   date         NOT NULL,
  TSEGMENTNAME  nvarchar(40) NOT NULL,
  promo_rate    decimal(9,4) NOT NULL,
  CONSTRAINT PK_NSPromo PRIMARY KEY(month_first,TSEGMENTNAME)
);

/*–– 2. календарь ––––––––––––––––––––––––––––––––––––––––––––––*/
IF OBJECT_ID('tempdb..#cal') IS NOT NULL DROP TABLE #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal)<@HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/*–– 3. spot-KEY и KEY-open –––––––––––––––––––––––––––––––––––*/
IF OBJECT_ID('tempdb..#key_spot')  IS NOT NULL DROP TABLE #key_spot;
IF OBJECT_ID('tempdb..#key_open')  IS NOT NULL DROP TABLE #key_open;

SELECT fc.DT_REP,fc.KEY_RATE
INTO   #key_spot
FROM   WORK.ForecastKey_Cache_Scen fc
JOIN   #cal c ON c.d=fc.DT_REP
WHERE  fc.Scenario=@scen AND fc.TERM=1;

SELECT DT_REP,TERM,AVG_KEY_RATE
INTO   #key_open
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen
  AND  DT_REP BETWEEN '2000-01-01' AND @HorizonTo;

/*–– 4. факт-портфель NS –––––––––––––––––––––––––––––––––––––––*/
IF OBJECT_ID('tempdb..#bal_fact') IS NOT NULL DROP TABLE #bal_fact;

SELECT t.con_id,t.cli_id,t.prod_id,
       t.out_rub,t.rate_con,t.is_floatrate,t.termdays,
       CAST(t.dt_open  AS date) AS dt_open,
       CAST(t.dt_close AS date) AS dt_close,
       t.TSEGMENTNAME,
       t.conv
INTO   #bal_fact
FROM   ALM.ALM.vw_balance_rest_all t WITH(NOLOCK)
WHERE  t.dt_rep = @Anchor
  AND  t.section_name = N'Накопительный счёт'
  AND  t.block_name   = N'Привлечение ФЛ'
  AND  t.od_flag      = 1
  AND  t.cur          = '810'
  AND  t.out_rub IS NOT NULL
  AND (t.dt_close_fact IS NULL OR t.dt_close_fact >= @Anchor);

/*–– 5. расчёт спредов (promo-FIX за июль-август) –––––––––––––*/
;WITH promo_wavg AS (
    SELECT bf.TSEGMENTNAME,
           w_rate = SUM(bf.out_rub*bf.rate_con)/SUM(bf.out_rub)
    FROM   #bal_fact bf
    WHERE  bf.prod_id = 654
      AND  DATEDIFF(month,bf.dt_open,@Anchor) IN (0,1)
    GROUP BY bf.TSEGMENTNAME
)
INSERT INTO WORK.NS_Spreads
SELECT @Anchor,
       p.TSEGMENTNAME,
       p.w_rate - ks.KEY_RATE
FROM   promo_wavg p
CROSS  JOIN (SELECT KEY_RATE FROM #key_spot WHERE DT_REP=@Anchor) ks;

/*–– 6. рекурсивная цепочка promo-окон (2 мес.) –––––––––––––––*/
IF OBJECT_ID('tempdb..#promo_seq') IS NOT NULL DROP TABLE #promo_seq;
;WITH seed AS (
      SELECT con_id,cli_id,TSEGMENTNAME,out_rub,
             win_start = dt_open,
             win_end   = DATEADD(day,-1,EOMONTH(DATEADD(month,1,dt_open)))
      FROM   #bal_fact
      WHERE  prod_id = 654
        AND  DATEDIFF(month,dt_open,@Anchor) IN (0,1)
), seq AS (
      SELECT *, n=0 FROM seed
      UNION ALL
      SELECT s.con_id,s.cli_id,s.TSEGMENTNAME,s.out_rub,
             DATEADD(day,1,s.win_end),
             DATEADD(day,-1,EOMONTH(DATEADD(month,1,DATEADD(day,1,s.win_end)))),
             n+1
      FROM   seq s
      WHERE  DATEADD(day,1,s.win_end) <= @HorizonTo
)
SELECT * INTO #promo_seq FROM seq OPTION (MAXRECURSION 0);

/*–– 7. три слоя ставок ––––––––––––––––––––––––––––––––––––––––*/
-- 7.1 FIX-promo (+ день base)
IF OBJECT_ID('tempdb..#rate_fixed') IS NOT NULL DROP TABLE #rate_fixed;
SELECT p.con_id,p.cli_id,p.TSEGMENTNAME,p.out_rub,
       c.d AS dt_rep,
       rate_con = CASE
                     WHEN c.d <= p.win_end
                          THEN s.spread + ks.KEY_RATE
                     ELSE @BaseRate
                  END
INTO   #rate_fixed
FROM   #promo_seq p
JOIN   WORK.NS_Spreads s ON s.anchor=@Anchor
                        AND s.TSEGMENTNAME=p.TSEGMENTNAME
JOIN   #cal      c  ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
JOIN   #key_spot ks ON ks.DT_REP=c.d;

-- 7.2 FLOAT
IF OBJECT_ID('tempdb..#rate_float') IS NOT NULL DROP TABLE #rate_float;
SELECT f.con_id,f.cli_id,f.TSEGMENTNAME,f.out_rub,
       c.d AS dt_rep,
       rate_con = (f.rate_con - ks.KEY_RATE) + ks2.KEY_RATE
INTO   #rate_float
FROM   #bal_fact f
JOIN   #key_spot ks  ON ks.DT_REP=@Anchor
JOIN   #cal      c   ON c.d BETWEEN f.dt_open AND ISNULL(f.dt_close,@HorizonTo)
JOIN   #key_spot ks2 ON ks2.DT_REP=c.d
WHERE  f.prod_id=3103;

-- 7.3 FIX-base
IF OBJECT_ID('tempdb..#rate_base') IS NOT NULL DROP TABLE #rate_base;
SELECT b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
       c.d AS dt_rep,
       rate_con = b.rate_con
INTO   #rate_base
FROM   #bal_fact b
JOIN   #cal c ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
WHERE  b.prod_id=654
  AND  DATEDIFF(month,b.dt_open,@Anchor) >= 2;

/*–– 8. promo-ставка месяца (min KEY + spread) ––––––––––––––––*/
WITH min_key AS (
      SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) AS m1,
             MIN(KEY_RATE) AS min_key
      FROM   #key_spot
      GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1)
)
INSERT INTO WORK.NS_PromoRates
SELECT k.m1 , s.TSEGMENTNAME , k.min_key + s.spread  AS promo_rate
FROM   min_key k
CROSS  JOIN WORK.NS_Spreads s
WHERE  s.anchor=@Anchor;

/*–– 9. конструктор портфеля день-в-день ––––––––––––––––––––––*/
IF OBJECT_ID('tempdb..#all_raw') IS NOT NULL DROP TABLE #all_raw;

SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con
INTO   #all_raw
FROM (
      SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con FROM #rate_fixed
      UNION ALL
      SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con FROM #rate_float
      UNION ALL
      SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con FROM #rate_base
) z;

/* на 1-е число месяца клиент держит деньги на **одном** лучшем promo-FIX */
;WITH promo_1 AS (
      SELECT cli_id, DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1) AS m1
      FROM   #rate_fixed
      WHERE  dt_rep = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
      GROUP  BY cli_id,DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
)
, best AS (
      SELECT f.cli_id,
             f.dt_rep,
             SUM(out_rub)          AS out_rub,
             MAX(rate_con)         AS rate_con
      FROM   #rate_fixed f
      JOIN   promo_1 p ON p.cli_id=f.cli_id AND p.m1=f.dt_rep
      GROUP  BY f.cli_id,f.dt_rep
)
, union_all AS (
      SELECT NULL AS con_id, cli_id, out_rub, dt_rep, rate_con FROM best
      UNION ALL
      SELECT con_id,cli_id,out_rub,dt_rep,rate_con
      FROM   #all_raw d
      WHERE  NOT EXISTS (SELECT 1
                         FROM promo_1 p
                         WHERE p.cli_id=d.cli_id
                           AND p.m1   =DATEFROMPARTS(YEAR(d.dt_rep),MONTH(d.dt_rep),1))
)
INSERT INTO WORK.Forecast_BalanceDaily_NS(dt_rep,out_rub_total,rate_avg)
SELECT  dt_rep,
        SUM(out_rub),
        SUM(out_rub*rate_con)/SUM(out_rub)
FROM    union_all
GROUP  BY dt_rep;

/*–– 10. отчёт ––––––––––––––––––––––––––––––––––––––––––––––––*/
SELECT * FROM WORK.NS_Spreads;
SELECT * FROM WORK.NS_PromoRates ORDER BY month_first,TSEGMENTNAME;
SELECT * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep;

ключевые отличия

участок	что исправлено
таблицы NS_*	всегда DROP TABLE + CREATE, поэтому колонка TSEGMENTNAME существ-ет
§ 5 спред	взвешенная ставка промо-FIX – spot-KEY на @Anchor
§ 8 promo-ставки	CTE min_key ⇒ GROUP BY только m1; spread подставляется через CROSS JOIN, больше нет ошибки «q.spread not in GROUP BY»
портфель	“перелив” promo-FIX только 1-го числа, остальное остаётся без изменений

запускай скрипт целиком — он соберёт спреды, промо-ставки и дневной прогноз без компиляционных ошибок.
