/* ══════════════════════════════════════════════════════════════
   NS-forecast : FIX-promo (2 мес), FIX-base, FLOAT
   rollover + перелив promo на 1-е число
   version 2025-08-07 m4  (ambiguous cli_id → префиксы, JOIN = *on* поля)
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
DECLARE
    @scen      tinyint      = 1,
    @Anchor    date         = '2025-08-04',
    @HorizonTo date         = '2025-12-31',
    @BaseRate  decimal(9,4) = 0.0650;

/* ── 1. итоговые таблицы ─────────────────────────────────────── */
DROP TABLE IF EXISTS WORK.Forecast_BalanceDaily_NS;
CREATE TABLE WORK.Forecast_BalanceDaily_NS(
  dt_rep date PRIMARY KEY,
  out_rub_total decimal(20,2),
  rate_avg      decimal(9,4));

DROP TABLE IF EXISTS WORK.NS_Spreads;
CREATE TABLE WORK.NS_Spreads(
  TSEGMENTNAME nvarchar(40) NOT NULL,
  spread       decimal(9,4) NOT NULL,
  CONSTRAINT PK_NSSpreads PRIMARY KEY(TSEGMENTNAME));

DROP TABLE IF EXISTS WORK.NS_PromoRates;
CREATE TABLE WORK.NS_PromoRates(
  month_first  date         NOT NULL,
  TSEGMENTNAME nvarchar(40) NOT NULL,
  promo_rate   decimal(9,4) NOT NULL,
  CONSTRAINT PK_NSPromo PRIMARY KEY(month_first,TSEGMENTNAME));

/* ── 2. календарь ────────────────────────────────────────────── */
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* ── 3. ключевая ставка spot ────────────────────────────────── */
DROP TABLE IF EXISTS #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/* ── 4. факт-портфель ───────────────────────────────────────── */
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

/* ── 5. spread (по сегменту) ────────────────────────────────── */
WITH aug AS (
     SELECT TSEGMENTNAME,
            w_rate = SUM(out_rub*rate_con)/SUM(out_rub)
     FROM   #bal
     WHERE  prod_id=654
       AND  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
     GROUP  BY TSEGMENTNAME)
INSERT WORK.NS_Spreads
SELECT a.TSEGMENTNAME,
       a.w_rate - k.KEY_RATE
FROM   aug a
JOIN   #key k ON k.DT_REP=@Anchor;

/* ── 6. promo-окна (2 мес + rollover) ───────────────────────── */
DROP TABLE IF EXISTS #promo;
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
SELECT * INTO #promo FROM seq OPTION (MAXRECURSION 0);

/* ── 7. ставки ──────────────────────────────────────────────── */
DROP TABLE IF EXISTS #fix;
SELECT p.con_id,p.cli_id,p.TSEGMENTNAME,p.out_rub,
       c.d AS dt_rep,
       CASE WHEN c.d<=p.win_end
            THEN s.spread + k.KEY_RATE
            ELSE @BaseRate END     AS rate_con,
       654 AS prod_id
INTO   #fix
FROM   #promo p
JOIN   WORK.NS_Spreads s ON s.TSEGMENTNAME=p.TSEGMENTNAME
JOIN   #cal c            ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
JOIN   #key k            ON k.DT_REP=c.d;

DROP TABLE IF EXISTS #float;
SELECT f.con_id,f.cli_id,f.TSEGMENTNAME,f.out_rub,
       c.d AS dt_rep,
       (f.rate_con - k0.KEY_RATE) + k1.KEY_RATE AS rate_con,
       3103 AS prod_id
INTO   #float
FROM   #bal f
JOIN   #key k0 ON k0.DT_REP=f.dt_open
JOIN   #cal c  ON c.d BETWEEN f.dt_open AND ISNULL(f.dt_close,@HorizonTo)
JOIN   #key k1 ON k1.DT_REP=c.d
WHERE  f.prod_id=3103;

DROP TABLE IF EXISTS #base;
SELECT b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
       c.d AS dt_rep,
       b.rate_con AS rate_con,
       654 AS prod_id
INTO   #base
FROM   #bal b
JOIN   #cal c ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
WHERE  b.prod_id=654
  AND  b.dt_open<'2025-07-01';

/* ── 8. promo-ставка каждого месяца ─────────────────────────── */
WITH mkey AS (
     SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) AS m1,
            MIN(KEY_RATE) min_key
     FROM   #key
     GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1))
INSERT INTO WORK.NS_PromoRates
SELECT m.m1, s.TSEGMENTNAME,
       m.min_key + s.spread
FROM   mkey m
CROSS  JOIN WORK.NS_Spreads s;

/* ── 9. дневная лента + перелив ─────────────────────────────── */
DROP TABLE IF EXISTS #daily_all;
SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con,prod_id
INTO   #daily_all
FROM   #fix
UNION ALL SELECT * FROM #float
UNION ALL SELECT * FROM #base;               -- поля совпадают

;WITH first_day AS (        -- FIX: префиксы ⇒ нет ambiguous
        SELECT DISTINCT d.cli_id,
               DATEFROMPARTS(YEAR(d.dt_rep),MONTH(d.dt_rep),1) AS m1
        FROM   #daily_all d
        WHERE  d.prod_id=654
          AND  d.dt_rep = DATEFROMPARTS(YEAR(d.dt_rep),MONTH(d.dt_rep),1))
, best AS (
        SELECT  d.cli_id, fd.m1,
                SUM(d.out_rub)            AS out_rub,
                MAX(d.rate_con)           AS rate_con
        FROM   #daily_all d
        JOIN   first_day fd
               ON fd.cli_id=d.cli_id
              AND fd.m1    =d.dt_rep
        WHERE  d.prod_id=654
        GROUP  BY d.cli_id,fd.m1)
, union_all AS (
        SELECT NULL AS con_id, cli_id,NULL AS TSEGMENTNAME,
               out_rub, m1 AS dt_rep, rate_con,
               NULL AS prod_id
        FROM   best
        UNION ALL
        SELECT d.con_id,d.cli_id,d.TSEGMENTNAME,d.out_rub,
               d.dt_rep,d.rate_con,d.prod_id
        FROM   #daily_all d
        WHERE  NOT EXISTS (SELECT 1
                           FROM first_day fd
                           WHERE fd.cli_id=d.cli_id
                             AND fd.m1   =d.dt_rep
                             AND d.prod_id=654))
INSERT WORK.Forecast_BalanceDaily_NS
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   union_all
GROUP  BY dt_rep;

/* ── 10. вывод ──────────────────────────────────────────────── */
PRINT '===== SPREAD (модель) =====';
SELECT * FROM WORK.NS_Spreads;

PRINT '===== PROMO-ставка по месяцам =====';
SELECT * FROM WORK.NS_PromoRates ORDER BY month_first,TSEGMENTNAME;

PRINT '===== Портфель (TOP 40) =====';
SELECT TOP (40) * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep;
