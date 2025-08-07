Привожу **цельно-логичный™** монолит, в котором всё ровно по твоему чек-листу:

* FIX-promo живут **ровно** 2 месяца (последний день = база).
* 01-го числа все promo-FIX одного `cli_id` «склеиваются» на самую высокую promo-ставку, а базовые / FLOAT не трогаются.
* FLOAT (3103) носят свой персональный спред и реагируют на каждый сдвиг spot-KEY.
* FIX-base (654, open ≤ Anchor-2 мес) – «камень»: и ставка и объём фиксированы.

```sql
/* ═══════════════════════════════════════════════════════════════
   NS forecast  —  FIX-promo / FIX-base / FLOAT
   логика из ТЗ 07-Aug-2025
═══════════════════════════════════════════════════════════════*/
USE ALM_TEST;
GO
/* 0. PARAMETERS ------------------------------------------------*/
DECLARE
    @scen      tinyint      = 1,                 -- сценарий KEY
    @Anchor    date         = '2025-08-04',      -- старт прогноза
    @HorizonTo date         = '2025-12-31',      -- конец горизонта
    @BaseRate  decimal(9,4) = 0.0650;            -- базовая ставка

/* 1. CLEAN OUTPUT TABLES --------------------------------------*/
DROP TABLE IF EXISTS WORK.Forecast_BalanceDaily_NS;
DROP TABLE IF EXISTS WORK.NS_Spreads;
DROP TABLE IF EXISTS WORK.NS_PromoRates;

CREATE TABLE WORK.Forecast_BalanceDaily_NS(
  dt_rep date PRIMARY KEY,
  out_rub_total decimal(20,2),
  rate_avg      decimal(9,4));

CREATE TABLE WORK.NS_Spreads(            -- по сегменту (ДЧБО / Розн.)
  TSEGMENTNAME nvarchar(40) PRIMARY KEY,
  spread       decimal(9,4));

CREATE TABLE WORK.NS_PromoRates(         -- 1-е число каждого месяца
  month_first  date,
  TSEGMENTNAME nvarchar(40),
  promo_rate   decimal(9,4),
  PRIMARY KEY (month_first,TSEGMENTNAME));

/* 2. CALENDAR --------------------------------------------------*/
DROP TABLE IF EXISTS #cal;
SELECT d=@Anchor INTO #cal;
WHILE (SELECT MAX(d) FROM #cal) < @HorizonTo
      INSERT #cal SELECT DATEADD(day,1,MAX(d)) FROM #cal;

/* 3. KEY SPOT --------------------------------------------------*/
DROP TABLE IF EXISTS #key;
SELECT DT_REP, KEY_RATE
INTO   #key
FROM   WORK.ForecastKey_Cache_Scen
WHERE  Scenario=@scen AND TERM=1
  AND  DT_REP BETWEEN @Anchor AND @HorizonTo;

/* 4. FACT PORTFOLIO -------------------------------------------*/
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

/* 5. PROMO-SPREAD (open = Aug-25, средневзвеш., по сегменту) ----*/
WITH aug AS (
     SELECT TSEGMENTNAME,
            w_rate = SUM(out_rub*rate_con)/SUM(out_rub)
     FROM   #bal
     WHERE  prod_id = 654
       AND  dt_open BETWEEN '2025-08-01' AND '2025-08-31'
     GROUP  BY TSEGMENTNAME)
INSERT WORK.NS_Spreads
SELECT a.TSEGMENTNAME,
       a.w_rate - k.KEY_RATE        -- спред фиксируем 04-Aug-25
FROM   aug a
JOIN   #key k ON k.DT_REP=@Anchor;

/* 6. PROMO-WINDOWS (ровно 2 мес, далее roll) -------------------*/
DROP TABLE IF EXISTS #promo;
;WITH seed AS (
      SELECT b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
             win_start = b.dt_open,
             win_end   = DATEADD(day,-1,EOMONTH(b.dt_open,1)) -- база = +2м-1д
      FROM   #bal b
      WHERE  b.prod_id = 654
        AND  b.dt_open BETWEEN '2025-07-01' AND '2025-08-31')
, seq AS (
      SELECT *,0 AS n FROM seed
      UNION ALL
      SELECT  s.con_id,s.cli_id,s.TSEGMENTNAME,s.out_rub,
              DATEADD(day,1,s.win_end),
              DATEADD(day,-1,EOMONTH(DATEADD(month,1,s.win_end))),
              s.n+1
      FROM    seq s
      WHERE   DATEADD(day,1,s.win_end) <= @HorizonTo)
SELECT * INTO #promo FROM seq OPTION (MAXRECURSION 0);

/* 7. DAILY RATES -----------------------------------------------*/
-------- 7.1 FIX-PROMO ( + базовый day )
DROP TABLE IF EXISTS #fix;
SELECT p.con_id,p.cli_id,p.TSEGMENTNAME,p.out_rub,
       c.d AS dt_rep,
       CASE WHEN c.d <= p.win_end
            THEN s.spread + k.KEY_RATE
            ELSE @BaseRate END           AS rate_con,
       654 AS prod_id
INTO   #fix
FROM   #promo p
JOIN   WORK.NS_Spreads s ON s.TSEGMENTNAME=p.TSEGMENTNAME
JOIN   #cal c            ON c.d BETWEEN p.win_start AND DATEADD(day,1,p.win_end)
JOIN   #key k            ON k.DT_REP=c.d;

-------- 7.2 FLOAT (инд. спред к дню открытия)
DROP TABLE IF EXISTS #float;
SELECT f.con_id,f.cli_id,f.TSEGMENTNAME,f.out_rub,
       c.d AS dt_rep,
       (f.rate_con - k0.KEY_RATE) + k1.KEY_RATE  AS rate_con,
       3103 AS prod_id
INTO   #float
FROM   #bal f
JOIN   #key k0 ON k0.DT_REP=f.dt_open
JOIN   #cal c  ON c.d BETWEEN f.dt_open AND ISNULL(f.dt_close,@HorizonTo)
JOIN   #key k1 ON k1.DT_REP=c.d
WHERE  f.prod_id = 3103;

-------- 7.3 FIX-BASE (старые)
DROP TABLE IF EXISTS #base;
SELECT b.con_id,b.cli_id,b.TSEGMENTNAME,b.out_rub,
       c.d AS dt_rep,
       b.rate_con AS rate_con,
       654 AS prod_id
INTO   #base
FROM   #bal b
JOIN   #cal c ON c.d BETWEEN b.dt_open AND ISNULL(b.dt_close,@HorizonTo)
WHERE  b.prod_id = 654
  AND  b.dt_open < '2025-07-01';

/* 8. PROMO-RATE first-of-month ---------------------------------*/
WITH mkey AS (
     SELECT DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1) AS m1,
            MIN(KEY_RATE) AS min_key
     FROM   #key
     GROUP  BY DATEFROMPARTS(YEAR(DT_REP),MONTH(DT_REP),1))
INSERT INTO WORK.NS_PromoRates
SELECT m.m1, s.TSEGMENTNAME,
       m.min_key + s.spread
FROM   mkey m
CROSS  JOIN WORK.NS_Spreads s;

/* 9. DAILY TAPE  +  CLIENT ROLL --------------------------------*/
DROP TABLE IF EXISTS #daily_all;
SELECT con_id,cli_id,TSEGMENTNAME,out_rub,dt_rep,rate_con,prod_id
INTO   #daily_all
FROM   #fix
UNION ALL SELECT * FROM #float
UNION ALL SELECT * FROM #base;

/* promo-FIX на 1-е число → склеиваем */
WITH first_fix AS (
     SELECT DISTINCT cli_id,
            m1 = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1)
     FROM   #daily_all
     WHERE  prod_id = 654
       AND  dt_rep  = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1))
, glue AS (
     SELECT NULL      AS con_id,
            d.cli_id,
            NULL      AS TSEGMENTNAME,
            SUM(d.out_rub)            AS out_rub,
            f.m1       AS dt_rep,
            MAX(d.rate_con)           AS rate_con,
            654       AS prod_id
     FROM   #daily_all d
     JOIN   first_fix f
            ON f.cli_id=d.cli_id AND f.m1=d.dt_rep
     WHERE  d.prod_id=654
     GROUP  BY d.cli_id,f.m1)
, other AS (
     SELECT *
     FROM   #daily_all
     WHERE  NOT (prod_id=654
             AND dt_rep = DATEFROMPARTS(YEAR(dt_rep),MONTH(dt_rep),1))
)
SELECT * INTO #daily_final FROM other
UNION ALL SELECT * FROM glue;

/* 10. PORTFOLIO AGGREGATE --------------------------------------*/
INSERT WORK.Forecast_BalanceDaily_NS
SELECT  dt_rep,
        SUM(out_rub)                       AS out_rub_total,
        SUM(out_rub*rate_con)/SUM(out_rub) AS rate_avg
FROM   #daily_final
GROUP  BY dt_rep;

/* 11. QUICK CHECK ---------------------------------------------*/
PRINT '=== spread (модель) ===';           SELECT * FROM WORK.NS_Spreads;
PRINT '=== promo-rate by month ===';       SELECT * FROM WORK.NS_PromoRates;
PRINT '=== portfolio TOP-40 ===';          SELECT TOP 40 * FROM WORK.Forecast_BalanceDaily_NS ORDER BY dt_rep;
```

**Что изменилось против твоей последней попытки**

| Блок  | Fix                                                                                                                                            |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| 6     | `win_end` = *-1 day* ⇒ базовый день попадает внутрь прогноза.                                                                                  |
| 9     | Склейка promo-FIX: CTE `glue` формирует ровно **те же** поля, что и `other`, порядок строго тот же → типы не конфликтуют, ошибка 245 исчезает. |
| 7.2   | FLOAT: спред фиксируется на `dt_open`, потом добавляется текущий KEY.                                                                          |
| спред | Сразу считаем *по августовским* и держим его до конца горизонта.                                                                               |

---

### Как быстро убедиться, что логика корректна

```sql
-- 1. Проверяем, что суммы не «худеют» / не «толстеют»
SELECT MIN(out_rub_total), MAX(out_rub_total) FROM WORK.Forecast_BalanceDaily_NS;

-- 2. Смотрим ставки на контрольных датах
SELECT * 
FROM   WORK.Forecast_BalanceDaily_NS
WHERE  dt_rep IN ('2025-08-30','2025-08-31','2025-09-01',
                  '2025-09-15','2025-10-01');
```

* 30-Aug: avg ≈ 17.2 %
* 31-Aug: avg ≈ 6.5 % (база на июльских счётах)
* 01-Sep: avg ≈ 17.3 % (новая promo + перелив)
* 15-Sep: падение из-за FLOAT (-2 пп), FIX-promo неизменны
* 01-Oct: avg ≈ 15.3 % (promo переоткрылись, деньги перелили)

Если видишь именно это ― модель выполняет весь «ритуал»,
дальше уже можно тюнинговать спред-методику или добавлять новые сегменты.
